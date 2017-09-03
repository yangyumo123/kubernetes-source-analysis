api.Scheme分析
===============================================================
## 简介
api.Scheme是"/api/v1"中使用的Scheme。api.Scheme是Copier接口类型（Copier又是runtime.ObjectCopier接口类型 ）的对象，因此需要实现Copier接口中的方法。

## 1. runtime.ObjectCopier
含义：

    返回一个对象的精确复制，或者当复制不完全时返回错误。

路径：

    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/runtime/interfaces.go

定义：

    type ObjectCopier interface{
        Copy(Object) (Object, error)
    }

    什么是ObjectCopier中的Object？ObjectCopier中使用的Object是API对象需要实现的一个接口，并不能说实现了Object接口就能称为对象。因为Object接口仅仅是对象的一个属性，即把scheme中的type序列化到网络中的gvk（group、version、kind）对象。要求所有注册到Scheme中的对象是都必须实现Object接口。
    type Object interface {
        GetObjectKind() schema.ObjectKind
    }

    //ObjectKind接口：用于序列化。把Scheme中的type转换为序列化版本（gvk）的对象。
    //路径：k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/runtime/schema/interfaces.go
    type ObjectKind interface{
        SetGroupVersionKind(kind GroupVersionKind)
        GroupVersionKind() GroupVersionKind
    }

    //k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/runtime/schema/group_version.go
    type GroupVersionKind struct {
        Group string
        Version string
        Kind string
    }

实现Object接口的对象包括：

### 1.1 api.ObjectReference：
含义：

    包含对象的足够信息供检查或修改对象。内部版本对象。ObjectReference结构体实现Object接口，即实现GetObjectKind方法。GetObjectKind方法返回ObjectReference对象，而返回值又是schema.ObjectKind类型接口，因此ObjectReference对象也要实现schema.ObjectKind接口，即实现GetObjectKind方法和SetGroupVersionKind方法。

路径：

    k8s.io/kubernetes/pkg/api/types.go

定义：

    type ObjectReference struct{
        Kind            string
        Namespace       string
        Name            string
        UID             types.UID
        APIVersion      string
        ResourceVersion string      //内部版本，用于并发控制。目前使用etcd中的modifiedIndex表示，未来可能会使用时间戳等方式。关于ResourceVersion的使用可看参考文献[resourceversion分析]。
        FieldPath       string      //定义对象的一部分而不是整个对象。例如，一个表示Pod里容器的对象，值可能是："spec.containers{name}"或者"spec.containers[2]"。该参数未来可能会改变。
    }

    //ObjectReference对象要实现Object接口，即实现GetObjectKind方法。
    //k8s.io/kubernetes/pkg/api/objectreference.go
    func (obj *ObjectReference) GetObjectKind() schema.ObjectKind { 
        return obj 
    }
    
    //ObjectReference对象要实现schema.ObjectKind接口，即实现SetGroupVersionKind和GroupVersionKind方法。
    func (obj *ObjectReference) SetGroupVersionKind(gvk schema.GroupVersionKind){
        obj.APIVersion, obj.Kind = gvk.ToAPIVersionAndKind()
    }
    func (obj *ObjectReference) GroupVersionKind() schema.GroupVersionKind{
        return schema.FromAPIVersionAndKind(obj.APIVersion, obj.Kind)
    }

### 1.2 v1.ObjectReference
含义：

    和前面介绍的对象类似，只是此处对象是对外的v1版本对象，因此有tag。

路径：

    k8s.io/kubernetes/pkg/api/v1/types.go

