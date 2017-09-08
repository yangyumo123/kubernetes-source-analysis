kube-scheduler原理
======================================================================
## 简介
kube-scheduler是kubernetes集群中的调度器，经过预选算法和优选算法，为Pod绑定（binding）一个最合适的Node。

## 预选算法 - Predicates
根据配置的Predicates Polices过滤掉不满足这些Polices的Nodes，剩下的Nodes就作为优选算法的输入。

## 优选算法 - Priorities
根据配置的Priorities Policies给预选后的Nodes进行打分排名，得分最高的Node即作为最合适的Node，如果有几个Nodes并列得分最高，则scheduler随机选择一个Node作为目标Node。

## Policies
算法的关键在于Policies。有两种Policies：Predicates Policies和Priorities Policies。

### Predicates Policies
优选算法使用Predicates Policies来过滤掉不符合条件的Nodes。并发启动多个goroutine，对每个node执行Predicates Policies遍历，看是否满足Predicates Policies，若有一个Policy不满足，则直接淘汰。
可以在kube-scheduler启动参数中添加"--policy-config-file"来指定要运用的Policies集合，这个配置文件格式如下：
{
    "kind": "Policy",
    "apiVersion": "v1",
    "predicates": [
        {"name": "PodFitsPorts"},
        {"name": "PodFitsResources"},
        {"name": "NoDiskConflict"},
        {"name": "NoVolumeZoneConflict"},
        {"name": "MatchNodeSelector"},
        {"name": "HostName"}
    ],
    "priorities: [
        ...
    ]

}


### Priorities Policies
优选算法使用Priorities Policies。


_______________________________________________________________________
[[返回kube-scheduler/kube-scheduler.md]](./kube-scheduler.md) 


