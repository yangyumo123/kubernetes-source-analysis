kube-scheduler预选算法
=========================================================
## 简介
预选算法

## findNodesThatFit
含义：

    为每个node执行预选算法，不符合条件的就舍去，符合条件的返回。

路径：

    k8s.io/kubernetes/plugin/pkg/scheduler/core/generic_scheduler.go

定义：

    func findNodesThatFit(
        pod *v1.Pod,
        nodeNameToInfo map[string]*schedulercache.NodeInfo,
        nodes []*v1.Node,
        predicateFuncs map[string]algorithm.FitPredicate,
        extenders []algorithm.SchedulerExtender,
        metadataProducer algorithm.MetadataProducer,
        ecache *EquivalenceCache,
    )([]*v1.Node, FailedPredicateMap, error){
        //经过预选算法过滤后的nodes。
        var filtered []*v1.Node
        failedPredicateMap := FailedPredicateMap{}

        //如果预选算法函数为空，则直接把nodes列表返回。否则执行预选算法。
        if len(predicateFuncs) == 0 {
            filtered = nodes
        } else {
            // 创建足够大的slice，避免计算过程中需要增长空间。
            filtered = make([]*v1.Node, len(nodes))
            errs := errors.MessageCountMap{}
            var predicateResultLock sync.Mutex
            var filteredLen int32

            // 所有nodes使用相同的metadata
            meta := metadataProducer(pod, nodeNameToInfo)

            //为node执行预选算法，该函数类型作为下面的Parallelize参数传入，在Parallelize中执行。
            checkNode := func(i int) {
                nodeName := nodes[i].Name

                //为node执行预选算法
                fits, failedPredicates, err := podFitsOnNode(pod, meta, nodeNameToInfo[nodeName], predicateFuncs, ecache)
                if err != nil {
                    predicateResultLock.Lock()
                    errs[err.Error()]++
                    predicateResultLock.Unlock()
                    return
                }
                if fits {
                    filtered[atomic.AddInt32(&filteredLen, 1)-1] = nodes[i]
                } else {
                    predicateResultLock.Lock()
                    failedPredicateMap[nodeName] = failedPredicates
                    predicateResultLock.Unlock()
                }
            }

            //Parallelize函数很简单，就是为每个node启动一个goroutine来执行预选算法，但是最多只能启动16个goroutine。
            workqueue.Parallelize(16, len(nodes), checkNode)
            filtered = filtered[:filteredLen]
            if len(errs) > 0 {
                return []*v1.Node{}, FailedPredicateMap{}, errors.CreateAggregateFromMessageCountMap(errs)
            }
        }

        //如果设置了extender，则还需要执行extender的filter过滤node。
        if len(filtered) > 0 && len(extenders) != 0 {
            for _, extender := range extenders {

                //执行extender.Filter过滤node
                filteredList, failedMap, err := extender.Filter(pod, filtered, nodeNameToInfo)
                if err != nil {
                    return []*v1.Node{}, FailedPredicateMap{}, err
                }

                for failedNodeName, failedMsg := range failedMap {
                    if _, found := failedPredicateMap[failedNodeName]; !found {
                        failedPredicateMap[failedNodeName] = []algorithm.PredicateFailureReason{}
                    }
                    failedPredicateMap[failedNodeName] = append(failedPredicateMap[failedNodeName], predicates.NewFailureReason(failedMsg))
                }
                filtered = filteredList
                if len(filtered) == 0 {
                    break
                }
            }
        }
        return filtered, failedPredicateMap, nil
    }

### podFitsOnNode
含义：

    检查node是否满足所有的预选算法。

路径：

    k8s.io/kubernetes/plugin/pkg/scheduler/core/generic_scheduler.go

