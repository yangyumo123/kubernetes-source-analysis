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




## 参考文献
* [[api.Scheme分析]](../../reference/k8s/api.Scheme.md)

_______________________________________________________________________
[[返回/kube-apiserver/flag/flag.md]](./flag.md)