定义：

    type ObjectReference struct {
        Kind            string `json:"kind,omitempty" protobuf:"bytes,1,opt,name=kind"`
        Namespace       string `json:"namespace,omitempty" protobuf:"bytes,2,opt,name=namespace"`
        Name            string `json:"name,omitempty" protobuf:"bytes,3,opt,name=name"`
        UID             types.UID `json:"uid,omitempty" protobuf:"bytes,4,opt,name=uid,casttype=k8s.io/apimachinery/pkg/types.UID"`
        APIVersion      string `json:"apiVersion,omitempty" protobuf:"bytes,5,opt,name=apiVersion"`
        ResourceVersion string `json:"resourceVersion,omitempty" protobuf:"bytes,6,opt,name=resourceVersion"`
        FieldPath       string `json:"fieldPath,omitempty" protobuf:"bytes,7,opt,name=fieldPath"`
    }

    //ObjectReference对象要实现Object接口，即实现GetObjectKind方法。
    //k8s.io/kubernetes/pkg/api/v1/objectreference.go
    func (obj *ObjectReference) GetObjectKind() schema.ObjectKind { 
        return obj 
    }
    
    //ObjectReference对象要实现schema.ObjectKind接口，即实现SetGroupVersionKind和GroupVersionKind方法。
    func (obj *ObjectReference) SetGroupVersionKind(gvk schema.GroupVersionKind){
        obj.APIVersion, obj.Kind = gvk.ToAPIVersionAndKind()
    }
    func (obj *ObjectReference) GroupVersionKind() schema.GroupVersionKind{
        return schema.FromAPIVersionAndKind(obj.APIVersion, obj.Kind)
    }

约定：以下各对象的GetObjectKind()方法只返回TypeMeta，因此忽略所有对象的字段定义。只列出与ObjectKind中的Object相关的方法。

### 1.3 v0.Policy
含义：

    Policy对象包含ABAC策略规则。

路径：

    k8s.io/kubernetes/pkg/apis/abac/v0/types.go

定义：

    type Policy struct {
        metav1.TypeMeta `json:",inline"`
        ...
    }

    //k8s.io/kubernetes/pkg/apis/abac/v0/register.go
    func (obj *Policy)GetObjectKind() schema.ObjectKind {
        return &obj.TypeMeta 
    }

    //TypeMeta要实现schema.ObjectKind接口：
    //k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1/types.go
    type TypeMeta struct{
        Kind       string `json:"kind,omitempty" protobuf:"bytes,1,opt,name=kind"`
        APIVersion string `json:"apiVersion,omitempty" protobuf:"bytes,2,opt,name=apiVersion"`  //"group/version"格式。
    }

    //k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1/meta.go
    func (obj *TypeMeta) SetGroupVersionKind(gvk schema.GroupVersionKind) {
        obj.APIVersion, obj.Kind = gvk.ToAPIVersionAndKind()                                    //将schema.GroupVersionKind转换为obj.APIVersion（"group/version"格式）和obj.Kind。实现细节请参见源码。
    }
    func (obj *TypeMeta) GroupVersionKind() schema.GroupVersionKind {
        return schema.FromAPIVersionAndKind(obj.APIVersion, obj.Kind)                           //用TypeMeta中的APIVersion和Kind填充schema.GroupVersionKind，并返回。
    }

### 1.4 v1beta1.Policy
含义：

    同v0.Policy。

路径：

    k8s.io/kubernetes/pkg/apis/abac/v1beta1/types.go

定义：

    type Policy struct{...}

    //k8s.io/kubernetes/pkg/apis/abac/v1beta1/register.go
    func (obj *Policy) GetObjectKind() schema.ObjectKind {
        return &obj.TypeMeta
    }

### 1.5 metaonly.MetadataOnlyObject & metaonly.MetadataOnlyObjectList
含义：

    MetadataOnlyObject仅仅允许apiVersion, kind和metadata。

路径：

    k8s.io/kubernetes/pkg/controller/garbagecollector/metaonly/types.go

定义：

    type MetadataOnlyObject struct{
        metav1.TypeMeta `json:",inline"`
        metav1.ObjectMeta `json:"metadata,omitempty"`  //metadata
    }
    type MetadataOnlyObjectList struct{
        metav1.TypeMeta `json:",inline"`
        metav1.ListMeta `json:"metadata,omitempty"`
        Items []MetadataOnlyObject `json:"items"`
    }
    //k8s.io/kubernetes/pkg/controller/garbagecollector/metaonly/metaonly.go
    func (obj *MetadataOnlyObject) GetObjectKind() schema.ObjectKind {
        return obj
    }
    func (obj *MetadataOnlyObjectList) GetObjectKind() schema.ObjectKind {
        return obj
    }
     
### 1.6 v1.TypeMeta
含义：

    该对象已经在v0.Policy对象中介绍过了。

路径：

    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1/types.go