定义：

    func podFitsOnNode(pod *v1.Pod, meta interface{}, info *schedulercache.NodeInfo, predicateFuncs map[string]algorithm.FitPredicate,ecache *EquivalenceCache) (bool, []algorithm.PredicateFailureReason, error){
        var (
            equivalenceHash  uint64
            failedPredicates []algorithm.PredicateFailureReason
            eCacheAvailable  bool
            invalid          bool
            fit              bool
            reasons          []algorithm.PredicateFailureReason
            err              error
        )
        if ecache != nil {
            equivalenceHash = ecache.getHashEquivalencePod(pod)
            eCacheAvailable = (equivalenceHash != 0)
        }
        //执行每个预选算法
        for predicateKey, predicate := range predicateFuncs {
            if eCacheAvailable {
                // 如果缓存可用，则使用缓存的预选结果
                fit, reasons, invalid = ecache.PredicateWithECache(pod, info.Node().GetName(), predicateKey, equivalenceHash)
            }

            if !eCacheAvailable || invalid {
                // 如果缓存不可用，则要计算预选算法
                fit, reasons, err = predicate(pod, meta, info)
                if err != nil {
                    return false, []algorithm.PredicateFailureReason{}, err
                }

                if eCacheAvailable {
                    // 计算完预选算法后，更新缓存
                    ecache.UpdateCachedPredicateItem(pod, info.Node().GetName(), predicateKey, fit, reasons, equivalenceHash)
                }
            }

            if !fit {
                failedPredicates = append(failedPredicates, reasons...)
            }
        }
        return len(failedPredicates) == 0, failedPredicates, nil
    }

**所有的预选算法都定义在k8s.io/kubernetes/plugin/pkg/scheduler/algorithm/predicates/predicates.go中。下面介绍每种预选算法的含义。**

1. NoDiskConflict

含义：

    检查是否有磁盘冲突。如果在node上已经挂载了一个volume，则使用相同volume的其它Pod就不能被调度到这个node上。只用来检查GCEPersistentDisk、AWSElasticBlockStore、RBD和ISCSI这4种volume。

定义：

    func NoDiskConflict(pod *v1.Pod, meta interface{}, nodeInfo *schedulercache.NodeInfo) (bool, []algorithm.PredicateFailureReason, error) {
        for _, v := range pod.Spec.Volumes {
            for _, ev := range nodeInfo.Pods() {
                //如果GCEPersistentDisk、AWSElasticBlockStore、RBD和ISCSI都为nil，则返回false，即没有冲突。对应普通的volume来说是没有冲突的。
                if isVolumeConflict(v, ev) {
                    return false, []algorithm.PredicateFailureReason{ErrDiskConflict}, nil   //如果冲突，则返回错误原因。
                }
            }
        }
        return true, nil, nil
    }

2. PodFitsResources

含义：

    检查node是否满足pod的资源配额。检查计算资源：cpu、memory、gpu、StorageScratch、StorageOverlay、OpaqueIntResources。

