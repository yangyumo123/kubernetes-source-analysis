CreateKubeAPIServerConfig
====================================================
## 简介
创建运行API server所需的所有资源。

## 1. CreateKubeAPIServerConfig
含义：

    创建运行API server所需的所有资源，创建master.Config对象。

路径：

    k8s.io/kubernetes/cmd/kube-apiserver/app/server.go

定义：

    func CreateKubeAPIServerConfig(s *options.ServerRunOptions, nodeTunneler tunneler.Tunneler, proxyTransport http.RoundTripper)(*master.Config, informers.SharedInformerFactory, *kubeserver.InsecureServingInfo, error){
        //注册所有准入控制插件。
        registerAllAdmissionPlugins(s.Admission.Plugins)

        //创建generic config之前，设置默认参数。
        if err:=defaultOptions(s); err!= nil{
            return nil, nil, nil, err
        }

        //验证参数
        if errs:=s.Validate(); len(errs)!=0{
            return nil, nil, nil, utilerrors.NewAggregator(errs)
        }

        //创建generic config
        genericConfig, sharedInformers, insecureServingOptions, err:=BuildGenericConfig(s)     
        if err!=nil{
            return nil, nil, nil, err
        }

        if err:=utilwait.PollImmediate(etcdRetryInterval, etcdRetryLimit*etcdRetryInterval, preflight.EtcdConnection{ServerList: s.Etcd.StorageConfig.ServerList}.CheckEtcdServers); err!=nil{
            return nil, nil, nil, fmt.Errorf("error waiting for etcd connection: %v", err)
        }

        //capbilities security
        capabilities.Initialize(capabilities.Capabilities{
            AllowPrivileged:   s.AllowPrivileged,
            PrivilegedSources: capabilities.PrivilegedSources{
                HostNetworkSources: []string{},
                HostPIDSources:     []string{},
                HostIPCSources:     []string{},
            },
            PerConnectionBandwidthLimitBytesPerSec: s.MaxConnectionBytesPerSec,
        })

        //service ip range,  apiserver service ip
        serviceIPRange, apiServerServiceIP, err := master.DefaultServiceIPRange(s.ServiceClusterIPRange)
        if err!=nil{
            return nil, nil, nil, err
        }

        //Storage factory
        storageFactory, err := BuildStorageFactory(s)
        if err!=nil {
            return nil, nil, nil, err
        }
        clientCA, err:= readCAorNil(s.Authentication.ClientCert.ClientCA)
        if err!=nil{
            return nil,nil,nil,err
        }
        requestHeaderProxyCA,err:=readCAorNil(s.Authentication.RequestHeader.ClientCAFile)
        if err!=nil{
            return nil, nil, nil, err
        }

        //master.Config
        config:=&master.Config{
            GenericConfig: genericConfig,
            ClientCARegistrationHook: master.ClientCARegistrationHook{
                ClientCA: clientCA,
                RequestHeaderUsernameHeaders: s.Authentication.RequestHeader.UsernameHeaders,
                RequestHeaderGroupHeaders: s.Authentication.RequestHeader.GroupHeaders,
                RequestHeaderExtraHeaderPrefixes: s.Authentication.RequestHeader.ExtraHeaderPrefixes,
                RequestHeaderCA: requestHeaderProxyCA,
                RequestHeaderAllowedNames: s.Authentication.RequestHeader.AllowedNames,
            },
            APIResourceConfigSource: storageFactory.APIResourceConfigSource,
            StorageFactory: storageFactory,
            EnableCoreControllers: true,
            EventTTL: s.EventTTL,
            KubeletClientConfig: s.KubeletConfig,
            EnableUISupport: true,
            EnableLogsSupport: s.EnableLogsHandler,
            ProxyTransport: proxyTransport,
            Tunneler: nodeTunneler,
            ServiceIPRange: serviceIPRange,
            APIServerServiceIP: apiServerServiceIP,
            APIServerServicePort: 443,
            ServiceNodePortRange: s.ServiceNodePortRange,
            KubernetesServiceNodePort: s.KubernetesServiceNodePort,
            MasterCount: s.MasterCount,
        }
        return config, sharedInformers, insecureServingOptions, nil
    }

## 1.1 registerAllAdmissionPlugins
含义：

    注册所有准入控制插件。插件要实现admission.Interface接口的Admit和Handles方法。Handles方法决定插件支持的操作类型，Admit方法决定插件的执行行为。另外，前面已经注册过一种插件叫"NamespaceLifecycle"。

路径：

    k8s.io/kubernetes/cmd/kube-apiserver/app/plugins.go

定义：

    func registerAllAdmissionPlugins(plugins *admission.Plugins){
        admit.Register(plugins)                                //注册插件"AlwaysAdmit"，表示对所有请求都允许通过。
        alwayspullimages.Register(plugins)                     //注册插件"AlwaysPullImages"，表示总是拉取镜像，支持CREATE、UPDATE操作。后面一一介绍各插件。
        antiaffinity.Register(plugins)
        defaulttolerationseconds.Register(plugins)
        deny.Register(plugins)
        exec.Register(plugins)
        gc.Register(plugins)
        imagepolicy.Register(plugins)
        initialization.Register(plugins)
        initialresources.Register(plugins)
        limitranger.Register(plugins)
        autoprovision.Register(plugins)
        exists.Register(plugins)
        noderestriction.Register(plugins)
        label.Register(plugins)
        podnodeselector.Register(plugins)
        podpreset.Register(plugins)
        podtolerationrestriction.Register(plugins)
        resourcequota.Register(plugins)
        podsecuritypolicy.Register(plugins)
        scdeny.Register(plugins)
        serviceaccount.Register(plugins)
        setdefault.Register(plugins)
        webhook.Register(plugins)
    }

1. admit

        含义：注册插件"AlwaysAdmit"，表示对所有请求都允许通过。支持所有操作。必须实现admission.Interface接口的Admit和Handles方法。
        路径：k8s.io/kubernetes/plugin/pkg/admission/admit/admission.go
        func Register(plugins *admission.Plugins){
            plugins.Register("AlwaysAdmit", func(config io.Reader)(admission.Interface, error){
                return NewAlwaysAdmit(), nil
            })
        }
        func NewAlwaysAdmit() admission.Interface{
            return new(alwaysAdmit)
        }

        //alwaysAdmit实现admission.Interface接口。
        type alwaysAdmit struct{}
        func (alwaysAdmit) Admit(a admission.Attributes)(err error){
            return nil
        }
        func (alwaysAdmit)Handles(operation admission.Operation)bool{
            return true
        }

        下面的插件注册方式和AlwaysAdmit类似，只是Admit和Handles方法不同，就不一一介绍了。

2. alwayspullimages

        注册插件"AlwaysPullImages"，表示访问pod资源时总是会拉取镜像，如果请求不是访问pod资源，则忽略准入控制，允许请求通过；否则执行准入控制判断。
        支持CREATE、UPDATE操作。
        查看所有新pod，覆盖每个容器的镜像拉取策略。
        设置pod.spec.InitContainer[i].ImagePullPolicy为"Always"，设置pod.spec.Container[i].ImagePullPolicy为"Always"。
     
3. antiaffinity

        注册插件"LimitPodHardAntiAffinityTopology"，表示拒绝所有标记反亲和（除了kubeletapis.LabelHostname，例如"kubernetes.io/hostname"之外）的pod。和（2）一样，只对访问pod资源，才会考虑这个准入控制，如果是其他请求，则直接通过。
        支持CREATE、UPDATE操作。

4. defaulttolerationseconds

        注册插件"DefaultTolerationSeconds"，表示为每个pod增加默认tolerations taints："notReady:NoExecute"和"unreachable:NoExecute"，tolerationSeconds是300s。如果Pod已经设置了toleration taint："notReady:NoExecute"和"unreachable:NoExecute"，则不会触发插件。
        支持CREATE、UPDATE操作。

5. deny

        注册插件"AlwaysDeny"，表示拒绝所有请求，仅用于测试。支持所有操作，但是拒绝所有请求。

6. exec

        注册插件"DenyEscalatingExec"和"DenyExecOnPrivileged"。大部分时间"DenyEscalatingExec "优先。
        支持CONNECT操作。
        仅仅处理pod上的exec和attach请求，即访问："pods/exec"和"pods/attach"时才执行该策略，否则直接通过。
        不能使用hostPID来exec或attach到容器，不能使用hostIPC来exec或attach到容器，不能exec或attach到一个特权容器。

7. gc

        注册插件"OwnerReferencesPermissionEnforcement"。如果请求在白名单里，则跳过检查。该插件还没有研究过。 支持CREATE、UPDATE操作。

8. imagepolicy

        注册插件"ImagePolicyWebhook"。只验证pod上的请求。 支持CREATE、UPDATE操作。

9. initialization

        注册插件"Initializers"。支持CREATE、UPDATE操作。

10. initialresources

        注册插件"InitialResources"。支持CREATE操作。

11. limitranger

        注册插件"LimitRanger"。支持CREATE、UPDATE操作。

12. autoprovision

        注册插件"NamespaceAutoProvision"。支持CREATE操作。

13. exists

        注册插件"NamespaceExists"。支持CREATE、UPDATE、DELETE操作。

14. noderestriction

        注册插件"NodeRestriction"。支持CREATE、UPDATE、DELETE操作。

15. label

        注册插件"PersistentVolumeLabel"。支持CREATE操作。
     
16. podnodeselector

        注册插件"PodNodeSelector"。支持CREATE操作。

17. podpreset

        注册插件"PodPreset"。支持CREATE、UPDATE操作。

18. podtolerationrestriction

        注册插件"PodTolerationRestriction"。支持CREATE、UPDATE操作。

19. resourcequota

        注册插件"ResourceQuota"。支持CREATE、UPDATE操作。

20. podsecuritypolicy

        注册插件"PodSecurityPolicy"。支持CREATE、UPDATE操作。

21. scdeny

        注册插件"SecurityContextDeny"。支持CREATE、UPDATE操作。

22. serviceaccount

        注册插件"ServiceAccount"。支持CREATE操作。

23. setdefault

        注册插件"DefaultStorageClass"。支持CREATE操作。

24. webhook

        注册插件"GenericAdmissionWebhook"。支持CREATE、UPDATE、DELETE、CONNECT操作。

所以，如果你想扩展自己的准入控制插件，需要执行以下几步：

首先，在"k8s.io/kubernetes/plugins/pkg/admission"目录下创建你自己的插件子目录。例如，你想创建名字叫"myadmit"的插件，你可以创建"myadmit"子目录，并在该子目录下创建一个"admission.go"文件。

然后，在"admission.go"文件中编写你自己的插件注册函数Register、创建插件结构体函数、插件结构体及要实现的Admit和Handles方法，可以参考已经存在的插件注册方式。你需要关注的是Admit和Handles方法，其中Admit方法编写你的准入控制流程，Handles方法编写请求允许的操作。

最后，在"k8s.io/kubernetes/cmd/kube-apiserver/app/plugins.go "的registerAllAdmissionPlugins函数中添加你的插件注册函数：myadmit.Register(plugins)。


## 1.2 defaultOptions
含义：

    在创建generic config之前设置默认参数。前面已经创建了默认的服务器运行参数-ServerRunOptions，为什么这里还要设置默认参数-defaultOptions呢？有几种情况：1.之前有些特殊参数并未被设置，2.如果命令行传入的参数覆盖了默认值，而传入的参数又为空，则需要在这里补充设置默认值，3.有些默认值之间有关联，例如需要根据授权参数来修改认证参数，则可以在这里修改默认值。

路径：

    k8s.io/kubernetes/cmd/kube-apiserver/app/server.go

定义：

    func defaultOptions(s *options.ServerRunOptions) error{
        //设置AdvertiseAddress的默认值，如果AdvertiseAddress未设置值，则此处会设置一个默认值。
        if err:=s.GenericServerRunOptions.DefaultAdvertiseAddress(s.SecureServing);err!=nil{
            return err
        }
        if err:=kubeoptions.DefaultAdvertiseAddress(s.GenericServerRunOptions, s.InsecureServing); err!=nil{
            return err
        }

        //获取apiserver的Service IP，为创建自签名证书时使用。
        _,apiServerServiceIP, err:=master.DefaultServiceIPRange(s.ServiceClusterIPRange)
        if err!=nil{
            return fmt.Errorf("error determing service IP ranges: %v", err)
        }

        //设置安全访问的证书cert和key。首选"--tls-cert-file"或"--tls-key-file"，如果它们不存在，则选择CertDirectory和PairName，如果它们也不存在，则创建自签名证书。
        if err:=s.SecureServing.MaybeDefaultWithSelfSignedCerts(s.GenericServerRunOptions.AdvertiseAddress.String(), []string{"kubernetes.default.svc", "kubernetes.default", "kubernetes"}, []net.IP{apiServerServiceIP}); err!=nil{
            return fmt.Errorf("error create self-signed certificates: %v", err)
        }

        //如果是gce或aws云，则设置相应的ExternalHost。
        if err:=s.CloudProvider.DefaultExternalHost(s.GenericServerRunOptions); err!=nil{
            return fmt.Errorf("error setting the external host value: %v", err)
        }

        //根据授权参数有条件地修改认证参数。如果授权模式是AlwaysAllow，则会强制把Anonymous.Allow（匿名认证）改为false，并发送警告信息。
        s.Authentication.ApplyAuthorization(s.Authorization)

        //设置ServiceAccount认证的keyfile。如果未设置Authentication.ServiceAccounts.KeyFile（"--service-account-key-file"），则使用SecureServing.ServerCert.CertKey.KeyFile（此处已经被前面的自签名证书赋值了，默认是/var/run/kubernetes/apiserver.key）来赋值。
        if len(s.Authentication.ServiceAccounts.KeyFiles)==0 && s.SecureServing.ServerCert.CertKey.KeyFile!=""{
            if kubeauthenticator.IsValidServiceAccountKeyFile(s.SecureServing.ServerCert.CertKey.KeyFile){
                s.Authentication.ServiceAccounts.KeyFiles = []string{s.SecureServing.ServerCert.CertKey.KeyFile}
            }else{
                glog.Warning("No TLS key provided, service account token authentication disabled")
            }
        }

        //重新设置Etcd.StorageConfig.DeserializationCacheSize。官方建议每个node内存60MB，2000个node共120G。DeserializationCacheSize最小值为1000MB。
        if s.Etcd.StorageConfig.DeserializationCacheSize == 0{
            glog.V(2).Infof("Initializing deserialization cache size based on %dMB limit", s.GenericServerRunOptions.TargetRAMMB)

            clusterSize:=s.GenericServerRunOptions.TargetRAMMB / 60
            s.Etcd.StorageConfig.DeserializationCacheSize = 25*clusterSize
            if s.Etcd.StorageConfig.DeserializationCacheSize < 1000 {
                s.Etcd.StorageConfig.DeserializationCacheSize = 1000
            }
        }
        //设置WatchCacheSizes
        if s.Etcd.EnableWatchCache{
            glog.V(2).Infof("Initializing cache sizes based on %dMB limit", s.GenericServerRunOptions.TargetRAMMB)
            cachesize.InitializeWatchCacheSizes(s.GenericServerRunOptions.TargetRAMMB)
            cachesize.SetWatchCacheSizes(s.GenericServerRunOptions.WatchCacheSizes)
        }
        return nil
    }

### 1.2.1 s.GenericServerRunOptions.DefaultAdvertiseAddress & kubeoptions.DefaultAdvertiseAddress
含义：

    设置ServerRunOptions结构体的"AdvertiseAddress"参数。如果存在安全服务器参数，则使用安全参数来设置，否则使用非安全参数来设置。具体步骤如下：如果未设置AdvertiseAddress，则根据BindAddress来设置AdvertiseAddress。执行如下：如果BindAddress为nil或者0.0.0.0或者loopback，则根据"/proc/net/route"找到host ip（从CIDR地址中获得）作为BindAddress的值，并赋给AdvertiseAddress；否则将BindAddress的值赋给AdvertiseAddress。建议：显示设置AdvertiseAddress参数，例如设置命令行参数--advertise-address。

路径：

    k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/server/options/server_run_options.go和k8s.io/kubernetes/pkg/kubeapiserver/options/serving.go

定义：

    func (s *ServerRunOptions) DefaultAdvertiseAddress(secure *SecureServingOptions) error{
        if secure==nil{
            return nil
        }
        if s.AdvertiseAddress==nil || s.AdvertiseAddress.IsUnspecified(){
            hostIP,err:=secure.DefaultExternalAddress()
            if err!=nil{
                return fmt.Errof("Unable to find suitable network address.error='%v'. Try to set the AdvertiseAddress directly or provide a valid BindAddress to fix this.",err)
            }
            s.AdvertiseAddress = hostIP
        }
        return nil
    }
    func DefaultAdvertiseAddress(s *genericoptions.ServerRunOptions, insecure *InsecureServingOptions) error{
        if insecure==nil{
            return nil
        }
        if s.AdvertiseAddress==nil || s.AdvertiseAddress.IsUpspecified(){
            hostIP, err:=insecure.DefaultExternalAddress()
            if err!=nil{
                return fmt.Errorf("Unable to find suitable network address.error='%v'. Try to set the AdvertiseAddress directly or provide a valid BindAddress to fix this.", err)
            }
            s.AdvertiseAddress=hostIP
        }
        return nil
    }

### 1.2.2 master.DefaultServiceIPRange
含义：

    获取apiserver的Service IP，即ServiceClusterIPRange的第一个有效ip。

路径：

    k8s.io/kubernetes/pkg/master/services.go

定义：

    func DefaultServiceIPRange(passedServiceClusterIPRange net.IPNet) (net.IPNet, net.IP, error){
        serviceClusterIPRange := passedServiceClusterIPRange

        //如果未指定ServiceClusterIPRange，则默认CIDR为"10.0.0.0/24"，ServiceClusterIPRange为{10.0.0.0 ffffff00}
        if passedServiceClusterIPRange.IP == nil{
            defaultNet:="10.0.0.0/24"
            glog.Infof("Network range for service cluster IPs is unspecified. Defaulting to %v.", defaultNet)
            _,defaultServiceClusterIPRange, err:=net.ParseCIDR(defaultNet)
            if err!=nil{
                return net.IPNet{}m net.IP{}, err
            }
            serviceClusterIPRange = *defaultServiceClusterIPRange
        }
        if size:=ipallocator.RangeSize(&serviceClusterIPRange); size<8{
            return net.IPNet{}, net.IP{}, fmt.Errorf("The service cluster IP range must be at least %d IP addresses", 8)
        }

        //取ServiceClusterIPRange第一个ip作为apiserver的服务ip。如果ServiceClusterIPRange取默认值，则apiServerServiceIP为10.0.0.1。如果设置了"--service-cluster-ip-range"，则取该子网的第一个ip，例如xx.xx.xx.1。
        apiServerServiceIP, err:=ipallocator.GetIndexedIP(&serviceClusterIPRange,1)
        if err!=nil{
            return net.IPNet{}, net.IP{}, err
        }
        glog.V(4).Infof("Setting service IP to %q (read-write).", apiServerServiceIP)

        //虽然返回值有3个，但是我们只用第2个参数，即apiserver的服务地址-apiServerServiceIP。
        return serviceClusterIPRange, apiServerServiceIP, nil
    }

