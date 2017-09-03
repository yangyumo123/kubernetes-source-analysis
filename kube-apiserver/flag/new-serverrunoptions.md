创建ServerRunOptions对象
===========================================================
## 简介
创建服务器运行参数ServerRunOptions对象，设置默认值。命令行flag参数会覆盖ServerRunOptions对象的值。

## 创建实例
含义：

    创建服务器运行参数对象。main函数中是创建ServerRunOptions对象的入口。

路径：

    k8s.io/kubernetes/cmd/kube-apiserver/apiserver.go           - 入口
    k8s.io/kubernetes/cmd/kube-apiserver/app/options/options.go - NewServerRunOptions函数

定义：

    //入口
    func main(){
        rand.Seed(time.Now().UTC().UnixNano())
        s := options.NewServerRunOptions()            //创建ServerRunOptions对象的入口，下面将对该函数进行详细解释。
        ...
    }

    //创建ServerRunOptions对象
    func NewServerRunOptions() *ServerRunOptions {
        s := ServerRunOptions{
            GenericServerRunOptions:   genericoptions.NewServerRunOptions(),
            Etcd:                      genericoptions.NewEtcdOptions(storagebackend.NewDefaultConfig(kubeoptions.DefaultEtcdPathPrefix, api.Scheme, nil)),
            SecureServing:             kubeoptions.NewSecureServingOptions(),
            InsecureServing:           kubeoptions.NewInsecureServingOptions(),
            Audit:                     genericoptions.NewAuditOptions(),
            Features:                  genericoptions.NewFeatureOptions(),
            Admission:                 genericoptions.NewAdmissionOptions(),
            Authentication:            kubeoptions.NewBuiltInAuthenticationOptions().WithAll(),
            Authorization:             kubeoptions.NewBuiltInAuthorizationOptions(),
            CloudProvider:             kubeoptions.NewCloudProviderOptions(),
            StorageSerialization:      kubeoptions.NewStorageSerializationOptions(),
            APIEnablement:             kubeoptions.NewAPIEnablementOptions(),

            EnableLogsHandler:         true,                                            //默认值：true
            EventTTL:                  1*time.hour,                                     //默认值：1h
            MasterCount:               1,                                               //默认值：1
            KubeletConfig:             kubeletclient.KubeletClientConfig {       
                Port:                       ports.KubeletPort,                          //默认值：10250。
                ReadOnlyPort:               ports.KubeletReadOnlyPort,                  //默认值：10255。
                PreferredAddressTypes:      []string{                                   //默认值：[Hostname, InternalDNS, InternalIP, ExternalDNS, ExternalIP]
                    string(api.NodeHostName),                                                                  
                    string(api.NodeInternalDNS),
                    string(api.NodeInternalIP),
                    string(api.NodeExternalDNS),
                    string(api.NodeExternalIP),
                },
                EnableHttps:                true,                                       //默认值：true
                HTTPTimeout:                time.Duration(5) * time.Second,             //默认值：5s
            },
            ServiceNodePortRange:      DefaultServiceNodePortRange,                     //默认值：[30000,32767]
        }
        s.Etcd.DefaultStorageMediaType = "application/vnd.kubernetes.protobuf"          //默认值："application/vnd.kubernetes.protobuf"
        s.Admission.PluginNames        = []string{"AlwaysAdmit"}                        //默认值：["AlwaysAdmit"]

        return &s
    }

    var DefaultServiceNodePortRange = utilnet.PortRange{Base: 30000, Size: 2768}

    //k8s.io/kubernetes/pkg/master/ports/ports.go
    const KubeletPort               = 10250
    const KubeletReadOnlyPort       = 10255

    //k8s.io/kubernets/pkg/api/types.go
    const (
        NodeHostName    NodeAddressType = "Hostname"
        NodeExternalIP  NodeAddressType = "ExternalIP"
        NodeInternalIP  NodeAddressType = "InternalIP"
        NodeExternalDNS NodeAddressType = "ExternalDNS"
        NodeInternalDNS NodeAddressType = "InternalDNS"
    )
    type NodeAddressType string

## 1. genericoptions.NewServerRunOptions()
含义：

    创建GenericServerRunOptions对象。

路径：

    k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/server/options/server_run_options.go