定义：

    func PodFitsResources(pod *v1.Pod, meta interface{}, nodeInfo *schedulercache.NodeInfo) (bool, []algorithm.PredicateFailureReason, error) 
    {
        node := nodeInfo.Node()
        if node == nil {
            return false, nil, fmt.Errorf("node not found")
        }

        var predicateFails []algorithm.PredicateFailureReason
        allowedPodNumber := nodeInfo.AllowedPodNumber()
        if len(nodeInfo.Pods())+1 > allowedPodNumber {
            predicateFails = append(predicateFails, NewInsufficientResourceError(v1.ResourcePods, 1, int64(len(nodeInfo.Pods())), int64(allowedPodNumber)))
        }

        var podRequest *schedulercache.Resource
        if predicateMeta, ok := meta.(*predicateMetadata); ok {
            podRequest = predicateMeta.podRequest
        } else {
            // We couldn't parse metadata - fallback to computing it.
            podRequest = GetResourceRequest(pod)
        }
        if podRequest.MilliCPU == 0 && podRequest.Memory == 0 && podRequest.NvidiaGPU == 0 && len(podRequest.OpaqueIntResources) == 0 {
            return len(predicateFails) == 0, predicateFails, nil
        }

        //检查cpu、memory和gpu。
        allocatable := nodeInfo.AllocatableResource()
        //node上可分配的cpu必须大于pod资源限额requests中指定的cpu，才能通过；否则被淘汰。
        if allocatable.MilliCPU < podRequest.MilliCPU+nodeInfo.RequestedResource().MilliCPU {
            predicateFails = append(predicateFails, NewInsufficientResourceError(v1.ResourceCPU, podRequest.MilliCPU, nodeInfo.RequestedResource().MilliCPU, allocatable.MilliCPU))
        }
        //node上可分配的memory必须大于pod资源限额requests中指定的memory，才能通过；否则被淘汰。
        if allocatable.Memory < podRequest.Memory+nodeInfo.RequestedResource().Memory {
            predicateFails = append(predicateFails, NewInsufficientResourceError(v1.ResourceMemory, podRequest.Memory, nodeInfo.RequestedResource().Memory, allocatable.Memory))
        }
        //node上可分配的gpu必须大于pod资源限额requests中指定的gpu，才能通过；否则被淘汰。
        if allocatable.NvidiaGPU < podRequest.NvidiaGPU+nodeInfo.RequestedResource().NvidiaGPU {
            predicateFails = append(predicateFails, NewInsufficientResourceError(v1.ResourceNvidiaGPU, podRequest.NvidiaGPU, nodeInfo.RequestedResource().NvidiaGPU, allocatable.NvidiaGPU))
        }

        //检查存储。overlay是一种存储驱动。docker可以换成overlay存储驱动。
        scratchSpaceRequest := podRequest.StorageScratch
        if allocatable.StorageOverlay == 0 {
            scratchSpaceRequest += podRequest.StorageOverlay
            //scratchSpaceRequest += nodeInfo.RequestedResource().StorageOverlay
            nodeScratchRequest := nodeInfo.RequestedResource().StorageOverlay + nodeInfo.RequestedResource().StorageScratch
            if allocatable.StorageScratch < scratchSpaceRequest+nodeScratchRequest {
                predicateFails = append(predicateFails, NewInsufficientResourceError(v1.ResourceStorageScratch, scratchSpaceRequest, nodeScratchRequest, allocatable.StorageScratch))
            }

        } else if allocatable.StorageScratch < scratchSpaceRequest+nodeInfo.RequestedResource().StorageScratch {
            predicateFails = append(predicateFails, NewInsufficientResourceError(v1.ResourceStorageScratch, scratchSpaceRequest, nodeInfo.RequestedResource().StorageScratch, allocatable.StorageScratch))
        }
        if allocatable.StorageOverlay > 0 && allocatable.StorageOverlay < podRequest.StorageOverlay+nodeInfo.RequestedResource().StorageOverlay {
            predicateFails = append(predicateFails, NewInsufficientResourceError(v1.ResourceStorageOverlay, podRequest.StorageOverlay, nodeInfo.RequestedResource().StorageOverlay, allocatable.StorageOverlay))
        }

        //OpaqueIntResources也是一种计算资源
        for rName, rQuant := range podRequest.OpaqueIntResources {
            if allocatable.OpaqueIntResources[rName] < rQuant+nodeInfo.RequestedResource().OpaqueIntResources[rName] {
                predicateFails = append(predicateFails, NewInsufficientResourceError(rName, podRequest.OpaqueIntResources[rName], nodeInfo.RequestedResource().OpaqueIntResources[rName], allocatable.OpaqueIntResources[rName]))
            }
        }

        if glog.V(10) && len(predicateFails) == 0 {
            // We explicitly don't do glog.V(10).Infof() to avoid computing all the parameters if this is
            // not logged. There is visible performance gain from it.
            glog.Infof("Schedule Pod %+v on Node %+v is allowed, Node is running only %v out of %v Pods.",
                podName(pod), node.Name, len(nodeInfo.Pods()), allowedPodNumber)
        }
        return len(predicateFails) == 0, predicateFails, nil
    }

3. PodMatchNodeSelector

含义：

    检查node上的标签是否配置Pod中定义的nodeSelector。

定义：

    func PodMatchNodeSelector(pod *v1.Pod, meta interface{}, nodeInfo *schedulercache.NodeInfo) (bool, []algorithm.PredicateFailureReason, error){
        node:=nodeInfo.Node()
        if node==nil{
            return false, nil, fmt.Errorf("node not found")
        }
        if podMatchesNodeLabels(pod, node) {
            return true, nil, nil
        }
        return false, []algorithm.PredicateFailureReason{ErrNodeSelectorNotMatch}, nil
    }

