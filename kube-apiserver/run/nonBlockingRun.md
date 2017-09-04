kubeserver.NonBlockingRun
===========================================================
## 简介
非阻塞运行。

## 1. kubeserver.NonBlockingRun
含义：

    非阻塞运行。

路径：

    k8s.io/kubernetes/pkg/kubeapiserver/server/insecure_handler.go

定义：

    func NonBlockingRun(insecureServingInfo *InsecureServingInfo, insecureHandler http.Handler, stopCh <-chan struct{}) error{
        internalStopCh:=make(chan struct{})
        if insecureServingInfo!=nil && insecureHandler!=nil{
            if err:=serveInsecurely(insecureServingInfo, insecureHandler, internalStopCh); err!=nil{
                close(internalStopCh)
                return err
            }
        }
        go func(){
            <-stopCh
            close(internalStopCh)
        }()
        return nil
    }


_______________________________________________________________________
[[返回/kube-apiserver/run/run.md]](./run.md) 