定义：

    func NewServerRunOptions() *ServerRunOptions{
        defaults := server.NewConfig(serializer.CodecFactory{})
        return &ServerRunOptions{
            MaxRequestInFlight:            defaults.MaxRequestsInFlight,             //默认值：400
            MaxMutatingRequestsInFlight:   defaults.MaxMutatingRequestsInFlight,     //默认值：200
            MinRequestTimeout:             defaults.MinRequestTimeout,               //默认值：1800
        }
    }

    //因为上面的NewServerRunOptions函数只使用了Config中的MaxRequestsInFlight、MaxMutatingRequestsInFlight和MinRequestTimeout参数，因此，此处不详细解释Config结构体，留在后面章节中介绍。
    //k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/server/config.go
    func NewConfig(codec serializer.CodecFactory) *Config{
        return &Config{
            ...
            MaxRequestsInFlight:                 400,
            MaxMutatingRequestsInFlight:   200,
            MinRequestTimeout:                  1800,
            ...
        }
    }

## 2. genericoptions.NewEtcdOptions()
含义：

    创建EtcdOptions对象。

路径：

    k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/server/options/etcd.go

定义：

    func NewEtcdOptions(backendConfig *storagebackend.Config) *EtcdOptions {
        return &EtcdOptions{
            StorageConfig:             *backendConfig        //由参数传入，后面会详细介绍。
            DefaultStorageMediaType:   "application/json",   //此参数会被options.NewServerRunOptions()里面的值覆盖，最终默认值："application/vnd.kubernetes.protobuf"。
            DeleteCollectionWorkers:   1,                    //默认值：1
            EnableGarbageCollection:   true,                 //默认值：true
            EnableWatchCache:          true,                 //默认值：true
            DefaultWatchCacheSize:     100,                  //默认值：100，无flag。
        }
    }

### 2.1 StorageConfig参数
含义：

    StorageConfig的值是由实参：storagebackend.NewDefaultConfig(kubeoptions.DefaultEtcdPathPrefix, api.Scheme, nil)来确定。

路径：

    k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/storage/storagebackend/config.go - NewDefaultConfig函数

定义：

    func NewDefaultConfig(prefix string, copier runtime.ObjectCopier, codec runtime.Codec) *Config {
        return &Config{
            Prefix:                   prefix,                //默认值："/registry"，由参数：kubeoptions.DefaultEtcdPathPrefix传入。
            DeserializationCacheSize: 0,                     //默认值：0
            Copier:                   copier,                //默认值：api.Scheme，由参数：api.Scheme传入，因此api.Scheme必须实现runtime.ObjectCopier接口。
            Codec:                    codec,                 //默认值：nil，由参数：nil传入。
        }
    }

    //参数：kubeoptions.DefaultEtcdPathPrefix
    //路径：k8s.io/kubernetes/pkg/kubeapiserver/options/storage_versions.go
    const DefaultEtcdPathPrefix = "/registry"
    
    //参数：api.Scheme
    //路径：k8s.io/kubernetes/pkg/api/register.go
    var Scheme = runtime.NewScheme()
    
    //api.Scheme需要实现runtime.ObjectCopier接口，即实现Copy方法，Copy方法内部又调用Scheme中的cloner的DeepCopy方法来实现深度拷贝。
    func (s *Scheme)Copy(src Object) (Object, error){
        dst,err := s.DeepCopy(src)
        if err!=nil{
            return nil, err
        }
        return dst.(Object), nil
    }
    func (s *Scheme)DeepCopy(src interface{}) (interface{}, error){
        return s.cloner.DeepCopy(src)
    }

关于api.Scheme的详细介绍请看参考文献[[api.Scheme分析]](../../reference/k8s/api.Scheme.md)

## 3. kubeoptions.NewSecureServingOptions
含义：

    创建安全服务器参数。配置安全ip、port、cert、key、默认没有配置sni。

路径：

    k8s.io/kubernetes/pkg/kubeapiserver/options/serving.go

定义：

    func NewSecureServingOptions() *genericoptions.SercureServingOptions{
        return &genericoptions.SecureServingOptions{
            BindAddress: net.ParseIP("0.0.0.0"),                            //默认值："0.0.0.0"。
            BindPort:    6443,                                              //默认值：6443。
            ServerCert:  genericoptions.GenerableKeyCert{
                PairName:      "apiserver",                                 //默认值："apiserver"。
                CertDirectory: "/var/run/kubernetes",                       //默认值："/var/run/kubernetes"。
            },
        }
    }

## 4. kubeoptions.NewInsecureServingOptions
含义：

    创建非安全服务器参数。不认证、不授权、insecure port。

路径：

    k8s.io/kubernetes/pkg/kubeapiserver/options/serving.go

