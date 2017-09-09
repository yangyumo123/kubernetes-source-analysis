kube-scheduler 调度
======================================================================
## 简介
本文介绍kube-scheduler调度算法及源码分析。一句话概括kube-scheduler功能：为PodSpec.NodeName为空的pod，经过预选算法和优选算法，选择一个最合适的node作为该pod的运行节点。

## 约定
本文研究的版本是：kubernetes v1.8.0-alpha.0.450+574a6cab2ceada-dirty，后续可能会相应的更新版本。

## 目录
1. [kube-scheduler原理](./kube-scheduler-introduce.md)
2. [创建SchedulerServer](./create-scheduler-server.md)
3. [kube-scheduler参数](./kube-scheduler-flag.md)
4. [kube-scheduler运行](./kube-scheduler-run.md)
5. [kube-scheduler预选算法]()
6. [kube-scheduler优选算法]()


_______________________________________________________________________
[[返回README.md]](../README.md) 