### 1.7 unstructured.Unstructured & unstructured.UnstructuredList
含义：

    Unstructured对象允许不具有Golang结构体的对象以通用方式进行操作，可以用来处理来自插件的API对象。Unstructured对象仍然具有TypeMeta特征-kind，version等。

路径：

    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1/unstructured/unstructured.go

定义：

    type Unstructured struct{
        Object map[string]interface{}
    }
    type UnstructuredList struct{
        Object map[string]interface{}
        Items []Unstructured `json:"items"`
    }
    func (obj *Unstructured) GetObjectKind() schema.ObjectKind {
        return obj
    }
    func (obj *UnstructuredList) GetObjectKind() schema.ObjectKind {
        return obj
    }
    func (u *Unstructured)SetGroupVersionKind(gvk schema.GroupVersionKind){...}
    func (u *Unstructured)GroupVersionKind() schema.GroupVersionKind{...}
    func (u *UnstructuredList)SetGroupVersionKind(gvk schema.GroupVersionKind){...}
    func (u *UnstructuredList)GroupVersionKind() schema.GroupVersionKind{...}

### 1.8 watch.InternalEvent & watch.WatchEvent
含义：

    watch事件对象。

路径：

    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/apis/meta/v1/watch.go

定义：

    type InternalEvent watch.Event
    type Event struct{
        Type EventType
        Object runtime.Object
    }
    type WatchEvent struct{
        Type string `json:"type" protobuf:"bytes,1,opt,name=type"`
        Object runtime.RawExtension `json:"object" protobuf:"bytes,2,opt,name=object"`
    }

    //返回空类型对象。
    func (e *InternalEvent) GetObjectKind() schema.ObjectKind {
        return schema.EmptyObjectKind
    }
    func (e *WatchEvent) GetObjectKind() schema.ObjectKind {
        return schema.EmptyObjectKind
    }
    //k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/runtime/schema/interfaces.go
    var EmptyObjectKind = emptyObjectKind{}
    type emptyObjectKind struct{}
    func (emptyObjectKind) SetGroupVersionKind(gvk GroupVersionKind){}
    func (emptyObjectKind) GroupVersionKind() GroupVersionKind {
        return GroupVersionKind{} 
    }

### 1.9 runtime.encodable
含义：
    
    可编码的对象。

路径：

    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/runtime/embedded.go

定义：

    type encodable struct{
        E        Encoder `json:"-"`
        obj      Object
        versions []schema.GroupVersion
    }
    func (e encodable)GetObjectKind() schema.ObjectKind {
        return e.obj.GetObjectKind() 
    }

### 1.10 runtime.Unknown & runtime.VersionedObjects
路径：

    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/runtime/types.go

定义：

    //未知类型对象，用于处理来自插件的API对象。Unknown对象仍然有TypeMeta特征-kind，version等。
    type Unknown struct{
        TypeMeta `json:",inline" protobuf:"bytes,1,opt,name=typeMeta"`
        ...
    }

    //VersionedObjects对象允许在decode过程中访问对象的所有版本。
    type VersionedObjects struct{
        Objects []Object
    }

    //k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/runtime/register.go
    func (obj *Unknown) GetObjectKind() schema.ObjectKind {return &obj.TypeMeta}

    //取最后一个Object对象。如果存在，则取该对象的ObjectKind；如果不存在，则创建一个EmptyObjectKind。
    func (obj *VersionedObjects)GetObjectKind() schema.ObjectKind{
        last:=obj.Last()
        if last == nil{
            return schema.EmptyObjectKind
        }
        return last.GetObjectKind()
    }
    func (obj *VersionedObjects)Last()Object{
        if len(obj.Objects)==0{
            return nil
        }
        return obj.Objects[len(obj.Objects)-1]
    }

### 1.11 watch.functionFakeRuntimeObject
含义：

    函数类型对象，我们可以把它塞进队列。

路径：

    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/watch/mux.go

定义：

    type functionFakeRuntimeObject func()
    func (obj functionFakeRuntimeObject)GetObjectKind() schema.ObjectKind{
        return schema.EmptyObjectKind
    }

### 1.12 rest.LocationStreamer
含义：

    将指定URL的内容转换为一个流。

