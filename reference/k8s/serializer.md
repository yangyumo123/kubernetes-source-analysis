Serializer序列化器
=========================================================
## 简介
序列化器的作用是：编码和解码。编码是指把结构化对象（Go struct Object）转换为序列化格式对象（gvk对象或者内部无版本对象），解码是对应的反向操作。

## Serializer接口
含义：

    序列化的核心接口，所有序列化器必须实现该接口。

路径：

    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/runtime/interfaces.go

定义：

    type Serializer interface{
        Encoder                                   //编码器接口
        Decoder                                   //解码器接口
    }

1. Encoder接口

含义：

    编码器接口。将一个对象写入一个流中。

路径：

    k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/runtime/interfaces.go

定义：

    type Encoder interface{
        Encode(obj Object, w io.Writer) error     //如果没有定义conversion，或者版本不兼容，则返回错误。
    }

2. Decoder接口

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

