Serializer序列化器
=========================================================
## 简介
序列化器的作用是：编码和解码。编码是指把结构化对象（Go struct Object）转换为序列化格式对象（gvk对象或者内部无版本对象），解码是对应的反向操作。

## 1. Serializer接口
含义：

    序列化的核心接口，所有序列化器必须实现该接口。

路径：

    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/runtime/interfaces.go

定义：

    type Serializer interface{
        Encoder                                   //编码器接口
        Decoder                                   //解码器接口
    }

### 1.1 Encoder接口
含义：

    编码器接口。将一个对象写入一个流中。

路径：

    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/runtime/interfaces.go

定义：

    type Encoder interface{
        Encode(obj Object, w io.Writer) error     //如果没有定义conversion，或者版本不兼容，则返回错误。
    }

### 1.2 Decoder接口
含义：

    解码器接口。将一个输入数据转换为一个对象，并返回转换后对象的gvk。

路径：

    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/runtime/interfaces.go

参数：

    defaults是默认的gvk，可能被用于转换过程中。
    into-可能会被用于目标对象。

定义：

    type Decoder interface{
        Decode(data []byte, defaults *schema.GroupVersionKind, into Object) (Object, *schema.GroupVersionKind, error)
    }

## 2. Serializer结构体
含义：

    用于json或yaml对象和Go结构化对象之间转换的序列化器。Serializer结构体需要实现Serializer接口，即实现Encode和Decode方法。
     
路径：

    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/runtime/serializer/json/json.go

定义：

    type Serializer struct{
        meta    MetaFactory                            //获取JSON对象的gvk信息。
        creater runtime.ObjectCreater
        typer   runtime.ObjectTyper
        yaml    bool                                   //是否是ymal对象。
        pretty  bool                                   //是否输出漂亮格式，即有缩进排版，更方便人类阅读。
    }

### 2.1 MetaFactory
含义：

    获取JSON对象的gvk信息。

路径：

    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/runtime/serializer/json/meta.go

定义：

    type MetaFactory interface{
        Interpret(data []byte) (*schema.GroupVersionKind, error)   //返回从data中提取的gvk信息。
    }

### 2.2 创建DefaultMetaFactory对象
含义：

    创建默认的实现MetaFactory接口的对象，即实现Interpret方法。从JSON对象中获取Go结构对象的gvk信息。

路径：

    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/runtime/serializer/json/meta.go

定义：

    var DefaultMetaFactory = SimpleMetaFactory{}
    type SimpleMetaFactory struct{}
    func (SimpleMetaFactory) Interpret(data []byte) (*schema.GroupVersionKind, error){
        findKind:=struct{
            APIVersion string `json:"apiVersion,omitempty"`
            Kind       string `json:"kind,omitempty"`
        }{}
        if err:=json.Unmarshal(data, &findKind);err!=nil{
            return nil,fmt.Errorf("couldn't get version/kind; json parse error: %v", err)
        }
        gv,err:=schema.ParseGroupVersion(findKind.APIVersion)
        if err!=nil {
            return nil,err
        }
        return &schema.GroupVersionKind{Group: gv.Group, Version: gv.Version, Kind: findKind.Kind}, nil
    }

### 2.3 创建JSON-Serializer结构体对象
含义：

    创建jsonSerializer结构体对象。Serializer结构体对象必须实现Serializer接口，即实现Encode和Decode方法。

路径：

    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/runtime/serializer/json/json.go

定义：

    func NewSerializer(meta MetaFactory, creater runtime.ObjectCreater, typer runtime.ObjectTyper, pretty bool)*Serializer{
        return &Serializer{
            meta:     meta,
            creater:  creater,
            type:     typer,
            yaml:     false,        //表示创建json序列化器
            pretty:   pretty,
        }
    }

### 2.4 创建YAML-Serializer结构体对象
含义：

    创建yaml Serializer结构体对象。Serializer结构体对象必须实现Serializer接口，即实现Encode和Decode方法。

路径：

    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/runtime/serializer/json/json.go

定义：

    func NewYAMLSerializer(meta MetaFactory, creater runtime.ObjectCreater, typer runtime.ObjectTyper, pretty bool)*Serializer{
        return &Serializer{
            meta:     meta,
            creater:  creater,
            type:     typer,
            yaml:     true,        //表示创建yaml序列化器
        }
    }

### 2.5 Decode方法
含义：

    json解码器，把json对象转换为Go结构对象，并返回该对象和该从json对象中提取的实际gvk信息。

路径：

    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/runtime/serializer/json/json.go

