数据结构ServerRunOptions
======================================================================
## 简介
ServerRunOptions结构体表示服务器运行参数，用于接收flag参数。在后面的Run函数中，ServerRunOptions中的值又会赋给master.Config结构体，用于服务器运行时。

## 1. 数据结构ServerRunOptions
含义：

    服务器的运行参数。

路径：

    k8s.kubernetes/cmd/kube-apiserver/app/options/options.go

定义：

    type ServerRunOptions struct {
        GenericServerRunOptions     *genericoptions.ServerRunOptions                //服务器通用的运行参数
        Etcd                        *genericoptions.EtcdOptions                     //etcd相关参数
        SecureServing               *genericoptions.SecureServingOptions            //安全服务器参数
        InsecureServing             *kubeoptions.InsecureServingOptions             //非安全服务器参数
        Audit                       *genericoptions.AuditOptions                    //审计参数
        Features                    *genericoptions.FeatureOptions                  //特征参数
        Admission                   *genericoptions.AdmissionOptions                //准入控制参数
        Authentication              *kubeoptions.BuiltInAuthenticationOptions       //认证参数
        Authorization               *kubeoptions.BuiltInAuthorizationOptions        //授权参数
        CloudProvider               *kubeoptions.CloudProviderOptions               //云提供商参数
        StorageSerialization        *kubeoptions.StorageSerializationOptions        //存储版本参数
        APIEnablement               *kubeoptions.APIEnablementOptions               //API可用集参数

        AllowPrivileged              bool
        EnableLogsHandler            bool
        EventTTL                     time.Duration
        KubeletConfig                kubeletclient.KubeletClientConfig
        KubernetesServiceNodePort    int
        MasterCount                  int
        MaxConnectionBytesPerSec     int64
        ServiceClusterIPRange        net.IPNet
        ServiceNodePortRange         utilnet.PortRange
        SSHKeyfile                   string 
        SSHUser                      string
        ProxyClientCertFile          string
        ProxyClientKeyFile           string
        EnableAggregatorRouting      bool
    }

下面对ServerRunOptions结构体中的字段进行详细说明：
### 1.1 GenericServerRunOptions字段
含义：

    服务器通用运行参数。

路径：

    k8s.kubernetes/vendor/k8s.io/apiserver/pkg/server/options/server_run_options.go

定义：

    //这里的ServerRunOptions结构体表示GenericServerRunOptions。后面使用该结构体时还会再次分析它。
    type ServerRunOptions struct {
        AdvertiseAddress            net.IP    //此处flag："--advertise-address=<nil>"。把自己的ip广播给集群中其他组件。后面会修改该默认值，即如果该字段为空，则使用"--bind-address"；如果未设置"--bind-address"，则使用host默认的网络接口。建议设置该flag参数。
        CorsAllowedOriginList       []string  //flag："--cors-allowed-origins=[]"。CORS（Cross-Origin Resource Sharing，跨域资源共享）允许的域列表。
        ExternalHost                string    //此处flag："--external-hostname="。用于产生master对外URL的主机名。例如Swagger API Docs。后面可能会修改该默认值，如果是gce或aws云，则会修改该默认值。
        MaxRequestsInFlight         int       //flag："--max-requests-inflight=400"。最大并发请求数。超过该值，拒绝请求，0值表示不限制请求数。
        MaxMutatingRequestsInFlight int       //flag："--max-mutating-requests-inflight=200"。和MaxRequestsInFlight类似，表示的是mutating请求。Mutating请求指的是被拦截修改后的请求。
        MinRequestTimeout           int       //flag："--min-request-timeout=1800"。最小请求超时时间，仅用于watch请求。
        TargetRAMMB                 int       //此处flag："--target-ram-mb=0"。apiserver内存限制，单位MB，整个集群node使用的最大内存。用于配置缓存，等等。后面会修改该默认值。
        WatchCacheSizes             []string  //此处flag："--watch-cache-sizes=[]"。各资源对象的watch缓存大小列表，以逗号分隔，格式为：resource1#size1,resource2#size2...。当EnableWatchCache为true时有效。后面会修改该默认值。
    }

