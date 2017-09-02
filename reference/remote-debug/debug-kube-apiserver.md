调试kube-apiserver
=================================================================================================
## 简介
按照调试执行顺序编写.go文件，每个.go文件只列出调试能走到的全局var和init函数。

## 调试过程
**初始化k8s.io/kubernetes/vendor/github.com/golang/glog**

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

________________________________________________________________________

**初始化k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/util/runtime**

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
**初始化k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/util/wait**

k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/util/wait/wait.go

    var NeverStop <-chan struct{} = make(chan struct{})                     //NeverStop 是阻塞式channel，作为服务器运行函数Run的参数传入，保证服务器一直运行不退出。
    var ErrWaitTimeout = errors.New("timed out waiting for the condition")  // ErrWaitTimeout 超时错误。

________________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/spf13/pflag/flag.go**

k8s.io/kubernetes/vendor/github.com/spf13/pflag/flag.go

    var ErrHelp = errors.New("pflag: help requested")            //ErrHelp 当存在命令行中有flag "--help"，但是又没有定义该类型的Flag，则返回该错误。
    var CommandLine = NewFlagSet(os.Args[0], ExitOnError)        //这里的CommandLine是spf13包里的FlagSet

________________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/util/logs**

k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/util/logs/logs.go

    //日志刷新周期，默认5s。这里将flag：--log-flush-frequency注册到pflag中的FlagSet的formal中
    var logFlushFreq = pflag.Duration("log-flush-frequency", 5*time.Second, "Maximum number of seconds between log flushes")

    // func Set(name, value string) error {return CommandLine.Set(name,value)}   这里的CommandLine是原生的FlagSet。
    // flag.Set()设置的是原生FlagSet的actual，即f.actual["logtostderr"] = flag，flag的值为true。并没有改变formal的值。
    func init(){
        flag.Set("logtostderr", "true")     
    }

________________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/gogo/protobuf/proto**

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

___________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apimachineray/pkg/runtime/schema**

k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/runtime/schema/generated.pb.go

    funt init(){
        proto.RegisterFile("k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/runtime/schema/generated.proto", fileDescriptorGenerated)
    }

_____________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/conversion**

k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/conversion/converter.go

    var stringType = reflect.TypeOf("")

_____________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/util/errors**

k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/util/errors/errors.go

    var ErrPreconditionViolated = errors.New("precondition is violated")

____________________________________________________________________________________
**初始化ks8.io/kubernetes/vendor/k8s.io/apimachinery/pkg/runtime**

k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/runtime/generated.pb.go

    var(
        ErrInvalidLengthGenerated = fmt.Errorf("proto: negative length found during unmarshaling")
        ErrIntOverflowGenerated = fmt.Errorf("proto: integer overflow")
    )
    func init(){
        proto.RegisterType((*RawExtension)(nil), "k8s.io.apimachinery.pkg.runtime.RawExtension")
        proto.RegisterType((*TypeMeta)(nil), "k8s.io.apimachinery.pkg.runtime.TypeMeta")
        proto.RegisterType((*Unknown)(nil), "k8s.io.apimachinery.pkg.runtime.Unknown")
    }
    func init(){
        proto.RegisterFile("k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/runtime/generated.proto", fileDescriptorGenerated)
    }

**注意：**

        protobuf是google的一种数据交换格式，类似于JSON，但是比JSON效率更高。protobuf支持多种语言，Go语言提供了"github.com/gogo/protobuf/gogoproto"包用于protobuf的操作。
        protobuf的正常操作过程是：
            首先，创建.proto文件，并按照protobuf schema要求编写.proto文件。
            然后，执行protoc-gen-gogo命令，生成.pb.go，.pb.go文件中包含类型定义和编解码函数等等，供运行时使用。
        然而，kubernetes并未使用上述操作流程，而是先编写类型定义文件types.go，然后执行脚本，生成.proto文件和.pb.go。这样做是基于一些考虑的：程序员只需要编写Go的类型定义文件，不需要再学习一门新技术-protobuf schema，由kubernetes脚本自动生成.pb.go文件。另外，如果type.go是由.proto文件生成的，那么kubernetes将依赖protobuf schema，耦合性太强，换成由type.go来生成.proto和.pb.go，则kubernetes不依赖于protobuf schema了。
        关于protobuf的详细介绍请看参考文献。

k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/runtime/types.go

    //TypeMeta：所有顶层对象共享。
    type TypeMeta struct{
        APIVersion string `json:"apiVersion,omitempty" yaml:"apiVersion,omitempty" protobuf:"bytes,1,opt,name=apiVersion"`
        Kindstring `json:"apiVersion,omitempty" yaml:"apiVersion,omitempty" protobuf:"bytes,2,opt,name=Kind"`
    }

    //RawExtension：支持外部版本扩展。
    type RawExtension struct{
        Raw []byte `protobuf:"bytes,1,opt,name=raw"`
        Object Object `json:"-"`
    }

    //Unknown：允许未知类型的API对象。可以用来处理来自插件的API对象。
    type Unknown struct{
        TypeMeta `json:",inline" protobuf:"bytes,1,opt,name=typeMeta"`
        // Raw will hold the complete serialized object which couldn't be matched with a registered type. Most likely, nothing should be done with this except for passing it through the system.
        Raw []byte `protobuf:"bytes,2,opt,name=raw"`
        // ContentEncoding is encoding used to encode 'Raw' data. Unspecified means no encoding.
        ContentEncoding string `protobuf:"bytes,3,opt,name=contentEncoding"`
        // ContentType is serialization method used to serialize 'Raw'. Unspecified means ContentTypeJSON.
        ContentType string `protobuf:"bytes,4,opt,name=contentType"`
    }

    //VersionedObjects：解码调用它给调用者提供一种访问解码过程中对象的所有版本信息。
    type VersionedObjects struct{
        Objects []Object
    }

________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/gopkg.in/inf.v0**

src/math/big/int.go

    // An Int represents a signed multi-precision integer. The zero value for an Int represents the value 0.
    type Int struct{
        neg bool       //sign
        abs nat        // absolute value of the integer
    }
    func NewInt(x int64) *Int{
        return new(Int).SetInt64(x)
    }
    func (z *Int) SetInt64(x int64) *Int{
        neg:=false
        if x<0{
            neg=true
            x=-x
        }
        z.abs=z.abs.setUint64(uint64(x))
        z.neg=neg
        return z
    }

src/math/big/nat.go

    type nat []Word

src/math/big/arith.go

    type Word uintptr

k8s.io/kubernetes/vendor/gopkg.in/inf.v0/dec.go

    var bigInt = [...]*big.Int{
        big.NewInt(0), big.NewInt(1), big.NewInt(2), big.NewInt(3), big.NewInt(4),
        big.NewInt(5), big.NewInt(6), big.NewInt(7), big.NewInt(8), big.NewInt(9),
        bit.NewInt(10),
    }
    var exp10cache [64]big.Int = func() [64]big.Int{
        e10, e10i := [64]big.Int{}, bigInt[1]
        for i, _ := range e10{
            e10[i].Set(e10i)
            e10i=new(big.Int).Mul(e10i, bigInt[10])
        }
        return e10
    }()
    var zeros = []byte("0000000000000000000000000000000000000000000000000000000000000000")
    var lzeros = Scale(len(zeros))   //len=64
    type Scale int32

