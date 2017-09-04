genericAPIServer.Run
=================================================================
## 简介
运行服务器。

## 1. Run
含义：

    运行服务器。

路径：

    k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/server/genericapiserver.go

定义：

    func (s preparedGenericAPIServer)Run(stopCh <-chan struct{})error{
        err:=s.NonBlockingRun(stopCh)
        if err!=nil{
            return err
        }

        //阻塞
        <-stopCh
        return nil
    }

### 1.1 s.NonBlockingRun
含义：

    非阻塞运行。

路径：

    k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/server/genericapiserver.go

定义：

    func (s preparedGenericAPIServer) NonBlockingRun(stopCh <-chan struct{}) error{
        internalStopCh:=make(chan struct{})
        if s.SecureServingInfo!=nil && s.Handler!=nil{
            //运行安全http server。
            if err:=s.serveSecurely(internalStopCh);err!=nil{
                close(internalStopCh)
                return err
            }
        }
        go func(){
            <-stopCh
            close(internalStopCh)
        }()

        if s.AuditBackend!=nil{
            if err:=s.AuditBackend.Run(stopCh);err!=nil{
                return fmt.Errorf("failed to run the audit backend: %v", err)
            }
        }

        s.RunPostStartHooks(stopCh)

        if _,err:=systemd.SdNotify(true, "READY=1\n");err!=nil{
            glog.Errorf("Unable to send systemd daemon successful start message: %v\n", err)
        }
        return nil
    }

（1）serveSecurely
     
含义：

    运行安全http server。

路径：

    k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/server/serve.go

定义：

    func (s *GenericAPIServer) serveSecurly(stopCh <-chan struct{}) error{
        //创建http.Server
        secureServer:=&http.Server{
            Addr: s.SecureServingInfo.BindAddress,   //
            Handler: s.Handler,
            MaxHeaderBytes: 1<<20,
            TLSConfig: &tls.Config{
                NameToCertificate: s.SecureServingInfo.SNICerts,
                MinVersion: tls.VersionTLS12,
                NextProtos: []string{"h2", "http/1.1"},
            },
        }
        if s.SecureServingInfo.MinTLSVersion > 0{
            secureServer.TLSConfig.MinVersion = s.SecureServingInfo.MinTLSVersion
        }
        if len(s.SecureServingInfo.CipherSuites)>0{
            secureServer.TLSConfig.CipherSuites = s.SecureServingInfo.CipherSuites
        }
        if s.SecureServingInfo.Cert!=nil{
            secureServer.TLSConfig.Certificates = []tls.Certificates{*s.SecureServingInfo.Cert}
        }

        for _,c:=range s.SecureServingInfo.SNICerts{
            secureServer.TLSConfig.Certificates = append(secureServer.TLSConfig.Certificates, *c)
        }
        if s.SecureServingInfo.ClientCA!=nil{
            secureServer.TLSConfig.ClientAuth = tls.RequestClientCert
            secureServer.TLSConfig.ClientCAs = s.SecureServingInfo.ClientCA
        }
        glog.Infof("Serving securely on %s", s.SecureServingInfo.BindAddress)
        var err error

        //运行server
        s.effectiveSecurePort, err=RunServer(secureServer, s.SecureServingInfo.BindNetwork, stopCh)
        return err
    }

    //k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/server/serve.go
    func RunServer(server *http.Server, network string, stopCh <-chan struct{}) (int, error){
        if len(server.Addr)==0{
            return 0, errors.New("address cannot be empty")
        }
        if len(network)==0{
            network="tcp"
        }
        ln,err:=net.Listen(network, server.Addr)
        if err!=nil{
            return 0, fmt.Errorf("failed to listen on %v: %v", server.Addr, err)
        }
        tcpAddr, ok:=ln.Addr().(*net.TCPAddr)
        if !ok{
            ln.Close()
            return 0,fmt.Errorf("invalid listen address: %q", ln.Addr().String())
        }
        go func(){
            <-stopCh
            ln.Close()
        }()
        go func(){
            defer utilruntime.HandlerCrash()
            var listener net.Listener
            listener=tcpKeepAliveListener{ln.(*net.TCPListener)}
            if server.TLSConfig!=nil{
                listener=tls.NewListener(listener, server.TLSConfig)
            }
            err:=server.Serve(listener)
            msg:=fmt.Sprintf("Stopped listening on %s", tcpAddr.String())
            select{
            case <-stopCh:
                glog.Info(msg)
            default:
                panic(fmt.Sprintf("%s due to error: %v", msg, err))
            }
        }()
        return tcpAddr.Port, nil
    }


_______________________________________________________________________
[[返回/kube-apiserver/run/run.md]](./run.md) 
