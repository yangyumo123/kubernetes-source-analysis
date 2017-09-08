创建SchedulerServer
=========================================================
## 简介
SchedulerServer结构体拥有运行一个Scheduler的所有内容和参数。

## SchedulerServer结构体
含义：

    SchedulerServer定义了Scheduler的配置信息。

路径：

    k8s.io/kubernetes/plugin/cmd/kube-scheduler/app/options/options.go

定义：

    type SchedulerServer struct{
        componentconfig.KubeSchedulerConfiguration
        Master     string                               //kube-apiserver的地址，会覆盖kubeconfig文件中的值。对应的flag："--master"。
        Kubeconfig string                           //kubeconfig配置文件的路径，包含授权信息和master地址信息。对应flag："--kubeconfig"。
    }

### 1. KubeSchedulerConfiguration
含义：

    调度器通用配置。

路径：

    k8s.io/kubernetes/pkg/apis/componentconfig/types.go

定义：

    type KubeSchedulerConfiguration struct {
        metav1.TypeMeta                                               //Kind、APIVersion
        Port                           int32                          //调度器服务监听端口。flag："--port"。
        Address                        string                         //调度器服务监听ip。flag："--address"。
        AlgorithmProvider              string                         //调度算法提供者。flag："--algorithm-provider"。
        PolicyConfigFile               string                         //policy配置文件的路径。flag："--policy-config-file"。如果未设置policy ConfigMap，则使用该文件；或者如果设置了"--use-legacy-policy-config=true"，则使用该文件。
        EnableProfiling                bool                           //打开profiling开关，可以通过host:ip/debug/pprof查看性能数据。apiserver中已经介绍过了。flag："--profiling=true"。
        EnableContentionProfiling      bool                           //打开contention profiling开关，当打开profiling开关时，需要锁住contention profiling。flag："--contention-profiling=false"。
        ContentType                    string                         //scheduler发送给apiserver的http请求中的ContentType。flag："--kube-api-content-type"。
        KubeAPIQPS                     float32                        //给apiserver发送请求的QPS。flag："--kube-api-qps"。
        KubeAPIBurst                   int32                          //给apiserver发送请求的QPS burst。flag："--kube-api-burst"。
        SchedulerName                  string                         //调度器名字。在pod的spec.SchedulerName可以选择使用哪种调度器，就是根据该参数来选择调度器。flag："--scheduler-name"。
        HardPodAffinitySymmetricWeight int                            //implicit PreferredDuringScheduling规则的权重，范围0-100。flag："--hard-pod-affinity-symmetric-weight"。
        FailureDomains                 string                         //以废弃。
        LeaderElection                 LeaderElectionConfiguration    //leader选举客户端配置。
        LockObjectNamespace            string                         //lock对象namespace。flag："--lock-object-namespace"。
        LockObjectName                 string                         //lock对象名字。flag："--lock-object-name"。
        PolicyConfigMapName            string                         //指定policy配置的configmap的名字。在调度器初始化之前，ConfigMap对象必须存在于PolicyConfigMapNamespace中。flag："--policy-configmap"。
        PolicyConfigMapNamespace       string                         //上面的configmap所在的namespace。none表示默认在"kube-system"中。flag："--policy-configmap-namespace"。
        UseLegacyPolicyConfig          bool                           //判断是使用policy配置文件，还是使用configmap。flag："--use-legacy-policy-config=false"。
    }

1. LeaderElectionConfiguration

含义：

    leader选择客户端配置。

路径：

    k8s.io/kubernetes/pkg/apis/componentconfig/types.go

定义：

    type LeaderElectionConfiguration struct {
        LeaderElect   bool              //在执行主循环之前，leader选举可以使leader选举客户端获得leadership。在高可用运行复制组件时启用这个功能。flag："--leader-elect"。
        LeaseDuration metav1.Duration   //flag："--leader-elect-lease-duration"。
        RenewDealine  metav1.Duration   //试图执行master去更新一个leadership到停止leading之间的时间间隔。该值小于等于LeaseDuration。仅当LeaderElect为true时有效。flag："--leader-elect-renew-deadline"。
        RetryPeriod   metav1.Duration   //在试图获取或更新leadership中，客户端需要等待的周期。仅当LeaderElect为true时有效。flag："--leader-elect-retry-period"。
        ResourceLock  string            //在leader选举周期内资源对象类型需要被锁定。flag："--leader-elect-resource-lock"。
    }

## 创建SchedulerServer对象
含义：

    使用默认参数创建SchedulerServer对象。

路径：

    k8s.io/kubernetes/plugin/cmd/kube-scheduler/app/options/options.go

定义：

    func NewSchedulerServer() *SchedulerServer {
        versioned := &v1alpha1.KubeSchedulerConfiguration{}
        api.Scheme.Default(versioned)
        cfg := componentconfig.KubeSchedulerConfiguration{}
        api.Scheme.Convert(versioned, &cfg, nil)
        cfg.LeaderElection.LeaderElect = true
        s := SchedulerServer{
            KubeSchedulerConfiguration: cfg,
        }
        return &s
    }
