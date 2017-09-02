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
**初始化k8s.io/kubernetes/vendor/golang.org/x/net/http2**

k8s.io/kubernetes/vendor/golang.org/x/net/http2/errors.go

     // An ErrCode is an unsigned 32-bit error code as defined in the HTTP/2 spec.
     type ErrCode uint32
     const(
          ErrCodeNo  ErrCode = 0x0
          ErrCodeProtocol  ErrCode = 0x1
          ErrCodeInternal  ErrCode = 0x2
          ErrCodeFlowControl  ErrCode = 0x3
          ErrCodeSettingsTimeout  ErrCode = 0x4
          ErrCodeStreamClosed  ErrCode = 0x5
          ErrCodeFrameSize  ErrCode = 0x6
          ErrCodeRefusedStream ErrCode = 0x7
          ErrCodeCancel  ErrCode = 0x8
          ErrCodeCompression  ErrCode = 0x9
          ErrCodeConnect ErrCode = 0xa
          ErrCodeEnhanceYourClaim  ErrCode = 0xb
          ErrCodeInadequateSecurity  ErrCode = 0xc
          ErrCodeHTTP11Required  ErrCode = 0xd
     )
     var errCodeName = map[ErrCode]string{
          ErrCodeNo: "NO_ERROR",
          ErrCodeProtocol: "PROTOCOL_ERROR",
          ErrCodeInternal: "INTERNAL_ERROR",
          ErrCodeFlowControl: "FLOW_CONTROL_ERROR",
          ErrCodeSettingsTimeout: "SETTINGS_TIMEOUT",
          ErrCodeStreamClosed: "STREAM_CLOSED",
          ErrCodeFrameSize: "FRAME_SIZE_ERROR",
          ErrCodeRefusedStream: "REFUSED_STREAM",
          ErrCodeCancel: "CANCEL",
          ErrCodeCompression: "COPRESSION_ERROR",
          ErrCodeConnect: "CONNECT_ERROR",
          ErrCodeEnhanceYourClaim: "ENHANCE_YOUR_CALM",
          ErrCodeInadequateSecurity: "INADEQUATE_SECURITY",
          ErrCodeHTTP11Required: "HTTP_1_1_REQUIRED",
     }

     var(
          errMixPseudoHeaderTypes = errors.New("mix of request and response pseudo headers")
          errPseudoAfterRegular = errors.New("pseudo header field after regular")
     )

k8s.io/kubernetes/vendor/golang.org/x/net/http2/fixed_buffer.go

     var(
          errReadEmpty = errors.New("read from empty fixedBuffer")
          errWriteFull = errors.New("write on full fixedBuffer")
     )

k8s.io/kubernetes/vendor/golang.org/x/net/http2/frame.go

     var padZeros = make([]byte, 255)   // zeros for padding

     var frameName = map[FrameType]string{
          FrameData: "DATA",
          FrameHeaders: "HEADERS",
          FramePriority: "PRIORITY",
          FrameRSTStream: "RST_STREAM",
          FrameSettings: "SETTINGS",
          FramePushPromise: "PUSH_PROMISE",
          FramePing: "PING",
          FrameGoAway: "GOAWAY",
          FrameWindowUpdate: "WINDOW_UPDATE",
          FrameContinuation: "CONTINUATION",
     }
     type FrameType uint8
     const(
          FrameData FrameType = 0x0
          FrameHeaders FrameType = 0x1
          FramePriority FrameType = 0x2
          FrameRSTStream FrameType = 0x3
          FrameSettings FrameType = 0x4
          FramePushPromise FrameType = 0x5
          FramePing FrameType = 0x6
          FrameGoAway FrameType = 0x7
          FrameWindowUpdate FrameType = 0x8
          FrameContinuation FrameType = 0x9
     )

     var flagName = map[FrameType]map[Flags]string{
          FrameData:{
               FlagDataEndStream: "END_STREAM",
               FlagDataPadded: "PADDED",
          },
          FrameHeaders:{
               FlagHeadersEndStream: "END_STREAM",
               FlagHeadersEndHeaders: "END_HEADERS",
               FlagHeadersPadded: "PADDED",
               FlagHeadersPriority: "PRIORITY",
          },
          FrameSettings:{
               FlagSettingsAck: "ACK",
          },
          FramePing:{
               FlagPingAck: "ACK",
          },
          FrameContinuation:{
               FlagContinuationEndHeaders: "END_HEADESS",
          },
          FramePushPromise:{
               FlagPushPromiseEndHeaders: "END_HEADERS",
               FlagPushPromisePadded: "PADDED",
          },
     }
     type Flags uint8
     const(
          FlagDataEndStream Flags = 0x1
          FlagDataPadded Flags = 0x8
          FlagHeadersEndStream Flags = 0x1
          FlagHeadersEndHeaders Flags = 0x4
          FlagHeadersPadded Flags = 0x8
          FlagHeadersPriority Flags = 0x20
          FlagSettingsAck Flags = 0x1
          FlagPingAck Flags = 0x1
          FlagContinuationEndHeaders Flags = 0x4
          FlagPushPromiseEndHeaders Flags = 0x4
          FlagPushPromisePadded Flags = 0x8
     )

     var frameParsers = map[FrameType]frameParser{
          FrameData: parseDataFrame,
          FrameHeaders: parseHeadersFrame,
          FramePriority: parsePriorityFrame,
          FrameRSTStream: parseRSTStreamFrame,
          FrameSettings: parseSettingsFrame,
          FramePushPromise: parsePushPromise,
          FramePing: parsePingFrame,
          FrameGoAway: parseGoAwayFrame,
          FrameWindowUpdate: parseWindowUpdateFrame,
          FrameContinuation: parseContinuationFrame,
     }
     type frameParser func(fh FrameHeader, payload []byte) (Frame, error)

     // A FrameHeader is the 9 byte header of all HTTP/2 frames.
     type FrameHeader struct{
          valid bool
          Type FrameType
          Flags Flags
          Length uint32
          StreamID uint32
     }
     type Frame interface{
          Header() FrameHeader
          invalidate()
     }

     var ErrFrameTooLarge = errors.New("http2: frame too large")

     var(
          errStreamID = errors.New("invalid stream ID")
          errDepStreamID = errors.New("invalid dependent stream ID")
          errPadLength = errors.New("pad length too large")
     )

k8s.io/kubernetes/vendor/golang.org/x/net/http2/gotrack.go

     var DebugGoroutines = os.Getenv("DEBUG_HTTP2_GOROUTINES") == "1"

k8s.io/kubernetes/vendor/golang.org/x/net/http2/headermap.go

     var(
          commonLowerHeader = map[string]string{}  // Go-Canonical-Case -> lower-case
          commonCanonHeader = map[string]string{} // lower-case -> Go-Canonical-Case
     )

k8s.io/kubernetes/vendor/golang.org/x/net/http2/http2.go

     type SettingID uint16
     const(
          SettingHeaderTableSize SettingID = 0x1
          SettingEnablePush SettingID = 0x2
          SettingMaxConcurrentStreams SettingID = 0x3
          SettingInitialWindowSize SettingID = 0x4
          SettingMaxFrameSize SettingID = 0x5
          SettingMaxHeaderListSize SettingID = 0x6
     )
     var settingName = map[SettingID]string{
          SettingHeaderTableSize: "HEADER_TABLE_SIZE",
          SettingEnablePush: "ENABLE_PUSH",
          SettingMaxConcurrentStreams: "MAX_CONCURRENT_STREAMS",
          SettingInitialWindowSize: "INITIAL_WINDOW_SIZE",
          SettingMaxFrameSize: "MAX_FRAME_SIZE",
          SettingMaxHeaderListSize: "MAX_HEADER_LIST_SIZE",
     }

     var(
          errInvalidHeaderFieldName = errors.New("http2: invalid header field name")
          errInvalidHeaderFieldValue = errors.New("http2: invalid header field value")
     )
     var httpCodeStringCommon = map[int]string{}  // n-> strconv.Itoa(n)

k8s.io/kubernetes/vendor/golang.org/x/net/http2/pipe.go

     var errClosedPipeWrite = errors.New("write on closed buffer")

k8s.io/kubernetes/vendor/golang.org/x/net/http2/server.go

     var (
          errClientDisconnected = errors.New("client disconnected")
          errClosedBody = errors.New("body closed by handler")
          errHandlerComplete = errors.New("http2: request body closed due to handler exiting")
          errStreamClosed = errors.New("http2: stream closed")
     )
     var errHandlerPanicked = errors.New("http2: handler panicked")

     var reqBodyCache = make(chan []byte, 8)

     var(
          ErrRecursivePush = errors.New("http2: recursive push not allowed")
          ErrPushLimitReached = errors.New("http2: push would exceed peer's SETTINGS_MAX_CONCURRENT_STREAMS")
     )
     var badTrailer = map[string]bool{
          "Authorization": true,
          "Cache-Control": true,
          "Connection": true,
          "Content-Encoding": true,
          "Content-Length": true,
          "Content-Range": true,
          "Content-Type": true,
          "Expect": true,
          "Host": true,
          "Keep-Alive": true,
          "Max-Forwards": true,
          "Progma": true,
          "Proxy-Authenticate": true,
          "Proxy-Authorization": true,
          "Proxy-Connection": true,
          "Range": true,
          "Realm": true,
          "Te": true,
          "Trailer": true,
          "Transfer-Encoding": true,
          "Wwww-Authenticate": true,
     }

k8s.io/kubernetes/vendor/golang.org/x/net/http2/transport.go

     var errTransportVersion = errors.New("http2: ConfigureTransport is only supported starting at Go 1.6")
     var ErrNoCachedConn = errors.New("http2: no cached connection was available")
     var(
          errClientConnClosed = errors.New("http2: client conn is closed")
          errClientConnUnusable = errors.New("http2: client conn not usable")
          errClientConnGotGoAway = errors.New("http2: Transport received Servers graceful shutdown GOAWAY")
          errClientConnGotGoAwayAfterSomeReqBody = errors.New("http2: Tansport received Server's graceful shutdown GOAWAY; some request body already witten")
     )
     var errRequestCanceled = errors.New("net/http: request canceled")
     var(
          errStopReqBodyWrite = errors.New("http2: aborting request body write")
          errStopReqBodyWriteAndCancel = errors.New("http2: canceling request")
     )
     var errClosedResponseBody = errors.New("http2: response body closed")
     var errInvalidTrailers = errors.New("http2: invalid trailers")
     var(
           errResponseHeaderListSize = errors.New("http2: response header list larger than advertised limit")
           errPseudoTrailers = errors.New("http2: invalid pseudo header in trailers")    
     )

     var noBody io.ReadCloser = ioutil.NopCloser(bytes.NewReader(nil))
     func NopCloser(r io.Reader) io.ReadCloser{
          return nopCloser{r}
     }
     type nopCloser struct{
          io.Reader
     }

k8s.io/kubernetes/vendor/golang.org/x/net/http2/headermap.go

     func init(){
          for _,v:=range []string{
               "accept",
               "accept-charset",
               "accept-encoding",
               ...
          }{
               chk:=http.CanonicalHeaderKey(v)
               commonLowerHeader[chk]=v
               commonCanonHeader[v]=chk
          }
     }
     const(
          commonLowerHeader = map[string]string{}
          commonCanonHeader = map[string]string{}
     )
     
k8s.io/kubernetes/vendor/golang.org/x/net/http2/http2.go

     var(
          VerboseLogs bool
          logFrameWrites bool
          logFrameReads bool
          inTests bool
     )
     func init(){
          e:=os.Getenv("GODEBUG")
          if strings.Contains(e,"http2debug=1"){
               VerboseLogs = true
          }
          if strings.Contains(e, "http2debug=2"){
               VerboseLogs = true
               logFrameWrites = true
               logFrameReads = true
          }
     }

     func init(){
          for i:=100;i<=999;i++{
               if v:=http.StatusText(i); v!=""{
                    httpCodeStringCommon[i] = strconv.Itoa(i)
               }
          }
     }
     var httpCodeStringCommon = map[int]string{}
     func StatusText(code int)string{
          return statusText[code]
     }
     const(
          StatusContinue = 100
          StatusSwitchingProtocols = 101
          StatusProcessing = 102
          StatusOK = 200
          StatusCreated = 201
          StatusAccepted = 202
          StatusNonAuthoritativeInfo = 203
          StatusNoContent = 204
          StatusResetContent = 205
          StatusPartialContent = 206
          StatusMultiStatus = 207
          StatusAlreadyReported = 208
          StatusIMUsed = 226
          StatusMultipleChoices = 300
          StatusMovedPermanently = 301
          StatusFound = 302
          StatusSeeOther = 303
          StatusNotModified = 304
          StatusUseProxy = 305
          _ = 306
          StatusTemporaryRedirect = 307
          StatusPermanentRedirect = 308
          StatusBadRequest = 400
          StatusUnauthorized = 401
          ...
     )
     var statusText = map[int]string{
          StatusContinue: "Continue",
     }

___________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/util/net**

k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/util/net/http.go

     var defaultTransport = http.DefaultTransport.(*http.Transport)
     var defaultProxyFuncPointer = fmt.Sprintf("%p", http.ProxyFromEnvironment)

k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/util/net/port_split.go

     var validSchemes = sets.NewString("http", "https", "")

k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/util/sets/string.go

     type String map[string]Empty
     func NewString(items ..string)String{
          ss:=String{}
          ss.Insert(items...)
          return ss
     }
     func (s String) Insert(items ..string){
          for _,item:=range items{
               s[item] = Empty{}
          }
     }

k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/watch/util.go

     var ErrWatchClosed = errors.New("watch closed before Until timeout")

_________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1**

k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1/generated.pb.go

     var(
          ErrInvalidLengthGenerated = fmt.Errorf("proto: negative length found during unmarshaling")
          ErrIntOverflowGenerated = fmt.Errorf("proto: integer overflow")
     )

k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1/register.go

     // scheme is the registry for the common types that adhere to the meta v1 API spec.
     var scheme = runtime.NewScheme()
     // ParameterCodec knows about query parameters used with the meta v1 API spec.
     var ParameterCodec = runtime.NewParameterCodec(scheme)

k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/runtime/scheme.go

     //Function to convert a field selector to internal representation.
     type FieldLabelConversionFunc func(label, value string)(internalLabel, internalValue string, err error)

     //Scheme are not expected to change at runtime and are only threadsafe after registration is complete.
     type Scheme struct{
          gvkToType map[schema.GroupVersionKind]reflect.Type
          typeToGVK map[reflect.Type][]schema.GroupVersionKind
          unversionedTypes map[reflect.Type]schema.GroupVersionKind
          unversionedKinds map[string]reflect.Type
          fieldLabelConversionFuncs map[string]map[string]FieldLabelConversionFunc
          defaulterFuncs map[reflect.Type]func(interface{})
          converter *conversion.Converter
          cloner *conversion.Cloner
     }
     type FieldLabelConversionFunc func(label, value string)(internalLabel, internalValue string, err error)

     type GroupVersionKind struct{
          Group string
          Version string
          Kind string
     }
     // Converter knows how to convert one type to another
     type Converter struct{
          conversionFuncs ConversionFuncs
          generatedConversionFuncs ConversionFuncs
          genericConversions []GenericConversionFunc
          ignoredConversions map[typePair]struct{}
          structFieldDests map[typeNamePair][]typeNamePair
          structFieldSources map[typeNamePair][]typeNamePair
          inputFieldMappingFuncs map[reflect.Type]FieldMappingFunc
          inputDefaultFlags map[reflect.Type]FieldMatchingFlags
          Debug DebugLogger
          nameFunc func(t reflect.Type)string
     }
     type GenericConversionFunc func(a,b interface{}, scope Scope)(bool, error)
     type ConversionFuncs struct{
          fns map[typePair]reflect.Value
     }
     type typePair struct{
          source reflect.Type
          dest reflect.Type
     }
     type typeNamePair struct{
          fieldType reflect.Type
          fieldName string
     }
     //Cloner knows how to copy one type to another
     type Cloner struct{
          deepCopyFuncs map[reflect.Type]reflect.Value
          generatedDeepCopyFuncs map[reflect.Type]func(in interface{}, out interface{}, c *Cloner) error
     }
     //NewCloner creates a new Cloner object.
     func NewCloner() *Cloner{
          c:=&Cloner{
               deepCopyFuncs: map[reflect.Type]reflect.Value{},
               generatedDeepCopyFuncs: map[reflect.Type]func(in interface{}, out interface{}, c *Cloner) error{},
          }
          if err:=c.RegisterDeepCopyFunc(byteSliceDeepCopy); err!=nil{
               panic(err)
          }
          return c
     }
     func byteSliceDeepCopy(in *[]byte, out *[]byte, c *Cloner) error{
          if *in!=nil{
               *out = make([]byte, len(*in))
               copy(*out, *in)
          }else{
               *out = nil
          }
          return nil
     }
     func (c *Cloner) RegisterDeepCopyFunc(deepCopyFunc interface{}) error{
          fv := reflect.ValueOf(deepCopyFunc)
          ft:= fv.Type()
          if err:=verifyDeepCopyFunctionSignature(ft); err!=nil{
               return err
          }
          c.deepCopyFuncs[ft.In(0)]=fv
          return nil
     }
     func verifyDeepCopyFunctionSignature(ft reflect.Type)error{
          if kt.Kind() != reflect.Func{
               return fmt.Errorf("expected func, got: %v", ft)
          }
          if ft.NumIn()!=3{
               return fmt.Errorf("expected three 'in' params, got %v", ft)
          }
          if ft.Num.Out()!=1{
               return fmt.Errorf("expected one 'out' param, got %v", ft)
          }
          if ft.In(0).Kind()!=reflect.Ptr{
               return fmt.Errorf("expected pointer arg for 'in' param 0, got: %v", ft)
          }
          if ft.In(1) != ft.In(0){
               return fmt.Errorf("expected 'in' param 0 the same as param 1, got: %v", ft)
          }
          var forClonerType Cloner
          if expected:=reflect.TypeOf(&forClonerType): ft.In(2)!=expected{
               return fmt.Errorf("expected '%v' arg for 'in' param 2, got: '%v'", expected, ft.In(2))
          }
          var forErrorType error
          errorType:=reflect.TypeOf(&forErrorType).Elem()
          if ft.Out(0)!=errorType{
               return fmt.Errorf("expected error return, got: %v", ft)
          }
          retur nil
     }

     //NewScheme creates a new Scheme. This scheme is pluggable by default.
     func NewScheme() *Scheme{
          s:= &Scheme{
               gvkToType: map[schema.GroupVersionKind]reflect.Type{},
               typeToGVK: map[reflect.Type][]schema.GroupVersionKind{},
               unversionedTyped: map[reflect.Type][schema.GroupVersionKind{},
               unversionedKinds: map[string]reflect.Type{},
               cloner: conversion.NewCloner(),
               fieldLabelConversionFuncs: map[string]map[string]FieldLabelConversionFunc{},
               defaultFuncs: map[reflect.Type]func(interface{}){},
          }
          s.converter = conversion.NewConverter(s.nameFunc)
          s.AddConversionFuncs(DefaultEmbeddedConversions()...)
          if err:=s.AddConversionFuncs(DefaultStringConversions...); err!=nil{
               panic(err)
          }
          if err:=s.RegisterInputDefaults(&map[string][]string{}, JSONKeyMapper, conversion.AllowDifferentFieldTypeNames|conversion.IgnoreMissingFields); err!=nil{
               panic(err)
          }
          if err:=s.RegisterInputDefaults(&url.Values{}, JSONKeyMapper, conversion.AllowDifferentFieldTypeNames|conversion.IgnoreMissingFields); err!=nil{
               panic(err)
          }
          return s
     }

k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/runtime/codec.go

     func NewParameterCodec(scheme *Scheme)ParameterCodec{
          return &paramterCodec{
               typer: scheme,
               convertor: scheme,
               creator: scheme,
          }
     }
     type parameterCodec struct{
          typer ObjectTyper
          convertor ObjectConvertor
          creator ObjectCreater
     }
     //ObjectConverter converts an object to a different version.
     type ObjectConverter interface{
          Convert(in, out, context interface{}) error
          ConvertToVersion(in Object, gv GroupVersioner) (out Object, err error)
          ConvertFieldLabel(version, kind, label, value string)(string,string,error)
     }
     //ObjectTyper contains methods for extracting the APIVersion and Kind of objects
     type ObjectTyper interface{
          ObjectKinds(Object)([]schema.GroupVersionKind, bool, error)
          Recognizes(gvk schema.GroupVersionKind) bool
     }
     //ObjectCreater contains methods for instantiating an object by kind and version.
     type ObjectCreater interface{
          New(kind schema.GroupVersionKind)(out Object, err error)
     }

k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1/types_swagger_doc_generated.go

     var map_APIGroup = map[string]string{
          "": "APIGroup contains the name, the supported versions, and the preferred version of a group.",
          "name": "name is the name of the group",
          "versions": "versions are the versions supported in this group.",
          "preferredVersion": "preferredVersion is the version preferred by the API server, which probably is the storage version.",
          "serverAddressByClientCIDRs": "a map of client CIDR to server address that is serving this group. This is to help clients reach servers in the most network-efficient way possible. Clients can use the appropriate server address as per the CIDR that they match. In case of multiple matches, clients should use the longest matching CIDR. The server returns only those CIDRs that it thinks that the client can match. For example: the master will return an internal IP CIDR only, if the client reaches the server using an internal IP. Server looks at X-Forwarded-For header or X-Real-IP header or request.RemoteAddr (in that order) to get the client IP.",
     }
     var map_APIGroupList = map[string]string{
          "": "APIGroupList is a lsit of APIGroup, to allow clients to discover the API at /apis.",
          "groups":"groups is a list of APIGroup",
     }
     var map_APIResource=map[string]string{
          "": "APIResource specifies the name of a resource and whether it is namespaced.",
          "name":"name is the plural name of the resource",
          ...
     }
     var map_APIResourceList = map[string]string{
          "":"",
          ...
     }
     var map_APIVersions = map[string]string{
          "": "",
          ...
     }
     var map_DeleteOptions = map[string]string{
          "":"",
          ...
     }
     var map_ExportOptions = map[string]string{
          "":"",
          ...
     }
     var map_GetOptions = map[string]string{
          "":"",
          ...
     }
     var map_GroupVersionForDiscovery= map[string]string{
          "":"",
          ...
     }
     var map_Initializer= map[string]string{
          "":"",
          ...
     }
     var map_Initializers= map[string]string{
          "":"",
          ...
     }
     var map_LabelSelector= map[string]string{
          "":"",
          ...
     }
     var map_LabelSelectorRequirement= map[string]string{
          "":"",
          ...
     }
     var map_ListMeta= map[string]string{
          "":"",
          ...
     }
     var map_ListOptions= map[string]string{
          "":"",
          ...
     }
     var map_ObjectMeta= map[string]string{
          "":"",
          ...
     }
     var map_OwnerReference= map[string]string{
          "":"",
          ...
     }
     var map_Patch= map[string]string{
          "":"",
          ...
     }
     var map_Preconditions= map[string]string{
          "":"",
          ...
     }
     var map_RootPaths= map[string]string{
          "":"",
          ...
     }
     var map_ServerAddressByClientCIDR= map[string]string{
          "":"",
          ...
     }
     var map_Status= map[string]string{
          "":"",
          ...
     }
     var map_StatusCause= map[string]string{
          "":"",
          ...
     }
     var map_StatusDetails= map[string]string{
          "":"",
          ...
     }
     var map_TypeMeta= map[string]string{
          "":"",
          ...
     }

k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1/generated.pb.go

     func init(){
          proto.RegisterType((*APIGroup)(nil), "k8s.io.apimachinery.pkg.apis.meta.v1.APIGroup")
          proto.RegisterType((*APIGroup)(nil), "k8s.io.apimachinery.pkg.apis.meta.v1.APIGroupList")
          proto.RegisterType((*APIResource)(nil), "k8s.io.apimachinery.pkg.apis.meta.v1.APIResource")
          proto.RegisterType((*APIResourceList)(nil), "k8s.io.apimachinery.pkg.apis.meta.v1.APIResourceList")
          proto.RegisterType((*APIVersions)(nil), "k8s.io.apimachinery.pkg.apis.meta.v1.APIVersions")
          proto.RegisterType((*DeleteOptions)(nil), "k8s.io.apimachinery.pkg.apis.meta.v1.DeleteOptions")
          proto.RegisterType((*Duration)(nil), "k8s.io.apimachinery.pkg.apis.meta.v1.Duration")
          proto.RegisterType((*ExportOptions)(nil), "k8s.io.apimachinery.pkg.apis.meta.v1.ExportOptions")
          ...
     }
     type APIGroup struct{
          TypeMeta `json:",inline"`
          Name string `json:"name" protobuf:"bytes,1,opt,name=name"`
          PreferredVersion GroupVersionForDiscovery `json:"preferredVersion,omitempty" protobuf:"bytes,3,opt,name=preferredVersion"`
          ServerAddressByClinetCIDRs []ServerAddressByClinetCIDR `json:"serverAddressByClientCIDRs" protobuf:"bytes,4,rep,name=serverAddressByClientCIDRs"`
     }


     func init(){
          proto.RegisterFile("k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1/generated.proto", fileDescriptorGenerated)
     }
     var fileDescriptorGenerated = []byte{
          0x1f, 0x8b,...
     }

k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1/register.go

     func init(){
          scheme.AddUnversionedTypes(SchemeGroupVersion,
               &ListOptions{},
               &ExportOptions{},
               &GetOptions{},
               &DeleteOptions{},
          )
          // register manually. This usually goes through the SchemeBuilder, which we cannot use here.
          scheme.AddGeneratedDeepCopyFuncs(GetGeneratedDeepCopyFuncs()...)
          RegisterDefaults(scheme)
     }

     var SchemeGroupVersion = schema.GroupVersion{Group: GroupName, Version: "v1"}

_________________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apiextensions-apiserver/pkg/apis/apiextensions**

k8s.io/kubernetes/vendor/k8s.io/apiextensions-apiserver/pkg/apis/apiextensions/register.go

     var(
          SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)
          AddToScheme = SchemeBuilder.AddToScheme
     )
     //NewSchemeBuilder calls Register for you
     func NewSchemeBuilder(funcs ...func(*Scheme) error)SchemeBuilder{
          var sb SchemeBuilder
          sb.Register(funcs...)
          return sb
     }
     //Register adds a scheme setup function to the list
     func (sb *SchemeBuilder) Register(funcs ..func(*Scheme)error){
          for _,f:=range funcs{
               *sb=append(*sb,f)
          }
     }
     type SchemeBuilder []func(*Scheme) error
     //AddToScheme applies all the stored functions to the scheme. A non-nil error indicates that one function failed and the attempt was abandoned.
     func (sb *SchemeBuilder)AddToScheme(s *Scheme) error{
          for _,f:=range *sb{
               if err:=f(s);err!=nil{
                    return err
               }
          }
          return nil
     }
     // Adds the list of known types to api.Scheme.
     func addKnownTypes(scheme *runtime.Scheme) error{
          scheme.AddKnownTypes(SchemeGroupVersion,
               &CustomResourceDefinition{},
               &CustomResourceDefinitionList{},
          }
          return nil
     }

k8s.io/kubernetes/vendor/k8s.io/apiextensions-apiserver/pkg/apis/apiextensions/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }
     func RegisterDeepCopies(scheme *runtime.Scheme) error{
          return scheme.AddGeneratedDeepCopyFuncs(
               conversion.GeneratedDeepCopyFunc{Fn: DeepCopy_apiextensions_CustomResourceDefinition, InType: reflect.TypeOf(&CustomResourceDefinition{})},
               ...
          )
     }
     func (s *Scheme) AddGeneratedDeepCopyFuncs(deepCopyFuncs ...conversion.GenerateDeepCopyFuncs) error{
          for _, fn:=range deepCopyFuncs{
               if err:=s.cloner.RegisterGeneratedDeepCopyFunc(fn); err!=nil{
                    return err
               }
          }
          retur nil
     }
     func (c *Cloner)RegisterGeneratedDeepCopyFunc(fn GeneratedDeepCopyFunc) error{
          c.generatedDeepCopyFuncs[fn.InType] = fn.Fn
          retur nil
     }
     type GeneratedDeepCopyFunc struct{
          Fn func(in interface{}, out interface{}, c *Cloner) error
          InType reflect.Type
     }

_______________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apiextensions-apiserver/pkg/apis/apiextensions/v1beta1**

k8s.io/kubernetes/vendor/k8s.io/apiextensions-apiserver/pkg/apis/apiextensions/v1beta1/generated.pb.go

     var(
          ErrInvalidLengthGenerated = fmt.Errorf("proto: negative length found during unmarshaling")
          ErrIntOverflowGenerated = fmt.Errorf("proto: integer overflow")
     )

k8s.io/kubernetes/vendor/k8s.io/apiextensions-apiserver/pkg/apis/apiextensions/v1beta1/register.go

     var(
          SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes, addDefaultingFuncs)
          AddToScheme = SchemeBuilder.AddToScheme
     )

k8s.io/kubernetes/vendor/k8s.io/apiextensions-apiserver/pkg/apis/apiextensions/v1beta1/generated.pb.go

     func init(){
          proto.RegisterType((*CustomResourceDefinition)(nil), "k8s.io.apiextensions_apiserver.pkg.apis.apiextensions.v1beta1.CustomResourceDefinition")
          ...
     }
     func init(){
          proto.RegisterFile("k8s.io/kubernetes/vendor/k8s.io/apiextensions-apiserver/pkg/apis/apiextensions/v1beta1/generated.proto", fileDescriptorGenerated)
     }

k8s.io/kubernetes/vendor/k8s.io/apiextensions-apiserver/pkg/apis/apiextensions/v1beta1/zz_generated.conversion.go

     func init(){
          SchemeBuilder.Register(RegisterConversions)
     }

k8s.io/kubernetes/vendor/k8s.io/apiextensions-apiserver/pkg/apis/apiextensions/v1beta1/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

________________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1lapha1**

k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1lapha1/generated.pb.go

     var(
          ErrInvalidLengthGenerated = fmt.Errorf("proto: negative length found during unmarshaling")
          ErrIntOverflowGenerated = fmt.Errorf("proto: integer overflow")
     )

k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1lapha1/register.go

     // scheme is the registry for the common types that adhere to the meta v1lapha1 API spec.
     var scheme = runtime.NewScheme()
     var ParameterCodec = runtime.NewParmeterCodec(scheme)

k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1lapha1/types_swagger_doc_generated.go

     var map_PartialObjectMetadata = map[string]string{
          "":"",
          "metadata":"",
     }
     var map_PartialObjectMetadataList = map[string]string{
          "":"",
          "items":"",
     }
     var map_Table= map[string]string{
          "":"",
          ...
     }
     var map_TableColumnDefinition= map[string]string{
          "":"",
          ...
     }
     var map_TableOptions= map[string]string{
          "":"",
          ...
     }
     var map_TableRow= map[string]string{
          "":"",
          ...
     }
     var map_TableRowCondition= map[string]string{
          "":"",
          ...
     }

k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1lapha1/generated.pb.go

     func init(){
          proto.RegisterType((*PartialObjectMetadata)(nil), "k8s.io.apimachimery.pkg.apis.meta.v1alpha1.PartialObjectMetadata")
          proto.RegisterType((*PartialObjectMetadataList)(nil), "k8s.io.apimachimery.pkg.apis.meta.v1alpha1.PartialObjectMetadataList")
          proto.RegisterType((*TableOptions)(nil), "k8s.io.apimachimery.pkg.apis.meta.v1alpha1.TableOptions")
     }
     func init(){
          proto.RegisterFile("k8s.io/kubernetes/vendor/k8s.io/apichaminery/pkg/apis/meta/v1alpha1/generated.proto", fileDescriptorGenerated)
     }

k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1lapha1/register.go

     func init(){
          scheme.AddKnownTypes(SchemeGroupVersion,
               &Table{},
               &TableOptions{},
               &PartialObjectMetadata{},
               &PartialObjectMetadataList{},
          )
     }

__________________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/davecgh/go-spew/spew**

k8s.io/kubernetes/vendor/github.com/davecgh/go-spew/spew/dump.go

     var(
          uint8Type = reflect.TypeOf(uint8(0))
          cCharRE = regexp.MustCompile("^.*\\._Ctype_char$")
          cUnsignedCharRE = regexp.MustCompile("^.*\\._Ctype_unsignedchar$")
          cUint8CharRE = regexp.MustCompile("^.*\\._Ctype_uint8_t$")
     )

k8s.io/kubernetes/vendor/github.com/davecgh/go-spew/spew/bypass.go

     func init(){
          vv:=reflect.ValueOf(0xf00)
          if unsafe.Sizeof(vv) == (ptrSize * 4){
               offsetScalar = ptrSize *2
               offsetFlag = ptrSize * 3
          }

          upf:=unsafe.Pointer(uintptr(unsafe.Pointer(&vv))+offsetFlag)
          upfv := *(*uintptr)(upf)
          flagKindMask:=uintptr((1<<flagKindWidth-1)<<flagKindShift)
          if (upfv&flagKindMask)>>flagKindShift!=uintptr(reflect.Int){
               flagKindShift=0
               flagRO=1<<5
               flagIndir = 1<<6
               if upfv&flagIndir==0{
                    flagRO=3<<5
                    flagIndir = 1<<7
               }
          }
     }

___________________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/conversion/unstructured**

k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/conversion/unstructured/converter.go

     var(
          marshalerType = reflect.TypeOf(new(encodingjson.Marshaler)).Elem()
          unmarshalerType = reflect.TypeOf(new(encodingjson.Unmarshaler)).Elem()
          mapStringInterfaceType = reflect.TypeOf(map[string]interface{}{})
          stringType = reflect.TypeOf(string(""))
          int64Type = reflect.TypeOf(int64(0))
          uint64Type = reflect.TypeOf(uint64(0))
          float64Type = reflect.TypeOf(float64(0))
          boolType = reflect.TypeOf(bool(false))
          fieldCache = newFieldCache()
          DefaultConverter = NewConverter(parseBool(os.Getenv("KUBE_PATH_CONVERSION_DETECTOR")))
     )

___________________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1/unstructured/unstructured.go**

k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1/unstructured/unstructured.go

     var converter = unstructured.NewConverter(false)

     func NewConverter(mismatchDetection bool) Converter{
          return &converterImpl{
               mismatchDetection: mismatchDetection,
          }
     }
     type converterImpl struct{
          mismatchDetection bool
     }

___________________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta**

k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/help.go

     var objectSliceType = reflect.TypeOf([]runtime.Object{})

k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/meta.go

     var errNotList = fmt.Errorf("object does not implement the List interfaces")
     var errNotObject = fmt.Errorf("object does not implement the Object interfaces")

_________________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/gopkg.in/yaml.v2**

k8s.io/kubernetes/vendor/gopkg.in/yaml.v2/decode.go

     var(
          mapItemType = reflect.TypeOf(MapItem{})
          durationType = reflect.TypeOf(time.Duration(0))
          defaultMapType = reflect.TypeOf(map[interface{}]interface{}{})
          ifaceType = defaultMapType.Elem()
     )

k8s.io/kubernetes/vendor/gopkg.in/yaml.v2/encode.go

     var base60float = regexp.MustCompile(`^[-+]?[0-9][0-9_]*(?::[0-5]?[0-9])+(?:\.[0-9_]*)?$`)

k8s.io/kubernetes/vendor/gopkg.in/yaml.v2/resolve.go

     var resolveTable = make([]byte, 256)
     var resolveMap = make(map[string]resolveMapItem)

k8s.io/kubernetes/vendor/gopkg.in/yaml.v2/yaml.go

     var structMap = make(map[reflect.Type]*structInfo)
     var fieldMapMutex sync.RWMutex

k8s.io/kubernetes/vendor/gopkg.in/yaml.v2/resolve.go

     func init(){
          t:=resolveTable
          t[int('+')] = 'S'
          t[int('-')] = 'S'
          for _,c:=range "0123456789"{
               t[int(c)]='D'
          }
          for _,c:=range "yYnNtTfFoO~{
               t[int(c)] = 'M'
          }
          t[int('.')]='.'
          var resolveMapList = []struct{
               v interface{}
               tag string
               l []string
          }{
               {true, yaml_BOOL_TAG, []string{"y", "Y", "yes", "Yes", "YES"}},
               {true, yaml_BOOL_TAG, []string{"true", "True", "TRUE"}},
               ...
          }
          m:=resolveMap
          for _, item:=range resolveMapList{
               for _,s:=range item.l{
                    m[s]=resolveMapItem{item.v, item.tag}
               }
          }
     }

____________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/ugorji/go/codec**

k8s.io/kubernetes/vendor/github.com/ugorji/go/codec/decode.go

     var(
          onlyMapOrArrayCanDecodeIntoStructErr = errors.New("only encoded map or array can be decoded into a struct")
          cannotDecodeIntoNilErr = errors.New("cannot decode into nil")
     )
     var bytesDesReaderCannotUnreadErr = errors.New("cannot unread last byte read")
     
k8s.io/kubernetes/vendor/github.com/ugorji/go/codec/gen.go

     var(
          genAllTypesSamePkgErr = errors.New("All types must be in the same package")
          genExpectArrayOrMapErr = errors.New("unexpected type. Expecting array/map/slice")
          genBase64enc = base64.NewEncoding("ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz")
          genQNameRegex = regexp.MustCompile(`[A-Za-z_.]+`)
          genCheckVendor bool

k8s.io/kubernetes/vendor/github.com/ugorji/go/codec/helper.go

     var(
          oneByteArr = [1]byte{0}
          zeroByteSlice = oneByteArr[:0:0]
     )
     var rgetPool = sync.Pool{
          New: func() interface{}{return new(rgetPoolT)},
     }
     var(
          bigen = binary.BigEndian
          structInfoFieldName = "_struct"
          mapStrIntTyp = reflect.TypeOf(map[string]interface{}(nil))
          mapIntfIntfTyp = reflect.TypeOf(map[interface{}]interface{}(nil))
          intfSliceTyp = reflect.TypeOf([]interface{}(nil))
          intfTyp = intfSliceTyp.Elem()
          stringTyp = reflect.TypeOf("")
          timeTyp = reflect.TypeOf(time.Time{})
          rawExtTyp = reflect.TypeOf(RawExt{})
          rawTyp = reflect.TypeOf(Raw{})
          uint8SliceTyp = reflect.TypeOf([]uint8(nil))
          mapBySliceTyp = reflect.TypeOf((*MapBySlice)(nil)).Elem()
          binaryMarshalerTyp = reflect.TypeOf((*encoding.BinaryMarshaler)(nil)).Elem()
          binaryUnmarshalerTyp = reflect.TypeOf((*encoding.BinaryUnmarshaler)(nil)).Elem()
          textMarshalerTyp = reflect.TypeOf((*encoding.TextMarshaler)(nil)).Elem()
          textUnmarshalerTyp = reflect.TypeOf((*encoding.TextUnamrshaler)(nil)).Elem()
          jsonMarshalerTyp = reflect.TypeOf((*jsonMarshaler)(nil)).Elem()
          jsonUnmarshalerTyp = reflect.TypeOf((*jsonUnmarshaler)(nil)).Elem()
          selferTyp = reflect.TypeOf((*Selfer)(nil)).Elem()
          uint8SliceTypId = reflect.ValueOf(uint8SliceTyp).Ponter()
          rawExtTypId= reflect.ValueOf(rawExtTyp).Ponter()
          rawTypId = reflect.ValueOf(rawTyp).Ponter()
          intTypId = reflect.ValueOf(intTyp).Ponter()
          timeTypId = reflect.ValueOf(timeTyp).Ponter()
          stringTypId = reflect.ValueOf(stringTyp).Ponter()
          mapStrIntfTypId = reflect.ValueOf(mapStrIntfTyp).Ponter()
          mapIntIntfTypId = reflect.ValueOf(mapIntfIntfTyp).Pointer()
          intfSliceTypId = reflect.ValueOf(intfSliceTyp).Pointer()
          intBitsize uint8 = uint8(reflect.TypeOf(int(0)).Bits())
          uintBitsize uint8 = uint8(reflect.TypeOf(uint(0)).Bits())
          bsAll0x00 = []byte{0,0,0,0,0,0,0,0}
          bsAll0xff = []byte{0xff, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff}
          chkOvf checkOverflow
          noFieldNameToStructFieldInfoErr = errors.New("no field name passed to parseStructFieldInfo")
     )
     var defTypeInfos = NewTypeInfos([]string{"codec", "json"})
     
k8s.io/kubernetes/vendor/github.com/ugorji/go/codec/encode.go

     func init(){
          encStructPool[0].New = func()interface{}{return new([8]stringRv)}
          encStructPool[1].New = func()interface{}{return new([16]stringRv)}
          encStructPool[2].New = func()interface{}{return new([32]stringRv)}
          encStructPool[3].New = func()interface{}{return new([64]stringRv)}
          encStructPool[4].New = func()interface{}{return new([128]stringRv)}
     }

k8s.io/kubernetes/vendor/github.com/ugorji/go/codec/fast-path.generated.go

     func init(){
          i:=0
          fn:=func(v interface{}, fe func(*encFnInfo, reflect.Value), fd func(*decFnInfo, reflect.Value))(f fastpathE){
               xrt:=reflect.TypeOf(v)
               xptr:=reflect.ValueOf(xrt).Pointer()
               fastpathAV[i]=fastpathE{xptr, xrt,fe,fd}
               i++
               return
          }
          fn([]interface{}(nil), (*encFnInfo).fastpathEncSliceIntfR, (*decFnInfo).fastpathDecSliceIntfR)
          ...

          sort.Sort(fastpathAslice(fastpathAV[:]))
     }

k8s.io/kubernetes/vendor/github.com/ugorji/go/codec/gen_16.go

     func init(){
          genCheckVendor = os.Getenv("GO15VENDOREXPERIMENT")!="0"
     }

k8s.io/kubernetes/vendor/github.com/ugorji/go/codec/gen_17.go

     func init(){
          genCheckVendor = true
     }

k8s.io/kubernetes/vendor/github.com/ugorji/go/codec/json.go

     const(
          jsonSpacesOrTabsLen = 128
     )
     func init(){
          var bs [jsonSpacesOrTabsLen]byte
          for i:=0; i<jsonSpacesOrTabsLen; i++{
               bs[i]=' '
          }
          jsonSpaces = string(bs[:])
          for i:=0;i<jsonSpacesOrTabsLen;i++{
               bs[i]='\t'
          }
          jsonTas = string(bs[:])
     }

____________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/runtime/serializer**

k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/runtime/serializer/protobuf_extension.go

     func init(){
          serializerExtensions = append(serializerExtensions, protobufSerializer)
     }
     func protobufSerializer(scheme *runtime.Scheme)(serializerType, bool){
          serializer:=protobuf.NewSerializer(scheme, scheme, contentTypeProtobuf)
          raw:=protobuf.NewRawSerializer(scheme, scheme, contentTypeProtobuf)
          return serializerType{
               AcceptContentTypes: []string{contentTypeProtobuf},
               ContentType: contentTypeProtobuf,
               FileExtensions: []string{"pb"},
               Serializer: serializer,
               Framer: protobuf.LengthDelimitedFramer,
               StreamSerializer: raw,
          }, true
     }

_________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apiextensions-apiserver/pkg/client/clientset/internalclientset/scheme**

k8s.io/kubernetes/vendor/k8s.io/apiextensions-apiserver/pkg/client/clientset/internalclientset/scheme/register.go

     var Scheme = runtime.NewScheme()
     var Codecs = serializer.NewCodecFactory(Scheme)
     var ParameterCodec = runtime.NewParameterCodec(Scheme)
     var Registry = registered.NewOrDie(os.Getenv("KUBE_API_VERSIONS"))
     var GroupFactoryRegistry = make(announced.APIGroupFactoryRegistry)
     
     func init(){
          v1.AddToGroupVersion(Scheme, schema.GroupVersion{Version: "v1"})
          Install(GroupFactoryRegistry, Registry, Scheme)
     }

_____________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/api**

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/api/register.go

     var GroupFactoryRegistry = make(announced.APIGroupFactoryRegistry)
     var Registry = registered.NewOrDie(os.Getenv("KUBE_API_VERSIONS"))
     var Scheme = runtime.NewScheme()
     var Codec = serializer.NewCodecFactory(Scheme)

     var ParameterCodec = runtime.NewParameterCodec(Scheme)
     var(
          SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)
          AddToScheme = SchemeBuilder.AddToScheme
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/api/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }
     func RegisterDeepCopies(scheme *runtime.Scheme)error{
          return scheme.AddGeneratedDeepCopyFuncs(
               conversion.GeneratedDeepCopyFunc{Fn: DeepCopy_api_AWSElasticBlockStoreVolumeSource, InType:reflect.TypeOf(&AWSElasticBlockStoreVolumeSource{})},
               ...
          )
     }

____________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/extensions**

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/extensions/register.go

     var(
          SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)
          AddToScheme = SchemeBuilder.AddToScheme
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/extensions/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

___________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/docker/distribution/digest/digest.go**

k8s.io/kubernetes/vendor/github.com/docker/distribution/digest/digest.go

     var DigestRegexp = regexp.MustCompile(`[a-zA-Z0-9-_+.]+:[a-fA-F0-9]+`)
     var DigestRegexpAnchored = regexp.MustCompile(`^` + DigestRegexp.String() + `$`)
     var(
          ErrDigestInvalidFormat = fmt.Errorf("invalid checksum digest format")
          ErrDigestInvalidLength = fmt.Errorf("invalid checksum digest length")
          ErrDigestUnsupported = fmt.Errorf("unsupported digest algorithm")
     )
     var(
          algorithms = map[Algorithm]crypto.Hash{
               SHA256: crypto.SHA256,
               SHA384: crypto.SHA384,
               SHA512: crypto.SHA512,
          }
     )

k8s.io/kubernetes/vendor/github.com/docker/distribution/digest/set.go

     var(
          ErrDigestNotFound = errors.New("digest not found")
          ErrDigestAmbiguous = errors.New("ambiguous digest string")
     )

k8s.io/kubernetes/vendor/github.com/docker/distribution/reference/reference.go

     var(
          ErrReferenceInvalidFormat = errors.New("invalid reference format")
          ErrTagInvalidFormat = errors.New("invalid tag format")
          ErrDigestInvalidFormt = errors.New("invalid digest format")
          ErrNameEmpty = errors.New("repository name must have at least one component")
          ErrNameTooLong = errors.New("repository name must not be more than %v characters", NameTotalLengthMax)
     )

k8s.io/kubernetes/vendor/github.com/docker/distribution/reference/regexp.go

     var(
          alphaNumbericRegexp = match(`[a-z0-9]+`)
          separatorRegexp = match(`(?:[._]|__|[-]*)`)
          nameComponentRegexp = expression(
               alphaNumbericRegexp,
               optional(repeated(separatorRegexp, alphaNumbericRegexp)))
          
          hostnameComponentRegexp = match(`(?:[a-zA-Z0-9]|[a-zA-Z0-9][a-zA-Z0-9-]*[a-zA-Z0-9])`)
          hostnameRegexp = expression(
               hostnameComponentRegexp,
               optional(repeated(literal(`.`), hostnameComponentRegexp)),
               optional(literal(`:`), match(`[0-9+]`)))
          TagRegexp = match(`[\w][\w.-]{0,127}`)
          anchoredTagRegexp = anchored(TagRegexp)
          DigestRegexp = match(`[A-Za-z][A-Za-z0-9]*(?:[-_+.][A-Za-z][A-Za-z0-9]*)*[:][[:xdigit:]]{32,}`)
          anchoredDigestRegexp = ahchored(DigestRegexp)
          NameRegexp = expression(
               optional(hostnameRegexp, literal(`/`)),
               nameComponentRegexp,
               optional(repeated(literal(`/`), nameComponentRegexp)))
          anchoredNameRegexp = anchored(
               optional(capture(hostnameRegexp), literal(`/`),
               capture(nameComponentRegexp,
                    optional(repeated(literal(`/`), nameComponentRegexp))))
          ReferenceRegexp = anchored(capture(NameRegexp),
               optional(literal(":"), capture(TagRegexp)),
               optional(literal("@"), capture(DigestRegexp)))
     )

k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/util/rand/rand.go

     var rng = struct{
          sync.Mutex
          rand *rand.Rand
     }{
          rand: rand.New(rand.NewSource(time.Now().UTC().UnixNano())),
     }

_____________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/api/v1**

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/api/v1/generated.pb.go

     var(
          ErrInvalidLengthGenerated = fmt.Errorf("proto: negative length found during unmarshaling")
          ErrIntOverflowGenerated = fmt.Errorf("proto: integer overflow")
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/api/v1/register.go

     var(
          SchemeBuilder runtime.SchemeBuilder
          localSchemeBuilder = &SchemeBuilder
          AddToScheme = localSchemeBuilder.AddToScheme
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/api/v1/types.generated.go

     var(
          codecSelferBitsize1234 = uint8(reflect.TypeOf(uint(0)).Bits())
          codecSelferOnlyMapOrArrayEncodeToStructErr1234 = errors.New("`only encoded map or array can be decoded into a struct`)
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/api/v1/types_swagger_doc_generated.go

     var map_AWSElasticBlockStoreVolumeSource = map[string]string{
          "": "",
          ...
     }
     var map_Affinity = map[string]string{
          "":"",
          ...
     }
     var map_AttachedVolume = map[string]string{
          "":"",
          ...
     }
     var map_AvoidPods = map[string]string{
          "":"",
          ...
     }
     var map_AzureDiskVolumeSource = map[string]string{
          "":"",
          ...
     }
     var map_AzureFileVolumeSource = map[string]string{
          "":"",
          ...
     }
     var map_Binding = map[string]string{
          "":"",
          ...
     }
     var map_Capabilities = map[string]string{
          "":"",
          ...
     }
     var map_CephFSVolumeSource = map[string]string{
          "":"",
          ...
     }
     var map_CinderVolumeSource = map[string]string{
          "":"",
          ...
     }
     var map_ComponentCondition = map[string]string{
          "":"",
          ...
     }
     var map_ComponentStatus = map[string]string{
          "":"",
          ...
     }
     var map_ComponentStatusList = map[string]string{
          "":"",
          ...
     }
     var map_ConfigMap = map[string]string{
          "":"",
          ...
     }
     var map_ConfigMapEnvSource = map[string]string{
          "":"",
          ...
     }
     var map_ConfigMapKeySelector = map[string]string{
          "":"",
          ...
     }
     var map_ConfigMapList = map[string]string{
          "":"",
          ...
     }
     var map_ConfigMapProjection = map[string]string{
          "":"",
          ...
     }
     var map_ConfigMapVolumeSource = map[string]string{
          "":"",
          ...
     }
     var map_Container = map[string]string{
          "":"",
          ...
     }
     var map_ContainerImage = map[string]string{
          "":"",
          ...
     }
     var map_ContainerPort = map[string]string{
          "":"",
          ...
     }
     var map_ContainerState = map[string]string{
          "":"",
          ...
     }
     var map_ContainerStateRunning = map[string]string{
          "":"",
          ...
     }
     var map_ContainerStateTerminated = map[string]string{
          "":"",
          ...
     }
     var map_ContainerStateWaiting = map[string]string{
          "":"",
          ...
     }
     var map_ContainerStatus= map[string]string{
          "":"",
          ...
     }
     var map_DaemonEndpoint= map[string]string{
          "":"",
          ...
     }
     var map_DeleteOptions= map[string]string{
          "":"",
          ...
     }
     var map_DownwardAPIProjection = map[string]string{
          "":"",
          ...
     }
     var map_DownwardAPIVolumeFile= map[string]string{
          "":"",
          ...
     }
     var map_DownwardAPIVolumeSource= map[string]string{
          "":"",
          ...
     }
     var map_EmptyDirVolumeSource= map[string]string{
          "":"",
          ...
     }
     var map_EndpointAddress= map[string]string{
          "":"",
          ...
     }
     var map_EndpointPort= map[string]string{
          "":"",
          ...
     }
     var map_EndpointSubset= map[string]string{
          "":"",
          ...
     }
     var map_Endpoints= map[string]string{
          "":"",
          ...
     }
     var map_EndpointsList= map[string]string{
          "":"",
          ...
     }
     var map_EnvFromSource= map[string]string{
          "":"",
          ...
     }
     var map_EnvVar= map[string]string{
          "":"",
          ...
     }
     var map_EnvVarSource= map[string]string{
          "":"",
          ...
     }
     var map_Event= map[string]string{
          "":"",
          ...
     }
     var map_EventList= map[string]string{
          "":"",
          ...
     }
     var map_EventSource= map[string]string{
          "":"",
          ...
     }
     var map_ExecAction= map[string]string{
          "":"",
          ...
     }
     var map_FCVolumeSource= map[string]string{
          "":"",
          ...
     }
     var map_FlexVolumeSource= map[string]string{
          "":"",
          ...
     }
     var map_FlockerVolumeSource= map[string]string{
          "":"",
          ...
     }
     var map_GCEPersistentDiskVolumeSource= map[string]string{
          "":"",
          ...
     }
     var map_GitRepoVolumeSource= map[string]string{
          "":"",
          ...
     }
     var map_GlusterfsVolumeSource= map[string]string{
          "":"",
          ...
     }
     var map_HTTPGetAction= map[string]string{
          "":"",
          ...
     }
     var map_HTTPHeader= map[string]string{
          "":"",
          ...
     }
     var map_Handler= map[string]string{
          "":"",
          ...
     }
     var map_HostAlias= map[string]string{
          "":"",
          ...
     }
     var map_HostPathVolumeSource= map[string]string{
          "":"",
          ...
     }
     var map_ISCSIVolumeSource= map[string]string{
          "":"",
          ...
     }
     var map_KeyToPath= map[string]string{
          "":"",
          ...
     }
     var map_Lifecycle= map[string]string{
          "":"",
          ...
     }
     var map_LimitRange= map[string]string{
          "":"",
          ...
     }
     var map_LimetRangeItem= map[string]string{
          "":"",
          ...
     }
     var map_LimetRangeList= map[string]string{
          "":"",
          ...
     }
     var map_LimitRangeSpec= map[string]string{
          "":"",
          ...
     }
     var map_List= map[string]string{
          "":"",
          ...
     }
     var map_ListOptions= map[string]string{
          "":"",
          ...
     }
     var map_LoadBalancerIngress= map[string]string{
          "":"",
          ...
     }
     var map_LoadBalancerStatus= map[string]string{
          "":"",
          ...
     }
     var map_LocalObjectReference= map[string]string{
          "":"",
          ...
     }
     var map_LocalVolumeSource= map[string]string{
          "":"",
          ...
     }
     var map_NFSVolumeSource= map[string]string{
          "":"",
          ...
     }
     var map_Namespace= map[string]string{
          "":"",
          ...
     }
     var map_NamespaceList= map[string]string{
          "":"",
          ...
     }
     var map_NamespaceSpec= map[string]string{
          "":"",
          ...
     }
     var map_NamespaceStatus= map[string]string{
          "":"",
          ...
     }
     var map_Node= map[string]string{
          "":"",
          ...
     }
     var map_NodeAddress= map[string]string{
          "":"",
          ...
     }
     var map_NodeAffinity= map[string]string{
          "":"",
          ...
     }
     var map_NodeCondition= map[string]string{
          "":"",
          ...
     }
     var map_NodeDaemonEndpoints= map[string]string{
          "":"",
          ...
     }
     var map_NodeList= map[string]string{
          "":"",
          ...
     }
     var map_NodeProxyOptions= map[string]string{
          "":"",
          ...
     }
     var map_NodeResources= map[string]string{
          "":"",
          ...
     }
     var map_NodeSelector= map[string]string{
          "":"",
          ...
     }
     var map_NodeSelectorRequirement= map[string]string{
          "":"",
          ...
     }
     var map_NodeSelectorTerm= map[string]string{
          "":"",
          ...
     }
     var map_NodeSpec= map[string]string{
          "":"",
          ...
     }
     var map_NodeStatus= map[string]string{
          "":"",
          ...
     }
     var map_NodeSystemInfo= map[string]string{
          "":"",
          ...
     }
     var map_ObjectFieldSelector= map[string]string{
          "":"",
          ...
     }
     var map_ObjectMeta= map[string]string{
          "":"",
          ...
     }
     var map_ObjectReference= map[string]string{
          "":"",
          ...
     }
     var map_PersistentVolume= map[string]string{
          "":"",
          ...
     }
     var map_PersistentVolumeClaim= map[string]string{
          "":"",
          ...
     }
     var map_PersistentVolumeClaimList= map[string]string{
          "":"",
          ...
     }
     var map_PersistentVolumeClaimSpec= map[string]string{
          "":"",
          ...
     }
     var map_PersistentVolumeClaimStatus= map[string]string{
          "":"",
          ...
     }
     var map_PersistentVolumeClaimVolumeSource= map[string]string{
          "":"",
          ...
     }
     var map_PersistentVolumeList= map[string]string{
          "":"",
          ...
     }
     var map_PersistentVolumeSource= map[string]string{
          "":"",
          ...
     }
     var map_PersistentVolumeSpec= map[string]string{
          "":"",
          ...
     }
     var map_PersistentVolumeStatus= map[string]string{
          "":"",
          ...
     }
     var map_PhonePersistentDiskVolumeSource= map[string]string{
          "":"",
          ...
     }
     var map_Pod= map[string]string{
          "":"",
          ...
     }
     var map_PodAffinity= map[string]string{
          "":"",
          ...
     }
     var map_PodAffinityTerm= map[string]string{
          "":"",
          ...
     }
     var map_PodAntiAffinity= map[string]string{
          "":"",
          ...
     }
     var map_PodAttachOptions= map[string]string{
          "":"",
          ...
     }
     var map_PodCondition= map[string]string{
          "":"",
          ...
     }
     var map_PodExecOptions= map[string]string{
          "":"",
          ...
     }
     var map_PodList= map[string]string{
          "":"",
          ...
     }
     var map_PodLogOptions= map[string]string{
          "":"",
          ...
     }
     var map_PodPortForwardOptions= map[string]string{
          "":"",
          ...
     }
     var map_PodProxyOptions= map[string]string{
          "":"",
          ...
     }
     var map_PodSecurityContext= map[string]string{
          "":"",
          ...
     }
     var map_PodSignature= map[string]string{
          "":"",
          ...
     }
     var map_PodSpec= map[string]string{
          "":"",
          ...
     }
     var map_PodStatus= map[string]string{
          "":"",
          ...
     }
     var map_PodStatusResult= map[string]string{
          "":"",
          ...
     }
     var map_PodTemplate= map[string]string{
          "":"",
          ...
     }
     var map_PodTemplateList= map[string]string{
          "":"",
          ...
     }
     var map_PodTemplateSpec= map[string]string{
          "":"",
          ...
     }
     var map_PortworxVolumeSource= map[string]string{
          "":"",
          ...
     }
     var map_Preconditions= map[string]string{
          "":"",
          ...
     }
     var map_PrefererAvoidPodEntry= map[string]string{
          "":"",
          ...
     }
     var map_PreferredSchedulingTerm= map[string]string{
          "":"",
          ...
     }
     var map_Probe= map[string]string{
          "":"",
          ...
     }
     var map_ProjectedVolumeSource= map[string]string{
          "":"",
          ...
     }
     var map_QuobyteVolumeSource= map[string]string{
          "":"",
          ...
     }
     var map_RBDVolumeSource= map[string]string{
          "":"",
          ...
     }
     var map_RangeAllocation= map[string]string{
          "":"",
          ...
     }
     var map_ReplicationController= map[string]string{
          "":"",
          ...
     }
     var map_ReplicationControllerCondition= map[string]string{
          "":"",
          ...
     }
     var map_ReplicationControllerList= map[string]string{
          "":"",
          ...
     }
     var map_ReplicationControllerSpec= map[string]string{
          "":"",
          ...
     }
     var map_ReplicationControllerStatus= map[string]string{
          "":"",
          ...
     }
     var map_ResourceFieldSelector= map[string]string{
          "":"",
          ...
     }
     var map_ResourceQuota= map[string]string{
          "":"",
          ...
     }
     var map_ResourceQuotaList= map[string]string{
          "":"",
          ...
     }
     var map_ResourceQuotaSpec= map[string]string{
          "":"",
          ...
     }
     var map_ResourceQuotaStatus= map[string]string{
          "":"",
          ...
     }
     var map_ResourceRequirements= map[string]string{
          "":"",
          ...
     }
     var map_SELinuxOptions= map[string]string{
          "":"",
          ...
     }
     var map_ScaleIOVolumeSource= map[string]string{
          "":"",
          ...
     }
     var map_Secret= map[string]string{
          "":"",
          ...
     }
     var map_SecretEnvSource= map[string]string{
          "":"",
          ...
     }
     var map_SecretKeySelector= map[string]string{
          "":"",
          ...
     }
     var map_SecretList= map[string]string{
          "":"",
          ...
     }
     var map_SecretProjection = map[string]string{
          "":"",
          ...
     }
     var map_SecretVolumeSource= map[string]string{
          "":"",
          ...
     }
     var map_SecurityContext= map[string]string{
          "":"",
          ...
     }
     var map_SerializedReference= map[string]string{
          "":"",
          ...
     }
     var map_Service= map[string]string{
          "":"",
          ...
     }
     var map_ServiceAccount= map[string]string{
          "":"",
          ...
     }
     var map_ServiceAccountList= map[string]string{
          "":"",
          ...
     }
     var map_ServiceList= map[string]string{
          "":"",
          ...
     }
     var map_ServicePort= map[string]string{
          "":"",
          ...
     }
     var map_ServiceProxyOptions= map[string]string{
          "":"",
          ...
     }
     var map_ServiceSpec= map[string]string{
          "":"",
          ...
     }
     var map_ServiceStatus= map[string]string{
          "":"",
          ...
     }
     var map_Sysctl= map[string]string{
          "":"",
          ...
     }
     var map_TCPSocketAction= map[string]string{
          "":"",
          ...
     }
     var map_Taint= map[string]string{
          "":"",
          ...
     }
     var map_Toleration= map[string]string{
          "":"",
          ...
     }
     var map_Volume= map[string]string{
          "":"",
          ...
     }
     var map_VolumeMount= map[string]string{
          "":"",
          ...
     }
     var map_VolumeProjection= map[string]string{
          "":"",
          ...
     }
     var map_VolumeSource= map[string]string{
          "":"",
          ...
     }
     var map_VsphereVirtualDiskVolumeSource= map[string]string{
          "":"",
          ...
     }
     var map_WeightedPodAffinityTerm= map[string]string{
          "":"",
          ...
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/api/v1/generated.pb.go

     func init(){
          proto.RegisterType((*AWSElasticBlockStoreVolumeSource)(nil), "k8s.io.client-go.pkg.api.v1.AWSElasticBlockStoreVolumeSource")
          ...
     }
     func init(){
          proto.RegisterFile("k8s.io/client-go/pkg/api/v1/generated.proto", fileDescriptorGenerated)
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/api/v1/register.go

     func init(){
          localSchemeBuilder.Register(addKnownTypes, addDefaultingFuncs, addConversionFuncs, addFastPathConversionFuncs)
     }
     func addKnownTypes(scheme *runtime.Scheme)error{
          scheme.AddKnownTypes(SchemeGroupVersion,
               &Pod{},
               &PodList{},

          ...
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/api/v1/types.generated.go

     func init(){
          if codec1978.GenVersion != 5{
               _,file,_,_:=runtime.Caller(0(
               err:=fmt.Errorf("codecgen version mismatch: current: %v, need %v. Re-generate file: %v", 5, codec1978.GenVersion, file)
               panic(err)
          }
          if false{
               var v0 pkg3_resource.Quantity
               var v1 pkg2_v1.Time
               var v2 pkg5_runtime.RawExtension
               var v3 pkg1_types.UID
               var v4 pkg4_intstr.IntOrString
               var v5 time.Time
               _,_,_,_,_,_ = v0,v1,v2,v3,v4,v5
          }
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/api/v1/zz_generated.conversion.go

     func init(){
          SchemeBuilder.Register(RegisterConversions)
     }
     func RegisterConversions(scheme *runtime.Scheme)error{
          return scheme.AddGeneratedConversionsFuncs(
               Convert_v1_AWSElasticBlockStoreVolumeSource_To_api_AWSElasticBlockStoreVolumeSource,
               ...
          )
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/api/v1/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }
     func RegisterDeepCopies(scheme *runtime.Scheme)error{
          return scheme.AddGeneratedDeepCopyFuncs(
               conversion.GeneratedDeepCopyFunc{Fn: DeepCopy_v1_AWSElasticBlockStoreVolumeSource, InType: reflect.TypeOf(&AWSElasticBlockStoreVolumeSource{})},
               ...
          )
     }

________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/client-go/tools/clientcmd/api**

k8s.io/kubernetes/vendor/k8s.io/client-go/tools/clientcmd/api/register.go

     var(
          SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)
          AddToScheme = SchemeBuilder.AddToScheme
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/tools/clientcmd/api/helpers.go

     func init(){
          sDec,_:=base64.StdEncoding.DecodeString("REDACTED+")
          redactedBytes = []byte(string(sDec))
     }

k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/runtime/serializer/streaming/streaming.go

     var ErrObjectTooLarge = fmt.Errorf("object to decode was longer than maximum allowed size")

____________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/client-go/transport/cache.go**

k8s.io/kubernetes/vendor/k8s.io/client-go/transport/cache.go

     var tlsCache = &tlsTransportCache{transports: make(map[string]*http.Transport)}
     type tlsTransportCache struct{
          mu sync.Mutex
          transports map[string]*http.Transport
     }

____________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/client-go/rest**

k8s.io/kubernetes/vendor/k8s.io/client-go/rest/plugin.go

     var pluginsLock sync.Mutex
     var plugins = make(map[string]Factory)
     type Factory func(clusterAddress string, config map[string]string, persister AuthProviderConfigPersister)(AuthProvider,error)

k8s.io/kubernetes/vendor/k8s.io/client-go/rest/request.go

     var(
          specialParms = sets.NewString("timeout")
          longThrottleLatency = 50*time.Millisecond
     )
     var fieldMappings = versionToResourceToFieldMapping{
          v1.SchemeGroupVersion: resourceTypeToFieldMapping{
               "nodes": clientFieldNameToAPIVersionFieldName{
                    objectNameField: objectNameField,
                    nodeUnschedulable: nodeUnschedulable,
               },
               "pods": clientFieldNameToAPIVersionFieldName{
                    objectNameField: objectNameField,
                    podHost: podHost,
                    podStatus: podStatus,
               },
               "secrets": clientFieldNameToAPIVersionFieldName{
                    secretType: secretType,
               },
               "serviceAccounts": clientFieldNameToAPIVersionFieldName{
                    objectNameField: objectNameField,
               },
               "endpoints": clientFieldNameToAPIVersionFieldName{
                    objectNameField: objectNameField,
               },
               "events": clientFieldNameToAPIVersionFieldName{
                    objectNameField: objectNameField,
                    eventReason: eventReason,
                    eventSource: eventSource,
                    eventInvolvedKind: eventInvolvedKind,
                    eventInvolvedNamespace: eventInvolvedNamespace,
                    eventInvolvedName: eventInvolvedName,
                    eventInvolvedUID: eventInvolvedUID,
                    eventInvolvedAPIVersion: eventInvolvedAPIVersion,
                    eventInvolvedResourceVersion: eventInvolvedResourceVersion,
                    eventInvolvedFieldPath: eventInvolvedFieldPath,
               },
          },
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/rest/unbackoff.go

     var serverIsOverloadedSet = sets.NewInt(429)

__________________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/go-openapi/loads**

k8s.io/kubernetes/vendor/github.com/go-openapi/loads/spec.go

     var swag20Schema = spec.MustLoadSwagger20Schema()
     // MustLoadSwagger20Schema panics when Swagger20Schema returns an error
     func MustLoadSwagger20Schema()*Schema{
          d,e:=Swagger20Schema()
          if e!= nil{
               panic(e)
          }
          return d
     }

__________________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/admissionregistration**

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/admissionregistration/register.go

     var(
          SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)
          AddToScheme = SchemeBuilder.AddToScheme
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/admissionregistration/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

__________________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/admissionregistration/v1alpha1**

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/admissionregistration/v1alpha1/generated.pb.go

     var(
          ErrInvalidLengthGenerated = fmt.Errorf("proto: negative length found during unmarshaling")
          ErrIntOverflowGenerated = fmt.Errorf("proto: integer overflow")
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/admissionregistration/v1alpha1/register.go

     var(
          SchemeBuilder runtime.SchemeBuilder
          localSchemeBuilder = &SchemeBuilder
          AddToScheme = localSchemeBuilder.AddToScheme
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/admissionregistration/v1alpha1/types.generated.go

     var(
          codecSelferBitsize1234 = uint8(reflect.TypeOf(uint(0)).Bits())
          codecSelferOnlyMapOrArrayEncodeToStructErr1234 = errors.New(`only encoded map or array can be decoded into a struct`)
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/admissionregistration/v1alpha1/types_swagger_doc_generated.go

     var map_AdmissionHookClientConfig = map[string]string{
          "":"",
          ...
     }
     var map_ExternalAdmissionHook= map[string]string{
          "":"",
          ...
     }
     var map_ExternalAdmissionHookConfiguration= map[string]string{
          "":"",
          ...
     }
     var map_ExternalAdmissionHookConfigurationList= map[string]string{
          "":"",
          ...
     }
     var map_Intializer= map[string]string{
          "":"",
          ...
     }
     var map_InitializerConfiguration= map[string]string{
          "":"",
          ...
     }
     var map_InitializerConfigurationList= map[string]string{
          "":"",
          ...
     }
     var map_Rule= map[string]string{
          "":"",
          ...
     }
     var map_RuleWithOperations= map[string]string{
          "":"",
          ...
     }
     var map_ServiceReference= map[string]string{
          "":"",
          ...
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/admissionregistration/v1alpha1/generated.pb.go

     func init(){
          proto.RegisterType((*AdmissionHookClientConfig)(nil), "k8s.io.clieng-go.pkg.apis.admissionregistration.v1alpha1.AdmissionHookClientConfig")
          ...
     }
     func init(){
          proto.RegisterFile("k8s.io/client-go/pkg/apis/admissionregistration/v1alpha1/generated.proto", fileDescriptorGenerated)
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/admissionregistration/v1alpha1/register.go

     func init(){
          localSchemeBuilder.Register(addKnownTypes, addDefaultingFuncs)
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/admissionregistration/v1alpha1/types.generated.go

     func init(){
          if codec1978.GenVersion != 5{
               _,file,_,_:=runtime.Caller(0)
               err:=fmt.Errorf("codecgen version mismatch: current: %v, need %v. Re-generate file: %v", 5, codec1978.GenVersion, file)
               panic(err)
          }
          if false{
               var v0 pkg1_v1.TypeMeta
               var v1 pkg2_types.UID
               var v2 time.Time
               _,_,_=v0,v1,v2
          }
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/admissionregistration/v1alpha1/zz_generated.conversion.go

     func init(){
          SchemeBuilder.Register(RegisterConversions)
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/admissionregistration/v1alpha1/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

________________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/apps**

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/apps/register.go

     var(
          SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)
          AddToScheme = SchemeBuilder.AddToScheme
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/apps/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

______________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/apps/v1beta1**

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/apps/v1beta1/generated.pb.go

     var(
          ErrInvalidLengthGenerated = fmt.Errorf("proto: negative length found during unmarshaling")
          ErrIntOverflowGenerated = fmt.Error("proto: integer overflow")
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/apps/v1beta1/register.go

     var(
          SchemeBuilder runtime.SchemeBuilder
          localSchemeBuilder = &SchemeBuilder
          AddToScheme = localSchemeBuilder.AddToScheme
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/apps/v1beta1/types.generated.go

     var(
          codecSelferBitsize1234 = uint8(reflect.TypeOf(uint(0)).Bites())
          codecSelferOnlyMapOrArrayEncodeToStructErr1234 = errors.New(`only encoded map or array can be decoded into a struct`)
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/apps/v1beta1/types_swagger_doc_generated.go

     var map_ControllerRevision = map[string]string{
          "":"",
          ...
     }
     var map_ControllerRevisionList= map[string]string{
          "":"",
          ...
     }
     var map_Deployment= map[string]string{
          "":"",
          ...
     }
     var map_DeploymentCondition= map[string]string{
          "":"",
          ...
     }
     var map_DeploymentList= map[string]string{
          "":"",
          ...
     }
     var map_DeploymentRollback= map[string]string{
          "":"",
          ...
     }
     var map_DeploymentSpec= map[string]string{
          "":"",
          ...
     }
     var map_DeploymentStatus= map[string]string{
          "":"",
          ...
     }
     var map_DeploymentStrategy= map[string]string{
          "":"",
          ...
     }
     var map_PartitionStatefulSetStrategy= map[string]string{
          "":"",
          ...
     }
     var map_RollbackConfig= map[string]string{
          "":"",
          ...
     }
     var map_RollingUpdateDeployment= map[string]string{
          "":"",
          ...
     }
     var map_Scale= map[string]string{
          "":"",
          ...
     }
     var map_ScaleSpec= map[string]string{
          "":"",
          ...
     }
     var map_ScaleStatus= map[string]string{
          "":"",
          ...
     }
     var map_StatefulSet= map[string]string{
          "":"",
          ...
     }
     var map_StatefulSetList= map[string]string{
          "":"",
          ...
     }
     var map_StatefulSetSpec= map[string]string{
          "":"",
          ...
     }
     var map_StatefulSetStatus= map[string]string{
          "":"",
          ...
     }
     var map_StatefulSetUpdateStrategy= map[string]string{
          "":"",
          ...
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/apps/v1beta1/generated.pb.go

     func init(){
          proto.RegisterType((*ControllerRevision)(nil), "k8s.io.client-go.pkg.apis.apps.v1beta1.ControllerRevision")
          ...
     }
     func init(){
          proto.RegisterFile("k8s.io/client-go/pkg/apis/apps/v1beta1/generated.proto", fileDescriptorGenerated)
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/apps/v1beta1/register.go

     func init(){
          localSchemeBuilder.Register(addKnownTypes, addDefaultingFuncs, addConversionFuncs)
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/apps/v1beta1/types.generated.go

     func init(){
          if codec1978.GenVersion != 5{
               _,file,_,_:=runtime.Caller(0)
               err:=fmt.Errorf("codecgen version mismatch: current: %v, need %v. Re-generate file: %v",5,codec1978.GenVersion, file)
               panic(err)
          }
          if false{
               var v0 pkg4_resource.Quantity
               var v1 pkg1_v1.TypeMeta
               var v2 pkg6_runtime.RawExtension
               var v3 pkg2_types.UID
               var v4 pkg5_intstr.IntOrString
               var v5 pkg3_v1.PodTemplateSpec
               var v6 time.Time
               _,_,_,_,_,_,_=v0,v1,v2,v3,v4,v5,v6
          }
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/apps/v1beta1/zz_generated.conversion.go

     func init(){
          SchemeBuilder.Register(RegisterConversions)
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/apps/v1beta1/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

_______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/authentication**

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/authentication/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

_________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/authentication/v1**

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/authentication/v1/generated.pb.go

     var(
          ErrInvalidLengthGenerated = fmt.Errorf("proto: negative length found during unmarshaling")
          ErrIntOverflowGenerated = fmt.Errorf("proto: integer overflow")
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/authentication/v1/register.go

     var(
          SchemeBuilder runtime.SchemeBuilder
          localSchemeBuilder = &SchemeBuilder
          AddToScheme = localSchemeBuilder.AddToScheme
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/authentication/v1/types_swagger_doc_generated.go

     var map_TokenReview = map[string]string{
          "":"",
          ...
     }
     var map_TokenReviewSpec= map[string]string{
          "":"",
          ...
     }
     var map_TokenReviewStatus= map[string]string{
          "":"",
          ...
     }
     var map_UserInfo= map[string]string{
          "":"",
          ...
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/authentication/v1/generated.pb.go

     func init(){
          proto.RegisterType((*ExtraValue)(nil), "k8s.io.client-go.pkg.apis.authentication.v1.ExtraValue")
          ...
     }
     func init(){
          proto.RegisterFile("k8s.io/client-go/pkg/apis/authentication/v1/generated.proto", fileDescriptorGenerated)
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/authentication/v1/register.go

     func init(){
          localSchemeBuilder.Register(addKnownTypes, addDefaultingFuncs, addConversionFuncs)
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/authentication/v1/zz_generated.conversion.go

     func init(){
          SchemeBuilder.Register(RegisterConversions)
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/authentication/v1/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

__________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/authentication/v1beta1**

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/authentication/v1beta1/generated.pb.go

     var(
          ErrInvalidLengthGenerated = fmt.Errorf("proto: negative length found during unmashaling")
          ErrIntOverflowGenerated = fmt.Errorf("prot: integer overflow")
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/authentication/v1beta1/register.go

     var(
          SchemeBuilder runtime.SchemeBuilder
          localSchemeBuilder = &SchemeBuilder
          AddToScheme = localSchemeBuilder.AddToScheme
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/authentication/v1beta1/types.generated.go

     var(
          codecSelferBitsize1234 = uint8(reflect.TypeOf(uint(0)).Bits())
          codecSelferOnlyMapOrArrayEncodeToStructErr1234 = errors.New(`only encoded map or array can be decoded into a struct`)
     )
     
k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/authentication/v1beta1/types_swagger_doc_generated.go

     var map_TokenReview = map[string]string{
          "":"",
          ...
     }     
     var map_TokenReviewSpec= map[string]string{
          "":"",
          ...
     }
     var map_TokenReviewStatus= map[string]string{
          "":"",
          ...
     }
     var map_UserInfo= map[string]string{
          "":"",
          ...
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/authentication/v1beta1/generated.pb.go

     func init(){
          proto.RegisterType((*ExtraValue)(nil), "k8s.io.client-go.pkg.apis.authentication.v1beta1.ExtraValue")
          ...
     }
     func init(){
          proto.RegisterFile("k8s.io/client-go/pkg/apis/authentication/v1beta1/generated.proto", fileDescriptorGenerated)
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/authentication/v1beta1/register.go

     func init(){
          localSchemeBuilder.Register(addKnownTypes, addDefaultingFuncs, addConversionFuncs)
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/authentication/v1beta1/types.generated.go

     func init(){
          if codec1978.GenVesion != 5{
               _,file,_,_:=runtime.Caller(0)
               err:=fmt.Errorf("codecgen version mismatch: current: %v, need %v. Re-generate file: %v",5,codec1978.GenVersion, file)
               panic(err)
          }
          if false{
               var v0 pkg1_v1.TypeMeta
               var v1 pkg2_types.UID
               var v2 time.Time
               _,_,_=v0,v1,v2
          }
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/authentication/v1beta1/zz_generated.conversion.go

     func init(){
          SchemeBuilder.Register(RegisterConversions)
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/authentication/v1beta1/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

___________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/authorization**

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/authorization/register.go

     var(
          SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)
          AddToScheme = SchemeBuilder.AddToScheme
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/authorization/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

___________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/authorization/v1**

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/authorization/v1/generated.pb.go

     var(
          ErrInvalidLengthGenerated = fmt.Errorf("proto: negative length found during unmarshaling")
          ErrIntOverflowGenerated = fmt.Errorf("proto: integer overflow")
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/authorization/v1/register.go

     var(
          SchemeBuilder runtime.SchemeBuilder
          localSchemeBuilder = &SchemeBuilder
          AddToScheme = localSchemeBuilder.AddToScheme
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/authorization/v1/types.generated.go

     var(
          codecSelferBitsize1234 = uint8(reflect.TypeOf(uint(0)).Bits())
          codecSelferOnlyMapOrArrayEncodeToStructErr1234 = errors.New(`only encoded map or array can be decoded into a struct`)
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/authorization/v1/types_swagger_doc_generated.go

     var map_LocalSubjectAccessReview = map[string]string{
          "":"",
          ...
     }
     var map_NonResourceAttributes= map[string]string{
          "":"",
          ...
     }
     var map_ResourceAttributes= map[string]string{
          "":"",
          ...
     }
     var map_SelfSubjectAccessReview= map[string]string{
          "":"",
          ...
     }
     var map_SelfSubjectAccessReviewSpec= map[string]string{
          "":"",
          ...
     }
     var map_SubjectAccessReview= map[string]string{
          "":"",
          ...
     }
     var map_SubjectAccessReviewSpec= map[string]string{
          "":"",
          ...
     }
     var map_SubjectAccessReviewStatus= map[string]string{
          "":"",
          ...
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/authorization/v1/generated.pb.go

     func init(){
          proto.RegisterType((*ExtraValue)(nil), "k8s.io.client-go.pkg.apis.authorization.v1.ExtraValue")
          ...
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/authorization/v1/generated.pb.go

     func init(){
          proto.RegisterFile("k8s.io/client-go/pkg/apis/authorization/v1/generated.proto", fileDescriptorGenerated)
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/authorization/v1/register.go

     func init(){
          localSchemeBuilder.Register(addKnownTypes, addDefaultingFuncs, addConversionFuncs)
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/authorization/v1/types.generated.go

     func init(){
          if codec1978.GenVersion != 5{
               _,file,_,_:=runtime.Caller(0)
               err:=fmt.Errorf("codecgen version mismatch: current: %v, need %v. Re-generate file: %v",5,codec1978.GenVersion,file)
               panic(err)
          }
          if false{
               var v0 pkg1_v1.TypeMeta
               var v1 pkg2_types.UID
               var v2 time.Time
               _,_,_=v0,v1,v2
          }
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/authorization/v1/zz_generated.conversion.go

     func init(){
          SchemeBuilder.Register(RegisterConversions)
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/authorization/v1/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

_______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/authorization/v1beta1**

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/authorization/v1beta1/generated.pb.go

     var(
          ErrInvalidLengthGenerated = fmt.Errorf("proto: negative length found during unmarshaling")
          ErrIntOverflowGenerated = fmt.Errorf("proto: integer overflow")
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/authorization/v1beta1/register.go

     var(
          SchemeBuilder runtime.SchemeBuilder
          localSchemeBuilder = &SchemeBuilder
          AddToScheme = localSchemeBuilder.AddToScheme
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/authorization/v1beta1/types.generated.go

     var(
          codecSelferBitsize1234 = uint8(reflect.TypeOf(uint(0)).Bits())
          codecSelferOnlyMapOrArrayEncodeToStructErr1234 = errors.New(`only encoded map or array can be decoded into a struct`)
     )

     
k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/authorization/v1beta1/types_swagger_doc_generated.go

     var map_LocalSubjectAccessReview = map[string]string{
          "":"",
          ...
     }
     var map_NonresourceAttributes= map[string]string{
          "":"",
          ...
     }
     var map_ResourceAttributes= map[string]string{
          "":"",
          ...
     }
     var map_SelfSubjectAccessReview= map[string]string{
          "":"",
          ...
     }
     var map_SelfSubjectAccessReviewSpec= map[string]string{
          "":"",
          ...
     }
     var map_SubjectAccessReview= map[string]string{
          "":"",
          ...
     }
     var map_SubjectAccessReviewSpec= map[string]string{
          "":"",
          ...
     }
     var map_SubjectAccessReviewStatus= map[string]string{
          "":"",
          ...
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/authorization/v1beta1/generated.pb.go

     func init(){
          proto.RegisterType((*ExtraValue)(nil), "k8s.io.client-go.pkg.apis.authorization.v1beta1.ExtraValue")
          ...
     }
     func init(){
          proto.RegisterFile("k8s.io/client-go/pkg/apis/authorization/v1beta1/generated.proto", fileDescriptorGenerated)
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/authorization/v1beta1/register.go

     func init(){
          localSchemeBuilder.Register(addKnownTypes, addDefaultingFuncs, addConversionFuncs)
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/authorization/v1beta1/types.generated.go

     func init(){
          if codec1978.GenVersion != 5{
               _,file,_,_:= runtime.Caller(0)
               err:=fmt.Errorf("codecgen version mismatch: current: %v, need %v. Re-generate file: %v,5,codec1978.GenVersion,file)
               panic(err)
          }
          if false{
               var v0 pkg1_v1.TypeMeta
               var v1 pkg2_types.UID
               var v2 time.Time
               _,_,_=v0,v1,v2
          }
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/authorization/v1beta1/zz_generated.conversion.go

     func init(){
          SchemeBuilder.Register(RegisterConversions)
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/authorization/v1beta1/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

_____________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/autoscaling**

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/autoscaling/register.go

     var(
          SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)
          AddToScheme = SchemeBuilder.AddToScheme
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/autoscaling/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

____________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/autoscaling/v1**

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/autoscaling/v1/generated.pb.go

     var(
          ErrInvalidLengthGenerated = fmt.Errorf("proto: negative length found during unmarshaling")
          ErrIntOverflowGenerated = fmt.Errorf("proto: integer overflow")
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/autoscaling/v1/register.go

     var(
          SchemeBuilder runtime.SchemeBuilder
          localSchemeBuilder = &SchemeBuilder
          AddToScheme = localSchemeBuilder.AddToScheme
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/autoscaling/v1/types.generated.go

     var(
          codecSelferBitsize1234 = uint8(reflect.TypeOf(uint(0)).Bits())
          codecSelferOnlyMapOrArrayEncodeToStructErr1234 = errors.New(`only encoded map or array can be decoded into a struct`)
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/autoscaling/v1/types_swagger_doc_generated.go

     var map_CrossVersionObjectReference = map[string]string{
          "":"",
          ...
     }
     var map_HorizontalPodAutoscaler= map[string]string{
          "":"",
          ...
     }
     var map_HorizontalPodAutoscalerCondition= map[string]string{
          "":"",
          ...
     }
     var map_HorizontalAutoscalerList= map[string]string{
          "":"",
          ...
     }
     var map_HorizontalPodAutoscalerSpec= map[string]string{
          "":"",
          ...
     }
     var map_MetricSpec= map[string]string{
          "":"",
          ...
     }
     var map_MetricStatus= map[string]string{
          "":"",
          ...
     }
     var map_ObjectMetricSource= map[string]string{
          "":"",
          ...
     }
     var map_ObjectMetricStatus= map[string]string{
          "":"",
          ...
     }
     var map_PodMetricSource= map[string]string{
          "":"",
          ...
     }

     var map_PodMetricStatus= map[string]string{
          "":"",
          ...
     }
     var map_PodMetricSource= map[string]string{
          "":"",
          ...
     }
     var map_ResourceMetricStatus= map[string]string{
          "":"",
          ...
     }
     var map_Scale= map[string]string{
          "":"",
          ...
     }
     var map_ScaleSpec= map[string]string{
          "":"",
          ...
     }
     var map_ScaleStatus= map[string]string{
          "":"",
          ...
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/autoscaling/v1/generated.pb.go

     func init(){
          proto.RegisterType((*CrossVersionObjectReference)(nil), "k8s.io.client-go.pkg.apis.autoscaling.v1.CrossVersionObjectReference")
          ...
     }

     func init(){
          proto.RegisterFile("k8s.io/client-go/pkg/apis/autoscaling/v1/generated.proto", fileDescriptorGenerated)
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/autoscaling/v1/register.go

     func init(){
          localSchemeBuilder.Register(addKnownTypes, addDefaultingFuncs, addConversionFuncs)
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/autoscaling/v1/types.generated.go

     func init(){
          if codec1978.GenVersion != 5{
               _,file,_,_:=runtime.Caller(0)
               err:=fmt.Errorf("codecgen version mismatch: current: %v, need %v. Re-generate file: %v",5,codec1978.GenVersion,file)
               panic(err)
          }
          if false{
               var v0 pkg3_resource.Quantity
               var v1 pkg1_v1.Time
               var v2 pkg2_types.UID
               var v3 pkg4_v1.ResourceName
               var v4 time.Time
               _,_,_,_,_=v0,v1,v2,v3,v4
          }
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/autoscaling/v1/zz_generated.conversion.go

     func init(){
          SchemeBuilder.Register(RegisterConversions)
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/autoscaling/v1/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

_____________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/autoscaling/v2alpha1**

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/autoscaling/v2alpha1/generated.pb.go

     var(
          ErrInvalidLengthGenerated = fmt.Errorf("proto: negative length found during unmarshaling")
          ErrIntOverflowGenerated = fmt.Errorf("proto: integer overflow")
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/autoscaling/v2alpha1/register.go

     var(
          SchemeBuilder runtime.SchemeBuilder
          localSchemeBuilder = &SchemeBuilder
          AddToScheme = localSchemeBuilder.AddToScheme
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/autoscaling/ v2alpha1/types.generated.go

     var(
          codecSelferBitsize1234 = uint8(reflect.TypeOf(uint(0)).Bits())
          codecSelferOnlyMapOrArrayEncodeToStructErr1234 = errors.New(`only encoded map or array can be decoded into a struct`)
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/autoscaling/ v2alpha1/types_swagger_doc_generated.go

     var map_CrossVersionObjectReference = map[string]string{
          "":"",
          ...
     }
     var map_HorizontalPodAutoscaler= map[string]string{
          "":"",
          ...
     }
     var map_HorizontalPodAutoscalerCondition= map[string]string{
          "":"",
          ...
     }
     var map_HorizontalAutoscalerList= map[string]string{
          "":"",
          ...
     }
     var map_HorizontalPodAutoscalerSpec= map[string]string{
          "":"",
          ...
     }
     var map_HorizontalPodAutoscalerSatus= map[string]string{
          "":"",
          ...
     }
     var map_MetricSpec= map[string]string{
          "":"",
          ...
     }
     var map_MetricStatus= map[string]string{
          "":"",
          ...
     }
     var map_ObjectMetricSource= map[string]string{
          "":"",
          ...
     }
     var map_ObjectMetricStatus= map[string]string{
          "":"",
          ...
     }
     var map_PodMetricSource= map[string]string{
          "":"",
          ...
     }

     var map_PodMetricStatus= map[string]string{
          "":"",
          ...
     }
     var map_PesourceMetricSource= map[string]string{
          "":"",
          ...
     }
     var map_ResourceMetricStatus= map[string]string{
          "":"",
          ...
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/autoscaling/v2alpha1/generated.pb.go

     func init(){
          proto.RegisterType((*CrossVersionObjectReference)(nil), "k8s.io.client-go.pkg.apis.autoscaling. v2alpha1.CrossVersionObjectReference")
          ...
     }

     func init(){
          proto.RegisterFile("k8s.io/client-go/pkg/apis/autoscaling/ v2alpha1/generated.proto", fileDescriptorGenerated)
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/autoscaling/ v2alpha1/register.go

     func init(){
          localSchemeBuilder.Register(addKnownTypes, addDefaultingFuncs)
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/autoscaling/ v2alpha1/types.generated.go

     func init(){
          if codec1978.GenVersion != 5{
               _,file,_,_:=runtime.Caller(0)
               err:=fmt.Errorf("codecgen version mismatch: current: %v, need %v. Re-generate file: %v",5,codec1978.GenVersion,file)
               panic(err)
          }
          if false{
               var v0 pkg1_resource.Quantity
               var v1 pkg3_v1.Time
               var v2 pkg4_types.UID
               var v3 pkg2_v1.ResourceName
               var v4 time.Time
               _,_,_,_,_=v0,v1,v2,v3,v4
          }
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/autoscaling/ v2alpha1/zz_generated.conversion.go

     func init(){
          SchemeBuilder.Register(RegisterConversions)
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/autoscaling/ v2alpha1/zz_generated_deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

_______________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/batch/register.go**

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/batch/register.go

     var(
          SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)
          AddToScheme = SchemeBuilder.AddToScheme
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/batch/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

_____________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/batch/v1**

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/batch/v1/generated.pb.go

     var(
          ErrInvalidLengthGenerated = fmt.Errorf("proto: negative length found during unmarshaling")
          ErrIntOverflowGenerated = fmt.Errorf("proto: integer overflow")
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/batch/v1/register.go

     var(
          SchemeBuilder runtime.SchemeBuilder
          localSchemeBuilder &SchemeBuilder
          AddToScheme = localSchemeBuilder.AddToScheme
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/batch/v1/types.generated.go

     var(
          codecSelferBitsize1234 = uint8(reflect.TypeOf(uint(0)).Bits())
          codecSelferOnlyMapOrArrayEncodeToStructErr1234 = errors.New(`only encoded map or array can be decoded into a struct`)
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/batch/v1/types_swagger_doc_generated.go

     var map_Job = map[string]string{
          "":"",
          ...
     }
     var map_JobCondition= map[string]string{
          "":"",
          ...
     }
     var map_JobList = map[string]string{
          "":"",
          ...
     }
     var map_JobSpec = map[string]string{
          "":"",
          ...
     }
     var map_JobStatus = map[string]string{
          "":"",
          ...
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/batch/v1/generated.pb.go

     func init(){
          proto.RegisterType((*Job)(nil), "k8s.io.client-go.pkg.apis.batch.v1.Job")
          ...
     }
     func init(){
          proto.RegisterFile("k8s.io/client-go/pkg/apis/batch/v1/generated.proto", fileDescriptorGenerated)
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/batch/v1/register.go

     func init(){
          localSchemeBuilder.Register(addKnownTypes, addDefaultingFuncs, addConversionFuncs)
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/batch/v1/types.generated.go

     func init(){
          if codec1978.GenVersion != 5{
               _,file,_,_:=runtime.Caller(0)
               err:=fmt.Errorf("codecgen version mismatch: current: %v, need %v. Re-generate file: %v",5,codec1978.GenVersion,file)
               panic(err)
          }
          if false{
               var v0 pkg4_resource.Quantity
               var v1 pkg1_v1.TypeMeta
               var v2 pkg2_types.UID
               var v3 pkg5_intstr.IntOrString
               var v4 pkg3_v1.PodTemplateSpec
               var v5 time.Time
               _,_,_,_,_,_=v0,v1,v2,v3,v4,v5
          }
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/batch/v1/zz_generated.conversion.go

     func init(){
          SchemeBuilder.Register(RegisterConversions)
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/batch/v1/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/batch/v2alpha1**

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/batch/v2alpha1/generated.pb.go

     var(
          ErrInvalidLengthGenerated = fmt.Errorf("proto: negative length found during unmarshaling")
          ErrIntOverflowGenerated = fmt.Errorf("proto: integer overflow")
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/batch/v2alpha1/register.go

     var(
          SchemeBuilder runtime.SchemeBuilder
          localSchemeBuilder = &SchemeBuilder
          AddToScheme = localSchemeBuilder.AddToScheme
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/batch/v2alpha1/types.generated.go

     var(
          codecSelferBitsize1234 = uint8(reflect.TypeOf(uint(0)).Bits())
          codecSelferOnlyMapOrArrayEncodeToStructErr1234 = errors.New(`only encoded map or array can be decoded into a struct`)
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/batch/v2alpha1/types_swagger_doc_generated.go

     var map_CronJob = map[string]string{
          "":"",
          ...
     }
     var map_CronJobList = map[string]string{
          "":"",
          ...
     }
     var map_CronJobSpec = map[string]string{
          "":"",
          ...
     }
     var map_CronJobStatus = map[string]string{
          "":"",
          ...
     }
     var map_JobTemplate= map[string]string{
          "":"",
          ...
     }
     var map_JobTemplateSpec= map[string]string{
          "":"",
          ...
     }

     var map_JobTemplateSpec= map[string]string{
          "":"",
          ...
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/batch/v2alpha1/generated.pb.go

     func init(){
          proto.RegisterType((*CronJob)(nil), "k8s.io.client-go.pkg.apis.batch.v2alpha1.CronJob")
          ...
     }
     func init(){
          proto.RegisterFile("k8s.io/client-go/pkg/apis/batch/v2alpha1/generated.proto", fileDescriptorGenerated)
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/batch/v2alpha1/register.go

     func init(){
          localSchemeBuilder.Register(addKnownTypes, addDefaultingFuncs, addConversionFuncs)
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/batch/v2alpha1/types.generated.go

     func init(){
          if codec1978.GenVersion != 5{
               _,file,_,_:=runtime.Caller(0)
               err:=fmt.Errorf("codecgen version mismatch: current: %v, need %v. Re-generate file: %v",5,codec1978.GenVersion,file)
               panic(err)
          }
          if false{
               var v0 pkg5_resource.Quantity
               var v1 pkg1_v1.TypeMeta
               var v2 pkg2_types.UID
               var v3 pkg6_intstr.IntOrString
               var v4 pkg4_v1.PodTemplateSpec
               var v5 pkg3_v1.JobSpec
               var v6 time.Time
               _,_,_,_,_,_,_=v0,v1,v2,v3,v4,v5,v6
          }
     }
     
k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/batch/v2alpha1/zz_generated.conversion.go

     func init(){
          SchemeBuilder.Register(RegisterConversions)
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/batch/v2alpha1/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

___________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/certificates**

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/certificates/register.go

     var(
          SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)
          AddToScheme = SchemeBuilder.AddToScheme
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/certificates/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/certificates/v1beta1**

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/certificates/v1beta1/generated.pb.go

     var(
          ErrInvalidLengthGenerated = fmt.Errorf("proto: negative length found during unmarshaling")
          ErrIntOverflowGenerated = fmt.Errorf("proto: integer overflow")
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/certificates/v1beta1/register.go

     var(
          SchemeBuilder runtime.SchemeBuilder
          localSchemeBuilder = &SchemeBuilder
          AddToScheme = localSchemeBuilder.AddToScheme
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/certificates/v1beta1/types.generated.go

     var(
          codecSelferBitsize1234 = uint8(reflect.TypeOf(uint(0)).Bits())
          codecSelferOnlyMapOrArrayEncodeToStructErr1234 = errors.New(`only encoded map or array can be decoded into a struct`)
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/certificates/v1beta1/types_swagger_doc_generated.go

     var map_CertificateSigningRequest = map[string]string{
          "":"",
          ...
     }
     var map_CertificateSigningRequestCondition= map[string]string{
          "":"",
          ...
     }
     var map_CertificateSigningRequestSpec= map[string]string{
          "":"",
          ...
     }
     var map_CertificateSigningRequestStatus= map[string]string{
          "":"",
          ...
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/certificates/v1beta1/generated.pb.go

     func init(){
          proto.RegisterType((*CertificateSigningRequest)(nil), "k8s.io.client-go.pkg.apis.certificates.v1beta1.CertificateSigningRequest")
          ...
     }
     func init(){
          proto.RegisterFile("k8s.io/client-go/pkg/apis/certificates/v1beta1/generated.proto", fileDescriptorGenerated)
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/certificates/v1beta1/register.go

     func init(){
          localSchemeBuilder.Register(addKnownTypes, addConversionFuncs, addDefaultingFuncs)
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/certificates/v1beta1/types.generated.go

     func init(){
          if codec1978.GenVersion != 5{
               _,file,_,_:=runtime.Caller(0)
               err:=fmt.Errorf("codecgen version mismatch: current: %v, need %v. Re-generate file: %v",5,codec1978.GenVersion,file)
               panic(err)
          }
          if false{
               var v0 pkg1_v1.TypeMeta
               var v1 pkg2_types.UID
               var v2 time.Time
               _,_,_=v0,v1,v2
          }
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/certificates/v1beta1/zz_generated.conversion.go

     func init(){
          SchemeBuilder.Register(RegisterConversions)
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/certificates/v1beta1/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

________________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/networking**

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/networking/register.go

     var(
          SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)
          AddToScheme = SchemeBuilder.AddToScheme
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/networking/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

_______________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/extensions/v1beta1**

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/extensions/v1beta1/generated.pb.go

     var(
          ErrInvalidLengthGenerated = fmt.Errorf("proto: negative length found during unmarshaling")
          ErrIntOverflowGenerated = fmt.Errorf("proto: integer overflow")
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/extensions/v1beta1/register.go

     var(
          SchemeBuilder runtime.SchemeBuilder
          localSchemeBuilder = &SchemeBuilder
          AddToScheme = localSchemeBuilder.AddToScheme
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/extensions/v1beta1/types.generated.go

     var(
          codecSelferBitsize1234 = uint8(reflect.TypeOf(uint(0)).Bits())
          codecSelferOnlyMapOrArrayEncodeToStructErr1234 = errors.New(`only encoded map or array can be decode into a struct`)
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/extensions/v1beta1/types_swagger_doc_generated.go

     var map_APIVersion = map[string]string{
          "":"",
          ...
     }
     var map_CustomMetricCurrentStatus= map[string]string{
          "":"",
          ...
     }
     var map_CustomMetricTarget= map[string]string{
          "":"",
          ...
     }
     var map_DaemonSet= map[string]string{
          "":"",
          ...
     }
     var map_DaemonSetList= map[string]string{
          "":"",
          ...
     }
     var map_DaemonSetSpec= map[string]string{
          "":"",
          ...
     }
     var map_DaemonSetStatus= map[string]string{
          "":"",
          ...
     }
     var map_DaemonSetUpdateStrategy= map[string]string{
          "":"",
          ...
     }
     var map_Deployment= map[string]string{
          "":"",
          ...
     }
     var map_DeploymentCondition= map[string]string{
          "":"",
          ...
     }
     var map_DeploymentList= map[string]string{
          "":"",
          ...
     }
     var map_DeploymentRollback= map[string]string{
          "":"",
          ...
     }
     var map_DeploymentSpec= map[string]string{
          "":"",
          ...
     }
     var map_DeploymentStatus= map[string]string{
          "":"",
          ...
     }

     var map_DeploymentStrategy= map[string]string{
          "":"",
          ...
     }

     var map_FSGroupStrategyOptions= map[string]string{
          "":"",
          ...
     }

     var map_HTTPIngressPath= map[string]string{
          "":"",
          ...
     }

     var map_HTTPIngressRuleValue= map[string]string{
          "":"",
          ...
     }
     var map_HostPortRange= map[string]string{
          "":"",
          ...
     }
     var map_IDRange= map[string]string{
          "":"",
          ...
     }

     var map_Ingress= map[string]string{
          "":"",
          ...
     }
     var map_IngressBackend= map[string]string{
          "":"",
          ...
     }
     var map_IngressList= map[string]string{
          "":"",
          ...
     }
     var map_IngressRule= map[string]string{
          "":"",
          ...
     }
     var map_IngressRuleValue= map[string]string{
          "":"",
          ...
     }
     var map_IngressSpec= map[string]string{
          "":"",
          ...
     }
     var map_IngressStatus= map[string]string{
          "":"",
          ...
     }
     var map_IngressTLS= map[string]string{
          "":"",
          ...
     }
     var map_NetworkPolicy= map[string]string{
          "":"",
          ...
     }
     var map_NetworkPolicyIngressRule= map[string]string{
          "":"",
          ...
     }
     var map_NetworkPolicyList= map[string]string{
          "":"",
          ...
     }
     var map_NetworkPolicyPeer= map[string]string{
          "":"",
          ...
     }
     var map_NetworkPolicyPort= map[string]string{
          "":"",
          ...
     }
     var map_NetworkPolicySpec= map[string]string{
          "":"",
          ...
     }
     var map_PodSecurityPolicy= map[string]string{
          "":"",
          ...
     }
     var map_PodSecurityPolicyList= map[string]string{
          "":"",
          ...
     }
     var map_PodSecurityPolicySpec= map[string]string{
          "":"",
          ...
     }
     var map_ReplicaSet= map[string]string{
          "":"",
          ...
     }
     var map_ReplicaSetCondition= map[string]string{
          "":"",
          ...
     }
     var map_ReplicaSetList= map[string]string{
          "":"",
          ...
     }
     var map_ReplicaSetSpec= map[string]string{
          "":"",
          ...
     }
     var map_ReplicaSetStatus= map[string]string{
          "":"",
          ...
     }
     var map_ReplicationControllerDummy= map[string]string{
          "":"",
          ...
     }
     var map_RollbackConfig= map[string]string{
          "":"",
          ...
     }
     var map_RollingUpdateDaemonSet= map[string]string{
          "":"",
          ...
     }
     var map_RollingUpdateDeployment= map[string]string{
          "":"",
          ...
     }
     var map_RunAsUserStrategyOptions= map[string]string{
          "":"",
          ...
     }
     var map_SELinuxStrategyOptions= map[string]string{
          "":"",
          ...
     }
     var map_Scale= map[string]string{
          "":"",
          ...
     }
     var map_ScaleSpec= map[string]string{
          "":"",
          ...
     }
     var map_ScaleStatus= map[string]string{
          "":"",
          ...
     }
     var map_SupplementalGroupsStrategyOptions= map[string]string{
          "":"",
          ...
     }
     var map_ThirdPartyResource= map[string]string{
          "":"",
          ...
     }
     var map_ThirdPartyResourceData= map[string]string{
          "":"",
          ...
     }
     var map_ThirdPartyResourceDataList= map[string]string{
          "":"",
          ...
     }
     var map_ThirdPartyResourceList= map[string]string{
          "":"",
          ...
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/extensions/v1beta1/generated.pb.go

     func init(){
          proto.RegisterType((*APIVersion)(nil), "k8s.io.client-go.pkg.apis.extensions.v1beta1.APIVersion")
          ...
     }
     func init(){
          proto.RegisterFile("k8s.io/client-go/pkg/apis/extensions/v1beta1/generated.proto", fileDescriptorGenerated)
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/extensions/v1beta1/register.go

     func init(){
          localSchemeBuilder.Register(addKnownTypes, addDefaultingFuncs, addConversionFuncs)
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/extensions/v1beta1/types.generated.go

     func init(){
          if codec1978.GenVersion != 5{
               _,file,_,_:=runtime.Caller(0)
               err:=fmt.Errorf("condecgen version mismatch: current: %v, need %v. Re-generate file: %v", 5, codec1978.GenVersion,file)
               panic(err)
          }
          if false{
               var v0 pkg3_resource.Quantity
               var v1 pkg1_v1.TypeMeta
               var v2 pkg2_types.UID
               var v3 pkg5_intstr.IntOrString
               var v4 pkg4_v1.PodTemplateSpec
               var v5 time.Time
               _,_,_,_,_,_=v0,v1,v2,v3,v4,v5
          }
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/extensions/v1beta1/zz_generated.conversion.go

     func init(){
          SchemeBuilder.Register(RegisterConversions)
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/extensions/v1beta1/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/networking/v1**

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/networking/v1/generated.pb.go

     var(
          ErrInvalidLengthGenerated = fmt.Errorf("proto: negative length found during unmarshaling")
          ErrIntOverflowGenerated = fmt.Errorf("proto: integer overflow")
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/networking/v1/register.go

     var(
          SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes, addDefaultingFuncs, addConversionFuncs)
          AddToScheme = SchemeBuilder.AddToScheme
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/networking/v1/types.generated.go

     var(
          codecSelferBitsize1234 = uint8(reflect.TypeOf(uint(0)).Bits())
          codecSelferOnlyMapOrArrayEncodeToStructErr1234 = errors.New(`only encoded map or array can be decoded into a struct`)
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/networking/v1/types_swagger_doc_generated.go

     var map_NetworkPolicy = map[string]string{
          "":"",
          ...
     }
     var map_NetworkPolicyIngressRule= map[string]string{
          "":"",
          ...
     }
     var map_NetworkPolicyList= map[string]string{
          "":"",
          ...
     }
     var map_NetworkPolicyPeer = map[string]string{
          "":"",
          ...
     }
     var map_NetworkPolicyPort = map[string]string{
          "":"",
          ...
     }
     var map_NetworkPolicySpec = map[string]string{
          "":"",
          ...
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/networking/v1/generated.pb.go

     func init(){
          proto.RegisterType((*NetworkPolicy)(nil), "k8s.io.client-go.pkg.apis.networking.v1.NetworkPolicy")
          ...
     }
     func init(){
          proto.RegisterFile("k8s.io/client-go/pkg/apis/networking/v1/generated.proto", fileDescriptorGenerated)
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/networking/v1/types.generated.go

     func init(){
          if codec1978.GenVersion != 5{
               _,file,_,_:=runtime.Caller(0)
               err:=fmt.Errorf("condecgen version mismatch: current: %v, need %v. Re-generate file: %v", 5, codec1978.GenVersion,file)
               panic(err)
          }
          if false{
               var v0 pkg1_v1.TypeMeta
               var v1 pkg2_types.UID
               var v2 pkg4_intstr.IntOrString
               var v3 pkg3_v1.Protocol
               var v4 time.Time
               _,_,_,_,_=v0,v1,v2,v3,v4
          }
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/networking/v1/zz_generated.conversion.go

     func init(){
          SchemeBuilder.Register(RegisterConversions)
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/networking/v1/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/policy**

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/policy/register.go

     var(
          SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)
          AddToScheme = SchemeBuilder.AddToScheme
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/policy/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/policy/v1beta1**

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/policy/v1beta1/generated.pb.go

     var(
          ErrInvalidLengthGenerated = fmt.Errorf("proto: negative length found during unmarshaling")
          ErrIntOverflowGenerated = fmt.Errorf("proto: integer overflow")
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/policy/v1beta1/register.go

     var(
          SchemeBuilder = runtime.SchemeBuilder
          localSchemeBuilder = &SchemeBuilder
          AddToScheme = localSchemeBuilder.AddToScheme
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/policy/v1beta1/types.generated.go

     var(
          codecSelferBitsize1234 = uint8(reflect.TypeOf(uint(0)).Bits())
          codecSelferOnlyMapOrArrayEncodeToStructErr1234 = errors.New(`only encoded map or array can be decoded into a struct`)
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/policy/v1beta1/types_swagger_doc_generated.go

     var map_Eviction= map[string]string{
          "":"",
          ...
     }
     var map_PodDisruptionBudget= map[string]string{
          "":"",
          ...
     }
     var map_PodDisruptionBudgetList= map[string]string{
          "":"",
          ...
     }
     var map_PodDisruptionBudgetSpec= map[string]string{
          "":"",
          ...
     }
     var map_PodDisruptionBudgetStatus= map[string]string{
          "":"",
          ...
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/policy/v1beta1/generated.pb.go

     func init(){
          proto.RegisterType((*Eviction)(nil), "k8s.io.client-go.pkg.apis.policy.v1beta1.Eviction")
          ...
     }
     func init(){
          proto.RegisterFile("k8s.io/client-go/pkg/apis/policy/v1beta1/generated.proto", fileDescriptorGenerated)
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/policy/v1beta1/register.go

     func init(){
          localSchemeBuilder.Register(addKnownTypes, RegisterDefaults)
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/policy/v1beta1/types.generated.go

     func init(){
          if codec1978.GenVersion != 5{
               _,file,_,_:=runtime.Caller(0)
               err:=fmt.Errorf("condecgen version mismatch: current: %v, need %v. Re-generate file: %v", 5, codec1978.GenVersion,file)
               panic(err)
          }
          if false{
               var v0 pkg2_v1.LabelSelector
               var v1 pkg3_types.UID
               var v2 pkg1_intstr.IntOrString
               var v3 time.Time
               _,_,_,_=v0,v1,v2,v3
          }
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/policy/v1beta1/zz_generated.conversion.go

     func init(){
          SchemeBuilder.Register(RegisterConversions)
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/policy/v1beta1/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

_____________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/rbac**

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/rbac/register.go

     var(
          SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)
          AddToScheme = SchemeBuilder.AddToScheme
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/rbac/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

_____________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/rbac/v1alpha1**

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/rbac/v1alpha1/generated.pb.go

     var(
          ErrInvalidLengthGenerated = fmt.Errorf("proto: negative length found during unmarshaling")
          ErrIntOverflowGenerated = fmt.Errorf("proto: integer overflow")
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/rbac/v1alpha1/register.go

     var(
          SchemeBuilder = runtime.SchemeBuilder
          localSchemeBuilder = &SchemeBuilder
          AddToScheme = localSchemeBuilder.AddToScheme
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/rbac/v1alpha1/types.generated.go

     var(
          codecSelferBitsize1234 = uint8(reflect.TypeOf(uint(0)).Bits())
          codecSelferOnlyMapOrArrayEncodeToStructErr1234 = errors.New(`only encoded map or array can be decoded into a struct`)
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/rbac/v1alpha1/types_swagger_doc_generated.go

     var map_ClusterRole= map[string]string{
          "":"",
          ...
     }
     var map_ClusterRoleBinding= map[string]string{
          "":"",
          ...
     }
     var map_ClusterRoleBindingList= map[string]string{
          "":"",
          ...
     }
     var map_ClusterRoleList= map[string]string{
          "":"",
          ...
     }
     var map_PolicyRule= map[string]string{
          "":"",
          ...
     }
     var map_Role= map[string]string{
          "":"",
          ...
     }
     var map_RoleBinding= map[string]string{
          "":"",
          ...
     }
     var map_RoleBindingList= map[string]string{
          "":"",
          ...
     }
     var map_RoleList= map[string]string{
          "":"",
          ...
     }
     var map_RoleRef= map[string]string{
          "":"",
          ...
     }
     var map_Subject= map[string]string{
          "":"",
          ...
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/rbac/v1alpha1/generated.pb.go

     func init(){
          proto.RegisterType((*ClusterRole)(nil), "k8s.io.client-go.pkg.apis.rbac.v1alpha1.ClusterRole")
          ...
     }
     func init(){
          proto.RegisterFile("k8s.io/client-go/pkg/apis/rbac/v1alpha1/generated.proto", fileDescriptorGenerated)
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/rbac/v1alpha1/register.go

     func init(){
          localSchemeBuilder.Register(addKnownTypes, RegisterDefaults)
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/rbac/v1alpha1/types.generated.go

     func init(){
          if codec1978.GenVersion != 5{
               _,file,_,_:=runtime.Caller(0)
               err:=fmt.Errorf("condecgen version mismatch: current: %v, need %v. Re-generate file: %v", 5, codec1978.GenVersion,file)
               panic(err)
          }
          if false{
               var v0 pkg1_v1.TypeMeta
               var v1 pkg2_types.UID
               var v2 time.Time
               _,_,_=v0,v1,v2
          }
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/rbac/v1alpha1/zz_generated.conversion.go

     func init(){
          SchemeBuilder.Register(RegisterConversions)
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/rbac/v1alpha1/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/rbac/v1beta1**

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/rbac/v1beta1/generated.pb.go

     var(
          ErrInvalidLengthGenerated = fmt.Errorf("proto: negative length found during unmarshaling")
          ErrIntOverflowGenerated = fmt.Errorf("proto: integer overflow")
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/rbac/v1beta1/register.go

     var(
          SchemeBuilder = runtime.SchemeBuilder
          localSchemeBuilder = &SchemeBuilder
          AddToScheme = localSchemeBuilder.AddToScheme
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/rbac/v1beta1/types.generated.go

     var(
          codecSelferBitsize1234 = uint8(reflect.TypeOf(uint(0)).Bits())
          codecSelferOnlyMapOrArrayEncodeToStructErr1234 = errors.New(`only encoded map or array can be decoded into a struct`)
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/rbac/v1beta1/types_swagger_doc_generated.go

     var map_ClusterRole= map[string]string{
          "":"",
          ...
     }
     var map_ClusterRoleBinding= map[string]string{
          "":"",
          ...
     }
     var map_ClusterRoleBindingList= map[string]string{
          "":"",
          ...
     }
     var map_ClusterRoleList= map[string]string{
          "":"",
          ...
     }
     var map_PolicyRule= map[string]string{
          "":"",
          ...
     }
     var map_Role= map[string]string{
          "":"",
          ...
     }
     var map_RoleBinding= map[string]string{
          "":"",
          ...
     }
     var map_RoleBindingList= map[string]string{
          "":"",
          ...
     }
     var map_RoleList= map[string]string{
          "":"",
          ...
     }
     var map_RoleRef= map[string]string{
          "":"",
          ...
     }
     var map_Subject= map[string]string{
          "":"",
          ...
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/rbac/v1beta1/generated.pb.go

     func init(){
          proto.RegisterType((*ClusterRole)(nil), "k8s.io.client-go.pkg.apis.rbac.v1beta1.ClusterRole")
          ...
     }
     func init(){
          proto.RegisterFile("k8s.io/client-go/pkg/apis/rbac/v1beta1/generated.proto", fileDescriptorGenerated)
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/rbac/v1beta1/register.go

     func init(){
          localSchemeBuilder.Register(addKnownTypes, RegisterDefaults)
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/rbac/v1beta1/types.generated.go

     func init(){
          if codec1978.GenVersion != 5{
               _,file,_,_:=runtime.Caller(0)
               err:=fmt.Errorf("condecgen version mismatch: current: %v, need %v. Re-generate file: %v", 5, codec1978.GenVersion,file)
               panic(err)
          }
          if false{
               var v0 pkg1_v1.TypeMeta
               var v1 pkg2_types.UID
               var v2 time.Time
               _,_,_=v0,v1,v2
          }
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/rbac/v1beta1/zz_generated.conversion.go

     func init(){
          SchemeBuilder.Register(RegisterConversions)
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/rbac/v1beta1/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

__________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/settings**

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/settings/register.go

     var(
          SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)
          AddToScheme = SchemeBuilder.AddToScheme
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/settings/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

__________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/settings/v1alpha1**

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/settings/v1alpha1/generated.pb.go

     var(
          ErrInvalidLengthGenerated = fmt.Errorf("proto: negative length found during unmarshaling")
          ErrIntOverflowGenerated = fmt.Errorf("proto: integer overflow")
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/settings/v1alpha1/register.go

     var(
          SchemeBuilder = runtime.SchemeBuilder
          localSchemeBuilder = &SchemeBuilder
          AddToScheme = localSchemeBuilder.AddToScheme
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/settings/v1alpha1/types_swagger_doc_generated.go

     var map_PodPreset= map[string]string{
          "":"",
          ...
     }
     var map_PodPresetList= map[string]string{
          "":"",
          ...
     }
     var map_PodPresetSpec= map[string]string{
          "":"",
          ...
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/settings/v1alpha1/generated.pb.go

     func init(){
          proto.RegisterType((*PodPreset)(nil), "k8s.io.client-go.pkg.apis.settings.v1alpha1.PodPreset")
          ...
     }
     func init(){
          proto.RegisterFile("k8s.io/client-go/pkg/apis/settings/v1alpha1/generated.proto", fileDescriptorGenerated)
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/settings/v1alpha1/register.go

     func init(){
          localSchemeBuilder.Register(addKnownTypes, RegisterDefaults)
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/settings/v1alpha1/zz_generated.conversion.go

     func init(){
          SchemeBuilder.Register(RegisterConversions)
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/settings/v1alpha1/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

____________________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/storage**

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/storage/register.go

     var(
          SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)
          AddToScheme = SchemeBuilder.AddToScheme
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/storage/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

_________________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/storage/v1**

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/storage/v1/generated.pb.go

     var(
          ErrInvalidLengthGenerated = fmt.Errorf("proto: negative length found during unmarshaling")
          ErrIntOverflowGenerated = fmt.Errorf("proto: integer overflow")
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/storage/v1/register.go

     var(
          SchemeBuilder = runtime.SchemeBuilder
          localSchemeBuilder = &SchemeBuilder
          AddToScheme = localSchemeBuilder.AddToScheme
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/storage/v1/types_swagger_doc_generated.go

     var map_StorageClass= map[string]string{
          "":"",
          ...
     }
     var map_StorageClassList= map[string]string{
          "":"",
          ...
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/storage/v1/generated.pb.go

     func init(){
          proto.RegisterType((*StorageClass)(nil), "k8s.io.client-go.pkg.apis.storage.v1.StorageClass")
          ...
     }
     func init(){
          proto.RegisterFile("k8s.io/client-go/pkg/apis/storage/v1/generated.proto", fileDescriptorGenerated)
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/storage/v1/register.go

     func init(){
          localSchemeBuilder.Register(addKnownTypes, RegisterDefaults)
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/storage/v1/zz_generated.conversion.go

     func init(){
          SchemeBuilder.Register(RegisterConversions)
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/storage/v1/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

__________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/storage/v1beta1**

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/storage/v1beta1/generated.pb.go

     var(
          ErrInvalidLengthGenerated = fmt.Errorf("proto: negative length found during unmarshaling")
          ErrIntOverflowGenerated = fmt.Errorf("proto: integer overflow")
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/storage/v1beta1/register.go

     var(
          SchemeBuilder = runtime.SchemeBuilder
          localSchemeBuilder = &SchemeBuilder
          AddToScheme = localSchemeBuilder.AddToScheme
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/storage/v1beta1/types.generated.go

     var(
          codecSelferBitsize1234 = uint8(reflect.TypeOf(uint(0)).Bits())
          codecSelferOnlyMapOrArrayEncodeToStructErr1234 = errors.New(`only encoded map or array can be decoded into a struct`)
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/storage/v1beta1/types_swagger_doc_generated.go

     var map_StorageClass= map[string]string{
          "":"",
          ...
     }
     var map_StorageClassList= map[string]string{
          "":"",
          ...
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/storage/v1beta1/generated.pb.go

     func init(){
          proto.RegisterType((*StorageClass)(nil), "k8s.io.client-go.pkg.apis.storage.v1beta1.StorageClass")
          ...
     }
     func init(){
          proto.RegisterFile("k8s.io/client-go/pkg/apis/storage/v1beta1/generated.proto", fileDescriptorGenerated)
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/storage/v1beta1/register.go

     func init(){
          localSchemeBuilder.Register(addKnownTypes, RegisterDefaults)
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/storage/v1beta1/types.generated.go

     func init(){
          if codec1978.GenVersion != 5{
               _,file,_,_:=runtime.Caller(0)
               err:=fmt.Errorf("condecgen version mismatch: current: %v, need %v. Re-generate file: %v", 5, codec1978.GenVersion,file)
               panic(err)
          }
          if false{
               var v0 pkg1_v1.TypeMeta
               var v1 pkg2_types.UID
               var v2 time.Time
               _,_,_=v0,v1,v2
          }
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/storage/v1beta1/zz_generated.conversion.go

     func init(){
          SchemeBuilder.Register(RegisterConversions)
     }

k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/apis/storage/v1beta1/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

___________________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/client-go/kubernetes/scheme**

k8s.io/kubernetes/vendor/k8s.io/client-go/kubernetes/scheme/register.go

     var Scheme = runtime.NewScheme()
     var Codec = serializer.NewCodecFactory(Scheme)
     var ParameterCodec = runtime.NewParameterCodec(Scheme)

     func init(){
          v1.AddToGroupVersion(Scheme, schema.GroupVersion{Version: "v1"})
          AddToScheme(Scheme)
     }

______________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/client-go/tools/cache**

k8s.io/kubernetes/vendor/k8s.io/client-go/tools/cache/delta_fifo.go

     var(
          ErrZeroLengthDeltasObject = errors.New("0 length Deltas object; can't get key")
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/tools/cache/fifo.go

     var FIFOClosedError error = errors.New("DeltaFIFO: manipulating with closed queue")

k8s.io/kubernetes/vendor/k8s.io/client-go/tools/cache/reflector.go

     var stackCreator = regexp.MustCompile(`(?m)^created by (.*)\n\s+(.*):(\d+) \+0x[[:xdigit:]]+$`)

     var(
          neverExitWatch <-chan time.Time = make(chan time.Time)
          errorResyncRequested = errors.New("resync channel fired")
          errorStopRequested = errors.New("Stop requested")
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/tools/cache/mutation_detector.go

     func init(){
          mutationDetectionEnabled,_=strconv.ParseBool(os.Getenv("KUBE_CACHE_MUTATION_DETECTOR"))
     }

____________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/pborman/uuid**

k8s.io/kubernetes/vendor/github.com/pborman/uuid/hash.go

     var(
          NameSpace_DNS = Parse("6ba7b810-9dad-11d1-80b4-00c04fd430c8")
          Namespace_URL = Parse("6ba7b811-9dad-11d1-80b4-00c04fd430c8")
          Namespace_OID = Parse("6ba7b811-9dad-11d1-80b4-00c04fd430c8")
          Namespace_X500 = Parse("6ba7b811-9dad-11d1-80b4-00c04fd430c8")
          NIL = Parse("00000000-0000-0000-0000-000000000000")
     )

k8s.io/kubernetes/vendor/github.com/pborman/uuid/uuid.go

     var rander = rand.Reader

__________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/apis/apiserver**

k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/apis/apiserver/register.go

     var(
          SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)
          AddToScheme = SchemeBuilder.AddToScheme
     )

k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/apis/apiserver/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

_______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/apis/apiserver/v1alpha1**

k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/apis/apiserver/v1alpha1/register.go

     var(
          SchemeBuilder runtime.SchemeBuilder
          localSchemeBuilder = &SchemeBuilder
          AddToScheme = localSchemeBuilder.AddToScheme
     )
     func init(){
          localSchemeBuilder.Register(addKnownTypes)
     }

k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/apis/apiserver/v1alpha1/zz_generated.conversion.go

     func init(){
          SchemeBuilder.Register(RegisterConversions)
     }

k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/apis/apiserver/v1alpha1/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

___________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/admission**

k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/admission/config.go

     var(
          groupFactoryRegistry = make(announced.APIGroupFactoryRegistry)
          registry = registered.NewOrDie(os.Getenv("KUBE_API_VERSIONS"))
          scheme = runtime.NewScheme()
          codecs = serializer.NewCodecFactory(scheme)
     )

     func init(){
          install.Install(groupFactoryRegistry, registry, scheme)
     }

__________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/apis/audit**

k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/apis/audit/register.go

     var(
          SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)
          AddToScheme = SchemeBuilder.AddToScheme
     )

k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/apis/audit/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/golang/protobuf/proto**

k8s.io/kubernetes/vendor/github.com/golang/protobuf/proto/decode.go

     var errOverflow = errors.New("proto: integer overflow")
     var ErrInternalBadWireType = errors.New("proto: internal error: bad wiretype for oneof")

k8s.io/kubernetes/vendor/github.com/golang/protobuf/proto/encode.go

     var(
          errRepeatedHasNil = errors.New("proto: repeated field has nil element")
          errOneofHasNil = errors.New("proto: oneof field has nil value")
          ErrNil = errors.New("proto: Marshal called with nil")
          ErrTooLarge = errors.New("proto: message encodes to over 2GB")
     )

k8s.io/kubernetes/vendor/github.com/golang/protobuf/proto/extensions.go

     var ErrMissingExtension = errors.New("proto: missing extension")
     var extendableProtoType = reflect.TypeOf((*extendableProto)(nil)).Elem()
     var extendableProtoV1Type = reflect.TypeOf((*extendableProtoV1)(nil)).Elem()

     var extProp = struct{
          sync.RWMutex
          m map[extPropKey]*Properties
     }{
          m: make(map[extPropKey]*Properties),
     }
     var extensionMaps = make(map[reflect.Type]map[int32]*ExtensionDesc)

k8s.io/kubernetes/vendor/github.com/golang/protobuf/proto/lib.go

     var(
          defaultMu sync.RWMutex
          defaults = make(map[reflect.Type]defaultMessage)
          int32PtrType = reflect.TypeOf((*int32)(nil))
     )

k8s.io/kubernetes/vendor/github.com/golang/protobuf/proto/message_set.go

     var errNoMessageTypeID = errors.New("proto does not have a message type ID")
     var messageSetMap = make(map[int32]messageSetDesc)

k8s.io/kubernetes/vendor/github.com/golang/protobuf/proto/properties.go

     var protoMessageType = reflect.TypeOf((*Message)(nil)).Elem()
     var(
          marshalerType = reflect.TypeOf((*Marshaler)(nil)).Elem()
          unmarshalerType = reflect.TypeOf((*Unmarshaler)(nil)).Elem()
     )
     var(
          propertiesMu sync.RWMutex
          propertiesMap = make(map[reflect.Type]*StructProperties)
     )
     var enumValueMaps = make(map[string]map[string]int32)
     var(
          protoTypes = make(map[string]reflect.Type)
          revProtoTypes = make(map[reflect.Type]string)
     )
     var(
          protoFiles = make(map[string][]byte)
     )

k8s.io/kubernetes/vendor/github.com/golang/protobuf/proto/text_parser.go

     var(
          errBadUTF8 = errors.New("proto: bad UTF-8")
          errBadHex = errors.New("proto: bad hexadecimal")
     )

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/prometheus/common/model**

k8s.io/kubernetes/vendor/github.com/prometheus/common/model/labels.go

     var LabelNameRE = regexp.MustCompile("^[a-zA-Z_][a-zA-Z0-9_]*$")

k8s.io/kubernetes/vendor/github.com/prometheus/common/model/metric.go

     var(
          separator = []byte{0}
          MetricNameRE = regexp.MustCompile(`^[a-zA-Z_:][a-zA-Z0-9_:]*$`)
     )

k8s.io/kubernetes/vendor/github.com/prometheus/common/model/signature.go

     var(
          emptyLabelSignature = hashNew()
     )

k8s.io/kubernetes/vendor/github.com/prometheus/common/model/time.go

     var dotPrecision = int(math.Log10(float64)second)))
     var durationRE = regexp.MustCompile("^([0-9]+)(y|w|d|h|m|s|ms)$")

____________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/prometheus/client_model/go**

k8s.io/kubernetes/vendor/github.com/prometheus/client_model/go/metrics.pb.go

     var MetricType_name = map[int32]string{
          0: "COUNTER",
          1: "GAUGE",
          2: "SUMMARY",
          3: "UNTYPED",
          4: "HISTOGRAM",
     }
     var MetricType_value = map[string]int32{
          "COUNTER": 0,
          "GAUGE": 1,
          "SUMMARY": 2,
          "UNTYPED": 3,
          "HISTOGRAM": 4,
     }
     func init(){
          proto.RegisterEnum("io.prometheus.client.MetricType", MetricType_name, MetricType_value)
     }

__________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/matttproud/golang_protobuf_extensions/pbutil**

k8s.io/kubernetes/vendor/github.com/matttproud/golang_protobuf_extensions/pbutil/decode.go

     var errInvalidVarint = errors.New("invalid varint32 encountered")

____________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/prometheus/common/expfmt**

k8s.io/kubernetes/vendor/github.com/prometheus/common/expfmt/text_create.go

     var(
          escape = strings.NewReplacer("\\", `\\`, "\n", `\n`)
          escapeWithDoubleQuote = strings.NewReplacer("\\", `\\`, "\n", `\n`, "\"", `\"`)
     )

___________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/prometheus/procfs**

k8s.io/kubernetes/vendor/github.com/prometheus/procfs/mdstat.go

     var(
          statuslineRE = regexp.MustCompile(`(\d+) blocks .*\[(\d+)/(\d+)\] \[[U_]+]`)
          buildlineRE = regexp.MustCompile(`\((\d+)/\d+\)`)
     )

k8s.io/kubernetes/vendor/github.com/prometheus/procfs/proc_limits.go

     var(
          limitsDelimiter = regexp.MustCompile(" +")
     )

_____________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/prometheus/client_golang/prometheus**

k8s.io/kubernetes/vendor/github.com/prometheus/client_golang/prometheus/histogram.go

     var(
          DefBuckets = []float64{.005, .01, .025, .05, .1, .25, .5, .1, 2.5, 5, 10}
          errBucketLabelNotAllowed = fmt.Errorf(
               "%q is not allowed as label name in histograms", bucketLabel,
          )
     )

k8s.io/kubernetes/vendor/github.com/prometheus/client_golang/prometheus/register.go

     var(
          defaultRegistry = NewRegistry()
          DefaultRegisterer Register = defaultRegistry
          DefaultGatherer Gatherer = defaultRegistry
     )

k8s.io/kubernetes/vendor/github.com/prometheus/client_golang/prometheus/summary.go

     var(
          DefObjectives = map[float64]float64{0.5: 0.05, 0.9: 0.01, 0.99: 0.001}
          errQuantileLabelNotAllowed = fmt.Errorf(
               "%q is not allowed as label name in summaries", quantileLabel,
          )
     )

k8s.io/kubernetes/vendor/github.com/prometheus/client_golang/prometheus/value.go

     var errInconsistentCardinality = errors.New("inconsistent label cardinality")

k8s.io/kubernetes/vendor/github.com/prometheus/client_golang/prometheus/registry.go

     func init(){
          MustRegister(NewProcessCollector(os.Getpid(), ""))
          MustRegister(NewGoCollector())
     }

_______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/apis/audit/v1alpha1**

k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/apis/audit/v1alpha1/generated.pb.go

     var(
          ErrInvalidLengthGenerated = fmt.Errorf("proto: negative length found during unmarshaling")
          ErrIntOverflowGenerated = fmt.Errorf("proto: integer overflow")
     )

k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/apis/audit/v1alpha1/register.go

     var(
          SchemeBuilder runtime.SchemeBuilder
          localSchemeBuilder = &SchemeBuilder
          AddToScheme = localSchemeBuilder.AddToScheme
     )

k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/apis/audit/v1alpha1/types.generated.go

     var(
          codecSelferBitsize1234 = uint8(reflect.TypeOf(uint(0)).Bits())
          codecSelferOnlyMapOrArrayEncodeToStructErr1234 = errors.New(`only encoded map or array can be decoded into a struct`)
     )

k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/apis/audit/v1alpha1/generated.pb.go

     func init(){
          proto.RegisterType((*Event)(nil), "k8s.io.apiserver.pkg.apis.audit.v1alpha1.Event")
          ...
     }
     func init(){
          proto.RegisterFile("k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/apis/audit/v1alpha1/generated.proto", fileDescriptorGenerated)
     }

k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/apis/audit/v1alpha1/register.go

     func init(){
          localSchemeBuilder.Register(addKnownTypes)
     }

k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/apis/audit/v1alpha1/types.generated.go

     func init(){
          if codec1978.GenVersion != 5{
               _,file,_,_:=runtime.Caller(0)
               err:=fmt.Errorf("codecgen version mismatch: current: %v, need %v. Re-generate file: %v",5,codec1978.GenVersion,file)
               panic(err)
          }
          if false{
               var v0 pkg1_v1.TypeMeta
               var v1 pkg4_runtime.Unknown
               var v2 pkg2_types.UID
               var v3 pkg3_v1.UserInfo
               var v4 time.Time
               _,_,_,_,_=v0,v1,v2,v3,v4
          }
     }

k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/apis/audit/v1alpha1/zz_generated.conversion.go

     func init(){
          SchemeBuilder.Register(RegisterConversions)
     }

k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/apis/audit/v1alpha1/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/audit**

k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/audit/metrics.go

     var(
          eventCounter = prometheus.NewCounter(
               prometheus.CounterOpts{
                    Subsystem: subsystem,
                    Name: "event_count",
                    Help: "Counter of audit events generated and sent to the audit backend.",
               })
          errorCounter = prometheus.NewCounterVec(
               prometheus.CounterOpts{
                    Subsystem: subsystem,
                    Name: "error_count",
                    Help: "Counter of audit events that failed to be audited property. " +
                         "Plugin identifies the plugin affected by the error.",
               },
               []string{"plugin"},
          )
          levelCounter = prometheus.NewCounterVec(
               prometheus.CounterOpts{
                    Subsystem: subsystem,
                    Name: "level_count",
                    Help: "Counter of policy levels for audit events (1 per request).",
               },
               []string{"level"},
          )
     )

k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/audit/scheme.go

     var Scheme = runtime.NewScheme()
     var Codecs = serializer.NewCodecFactory(Scheme)

k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/audit/metrics.go

     func init(){
          prometheus.MustRegister(eventCounter)
          prometheus.MustRegister(errorCounter)
          prometheus.MustRegister(levelCounter)
     }

k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/audit/scheme.go

     func init(){
          v1.AddToGroupVersion(Scheme, schema.GroupVersion{Version: "v1"})
          v1alpha1.AddToScheme(Scheme)
     }

__________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/authentication/request/bearertoken**

k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/authentication/request/bearertoken/bearertoken.go

     var invalidToken = errors.New("invalid bearer token")

____________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/howeyc/gopass**

k8s.io/kubernetes/vendor/github.com/howeyc/gopass/pass.go

     var(
          maxLength = 512
          ErrInterrupted = errors.New("interrupted")
          ErrMaxLengthExceeded = fmt.Errorf("maximum byte limit (%v) exceeded", maxLength)
          getch = defaultGetCh
     )

_____________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/imdario/mergo**

k8s.io/kubernetes/vendor/github.com/imdario/mergo/mergo.go

     var(
          ErrNilArguments = errors.New("src and dst must not be nil")
          ErrDifferentArgumentsTypes = errors.New("src and dst must be of same type")
          ErrNotSupported = errors.New("only structs and maps are supported")
          ErrExpectedMapAsDestination = errors.New("dst was expected to be a map")
          ErrExpectedStructAsDestination = errors.New("dst was expected to be a struct")
     )

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/client-go/tools/clientcmd/api/v1**

k8s.io/kubernetes/vendor/k8s.io/client-go/tools/clientcmd/api/v1/register.go

     var(
          SchemeBuilder runtime.SchemeBuilder
          localSchemeBuilder = &SchemeBuilder
          AddToScheme = localSchemeBuilder.AddToScheme
     )
     func init(){
          localSchemeBuilder.Register(addKnownTypes, addConversionFuncs)
     }

_____________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/client-go/tools/clientcmd/api/latest**

k8s.io/kubernetes/vendor/k8s.io/client-go/tools/clientcmd/api/latest/latest.go

     func init(){
          Scheme = runtime.NewScheme()
          if err:=api.AddToScheme(Scheme);err!=nil{
               panic(err)
          }
          if err:=v1.AddToScheme(Scheme); err!=nil{
               panic(err)
          }
          yamlSerializer := json.NewYAMLSerializer(json.DefaultMetaFactory, Scheme, Scheme)
          Codec = versioning.NewDefaultingCodecForScheme(
               Scheme,
               yamlSerializer,
               yamlSerializer,
               schema.GroupVersion{Version: Version},
               runtime.InternalGroupVersioner,
          )
     }

_____________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/client-go/tools/clientcmd**

k8s.io/kubernetes/vendor/k8s.io/client-go/tools/clientcmd/client_config.go

     var(
          ClusterDefaults = clientcmdapi.Cluster{Server: getDefaultServer()}
          DefaultClientConfig - DirectClientConfig{*clientcmdapi.NewConfig(), "", &ConfigOverides{
               ClusterDefaults: ClusterDefaults,
          , nil, NewDefaultClientConfigLoadingRules(), promptedCredentials{}}}
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/tools/clientcmd/loader.go

     var(
          RecommentedConfigDir = path.Join(homedir.HomeDir(), RecommendedHomeDir)
          RecommentedHomeFile = path.Join(RecommendedConfigDir, RecommendedFileName)
          RecommendtedSchemaFile = path.Join(RecommendedConfigDir, RecommendedSchemaName)
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/tools/clientcmd/client_config.go

     var(
          ClusterDefaults = clientcmdapi.Cluster{Server: getDefaultServer()}
          DefaultClientConfig = DirectClientConfig{*clientcmdapi.NewConfig(), "", &ConfigOverrides{
               ClusterDefaults: ClusterDefaults,
          }, nil, NewDefaultClientConfigLoadingRules(), promptedCredentials{}}
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/tools/clientcmd/validation.go

     var(
          ErrNoContext = errors.New("no context chosen")
          ErrEmptyConfig = errors.New("no configuration has been provided")
          ErrEmptyCluster = errors.New("cluster has no server defined")
     )

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apiserver/plugin/pkg/authenticator/token/webhook**

k8s.io/kubernetes/vendor/k8s.io/apiserver/plugin/pkg/authenticator/token/webhook/webhook.go

     var(
          groupVersions = []schema.GroupVersion{authentication.SchemeGroupVersion}
     )
     var registry = registered.NewOrDie("")
     func init(){
          registry.RegisterVersions(groupVersions)
          if err:=registry.EnableVersions(groupVersions...);err!=nil{
               panic(fmt.Sprintf("failed to enable version %v", groupVersions))
          }
     }

_____________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/golang.org/x/net/context**

k8s.io/kubernetes/vendor/golang.org/x/net/context/go17.go

     var(
          todo = context.TODO()
          background = context.Background()
     )
     var Canceled = context.Canceled
     var DeadlineExceeded = context.DeadlineExceeded

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/storage**

k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/storage/errors.go

     var errCodeToMessage = map[int]string{
          ErrCodeKeyNotFound: "key not found",
          ErrCodeKeyExists: "key exists",
          ErrCodeResourceVersionConflicts: "resource version conflicts",
          ErrCodeInvalidObj: "invalid object",
          ErrCodeUnreachable: "server unreachable",
     }

k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/storage/interfaces.go

     var Everything = SelectionPredicate{
          Label: labels.Everything(),
          Field: fields.Everything(),
          IncludeUninitialized: true,
     }

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/endpoints/request**

k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/endpoints/request/requestinfo.go

     var specialVerbs = sets.NewString("proxy", "redirect","watch")
     var specialVerbsNoSubresources = sets.NewString("proxy", "redirect")
     var namespaceSubresources = sets.NewString("status", "finalize")
     var NamespaceSubResourcesForTest = sets.NewString(namespaceSubresources.List()...)

_____________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/api/validation**

k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/api/validation/objectmeta.go

     var BannedOwners = map[schema.GroupVersionKind]struct{}{
          {Group: "", Version: "v1", Kind: "Event"}: {},
     }

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/internalversion**

k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/internalversion/register.go

     var scheme = runtime.NewScheme()
     var Copier = runtime.ObjectCopier = scheme
     var Codecs = serializer.NewCodecFactory(scheme)
     var SchemeGroupVersion = schema.GroupVersion{Group: GroupName, Version: runtime.APIVersionInternal}
     var ParmeterCodec = runtime.NewParameterCodec(scheme)

     func init(){
          if err:=addToGroupVersion(scheme, SchemeGroupVersion);err!=nil{
               panic(err)
          }
     }

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/registry/rest**

k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/registry/rest/table.go

     var swaggerMetadataDescriptions = meta1.ObjectMeta{}.SwaggerDoc()

_____________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/golang.org/x/net/websocket**

k8s.io/kubernetes/vendor/golang.org/x/net/websocket/client.go

     var portMap = map[string]string{
          "ws": "80",
          "wss": "443",
     }

k8s.io/kubernetes/vendor/golang.org/x/net/websocket/hybi.go

     var(
          ErrBadMaskingKey = &ProtocolError{"bad masking key"}
          ErrBadPongMessage = &ProtocolError{"bad pong message"}
          ErrBadClosingStatus = &ProtocolError{"bad closing status"}
          ErrUnsupportedExtensions = &ProtocolError{"unsupported extensions"}
          ErrNotImplemented = &ProtocolError{"not implemented"}
          handshakeHeader = map[string]bool{
               "Host": true,
               "Upgrade": true,
               "Connection": true,
               "Sec-Websocket-Key": true,
               "Sec-Websocket-Origin": true,
               "Sec-Websocket-Version": true,
               "Sec-Websocket-Protocol": true,
               "Sec-Websocket-Accept": true,
          }
     )

k8s.io/kubernetes/vendor/golang.org/x/net/websocket/websocket.go

     var ErrFrameTooLarge = errors.New("websocket: frame payload size exceeds limit")
     var errSetDeadline = errors.New("websocket: cannot set deadline: not using a net.Conn")

_______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/util/wsstream**

k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/util/wsstream/conn.go

     var(
          connectionUpgradeRegex = regexp.MustCompile("(^|.*,\\s*)upgrade($|\\s*,)")
     )

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/endpoints/handlers/responsewriters**

k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/endpoints/handlers/responsewriters/errors.go

     var sanitizer = strings.NewReplacer(`&`, "&amp;", `<`, "&lt;", `>`, "&gt;")

_______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/authentication/serviceaccount**

k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/authentication/serviceaccount/util.go

     var invalidUsernameErr = fmt.Errorf("Username must be in the form %s", MakeUsername("namespace", "name"))

_______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/endpoints/filters**

k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/endpoints/filters/authentication.go

     var(
          authenticatedUserCounter = prometheus.NewCounterVec(
               prometheus.CounterOpts{
                    Name: "authenticated_user_requests",
                    Help: "Counter of authenticated requests broken out by username.",
               },
               []string{"username"},
          )
     )
     func init(){
          prometheus.MustRegister(authenticatedUserCounter)
     }

____________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/endpoints/openapi**

k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/endpoints/openapi/openapi.go

     var verbs = trie.New([]string{"get", "log", "read", "replace", "patch", "delete", "deletecollection", "watch", "connect", "proxy", "list", "create", "patch"})

___________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/util/feature**

k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/util/feature/feature_gate.go

     var(
          defaultFeatures = map[Feature]FeatureSpec{
               allAlphaGate: {Default: false, PreRelease: Alpha},
          }
          specialFeatures = map[Feature]func(f *featureGate, val bool){
               allAlphaGate: setUnsetAlphaGates,
          }
          DefaultFeatureGate FeatureGate = NewFeatureGate()
     )

___________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/features**

k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/features/kube_features.go

     var defaultKubernetesFeatureGates = map[utilfeature.Feature]utilfeature.FeatureSpec{
          StreamingProxyRedirects: {Default: true, PreRelease: utilfeature.Beta},
          AdvancedAuditing: {Default: false, PreRelease: utilfeature.Alpha},
     }
     func init(){
          utilfeature.DefaultFeatureGate.Add(defaultKubernetesFeatureGates)
     }

___________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/coreos/etcd/client**

k8s.io/kubernetes/vendor/github.com/coreos/etcd/client/client.go

     var(
          ErrNoEndpoints = errors.New("client: no endpoints available")
          ErrTooManyRedirects = errors.New("client: too many redirects")
          ErrClusterUnavailable = errors.New("client: etcd cluster is unavailable or misconfigured")
          ErrNoLeaderEndpoint = errors.New("clint: no leader endpoint available")
          errTooManyRedirectChecks = errors.New("client: too many redirect checks")
          oneShotCtxValue interface{}
     )
     var DefaultRequestTimeout = 5 * time.Second
     var DefaultTransport CancelableTransport = &http.Transport{
          Proxy: http.ProxyFromEnvironment,
          Dial: (&net.Dialer{
               Timeout: 30*time.Second,
               KeepAlive: 30*time.Second,
          }).Dial,
          TLSHandshakeTimeout: 10*time.Second,
     }

k8s.io/kubernetes/vendor/github.com/coreos/etcd/client/keys.generated.go

     var(
          codecSelferBitsize1819 = uint8(reflect.TypeOf(uint(0)).Bits())
          codecSelferOnlyMapOrArrayEncodeToStructErr1819 = errors.New(`only encoded map or array can be decoded into a struct`)
     )

k8s.io/kubernetes/vendor/github.com/coreos/etcd/client/keys.go

     var(
          ErrInvalidJSON = errors.New("client: response is invalid json. The endpoint is probably not valid etcd cluster endpoint.")
          ErrEmptyBody = errors.New("client: response body is empty")
     )

k8s.io/kubernetes/vendor/github.com/coreos/etcd/client/keys.generated.go

     func init(){
          if codec1978.GenVersion!=5{
               _,file,_,_:=runtime.Caller(0)
               err:=fmt.Errorf("codecgen version mismatch: current: %v, need %v. Re-generate file: %v",5,code1978.GenVersion, file)
               panic(err)
          }
          if false{
               var v0 time.Time
               _=v9
          }
     }

k8s.io/kubernetes/vendor/github.com/coreos/etcd/client/util.go

     func init(){
          roleNotFoundRegExp = regexp.MustCompile("auth: Role .* does not exist.")
          userNotFoundRegExp = regexp.MustCompile("auth: User .* does not exist.")
     }

__________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/coreos/go-systemd/journal**

k8s.io/kubernetes/vendor/github.com/coreos/go-systemd/journal/journal.go

     func init(){
          var err error
          conn, err=net.Dial("unixgram", "/run/systemd/journal/socket")
          if err!=nil{
               conn = nil
          }
     }

__________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/coreos/pkg/capnslog**

k8s.io/kubernetes/vendor/github.com/coreos/pkg/capnslog/glog_formatter.go

     var pid = os.Getpid()

k8s.io/kubernetes/vendor/github.com/coreos/pkg/capnslog/logmap.go

     var logger = new(loggerStruct)

k8s.io/kubernetes/vendor/github.com/coreos/pkg/capnslog/init.go

     func init(){
          initHijack()
          SetFormatter(NewDefaultFormatter(os.Stderr))
          SetGlobalLogLevel(INFO)
     }

_________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/coreos/pkg/fileutil**

k8s.io/kubernetes/vendor/github.com/coreos/pkg/fileutil/fileutil.go

     var(
          plog = capnslog.NewPackageLogger("github.com/coreos/etcd", "pkg/fileutil")
     )

k8s.io/kubernetes/vendor/github.com/coreos/pkg/fileutil/lock.go

     var(
          ErrLocked = errors.New("fileutil: file already locked")
     )

k8s.io/kubernetes/vendor/github.com/coreos/pkg/fileutil/lock_linux.go

     func init(){
          getlk:=syscall.Flock_t{Type: syscall.F_RDLCK}
          if err:=syscall.FcntFlock(0, F_OFD_GETLK, &getlk); err==nil{
               linuxTryLockFile = ofdTryLockFile
               linuxLockFile = ofdLockFile
          }
     }

________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/coreos/etcd/pkg/transport**

k8s.io/kubernetes/vendor/github.com/coreos/etcd/pkg/transport/limit_listen.go

     var(
          ErrNotTCP = errors.New("only tcp connections have keepalive")
     )

______________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/storage/etcd/metrics**

k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/storage/etcd/metrics/metrics.go

     var(
          cacheHitCounterOpts = prometheus.CounterOpts{
               Name: "etcd_helper_cache_hit_count",
               Help: "Counter of etcd helper cache hits.",
          }
          cacheHitCounter = prometheus.NewCounter(cacheHitCounterOpts)
          cacheMissCounterOpts = prometheus.CounterOpts{
               Name: "etcd_helper_cache_miss_count",
               Help: "Counter of etcd helper cache miss.",
          }
          cacheMissCounter = prometheus.NewCounter(cacheMissCounterOpts)
          cacheEntryCounterOpts = prometheus.CounterOpts{
               Name: "etcd_helper_cache_entry_count",
               Help: "Counter of etcd helper cache entries. This can be different from etcd_helper_cache_miss_count"+
                    "because two concurrent threads can miss the cache and generate the same entry twice.",
          }
          cacheEntryCounter = prometheus.NewCounter(cacheEntryCounterOpts)
          cacheGetLatency = prometheus.NewSummary(
               prometheus.SummaryOpts{
                    Name: "etcd_request_cache_get_latencies_summary:,
                    Help: "Latency in microseconds of getting an object from etcd cache",
               },
          )
          cacheAddLatency = prometheus.NewSummary(
               prometheus.SummaryOpts{
                    Name: "etcd_request_latencies_summary",
                    Help: "Etcd request latency summary in microseconds for each operation and object type.",
               },
               []string{"operation", "type"},
          )
     )

_________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/storage/etcd**

k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/storage/etcd/etcd_helper.go

     func init(){
          metrics.Register()
     }

__________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/coreos/etcd/auth/authpb**

k8s.io/kubernetes/vendor/github.com/coreos/etcd/auth/authpb/auth.pb.go

     var Permission_Type_name = map[int32]string{
          0: "READ",
          1: "WRITE",
          2: "READWRITE",
     }
     var Permission_Type_value = map[string]int32{
          "READ": 0,
          "WRITE": 1,
          "READWRITE": 2,
     }
     var(
          ErrInvalidLengthAuth = fmt.Errorf("proto: negative length found during unmarshaling")
          ErrIntOverflowAuth = fmt.Errorf("proto: integer overflow")
     )
     func init(){
          proto.RegisterType((*User)(nil), "authpb.User")
          ...
     }
     func int(){ proto.RegisterFile("auth.proto", fileDescriptorAuth)}

____________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/coreos/etcd/mvcc/mvccpb**

k8s.io/kubernetes/vendor/github.com/coreos/etcd/mvcc/mvccpb/kv.pb.go

     var Event_EventType_name = map[int32]string{
          0: "PUT",
          1: "DELETE",
     }
     var Event_EventType_value = map[string]int32{
          "PUT": 0,
          "DELETE": 1,
     }

     var(
          ErrInvalidLengthKv = fmt.Errorf("proto: negative length found during unmarshaling")
          ErrIntOverflowKv = fmt.Errorf("proto: integer overflow")
     )

     func init(){
          proto.RegisterType((*KeyValue)(nil), "mvccpb.KeyValue")
          ...
     }
     func init(){ proto.RegisterFile("kv.proto", fileDescriptorKv)}

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/google.golang.org/grpc/credentials**

k8s.io/kubernetes/vendor/google.golang.org/grpc/credentials/credentials.go

     var(
          ErrConnDispatched = errors.New("credentials: rawConn is dispatched out of gRPC")
     )

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/google.golang.org/grpc/grpclog**

k8s.io/kubernetes/vendor/google.golang.org/grpc/grpclog/logger.go

     var logger Logger = log.New(os.Stderr, "", log.LstdFlags)

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/golang.org/x/net/trace**

k8s.io/kubernetes/vendor/golang.org/x/net/trace/events.go

     var(
          famMu sync.RWMutex
          families = make(map[string]*eventFamily)
     )
     var freeEventLogs = make(chan *eventLog, 1000)

k8s.io/kubernetes/vendor/golang.org/x/net/trace/trace.go

     var(
          activeMu sync.RWMutex
          activeTraces = make(map[string]*traceSet)
          completeMu sync.RWMutex
          completedTraces = make(map[string]*family)
     )
     var traceFreeList = make(chan *trace, 1000)

     func init(){
          http.HandlerFunc("/debug/requests", func(w http.ResponseWriter, req *http.Request){
               any,sensitive := AuthRequest(req)
               if !any {
                    http.Error(w, "not allowed", http.StatusUnauthorized)
                    return
               }
               w.Header().Set("Content-Type", "text/html; charset=utf-8")
               Render(w,req,sensitive)
          })
          http.HandlerFunc("/debug/events", func(w http.ResponseWriter, req *http.Request){
               any,sensitive := AuthRequest(req)
               if !any {
                    http.Error(w, "not allowed", http.StatusUnauthorized)
                    return
               }
               w.Header().Set("Content-Type", "text/html; charset=utf-8")
               RenderEvents(w,req,sensitive) 
          })
     }

________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/google.golang.org/grpc/transport**

k8s.io/kubernetes/vendor/google.golang.org/grpc/transport/http2_server.go

     var ErrIllegalHeaderWrite = errors.New("transport: teh stream is done or WriteHeader was already called")

k8s.io/kubernetes/vendor/google.golang.org/grpc/transport/http_util.go

     var(
          clientPreface = []byte(http2.ClientPreface)
          http2ErrConvTab = map[http2.ErrCode]codes.Code{
               http2.ErrCodeNo: codes.Internal,
               http2.ErrCodeProtocol: codes.Internal,
               http2.ErrCodeInternal: codes.Internal,
               http2.ErrCodeFlowControl: codes.ResourceExhausted,
               http2.ErrCodeSettingsTimeout: codes.Internal,
               http2.ErrCodeStreamClosed: codes.Internal,
               http2.ErrCodeFrameSize: codes.Internal,
               http2.ErrCodeRefusedStream: codes.Unavailable,
               http2.ErrCodeCancel: codes.Canceled,
               http2.ErrCodeCompression: codes.Internal,
               http2.ErrCodeConnect: codes.Internal,
               http2.ErrCodeEnhanceYourCalm: codes.ResourceExhausted,
               http2.ErrCodeInadequateSecurity: codes.PermissionDenied,
               http2.ErrCodeHTTP11Required: codes.FailedPrecondition,
          }
          statusCodeConvTab = map[codes.Code]http2.ErrCode{
               codes.Internal: http2.ErrCodeInternal,
               codes.Canceled: http2.ErrCodeCancel,
               codes.Unavailable: http2.ErrCodeRefusedStream,
               codes.ResourceExhausted: http2.ErrCodeEnhanceYourCalm,
               codes.PermissionDenied: http2.ErrCodeInadequateSecurity,
          }
     )

k8s.io/kubernetes/vendor/google.golang.org/grpc/transport/transport.go

     var(
          ErrConnClosing = connectionErrorf(true, nil, "transport is closing")
          ErrStreamDrain = streamErrorf(codes.Unavailable, "the server stops accepting new RPCs")
     )

_______________________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/google.golang.org/grpc**

k8s.io/kubernetes/vendor/google.golang.org/grpc/clientconn.go

     var(
          ErrClientConnClosing = errors.New("grpc: the client connection is closing")
          ErrClientConnTimeout = errors.New("grpc: timed out when dialing")
          errNoTransportSecurity = errors.New("grpc: no transport security set (use grpc.WithInsecure() explicitly or set credentials)")
          errTransportCredentialsMissing = errors.New("grpc: the credentials require transport level security (use grpc.WithTransportCredentials() to set)")
          errCredentialsConflict = errors.New("grpc: tansport credentials are set for an insecure connection (grpc.WithTransportCredentials() and grpc.WithInsecure() are both called)")
          errNetworkIO = errors.New("grpc: failed with network I/O error")
          errConnDrain = errors.New("grpc: the connection is drained")
          errConnClosing = errors.New("grpc: the connection is closing")
          errConnUnavailable= errors.New("grpc: the connection is unavailable")
          errNoAddr = errors.New("grpc: there is no address available to dial")
          minConnectTimeout = 20 * time.Second
     )

k8s.io/kubernetes/vendor/google.golang.org/grpc/server.go

     var(
          ErrServerStopped = errors.New("grpc: the server has been stopped")
     )
     func init(){
          internal.TestingCloseConns = func(arg interface{}){
               arg.(*Server).testingCloseConns()
          }
          internal.TestingUseHandlerImpl = func(arg interface{}){
               arg.(*Server).opts.useHandlerImpl = true
          }
     }

_________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/grpc-ecosystem/grpc-gateway/runtime/internal**

k8s.io/kubernetes/vendor/github.com/grpc-ecosystem/grpc-gateway/runtime/internal/stream_chunk.pb.go

     func init(){
          proto.RegisterType((*StreamError)(nil), "grpc.gateway.runtime.StreamError")
     }
     func init(){ proto.RegisterFile("runtime/internal/stream_chunk.proto", fileDescriptor0)}

________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/grpc-ecosystem/grpc-gateway/runtime**

k8s.io/kubernetes/vendor/github.com/grpc-ecosystem/grpc-gateway/runtime/marshal_jsonpb.go

     var typeProtoMessage = reflect.TypeOf((*proto.Message)(nil)).Elem()

k8s.io/kubernetes/vendor/github.com/grpc-ecosystem/grpc-gateway/runtime/marshaler_registry.go

     var(
          acceptHeader = http.CanonialHeaderKey("Accept")
          contentTypeHeader = http.CanonicalHeaderKey("Content-Type")
          defaultMarshaler = &JSONPb{OrigName: true}
     )

k8s.io/kubernetes/vendor/github.com/grpc-ecosystem/grpc-gateway/runtime/pattern.go

     var(
          ErrNotMatch = errors.New("not match to the path pattern")
          ErrInvalidPattern = errors.New("invalid pattern")
     )

k8s.io/kubernetes/vendor/github.com/grpc-ecosystem/grpc-gateway/runtime/query.go

     var(
          convFromType = map[reflect.Kind]reflect.Value{
               reflect.String: reflect.ValueOf(String),
               reflect.Bool: reflect.ValueOf(Bool),
               reflect.Float64: reflect.ValueOf(Float64),
               reflect.Float32: reflect.ValueOf(Float32),
               reflect.Int64: reflect.ValueOf(Int64),
               reflect.Int32: reflect.ValueOf(Int32),
               reflect.Uint64: reflect.ValueOf(Uint64),
               reflect.Uint32: reflect.ValueOf(Uint32),
          }
     )

_____________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/coreos/etcd/etcdserver/etcdserverpb**

k8s.io/kubernetes/vendor/github.com/coreos/etcd/etcdserver/etcdserverpb/etcdserver.pb.go

     var(
          ErrInvalidLengthEtcdserver = fmt.Errorf("proto: negative length found during unmarshaling")
          ErrIntOverflowEtcdserver = fmt.Errorf("proto: integer overflow")
     )

k8s.io/kubernetes/vendor/github.com/coreos/etcd/etcdserver/etcdserverpb/raft_internal.pb.go

     var(
          ErrInvalidLengthRaftInternal = fmt.Errorf("proto: negative length found during unmarshaling")
          ErrIntOverflowRaftInternal = fmt.Errorf("proto: integer overflow")
     )

k8s.io/kubernetes/vendor/github.com/coreos/etcd/etcdserver/etcdserverpb/rpc.pb.go

     var AlarmType_name = map[int32]string{
          0: "NONE",
          1: "NOSPACE",
     }
     var AlarmType_value = map[string]int32{
          "NONE": 0,
          "NOSPACE": 1,
     }
     var RangeRequest_SortOrder_name = map[int32]string{
          0: "NONE",
          1: "ASCEND",
          2: "DESCEND",
     }
     var RangeRequest_SortOrder_value = map[string]int32{
          "NONE": 0,
          "ASCEND": 1,
          "DESCEND": 2,
     }
     var RangeRequest_SortTarget_name = map[int32]string{
          0:"KEY",
          1:"VERSION",
          2:"CREATE",
          3:"MOD",
          4:"VALUE",
     }
     var RangeRequest_SortTarget_value = map[string]int32{
          "KEY": 0,
          "VERSION": 1,
          "CREATE": 2,
          "MOD": 3,
          "VALUE": 4,
     }
     var Compare_CompareResult_name = map[int32]string{
          0: "EQUAL",
          1: "GREATER",
          2: "LESS",
          3: "NOT_EQUAL",
     }
     var Compare_CompareResult_value = map[string]int32{
          "EQUAL": 0,
          "GREATER": 1,
          "LESS": 2,
          "NOT_EQUAL": 3,
     }
     var Compare_CompareTarget_name = map[int32]string{
          0: "VERSION",
          1: "CREATE",
          2: "MOD",
          3: "VALUE",
     }
     var Compare_CompareTarget_value = map[string]int32{
          "VERSION": 0,
          "CREATE": 1,
          "MOD": 2,
          "VALUE": 3,
     }
     var WatchCreateRequest_FilterType_name = map[int32]string{
          0: "NOPUT",
          1: "NODELETE",
     }
     var WatchCreateRequest_FilterType_value = map[string]int32{
          "NOPUT": 0,
          "NODELETE": 1,
     }
     var AlarmRequest_AlarmAction_name= map[int32]string{
          0: "GET",
          1: "ACTIVATE",
          2: "DEACTIVATE",
     }
     var AlarmRequest_AlarmAction_value= map[string]int32{
          "GET": 0,
          " ACTIVATE": 1,
          " DEACTIVATE": 2,
     }

     var (
          ErrInvalidLengthRpc = fmt.Errorf("proto: negative length found during unmarshaling")
          ErrIntOverflowRpc = fmt.Errorf("proto: integer overflow")
     )

k8s.io/kubernetes/vendor/github.com/coreos/etcd/etcdserver/etcdserverpb/rpc.pb.gw.go

     var(
          pattern_KV_Range_0 = runtime.MustPattern(runtime.NewPattern(1， []int{2,0,2,1,2,2},[]string{"v3alpha", "kv", "range"}, ""))
          pattern_KV_Put_0 = runtime. MustPattern(runtime.NewPattern(1, []int{2,0,2,1,2,2},[]string{"v3alpha", "kv", "put"}, ""))
          pattern_KV_DeleteRange_0 = runtime.MustPattern(runtime.NewPattern(1, []int{2,0,2,1,2,2},[]string{"v3alpha", "kv", "deleterange"}, ""))
          pattern_KV_Txn_0 = runtime.MustPattern(runtime.NewPattern(1, []int{2,0,2,1,2,2},[]string{"v3alpha", "kv", "txn"}, ""))
          pattern_KV_Compact_0 = runtime.MustPattern(runtime.NewPattern(1, []int{2,0,2,1,2,2},[]string{"v3alpha", "kv", "compaction"}, ""))
     )
     var(
          pattern_Watch_Watch_0 = runtime.MustPattern(runtime.NewPattern(1, []int{2,0,2,1},[]string{"v3alpha", "watch"}, ""))
     )
     var(
          pattern_Lease_LeaseGrant_0 = runtime.MustPattern(runtime.NewPattern(1, []int{2,0,2,1,2,2},[]string{"v3alpha", "lease", "grant"}, ""))
          pattern_Lease_LeaseRevoke_0 = runtime.MustPattern(runtime.NewPattern(1, []int{2,0,2,1,2,2,2,3},[]string{"v3alpha", "kv", "lease", "revoke"}, ""))
          pattern_Lease_LeaseKeepAlive_0 = runtime.MustPattern(runtime.NewPattern(1, []int{2,0,2,1,2,2},[]string{"v3alpha", "lease", "keepalive"}, ""))
          pattern_Lease_LeaseTimeToLive_0 = runtime.MustPattern(runtime.NewPattern(1, []int{2,0,2,1,2,2,2,3},[]string{"v3alpha", "kv", "lease", "timetolive"}, ""))
     )
     var(
          pattern_Cluster_MemberAdd_0 = runtime.MustPattern(runtime.NewPattern(1, []int{2,0,2,1,2,2,2,3},[]string{"v3alpha", "cluster", "member", "add"}, ""))
          pattern_ Cluster_MemberRemove_0 = runtime.MustPattern(runtime.NewPattern(1, []int{2,0,2,1,2,2,2,3},[]string{"v3alpha", " cluster", " member", "remove"}, ""))
          pattern_ Cluster_MemberUpdate_0 = runtime.MustPattern(runtime.NewPattern(1, []int{2,0,2,1,2,2,2,3},[]string{"v3alpha", " cluster", " member" ,"update"}, ""))
          pattern_ Cluster_MemberList_0 = runtime.MustPattern(runtime.NewPattern(1, []int{2,0,2,1,2,2,2,3},[]string{"v3alpha", "kv", " cluster", " member", "list"}, ""))
     )
     var(
          pattern_Maintenance_Alarm_0 = runtime.MustPattern(runtime.NewPattern(1， []int{2,0,2,1,2,2},[]string{"v3alpha", "maintenance", "alarm"}, ""))
          pattern_Maintenance_Status_0 = runtime. MustPattern(runtime.NewPattern(1, []int{2,0,2,1,2,2},[]string{"v3alpha", " maintenance", "status"}, ""))
          pattern_Maintenance_Defragment_0 = runtime.MustPattern(runtime.NewPattern(1, []int{2,0,2,1,2,2},[]string{"v3alpha", " maintenance", "defragment"}, ""))
          pattern_Maintenance_Hash_0 = runtime.MustPattern(runtime.NewPattern(1, []int{2,0,2,1,2,2},[]string{"v3alpha", " maintenance", "hash"}, ""))
          pattern_Maintenance_Snapshot_0 = runtime.MustPattern(runtime.NewPattern(1, []int{2,0,2,1,2,2},[]string{"v3alpha", " maintenance", "snapshot"}, ""))
     )
     var(
          pattern_Auth_AuthEnable_0 = runtime.MustPattern(runtime.NewPattern(1， []int{2,0,2,1,2,2},[]string{"v3alpha", "auth", "enable"}, ""))
          pattern_Auth_AuthDisable_0 = runtime. MustPattern(runtime.NewPattern(1, []int{2,0,2,1,2,2},[]string{"v3alpha", " auth", "disable"}, ""))
          pattern_Auth_Authenticate_0 = runtime.MustPattern(runtime.NewPattern(1, []int{2,0,2,1,2,2},[]string{"v3alpha", " auth", "authenticate"}, ""))
          pattern_Auth_UserAdd_0 = runtime.MustPattern(runtime.NewPattern(1, []int{2,0,2,1,2,2,2,3},[]string{"v3alpha", " auth", "user", "add"}, ""))
          pattern_Auth_UserGet_0 = runtime.MustPattern(runtime.NewPattern(1, []int{2,0,2,1,2,2,2,3},[]string{"v3alpha", " auth", "user", "get"}, ""))
          pattern_Auth_UserList_0 = runtime.MustPattern(runtime.NewPattern(1， []int{2,0,2,1,2,2,2,3},[]string{"v3alpha", "auth", "user", "list"}, ""))
          pattern_Auth_UserDelete_0 = runtime. MustPattern(runtime.NewPattern(1, []int{2,0,2,1,2,2,2,3},[]string{"v3alpha", " auth", "user", "delete"}, ""))
          pattern_Auth_UserChangePassword_0 = runtime.MustPattern(runtime.NewPattern(1, []int{2,0,2,1,2,2,2,3},[]string{"v3alpha", " auth", "user", "changepw"}, ""))
          pattern_Auth_UserGrantRole_0 = runtime.MustPattern(runtime.NewPattern(1, []int{2,0,2,1,2,2,2,3},[]string{"v3alpha", " auth", "user", "grant"}, ""))
          pattern_Auth_UserRevokeRole_0 = runtime.MustPattern(runtime.NewPattern(1, []int{2,0,2,1,2,2,2,3},[]string{"v3alpha", " auth", "user", "revoke"}, ""))
          pattern_Auth_RoleAdd_0 = runtime.MustPattern(runtime.NewPattern(1, []int{2,0,2,1,2,2,2,3},[]string{"v3alpha", " auth", "role", "add"}, ""))
          pattern_Auth_RoleGet_0 = runtime.MustPattern(runtime.NewPattern(1， []int{2,0,2,1,2,2,2,3},[]string{"v3alpha", "auth", " role", "get"}, ""))
          pattern_Auth_RoleList_0 = runtime. MustPattern(runtime.NewPattern(1, []int{2,0,2,1,2,2,2,3},[]string{"v3alpha", " auth", " role", "list"}, ""))
          pattern_Auth_RoleDelete_0 = runtime.MustPattern(runtime.NewPattern(1, []int{2,0,2,1,2,2,2,3},[]string{"v3alpha", " auth", " role", "delete"}, ""))
          pattern_Auth_RoleGrantPermission_0 = runtime.MustPattern(runtime.NewPattern(1, []int{2,0,2,1,2,2,2,3},[]string{"v3alpha", " auth", " role", "grant"}, ""))
          pattern_Auth_RoleRevokePermission_0 = runtime.MustPattern(runtime.NewPattern(1, []int{2,0,2,1,2,2,2,3},[]string{"v3alpha", " auth", " role", "revoke"}, ""))
     )

k8s.io/kubernetes/vendor/github.com/coreos/etcd/etcdserver/etcdserverpb/etcdserver.pb.go

     func init(){
          proto.RegisterType((*Request)(nil), "etcdserverpb.Request")
          proto.RegisterType((*Metadata)(nil), "etcdserverpb.Metadata")
     }
     func init(){ proto.RegisterFile("etcdserver.proto", fileDescriptorEtcdserver)}

k8s.io/kubernetes/vendor/github.com/coreos/etcd/etcdserver/etcdserverpb/raft_internal.pb.go

     func init(){
          proto.RegisterType((*RequestHeader)(nil), "etcdserverpb.RequestHeader")
          ...
     }
     func init(){proto.RegisterFile("raft_internal.proto", fileDescriptorRaftInternal) }

k8s.io/kubernetes/vendor/github.com/coreos/etcd/etcdserver/etcdserverpb/rpc.pb.go

     func init(){
          proto.RegisterType((*ResponseHeader)(nil), "etcdserverpb.ResponseHeader")
          ...
     }
     func init(){proto.RegisterFile("rpc.proto", fileDescriptorRpc)}

__________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/coreos/etcd/etcdserver/api/v3rpc/rpctypes**

k8s.io/kubernetes/vendor/github.com/coreos/etcd/etcdserver/api/v3rpc/rpctypes/error.go

     var(
          ErrGRPCEmptyKey = grpc.Errorf(codec.InvalidArgument, "etcdserver: key is not provided")
          ...
     )

________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/grpc-ecosystem/go-grpc-prometheus**

k8s.io/kubernetes/vendor/github.com/grpc-ecosystem/go-grpc-prometheus/client_reporter.go

     var(
          clientStartedCounter = prom.NewCounterVec(
               prom.CounterOpts{
                    Namespce: "grpc",
                    Subsystem: "client",
                    Name: "started_total",
                    Help: "Total number of RPCs started on the client.",
               }, []string{"grpc_type","grpc_service","grpc_method"})

          ...
     )

k8s.io/kubernetes/vendor/github.com/grpc-ecosystem/go-grpc-prometheus/server_reporter.go

     var(
          serverStartedCounter = prom.NewCounterVec(
               prom.CounterOpts{
                    Namespce: "grpc",
                    Subsystem: "server",
                    Name: "started_total",
                    Help: "Total number of RPCs started on the server.",
               }, []string{"grpc_type","grpc_service","grpc_method"})

          ...
     )

k8s.io/kubernetes/vendor/github.com/grpc-ecosystem/go-grpc-prometheus/client_reporter.go

     func init(){
          prom.MustRegister(clientStartedCounter)
          prom.MustRegister(clientHandledCounter)
          prom.MustRegister(clientStreamMsgReceived)
          prom.MustRegister(clientStreamMsgSent)
     }

k8s.io/kubernetes/vendor/github.com/grpc-ecosystem/go-grpc-prometheus/server_reporter.go

     func init(){
          prom.MustRegister(serverStartedCounter)
          prom.MustRegister(serverHandledCounter)
          prom.MustRegister(serverStreamMsgReceived)
          prom.MustRegister(serverStreamMsgSent)
     }

________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/coreos/etcd/clientv3**

k8s.io/kubernetes/vendor/github.com/coreos/etcd/clientv3/balancer.go

     var ErrNoAddrAvilable = grpc.Errorf(codes.Unavailable, "there is no address available")

k8s.io/kubernetes/vendor/github.com/coreos/etcd/clientv3/client.go

     var(
          ErrNoAvailableEndpoints = errors.New("etcdclient: no available endpoints")
     )

k8s.io/kubernetes/vendor/github.com/coreos/etcd/clientv3/watch.go

     var valCtxCh = make(chan struct{})
     var zeroTime = time.Unix(0,0)

k8s.io/kubernetes/vendor/github.com/coreos/etcd/clientv3/logger.go

     func init(){
          logger.mu.Lock()
          logger.l = log.New(ioutil.Discard, "", 0)
          grpclog.SetLogger(&logger)
          logger.mu.Unlock()
     }

________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/storage/etcd3**

k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/storage/etcd3/watcher.go

     var errTestingDecode = errors.New("sentinel error only used during testing to indicate watch decoding error")

k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/storage/etcd3/compact.go

     func init(){
          endpointsMap = make(map[string]struct{})
     }

k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/storage/etcd3/watcher.go

     func init(){
          fatalOnDecodeError,_=strconv.ParseBool(os.Getenv("KUBE_PANIC_WATCH_DECODE_ERROR"))
     }

________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/server/filters**

k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/server/filters/maxinflight.go

     var nonMutatingRequestVerbs = sets.NewString("get", "list", "watch")

k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/server/filters/timeout.go

     var errConnKilled = fmt.Errorf("kill connection/stream")

________________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/endpoints/metrics**

k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/endpoints/metrics/metrics.go

     var(
          requestCounter = prometheus.NewCounterVec(
               prometheus.CounterOpts{
                    Name: "apiserver_request_count",
                    Help: "Counter of apiserver requests broken out for each verb, API resource, client, and HTTP response contentType and code.",
               },
               []string{"verb", "resource", "subresource", "client", "contentType", "code"},
          )
          requestLatencies = prometheus.NewHistogramVec(
               prometheus.HistogramOpts{
                    Name: "apiserver_request_latencies",
                    Help: "Response latency distribution in microseconds for each verb, resource and client.",
                    Buckets: prometheus.ExponentialBuckets9125000，2.0，7），
               }，
               []string{"verb", "resource", "subresource"},
          )
          requestLatenciesSummary = prometheus.NewSummaryVec(
               prometheus.SummaryOpts{
                    Name: "apiserver_request_latencies_summary",
                    Help: "Response latency summary in microseconds for each verb and resource.",
                    MaxAge: time.Hour，
               }，
               []string{"verb", "resource", "subresource"},
          )
          kubectlExeRegexp = regexp.MustCompile(`^.*((?i:kubectl\.exe))`)
     )

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/golang/protobuf/ptypes/any**

k8s.io/kubernetes/vendor/github.com/golang/protobuf/ptypes/any/any.pb.go

     func init(){
          proto.RegisterType((*Any)(nil), "google.protobuf.Any")
     }
     func init(){ proto.RegisterFile("github.com/golang/protobuf/ptypes/any/any.proto", fileDescriptor0) }

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/golang/protobuf/ptypes/duration**

k8s.io/kubernetes/vendor/github.com/golang/protobuf/ptypes/duration/duration.pb.go

     func init(){
          proto.RegisterType((*Duration)(nil), "google.protobuf.Duration")
     }
     func init(){
          proto.RegisterFile("github.com/golang/protobuf/ptypes/duration/duration.proto", fileDescriptor0)
     }

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/golang/protobuf/ptypes/timestamp**

k8s.io/kubernetes/vendor/github.com/golang/protobuf/ptypes/timestamp/timestamp.pb.go

     func init(){
          proto.RegisterType((*Timestamp)(nil), "google.protobuf.Timestamp")
     }
     func init(){
          proto.RegisterFile("github.com/golang/protobuf/ptypes/timestamp/timestamp.proto", fileDescriptor0)
     }

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/googleapis/gnostic/extensions**

k8s.io/kubernetes/vendor/github.com/googleapis/gnostic/extensions/extension.pb.go

     func init(){
          proto.RegisterType((*Version)(nil), "openapiextension.v1.Version")
          proto.RegisterType((*ExtensionHandlerRequest)(nil), "openapiextension.v1.ExtensionHandlerRequest")
          proto.RegisterType((*ExtensionHandlerResponse)(nil), "openapiextension.v1.ExtensionHandlerResponse")
          proto.RegisterType((*Wrapper)(nil), "openapiextension.v1.Wrapper")
     }
     func init(){proto.RegisterFile("extension.proto", fileDescriptor0)}

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/googleapis/gnostic/OpenAPIv2**

k8s.io/kubernetes/vendor/github.com/googleapis/gnostic/OpenAPIv2/OpenAPIv2.pb.go

     func init(){
          proto.RegisterType((*AdditionalPropertiesItem)(nil), "openapi.v2.AdditionalPropertiesItem")
          ...
     }
     func init(){proto.RegisterFile("OpenAPIv2/OpenAPIv2.proto", fileDescriptor0)}

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/server/openapi**

k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/server/openapi/openapi_aggregator.go

     var cloner = conversion.NewCloner()

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/elazarl/go-bindata-assetfs**

k8s.io/kubernetes/vendor/github.com/elazarl/go-bindata-assetfs/assetfs.go

     var(
          fileTimestamp = time.Now()
     )

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/cleint-go/pkg/api/v1/ref**

k8s.io/kubernetes/vendor/k8s.io/cleint-go/pkg/api/v1/ref/ref.go

     var(
          ErrNilObject = errors.New("can't reference a nil object")
          ErrNoSelfLink = errors.New("selfLink was empty, can't make reference")
     )

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/util/mergepatch**

k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/util/mergepatch/errors.go

     var(
          ErrBadJSONDoc = errors.New("invalid JSON document")
          ...
     )

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/evanphx/json-patch**

k8s.io/kubernetes/vendor/github.com/evanphx/json-patch/merge.go

     var errBadJSONDoc = fmt.Errorf("Invalid JSON Document")
     var errBadJSONPatch = fmt.Errorf("Invalid JSON Patch")
     var(
          rfc6901Encoder = strings.NewReplacer("~", "~0", "/", "~1")
          rfc6901Decoder = strings.NewReplacer("~1", "/", "~0", "~")
     )

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apimachinery/third_party/forked/golang/netutil**

k8s.io/kubernetes/vendor/k8s.io/apimachinery/third_party/forked/golang/netutil/addr.go

     var portMap = map[string]string{
          "http": "80",
          "https": "443",
     }

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/golang.org/x/net/html**

k8s.io/kubernetes/vendor/golang.org/x/net/html/const.go

     var isSpecialElementMap = map[string]bool{
          "address": true,
          "applet": true,
          ...
     }

k8s.io/kubernetes/vendor/golang.org/x/net/html/entity.go

     var entity = map[string]rune{
          "AElig;": '\U000000C6',
          ...
     }
     var entity2 = map[string][2]rune{
          "NotEqualTilde;": {'\u2242', '\u0338'},
          ...
     }

k8s.io/kubernetes/vendor/golang.org/x/net/html/foreign.go

     var breakout = map[string]bool{
          "b": true,
          ...
     }
     var svgTagNameAdjustments = map[string]string{
          "altglyph": "altGlyph",
          ...
     }
     var mathMLAttributeAdjustments = map[string]string{
          "definitionurl": "definitionURL",
     }
     var svgAttributeAdjustments = map[string]string{
          "attributename": "attributeName",
          ...
     }

k8s.io/kubernetes/vendor/golang.org/x/net/html/parse.go

     var(
          defaultScopeStopTags = map[string][]a.Atom{
               "": {a.Applet, a.Caption, a.Html, a.Table, a.Td, a.Th, a.Marquee, a.Object, a.Template},
               "math": {a.AnnotationXml, a.Mi, a.Mn, a.Mo, a.Ms, a.Mtext},
               "svg": {a.Desc, a.ForeignObject, a.Title},
          }
     )

k8s.io/kubernetes/vendor/golang.org/x/net/html/render.go

     var plaintextAbort = errors.New("html: internal error (plaintext abort)")
     var voidElements = map[string]bool{
          "area": true,
          ...
     }

k8s.io/kubernetes/vendor/golang.org/x/net/html/token.go

     var ErrBufferExceeded = errors.New("max buffer exceeded")

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/util/proxy**

k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/util/proxy/transport.go

     var atomsToAttrs = map[atom.Atom]sets.String{
          atom.A: sets.NewString("href"),
          ...
     }

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/endpoints/handlers**

k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/endpoints/handlers/namer.go

     var errEmptyName = errors.NewBadRequest("name must be provided")

k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/endpoints/handlers/watch.go

     var neverExitWatch <-chan time.Time = make(chan time.Time)

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/endpoints**

k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/endpoints/installer.go

     var toDiscoveryKubeVerb = map[string]string{
          "CONNECT": "",
          ...
     }

k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/endpoints/apiserver.go

     func init(){
          metrics.Register()
     }

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/admission/plugin/namespace/lifecycle**

k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/admission/plugin/namespace/lifecycle/admission.go

     var accessReviewResources = map[schema.GroupResource]bool{
          {Group: "authorization.k8s.io", Resource: "localsubjectaccessreviews"}: true,
     }

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/server**

k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/server/genericapiserver.go

     var EmptyDelegate = emptyDelegate{
          requestContextMapper: apirequest.NewRequestContextMapper(),
     }

k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/server/hooks.go

     var hookNotFinished = errors.New("not finished")

k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/server/genericapiserver.go

     func init(){
          mime.AddExtensionType(".svg", "image/svg+xml")
     }

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apiserver/plugin/pkg/audit/webhook**

k8s.io/kubernetes/vendor/k8s.io/apiserver/plugin/pkg/audit/webhook/webhook.go

     var(
          groupFactoryRegistry = make(announced.APIGroupFactoryRegistry)
          groupVersions = []schema.GroupVersion{auditv1alpha1.SchemeGroupVersion}
          registry = registered.NewOrDie("")
     )
     var auditDeepCopyFuncs = []conversion.GeneratedDeepCopyFunc{
          {Fn: auditinternal.DeepCopy_audit_Event, InType: reflect.TypeOf(&auditinternal.Event{})},
          ...
     }

     func init(){
          registry.RegisterVersions(groupVersions)
          if err:=registry.EnableVersions(groupVersions...);err!=nil{
               panic(fmt.Sprintf("failed to enable version %v", groupVersions))
          }
          install.Install(groupFactoryRegistry, registry, audit.Scheme)
     }

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/registry/generic/registry**

k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/registry/generic/registry/store.go

     var(
          errAlreadyDeleting = fmt.Errorf("abort delete")
          errDeleteNow = fmt.Errorf("delete now")
          errEmptiedFinalizers = fmt.Errorf("emptied finalizers")
     )

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/server/storage**

k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/server/storage/storage_factory.go

     var specialDefaultResourcePrefixes = map[schema.GroupResource]string{
          {Group: "", Resource: "replicationControllers"}: "controllers",
          ...
     }

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/kube-aggregator/pkg/apis/apiregistration**

k8s.io/kubernetes/vendor/k8s.io/kube-aggregator/pkg/apis/apiregistration/register.go

     var(
          SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)
          AddToScheme = SchemeBuilder.AddToScheme
     )

k8s.io/kubernetes/vendor/k8s.io/kube-aggregator/pkg/apis/apiregistration/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/kube-aggregator/pkg/apis/apiregistration/v1beta1**

k8s.io/kubernetes/vendor/k8s.io/kube-aggregator/pkg/apis/apiregistration/v1beta1/generated.pb.go

     var(
          ErrInvalidLengthGenerated = fmt.Errorf("proto: negative length found during unmarshaling")
          ErrIntOverflowGenerated = fmt.Errorf("proto: integer overflow")
     )

k8s.io/kubernetes/vendor/k8s.io/kube-aggregator/pkg/apis/apiregistration/v1beta1/register.go

     var(
          SchemeBuilder runtime.SchemeBuilder
          localSchemeBuilder = &SchemeBuilder
          AddToScheme = localSchemeBuilder.AddToScheme
     )

k8s.io/kubernetes/vendor/k8s.io/kube-aggregator/pkg/apis/apiregistration/v1beta1/generated.pb.go

     func init(){
          proto.RegisterType((*APIService)(nil), "k8s.io.kube_aggregator.pkg.apis.apiregistration.v1beta1.APIService")
          ...
     }
     func init(){
          proto.RegisterFile("k8s.io/kubernetes/vendor/k8s.io/kube-aggregator/pkg/apis/apiregistration/v1beta1/generated.proto", fileDescriptorGenerated)
     }

k8s.io/kubernetes/vendor/k8s.io/kube-aggregator/pkg/apis/apiregistration/v1beta1/register.go

     func init(){
          localSchemeBuilder.Register(addKnownTypes)
     }

k8s.io/kubernetes/vendor/k8s.io/kube-aggregator/pkg/apis/apiregistration/v1beta1/zz_generated.conversion.go

     func init(){
          SchemeBuilder.Register(RegisterConversions)
     }

k8s.io/kubernetes/vendor/k8s.io/kube-aggregator/pkg/apis/apiregistration/v1beta1/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/kube-aggregator/pkg/client/clientset_generated/internalclientset/scheme**

k8s.io/kubernetes/vendor/k8s.io/kube-aggregator/pkg/client/clientset_generated/internalclientset/scheme/register.go

     var Scheme = runtime.NewScheme()
     var Codecs = serializer.NewCodecFactory(Scheme)
     var ParameterCodec = runtime.NewParameterCodec(Scheme)
     var Registry = registered.NewOrDie(os.Getenv("KUBE_API_VERSIONS))
     var GroupFactoryRegistry = make(announced.APIGroupFactoryRegistry))

     func init(){
          v1.AddToGroupVersion(Scheme, schema.GroupVersion{Version: "v1"})
          Install(GroupFactoryRegistry, Registry, Scheme)
     }

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/kube-aggregator/pkg/controllers/status**

k8s.io/kubernetes/vendor/k8s.io/kube-aggregator/pkg/controllers/status/available_controller.go

     var cloner = conversion.NewCloner()

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/docker/spdystream/spdy**

k8s.io/kubernetes/vendor/github.com/docker/spdystream/spdy/read.go

     var cframeCtor = map[ControlFrameType]func() controlFrame{
          TypeSynStream: func() controlFrame {return new(SynStreamFrame)},
          ...
     }

k8s.io/kubernetes/vendor/github.com/docker/spdystream/spdy/types.go

     var invalidReqHeaders = map[string]bool{
          "Connection": true,
          "Host": true,
          "Keep-Alive": true
          "Proxy-Connection": true,
          "Transfer-Encoding": true,
     }
     var invalidRespHeaders = map[string]bool{
          "Connection": true,
          "Keep-Alive": true
          "Proxy-Connection": true,
          "Transfer-Encoding": true,
     }

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/docker/spdystream**

k8s.io/kubernetes/vendor/github.com/docker/spdystream/connection.go

     var(
          ErrInvalidStreamId = errors.New("Invalid stream id")
          ErrTimeout = errors.New("Timeout occured")
          ErrReset = errors.New("Stream reset")
          ErrWriteClosedStream = errors.New("Write on closed stream")
     )

k8s.io/kubernetes/vendor/github.com/docker/spdystream/stream.go

     var(
          ErrUnreadPartialData = errors.New("unread partial data")
     )

k8s.io/kubernetes/vendor/github.com/docker/spdystream/utils.go

     var(
          DEBUG = os.Getenv("DEBUG")
     )

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/util/httpstream/spdy**

k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/util/httpstream/spdy/roundtripper.go

     var statusScheme = runtime.NewScheme()
     var statusCodecs = serializer.NewCodecFactory(statusScheme)

     func init(){
          statusScheme.AddUnversionedTypes(metav1.SchemeGroupVersion,
               &metav1.Status{},
          )
     }

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/mxk/go-flowrate/flowrate**

k8s.io/kubernetes/vendor/github.com/mxk/go-flowrate/flowrate/io.go

     var ErrLimit = errors.New("flowrate: flow rate limit exceeded")

k8s.io/kubernetes/vendor/github.com/mxk/go-flowrate/flowrate/util.go

     var czero = time.Duration(time.Now().UnixNano())

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/kube-aggregator/pkg/apiserver**

k8s.io/kubernetes/vendor/k8s.io/kube-aggregator/pkg/apiserver/apiserver.go

     var(
          groupFactoryRegistry = make(announced.APIGroupFactoryRegistry)
          registry = registered.NewOrDir("")
          Scheme = runtime.NewScheme()
          Codecs = serializer.NewCodecFactory(Scheme)
     )

k8s.io/kubernetes/vendor/k8s.io/kube-aggregator/pkg/apiserver/handler_apis.go

     var discoveryGroup = metav1.APIGroup{
          Name: apiregistrationapi.GroupName,
          Versions: []metav1.GroupVersionForDiscovery{
               {
                    GroupVersion: apiregistrationv1beta1api.SchemeGroupVersion.String(),
                    Version: apiregistrationv1beta1api.SchemeGroupVersion.Version,
               },
          },
          PreferredVersion: metav1.GroupVersionForDiscovery{
               GroupVersion: apiregistrationv1beta1api.SchemeGroupVersion.String(),
               Version: apiregistrationv1beta1api.SchemeGroupVersion.Version,
          },
     }

k8s.io/kubernetes/vendor/k8s.io/kube-aggregator/pkg/apiserver/apiserver.go

     func init(){
          install.Install(groupFactoryRegistry, registry, Scheme)
          metav1.AddToGroupVersion(Scheme, schema.GroupVersion{Group: "", Version: "v1"})
          unversioned := schema.GroupVersion{Group: "", Version: "v1"}
          Scheme.AddUnversionedTypes(unversioned,
               &metav1.Status{},
               &metav1.APIVersion{},
               &metav1.APIGroupList{},
               &metav1.APIGroup{},
               &metav1.APIResourceList{},
     }

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/kube-aggregator/pkg/controllers/autoregister**

k8s.io/kubernetes/vendor/k8s.io/kube-aggregator/pkg/controllers/autoregister/autoregister_controller.go

     var(
          cloner = conversion.NewCloner()
     )

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/api**

src/k8s.io/kubernetes/pkg/api/register.go

     var GroupFactoryRegistry = make(announced.APIGroupFactoryRegistry)
     var Registry = registered.NewOrDir(os.Getenv("KUBE_API_VERSIONS"))
     var Scheme = runtime.NewScheme()
     var Codecs = serializer.NewCodecFactory(Scheme)
     var SchemeGroupVersion = schema.GroupVersion{Group: GroupName, Version: runtime.APIVersionInternal}
     var ParameterCodec = runtime.NewParameterCodec(Scheme)

     var(
          SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)
          AddToScheme = SchemeBuilder.AddToScheme
     )

src/k8s.io/kubernetes/pkg/api/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/api/helper**

src/k8s.io/kubernetes/pkg/api/helper/helpers.go

     var podObjectCountQuotaResources = sets.NewString(
          string(api.ResourcePods),
     )
     var podComputeQuotaResources = sets.NewString(
          string(api.ResourceCPU),
          string(api.ResourceMemory),
          string(api.ResourceLimitsCPU),
          string(api.ResourceLimitsMemory),
          string(api.ResourceRequestsCPU),
          string(api.ResourceRequestsMemory),
     )
     var standardContainerResources = sets.NewString(
          string(api.ResourceCPU),
          string(api.ResourceMemory),
     )
     var standardLimitRangeTypes = sets.NewString(
          string(api.LimitTypePod),
          string(api.LimitTypeContainer),
          string(api.LimitTypePersistentVolumeClaim),
     )
     var standardQuotaResources = sets.NewString(
          string(api.ResourceCPU),
          ...
     )
     var standardResources = sets.NewString(
          string(api.ResourceCPU),
          ...
     )
     var integerResources = sets.NewString(
          string(api.ResourcePods),
          ...
     )
     var standardFinalizers = sets.NewString(
          string(api.FinalizerKubernetes),
          metav1.FinalizerOrphanDependents,
          metav1.FinalizerDeleteDependents,
     )

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/extensions**

src/k8s.io/kubernetes/pkg/apis/extensions/register.go

     var(
          SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)
          AddToScheme = SchemeBuilder.AddToScheme
     )

src/k8s.io/kubernetes/pkg/apis/extensions/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/api/v1**

src/k8s.io/kubernetes/pkg/api/v1/generated.pb.go

     var(
          ErrInvalidLengthGenerated = fmt.Errorf("proto: negative length found during unmarshaling")
          ErrIntOverflowGenerated = fmt.Errorf("proto: integer overflow")
     )

src/k8s.io/kubernetes/pkg/api/v1/register.go

     var(
          SchemeBuilder runtime.SchemeBuilder
          localSchemeBuilder = &SchemeBuilder
          AddToScheme = localSchemeBuilder.AddToScheme
     )

src/k8s.io/kubernetes/pkg/api/v1/types.generated.go

     var(
          codecSelferBitsize1234 = uint8(reflect.TypeOf(uint(0)).Bits())
          codecSelferOnlyMapOrArrayEncodeToStructErr1234 = errors.New(`only encoded map or array can be decode into a struct`)
     )

src/k8s.io/kubernetes/pkg/api/v1/types_swagger_doc_generated.go

     var map_AWSElasticBlockStoreVolumeSource = map[string]string{
          "":"",
          ...
     }
     ...

src/k8s.io/kubernetes/pkg/api/v1/generated.pb.go

     func init(){
          proto.RegisterType((*AWSElasticBlockStoreVolumeSource)(nil), "k8s.io.kubernetes.pkg.api.v1.AWSElasticBlockStoreVolumeSource")
          ...
     }
     func init(){
          proto.RegisterFile("k8s.io/kubernetes/pkg/api/v1/generated.proto", fileDescriptorGenerated)
     }

src/k8s.io/kubernetes/pkg/api/v1/register.go

     func init(){
          localSchemeBuilder.Register(addKnownTypes, addDefaultingFuncs, addConversionFuncs, addFastPathConversionFuncs)
     }

src/k8s.io/kubernetes/pkg/api/v1/types_generated.go

     func init(){
          if codec1978.GenVersion != 5{
               _,file,_,_:=runtime.Caller(0)
               err:=fmt.Errorf("codecgen version mismatch: current: %v, need %v. Re-generate file: %v",5,codec1978.GenVersion, file)
               panic(err)
          }
          if false{
               var v0 pkg3_resource.Quantity
               var v1 pkg2_v1.Time
               var v2 pkg5_runtime.RawExtension
               var v3 pkg1_types.UID
               var v4 pkg4_intstr.IntOrString
               var v5 time.Time
               _,_,_,_,_,_=v0,v1,v2,v3,v4,v5
          }
     }

src/k8s.io/kubernetes/pkg/api/v1/zz_generated.conversion.go

     func init(){
          SchemeBuilder.Register(RegisterConversions)
     }

src/k8s.io/kubernetes/pkg/api/v1/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/api/validation**

src/k8s.io/kubernetes/pkg/api/validation/schema.go

     var versionRegexp = regexp.MustCompile(`^(v.+|unversioned|types)\..*`)

src/k8s.io/kubernetes/pkg/api/validation/validation.go

     var RepairMalformedUpdates bool = genericvalidation.RepairMalformedUpdates
     var pdPartitionErrorMsg string = validation.InclusiveRangeError(1,255)
     var volumeModeErrorMsg string = "must be a number between 0 and 0777 (octal), both inclusive"
     var BannedOwners = genericvalidation.BannedOwners

     var ValidateNamespaceName = apimachineryvalidation.ValidateNamespaceName

     var ValidateServiceAccountName = apimachineryvalidation.ValidateServiceAccountName

     var ValidateClusterName = genericvalidation.ValidateClusterName

     var validDownwardAPIFieldPathExpressions = sets.NewString(
          "metadata.name",
          "metadata.namespace",
          "metadata.labels",
          "metadata.annotations")

     var supportedAccessModes = sets.NewString(string(api.ReadWriteOnce), string(api.ReadOnlyMany), string(api.ReadWriteMany))
     var supportedReclaimPolicy = sets.NewString(string(api.PersistentVolumeReclaimDelete), string(api.PersistentVolumeReclaimRecycle), string(api.PersistentVolumeReclaimRetain))

     var supportedPortProtocols = sets.NewString(string(api.ProtocolTCP), string(api. ProtocolUDP))

     var validFieldPathExpressionsEnv = sets.NewString("metadata.name", "metadata.namespace", "spec.nodeName", "spec.serviceAccountName", "status.hostIP", "status.podIP")
     var validContainerResourceFieldPathExpressions = sets.NewString("limits.cpu", "limits.memory", "requests.cpu", "requests.memory")

     var validContainerResourceDivisorForCPU = sets.NewString("1m", "1")
     var validContainerResourceDivisorForMemory = sets.NewString("1", "1k", "1M", "1G", "1T", "1P", "1E", "1Ki", "1Mi", "1Gi", "1Ti", "1Pi", "1Ei")

     var supportedHTTPSchemes = sets.NewString(string(api.URISchemeHTTP), string(api.URISchemeHTTPS))

     var supportedPullPolicies = sets.NewString(string(api.PullAlways), string(api.PullIfNotPresent), string(api.PullNever))

     var sysctlRegexp = regexp.MustCompile("^"+SysctlFmt+"$")

     var supportedSessionAffinityType = sets.NewString(string(api.ServiceAffinityClientIP), string(api.ServiceAffinityNone))
     var supportedServiceType = sets.NewString(string(api.ServiceTypeClusterIP), string(api.ServiceTypeNodePort),
           string(api.ServiceTypeLoadBalancer), string(api.ServiceTypeExternalName))

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/gophercloud/gophercloud/pagination**

k8s.io/kubernetes/vendor/github.com/gophercloud/gophercloud/pagination/pager.go

     var(
          ErrPageNotAvailable = errors.New("The requested page does not exist.")
     )

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/gophercloud/gophercloud/openstack/utils**

k8s.io/kubernetes/vendor/github.com/gophercloud/gophercloud/openstack/utils/choose_version.go

     var goodStatus = map[string]bool{
          "current": true,
          "supported": true,
          "stable": true,
     }

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apiserver/plugin/pkg/authenticator/request/basicauth**

k8s.io/kubernetes/vendor/k8s.io/apiserver/plugin/pkg/authenticator/request/basicauth/basicauth.go

     var errInvalidAuth = errors.New("invalid username/password combination")

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/coreos/go-oidc/key**

k8s.io/kubernetes/vendor/github.com/coreos/go-oidc/key/repo.go

     var ErrorNoKeys = errors.New("no keys found")

k8s.io/kubernetes/vendor/github.com/coreos/go-oidc/key/rotate.go

     var(
          ErrorPrivateKeysExpired = errors.New("private keys have expired")
     )

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/coreos/go-oidc/oidc**

k8s.io/kubernetes/vendor/github.com/coreos/go-oidc/oidc/client.go

     var(
          DefaultScope = []string{"openid", "email", "profile"{
          supportedAuthMethods = map[string]struct{}{
               oauth2.AuthMethodClientSecretBasic: struct{}{},
               oauth2.AuthMethodClientSecretPost: struct{}{},
          }
     )

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/dgrijalva/jwt-go**

k8s.io/kubernetes/vendor/github.com/dgrijalva/jwt-go/ecdsa.go

     var(
          ErrECDSAVerification = errors.New("crypto/ecdsa: verification error")
     )

k8s.io/kubernetes/vendor/github.com/dgrijalva/jwt-go/ecdsa_utils.go

     var(
          ErrNotECPublicKey = errors.New("Key is not a valid ECDSA public key")
          ErrNotECPrivateKey = errors.New("Key is not a valid ECDSA private key")
     )

k8s.io/kubernetes/vendor/github.com/dgrijalva/jwt-go/errors.go

     var(
          ErrInvalidKey = errors.New("key is invalid")
          ErrInvalidKeyType = errors.New("key is of invalid type")
          ErrHashUnavailable = errors.New("the requested hash function is unavailable")
     )

k8s.io/kubernetes/vendor/github.com/dgrijalva/jwt-go/hmac.go

     var(
          SigningMethodHS256 *SigningMethodHMAC
          SigningMethodHS384 *SigningMethodHMAC
          SigningMethodHS512 *SigningMethodHMAC
          ErrSignatureInvalid = errors.New("signature is invalid")
     )

k8s.io/kubernetes/vendor/github.com/dgrijalva/jwt-go/rsa-utils.go

     var(
          ErrKeyMustBePEMEncoded = errors.New("Invalid Key: Key must be PEM encoded PKCS1 or PKCS8 private key")
          ErrNotRSAPrivateKey = errors.New("Key is not a valid RSA private key")
          ErrNotRSAPublicKey = errors.New("Key is not a valid RSA public key")
     )

k8s.io/kubernetes/vendor/github.com/dgrijalva/jwt-go/signing_method.go

     var signingMethods = map[string]func() SigningMethod{}
     var signingMethodLock = new(sync.RWMutex)

k8s.io/kubernetes/vendor/github.com/dgrijalva/jwt-go/ecdsa.go

     func init(){
          SigningMethodES256 = &SigningMethodECDSA{"ES256", crypto.SHA256, 32, 256}
          RegisterSigningMethod(SigningMethodES256.Alg(), func() SigningMethod{
               return SigningMethodES256
          })
          SigningMethodES384 = &SigningMethodECDSA{"ES384", crypto.SHA384, 48, 384}
          RegisterSigningMethod(SigningMethodES256.Alg(), func() SigningMethod{
               return SigningMethodES384
          })
          SigningMethodES512 = &SigningMethodECDSA{"ES512", crypto.SHA512, 66, 521}    //521 is a bug?
          RegisterSigningMethod(SigningMethodES256.Alg(), func() SigningMethod{
               return SigningMethodES512
          })
     }

k8s.io/kubernetes/vendor/github.com/dgrijalva/jwt-go/hmac.go

     func init(){
          SigningMethodHS256 = &SigningMethodHMAC{"HS256", crypto.SHA256}
          RegisterSigningMethod(SigningMethodHS256.Alg(), func() SigningMethod{
               return SigningMethodHS256
          })
          SigningMethodHS384 = &SigningMethodHMAC{"HS384", crypto.SHA384}
          RegisterSigningMethod(SigningMethodHS384.Alg(), func() SigningMethod{
               return SigningMethodHS384
          })
          SigningMethodHS512 = &SigningMethodHMAC{"HS512", crypto.SHA512}
          RegisterSigningMethod(SigningMethodHS512.Alg(), func() SigningMethod{
               return SigningMethodHS512
          })
     }

k8s.io/kubernetes/vendor/github.com/dgrijalva/jwt-go/none.go

     func init(){
          SigningMethodNone = &signingMethodNone{}
          NoneSignatureTypeDisallowedError = NewValidationError("'none' signature type is not allowed", ValidationErrorSignatureInvalid)
          RegisterSigningMethod(SigningMethodNone.Alg(), func() SigningMethod{
               return SigningMethodNone
          })
     }

k8s.io/kubernetes/vendor/github.com/dgrijalva/jwt-go/rsa.go

     func init(){
          SigningMethodRS256 = &SigningMethodRSA{"RSA256", crypto.SHA256}
          RegisterSigningMethod(SigningMethodRS256.Alg(), func() SigningMethod{
               return SigningMethodRS256
          })
          SigningMethodRS384 = &SigningMethodRSA{"RSA384", crypto.SHA384}
          RegisterSigningMethod(SigningMethodRS384.Alg(), func() SigningMethod{
               return SigningMethodRS384
          })
          SigningMethodRS512 = &SigningMethodRSA{"RSA512", crypto.SHA512}
          RegisterSigningMethod(SigningMethodRS512.Alg(), func() SigningMethod{
               return SigningMethodRS512
          })
     }

k8s.io/kubernetes/vendor/github.com/dgrijalva/jwt-go/rsa_pss.go

     func init(){
          SigningMethodPS256 = &SigningMethodRSAPSS{
               &SigningMethodRSA{
                    Name: "PS256",
                    Hash: crypto.SHA256,
               },
               &rsa.PSSOptions{
                    SaltLength: ras.PSSSaltLengthAuto,
                    Hash: crypto.SHA256,
               },
          }
          RegisterSigningMethod(SigningMethodPS256.Alg(), func() SigningMethod{
               return SigningMethodPS256
          })
          SigningMethodPS384 = &SigningMethodRSAPSS{
               &SigningMethodRSA{
                    Name: "PS384",
                    Hash: crypto.SHA384,
               },
               &rsa.PSSOptions{
                    SaltLength: ras.PSSSaltLengthAuto,
                    Hash: crypto.SHA384,
               },
          }
          RegisterSigningMethod(SigningMethodPS384.Alg(), func() SigningMethod{
               return SigningMethodPS384
          })
          SigningMethodPS512 = &SigningMethodRSAPSS{
               &SigningMethodRSA{
                    Name: "PS512",
                    Hash: crypto.SHA512,
               },
               &rsa.PSSOptions{
                    SaltLength: ras.PSSSaltLengthAuto,
                    Hash: crypto.SHA512,
               },
          }
          RegisterSigningMethod(SigningMethodPS512.Alg(), func() SigningMethod{
               return SigningMethodPS512
          })
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/serviceaccount**

src/k8s.io/kubernetes/pkg/serviceaccount/jwt.go

     var errMismatchedSigningMethod = errors.New("invalid signing method")

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/golang.org/x/oauth2**

k8s.io/kubernetes/vendor/golang.org/x/oauth2/oauth2.go

     var NoContext = context.TODO()
     var(
          AccessTypeOnline  AuthCodeOption = SetAuthURLParam("access_type", "online")
          AccessTypeOffline AuthCodeOption = SetAuthURLParam("access_type", "offline")
          ApprovalForce AuthCodeOption = SetAuthURLParam("approval_promt", "force")
     )

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/api/install**

src/k8s.io/kubernetes/pkg/api/install/install.go

     func init(){
          Install(api.GroupFactoryRegistry, api.Registry, api.Scheme)
     }

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/golang.org/x/oauth2/google**

k8s.io/kubernetes/vendor/golang.org/x/oauth2/google/appengine.go

     var(
          aeTokensMu sync.Mutex
          aeTokens = make(map[string]*tokenLock)
     )

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/admissionregistration**

src/k8s.io/kubernetes/pkg/apis/admissionregistration/register.go

     var(
          SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)
          AddToScheme = SchemeBuilder.AddToScheme
     )

src/k8s.io/kubernetes/pkg/apis/admissionregistration/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/client-go/third_party/forked/golang/template**

k8s.io/kubernetes/vendor/k8s.io/client-go/third_party/forked/golang/template/exec.go

     var(
          errorType = reflect.TypeOf((*error)(nil)).Elem()
          fmtStringerType = reflect.TypeOf((*fmt.Stringer)(nil)).Elem()
     )

k8s.io/kubernetes/vendor/k8s.io/client-go/third_party/forked/golang/template/funcs.go

     var(
          errBadComparisonType = errors.New("invalid type for comparison")
          errBadComparison = errors.New("incompatible types for comparison")
          errNoComparison = errors.New("missing argument for comparison")
     )
     var builtins = FuncMap{
          "and": and,
          ...
     }
     var builtinFuncs = createValueFuncs(builtins)

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/admissionregistration/v1alpha1**

src/k8s.io/kubernetes/pkg/apis/admissionregistration/v1alpha1/generated.pb.go

     var(
          ErrInvalidLengthGenerated = fmt.Errorf("proto: negative length found during unmarshaling")
          ErrIntOverflowGenerated = fmt.Errorf("proto: integer overflow")
     )

src/k8s.io/kubernetes/pkg/apis/admissionregistration/v1alpha1/register.go

     var(
          SchemeBuilder runtime.SchemeBuilder
          localSchemeBuilder = &SchemeBuilder
          AddToScheme = localSchemeBuilder.AddToScheme
     )

src/k8s.io/kubernetes/pkg/apis/admissionregistration/v1alpha1/types.generated.go

     var(
          codecSelferBitsize1234 = uint8(reflect.TypeOf(uint(0)).Bits())
          codecSelferOnlyMapOrArrayEncodeToStructErr1234 = errors.New(`only encoded map or array can be decoded into a struct`)
     )

src/k8s.io/kubernetes/pkg/apis/admissionregistration/v1alpha1/types_swagger_doc_generated.go

     var map_AdmissionHookClientConfig = map[string]string{
          "":"",
          ...
     }
     ...

src/k8s.io/kubernetes/pkg/apis/admissionregistration/v1alpha1/generated.pb.go

     func init(){
          proto.RegisterType((*AdmissionHookClientConfig)(nil), "k8s.io.kubernetes.pkg.apis.admissionregistration.v1alpha1.AdmissionHookClientConfig")
          ...
     }
     func init(){
          proto.RegisterFile("k8s.io/kubernetes/pkg/apis/admissionregistration/v1alpha1/generated.proto", fileDescriptorGenerated)
     }

src/k8s.io/kubernetes/pkg/apis/admissionregistration/v1alpha1/register.go

     func init(){
          localSchemeBuilder.Register(addKnownTypes, addDefaultingFuncs)
     }

src/k8s.io/kubernetes/pkg/apis/admissionregistration/v1alpha1/types.generated.go

     func init(){
          if codec1978.GenVersion!=5{
               _,file,_,_:=runtime.Caller(0)
               err:=fmt.Errorf("codecgen version mismatch: current: %v, need %v. Re-generate file: %v",5,codec1978.GenVersion,file)
               panic(err)
          }
          if false{
               var v0 pkg1_v1.TypeMeta
               var v1 pkg2_types.UID
               var v2 time.Time
               _,_,_=v0,v1,v2
          }
     }

src/k8s.io/kubernetes/pkg/apis/admissionregistration/v1alpha1/zz_generated.conversion.go

     func init(){
          SchemeBuilder.Register(RegisterConversions)
     }

src/k8s.io/kubernetes/pkg/apis/admissionregistration/v1alpha1/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/client-go/util/jsonpath**

k8s.io/kubernetes/vendor/k8s.io/client-go/util/jsonpath/node.go

     var NodeTypeName = map[NodeType]string{
          NodeText: "NodeText",
          ...
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/admissionregistration/install**

src/k8s.io/kubernetes/pkg/apis/admissionregistration/install/install.go

     func init(){
          Install(api.GroupFactoryRegistry, api.Registry, api.Scheme)
     }

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/client-go/plugin/pkg/client/auth/gcp**

k8s.io/kubernetes/vendor/k8s.io/client-go/plugin/pkg/client/auth/gcp/gcp.go

     func init(){
          if err:=restclient.RegisterAuthProviderPlugin("gcp", newGCPAuthProvider);err!=nil{
               glog.Fatalf("Failed to register gcp auth plugin: %v", err)
          }
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/apps**

src/k8s.io/kubernetes/pkg/apis/apps/register.go

     var(
          SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)
          AddToScheme = SchemeBuilder.AddToScheme
     )

src/k8s.io/kubernetes/pkg/apis/apps/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/client-go/plugin/pkg/client/auth/oidc**

k8s.io/kubernetes/vendor/k8s.io/client-go/plugin/pkg/client/auth/oidc/oidc.go

     func init(){
          if err:=restclient.RegisterAuthProviderPlugin("oidc", newOIDCAuthProvider): err!=nil{
               glog.Fatalf("Failed to register oidc auth plugin: %v", err)
          }
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/apps/v1beta1**

src/k8s.io/kubernetes/pkg/apis/apps/v1beta1/generated.pb.go

     var(
          ErrInvalidLengthGenerated = fmt.Errorf("proto: negative length found during unmarshaling")
          ErrIntOverflowGenerated = fmt.Errorf("proto: integer overflow")
     )

src/k8s.io/kubernetes/pkg/apis/apps/v1beta1/register.go

     var(
          SchemeBuilder runtime.SchemeBuilder
          localSchemeBuilder = &SchemeBuilder
          AddToScheme = localSchemeBuilder.AddToScheme
     )

src/k8s.io/kubernetes/pkg/apis/apps/v1beta1/types.generated.go

     var(
          codecSelferBitsize1234 = uint8(reflect.TypeOf(uint(0)).Bits())
          codecSelferOnlyMapOrArrayEncodeToStructErr1234 = errors.New(`only encoded map or array can be decoded into a struct`)
     )

src/k8s.io/kubernetes/pkg/apis/apps/v1beta1/types_swagger_doc_generated.go

     var map_ControllerRevision = map[string]string{
          "":"",
          ...
     }
     ...

src/k8s.io/kubernetes/pkg/apis/apps/v1beta1/generated.pb.go

     func init(){
          proto.RegisterType((*ControllerRevision)(nil), "k8s.io.kubernetes.pkg.apis.apps.v1beta1.ControllerRevision")
          ...
     }

     func init(){
          proto.RegisterFile("k8s.io/kubernetes/pkg/apis/apps/v1beta1/generated.proto", fileDescriptorGenerated)
     }

src/k8s.io/kubernetes/pkg/apis/apps/v1beta1/register.go

     func init(){
          localSchemeBuilder.Register(addKnownTypes, addDefaultingFuncs, addConversionFuncs)
     }

src/k8s.io/kubernetes/pkg/apis/apps/v1beta1/types.generated.go

     func init(){
          if codec1978.GenVersion != 5{
               _,file,_,_:=runtime.Caller(0)
               err:=fmt.Errorf("codecgen version mismatch: current: %v, need %v. Re-generate file: %v",5,codec1978.GenVersion,file)
               panic(err)
          }
          if false{
               var v0 pkg4_resource.Quantity
               var v1 pkg1_v1.TypeMeta
               var v2 pkg6_runtime.RawExtension
               var v3 pkg2_types.UID
               var v4 pkg5_intstr.IntOrString
               var v5 pkg3_v1.PodTemplateSpec
               var v6 time.Time
               _,_,_,_,_,_,_=v0,v1,v2,v3,v4,v5,v6
          }
     }

src/k8s.io/kubernetes/pkg/apis/apps/v1beta1/zz_generated.conversion.go

     func init(){
          SchemeBuilder.Reigster(RegisterConversions)
     }

src/k8s.io/kubernetes/pkg/apis/apps/v1beta1/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/apps/install**

src/k8s.io/kubernetes/pkg/apis/apps/install/install.go

     func init(){
          Install(api.GroupFactoryRegistry, api.Registry, api.Scheme)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/authentication**

src/k8s.io/kubernetes/pkg/apis/authentication/register.go

     var(
          SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)
          AddToScheme = SchemeBuilder.AddToScheme
     )

src/k8s.io/kubernetes/pkg/apis/authentication/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/authentication/v1**

src/k8s.io/kubernetes/pkg/apis/authentication/v1/generated.pb.go

     var(
          ErrInvalidLengthGenerated = fmt.Errorf("proto: negative length found during unamrshaling")
          ErrIntOverflowGenerated = fmt.Errorf("proto: integer overflow")
     )

src/k8s.io/kubernetes/pkg/apis/authentication/v1/register.go

     var(
          SchemeBuilder runtime.SchemeBuilder
          localSchemeBuilder = &SchemeBuilder
          AddToScheme = localSchemeBuilder.AddToScheme
     )

src/k8s.io/kubernetes/pkg/apis/authentication/v1/types_swagger_doc_generated.go

     var map_TokenReview = map[string]string{
          "":"",
          ...
     }
     ...

src/k8s.io/kubernetes/pkg/apis/authentication/v1/generated.pb.go

     func init(){
          proto.RegisterType((*ExtraValue)(nil), "k8s.io.kubernetes.pkg.apis.authentication.v1.ExtraValue")
          ...
     }
     func init(){
          proto.RegisterFile("k8s.io/kubernetes/pkg/apis/authentication/v1/generated.proto", fileDescriptorGenerated)
     }

src/k8s.io/kubernetes/pkg/apis/authentication/v1/register.go

     func init(){
          localSchemeBuilder.Register(addKnownTypes, addDefaultingFuncs, addConversionFuncs)
     }

src/k8s.io/kubernetes/pkg/apis/authentication/v1/zz_generated.conversion.go

     func init(){
          SchemeBuilder.Register(RegisterConversions)
     }
src/k8s.io/kubernetes/pkg/apis/authentication/v1/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/authentication/v1beta1**

src/k8s.io/kubernetes/pkg/apis/authentication/v1beta1/generated.pb.go

     var(
          ErrInvalidLengthGenerated = fmt.Errorf("proto: negative length found during unamrshaling")
          ErrIntOverflowGenerated = fmt.Errorf("proto: integer overflow")
     )

src/k8s.io/kubernetes/pkg/apis/authentication/v1beta1/register.go

     var(
          SchemeBuilder runtime.SchemeBuilder
          localSchemeBuilder = &SchemeBuilder
          AddToScheme = localSchemeBuilder.AddToScheme
     )

src/k8s.io/kubernetes/pkg/apis/authentication/v1beta1/types.generated.go

     var(
          codecSelferBitsize1234 = uint8(reflect.TypeOf(uint(0)).Bits())
          codecSelferOnlyMapOrArrayEncodeToStructErr1234 = errors.New(`only encoded map or array can be decoded into a struct`)
     )

src/k8s.io/kubernetes/pkg/apis/authentication/v1beta1/types_swagger_doc_generated.go

     var map_TokenReview = map[string]string{
          "":"",
          ...
     }
     ...

src/k8s.io/kubernetes/pkg/apis/authentication/v1beta1/generated.pb.go

     func init(){
          proto.RegisterType((*ExtraValue)(nil), "k8s.io.kubernetes.pkg.apis.authentication.v1beta1.ExtraValue")
          ...
     }
     func init(){
          proto.RegisterFile("k8s.io/kubernetes/pkg/apis/authentication/v1beta1/generated.proto", fileDescriptorGenerated)
     }

src/k8s.io/kubernetes/pkg/apis/authentication/v1beta1/register.go

     func init(){
          localSchemeBuilder.Register(addKnownTypes, addDefaultingFuncs, addConversionFuncs)
     }

src/k8s.io/kubernetes/pkg/apis/authentication/v1beta1/types.generated.go

     func init(){
          if codec1978.GenVersion != 5{
               _,file,_,_:=runtime.Caller(0)
               err:=fmt.Errorf("codecgen version mismatch: current: %v, need %v. Re-generate file: %v",5,codec1978.GenVersion,file)
               panic(err)
          }
          if false{
               var v0 pkg1_v1.TypeMeta
               var v1 pkg2_types.UID
               var v2 time.Time
               _,_,_ = v0,v1,v2
          }
     }

src/k8s.io/kubernetes/pkg/apis/authentication/v1beta1/zz_generated.conversion.go

     func init(){
          SchemeBuilder.Register(RegisterConversions)
     }
src/k8s.io/kubernetes/pkg/apis/authentication/v1beta1/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/authentication/install**

src/k8s.io/kubernetes/pkg/apis/authentication/install/install.go

     func init(){
          Install(api.GroupFactoryRegistry, api.Registry, api.Scheme)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/authorization**

src/k8s.io/kubernetes/pkg/apis/authorization/register.go

     var(
          SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)
          AddToScheme = SchemeBuilder.AddToScheme
     )

src/k8s.io/kubernetes/pkg/apis/authorization/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/authorization/v1**

src/k8s.io/kubernetes/pkg/apis/authorization/v1/generated.pb.go

     var(
          ErrInvalidLengthGenerated = fmt.Errorf("proto: negative length found during unamrshaling")
          ErrIntOverflowGenerated = fmt.Errorf("proto: integer overflow")
     )

src/k8s.io/kubernetes/pkg/apis/ authorization/v1/register.go

     var(
          SchemeBuilder runtime.SchemeBuilder
          localSchemeBuilder = &SchemeBuilder
          AddToScheme = localSchemeBuilder.AddToScheme
     )

src/k8s.io/kubernetes/pkg/apis/ authorization/v1/types.generated.go

     var(
          codecSelferBitsize1234 = uint8(reflect.TypeOf(uint(0)).Bits())
          codecSelferOnlyMapOrArrayEncodeToStructErr1234 = errors.New(`only encoded map or array can be decoded into a struct`)
     )

src/k8s.io/kubernetes/pkg/apis/ authorization/v1/types_swagger_doc_generated.go

     var map_LocalSubjectAccessReview = map[string]string{
          "":"",
          ...
     }
     ...

src/k8s.io/kubernetes/pkg/apis/ authorization/v1/generated.pb.go

     func init(){
          proto.RegisterType((*ExtraValue)(nil), "k8s.io.kubernetes.pkg.apis. authorization.v1.ExtraValue")
          ...
     }
     func init(){
          proto.RegisterFile("k8s.io/kubernetes/pkg/apis/ authorization/v1/generated.proto", fileDescriptorGenerated)
     }

src/k8s.io/kubernetes/pkg/apis/ authorization/v1/register.go

     func init(){
          localSchemeBuilder.Register(addKnownTypes, addDefaultingFuncs, addConversionFuncs)
     }

src/k8s.io/kubernetes/pkg/apis/ authorization/v1/types.generated.go

     func init(){
          if codec1978.GenVersion != 5{
               _,file,_,_:=runtime.Caller(0)
               err:=fmt.Errorf("codecgen version mismatch: current: %v, need %v. Re-generate file: %v",5,codec1978.GenVersion,file)
               panic(err)
          }
          if false{
               var v0 pkg1_v1.TypeMeta
               var v1 pkg2_types.UID
               var v2 time.Time
               _,_,_ = v0,v1,v2
          }
     }

src/k8s.io/kubernetes/pkg/apis/ authorization/v1/zz_generated.conversion.go

     func init(){
          SchemeBuilder.Register(RegisterConversions)
     }
src/k8s.io/kubernetes/pkg/apis/ authorization/v1/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/authorization/v1beta1**

src/k8s.io/kubernetes/pkg/apis/authorization/v1beta1/generated.pb.go

     var(
          ErrInvalidLengthGenerated = fmt.Errorf("proto: negative length found during unamrshaling")
          ErrIntOverflowGenerated = fmt.Errorf("proto: integer overflow")
     )

src/k8s.io/kubernetes/pkg/apis/ authorization/v1beta1/register.go

     var(
          SchemeBuilder runtime.SchemeBuilder
          localSchemeBuilder = &SchemeBuilder
          AddToScheme = localSchemeBuilder.AddToScheme
     )

src/k8s.io/kubernetes/pkg/apis/ authorization/v1beta1/types.generated.go

     var(
          codecSelferBitsize1234 = uint8(reflect.TypeOf(uint(0)).Bits())
          codecSelferOnlyMapOrArrayEncodeToStructErr1234 = errors.New(`only encoded map or array can be decoded into a struct`)
     )

src/k8s.io/kubernetes/pkg/apis/ authorization/v1beta1/types_swagger_doc_generated.go

     var map_LocalSubjectAccessReview = map[string]string{
          "":"",
          ...
     }
     ...

src/k8s.io/kubernetes/pkg/apis/ authorization/v1beta1/generated.pb.go

     func init(){
          proto.RegisterType((*ExtraValue)(nil), "k8s.io.kubernetes.pkg.apis. authorization.v1beta1.ExtraValue")
          ...
     }
     func init(){
          proto.RegisterFile("k8s.io/kubernetes/pkg/apis/ authorization/v1beta1/generated.proto", fileDescriptorGenerated)
     }

src/k8s.io/kubernetes/pkg/apis/ authorization/v1beta1/register.go

     func init(){
          localSchemeBuilder.Register(addKnownTypes, addDefaultingFuncs, addConversionFuncs)
     }

src/k8s.io/kubernetes/pkg/apis/ authorization/v1beta1/types.generated.go

     func init(){
          if codec1978.GenVersion != 5{
               _,file,_,_:=runtime.Caller(0)
               err:=fmt.Errorf("codecgen version mismatch: current: %v, need %v. Re-generate file: %v",5,codec1978.GenVersion,file)
               panic(err)
          }
          if false{
               var v0 pkg1_v1.TypeMeta
               var v1 pkg2_types.UID
               var v2 time.Time
               _,_,_ = v0,v1,v2
          }
     }

src/k8s.io/kubernetes/pkg/apis/ authorization/v1beta1/zz_generated.conversion.go

     func init(){
          SchemeBuilder.Register(RegisterConversions)
     }
src/k8s.io/kubernetes/pkg/apis/ authorization/v1beta1/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/authorization/install**

src/k8s.io/kubernetes/pkg/apis/authorization/install/install.go

     func init(){
          Install(api.GroupFactoryRegistry, api.Registry, api.Scheme)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/autoscaling**

src/k8s.io/kubernetes/pkg/apis/autoscaling/register.go

     var(
          SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)
          AddToScheme = SchemeBuilder.AddToScheme
     )

src/k8s.io/kubernetes/pkg/apis/autoscaling/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/autoscaling/v1**

src/k8s.io/kubernetes/pkg/apis/autoscaling/v1/generated.pb.go

     var(
          ErrInvalidLengthGenerated = fmt.Errorf("proto: negative length found during unamrshaling")
          ErrIntOverflowGenerated = fmt.Errorf("proto: integer overflow")
     )

src/k8s.io/kubernetes/pkg/apis/ autoscaling/v1/register.go

     var(
          SchemeBuilder runtime.SchemeBuilder
          localSchemeBuilder = &SchemeBuilder
          AddToScheme = localSchemeBuilder.AddToScheme
     )

src/k8s.io/kubernetes/pkg/apis/ autoscaling/v1/types.generated.go

     var(
          codecSelferBitsize1234 = uint8(reflect.TypeOf(uint(0)).Bits())
          codecSelferOnlyMapOrArrayEncodeToStructErr1234 = errors.New(`only encoded map or array can be decoded into a struct`)
     )

src/k8s.io/kubernetes/pkg/apis/ autoscaling/v1/types_swagger_doc_generated.go

     var map_CrossVersionObjectReference= map[string]string{
          "":"",
          ...
     }
     ...

src/k8s.io/kubernetes/pkg/apis/ autoscaling/v1/generated.pb.go

     func init(){
          proto.RegisterType((*CrossVersionObjectReference)(nil), "k8s.io.kubernetes.pkg.apis. autoscaling.v1. CrossVersionObjectReference")
          ...
     }
     func init(){
          proto.RegisterFile("k8s.io/kubernetes/pkg/apis/ autoscaling/v1/generated.proto", fileDescriptorGenerated)
     }

src/k8s.io/kubernetes/pkg/apis/ autoscaling/v1/register.go

     func init(){
          localSchemeBuilder.Register(addKnownTypes, addDefaultingFuncs, addConversionFuncs)
     }

src/k8s.io/kubernetes/pkg/apis/ autoscaling/v1/types.generated.go

     func init(){
          if codec1978.GenVersion != 5{
               _,file,_,_:=runtime.Caller(0)
               err:=fmt.Errorf("codecgen version mismatch: current: %v, need %v. Re-generate file: %v",5,codec1978.GenVersion,file)
               panic(err)
          }
          if false{
               var v0 pkg3_resource.Quantity
               var v1 pkg1_v1.Time
               var v2 pkg2_types.UID
               var v3 pkg4_v1.ResourceName
               var v4 time.Time
               _,_,_,_,_ = v0,v1,v2,v3,v4
          }
     }

src/k8s.io/kubernetes/pkg/apis/ autoscaling/v1/zz_generated.conversion.go

     func init(){
          SchemeBuilder.Register(RegisterConversions)
     }
src/k8s.io/kubernetes/pkg/apis/ autoscaling/v1/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/autoscaling/v2alpha1**

src/k8s.io/kubernetes/pkg/apis/autoscaling/v2alpha1/generated.pb.go

     var(
          ErrInvalidLengthGenerated = fmt.Errorf("proto: negative length found during unamrshaling")
          ErrIntOverflowGenerated = fmt.Errorf("proto: integer overflow")
     )

src/k8s.io/kubernetes/pkg/apis/ autoscaling/ v2alpha1/register.go

     var(
          SchemeBuilder runtime.SchemeBuilder
          localSchemeBuilder = &SchemeBuilder
          AddToScheme = localSchemeBuilder.AddToScheme
     )

src/k8s.io/kubernetes/pkg/apis/ autoscaling/ v2alpha1/types.generated.go

     var(
          codecSelferBitsize1234 = uint8(reflect.TypeOf(uint(0)).Bits())
          codecSelferOnlyMapOrArrayEncodeToStructErr1234 = errors.New(`only encoded map or array can be decoded into a struct`)
     )

src/k8s.io/kubernetes/pkg/apis/ autoscaling/ v2alpha1/types_swagger_doc_generated.go

     var map_CrossVersionObjectReference= map[string]string{
          "":"",
          ...
     }
     ...

src/k8s.io/kubernetes/pkg/apis/ autoscaling/ v2alpha1/generated.pb.go

     func init(){
          proto.RegisterType((*CrossVersionObjectReference)(nil), "k8s.io.kubernetes.pkg.apis. autoscaling. v2alpha1. CrossVersionObjectReference")
          ...
     }
     func init(){
          proto.RegisterFile("k8s.io/kubernetes/pkg/apis/ autoscaling/ v2alpha1/generated.proto", fileDescriptorGenerated)
     }

src/k8s.io/kubernetes/pkg/apis/ autoscaling/ v2alpha1/register.go

     func init(){
          localSchemeBuilder.Register(addKnownTypes, addDefaultingFuncs, addConversionFuncs)
     }

src/k8s.io/kubernetes/pkg/apis/ autoscaling/ v2alpha1/types.generated.go

     func init(){
          if codec1978.GenVersion != 5{
               _,file,_,_:=runtime.Caller(0)
               err:=fmt.Errorf("codecgen version mismatch: current: %v, need %v. Re-generate file: %v",5,codec1978.GenVersion,file)
               panic(err)
          }
          if false{
               var v0 pkg1_resource.Quantity
               var v1 pkg3_v1.Time
               var v2 pkg4_types.UID
               var v3 pkg2_v1.ResourceName
               var v4 time.Time
               _,_,_,_,_ = v0,v1,v2,v3,v4
          }
     }

src/k8s.io/kubernetes/pkg/apis/ autoscaling/ v2alpha1/zz_generated.conversion.go

     func init(){
          SchemeBuilder.Register(RegisterConversions)
     }
src/k8s.io/kubernetes/pkg/apis/ autoscaling/ v2alpha1/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/autoscaling/install**

src/k8s.io/kubernetes/pkg/apis/autoscaling/install/install.go

     func init(){
          Install(api.GroupFactoryRegistry, api.Registry, api.Scheme)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/batch**

src/k8s.io/kubernetes/pkg/apis/batch/register.go

     var(
          SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)
          AddToScheme = SchemeBuilder.AddToScheme
     )

src/k8s.io/kubernetes/pkg/apis/ batch/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/batch/v1**

src/k8s.io/kubernetes/pkg/apis/batch/v1/generated.pb.go

     var(
          ErrInvalidLengthGenerated = fmt.Errorf("proto: negative length found during unamrshaling")
          ErrIntOverflowGenerated = fmt.Errorf("proto: integer overflow")
     )

src/k8s.io/kubernetes/pkg/apis/ batch/v1/register.go

     var(
          SchemeBuilder runtime.SchemeBuilder
          localSchemeBuilder = &SchemeBuilder
          AddToScheme = localSchemeBuilder.AddToScheme
     )

src/k8s.io/kubernetes/pkg/apis/ batch/v1/types.generated.go

     var(
          codecSelferBitsize1234 = uint8(reflect.TypeOf(uint(0)).Bits())
          codecSelferOnlyMapOrArrayEncodeToStructErr1234 = errors.New(`only encoded map or array can be decoded into a struct`)
     )

src/k8s.io/kubernetes/pkg/apis/ batch/v1/types_swagger_doc_generated.go

     var map_Job= map[string]string{
          "":"",
          ...
     }
     ...

src/k8s.io/kubernetes/pkg/apis/ batch/v1/generated.pb.go

     func init(){
          proto.RegisterType((*Job)(nil), "k8s.io.kubernetes.pkg.apis. batch.v1. Job")
          ...
     }
     func init(){
          proto.RegisterFile("k8s.io/kubernetes/pkg/apis/ batch/v1/generated.proto", fileDescriptorGenerated)
     }

src/k8s.io/kubernetes/pkg/apis/ batch/v1/register.go

     func init(){
          localSchemeBuilder.Register(addKnownTypes, addDefaultingFuncs, addConversionFuncs)
     }

src/k8s.io/kubernetes/pkg/apis/ batch/v1/types.generated.go

     func init(){
          if codec1978.GenVersion != 5{
               _,file,_,_:=runtime.Caller(0)
               err:=fmt.Errorf("codecgen version mismatch: current: %v, need %v. Re-generate file: %v",5,codec1978.GenVersion,file)
               panic(err)
          }
          if false{
               var v0 pkg4_resource.Quantity
               var v1 pkg1_v1.TypeMeta
               var v2 pkg2_types.UID
               var v3 pkg5_intstr.IntOrString
               var v4 pkg3_v1.PodTemplateSpec
               var v5 time.Time
               _,_,_,_,_,_ = v0,v1,v2,v3,v4,v5
          }
     }

src/k8s.io/kubernetes/pkg/apis/ batch/v1/zz_generated.conversion.go

     func init(){
          SchemeBuilder.Register(RegisterConversions)
     }
src/k8s.io/kubernetes/pkg/apis/ batch/v1/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/batch/v2alpha1**

src/k8s.io/kubernetes/pkg/apis/batch/v2alpha1/generated.pb.go

     var(
          ErrInvalidLengthGenerated = fmt.Errorf("proto: negative length found during unamrshaling")
          ErrIntOverflowGenerated = fmt.Errorf("proto: integer overflow")
     )

src/k8s.io/kubernetes/pkg/apis/ batch/v2alpha1/register.go

     var(
          SchemeBuilder runtime.SchemeBuilder
          localSchemeBuilder = &SchemeBuilder
          AddToScheme = localSchemeBuilder.AddToScheme
     )

src/k8s.io/kubernetes/pkg/apis/ batch/ v2alpha1/types.generated.go

     var(
          codecSelferBitsize1234 = uint8(reflect.TypeOf(uint(0)).Bits())
          codecSelferOnlyMapOrArrayEncodeToStructErr1234 = errors.New(`only encoded map or array can be decoded into a struct`)
     )

src/k8s.io/kubernetes/pkg/apis/ batch/ v2alpha1/types_swagger_doc_generated.go

     var map_CronJob= map[string]string{
          "":"",
          ...
     }
     ...

src/k8s.io/kubernetes/pkg/apis/ batch/ v2alpha1/generated.pb.go

     func init(){
          proto.RegisterType((*CronJob)(nil), "k8s.io.kubernetes.pkg.apis. batch. v2alpha1.CronJob")
          ...
     }
     func init(){
          proto.RegisterFile("k8s.io/kubernetes/pkg/apis/ batch/ v2alpha1/generated.proto", fileDescriptorGenerated)
     }

src/k8s.io/kubernetes/pkg/apis/ batch/ v2alpha1/register.go

     func init(){
          localSchemeBuilder.Register(addKnownTypes, addDefaultingFuncs, addConversionFuncs)
     }

src/k8s.io/kubernetes/pkg/apis/ batch/ v2alpha1/types.generated.go

     func init(){
          if codec1978.GenVersion != 5{
               _,file,_,_:=runtime.Caller(0)
               err:=fmt.Errorf("codecgen version mismatch: current: %v, need %v. Re-generate file: %v",5,codec1978.GenVersion,file)
               panic(err)
          }
          if false{
               var v0 pkg5_resource.Quantity
               var v1 pkg1_v1.TypeMeta
               var v2 pkg2_types.UID
               var v3 pkg6_intstr.IntOrString
               var v4 pkg4_v1.PodTemplateSpec
               var v5 pkg3_v1.JobSpec
               var v6 time.Time
               _,_,_,_,_,_,_ = v0,v1,v2,v3,v4,v5,v6
          }
     }

src/k8s.io/kubernetes/pkg/apis/ batch/ v2alpha1/zz_generated.conversion.go

     func init(){
          SchemeBuilder.Register(RegisterConversions)
     }
src/k8s.io/kubernetes/pkg/apis/ batch/ v2alpha1/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/batch/install**

src/k8s.io/kubernetes/pkg/apis/batch/install/install.go

     func init(){
          Install(api.GroupFactoryRegistry, api.Registry, api.Scheme)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/certificates**

src/k8s.io/kubernetes/pkg/apis/certificates/register.go

     var(
          SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)
          AddToScheme = SchemeBuilder.AddToScheme
     )

src/k8s.io/kubernetes/pkg/apis/ certificates/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/certificates/v1beta1**

src/k8s.io/kubernetes/pkg/apis/certificates/v1beta1/generated.pb.go

     var(
          ErrInvalidLengthGenerated = fmt.Errorf("proto: negative length found during unamrshaling")
          ErrIntOverflowGenerated = fmt.Errorf("proto: integer overflow")
     )

src/k8s.io/kubernetes/pkg/apis/ certificates/ v1beta1/register.go

     var(
          SchemeBuilder runtime.SchemeBuilder
          localSchemeBuilder = &SchemeBuilder
          AddToScheme = localSchemeBuilder.AddToScheme
     )

src/k8s.io/kubernetes/pkg/apis/ certificates/ v1beta1/types.generated.go

     var(
          codecSelferBitsize1234 = uint8(reflect.TypeOf(uint(0)).Bits())
          codecSelferOnlyMapOrArrayEncodeToStructErr1234 = errors.New(`only encoded map or array can be decoded into a struct`)
     )

src/k8s.io/kubernetes/pkg/apis/ certificates/ v1beta1/types_swagger_doc_generated.go

     var map_CertificateSigningRequest= map[string]string{
          "":"",
          ...
     }
     ...

src/k8s.io/kubernetes/pkg/apis/ certificates/ v1beta1/generated.pb.go

     func init(){
          proto.RegisterType((*CertificateSigningRequest)(nil), "k8s.io.kubernetes.pkg.apis. certificates. v1beta1. CertificateSigningRequest")
          ...
     }
     func init(){
          proto.RegisterFile("k8s.io/kubernetes/pkg/apis/ certificates/ v1beta1/generated.proto", fileDescriptorGenerated)
     }

src/k8s.io/kubernetes/pkg/apis/ certificates/ v1beta1/register.go

     func init(){
          localSchemeBuilder.Register(addKnownTypes, addDefaultingFuncs, addConversionFuncs)
     }

src/k8s.io/kubernetes/pkg/apis/ certificates/ v1beta1/types.generated.go

     func init(){
          if codec1978.GenVersion != 5{
               _,file,_,_:=runtime.Caller(0)
               err:=fmt.Errorf("codecgen version mismatch: current: %v, need %v. Re-generate file: %v",5,codec1978.GenVersion,file)
               panic(err)
          }
          if false{
               var v0 pkg1_v1.TypeMeta
               var v1 pkg2_types.UID
               var v3 time.Time
               _,_,_ = v0,v1,v2
          }
     }

src/k8s.io/kubernetes/pkg/apis/ certificates/ v1beta1/zz_generated.conversion.go

     func init(){
          SchemeBuilder.Register(RegisterConversions)
     }
src/k8s.io/kubernetes/pkg/apis/ certificates/ v1beta1/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/certificates/install**

src/k8s.io/kubernetes/pkg/apis/certificates/install/install.go

     func init(){
          Install(api.GroupFactoryRegistry, api.Registry, api.Scheme)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/networking**

src/k8s.io/kubernetes/pkg/apis/networking/register.go

     var(
          SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)
          AddToScheme = SchemeBuilder.AddToScheme
     )

src/k8s.io/kubernetes/pkg/apis/ networking/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/extensions/v1beta1**

src/k8s.io/kubernetes/pkg/apis/extensions/v1beta1/generated.pb.go

     var(
          ErrInvalidLengthGenerated = fmt.Errorf("proto: negative length found during unamrshaling")
          ErrIntOverflowGenerated = fmt.Errorf("proto: integer overflow")
     )

src/k8s.io/kubernetes/pkg/apis/ extensions/ v1beta1/register.go

     var(
          SchemeBuilder runtime.SchemeBuilder
          localSchemeBuilder = &SchemeBuilder
          AddToScheme = localSchemeBuilder.AddToScheme
     )

src/k8s.io/kubernetes/pkg/apis/ extensions/ v1beta1/types.generated.go

     var(
          codecSelferBitsize1234 = uint8(reflect.TypeOf(uint(0)).Bits())
          codecSelferOnlyMapOrArrayEncodeToStructErr1234 = errors.New(`only encoded map or array can be decoded into a struct`)
     )

src/k8s.io/kubernetes/pkg/apis/ extensions/ v1beta1/types_swagger_doc_generated.go

     var map_APIVersion= map[string]string{
          "":"",
          ...
     }
     ...

src/k8s.io/kubernetes/pkg/apis/ extensions/ v1beta1/generated.pb.go

     func init(){
          proto.RegisterType((*APIVersion)(nil), "k8s.io.kubernetes.pkg.apis. extensions. v1beta1. APIVersion")
          ...
     }
     func init(){
          proto.RegisterFile("k8s.io/kubernetes/pkg/apis/ extensions/ v1beta1/generated.proto", fileDescriptorGenerated)
     }

src/k8s.io/kubernetes/pkg/apis/ extensions/ v1beta1/register.go

     func init(){
          localSchemeBuilder.Register(addKnownTypes, addDefaultingFuncs, addConversionFuncs)
     }

src/k8s.io/kubernetes/pkg/apis/ extensions/ v1beta1/types.generated.go

     func init(){
          if codec1978.GenVersion != 5{
               _,file,_,_:=runtime.Caller(0)
               err:=fmt.Errorf("codecgen version mismatch: current: %v, need %v. Re-generate file: %v",5,codec1978.GenVersion,file)
               panic(err)
          }
          if false{
               var v0 pkg3_resource.Quantity
               var v1 pkg1_v1.TypeMeta
               var v2 pkg2_types.UID
               var v3 pkg5_intstr.IntOrString
               var v4 pkg4_v1.PodTemplateSpec
               var v5 time.Time
               _,_,_,_,_,_ = v0,v1,v2,v3,v4,v5
          }
     }

src/k8s.io/kubernetes/pkg/apis/ extensions/ v1beta1/zz_generated.conversion.go

     func init(){
          SchemeBuilder.Register(RegisterConversions)
     }
src/k8s.io/kubernetes/pkg/apis/ extensions/ v1beta1/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/extensions/install**

src/k8s.io/kubernetes/pkg/apis/extensions/install/install.go

     func init(){
          Install(api.GroupFactoryRegistry, api.Registry, api.Scheme)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/networking/v1**

src/k8s.io/kubernetes/pkg/apis/networking/v1/generated.pb.go

     var(
          ErrInvalidLengthGenerated = fmt.Errorf("proto: negative length found during unamrshaling")
          ErrIntOverflowGenerated = fmt.Errorf("proto: integer overflow")
     )

src/k8s.io/kubernetes/pkg/apis/ networking/v1/register.go

     var(
          SchemeBuilder runtime.NewSchemeBuilder(addKnownTypes, addDefaultingFuncs, addConversionFuncs)
          AddToScheme = SchemeBuilder.AddToScheme
     )

src/k8s.io/kubernetes/pkg/apis/ networking/v1/types.generated.go

     var(
          codecSelferBitsize1234 = uint8(reflect.TypeOf(uint(0)).Bits())
          codecSelferOnlyMapOrArrayEncodeToStructErr1234 = errors.New(`only encoded map or array can be decoded into a struct`)
     )

src/k8s.io/kubernetes/pkg/apis/ networking/v1/types_swagger_doc_generated.go

     var map_NetworkPolicy= map[string]string{
          "":"",
          ...
     }
     ...

src/k8s.io/kubernetes/pkg/apis/ networking/v1/generated.pb.go

     func init(){
          proto.RegisterType((*NetworkPolicy)(nil), "k8s.io.kubernetes.pkg.apis. networking.v1. NetworkPolicy")
          ...
     }
     func init(){
          proto.RegisterFile("k8s.io/kubernetes/pkg/apis/ networking/v1/generated.proto", fileDescriptorGenerated)
     }

src/k8s.io/kubernetes/pkg/apis/ networking/v1/register.go

     func init(){
          localSchemeBuilder.Register(addKnownTypes, addDefaultingFuncs, addConversionFuncs)
     }

src/k8s.io/kubernetes/pkg/apis/ networking/v1/types.generated.go

     func init(){
          if codec1978.GenVersion != 5{
               _,file,_,_:=runtime.Caller(0)
               err:=fmt.Errorf("codecgen version mismatch: current: %v, need %v. Re-generate file: %v",5,codec1978.GenVersion,file)
               panic(err)
          }
          if false{
               var v0 pkg1_v1.TypeMeta
               var v1 pkg2_types.UID
               var v2 pkg4_intstr.IntOrString
               var v3 pkg3_v1.Protocol
               var v4 time.Time
               _,_,_,_,_ = v0,v1,v2,v3,v4
          }
     }

src/k8s.io/kubernetes/pkg/apis/ networking/v1/zz_generated.conversion.go

     func init(){
          SchemeBuilder.Register(RegisterConversions)
     }
src/k8s.io/kubernetes/pkg/apis/ networking/v1/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/networking/install**

src/k8s.io/kubernetes/pkg/apis/networking/install/install.go

     func init(){
          Install(api.GroupFactoryRegistry, api.Registry, api.Scheme)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/policy**

src/k8s.io/kubernetes/pkg/apis/policy/register.go

     var(
          SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)
          AddToScheme = SchemeBuilder.AddToScheme
     )

src/k8s.io/kubernetes/pkg/apis/ policy/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/policy/v1beta1**

src/k8s.io/kubernetes/pkg/apis/policy/v1beta1/generated.pb.go

     var(
          ErrInvalidLengthGenerated = fmt.Errorf("proto: negative length found during unamrshaling")
          ErrIntOverflowGenerated = fmt.Errorf("proto: integer overflow")
     )

src/k8s.io/kubernetes/pkg/apis/ policy/ v1beta1/register.go

     var(
          SchemeBuilder runtime.SchemeBuilder
          localSchemeBuilder = &SchemeBuilder
          AddToScheme = localSchemeBuilder.AddToScheme
     )

src/k8s.io/kubernetes/pkg/apis/ policy/ v1beta1/types.generated.go

     var(
          codecSelferBitsize1234 = uint8(reflect.TypeOf(uint(0)).Bits())
          codecSelferOnlyMapOrArrayEncodeToStructErr1234 = errors.New(`only encoded map or array can be decoded into a struct`)
     )

src/k8s.io/kubernetes/pkg/apis/ policy/ v1beta1/types_swagger_doc_generated.go

     var map_Eviction= map[string]string{
          "":"",
          ...
     }
     ...

src/k8s.io/kubernetes/pkg/apis/ policy/ v1beta1/generated.pb.go

     func init(){
          proto.RegisterType((*Eviction)(nil), "k8s.io.kubernetes.pkg.apis. policy. v1beta1. Eviction")
          ...
     }
     func init(){
          proto.RegisterFile("k8s.io/kubernetes/pkg/apis/ policy/ v1beta1/generated.proto", fileDescriptorGenerated)
     }

src/k8s.io/kubernetes/pkg/apis/ policy/ v1beta1/register.go

     func init(){
          localSchemeBuilder.Register(addKnownTypes, addDefaultingFuncs, addConversionFuncs)
     }

src/k8s.io/kubernetes/pkg/apis/ policy/ v1beta1/types.generated.go

     func init(){
          if codec1978.GenVersion != 5{
               _,file,_,_:=runtime.Caller(0)
               err:=fmt.Errorf("codecgen version mismatch: current: %v, need %v. Re-generate file: %v",5,codec1978.GenVersion,file)
               panic(err)
          }
          if false{
               var v0 pkg2_v1.LabelSelector
               var v1 pkg3_types.UID
               var v2 pkg1_intstr.IntOrString
               var v3 time.Time
               _,_,_,_ = v0,v1,v2,v3
          }
     }

src/k8s.io/kubernetes/pkg/apis/ policy/ v1beta1/zz_generated.conversion.go

     func init(){
          SchemeBuilder.Register(RegisterConversions)
     }
src/k8s.io/kubernetes/pkg/apis/ policy/ v1beta1/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/policy/install**

src/k8s.io/kubernetes/pkg/apis/policy/install/install.go

     func init(){
          Install(api.GroupFactoryRegistry, api.Registry, api.Scheme)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/rbac**

src/k8s.io/kubernetes/pkg/apis/rbac/register.go

     var(
          SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)
          AddToScheme = SchemeBuilder.AddToScheme
     )

src/k8s.io/kubernetes/pkg/apis/ rbac/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/rbac/v1alpha1**

src/k8s.io/kubernetes/pkg/apis/rbac/v1alpha1/generated.pb.go

     var(
          ErrInvalidLengthGenerated = fmt.Errorf("proto: negative length found during unamrshaling")
          ErrIntOverflowGenerated = fmt.Errorf("proto: integer overflow")
     )

src/k8s.io/kubernetes/pkg/apis/rbac/v1alpha1/register.go

     var(
          SchemeBuilder runtime.SchemeBuilder
          localSchemeBuilder = &SchemeBuilder
          AddToScheme = localSchemeBuilder.AddToScheme
     )

src/k8s.io/kubernetes/pkg/apis/rbac/v1alpha1/types.generated.go

     var(
          codecSelferBitsize1234 = uint8(reflect.TypeOf(uint(0)).Bits())
          codecSelferOnlyMapOrArrayEncodeToStructErr1234 = errors.New(`only encoded map or array can be decoded into a struct`)
     )

src/k8s.io/kubernetes/pkg/apis/rbac/v1alpha1/types_swagger_doc_generated.go

     var map_ClusterRole= map[string]string{
          "":"",
          ...
     }
     ...

src/k8s.io/kubernetes/pkg/apis/rbac/v1alpha1/generated.pb.go

     func init(){
          proto.RegisterType((*ClusterRole)(nil), "k8s.io.kubernetes.pkg.apis.rbac.v1alpha1.ClusterRole")
          ...
     }
     func init(){
          proto.RegisterFile("k8s.io/kubernetes/pkg/apis/rbac/v1alpha1/generated.proto", fileDescriptorGenerated)
     }

src/k8s.io/kubernetes/pkg/apis/rbac/v1alpha1/register.go

     func init(){
          localSchemeBuilder.Register(addKnownTypes, addDefaultingFuncs, addConversionFuncs)
     }

src/k8s.io/kubernetes/pkg/apis/rbac/v1alpha1/types.generated.go

     func init(){
          if codec1978.GenVersion != 5{
               _,file,_,_:=runtime.Caller(0)
               err:=fmt.Errorf("codecgen version mismatch: current: %v, need %v. Re-generate file: %v",5,codec1978.GenVersion,file)
               panic(err)
          }
          if false{
               var v0 pkg1_v1.TypeMeta
               var v1 pkg2_types.UID
               var v2 time.Time
               _,_,_= v0,v1,v2
          }
     }

src/k8s.io/kubernetes/pkg/apis/rbac/v1alpha1/zz_generated.conversion.go

     func init(){
          SchemeBuilder.Register(RegisterConversions)
     }
src/k8s.io/kubernetes/pkg/apis/rbac/v1lapha1/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/rbac/v1beta1**

src/k8s.io/kubernetes/pkg/apis/rbac/v1beta1/generated.pb.go

     var(
          ErrInvalidLengthGenerated = fmt.Errorf("proto: negative length found during unamrshaling")
          ErrIntOverflowGenerated = fmt.Errorf("proto: integer overflow")
     )

src/k8s.io/kubernetes/pkg/apis/rbac/ v1beta1/register.go

     var(
          SchemeBuilder runtime.SchemeBuilder
          localSchemeBuilder = &SchemeBuilder
          AddToScheme = localSchemeBuilder.AddToScheme
     )

src/k8s.io/kubernetes/pkg/apis/rbac/ v1beta1/types.generated.go

     var(
          codecSelferBitsize1234 = uint8(reflect.TypeOf(uint(0)).Bits())
          codecSelferOnlyMapOrArrayEncodeToStructErr1234 = errors.New(`only encoded map or array can be decoded into a struct`)
     )

src/k8s.io/kubernetes/pkg/apis/rbac/ v1beta1/types_swagger_doc_generated.go

     var map_ClusterRole= map[string]string{
          "":"",
          ...
     }
     ...

src/k8s.io/kubernetes/pkg/apis/rbac/ v1beta1/generated.pb.go

     func init(){
          proto.RegisterType((*ClusterRole)(nil), "k8s.io.kubernetes.pkg.apis.rbac. v1beta1.ClusterRole")
          ...
     }
     func init(){
          proto.RegisterFile("k8s.io/kubernetes/pkg/apis/rbac/ v1beta1/generated.proto", fileDescriptorGenerated)
     }

src/k8s.io/kubernetes/pkg/apis/rbac/ v1beta1/register.go

     func init(){
          localSchemeBuilder.Register(addKnownTypes, addDefaultingFuncs, addConversionFuncs)
     }

src/k8s.io/kubernetes/pkg/apis/rbac/ v1beta1/types.generated.go

     func init(){
          if codec1978.GenVersion != 5{
               _,file,_,_:=runtime.Caller(0)
               err:=fmt.Errorf("codecgen version mismatch: current: %v, need %v. Re-generate file: %v",5,codec1978.GenVersion,file)
               panic(err)
          }
          if false{
               var v0 pkg1_v1.TypeMeta
               var v1 pkg2_types.UID
               var v2 time.Time
               _,_,_= v0,v1,v2
          }
     }

src/k8s.io/kubernetes/pkg/apis/rbac/ v1beta1/zz_generated.conversion.go

     func init(){
          SchemeBuilder.Register(RegisterConversions)
     }
src/k8s.io/kubernetes/pkg/apis/rbac/ v1beta1/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/rbac/install**

src/k8s.io/kubernetes/pkg/apis/rbac/install/install.go

     func init(){
          Install(api.GroupFactoryRegistry, api.Registry, api.Scheme)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/settings**

src/k8s.io/kubernetes/pkg/apis/settings/register.go

     var(
          SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)
          AddToScheme = SchemeBuilder.AddToScheme
     )

src/k8s.io/kubernetes/pkg/apis/settings/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/settings/v1alpha1**

src/k8s.io/kubernetes/pkg/apis/settings/v1alpha1/generated.pb.go

     var(
          ErrInvalidLengthGenerated = fmt.Errorf("proto: negative length found during unamrshaling")
          ErrIntOverflowGenerated = fmt.Errorf("proto: integer overflow")
     )

src/k8s.io/kubernetes/pkg/apis/ settings/v1alpha1/register.go

     var(
          SchemeBuilder runtime.SchemeBuilder
          localSchemeBuilder = &SchemeBuilder
          AddToScheme = localSchemeBuilder.AddToScheme
     )


src/k8s.io/kubernetes/pkg/apis/ settings/v1alpha1/types_swagger_doc_generated.go

     var map_PodPreset= map[string]string{
          "":"",
          ...
     }
     ...

src/k8s.io/kubernetes/pkg/apis/ settings/v1alpha1/generated.pb.go

     func init(){
          proto.RegisterType((*PodPreset)(nil), "k8s.io.kubernetes.pkg.apis. settings.v1alpha1.PodPreset")
          ...
     }
     func init(){
          proto.RegisterFile("k8s.io/kubernetes/pkg/apis/ settings/v1alpha1/generated.proto", fileDescriptorGenerated)
     }

src/k8s.io/kubernetes/pkg/apis/ settings/v1alpha1/register.go

     func init(){
          localSchemeBuilder.Register(addKnownTypes, addDefaultingFuncs, addConversionFuncs)
     }


src/k8s.io/kubernetes/pkg/apis/ settings/v1alpha1/zz_generated.conversion.go

     func init(){
          SchemeBuilder.Register(RegisterConversions)
     }
src/k8s.io/kubernetes/pkg/apis/ settings/v1lapha1/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/settings/install**

src/k8s.io/kubernetes/pkg/apis/settings/install/install.go

     func init(){
          Install(api.GroupFactoryRegistry, api.Registry, api.Scheme)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/storage**

src/k8s.io/kubernetes/pkg/apis/storage/register.go

     var(
          SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)
          AddToScheme = SchemeBuilder.AddToScheme
     )

src/k8s.io/kubernetes/pkg/apis/ storage/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/storage/v1**

src/k8s.io/kubernetes/pkg/apis/storage/v1/generated.pb.go

     var(
          ErrInvalidLengthGenerated = fmt.Errorf("proto: negative length found during unamrshaling")
          ErrIntOverflowGenerated = fmt.Errorf("proto: integer overflow")
     )

src/k8s.io/kubernetes/pkg/apis/storage/v1/register.go

     var(
          SchemeBuilder runtime.SchemeBuilder
          localSchemeBuilder = &SchemeBuilder
          AddToScheme = localSchemeBuilder.AddToScheme
     )

src/k8s.io/kubernetes/pkg/apis/ storage/v1/types_swagger_doc_generated.go

     var map_StorageClass= map[string]string{
          "":"",
          ...
     }
     ...

src/k8s.io/kubernetes/pkg/apis/ storage/v1/generated.pb.go

     func init(){
          proto.RegisterType((*StorageClass)(nil), "k8s.io.kubernetes.pkg.apis. storage.v1.StorageClass")
          ...
     }
     func init(){
          proto.RegisterFile("k8s.io/kubernetes/pkg/apis/ storage/v1/generated.proto", fileDescriptorGenerated)
     }

src/k8s.io/kubernetes/pkg/apis/ storage/v1/register.go

     func init(){
          localSchemeBuilder.Register(addKnownTypes, addDefaultingFuncs, addConversionFuncs)
     }

src/k8s.io/kubernetes/pkg/apis/ storage/v1/zz_generated.conversion.go

     func init(){
          SchemeBuilder.Register(RegisterConversions)
     }
src/k8s.io/kubernetes/pkg/apis/ storage/v1/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/storage/v1beta1**

src/k8s.io/kubernetes/pkg/apis/storage/v1beta1/generated.pb.go

     var(
          ErrInvalidLengthGenerated = fmt.Errorf("proto: negative length found during unamrshaling")
          ErrIntOverflowGenerated = fmt.Errorf("proto: integer overflow")
     )

src/k8s.io/kubernetes/pkg/apis/storage/ v1beta1/register.go

     var(
          SchemeBuilder runtime.SchemeBuilder
          localSchemeBuilder = &SchemeBuilder
          AddToScheme = localSchemeBuilder.AddToScheme
     )

src/k8s.io/kubernetes/pkg/apis/ storage/ v1beta1/types.generated.go

     var(
          codecSelferBitsize1234 = uint8(reflect.TypeOf(uint(0)).Bits())
          codecSelferOnlyMapOrArrayEncodeToStructErr1234 = errors.New(`only encoded map or array can be decoded into a struct`)
     )

src/k8s.io/kubernetes/pkg/apis/ storage/ v1beta1/types_swagger_doc_generated.go

     var map_StorageClass= map[string]string{
          "":"",
          ...
     }
     ...

src/k8s.io/kubernetes/pkg/apis/ storage/ v1beta1/generated.pb.go

     func init(){
          proto.RegisterType((*StorageClass)(nil), "k8s.io.kubernetes.pkg.apis. storage. v1beta1.StorageClass")
          ...
     }
     func init(){
          proto.RegisterFile("k8s.io/kubernetes/pkg/apis/ storage/ v1beta1/generated.proto", fileDescriptorGenerated)
     }

src/k8s.io/kubernetes/pkg/apis/ storage/ v1beta1/register.go

     func init(){
          localSchemeBuilder.Register(addKnownTypes, addDefaultingFuncs, addConversionFuncs)
     }

src/k8s.io/kubernetes/pkg/apis/storage/v1beta1/types.generated.go

     func init(){
          if codec1978.GenVersion != 5{
               _,file,_,_:=runtime.Caller(0)
               err:=fmt.Errorf("codecgen version mismatch: current: %v, need %v. Re-generate file: %v",5,codec1978.GenVersion,file)
               panic(err)
          }
          if false{
               var v0 pkg1_v1.TypeMeta
               var v1 pkg2_types.UID
               var v2 time.Time
               _,_,_= v0,v1,v2
          }
     }

src/k8s.io/kubernetes/pkg/apis/ storage/ v1beta1/zz_generated.conversion.go

     func init(){
          SchemeBuilder.Register(RegisterConversions)
     }
src/k8s.io/kubernetes/pkg/apis/ storage/ v1beta1/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/storage/install**

src/k8s.io/kubernetes/pkg/apis/storage/install/install.go

     func init(){
          Install(api.GroupFactoryRegistry, api.Registry, api.Scheme)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/componentconfig**

src/k8s.io/kubernetes/pkg/apis/componentconfig/register.go

     var(
          SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)
          AddToScheme = SchemeBuilder.AddToScheme
     )

src/k8s.io/kubernetes/pkg/apis/componentconfig/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/client/clientset_generated/clientset/scheme**

src/k8s.io/kubernetes/pkg/client/clientset_generated/clientset/scheme/register.go

     var Scheme = runtime.NewScheme()
     var Codec = serializer.NewCodecFactory(Scheme)
     var ParameterCodec = runtime.NewParameterCodec(Scheme0
     func init(){
          v1.AddToGroupVersion(Scheme, schema.GroupVersion{Version: "v1"})
          AddToScheme(Scheme)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/api/v1/ref**

src/k8s.io/kubernetes/pkg/api/v1/ref/ref.go

     var(
          ErrNilObject = errors.New("can't reference a nil object")
          ErrNoSelfLink = errors.New("selfLink was empty, can't make reference")
     )

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/client/clientset_generated/clientset**

src/k8s.io/kubernetes/pkg/client/clientset_generated/clientset/import_known_versions.go

     func init(){
          if missingVersions := api.Registry.ValidateEnvRequestedVersions(); len(missingVersions) != 0{
               panic(fmt.Sprintf("KUBE_API_VERSIONS contains versions that are not installed: %q.", missingVersions))
          }
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/api/v1/helper/qos**

src/k8s.io/kubernetes/pkg/api/v1/helper/qos/qos.go

     var supportedQoSComputeResources = sets.NewString(string(v1.ResourceCPU), string(v1.ResourceMemory))

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/componentconfig/v1alpha1**

src/k8s.io/kubernetes/pkg/apis/componentconfig/v1alpha1/register.go

     var(
          SchemeBuilder runtime.SchemeBuilder
          localSchemeBuilder = &SchemeBuilder
          AddToScheme = localSchemeBuilder.AddToScheme
     )
     func init(){
          localSchemeBuilder.Register(addKnownTypes, addDefaultingFuncs)
     }

src/k8s.io/kubernetes/pkg/apis/componentconfig/v1alpha1/zz_generated.conversion.go

     func init(){
          SchemeBuilder.Register(RegisterConversions)
     }

src/k8s.io/kubernetes/pkg/apis/componentconfig/v1alpha1/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/componentconfig/install**

src/k8s.io/kubernetes/pkg/apis/componentconfig/install/install.go

     func init(){
          Install(api.GroupFactoryRegistry, api.Registry, api.Scheme)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/client/clientset_generated/internalclientset/scheme**

src/k8s.io/kubernetes/pkg/client/clientset_generated/internalclientset/scheme/register.go

     var Scheme = runtime.NewScheme()
     var Codecs = serializer.NewCodecFactory(Scheme)
     var ParameterCodec = runtime.NewParameterCodec(Scheme)
     var Registry = registered.NewOrDir(os.Getenv("KUBE_API_VERSIONS"))
     var GroupFactoryRegistry = make(announced.APIGroupFactoryRegistry))

     func init(){
          v1.AddToGroupVersion(Scheme, schema.GroupVersion{Version: "v1"})
          Install(GroupFactoryRegistry, Registry, Scheme)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/api/ref**

src/k8s.io/kubernetes/pkg/api/ref/ref.go

     var(
          ErrNilObject = errors.New("can't reference a nil object")
          ErrNoSelfLink = errors.New("selfLink was empty, can't make reference")
     )

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/abac**

src/k8s.io/kubernetes/pkg/apis/abac/register.go

     var Scheme = runtime.NewScheme()
     var Codecs = serializer.NewCodecFactory(Scheme)
     var(
          SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)
          AddToScheme = SchemeBuilder.AddToScheme
     )
     func init(){
          addKnownTypes(Scheme)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/abac/v0**

src/k8s.io/kubernetes/pkg/apis/abac/v0/register.go

     func init(){
          if err:=addKnownTypes(api.Scheme);err!=nil{
               panic(err)
          }
          if err:=addConversionFuncs(api.Scheme);err!=nil{
               panic(err)
          }
     }
     func init(){
          localSchemeBuilder.Register(addKnownTypes, addConversionFuncs)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/abac/v1beta1**

src/k8s.io/kubernetes/pkg/apis/abac/v1beta1/register.go

     var(
          SchemeBuilder runtime.SchemeBuilder
          localSchemeBuilder = &SchemeBuilder
          AddToScheme = localSchemeBuilder.AddToScheme
     )
     func init(){
          if err:=addKnownTypes(api.Scheme); err!=nil{
               panic(err)
          }
          if err:=addConversionFuncs(api.Scheme); err!=nil{
               panic(err)
          }
     }
     func init(){
          localSchemeBuilder.Register(addKnownTypes, addConversionFuncs, RegisterDefaults)
     }

src/k8s.io/kubernetes/pkg/apis/abac/v1beta1/zz_generated.conversion.go

     func init(){
          SchemeBuilder.Register(RegisterConversions)
     }

src/k8s.io/kubernetes/pkg/apis/abac/v1beta1/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/golang.org/x/tools/container/intsets**

k8s.io/kubernetes/vendor/golang.org/x/tools/container/intsets/popcnt_amd64.go

     var hasPOPCNT = havePOPCNT()

     func init(){
          for i:=range a{
               var n byte
               for x:=i;x!=0;x>>=1{
                    if x&1 != 0{
                         n++
                    }
               }
               a[i] = n
          }
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/plugin/pkg/auth/authorizer/node**

src/k8s.io/kubernetes/plugin/pkg/auth/authorizer/node/graph.go

     var vertexTypes = map[vertexType]string{
          configMapVertexType: "configmap",
          nodeVertexType: "node",
          podVertexType: "pod",
          pvcVertexType: "pvc",
          pvVertexType: "pv",
          secretVertexType: "secret",
     }

src/k8s.io/kubernetes/plugin/pkg/auth/authorizer/node/node_authorizer.go

     var(
          configMapResource = api.Resource("configmaps")
          secretResource = api.Resource("secrets")
          pvcResource = api.Resource("persistentvolumeclaims")
          pvResource = api.Resource("persistentvolumes")
     )

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/plugin/pkg/auth/authorizer/rbac/bootstrappolicy**

src/k8s.io/kubernetes/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/namespace_policy.go

     var(
          namespaceRoles = map[string][]rbac.Role{}
          namespaceRoleBindings = map[string][]rbac.RoleBinding{}
     )
     
src/k8s.io/kubernetes/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/policy.go

     var(
          ReadWrite = []string{"get", "list", "watch", "create", "update", "patch", "delete", "deletecollection"}
          Read = []string{"get", "list", "watch"}
          Label = map[string]string{"kubernetes.io/bootstrapping": "rbac-defaults"}
          Annotation = map[string]string{rbac.AutoUpdateAnnotationKey: "true"}
     )

src/k8s.io/kubernetes/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/controller_policy.go

     func init(){
          addControllerRole(rbac.ClusterRole{
               ObjectMeta: metav1.ObjectMeta{Name: saRolePrefix + "attachedetach-controller"},
               Rules: []rbac.PolicyRule{
                    rbac.NewRule("list", "watch").Groups(legacyGroup).Resources("persistentvolumes", "persistentvolumeclaims").RuleOrDie(),
                    ...
               },
          })
          ...
     }

src/k8s.io/kubernetes/plugin/pkg/auth/authorizer/rbac/bootstrappolicy/namespace_policy.go

     func init(){
          addNamespaceRole(metav1.NamespaceSystem, rbac.Role{
               ObjectMeta: metav1.ObjectMeta{Name: "extension-apiserver-authentication-reader"},
               Rules: []rbac.PolicyRule{
                    rbac.NewRule("get").Groups(legacyGroup).Resources("configmaps").Names("extension-apiserver-authentication").RuleOrDie(),
               },
          })
          ...
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/cloudprovider**

src/k8s.io/kubernetes/pkg/cloudprovider/cloud.go

     var(
          InstanceNotFound = errors.New("instance not found")
          DiskNotFound = errors.New("disk is not found")
     )

src/k8s.io/kubernetes/pkg/cloudprovider/plugins.go

     var(
          providersMutex sync.Mutex
          providers = make(map[string]Factory)
     )

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apiextensions-apiserver/pkg/controller/finalizer**

k8s.io/kubernetes/vendor/k8s.io/apiextensions-apiserver/pkg/controller/finalizer/crd_finalizer.go

     var cloner = conversion.NewCloner()

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apiextensions-apiserver/pkg/controller/status**

k8s.io/kubernetes/vendor/k8s.io/apiextensions-apiserver/pkg/controller/status/naming_controller.go

     var cloner = conversion.NewCloner()

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apiextensions-apiserver/pkg/client/clientset/clientset/scheme**

k8s.io/kubernetes/vendor/k8s.io/apiextensions-apiserver/pkg/client/clientset/clientset/scheme/register.go

     var Scheme = runtime.NewScheme()
     var Codecs = serializer.NewCodecFactory(Scheme)
     var ParameterCodec = runtime.NewParameterCodec(Scheme)
     func init(){
          v1.AddToGroupVersion(Scheme, schema.GroupVersion{Version: "v1"})
          AddToScheme(Scheme)
     }

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/k8s.io/apiextensions-apiserver/pkg/apiserver**

k8s.io/kubernetes/vendor/k8s.io/apiextensions-apiserver/pkg/apiserver/apiserver.go

     var(
          groupFactoryRegistry = make(announced.APIGroupFactoryRegistry)
          registry = registered.NewOorDie("")
          Scheme = runtime.NewScheme()
          Codecs = serializer.NewCodecFactory(Scheme)
          ...
     )
     func init(){
          install.Install(groupFactoryRegistry, registry, Scheme)
          metav1.AddToGroupVersion(Scheme, schema.GroupVersion{Group: "", Version: "v1"})
          Scheme.AddUnversionedTypes(unversionedVersion, unversionedTypes...)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/extensions/validation**

src/k8s.io/kubernetes/pkg/apis/extensions/validation/validation.go

     var sysctlPatternRegexp = regexp.MustCompile("^"+SysctlPatternFmt + "$")

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/gopkg.in/gcfg.v1/types**

k8s.io/kubernetes/vendor/gopkg.in/gcfg.v1/types/bool.go

     var BoolValues = map[string]interface{}{
          "true": true, "yes": true, "on": true, "1": true,
          "false": false, "no": false, "off": false, "0": false,
     }
     var boolParser = func() *EnumParser{
          ep:=&EnumParser{}
          ep.AddVals(BoolValues)
          return ep
     }()

k8s.io/kubernetes/vendor/gopkg.in/gcfg.v1/types/int.go

     var errIntAmbig = fmt.Errorf("ambiguous integer value; must include '0' prefix")

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/gopkg.in/gcfg.v1**

k8s.io/kubernetes/vendor/gopkg.in/gcfg.v1/read.go

     var unescape = map[rune]rune{'\\': '\\', '"':'"', 'n': '\n', 't':'\t'

k8s.io/kubernetes/vendor/gopkg.in/gcfg.v1/set.go

     var errUnsupportedType = fmt.Errorf("unsupported type")
     var errBlankUnsupported = fmt.Errorf("blank value not supported for type")

     var typeModes = map[reflect.Type]types.IntMode{
          reflect.TypeOf(int(0)): types.Dec | types.Hex,
          ...
     }
     var typeSetters = map[reflect.Type[setter{
          reflect.TypeOf(big.Int{}): intSetter,
     }
     var kindSetters = map[reflect.Kind]setter{
          reflect.String: stringSetter,
          ...
     }

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/go-ini/ini**

k8s.io/kubernetes/vendor/github.com/go-ini/ini/ini.go

     var(
          LineBreak = "\n"
          varPattern = regexp.MustCompile(`%\(([^\)]+)\)s`)
          PrettyFormat = true
     )

k8s.io/kubernetes/vendor/github.com/go-ini/ini/struct.go

     var reflectTime = reflect.TypeOf(time.Now()).Kind()

k8s.io/kubernetes/vendor/github.com/go-ini/ini/ini.go

     func init(){
          if runtime.GOOS == "windows"{
               LineBreak = "\r\n"
          }
     }

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/aws/aws-sdk-go/aws/credentials**

k8s.io/kubernetes/vendor/github.com/aws/aws-sdk-go/aws/credentials/chain_provider.go

     var(
          ErrNoValidProvidersFoundInChain = awserr.New("NoCredentialProviders",
               `no valid providers in chain. Deprecated.
          For verbose messaging see aws.Config.CredentialsChainVerboseErrors`,
               nil)
     )

k8s.io/kubernetes/vendor/github.com/aws/aws-sdk-go/aws/credentials/credentials.go

     var AnonymousCredentials = NewStaticCredentials("", "", "")

k8s.io/kubernetes/vendor/github.com/aws/aws-sdk-go/aws/credentials/env_provider.go

     var(
          ErrAccessKeyIDNotFound = awserr.New("EnvAccessKeyNotFound", "AWS_ACCESS_KEY_ID or AWS_ACCESS_KEY not found in environment", nil)
          ErrSecretAccessKeyNotFound = awserr.New("EnvSecretNotFound", "AWS_SECRET_ACCESS_KEY or AWS_SECRET_KEY not found in environment", nil)
     )

k8s.io/kubernetes/vendor/github.com/aws/aws-sdk-go/aws/credentials/shared_credentials_provider.go

     var(
          ErrSharedCredentialsHomeNotFound = awserr.New("UserHomeNotFound", "user home directory not found.", nil)
     )

k8s.io/kubernetes/vendor/github.com/aws/aws-sdk-go/aws/credentials/static_provider.go

     var(
          ErrStaticCredentialsEmpty = awserr.New("EmptyStaticCreds", "static credentials are empty", nil)
     )

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/aws/aws-sdk-go/aws/endpoints**

k8s.io/kubernetes/vendor/github.com/aws/aws-sdk-go/aws/endpoints/defaults.go

     var awsPartition = partition{
          ID: "aws",
          Name: "AWS Standard",
          DNSSuffix: "amazonaws.com",
          ...
     }
     var awscnPartition = partition{
          ID: "aws-cn",
          Name: "AWS China",
          ...
     }
     var awsusgovPartition = partition{
          ID: "aws-us-gov",
          Name: "AWS GovCloud (US)",
          ...
     }

k8s.io/kubernetes/vendor/github.com/aws/aws-sdk-go/aws/endpoints/endpoints.go

     var schemeRE = regexp.MustCompile("^([^:]+)://")

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/aws/aws-sdk-go/aws**

k8s.io/kubernetes/vendor/github.com/aws/aws-sdk-go/aws/errors.go

     var(
          ErrMissingRegion = awserr.New("MissingRegion", "could not find region configuration", nil)
          ErrMissingEndpoint = awserr.New("MissingEndpoint", " 'Endpoint' configuration is required for this service", nil)
     )

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/jmespath/go-jmespath**

k8s.io/kubernetes/vendor/github.com/jmespath/go-jmespath/lexer.go

     var basicTokens = map[rune]tokType{
          '.': tDot,
          '*': tStar,
          ',':tComma,
          ':': tColon,
          '{': tLbrace,
          '}': tRbrace,
          ']': tRbracket,
          '(': tLparen,
          ')': tRparen,
          '@': tCurrent,
     }
     var whiteSpace = map[rune]bool{
          ' ': true, '\t': true, '\n': true, '\r': true,
     }

k8s.io/kubernetes/vendor/github.com/jmespath/go-jmespath/parser.go

     var bindingPowers = map[tokType]int{
          tEOF: 0,
          tUnquotedIdentifier: 0,
          ...
     }

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/aws/aws-sdk-go/aws/awsutil**

k8s.io/kubernetes/vendor/github.com/aws/aws-sdk-go/aws/awsutil/path_value.go

     var indexRe = regexp.MustCompile(`(.+)\[(-?\d+)?\]$`)

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/aws/aws-sdk-go/aws/request**

k8s.io/kubernetes/vendor/github.com/aws/aws-sdk-go/aws/request/retryer.go

     var retryableCodes = map[string]struct{}{
          "RequestError": {},
          "RequestTimeout": {},
     }
     var throttleCodes = map[string]struct{}{
          "ProvisionedThroughputExceededException": {},
          ...
     }
     var credsExpiredCodes = map[string]struct{}{
          "ExpiredToken": {},
          ...
     }

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/aws/aws-sdk-go/aws/client**

k8s.io/kubernetes/vendor/github.com/aws/aws-sdk-go/aws/client/default_retryer.go

     var seededRand = rand.New(&lockedSource{src: rand.NewSource(time.Now().UnixNano())})

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/aws/aws-sdk-go/aws/corehandlers**

k8s.io/kubernetes/vendor/github.com/aws/aws-sdk-go/aws/corehandlers/handlers.go

     var SDKVersionUserAgentHandler = request.NamedHandler{
          Name: "core.SDKVersionUserAgentHandler",
          Fn: request.MakeAddToUserAgentHandler(aws.SDKName, aws.SDKVersion,
               runtime.Version(), runtime.GOOS, runtime.GOARCH),
     }
     var reStatusCode = regexp.MustCompile(`^(\d{3})`)

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/aws/aws-sdk-go/private/protocol/rest**

k8s.io/kubernetes/vendor/github.com/aws/aws-sdk-go/private/protocol/rest/build.go

     var errValueNotSet = fmt.Errorf("value not set")

     func init(){
          for i:=0;i<len(noEscape);i++{
               noEscape[i]=(i>='A' && i<='Z') ||
                    (i>='a' && i<='z') ||
                    (i>='0' && i<='9') ||
                    i=='-' ||
                    i==''.' ||
                    i=='_' ||
                    i=='~' 
          }
     }

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/aws/aws-sdk-go/aws/signer/v4**

k8s.io/kubernetes/vendor/github.com/aws/aws-sdk-go/aws/signer/v4/v4.go

     var ignoredHeaders = rules{
          blacklist{
               mapRule{
                    "Authorization": struct{}{},
                    "User-Agent": struct{}{},
                    "X-Amzn-Trace-Id": struct{}{},
               },
          },
     }
     var requiredSignedHeaders = rules{
          whitelist{
               mapRule{
                    "Cache-Control": struct{}{},
                    ...
               }
          }
     }

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/aws/aws-sdk-go/service/sts**

k8s.io/kubernetes/vendor/github.com/aws/aws-sdk-go/service/sts/customizations.go

     func init(){
          initRequest = func(r *request.Request){
               switch r.Operation.Name{
               case opAssumeRoleWithSAML, opAssumeRoleWithWebIdentity:
                    r.Handlers.Sign.Clear()
               }
          }
     }

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/aws/aws-sdk-go/service/ec2**

k8s.io/kubernetes/vendor/github.com/aws/aws-sdk-go/service/ec2/customizations.go

     func init(){
          initRequest = func(r *request.Request){
               if r.Operation.Name == opCopySnapshot{
                    r.Handlers.Build.PushFront(fillPresignedURL)
               }
          }
     }

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/aws/aws-sdk-go/private/protocol/json/jsonutil**

k8s.io/kubernetes/vendor/github.com/aws/aws-sdk-go/private/protocol/json/jsonutil/build.go

     var timeType = reflect.ValueOf(time.Time{}).Type()
     var byteSliceType = reflect.ValueOf([]byte{}).Type()

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/docker/go-units**

k8s.io/kubernetes/vendor/github.com/docker/go-units/size.go

     var(
          decimalMap = unitMap{"k": KB, "m": MB, "g": GB, "t": TB, "p": PB}
          binaryMap = unitMap{"k": KiB, "m": MiB, "g": GiB, "t": TiB, "p": PiB}
          sizeRegex = regexp.MustCompile(`^(\d+(\.\d+)*)?([kKmMgGtTpP])?[bB]?$`)
     )

k8s.io/kubernetes/vendor/github.com/docker/go-units/ulimit.go

     var ulimitNameMapping = map[string]int{
          "core": rlimitCore,
          ...
     }

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/docker/engine-api/types/filters**

k8s.io/kubernetes/vendor/github.com/docker/engine-api/types/filters/parse.go

     var ErrBadFormat = errors.New("bad format of filter (expected name=value)")

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/credentialprovider**

src/k8s.io/kubernetes/pkg/credentialprovider/config.go

     var(
          preferredPathLock sync.Mutex
          preferredPath = ""
          workingDirPath = ""
          homeDirPath = os.Getenv("HOME")
          rootDirPath = "/"
          homeJsonDirPath = filepath.Join(homeDirPath, ".docker")
          rootJsonDirPath = filepath.Join(rootDirPath, ".docker")
          configFileName = ".dockercfg"
          configJsonFileName = "config.json"
     )

src/k8s.io/kubernetes/pkg/credentialprovider/plugins.go

     var providersMutex sync.Mutex
     var providers = make(map[string]DockerConfigProvider)

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/cloudprovider/providers/aws**

src/k8s.io/kubernetes/pkg/cloudprovider/providers/aws/aws.go

     var awsTagNameMasterRoles = sets.NewString("kubernetes.io/role/master", "k8s.io/role/master")
     var backednProtocolMapping =map[string]string{
          "https": "https",
          "http":"https",
          "ssl": "ssl",
          "tcp": "ssl",
     }

src/k8s.io/kubernetes/pkg/cloudprovider/providers/aws/aws_metrics.go

     var awsApiMetric = prometheus.NewHistogramVec(
          prometheus.HistogramOpts{
               Name: "cloudprovider_aws_api_request_duration_seconds",
               Help: "Latency of aws api call",
          },
          []string{"request"},
     )
     var awsApiErrorMetric = prometheus.NewCounterVec(
          prometheus.CounterOpts{
               Name: "cloudprovider_aws_api_request_errors",
               Help: "AWS Api errors",
          },
          []string{"request"},
     )

src/k8s.io/kubernetes/pkg/cloudprovider/providers/aws/aws.go

     func init(){
          registerMetrics()
          cludprovider.RegisterCloudProvider(ProviderName, func(config io.Reader) (cloudprovider.Interface, error){
               creds:=credentials.NewChainCredentials(
                    []credentials.Provider{
                         &credentials.EnvProvider{},
                         &ec2rolecreds.EC2RoleProvider{
                              Client: ec2metadata.New(session.New(&aws.Config{})),
                         },
                         &credentials.SharedCredentialsProvider{},
                    })
               aws:=newAWSSDKProvider(creds)
               return newAWSCloud(config, aws)
          })
     }

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/Azure/go-autorest/autorest/azure**

k8s.io/kubernetes/vendor/github.com/Azure/go-autorest/autorest/azure/devicetoken.go

     var (
          ErrDeviceGeneric = fmt.Errorf("%s Error while retrieving OAuth token: Unknown Error", logPrefix)
          ...
     )

k8s.io/kubernetes/vendor/github.com/Azure/go-autorest/autorest/azure/environment.go

     var environments = map[string]Environment{
          "AZURECHINACLOUD": ChinaCloud,
          ...
     }

k8s.io/kubernetes/vendor/github.com/Azure/go-autorest/autorest/azure/token.go

     func init(){
          expirationBase,_ = time.Parse(time.RFC3339m tokenBaseDate)
     }

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/Azure/azure-sdk-for-go/storage**

k8s.io/kubernetes/vendor/github.com/Azure/azure-sdk-for-go/storage/blob.go

     var(
          errBlobCopyAborted = errors.New("storage: blob copy is aborted")
          errBlobCopyIDMismatch = errors.New("storage: blob copy id is a mismatch")
     )

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/xanzy/go-cloudstack/cloudstack**

k8s.io/kubernetes/vendor/github.com/xanzy/go-cloudstack/cloudstack/cloudstack.go

     var idRegex = regexp.MustCompile(`^([0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}|-1)$`)
     var AsyncTimeoutErr = errors.New("Timeout while waiting for async job to finish")

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/cloudprovider/providers/cloudstack**

src/k8s.io/kubernetes/pkg/cloudprovider/providers/cloudstack/cloudstack.go

     func init(){
          cloudprovider.RegisterCloudProvider(ProviderName, func(config io.Reader) (cloudprovider.Interface,error){
               cfg,err:=readConfig(config)
               if err!=nil{
                    return nil, err
               }
               return newCSCloud(cfg)
          })
     }

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/google.golang.org/api/googleapi/internal/uritermplates**

k8s.io/kubernetes/vendor/google.golang.org/api/googleapi/internal/uritermplates/uritemplates.go

     var(
          unreserved = regexp.MustCompile("[^A-Za-z0-9\\-._~]")
          reserved = regexp.MustCompile("[^A-Za-z0-9\\-._~:/?#[\\]@!$&'()*+,;=]")
          validname = regexp.MustCompile("^([A-Za-z0-9_\\.]|%[0-9A-Fa-f][0-9A-Fa-f])+$")
          hex = []byte("0123456789ABCDEF")
     )

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/cloudprovider/providers/gce**

src/k8s.io/kubernetes/pkg/cloudprovider/providers/gce/gce.go

     var uselessDNSSearchRE = regexp.MustCompile(`^[0-9]+.google.internal.$`)

src/k8s.io/kubernetes/pkg/cloudprovider/providers/gce/gce_util.go

     var providerIdRE = regexp.MustCompile(`^` + ProviderName + `://([^/]+)/([^/]+)/([^/]+)$`)

src/k8s.io/kubernetes/pkg/cloudprovider/providers/gce/metrics.go

     var(
          apiMetrics = registerAPIMetrics(
               "request",
               "region",
               "zone",
          )
     )

src/k8s.io/kubernetes/pkg/cloudprovider/providers/gce/token_source.go

     var(
          getTokenCounter = prometheus.NewCounter(
               prometheus.CounterOpts{
                    Name: "get_token_count",
                    Help: "Counter of total Token() requests to the alternate token source",
               },
          )
          getTokenFailCounter = prometheus.NewCounter(
               prometheus.CounterOpts{
                    Name: "get_token_fail_count",
                    Help: "Counter of failed Token() requests to the alternate token source",
               },
          )
     )

src/k8s.io/kubernetes/pkg/cloudprovider/providers/gce/gce.go

     func init(){
          cloudprovider.RegisterCloudProvider(
               ProviderName,
               func(config io.Reader)(cloudprovider.Interface, error){
                    return newGCECloud(config)
               })
     }

src/k8s.io/kubernetes/pkg/cloudprovider/providers/gce/gce_healthchecks.go

     func init(){
          if v,err:=utilversion.ParseGeneric("1.7.0"); err!=nil{
               panic(err)
          }else{
               minNodesHealthCheckVersion = v
          }
     }

src/k8s.io/kubernetes/pkg/cloudprovider/providers/gce/gce_loadbalancer.go

     func init(){
          var err error
          lbSrcRngsFlag.ipn, err=netsets.ParseIPNets([]string{"130.221.0.0/22", "35.191.0.0/16, "209.85.152.0/22", "209.85.204.0/22"}...)
          if err!=nil{
               panic("Incorrect default GCE L7 source ranges")
          }
          flag.Var(&lbSrcRngsFlag, "cloud-provider-gce-lb-src-cidrs", "CIDRS opened in GCE firewall for LB traffic proxy & health checks")
     }

src/k8s.io/kubernetes/pkg/cloudprovider/providers/gce/token_source.go

     func init(){
          prometheus.MustRegister(getTokenCounter)
          prometheus.MustRegister(getTokenFailCounter)
     }

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/mesos/mesos-go/mesosproto**

k8s.io/kubernetes/vendor/github.com/mesos/mesos-go/mesosproto/authentication.pb.go

     var(
          ErrInvalidLengthAuthentication = fmt.Errorf("proto: negative length found during unmarshaling")
          ErrIntOverflowAuthentication = fmt.Errorf("proto: integer overflow")
     )

k8s.io/kubernetes/vendor/github.com/mesos/mesos-go/mesosproto/mesos.pb.go

     var Status_name = map[int32]string{
          1: "DRIVER_NOT_STARTED",
          2: "DRIVER_RUNNING",
          3: "DRIVER_ABORTED",
          4: "DRIVER_STOPPED",
     }
     var Status_value = map[string]int32{
          "DRIVER_NOT_STARTED:: 1,
          "DRIVER_RUNNING": 2,
          "DRIVER_ABORTED": 3,
          "DRIVER_STOPPED": 4,
     }
     var TaskState_name = map[int32]string{
          6: "TASK_STAGING",
          ...
     }
     var TaskState_value = map[string]int32{
          ...
     }
     var MachineInfo_Mode_name = map[int32]string{
          ...
     }
     var MachineInfo_Mode_value = map[string]int32{
          ...
     }
     var FrameworkInfo_Capability_Type_name = map[int32]string{
          ...
     }
     var FrameworkInfo_Capability_Type_value = map[string]int32{
          ...
     }
     var Value_Type_name = map[int32]string{
          ...
     }
     var Value_Type_value = map[string]int32{
          ...
     }
     var Offer_Operation_Type_name = map[int32]string{
          ...
     }
     var Offer_Operation_Type_value = map[string]int32{
          ...
     }
     var TaskStatus_Source_name = map[int32]string{
          ...
     }
     var TaskStatus_Source_value = map[string]int32{
          ...
     }
     var TaskStatus_Reason_name = map[int32]string{
          ...
     }
     var TaskStatus_Reason_value = map[string]int32{
          ...
     }
     var Image_Type_name = map[int32]string{
          ...
     }
     var Image_Type_value = map[string]int32{
          ...
     }
     var Volume_Mode_name = map[int32]string{
          ...
     }
     var Volume_Mode_value = map[string]int32{
          ...
     }
     var NetworkInfo_Protocol_name = map[int32]string{
          ...
     }
     var NetworkInfo_Protocol_value = map[string]int32{
          ...
     }
     var ContainerInfo_Type_name = map[int32]string{
          ...
     }
     var ContainerInfo_Type_value = map[string]int32{
          ...
     }
     var ContainerInfo_DockerInfo_Network_name = map[int32]string{
          ...
     }
     var ContainerInfo_DockerInfo_Network_value = map[string]int32{
          ...
     }
     var DiscoveryInfo_Visibility_name = map[int32]string{
          ...
     }
     var DiscoveryInfo_Visibility_value = map[string]int32{
          ...
     }
     var(
          ErrInvalidLengthMesos = fmt.Errorf("proto: negative length found during unmarshaling")
          ErrIntOverflowMesos = fmt.Errorf("proto: integer overflow")
     )

k8s.io/kubernetes/vendor/github.com/mesos/mesos-go/mesosproto/messages.pb.go

     var StatusUpdateRecord_Type_name = map[int32]string{
          ...
     }
     var StatusUpdateRecord_Type_value = map[string]int32{
          ...
     }
     var(
          ErrInvalidLengthMesos = fmt.Errorf("proto: negative length found during unmarshaling")
          ErrIntOverflowMesos = fmt.Errorf("proto: integer overflow")
     )

k8s.io/kubernetes/vendor/github.com/mesos/mesos-go/mesosproto/mesos.pb.go

     func init(){
          proto.RegisterEnum("mesosproto.Status", Status_name, Status_value)
          ...
     }

k8s.io/kubernetes/vendor/github.com/mesos/mesos-go/mesosproto/messages.pb.go

     func init(){
          proto.RegisterEnum("mesosproto.StatusUpdateRecord_Type", StatusUpdateRecord_Type_name, StatusUpdateRecord_Type_value)
     }

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/mesos/mesos-go/detector**

k8s.io/kubernetes/vendor/github.com/mesos/mesos-go/detector/factory.go

     var(
          pluginLock sync.Mutex
          plugins = map[string]PluginFactory{}
          EmptySpecError = errors.New("empty master specification")
          ...
     )

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/samuel/go-zookeeper/zk**

k8s.io/kubernetes/vendor/github.com/samuel/go-zookeeper/zk/conn.go

     var ErrNoServer = errors.New("zk: could not connect to a server")
     var ErrInvalidPath = errors.New("zk: invalid path")

k8s.io/kubernetes/vendor/github.com/samuel/go-zookeeper/zk/constants.go

     var(
          eventNames = map[EventType]string{
               ...
          }
     )
     var(
          stateNames = map[State]string{
               ...
          }
     )
     var(
          ErrConnectionClosed = errors.New("zk: connection closed")
          ...
     )
     var(
          emptyPassword = []byte{0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0}
          opNames = map[int32]string{
               opNotify: "notify",
               ...
          }
     )
     var(
          modeNames = map[Mode]string{
               ModeLeader: "leader",
               ...
          }
     )

k8s.io/kubernetes/vendor/github.com/samuel/go-zookeeper/zk/lock.go

     var(
          ErrDeadlock = errors.New("zk: trying to acquire a lock twice")
          ErrNotLocked = errors.New("zk: not locked")
     )

k8s.io/kubernetes/vendor/github.com/samuel/go-zookeeper/zk/structs.go

     var(
          ErrUnhandledFieldType = errors.New("zk: unhandled field type")
          ...
     )

k8s.io/kubernetes/vendor/github.com/samuel/go-zookeeper/zk/tracer.go

     var(
          requests = make(map[int32]int32)
          requestsLock = &sync.Mutex{}
     )

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/mesos/mesos-go/detector/zoo**

k8s.io/kubernetes/vendor/github.com/mesos/mesos-go/detector/zoo/plugin.go

     func init(){
          detector.Register("zk://", detector.PluginFactory(func(spec string, options ...detector.Option) (detector.Master, error){
               return NewMasterDetector(spec, options...)
          }))
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/cloudprovider/providers/mesos**

src/k8s.io/kubernetes/pkg/cloudprovider/providers/mesos/client.go

     var noLeadingMasterError = errors.New("there is no current leading master available to query")

src/k8s.io/kubernetes/pkg/cloudprovider/providers/mesos/mesos.go

     var(
          CloudProvider *MesosCloud
          noHostNameSpecified = errors.New("No hostname specified")
     )
     func init(){
          cloudprovider.RegisterCloudProvider(
               ProviderName,
               func(configReader io.Reader)(cloudprovider.Interface, error){
                    provider, err:=newMesosCloud(configReader)
                    if err==nil{
                         CloudProvider = provider
                    }
                    return provider, err
               })
     }

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/gophercloud/gophercloud/openstack/networking/v2/extensions/lbaas_v2/monitors**

k8s.io/kubernetes/vendor/github.com/gophercloud/gophercloud/openstack/networking/v2/extensions/lbaas_v2/monitors/requests.go

     var(
          errDelayMustGETTimeout = fmt.Errorf("Dealy must be greater than or equal to timeout")
     )

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/cloudprovider/providers/openstack**

src/k8s.io/kubernetes/pkg/cloudprovider/providers/openstack/metadata.go

     var ErrBadMetadata = errors.New("Invalid OpenStack metadata, got empty uuid")

src/k8s.io/kubernetes/pkg/cloudprovider/providers/openstack/openstack.go

     var ErrNotFound = errors.New("Failed to find object")
     var ErrMultipleResults = errors.New("Multiple results where only one expected")
     var ErrNoAddressFound = errors.New("No address found for host")

src/k8s.io/kubernetes/pkg/cloudprovider/providers/openstack/openstack_metrics.go

     var(
          OpenstackOperationsLatency = prometheus.NewHistogramVec(
               ...
          )
          ...
     )

src/k8s.io/kubernetes/pkg/cloudprovider/providers/openstack/openstack_routes.go

     var ErrNoRouterId = errors.New("router-id not set in cloud provider config")

src/k8s.io/kubernetes/pkg/cloudprovider/providers/openstack/openstack.go

     func init(){
          RegisterMetrics()
          cloudprovider.RegisterCloudProvider(ProviderName, func(config io.Reader)(cloudprovider.Interface,error){
               cfg.err:=readConfig(config)
               if err!=nil{
                    return nil,err
               }
               return newOpenStack(cfg)
          })
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/cloudprovider/providers/ovirt**

src/k8s.io/kubernetes/pkg/cloudprovider/providers/ovirt/ovirt.go

     func init(){
          cloudprovider.RegisterCloudProvider(ProviderName,
               func(config io.Reader)(cloudprovider.Interface, error){
                    return newOVirtCloud(config)
               })
     }

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/vmware/photon-controller-go-sdk/photon**

k8s.io/kubernetes/vendor/github.com/vmware/photon-controller-go-sdk/photon/restclient.go

     var quoteEscaper = strings.NewReplacer("\\", "\\\\", `"`, "\\\"")

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/cloudprovider/providers/photon**

src/k8s.io/kubernetes/pkg/cloudprovider/providers/photon/photon.go

     func init(){
          cloudprovider.RegisterCloudProvider(ProviderName, func(config io.Reader)(cloudprovider.Interface, error){
              cfg,err:=readConfig(config)
               if err!=nil{
                    glog.Errorf("Photon Cloud Provider: failed to read in cloud provider config file. Error [%v]",err)
                    return nil,err
               } 
               return newPCCloud(cfg)
          })
     }

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/rackspace/gophercloud**

k8s.io/kubernetes/vendor/github.com/rackspace/gophercloud/endpoint_search.go

     var(
          ErrServiceNotFound = errors.New("No suitable service could be found in the service catalog.")
          ErrEndpointNotFound = errors.New("No suitable endpoint could be found in the service catalog.")
     )

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/rackspace/gophercloud/pagination**

k8s.io/kubernetes/vendor/github.com/rackspace/gophercloud/pagination/pager.go

     var(
          ErrPageNotAvailable = errors.New("The requested page does not exist.")
     )

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/rackspace/gophercloud/openstack/compute/v2/flavors**

k8s.io/kubernetes/vendor/github.com/rackspace/gophercloud/openstack/compute/v2/flavors/results.go

     var ErrCannotInterpet = errors.New("Unable to interpret a response body.")

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/rackspace/gophercloud/openstack/identity/v2/tokens**

k8s.io/kubernetes/vendor/github.com/rackspace/gophercloud/openstack/identity/v2/tokens/errors.go

     var(
          ErrUserIDProvided = unacceptAttributeErr("UserID")
          ...
     )

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/rackspace/gophercloud/openstack/identity/v3/tokens**

k8s.io/kubernetes/vendor/github.com/rackspace/gophercloud/openstack/identity/v3/tokens/errors.go

     var(
          ErrAPIKeyPrivided = unacceptedAttributeErr("APIKey")
          ...
     )

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/rackspace/gophercloud/openstack/utils**

k8s.io/kubernetes/vendor/github.com/rackspace/gophercloud/openstack/utils/choose_version.go

     var goodStatus = map[string]bool{
          "current": true,
          "supported": true,
          "stable": true,
     }

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/rackspace/gophercloud/openstack**

k8s.io/kubernetes/vendor/github.com/rackspace/gophercloud/openstack/auth_env.go

     var(
          ErrNoAuthURL = fmt.Errof("Environment variable OS_AUTH_URL needs to be set.")
          ...
     )

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/rackspace/gophercloud/rackspace/identity/v2/tokens**

k8s.io/kubernetes/vendor/github.com/rackspace/gophercloud/rackspace/identity/v2/tokens/delegate.go

     var(
          ErrPasswordProvided = errors.New("Please provide either a password or an API key.")
     )

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/rackspace/gophercloud/rackspace**

k8s.io/kubernetes/vendor/github.com/rackspace/gophercloud/rackspace/auth_env.go

     var(
          ErrNoAuthURL = fmt.Errorf("Environment variable RS_AUTH_URL or OS_AUTH_URL need to be set.")
          ...
     )

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/rackspace/gophercloud/openstack/compute/v2/extensions/diskconfig**

k8s.io/kubernetes/vendor/github.com/rackspace/gophercloud/openstack/compute/v2/extensions/diskconfig/requests.go

     var ErrInvalidDiskConfig = errors.New("DiskConfig must be either diskconfig.Auto or diskconfig.Manual.")

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/cloudprovider/providers/rackspace**

src/k8s.io/kubernetes/pkg/cloudprovider/providers/rackspace/rackspace.go

     var ErrNotFound = errors.New("Failed to find object")
     var ErrMultipleResults = errors.New("Multiple results where only one expected")
     var ErrNoAddressFound = errors.New("No address found for host")
     var ErrAttrNotFound = errors.New("Expected attribute not found")

     func init(){
          cloudprovider.RegisterCloudProvider(ProviderName, func(config io.Reader) (cloudprovider.Interface,error){
               ...
          })
     }

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/vmware/govmomi/vim25/types**

k8s.io/kubernetes/vendor/github.com/vmware/govmomi/vim25/types/enum.go

     func init(){
          t["ActionParameter"] = reflect.TypeOf((*ActionParameter)(nil)).Elem()
     }
     func init(){
          t["ActionType"] = reflect.TypeOf((*ActionType)(nil)).Elem()
     }
     func init(){
          t["AffinityType"] = reflect.TypeOf((*AffinityType)(nil)).Elem()
     }
     func init(){
          t["AgentInstallFailedReason"] = reflect.TypeOf((*AgentInstallFailedReason)(nil)).Elem()
     }
     func init(){
          t["ArrayUpdateOperation"] = reflect.TypeOf((*ArrayUpdateOperation)(nil)).Elem()
     }
     func init(){
          t["AutoStartAction"] = reflect.TypeOf((*AutoStartAction)(nil)).Elem()
     }
     func init(){
          t["AutoStartWaitHeartbeatSetting"] = reflect.TypeOf((*AutoStartWaitHeartbeatSetting)(nil)).Elem()
     }
     func init(){
          t["BaseConfigInfoDiskFileBackingInfoProvisioningType"] = reflect.TypeOf((*BaseConfigInfoDiskFileBackingInfoProvisioningType)(nil)).Elem()
     }
     func init(){
          t["BatchResultREsult"] = reflect.TypeOf((*BatchResultResult)(nil)).Elem()
     }
     func init(){
          t["CannotEnableVmcpForClusterReason"] = reflect.TypeOf((*CannotEnableVmcpForClusterReason)(nil)).Elem()
     }
     func init(){
          t["CannotMoveFaultToleranceVmMoveType"] = reflect.TypeOf((*CannotMoveFaultToleranceVmMoveType)(nil)).Elem()
     }
     func init(){
          t["CannotPowerOffVmInClusterOperation"] = reflect.TypeOf((*CannotPowerOffVmInClusterOperation)(nil)).Elem()
     }
     func init(){
          t["CannotUseNetworkReason"] = reflect.TypeOf((*CannotUseNetworkReason)(nil)).Elem()
     }
     func init(){
          t["CheckTestType"] = reflect.TypeOf((*CheckTestType)(nil)).Elem()
     }
     func init(){
          t["ClusterDasAamNodeStateDasState"] = reflect.TypeOf((*ClusterDasAamNodeStateDasState)(nil)).Elem()
     }
     func init(){
          t["ClusterDasConfigInfoHBDatastoreCandidate"] = reflect.TypeOf((*ClusterDasConfigInfoHBDatastoreCandidate)(nil)).Elem()
     }
     func init(){
          t["ClusterDasConfigInfoServiceState"] = reflect.TypeOf((*ClusterDasConfigInfoServiceState)(nil)).Elem()
     }
     func init(){
          t["ClusterDasConfigInfoVmMonitoringState"] = reflect.TypeOf((*ClusterDasConfigInfoVmMonitoringState)(nil)).Elem()
     }
     func init(){
          t["ClusterDasFdmAvailabilityState"] = reflect.TypeOf((*ClusterDasFdmAvailability)(nil)).Elem()
     }
     func init(){
          t["ClusterDasVmSettingsIsolationResponse"] = reflect.TypeOf((*ClusterDasVmSettingsIsolationResponse)(nil)).Elem()
     }
     func init(){
          t["ClusterDasVmSettingsRestartPriority"] = reflect.TypeOf((*ClusterDasVmSettingsRestartPriority)(nil)).Elem()
     }
     func init(){
          t["ClusterHostInfraUpdateHaModeActionOperationType"] = reflect.TypeOf((*ClusterHostInfraUpdateHaModeActionOperationType)(nil)).Elem()
     }
     func init(){
          t["ClusterInfraUpdateHaConfigInfoBehaviorType"] = reflect.TypeOf((*ClusterInfraUpdateHaConfigInfoBehavior)(nil)).Elem()
     }
     func init(){
          t["ClusterInfraUpdateHaConfigInfoRemediationType"] = reflect.TypeOf((*ClusterInfraUpdateHaConfigInfoRemediationType)(nil)).Elem()
     }
     func init(){
          t["ClusterPowerOnVmOption"] = reflect.TypeOf((*ClusterPowerOnVmOption)(nil)).Elem()
     }
     func init(){
          t["ClusterProfileServiceType"] = reflect.TypeOf((*ClusterProfileServiceType)(nil)).Elem()
     }
     func init(){
          t["ClusterVmComponentProtectionSettingsStorageVmReaction"] = reflect.TypeOf((*ClusterVmComponentProtectionSettingsStorageVmReaction)(nil)).Elem()
     }
     func init(){
          t["ClusterVmComponentProtectionSettingsVmReactionOnAPDCleared"] = reflect.TypeOf((*ClusterVmComponentProtectionSettingsVmReactionOnAPDCleared)(nil)).Elem()
     }
     func init(){
          t["ClusterVmReadinessReadyCondition"] = reflect.TypeOf((*ClusterVmReadinessReadyCondition)(nil)).Elem()
     }
     func init(){
          t["ComplianceResultStatus"] = reflect.TypeOf((*ComplianceResultStatus)(nil)).Elem()
     }
     func init(){
          t["ComputeResourceHostSPBMLicenseInfoHostSPBMLicenseState"] = reflect.TypeOf((*ComputeResourceHostSPBMLicenseInfoHostSPBMLicenseState)(nil)).Elem()
     }
     func init(){
          t["ConfigSpecOperation"] = reflect.TypeOf((*ConfigSpecOperation)(nil)).Elem()
     }
     func init(){
          t["CustomizationLicenseDateMode"] = reflect.TypeOf((*CustomizationLicenseDateMode)(nil)).Elem()
     }
     func init(){
          t["CustomizationNetBIOSMode"] = reflect.TypeOf((*CustomizationNetBIOSMode)(nil)).Elem()
     }
     func init(){
          t["CustomizationSysprepRebootOption"] = reflect.TypeOf((*CustomizationSysprepRebootOption)(nil)).Elem()
     }
     func init(){
          t["DVPortStatusVmDirectPathGen2InactiveReasonNetwork"] = reflect.TypeOf((*DVPortStatusVmDirectPathGen2InactiveReasonNetwork)(nil)).Elem()
     }
     func init(){
          t["DVPortStatusVmDirectPathGen2InactiveReasonOther"] = reflect.TypeOf((*DVPortStatusVmDirectPathGen2InactiveReasonOther)(nil)).Elem()
     }
     func init(){
          t["DasConfigFaultDasConfigFaultReason"] = reflect.TypeOf((*DasConfigFaultDasConfigFaultReason)(nil)).Elem()
     }
     func init(){
          t["DasVmPriority"] = reflect.TypeOf((*DasVmPriority)(nil)).Elem()
     }
     func init(){
          t["DatastoreAccessible"] = reflect.TypeOf((*DatastoreAccessible)(nil)).Elem()
     }
     func init(){
          t["DatastoreSummaryMaintenanceModeState"] = reflect.TypeOf((*DatastoreSummaryMaintenanceModeState)(nil)).Elem()
     }
     func init(){
          t["DayOfWeek"] = reflect.TypeOf((*DayOfWeek)(nil)).Elem()
     }
     func init(){
          t["DeviceNotSupportedReason"] = reflect.TypeOf((*DeviceNotSupportedReason)(nil)).Elem()
     }
     func init(){
          t["DiagnosticManagerLogCreator"] = reflect.TypeOf((*DiagnosticManagerLogCreator)(nil)).Elem()
     }
     func init(){
          t["DiagnosticPartitionStorageType"] = reflect.TypeOf((*DiagnosticPartitionStorageType)(nil)).Elem()
     }
     func init(){
          t["DiagnosticPartitionType"] = reflect.TypeOf((*DiagnosticPartitionType)(nil)).Elem()
     }
     func init(){
          t["DisallowedChangeByServiceDisallowedChange"] = reflect.TypeOf((*DisallowedChangeByServiceDisallowedChange)(nil)).Elem()
     }
     func init(){
          t["DistributedVirtualPortgroupMetaTagName"] = reflect.TypeOf((*DistributedVirtualPortgroupMetaTagName)(nil)).Elem()
     }
     func init(){
          t["DistributedVirtualPortgroupPortgroupType"] = reflect.TypeOf((*DistributedVirtualPortgroupPortgroupType)(nil)).Elem()
     }
     func init(){
          t["DistributedVirtualSwitchHostInfrastructureTrafficClass"] = reflect.TypeOf((*DistributedVirtualSwitchHostInfrastructure)(nil)).Elem()
     }
     func init(){
          t["DistributedVirtualSwitchHostMemeberHostComponentState"] = reflect.TypeOf((*DistributedVirtualSwitchHostMemberHostComponentState)(nil)).Elem()
     }
     func init(){
          t["DistributedVirtualSwitchNetworkResourceControlVersion"] = reflect.TypeOf((*DistributedVirtualSwitchNetworkResourceControlVersion)(nil)).Elem()
     }
     func init(){
          t["DistributedVirtualSwitchNicTeamingPolicyMode"] = reflect.TypeOf((*DistributedVirtualSwitchNicTeamingPolicyMode)(nil)).Elem()
     }
     func init(){
          t["DistributedVirtualSwitchPortConnecteeConnecteeType"] = reflect.TypeOf((*DistributedVirtualSwitchPortConnecteeConnecteeType)(nil)).Elem()
     }
     func init(){
          t["DistributedVirtualSwitchProductSpecOperationType"] = reflect.TypeOf((*DistributedVirtualSwitchProductSpecOperation)(nil)).Elem()
     }
     func init(){
          t["DpmBehavior"] = reflect.TypeOf((*DpmBehavior)(nil)).Elem()
     }
     func init(){
          t["DrsBehavior"] = reflect.TypeOf((*DrsBehavior)(nil)).Elem()
     }
     func init(){
          t["DrsInjectorWorkloadCorrelationState"] = reflect.TypeOf((*DrsInjectorWorkloadCorrelation)(nil)).Elem()
     }
     func init(){
          t["DrsRecommendationReasonCode"] = reflect.TypeOf((*DrsRecommendationReasonCode)(nil)).Elem()
     }
     func init(){
          t["DvsEventPortBlockState"] = reflect.TypeOf((*DvsEventPortBlockState)(nil)).Elem()
     }
     func init(){
          t["DvsFilterOnFailture"] = reflect.TypeOf((*DvsFilterOnFailture)(nil)).Elem()
     }
     func init(){
          t["DvsNetworkRuleDirectionType"] = reflect.TypeOf((*DvsNetworkRuleDirection)(nil)).Elem()
     }
     func init(){
          t["EntityImportType"] = reflect.TypeOf((*EntityImportType)(nil)).Elem()
     }
     func init(){
          t["EntityType"] = reflect.TypeOf((*EntityType)(nil)).Elem()
     }
     func init(){
          t["EventAlarmExpressionComparisonOperator"] = reflect.TypeOf((*EventAlarmExpressionComparisonOperator)(nil)).Elem()
     }
     func init(){
          t["EventCategory"] = reflect.TypeOf((*EventGategory)(nil)).Elem()
     }
     func init(){
          t["EventEventSeverity"] = reflect.TypeOf((*EventEventSeverity)(nil)).Elem()
     }
     func init(){
          t["EventFilterSpecRecursionOption"] = reflect.TypeOf((*EventFilterSpecRecursionOption)(nil)).Elem()
     }
     func init(){
          t["FibreChannelPortType"] = reflect.TypeOf((*FibreChannelPortType)(nil)).Elem()
     }
     func init(){
          t["FileSystemMountInfoVStorageSupportStatus"] = reflect.TypeOf((*FileSystemMountInfoVStorageSupportStatus)(nil)).Elem()
     }
     func init(){
          t["FtIssuesOnHostHostSelectionType"] = reflect.TypeOf((*FtIssuesOnHostHostSelectionType)(nil)).Elem()
     }
     func init(){
          t["GuestFileType"] = reflect.TypeOf((*GuestFileType)(nil)).Elem()
     }
     func init(){
          t["GuestInfoAppStateType"] = reflect.TypeOf((*GuestInfoAppStateType)(nil)).Elem()
     }
     func init(){
          t["GuestOsDescriptorFirmwareType"] = reflect.TypeOf((*GuestOsDescriptorFirmwareType)(nil)).Elem()
     }
     func init(){
          t["GuestOsDescriptorSupportLevel"] = reflect.TypeOf((*GuestOsDescriptorSupportLevel)(nil)).Elem()
     }
     func init(){
          t["GuestRegKeyWowSpec"] = reflect.TypeOf((*GuestRegKeyWowSpec)(nil)).Elem()
     }
     func init(){
          t["HealthUpdateInfoComponentType"] = reflect.TypeOf((*HealthUpdateInfoComponentType)(nil)).Elem()
     }
     func init(){
          t["HostAccessMode"] = reflect.TypeOf((*HostAccessMode)(nil)).Elem()
     }
     func init(){
          t["HostActiveDirectoryAuthenticationCertificateDigest"] = reflect.TypeOf((*HostActiveDirectoryAuthenticationCertificateDigest)(nil)).Elem()
     }
     func init(){
          t["HostActiveDirectoryInfoDomainMembershipStatus"] = reflect.TypeOf((*HostActiveDirectoryInfoDomainMembershipStatus)(nil)).Elem()
     }
     func init(){
          t["HostCapabilityFtUnsupportedReason"] = reflect.TypeOf((*HostCapabilityFtUnsupportedReason)(nil)).Elem()
     }
     func init(){
          t["HostCapabilityVmDirectPathGen2UnsupportedReason"] = reflect.TypeOf((*HostCapabilityVmDirectPathGen2UnsupportedReason)(nil)).Elem()
     }
     func init(){
          t["HostCertificateManagerCertificateInfoCertificateStatus"] = reflect.TypeOf((*HostCertificateManagerCertificateInfoCertificateStatus)(nil)).Elem()
     }
     func init(){
          t["HostConfigChangeMode"] = reflect.TypeOf((*HostConfigChangeMode)(nil)).Elem()
     }
     func init(){
          t["HostConfigChangeOperation"] = reflect.TypeOf((*HostConfigChangeOperation)(nil)).Elem()
     }
     func init(){
          t["HostCpuPackageVendor"] = reflect.TypeOf((*HostCpuPackageVendor)(nil)).Elem()
     }
     func init(){
          t["HostCpuPowerManagementInfoPolicyType"] = reflect.TypeOf((*HostCpuPowerManagementInfoPolicyType)(nil)).Elem()
     }
     func init(){
          t["HostCryptoState"] = reflect.TypeOf((*HostCryptoState)(nil)).Elem()
     }
     func init(){
          t["HostDasErrorEventHostDasErrorReason"] = reflect.TypeOf((*HostDasErrorEventHostDasErrorReason)(nil)).Elem()
     }
     func init(){
          t["HostDigestInfoDigestMethodType"] = reflect.TypeOf((*HostDigestInfoDigestMethodType)(nil)).Elem()
     }
     func init(){
          t["HostDisconnectedEventReasonCode"] = reflect.TypeOf((*HostDisconnectedEventReasonCode)(nil)).Elem()
     }
     func init(){
          t["HostDiskPartitionInfoPartitionFormat"] = reflect.TypeOf((*HostDiskPartitionInfoPartitionFormat)(nil)).Elem()
     }
     func init(){
          t["HostDiskPartitionInfoType"] = reflect.TypeOf((*HostDiskPartitionInfoType)(nil)).Elem()
     }
     func init(){
          t["HostFeatureVersionKey"] = reflect.TypeOf((*HostFeatureVersionKey)(nil)).Elem()
     }
     func init(){
          t["HostFileSystemVolumeFileSystemType"] = reflect.TypeOf((*HostFileSystemVolumeFileSystemType)(nil)).Elem()
     }
     func init(){
          t["HostFirewallRuleDirection"] = reflect.TypeOf((*HostFirewallRuleDirection)(nil)).Elem()
     }
     func init(){
          t["HostFirewallRulePortType"] = reflect.TypeOf((*HostFirewallRulePortType)(nil)).Elem()
     }
     func init(){
          t["HostFirewallRuleProtocol"] = reflect.TypeOf((*HostFirewallRuleProtocol)(nil)).Elem()
     }
     func init(){
          t["HostGraphicsConfigGraphicsType"] = reflect.TypeOf((*HostGraphicsConfigGraphicsType)(nil)).Elem()
     }
     func init(){
          t["HostGraphicsConfigSharedPassthruAssignmentPolicy"] = reflect.TypeOf((*HostGraphicsConfigSharedPassthruAssignmentPolicy)(nil)).Elem()
     }
     func init(){
          t["HostGraphicsInfoGraphicsType"] = reflect.TypeOf((*HostGraphicsGraphicsType)(nil)).Elem()
     }
     func init(){
          t["HostHardwareElementStatus"] = reflect.TypeOf((*HostHardwareElementStatus)(nil)).Elem()
     }
     func init(){
          t["HostHasComponentFailureHostComponentType"] = reflect.TypeOf((*HostHasComponentFailureHostComponentType)(nil)).Elem()
     }
     func init(){
          t["HostImageAcceptanceLevel"] = reflect.TypeOf((*HostImageAcceptanceLevel)(nil)).Elem()
     }
     func init(){
          t["HostIncompatibleForFaultToleranceReason"] = reflect.TypeOf((*HostIncompatibleForFaultToleranceReason)(nil)).Elem()
     }
     func init(){
          t["HostIncompatibleForRecordReplayReason"] = reflect.TypeOf((*HostIncompatibleForRecordReplayReason)(nil)).Elem()
     }
     func init(){
          t["HostInternetScsiHbaChapAuthenticationType"] = reflect.TypeOf((*HostInternetScsiHbaChapAuthentication)(nil)).Elem()
     }
     func init(){
          t["HostInternetScsiHbaDigestType"] = reflect.TypeOf((*HostInternetScsiHbaDigestType)(nil)).Elem()
     }
     func init(){
          t["HostInternetScsiHbaIscsiIpv6AddressAddressConfigurationType"] = reflect.TypeOf((*HostInternetScsiHbaIscsiIpv6AddressAddressConfigurationType )(nil)).Elem()
     }
     func init(){
          t["HostInternetScsiHbaIscsiIpv6AddressIPv6AddressOperation"] = reflect.TypeOf((*HostInternetScsiHbaIscsiIpv6AddressIPv6AddressOperation )(nil)).Elem()
     }
     func init(){
          t["HostInternetScsiHbaNetworkBindingSupportType"] = reflect.TypeOf((*HostInternetScsiHbaNetworkBindingSupportType )(nil)).Elem()
     }
     func init(){
          t["HostInternetScsiHbaStaticTargetTargetDiscoveryMethod"] = reflect.TypeOf((*HostInternetScsiHbaStaticTargetTargetDiscoveryMethod )(nil)).Elem()
     }
     func init(){
          t["HostIpConfigIpV6AddressConfigType"] = reflect.TypeOf((*HostIpConfigIpV6AddressConfigType)(nil)).Elem()
     }
     func init(){
          t["HostIpConfigIpV6AddressStatus"] = reflect.TypeOf((*HostIpConfigIpV6AddressStatus)(nil)).Elem()
     }
     func init(){
          t["HostLicensableResourceKey"] = reflect.TypeOf((*HostLicensableResourceKey)(nil)).Elem()
     }
     func init(){
          t["HostLockdownMode"] = reflect.TypeOf((*HostLockdownMode)(nil)).Elem()
     }
     func init(){
          t["HostLowLevelProvisioningManagerFileType"] = reflect.TypeOf((*HostLowLevelProvisioningManagerFileType)(nil)).Elem()
     }
     func init(){
          t["HostLowLevelProvisioningManagerReloadTarget"] = reflect.TypeOf((*HostLowLevelProvisioningManagerReloadTarget)(nil)).Elem()
     }
     func init(){
          t["HostMountInfoInaccessibleReason"] = reflect.TypeOf((*HostMountInfoInaccessibleReason)(nil)).Elem()
     }
     func init(){
          t["HostMountMode"] = reflect.TypeOf((*HostMountMode)(nil)).Elem()
     }
     func init(){
          t["HostNasVolumeSecurityType"] = reflect.TypeOf((*HostNasVolumeSecurityType)(nil)).Elem()
     }
     func init(){
          t["HostNetStackInstanceCongestionControlAlgorithmType"] = reflect.TypeOf((*HostNetStackInstanceCongestionControlAlgorithmType)(nil)).Elem()
     }
     func init(){
          t["HostNetStackInstanceSystemStackKey"] = reflect.TypeOf((*HostNetStackInstanceSystemStackKey)(nil)).Elem()
     }
     func init(){
          t["HostNumericSensorHealthState"] = reflect.TypeOf((*HostNumericSensorHealthState )(nil)).Elem()
     }
     func init(){
          t["HostNumericSensorType"] = reflect.TypeOf((*HostNumericSensorType )(nil)).Elem()
     }
     func init(){
          t["HostOpaqueSwitchOpaqueSwitchState"] = reflect.TypeOf((*HostOpaqueSwitchOpaqueSwitchState )(nil)).Elem()
     }
     ...
 
     func init(){
          t["WillLoseHAProtectionResolution"] = reflect.TypeOf((*WillLoseHAProtectionResolution )(nil)).Elem()
     }
   

k8s.io/kubernetes/vendor/github.com/vmware/govmomi/vim25/types/if.go

     func init(){
          t["BaseAction"] = reflect.TypeOf((*BaseAction)(nil)).Elem()
     }
     ...
     func init(){
          t["BaseVslmMigrateSpec"] = reflect.TypeOf((*BaseVslmMigrateSpec)(nil)).Elem()
     }

k8s.io/kubernetes/vendor/github.com/vmware/govmomi/vim25/types/internal.go

     func init(){
          t["DynamicTypeEnumTypeInfo"] = reflect.TypeOf((*DynamicTypeEnumTypeInfo)(nil)).Elem()
     }
     ...
     func init(){
          t["RetrieveManagedMethodExecuter"] = reflect.TypeOf((*RetrieveManagerMethodExecuter)(nil)).Elem()
     }

k8s.io/kubernetes/vendor/github.com/vmware/govmomi/vim25/types/types.go

     func init(){
          t["AbdicateDomOwnership"] = reflect.TypeOf((*AbdicateDomOwnership )(nil)).Elem()
     }
     ...
     func init(){
          t["ZeroFillVirtualDisk_Task"] = reflect.TypeOf((*ZeroFillVirtualDisk_Task )(nil)).Elem()
     }

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/vmware/govmomi/vim25/xml**

k8s.io/kubernetes/vendor/github.com/vmware/govmomi/vim25/xml/extras.go

     var stringToTypeMap = map[string]reflect.Type{
          "xsd:boolean": reflect.TypeOf((*bool)(nil)).Elem(),
          ...
     }

k8s.io/kubernetes/vendor/github.com/vmware/govmomi/vim25/xml/marshal.go

     var(
          marshalerType = reflect.TypeOf((*Marshaler)(nil)).Elem()
          marshalerAttrType = reflect.TypeOf((*MarshalerAttr)(nil)).Elem()
          textMarshalerType = reflect.TypeOf((*encoding.TextMarshaler)(nil)).Elem()
     )

k8s.io/kubernetes/vendor/github.com/vmware/govmomi/vim25/xml/read.go

     var(
          unmarshalerType = reflect.TypeOf((*Unmarshaler)(nil)).Elem()
          unmarshalerAttrType = reflect.TypeOf((*UnmarshalerAttr)(nil)).Elem()
          textUnmarshalerType = reflect.TypeOf((*encoding.TextUnmarshaler)(nil)).Elem()
     )

k8s.io/kubernetes/vendor/github.com/vmware/govmomi/vim25/xml/typeinfo.go

     var tinfoMap = make(map[reflect.Type]*typeInfo)
     var tinfoLock sync.RWMutex
     var nameType = reflect.TypeOf(Name{})

k8s.io/kubernetes/vendor/github.com/vmware/govmomi/vim25/xml/xml.go

     var errRawToken = errors.New("xml: cannot use RawToken from UnmarshalXML method")

     var entity = map[string]int{
          "lt": '<',
          "gt": '>',
          "amp": '&',
          "apos": '\'',
          "quot": '"',
     }
     
     var HTMLEntity = htmlEntity
     var htmlEntity = map[string]string{
          "nbsp": "\u00A0",
          
     }

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/vmware/govmomi/vim25/soap**

k8s.io/kubernetes/vendor/github.com/vmware/govmomi/vim25/soap/client.go

     var schemeMatch = regexp.MustCompile(`^\w+://`)

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/vmware/govmomi/vim25/mo**

k8s.io/kubernetes/vendor/github.com/vmware/govmomi/vim25/mo/registry.go

     var t = map[string]reflect.Type{}

k8s.io/kubernetes/vendor/github.com/vmware/govmomi/vim25/mo/type_info.go

     var typeInfoLock sync.RWMutex
     var typeInfoMap = make(map[string]*typeInfo)

     var managedObjectRefType = reflect.TypeOf((*types.ManagedObjectReference)(nil)).Elem()

     var arrayOfRegexp = regexp.MustCompile("ArrayOf(.*)$")

k8s.io/kubernetes/vendor/github.com/vmware/govmomi/vim25/mo/mo.go

     func init(){
          t["Alarm"] = reflect.TypeOf((*Alarm)(nil)).Elem()
     }
     ..
     func init(){
          t["VsanUpgradeSystem"] = reflect.TypeOf((*VsanUpgradeSystem )(nil)).Elem()
     }

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/vmware/govmomi/session**

k8s.io/kubernetes/vendor/github.com/vmware/govmomi/session/manager.go

     var Locale = os.Getenv("GOVMOMI_LOCATE")

     func init(){
          if Locale == "_"{
               Locale = ""
          }else if Locale ==""{
               Locale = "en_US"
          }
     }

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/vmware/govmomi/object**

k8s.io/kubernetes/vendor/github.com/vmware/govmomi/object/common.go

     var(
          ErrNotSupported = errors.New("product/version specific feature not supported by target")
     )

k8s.io/kubernetes/vendor/github.com/vmware/govmomi/object/custom_fields_manager.go

     var(
          ErrKeyNameNotFound = errors.New("key name not found")
     )

k8s.io/kubernetes/vendor/github.com/vmware/govmomi/object/virtual_device_list.go

     var bootableDevices = map[string]func(device types.BaseVirtualDevice) types.BaseVirtualMachineBootOptionsBootableDevice{
          DeviceTypeCdrom: func(types.BaseVirtualDevice) types.BaseVirtualMachineBootOptionsBootableDevice{
               return &types.VirtualMachineBootOptionsBootableCdromDevice{}
          },
          ...
     }
     var deviceNameRegexp = regexp.MustCompile(`(?:Virtual)?(?:Machine)?(\w+?)(?:Card|Device|Controller)?$`)

k8s.io/kubernetes/vendor/github.com/vmware/govmomi/object/virtual_disk_manager_internal.go

     func init(){
          types.Add("ArrayOfVirtualDiskInfo", reflect.TypeOf((*arrayOfVirtualDiskInfo)(nil)).Elem())
          types.Add("VirtualDiskInfo", reflect.TypeOf(((VirtualDiskInfo)(nil)).Elem())
     }

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/vmware/govmomi/find**

k8s.io/kubernetes/vendor/github.com/vmware/govmomi/find/recurser.go

     var listMode = os.Getenv("GOVMOMI_FINDER_LIST_MODE") == "true"

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/vmware/govmomi/pbm/types**

k8s.io/kubernetes/vendor/github.com/vmware/govmomi/pbm/types/enum.go

     func init(){
          types.Add("pbm:PbmBuiltinGenericType", reflect.TypeOf((*PbmBuiltinGenericType)(nil)).Elem())
     }
     func init(){
          types.Add("pbm:PbmBuiltinType", reflect.TypeOf((*PbmBuiltinType)(nil)).Elem())
     }
     ...
     func init(){
          types.Add("pbm:PbmVvolType", reflect.TypeOf((*PbmVvolType)(nil)).Elem())
     }

k8s.io/kubernetes/vendor/github.com/vmware/govmomi/pbm/types/if.go

     func init(){
          types.Add("BasePbmCapabilityConstraints", reflect.TypeOf((*BasePbmCapabilityConstraints )(nil)).Elem())
     }
     ...
     func init(){
          types.Add("BasePbmPropertyMismatchFault", reflect.TypeOf((*BasePbmPropertyMismatchFault )(nil)).Elem())
     }

k8s.io/kubernetes/vendor/github.com/vmware/govmomi/pbm/types/types.go

     func init(){
          types.Add("pbm:ArrayOfPbmCapabilityConstraintInstance", reflect.TypeOf((*ArrayOfPbmCapabilityConstraintInstance )(nil)).Elem())
     }
     ...
     func init(){
          types.Add("pbm:PbmVaioDataServiceInfo", reflect.TypeOf((*PbmVaioDataServiceInfo )(nil)).Elem())
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/cloudprovider/provider/vsphere**

src/k8s.io/kubernetes/pkg/cloudprovider/provider/vsphere/vsphere.go

     var supportedSCSIControllerType = []string{strings.ToLower(LSTLogicSASControllerType), PVSCSIControllerType}
     var diskFormatValidType = map[string]string{
          ThinDiskType: ThinDiskType,
          strings.ToLower(EagerZeroedThickDiskType): EagerZeroedThickDiskType,
          strings.ToLower(ZeroedThickDiskType): PreallocatedDiskType,
     }
     var DiskformatValidOptions = generateDiskFormatValidOptions()
     var cleanUpRoutineInitialized = false
     var ErrNoDiskUUIDFound = errors.New(NoDiskUUIDFoundErrMsg)
     var ErrNoDiskIDFound = errors.New(DiskNotFoundErrMsg)
     var ErrNoDevicesFound = errors.New(NoDevicesFoundErrMsg)
     var ErrNonSupportedControllerType = errors.New(NonSupportedControllerTypeErrMsg)
     var ErrFileAlreadyExist = errors.New(FileAlreadyExistErrMsg)
     var clientLock sync.Mutex
     var cleanUpRoutineInitLock sync.Mutex
     var cleanUpDummyVMLock sync.RWMutex

src/k8s.io/kubernetes/pkg/cloudprovider/provider/vsphere/vsphere_metrics.go

     var vsphereApiMetric = prometheus.NewHistogramVec(
          prometheus.HistogramOpts{
               Name: "cloudprovider_vsphere_api_request_duration_seconds",
               Help: "Latency of vsphere api call",
          },
          []string{"request"},
     )
     ...

src/k8s.io/kubernetes/pkg/cloudprovider/provider/vsphere/vsphere.go

     func init(){
          registerMetrics()
          cloudprovider.RegisterCloudProvider(ProviderName, func(config io.Reader) (cloudprovider.Interface, error){
               cfg, err:=readConfig(config)
               if err!=nil{
                    return nil, err
               }
               return newVSphere(cfg)
          }
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/plugin/pkg/scheduler/api**

src/k8s.io/kubernetes/plugin/pkg/scheduler/api/register.go

     var Scheme = runtime.NewScheme()

     var (
          SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)
          AddToScheme = SchemeBuilder.AddToScheme
     )
     func init(){
          if err:=addKnownTypes(Scheme); err!=nil{
               panic(err)
          }
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/plugin/pkg/admission/defaulttolerationseconds**

src/k8s.io/kubernetes/plugin/pkg/admission/defaulttolerationseconds/admission.go

     var(
          defaultNotReadyTolerationSeconds = flag.Int64("default-not-ready-toleration-seconds", 300,
               "Indicates the tolerationSeconds of the toleration for notReady: NoExecute"+
                    " that is added by default to every pod that does not already have such a toleration.")
          defaultUnreachableTolerationSeconds = flag.Int64("default-unreachable-toleration-seconds", 300,
               "Indicates the tolerationSeconds of the toleration for unreachable:NoExecute"+
                    " that is added by default to every pod that does not already have such a toleration.")
     )

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/imagepolicy**

src/k8s.io/kubernetes/pkg/apis/imagepolicy/register.go

     var(
          SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)
          AddToScheme = SchemeBuilder.AddToScheme
     )

src/k8s.io/kubernetes/pkg/apis/imagepolicy/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/imagepolicy/v1alpha1**

src/k8s.io/kubernetes/pkg/apis/imagepolicy/v1alpha1/generated.pb.go

     var(
          ErrInvalidLengthGenerated = fmt.Errorf("proto: negative length found during unmarshaling")
          ErrIntOverflowGenerated = fmt.Errorf("proto: integer overflow")
     )

src/k8s.io/kubernetes/pkg/apis/imagepolicy/v1alpha1/register.go

     var(
          SchemeBuilder runtime.SchemeBuilder
          localSchemeBuilder = &SchemeBuilder
          AddToScheme = localSchemeBuilder.AddToScheme
     )

src/k8s.io/kubernetes/pkg/apis/imagepolicy/v1alpha1/types.generated.go

     var(
          codecSelferBitsize1234 = uint8(reflect.TypeOf(uint(0)).Bits())
          codecSelferOnlyMapOrArrayEncodeToStructErr1234 = errors.New(`only encoded map or array can be decoded into a struct`)
     )

src/k8s.io/kubernetes/pkg/apis/imagepolicy/v1alpha1/types_swagger_doc_generated.go

     var map_ImageReview = map[string]string{
          "":"",
          ...
     }
     ...

src/k8s.io/kubernetes/pkg/apis/imagepolicy/v1alpha1/generated.pb.go

     func init(){
          proto.RegisterType((*ImageReview)(nil), "k8s.io.kubernetes.pkg.apis.imagepolicy.v1alpha1.ImageReview")
          ...
     }
     func init(){
          proto.RegisterFile("k8s.io/kubernetes/pkg/apis/imagepolicy/v1alpha1/generated.proto", fileDescriptorGenerated)
     }

src/k8s.io/kubernetes/pkg/apis/imagepolicy/v1alpha1/register.go

     func init(){
          localSchemeBuilder.Register(addKnownTypes, RegisterDefaults)
     }

src/k8s.io/kubernetes/pkg/apis/imagepolicy/v1alpha1/types.generated.go

     func init(){
          if codec1978.GenVersion != 5{
               _,file,_,_:=runtime.Caller(0)
               err:=fmt.Errorf("codecgen version mismatch: current: %v, need %v. Re-generate file: %v",
                    5, codec1978.GenVersion,file)
               panic(err)
          }
          if false{
               var v0 pkg1_v1.TypeMeta
               var v1 pkg2_types.UID
               var v2 time.Time
               _,_,_ = v0, v1, v2
          }
     }

src/k8s.io/kubernetes/pkg/apis/imagepolicy/v1alpha1/zz_generated.conversion.go

     func init(){
          SchemeBuilder.Register(RegisterConversions)
     }

src/k8s.io/kubernetes/pkg/apis/imagepolicy/v1alpha1/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/imagepolicy/install**

src/k8s.io/kubernetes/pkg/apis/imagepolicy/install/install.go

     func init(){
          Install(api.GroupFactoryRegistry, api.Registry, api.Scheme)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/plugin/pkg/admission/imagepolicy**

src/k8s.io/kubernetes/plugin/pkg/admission/imagepolicy/admission.go

     var(
          groupVersions = []schema.GroupVersion{v1alpha1.SchemeGroupVersion}
     )

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/plugin/pkg/admission/initialization**

src/k8s.io/kubernetes/plugin/pkg/admission/initialization/initialization.go

     var initializerFieldPath = field.NewPath("metadata", "initializers")

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/influxdata/influxdb/pkg/escape**

k8s.io/kubernetes/vendor/github.com/influxdata/influxdb/pkg/escape/strings.go

     var(
          Codes = map[byte][]byte{
               '.': []byte(`\,`),
               '"': []byte(`\"`),
               ' ': []byte(`\ `),
               '=': []byte(`\=`),
          }
          codesStr = map[string]string{}
     )
     func init(){
          for k,v:=range Codes{
               codesStr[string(k"] = string(v)
          }
     }

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/github.com/influxdata/influxdb/models**

k8s.io/kubernetes/vendor/github.com/influxdata/influxdb/models/consistency.go

     var(
          ErrInvalidConsistencyLevel = errors.New("invalid consistency level")
     )

k8s.io/kubernetes/vendor/github.com/influxdata/influxdb/models/points.go

     var(
          measurementEscapeCodes = map[byte][]byte{
               ',': []byte(`\,`),
               ' ': []byte(`\ `),
          }
          tagEscapeCodes = map[byte][]byte{
               ',': []byte(`\,`),
               ' ': []byte(`\ `),
               '=': []byte(`\=`),
          }
          ErrPointMustHaveAField = errors.New("point without fields is unsupported")
          ErrInvalidNumber = errors.New("invalid number")
          ErrInvalidPoint = errors.New("point is invalid")
          ErrMaxKeyLengthExceeded = errors.New("max key length exceeded")
     )

k8s.io/kubernetes/vendor/github.com/influxdata/influxdb/models/time.go

     var(
          minNanoTime = time.Unix(0, MinNanoTime).UTC()
          maxNanoTime = time.Unix(0, MaxNanoTime).UTC()
          ErrTimeoutOfRange = fmt.Errorf("time outside range %d - %d", MinNanoTime, MaxNanoTime)
     )

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/plugin/pkg/admission/initialresources**

src/k8s.io/kubernetes/plugin/pkg/admission/initialresources/admission.go

     var(
          source = flag.String("ir-data-source", "influxdb", "Data source used by InitialResources. Supported options: influxdb, gcm.")
          percentile = flag.Int64("ir-percentile", 90, "Which percentile of samples should InitialResources use when estimating resources. For experiment purposes.")
          nsOnly = flag.Bool("ir-namespace-only", false, "Whether the estimation should be made only based on data from the same namespace.")
     )

src/k8s.io/kubernetes/plugin/pkg/admission/initialresources/data_source.go

     var(
          influxdbHost = flag.String("ir-influxdb-host", "localhost:8080/api/v1/namespaces/kube-system/services/monitoring-influxdb:api/proxy", "Address of InfluxDB which contains metrics required by InitialResources")
          user = flag.String("ir-user", "root", "User used for connecting to InfluxDB")
          password = flag.String("ir-password", "root", "Password used for connecting to InfluxDB")
          db = flag.String("ir-dbname", "k8s", "InfluxDB database name which contains metrics required by InitialResources")
          hawkularConfig = flag.String("ir-hawkular", "", "Hawkular configuration URL")
     )

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/plugin/pkg/admission/noderestriction**

src/k8s.io/kubernetes/plugin/pkg/admission/noderestriction/admission.go

     var(
          podResource = api.Resource("pods")
          nodeResource = api.Resource("nodes")
     )

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/plugin/pkg/admission/podtolerationrestriction/apis/podtolerationrestriction**

src/k8s.io/kubernetes/plugin/pkg/admission/podtolerationrestriction/apis/podtolerationrestriction/register.go

     var(
          SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)
          AddToScheme = SchemeBuilder.AddToScheme
     )

src/k8s.io/kubernetes/plugin/pkg/admission/podtolerationrestriction/apis/podtolerationrestriction/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/plugin/pkg/admission/podtolerationrestriction/apis/podtolerationrestriction/v1alpha1**

src/k8s.io/kubernetes/plugin/pkg/admission/podtolerationrestriction/apis/podtolerationrestriction/v1alpha1/register.go

     var(
          SchemeBuilder runtime.SchemeBuilder
          localSchemeBuilder = &SchemeBuilder
          AddToScheme = localSchemeBuilder.AddToScheme
     )

     func init(){
          localSchemeBuilder.Register(addKnownTypes, RegisterDefaults)
     }

src/k8s.io/kubernetes/plugin/pkg/admission/podtolerationrestriction/apis/podtolerationrestriction/v1alpha1/zz_generated.conversion.go

     func init(){
          SchemeBuilder.Register(RegisterConversions)
     }

src/k8s.io/kubernetes/plugin/pkg/admission/podtolerationrestriction/apis/podtolerationrestriction/v1alpha1/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/plugin/pkg/admission/podtolerationrestriction**

src/k8s.io/kubernetes/plugin/pkg/admission/podtolerationrestriction/config.go

     var(
          groupFactoryRegistry = make(announced.APIGroupFactoryRegistry)
          registry = registered.NewOrDir(os.Getenv("KUBE_API_VERSIONS"))
          scheme = runtime.NewScheme()
          codecs = serializer.NewCodecFactory(scheme)
     )
     func init(){
          install.Install(groupFactoryRegistry, registry, scheme)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/plugin/pkg/admission/resourcequota/apis/resourcequota**

src/k8s.io/kubernetes/plugin/pkg/admission/resourcequota/apis/resourcequota/register.go

     var(
          SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)
          AddToScheme = SchemeBuilder.AddToScheme
     )

src/k8s.io/kubernetes/plugin/pkg/admission/resourcequota/apis/resourcequota/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/plugin/pkg/admission/resourcequota/apis/resourcequota/v1alpha1**

src/k8s.io/kubernetes/plugin/pkg/admission/resourcequota/apis/resourcequota/v1alpha1/register.go

     var(
          SchemeBuilder runtime.SchemeBuilder
          localSchemeBuilder = &SchemeBuilder
          AddToScheme = localSchemeBuilder.AddToScheme
     )

     func init(){
          localSchemeBuilder.Register(addKnownTypes, RegisterDefaults)
     }

src/k8s.io/kubernetes/plugin/pkg/admission/resourcequota/apis/resourcequota/v1alpha1/zz_generated.conversion.go

     func init(){
          SchemeBuilder.Register(RegisterConversions)
     }

src/k8s.io/kubernetes/plugin/pkg/admission/resourcequota/apis/resourcequota/v1alpha1/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/plugin/pkg/admission/resourcequota**

src/k8s.io/kubernetes/plugin/pkg/admission/resourcequota/config.go

     var(
          groupFactoryRegistry = make(announced.APIGroupFactoryRegistry)
          registry = registered.NewOrDie(os.Getenv("KUBE_API_VERSIONS"))
          scheme = runtime.NewScheme()
          codecs = serializer.NewCodecFactory(scheme)
     )
     func init(){
          install.Install(groupFactoryRegistry, registry, scheme)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/admission**

src/k8s.io/kubernetes/pkg/apis/admission/register.go

     var(
          SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)
          AddToScheme = SchemeBuilder.AddToScheme
     )

src/k8s.io/kubernetes/pkg/apis/admission/types.generated.go

     var(
          codecSelferBitsize1234 = uint8(reflect.TypeOf(uint(0)).Bits())
          codecSelferOnlyMapOrArrayEncodeToStructErr1234 = errors.New(`only encoded map or array can be decoded into a struct`)
     )

src/k8s.io/kubernetes/pkg/apis/admission/types.generated.go

     func init(){
          if codec1978.GenVersion != 5{
               _,file,_,_:=runtime.Caller(0)
               err:=fmt.Errorf("codecgen version mismatch: current: %v, need %v. Re-generate file: %v",
                    5, codec1978.GenVersion,file)
               panic(err)
          }
          if false{
               var v0 pkg1_v1.TypeMeta
               var v1 pkg2_runtime.Object
               var v2 pkg5_types.UID
               var v3 pkg3_admission.Operation
               var v4 pkg4_authentication.UserInfo
               _,_,_,_,_ = v0, v1, v2,v3,v4
          }
     }

src/k8s.io/kubernetes/pkg/apis/admission/zz_generated.conversion.go

     func init(){
          SchemeBuilder.Register(RegisterConversions)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/admission/v1alpha1**

src/k8s.io/kubernetes/pkg/apis/admission/v1alpha1/generated.pb.go

     var(
          ErrInvalidLengthGenerated = fmt.Errorf("proto: negative length found during unamrshaling")
          ErrIntOverflowGenerated = fmt.Errorf("proto: integer overflow")
     )

src/k8s.io/kubernetes/pkg/apis/admission/v1alpha1/register.go

     var(
          SchemeBuilder runtime.SchemeBuilder
          localSchemeBuilder = &SchemeBuilder
          AddToScheme = localSchemeBuilder.AddToScheme
     )

src/k8s.io/kubernetes/pkg/apis/admission/v1alpha1/types.generated.go

     var(
          codecSelferBitsize1234 = uint8(reflect.TypeOf(uint(0)).Bits())
          codecSelferOnlyMapOrArrayEncodeToStructErr1234 = errors.New(`only encoded map or array can be decoded into a struct`)
     )

src/k8s.io/kubernetes/pkg/apis/admission/v1alpha1/types_swagger_doc_generated.go

     var map_AdmissionReview= map[string]string{
          "":"",
          ...
     }
     ...

src/k8s.io/kubernetes/pkg/apis/admission/v1alpha1/generated.pb.go

     func init(){
          proto.RegisterType((*AdmissionReview)(nil), "k8s.io.kubernetes.pkg.apis.admission.v1alpha1.AdmissionReview")
          ...
     }
     func init(){
          proto.RegisterFile("k8s.io/kubernetes/pkg/apis/admission/v1alpha1/generated.proto", fileDescriptorGenerated)
     }

src/k8s.io/kubernetes/pkg/apis/admission/v1alpha1/register.go

     func init(){
          localSchemeBuilder.Register(addKnownTypes, addDefaultingFuncs, addConversionFuncs)
     }

src/k8s.io/kubernetes/pkg/apis/admission/v1alpha1/types.generated.go

     func init(){
          if codec1978.GenVersion != 5{
               _,file,_,_:=runtime.Caller(0)
               err:=fmt.Errorf("codecgen version mismatch: current: %v, need %v. Re-generate file: %v",5,codec1978.GenVersion,file)
               panic(err)
          }
          if false{
               var v0 pkg1_v1.TypeMeta
               var v1 pkg2_runtime.RawExtension
               var v2 pkg5_types.UID
               var v3 pkg3_admission.Operation
               var v4 pkg4_v1.UserInfo
               _,_,_,_,_= v0,v1,v2,v3,v4
          }
     }

src/k8s.io/kubernetes/pkg/apis/admission/v1alpha1/zz_generated.conversion.go

     func init(){
          SchemeBuilder.Register(RegisterConversions)
     }
src/k8s.io/kubernetes/pkg/apis/admission/v1alpha1/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/apis/admission/install**

src/k8s.io/kubernetes/pkg/apis/admission/install/install.go

     func init(){
          Install(api.GroupFactoryRegistry, api.Registry, api.Scheme)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/plugin/pkg/admission/webhook**

src/k8s.io/kubernetes/plugin/pkg/admission/webhook/admission.go

     var(
          groupVersions = []schema.GroupVersion{
               admissionv1alpha1.SchemeGroupVersion,
          }
     )

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/federation/apis/federation**

src/k8s.io/kubernetes/federation/apis/federation/register.go

     var(
          SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)
          AddToScheme = SchemeBuilder.AddToScheme
     )

src/k8s.io/kubernetes/federation/apis/federation/zz_generated.conversion.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/federation/apis/federation/v1beta1**

src/k8s.io/kubernetes/federation/apis/federation/v1beta1/generated.pb.go

     var(
          ErrInvalidLengthGenerated = fmt.Errorf("proto: negative length found during unamrshaling")
          ErrIntOverflowGenerated = fmt.Errorf("proto: integer overflow")
     )

src/k8s.io/kubernetes/federation/apis/federation/v1beta1/register.go

     var(
          SchemeBuilder runtime.SchemeBuilder
          localSchemeBuilder = &SchemeBuilder
          AddToScheme = localSchemeBuilder.AddToScheme
     )

src/k8s.io/kubernetes/federation/apis/federation/v1beta1/types.generated.go

     var(
          codecSelferBitsize1234 = uint8(reflect.TypeOf(uint(0)).Bits())
          codecSelferOnlyMapOrArrayEncodeToStructErr1234 = errors.New(`only encoded map or array can be decoded into a struct`)
     )

src/k8s.io/kubernetes/federation/apis/federation/v1beta1/types_swagger_doc_generated.go

     var map_Cluster= map[string]string{
          "":"",
          ...
     }
     ...

src/k8s.io/kubernetes/federation/apis/federation/v1beta1/generated.pb.go

     func init(){
          proto.RegisterType((*Cluster)(nil), "k8s.io.kubernetes.federation.apis.federation.v1beta1.Cluster")
          ...
     }
     func init(){
          proto.RegisterFile("k8s.io/kubernetes/federation/apis/federation/v1beta1/generated.proto", fileDescriptorGenerated)
     }

src/k8s.io/kubernetes/federation/apis/federation/v1beta1/register.go

     func init(){
          localSchemeBuilder.Register(addKnownTypes, addDefaultingFuncs, addConversionFuncs)
     }

src/k8s.io/kubernetes/federation/apis/federation/v1beta1/types.generated.go
     func init(){
          if codec1978.GenVersion != 5{
               _,file,_,_:=runtime.Caller(0)
               err:=fmt.Errorf("codecgen version mismatch: current: %v, need %v. Re-generate file: %v",5,codec1978.GenVersion,file)
               panic(err)
          }
          if false{
               var v0 pkg2_v1.Time
               var v1 pkg3_types.UID
               var v2 pkg1_v1.LocalObjectReference
               var v3 time.Time
               _,_,_,_= v0,v1,v2,v3
          }
     }

src/k8s.io/kubernetes/federation/apis/federation/v1beta1/zz_generated.conversion.go

     func init(){
          SchemeBuilder.Register(RegisterConversions)
     }
src/k8s.io/kubernetes/federation/apis/federation/v1beta1/zz_generated.deepcopy.go

     func init(){
          SchemeBuilder.Register(RegisterDeepCopies)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/federation/apis/federation/install**

src/k8s.io/kubernetes/federation/apis/federation/install/install.go

     func init(){
          Install(api.GroupFactoryRegistry, api.Registry, api.Scheme)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/federation/client/clientset_generated/federation_internalclientset/scheme**

src/k8s.io/kubernetes/federation/client/clientset_generated/federation_internalclientset/scheme/register.go

     var Scheme = runtime.NewScheme()
     var Codecs = serializer.NewCodecFactory(Scheme)
     var ParameterCodec = runtime.NewParameterCodec(Scheme)
     var Registry = registered.NewOrDie(os.Getenv("KUBE_API_VERSIONS"))
     var GroupFactoryRegistry = make(announced.APIGroupFactoryRegistry))

     func init(){
          v1.AddToGroupVersion(Scheme, schema.GroupVersion{Version:v"v1"})
          Install(GroupFactoryRegistry, Registry, Scheme)
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/cmd/kubeadm/app/apis/kubeadm**

src/k8s.io/kubernetes/cmd/kubeadm/app/apis/kubeadm/env.go

     var GlobalEnvParams = SetEnvParams()

src/k8s.io/kubernetes/cmd/kubeadm/app/apis/kubeadm/register.go

     var(
          SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)
          AddToScheme = SchemeBuilder.AddToScheme
     )

______________________________________________________________________________________________
**初始化k8s.io/kubernetes/vendor/golang.org/x/crypto/ssh**

k8s.io/kubernetes/vendor/golang.org/x/crypto/ssh/certs.go

     var certAlgoNames = map[string]string{
          KeyAlgoRSA: CertAlgoRSAv01,
          KeyAlgoDSA: CertAlgoDSAv01,
          KeyAlgoECDSA256: CertAlgoECDSA256v01,
          KeyAlgoECDSA384: CertAlgoECDSA384v01,
          KeyAlgoECDSA512: CertAlgoECDSA512v01,
          KeyAlgoED25519: CertAlgoED25519v01,
     }

k8s.io/kubernetes/vendor/golang.org/x/crypto/ssh/channel.go

     var errUndecided = errors.New("ssh: must Accept or Reject channel")
     var errDecidedAlready = errors.New("ssh: can call Accept or Reject only once")

k8s.io/kubernetes/vendor/golang.org/x/crypto/ssh/cipher.go

     var cipherModes = map[string]*streamCipherMode{
          "aes128-ctr": {16, aes.BlockSize, 0, newAESCTR},
          ...
     }

k8s.io/kubernetes/vendor/golang.org/x/crypto/ssh/common.go

     var hashFuncs = map[string]crypto.Hash{
          KeyAlgoRSA: crypto.SHA1,
          ...
     }

k8s.io/kubernetes/vendor/golang.org/x/crypto/ssh/kex.go

     var kexAlgoMap = map[string]kexAlgorithm{}

k8s.io/kubernetes/vendor/golang.org/x/crypto/ssh/mac.go

     var macModes = map[string]*macMode{
          "hmac-sha2-256": {32, func(key []byte) hash.Hash{
               return hmac.New(sha256.New, key)
          }},
          ...
     }

k8s.io/kubernetes/vendor/golang.org/x/crypto/ssh/messages.go

     var errShortRead = errors.New("ssh: short read")
     var bigOne = big.NewInt(1)
     var bitIntType = reflect.TypeOf((*big.Int)(nil))

k8s.io/kubernetes/vendor/golang.org/x/crypto/ssh/session.go

     var signals = map[Signal]int{
          SIGABRT: 6,
          ...
     }

k8s.io/kubernetes/vendor/golang.org/x/crypto/ssh/tcpip.go

     var portRandomizer = rand.New(rand.NewSource(time.Now().UnixNano()))

k8s.io/kubernetes/vendor/golang.org/x/crypto/ssh/kex.go

     func init(){
          p,_:=new(big.Int).SetString("FFFFFFFFFFFFFFFFC90FDAA22168C234C4C6628B80DC1CD129024E088A67CC74020BBEA63B139B22514A08798E3404DDEF9519B3CD3A431B302B0A6DF25F14374FE1356D6D51C245E485B576625E7EC6F44C42E9A637ED6B0BFF5CB6F406B7EDEE386BFB5A899FA5AE9F24117C4B1FE649286651ECE65381FFFFFFFFFFFFFFFF",16)
          kexAlgoMap[kexAlgoDH1SHA1] = &dhGroup{
               g: new(big.Int).SetInt64(2),
               p: p,
          }
          ...
     }

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/ssh**

src/k8s.io/kubernetes/pkg/ssh/ssh.go

     var(
          tunnelOpenCounter = prometheus.NewCounter(
               prometheus.CounterOpts{
                    Name: "ssh_tunnel_open_count",
                    Help: "Counter of ssh tunnel total open attepts",
               },
          )
          ...
     )

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/plugin/pkg/auth/authenticator/token/bootstrap**

src/k8s.io/kubernetes/plugin/pkg/auth/authenticator/token/bootstrap/bootstrap.go

     var(
          tokenRegexpString = "^([a-z0-9]{6})\\.([a-z0-9]{16})$"
          tokenRegexp = regexp.MustCompile(tokenRegexpString)
     )

______________________________________________________________________________________________
**初始化src/k8s.io/kubernetes/pkg/client/metrics/prometheus**

src/k8s.io/kubernetes/pkg/client/metrics/prometheus/prometheus.go

     var(
          requestLatency = prometheus.NewHistogramVec(
               prometheus.HistogramOpts{
                    Name: "rest_client_request_latency_seconds",
                    Help: "Request latency in seconds. Broken down by verb and URL.",
               },
               []string{"verb", "url"},
          )
          ...
     )

     func init(){
          prometheus.MustRegister(requestLatency)
          prometheus.MustRegister(requestResult)
          metrics.Register(&latencyAdapter{requestLatency}, &resultAdapter{requestResult})
     }

src/k8s.io/kubernetes/pkg/version/verflag/verflag.go

     var(
          versionFlag = Version("version", VersionFalse, "Print version information and quit")
     )

______________________________________________________________________________________________
**初始化src/runtime/proc.go**

fn:=main_init

______________________________________________________________________________________________
**k8s.io/kubernetes/cmd/kube-apiserver/apiserver.go**

    func main() {
	    rand.Seed(time.Now().UTC().UnixNano())
	
        s := options.NewServerRunOptions()
	    s.AddFlags(pflag.CommandLine)

	    flag.InitFlags()
	    logs.InitLogs()
	    defer logs.FlushLogs()

	    verflag.PrintAndExitIfRequested()

	    if err := app.Run(s, wait.NeverStop); err != nil {
		    fmt.Fprintf(os.Stderr, "%v\n", err)
		    os.Exit(1)
	    }
    }


## 参考文献
* [bytes.Buffer原理及使用](../Golang/bytes.Buffer.md)

_______________________________________________________________________
[[返回/reference/remote-debug.md]](./remote-debug.md) 