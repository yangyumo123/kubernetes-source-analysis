初始化Flag
=========================================================================
## 简介
初始化Flag，解析命令行传入的Flag参数，存入FlagSet的actual中。

## 1. 初始化Flag
含义：

    解析命令行传入的Flag参数，存入FlagSet的actual中。

路径：

    k8s.io/kubernetes/cmd/kube-apiserver/apiserver.go

定义：

    //入口
    func main(){
        ...
        flag.InitFlags()
        ...
    }

    //k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/util/flag/flags.go
    func InitFlags(){
        pflag.CommandLine.SetNormalizeFunc(WordSepNormalizeFunc)        //规范化FlagSet的formal中的key的名字，将所有"_"改为"-"。
        pflag.CommandLine.AddGoFlagSet(goflag.CommandLine)              //把Go原生flag包中的formal加入到pflag包中的formal中。
        pflag.Parse()
    }
     
## 2. 通用Flag
含义：

    在main函数执行之前，已经注册了一些Go原生的Flag和pflag包的CommandLine中注册的一些通用Flag，下面就介绍一下这些在init函数中注册的Flag。

定义：

    //k8s.io/kubernetes/vendor/github.com/golang/glog/glog.go
    // 这里的flag是go原生flag包
    func init(){
        flag.BoolVar(&logging.toStderr, "logtostderr", false, "log to standard error instead of files")                 //BoolVar向go原生FlagSet注册bool类型的flag。--logtostderr表示日志写入标准错误。
        flag.BoolVar(&logging.alsoToStderr, "alsologtostderr", false, "log to standard error as well as files")         //日志既写入标准错误，又写入文件。
        flag.Var(&logging.verbosity, "v", "log level for V logs")                                                       //Var向go原生FlagSet注册Value类型的flag，verbosity必须实现Value接口，下面类似。
        flag.Var(&logging.stderrThreshold, "stderrthreshold", "logs at or above this threshold go to stderr")                       
        flag.Var(&logging.vmodule, "vmodule", "comma-separated list of pattern=N settings for file-filtered logging")
        flag.Var(&logging.traceLocation, "log_backtrace_at", "when logging hits line file:N, emit a stack trace")
        logging.stderrThreshold = errorLog                                                                              //修改logging.stderrThreshold的值，默认：级别大于等于2会写入stderr
        logging.setVState(0, nil, false)                                                                                //设置v级别为0，moduleSpec为空，不设置filter。
        go logging.flushDaemon()                                                                                        //每个30s刷新一次日志（所有级别的日志）并同步到磁盘。
    }

    //k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/util/logs/logs.go
    //这里将flag：log-flush-frequency注册到pflag中的FlagSet的formal中
    var logFlushFreq = pflag.Duration("log-flush-frequency", 5*time.Second, "Maximum number of seconds between log flushes")

    // func Set(name, value string) error {return CommandLine.Set(name,value)}   这里的CommandLine是Go原生的FlagSet。
    // flag.Set()设置的是原生FlagSet的actual，即f.actual["logtostderr"] = flag，flag的值为true。并没有改变formal的值。
    func init(){
        flag.Set("logtostderr", "true")     
    }

    //k8s.io/kubernetes/pkg/cloudprovider/providers/gce/gce_loadbalancer.go
    func init(){
        var err error
        lbSrcRngsFlag.ipn, err=netsets.ParseIPNets([]string{"130.221.0.0/22", "35.191.0.0/16, "209.85.152.0/22", "209.85.204.0/22"}...)
        if err!=nil{
            panic("Incorrect default GCE L7 source ranges")
        }
        flag.Var(&lbSrcRngsFlag, "cloud-provider-gce-lb-src-cidrs", "CIDRS opened in GCE firewall for LB traffic proxy & health checks")  //此处是Go原生的FlagSet。
    }

## 3. 解析命令行参数
含义：

    从命令行列表中解析出Flag，存入spf13/pflag包中的FlagSet的actual中。

路径：

    k8s.io/kubernetes/vendor/github.com/spf13/pflag/flag.go

定义：

    func Parse(){
        CommandLine.Parse(os.Args[1:])            
    }
    func (f *FlagSet)Parse(arguments []string)error{
        f.parsed = true
        f.args = make([]string, 0, len(arguments))
        assign :=func(flag *Flag, value, origArg string) error{              //设置flag到actual中。
            return f.setFlag(flag, value, origArg)
        }
        err:=f.parseArgs(arguments, assign)f                                 //解析命令行列表中的参数，覆盖FlagSet中的Flag默认值，覆盖ServerRunOptions中的默认值。
        if err!=nil{
            switch f.errorHandling{
            case ContinueOnError:
                return err
            case ExitOnError:
                os.Exit(2)
            case PanicOnError:
                panic(err)
            }
        }
        return nil
    }


_______________________________________________________________________
[[返回/kube-apiserver/flag/flag.md]](./flag.md) 