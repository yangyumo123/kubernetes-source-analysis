Run 运行apiserver
===============================================================
## 简介
Run函数是服务器运行的核心程序，大致包括：配置服务器、创建服务器、启动服务器。

## 目录
1. [入口Run](./main-run.md)
2. [CreateDialer](./createDialer.md)
3. [CreateKubeAPIServerConfig](./createKubeAPIServerConfig.md)
4. [createAPIExtensionsConfig](./createAPIExtensionsConfig.md)
5. [createAPIExtensionsServer](./createAPIExtensionsServer.md)
6. [CreateKubeAPIServer](./createKubeAPIServer.md)
7. [kubeAPIServer.GenericAPIServer.PrepareRun](./genericAPIServer.md)
8. [createAggregatorConfig](./createAggregatorConfig.md)
9. [createAggregatorServer](./createAggregatorServer.md)
10. [kubeserver.BuildInsecureHandlerChain](./buildInsecureHandlerChain.md)
11. [kubeserver.NonBlockingRun](./nonBlockingRun.md)
12. [aggregatorServer.GenericAPIServer.PrepareRun](./prepareRun.md)
13. [genericAPIServer.Run](./generic-run.md)


_______________________________________________________________________
[[返回/kube-apiserver/kube-apiserver.md]](../kube-apiserver.md) 




