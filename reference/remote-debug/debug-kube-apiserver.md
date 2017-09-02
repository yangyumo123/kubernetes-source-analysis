调试kube-apiserver
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
        flag.BoolVar(&logging.toStderr, "logtostderr", false, "log to standard error instead of files")                 //BoolVar向go原生FlagSet注册bool类型的flag。后面会重新设置为true。
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

________________________________________________________________________________________________________
初始化k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/util/logs，该包有1个文件：

    k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/util/logs/logs.go

k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/util/logs/logs.go

    //日志刷新周期，默认5s。这里将flag：--log-flush-frequency注册到pflag中的FlagSet的formal中
    var logFlushFreq = pflag.Duration("log-flush-frequency", 5*time.Second, "Maximum number of seconds between log flushes")

    // func Set(name, value string) error {return CommandLine.Set(name,value)}   这里的CommandLine是原生的FlagSet。
    // flag.Set()设置的是原生FlagSet的actual，即f.actual["logtostderr"] = flag，flag的值为true。并没有改变formal的值。
    func init(){
        flag.Set("logtostderr", "true")     
    }

________________________________________________________________________________________________________
初始化k8s.io/kubernetes/vendor/github.com/gogo/protobuf/proto

k8s.io/kubernetes/vendor/github.com/gogo/protobuf/proto/decode.go

    // errOverflow is returned when an integer is too large to be represented.
    var errOverflow = erros.New("proto: integer overflow")
    //ErrInternalBadWireType is returned by generated code when an incorrect wire type is encountered. It does not get returned to user code.
    var ErrInternalBadWireType = errors.New("proto: internal error: bad wiretype for oneof")

k8s.io/kubernetes/vendor/github.com/gogo/protobuf/proto/duration_gogo.go

    var durationType = reflect.TypeOf((*time.Duration)(nil)).Elem()

k8s.io/kubernetes/vendor/github.com/gogo/protobuf/proto/encode.go

    var (
        // errRepeatedHasNil is the error returned if Marshal is called with a struct with a repeated field containing a nil element.
        errRepeatedHasNil = errors.New("proto: repeated field has nil element")
        // errOneOfHasNil is the error returned if Marshal is called with a struct with a oneof field containing a nil element.
        errOneofHasNil = errors.New("proto: oneof field has nil value")
        //ErrNil is the error returned if Marshal is called with nil.
        ErrNil = errors.New("proto: Marshal called with nil")
        // ErrTooLarge is the error returned if Marshal is called with a message that encodes to > 2GB.
        ErrTooLarge = errors.New("proto: message encodes to over 2 GB")
    )

k8s.io/kubernetes/vendor/github.com/gogo/protobuf/proto/extensions.go

    //ErrMissingExtension is the error returned by GetExtension if the named extension is not in the message.
    var ErrMissingExtension = errors.New("proto: missing extension")

    var extendableProtoType = reflect.TypeOf((*extendableProto)(nil)).Elem()
    var extendableProtoV1Type = reflect.TypeOf((*extendableProtoV1)(nil)).Elem()
    var extendableBytesType = reflect.TypeOf((*extensionsBytes)(nil)).Elem()
    var extensionRangeType = reflect.TypeOf((*extensionRange)(nil)).Elem()

    // extPropKey is sufficient to uniquely identify an extension.
    type extPropKey struct{
        base reflect.Type
        field int32
    }
    var extProp = struct{
        sync.RWMutex
        m map[extPropKey]*Properties
    }{
        m: make(map[extPropKey]*Properties),
    }

    // A global registry of extensions.
    // The generated code will register the generated descriptors by calling RegisterExtension.
    var extensionMaps = make(map[reflect.Type]map[int32]*ExtensionDesc)

k8s.io/kubernetes/vendor/github.com/gogo/protobuf/proto/lib.go

    // defaults maps a protocol buffer struct type to a slice of the fields, with its scalar fields set to their proto-declared non-zero default values.
    var (
        defaultMu sync.RWMutex
        defaults = make(map[reflect.Type]defaultMessage)
        int32PtrType = reflect.TypeOf((*int32)(nil))
    )
    type defaultMessage struct{
        scalars []scalarField
        nested []int           // struct field index of nested messages
    }
    type scalarField struct{
        index int
        kind reflect.Kind
        value interface{}
    }

k8s.io/kubernetes/vendor/github.com/gogo/protobuf/proto/message_set.go

    // errNoMessageTypeID occurs when a protocol buffer does not have a message type ID. A message type ID is required for storing a protocol buffer in a message set.
    var errNoMessageTypeID = errors.New("proto does not have a message type ID")
    // a global registry of types that can be used in a MessageSet.
    var messageSetMap = make(map[int32]messageSetDesc)
    type messageSetDesc struct{
        t reflect.Type   // pointer to struct
        name string
    }

