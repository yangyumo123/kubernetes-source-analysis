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

_______________________________________________________________________
[[返回/kube-apiserver/flag/flag.md]](./flag.md)