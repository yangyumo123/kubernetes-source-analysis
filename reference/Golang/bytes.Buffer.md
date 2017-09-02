bytes.Buffer 源码及用法
====================================================================================================
## 包所在文件：src/bytes/buffer.go
    package bytes
    //下面的bytes.Buffer的主要源码。
    // Buffer是一个变长字节缓冲区，提供读写方法。零值表示即将被使用的空缓冲区。
    type Buffer struct{
        buf []byte                           // 缓冲区内容是：buf[off: len(buf)]
        off int                              // 从&buf[off]位置开始读, 从&buf[len(buf)]位置开始写，这样可以避免了读写冲突。
        bootstrap [64]byte                   // 初始使用的内存大小，避免多次分配内存。
        lastRead readOp                      // 最后一次读的操作，以便Unread*能正确工作。
    }
    type readOp int
    const(
        opRead readOp = -1                   //有读操作
        opInvalid     = 0                    //没有读操作
        opReadRune1   = 1                    //读1个rune
        opReadRune2   = 2                    //读2个rune
        opReadRune3   = 3                    //读3个rune
        opReadRune4   = 4                    //读4个rune
    )
    //写操作
    func (b *Buffer) Write(p []byte) (n int, err error){
        b.lastRead = opInvalid               //设置没有读操作标记，以便后面开始写操作。
        m:=b.grow(len(p))                    //如果缓冲区能存放下p，则不增加缓冲区长度；否则增加缓冲区长度以便存放下p。
        return copy(b.buf[m:],p), nil        //把p复制到缓冲区中。
    }
    //读操作
    func (b *Buffer) Read(p []byte) (n int, err error){
        b.lastRead = opInvalid               //设置没有读操作标记，以便后面开始读操作。
        if b.off >= len(b.buf){
            b.Truncate(0)
            if len(p)==0{
                return
            }
            return 0, io.EOF                 //如果读到末尾，则返回。
        }
        n=copy(p,b.buf[b.off:])              //读取数据到p中，并返回读取的n个字节数。
        b.off+=n                             //将缓冲区的off向后移动n个字节数。以便下次读。
        if n>0{
            b.lastRead = opRead              //设置本次读操作标记。
        }
        return
    }

## bytes.Buffer的使用：
1.创建Buffer缓冲器
    var b bytes.Buffer                       //直接定义一个Buffer变量，而不用初始化
    //b1:=new(bytes.Buffer)                  //直接使用new初始化，可以直接使用
2.写缓冲区
    b.Write([]byte("Hello "))                //可以直接使用

3.其他两种定义方式
    func NewBuffer(buf []byte) *Buffer
    func NewBufferString(s string) *Buffer

___________________________________________________________________________________________________
[[返回/reference/remote-debug.md]](/reference/remote-debug/debug-kube-apiserver.md) 