在设置通用参数时，还有一个flag很重要："--feature-gates"。

#### 1.1.1 feature-gates
含义：

    一组特征开关，描述alpha/实验版本。格式为：key1=value1,key2=value2...。选项有：
    Accelerators=true|false (ALPHA - default=false)
    AdvancedAuditing=true|false (ALPHA - default=false)
    AffinityInAnnotations=true|false (ALPHA - default=false)
    AllAlpha=true|false (ALPHA - default=false)
    AllowExtTrafficLocalEndpoints=true|false (default=true)
    AppArmor=true|false (BETA - default=true)
    DynamicKubeletConfig=true|false (ALPHA - default=false)
    DynamicVolumeProvisioning=true|false (ALPHA - default=true)
    ExperimentalCriticalPodAnnotation=true|false (ALPHA - default=false)
    ExperimentalHostUserNamespaceDefaulting=true|false (BETA - default=false)
    LocalStorageCapacityIsolation=true|false (ALPHA - default=false)
    PersistentLocalVolumes=true|false (ALPHA - default=false)
    RotateKubeletClientCertificate=true|false (ALPHA - default=false)
    RotateKubeletServerCertificate=true|false (ALPHA - default=false)
    StreamingProxyRedirects=true|false (BETA - default=true)
    TaintBasedEvictions=true|false (ALPHA - default=false)

执行步骤：

1. DefaultFeatureGate - 共享的全局特征开关。

        // 路径：k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/util/feature/feature_gate.go

        var DefaultFeatureGate FeatureGate = NewFeatureGate()

        // FeatureGate接口用于解析和存储flag："--feature-gates"。
        type FeatureGate interface{
            AddFlag(fs *pflag.FlagSet)                                //重点关注AddFlag方法，该方法把flag"--feature-gates"注册的命令行FlagSet中。
            Set(value string) error
            Enabled(key Feature) bool
            Add(features map[Feature]FeatureSpec) error               //Add方法往DefaultFeatureGate.known中增加feature。
            KnownFeatures() []string
        }

    // 创建featureGate结构体。featureGate实现了FeatureGate接口，这里就不列出FeatureGate的所有方法了。
    func NewFeatureGate() *featureGate{
        f := &featureGate{
            //AllAlpha是默认feature gate，会被覆盖。
            known: map[string]FeatureSpec{
                "AllAlpha": {Default: false, PreRelease: Alpha},
            },
            special: map[string]func(f *featureGate, val bool){
                "AllAlpha": setUnsetAlphaGates,
            }
            enabled: map[string]bool{},
        }
        return f
    }

    type featureGate struct{
        known   map[string]FeatureSpec
        special map[string]func(*featureGate, bool)               //special的作用是什么？
        enabled map[string]bool                                   //enabled的作用是什么？
        closed  bool                                              //当调用AddFlag时，设置为true。Add判断为true时，会错误退出。注意：初始化不是go-routine安全，查询是go-routine安全的。
    }
    type FeatureSpec struct{
        Default bool
        PreRelease string
    }
    func setUnsetAlphaGates(f *featureGate, val bool){
        for k,v := range f.known {
            if v.PreRelease == Alpha{
                if _, found := f.enabled[k]; !found{
                    f.enabled[k] = val
                }
            }
        }
    }
    const(
        Alpha = "ALPHA"
        Beta  = "BETA"
        GA    = ""
    )