路径：

    k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/registry/generic/rest/streamer.go

定义：

    type LocationStreamer struct{
        ...
    }
    func (obj *LocationStreamer)GetObjectKind() schema.ObjectKind{ 
        return schema.EmptyObjectKind 
    }

### 1.13 rest.ConnectRequest
含义：

    传递给准入控制的对象，用于连接请求。

路径：

    k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/registry/rest/rest.go

定义：

    type ConnectRequest struct{
        ...
    }
    func (obj *ConnectRequest)GetObjectKind()schema.ObjectKind { 
        return schema.EmptyObjectKind
    }

### 1.14 api.ObjectReference
含义：

    类似于1.1中对象。

路径：

    k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/api/types.go

定义：

    type ObjectReference struct{
        ...
    }
    //k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/api/objectreference.go
    func (obj *ObjectReference)GetObjectKind() schema.ObjectKind { 
        return obj 
    }

### 1.15 v1.ObjectReference
含义：

    类似于1.2中对象。
    
路径：

    k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/api/v1/types.go

定义：

    type ObjectReference struct{
        ...
    }
    //k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/api/v1/objectreference.go
    func (obj *ObjectReference)GetObjectKind() schema.ObjectKind { 
        return obj 
    }

### 1.16 api.Config
含义：

    给定用户连接到远程kubernetes集群所需的信息。

路径：

    k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/api/types.go

定义：

    type ObjectReference struct{
        ...
    }
    //k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/api/objectreference.go
    func (obj *Config)GetObjectKind() schema.ObjectKind { 
        return obj 
    }
    func (obj *Config)SetGroupVersionKind(gvk schema.GroupVersionKind){...}
    func (obj *Config)GroupVersionKind()schema.GroupVersionKind{...}

### 1.17 v1.Config
含义：

    给定用户连接到远程kubernetes集群所需的信息。

路径：

    k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/api//v1/types.go

定义：

    type ObjectReference struct{
        ...
    }
    //k8s.io/kubernetes/vendor/k8s.io/client-go/pkg/api/v1/objectreference.go
    func (obj *Config)GetObjectKind() schema.ObjectKind { return obj }
    func (obj *Config)SetGroupVersionKind(gvk schema.GroupVersionKind){...}
    func (obj *Config)GroupVersionKind()schema.GroupVersionKind{...}

## 2. runtime.NewScheme()
含义：

    创建一个默认的Scheme，这个Scheme仅用于api group，如果你想创建新的api group，那么不能使用这个Scheme定义，而应该使用extensions group。

路径：

    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/runtime/scheme.go

定义：

    func NewScheme() *Scheme{
        s := &Scheme{
            gvkToType:                 map[scheme.GroupVersionKind]reflect.Type{},
            typeToGVK:                 map[reflect.Type][]schema.GroupVersionKind{},
            unversionedTypes:          map[reflect.Type]schema.GroupVersionKind{},
            unversionedKinds:          map[string]reflect.Type{},
            cloner:                    conversion.NewCloner(),
            fieldLabelConversionFuncs: map[string]map[string]FieldLabelConversionFunc{},
            defaulterFuncs:            map[reflect.Type]func(interface{}){},
        }
        s.converter = conversion.NewConverter(s.nameFunc)
        s.AddConversionFuncs(DefaultEmbeddedConversions()...)
        if err:=s.AddConversionFuncs(DefaultStringConversions...); err!=nil{
            panic(err)
        }
        if err:=s.RegisterInputDefaults(&map[string][]string{}, JSONKeyMapper, conversion.AllowDifferentFieldTypeNames|conversion.IgnoreMissingFields); err!=nil{
            panic(err)
        }
        if err:=s.RegisterInputDefaults(&url.Values{}, JSONKeyMapper, conversion.AllowDifferentFieldTypeNames|conversion.IgnoreMissingFields);err!=nil{
            panic(err)
        }
        return s
    }
    
    //Scheme定义了序列化和反序列化API对象的方法，Scheme是type registry，用于gvk和Go schema之间的转换，以及不同版本Go schema之间的映射。在一个Scheme中，Type（reflect.Type）指Go struct，Version指Type的一个指定版本，Kind指Type在某个Version下的唯一名字，Group指一组Version、Kind和随时间变化的Type。Scheme在运行时不会改变，在注册完成以后才是线程安全的。
    type Scheme struct{
        gvkToType                   map[schema.GroupVersionKind]reflect.Type           //gvk -> Type。特定gvk对应唯一的Type
        typeToGVK                   map[reflect.Type][]schema.GroupVersionKind         //Type -> gvk。特定Type对应多个gvk，因此value是gvk slice。
        unversionedTypes            map[reflect.Type]schema.GroupVersionKind           //无版本类型。特定Type对应唯一的gvk。
        unversionedKinds            map[string]reflect.Type                            //一组kind名称，能在任意组或版本的上下文中被创建。
        fieldLabelConversionFuncs   map[string]map[string]FieldLabelConversionFunc     //version和resource到函数的映射，函数转换version中的resource field labels为内部版本。
        defaulterFuncs              map[reflect.Type]func(interface{})   
        converter                   *conversion.Converter                              //包含所有注册的conversion函数，也拥有默认的转换函数。
        cloner                      *conversion.Cloner                                 //包含所有注册的copy函数，也拥有默认的深度拷贝函数。
    }

    //转换一个field selector为内部表示。
    type FieldLabelConversionFunc func(label, value string) (internalLabel, internalValue string, err error)