4. PodFitsHost

含义：

    检查nodeName是否相同。

定义：

    func PodFitsHost(pod *v1.Pod, meta interface{}, nodeInfo *schedulercache.NodeInfo) (bool, []algorithm.PredicateFailureReason, error) {
        //PodSpec.NodeName为空，则通过。
        if len(pod.Spec.NodeName) == 0 {
            return true, nil, nil
        }
        node := nodeInfo.Node()
        if node == nil {
            return false, nil, fmt.Errorf("node not found")
        }

        //PodSpec.NodeName等于传入的node的name，则通过。
        if pod.Spec.NodeName == node.Name {
            return true, nil, nil
        }
        return false, []algorithm.PredicateFailureReason{ErrPodNotMatchHostName}, nil
    }

5. PodFitsHostPorts

含义：

    检查是否匹配node的端口。

定义：

    func PodFitsHostPorts(pod *v1.Pod, meta interface{}, nodeInfo *schedulercache.NodeInfo) (bool, []algorithm.PredicateFailureReason, error) 
    {
        var wantPorts map[int]bool
        if predicateMeta, ok := meta.(*predicateMetadata); ok {
            wantPorts = predicateMeta.podPorts
        } else {
            wantPorts = schedutil.GetUsedPorts(pod)
        }
        if len(wantPorts) == 0 {
            return true, nil, nil
        }

        //检查pod中定义的端口是否在node上已经使用了，如果已经使用，则不通过。
        existingPorts := nodeInfo.UsedPorts()
        for wport := range wantPorts {
            if wport != 0 && existingPorts[wport] {
                return false, []algorithm.PredicateFailureReason{ErrPodNotFitsHostPorts}, nil
            }
        }
        return true, nil, nil
    }

6. PodToleratesNodeTaints

含义：

    检查宽容node的污点，这是1.6引入的新特性，和亲和相关。

定义：

    func PodToleratesNodeTaints(pod *v1.Pod, meta interface{}, nodeInfo *schedulercache.NodeInfo) (bool, []algorithm.PredicateFailureReason, error) 
    {
        taints, err := nodeInfo.Taints()
        if err != nil {
            return false, nil, err
        }

        if v1helper.TolerationsTolerateTaintsWithFilter(pod.Spec.Tolerations, taints, func(t *v1.Taint) bool {
            return t.Effect == v1.TaintEffectNoSchedule || t.Effect == v1.TaintEffectNoExecute
        }) {
            return true, nil, nil
        }
        return false, []algorithm.PredicateFailureReason{ErrTaintsTolerationsNotMatch}, nil
    }

7. CheckNodeMemoryPressurePredicate

含义：

    检查内存压力情况。与服务级别QoS有关系。

定义：

    func CheckNodeMemoryPressurePredicate(pod *v1.Pod, meta interface{}, nodeInfo *schedulercache.NodeInfo) (bool, []algorithm.PredicateFailureReason, error) 
    {
        var podBestEffort bool
        if predicateMeta, ok := meta.(*predicateMetadata); ok {
            podBestEffort = predicateMeta.podBestEffort
        } else {
            podBestEffort = isPodBestEffort(pod)
        }
        // 非BestEffort pod通过。
        if !podBestEffort {
            return true, nil, nil
        }

        // is node under presure?
        if nodeInfo.MemoryPressureCondition() == v1.ConditionTrue {
            return false, []algorithm.PredicateFailureReason{ErrNodeUnderMemoryPressure}, nil
        }
        return true, nil, nil
    }

8. CheckNodeDiskPressurePredicate

含义：

    检查磁盘压力情况。

定义：

    func CheckNodeDiskPressurePredicate(pod *v1.Pod, meta interface{}, nodeInfo *schedulercache.NodeInfo) (bool, []algorithm.PredicateFailureReason, error) {
        if nodeInfo.DiskPressureCondition() == v1.ConditionTrue {
            return false, []algorithm.PredicateFailureReason{ErrNodeUnderDiskPressure}, nil
        }
        return true, nil, nil
    }

_______________________________________________________________________
[[返回kube-scheduler/kube-scheduler.md]](./kube-scheduler.md) 