2. init()初始化，往DefaultFeatureGate.known中增加feature。

    // k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/features/kube-features.go
    func init(){
        utilfeature.DefaultFeatureGate.Add(defaultKubernetesFeatureGates)
    }
    var defaultkubenetesFeatureGates = map[string]utilfeature.FeatureSpec{
        StreamingProxyRedirects: {Default: true, PreRelease: utilfeature.Beta},
        AdvancedAuditing: {Default: false, PreRelease: utilfeature.Alpha},
    }
    // k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/util/feature/feature_gate.go
    // Add的目的是往known中增加feature gate。
    // 如果已经执行了AddFlag，则closed被设置为true，再执行Add会错误退出。Add增加重复选项时值必须相同，否则错误退出。
    func (f *featureGate) Add(features map[string]FeatureSpec) error{
        if f.closed{
            return fmt.Errorf("cannot add a feature gate after adding it to the flag set")
        }
        for name, spec := range features{
            if existingSpec, found:=f.known[name]; found{
                if existingSpec == spec{
                    continue
                }
                return fmt.Errorf("feature gate %q with different spec already exists: %v", name, existingSpec)
            }
            f.known[name] = spec
        }
        return nil
    }

    // k8s.io/kubernetes/pkg/features/kube_features.go
    func init(){
        utilfeature.DefaultFeatureGate.Add(defaultKubernetesFeatureGates)
    }
    var defaultKubernetesFeatureGates = map[string]utilfeature.FeatureSpec{
        ExternalTrafficLocalOnly: {Default: true, PreRelease: utilfeature.GA},
        AppArmor: {Default: true, PreRelease: utilfeature.Beta},
        DynamicKubeletConfig: {Default: false, PreRelease: utilfeature.Alpha},
        DynamicVolumeProvisioning: {Default: true, PreRelease: utilfeature.Alpha},
        ExperimentalHostUserNamespaceDefaultingGate: {Default: false, PreRelease: utilfeature.Beta},
        ExperimentalCriticalPodAnnotation: {Default: false, PreRelease: utilfeature.Alpha},
        AffinityInAnnotations: {Default: false, PreRelease: utilfeature.Alpha},
        Accelerators: {Default: false, PreRelease: utilfeature.Alpha},
        TaintBasedEvictions: {Default: false, PreRelease: utilfeature.Alpha},
        RotateKubeletServerCertificate: {Default: false, PreRelease: utilfeature.Alpha},
        RotateKubeletClientCertificate: {Default: false, PreRelease: utilfeature.Alpha},
        PersistentLocalVolumes: {Default: false, PreRelease: utilfeature.Alpha},
        LocalStorageCapacityIsolation: {Default: false, PreRelease: utilfeature.Alpha}

        // inherited features from generic apiserver, relisted here to get a conflict if it is changed unintentionally on either side:
        StreamingProxyRedirects: {Default: true, PreRelease: utilfeature.Beta},
        genericfeatures.AdvancedAuditing: {Default: false, PreRelease: utilfeature.Alpha},
    }

3. 把flag"--feature-gates"加入命令行FlagSet中

    调用顺序如下：
    //k8s.io/kubernetes/cmd/kube-apiserver/apiserver.go
    func main(){
        ...
        s:=options.NewServerRunOptions()
        s.AddFlags(pflag.CommandLine)
        ...
    }
    //k8s.io/kubernetes/cmd/kube-apiserver/app/options/options.go
    func (s *ServerRunOptions) AddFlags(fs *pflag.FlagSet){
        s.GenericServerRunOptions.AddUniversalFlags(fs)
        ...
    }
    //k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/server/options/server_run_options.go
    func (s *ServerRunOptions) AddUniversalFlags(fs *pflag.FlagSet){
        ...
        utilfeature.DefaultFeatureGate.AddFlag(fs)
    }
    //k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/util/feature/feature_gate.go
    func (f *featureGate) AddFlag(fs *pflag.FlagSet){
        f.closed = true
        known:=f.KnownFeatures()
        fs.Var(f,flagName, "A set of key=value pairs that describe feature gates for alpha/experimental features. Options are:\n"+strings.Join(known, "\n"))
    }
    func (f *featureGate) KnownFeatures() []string{
        var known []string
        for k,v := range f.known{
            pre := ""
            if v.PreRelease != GA{
                pre = fmt.Sprintf("%s - ", v.PreRelease)
            }
            known = append(known, fmt.Sprintf("%s=true|false (%default=%t)", k, pre, v.Default))
        }
        sort.Strings(known)
        return known
    }


### 1.2 Etcd字段




_______________________________________________________________________
[[返回/kube-apiserver/flag/flag.md]](./flag.md)