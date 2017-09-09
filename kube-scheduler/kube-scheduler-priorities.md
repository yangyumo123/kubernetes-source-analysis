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

## 4. MostRequestedPriorityMap
含义：

    scope=(cpuScore+memoryScore)/2。其中cpuScore=10*所有请求cpu和/可用cpu，memoryScore=10*所有请求memory和/可用memory。

定义：

    func MostRequestedPriorityMap(pod *v1.Pod, meta interface{}, nodeInfo *schedulercache.NodeInfo) (schedulerapi.HostPriority, error) {
        var nonZeroRequest *schedulercache.Resource
        if priorityMeta, ok := meta.(*priorityMetadata); ok {
            nonZeroRequest = priorityMeta.nonZeroRequest
        } else {
            nonZeroRequest = getNonZeroRequests(pod)
        }
        return calculateUsedPriority(pod, nonZeroRequest, nodeInfo)
    }

## 5. CalculateNodeAffinityPriorityMap
含义：

    node亲和。

定义：

    func CalculateNodeAffinityPriorityMap(pod *v1.Pod, meta interface{}, nodeInfo *schedulercache.NodeInfo) (schedulerapi.HostPriority, error) {
        node := nodeInfo.Node()
        if node == nil {
            return schedulerapi.HostPriority{}, fmt.Errorf("node not found")
        }

        var affinity *v1.Affinity
        if priorityMeta, ok := meta.(*priorityMetadata); ok {
            affinity = priorityMeta.affinity
        } else {
            // We couldn't parse metadata - fallback to the podspec.
            affinity = schedulercache.ReconcileAffinity(pod)
        }

        var count int32
        // A nil element of PreferredDuringSchedulingIgnoredDuringExecution matches no objects.
        // An element of PreferredDuringSchedulingIgnoredDuringExecution that refers to an
        // empty PreferredSchedulingTerm matches all objects.
        if affinity != nil && affinity.NodeAffinity != nil && affinity.NodeAffinity.PreferredDuringSchedulingIgnoredDuringExecution != nil {
            // Match PreferredDuringSchedulingIgnoredDuringExecution term by term.
            for i := range affinity.NodeAffinity.PreferredDuringSchedulingIgnoredDuringExecution {
                preferredSchedulingTerm := &affinity.NodeAffinity.PreferredDuringSchedulingIgnoredDuringExecution[i]
                if preferredSchedulingTerm.Weight == 0 {
                    continue
                }

                // TODO: Avoid computing it for all nodes if this becomes a performance problem.
                nodeSelector, err := v1helper.NodeSelectorRequirementsAsSelector(preferredSchedulingTerm.Preference.MatchExpressions)
                if err != nil {
                    return schedulerapi.HostPriority{}, err
                }
                if nodeSelector.Matches(labels.Set(node.Labels)) {
                    count += preferredSchedulingTerm.Weight
                }
            }
        }

        return schedulerapi.HostPriority{
            Host:  node.Name,
            Score: int(count),
        }, nil
    }


## 6. CalculateNodeLabelPriorityMap
含义：

    node label存在，则满分。

定义：

    func (n *NodeLabelPrioritizer) CalculateNodeLabelPriorityMap(pod *v1.Pod, meta interface{}, nodeInfo *schedulercache.NodeInfo) (schedulerapi.HostPriority, error) {
        node := nodeInfo.Node()
        if node == nil {
            return schedulerapi.HostPriority{}, fmt.Errorf("node not found")
        }

        exists := labels.Set(node.Labels).Has(n.label)
        score := 0
        if (exists && n.presence) || (!exists && !n.presence) {
            score = 10
        }
        return schedulerapi.HostPriority{
            Host:  node.Name,
            Score: score,
        }, nil
    }

## 7. CalculateNodePreferAvoidPodsPriorityMap
含义：

    node prefer avoid pods.

定义：

    func CalculateNodePreferAvoidPodsPriorityMap(pod *v1.Pod, meta interface{}, nodeInfo *schedulercache.NodeInfo) (schedulerapi.HostPriority, error) {
        node := nodeInfo.Node()
        if node == nil {
            return schedulerapi.HostPriority{}, fmt.Errorf("node not found")
        }

        controllerRef := priorityutil.GetControllerRef(pod)
        if controllerRef != nil {
            // Ignore pods that are owned by other controller than ReplicationController
            // or ReplicaSet.
            if controllerRef.Kind != "ReplicationController" && controllerRef.Kind != "ReplicaSet" {
                controllerRef = nil
            }
        }
        if controllerRef == nil {
            return schedulerapi.HostPriority{Host: node.Name, Score: 10}, nil
        }

        avoids, err := v1helper.GetAvoidPodsFromNodeAnnotations(node.Annotations)
        if err != nil {
            // If we cannot get annotation, assume it's schedulable there.
            return schedulerapi.HostPriority{Host: node.Name, Score: 10}, nil
        }
        for i := range avoids.PreferAvoidPods {
            avoid := &avoids.PreferAvoidPods[i]
            if avoid.PodSignature.PodController.Kind == controllerRef.Kind && avoid.PodSignature.PodController.UID == controllerRef.UID {
                return schedulerapi.HostPriority{Host: node.Name, Score: 0}, nil
            }
        }
        return schedulerapi.HostPriority{Host: node.Name, Score: 10}, nil
    }

## 8. ComputeTaintTolerationPriorityMap
含义：

    ComputeTaintTolerationPriorityMap prepares the priority list for all the nodes based on the number of intolerable taints on the node

定义：

func ComputeTaintTolerationPriorityMap(pod *v1.Pod, meta interface{}, nodeInfo *schedulercache.NodeInfo) (schedulerapi.HostPriority, error) {
	node := nodeInfo.Node()
	if node == nil {
		return schedulerapi.HostPriority{}, fmt.Errorf("node not found")
	}
	// To hold all the tolerations with Effect PreferNoSchedule
	var tolerationsPreferNoSchedule []v1.Toleration
	if priorityMeta, ok := meta.(*priorityMetadata); ok {
		tolerationsPreferNoSchedule = priorityMeta.podTolerations

	} else {
		tolerationsPreferNoSchedule = getAllTolerationPreferNoSchedule(pod.Spec.Tolerations)
	}

	return schedulerapi.HostPriority{
		Host:  node.Name,
		Score: countIntolerableTaintsPreferNoSchedule(node.Spec.Taints, tolerationsPreferNoSchedule),
	}, nil
}

_______________________________________________________________________
[[返回kube-scheduler/kube-scheduler.md]](./kube-scheduler.md) 
