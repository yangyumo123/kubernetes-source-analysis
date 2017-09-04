kubeserver.BuildInsecureHandlerChain
==============================================================
## 简介
非安全处理器链。

## 1. kubeserver.BuildInsecureHandlerChain
含义：

    非安全处理器链。

路径：

    k8s.io/kubernetes/pkg/kubeapiserver/server/insecure_handler.go

定义：

    func BuildInsecureHandlerChain(apiHandler http.Handler, c *server.Config)http.Handler{
        handler := apiHandler
        if utilfeature.DefaultFeatureGate.Enabled(features.AdvancedAuditing){
            handler=genericapifilters.WithAudit(handler, c.RequestContextMapper, c.AuditBackend, c.AuditPolicyChecker, c.LongRunningFunc)
        }else{
            handler=genericapifilters.WithLegacyAudit(handler, c.RequestContextMapper, c.LegacyAuditWriter)
        }
        handler=genericapifilters.WithAuthentication(handler, c.RequestContextMapper, insecureSuperuser{}, nil)
        handler=genericapifilters.WithCORS(handler, c.CorsAllowedOriginList, nil,nil,nil,"true")
        handler=genericapifilters.WithPanicRecovery(handler)
        handler=genericapifilters.WithTimeoutForNonLongRunningRequests(handler, c.RequestContextMapper, c.LongRunningFunc)
        handler=genericapifilters.WithMaxInFlightLimit(handler, c.MaxRequestsInFlight, c.MaxMutatingReqeustsInFlight, c.RequestContextMapper, c.LongRunningFunc)
        handler=genericapifilters.WithRequestInfo(handler,server.NewRequestInfoResolver(c), c.RequestContextMapper)
        handler=genericapifilters.WIthRequestContext(handler, c.RequestContextMapper)
        return handler
    }


_______________________________________________________________________
[[返回/kube-apiserver/run/run.md]](./run.md) 
