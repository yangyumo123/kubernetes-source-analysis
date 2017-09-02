调试k8s
=================================================================================================
## 简介
按照调试执行顺序编写.go文件，每个.go文件只列出调试能走到的全局var和init函数。
## 调试过程
初始化k8s.io/kubernetes/vendor/github.com/golang/glog，该包有2个文件：
    k8s.io/kubernetes/vendor/github.com/golang/glog/glog.go
    k8s.io/kubernetes/vendor/github.com/golang/glog/glog_file.go

k8s.io/kubernetes/vendor/github.com/golang/glog/glog.go
    var errVmoduleSyntax = errors.New("syntax error: expect comma-separated list of filename=N")
    var errTraceSyntax   = errors.New("syntax error: expect file.go:234")

k8s.io/kubernetes/vendor/github.com/golang/glog/glog_file.go
    //logDir是日志目录，向Go原始的FlagSet的formal中注册log_dir参数，底层调用如下：
    //返回存储flag值的string变量的地址，默认值为""。这里的CommandLine是src/flag/flag.go中的FlagSet，即Go原生flag包，var CommandLine = NewFlagSet(os.Args[0], ExitOnError)。
    //func String(name string, value string, usage string) *string{
    //    return CommandLine.String(name, value, usage)
    //}
    var logDir   = flag.String("log_dir", "", "If non-empty, write log files in this directory")

    var (
        pid      = os.Getpid()                  //进程id
        program  = filepath.Base(os.Args[0])    //命令名称
        host     = "unknownhost"                //运行程序的主机的hostname，在init函数中设置值。
        userName = "unknownuser"                //运行程序的当前用户，在init函数中设置值。
    )

k8s.io/kubernetes/vendor/github.com/golang/glog/glog.go
    // 这里的flag是Go原生flag，CommandLine是Go原生的FlagSet。
    func init(){
        flag.BoolVar(&logging.toStderr, "logtostderr", false, "log to standard error instead of files")                 //BoolVar向go原生FlagSet注册bool类型的flag。
        flag.BoolVar(&logging.alsoToStderr, "alsologtostderr", false, "log to standard error as well as files")
        flag.Var(&logging.verbosity, "v", "log level for V logs")                                                       //Var向go原生FlagSet注册Value类型的flag，verbosity实现Value接口。
        flag.Var(&logging.stderrThreshold, "stderrthreshold", "logs at or above this threshold go to stderr")
        flag.Var(&logging.vmodule, "vmodule", "comma-separated list of pattern=N settings for file-filtered logging")
        flag.Var(&logging.traceLocation, "log_backtrace_at", "when logging hits line file:N, emit a stack trace")
        logging.stderrThreshold = errorLog                                                                              //修改logging.stderrThreshold的值，默认：级别大于等于2会写入stderr
        logging.setVState(0, nil, false)                                                                                //设置v级别为0，moduleSpec为空，不设置filter。
        go logging.flushDaemon()                                                                                        //每个30s刷新一次日志（所有级别的日志）并同步到磁盘。
     }

    var logging loggingT
    //loggingT：日志的全局参数。
    type loggingT struct{
        toStderr bool                                            // --logtostderr=false。true表示日志写到stderr中，false表示日志写到文件中。
        alsoToStderr bool                                        // --alsologtostderr=false  flag。true表示日志既写到stderr中，又写到文件中。false表示该参数无效。
        stderrThreshold severity                                 // --stderrthreshold。日志写到stderr中的阈值，大于等于stderrThreshold阈值的日志写到stderr中。
        freeList *buffer                                         // 一组byte buffers, 由freeListMu控制并发。关于字节缓存的介绍，请看参考文献 [bytes.Buffer原理及使用]。
        freeListMu sync.Mutex
        mu sync.Mutex
        file [numSeverity]flushSyncWriter
        pcs [1]uintptr
        vmap map[uintptr]Level
        filterLength int32
        traceLocation traceLocation                              // --log-backtrace-at
        vmodule moduleSpec                                       // --vmodule flag
        verbosity Level                                          // -v flag
    }

k8s.io/kubernetes/vendor/github.com/golang/glog/glog_file.go
    func init(){
        h, err := os.Hostname()
        if err == nil {
            host = shortHostname(h)                              //从/proc/sys/kernal/hostname文件获取hostname，并且取"."前面的短名字。
        }
        current, err := user.Current()
        if err==nil {
            username=current.Username                            //获取当前用户名，例如：root
        }
        userName = strings.Replace(userName, `\`,"_",-1)
    }

————————————————————————————————————————————————————————————————————————————————————————————————————————
初始化k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/util/runtime，该包有1个文件：
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/util/runtime/runtime.go

k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/util/runtime/runtime.go
    // ErrorHandlers 是一组错误处理函数。
    var ErrorHandlers = []func(error){
        logError,
        (&rudimentaryErrorBackoff{
            lastErrorTime: time.Now(),
            minPeriod: time.Millisecond,                         //1ms 表示输出日志错误周期。 如果你需要输出日志错误的周期小于1ms，则需要修改此处代码。
        }).OnError,
    }
    //解释：
    //    logError函数打印带有堆栈信息的日志。日志头的格式为：Lmmdd hh:mm:ss.uuuuuu threadid file:line] msg...
    //    OnError方法控制日志打印频率不能超过1ms，如果超过1ms，会sleep直到1ms。

________________________________________________________________________________________________________
初始化k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/util/wait，该包有1个文件：
    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/util/wait/wait.go

k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/util/wait/wait.go
    var NeverStop <-chan struct{} = make(chan struct{})                     //NeverStop 是阻塞式channel，作为服务器运行函数Run的参数传入，保证服务器一直运行不退出。
    var ErrWaitTimeout = errors.New("timed out waiting for the condition")  // ErrWaitTimeout 超时错误。

________________________________________________________________________________________________________
初始化k8s.io/kubernetes/vendor/github.com/spf13/pflag/flag.go

k8s.io/kubernetes/vendor/github.com/spf13/pflag/flag.go
    var ErrHelp = errors.New("pflag: help requested")            //ErrHelp 当存在命令行中有flag "--help"，但是又没有定义该类型的Flag，则返回该错误。
    var CommandLine = NewFlagSet(os.Args[0], ExitOnError)        //这里的CommandLine是spf13包里的FlagSet



## 参考文献
* [bytes.Buffer原理及使用](../Golang/bytes.Buffer.md)
_______________________________________________________________________
[[返回reference/remote-debug.md]](/remote-debug.md) 