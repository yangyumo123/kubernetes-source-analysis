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

        var defaultkubernetesFeatureGates = map[string]utilfeature.FeatureSpec{
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
含义：

    etcd相关配置。

路径：

    k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/server/options/etcd.go

定义：

    type EtcdOptions struct{
        StorageConfig                    storagebackend.Config   //后端存储（etcd）的配置
        EncryptionProviderConfigFilepath string                  //flag："--experimental-encryption-provider-config="。该配置文件供加密提供商用来在etcd中存储secrets。
        EtcdServersOverrides             []string                //flag："--etcd-servers-overrides=[]"。etcd servers覆盖的每个资源，逗号分隔，格式：group/resource#servers，其中servers格式：http://ip:port。
        DefaultStorageMediaType          string                  //flag："--storage-media-type=application/vnd.kubernetes.protobuf"。etcd默认存储MIME类型。
        DeleteCollectionWorkers          int                     //flag："--delete-collection-workers=1"。调用DeleteCollection方法启动的线程数量，用于提高namespace清理速度。
        EnableGarbageCollection          bool                    //flag："--enable-garbage-collector=true"。是否打开gc，必须和kube-controller-manager相应的flag同步。
        EnableWatchCache                 bool                    //flag："--watch-cache=true"。是否打开所有对象的watch cache功能。
        DefaultWatchCacheSize            int                     //无flag。默认的watch cache大小。此处默认值：100。单位是MB。
    }

#### 1.2.1 StorageConfig
含义：

    etcd后端存储的配置参数。

路径：

    k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/storage/storagebackend/config.go

定义：

    type Config struct{
        Type                     string                //flag："--storage-backend="。后端存储（storage backend）类型，例如："etcd2"，"etcd3"，默认""表示"etcd3"。
        Prefix                   string                //flag："--etcd-prefix=/registry"。etcd中所有资源路径的前缀。
        ServerList               []string              //flag："--etcd-servers=[]"。与apiserver连接的etcd server列表，格式：scheme://ip:port，逗号分隔。该参数通过命令行传入，必须被设置。
        KeyFile                  string                //flag："--etcd-keyfile="。apiserver和etcd安全连接的TLS秘钥文件（包含公钥和私钥）。
        CertFile                 string                //flag："--etcd-certfile="。apiserver和 etcd安全连接的TLS证书文件。
        CAFile                   string                //flag："--etcd-cafile="。apiserver和 etcd安全连接的TLS CA证书文件。
        Quorum                   bool                  //flag："--etcd-quorum-read=false"。设置read quorum。
        DeserializationCacheSize int                   //此处flag："--deseriazation-cache-size=0"。反序列化json对象的缓存大小，当前仅支持etcd2。一旦使用protobuf，就去除这个缓存。后面可能会修改该默认值，即如果默认值为0，则根据TargetRAMMB进行设置，或者通过命令行flag传入。
        Codec                    runtime.Codec         //无falg。
        Copier                   runtime.ObjectCopier  //无falg。
        Transformer              value.Transformer     //无falg。允许数据在持久化到etcd之前进行转换。
    }

1. Quorum

        Quorum机制是解决分布式系统的数据一致性问题的一种方法。为了保证分布式系统的可靠性，对于数据的存储采用多份数据副本，也就是其中一个节点上读取数据失败了那么可以转向另外一个存有相同数据副本的节点读取返回给用户。对于写操作，要保证所有副本都更新成功才给用户返回信息，用户才可以读取。这种方案存在的问题是：写操作时延较大，写并发也有很大影响。能否有一种方案可以不更新完全部数据，就能保证用户读取到更新后的数据呢？Quorum就是一种解决方案。

        先介绍一下抽屉模型：有两个抽屉，一个抽屉装了2个红苹果，另一个抽屉装了2个青苹果，如果拿取3个苹果，必有1个是红苹果。

        如果抽屉模型中的红苹果代表已更新的数据，青苹果代表未更新的数据。可以看出不需要更新完全部数据就可以取出已更新的数据。Quorum机制的实质是将写的部分负载转移给了读负载，读多个副本使得写不会过于劳累。
    
        下面介绍Quorum机制是怎么解决读写负载均衡的：
        假设总共有N个数据副本，其中k个已更新，N-k个未更新，那么任意读取N-k+1个数据就必定有1个数据是更新后的，只要取N-k+1中版本最高的数据给用户即可。
        对于写操作，只需要完成N个副本的更新后，就可以告诉用户操作完成，而不需要全部更新，当然在告诉用户更新完成后，系统内部会慢慢的更新剩余的数据副本，这对用户是透明的。
        在kubernetes中，默认不使用Qurorum机制。

2. Codec

含义：对象序列化和反序列化。详细内容请看参考文献[[Serializer序列化器]](../../reference/k8s/serializer.md/)

路径：

    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/runtime/interfaces.go

定义：

    type Codec Serializer                      //Codec处理版本对象的序列化和反序列化。Codec类型对象需要实现Serializer接口。
    type Serializer interface{
        Encoder
        Decoder
    }
    type Encoder interface{
        Encode(obj Object, w io.Writer) error  //将对象写到一个流中。如果版本不兼容，或者没有定义conversion，则返回错误。
    }
    type Decoder interface{
        //将data反序列化为对象，使用scheme本来的类型或者提供的defaults类型。返回对象和类型。如果into不为nil，将被用作目的类型，可能会使用它而不是重新分配一个对象。尽管如此，不保证填充这个对象，返回的对象也不保证匹配。如果提供defaults，默认会被用于data。
        Decode(data []byte, defaults *schema.GroupVersionKind, into Object) (Object, *schema.GroupVersionKind, error)
    }

3. Copier

     路径：k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/runtime/interfaces.go
     定义：
//复制对象
type ObjectCopier interface{
     //返回对象的精确拷贝，如果拷贝未完成，则返回error。
     Copy(Object) (Object, error)
}
//Scheme中注册的所有API type都必须实现Object接口，因为序列化/反序列化对象需要使用gvk，而Object接口就是提供对象的gvk。
type Object interface{
     GetObjectKind() schema.ObjectKind
}
//k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/runtime/schema/interfaces.go
type ObjectKind interface{
     SetGroupVersionKind(kind GroupVersionKind)
     GroupVersionKind() GroupVersionKind
}
//k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/runtime/schema/group_version.go
type GroupVersionKind struct{
     Group string
     Version string
     Kind string
}

4. Transformer

     Transformer接口允许数据在从后端存储中读取或者存入后端存储中之前进行转换。接口的方法必须能撤销由其它原因导致的转换。
type Transformer interface{
     //转换来自后端存储的数据或者返回错误。如果磁盘上对象过期了，则设置stale为true，并且要往etcd中写数据，即使数据内容没有改变。
     TransformFromStorage(data []byte, context Context) (out []byte, stale bool, err error)

     //转换数据到后端存储或返回错误。
     TransformToStorage(data []byte, context Context) (out []byte, err error)
}


## 参考文献
* [[Serializer序列化器]](../../reference/k8s/serializer.md/)


_______________________________________________________________________
[[返回/kube-apiserver/flag/flag.md]](./flag.md)