k8s.io/kubernetes/vendor/github.com/gogo/protobuf/proto/encode.go

    type Marshaler interface{
        Marshal() ([]byte, error)
    }
k8s.io/kubernetes/vendor/github.com/gogo/protobuf/proto/decode.go

    type Unmarshaler interface{
        Unmarshal([]byte) error
    }
k8s.io/kubernetes/vendor/github.com/gogo/protobuf/proto/properties.go

    var protoMessageType = reflect.TypeOf((*Message)(nil)).Elem()

    var (
        marshalerType = reflect.TypeOf((*Marshaler)(nil)).Elem()
        unmarshalerType = relfect.TypeOf((*Unmarshaler)(nil)).Elem()
    )
    var (
        propertiesMu  sync.RWMutex
        propertiesMap = make(map[reflect.Type]*StructProperties)
    )
    type StructProperties struct{
        Prop []*Properties                                       // properties for each field
        reqCount int                                               // required count
        decoderTags tagMap                                  // map for proto tag to struct field number
        decoderOrigNames map[string]int              // map from original name to struct field number
        order []int                                                   // list of struct field numbers in tag order
        unrecField field                                           // field id of the XXX_unrecognized []byte field
        extendable bool                                          // is this an extendable proto
        oneofMarshaler oneofMarshaler
        oneofUnmarshaler oneofUnmarshaler
        oneofSizer oneofSizer
        stype reflect.Type
        OneofTypes map[string]*OneofProperties
    }

    // A global registry of enum types. The generated code will register the generated maps by calling RegisterEnum.
    var enumValueMaps = make(map[string]map[string]int32)
    var enumStringMaps = make(map[string]map[int32]string)

    // A registry of all linked message types. The string is a fully-qualified proto name ("pkg.Message").
    var(
        protoType = make(map[string]reflect.Type)
        revProtoTypes = make(map[reflect.Type]string)
    )

    // A registry of all linked proto files.
    var(
        protoFiles = make(map[string][]byte)     // file name => fileDescriptor
    )

k8s.io/kubernetes/vendor/github.com/gogo/protobuf/proto/text_parser.go

    var(
        errBadUTF8 = errors.New("proto: bad UTF-8")
        errBadHex = errors.New("proto: bad hexadecimal")
    )

k8s.io/kubernetes/vendor/github.com/gogo/protobuf/proto/timestamp_gogo.go

    var timeType = reflect.TypeOf((*time.Time)(nil)).Elem()

k8s.io/kubernetes/vendor/github.com/gogo/protobuf/proto/duration_gogo.go

    func init(){
        RegisterType((*duration)(nil), "gogo.protobuf.proto.duration")   //注册类型，其实就是记录map：protoTypes和revProtoTypes
    }
    type duration struct{
        Seconds int64 `protobuf:"varint,1,opt,name=seconds,proto3" json:"seconds,omitempty"`
        Nanos int32 `protobuf:"varint,2,opt,name=nanos,proto3" json:"nanos,omitempty"`
    }
    func (m *duration) Reset() {*m = duration{}}
    func (*duration) ProtoMessage() {}
    func (*duration) String() string{return "duration<string>"}

k8s.io/kubernetes/vendor/github.com/gogo/protobuf/proto/properties.go

    func RegisterType(x Message, name string){
        if _,ok:=protoTypes[name]:ok{
            log.Printf("proto: duplicate proto type registered: %s", name)
            return
        }
        t:=reflect.TypeOf(x)
        protoTypes[name]=t
        revProtoTypes[t]=name
    }

k8s.io/kubernetes/vendor/github.com/gogo/protobuf/proto/timestamp_gogo.go

    func init(){
        RegisterType((*timestamp)(nil),"gogo.protobuf.proto.timestamp")
    }
    type timestamp struct{
        Seconds int64 `protobuf:"varint,1,opt,name-seconds,proto3" json:"seconds,omitempty"`
        Nanos int32 `protobuf:"varint,2,opt,name-nanos,proto3" json:"nanos,omitempty"`
    }
    func (m *timestamp) Reset() {*m = timestamp{}}
    func (*timestamp) ProtoMessage(){}
    func (*timestamp) String() string{return "timestamp<string>"}


## 参考文献
* [bytes.Buffer原理及使用](../Golang/bytes.Buffer.md)

_______________________________________________________________________
[[返回/reference/remote-debug.md]](./remote-debug.md) 