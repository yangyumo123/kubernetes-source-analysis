kube-scheduler配置的默认DefaultProvider
===============================================================
## 简介
kube-scheduler设置了默认的DefaultProvider，执行predicates和priorities。在k8s.io/kubernetes/plugin/pkg/scheduler/algorithmprovider/defaults/defaults.go中。

## init
含义：

    init函数应该在main之前执行，即在main之前就已经注册了各种调度算法相关数据结构。

定义：

    func init() {
        // Register functions that extract metadata used by predicates and priorities computations.
        factory.RegisterPredicateMetadataProducerFactory(
            func(args factory.PluginFactoryArgs) algorithm.MetadataProducer {
                return predicates.NewPredicateMetadataFactory(args.PodLister)
            })
        factory.RegisterPriorityMetadataProducerFactory(
            func(args factory.PluginFactoryArgs) algorithm.MetadataProducer {
                return priorities.PriorityMetadata
            })

        // Registers algorithm providers. By default we use 'DefaultProvider', but user can specify one to be used
        // by specifying flag.
        factory.RegisterAlgorithmProvider(factory.DefaultProvider, defaultPredicates(), defaultPriorities())
        // Cluster autoscaler friendly scheduling algorithm.
        factory.RegisterAlgorithmProvider(ClusterAutoscalerProvider, defaultPredicates(),
            copyAndReplace(defaultPriorities(), "LeastRequestedPriority", "MostRequestedPriority"))

        // Registers predicates and priorities that are not enabled by default, but user can pick when creating his
        // own set of priorities/predicates.

        // PodFitsPorts has been replaced by PodFitsHostPorts for better user understanding.
        // For backwards compatibility with 1.0, PodFitsPorts is registered as well.
        factory.RegisterFitPredicate("PodFitsPorts", predicates.PodFitsHostPorts)
        // Fit is defined based on the absence of port conflicts.
        // This predicate is actually a default predicate, because it is invoked from
        // predicates.GeneralPredicates()
        factory.RegisterFitPredicate("PodFitsHostPorts", predicates.PodFitsHostPorts)
        // Fit is determined by resource availability.
        // This predicate is actually a default predicate, because it is invoked from
        // predicates.GeneralPredicates()
        factory.RegisterFitPredicate("PodFitsResources", predicates.PodFitsResources)
        // Fit is determined by the presence of the Host parameter and a string match
        // This predicate is actually a default predicate, because it is invoked from
        // predicates.GeneralPredicates()
        factory.RegisterFitPredicate("HostName", predicates.PodFitsHost)
        // Fit is determined by node selector query.
        factory.RegisterFitPredicate("MatchNodeSelector", predicates.PodMatchNodeSelector)

        // Use equivalence class to speed up predicates & priorities
        factory.RegisterGetEquivalencePodFunction(GetEquivalencePod)

        // ServiceSpreadingPriority is a priority config factory that spreads pods by minimizing
        // the number of pods (belonging to the same service) on the same node.
        // Register the factory so that it's available, but do not include it as part of the default priorities
        // Largely replaced by "SelectorSpreadPriority", but registered for backward compatibility with 1.0
        factory.RegisterPriorityConfigFactory(
            "ServiceSpreadingPriority",
            factory.PriorityConfigFactory{
                Function: func(args factory.PluginFactoryArgs) algorithm.PriorityFunction {
                    return priorities.NewSelectorSpreadPriority(args.ServiceLister, algorithm.EmptyControllerLister{}, algorithm.EmptyReplicaSetLister{}, algorithm.EmptyStatefulSetLister{})
                },
                Weight: 1,
            },
        )

        // EqualPriority is a prioritizer function that gives an equal weight of one to all nodes
        // Register the priority function so that its available
        // but do not include it as part of the default priorities
        factory.RegisterPriorityFunction2("EqualPriority", core.EqualPriorityMap, nil, 1)
        // ImageLocalityPriority prioritizes nodes based on locality of images requested by a pod. Nodes with larger size
        // of already-installed packages required by the pod will be preferred over nodes with no already-installed
        // packages required by the pod or a small total size of already-installed packages required by the pod.
        factory.RegisterPriorityFunction2("ImageLocalityPriority", priorities.ImageLocalityPriorityMap, nil, 1)
        // Optional, cluster-autoscaler friendly priority function - give used nodes higher priority.
        factory.RegisterPriorityFunction2("MostRequestedPriority", priorities.MostRequestedPriorityMap, nil, 1)
    }

## defaultPredicates
含义：

    默认预选算法。

