createAggregatorConfig
==================================================
## 简介
创建aggregatorConfig。

## 1. createAggregatorConfig
含义：

    创建aggregatorConfig。

路径：

    k8s.io/kubernetes/cmd/kube-apiserver/app/aggregator.go

定义：

    func createAggregatorConfig(kubeAPIServerConfig genericapiserver.Config, commandOptions *options.ServerRunOptions)(*aggregatorapiserver.Config, error){
        genericConfig:=kubeAPIServerConfig

        //aggregator并没有连接它们，仅仅是把它们委派给kube-apiserver。
        genericConfig.EnableSwaggerUI = false
        genericConfig.SwaggerConfig=nil

        //复制etcd参数。
        etcdOptions:=*commandOptions.Etcd
        etcdOptions.StorageConfig.Codec = aggregatorapiserver.Codecs.LegacyCodec(vabeta1.SchemeGroupVersion)
        etcdOptions.StorageConfig.Copier = aggregatorapiserver.Scheme
        genericConfig.RESTOptionsGetter = &genericoptions.SimpleRestOptionsFactory{Options: etcdOptions}
        client,err:=kubeclientset.NewForConfig(genericConfig.LoopbackClientConfig)
        if err!=nil{
            return nil,err
        }
        var certBytes,keyBytes []byte
        if len(commandOptions.ProxyClientCertFile)>0 && len(commandOptions.ProxyClientkeyFile)>0{
            certBytes,err=ioutil.ReadFile(commandOptions.ProxyClientCertFile)
            if err!=nil{
                return nil,err
            }
            keyBytes,err=ioutil.ReadFile(commandOptions.ProxyClientKeyFile)
            if err!=nil{
                return nil,err
            }
        }
        aggregatorConfig:=&aggregatorapiserver.Config{
            GenericConfig: &genericConfig,
            CoreAPIServerClient: client,
            ProxyClientCert: certBytes,
            ProxyClientKey: keyBytes,
            EnableAggregatorRouting: commandOptions.EnableAggregatorRouting,
        }
        return aggregatorConfig, nil
    }


_______________________________________________________________________
[[返回/kube-apiserver/run/run.md]](./run.md) 