k8s.io/kubernetes/vendor/gopkg.in/inf.v0/rounder.go

    var intSign = []*big.Int{big.NewInt(-1), big.NewInt(0), big.NewInt(1)}

    func int(){
        RoundExact = rndr{true,
            func(z,q *Dec, rA, rB, *big.Int) *Dec{
                if rA.Sign()!=0{
                    return nil
                }
                return z.Set(q)  //把z设置为q，如果z==q，则直接返回z。
            }}
        RoundDown = rndr{false,
            func(z,q *Dec, rA, rB *big.Int) *Dec{
                return z.Set(q)
            }}
        RoundUp = rndr{true,
            func(z,q *Dec, rA, rB *big.Int) *Dec{
                z.Set(q)
                if rA.Sign()!=0{
                    z.UnscaledBig().Add(z.UnscaledBig(), intSign[rA.Sign()*rB.Sign()+1])
                }
                return z
            }}
        RoundFloor = rndr{true,
            func(z,q *Dec, rA, rB *big.Int) *Dec{
                z.Set(q)
                if rA.Sign()*rB.Sign()<0{
                    z.UnscaledBig().Add(z.UnscaledBig(), intSign[0])
                }
                return z
            }}
        RoundCeil = rndr{true,
            func(z,q *Dec, rA, rB *big.Int) *Dec{
                z.Set(q)
                if rA.Sign()*rB.Sign()>0{
                    z.UnscaledBig().Add(z.UnscaledBig(), intSign[2])
                }
                return z
            }}
        RoundHalfDown = rndr{true, roundHalf(
            func(c int, odd uint) bool{
                return c>0
            })}
        RoundHalfUp = rndr{true, roundHalf(
            func(c int, odd uint){
                return c>=0
            })}
        RoundHalfEven = rndr{true, roundHalf(
            func(c int, odd uint)bool{
                return c>0 || c==0 && odd==1
            })}
    }
    //rndr实现了rounder接口
    type rndr struct {
        useRem bool
        round func(z,quo *Dec, remNum, remDen *big.Int) *Dec
    }
    //注意这里的receiver不是指针
    func (r rndr) UseRemainder() bool{
        return r.useRem
    }
    func (r rndr) Round(z,quo *Dec, remNum, remDen *big.Int) *Dec{
        return r.round(z, quo, remNum, remDen)
    }

    var(
        RoundDown Rounder
        RoundUp Rounder
        RoundFloor Rounder
        RoundCeil Rounder
        RoundHalfDown Rounder
        RoundHalfUp Rounder
        RoundHalfEven Rounder
    )
    var RoundExact Rounder
    type Rounder rounder
    type rounder interface{
        UseRemainder() bool        //是否使用余数
        Round(z, quo *Dec, remNum, remDen *big.Int) *Dec
    }

    //Dec表示任意精度的小数，Dec的值=unscale * 10**(-scale)
    type Dec struct{
        unscaled big.Int
        scale Scale
    }
    type Scale int32
    func (x *Dec) UnscaledBig() *big.Int{
        return &x.unscaled
    }

    // unscaled     scale         String()
    //            0           0              "0"
    //            0           2              "0.00"
    //            0          -2              "0"
    //            1           0              "1"
    //         100          2              "1.00"
    //           10         0               "10"
    //            1          -1              "10"

____________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/mailru/easyjson/buffer**

k8s.io/kubernetes/vendor/github.com/mailru/easyjson/buffer/pool.go

    // Reuse pool: chunk size -> pool
    var buffers = map[int]*sync.Pool{}

    func init(){
        initBuffers()
    }
    func initBuffers(){
        for l:=config.PooledSize; l<=config.MaxSize; l*=2{
            buffers[l]=new(sync.Pool)
        }
    }
    type PoolConfig struct{
        StartSize int
        PooledSize int
        MaxSize int
    }
    var config = PoolConfig{
        StartSize: 128,
        PooledSize: 512,
        MaxSize: 32768,
    }

______________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/go-openapi/swag**
          
k8s.io/kubernetes/vendor/github.com/go-openapi/swag/convert.go

    var evaluatesAsTrue = map[string]struct{}{
        "true":        struct{}{},
        "1":            struct{}{},
        "yes":         struct{}{},
        "ok":          struct{}{},
        "y":            struct{}{},
        "on":          struct{}{},
        "selected":  struct{}{},
        "checked":  struct{}{},
        "t":             struct{}{},
        "enabled":  struct{}{},
    }

k8s.io/kubernetes/vendor/github.com/go-openapi/swag/json.go
    
    // DefaultJSONNameProvider the default cache for types
    var DefaultJSONNameProvider = NewNameProvider()
    // NameProvider represents an object capable of translating from go property names to json property names. This type is thread-safe.
    type NameProvider struct{
        lock *sync.Mutex
        index map[reflect.Type]nameIndex
    }
    type nameIndex struct{
        jsonNames map[string]string
        goNames map[string]string
    }
    func NewNameProvider() *NameProvider{
        return &NameProvider{
            lock: &sync.Mutex{},
            index: make(map[reflect.Type]nameIndex),
        }
    }

    var closers = map[byte]byte{
        '{': '}',
        '[': ']',
    }

k8s.io/kubernetes/vendor/github.com/go-openapi/swag/util.go

    var commonInitialisms = map[string]bool{
        "API": true,
        "ASCII": true,
        "CPU": true,
        "CSS": true,
        "DNS": true,
        "EOF": true,
        "GUID": true,
        "HTML": true,
        "HTTPS": true,
        "HTTP": true,
        "ID": true,
        "IP": true,
        "JSON": true,
        "LHS": true,
        "QPS": true,
        "RAM": true,
        "RHS": true,
        "RPC": true,
        "SLA": true,
        "SMTP": true,
        "SSH": true,
        "TCP": true,
        "TLS": true,
        "TTL": true,
        "UDP": true,
        "UUID": true,
        "UID": true,
        "UI": true,
        "URI": true,
        "URL": true,
        "UTF8": true,
        "VM": true,
        "XML": true,
        "XSRF": true,
        "XSS": true,
    }

    type byLength []string
    var initialisms []string
    func init(){
        for k:= range commonInitialisms{
            initialisms = append(initialsms,k)
        }
        sort.Sort(sort.Reverse(byLength(initialisms)))   //逆序排序
    }

______________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/go-openapi/jsonpointer**

k8s.io/kubernetes/vendor/github.com/go-openapi/jsonpointer/pointer.go

    var jsonPointableType = reflect.TypeOf(new(JSONPointable)).Elem()

    //JSONPointable is an interface for structs to implement when they need to customize the json pointer process
    type JSONPointable interface{
        JSONLookup(string) (interface{}, error)
    }

_______________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/golang.org/x/text/transform**

k8s.io/kubernetes/vendor/golang.org/x/text/transform/transform.go

    var(
        // the destination buffer was too short to receive all of the transformed bytes.
        ErrShortDst = errors.New("transform: short destination buffer")
        // the source buffer has insufficient data to complete the transformation.
        ErrShortSrc = errors.New("transform: short source buffer")
        // transform returned success (nil error) but also returned nSrc inconsistent with the src argument.
        errInconsistentByteCount = errors.New("transform: inconsistent byte count returned")
        //an internal buffer is not large enough to make progress and the Transform operation must be aborted.
        errShortInternal = errors.New("transform: short internal buffer")
    )

_______________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/golang.org/x/text/language**

k8s.io/kubernetes/vendor/golang.org/x/text/language/index.go

    var coreTags = map[uint32]uint16{
        0x0: 0,  // und
        0x00a00000: 3,  //af
        0x00a000d0: 4,  //af-NA
        ...
        0x2c60015e: 746, // zu-ZA
    }

k8s.io/kubernetes/vendor/golang.org/x/text/language/language.go

    var (
        errPrivateUse = errors.New("cannot set a key on a private use tag")
        errInvalidArguments = errors.New("invalid key or type")
    )
    var errNoTLD = errors.New("language: region is not a valid ccTLD")

    type Tag struct{
        lang langID
        region regionID
        script scriptID
        pVariant byte          // offset in str, includes preceding '-'
        pExt unit16             // offset of first extension, includes preceding '-'
        str string
    }
    const(
        All = BCP47 | Legacy | Macro
    )

k8s.io/kubernetes/vendor/golang.org/x/text/language/lookup.go

    // grandfatheredMap holds a mapping from legacy and grandfathered tags to their base language or index to more elaborate tag.
    var(
        grandfatheredMap = map[[maxLen]byte]int16 {
            [maxLen]byte{'a', 'r', 't', '-', 'l', 'o', 'j', 'b', 'a', 'n'}: _jbo,  // art-lojban
            ...
        }
        altTagIndex = [...]uint8{0,17,31,45,61,74,86,102}
        altTags = "xtg-x-cel-gaulishen-GB-oxendicten-x-i-defaultund-x-i-enochiansee-x-i-mingonan-x-zh-minen-US-u-va-posix"
    )
    const(
        maxAltTaglen = len("en-US-POSIX")
        maxLen = maxAltTaglen
    )
    type langID uint16
    type regionID uint16
    type scriptID uint8

