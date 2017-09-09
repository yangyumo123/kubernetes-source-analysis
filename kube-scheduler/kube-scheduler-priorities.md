kube-scheduler优选算法
=========================================================
## 简介
优选算法。所有的优选算法都在k8s.io/kubernetes/plugin/pkg/scheduler/algorithm/priorities/*.go中。在检查的同时会计算一个得分score。

## 1. BalancedResourceAllocationMap
含义：

    检查资源使用率。计算一个score得分。score=10-abs(cpuFraction-memoryFraction)*10。其中cpuFraction=请求cpu/可用cpu。memoryFraction=请求memory/可用memory。

定义：

    func BalancedResourceAllocationMap(pod *v1.Pod, meta interface{}, nodeInfo *schedulercache.NodeInfo) (schedulerapi.HostPriority, error) {
        var nonZeroRequest *schedulercache.Resource
        if priorityMeta, ok := meta.(*priorityMetadata); ok {
            nonZeroRequest = priorityMeta.nonZeroRequest
        } else {
            nonZeroRequest = getNonZeroRequests(pod)
        }
        //计算分数，检查是否通过。
        return calculateBalancedResourceAllocation(pod, nonZeroRequest, nodeInfo)
    }

## 2. ImageLocalityPriorityMap
含义：

    检查node上是否已经有待分配Pod需要的image，给出一个score，范围在0-10之间。如果不存在任何image，则给出最低值。如果存在部分image，数量最多的胜出。

定义：

    func ImageLocalityPriorityMap(pod *v1.Pod, meta interface{}, nodeInfo *schedulercache.NodeInfo) (schedulerapi.HostPriority, error) {
        node := nodeInfo.Node()
        if node == nil {
            return schedulerapi.HostPriority{}, fmt.Errorf("node not found")
        }

        var sumSize int64
        for i := range pod.Spec.Containers {
            sumSize += checkContainerImageOnNode(node, &pod.Spec.Containers[i])
        }
        return schedulerapi.HostPriority{
            Host:  node.Name,
            Score: calculateScoreFromSize(sumSize),
        }, nil
    }

## 3. LeastRequestedPriorityMap
含义：

    最小请求资源的node更容易胜出。score=（cpuScore+memoryScore）/2。其中cpuScore=(可用cpu-请求cpu)*10/可用cpu，memoryScore=(可用memory-请求memory)*10/可用memory。

定义：

    func LeastRequestedPriorityMap(pod *v1.Pod, meta interface{}, nodeInfo *schedulercache.NodeInfo) (schedulerapi.HostPriority, error) {
        var nonZeroRequest *schedulercache.Resource
        if priorityMeta, ok := meta.(*priorityMetadata); ok {
            nonZeroRequest = priorityMeta.nonZeroRequest
        } else {
            nonZeroRequest = getNonZeroRequests(pod)
        }
        return calculateUnusedPriority(pod, nonZeroRequest, nodeInfo)
    }

## 
含义：

    

定义：


## 
含义：

    

定义：


## 
含义：

    

定义：


## 
含义：

    

定义：


## 
含义：

    

定义：


## 
含义：

    

定义：


## 
含义：

    

定义：


## 
含义：

    

定义：


## 
含义：

    

定义：


## 
含义：

    

定义：


## 
含义：

    

定义：


## 
含义：

    

定义：


## 
含义：

    

定义：