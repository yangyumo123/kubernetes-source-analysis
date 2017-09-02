flag 命令行参数
=================================================================
## 简介
命令行参数有两种：flag和args。flag是以"-"或"--"开头的参数，args是除了命令名称和flag之外的参数。flag参数对应Go的结构体：Flag，flag参数的集合对应Go的结构体FlagSet（即CommandLine全局变量，FlagSet中有3个字段很重要：formal - 是一个map，表示预先注册的所有flag参数；actual - 是一个map，表示通过命令行传入的实际flag参数；args - 是一个string slice，表示args参数）。

kubernetes中使用两种flag包：flag（Go原生包）和github.com/spf13/pflag。最终合并成pflag包中的CommandLine。

Go语言中全局const、全局var和init函数都是在main函数之前执行的，而且是按照import包的顺序执行全局const、全局var和init函数。CommandLine全局变量就是在main函数之前定义的。

我在远程调试中跟踪了包括全局var、init函数和main函数在内的1500多个断点，放在参考文献[[remote debug kubernetes]](../../reference/remote-debug/remote-debug.md/)中，感兴趣的同事可以了解一下。

## 约定
本文注释中的使用"flag=默认值"的表示法。例如，"--allow-privileged=false"，表示AllowPrivileged字段对应的flag是"--allow-privileged"，默认值是false。其实实际的flag应该是"-allow-privileged"，为了简便，我们写作"--allow-privileged"。

## 目录
1. [数据结构ServerRunOptions]()
2. [创建ServerRunOptions对象]()
3. [添加Flag到FlagSet中]()
4. [初始化Flag]()

## 参考文献
* [[remote debug kubernetes]](../../reference/remote-debug/remote-debug.md/)

_______________________________________________________________________
[[返回/kube-apiserver/kube-apiserver.md]](../kube-apiserver.md) 