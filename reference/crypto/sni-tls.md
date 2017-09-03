SNI-TLS原理
=======================================================
## SNI介绍：
早期一台服务器只能有一个域名，随着服务器虚拟化技术的产生，一台服务器可以有多个虚拟主机，因此就需要多个域名。从而产生了SNI技术。SSL早期设计顺应经典的PKI设计，PKI认为一个服务器只为一个域名提供服务，从而一个服务器上也就只能使用一个证书。随着服务器对虚拟主机的支持，每个服务器通过相同的IP地址可以为很多域名提供服务，然而服务器端不知道客户端到底请求的是哪个域名下的服务。

SNI的设计目的是为了让服务器根据请求来决定为哪个域服务，这个信息通常从HTTP请求头获得。

### SSL/TLS握手：
1. Client -> Server : client_hello

        客户端发起请求，以明文传输请求信息，包含版本信息，加密套件候选列表，压缩算法候选列表，随机数，扩展字段等。
        版本信息（version）：SSLv2、SSLv3、TLSv1、TLSv1.1 TLSv1.2，当前不应该低于TLSv1。
        加密套件候选列表（cipher_suites）：认证算法（身份认证）、密钥交换算法（密钥协商）、对称加密算法（信息加密）和信息摘要（完整性验证）。
        压缩算法（compression methods）：用于后续信息压缩传输。
        随机数（random_C）：用于后续生成对称密钥。
        扩展字段（extensions）：SNI属于扩展字段。

2. server_hello + server_certificate + server_hello_done

        server_hello：服务端返回协商的信息结果，包括选择使用的协议版本，加密套件，压缩算法，随机数（random_S）等，其中随机数用于后续的密钥协商；
        server_certificate：服务端配置对应的证书链，用于身份验证与密钥交换；
        server_hello_done：通知客户端信息发送结束。

3. 证书校验

        客户端验证证书的合法性，如果验证通过才会进行后续通信，否则根据错误情况不同做出提示和操作，合法性验证包括如下：
        证书链的可信性；
        证书是否吊销，有两类方法离线CRL与在线OCSP，不同的客户端行为会不同；
        有效期，证书是否在有效时间范围；
        域名，核查证书域名是否与当前的访问域名匹配。

4. client_key_exchange + change_cipher_spec + encrypted_handshake_message

        client_key_exchange：合法性验证通过之后，客户端计算产生随机数字Pre-master，并用证书公钥加密，发送给服务器；
        此时客户端已经获取全部的计算协商密钥需要的信息：两个明文随机数random_C和random_S与自己计算产生的Pre-master，计算得到协商密钥；
        enc_key=Func(randmom_C, random_S, Pre-master)
        change_cipher_spec：客户端通知服务器后续的通信都采用协商的通信密钥和加密算法进行加密通信；
        encrypted_handshake-message：结合之前所有通信参数的hash值与其他相关信息生成一段数据，采用协商密钥session secret与算法进行加密，然后发送给服务器用于数据与握手验证；

5. change_cipher_spec + encrypted_handshake_message

        服务器用私钥解码加密Pre-master数据，基于之前交换的两个明文随机数random_C和random_S，计算得到协商密钥：enc_key=Func(random_C, random_S, Pre-master);
        计算之前所有接收信息的hash值，然后解密客户端发送的encrypted_handshake_message，验证数据和密钥正确性；
        change_cipher_spec：验证通过之后，服务器同样发送change_cipher_spec以告知客户端后续的通信都采用协商的密钥与算法进行加密通信；
        encrypted_handshake_message：服务器也结合所有当前的通信参数信息生成一段数据并采用协商密钥session secret与算法加密并发送到客户端；

6. 握手结束

        客户端计算所有接收信息的hash值，并采用协商密钥解密encrypted_handshake_message，验证服务器发送的数据和密钥，验证通过则握手完成；

7. 加密通信

        开始使用协商密钥与算法进行加密通信。

由以上过程可知，没有SNI的情况下，服务器无法预知客户端到底请求的是哪一个域名的服务。

### SNI应用：
SNI的TLS扩展通过发送虚拟域的名字作为TSL协商的一部分修正了这个问题，在client_hello阶段，通过SNI扩展，将域名信息提前告诉服务器，服务器根据域名取得对应的证书返回给客户端已完成校验过程。

curl：

     curl7.18.1+可以支持SNI，curl7.21.3支持-resolve参数，可以直接定位到IP地址进行访问，对于一个域名有多个部署节点的服务来说，这个参数可以定向的访问某个设备。
     基本语法：
     curl -k -I --resolve www.example.com:80:192.0.2.1 https://www.example.com/index.html
     
关于tls的介绍可以参见：https://github.com/k8sp/tls