定义：

    func defaultPredicates() sets.String {
        return sets.NewString(
            // Fit is determined by volume zone requirements.
            factory.RegisterFitPredicateFactory(
                "NoVolumeZoneConflict",
                func(args factory.PluginFactoryArgs) algorithm.FitPredicate {
                    return predicates.NewVolumeZonePredicate(args.PVInfo, args.PVCInfo)
                },
            ),
            // Fit is determined by whether or not there would be too many AWS EBS volumes attached to the node
            factory.RegisterFitPredicateFactory(
                "MaxEBSVolumeCount",
                func(args factory.PluginFactoryArgs) algorithm.FitPredicate {
                    // TODO: allow for generically parameterized scheduler predicates, because this is a bit ugly
                    maxVols := getMaxVols(aws.DefaultMaxEBSVolumes)
                    return predicates.NewMaxPDVolumeCountPredicate(predicates.EBSVolumeFilter, maxVols, args.PVInfo, args.PVCInfo)
                },
            ),
            // Fit is determined by whether or not there would be too many GCE PD volumes attached to the node
            factory.RegisterFitPredicateFactory(
                "MaxGCEPDVolumeCount",
                func(args factory.PluginFactoryArgs) algorithm.FitPredicate {
                    // TODO: allow for generically parameterized scheduler predicates, because this is a bit ugly
                    maxVols := getMaxVols(DefaultMaxGCEPDVolumes)
                    return predicates.NewMaxPDVolumeCountPredicate(predicates.GCEPDVolumeFilter, maxVols, args.PVInfo, args.PVCInfo)
                },
            ),
            // Fit is determined by whether or not there would be too many Azure Disk volumes attached to the node
            factory.RegisterFitPredicateFactory(
                "MaxAzureDiskVolumeCount",
                func(args factory.PluginFactoryArgs) algorithm.FitPredicate {
                    // TODO: allow for generically parameterized scheduler predicates, because this is a bit ugly
                    maxVols := getMaxVols(DefaultMaxAzureDiskVolumes)
                    return predicates.NewMaxPDVolumeCountPredicate(predicates.AzureDiskVolumeFilter, maxVols, args.PVInfo, args.PVCInfo)
                },
            ),
            // Fit is determined by inter-pod affinity.
            factory.RegisterFitPredicateFactory(
                "MatchInterPodAffinity",
                func(args factory.PluginFactoryArgs) algorithm.FitPredicate {
                    return predicates.NewPodAffinityPredicate(args.NodeInfo, args.PodLister)
                },
            ),

            // 检查磁盘冲突
            factory.RegisterFitPredicate("NoDiskConflict", predicates.NoDiskConflict),

            // GeneralPredicates are the predicates that are enforced by all Kubernetes components
            // (e.g. kubelet and all schedulers)
            factory.RegisterFitPredicate("GeneralPredicates", predicates.GeneralPredicates),

            // 检查pod容忍的污点node，亲和相关。
            factory.RegisterFitPredicate("PodToleratesNodeTaints", predicates.PodToleratesNodeTaints),

            // 检查node内存压力
            factory.RegisterFitPredicate("CheckNodeMemoryPressure", predicates.CheckNodeMemoryPressurePredicate),

            // 检查node磁盘压力
            factory.RegisterFitPredicate("CheckNodeDiskPressure", predicates.CheckNodeDiskPressurePredicate),

            // 检查磁盘volume冲突
            factory.RegisterFitPredicateFactory(
                "NoVolumeNodeConflict",
                func(args factory.PluginFactoryArgs) algorithm.FitPredicate {
                    return predicates.NewVolumeNodePredicate(args.PVInfo, args.PVCInfo, nil)
                },
            ),
        )
    }

## defaultPriorities
含义：

    默认的优选算法。

定义：

    func defaultPriorities() sets.String {
        return sets.NewString(
            // spreads pods by minimizing the number of pods (belonging to the same service or replication controller) on the same node.
            factory.RegisterPriorityConfigFactory(
                "SelectorSpreadPriority",
                factory.PriorityConfigFactory{
                    Function: func(args factory.PluginFactoryArgs) algorithm.PriorityFunction {
                        return priorities.NewSelectorSpreadPriority(args.ServiceLister, args.ControllerLister, args.ReplicaSetLister, args.StatefulSetLister)
                    },
                    Weight: 1,
                },
            ),
            // pods should be placed in the same topological domain (e.g. same node, same rack, same zone, same power domain, etc.)
            // as some other pods, or, conversely, should not be placed in the same topological domain as some other pods.
            factory.RegisterPriorityConfigFactory(
                "InterPodAffinityPriority",
                factory.PriorityConfigFactory{
                    Function: func(args factory.PluginFactoryArgs) algorithm.PriorityFunction {
                        return priorities.NewInterPodAffinityPriority(args.NodeInfo, args.NodeLister, args.PodLister, args.HardPodAffinitySymmetricWeight)
                    },
                    Weight: 1,
                },
            ),

            // Prioritize nodes by least requested utilization.
            factory.RegisterPriorityFunction2("LeastRequestedPriority", priorities.LeastRequestedPriorityMap, nil, 1),

            // Prioritizes nodes to help achieve balanced resource usage
            factory.RegisterPriorityFunction2("BalancedResourceAllocation", priorities.BalancedResourceAllocationMap, nil, 1),

            // Set this weight large enough to override all other priority functions.
            // TODO: Figure out a better way to do this, maybe at same time as fixing #24720.
            factory.RegisterPriorityFunction2("NodePreferAvoidPodsPriority", priorities.CalculateNodePreferAvoidPodsPriorityMap, nil, 10000),

            // Prioritizes nodes that have labels matching NodeAffinity
            factory.RegisterPriorityFunction2("NodeAffinityPriority", priorities.CalculateNodeAffinityPriorityMap, priorities.CalculateNodeAffinityPriorityReduce, 1),

            // TODO: explain what it does.
            factory.RegisterPriorityFunction2("TaintTolerationPriority", priorities.ComputeTaintTolerationPriorityMap, priorities.ComputeTaintTolerationPriorityReduce, 1),
        )
    }

_______________________________________________________________________
[[返回kube-scheduler/kube-scheduler.md]](./kube-scheduler.md) 

