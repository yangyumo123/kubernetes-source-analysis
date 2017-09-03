CreateDialer
=================================================================
## 简介
运行在云平台上，apiserver需要连接node，因此需要创建网络隧道。

## 1. CreateDialer
含义：

    如果运行在云平台上（例如：gce，aws），则需要创建nodeTunneler，安装本机SSH Key到kubernetes的所有node上。Tunneler会创建dialer，使用dialer连接kubelet。

路径：

    k8s.io/kubernetes/cmd/kube-apiserver/app/server.go

定义：

    func CreateDialer(s *options.ServerRunOptions) (tunnerler.Tunneler, *http.Transport, error){
        //如果运行在云平台上，则创建nodeTunneler（网络隧道）及Dialer（拨号器）；否则不创建。
        var nodeTunneler tunneler.Tunneler
        var proxyDialerFn utilnet.DialFunc

        //如果运行在云平台中，则需要安装本机的SSH Key到Kubernetes集群中所有node上。可以通过该SSHUser和SSHKeyFile，SSH到node上。
        if len(s.SSHUser)>0{
            //ssh密钥文件分发函数，负责将ssh的keyfile发送到各个node上。
            var installSSHKey tunneler.InstallSSHKey
            cloud, err:=cloudprovider.InitCloudProvider(s.CloudProvider.CloudProvider, s.CloudProvider.CloudConfigFile)
            if err!=nil{
                return nil,nil, fmt.Errorf("cloud provider could not be initialized: %v", err)
            }
            if cloud!=nil{
                if instances,supported:=cloud.Instances();supported{
                    installSSHKey = instances.AddSSHKeyToAllInstances
                }
            }
            if s.KubeletConfig.Port==0{
                return nil,nil, fmt.Errorf("must enable kubelet port if proxy ssh-tunneling is specified")
            }
            if s.KubeletConfig.ReadOnlyPort==0{
                return nil,nil, fmt.Errorf("must enable kubelet readonly port if proxy ssh-tunneling is specified")
            }

            //设置nodeTunneler
            healthCheckPath := &url.URL{
                Scheme:  "http",
                Host:    net.JoinHostPort("127.0.0.1", strconv.FormatUint(uint64(s.KubeletConfig.ReadOnlyPort), 10)),  //默认Host是：127.0.0.1:10255
                Path:    "healthz",
            }

            nodeTunneler          = tunneler.New(s.SSHUser, s.SSHKeyfile, healthCheckPath, installSSHKey)
            s.KubeletConfig.Dial  = nodeTunneler.Dial   //使用nodeTunneler的dialer来连接kubelet。
            proxyDialerFn         = nodeTunneler.Dial   //使用nodeTunnerler的dialer来连接pod，service和node。
        }

        proxyTLSClientConfig := &tls.Config{InsecureSkipVerify: true}   //到pod和service的代理是基于ip的。不期望能验证hostname。

        //Transport是http请求的载体，最终是调用Transport的RoudTrip(req *Request)(*Response,error)方法。关于Transport的介绍在其他文章中有解释。
        proxyTransport := utilnet.SetTransportDefaults(&http.Transport{
            Dial:              proxyDialerFn,
            TLSClientConfig:   proxyTLSClientConfig,
        })

        return nodeTunneler, proxyTransport, nil
    }

关于SSH Tunneler的相关知识请看参数文献[[SSH-Tunneler用法]](../../reference/k8s/ssh-tunneler.md)

下面详细介绍一下CrateDialer的执行。

## 1.1 Tunneler接口
含义：

    网络隧道接口。apiserver中只有当存在SSHUser时，才会创建Tunneler类型对象。

路径：

    k8s.io/kubernetes/pkg/master/tunneler/ssh.go

定义：

    type Tunneler interface {
        Run(AddressFunc)
        Stop()
        Dial(net, addr string) (net.Conn, error)
        SecondsSinceSync() int64
        SecondsSinceSSHKeySync() int64
    }

## 1.2 SSHTunneler结构体
含义：

    具体的ssh隧道，实现Tunneler接口。

路径：

    k8s.io/kubernetes/pkg/master/tunneler/ssh.go

