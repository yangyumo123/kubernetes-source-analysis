aggregatorServer.GenericAPIServer.PrepareRun
==============================================================
## 简介

## 1. aggregatorServer.GenericAPIServer.PrepareRun

路径：

    k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/server/genericapiserver.go

定义：

    func (s *GenericAPIServer) PrepareRun() preparedGenericAPIServer{
        if s.swaggerConfig!=nil{
            routes.Swagger{Config: s.swaggerConfig}.Install(s.Handler.GoRestfulContainer)
        }
        s.PrepareOpenAPIService()
        s.installHealthz()
        return preparedGenericAPIServer{s}
    }

### 1.1 Install

路径：

    k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/server/routes/swagger.go

定义：

    func (s Swagger)Install(c *restful.Container){
        s.Config.WebServices = c.RegisteredWebServices()
        swagger.RegisterSwaggerService(*s.Config, c)
    }

### 1.2 s.PrepareOpenAPIService

路径：

    k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/server/genericapiserver.go

定义：

    func(s *GenericAPIServer)PrepareOpenAPIService(){
        if s.openAPIConfig!=nil && s.OpenAPIService==nil{
            s.OpenAPIService==routes.OpenAPI{
                Config: s.openAPIConfig,
            }.Install(s.Handler.GoRestfulContainer, s.Handler.NonGoRestfulMux)
        }
    }

### 1.3 s.installHealthz

路径：

    k8s.io/kubernetes/vendor/k8s.io/apiserver

定义：

    func (s *GenericAPIServer) installHealthz(){
        s.healthzLock.Lock()
        defer s.healthzLock.Unlock()
        s.healthzCreated = true
        healthz.InstallHandler(s.Handler.NonGoRestfulMux, s.healthzChecks...)
    }


_______________________________________________________________________
[[返回/kube-apiserver/run/run.md]](./run.md) 