定义：

    func NewInsecureServingOptions() *InsecureServingOptions{
        return &InsecureServingOptions{
            BindAddress: net.ParseIP("127.0.0.1"),                          //默认值："127.0.0.1"
            BindPort:    8080,                                              //默认值：8080
        }
    }

## 5. genericoptions.NewAuditOptions
含义：

    创建审计参数，默认只设置webhook审计mode。webhook审计有2种模式：batch和blocking。batch是指异步批量处理，webhook在内部缓存审计事件，在接收到一定数量的事件后或者一定次数之后发送批量更新。blocking是同步阻塞。

路径：

    k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/server/options/audit.go

定义：

    func NewAuditOptions() *AuditOptions{
        return &AuditOptions{
            WebhookOptions: AuditWebhookOptions{Mode: pluginwebhook.ModeBatch},     //默认值：batch
        }
    }
    //k8s.io/kubernetes/vendor/k8s.io/apiserver/plugin/pkg/audit/webhook/webhook.go
    const ModeBatch = "batch"
    const ModeBlocking = "blocking"

## 6. genericoptions.NewFeatureOptions
含义：

    创建特征参数对象。默认打开debug性能参数。

路径：

    k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/server/options/feature.go

定义：

    func NewFeatureOptions() *FeatureOptions{
        defaults := server.NewConfig(serializer.CodecFactory{})
        return &FeatureOptions{
            EnableProfiling:            defaults.EnableProfiling,             //默认值：true
            EnableContentionProfiling:  defaults.EnableContentionProfiling,   //默认值：false
            EnableSwaggerUI:            defaults.EnableSwaggerUI,             //默认值：false
        }
    }

## 7. genericoptions.NewAdmissionOptions
含义：

    创建准入控制参数对象。

路径：

    k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/server/options/admission.go

定义：

    func NewAdmissionOptions() *AdmissionOptions{
        options := &AdmissionOptions{
            Plugins:     &admission.Plugins{},
            PluginNames: []string{},                                          //此处为空，但是在NewServerRunOptions函数中被设置为"AlwaysAdmit"。
        }
        //注册准入控制插件，此处注册NamespaceLifecycle插件，注册到Plugins中。
        server.RegisterAllAdmissionPlugins(options.Plugins)
        return options
    }

    //k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/server/plugins.go
    func RegisterAllAdmissionPlugins(plugins *admission.Plugins){
        lifecycle.Register(plugins)                                           //注册NamespaceLifecycle插件，对namespace生命周期相关操作进行准入控制。
    }

    //k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/admission/plugin/namespace/lifecycle/admission.go
    func Register(plugins *admission.Plugins){
        plugins.Register(PluginName, func(config io.Reader)(admission.Interface, error){
            return NewLifecycle(sets.NewString(metav1.NamespaceDefault, metav1.NamespaceSystem,metav1.NamespacePublic))
        })
    }

    const PluginName = "NamespaceLifecycle"

    //k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/admission/plugins.go
    func (ps *Plugins)Register(name string, plugin Factory){
        ps.lock.Lock()
        defer ps.lock.Unlock()
        _,found:=ps.registry[name]
        if found{
            glog.Fatalf("Admission plugin %q was registered twice", name)
        }
        if ps.registry==nil{
            ps.registry=map[string]Factory{}
        }
        glog.V(1).Infof("Registered admission plugin %q", name)
        ps.registry[name] = plugin
    }
     
    //注册的插件在Plugins的registry中，key="NamespaceLifecycle"，value=func(config io.Reader)(admission.Interface, error){return NewLifecycle(sets.NewString(metav1.NamespaceDefault, metav1.NamespaceSystem,metav1.NamespacePublic)) }
    const NamespaceDefault string = "default"
    const NamespaceSystem string = "kube-system"
    const NamespacePublic string = "kube-public"
     
    func NewLifecycle(immortalNamespaces sets.String)(admission.Interface,error){
        return newLifecycleWithClock(immortalNamespaces, clock.RealClock{})
    }
    func newLifecycleWithClock(immortalNamespaces sets.String, clock utilcache.Clock)(admission.Interface, error){
        forceLiveLookupCache := utilcache.NewRUExpireCacheWithClock(100, clock)
        return &lifecycle{
            Handler:                         admission.NewHandler(admission.Create, admission.Update, admission.Delete), //NamespaceLifecycle允许CREATE、UPDATE、DELETE操作。
            immortalNamespaces:    immortalNamespaces,
            forceLiveLookupCache:  forceLiveLookupCache,
        },nil
    }
    
    //lifecycle实现admission.Interface接口，因此要实现Admit方法和Handles方法：
    func (l *lifecycle)Admit(a admission.Attributes) error{
        //基于属性的准入控制
        //输入的gvk={"","v1","namespace"}，名称为"default"、"kube-system"和"kube-public"的namespace不能被删除。
        if a.GetOperation()==admission.Delete && a.GetKind().GroupKind() == v1.SchemeGroupVersion.WithKind("Namespace").GroupKind() && l.immortalNamespaces.Has(a.GetName()){
            return errors.NewForbidden(a.GetResource().GroupResource(), a.GetName(), fmt.Errorf("this namespaces may not be deleted"))
        }

        //输入的gvk={"","v1","namespace"}，或者Namespace为空，则执行下面分支。
        if len(a.GetNamespace())==0 || a.GetKind().GroupKind()==v1.SchemeGroupVersion.WithKind("Namespace").GroupKind(){
            //如果是删除namespace，则在删除之前，阻止创建新对象。为了减少缓存更新慢的问题，将namespace添加到一个强制实时查询列表中，以确保不会延迟查询。
            if a.GetOperation()==admission.Delete{
                l.forceLiveLookupCache.Add(a.GetName(), true, forceLiveLookupTTL)
            }
            return nil
        }

        //运行复查，输入的属性Group="authorization.k8s.io"，Resource="localsubjectaccessreviews"时，返回true；其他情况都返回false。
        if isAccessReview(a){
            return nil
        }

        //等待Handler中的readyFunc函数返回ready（表示admission准备处理请求），如果没有定义该函数，则立即返回；如果定义了，则等待ready，默认超时时间是10s。
        if !l.WaitForReady(){
            return admission.NewForbidden(a, fmt.Errorf("not yet ready to handle request"))
        }

        var(
            exists bool
            err error
        )
        ...
        //下面代码不详细列出，只简单解释。
        //等待namespace创建
        //是否查看本地缓存，还是直接访问server
        //在不存在的namespace中不能进行操作
        //在正在删除的namespace中不能创建新对象
    }

    //k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/admission/handler.go
    type Handler struct{
        operations sets.String
        readyFunc ReadyFunc 
    }

    //如果Handler的operations集合中包含operation操作，则返回true。
    func (h *Handler)Handles(operation Operation)bool{
        return h.operations.Has(string(operation))
    }