### 1.2.3 s.SecureServing.MaybeDefaultWithSelfSignedCerts
含义：

    设置证书文件certfile和keyfile。如果设置了"--tls-cert-file"或"--tls-private-key-file"，则使用它们指定的证书文件；否则使用CertDirectory和PairName指定的证书。如果CertDirectory和PairName指定的证书不可读，则创建自签名证书。创建自签名证书的方法前面已经介绍过了。

路径：

    k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/server/options/serving.go

实参：

    s.GenericServerRunOptions.AdvertiseAddress.String()（AdvertiseAddress）
    []string{"kubernetes.default.svc"
    "kubernetes.default", "kubernetes"}
    []net.IP{apiServerServiceIP（apiserver服务ip）

定义：

    func (s *SecureServingOptions)MaybeDefaultWithSelfSignedCerts(publicAddress string, alternateDNS []string, alternateIPs []net.IP)error{
        if s==nil{
            return nil
        }
        kerCert:=&s.ServerCert.CertKey
        //如果设置了"--tls-cert-file"或"--tls-private-key-file"，或者不是安全端口，则不进行下面的判断流程，直接返回。
        if s.BindPort==0 || len(keyCert.CertFile)!=0 || len(keyCert.KeyFile)!=0{
            return nil
        }
     
        //如果未设置"--tls-cert-file"和"--tls-key-file"，则使用CertDirectory和PairName共同决定crt和key。默认：/var/run/kubernetes/apiserver.crt、/var/run/kubernetes/apiserver.key。值写回keyCert.CertFile和keyCert.KeyFile参数中，后面的serviceaccount的keyfile会用到keyCert.KeyFile。
        keyCert.CertFile = path.Join(s.ServerCert.CertDirectory, s.ServerCert.PairName+".crt")
        keyCert.KeyFile = path.Join(s.ServerCert.CertDirectory, s.ServerCert.PairName+".key")

        //判断certfile和keyfile是否可读，2个文件都存在并且都能打开则为可读。
        canReadCertAndKey, err:=certutil.CanReadCertAndKey(keyCert.CertFile, keyCert.KeyFile)
        if err!=nil{
            return err
        }
        //如果可读，则直接退出，否则执行下面流程，即创建自签名certfile和keyfile。
        if !canReadCertAndKey{
            bindIP := s.BindAddress.String()
            if bindIP=="0.0.0.0"{
                alternateDNS = append(alternateDNS, "localhost")     //如果bind-address=0.0.0.0，则把localhost添加到后备DNS中。
            }else{
                alternateIPs = append(alternateIPs, s.BindAddress)      //如果bind-address != 0.0.0.0，则把BindAddress添加到后备IP中。
            }

            //创建证书，首先创建密钥（包含公钥和私钥）key，然后用密钥自签名创建证书cert，subj=AdvertiseAddress，有效期365天。关于自签名的方法前面已经详细介绍过了。
            if cert,key,err:=certuil.GenerateSelfSignedCertKey(publicAddress, alternateIPs, alternateDNS); err!=nil{
                return fmt.Errorf("unable to generate self signed cert: %v",err)
            }else{
                //如果想修改证书目录，就需要设置命令行参数：--cert-dir。证书文件名不能修改，除非修改源码。
                //写到keyCert.CertFile所指的文件中，默认为：/var/run/kubernetes/apiserver.crt。
                if err:=certutil.WriteCert(keyCert.CertFile, cert);err!=nil{
                    return err
                }
                //写到keyCert.KeyFile所指的文件中，默认为：/var/run/kubernetes/apiserver.key
                if err:=certutil.WriteKey(keyCert.KeyFile, key);err!=nil{
                    return err
                }
                glog.Infof("Generated self-signed cert (%s, %s)", KeyCert.CertFile, keyCert.keyFile)
            }
        }
        return nil
    }

### 1.2.4 s.CloudProvider.DefaultExternalHost
含义：

    如果是gce或aws云，则设置相应的ExternalHost。

路径：

    k8s.io/kubernetes/pkg/kubeapiserver/options/cloudprovider.go

定义：

    func (s *CloudProviderOptions) DefaultExternalHost(genericoptions *genericoptions.ServerRunOptions) error{
        //如果设置了ExternalHost，即"--external-hostname"，则使用该值；否则执行下面流程来设置ExternalHost。
        if len(genericoptions.ExternalHost)!=0{
            return nil
        }

        //如果是gce或aws云，则使用这两个云提供的node的地址作为ExternalHost，此时地址类型为ExternalIP。
        if s.CloudProvider=="gce" || s.CloudProvider=="aws"{
            cloud,err:=cloudprovider.InitCloudProvider(s.CloudProvider, s.CloudConfigFile)
            if err!=nil{
                return fmt.Errorf("%q cloud provider could not be initialized: %v", s.CloudProvider, err)
            }
            instances, supported:=cloud.Instances()
            if !supported{
                return fmt.Errorf("%q cloud provider has no instances", s.CloudProvider)
            }
            hostname, err:=os.Hostname()
            if err!=nil{
                return fmt.Errorf("failed to get hostname: %v", err)
            }
            nodeName,err:=instances.CurrentNodeName(hostname)
            if err!=nil{
                return fmt.Errorf("failed to get NodeName from %q cluod provider: %v", s.CloudProvider, err)
            }
            addrs,err:=instances.NodeAddresses(nodeName)
            if err!=nil{
                return fmt.Errorf("failled to get external host address from %q cloud provider: %v", s.CloudProvider, err)
            }else{
                for _,addr:=range addrs{
                    if addr.Type == v1.NodeExternalIP{
                        genericoptions.ExternalHost = addr.Address
                    }
                }
            }
        }
        return nil
    }

### 1.2.5 s.Authentication.ApplyAuthorization
含义：

    根据授权参数有条件地修改认证参数。具体是：如果设置了Anonymous.Allow=true（允许匿名认证），则和AlwaysAllow模式的授权策略有冲突，如果是AlwaysAllow模式的授权策略，则会把Anonymous.Allow改为false，并发送警告信息。

路径：

    k8s.io/kubernetes/pkg/kubeapiserver/options/authentication.go

定义：

    func (o *BuiltInAuthenticationOptions) ApplyAuthorization(authorization *BuiltInAuthorizationOptions){
        if o==nil || authorization==nil||o.Anonymous==nil{
            return
        }
        if o.Anonymous.Allow{
            found := false
            for _,mode:=range strings.Split(authorization.Mode, ","){
                if mode==authzmodes.ModeAlwaysAllow{
                    found=true
                    break
                }
            }
            if found{
                glog.Warningf("AnonymousAuth is not allowed with the AllowAll authorizer. Resetting AnonymousAuth to false. You should use a different authorizer")
                o.Anonymous.Allow = false
            }
        }
    }

### 1.2.6 cachesize.InitializeWatchCacheSizes
含义：

    根据TargetRAMMB来设置各资源watch缓存的初始值，该初始值会被命令行参数"--watch-cache-sizes"的值（即后面的SetWatchCacheSizes）覆盖。"--watch-cache-sizes"的格式：resource1#size1,resource2#size2,...（resource可以取："controllers", "endpoints", 等等，资源定义请参见："k8s.io/kubernetes/pkg/registry/cachesize/cachesize.go"。size单位是：MB。）

路径：

    k8s.io/kubernetes/pkg/registry/cachesize/cachesize.go

实参：

    s.GenericServerRunOptions.TargetRAMMB

定义：

    func InitializeWatchCacheSizes(expectedRAMCapacityMB int){
        clusterSize                 := expectedRAMCapacityMB / 60
        watchCacheSizes[Controllers] = maxInt(5*clusterSize, 100)       //Controllers，最小100MB
        watchCacheSizes[Endpoints]   = maxInt(10*clusterSize, 1000)     //Endpoints，最小1000MB
        watchCacheSizes[Nodes]       = maxInt(5*clusterSize, 1000)      //Nodes，最小1000MB
        watchCacheSizes[Pods]        = maxInt(50*clusterSize, 1000)     //Pods，最小1000MB
        watchCacheSizes[Services]    = maxInt(5*clusterSize, 1000)      //Services，最小1000MB
        watchCacheSizes[APIServices] = maxInt(5*clusterSize, 1000)      //APIServices，最小1000MB
    }
    var watchCacheSizes map[Resource]int
    type Resource string

### 1.2.7 cachesize.SetWatchCacheSizes
含义：

    设置WatchCacheSizes。使用命令行参数"--watch-cache-sizes"覆盖初始化值。

路径：

    k8s.io/kubernetes/pkg/registry/cachesize/cachesize.go

实参：

    s.GenericServerRunOptions.WatchCacheSizes

定义：

    func SetWatchCacheSizes(cacheSizes []string){
        for _,c:=range cacheSizes{
            tokens := strings.Split(c,"#")
            if len(tokens)!=2{
                glog.Errorf("invalid value of watch cache capabilities: %s", c)
                continue
            }
            size,err:=strconv.Atoi(tokens[1])
            if err!=nil{
                glog.Errorf("invalid size of watch cache capabilities: %s",c)
                continue
            }
            //resource资源名转为小写，size大小为MB。
            watchCacheSizes[Resource(strings.ToLower(tokens[0]))] = size
        }
    }

## 1.3 s.Validate
含义：

    验证参数。

路径：

    k8s.io/kubernetes/cmd/kube-apiserver/app/options/validation.go

定义：

    func (options *ServerRunOptions) Validate() []error{
        var errors []error

        //etcd必须指定StorageConfig.ServerList，即"--etcd-servers"
        if errs:=options.Etcd.Validata(); len(errs)>0{
            errors = append(errors, errs...)
        }

        //必须设置ServiceClusterIPRange，即"--service-cluster-ip-range"
        if errs:=validateClusterIPFlags(options); len(errs)>0{
            errors = append(errors, errs...)
        }

        //KubernetesServiceNodePort范围必须在[0-65535]之间，如果是0，则type=ClusterIP。KubernetesServiceNodePort 范围必须在ServiceNodePortRange之内，默认：[30000,32767]
        if errs:=validateServiceNodePort(options); len(errs)>0{
            errors = append(errors, errs...)
        }

        //安全端口--secure-port必须在[0-65535]之间。
        if errs:=options.SecureServing.Validate(); len(errs)>0{
            errors = append(errors, errs...)
        }

        //认证参数中，oidc-issuer-url和oidc-client-id必须一起指定。
        if errs:=options.Authentication.Validate(); len(errs)>0{
            errors = append(errors, errs...)
        }

        //非安全端口--insecure-port必须在[0-65535]之间。
        if errs:=options.InsecureServing.Validate("insecure-port"); len(errs)>0{
            errors = append(errors, errs...)
        }

        //master数量必须大于0
        if options.MasterCount <= 0{
            errors = append(errors, fmt.Errorf("--apiserver-count should be a positive number, but value '%d' provided", options.MasterCount))
        }
        return errors
    }

## 1.4 BuildGenericConfig
含义：

    利用ServerRunOptions配置参数创建服务器通用配置-genericapiserver.Config。

路径：

    k8s.io/kubernetes/cmd/kube-apiserver/app/server.go

参数：

    ServerRunOptions。

返回值：

    genericapiserver.Config
    informers.SharedInformerFactory
    kubeserver.InsecureServingInfo

定义：

    func BuildGenericConfig(s *options.ServerRunOptions)(*genericapiserver.Config, informers.SharedInformerFactory, *kubeserver.InsecureServingInfo, error){
        //创建一个通用Config对象。
        genericConfig := genericapiserver.NewConfig(api.Codecs)

        //将GenericServerRunOptions中的参数赋给genericConfig中的参数。
        if err:= s.GenericServerRunOptions.ApplyTo(genericConfig); err!=nil{
            return nil,nil, nil,err
        }

        //将InsecureServing参数赋给genericConfig中的参数。
        insecureServingOptions,err:=s.InsecureServing.ApplyTo(genericConfig)
        if err!=nil{
            return nil,nil,nil,err
        }

        //设置安全参数。包括客户端设置。
        if err:=s.SecureServing.ApplyTo(genericConfig); err!=nil {
            return nil,nil,nil,err
        }

        //设置认证参数
        if err:=s.Authentication.ApplyTo(genericConfig);err!=nil{
            return nil,nil,nil,err
        }

        //设置审计参数
        if err:=s.Audit.ApplyTo(genericConfig); err!=nil {
            return nil,nil,nil,err
        }

        //设置特征开关。
        if err:=s.Features.ApplyTo(genericConfig); err!=nil{
            return nil,nil,nil,err
        }

        //设置OpenAPI参数
        genericConfig.OpenAPIConfig                 = genericapiserver.DefaultOpenAPIConfig(generatedopenapi.GetOpenAPIDefinitions, api.Scheme)
        genericConfig.OpenAPIConfig.PostProcessSpec = postProcessOpenAPISpecForBackwardCompatibility
        genericConfig.OpenAPIConfig.Info.Title      = "Kubernetes"
        genericConfig.SwaggerConfig                 = genericapiserver.DefaultSwaggerConfig()
        genericConfig.EnableMetrics                 = true
        genericConfig.LongRunningFunc               = filters.BasicLongRunningRequestCheck{
            sets.NewString("watch", "proxy"),
            sets.NewString("attach", "exec", "proxy", "log", "postforward"),
        }

        kubeVersion          := version.Get()
        genericConfig.Version = &kubeVersion

        //设置etcd后端存储的storageFactory。
        storageFactory,err := BuildStorageFactory(s)
        if err!=nil{
            return nil,nil,nil,err
        }
        if err:=s.Etcd.ApplyWithStorageFactoryTo(storageFactory, genericConfig); err!=nil {
            return nil,nil,nil,err
        }

        //设置LoopbackClientConfig的类型
        genericConfig.LoopbackClientConfig.ContentConfig.ContentType = "application/vnd.kubernetes.protobuf"

        //创建内部版本客户端配置。
        client,err:=internalclientset.NewForConfig(genericConfig.LoopbackClientConfig)
        if err!=nil{
            kubeAPIVersions := os.Getenv("KUBE_API_VERSIONS")
            if len(kubeAPIVersions)==0{
                return nil,nil,nil,fmt.Errorf("failed to create clientset: %v", err)
            }
            glog.Errorf("Failed to create clientset with KUBE_API_VERSIONS=%q. KUBE_API_VERSIONS is only for testing. Things will break.", kubeAPIVersions)
        }

        //创建外部版本客户端配置。
        externalClient,err:=clientset.NewForConfig(genericConfig.LoopbackClientConfig)
        if err!=nil{
            return nil,nil,nil,fmt.Errorf("failed to create external clientset: %v", err)
        }

        //创建共享informers，客户端超时时间10m。
        sharedInformers := informers.NewSharedInformerFactory(client, 10*time.Minute)
     
        genericConfig.Authenticator, genericConfig.OpenAPIConfig.SecurityDefinitions, err = BuildAuthenticator(s, storageFactory, client, sharedInformers)
        if err!=nil{
            return nil,nil,nil,fmt.Errorf("invalid authentication config: %v", err)
        }
        genericConfig.Authorizer, err=BuildAuthorizer(s, sharedInformers)
        if err!=nil{
            return nil,nil,nil,fmr.Errorf("invalid authorization config: %v",err)
        }
        if !sets.NewString(s.Authorization.Modes()...).Has(modes.ModeRBAC){
            genericConfig.DisabledPostStartHooks.Insert(rbacrest.PostStartHookName)
        }
        pluginInitializer,err:=BuildAdmissionPluginInitializer(
            s,
            client,
            externalClient,
            sharedInformers,
            genericConfig.Authorizer,
        )
        if err!=nil{
            return nil,nil,nil,fmt.Errorf("failed to create admission plugin initializer: %v",err)
        }
        err=s.Admission.ApplyTo(
            genericConfig,
            pluginInitializer)
        if err!=nil{
          return nil,nil,nil,fmt.Errorf("failed to initialize admission: %v",err)
        }
        return genericConfig, sharedInformers, insecureServingOptions, nil
    }

### 1.4.1 genericapiserver.NewConfig
含义：

    创建通用Config对象。

路径：

    k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/server/config.go

实参：

    api.Codecs = serializer.NewCodecFactory(Scheme)。

定义：

    func NewConfig(codecs serializer.CodecFactory) *Config{
        return &Config{
            Serializer:                   codecs,                                        //序列化器，通过实参api.Codecs传入。后面会详细介绍Seiralizer。
            ReadWritePort:                443,                                           //PublicAddress绑定的端口。该值会在后面的secureServing过程中被安全BindPort端口覆盖，即变为6443。
            RequestContextMapper:         apirequest.NewRequestContextMapper(),          //该参数会传给下面的DefaultBuildHandlerChain使用。
            BuildHandlerChainFunc:        DefaultBuildHandlerChain,                      //定制handler chain，过滤器链。
            LegacyAPIGroupPrefixes:       sets.NewString(DefaultLegacyAPIPrefix),        //遗留API组的前缀，遗留API是指"/api"，新组都是"/apis"。
            DisabledPostStartHooks:       sets.NewString(), 
            HealthzChecks:                []healthz.HealthzChecker{healthz.PingHealthz}, //健康检查
            EnableIndex:                  true,             
            EnableDiscovery:              true,            
            EnableProfiling:              true,                                          //打开性能调试开关。后面可能会通过--profiling覆盖这里的值。
            MaxRequestsInFlight:          400,                                           //最大并发请求数。后面可能会通过--max-requests-inflight覆盖这里的值。
            MaxMutatingRequestsInFlight:  200,                                           //最大mutating并发请求数。后面可能通过--max-mutating-requests-inflight覆盖这个值。
            MinRequestTimeout:            1800,                                          //最小请求超时时间。后面可能会通过--min-request-timeout覆盖这里的值。
            LongRunningFunc:              genericfilters.BasicLongRunningRequestCheck(sets.NewString("watch"),sets.NewString()),    //长时运行请求。
        }
    }

    //GenericAPIServer配置信息。
    type Config struct{
        SecureServingInfo            *SecureServingInfo
        LoopbackClientConfig         *restclient.Config
        Authenticator                authenticator.Request
        Authorizer                   authorizer.Authorizer
        AdmissionControl             admission.Interface
        CorsAllowedOriginList        []string
        EnableSwaggerUI              bool
        EnableIndex                  bool
        EnableProfiling              bool                                                            //性能调试开关
        EnableDiscovery              bool
        EnableContentionProfiling    bool
        EnableMetrics                bool
        DisabledPostStartHooks       sets.String
        Version                      *version.Info
        LegacyAuditWriter            io.Writer
        AuditBackend                 audit.Backend
        AuditPolicyChecker           auditpolicy.Checker
        SupportsBasicAuth            bool
        ExternalAddress              string
        SharedInformerFactory        informers.SharedInfomerFactory
        BuildHandlerChainFunc        func(apiHandler http.Handler, c *Config)(secure http.Handler)
        DiscoveryAddresses           discovery.Address
        HealthzChecks                []healthz.HealthChecker
        LegacyAPIGroupPrefixes       sets.String
        RequestContextMapper         apirequest.RequestContextMapper
        Serializer                   runtime.NegotiatedSerializer                                    //序列化器
        OpenAPIConfig                *openapicommon.Config
        SwaggerConfig                *swagger.Config
        RESTOptionsGetter            genericregistry.RESTOptionsGetter
        MinRequestTimeout            int
        MaxRequestsInFlight          int
        MaxMutatingRequestsInFlight  int
        LongRunningFunc              apirequest.LongRunningRequestCheck
        ReadWritePort                int
        PublicAddress                net.IP
    }
    type SecureServingInfo struct{
        BindAddress                  string                         //ip:port
        BindNetwork                  string                         //网络类型，可取值："tcp", "tcp4", "tcp6"。"-"表示默认值，即"tcp"。
        Cert                         *tls.Certificate               //主要使用的服务器证书。
        CACert                       *tls.Certificate               //用于准入控制器的loopback连接。
        SNICerts                     map[string]*tls.Certificate    //SNI域名证书。
        ClientCA                     *x509.CertPool                 //证书池，拥有所有数字签名者，用于验证即将到来的客户端证书。
        MinTLSVersion                uint16                         //最小TLS版本，会覆盖支持的最小TLS版本。值来自于tls包常量。
        CipherSuites                 []uint16                       //加密套件，值来自tls包，会覆盖服务器允许的加密套件。
    }

1. Serializer
    
        含义：用于编解码。后面会详细介绍。
        值：实参api.Codecs。
        路径：k8s.io/kubernetes/pkg/api/registry.go
        var Codecs = serializer.NewCodecFactory(Scheme)

2. apirequest.NewRequestContextMapper
     
        含义：返回requestContextMap对象，该对象传给hander实现对请求进行过滤的功能。该返回值会被下面的DefaultBuildHandlerChain使用。
        路径：k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/endpoints/request/requestcontext.go
        func NewRequestContextMapper() RequestContextMapper {
            return &requestContextMap{
                contexts: make(map[*http.Request]Context),
            }     
        }

        //跟踪特定的request Context（请求上下文）
        type RequestContextMapper interface{
            Get(req *http.Request) (Context, bool)                  //获得请求的Context，如果请求包含Context，则返回true；否则返回false。
            Update(req *http.Request, context Context) error        //使用给定的Context来更新请求的Context。
        }
        //requestContextMap实现RequestContextMapper接口
        type requestContextMap struct{
            contexts map[*http.Request]Context
            lock sync.Mutex
        }
        func (c *requestContextMap) Get(req *http.Request) (Context, bool){
            c.lock.Lock()
            defer c.lock.Unlock()
            context,ok:=c.contexts[req]
            return context,ok
        }
        func (c *requestContextMap) Update(req *http.Request, context Context) error{
            c.lock.Lock()
            defer c.lock.Unlock()
            if _, ok:=c.contexts[req]; ok{
                return errors.New("No context associated")
            }
            c.contexts[req] = context
            return nil
        }

3. DefaultBuildHandlerChain
     
        含义：创建默认的过滤器链。注意：请求通过过滤器链是从后往前执行，即先执行WithRequest、在执行WithRequestInfo，最后执行WithAuthorization。
        路径：k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/server/config.go
        参数：http.Handler, Config
        func DefaultBuildHandlerChain(apiHandler http.Handler, c *Config) http.Handler{
            handler := genericapifilters.WithAuthorization(apiHandler, c.RequestContextMapper, c.Authorizer)                //Authorizer过滤
            handler := genericapifilters.WithImpersonation(handler, c.RequestContextMapper, c.Authorizer)                   //检查请求中user.Info信息，防止冒充用户。
            if utilfeature.DefaultFeatureGate.Enabled(features.AdvancedAuditing){
                handler = genericapifilters.WithAudit(handler, c.RequestContextMapper, c.AuditBackend, c.AuditPolicyChecker, c.LongRunningFunc) //审计日志过滤器
            }else{
                handler = genericapifilters.WithLegacyAudit(handler, c.RequestContextMapper, c.LegacyAuditWriter)           //审计日志过滤器
            }
            handler = genericapifilters.WithAuthentication(handler, c.RequestContextMapper, c.Authenticator, genericapifilters.Unauthorized(c.SupportsBasicAuth)) //认证过滤器
            handler = genericfilters.WithCORS(handler, c.CorsAllowedOriginList, nil, nil, nil, "true")
            handler = genericfilters.WithPanicRecovery(handler)
            handler = genericfilters.WithTimeoutForNonLongRunningRequests(handler, c.RequestContextMapper, c.LongRunningFunc)
            handler = genericfilters.WithMaxInFlightLimit(handler,c.MaxRequestsInFlight, c.MaxMutatingRequestsInFlight, c.RequestContextMapper, c.LongRunningFunc)
            handler = genericfilters.WithRequestInfo(handler, NewRequestInfoResolver(c), c.RequestContextMapper)
            handler = apirequest.WithRequestContext(handler, c.RequestContextMapper)
            return handler
        }

**过滤器链原理**：

    过滤器要实现http.Handler接口，该接口有个方法ServeHTTP(ResponseWriter,*Request)，即过滤器必须实现ServeHTTP方法。默认的HandlerFunc类型（一种函数类型）就实现了ServeHTTP方法，因此我们使用HandlerFunc来实现默认过滤器。

    //go/src/net/http/server.go 
    type Handler interface{
        ServeHTTP(ResponseWriter, *Request)
    }
    type HandlerFunc func(ResponseWriter, *Request)
    func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request){
        f(w, r)
    }