k8s.io/kubernetes/vendor/golang.org/x/text/language/match.go

    var ErrMissingLikelyTagsData = errors.New("missing likely tags data")

k8s.io/kubernetes/vendor/golang.org/x/text/language/parse.go

    var errSyntax = errors.New("language: tag is not well-formed")
    var errInvalidArgument = errors.New("invalid Extension or Variant")
    var errInvalidWeight = errors.New("ParseAcceptLanguage: invalid weight")
    var acceptFallback = map[string]langID{
        "english": _en,
        "deutsch": _de,
        "italian": _it,
        "french": _fr,
        "*": _mul,
    }

    func Parse(s string) (t Tag, err error){
        return Default.Parse(s)
    }

k8s.io/kubernetes/vendor/golang.org/x/text/language/tables.go

    // Size: 1444 bytes
    var variantIndex = map[string]uint8{
        "1606nict": 0x0,
        ...
    }
    var langAliasMap = [160]fromTo{
        0: {from: 0xc4, to: 0xda},
        1: {from: 0xff, to: 0xf8},
        ...
        159: {from: 0x473e, to: 0x2be}
    }
    type fromTo struct{
        from uint16
        to uint16
    }

k8s.io/kubernetes/vendor/golang.org/x/text/language/match.go

    var notEquivalent []langID
    // create a list of all languages for which canonicalization may alter the script or region
    func init(){
        // create a list of all languages for which canonicalization may alter the script or region.
        for _,lm:=range langAliasMap{
            tag:=Tag{lang: langID(lm.from)}
            if tag,_ = tag.canonicalize(All); tag.script != 0 || tag.region!=0{
                notEquivalent = append(notEquivalent, langID(lm.from))
            }
        }
    }

k8s.io/kubernetes/vendor/golang.org/x/text/language/tags.go

    func MustParse(s string) Tag{
        t,err:=Parse(s)
        if err!=nil{
            panic(err)
        }
        return t
    }

_________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/golang.org/x/text/unicode/norm**

k8s.io/kubernetes/vendor/golang.org/x/text/unicode/norm/tables.go

    var recompMap = map[uint32]rune{
        0x00410300: 0x00c0,
        ...
    }

k8s.io/kubernetes/vendor/golang.org/x/text/unicode/norm/transform.go

    var errs = []error{nil, transform.ErrShortDst, transform.ErrShortSrc}

k8s.io/kubernetes/vendor/golang.org/x/text/unicode/norm/trie.go

    var nfcSparse = sparseBlocks{
        values: nfcSparseValues[:],
        offset: nfcSparseOffset[:],
    }
    var nfkcSparse = sparseBlocks{
        values: nfkcSparseValues[:],
        offset: nfkcSparseOffset[:],
    }
    var (
        nfcData = newNfcTrie(0)
        nfkcData = newNfkcTrie(0)
    )
    type valueRange struct{
        value uint16    // header: value:stride
        lo, hi byte        // header: lo:n
    }
    type sparseBlocks struct{
        values []valuesRange
        offset []uint16
    }

k8s.io/kubernetes/vendor/golang.org/x/text/unicode/norm/tables.go

    type nfcTrie struct{}
    func newNfcTrie(i int) *nfcTrie{
        return &nfcTrie{}
    }
    type nfkcTrie struct{}
    func newNfkcTrie(i int) *nfcTrie{
        return &nfkcTrie{}
    }

    var nfcSparseOffset = []uint16{0x0, 0x5, 0x9, 0xb, 0xd, 0x18, ...}
    var nfcSparseValues = [688]valueRange{
        {value: 0x0000, lo: 0x04},
        ...
    }
    var nfkcSparseOffset = []uint16{....}
    var nfkcSparseValues = [875]valueRange{
        {value: 0x0002, lo: 0x0d},
        ...
    }

k8s.io/kubernetes/vendor/golang.org/x/text/unicode/norm/forminfo.go

    type lookupFunc func(b input, i int) Properties

    // formInfo holds Form-specific functions and tables.
    type formInfo struct{
        form Form
        composing, compatibility bool  // form type
        info lookupFunc
        nextMain iterFunc
    }
    var formTable []*formInfo
    func init(){
        formTable = make([]*formInfo, 4)
        for i:= range formTable{
            f:=&formInfo{}
            formTable[i]=f
            f.form = form(i)
            if Form(i)==NFKD || Form(i)==NFKC{
                f.compatibility=true
                f.info = lookupInfoNFKC
            }else{
                f.info = lookupInfoNFC
            }
            f.nextMain = nextDecomposed
            if Form(i)==NFC|| Form(i)==NFKC{
                f.nextMain=nextComposed
                f.composing=true
            }
        }
    }

k8s.io/kubernetes/vendor/golang.org/x/text/unicode/norm/input.go

    type input struct{
        str string
        bytes []byte
    }
    func (in *input) charinfoNFKC(p int) (uint16,int){
        if in.bytes == nil{
            return nfkcData.lookupString(in.str[p:])
        }
        return nfkcData.lookup(in.bytes[p:])
    }
    func (in *input) charinfoNFC(p int) (uint16, int){
        if in.bytes==nil{
            return nfcData.lookupString(in.str[p:])
        }
        return nfcData.lookup(in.bytes[p:])
    }

k8s.io/kubernetes/vendor/golang.org/x/text/unicode/norm/iter.go

    type Iter struct{
        rb reorderBuffer
        buf [maxByteBufferSize]byte
        info Properties
        next iterFunc
        asciiF iterFunc
        p int
        multiSeg []byte
    }
    type iterFunc func(*Iter) []byte

k8s.io/kubernetes/vendor/golang.org/x/text/unicode/norm/normalize.go

    type Form int
    const(
        NFC Form = iota
        NFD
        NFKC
        NFKD
    )

k8s.io/kubernetes/vendor/golang.org/x/text/unicode/norm/forminfo.go

    func lookupInfoNFC(b input, i int) Properties{
        v,sz:=b.charinfoNFC(i)
        return compInfo(v,sz)
    }
    func lookupInfoNFKC(b input, i int) Properties{
        v,sz:=b.charinfoNFKC(i)
        return compInfo(v,sz)
    }

______________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/golang.org/x/text/cases**

k8s.io/kubernetes/vendor/golang.org/x/text/cases/trieval.go

    var trie = newCaseTrie(0)
    var sparse = sparseBlocks{
        values: sparseValues[:],
        offsets: sparseOffsets[:],
    }

k8s.io/kubernetes/vendor/golang.org/x/text/cases/tables.go

    type caseTrie struct{}
    func newCaseTrie(i int) *caseTrie{
        return &caseTrie{}
    }
    var sparseOffsets = []uint16{0x0, 0x9, 0xf, 0x18, 0x24, 0x2e, 0x3a, 0x3d, 0x41, 0x44, 0x48, 0x52, 0x54, 0x59, 0x69, 0x70,.....}
    var sparseValues = [1362]valueRange{
        {value: 0x0004, lo: 0xa8, hi: 0xa8},
        ...
    }