定义：

    func (s *Serializer) Decode (originalData []byte, gvk *schema.GroupVersionKind, into runtime.Object)(runtime.Object, *schema.GroupVersionKind, error) {
        //如果into是VersionedObjects类型，则使用最后一个版本进行解码。VersionedObjects结构体供解码器访问一个对象的所有版本信息。
        if versioned, ok := into.(*runtime.VersionedObjects); ok{   
            into = versioned.Last()
            obj,actual,err := s.Decode(originalData, gvk, into)           //actual
            if err!=nil {
                return nil,actual,err
            }
            versioned.Objects = []runtime.Object{obj}
            return versioned,actual,nil                                   //actual表示实际的gvk。
        }
        data:=originalData
        //yaml要转为json
        if s.yaml{
            altered,err:=yaml.YAMLTOJSON(data)
            if err!=nil{
                return nil,nil,err
            }
            data = altered
        }
        actual,err := s.meta.Interpret(data)                              //返回实际的gvk信息，即从originalData中获取的gvk信息。
        if err!=nil{
            return nil, nil, err
        }

        //如果defaults不为空，并且actual的gvk信息缺少，则使用defaults的gvk信息来补全。
        if gvk!=nil{
            if len(actual.Kind)==0{
                actual.Kind = gvk.Kind
            }
            if len(actual.Version)==0 && len(actual.Group)==0{
                actual.Group = gvk.Group
                actual.Version = gvk.Version
            }
            if len(actual.Version)==0 && actual.Group==gvk.Group{
                actual.Version = gvk.Version
            }
        }

        //如果into是Unknown类型，则不对originalData进行解码，直接返回，并且返回从originalData中提取的gvk信息。
        if unk,ok := into.(*runtime.Unknown); ok&&unk !=nil {
            unk.Raw = originalData
            unk.ContentType = runtime.ContentTypeJSON
            unk.GetObjectKind().SetGroupVersionKind(*actual)
            return unk,actual,nil
        }
        if into!=nil{
            types,_,err:=s.typer.ObjectKinds(into)
            switch{
            case runtime.IsNotRegisteredError(err):
                if err:=codec.NewDecoderBytes(data, new(codec.JsonHandle)).Decode(into); err!=nil{
                    return nil,actual,err
                }
                return into,actual,nil
            case err!=nil:
                return nil,actual,err
            default:
                typed:=types[0]
                if len(actual.Kind)==0{
                    actual.Kind = typed.Kind
                }
                if len(actual.Version)==0 && len(actual.Group)==0{
                    actual.Group = typed.Group
                    actual.Version = typed.Version
                }
                if len(actual.Version)==0 && actual.Group==typed.Group{
                    actual.Version = typed.Version
                }
            }
        }
        if len(actual.Kind)==0{
            return nil,actual,runtime.NewMissingKindErr(string(originalData))
        }
        if len(actual.Version)==0{
            return nil,actual,runtime.NewMissingVersionErr(string(originalData))
        }

        obj,err:=runtime.UseOrCreateObject(s.typer, s.creater, *actual, into)
        if err!=nil{
            return nil,actual,err
        }
        if err:=codec.NewDecoderBytes(data, new(codec.JsonHandle)).Decode(obj); err!=nil{
            return nil,actual,err
        }
        return obj,actual,nil
    }

### 2.6 Encode方法
含义：

    编码就是把Go结构对象转换为json或yaml格式的对象，再写到流中，具体是写到bytes.Buffer中。bytes.Buffer又写到哪里，得看调用实参w要求写到哪里，如果是要求写入文件，则写到json或yaml文件中。

路径：

    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/runtime/serializer/json/json.go

定义：

    func (s *Serializer) Encode (obj runtime.Object, w io.Writer) error{
        //如果是yaml，则转成yaml格式写出去。底层没有使用sync.Pool。
        if s.yaml{
            json,err:=json.Marshal(obj)                        
            if err!=nil{
                return err
            }
            data,err:=yaml.JSONToYAML(json)
            if err!=nil{
                return err
            }
            _,err = w.Write(data)
            return err
        }

        //如果要求漂亮的格式，则输出文件行会缩进2个空格。底层没有使用sync.Pool。
        if s.pretty{
            data,err:=json.MarshalIndent(obj, "", "  ")  //pretty格式为缩进2个空格。
            if err!=nil {
                return err
            }
            _,err = w.Write(data)
            return err
        }

        //调用JSON原生编码器，把JSON数据写到bytes.Buffer中，底层使用了sync.Pool技术，sync.Pool在其他文章中已经介绍过。
        encoder := json.NewEncoder(w)        
        return encoder.Encode(obj)
    }