下面介绍各种过滤器：

（a）授权过滤器

含义：

    执行授权行为的过滤器，使用闭包概念，即函数和上下文包装在一起。requestContextMapper作为参数在http.HandlerFunc函数中使用。http.HandlerFunc函数作为返回值返回，http.HandlerFunc函数的处理方式依据包装的上下文，例如requestContextMapper变量。下面几种过滤器都是闭包实现。

定义：

    func WithAuthorization(handler http.Handler, requestContextMapper request.RequestContextMapper, a authorizer.Authorizer) http.Handler {
        //如果没有提供Auhorizer，则返回handler，执行后面的过滤器。
        if a==nil{
            glog.Warningf("Authorization is disabled")
            return handler
        }
        return http.HandlerFunc(func(w http.ResponseWriter, req *http.Request){
            //根据请求获取上下文，调用requestContextMapper.Get方法。
            ctx,ok:=requestContextMapper.Get(req)
            if !ok{
                responsewriters.InternalError(w,req,errors.New("no context found for request"))
                return
            }

            //从上下文中获取属性，用于授权过程。属性包括：User、Path、Verb、IsResourceRequest、APIGroup、APIVersion、Resource、Subresource、Namespace、Name、User。
            attributes,err:=GetAuthorizerAttributes(ctx)
            if err!=nil{
                responsewriters.InternalError(w,req,err)
                return
            }

            //授权过程
            authorized,reason,err:=a.Authorize(attributes)
            if authorized{
                handler.ServeHTTP(w,req)
                return
            }
            if err!=nil{
                responsewriters.InternalError(w,req,err)
                return
            }
            glog.V(4).Infof("Forbidden: %#v, Reason: %q", req.RequestURI, reason)
            responsewriters.Forbidden(attributes, w, req, reason)
        })
    }

（b）WithImpersonation-授权检查

含义：

    检查请求中user.Info信息，防止冒充用户。

定义：

    func WithImpersonation(handler http.Handler, requestContextMapper request.RequestContextMapper, a authorizer.Authorizer) http.Handler{
        return http.HandlerFunc(func(w http.ResponseWriter, req *http.Request){
            //从request中获取"Impersonate-User"、"Impersonate-Group"、"Impersonate-Extra"信息。
            impersonationRequests,err:=buildImpersonationRequests(req.Header)
            if err!=nil{
                glog.V(4).Infoof("%v", err)
                responsewriters.InternalError(w,req,err)
                return
            }

            //如果请求中不存在Impersonate类型字段，则退出，执行后续过滤器。
            if len(impersonationRequests)==0{
                hanlder.ServeHTTP(w,req)
                return
            }

            //从请求中获取上下文
            ctx,exists:=requestContextMapper.Get(req)
            if !exists{
                responsewriters.InternalError(w, req, errors.New("no context found for request"))
                return
            }

            //从上下文中获取用户
            requestor,exists:=request.UserForm(ctx)
            if !exists{
                responsewriters.InternalError(w, req, errors.New("no user found for request"))
                return
            }
            groupsSpecified:=len(req.Header[authenticationv1.ImpersonateGroupHeader')>0
            username:=""
            groups:=[]string{}
            userExtra:=map[string][]string{}
            for _,impersonationRequest:=range impersonationRequests{
                //设置属性，Verb为impersonate。
                actingAsAttributes := &authorizer.AttributesRecord{
                    User: requestor,
                    Verb: "impersonate",
                    APIGroup: impersonationRequest.GetObjectKind().GroupVersionKind().Group,
                    Namespace: impersonationRequest.Namespace,
                    Name: impersonationRequest.Name,
                    ResourceRequest: true,
                }
                switch impersonationRequest.GetObjectKind().GroupVersionKind().GroupKind(){
                case v1.SchemeGroupVersion.WithKind("ServiceAccount").GroupKind():
                    actingAsAttributes.Resource="serviceaccounts"
                    username=serviceaccount.MakeUsername(impersonationRequest.Namespace, impersonationRequest.Name)
                    if !groupsSpecified{
                        groups=serviceaccount.MakeGroupNames(impersonationRequest.Namespace, impersonationRequest.Name)
                    }
                case v1.SchemeGroupVersion.WithKind("User").GroupKind():
                    actingAsAttributes.Resource="users"
                    username=impersonationRequest.Name
                case v1.SchemeGroupVersion.WIthKind("Group").GroupKind():
                    actingAsAttributes.Resource="groups"
                    groups=append(groups, impersonationRequest.Name)
                case authenticationv1.SchemeGroupVersion.WithKind("UserExtra").GroupKind():
                    extraKey:=impersonationRequest.FieldPath
                    extraValue:=impersonationRequest.Name
                    actingAsAttributes.Resource="userextras"
                    actingAsAttributes.SubResource=extraKey
                    userExtra[extraKey]=append(userExtra[extraKey], extraValue)
                default:
                    glog.V(4).Infof("unknown impersonation request type: %v", impersonationRequest)
                    responsewriters.Forbidden(actingAsAttributes, w, req, fmt.Sprintf("unknown impersonation request type: %v", impersonationRequest))
                    return
                }

                //根据属性执行授权判断。
                allowed, reason, err := a.Authorize(actingAsAttributes)
                if err!=nil || !allowed{
                    glog.V(4).Infof("Forbidden: %#v, Reason: %s, Error: %v", req.RequestURI, reason, err)
                    responsewriter.Forbidden(actingAsAttributes, w,req,reason)
                    return
                }
            }
            if !groupsSpecified && username!=user.Anonymous{
                found:=false
                for _,group:=range groups{
                    if group==user.AllAuthenticated || group==user.AllUnauthenticated{
                        found=true
                        break
                    }
                }
                if !found{
                    groups=append(groups, user.AllAuthenticated)
                }
            }
            newUser:=&user.DefaultInfo{
                Name: username,
                Groups: groups,
                Extra: userExtra,
            }
            requestContextMapper.Update(req, request.WithUser(ctx, newUser))
            oldUser, _ := request.UserFrom(ctx)
            httplog.LogOf(req,w).Addf("%v is acting as %v", oldUser, newUser)

            //删除请求中的"Impersonate-"字段。
            req.Header.Del(authenticationv1.ImpersonateUserHeader)
            req.Header.Del(authenticationv1.ImpersonateGroupHeader)
            for headerName:=range req.Header{
                if strings.HasPrefix(headerName, authenticationv1.ImpersonateUserExtraHeaderPrefix){
                    req.Header.Del(headerName)
                }
            }
            handler.ServeHTTP(w,req)
        })
    }

（c）genericapifilters.WithAudit & genericapifilters.WithLegacyAudit

含义：

    审计过滤。如果打开AdvancedAuditing开关，则执行WithAudit过滤器，否则执行WithLegacyAudit过滤器。

定义：

    func WithAudit(handler http.Handler, requestContextMapper request.RequestContextMapper, sink audit.Sink, policy policy.Checker, longRunningCheck request.LongRunningRequestCheck) http.Handler{
        if sink==nil || policy==nil{
            return handler
        }
        return http.HandlerFunc(func(w http.ResponseWriter, req *http.Request){
            //从request中获取context
            ctx, ok:=requestContextMapper.Get(req)
            if !ok{
                responsewriters.InternalError(w,req,errors.New("no context found for request"))
                return
            }
            //从context中获取attributes
            attribs, err:=GetAuthorizerAttributes(ctx)
            if err!=nil{
                utilruntime.HandleError(fmt.Errorf("failed to GetAuthorizerAttributes: %v", err))
                responsewriters.InternalError(w,req,errors.New("failed to parse request"))
                return
            }
            //判断规则是否匹配请求的属性来决定审计级别。有效的基本有："None"（不审计，默认）、"Metadata"（基本审计）、"Request"（既有Metadata审计，又外加请求对象的审计日志，不用于非资源的请求）、"RequestResponse"（既有Request审计，又外加响应对象的审计日志，不用于非资源的请求 ）
            level:=policy.Level(attribs)                 
            audit.ObservePolicyLevel(level)         //根据审计级别设置prometheus metrics
            //不审计，就退出。执行后面的过滤器。
            if level==auditinternal.LevelNone{
                handler.ServeHTTP(w,req)
                return
            }

            //创建审计事件。审计事件包括：Timestamp（创建审计事件的时间）、Verb（属性的操作）、RequestURL（请求URL）、Level（审计级别）、AuditID（通过请求的"Audit-ID"字段来获取，如果存在该字段，则使用该字段的值，否则使用uuid方法创建一个AuditID）、SourceIp（请求源ip，如果请求包含"X-Forwarded-For"字段（逗号分隔的ip），则解析该字段获得源ip；否则检查请求是否包含"X-Real-Ip"字段，有则使用它，没有则检查请求的RemoteAddr，ip:port格式，取前面的ip作为源ip）、User（Username、Extra、Groups、UID）、ImpersonateUser（检查请求的"Impersonate-User"赋给Username，"Impersonate-Group"赋给Group，"Impersonate-Extra"赋给Extra）、ObjectRef（Namespace、Name、Resource、Subresource、APIVersion）。
            ev,err:=audit.NewEventFromRequest(req, level, attribs)
            if err!=nil{
                utilruntime.HandleError(fmt.Errorf("failed to complete audit event from request: %v", err))
                responsewriters.InternalError(w, req, errors.New("failed to update context"))
                return
            }

            //设置审计事件。
            ctx=request.WithAuditEvent(ctx,ev)
            if err:=requestContextMapper.Update(req, ctx); err!=nil{
                utilruntime.HandleError(fmt.Errorf("failed to attach audit event to the context: %v",err))
                responsewriters.InternalError(w,req,errors.New("failed to update context"))
                return
            }

            //审计处理器接收到请求的阶段
            ev.Stage=auditinternal.StageRequestReceived
            
            //处理审计事件
            processEvent(sink,ev)
            var longRunningSink audit.Sink
            if longRunningCheck != nil{
                ri,_:=request.RequestInfoFrom(ctx)
                if longRunningCheck(req, ri){
                    longRunningSink = sink
                }
            }
            respWriter:=decorateResponseWriter(w,ev,longRunningSink)
            defer func(){
                if r:=recover();r!=nil{
                    defer panic(r)
                    ev.Stage=auditinternal.StagePanic
                    ev.ResponseStatus = &metav1.Status{
                        Code: http.StatusInternalServerError,
                        Status: metav1.StatusFailure,
                        Reason: metav1.StatusReasonInternalError,
                        Message: fmt.Sprintf("APIServer panic'd:%v",r),
                    }
                    processEvent(sink,ev)
                    return
                }
                fakedSuccessStatus := &metav1.Status{
                    Code: http.StatusOK,
                    Status: metav1.StatusSuccess,
                    Message:"Connection closed early"
                }
                if ev.ResponseStatus==nil && longRunningSink!=nil{
                    ev.ResponseStatus=fakedSuccessStatus
                    ev.Stage=auditinternal.StageResponseStarted
                    processEvent(longRunningSink, ev)
                }
                ev.Stage=auditinternal.StageResponseComplete
                if ev.ResponseStatus==nil{
                    ev.ResponseStatus=fakedSuccessStatus
                }
                processEvent(sink,ev)
            }()
            handler.ServeHTTP(respWriter,req)
        })
    }

    //"/api"请求的审计。每个审计日志包括2部分：1：请求行，包括：uid，source ip，http method，original user，original group，impersonated user，impersonated group，namespace，uri。2：响应行，包括：uid，响应码。
    func WithLegacyAudit(handler http.Handler, requestContextMapper request.RequestContextMapper, out io.Writer) http.Handler{
        if out==nil{
            return handler
        }
        return http.HandlerFunc(func(w http.ResponseWriter, req *http.Request){
            //根据请求获取上下文
            ctx,ok:=requestContextMapper.Get(req)
            if !ok{
                responsewriters.InternalError(w,req,errors.New("no context found for request"))
                return
            }
            //根据上下文获取属性
            attribs, err:=GetAuthorizerAttributes(ctx)
            if err!=nil{
                responsewriters.InternalError(w,req,err)
                return
            }
            username:="<none>"
            groups:="<none>"
            if attribs.GetUser()!=nil{
                username=attribs.GetUser().GetName()    //user
                if userGroups:=attribs.GetUser().GetGroups(); len(userGroups)>0{
                    groups=auditStringSlice(userGroups)  //group
                }
            }
            asuser:=req.Header.Get(authenticationapi.ImpersonateUserHeader)  //impersonate user
            if len(asuser)==0{
                asuser="<self>"
            }
            asgroups:="<lookup>"
            requestedGroups:=req.Header[authenticationapi.ImpersonateGroupHeader]
            if len(requestedGroups)>0{
                asgroups=auditStringSlice(requestedGroups)  //impersonate group
            }
            namespace:=attribs.GetNamespace()  //namespace
            if len(namespace)==0{
                namespace="<none>"
            }
            id:=uuid.NewRandom().String()  
            line:=fmt.Sprintf("%s AUDIT: id=%q ip=%q method=%q groups=%q as=%q asgroups=%q namespace=%q uri=%q\n",
                time.Now().Format(time.RFC3339Nano), id, utilnet.GetClientIP(req), req.Method, username, groups, asuser, asgroups, namespace, req,URL)
            if _,err:=fmt.Fprintf(out,line);err!=nil{
                glog.Errorf("Unable to write audit log:%s, the error is: %v", line, err)
            }
            respWriter:=legacyDecorateResponseWriter(w,out,id)  //response 
            handler.ServeHTTP(respWriter,req)
        })
    }

（d）WithAuthentication

含义：

    认证过滤器。

定义：

    func WithAuthentication(handler http.Handler, mapper genericapirequest.RequestContextMapper, auth authenticator.Request, failed http.Handler)http.Handler{
        if auth==nil{
            glog.Warning("Authentication is disabled")
            return handler
        }
        return genericapirequest.WithRequestContext(
            http.HandlerFunc(func(w http.ResponseWriter, req *http.Request){
                //获取请求中认证的用户信息
                user,ok,err:=auth.AuthenticateRequest(req)
                if err!=nil || !ok{
                    if err!=nil{
                        glog.Errorf("Unable to authenticate the request due to an error: %v", err)
                    }
                    failed.ServeHTTP(w,req)
                    return
                }
                //认证成功后，删除请求中的"Authorization"字段。
                req.Header.Del("Authorization")
                if ctx,ok:=mapper.Get(req);ok{
                    mapper.Update(req, genericapirequest.WithUser(ctx,user))  //更新上下文中的user。
                }
                authenticatedUserCounter.WithLabelValues(compressUsername(user.GetName())).Inc()
                handler.ServeHTTP(w,req)
            }),
            mapper,
        )
    }

    func WithRequestContext(handler http.Handler, mapper RequestContextMapper) http.Handler{
        rcMap, ok:=mapper.(*requestContextMap)
        if !ok{
            glog.Fatal("Unknown RequestContextMapper implementation.")
        }
        return http.HandlerFunc(func(w http.ResponseWriter, req *http.Request){
            if rcMap.init(req, NewContext()){
                defer rcMap.remove(req)
            }
            handler.ServeHTTP(w,req)
        })
    }

（e）WithCORS

含义：

    跨域资源共享的过滤器。请求头有"Orgin"字段，才处理跨域资源共享过滤器。

定义：

    func WithCORS(handler http.Handler, allowedOriginPatterns []string, allowedMethods []string, allowedHeaders []string, exposedHeaders []string, allowCredentials []string)http.Handler{
        if len(allowedOriginPatterns)==0{
            return handler
        }
        allowedOriginPatternsREs := allowedOriginRegexps(allowedOriginPatterns)
        return http.HandlerFunc(func(w http.ResponseWriter, req *http.Request){
            origin:=req.Header.Get("Origin")
            if origin!=""{
                allowed:=false
                //判断Orgin字段的值是否在跨域资源共享列表中，即传入的参数，CorsAllowedOriginList。在列表中才执行过滤行为，否则直接退出，执行后面的过滤器。
                for _,re:=range allowedOriginPatternREs{
                    if allowed=re.MatchString(origin);allowed{
                        break
                    }
                }
                if allowed{
                    //设置请求头字段："Access-Control-Allow-Origin"=orgin值。
                    w.Header().Set("Access-Control-Allow-Origin", origin)
                    if allowedMethods==nil{
                        allowedMethods=[]string{"POST", "GET", "OPTIONS", "PUT", "DELETE", "PATCH"}  //设置http methods：allowedMethods
                    }
                    //设置请求头
                    if allowedHeaders==nil{
                        allowedHeaders=[]string{"Content-Type","Content-Length","Accept-Encoding","X-CSRF-Token","Authorization","X-Requested-With","If-Modified-Since"}
                    }
                    if exposedHeaders==nil{
                        exposedHeaders=[]string{"Date"}
                    }
                    w.Header().Set("Access-Control-Allow-Methods", strings.Join(allowedMethods, ", "))    //设置"Access-Control-Allow-Methods"字段
                    w.Header().Set("Access-Control-Allow-Headers",strings.Join(allowedHeaders,", "))      //设置"Access-Control-Allow-Headers"字段
                    w.Header().Set("Access-Control-Expose-Headers", strings.Join(exposeHeaders, ", "))    //设置"Access-Control-Expose-Headers"字段
                    w.Header().Set("Access-Control-Allow-Credentials", allowCredentials)                  //设置"Access-Control-Allow-Credentials"字段
                    if req.Method=="OPTIONS"{
                        w.WriteHeader(http.StatusNoContent)
                        return
                    }
                }
            }
            handler.ServeHTTP(w,req)
        })
    }

（f）WithPanicRecovery

含义：

    recovery和日志panic过滤器。

定义：

    func WithPanicRecovery(handler http.Handler) http.Handler{
        return http.HandlerFunc(func(w http.ResponseWriter, req *http.Request){
            defer runtime.HandlerCrash(func(err interface{}){
                http.Error(w,"This request caused apiserver to panic. Look in log for details", http.StatusInternalServerError)
                glog.Errorf("APIServer panic'd on %v %v: %v\n%s\n", req.Method, req.RequestURI, err, debug.Stack())
            })
            logger:=httplog.NewLogged(req, &w)
            defer logger.Log()
            handler.ServeHTTP(w,req)
        })
    }

（g）WithTimeoutForNonLongRunningRequests

含义：

    非长时运行请求的超时过滤器。

定义：

    func WithTimeoutForNonLongRunningRequests(handler http.Handler, requestContextMapper apirequest.RequestContextMapper, longRunning apirequest.LongRunningRequestCheck)http.Handler{
        if longRunning==nil{
            return handler
        }
        timeoutFunc:=func(req *http.Request)(<-chan time.Time, *apierrors.StatusError){
            ctx,ok:=requestContextMapper.Get(req)
            if !ok{
                return time.After(globalTimeout), apierrors.NewInternalError(fmt.Errorf("no context found for request during timeout"))
            }
            requestInfo,ok:=apirequest.RequestInfoFrom(ctx)
            if !ok{
                return time.After(globalTimeout), apierrors.NewInternalError(fmt.Errorf("no request info found for request during timeout"))
            }

            //执行长时运行函数。如果是长时运行请求，直接退出。
            if longRunning(req, requestInfo){
                return nil,nil
            }
            return time.After(globalTimeout), apierrors.NewServerTimeout(schema.GroupResource{Group: requestInfo.APIGroup, Resource: requestInfo.Resource}, requestInfo.Verb,0)
        }
        return WithTimeout(handler,timeoutFunc)
    }

    func WithTimeout(h http.Handler, timeoutFunc func(*http.Request)(timeout <-chan time.Time, err *apierrors.StatusError)) http.Handler{
        return &timeoutHandler{h, timeoutFunc}
    }

    type timeoutHandler struct{
        handler http.Handler
        timeout func(*http.Request)(<-chan time.Time, *apierrors.StatusError)  //超时时间由超时函数传入。
    }

    func (t *timeoutHandler)ServeHTTP(w http.ResponseWriter, r *http.Request){
        after,err:=t.timeout(r)
        if after==nil{
            t.handler.ServeHTTP(w,r)
            return
        }
        done:=make(chan struct{})
        tw:=newTimeoutWriter(w)
        go func(){
            t.handler.ServeHTTP(tw,r)
            close(done)
        }()
        select{
        case <-done:
            return
        case <-after:
            tw.timeout(err)
        }
    }

（h）WithMaxInFlightLimit

含义：

    限流过滤器。

定义：

    func WithMaxInFlightLimit(handler http.Handler, nonMutatingLimit int, mutatingLimit int, requestContextMapper genericapirequest.RequestContextMapper, longRunningRequestsCheck apirequest.LongRunningRquestCheck,)http.Handler{
        if nonMutatingLimit==0 && mutatingLimit==0{
            return handler
        }
        var nonMutatingChan chan bool
        var mutatingChan chan bool
        if nonMutatingLimit != 0{
            nonMutatingChan=make(chan bool, nonMutatingLimit)
        }
        if mutatingLimit!=0{
            mutatingChan=make(chan bool,mutatingLimit)
        }
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request){
            ctx,ok:=requestContextMapper.Get(r)
            if !ok{
                handleError(w,r,fmt.Errorf("no context found for request, handler chain must be wrong"))
                return
            }
            requestInfo,ok:=apirequest.RequestInfoFrom(ctx)
            if !ok{
                handleError(w,r,fmt.Errorf("no RequestInfo found in context, handler chain must be wrong"))
                return
            }

            //跳过长时运行请求检查
            if longRunningRequestCheck!=nil && longRunningRequestCheck(r, requestInfo){
                handler.ServeHTTP(w,r)
                return
            }
            var c chan bool
            if !nonMutatingRequestVerbs.Has(requestInfo.Verb){
                c=mutatingChan
            }else{
                c=nonMutatingChan
            }
            if c==nil{
                handler.ServeHTTP(w,r)
            }else{
                select{
                case c<-true:
                    defer func(){<-c}()
                    handler.ServeHTTP(w,r)
                default:
                    tooManyRequests(r,w)
                }
            }
        })
    }

    func tooManyRequests(req *http.Request, w http.ResponseWriter){
        defer httplog.NewLogged(req, &w).Log()
        w.Header().Set("Retry-After", retryAfter)   //1
        http.Error(w,"Too many requests, please try again later.", errors.StatusTooManyRequests)
    }

