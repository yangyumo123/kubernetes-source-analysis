添加Flag到FlagSet中
==================================================================
## 简介
在使用默认值创建完ServerRunOptions对象之后，就是把命令行参数flag添加到FlagSet中，此处的FlagSet是github.com/spf13/pflag包中的CommandLine。关于flag的用法请看参考文献[[flag用法]](../../reference/k8s/flag.md)。

## 1. 创建命令行参数Flag并添加的FlagSet中
含义：

    main函数中添加Flag到FlagSet中。

路径：

    k8s.io/kubernetes/cmd/kube-apiserver/apiserver.go

定义：

    //入口
    func main(){
        ...
        s.AddFlags(pflag.CommandLine)  
        ...
    }

    //ComandLine是全局变量，在main函数之前执行，表示命令行参数（Flag）的集合-FlagSet，存储了各种Flag。FlagSet中有3个字段很重要：formal、actual和args。formal表示预先注册的Flag，actual表示通过"--flag"命令行传入的实际Flag，args表示除了Flag之外的参数。
    var CommandLine = NewFlagSet(os.Args[0], ExitOnError)

    //k8s.io/kubernetes/cmd/kube-apiserver/app/options/options.go
    func (s *ServerRunOptions)AddFlags(fs *pflag.FlagSet){
        s.GenericServerRunOptions.AddUniversalFlags(fs)     //创建Flag，并将Flag注册到CommandLine表示的FlagSet中。此时的Flag是ServerRunOptions的默认值，会注册到FlagSet中的formal中。下面类似，就不一一列举了。
        s.Etcd.AddFlags(fs)
        s.SecureServing.AddFlags(fs)
        s.SecureServing.AddDeprecatedFlags(fs)
        s.InsecureServing.AddFlags(fs0
        s.InsecureServing.AddDeprecatedFlags(fs)
        s.Audit.AddFlags(fs)
        s.Features.AddFlags(fs)
        s.Authentication.AddFlags(fs)
        s.Authorization.AddFlags(fs)
        s.CloudProvider.AddFlags(fs)
        s.StorageSerialization.AddFlags(fs)
        s.APIEnablement.AddFlags(fs)
        s.Admission.AddFlags(fs)

        fs.DurationVar(&s.EventTTL, "event-ttl", s.EventTTL, "Amount of time to retain events.")
        fs.BoolVar(&s.AllowPrivileged, "allow-privileged", s.AllowPrivileged, "...")
        ...
    }


## 参考文献
* [[flag用法]](../../reference/k8s/flag.md/)


_______________________________________________________________________
[[返回/kube-apiserver/flag/flag.md]](./flag.md)