k8s.io/kubernetes/vendor/golang.org/x/text/cases/map.go

    var(
        matcher language.Matcher
        Supported language.Coverage
        // We keep the following lists separate, instead of having a single per-language struct, to give the compiler a chance to remove unused code.
        upperFunc = []mapFunc{
            nil,
            nil,
            aztrUpper(upper),
            elUpper,
            ltUpper(upper),
            nil,
            aztrUpper(upper),
        }
    )
    titleInfos = []struct{
        title, lower mapFunc
        rewrite func(*context)
    }{
        {title, lower, nil},
        {title, lower, afnlRewrite},
        {aztrUpper(title), aztrLower, nil},
        {title, lower, nil},
        {ltUpper(title), ltLower, nil},
        {nlTitle, lower, afnlRewrite},
        {aztrUpper(title), aztrLower, nil},
    }
    func aztrUpper(f mapFunc) mapFunc{
        return func(c *context) bool {
            if c.src[c.pSrc] == 'i'{
                return c.writeString("i")
            }
            return f(c)
        }
    }

    const supported = "und af az el lt nl tr"
    func init(){
        tags:=[]language.Tag{}
        for _,s:=range strings.Split(supported, " "){
            tags = append(tags, language.MustParse(s))
        }
        matcher = language.NewMatcher(tags)
        Supported = language.NewCoverage(tags)
    }

________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/golang.org/x/text/width**

k8s.io/kubernetes/vendor/golang.org/x/text/width/width.go

    var trie = newWidthTrie(0)

k8s.io/kubernetes/vendor/golang.org/x/text/width/tables.go

    type widthTrie struct{}
    func newWidthTrie(i int) *widthTrie{
        return &widthTrie{}
    }

_______________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/golang.org/x/text/unicode/bidi**