（i）WithRequestInfo

含义：

    处理请求信息的过滤器。

定义：

    func WithRequestInfo(handler http.Handler, resolver *request.RequestInfoFactory, requestContextMapper request.RequestContextMapper)http.Handler{
        return http.HandlerFunc(func(w http.ResponseWriter, req *http.Request){
            ctx,ok:=requestContextMapper.Get(req)
            if !ok{
                responsewriters.InternalError(w,req, errors.New("no context found for request"))
                return
            }

            //创建RequestInfo对象，包含：APIPrefix、APIGroup、IsResourceRequest、APIVersion、Verb、Namespace、Subresource、Name
            info, err:=resolver.NewRequestInfo(req)
            if err!=nil{
                responsewriters.InternalError(w,req,fmt.Errorf("failed to create RequestInfo: %v",err))
                return
            }
            requestContextMapper.Update(req, request.WithRequestInfo(ctx,Info))  //使用Info更新上下文，再更新请求。
            handler.ServeHTTP(w,req)
        })
    }

    func NewRequestInfoResolver(c *Config)*apirequest.RequestInfoFactory{
        apiPrefixes:=sets.NewString(strings.Trim(APIGroupPrefix, "/"))
        legacyAPIPrefixes:=sets.String{}
        for legacyAPIPrefix:=range c.LegacyAPIGroupPrefixes{
            apiPrefixes.Insert(strings.Trim(legacyAPIPrefix, "/"))
            legacyAPIPrefixes.Insert(strings.Trim(legacyAPIPrefix, "/"))
        }
        return &apirequest.ReqeustInfoFactory{
            APIPrefixes: apiPrefixes,                            //api, apis
            GrouplessAPIPrefixes: legacyAPIPrefixes,             //api
        }
    }

（j）WithRequestContext

含义：

    在执行过滤器链之前，判断请求是否有context，如果有context，则直接返回，执行后续过滤器；否则给请求创建一个新的context，并在过滤器链执行完后再移除该context。

定义：

    func WithRequestContext(handler http.Handler, mapper RequestContextMapper) http.Handler{
        rcMap, ok:=mapper.(*requestContextMap)
        if !ok{
            glog.Fatal("Unknown RequestContextMapper implementation.")
        }
        return http.HandlerFunc(func(w http.ResponseWriter, req *http.Request){
            if rcMap.init(req, NewContext()){         //创建context
                defer rcMap.remove(req)               //移除context
            }
            handler.ServeHTTP(w,req)
        })
    }

（4）healthz.HealthzChecker

含义：

    健康检查，最简单的健康检查ping。

路径：

    k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/server/healthz/healthz.go

实参：

    healthz.PingHealthz

定义：

    type HealthzChecker interface{
        Name() string
        Check(req *http.Request) error
    }
    var PingHealthz HealthzChecker = ping{}
    //ping实现了
    type ping struct{}
    func (ping) Name() string{
        return "ping"
    }
    func (ping)Check(_ *http.Request)error{
        return nil
    }

（5）genericfilters.BasicLongRunningRequestCheck(sets.NewString("watch"), sets.NewString())

含义：

    检查watch资源。

路径：

    k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/server/filters/longrunning.go

参数："watch"，""。watch是long running资源，watch没有子资源。

定义：

    func BasicLongRunningRequestCheck(longRunningVerbs, longRunningSubresources sets.String) apirequest.LongRunningRequestCheck {
        return func(r *http.Request, requestInfo *apirequest.RequestInfo) bool {
            if longRunningVerbs.Has(requestInfo.Verb){  //请求verb是否有"watch"
                return true
            }
            if requestInfo.IsResourceRequest && longRunningSubresources.Has(requestInfo.Subresource){  //请求是否有子资源。
                return true
            }
            return false
        }
    }

### 1.4.2 s.GenericServerRunOptions.ApplyTo
含义：

    将GenericServerRunOptions中的参数赋给genericConfig对象。

路径：

    k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/server/options/server_run_options.go

定义：

    func (s *ServerRunOptions) ApplyTo(c *server.Config) error{
        c.CorsAllowedOriginList         = s.CorsAllowedOriginList
        c.ExternalAddress               = s.ExternalHost
        c.MaxRequestsInFlight           = s.MaxRequestsInFlight
        c.MaxMutatingRequestsInFlight   = s.MaxMutatingRequestsInFlight
        c.MinRequestTimeout             = s.MinRequestTimeout
        c.PublicAddress                 = s.AdvertiseAddress                      
        return nil
    }

### 1.4.3 s.InsecureServing.ApplyTo
含义：

    配置Config的非安全参数LoopbackClientConfig，并返回。kubeserver.InsecureServingInfo结构体对象。

路径：

    k8s.io/kubernetes/pkg/kubeapiserver/options/serving.go

定义：

    func (s *InsecureServingOptions)ApplyTo(c *server.Config) (*kubeserver.InsecureServingInfo, error){
        if s.BindPort <=0 {
            return nil,nil
        }
        ret:=&kubeserver.InsecureServingInfo{
            BindAddress: net.JoinHostPort(s.BindAddress.String(), strconv.Itoa(s.BindPort)),   //BindAddress=ip:port
        }
        var err error
        privilegedLoopbackToken := uuid.NewRandom().String()

        //设置server.Config.LoopbackClientConfig
        if c.LoopbackClientConfig, err=ret.NewLoopbackClientConfig(privilegedLoopbackToken); err!=nil {
            return nil,err
        }
        return ret,nil  //返回InsecureServingInfo
    }

### 1.4.4 s.SecureServing.ApplyTo
含义：

    设置安全参数。

路径：

    k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/server/options/serving.go

定义：

    func (s *SecureServingOptions) ApplyTo(c *server.Config) error{
        if s.BindPort<=0{
            return nil
        }

        //配置Config中的安全参数secureServingInfo。secureServingInfo包含：BindAddress=安全BindAddress:安全BindPort，Cert=证书CertFile和KeyFile组成tlsCert，可能存在CACert=CACertFile，SNICerts=组合NamedTLSCert和SNICertKeys证书。另外，Config中的字段ReadWritePort=安全BindPort
        if err:=s.applyServingInfoTo(c); err!=nil{
            return err
        }

        //本地回环地址访问apiserver的安全端口，需要生成自签名证书和秘钥（存储在apiserver中，这里是单向tls，即client请求server的安全端口）。本地回环地址访问的host域名是"apiserver-loopback-client"，该域名及对应的自签名证书和秘钥最终存入secureServingInfo的SNICerts中。即本地回环地址访问apiserver的host（"apiserver-loopback-client"），apiserver是通过SNICerts找到host对应的证书和秘钥返回给本地客户端。
        //const LoopbackClientServerNameOverride = "apiserver-loopback-client"
        certPem, keyPem, err := certutil.GenerateSelfSignedCertKey(server.LoopbackClientServerNameOverride, nil, nil)
        if err!=nil{
            return fmt.Errorf("failed to generate self-signed certificate for loopback connection: %v", err)
        }
        tlsCert,err:=tls.X509KeyPair(certPem, keyPem)
        if err!=nil{
            return fmt.Errorf("failed to generate self-signed certificate for loopback connection: %v", err)
        }

        //创建本地回环地址访问apiserver安全端口的参数secureLoopbackClientConfig，该参数是要赋给Config的LoopbackClientConfig。本地回环地址访问安全端口比访问非安全端口多了token和证书参数，具体包括：QPS=50，Burst=100，Host=https://host:port，BearerToken=uuid随机产生的token，TLSClientConfig={ServerName="apiserver-loopback-client"，CADATA=前面生成的自签名证书certPem}。TLSClientConfig是为了让本地客户端比较apiserver发来的证书是否正确。
        secureLoopbackClientConfig,err:=c.SecureServingInfo.NewLoopbackClientConfig(uuid.NewRandom().String(), certPem)
        switch{
        case err!=nil && c.LoopbackClientConfig==nil:
            return err
        case err!=nil && c.LoopbackClientConfig!=nil:
        default:
            c.LoopbackClientConfig = secureLoopbackClientConfig                                                 //设置本地回环地址访问apiserver安全端口。
            c.SecureServingInfo.SNICerts[server.LoopbackClientServerNameOverride] = &tlsCert   //SNI域名添加回环地址证书配置。
        }

        //创建共享的informers。
        clientset,err:=kubernetes.NewForConfig(c.LoopbackClientConfig)
        if err!=nil{
            return err
        }
        c.SharedInformerFactory = informers.NewSharedInformerFactory(clientset, c.LoopbackClientConfig.Timeout)
        return nil
    }

下面详细介绍一下创建共享informers的方法：

（1）NewForConfig

含义：

    创建客户端集合。

路径：

    k8s.io/kubernetes/vendor/k8s.io/client-go/kubernetes/clientset.go

实参：

    c.LoopbackClientConfig

