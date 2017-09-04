createAPIExtensionsConfig
========================================================
## 简介
创建API extensions配置。

## 1. createAPIExtensionsConfig
含义：

    创建API extensions的配置。

路径：

    k8s.io/kubernetes/cmd/kube-apiserver/app/apiextensions.go

参数：

    genericConfig
    ServerRunOptions

定义：

    func createAPIExtensionsConfig(kubeAPIServerConfig genericapiserver.Config, commandOptions *options.ServerRunOptions)(*apiextensionsapiserver.Config, error){
        genericConfig:=kubeAPIServerConfig

        //复制etcd配置。
        etcdOptions := *commandOptions.Etcd

        //codec - group="apiextensions.k8s.io"，version="v1beta1"
        etcdOptions.StorageConfig.Codec = apiextensionsapiserver.Codecs.LegacyCodec(v1beta1.SchemeGroupVersion)
        
        //copier - group="apiextensions.k8s.io"
        etcdOptions.StorageConfig.Copier = apiextensionsapiserver.Scheme
        genericConfig.RESTOptionsGetter = &genericoptions.SimpleRestOptionsFactory{Options: etcdOptions}

        //apiextensions.k8s.io组的配置。
        apiextensionsConfig:=&apiextensionsapiserver.Config{
            GenericConfig: &genericConfig,
            CRDRESTOptionsGetter: apiextensionscmd.NewCRDRESTOptionsGetter(etcdOptions),
        }
        return apiextensionsConfig, nil
    }


_______________________________________________________________________
[[返回/kube-apiserver/run/run.md]](./run.md) 
