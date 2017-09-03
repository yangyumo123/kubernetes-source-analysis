glog go日志
================================================
## 简介
执行完命令行解析后，开始执行初始化日志。
     
含义：

    初始化日志。

路径：

    k8s.io/kubernetes/cmd/kube-apiserver/apiserver.go

定义：

    //入口
    func main(){
        ...
        logs.InitLogs()
        defer logs.FlushLogs()
        ...
    }

    //k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/util/logs/logs.go
    func InitLogs(){
        log.SetOutput(GlogWriter{})                                    //将日志输出到标准输出中
        log.SetFlags(0)                                                //设置logging的flag为0。
        go wait.Until(glog.Flush, *logFlushFreq, wait.NeverStop)       //默认glog刷新周期是30s，时间太长，所以这里重新设置为5s。
    }

    //标准log包和glog包之间的桥梁
    type GlogWriter struct{}

## 打印版本
含义：

    打印版本并退出。

路径：

    k8s.io/kubernetes/cmd/kube-apiserver/apiserver.go

定义：

    //入口
    func main(){
        ...
        verflag.PrintAndExistIfRequested()                  //如果命令行中有"--version"或者"--version=true"，则打印版本信息，并退出。否则，继续执行下面程序。
        ...
    }

_______________________________________________________________________
[[返回/kube-apiserver/kube-apiserver.md]](../kube-apiserver.md) 