定义：

    func NewForConfig(c *rest.Config) (*Clientset, error){
        configShallowCopy := *c

        //设置rest.Config.RateLimiter（即实参c.LoopbackClientConfig的RateLimiter），RateLimiter会限制客户端到apiserver的连接数。
        if configShallowCopy.RateLimiter==nil && configShallowCopy.QPS>0{
            configShallowCopy.RateLimiter = flowcontrol.NewTokenBucketRateLimiter(configShallowCopy.QPS, configShallowCopy.Burst)
        }

        //下面是创建各种group+version的客户端。
        var cs Clientset
        var err error
        cs.AdmissionregistrationV1alpha1Client, err = admissionregistrationv1alpha1.NewForConfig(&configShallowCopy)
        if err!=nil{
            return nil,err
        }
        cs.CoreV1Client,err=corev1.NewForConfig(&configShallowCopy)
        if err!=nil{
            return nil,err
        }
        cs.AppsV1beta1Client,err=appsv1beta1.NewForConfig(&configShallowCopy)
        if err!=nil{
            return nil,err
        }
        cs.AuthenticationV1Client,err=authenticationv1.NewForConfig(&configShallowCopy)
        if err!=nil{
            return nil,err
        }
        cs.AuthenticationV1beta1Client,err=authenticationv1beta1.NewForConfig(&configShallowCopy)
        if err!=nil{
            return nil,err
        }
        cs.AuthorizationV1Client,err=authorizationv1.NewForConfig(&configShallowCopy)
        if err!=nil{
            return nil,err
        }
        cs.AuthorizationV1beta1Client,err=authorizationv1beta1.NewForConfig(&configShallowCopy)
        if err!=nil{
            return nil,err
        }
        cs.AutoscalingV1Client,err=autoscalingv1.NewForConfig(&configShallowCopy)
        if err!=nil{
            return nil,err
        }
        cs.AutoscalingV2alpha1Client,err=autoscalingv2alpha1.NewForConfig(&configShallowCopy)
        if err!=nil{
            return nil,err
        }
        cs.BatchV1Client,err=batchv1.NewForConfig(&configShallowCopy)
        if err!=nil{
            return nil,err
        }
        cs.BatchV2alpha1Client,err=batchv2alpha1.NewForConfig(&configShallowCopy)
        if err!=nil{
            return nil,err
        }
        cs.CertificatesV1beta1Client,err=certificatesv1beta1.NewForConfig(&configShallowCopy)
        if err!=nil{
            return nil,err
        }
        cs.ExtensionsV1beta1Client,err=extensionsv1beta1.NewForConfig(&configShallowCopy)
        if err!=nil{
            return nil,err
        }
        cs.NetworkingV1Client,err=networkingv1.NewForConfig(&configShallowCopy)
        if err!=nil{
            return nil,err
        }
        cs.PolicyV1beta1Client,err=policyv1beta1.NewForConfig(&configShallowCopy)
        if err!=nil{
            return nil,err
        }
        cs.RbacV1beta1Client,err=rbacv1beta1.NewForConfig(&configShallowCopy)
        if err!=nil{
            return nil,err
        }
        cs.RbacV1alpha1Client,err=rbacv1alpha1.NewForConfig(&configShallowCopy)
        if err!=nil{
            return nil,err
        }
        cs.SettingsV1apha1Client,err=settingsv1alpha1.NewForConfig(&configShallowCopy)
        if err!=nil{
            return nil,err
        }
        cs.StorageV1beta1Client,err=storagev1beta1.NewForConfig(&configShallowCopy)
        if err!=nil{
            return nil,err
        }
        cs.StorageV1Client,err=storagev1.NewForConfig(&configShallowCopy)
        if err!=nil{
            return nil,err
        }
        cs.DiscoveryClient,err=discovery.NewDiscoveryClientForConfig(&configShallowCopy)
        if err!=nil{
            glog.Errorf("failed to create the DiscoveryClient: %v", err)
            return nil,err
        }
        return &cs,nil
    }
    Clientset结构体定义：
    type Clientset struct{
        *discovery.DiscoveryClient
        *admissionregistrationv1alpha1.AdmissionregistrationV1alpha1Client
        *corev1.CoreV1Client
        *appsv1beta1.AppsV1beta1Client
        *authenticationv1.AuthenticationV1Client
        *authenticationv1beta1.AuthenticationV1beta1Client
        *authorizationv1.AuthorizationV1Client
        *authorizationv1beta1.AuthorizationV1beta1Client
        *autoscalingv1.AutoscalingV1Client
        *autoscalingv2alhpa1.AutoscalingV2alpha1Client
        *batchv1.BatchV1Client
        *batchv2alpha1.BatchV2alpha1Client
        *certificatesv1beta1.CertificatesV1beta1Client
        *extensionsv1beta1.ExtensionsV1beta1Client
        *networkingv1.NetworkingV1Client
        *policyv1beta1.NetworkingV1Client
        *rbacv1beta1.RbacV1beta1Client
        *rbacv1alpha1.RbacV1alpha1Client
        *settingsv1alpha1.SettingsV1alpha1Client
        *storagev1beta1.StorageV1beta1Client
        *storagev1.StorageV1Client
    }
    
以CoreV1Client和DiscoveryClient为例，介绍创建带版本的客户端的方法。
CoreV1Client：
含义： 

    CoreV1Client包含rest.Interface接口，所以它要实现该接口的所有方法，即包含REST API的操作方法。换句话说，创建的客户端对象具有访问kubernetes REST API的各种方法。

路径：

    k8s.io/kubernetes/vendor/k8s.io/client-go/kubernetes/typed/core/v1/core_client.go

定义：

    func NewForConfig(c *restclient.Config)(*CoreV1Client, error){
        config := *c
        //设置本地回环地址客户端默认配置信息，即Config.LoopbackClientConfig，或restclient.Config。包括：GroupVersion={"","v1"}，APIPath="/api"，NegotiatedSerializer=serializer.DirectCodecFactory{CodecFactory: scheme.Codecs}，UserAgent=command/version (os/arch) kubernetes/commit
        if err:=setConfigDefaults(&config);err!=nil{
            return nil,err
        }

        //创建RESTClient。满足client Config对象上的请求属性。
        client,err:=rest.RESTClientFor(&config)
        if err!=nil{
            return nil,err
        }
        return &CoreV1Client{ client}, nil
    }

（a）设置客户端默认配置信息。

路径：

    k8s.io/kubernetes/vendor/k8s.io/client-go/kubernetes/typed/core/v1/core_client.go

定义：

    func setConfigDefault(config *rest.Config) error{
        gv                                      :=v1.SchemeGroupVersion
        config.GroupVersion           = &gv
        config.APIPath                    = "/api"
        config.NegotiatedSerializer = serializer.DirectCodeFactory{CodeFactory: scheme.Codecs}
        if config.UserAgent == ""{
            config.UserAgent           =rest.DefaultKubernetesUserAgent()
        }
        return nil
    }

（b）创建RESTClient。

路径：

    k8s.io/kubernetes/vendor/k8s.io/client-go/rest/config.go

定义：

    func RESTClientFor(config *Config)(*RESTClient, error){
        if config.GroupVersion==nil{
            return nil,fmt.Errorf("GroupVersion is required when initializing a RESTClient")
        }
        if config.NegotiatedSerialize==nil{
            return nil,fmt.Errorf("NegotiatedSerializer is required when initializing a RESTClient")
        }
        qps:=config.QPS
        if config.QPS==0.0{
            qps=DefaultQPS      //默认QPS=5.0
        }
        burst:=config.Burst
        if config.Burst==0{
            burst=DefaultBurst  //默认Burst=10
        }

        //baseURL=https://127.0.0.1:6443,    versionedAPIPath=/api/v1
        baseURL,versionedAPIPath,err:=defaultServerUrlFor(config)   
        if err!=nil{
            return nil,err
        }
        transport,err:=TransportFor(config)
        if err!=nil{
            return nil,err
        }
        var httpClient *http.Client
        if transport!=http.DefaultTransport{
            httpClient=&http.Client{Transport:transport}
            if config.Timeout>0{                                                //config.Timeout=0s
                httpClient.Timeout = config.Timeout
            }
        }

        //baseURL=https://127.0.0.1:6443,  versionedAPIPath=/api/v1,    config.ConfigConfig.GroupVersion.Group="", config.ConfigConfig.GroupVersion.version="v1", qps=50,burst=100
        return NewRESTClient(baseURL, versionedAPIPath, config.ContentConfig, qps, burst, config.RateLimiter, httpClient)
    }

NewRESTClient在一组资源路径上强加了kubernetes API的约定。创建RESTClient对象，RESTClient实现rest.Interface接口。

    //k8s.io/kubernetes/vendor/k8s.io/client-go/rest/client.go
    func NewRESTClient(baseURL *url.URL, versionedAPIPath string, config ContentConfig, maxQPS float32, maxBurst int, rateLimiter flowcontrol.RateLimiter, client *http.Client)(*RESTClient, error){
        base := *baseURL
        if !strings.HasSuffix(base.Path, "/"){
            base.Path += "/"
        }
        base.RawQuery = ""
        base.Fragment = ""
        if config.GroupVersion == nil{
            config.GroupVersion = &schema.GroupVersion{}
        }
        if len(config.ContentType)==0{
            config.ContentType = "application/json"                    //client和apiserver之间数据传输类型，此处是json，但是在后面被覆盖为protobuf，后面的BuildGenericConfig函数中设置：genericConfig.LoopbackClientConfig.ContentConfig.ContentType="application/vnd.kubernetes.protobuf"。
        }
        serializers,err:=createSerializers(config)
        if err!=nil{
            return nil,err
        }
        var throttle flowcontrol.RateLimiter
        if maxQPS>0 && rateLimiter==nil{
            throttle = flowcontrol.NewTokenBucketRateLimiter(maxQPS, maxBurst)
        }else if rateLimiter!=nil{
            throttle = rateLimiter
        }
        return &RESTClient{
            base: &base,                                                              //"https://127.0.0.1:6443/"
            versionedAPIPath: versionedAPIPath,                        //"/api/v1"
            contentConfig: config,                                              
            serializers: *serializers,
            createBackoffMgr: readExpBackoffConfig,
            Throttle: throttle,                                                      //rateLimiter
            Client: client,                                                            //http.Client
        },nil
    }

rest.Interface接口定义了REST API的操作方法。RESTClient必须实现该接口的方法。

    //k8s.io/kubernetes/vendor/k8s.io/client-go/rest/client.go
    type Interface interface{                               //kubernetes REST API的操作集接口。
        GetRateLimiter() flowcontrol.RateLimiter  //限流
        Verb(verb string) *Request                       //操作
        Post()*Request                                         //创建
        Put()*Request                                          //更新
        Patch(pt types.PatchType)*Request         //部分更新
        Get() *Request                                         //查询
        Delete()*Reqeust                                      //删除
        APIVersion() schema.GroupVersion          //查看API版本，group和version信息。
    }

RESTClient结构体定义，RESTClient需要实现rest.Interface接口。

    //k8s.io/kubernetes/vendor/k8s.io/client-go/rest/client.go
    type RESTClient struct{
        base                       *url.URL                        //root URL，此处取值：https://127.0.0.1:6443/
        versionedAPIPath   string                            //base URL，例如："/api/v1"、"/apps/extensions/v1"。
        contentConfig        ContentConfig
        serializers               Serializers
        createBackoffMgr  func()BackoffManager
        Throttle                  flowcontrol.RateLimiter //rateLimiter
        Client                     *http.Client
    }
    func (c *RESTClient)GetRateLimiter() flowcontrol.RateLimiter{
        if c==nil{
            return nil
        }
        return c.Throttle
    }

Verb创建一个REST请求。使用示例：

    //c,err:=NewRESTClient(...)
    //if err!=nil{...}
    //resp,err:=c.Verb("GET").Path("pods").SelectorParam("labels","area=staging").Timeout(10*time.Second).Do()
    //if err!=nil{...}
    //list,ok:=resp.(*api.PodList)
    func (c *RESTClient)Verb(verb string)*Request{
        backoff := c.createBackoffMgr()
        if c.Client==nil{
            return NewRequest(nil, verb, c.base, c.versionedAPIPath, c.contentConfig, c.serializers, backoff, c.Throttle)
        }
        return NewRequest(c.Client, verb, c.base, c.versionedAPIPath, c.contentConfig, c.serializers, backoff, c.Throttle)
    }

Verb方法创建一个rest 请求对象：
    
    路径：k8s.io/kubernetes/vendor/k8s.io/client-go/rest/request.go
    func NewRequest(client HTTPClient, verb string, baseURL *url.URL, versionedAPIPath string, content ContentConfig, serializers Serializers, backoff BackoffManager, throttle flowcontrol.RateLimiter) *Request{
        if backoff==nil{
            glog.V(2).Infof("Not implementing request backoff strategy.")
            backoff = &NoBackoff{}
        }
        pathPrefix:="/"
        if baseURL!=nil{
            pathPrefix=path.Join(pathPrefix,baseURL.Path)
        }
        r:=&Request{
            client: client,
            verb: verb,
            baseURL: baseURL,
            pathPrefix: path.Join(pathPrefix, versionedAPIPath),
            content: content,
            serializers: serializers,
            backoffMgr: backoff,
            throttle: throttle,
        }
        switch{
        case len(content.AcceptContentTypes)>0:
            r.SetHeader("Accept", content.AcceptContentTypes)
        case len(content.ContentType)>0:
            r.SetHeader("Accept",content.ContentType+", */*")
        }
        return r
    }
    
Request的主要方法包括：

    //设置资源路径，<resource>/[ns/<namespace>/]<name>
    func (r *Request)Resource(resource string)*Request{
        if r.err!=nil{
            return r
        }
        if len(r.resource)!=0{
            r.err=fmt.Errorf("resource already set to %q, cannot change to %q", r.resource, resource)
            return r
        }
        if msgs:=IsValidPathSegmentName(resource); len(msgs)!=0{
            r.err=fmt.Errorf("invalid resource %q: %v", resource, msgs)
            return r
        }
        r.resource=resource
        return r
    }

    //设置子资源路径
    func (r *Request)SubResource(subresources ...string)*Request{
        if r.err!=nil{
            return r
        }
        subresource:=path.Join(subresources...)
        if len(r.subresource)!=0{
            r.err=fmt.Errorf("subresource already set to %q, cannot change to %q", r.resource, subresource)
            return r
        }
        for _,s:=range subresources{
            if msgs:=IsValidPathSegmentName(s); len(msgs)!=0{
                r.err=fmt.Errorf("invalid subresource %q: %v", s, msgs)
                return r
            }
        }
        r.subresource=subresource
        return r
    }

    //设置资源名，<resource>/[ns/<namespace>/]<name>
    func (r *Request)Name(resourceName string)*Request{
        if r.err!=nil{
            return r
        }
        if len(resourceName)==0{
            r.err=fmt.Errorf("resource name may not be empty")
            return r
        }
        if len(r.resourceName)!=0{
            r.err=fmt.Errorf("resource name already set to %q, cannot change to %q", r.resourceName, resourceName)
            return r
        }
        if msgs:=IsValidPathSegmentName(resourceName); len(msgs)!=0{
            r.err=fmt.Errorf("invalid resource name %q: %v", resourceName, msgs)
            return r
        }
        r.resourceName=resourceName
        return r
    }

    //设置资源namespace，<resource>/[ns/<namespace>/]<name>
    func (r *Request)Namespace(namespace string)*Request{
        if r.err!=nil{
            return r
        }
        if r.namespaceSet{
            r.err=fmt.Errorf("namespace already set to %q, cannot change to %q", r.namespace, namespace)
            return r
        }
        if msgs:=IsValidPathSegmentName(namespace);len(msgs)!=0{
            r.err=fmt.Errorf("invalid namespace %q: %v", namespace, msgs)
            return r
        }
        r.namespaceSet = true
        r.namespace = namespace
        return r
    }

    //设置路径
    func (r *Request)AbsPath(segments ...string) *Request{
        if r.err!=nil{
            return r
        }
        r.pathPrefix=path.Join(r.baseURL.Path, path.Join(segments...))
        if len(segments)==1 && (len(r.baseURL.Path)>1|| len(segments[0])>1) && strings.HasSuffix(segments[0], "/"){
            r.pathPrefix+="/"
        }
        return r
    }

    //设置标签选择器，作为URL的查询参数。
    func (r *Request)FieldsSelectorParam(s fields.Selector) *Request{
        if r.err!=nil{
            return r
        }
        if s==nil{
            return r
        }
        if s.Empty(){
            return r
        }
        s2,err:=s.Transform(func(field, value string)(newField, newValue string, err error){
            return fieldMappings.filterField(r.content.GroupVersion, r.resource, field, value)
        })
        if err!=nil{
            r.err=err
            return r
        }
        //metav1.FieldSelectorQueryParam返回"fieldSelector"字符串。
        return r.setParam(metav1.FieldSelectorQueryParam(r.content.GroupVersion.String()),s2.String())
    }

    //设置标签选择器，
    func (r *Request)LabelsSelectorParam(s labels.Selector)*Request{
        if r.err!=nil{
            return r
        }
        if s==nil{
            return r
        }
        if s.Empty(){
            return r
        }
        //metav1.LabelSelectorQueryParam返回"labelSelector"。
        return r.setParam(metav1.LabelSelectorQueryParam(r.content.GroupVersion.String()), s.String())
    }

    //设置请求头
    func (r *Request) SetHeader(key,value string) *Request{
        if r.headers==nil{
            r.headers=http.Headler{}
        }
        r.headers.Set(key,value)
        return r
    }

    //设置超时
    func (r *Request)Timeout(d time.Duration) *Request{
        if r.err!=nil{
            return r
        }
        r.timeout=d
        return r
    }

    func (c *RESTClient)Post()*Request{
        return c.Verb("POST")
    }
    func (c *RESTClient)Put()*Request{
        return c.Verb("PUT")
    }
    func (c *RESTClient)Patch(pt types.PatchType)*Request{
        return c.Verb("PATCH").SetHeader("Content-Type", string(pt))
    }
    func (c *RESTClient)Get()*Request{
        return c.Verb("GET")
    }
    func (c *RESTClient)Delete()*Request{
        return c.Verb("DELETE")
    }
    func (c *RESTClient)APIVersion() schema.GroupVersion{
        return *c.contentConfig.GroupVersion
    }

DiscoveryClient：查询API server的资源。

路径：

    k8s.io/kubernetes/vendor/k8s.io/client-go/discovery/discovery_client.go

定义：

    func NewDiscoveryClientForConfig(c *restclient.Config)(*DiscoveryClient, error){
        config:=*c
        //设置config默认值，包括：APIPath=""，GroupVersion=nil，serializer=设置通用解码器。
        if err!=setDiscoveryDefaults(&config); err!=nil{
            return nil,err
        }
        client,err:=restclient.UnversionedRESTClientFor(&config)
        return &DiscoveryClient{restClient: client, LegacyPrefix: "/api"},err
    }
     
下面介绍一下serializer的通用解码器：

    //k8s.io/kubernetes/vendor/k8s.io/client-go/discovery/discovery_client.go
    func setDiscoveryDefaults(config *restclient.Config)error{
        config.APIPath=""
        config.GroupVersion=nil
        codec:=runtime.NoopEncoder{Decoder: scheme.Codecs.UniversalDecoder()}
        config.NegotiatedSerializer=serializer.NegotiatedSerializerWrapper(runtime.SerializerInfo{Serializer: codec})
        if len(config.UserAgent)==0{
            config.UserAgent=restclient.DefaultKubernetesUserAgent()
        }
        return nil
    }

    //k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/runtime/serializer/codec_factory.go
    func (f CodecFactory)UniversalDecoder(versions ...schema.GroupVersion)runtime.Decoder{
        var versioner runtime.GroupVersioner
        if len(versions)==0{
            versioner=runtime.InternalGroupVersioner    //内部版本
        }else{
            versioner=schema.GroupVersions(versions)   //外部版本
        }
        return f.CodecForVersions(nil, f.universal, nil, versioner)
    }

