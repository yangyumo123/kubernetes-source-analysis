createAggregatorServer
=============================================================
## 简介
创建aggregatorServer。

## 1. createAggregatorServer
含义：

    创建aggregatorServer。

路径：

    k8s.io/kubernetes/cmd/kube-apiserver/app/aggregator.go

定义：

    func createAggregatorServer(aggregatorConfig *aggregatorapiserver.Config, delegateAPIServer genericapiserver.DelegationTarget, kubeInformers informers.SharedInformerFactory, apiExtensionInformers apiextensionsinformers.SharedInformerFactory, proxyTransport *http.Transport)(*aggregatorapiserver.APIAggregator, error){
        aggregatorServer, err:=aggregatorConfig.Complete().NewWithDelegate(delegateAPIServer, proxyTransport)
        if err!=nil{
            return nil,err
        }
        apiRegistrationClient, err:=apiregistrationclient.NewForConfig(aggregatorConfig.GenericConfig.LoopbackClientConfig)
        if err!=nil{
            return nil,err
        }
        autoRegistrationController:=autoregister.NewAutoRegisterController (aggregatorServer.APIRegistrationInformers.Apiregistration().InternalVersion().APIServices().apiRegistrationClient)
        apiServices:=apiServicesToRegister(delegateAPIServer,autoRegistrationController)
        tprRegistrationController:=thirdparty.NewAutoRegistrationController(
            kubeInformers.Extensions().InternalVersion().ThirdPartyResources(),
            apiExtensionInformers.Apiextensions().InternalVersion().CustomResourceDefinitions(),
            autoRegistrationController)
        aggregatorServer.GenericAPIServer.AddPostStartHook("kube-apiserver-autoregistration", func(context genericapiserver.PostStartHookContext)error{
            go autoRegistrationController.Run(5,context.StopCh)
            go tprRegistrationController.Run(5,context.StopCh)
        })
        aggregatorServer.GenericAPIServer.AddHealthzChecks(healthz.NamedCheck("autoregister-completion", func(r *http.Request)error{
            items,err:=aggregatorServer.APIRegistrationInformers.Apiregistration().InternalVersion().APIServices().Lister().List(labels.Everything())
            if err!=nil{
                return err
            }
            missing:=[]apiregistration.APIService{}
            for _,apiService:=range apiServices{
                found:=false
                for _,item:=range items{
                    if item.Name!= apiService.Name{
                        continue
                    }
                    if apiregistration.IsAPIServiceConditionTrue(item, apiregistration.Available){
                        found=true
                        break 
                    }
                }
                if !found{
                    missing=append(missing, *apiService)
                }
            }
            if len(missing)>0{
                return fmt.Errorf("missing APIService: %v", missing)
            }
            return nil
        }))
        return aggregatorServer,nil
    }


_______________________________________________________________________
[[返回/kube-apiserver/run/run.md]](./run.md) 
