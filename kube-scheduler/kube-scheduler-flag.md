kube-scheduler参数
========================================================
## 简介
设置SchedulerServer的默认命令行参数。解析传入的命令行参数。

## 增加Flag参数到FlagSet中
含义：

    为每个SchedulerServer参数（默认值）创建一个Flag参数，并把Flag参数添加到pflag.CommandLine的formal中。方法跟apiserver中的一样。

路径：

    k8s.io/kubernetes/plugin/cmd/kube-scheduler/app/options/options.go

定义：

    func (s *SchedulerServer) AddFlags(fs *pflag.FlagSet) {
        fs.Int32Var(&s.Port, "port", s.Port, "The port that the scheduler's http service runs on")
        fs.StringVar(&s.Address, "address", s.Address, "The IP address to serve on (set to 0.0.0.0 for all interfaces)")
        fs.StringVar(&s.AlgorithmProvider, "algorithm-provider", s.AlgorithmProvider, "The scheduling algorithm provider to use, one of: " + factory.ListAlgorithmProviders())
        fs.StringVar(&s.PolicyConfigFile, "policy-config-file", s.PolicyConfigFile, "File with sheduler policy configuration. This file is used if policy ConfigMap is not provided or --use-legacy-policy-config=true")
        usage := fmt.Sprintf("Name of the ConfigMap object that contains scheduler's policy configuration. It must exist in the system namespace before scheduler initialization if --use-legacy-policy-config=false. The config must be provided as the value of an element in 'Data' map with the key='%v'", SchedulerPolicyConfigMapKey)
        fs.StringVar(&s.PolicyConfigMapName, "policy-configmap", s.PolicyConfigMapName, usage)
        fs.StringVar(&s.PolicyConfigMapNamespace, "policy-configmap-namespace", s.PolicyConfigMapNamespace, "The namespace where policy ConfigMap is located. The system namespace will be used if this is not provided or is empty.")
        fs.BoolVar(&s.UseLegacyPolicyConfig, "use-legacy-policy-config", false, "When set to true, scheduler will ignore policy ConfigMap and uses policy config file")
        fs.BoolVar(&s.EnableProfiling, "profiling", true, "Enable profiling via web interface host:port/debug/pprof/")
        fs.BoolVar(&s.EnableContentionProfiling, "contention-profiling", false, "Enable  lock contention profiling, if profiling is enabled")
        fs.StringVar(&s.Master, "master", s.Master, "The address of the Kubernetes API server (overrides any value in kubeconfig)")
        fs.StringVar(&s.Kubeconfig, "kubeconfig", s.Kubeconfig, "Path to kubeconfig file with authorization and master location information.")
        fs.StringVar(&s.ContentType, "kube-api-content-type", s.ContentType, "Content type of requests sent to apiserver.")
        fs.Float32Var(&s.KubeAPIQPS, "kube-api-qps", s.KubeAPIQPS, "QPS to use while talking with kubernetes apiserver")
        fs.Int32Var(&s.KubeAPIBurst, "kube-api-burst", s.KubeAPIBurst, "Burst to use while talking with kubernetes apiserver")
        fs.StringVar(&s.SchedulerName, "scheduler-name", s.SchedulerName, "Name of the scheduler, used to select which pods will be processed by this scheduler, based on pod's \"spec.SchedulerName\".")
        fs.StringVar(&s.LockObjectNamespace, "lock-object-namespace", s.LockObjectNamespace, "Define the namespace of the lock object.")
        fs.StringVar(&s.LockObjectName, "lock-object-name", s.LockObjectName, "Define the name of the lock object.")
        fs.InVar(&s.HardPodAffinitySymmetricWeight, "hard-pod-affinity-symmetric-weight", api.DefaultHardPodAffinitySymmetricWeight, "RequiredDuringScheduling affinity is not symmetric, but there is an implicit PreferredDuringScheduling affinity rule corresponding to every RequeredDuringScheduling affinity rule. --hard-pod-affinity-symmetric-weight represents the weight of implicit PreferredDuringScheduling affinity rule.")
        fs.MarkDeprecated("hard-pod-affinity-symmetric-weight", "This options was moved to the policy configuration file")
        fs.StringVar(&s.FailureDomains, "failure-domains", kubeletapis.DefaultFailureDomains, "Indicate the \"all topologies\" set for an empty topologyKey when it's used for PreferredDuringScheduling pod anti-affinity.")
        fs.MarkDeprecated("failure-domains", "Doesn't hava any effect. Will be removed in future version.")
        leaderelection.BindFlags(&s.LeaderElection, fs)
        utilfeature.DefaulltFeatureGate.AddFlag(fs)
    }

    //k8s.io/kubernetes/pkg/client/leaderelection/leaderelection.go
    func BindFlags(l *componentconfig.LeaderElectionConfiguration, fs *pflag.FlagSet) {
        fs.BoolVar(&l.LeaderElect, "leader-elect", l.LeaderElect, ""Start a leader election client and gain leadership before executing the main loop. Enable this when running replicated components for high availability.")
	    fs.DurationVar(&l.LeaseDuration.Duration, "leader-elect-lease-duration", l.LeaseDuration.Duration, "The duration that non-leader candidates will wait after observing a leadership renewal until attempting to acquire leadership of a led but unrenewed leader slot. This is effectively the maximum duration that a leader can be stopped before it is replaced by another candidate. This is only applicable if leader election is enabled.")
	    fs.DurationVar(&l.RenewDeadline.Duration, "leader-elect-renew-deadline", l.RenewDeadline.Duration,"The interval between attempts by the acting master to renew a leadership slot before it stops leading. This must be less than or equal to the lease duration. This is only applicable if leader election is enabled.")
	    fs.DurationVar(&l.RetryPeriod.Duration, "leader-elect-retry-period", l.RetryPeriod.Duration, "The duration the clients should wait between attempting acquisition and renewal of a leadership. This is only applicable if leader election is enabled.")
	    fs.StringVar(&l.ResourceLock, "leader-elect-resource-lock", l.ResourceLock, "The type of resource resource object that is used for locking during leader election. Supported options are `endpoints` (default) and `configmap`.")
    }

##初始化Flag
含义：

    规范化，并解析传入的命令行flag参数。

路径：

    k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/util/flag/flags.go

定义：

    func InitFlags() {
        pflag.CommandLine.SetNormalizeFunc(WordSepNormalizeFunc)
        pflag.CommandLine.AddGoFlagSet(goflag.CommandLine)
        pflag.Parse()
    }

### 规范化flag
含义：

    对FlagSet的formal中存储的Flag的名称进行规范处理，并将规范后的名称更新到formal中，规范化函数是WordSepNormalizeFunc，即将"_"替换成"-"。代码的分析在前面的apiserver中已经介绍过了，这里就不描述了。

### 添加Go原生flag.CommandLine到pflg.CommandLine中
含义：

    在init函数阶段已经在Go原生FlagSet中注册了一些Flag，这里将这些Flag添加到pflag包的FlagSet的formal中。代码分析在前面的apiserver中已经介绍过了，这里就不描述了。

### 解析flag
含义：

    解析命令行Flag参数，为解析后的每个值创建一个Flag存储到CommandLine的actual中，并更新formal中相对应的Flag值。代码分析在前面的apiserver中已经介绍过了，这里就不描述了。


## 日志处理
初始化日志、日志定时更新，代码分析在前面的apiserver中已经介绍过了，这里就不描述了。

    logs.InitLogs()
    defer logs.FlushLogs()

## 打印版本
打印版本信息，代码分析在前面的apiserver中已经介绍过了，这里就不描述了。

    verflag.PrintAndExitIfRequested()

_______________________________________________________________________
[[返回kube-scheduler/kube-scheduler.md]](./kube-scheduler.md) 