## 8.kubeoptions.NewBuiltInAuthenticationOptions
含义：

    创建认证参数对象。

路径：

    k8s.io/kubernetes/pkg/kuberapiserver/options/authentication.go

定义：

    func NewBuiltInAuthenticationOptions() *BuiltInAuthenticationOptions{
        return &BuiltInAuthenticationOptions{}
    }

    func (s *BuiltInAuthenticationOptions) WithAll() *BuiltInAuthenticationOptions{
        return s.
            WithAnyonymous().
            WithAnyToken().
            WithBootstrapToken().
            WithClientCert().
            WithKeystone().
            WithOIDC().
            WithPasswordFile().
            WithRequestHeader().
            WithServiceAccounts().
            WithTokenFile().
            WithWebHook()
    }

    func (s *BuiltInAuthenticationOptions) WithAnyonymous() *BuiltInAuthenticationOptions{
        s.Anonymous = &AnonymousAuthenticationOptions{Allow: true}                //默认值：true。--anonymous-auth=true
        return s
    }
    func (s *BuiltInAuthenticationOptions) WithAnyToken() *BuiltInAuthenticationOptions{
        s.AnyToken = &AnyTokenAuthenticationOptions{}                             //默认值：false。--insecure-allow-any-token=false
        return s
    }
    func (s *BuiltInAuthenticationOptions) WithBootstrapToken() *BuiltInAuthenticationOptions{
        s.BootstrapToken = &BootstrapTokenAuthenticationOptions{}                 //默认值：false。--experimental-bootstrap-token-auth=false
        return s
    }
    func (s *BuiltInAuthenticationOptions) WithClientCert() *BuiltInAuthenticationOptions{
        s.ClientCert= &genericoptions.ClientCertAuthenticationOptions{}           //默认值：false。--client-ca-file=""
        return s
    }
    func (s *BuiltInAuthenticationOptions) WithKeystone() *BuiltInAuthenticationOptions{
        s.Keystone= &KeystoneAuthenticationOptions{}                              //--experimental-keystone-url="" --experimental-keystone-ca-file=""
        return s
    }
    func (s *BuiltInAuthenticationOptions) WithOIDC() *BuiltInAuthenticationOptions{
        s.OIDC= &OIDCAuthenticationOptions{}                                      //--oidc-issuer-url="" --oidc-client-id="" --oidc-ca-file="" --oidc-username-claim="" --oidc--groups-claim=""
        return s
    }
    func (s *BuiltInAuthenticationOptions) WithPasswordFile() *BuiltInAuthenticationOptions{
        s.PasswordFile= &PasswordFileAuthenticationOptions{}                      //--basic-auth-file=""
        return s
    }
    func (s *BuiltInAuthenticationOptions) WithRequestHeader() *BuiltInAuthenticationOptions{
        s.RequestHeader= &genericoptions.RequestHeaderAuthenticationOptions{}     //--requestheader-username-headers="" --requestheader-group-headers="" --requestheader-extra-headers-prefix="" --requestheader-client-ca-file="" --requestheader-allowed-names=""
        return s
    }
    func (s *BuiltInAuthenticationOptions) WithServiceAccounts() *BuiltInAuthenticationOptions{
        s.ServiceAccounts= &ServiceAccountAuthenticationOptions{Lookup: true}     //--service-account-key-file="" --service-account-lookup=true
        return s
    }
    func (s *BuiltInAuthenticationOptions) WithTokenFile() *BuiltInAuthenticationOptions{
        s.TokenFile= &TokenFileAuthenticationOptions{}                            //--token-auth-file=""
        return s
    }
    func (s *BuiltInAuthenticationOptions) WithWebHook() *BuiltInAuthenticationOptions{
        s.WebHook= &WebHookAuthenticationOptions{
            CacheTTL: 2*time.Minute                                               //--authenctication-token-webhook-config-file="" --authentication-token-webhook-cache-ttl=2m
        }                    
        return s
    }

