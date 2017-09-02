远程调试配置
=================================================================
我使用调试k8s的IDE是：Visual Studio Code（vscode）。远程调试k8s，可以使用dlv命令，方法如下：
1.在远程Linux服务器和本地windows上各放置一份相同的kubernetes源码。

2.在两台机器上各自下载并安装dlv
     go get github.com/derekparker/delve/cmd/dlv

3.在远程Linux服务器上运行dlv服务器
     #dlv debug --headless --listen=:2345 --log
     注意：需要在包含main的.go文件所在目录下执行上面指令。例如：在/root/mygo/src/k8s.io/kubernetes/cmd/kube-apiserver下执行。

4.在本地vscode中编辑launch.json
![launch.png](/reference/remote-debug/launch.png/)

**注意**：
remotePath：指远程Linux服务器上kubernetes源码中main所在的.go文件
port：      指远程Linux服务器上dlv服务器的监听端口
host：      指远程Linux服务器ip
program：   指本地windows中kubernetes源码中main所在的.go文件
args：      指命令行参数

_______________________________________________________________________
[[返回reference/remote-debug.md]](/reference/remote-debug/remote-debug.md) 