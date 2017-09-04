createAPIExtensionsServer
=====================================================================
## 简介
创建apiExtensionsServer。

## 1. createAPIExtensionsServer
含义：

    创建apiExtensionsServer。

路径：

    k8s.io/kubernetes/cmd/kube-apiserver/app/apiextensions.go

定义：

    func createAPIExtensionsServer(apiextensionsConfig *apiextensionsapiserver.Config, delegateAPIServer genericapiserver.DelegationTarget)(*apiextensionsapiserver.CustomResourceDefinitions,error){
        apiextensionsServer,err:=apiextensionsConfig.Complete().New(delegateAPIServer)
        if err!=nil{
            return nil,err
        }
        return apiextensionsServer,nil
    }



_______________________________________________________________________
[[返回/kube-apiserver/run/run.md]](./run.md) 