UnversionedRESTClientFor：创建RESTCleint，支持无版本的客户端。
     
路径：

    k8s.io/kubernetes/vendor/k8s.io/client-go/rest/config.go

定义：

    func UnversionedRESTClientFor(config *Config)(*RESTClient,error){
        if config.NegotiatedSerializer==nil{
            return ni, fmt.Errorf("NegotiatedSerializer si required when initialing a RESTClient")
        }
        baseURL, versionedAPIPath,err:=defaultServerUrlFor(config)
        if err!=nil{
            return nil,err
        }
        transport,err:=TransportFor(config)
        if err!=nil{
            return nil,err
        }
        var httpClient *http.Client
        if transport!=http.DefaultTransport{
            httpCleint=&http.Client{Transport: transport}
            if config.Timeout>0{
                httpClient.Timeout=config.Timeout
            }
        }
        versionConfig:=config.ContentConfig
        if versionConfig.GroupVersion==nil{
            v:=metav1.SchemeGroupVersion
            versionConfig.GroupVersion=&v
        }
        //baseURL="https://127.0.0.1:6443"，versionedAPIPath="/"，versionConfig.GroupVersion.Group="meta.k8s.io"，versionConfig.GroupVersion.Version="v1"，QPS=50，Burst=100，httpClient.Timeout=0s
        return NewRESTClient(baseURL, versionedAPIPath, versionConfig, config.QPS, config.Burst, config.RateLimiter, httpClient)
    }

（2）informers.NewSharedInformerFactory

含义：

    创建共享信息仓库。

路径：

    k8s.io/kubernetes/vendor/k8s.io/client-go/informers/factory.go

定义：

    func NewSharedInformerFactory(client kubernetes.Interface, defaultResync time.Duration)SharedInformerFactory{
        return &sharedInformerFactory{
            client: client,                                             //RESTClient对象
            defaultResync: defaultResync,                  //0s
            informers: make(map[reflect.Type]cache.SharedIndexInformer),
            startedInformers: make(map[reflect.Type]bool),
        }
    }

    //k8s.io/kubernetes/vendor/k8s.io/client-go/tools/cache/shared_informer.go
    type SharedIndexInformer interface{
        SharedInformer
        AddIndexers(indexers Indexers)error
        GetIndexer()Indexer
    }
     
    //SharedInformer有一个共享数据缓存，可用于分布式系统中向多个监听器通知缓存的变化。通过AddEventHandler来注册监听器。SharedInformer比广播好的地方是，允许我们在多个监听器间共享一个缓存。一个监听器发生变化，更新缓存，其他监听器立即就得到响应。
    type SharedInformer interface{
        AddEventHandler(handler ResourceEventHandler)
        AddEventHandlerWithResyncPeriod(handler ResourceEventHandler, resyncPeriod time.Duration)
        GetStore() Store
        GetController() Controller
        Run(stopCh <-chan struct{})
        HasSynced() bool
        LastSyncResourceVersion() string
    }

### 1.4.5 s.Authentication.ApplyTo
含义：

    加载认证配置。包括ClientCert.ClientCA、RequestHeader.ClientCAFile、SupportBasicAuth。

路径：

    k8s.io/kubernetes/pkg/kubeapiserver/options/authentication.go

定义：

    func (o *BuiltInAuthenticationOptions)ApplyTo(c *genericapiserver.Config) error{
        if o==nil {
            return nil
        }
        var err error
        //客户端认证，加载客户端ca。
        if o.ClientCert!=nil{
            //把BuildInAuthenticationOptions.ClientCert.ClientCA或者flag参数"--client-ca-file"的值添加到SecureServingInfo.ClientCA中。默认没有该证书。
            c,err=c.ApplyClientCert(o.ClientCert.ClientCA)  //ClientCert--client-ca-file
            if err!=nil{
                return fmt.Errorf("unable to load client CA file: %v", err)
            }
        }
        //请求头认证，加载客户端ca。
        if o.RequestHeader!=nil{
            //把BuildInAuthenticationOptions.RequestHeader.ClientCAFile或者flag参数"--requestheader-client-ca-file"的值添加到SecureServingInfo.ClientCA中。默认没有该证书。
            c,err=c.ApplyClientCert(o.RequestHeader.ClientCAFile)
            if err!=nil{
                return fmt.Errorf("unable to load client CA file: %v", err)  
            }
        }

        //BuildInAuthenticationOptions.PasswordFile.BasicAuthFile或者flag参数"--basic-client-file"的值。默认没有BasicAuthFile，即SupportBasicAuth=false。
        c.SupportsBasicAuth = o.PasswordFile!=nil && len(o.PasswordFile.BasicAuthFile)>0
        return nil
    }

ApplyClientCert：
     
含义：

    添加客户端证书到SecureServingInfo.ClientCA证书池中。池的结构包含：证书slice（存储证书），根据域名查找的map，更加证书名称查找的map。

路径：

    k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/server/config.go

定义：

    func (c *Config)ApplyClientCert(clientCAFile string)(*Config, error){
        if c.SecureServingInfo !=nil{
            if len(clientCAFile)>0{
                clientCAs,err:=certutil.CertsFromFile(clientCAFile)
                if err!=nil{
                    return nil,fmt.Errorf("unable to load client CA file: %v", err)
                }
                if c.SecureServingInfo.ClientCA==nil{
                    c.SecureServingInfo.ClientCA = x509.NewCertPool()
                }
                for _,cert:=range clientCAs{
                    c.SecureServingInfo.ClientCA.AddCert(cert)  //把客户端证书添加到SecureServingInfo.ClientCA证书池中。
                }
            }
        }
        return c,nil
    }

### 1.4.6 s.Audit.ApplyTo
含义：

    加载审计配置。

路径：

    k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/server/options/audit.go

定义：

    func (o *AuditOptions)ApplyTo(c *server.Config) error{
        //加载审计通用参数，加载策略配置文件。
        if err:=o.applyTo(c);err!=nil{
            return err
        }

        //加载审计日志参数。
        if err:=o.LogOptions.applyTo(c);err!=nil{
            return err
        }
        //加载审计webhook参数。
        if err:=o.WebhookOptions.applyTo(c); err!=nil{
            return err
        }
        return nil
    }

（1）o.applyTo-通用审计配置

    func (o *AuditOptions)applyTo(c *server.Config)error{
        if o.PolicyFile==""{
            return nil
        }
        //必须打开AdvancedPolicy开关。
        if !advancedAuditingEnabled{
            return fmt.Errorf("feature '%s' must be enabled to set an audit policy", features.AdvancedAuditing)
        }
        //加载审计策略文件，默认是空。
        p,err:=policy.LoadPolicyFromFile(o.PolicyFile)
        if err!=nil{
            return fmt.Errorf("loading audit policy file: %v", err)
        }
        //创建审计检查器，审计检查器要实现Checker接口的Level(authorizer.Attributes) audit.Level方法，即检查请求的授权属性是否匹配审计规则，如果匹配则获得规则对应的审计级别；否则返回默认审计级别-"None"，表示不匹配审计规则。请求需要匹配的审计规则包括：Level、Users、UserGroups、Verbs、Resources、Namespaces、NonResourceURLs（"/metrics"、"/healthz"等）。
        c.AuditPolicyChecker = policy.NewChecker(p)
        return nil
    }

（2）o.LogOptions.applyTo-日志审计

路径：

    k8s.io/kubernetes/vendoer/k8s.io/apiserver/pkg/server/options/audit.go

定义：

    func (o *AuditLogOptions) applyTo(c *server.Config) error{
        if o.Path==""{
            return nil
        }
        var w io.Writer=os.Stdout
        if o.Path!="-"{
            w=&lumberjack.Logger{
                Filename: o.Path,
                MaxAge: o.MaxAge,
                MaxBackups: o.MaxBackups,
                MaxSize: o.MaxSize,
            }
        }
        c.LegacyAuditWriter=w                         //LegacyAuditWriter
        if advancedAuditingEnabled(){
            c.AuditBackend=appendBackend(c.AuditBackend, plugining.NewBackend(w)) //审计事件发送的地方
        }
        return nil
    }

（3）o.WebhookOptions.applyTo-审计webhook

路径：

    k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/server/options/audit.go

定义：

    function (o *AuditWebhookOptions) applyTo(c *server.Config)error{
        if o.ConfigFile==""{
            return nil
        }
        if !advancedAuditingEnabled(){
            return fmt.Errorf("feature '%s' must be enabled to set an audit webhook", features.AdvancedAuditing)
        }
        webhook,err:=pluginwebhook.NewBackend(o.ConfigFile, o.Mode)
        if err!=nil{
            return fmt.Errorf("initializing audit webhook: %v", err)
        }
        c.AuditBackend=appendBackend(c.AuditBackend, webhook)
    }

### 1.4.7 s.Features.ApplyTo
含义：

    加载Features参数。

路径：

    k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/server/options/feature.go

定义：

    func (o *FeatureOptions)ApplyTo (c *server.Config)error{
        c.EnableProfiling                  = o.EnableProfiling                     //默认true
        c.EnableContentionProfiling = o.EnableContentionProfiling    //默认false
        c.EnableSwaggerUI              = o.EnableSwaggerUI                 //默认false
        return nil
    }

### 1.4.8 OpenAPIConfig
含义：

    配置OpenAPIConfig参数。

（1）DefaultOpenAPIConfig-通用配置

含义： 

    OpenAPI的通用配置。关于OpenAPI spec的详细介绍请参见我的其他文章。

路径：

    k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/server/config.go

参数：

    generatedopenapi.GetOpenAPIDefinition
    api.Scheme

定义：

    func DefaultOpenAPIConfig(getDefinitions openapicommon.GetOpenAPIDefinitions, scheme *runtime.Scheme) *openapicommon.Config{
        //创建DefinitionNamer对象用于定制OpenAPI spec。
        defNamer := apiopenapi.NewDefinitionNamer(scheme)

        return &openapicommon.Config{
            ProtocolList: []string{"https"},
            IgnorePrefixes: []string{"/swaggerapi"},
            Info: &spec.Info{
                InfoProps: spec.InfoProps{
                    Title: "Generic API Server",
                },
            },
            DefaultResponse: &spec.Response{
                ResponseProps: spec.ResponseProps{
                    Description: "Default Response.",
                },
            },
            GetOperationIDAndTags: apiopenapi.GetOperationIDAndTags,
            GetDefinitionName: defNamer.GetDefinitionName,
            GetDefinitions: getDefinitions,
        }
    }

NewDefinitionNamer:

含义：

    创建定义的名称对象。表示type对应的gvk。

路径：

    k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/endpoints/openapi/openapi.go

定义：

    func NewDefinitionNamer(s *runtime.Scheme)DefinitionNamer{
        //DefinitionNamer结构体中有个map-DefinitionNamer表示type映射到gvk。
        ret:=DefinitionNamer{
            typeGroupVersionKinds: map[string]groupVersionKinds{},
        }
        //设置所有type和gvk到typeGroupVersionKinds中。
        for gvk,rtype:=range s.AllKnownTypes(){
            ret.typeGroupVersionKinds[typeName(rtype)]=append(ret.typeGroupVersionKinds[typeName(rtype)], gvkConvert(gvk))
        }
        for _,gvk:=range ret.typeGroupVersionKinds{
            sort.Sort(gvk)
        }
        return ret
    }

所有注册到typeGroupVersionKinds中的信息请看参考文献[[typeGroupVersionKinds注册的信息]](../../reference/k8s/typeGroupVersionKinds.md)

     
下面介绍参数：

（a） generatedopenapi.GetOpenAPIDefinition

含义：

    OpenAPI的定义，即OpenAPI的schema。OpenAPI的schema已经在其他章节介绍过了。

路径：

    k8s.io/kubernetes/pkg/generated/openapi/zz_generated.openapi.go

定义：

    func GetOpenAPIDefinition(ref openapi.ReferenceCallback) map[string]openapi.OpenAPIDefinition{
        return map[string]openapi.OpenAPIDefinition{
            "k8s.io/apimachinery/pkg/api/resource.Quantity": resource.Quantity{}.OpenAPIDefinition(),
            "k8s.io/apimachinery/pkg/api/resource.int64Amount": {
                Schema: spec.Schema{
                    SchemaProps: spec.SchemaProps{
                        Description: "...",
                        Properties: map[string]spec.Schema{
                            "value":{
                                SchemaProps: spec.SchemaProps{
                                    Type: []string{"integer"},
                                    Format: "int64",
                                },
                            },
                            "scale":{
                                SchemaProps: spec.SchemaProps{
                                    Type: []string{"integer"},
                                    Format: "int32",
                                },
                            },
                        },
                        Required: []string{"value", "scale"},
                    },
                },
                Dependencies: []string{},
            },
            "...":{},
        }
    }


（b）api.Scheme=runtime.NewScheme()前面已经介绍过了

（2）OpenAPIConfig.PostProcessSpec
     
含义：

    为了向后兼容，如果在Definitions的key中存入compatibilityMap的value值，则在Definitions中添加该value对应的key，并且该key的定义是引用value中的定义。举个例子：旧版本swagger的Definition中有一个key是"v1.Binding"，新版本改成了"k8s.io/kubernetes/pkg/api/v1/Binding"，此时需要把旧版本的定义修改为引用新版本的"k8s.io/kubernetes/pkg/api/v1/Binding"。

路径：

    k8s.io/kubernetes/cmd/kube-apiserver/app/server.go

定义：

    func postProcessOpenAPISpecForBackwardCompatibility(s *spec.Swagger)(*spec.Swagger, error){
        compatibilityMap:=map[string]string{
            "v1beta1.DeploymentStatus": "k8s.io/kubernetes/pkg/apis/extensions/v1beta1.DeploymentStatus",
            ...
        }
        for k,v:=range compatibilityMap{
            if _, found:=s.Definitions[v]; !found{
                continue
            }
            s.Definitions[k]=spec.Schema{
                SchemaProps:spec.SchemaProps{
                    Ref: spec.MustCreateRef("#/definitions/" + openapi.EscapeJsonPointer(v)),
                    Description: fmt.Sprintf("Deprecated. Please use %s instead.",v),
                },
            }
        }
        return s,nil
    }

（3）OpenAPIConfig.Info.Title="Kubernetes"。这个是Schema中的定义，在Schema一章中有详细说明。

（4）SwaggerConfig=genericapiserver.DefaultSwaggerConfig()

含义：

    默认swagger配置。

路径：

    k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/server/config.go

定义：

    func DefaultSwaggerConfig() *swagger.Config{
        return &swagger.Config{
            ApiPath: "/swaggerapi",
            SwaggerPath: "/swaggerui/",
            SwaggerFilePath": "/swagger-ui",
            SchemaFormatHandler: func(typeName string) string{
                switch typeName{
                case "metav1.Type", "*metav1.Time":
                    return "date-time"
                }
                return ""
            },
        }
    }

（5）filters.BasicLongRunningRequestCheck

含义：

    定义基本的长时运行函数。

定义：

    genericConfig.LongRunningFunc = filters.BasicLongRunningRequestCheck(
        sets.NewString("watch", "proxy"),
        sets.NewString("attach", "exex", "proxy", "log", "portforward"),
    )

（6）genericConfig.Version

含义：

    版本信息结构体，包含：Major、Minor、GitVersion、GitCommit、GitTreeState、BuildDate、GoVersion、Compiler、Platform（GOOS/GOARCH）。
    kubeVersion := version.Get()       
    genericConfig.Version = &kubeVersion

### 1.4.9 BuildStorageFactory
含义：

    创建storage factory。

路径：

    k8s.io/kubernetes/cmd/kube-apiserver/app/server.go

定义：
    func BuildStorageFactory(s *options.ServerRunOptions)(*serverstorage.DefaultStorageFactory, error){
        //设置etcd存储的group、version信息，map结构：{group : {group : version}}
        storageGroupsToEncodingVersion, err:= s.StorageSerialization.StorageGroupsToEncodingVersion()
        if err!=nil{
            return nil, fmt.Errorf("error generating storage version map: %s", err)
        }

        //创建storage factory
        storageFactory,err:=kubeapiserver.NewStorageFactory(
            s.Etcd.StorageConfig, s.Etcd.DefaultStorageMediaType, api.Codecs,
            serverstorage.NewDefaultResourceEncodingConfig(api.Registry), storageGroupsToEncodingVersion,
            []schema.GroupVersionResource{batch.Resource("cronjobs").WithVersion("v2alpha1")},
            master.DefaultAPIResourceConfigSource(), s.APIEnablement.RuntimeConfig)
        if err!=nil{
            return nil,fmt.Errorf("error in initializing storage factory: %s", err)
        }
        storageFactory.AddCohabitatingResources(extensions.Resource("deployments"), apps.Resource("deployments"))
        storageFactory.AddCohabitatingResources(extensions.Resource("networkpolicies"), networking.Resource("networkpolicies"))
        for _,override:=range s.Etcd.EtcdServersOverrides{
            tokens:=strings.Split(override, "#")
            if len(tokens)!=2{
                glog.Errorf("invalid value of etcd server overrides %s", override)
                continue
            }
            apiresource:=strings.Split(tokens[0], "/")
            if len(apiresource)!=2{
                glog.Errorf("invalid resource definition: %s", tokens[0])
                continue
            }
            group:=apiresource[0]
            resource:=apiresource[1]
            groupResource:=schema.GroupResource{Group: group, Resource: resource}
            servers:=strings.Split(tokens[1],";")
            storageFactory.SetEtcdLocation(groupResource, servers)
        }
        if s.Etcd.EncryptionProviderConfigFilepath != ""{
            transformerOverrides,err:=encryptionconfig.GetTransformerOverrides(s.Etcd.EncryptionProviderConfigFilepath)
            if err!=nil{
                return nil,err
            }
            for groupResource,transformer:=range transformerOverrides{
                storageFactory.SetTransformer(groupResource,transformer)
            }
        }

        //此处storageFactory的字段包含：
        //storageFactory.Overrides=
        //map[{extensions networkpolicies}:{[]    <nil> [{extensions networkpolicies} {networking.k8s.io networkpolicies}] <nil> <nil> <nil>}
        //    {networking.k8s.io networkpolicies}:{[]    <nil> [{extensions networkpolicies} {networking.k8s.io networkpolicies}] <nil> <nil> <nil>}
        //    {extensions deployments}:{[]    <nil> [{extensions deployments} {apps deployments}] <nil> <nil> <nil>}
        //    {apps deployments}:{[]    <nil> [{extensions deployments} {apps deployments}] <nil> <nil> <nil>}
        //]
        //storageFactory.DefaultResourcePrefixes=
        //map[{ services}:services/specs
        //    {extensions ingresses}:ingress
        //    {extensions podsecuritypolicies}:podsecuritypolicy
        //    { replicationControllers}:controllers
        //    { replicationcontrollers}:controllers
        //    { endpoints}:services/endpoints
        //    { nodes}:minions
        //]
        //storageFactory.StorageConfig.Type=""
        //storageFactory.StorageConfig.Prefix="/registry"
        //storageFactory.StorageConfig.ServerList=flag传入的值，例如：http://127.0.0.1:2379
        //storageFactory.StorageConfig.KeyFile=""
        //storageFactory.StorageConfig.CertFile=""
        //storageFactory.StorageConfig.CAFile=""
        //storageFactory.StorageConfig.Quorum=false
        //storageFactory.StorageConfig.DeserializationCacheSize=1000
        //storageFactory.DefaultMediaType="application/vnd.kubernetes.protobuf"

        return storageFactory,nil
    }

