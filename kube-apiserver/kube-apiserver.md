kube-apiserver 源码分析
==========================================================
## 简介
kube-apiserver是kubernetes中对象的CRUD & watch的HTTP/RESTful服务端。本文从源码角度分析kube-apiserver的运行机制。关于kube-apiserver的架构原理请看参考文献。
## 约定
本文研究的版本是：kubernetes v1.8.0-alpha.0.450+574a6cab2ceada-dirty，后续可能会相应的更新版本。
## 目录
1. [命令行参数](/flag/flag.md)
2. [日志](/log/log.md)
3. [运行服务器](/run/run.md)
## 参考文献
* [https://blog.openshift.com/kubernetes-deep-dive-api-server-part-1/](https://blog.openshift.com/kubernetes-deep-dive-api-server-part-1/)
* [https://blog.openshift.com/kubernetes-deep-dive-api-server-part-2/](https://blog.openshift.com/kubernetes-deep-dive-api-server-part-2/)

[返回README.md](../README.md)   [下一页](/flag/flag.md)