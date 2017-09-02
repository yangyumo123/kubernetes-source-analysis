remote debug kubernetes
===================================================================
## 简介
目前，不能在windows上运行kubernetes，所以使用IDE调试kubernetes非常麻烦。一般采用远程dlv的调试方法，即在远程Linux服务器和本地windows各自放一份相同的kubernetes代码，然后启动远程Linux服务器上的dlv服务器，并从本地windows上发送调试指令给远程dlv服务器，进行调试。

## 目录
1. [远程调试配置](../remote-debug/remote-debug-launch.md/)
2. [调试kube-apiserver](../remote-debug/debug-kube-apiserver.md/)

_______________________________________________________________________
[[返回/kube-apiserver/flag/flag.md]](../../kube-apiserver/flag/flag.md/) 