（1）StorageGroupsToEncodingVersion

含义：

    在etcd中存储的group/version信息。

路径：

    k8s.io/kubernetes/pkg/kubernetes/options/storage_versions.go

定义：

    func (s *StorageSerializationOptions) StorageGroupsToEncodingVersion()(map[string]schema.GroupVersion, error){
        storageVersionMap := map[string]schema.GroupVersion{}
        //在创建SeverRunOptions的StorageSerializations时已经默认注册了一些group/version对（详细请参见前面章节）。这里将默认的group/version添加到storageVersionMap中。支持2种格式，一种格式：group/version，添加到storageVersionMap中是：{group : {group : version}}。另一种格式：group1=group2/version，添加到storageVersionMap中是：{group1 : {group2 : version}}
        if err:=mergeGroupVersionIntoMap(s.DefaultStorageVersions, storageVersionMap);err!=nil{
            return nil,err
        }

        //用户指定的s.StorageVersions会覆盖默认group/version信息。用户是通过flag参数"--storage-versions"来设置s.StorageVersions参数的。
        if err:=mergeGroupVersionIntoMap(s.StorageVersions, storageVersionMap); err!=nil{
            return nil,err
        }
        return storageVersionMap,nil
    }

（2）NewStorageFactory

含义：

    创建默认Storage factory。DefaultStorageFactory的内容包含：StorageConfig（Type="", Prefix="/registry", DeserializationCacheSize=1000, Quorum=false, ServerList=flag传入的ip，KeyFile="",CertFile="",CAFile=""）、Overrides、DefaultMediaType（"application/vnd.kubernetes.protobuf"）、DefaultSerializer、ResourceEncodingConfig（就是前面的storageGroupsToEncodingVersion）、APIResourceConfigSource、DefaultResourcePrefixes、newStorageCodecFn。

路径：

    k8s.io/kubernetes/pkg/kubeapiserver/default_storage_factory_builder.go

参数：

    s.Etcd.StorageConfig
    s.Etcd.DefaultStorageMediaType
    s.api.Codecs
    serverstorage.NewDefaultResourceEncodingConfig(api.Registry)
    storageGroupsToEncodingVersion
    []schema.GroupVersionResource{batch.Resource("cronjobs").WithVersion("v2alpha1")},
    master.DefaultAPIResourceConfigSource(), s.APIEnablement.RuntimeConfig

定义：

    func NewStorageFactory(storageConfig storagebackend.Config, defaultMediaType string, serializer runtime.StorageSerializer, defaultResourceEncoding *serverstorage.DefaultResourceEncodingConfig, storageEncodingOverrides map[string]schema.GroupVersion, resourceEncodingOverrides []schema.GroupVersionResource, defaultAPIResourceConfig *serverstorage.ResourceConfig, resourceConfigOverrides utilflag.ConfigurationMap) (*serverstorage.DefaultStorageFactory, error) {
        resourceEncodingConfig:=mergeGroupEncodingConfigs(defaultResourceEncoding, storageEncodingOverrides)
        resourceEncodingConfig=mergeResourceEncodingConfigs(resourceEncodingConfig, resourceEncodingOverrides)
        apiResourceConfig, err:=mergeAPIResourceConfigs(defaultAPIResourceConfig, resourceConfigOverrides)
        if err!=nil{
            return nil, err
        }
        return serverstorage.NewDefaultStorageFactory(storageConfig, defaultMediaType, serializer, resourceEncodingConfig, apiResourceConfig), nil
    }

    func mergeGroupEncodingConfigs(defaultResourceEncoding *serverstorage.DefaultResourceEncodingConfig, storageEncodingOverrides map[string]schema.GroupVersion) *serverstorage.DefaultResourceEncodingConfig{
        resourceEncodingConfig:=defaultResourceEncoding
        for group,storageEncodingVersion:=range storageEncodingOverrides{
            resourceEncodingConfig.SetVersionEncoding(group, storageEncodingVersion, schema.GroupVersion{Group: group, Version: runtime.APIVersionInternal})
        }
        return resourceEncodingConfig
    }

    func mergeResourceEncodingConfigs(defaultResourceEncoding *serverstorage.DefaultResourceEncodingConfig, resourceEncodingOverrides []schema.GroupVersionResource) *serverstorage.DefaultResourceEncodingConfig{
        resourceEncodingConfig:=defaultResourceEncoding
        for _,gvr:=range resourceEncodingOverrides{
            resourceEncodingConfig.SetResourceEncoding(gvr.GroupResource(), gvr.GroupVersion(), schema.GroupVersion{Group: gvr.Group, Version: runtime.APIVersionInternal})
        }
        return resourceEncodingConfig
    }

    func mergeAPIResourceConfigs(defaultAPIResourceConfig *serverstorage.ResourceConfig, resourceConfigOverrides utilflag.ConfigurationMap)(*serverstorage.ResourceConfig,error){
        ...
    }

    //k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/server/storage/storage_factory.go
    type DefaultStorageFactory struct{
        StorageConfig storagebackend.Config
        Overrides map[schema.GroupResource]groupResourceOverrides
        DefaultResourcePrefixes map[schema.GroupResource]string
        DefaultMediaType string
        DefaultSerializer runtime.StorageSerializer
        ResourceEncodingConfig ResourceEncodingConfig   //返回资源的序列化表示或内存表示的GroupVersion。
        APIResourceConfigSource APIResourceConfigSource
        newStorageCodecFn func(opts StorageCodecConfig) (codec runtime.Codec, err error)
    }

    func NewDefaultStorageFactory(config storagebackend.Config, defaultMediaType string, defaultSerializer runtime.StorageSerializer, resourceEncodingConfig ResourceEncodingConfig, resourceConfig, resourceConfig APIResourceConfigSource)*DefaultStorageFactory{
        if len(defaultMediaType)==0{
            defaultMediaType=runtime.ContentTypeJSON
        }
        return &DefaultStorageFactory{
            StorageConfig: config,
            Overrides: map[schema.GroupResource]groupResourceOverrides{},
            DefaultMediaType: defaultMediaType,
            DefaultSerializer: defaultSerializer,
            ResourceEncodingConfig: resourceEncodingConfig,
            APIResourceConfigSource: resourceConfig,
            DefaultResourcePrefixes: specialDefaultResourcePrefixes,
            newStorageCodecFn: NewStorageCodec,
        }
    }

### 1.4.10 s.Etcd.ApplyWithStorageFactoryTo
含义：

    加载StorageFactory配置。设置RESTOptionsGetter。

路径：

    k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/server/options/etcd.go

定义：

    func (s *EtcdOptions) ApplyWithStorageFactoryTo(factory serverstorage.StorageFactory, c *server.Config) error{
        c.RESTOptionsGetter = &storageFactoryRestOptionsFactory{Options: *s, StorageFactory: factory}
        return nil
    }

### 1.4.11 internalclientset.NewForConfig
含义：

    创建内部客户端配置。具体方法在前面"创建共享informers"中已经介绍过了。只不过内部版本的客户端没有版本信息。

路径：

    k8s.io/kubernetes/pkg/client/clientset_generated/internalclientset/clientset.go

定义：

    func NewForConfig(c *rest.Config) (*Clientset, error){
        configShallowCopy := *c
        if configShallowCopy.RateLimiter == nil && configShallowCopy.QPS>0{
            configShallowCopy.RateLimiter = flowcontrol.NewTokenBucketRateLimiter(configShallowCopy.QPS, configShallowCopy.Burst)
        }
        var cs Clientset
        var err error
        cs.AdmissionregistrationClient,err = admissionregistrationinternalversion.NewForConfig(&configShallowCopy)
        if err!=nil{
            return nil,err
        }
        cs.CoreClient,err=coreinternalversion.NewForConfig(&configShallowCopy)
        if err!=nil{
            return nil,err
        }
        cs.AppsClient,err=appsinternalversion.NewForConfig(&configShallowCopy)
        if err!=nil{
            return nil,err
        }
        cs.AuthenticationClient,err=authenticationinternalversion.NewForConfig(&configShallowCopy)
        if err!=nil{
            return nil,err
        }
        cs.AuthorizationClient,err=authorizationinternalversion.NewForConfig(&configShallowCopy)
        if err!=nil{
            return nil,err
        }
        cs.AutoscalingClient,err=autoscalinginternalversion.NewForConfig(&configShallowCopy)
        if err!=nil{
            return nil,err
        }
        cs.BatchClient,err=batchinternalversion.NewForConfig(&configShallowCopy)
        if err!=nil{
            return nil,err
        }
        cs.CertificatesClient,err=certificatesinternalversion.NewForConfig(&configShallowCopy)
        if err!=nil{
            return nil,err
        }
        cs.ExtensionsClient,err=extensionsinternalversion.NewForConfig(&configShallowCopy)
        if err!=nil{
            return nil,err
        }
        cs.NetworkingClient,err=networkinginternalversion.NewForConfig(&configShallowCopy)
        if err!=nil{
            return nil,err
        }
        cs.PolicyClient,err=policyinternalversion.NewForConfig(&configShallowCopy)
        if err!=nil{
            return nil,err
        }
        cs.RbacClient,err=rbacinternalversion.NewForConfig(&configShallowCopy)
        if err!=nil{
            return nil,err
        }
        cs.SettingsClient,err=settingsinternalversion.NewForConfig(&configShallowCopy)
        if err!=nil{
            return nil,err
        }
        cs.StorageClient,err=storageinternalversion.NewForConfig(&configShallowCopy)
        if err!=nil{
            return nil,err
        }
        cs.DiscoveryClient,err=discovery.NewDiscoveryClientForConfig(&configShallowCopy)
        if err!=nil{
            glog.Errorf("failed to create the DiscoveryClient: %v", err)
            return nil,err
        }
        return &cs,nil
    }

### 1.4.12 clientset.NewForConfig
含义：

    创建外部部客户端配置。

路径：

    k8s.io/kubernetes/pkg/client/clientset_generated/clientset/clientset.go

定义：

    func NewForConfig(c *rest.Config) (*Clientset, error){
        configShallowCopy := *c
        if configShallowCopy.RateLimiter = nil && configShallowCopy.QPS >0{
            configShallowCopy.RateLimiter = flowcontrol.NewTokenBucketRateLimiter(configShallowCopy.QPS, configShallowCopy.Burst)
        }
        var cs Clientset
        var err error
        cs.AdmissionregistrationV1alpha1Client,err=admissionregistrationv1alhpa1.NewForConfig(&configShallowCopy)
        if err!=nil{
            return nil,err
        }
        cs.CoreV1Client,err=corev1.NewForConfig(&configShallowCopy)
        if err!=nil{
            return nil,err
        }
        cs.AppsV1beta1Client,err=appsv1beta1.NewForConfig(&configShallowCopy)
        if err!=nil{
            return nil,err
        }
        cs.AuthenticationV1Client,err=authenticationv1.NewForConfig(&configShallowCopy)
        if err!=nil{
            return nil,err
        }
        cs.AuthenticationV1beta1Client,err=authenticationv1beta1.NewForConfig(&configShallowCopy)
        if err!=nil{
            return nil,err
        }
        cs.AuthorizationV1Client,err=authorizationv1.NewForConfig(&configShallowCopy)
        if err!=nil{
            return nil,err
        }
        cs.AuthorizationV1beta1Client,err=authorizationv1beta1.NewForConfig(&configShallowCopy)
        if err!=nil{
            return nil,err
        }
        cs.AutoscalingV1Client,err=autoscalingv1.NewForConfig(&configShallowCopy)
        if err!=nil{
            return nil,err
        }
        cs.AutoscalingV2alpha1Client,err=autoscalingv1alpha1.NewForConfig(&configShallowCopy)
        if err!=nil{
            return nil,err
        }
        cs.BatchV1Client,err=batchv1.NewForConfig(&configShallowCopy)
        if err!=nil{
            return nil,err
        }
        cs.BatchV2alpha1Client,err=batchv2alpha1.NewForConfig(&configShallowCopy)
        if err!=nil{
            return nil,err
        }
        cs.CertificatesV1beta1Client,err=certificatesv1beta1.NewForConfig(&configShallowCopy)
        if err!=nil{
            return nil,err
        }
        cs.ExtensionV1betaClient,err=extensionsv1beta1.NewForConfig(&configShallowCopy)
        if err!=nil{
            return nil,err
        }
        cs.NetworkingV1Client,err=networkingv1.NewForConfig(&configShallowCopy)
        if err!=nil{
            return nil,err
        }
        cs.PolicyV1beta1Cient,err=policyv1beta1.NewForConfig(&configShallowCopy)
        if err!=nil{
            return nil,err
        }
        cs.RbacV1beta1Client,err=rbacv1beta1.NewForConfig(&configShallowCopy)
        if err!=nil{
            return nil,err
        }
        cs.RbacV1alpha1Client,err=rbacv1alpha1.NewForConfig(&configShallowCopy)
        if err!=nil{
            return nil,err
        }
        cs.SettingV1alpha1Client,err=settingsv1alpha1.NewForConfig(&configShallowCopy)
        if err!=nil{
            return nil,err
        }
        cs.StorageV1beta1Client,err=storagev1beta1.NewForConfig(&configShallowCopy)
        if err!=nil{
            return nil,err
        }
        cs.StorageV1Client,err=storagev1.NewForConfig(&configShallowCopy)
        if err!=nil{
            return nil,err
        }

        cs.DiscoveryClient,err=discovery.NewDiscoveryClientForConfig(&configShallowCopy)
        if err!=nil{
            glog.Errorf("failed to create the DiscoveryClient: %v", err)
            return nil,err
        }
        return &cs,nil
    }

### 1.4.13 informers.NewSharedInformerFactory
含义：

    创建共享informers，并返回。

路径：

    k8s.io/kubernetes/pkg/client/informers/informers_generated/internalversion/factory.go

定义：

    func NewSharedInformerFactory(cilent internalclientset.Interface, defaultResync time.Duration) SharedInformerFactory{
        return &sharedInformerFactory{
            client: client,
            defaultResync: defaultResync,
            informers: make(map[reflect.Type]cache.SharedIndexInformer),
            startedInformers: make(map[reflect.Type]bool),
        }
    }

### 1.4.14 BuildAuthenticator
含义：

    创建认证器。

路径：

    k8s.io/kubernetes/cmd/kube-apiserver/app/server.go

定义：

    func BuildAuthenticator(s *options.ServerRunOptions, storageFactory serverstorage.StorageFactory, client internalclientset.Inferface, sharedInformers informers.SharedInformerFactory) (authenticator.Request, *spec.SecurityDefinitions,error){
        //创建认证配置
        authenticatorConfig := s.Authentication.ToAuthenticationConfig()

        //默认Lookup=true，除非flag修改它。
        if s.Authentication.ServiceAccounts.Lookup{
            //设置serviceaccounts的后端存储配置。
            storageConfig,err:=storageFactory.NewConfig(api.Resource("serviceaccounts"))
            if err!=nil{
                return nil,nil,fmt,Errorf("unable to get serviceaccounts storage: %v",err)
            }
            //认证配置中获取serviceaccounttoken的配置。
            authenticatorConfig.ServiceAccountTokenGetter = serviceaccountcontroller.NewGetterFromStorageInterface(storageConfig, storageFactory.ResourcePrefix(api.Resource("serviceaccounts")),storageFactory.ResourcePrefix(api.Resource("secrets")))
        }
        if client==nil || reflect.ValueOf(client).IsNil(){
            glog.Errorf("Failed to setup bootstrap token authenticator because the loopback clientset was not setup properly.")
        }else{
            //认证配置中BootstrampToken的认证配置。
            authenticatorConfig.BootstrapTokenAuthenticator = bootstrap.NewTokenAuthenticator(
                sharedInformers.Core().InternalVersion().Secrets().Lister().Secrets(v1.NamespaceSystem),
            )
        }
        return authenticatorConfig.New()
    }

（1）ToAuthenticationConfig

含义：

    创建认证配置。

路径：

    k8s.io/kubernetes/pkg/kubeapiserver/options/authentication.go

定义：

    func (s *BuiltInAuthenticationOptions) ToAuthenticationConfig() authenticator.AuthenticatorConfig{
        ret:=authenticator.AuthenticatorConfig{}
        if s.Anonymous!=nil{
            ret.Anonymous=s.Anonymous.Allow
        }
        if s.AnyToken!=nil{
            ret.AnyToken=s.AnyToken.Allow
        }
        if s.BootstrapToken!=nil{
            ret.BootstrapToken=s.BootstrapToken.Allow
        }
        if s.ClientCert!=nil{
            ret.ClientCAFile=s.ClientCert.ClientCA
        }
        if s.Keystone!=nil{
            ret.KeystoneURL=s.Keystone.URL    
            ret.KeystoneCAFile=s.Keystone.CAFile
        }
        if s.OIDC!=nil{
            ret.OIDCCAFile=s.OIDC.CAFile
            ret.OIDCClientID=s.OIDC.ClientID
            ret.OIDCGroupsClaim=s.OIDC.GroupsClaim
            ret.OIDCIssuerURL=s.OIDC.IssuerURL
            ret.OIDCUsernameClaim=s.OIDC.UsernameClaim
        }
        if s.PasswordFile!=nil{
            ret.BasicAuthFile=s.PasswordFile.BasicAuthFile
        }
        if s.RequestHeader!=nil{
            ret.RequestHeaderConfig=s.RequestHeader.ToAuthenticationRequestHeaderConfig()
        }
        if s.ServiceAccounts!=nil{
            ret.ServiceAccountKeyFiles=s.ServiceAccounts.KeyFiles
            ret.ServiceAccountLookup=s.ServiceAccounts.Lookup
        }
        if s.TokenFile!=nil{
            ret.TokenAuthFile=s.TokenFile.TokenFile
        }
        if s.WEbHook!=nil{
            ret.WebhookTokenAuthnConfigFile=s.WebHook.ConfigFile
            ret.WebhookTokenAuthnCacheTTL=s.WebHook.CacheTTL
        }
        return ret
    }

（2）New

含义：

    创建各种类型认证配置，"HTTPBasic"和"BearerToken"认证的安全定义。

路径：

    k8s.io/kubernetes/pkg/kubeapiserver/authenticator/config.go