下面详细介绍NewScheme()函数中的实现细节：

### 2.1 cloner
含义：

    创建一个Cloner对象，并注册字节slice深度拷贝函数。Cloner执行type之间的拷贝。

路径：

    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/conversion/cloner.go

定义：

    func NewCloner() *Cloner{
        c := &Cloner{
            deepCopyFuncs: map[reflect.Type]reflect.Value{},
            generatedDeepCopyFuncs: map[reflect.Type]func(in interface{}, out interface{}, c *Cloner) error{},
        }
        //注册byteSliceDeepCopy函数。
        if err:=c.RegisterDeepCopyFunc(byteSliceDeepCopy); err!=nil {
            panic(err)
        }
        return c
    }
    type Cloner struct{
        deepCopyFuncs map[reflect.Type]reflect.Value
        generatedDeepCopyFuncs map[reflect.Type]func(in interface{}, out interface{}, c *Cloner) error
    }

    //注册deepCopy函数，该函数是func(in interface{}, out interface{}, c *Cloner)error类型。
    func (c *Cloner) RegisterDeepCopyFunc(deepCopyFunc interface{}) error{
        fv := reflect.ValueOf(deepCopyFunc)
        ft := fv.Type()

        //verifyDeepCopyFunctionSignature函数负责检查deepCopyFunc是否是func(in interface{}, out interface{}, c *Cloner)error类型？
        if err:=verifyDeepCopyFunctionSignature(ft); err!=nil {
            return err
        }
        c.deepCopyFuncs[ft.In(0)] = fv
        return nil
    }

    //默认拷贝函数，实现字节slice之间的copy。
    func byteSliceDeepCopy(in *[]byte, out *[]byte, c *Cloner) error{
        if *in!=nil{
            *out = make([]byte, len(*ln))
            copy(*out, *in)
        }else{
            *out = nil
        }
        return nil
    }

### 2.2 NewConverter
含义：

    创建一个type转换器。

路径：

    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/conversion/converter.go