## 9. NewBuiltInAuthorizationOptions
含义：

    创建授权参数对象。

路径：

    k8s.io/kubernetes/pkg/kubeapiserver/options/authorization.go

定义：

    func NewBuiltInAuthorizationOptions() *BuiltInAuthorizationOptions{
        return &BuiltInAuthorizationOptions{
            Mode:                        authzmodes.ModeAlwaysAllow,              //默认值：AlwaysAllow
            WebhookCacheAuthorizedTTL:   5*time.Minute,                           //默认值：5m
            WebhookCacheUnauthorizedTTL: 30*time.Second,                          //默认值：30s
        }
    }

## 10. NewCloudProviderOptions
含义：

    创建云提供商参数对象。默认为空。

路径：

    k8s.io/kubernetes/pkg/kubeapiserver/options/cloudprovider.go

定义：

    func NewCloudProviderOptions() *CloudProviderOptions{
        return &CloudProviderOptions{}
    }

## 11. NewStorageSerializationOptions
含义：

    创建存储版本信息。存储版本是存储在etcd中的版本，是已注册组中的最优版本，格式："group1/version1,group2/version2,..."，表示group1组中最优版本是version1，group2组中最优版本是version2，等等。

路径：

    k8s.io/kubernetes/pkg/kubeapiserver/options/storage_versions.go

定义：

    func NewStorageSerializationOptions() *StorageSerializationOptions{
        return &StorageSerializationOptions{
            DefaultStorageVersions:    api.Registry.AllPreferredGroupVersions(),
            StorageVersions:               api.Registry.AllPreferredGroupVersions(),
        }
    }

    //k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apimachinery/registered/registered.go
    func (m *APIRegistrationManager)AllPreferredGroupVersions() string{
        if len(m.groupMetaMap) == 0{
            return ""
        }
        var defaults []string
        //注册的版本为：admission.k8s.io/v1alpha1,admissionregistration.k8s.io/v1alpha1,....，详细请参见前面章节。版本在init中已经注册。
        for _,groupMeta:=range m.groupMetaMap{
            default=append(defaults, groupMeta.GroupVersion.String())
        }
        sort.Strings(defaults)
        return strings.Join(defaults, ",")
    }

## 12. NewAPIEnablementOptions
含义：

    创建APIEnablementOptions对象。正常不使用该参数。

路径：

    k8s.io/kubernetes/pkg/kubeapiserver/options/api_enablement.go

定义：

    func NewAPIEnablementOptions() *APIEnablementOptions{
        return &APIEnablementOptions{
            RuntimeConfig: make(utilflag.ConfigurationMap),
        }
    }




## 参考文献
* [[api.Scheme分析]](../../reference/k8s/api.Scheme.md)

_______________________________________________________________________
[[返回/kube-apiserver/flag/flag.md]](./flag.md)