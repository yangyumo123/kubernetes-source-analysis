kubeAPIServer.GenericAPIServer.PrepareRun
=================================================================
## 简介
运行服务器前准备工作。

## 1.kubeAPIServer.GenericAPIServer.PrepareRun
含义：

    注册swaggerui，webservice，注册健康检查。

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

1. Install

        含义：注册SwaggerUI webservice到mux中。
        路径：k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/server/routes/swagger.go
        func (s Swagger) Install (c *restful.Container){
            s.Config.WebServices = c.RegisteredWebServices()
            swagger.RegisterSwaggerService(*s.Config, c)
        }

        先介绍Swagger结构体：
        含义：Swagger安装/swaggerapi以允许发现和遍历schema。允许Kubernetes GenericAPIServer的使用者在初始化swagger之前注册它们自己的服务到Kubernetes mux中，以便其他资源类型能显示在文档中。
        路径：k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/server/routes/swagger.go
        type Swagger struct{
            Config *swagger.Config
        }

2. PrepareOpenAPIService

        func (s *GenericAPIServer) PrepareOpenAPIService(){
            if s.openAPIConfig != nil && s.OpenAPIService==nil{
                s.OpenAPIService==routes.OpenAPI{
                    Config: s.openAPIConfig,
                }.Install(s.Handler.GoRestfulContainer, s.Handler.NonGoRestfulMux)
            }
        }

3. InstallHealthz

        //k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/server/healthz.go
        func (s *GenericAPIServer) installHealthz(){
            s.healthzLock.Lock()
            defer s.healthzLock.Unlock()
            s.healthzCreated = true
            healthz.InstallHandler(s.Handler.NonGoRestfulMux, s.healthzChecks...)
        }



_______________________________________________________________________
[[返回/kube-apiserver/run/run.md]](./run.md) 
