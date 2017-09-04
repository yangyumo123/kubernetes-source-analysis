CreateKubeAPIServer
===========================================================
## 简介
创建一个kube-apiserver。

## 1. CreateKubeAPIServer
含义：

    创建一个kube-apiserver。

路径：

    k8s.io/kubernetes/cmd/kube-apiserver/app/server.go

定义：

    func CreateKubeAPIServer(kubeAPIServerConfig *master.Config, delegateAPIServer genericapiserver.DelegationTarget, sharedInformers informers.SharedInformerFactory, crdRESTOptionsGetter genericregistry.RESTOptionsGetter)(*master.Master, error){
        //Complete填充一些未设置值的字段。例如：ExternalAddress、OpenAPIConfig、SwaggerConfig、DiscoveryAddress、Authenticator、Authorizer等等。
        kubeAPIServer, err:=kubeAPIServerConfig.Complete().New(delegateAPIServer, crdRESTOptionsGetter)
        if err!=nil{
            return nil, err
        }
        kubeAPIServer.GenericAPIServer.AddPostStartHook("start-kube-apiserver-informers", func(context genericapiserver.PostStartHookContext)error{
            sharedInformers.Start(context.StopCh)
            return nil
        })
        return kubeAPIServer,nil
    }

    //k8s.io/kubernetes/pkg/master/master.go
    func (c *Config)Complete()completedConfig{
        c.GenericConfig.Complete()
        serviceIPRange, apiServerServiceIP, err:=DefaultServiceIPRange(c.ServiceIPRange)
        if err!=nil{
            glog.Fatalf("Error determining service IP ranges: %v",err)
        }
        if c.ServiceIPRange.IP==nil{
            c.ServiceIPRange=serviceIPRange
        }
        if c.APIServerServiceIP==nil{
            c.APIServerServiceIP = apiServerServiceIP
        }
        discoveryAddresses := discovery.DefaultAddresses{DefaultAddress: c.GenericConfig.ExternalAddress}
        discoveryAddresses.CIDRRules = append(discoveryAddresses.CIDRRules,discovery.CIDRRule{IPRange: c.ServiceIPRange, Address: net.JoinHostPort(c.APIServerServiceIP.String(), strconv.Itoa(c.APIServerServicePort))})
        c.GenericConfig.DiscoveryAddresses = discoveryAddresses
        if c.ServiceNodePortRange.Size==0{
            c.ServiceNodePortRange = options.DefaultServiceNodePortRange
            glog.Infof("Node port range unspecified. Defaulting to %v.", c.ServiceNodePortRange)
        }
        c.GenericConfig.EnableSwaggerUI = c.GenericConfig.EnableSwaggerUI && c.EnableUISupport
        if c.EndpointReconcilerConfig.Interval==0{
            c.EndpointReconcilerConfig.Interval = DefaultEndpointReconcilerInterval
        }
        if c.EndpointReconcilerConfig.Reconciler == nil{
            endpointClient:=coreclient.NewForConfigOrDie(c.GenericConfig.LoopbackClientConfig)
            c.EndpointReconcilerConfig.Reconciler = NewMasterCountEndpointReconciler(c.MasterCount, endpointClient)
        }
        c.GenericConfig.EnableMetrics=true
        return completedConfig{c}
    }
    
    //创建一个kube-apiserver对象，未设置值的字段设置默认值。
    //func (c completeConfig)New(delegationTarget genericapiserver.DelegationTarget, crdRESTOptionsGetter genericregistry.RESTOptionsGetter)(*Master, error){
        if reflect.DeepEqual(c.KubeletClientConfig, kubeletclient.KubeletClientConfig{}){
            return nil, fmt.Errorf("Master.New() called with empty config.KubeletClientConfig")
        }
        s,err:=c.Config.GenericConfig.SkipComplete().New("kube-apiserver", delegationTarget)
        if err!=nil{
            return nil,err
        }
        if c.EnableUISupport{
            routes.UIRedirect{}.Install(s.Handler.NonGoRestfulMux)
        }
        if c.EnableLogsSupport{
            routes.Logs{}.Install(s.Handler.GoRestfulContainer)
        }
        m:=&Master{
            GenericAPIServer:s,
        }
        if c.APIResourceConfigSource.AnyResourceForVersionEnabled(apiv1.SchemeGroupVersion){
            legacyRESTStorageProvider := corerest.LegacyRESTStorageProvider{
                StorageFactory: c.StorageFactory,
                ProxyTransport: c.ProxyTransport,
                KubeletClientConfig: c.KubeletClientConfig,
                EventTTL: c.EventTTL,
                ServiceIPRange: c.ServiceIPRange,
                ServiceNodePortRange: c.ServiceNodePortRange,
                LoopbackClientConfig: c.GenericConfig.LoopbackClientConfig,
            }
            m.InstallLegacyAPI(c.Config, c.Config.GenericConfig.RESTOptionsGetter, legacyRESTStorageProvider)
        }
        restStorageProviders := []RESTStorageProvider{
            authenticationrest.RESTStorageProvider{Authenticator: c.GenericConfig.Authenticator},
            authorizationrest.RESTStorageProvider{Authorizer: c.GenericConfig.Authorizer},
            autoscalingrest.RESTStorageProvider{},
            batchrest.RESTStorageProvider{},
            certificaterest.RESTStorageProvider{},
            extensionsrest.RESTStorageProvider{ResourceInterface: thirdparty.NewThiredPartyResourceServer(s,s.DiscoveryGroupManager, c.StorageFactory, crdRESTOptionsGetter)},
            networkingrest.RESTStorageProvider{},
            policyrest.RESTStorageProvider{},
            rbacrest.RESTStorageProvider{Authorizer: c.GenericConfig.Authorizer},
            settingsrest.RESTStorageProvider{},
            storagerest.RESTStorageProvider{},
            appsrest.RESTStorageProvider{},
            admissionregistrationrest.RESTStorageProvider{},
        }
        m.InstallAPIs(c.Config.APIResourceConfigSource, c.Config.GenericConfig.RESTOptionsGetter, restStorageProviders...)
        if c.Tunneler!=nil{
            m.installTunneler(c.Tunneler, corev1client.NewForConfigOrDie(c.GenericConfig.LoopbackClientConfig).Nodes())
        }
        if err:=m.GenericAPIServer.AddPostStartHook("ca-registration", c.ClientCARegistrationHook.PostStartHook); err!=nil{
            glog.Fatalf("Error registering PostStartHook %q: %v", "ca-registration", err)
        }
        return m,nil
    }

    //k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/sever/hooks.go
    func (s *GenericAPIServer) AddPostStartHook(name string, hook PostStartHookFunc)error{
        if len(name)==0{
            return fmt.Errorf("missing name")
        }
        if hook==nil{
            return nil
        }
        if s.disabledPostStartHooks.Has(name){
            return nil
        }
        s.postStartHookLock.Lock()
        defer s.postStartHookLock.Unlock()
        if s.postStartHooksCalled{
            return fmt.Errorf("unable to add %q because PostStartHooks have already been called", name)
        }
        if _,exists:=s.postStartHooks[name];exists{
            return fmt.Errorf("unable to add %q because it is already registered", name)
        }
        done:=make(chan struct{})
        s.AddHealthzChecks(postStartHookHealthz{name: "poststarthook/"+name, done:done})
        s.postStartHooks[name] = postStartHookEntry{hook: hook, done: done}
        return nil
    }



_______________________________________________________________________
[[返回/kube-apiserver/run/run.md]](./run.md) 
