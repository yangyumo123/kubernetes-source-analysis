入口Run
====================================================
## 简介
运行apiserver。

## 入口
含义：

    运行服务器的入口。

路径：

    k8s.io/kubernetes/cmd/kube-apiserver/apiserver.go

定义：

    func main(){
        ...
        //运行服务器，对外监听在安全端口上，对内监听在非安全端口上，不停止。
        if err:=app.Run(s, wait.NeverStop); err!=nil {
            fmt.Fprintf(os.Stderr, "%v\n", err)
            os.Exit(1)
        }
    }

## Run函数
含义：

    运行指定的APIServer。

路径：

    k8s.io/kubernetes/cmd/kube-apiserver/app/server.go

实参：

    runOptions：前面创建好的ServerRunOptions对象。
    stopCh：传入的是一个NeverStop channel。var NeverStop <-chan struct{} = make(chan struct{})，作用是让服务器一直运行，不停止。

定义：

    func Run(runOptions *options.ServerRunOptions, stopCh <-chan struct{}) error{ 
        //当运行在云平台上，才需要创建ssh nodeTunneler，用于apiserver和aggregator连接到node上。返回的tunneler赋给master.Config.Tunneler，proxyTransport赋给master.Config.ProxyTransport。
        tunneler, proxyTransport, err:=CreateDialer(runOptions)
        if err!=nil {
            return err
        }

        //创建运行API server所需的所有资源。
        kubeAPIServerConfig, sharedInformers, insecureServingOptions, err := CreateKubeAPIServerConfig(runOptions, tunneler, proxyTransport)
        if err!= nil {
            return err
        }

        //创建扩展API配置。
        apiExtensionsConfig, err:=createAPIExtensionsConfig(*kubeAPIServerConfig.GenericConfig, runOptions)
        if err!=nil {
            return err
        }
     
        //创建扩展API服务器。
        apiExtensionsServer, err:=createAPIExtensionsServer(apiExtensionsConfig, genericapiserver.EmptyDelegate)
        if err!=nil {
            return err
        }
     
    
        //创建API服务器。
        kubeAPIServer,err := CreateKubeAPIServer(kubeAPIServerConfig, apiExtensionsServer.GenericAPIServer, sharedInformers, apiExtensionsConfig.CRDRESTOptionsGetter)
        if err!=nil {
            return err
        }

        //直接调用API server，用于测试，不连接openapi。
        if len(os.Getenv("KUBE_API_VERSIONS"))>0{
            if insecureServingOptions != nil{
                insecureHandlerChain := kubeserver.BuildInsecureHandlerChain(kubeAPIServer.GenericAPIServer.UnprotectedHandler(), kubeAPIServerConfig.GenericConfig)
                if err:=kubeserver.NonBlockingRun(insecureServingOptions, insecureHandlerChain, stopCh); err!=nil{
                    return err
                }
            }
            return kubeAPIServer.GenericAPIServer.PrepareRun().Run(stopCh)
        }

        //下面流程是正常调用API server，在API server之前启动aggregator，并且连接到openapi。
        kubeAPIServer.GenericAPIServer.PrepareRun()

        aggregatorConfig,err:=createAggregatorConfig(*kubeAPIServerConfig.GenericConfig, runOptions)
        if err!=nil{
            return err
        }
        aggregatorServer,err:=createAggregatorServer(aggregatorConfig, kubeAPIServer.GenericAPIServer, sharedInformers, apiExtensionServer.Informers, proxyTransport)
        if err!=nil{
            return err
        }

        if insecureServingOptions != nil{
            insecureHandlerChain := kubeserver.BuildInsecureHandlerChain(aggregatorServer.GenericAPIServer.UnprotectedHandler(), kubeAPIServerConfig.GenericConfig)
            if err:=kubeserver.NonBlockingRun(insecureServingOptions, insecureHandlerChain, stopCh); err!=nil{
                return err
            }
        }
        return aggregatorServer.GenericAPIServer.PrepareRun().Run(stopCh)
    }


_______________________________________________________________________
[[返回/kube-apiserver/run/run.md]](./run.md) 