定义：

    func NewConverter(nameFn NameFunc) *Converter{
        c:=&Converter{
            conversionFuncs:          NewConversionFuncs(),
            generatedConversionFuncs: NewConversionFuncs(),
            ignoredConversions:       make(map[typePair]struct{}),
            nameFunc:                 nameFn,
            structFieldDests:         make(map[typeNamePair][]typeNamePair),
            structFieldSources:       make(map[typeNamePair][]byteNamePair),
            inputFieldMappingFuncs:   make(map[reflect.Type]FieldMappingFunc),
            inputDefaultFlags:        make(map[reflect.Type]FieldMatchingFlags),
        }
        c.RegisterConversionFunc(Convert_Slice_byte_To_Slice_byte)
        return c
    }
    type Converter struct{
        conversionFuncs          ConversionFuncs
        generatedConversionFuncs ConversionFuncs
        genericConversions       []GenericConversionFunc                 //正常转换过程中调用它提供一种"快速途径"，避免全部反射。这些函数不会被.Convert()外部调用。
        ignoredConversions       map[typePair]struct{}                   //no-op conversions
        structFieldDests         map[typeNamePair][]typeNamePair         //源type和name到目的type和name的映射。
        structFieldSources       map[typeNamePair][]typeNamePair         //目的type和name的映射到源type和name。
        inputFieldMappingFuncs   map[reflect.Type]FieldMappingFunc       //一个输入type到一个函数的映射，该函数能应用一个key name mapping。
        inputDefaultFlags        map[reflect.Type]FieldMatchingFlags     //一个输入type到一组默认conversion flags的映射。
        Debug                    DebugLogger                             //打印调试信息，退出。
        nameFunc                 func(t reflect.Type) string             //获得Type名字，这个名字用来决定两种type是否匹配。默认返回go type name。
    }

    //conversion pair到conversion函数的映射
    type ConversionFuncs struct{
        fns map[typePair]reflect.Value
    }
    type typePair struct{
        source reflect.Type
        dest reflect.Type
    }
    type GenericConversionFunc func(a, b interface{}, scope Scope) (bool, error)
    type Scope interface{
        Convert(src, dest interface{}, flags FieldMatchingFlags) error
        DefaultConvert(src, dest interface{}, flags FieldMatchingFlags) error
        SrcTag() reflect.StructTag
        DestTag() reflect.StructTag
        Flags() FieldMatchingFlags
        Meta() *Meta
    }
    type typeNamePair struct{
        fieldType reflect.Type
        fieldName string
    }
    type FieldMappingFunc func(key string, sourceTag, destTag reflect.StructTag)(source string, dest string)
    type FieldMatchingFlags int

    func NewConversionFuncs() ConversionFuncs{
        return ConversionFuncs{fns: make(map[typePair]reflect.Value)}
    }

    //注册conversion函数，即。该函数是func(src, dest interface{}, scope Scope)error类型。
    func (c *Converter)RegisterConversionFunc(conversionFunc interface{}) error{
        return c.conversionFuncs.Add(conversionFunc)
    }
    //默认conversion函数，字节slice之间的copy。
    func Convert_Slice_byte_To_Slice_byte(in *[]byte, out *[]byte, s Scope) error{
        if *in==nil{
            *out=nil
            return nil
        }
        *out=make([]byte, len(*len))
        copy(*out, *in)
        return nil
    }

    //k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/runtime/scheme.go
    func (s *Scheme)nameFunc(t reflect.Type)string{
        gvks,ok:=s.typeToGVK[t]
        if !ok{
            return t.Name()
        }
        for _, gvk:=range gvks{
            internalGV:=gvk.GroupVersion()
            internalGV.Version = "__internal"
            internalGVK:=internalGV.WithKind(gvk.Kind)

            if internalType, exists:=s.gvkToType[internalGVK'; exists{
                return s.typeToGVK[internalType][0].Kind
            }
        }
        return gvks[0].Kind
    }

### 2.3 AddConversionFuncs
含义：

    增加conversion函数。

路径：

    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/runtime/scheme.go

定义：

    func (s *Scheme)AddConversionFuncs(conversionFuncs ...interface{})error{
        for _,f:=range conversionFuncs{
            if err:=s.converter.RegisterConversionFunc(f);err!=nil{
                return err
            }
        }
        return nil
    }
    
    //增加conversion函数：
    func DefaultEmbeddedConversions() \[\]interface{}{
        return []interface{}{
            Convert_runtime_Object_To_runtime_RawExtension,
            Convert_runtime_RawExtension_To_runtime_Object,
        }
    }
    var DefaultStringConversions = []string{}{
        Convert_Slice_string_To_string,
        Convert_Slice_string_To_int,
        Convert_Slice_string_To_bool,
        Convert_Slice_string_To_int64,
    }

### 2.4 AddStructFieldConversion
定义：

    func (s *Scheme) RegisterInputDefaults(in interface{}, fn conversion.FieldMappingFunc, defaultFlags conversion.FieldMatchingFlags)error{
        return s.converter.RegisterInputDefaults(in, in, defaultFlags)
    }



## 参考文献
* [[resourceversion分析]](./resourceversion.md)