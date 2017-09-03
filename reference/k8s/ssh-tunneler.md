SSH-Tunneler用法
========================================================================
## 简介
SSH（secure shell）利用ssh tunnel进行端口转发，它在ssh连接上建立一个加密的隧道。创建ssh tunnel之后，可以突破一些网络限制，访问不能直接访问的资源。

ssh tunnel分为三种：本地（L）、远程（R）和动态（D）。下面举例说明，假设有一台sshserver机器，ip是10.221.0.1。

1. 本地Local（ssh -NfL，其中-Nf一起使用表示这个SSH连接只用来传数据，不执行远程操作）

        指令：ssh -L <local_port>:<remote_host>:<remote_port> <SSH hostname>
        ssh -NfL 10.221.0.1:1234:www.google.com:80 10.221.0.1
        此时，在浏览器键入：http://10.221.0.1:1234，就能看到google页面了。
        省略前面的ip，1234端口就仅仅绑定在localhost上，更安全：
        ssh -NfL 1234:www.google.com:80 10.221.0.1
        此时，在浏览器键入：http://localhost:1234才能访问。

        何时使用本地tunnel呢？比如，你在本地访问不了www.google.com，而有一台机器（例如：10.221.0.1）可以访问www.google.com，那么你就可以通过这台机器来访问。

2. 本地Local
        
        指令：ssh -L <local_port>:<localhost>:<local_port> <SSH hostname>
        ssh -NfL 8888:localhost:80 10.221.0.1
        此时，访问127.0.0.1:8888，就等于访问127.0.0.1:80端口。
        ssh -NfL 0.0.0.0:8888:localhost:80 10.221.0.1
        此时，只有访问本地8888端口，就等于访问127.0.0.1:80端口。

3. 远程Remote（ssh -NfR）
   
        指令：ssh -R <local port>:<remote host>:<remote port> <SSH hostname>
        在需要访问的内网机器上执行：ssh -NfR 1234:localhost:22 10.221.0.1
        登录到10.221.0.1机器，使用如下命令连接内网机器：ssh -p 1234 localhost
        这时就等登录到内网机器了。

        何时使用远程tunnel？比如，你在家访问不了公司内网的机器，可以事先在公司内网的机器上执行远程tunnel，连上一台公司外网的机器，在家就可以通过外网的那台机器来访问公司内网的机器了。

4. 动态Dynamic（ssh -NfD）-Socket代理
        
        指令：ssh -D port remoteHost
        何时使用socket代理呢？比如，因为防火墙，本地机器不能访问某些资源，但是远程ssh主机可以访问。你可以从本地ssh到远程主机上，这时你希望用远程主机做代理以方便本地的网络访问。
        ssh -NfD 1234 10.221.0.1
        此时，建立了一台socket代理机器，接着在浏览器上设置socket代理：地址是localhost，端口是1234，从此，你的访问都是加密的了，而且走的是远程主机，IP变成了远程主机的IP。

