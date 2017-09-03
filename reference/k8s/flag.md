flag用法
========================================================
## 简介
Go原生flag包用来解析命令行参数，flag支持语言格式如下：

    -flag       //仅支持bool类型
    -flag=x  
    -flag x     //仅支持非bool类型

定义flag参数有三种方式：

1.  flag.Xxx()方法，返回一个相应的指针

    import "flag"
    var ip = flag.Int("flagname", 1234, "help message for flagname")

2.  通过flag.XxxVar()方法将flag绑定到一个变量，该种方式返回值类型，如
    
    var flagvar int
    func init(){
        flag.IntVar(&flagVar, "flagname", 1234, "help message for flagname")
    }

3.  通过flag.Var()绑定自定义类型，自定义类型需要实现Value接口（Receives必须为指针），如

    flag.Var(&flagVal, "name", "help message for flagname")
    对于这种类型的flag，默认值为该变量类型的初始值。

    调用flag.Parse()解析命令行参数到定义的flag。解析函数将会碰到第一个非flag命令行参数时停止。
    调用Parse解析后，就可以直接使用flag本身（指针类型）或者绑定了的变量了（值类型）
    fmt.Println("ip has value", *ip)
    fmt.Println("flagvar has value ", flagvar)
    还可以通过flag.Args()，flag.Arg(i)来获取非flag命令行参数。
     
    type Value interface{
        String() string
        Set(string) error
    }