定义：

    func (config AuthenticatorConfig)New() (authenticator.Request, *spec.SecurityDefinitions, error){
        var authenticators []authenticator.Request
        securityDefinitions := spec.SecurityDefinitions{}
        hasBasicAuth:=false
        hasTokenAuth:=false
        if config.RequestHeaderConfig!=nil{
            requestHeaderAuthenticator,err:=headerrequest.NewSecure(
                config.RequestHeaderConfig.ClientCA,
                config.RequestHeaderConfig.AllowedClientNames,
                config.RequestHeaderConfig.UsernameHeaders,
                config.RequestHeaderConfig.GroupHeaders,
                config.RequestHeaderConfig.ExtraHeaderPrefixes,
            )
            if err!=nil{
                return nil,nil,err
            }
            authenticators=append(authenticators, requestHeaderAuthenticator)
        }
        if len(config.BasicAuthFile)>0{
            basicAuth,err:=newAuthenticatorFromBasicAuthFile(config.BasicAuthFile)
            if err!=nil{
                return nil,nil,err
            }
            authenticators=append(authenticators, basicAuth)
            hasBasicAuth=true
        }
        if len(config.KeystoneURL)>0{
            keystoneAuth,err:=newAuthenticatorFromKeystoneURL(config.KeystoneURL, config.KeystoneCAFile)
            if err!=nil{
                return nil,nil,err
            }
            authenticators=append(authenticators, keystoneAuth)
            hasBasicAuth=true
        }
        if len(config.ClientCAFile)>0{
            certAuth,err:=newAuthenticatorFromClientCAFile(config.ClientCAFile)
            if err!=nil{
                return nil,nil,err
            }
            authenticators=append(authenticators, certAuth)
        }
        if len(config.TokenAuthFile)>0{
            tokenAuth, err:=newAuthenticatorFromTokenFile(config.TokenAuthFile)
            if err!=nil{
                return nil,nil,err
            }
            authenticators=append(authenticators, tokenAuth)
            hasTokenAuth=true
        }
        if len(config.ServiceAccountKeyFiles)>0{
            serviceAccountAuth, err:=newServiceAccountAuthenticator(config.ServiceAccountKeyFiles, config.ServiceAccountLookup, config.ServiceAccountTokenGetter)
            if err!=nil{
                return nil,nil,err
            }
            authenticators=append(authenticators, serviceAccountAuth)
            hasTokenAuth=true
        }
        if config.BootstrapToken{
            if config.BootstrapTokenAuthenticator!=nil{
                authenticators=append(authenticators, beerertoken.New(config.BootstrapTokenAuthenticator))
                hasTokenAuth=true
            }
        }
        if len(config.OIDCIssuerURL)>0 && len(config.OIDCClientID)>0{
            oidcAuth,err:=newAuthenticatorFromOIDCIssuerURL(config.OIDCIssuerURL, config.OIDCClientID, config.OIDCCAFile, config.OIDCUsernameClaim, config.OIDCGroupsClaim)
            if err!=nil{
                return nil,nil,err
            }
            authenticators=append(authenticators,oidcAuth)
            hasTokenAuth=true
        }
        if len(config.WebhookTokenAuthnConfigFile)>0{
            webhookTokenAuth,err:=newWebhookTokenAuthenticator(config.WebhookTOkenAuthnConfigFile, config.WebhookTokenAuthnCacheTTL)
            if err!=nil{
                return nil,nil,err
            }
            authenticators=append(authenticators, webhookTokenAuth)
            hasTokenAuth=true
        }
        if config.AnyToken{
            authenticators=append(authenticators, bearertoken.New(anytoken.AnyTokenAuthenticator{}))
            hasTokenAuth=true
        }
        if hasBasicAuth{
            securityDefinitions["HTTPBasic"]=&spec.SecurityScheme{
                SecuritySchemeProprs: spec.SecuritySchemeProps{
                    Type:"basic",
                    Description: "HTTP Basic authentication",
                },
            }
        }
        if hasTokenAuth{
            securityDefinition["BearerToken"]=&spec.SecurityScheme{
                SecuritySchemeProps: spec.SecuritySchemProps{
                    Type: "apiKey",
                    Name: "authorization",
                    In: "header",
                    Description: "Bearer Token authentication",
                },
            }
        }
        switch len(ahtnenticators){
        case 0:
            return ni, &securityDefinitions,nil
        }
        authenticator:=union.New(authenticators...)
        authenticator=group.NewAuthenticatedGroupAdder(authenticator)
        if config.Anonymous{
            authenticator=union.NewFailOnError(authenticator, anonymous.NewAuthenticator())
        }
        return authenticator, &securityDefinitions, nil
    }


### 1.4.15 BuildAuthorizer
含义：

    创建授权器。

路径：

    k8s.io/kubernetes/cmd/kube-apiserver/app/server.go

定义：

    func BuildAuthorizer(s *options.ServerRunOptions, sharedInformers informers.SharedInformerFactory)(authorizer.Authorizer, error){
        authorizationConfig:=s.Authorization.ToAuthorizationConfig(sharedInformers)
        return authorizationConfig.New()
    }

（1）ToAuthorizationConfig

含义：

    创建授权配置结构体。

路径：

    k8s.io/kubernetes/pkg/kubeapiserver/options/authorization.go

定义：

    func (s *BuiltInAuthorizationOptions) ToAuthorizationConfig(informerFactory informers.SharedInformerFactory)authorizer.AuthorizationConfig{
        return authorizer.AuthorizationConfig{
            AuthorizationModes: s.Modes(),           //s.Mode中逗号分隔的字符串slice。
            PolicyFile: s.PolicyFile,
            WebhookConfigFile: s.WebhookConfigFile,
            WebhookCacheAuthorizedTTL: s.WebhookCacheAuthorizedTTL,
            WebhookCacheUnauthorizedTTL:s.WebhookCacheUnauthorizedTTL,
            InformerFactory: informerFactory,
        }
    }

（2）New

含义：

    创建授权配置。

路径：

    k8s.io/kubernetes/pkg/kubeapiserver/authorizer/config.go

定义：

    func (config AuthorizationConfig) New() (authorizer.Authorizer, error){
        if len(config.AuthorizationModes)==0{
            return nil, errors.New("At least one authorization mode should be passed")
        }
        var authorizers []authorizer.Authorizer
        authorizerMap:=make(map[string]bool)
        for _, authorizationMode:=range config.AuthorizationModes{
            if authorizerMap[authorizationMode]{
                return nil, fmt.Errorf("Authorization mode %s specified more than once", authorizationMode)
            }
            switch authorizationMode{
            case modes.ModeNode:
                graph:=node.NewGraph()
                node.AddGraphEventHandlers(
                    graph,
                    config.InformerFactory.Core().InternalVersion().Pods(),
                    config.InformerFactory.Core().InternalVersion().PersistentVolumes(),
                )
                nodeAuthorizer:=node.NewAuthorizer(graph, nodeidentifier.NewDefaultNodeIdentifier(), bootstrappolicy.NodeRules())
                authorizers=append(authorizers, nodeAuthorizer)
                bootstrappolicy.AddClusterRoleBindingFilter(bootstrappolicy.OmitNodesGroupBinding)
            case modes.ModeAlwaysAllow:
                authorizers=append(authorizers, authorizerfactory.NewAlwaysAllowAuthorizer())
            case modes.ModeAlwaysDeny:
                authorizers=append(authorizers, authorizerfactory.NewAlwaysDenyAuthorizer())
            case modes.ModeABAC:
                if config.PolicyFile=""{
                    return nil,errors.New("ABAC's authorization policy file not passed")
                }
                abacAuthorizer,err:=abac.NewFromFile(config.PolicyFile)
                if err!=nil{
                    return nil,err
                }
                authorizers=append(authorizers, abacAuthorizer)
            case modes.ModeWebhook:
                if config.WebhookConfigFile==""{
                    return nil,errors.New("Webhook's configuration file not passed")
                }
                webhookAuthorizer, err:=webhook.New(config.WebhookConfigFIle, config.WebhookCacheAuthorizedTTL,Config.WebhookCacheUnauthorizedTTL)
                if err!=nil{
                    return nil,err
                }
                authorizers=append(authorizers, webhookAuthorizer)
            case modes.ModeRBAC:
                //目前一般使用rbac的授权模式。
                rbacAuthorizer:=rbac.New(
                    &roleGetter{config.InformerFactory.Rbac().InternalVersion().Roles(),Lister()},
                    &roleBindingLister{config.InformerFactory.Rbac().InternalVesion().RoleBindings().Lister()},
                    &clusterRoleGetter{config.InformerFactory.Rbac().InternalVersion().ClusterRoles().Lister()},
                    &clusterRoleBindingLister{config.InformerFactory.Rbac().InternalVersion().ClusterRoleBinding().Lister()},
                )
                authorizers=append(authorizers, rbacAuthorizer)
            default:
                return nil, fmt.Errorf("Unknown authorization mode %s specified", authorizationMode)
            }
            authorizerMap[authorizationMode]=true
        }
        if !authorizerMap[modes.ModeABAC] && config.PolicyFile!=""{
            return ni, errors.New("Cannot specify --authorization-policy-file without mode ABAC")
        }
        if !authorizerMap[modes.ModeWebhook] && config.WebhookConfigFile!=""{
            return nil, errors.New("Cannot specify --authorization-webhook-config-file without mode Webhook")
        }
        return union.New(authorizers...), nil
    }

### 1.4.16 BuildAdmissionPluginInitializer
含义：

    初始化准入控制插件。

路径：

    k8s.io/kubernetes/cmd/kube-apiserver/app/server.go

参数：

    ServerRunOptions
    clientset
    externalClient
    sharedInformers
    genericConfig.Authorizer

定义：

    func BuildAdmissionPluginInitializer(s *options.ServerRunOptions, client internalclientset.Interface, externalClient clientset.Interface, sharedInformers informers.SharedInformerFactory, apiAuthorizer authorizer.Authorizer)(admission.PluginInitializer, error){
        var cloudConfig []byte
        //云提供商配置文件
        if s.CloudProvider.CloudConfigFile!=""{
            var err error
            cloudConfig,err=ioutil.ReadFile(s.CloudProvider.CloudConfigFile)
            if err!=nil{
                glog.Fatalf("Error reading from cloud configuration file %s: %#v", s.CloudProvider.CloudConfigFile, err)
            }
        }
        restMapper:=api.Registry.RESTMapper()
        quotaRegistry:=quotainstall.NewRegistry(nil,nil)
        pluginInitializer:=kubeapiserveradmission.NewPluginInitializer(client, externalClient, sharedInformers, apiAuthorizer, cloudConfig, restMapper, quotaRegistry)
        if len(s.ProxyClientCertFile)>0 && len(s.ProxyClientKeyFile)>0{
            certBytes,err:=ioutil.ReadFile(s.ProxyClientCertFile)
            if err!=nil{
                return nil, err
            }
            keyBytes,err:=ioutil.ReadFile(s.ProxyClientKeyFile)
            if err!=nil{
                return nil,err
            }
            pluginInitializer=pluginInitialier.SetClientCert(certBytes, keyBytes)
        }
        return pluginInitializer,nil
    }

### 1.4.17 s.Admission.ApplyTo
含义：

    设置server.Config的AdmissionControl参数。

路径：

    k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/server/options/admission.gp

定义：

    func (a *AdmissionOptions)ApplyTo(serverCfg *server.Config, pluginInitializers ...admission.PluginInitializer) error{
        pluginConfigProvider,err:=admission.ReadAdmissionConfiguration(a.PluginNames, a.ConfigFile)
        if err!=nil{
            retrn fmt.Errorf("failed to read plugin config: %v", err)
        }
        clientset, err:=kubernetes.NewForConfig(serverCfg.LoopbackClientConfig)
        if err!=nil{
            return err
        }
        genericInitializer,err:=initializer.New(clientset, serverCfg.SharedInformerFactory, serverCfg.Authorizer)
        if err!=nil{
            return err
        }
        initializersChain:=admission.PluginInitializers{}
        pluginInitializers = append(pluginInitializers, genericInitializer)
        initializersChain = append(initializersChain, pluginInitializers...)
        admissionChain, err:=a.Plugins.NewFromPlugins(a.PluginNames, pluginsConfigProvider, initializerChain)
        if err!=nil{
            return err
        }
        serverCfg.AdmissionControl=admissionChain
        return nil
    }

## 1.5 utilwait.PollImmediate
含义：

    尝试连接etcd。

路径：

    k8s.io/kubernetes/vendor/k8s.io/kubernetes/apimachinery/pkg/util/wait/wati.go

实参：

    etcdRetryInterval=1s
    etcdRetryLimit*etcdRetryInterval=60s
    preflight.EtcdConnection{ServerList: s.Etcd.StorageConfig.ServerList}.CheckEtcdServers

定义：

    func PollImmediate(interval, timeout time.Duration, condition ConditionFunc) error{
        return pollImmediateInternal(poller(interval, timeout), condition)
    }

    func pollImmediateInternal(wait WaitFunc, condition ConditionFunc)error{
        done,err:=condition()
        if err!=nil{
            return err
        }
        if done{
            return nil
        }
        return pollInternal(wait,condition)
    }

    func pollInternal(wait WaitFunc, condition ConditionFunc)error{
        done:=make(chan struct{})
        defer close(done)
        return WaitFor(wait, condition,done)
    }

## 1.6 capabilities.Initialize
含义：

    调用capabilities。

路径：

    k8s.io/kubernetes/pkg/capabilities/capabilities.go

定义：

    func Initialize(c Capabilities){
        once.Do(func(){
            capabilities = &c
        })
    }

## 1.7 master.DefaultServiceIPRange
含义：

    默认service ip range。

路径：

    k8s.io/kubernetes/pkg/master/services.go

定义：

    func DefaultServiceIPRange(passedServiceClusterIPRange net.IPNet)(net.IPNet, net.IP, error){
        serviceClusterIPRange := passedServiceClusterIPRange
        if passedServiceClusterIPRange.IP == nil{
            defaultNet:="10.0.0.0/24"
            glog.Infof("Network range for service cluster IPs is unspecified. Defaulting to %v.", defaultNet)
            _,defaultServiceClusterIPRange,err:=net.ParseCIDR(defaultNet)
            if err!=nil{
                return net.IPNet{}, net.IP{}, err
            }
            serviceClusterIPRange=*defaultServiceClusterIPRange
        }
        apiServerServiceIP,err:=ipallocator.GetIndexedIP(&serviceClusterIPRange,1)
        if err!=nil{
            return net.IPNet{}, net.IP{}, err
        }
        glog.V(4).Infof("Setting service IP to %q (read-write).", apiServerServiceIP)
        return serviceClusterIPRange, apiServerServiceIP, nil
    }

## 1.7 BuildStorageFactory
含义：

    创建storage factory，前面已经介绍过了。

路径：

    k8s.io/kubernetes/cmd/kube-apiserver/app/server.go

定义：

    func BuildStorageFactory(s *options.ServerRunOptions)(*serverstorage.DefaultStorageFactory, error){
        storageGroupsToEncodingVersion, err:=s.StorageSerialization.StorageGroupsToEncodingVersions()
        if err!=nil{
            return nil,fmt.Errorf("error generating storage version map: %s", err)
        }
        storageFactory,err:=kubeapiserver.NewStorageFactory(
            s.Etcd.StorageConfig, s.Etcd.DefaultStorageMediaType, api.Codecs,
            serviceStorage.NewDefaultResourceEncodingConfig(api.Registry), storageGroupsToEncodingVersion,
            []schema.GroupVersionResource{batch.Resource("cronjobs").WithVersion("v2alpha1")},
            master.DefaultAPIResourceConfigSource(), s.APIEnablement.RuntimeConfig)
        if err!=nil{
            return nil,fmt.Errorf("error in initializing storage factory: %s", err)
        }
        storageFactory.AddCohabitatingResources(extensions.Resource("deployments"), apps.Resource("deployments"))
        storageFactory.AddCohabitatingResources(extensions.Resource("networkpolicies"), networking.Resource("networkpolicies"))
        for _,override:=range s.Etcd.EtcdServersOverrides{
            tokens:=strings.Split(override, '#')
            if len(tokens)!=2{
                glog.Errorf("invalid value of etcd server override: %s", override)
                continue
            }
            apiresource:=strings.Split(tokens[0],"/")
            if len(apiresource)!=2{
                glog.Errorf("invalid resource definition: %s", tokens[0])
                continue
            }
            group:=apiresource[0]
            resource:=apiresource[1]
            groupResource:=schema.GroupResource{Group: group, Resource: resource}
            servers:=strings.Split(tokens[1],";")
            storageFactory.SetEtcdLocation(groupResource, servers)
        }
        if s.Etcd.EncryptionProviderCondigFilepath!=""{
            transformerOverrides,err:=encryptionconfig.GetTransformerOverrides(s.Etcd.EncryptionProviderConfigFilepath)
            if err!=nil{
                return nil,err
            }
            for groupResource, transformer:=range transformerOverrides{
                storageFactory.SetTransformer(groupResource, transformer)
            }
        }
        return storageFactory,nil
    }

## 1.8 readCAorNil
含义：

    读ca。

路径：

    k8s.io/kubernetes/cmd/kube-apiserver/app/server.go

定义：

    func readCAorNil(file string){[]byte, error}{
        if len(file)==0{
            return nil,nil
        }
        return ioutil.ReadFile(file)
    }

## 1.9 master.Config
含义：

    配置master.Config。

路径：

    k8s.io/kubernetes/cmd/kube-apiserver/app/server.go

定义：

    config:=&master.Config{
        GenericConfig: genericConfig,
        ClientCARegistrationHook: master.ClientCARegistrationHook{
            ClientCA: clientCA,
            RequestHeaderUsernameHeaders: s.Authentication.RequestHeader.UsernameHeaders,
            RequestHeaderGroupHeaders: s.Authentication.RequestHeader.GroupHeaders,
            RequestHeaderExtraHeaderPrefixes: s.Authentication.RequestHeader.ExtraHeaderPrefixes,
            RequestHeaderCA: requestHeaderProxyCA,
            RequestHeaderAllowedNames: s.Authentication.RequestHeader.AllowedNames,
        },
        APIResourceConfigSource: storageFactory.APIResourceConfigSource,
        StorageFactory: storageFactory,
        EnableCoreControllers: true
        EventTTL: s.EventTTL,
        KubeletClientConfig: s.KubeletConfig,
        EnableUISupport: true,
        EnableLogsSupport: s.EnableLogsHandler,
        ProxyTransport: proxyTransport,
        Tunneler: nodeTunneler,
        ServiceIPRange: serviceIPRange,
        APIServerServiceIP: apiServerServiceIP,
        APIServerServicePort: 443
        ServiceNodePortRange: s.ServiceNodePortRange,
        KubernetesServiceNodePort: s.KubernetesServiceNodePort,
        MasterCount: s.MasterCount,
    }


## 参考文献
* [[typeGroupVersionKinds注册的信息]](../../reference/k8s/typeGroupVersionKinds.md)

_______________________________________________________________________
[[返回/kube-apiserver/run/run.md]](./run.md) 