定义：

    type SSHTunneler struct{
        lastSync            int64
        lastSSHKeySync      int64
        SSHUser             string                     //ssh user
        SSHKeyfile          string                     //ssh keyfile
        InstallSSHKey       InstallSSHKey
        HealthCheckURL      *url.URL                   //健康检查URL
        tunnels             *ssh.SSHTunnelList
        lastSyncMetric      prometheus.GaugeFunc
        clock               clock.Clock               
        getAddresses        AddressFunc
        stopChan            chan struct{}
    }
    type InstallSSHKey func(user string, data []byte)error
    type AddressFunc func()(addresses []string, err error)

    //Run建立隧道循环并返回。
    func (c *SSHTunneler)Run(getAddresses AddressFunc){
        if c.stopChan != nil{
            return
        }
        c.stopChan = make(chan struct{})
        if getAddresses!=nil{
            c.getAddresses = getAddresses
        }
        if len(c.SSHUser)>32{
            glog.Warning("SSH User is too long, truncating to 32 chars")
            c.SSHUser = c.SSHUser[0:32]
        }
        glog.Infof("Setting up proxy: %s %s", c.SSHUser, c.SSHKeyfile)

        publicKeyFile := c.SSHKeyfile + ".pub"
        exists,err    := util.FileExists(publicKeyFile)
        if err!=nil{
            glog.Errorf("Error detecting if key exists: %v", err)
        }else if !exists{
            glog.Infof("Key doesn't exist, attempting to create")
            if err := generateSSHKey(c.SSHKeyfile, publicKeyFile);err!=nil{
                glog.Errorf("Failed to create key pair: %v", err)
            }
        }
        c.tunnels        = ssh.NewSSHTunnelList(c.SSHUser,c.SSHKeyfile, c.HealthCheckURL, c.stopChan)
        c.lastSSHKeySync = c.clock.Now().Unix()
        c.installSSHKeySyncLoop(c.SSHUser, publicKeyFile)
        c.lastSync       = c.clock.Now().Unix()
        c.nodesSyncLoop()
    }

    //k8s.io/kubernetes/pkg/ssh/ssh.go
    func NewSSHTunnelList(user, keyfile string, healthCheckURL *url.URL, stopChan chan struct{}) *SSHTunnelList{
        l:=&SSHTunnelList{
            adding:         make(map[string]bool),
            tunnelCreator:  &realTunnelCreator{},
            user:           user,
            keyfile:        keyfile,
            healthCheckURL: healthCheckURL,
        }
        healthCheckPoll := 1*time.Minute
        go wait.Until(func(){
            l.tunnelsLock.Lock()
            defer l.tunnelsLock.Unlock()
            numTunnels:=len(l.enties)
            for i,entry:=range l.entries{
                delay:=healthCheckPoll * time.Duration(i) / time.Duration(numTunnels)
                l.delayedHealthCheck(entry, delay) 
            }
        }, healthChckPoll, stopChan)
        return l
    }

    //Stop优雅关闭隧道。
    func (c *SSHTunneler)Stop(){
        if c.stopChan != nil{
            close(c.stopChan)
            c.stopChan = nil
        }
    }

    //Dial拨号连接
    func (c *SSHTunneler)Dial(net, addr string) (net.Conn, error){
        return c.tunnels.Dial(net, addr)
    }
    //k8s.io/kubernetes/pkg/ssh/ssh.go
    func (l *SSHTunnelList) Dial (net,addr string)(net.Conn, error){
        start := time.Now()
        id:=mathrand.Int63()
        glog.Infof("[%x: %v] Dialing...", id, addr)
        defer func(){
            glog.Infof("[%x: %v] Dialed in %v.", id,addr,time.Now().Sub(strart))
        }()
        tunnel, err:=l.pickTunnel(strings.Split(addr, ":")[0])
        if err!=nil{
            return nil,err
        }
        return tunnel.Dial(net,addr)
    }

    //SecondsSinceSync
    func (c *SSHTunneler)SecondsSinceSync()int64{
        now:=c.clock.Now().Unix()
        then:=atomic.LoadInt64(&c.lastSync)
        return now-then
    }

    //SecondsSinceSSHKeySync表示key最后一次更新到现在的时间。
    func (c* SSHTunneler)SecondsSinceSSHKeySync()int64{
        now:=c.clock.Now().Unix()
        then:=atomic.LoadInt64(&c.lastSSHKeySync)
        return now-then
    }


## 1.3 创建SSHTunneler对象
含义：

    创建SSHTunneler对象。

路径：

    k8s.io/kubernetes/pkg/master/tunneler/ssh.go

定义：

    func New(sshUser, sshKeyfile string, healthCheckURL *url.URL, installSSHKey InstallSSHKey) Tunneler{
        return &SSHTunneler{
            SSHUser:             sshUser,
            SSHKeyfile:          sshKeyfile,
            InstallSSHKey:       installSSHKey,
            HealthCheckURL:      healthCheckURL,
            clock:               clock.RealClock{},
        }
    }

## 1.4 cloudprovider.InitCloudProvider
含义：

    创建一个云提供商实例。

路径：

    k8s.io/kubernetes/pkg/cloudprovider/plugins.go

定义：

    func InitCloudProvider(name string, configFilePath string)(Interface, error){
        var cloud Interface
        var err error
        if name==""{
            glog.Info("No cloud provider specified")
            return nil,nil
        }
        if IsExternal(name){
            glog.Info("External cloud provider specified")
            return nil,nil
        }
        if configFilePath!=""{
            var config *os.File
            config,err = os.Open(configFilePath)
            if err!=nil{
                glog.Fatalf("Couldn't open cloud provider configuration %s: %#v", configFilePath, err)
            }
            defer config.Close()
            cloud,err=GetCloudProvider(name, config)
        }else{
            cloud,err=GetCloudProvider(name,nil)
        }
        if err!=nil{
            return nil,fmt.Errorf("could not init cloud provider %q: %v", name,err)
        }
        if cloud==nil{
            return nil, fmt.Errorf("unknown cloud provider %q", name)
        }
        return cloud,nil
    }
    func GetCloudProvider(name string, config io.Reader) (Interface, error){
        providersMutex.Lock()
        defer providersMutex.Unlock()
        f,found:=providers[name]
        if !found{
            return nil, nil
        }
        return f(config)
    }


## 参考文献
* [[SSH-Tunneler用法]](../../reference/k8s/ssh-tunneler.md)

_______________________________________________________________________
[[返回/kube-apiserver/run/run.md]](./run.md) 