k8s.io/kubernetes/vendor/golang.org/x/text/unicode/bidi/prop.go

    var trie = newBidiTrie(0)

    type Properties struct{
        entry uint8
        last uint8
     

k8s.io/kubernetes/vendor/golang.org/x/text/unicode/bidi/tables.go

    type bidiTrie struct{}
    func newBidiTrie(i int) *bidiTrie{
        return &bidiTrie{}
    }

k8s.io/kubernetes/vendor/golang.org/x/text/unicode/bidi/trieval.go

    var controlToClass = map[rune]Class{
        0x202D: LRO,  // LeftToRightOverride,
        0x202E: RLO,   // RightToLeftOverride,
        0x202A: LRE,   // LeftToRightEmbedding,
        0x202B: RLE,  // RightToLeftEmbedding,
        0x202C: PDF,  // PopDirectionalFormat,
        0x2066: LRI,   // LeftToRightIsolate,
        0x2067: RLI,   // RightToLeftIsolate,
        0x2068: FSI,   // FirstStrongIsolate,
        0x2069: PDI,  // PopDirectionalIsolate,
    }

_____________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/golang.org/x/text/secure/bidirule**

k8s.io/kubernetes/vendor/golang.org/x/text/secure/bidirule/bidirule.go

    // ErrInvalid indicates a label is invalid according to the Bidi Rule.
    var ErrInvalid = errors.New("bidirule: failed Bidi Rule")

    // Precomputing the ASCII values decreases running time for the ASCII fast path by about 30%
    var asciiTable [128]bidi.Properties
    func init(){
        for i:= range asciiTable{
            p,_:=bidi.LookupRune(rune(i))
            asciiTable[i] = p
        }
    }

    type Properties struct{
        entry uint8
        last uint8
    }
    func LookupRune(r rune) (p Properties, size int){
        var buf [4]byte
        n:=utf8.EncodeRune(buf[:], r)
        return Lookup(buf[:n])
    }

______________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/golang.org/x/text/secure/precis**

k8s.io/kubernetes/vendor/golang.org/x/text/secure/precis/context.go

    var errContext = errors.New("precis: contextual rule violated")

k8s.io/kubernetes/vendor/golang.org/x/text/secure/precis/options.go

    var(
        IgnoreCase Option = ignoreCase
        FoldWidth Option = foldWidth
        DisallowEmpty Option = disallowEmpty
        BidiRule Option = bidiRule
    )
    var(
        ignoreCase = func(o *options){
            o.ignorecase = true
        }
        foldWidth = func(o *options){
            o.foldWidth = true
        }
        disallowEmpty = func(o *options){
            o.disallowEmpty = true
        }
        bidiRule = func(o *options){
            o.bidiRule = true
        }
    )
    // An Option is used to define the behavior and rules of a Profile
    type Option func(*options)
    type options struct{
        foldWidth bool
        cases transform.Transformer
        disallow runes.Set
        norm norm.Form
        additional []func() transform.Transformer
        width *width.Transformer
        disallowEmpty bool
        bidiRule bool
        ignorecase bool
    }

k8s.io/kubernetes/vendor/golang.org/x/text/secure/precis/profile.go

    var(
        errDisallowedRune = errors.New("precis: disallowed rune encountered")
    )
    var dpTrie = newDerivedPropertiesTrie(0)

    var errEmptyString = errors.New("precis: transformation resulted in empty string")
    var(
        Nickname *Profile = nickname
        UsernameCaseMapped *Profile = usernameCaseMap
        UsernameCasePreserved *Profile = usernameNoCaseMap
        OpaqueString *Profile = opaquestring
    )
    var(
        nickname = NewFreeform(
            AdditionalMapping(func() transform.Transformer{
                return &nickAdditionalMapping{}
            }),
            IgnoreCase,
            Norm(norm.NFKC),
            DisallowEmpty,
        )
        usernameCaseMap = NewIdentifier(
            FoldWidth,
            FoldCase(),
            Norm(norm.NFC),
            BidiRule,
        )
        usernameNoCaseMap = NewIdentifier(
            FoldWidth,
            Norm(norm.NFC),
            BidiRule,
        )
        opaquestring = NewFreeform(
            AdditionalMapping(func() transform.Transformer{
                return runes.Map(func(r rune) rune{
                    if unicode.Is(unicode.Zs, r){
                        return ' '
                    }
                    return r
                })
            }),
            Norm(norm.NFC),
            DisallowEmpty,
        )
    )
    func NewIdentifiler(opts ...Option) *Profile{
        return &Profile{
            options: getOpts(opts...),
            class: identifier,
        }
    }
    func NewFreeform(opts ...Option) *Profile{
        return &Profile{
            options: getOpts(opts...),
            class: freeform,
        }
    }

k8s.io/kubernetes/vendor/golang.org/x/text/secure/precis/tables.go

    type derivedPropertiesTrie struct{}
    func newDerivedPropertiesTrie(i int) *derivedPropertiesTrie{
        return &derivedPropertiesTrie{}
    }

k8s.io/kubernetes/vendor/golang.org/x/text/secure/precis/context.go

    func init(){
        for i, ct:=range categoryTransitions{
            categoryTransitions[i].keep != permanent
            categoryTransitions[i].accept != ct.term
        }
    }
    var categoryTransitions = []struct{
        keep catBitmap
        set catBitmap
        term catBitmap
        accept catBitmap
        rule func(beforeBits catBitmap) (doLookahead bool, err error)
    }{
        joiningL: {set: bJoinStart},
        ...
    }
    type catBitmap uint16
    const(
        bJapanese catBitmap = 1<<iota
        bArabicIndicDigit
        bExtendedArabicIndicDigit
        bJoinStart
        bJoinMid
        bJoinEnd
        bVirama
        bLainSmallL
        bGreek
        bHebrew
        bMustHavaJapn
        permanent = bJapanses | bArabicIndicDigit | bExtendedArabicIndicDigit | bMustHaveJapn
    )

___________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/PuerkitoBio/purell**

k8s.io/kubernetes/vendor/github.com/PuerkitoBio/purell/purell.go

    var rxPort = regexp.MustCompile(`(:\d+)/?$`)
    var rxDirIndex = regexp.MustCompile(`(^|/)((?:default|index)\.\w{1,4})$`)
    var rxDupSlashes = regexp.MustCompile(`/{2,}`)
    var rxDWORDHost = regexp.MustCompile(`^(\d+)(?:\.+)?(?:\:\d*)?)$`)
    var rxOctalHost = regexp.MustCompile(`^(0\d*)\.(0\d*)\.(0\d*)\.(0\d*)((?:\.+)?(?:\:\d*)?)$`)
    var rxHexHost = regexp.MustCompile(`^0x([0-9A-Fa-f]+)((?:\.+)?(?:\:\d*)?)$`)
    var rxHostDots = regexp.MustCompile(`^(.+?)(:\d+)?$`)
    var rxEmptyPort = regexp.MustCompile(`:+$`)

    type NormalizationFlags int
    var flags=map[NormalizationFlags]func(*url.URL){
        FlagLowercaseScheme: lowercaseScheme,
        FlagLowercaseHost: lowercaseHost,
        FlagRemoveDefaultPort: removeDefaultPort,
        FlagRemoveDirectoryIndex: removeDirectoryIndex,
        FlagRemoveDotSegments: removeDotSegments,
        FlagRemoveFragment: removeFragment,
        FlagForceHTTP: forceHTTP,
        FlagRemoveDuplicateSlashes: removeDuplicateSlashes,
        FlagRemoveWWW: removeWWW,
        FlagAddWWW: addWWW,
        FlagSortQuery: sortQuery,
        FlagDecodeDWORDHost: decodeDWORDHost,
        FlagDecodeOctualHost: decodeOctalHost,
        FlagDecodeHexHost: decodeHexHost,
        FlagRemoveUnnecessaryHostDots: removeUnncessaryHostDots,
        FlagRemoveEmptyPortSeparator: removeEmptyPortSeparator,
        FlagRemoveTrailingSlash: removeTrailingSlash,
        FlagAddTrailingSlash: addTrailingSlash,
    }

    type URL struct{
        Scheme string
        Opaque string
        User *UserInfo
        Host string
        Path string
        PawPath string
        ForceQuery bool
        RawQuery string
        Fragment string
    }
    func lowercaseScheme(u *url.URL){
        if len(u.Scheme)>0{
            u.Scheme = strings.ToLower(u.Scheme)
        }
    }
    func lowercaseHost(u *url.URL){
        if len(u.Host) >0{
            u.Host = strings.ToLower(u.Host)
        }
    }
    const(
        defaultHttpPort = ":80"
        defaultHttpsPort = ":443"
    )
    func removeDefaultPort(u *url.URL){
        if len(u.Host)>0{
            scheme:=strings.ToLower(u.Scheme)
            u.Host = rxPort.ReplaceAllStringFunc(u.Host, func(val string) string{
                    
            }
        }
    }
    func removeDirectoryIndex(u *url.URL){
        if len(u.Path)>0{
            u.Path = rxDirIndex.ReplaceAllString(u.Path, "$1")
        }
    }

_______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/go-openapi/spec**

k8s.io/kubernetes/vendor/github.com/go-openapi/spec/bindata.go

    // _bindata is a table, holding each asset generator, mapped to its name.
    var _bindata = map[string]func()(*asset, error){
        "jsonschema-draft-04.json": jsonschemaDraft04JSON,
        "v2/schema.json": v2SchemaJSON,
    }
    type asset struct{
        bytes []byte
        info os.FileInfo
    }
    func jsonschemaDraft04JSON()(*asset, error){
        bytes, err:=jsonschemaDraft04JSONBytes()
        if err!=nil{
            return nil, err
        }
        info:=bindataFileInfo{name: "jsonschema-draft-04.json", size:4375, mode:os.FileMode(420), modTime: time.Unix(1441640690,0)}
        a:=&asset{bytes:bytes, info: info}
        return a,nil
    }

    type bintree struct{
        Func func() (*asset, error)
        Children map[string]*bintree
    }
    var _bintree = &bintree{nil, map[string]*bintree{
        "jsonschema-draft-04.json": &bintree{jsonschemaDraft04JSON, map[string]*bintree{}},
        "v2": &bintree{nil, map[string]*bintree{
            "schema.json": &bintree{v2SchemaJSON, map[string]*bintree{}},
        }},
    }}

k8s.io/kubernetes/vendor/github.com/go-openapi/spec/expander.go

    var resCache = initResolutionCache()
    func initResolutionCache() ResolutionCache{
        return &simpleCache{store: map[string]interface{}{
            "http://swagger.io/v2/schema.json": MustLoadSwagger20Schema(),
            "http://json-schema.org/draft-04/schema": MustLoadJSONSchemaDraft04(),
        }}
    }
    // ResolutionCache a cache for resolving urls
    type ResolutionCache interface{
        Get(string) (interface{}, bool)
        Set(string, interface{})
    }
    type simpleCache struct{
        lock sync.Mutex
        store map[string]interface{}
    }
    func (s *simpleCache) Get(url string)(interface{}, bool){
        s.lock.Lock()
        v,ok:=s.store[uri]
        s.lock.Unlock()
        return v,ok
    }
    func (s *simpleCache)Set(uri string, data interface{}){
        s.lock.Lock()
        s.store[uri] = data
        s.lock.Unlock()
    }


    var idPtr,_ = jsonpointer.New("/id")
    var schemaPtr,_=jsonpointer.New("/$schema")
    var refPtr,_=jsonpointer.New("/$ref")
    //New creates a new json pointer for the given string
    func New(jsonPointerString string)(Pointer, error){
        var p Pointer
        err:=p.parse(jsonPointerString)
        return p,err
    }
    type Pointer struct{
        referenceTokens []string
    }
    func (p *Pointer)parse(jsonPointerString string) error{
        var err error
        if jsonPointerString != emptyPointer{
            if !strings.HasPrefix(jsonPointerString, pointerSeparator){
                err=errors.New(invalidStart)
            }else{
                referneceTokens := strings.Split(jsonPointerString, pointerSeparator)
                for _,referenceToken:=range referenceTokens[1:]{
                    p.referenceTokens = append(p.referenceTokens, referenceToken)
                }
            }
        }
        return err
    }
    const(
        emptyPonter = ``
        pointerSeparator = `/`
        invalidStart = `JSON pointer must be empty or start with a "` + pionterSeparator
    )

k8s.io/kubernetes/vendor/github.com/go-openapi/spec/spec.go

    var(
        jsonSchema = MustLoadJSONSchemaDraft04()
        swaggerSchema = MustLoadSwagger20Schema()
    )
    func MustLoadJSONSchemaDraft04() *Schema{
        d,e:=JSONSchemaDraft04()
        if e!=nil{
            panic(e)
        }
        return d
    }
    func JSONSchemaDraft04()(*Schema, error){
        b,err:=Asset("jsonschema-draft-04.json")
        if err!=nil{
            return nil,err
        }
        schema:=new(Schema)
        if err:=json.Unmarshal(b,schema);err!=nil{
            return nil,err
        }
        return schema,nil
    }
    func MustLoadSwagger20Schema()*Schema{
        d,e:=Swagger20Schema()
        if e!=nil{
            pinic(e)
        }
        return d
    }
    func Swagger20Schema()(*Schema,error){
        b,err:=Asset("v2/schema.json")
        if err!=nil{
            return nil,err
        }
        schema:=new(Schame)
        if err:=json.Unmarshal(b,schema);err!=nil{
            return nil,err
        }
        return schema,nil
    }

    func Asset(name string)([]byte, error){
        cannonicalName := strings.Replace(name, "\\", "/", -1)
        if f,ok:=_bindata[cannonicalName];ok{
            a,err:=f()
            if err!= nil{
                return nil, fmt.Errorf("Asset %s can't read by error: %v", name, err)
            }
            return a.bytes, nil
        }
        return nil, fmt.Errorf("Asset %s not found", name)
    }

k8s.io/kubernetes/vendor/github.com/go-openapi/spec/schema.go

    //Schema the schema object allows the definition of input and output data types.
    //These types can be objects, but also primitives and arrays.
    //This object is based on the [JSON Schema Specification Draft 4](http://json-schema.org/)
    //and uses a predefined subset of it.
    //On top of this subset, there are extensions provided by this specification to allow for more complete documentation.
    type Schema struct{
        VendorExtensible
        SchemaProps
        SwaggerSchemaProps
        ExtraProps map[string]interface{} `json:"-"`
    }
    type VendorExtensible struct{
        Extensions Extensions
    }
    type Extensions map[string]interface{}

    type SchemaProps struct{
        ID string `json:"id,omitempty"`
        Ref Ref `json:"-,omitempty"`
        Schema SchemaURL `json:"-,omitempty"`
        Description string `json:"description,omitempty"`
        Type StringOrArray `json:"type,omitempty"`
        Format string `json:"format,omitempty"`
        Title string `json:"title,omitempty"`
        Default interface{} `json:"default,omitempty"`
        Maximum *float64 `json:"maximum,omitempty"`
        ExclusiveMaximum bool `json:"exclusiveMaximum,omitempty"`
        Minimum *float64 `json:"minimum,omitempty"`
        ExclusiveMinimum bool `json:"exclusiveMinimum,omitempty"`
        MaxLength *int64 `json:"maxLength,omitempty"`
        MinLength *int64 `json:"minLength,omitempty"`
        Pattern string `json:"pattern,omitempty"`
        MaxItems *int64 `json:"maxItems,omitempty"`
        MinItems *int64 `json:"minItems,omitempty"`
        UniqueItems bool `json:"uniqueItems,omitempty"`
        MultipleOf *float64 `json:"multipleOf,omitempty"`
        Enum []interface{} `json:"enum,omitempty"`
        MaxProperties *int64 `json:"maxProperties,omitempty"`
        MinProperties *int64 `json:"minProperties,omitempty"`
        Required []string `json:"required,omitempty"`
        Items *SchemaOrArray `json:"items,omitempty"`
        AllOf []Schema `json:"allOf,omitempty"`
        OneOf []Schema `json:"oneOf,omitempty"`
        AnyOf []Schema `json:"anyOf,omitempty"`
        Not *Schema `json:"not,omitempty"`
        Properties map[string]Schema `json:"properties,omitempty"`
        AdditionalProperties *SchemaOrBool `json:"additionalProperties,omitempty"`
        PatternProperties map[string]Schema `json:"patternProperties,omitempty"`
        Dependencies Dependencies `json:"dependencies,omitempty"`
        AdditionalItems *SchemaOrBool `json:"additionalItems,omitempty"`
        Definitions Definition `json:"definitions,omitempty"`
    }
    type SchemaURL string
    type StringOrArray []string
    type SchemaOrArray struct{
        Schema *Schema
        Schemas []Schema
    }
    type Dependencies map[string]SchemaOrStringArray
    type SchemaOrStringArray struct{
        Schema *Schema
        Property []string
    }
    type SchemaOrBool struct{
        Allows bool
        Schema *Schema
    }
    type Definitions map[string]Schema

_______________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/go-openapi/log**

k8s.io/kubernetes/vendor/github.com/go-openapi/log/log.go

    type StdLogger interface{
        Print(v ...interface{})
        Printf(format string, v ...interface{})
    }
    func init(){
        // default Logger
        SetLogger(stdlog.New(os.Stderr, "[restful]", stdlog.LstdFlags|Lshortfile))
    }
    var Logger StdLogger
    // SetLogger sets the logger for this package
    func SetLogger(customLogger StdLogger){
        Logger = customLogger
    }
    func Print(v ...interfact{}){
        Logger.Print(v...)
    }
    func Printf(format string, v ...interface{}){
        Logger.Printf(format, v...)
    }

    type Logger struct{
        mu sync.Mutex
        prefix string
        flag int
        out io.Writer
        buf []byte
    }
    const(
        Ldate = 1<<iota   // the date in the local time zone: 2009/01/23
        Ltime                    // the time in the local time zone: 01:23:23
        Lmicroseconds
        Lloongfile             // final file name and line number: /a/b/c/d.go:23
        Lshortfile              // final file name element and line number: d.go:23. overrides Llongfile
        LUTC
        LstdFlags = Ldate | Ltime
    )

_________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/emicklei/go-restful**

k8s.io/kubernetes/vendor/github.com/emicklei/go-restful/entity_accessors.go

    type EntityReaderWriter interface{
        Read(req *Request, v interface{})error
        Write(resp *Response, status int, v interface{})error
    }
    // entityAccessRegistry is a singleton
    var entityAccessRegistry = &entityReaderWriters{
        protection: new(sync.RWMutex),
        accessors: map[string]EntityReaderWriter{},
    }
    //entityReaderWriters associates MIME to an EntityReaderWriter
    type entityReaderWriters struct{
        protection *sync.RWMutex
        accessors map[string]EntityReaderWriter
    }

k8s.io/kubernetes/vendor/github.com/emicklei/go-restful/compressors.go

    func init(){
        currentCompressorProvider = NewSyncPoolCompessors()
    }

k8s.io/kubernetes/vendor/github.com/emicklei/go-restful/compressor_pools.go

    func NewSyncPoolCompessors() *SyncPoolCompessors{
        return &SyncPoolCompessors{
            GzipWriterPool: &sync.Pool{
                New: func() interface{}{return newGzipWriter()},
            },
            GzipReaderPool: &sync.Pool{
                New: func() interface{}{return newGzipReader()},
            },
            ZlibWriterPool: &sync.Pool{
                New: func() interface{}{return newZlibWriter()},
            },
        }
    }
    type SyncPoolCompessors struct{
        GzipWriterPool *sync.Pool
        GzipReaderPool *sync.Pool
        ZlibWriterPool *sync.Pool
    }
    func newGzipWriter() *gzip.Writer{
        writer,err:=gzip.NewWriterLevel(new(bytes.Buffer), gzip.BestSpeed)
        if err!=nil{
            panic(err.Error())
        }
        return writer
    }

k8s.io/kubernetes/vendor/github.com/emicklei/go-restful/entity_accessors.go

    func init(){
        RegisterEntityAccessor(MIME_JSON, NewEntityAccessorJSON(MIME_JSON))
        RegisterEntityAccessor(MIME_XML, NewEntityAccessorXML(MIME_XML))
    }
    func RegisterEntityAccessor(mine string, erw EntityReaderWriter){
        entityAccessRegistry.protection.Lock()
        defer entityAccessRegistry.protection.Unlock()
        entityAccessRegistry.accessors[mime] = erw
    }
    func NewEntityAccessorJSON(contentType string) EntityReaderWriter{
        return entityJSONAccess{ContentType: contentType}
    }
    func NewEntityAccessorXML(contentType string) EntityReaderWriter{
        return entityXMLAccess{ContentType: contentType}
    }
    type entityJSONAccess struct{
        ContentType string
    }

k8s.io/kubernetes/vendor/github.com/emicklei/go-restful/constants.go

    const(
        MIME_XML = "application/xml"
        MIME_JSON = "application/json"
    )

k8s.io/kubernetes/vendor/github.com/emicklei/go-restful/logger.go

    var traceLogger log.StdLogger
    func init(){
        traceLogger = log.Logger
    }

k8s.io/kubernetes/vendor/github.com/emicklei/go-restful/web_service_container.go

    var DefaultContainer *Container
    func init(){
        DefaultContainer = NewContainer()
        DefaultContainer.ServeMux = http.DefaultServeMux
    }

k8s.io/kubernetes/vendor/github.com/emicklei/go-restful/container.go

    type Container struct{
        webServicesLock sync.RWMutex
        webServices []*WebService
        ServeMux *http.ServeMux
        isRegisteredOnRoot bool
        containerFilters []FilterFunction
        doNotRecover bool
        recoverHandlerFunc RecoverHandleFunction
        serviceErrorHandleFunc ServiceErrorHandleFunction
        router RouteSelector
        contentEncodingEnabled bool
    }
    func NewContainer() *Container {
        return &Container{
            webServices: []*WebService{},
            ServeMux: http.NewServeMux(),
            isRegisteredOnRoot: false,
            containerFilters: []FilterFunction{},
            doNotRecover: true,
            recoverHandleFunc: logStackOnRecover,
            serviceErrorHandleFunc: writeServiceError,
            router: CurlyRouter{},
            contentEncodingEnabled: false,
        }
    }
    var DefaultServeMux = &defaultServeMux
    var defaultServeMux ServeMux
    type ServeMux struct{
        mu sync.RWMutex
        m map[string]muxEntry
        hosts bool
    }
    type muxEntry struct{
        explicit bool
        h Handler
        pattern string
    }

______________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/api/resource**

k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/api/resource/math.go

    var(
        bigTen=big.NewInt(10)
        bigZero=big.NewInt(10)
        bigOne=big.NewInt(1)
        bigThousand=big.NewInt(1000)
        big1024=big.NewInt(1024)
        decZero=inf.NewDec(0,0)
        decOne=inf.NewDec(1,0)
        decMinusOne=inf.NewDec(-1,0)
        decThousand=inf.NewDec(1000,0)
        dec1024=inf.NewDec(1024,0)
        decMinus1024=inf.NewDec(-1024,0)
        maxAllowed=infDecAmount{inf.NewDec((1<<63),0)}
        MaxMilliValue=int64(((1<<63)-1)/1000)
    )

k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/api/resource/quantity.go

    const(
        splitREString = "^([+-]?[0-9.]+)([eEinumkKMGTP]*[-+]?[0-9]*)$"
    )
    var(
        splitRE=regexp.MustCompile(splitREString)
        ErrFormatWrong=errors.New("quantities must match the regular expression '"+splitREString+"'")
        ErrNumeric=errors.New("unable to parse numeric part of quantity")
        ErrSuffix=errors.New("unable to parse quantity's suffix")
    )

k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/api/resource/quantity_proto.go

    var(
        ErrInvalidLengthGenerated=fmt.Errorf("proto: negative length found during unmarshaling")
        ErrIntOverflowGenerated=fmt.Errorf("proto: integer overflow")
    )

k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/api/resource/scale_int.go

    var(
        intPool sync.Pool
        maxInt64 = big.NewInt(math.MaxInt64)
    )

k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/api/resource/suffix.go

    var quantitySuffixer=newSuffixer()
    type suffixHandler struct{
        decSuffixes listSuffixer
        binSuffixes listSuffixer
    }
    type listSuffixer struct{
        suffixToBE map[suffix]bePair
        beToSuffix map[bePair]suffix
        beToSuffixBytes map[bePair][]byte
    }
    type bePair struct{
        base, exponent int32
    }
    type fastLookup struct{
        *suffixHandler
    }
    type suffix string
    type suffixer struct{
        interpret(suffix) (base, exponent int32, fmt Format, ok bool)
        construct(base, exponent int32, fmt Format)(s suffix, ok bool)
        constructBytes(base, exponent int32, fmt Format)(s []byte, ok bool)
    }
    func newSuffixer() suffixer{
        sh:=&suffixHandler{}
        sh.binSuffixes.addSuffix("Ki",bePair{2,10})
        sh.binSuffixes.addSuffix("Mi",bePair{2,20})
        sh.binSuffixes.addSuffix("Gi",bePair{2,30})
        sh.binSuffixes.addSuffix("Ti",bePair{2,40})
        sh.binSuffixes.addSuffix("Pi",bePair{2,50})
        sh.binSuffixes.addSuffix("Ei",bePair{2,60})
        sh.binSuffixes.addSuffix("",bePair{2,0})
        sh.binSuffixes.addSuffix("n",bePair{10,-9})
        sh.binSuffixes.addSuffix("u",bePair{10,-6})
        sh.binSuffixes.addSuffix("m",bePair{10,-3})
        sh.binSuffixes.addSuffix("",bePair{10,0})
        sh.binSuffixes.addSuffix("k",bePair{10,3})
        sh.binSuffixes.addSuffix("M",bePair{10,6})
        sh.binSuffixes.addSuffix("G",bePair{10,9})
        sh.binSuffixes.addSuffix("T",bePair{10,12})
        sh.binSuffixes.addSuffix("P",bePair{10,15})
        sh.binSuffixes.addSuffix("E",bePair{10,18})
        return fastLookup{sh}
    }

k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/api/resource/generated.pb.go

    func (m *Quantity) Reset() {*m = Quantity{}}
    func (*Quantity)ProtoMessage() {}
    func (*Quantity) Descriptor() ([]byte, []int) {return fileDescriptorGenerated, []int{0}}
    func init(){
        proto.RegisterType((*Quantity)(nil), "k8s.io.apimachinery.pkg.api.resource.Quantity")
    }
    func init(){
        proto.RegisterFile("k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/api/resource/generated.proto", fileDescriptorGenerated)
    }
    var fileDescriptorGenerated = []byte{
        0x1f, 0x8b,...
    }

k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/api/resource/quantity.go

    type Quantity struct {
        i int64Amount
        d infDecAmount
        s string
        Format
    }

k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/api/resource/scale_int.go

    func init(){
        intPool.New = func() interface{}{
            return &big.Int{}
        }
    }

_______________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/util/validation**

k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/util/validation/validation.go

    var qualifiedNameRegexp = regexp.MustCompile("^" + qualifiedNameFmt + "$")

    const qnameCharFmt string = "[A-Za-z0-9]"
    const qnameExtCharFmt string = "[-A-Za-z0-9_.]"
    const qualifiedNameFmt string = "(" + qnameCharFmt + qnameExtCharFmt + "*)?" + qnameCharFmt

    const labelValueFmt string = "(" + qualifiedNameFmt = ")?"
    var labelValueRegexp = regexp.MustCompile("^" + labelValueFmt + "$")

    const dns1123LabelFmt string = "[a-z0-9]([-a-z0-9]*[a-z0-9]?)"
    var dns1123LabelRegexp = regexp.MustCompile("^" + dns1123LabelFmt + "$")
    const dns1123SubdomainFmt string = dns1123LabelFmt + "(\\." + dns1123LabelFmt + ")*"
    var dns1123SubdomainRegexp = regexp.MustCompile("^" + dns1123SubdomainFmt + "$")
    const dns1035LabelFmt string = "[a-z]([-a-z0-9]*[a-z0-9])?"
    var dns1035LabelRegexp = regexp.MustCompile("^" + dns1035LabelFmt + "$")

    const cIdentifierFmt string = "[A-Za-z_][A-Za-z0-9_]*"
    var cIdentifierRegexp = regexp.MustCompile("^" + cIdentifierFmt + "$")

    var portNameCharsetRegex = regexp.MustCompile("^[-a-z0-9]+$")
    var portNameOneLetterRegexp = regexp.MustCompile("[a-z]")

    const percentFmt string = "[0-9]+%"
    var percentRegexp = regexp.MustCompile("^" + percentFmt + "$")

    const httpHeaderNameFmt string = "[-A-Za-z0-9]+"
    var httpHeaderNameRegexp = regexp.MustCompile("^" + httpHeaderNameFmt + "$")

    const configMapKeyFmt = `[-._a-zA-Z0-9]+`
    var configMapKeyRegexp = regexp.MustCompile("^" + configMapKeyFmt + "$")

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/labels**

k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/labels/selector.go

    // Token represents constant definition for lexer token
    type Token int
    const(
        ErrorToken Token = iota            // ErrorToken represents scan error
        EndOfStringToken                   // EndOfStringToken represents end of string
        ClosedParToken                     // ClosedParToken represents close parenthesis
        CommaToken                         // CommaToken represents the comma
        DoesNotExistToken                  // DoesNotExistToken represents logic not
        DoubleEqualsToken                  // DoubleEqualsToken represents double equals
        EqualsToken                        // EqualsToken represents equal
        GreaterThanToken                   // GreaterThanToken represents greater then
        IdentifierToken                    // IdentifierToken represents identifier, e.g. keys and values
        InToken                            // InToken represents in
        LessThanToken                      // LessThanToken represents less than
        NotEqualsToken                     // NotEqualsToken represents not equal
        NotInToken                         // NotInToken represents not in
        OpenParToken                       // OpenParToken represents open parenthesis
    )
    var string2token = map[string]Token{
        ")": ClosedParToken,
        ",": CommaToken,
        "!": DoesNotExistToken,
        "==": DoubleEqualsToken,
        "=": EqualsToken,
        ">": GreaterThanToken,
        "in": InToken,
        "<": LessThanToken,
        "!=": NotEqualsToken,
        "notin": NotInToken,
        "(": OpenParToken,
    }

_________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/google/gofuzz**

k8s.io/kubernetes/vendor/github.com/google/gofuzz/fuzz.go

    func fuzzInt(v reflect.Value, r *rand.Rand){
        v.SetInt(int64(randUint64(r)))
    }
    var fillFuncMap = map[reflect.Kind]func(reflect.Value, *rand.Rand){
        reflect.Bool: func(v reflect.Value, r *rand.Rand){
            v.SetBool(randBool(r))
        },
        reflect.Int: fuzzInt,
        reflect.Int8: fuzzInt,
        reflect.Int16: fuzzInt,
        reflect.Int32: fuzzInt,
        reflect.Int64: fuzzInt,
        reflect.Uint: fuzzInt,
        reflect.Uint8: fuzzInt,
        reflect.Uint16: fuzzInt,
        reflect.Uint32: fuzzInt,
        reflect.Uint64: fuzzInt,
        reflect.Uintptr: fuzzInt,
        reflect.Float32: func(v reflect.Value, r *rand.Rand){
            v.SetFloat(float64(r.Float32))
        },
        reflect.Float64: func(v reflect.Value, r *rand.Rand){
            v.SetFloat(r.Float64))
        },
        reflect.Complex64: func(v reflect.Value, r *rand.Rand){
            panic("unimplemented")
        },
        reflect.Complex128: func(v reflect.Value, r *rand.Rand){
            panic("unimplemented")
        },
        reflect.String: func(v reflect.Value, r *rand.Rand){
            v.SetString(randString(r))
        },
        reflect.UnsafePointer: func(v reflect.Value, r *rand.Rand){
            panic("unimplemented")
        },
    }

    // randBool returns true or false randomly
    func randBool(r *rand.Rand) bool{
        if r.Int()&1 == 1{
            return true
        }
        return false
    }
    //Float32 returns, as a float32, a pseudo-random number in [0.0, 1.0)
    func (r *Rand) Float32() float32{
    again:
        f:=float32(r.Float64())
        if f==1{
            goto again
        }
        return f
    }
    //randString makes a random string up to 20 characters long. The returned string may include a variety of (valid) UTF-8 encodings.
    func randString(r *rand.Rand) string{
        n:=r.Intn(20)
        runes:=make([]rune, n)
        for i:=range runes{
            runes[i] = unicodeRanges[r.Intn(len(unicodeRanges))].choose(r)
        }
        return string(runes)
    }

_____________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/util/intstr**

k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/util/intstr/generated.pb.go

    var(
        ErrInvalidLengthGenerated = fmt.Errorf("proto: negative length found during unmarshaling")
        ErrIntOverflowGenerated = fmt.Errorf("proto: integer overflow")
    )

    func (m *IntOrString) Reset() {*m=IntOrString{}}
    func (*IntOrString) ProtoMessage() {}
    func (*IntOrString) Descriptor() ([]byte, []int) {return fileDescriptorGenerated, []int{0}}
    func init(){
        proto.RegisterType((*IntOrString)(nil), "k8s.io.apimachinery.pkg.util.intstr.IntOrString")
    }

    func init(){
        proto.RegisterFile("k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/util/intstr/generated.proto", fileDescriptorGenerated)
    }
    var fileDescriptorGenerated = []byte{
        0x1f, 0x8b,....
    }

k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/util/intstr/intstr.go

    type IntOrString struct{
        Type Type `protobuf:"varint,1,opt,name=type,casttype=Type"`
        IntVal int32 `protobuf:"varint,2,opt,name=intVal"`
        StrVal string `protobuf:"bytes,3,opt,name=strVal"`
    }
    // Type represents the stored type of IntOrString
    type Type int

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/golang.org/x/net/http2/hpack**

k8s.io/kubernetes/vendor/golang.org/x/net/http2/hpack/hpack.go

     var ErrStringLength = errors.New("hpack: string too long")
     var errNeedMore = errors.New("need more data")
     var errVarintOverflow = DecodingError{errors.New("varint integer overflow")}

k8s.io/kubernetes/vendor/golang.org/x/net/http2/hpack/huffman.go

     var bufPool = sync.Pool{
          New: func() interface{}{return new(bytes.Buffer)},
     }
     var ErrInvalidHuffman = errors.New("hpack: invalid Huffman-encoded data")
     var rootHuffmanNode = newInternalNode()
     func newInternalNode() *node{
          return &node{children: make([]*node, 256)}
     }
     type node struct{
          children []*node     // children is non-nil for internal nodes
          // The following are only valid if chidren is nil.
          codeLen uint8       // number of bits that led to the output of sym
          sym byte               // output symbol
     }

     type HeaderField struct{
          Name, Value string
          Sensitive bool
     }

k8s.io/kubernetes/vendor/golang.org/x/net/http2/hpack/tables.go

     func pair(name, value string) HeaderField{
          return HeaderField{Name: name, Value: value}
     }
     var staticTable = [...]HeaderField{
          pair(":authority", ""),
          pair(":method", "GET"),
          pair(":method", "POST"),
          pair(":path", "/"),
          pair(":path", "/index.html"),
          pair(":scheme", "http"),
          pair(":scheme", "https"),
          pair(":status", "200"),
          pair(":status", "204"),
          pair(":status", "206"),
          pair(":status", "304"),
          pair(":status", "400"),
          pair(":status", "404"),
          pair(":status", "500"),
          pair("accept-charset", ""),
          pair("accept-encoding", "gzip, deflate"),
          pair("accept-language", ""),
          pair("accept-ranges", ""),
          pair("accept", ""),
          pair("access-control-allow-origin", ""),
          pair("age", ""),
          pair("allow", ""),
          pair("authorization", ""),
          pair("cache-control", ""),
          pair("content-disposition", ""),
          pair("content-encoding", ""),
          pair("content-language", ""),
          pair("content-length", ""),
          pair("content-location", ""),
          pair("content-range", ""),
          pair("content-type", ""),
          pair("cookie", ""),
          pair("date", ""),
          pair("etag", ""),
          pair("expect", ""),
          pair("expires", ""),
          pair("from", ""),
          pair("host", ""),
          pair("if-match", ""),
          pair("if-modified-since", ""),
          pair("if-none-match", ""),
          pair("if-range", ""),
          pair("if-unmodified-since", ""),
          pair("last-modified", ""),
          pair("link", ""),
          pair("location", ""),
          pair("max-forwards", ""),
          pair("proxy-authenticate", ""),
          pair("proxy-authorization", ""),
          pair("range", ""),
          pair("referer", ""),
          pair("refresh", ""),
          pair("retry-after", ""),
          pair("server", ""),
          pair("set-cookie", ""),
          pair("strict-transport-security", ""),
          pair("transfer-encoding", ""),
          pair("user-agent", ""),
          pair("vary", ""),
          pair("via", ""),
          pair("www-authenticate", ""),
     }
     var huffmanCodes = [256]uint32{
          0x1ff8,
          ...
     }
     var huffmanCodeLen = [256]uint8{
          13,23,...
     }

k8s.io/kubernetes/vendor/golang.org/x/net/http2/hpack/huffman.go

     func init(){
          if len(huffmanCodes)!=256{
               panic("unexpected size")
          }
          for i,code:=range huffmanCodes{
               addDecoderNode(byte(i), code, huffmanCodeLen[i])
          }
     }
     func addDecoderNode(sym byte, code uint32, codeLen uint8){
          cur := rootHuffmanNode
          for codeLen > 8{
               codeLen -= 8
               i:=uint8(code >. codeLen)
               if cur.children[i] == nil{
                    cur.children[i]=newInternalNode()
               }
               cur = cur.children[i]
          }
          shift:=8-codeLen
          start,end:= int(uint8(code<<shift)), int(1<<shift)
          for i:=start; i<start+end; i++{
               cur.children[i]=&node{sym: sym, codeLen: codeLen}
          }
     }

______________________________________________________________________________________________











## 参考文献
* [bytes.Buffer原理及使用](../Golang/bytes.Buffer.md)

_______________________________________________________________________
[[返回/reference/remote-debug.md]](./remote-debug.md) 