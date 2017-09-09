kube-scheduler运行
======================================================================
## 简介
启动运行一个指定的SchedulerServer，不会退出。

## Run
定义：

    启动运行一个指定的SchedulerServer，不会退出。

路径：

    k8s.io/kubernetes/plugin/cmd/kube-scheduler/app/server.go

定义：

    func Run(s *options.SchedulerServer) error {
        //创建客户端，相对于apiserver。客户端参数有3块：Host，BearearToken和TLSClientConfig。具体内容下面会介绍。客户端的作用就是访问apiserver。
        kubecli, err := createClient(s)
        if err!=nil {
            return fmt.Errorf("unable to create kube client: %v", err)
        }

        //创建事件记录器。事件记录器接收来自事件源的事件，并将事件发送给事件广播，事件广播再将事件发送给日志处理器、自己都有的事件接收器。事件广播包含一个监听器，监听器channel大小默认是1000，事件广播的广播周期默认是10s。
        recorder := createRecorder(kubecli, s)

        //创建共享信息库。apiserver中已经介绍了。这里共享信息库的主要作用是把与scheduler有关的对象信息添加进来，以便scheduler操作。
        informerFactory := informers.NewSharedInformerFactory(kubecli, 0)

        //创建pod的信息。这里的pod不包括静态pod。
        podInformer := factory.NewPodInformer(kubecli, 0)

        //创建调度器。
        sched, err:=CrateScheduler(
            s,                                                       //SchedulerServer配置
            kubecli,                                                 //访问apiserver的客户端
            informerFactory.Core().V1().Nodes(),                     //Node信息
            podInformer,                                             //Pod信息
            informerFactory.Core().V1().PersistentVolumes(),         //PV信息
            informerFactory.Core().V1().PersistentVolumeClaims(),    //PVC信息
            informerFactory.Core().V1().ReplicationControllers(),    //ReplicationControllers信息
            informerFactory.Extensions().V1beta1().ReplicaSets(),    //ReplicaSets信息
            informerFactory.Apps().V1beta1().StatefulSets(),         //状态信息
            informerFactory.Core().V1().Services(),                  //service信息
            recorder,                                                //事件记录器
        )
        if err!=nil{
            return fmt.Errorf("error creating scheduler: %v", err)
        }
        
        //启动http server
        go startHTTP(s)

        stop:=make(chan struct{})
        defer close(stop)
        //启动podInformer
        go podInformer.Informer().Run(stop)
        //启动其他informer
        informerFactory.Start(stop)

        //调度前要同步cache。
        informerFactory.WaitForCacheSync(stop)
        controller.WaitForCacheSync("scheduler", stop, podInformer.Informer().HasSynced)

        //运行调度器
        run:=func(_ <-chan struct{}) {
            sched.Run()
            select{}
        }

        if !s.LeaderElection.LeaderElect{
            run(nil)
            panic("unreachable")
        }
        id,err:=os.Hostname()
        if err!=nil{
            return fmt.Errorf("unable to get hostname: %v", err)
        }
        rl,err:=resourcelock.New(s.LeaderElection.ResourceLock,
            s.LockObjectNamespace,
            s.LockObjectName,
            kubecli,
            resourcelock.ResourceLockConfig{
                Identity: id,
                EventRecorder: recorder,
            })
        if err!=nil{
            glog.Fatalf("error creating lock: %v", err)
        }

        leaderelection.RunOrDie(leaderelection.LeaderElectionConfig{
            Lock: rl,
            LeaseDuration: s.LeaderElection.LeaseDuration.Duration,
            RenewDeadline: s.LeaderElection.RenewDeadline.Duration,
            RetryPeriod: s.LeaderElection.RetryPeriod.Duration,
            Callbacks: leaderelection.LeaderCallbacks{
                OnStartedLeading: run,
                OnStoppedLeading: func(){
                    glog.Fatalf("lost master")
                },
            },
        })
        panic("unreachable")
    }

### createClient
含义：

    创建客户端。客户端是相对apiserver来说的。

路径：

    k8s.io/kubernetes/plugin/cmd/kube-scheduler/app/configurator.go

定义：

    func createClient(s *options.SchedulerServer) (*clientset.Clientset, error) {
        //根据master和kubeconfig参数创建一个客户端配置。
        kubeconfig, err:=clientcmd.BuildConfigFromFlags(s.Master, s.Kubeconfig)
        if err!=nil{
            return nil, fmt.Errorf("unable to build config from flags: %v", err)
        }

        //设置客户端配置ContentType、QPS、Burst。
        kubeconfig.ContentType = s.ContentType
        kubeconfig.QPS = s.KubeAPIQPS
        kubeconfig.Burst = int(s.KubeAPIBurst)

        //创建客户端，这个方法在apiserver中已经介绍过了，这里的客户端是带版本信息的。
        cli,err:=clientset.NewForConfig(restclient.AddUserAgent(kubeconfig, "leader-election"))
        if err!=nil{
            return nil, fmt.Errorf("invalid API configuration: %v", err)
        }
        return cli, nil
    }

(1)clientcmd.BuildConfigFromFlags
含义：

    根据master和kubeconfig参数创建一个客户端配置。

路径：

    k8s.io/kubernetes/vendor/k8s.io/client-go/tools/clientcmd/client_config.go

定义：

    func BuildConfigFromFlags(masterUrl, kubeconfigPath string) (*restclient.Config, error){
        //如果kubeconfigPath和masterUrl都为空，则创建一个客户端配置。客户端配置信息包括：Host，BearerToken和TLSClientConfig。其中，Host是ip:port形式，ip和port分别取自环境变量"KUBERNETES_SERVICE_HOST"和"KUBERNETES_SERVICE_PORT"。BearerToken取自文件："/var/run/secrets/kubernetes.io/serviceaccount/token"。TLSClientConfig包含rootCAFile，取自文件："/var/run/secrets/kubernetes.io/serviceaccount/ca.crt"。
        if kubeconfigPath == "" && masterUrl == ""{
            glog.Warningf("Neither --kubeconfig nor --master was specified. Using the inClusterConfig. This might not work.")
            kubeconfig,err:=restclient.InClusterConfig()
            if err==nil{
                return kubeconfig,nil
            }
            glog.Warning("error creating inClusterConfig, failing back to default config: ", err)
        }
        return NewNonInteractiveDeferredLoadingClientConfig(
            &ClientConfigLoadingRules{ExplicitPath: kubeconfigPath},
            &ConfigOverrides{ClusterInfo: clientcmdapi.Cluster{Server: masterUrl}}).ClientConfig()
    }

### 创建事件记录
含义：

    创建事件记录。

路径：

    k8s.io/kubernetes/plugin/cmd/kube-scheduler/app/configurator.go

定义：

    func createRecoder(kubecli *clientset.Clientset, s *options.SchedulerServer) record.EventRecorder {
        //创建一个event广播。
        eventBroadcaster := record.NewBroadcaster()

        //将event广播接收到的事件写到日志处理器中，至于日志处理器是怎么操作，那是由日志处理器决定，比如，写文件或标准输出。
        eventBroadcaster.StartLogging(glog.Infof)

        //将event广播接收到的事件写到eventsink中。
        eventBroadcaster.StartRecordingToSink(&v1core.EventSinkImpl{Interface: v1core.New(kubecli.Core().RESTClient()).Events("")})

        //创建事件记录器。
        return eventBroadcaster.NewRecoder(api.Scheme, clientv1.EventSource{Component: s.SchedulerName})
    }

（1）record.NewBroadcaster()
含义：

    创建一个event广播。EventBroadcaster是一个event广播接口。该接口的方法实现接收事件，并发送事件给EventSink（事件接收器），watcher，log。

路径：

    k8s.io/kubernetes/vendor/k8s.io/client-go/tools/record/event.go

定义：

    func NewBroadcaster() EventBroadcaster {
        //创建事件广播，包含监听器和广播时间间隔。
        //maxQueuedEvents=1000，表示监听器watcher的channel中最大可放置事件的数量。
        //defaultSleepDuration=10s，表示广播时间间隔默认是10s。
        return &eventBroadcasterImpl{watch.NewBroadcaster(maxQueuedEvents, watch.DropIfChannelFull), defaultSleepDuration}
    }

（2）StartLogging
含义：

    将事件广播接收到的事件发送给日志处理器。eventBroadcasterImpl结构体实现了EventBroadcaster接口。

路径：

    k8s.io/kubernetes/vendor/k8s.io/client-go/tools/record/event.go

定义：

    func (eventBroadcaster *eventBroadcasterImpl) StartLogging (logf func(format string, args ...interface{})) watch.Interface {
        return eventBroadcaster.StartEventWatcher(
            //实际将事件信息写到日志中。
            func(e *v1.Event){
                logf("Event(%#v): type: '%v' reason: '%v' %v", e.InvolvedObject, e.Type, e.Reason, e.Message)
            })
    }

（3）StartRecordingToSink
含义：

    将event广播接收到的事件写到eventsink中。

路径：

    k8s.io/kubernetes/vendor/k8s.io/client-go/tools/record/event.go

参数：

    &v1.core.EventSinkImpl{Interface: v1core.New(kubecli.Core().RESTClient()).Event("")} 这是一个临时方案。

定义：

    func (eventBraodcaster *eventBroadcasterImpl) StartRecordingToSink(sink EventSink) watch.Interface {
        randGen := rand.New(rand.NewSource(time.Now().UnixNamo()))
        eventCorrelator := NewEventCorrelator(clock.RealClock{})
        return eventBroadcaster.StartEventWatcher(
            func(event *v1.Event){
                recordToSink(sink, event, eventCorrelator, randGen, eventBraodcaster.sleepDuration)
            })
    }

（4）NewRecoder
含义：

    创建事件记录器。记录给定事件源（Scheduler就是一个事件源）的事件。记录器会发生事件给event广播。

路径：

    k8s.io/kubernetes/vendor/k8s.io/client-go/tools/record/event.go

定义：

    func (eventBroadcaster *eventBroadcasterImpl) NewRecorder(scheme *runtime.Scheme, source v1.EventSource) EventRecorder {
        return &recorderImpl{scheme, source, eventBroadcaster.Broadcaster, clock.RealClock{}}
    }

### CreateScheduler
含义：

路径：

    k8s.io/kubernetes/plugin/cmd/kube-scheduler/app/configuration.go

定义：

    func CreateScheduler(
        s *options.SchedulerServer,
        kubecli *clientset.Clientset,
        nodeInformer coreinformers.NodeInformer,
        podInformer coreinformers.PodInformer,
        pvInformer coreinformers.PersistentVolumeInformer,
        pvcInformer coreinformers.PersistentVolumeClaimInformer,
	    replicationControllerInformer coreinformers.ReplicationControllerInformer,
	    replicaSetInformer extensionsinformers.ReplicaSetInformer,
	    statefulSetInformer appsinformers.StatefulSetInformer,
	    serviceInformer coreinformers.ServiceInformer,
	    recorder record.EventRecorder,
    )(*scheduler.Scheduler, error){
        configurator := factory.NewConfigFactory(
            s.SchedulerName,
            kubecli,
            nodeInformer,
            podInformer,
            pvInformer,
            pvcInformer,
            replicationControllerInformer,
            replicaSetInformer,
            statefulSetInformer,
            serviceInformer,
            s.HardPodAffinitySymmetricWeight,
        )

        //重新创建配置，添加PolicyConfigFile，AlgorithmProvider等等。
        configurator = &schedulerConfigurator{
            configurator,
            s.PolicyConfigFile,
            s.AlgorithmProvider,
            s.PolicyConfigMapName,
            s.PolicyConfigMapNamespace,
            s.UsaLegacyPolicyConfig,
        }

        return scheduler.NewFromConfigurator(configurator, func(cfg *scheduler.Config){
            cfg.Recorder = recoder
        })
    }

（1）NewConfigFactory
含义：

    初始化配置。

路径：

    k8s.io/kubernetes/plugin/pkg/scheduler/factory/factory.go

定义：

    func NewConfigFactory(
        schedulerName string,
        client clientset.Interface,
        nodeInformer coreinformers.NodeInformer,
        podInformer coreinformers.PodInformer,
        pvInformer coreinformers.PersistentVolumeInformer,
        pvcInformer coreinformers.PersistentVolumeClaimInformer,
	    replicationControllerInformer coreinformers.ReplicationControllerInformer,
	    replicaSetInformer extensionsinformers.ReplicaSetInformer,
	    statefulSetInformer appsinformers.StatefulSetInformer,
	    serviceInformer coreinformers.ServiceInformer,
	    hardPodAffinitySymmetricWeight int,
    )scheduler.Configurator {
        //创建scheduler缓存。
        stopEverything := make(chan struct{})
        schedulerCache := schedulercache.New(30*time.Second, stopEverything)

        c:=&ConfigFactory{
            client: client,                                                        //服务apiserver的客户端。
            podLister: schedulerCache,                                             //Pod监听器，创建一个缓存，当Pod失效期到了，则清理缓存。schedulerCache结构体有List方法来实现监听。
            podQueue: cache.NewFIFO(cache.MetaNamespaceKeyFunc),                   //创建FIFO对象，存储pod
            pVLister: pvInformer.Lister(),                                         //创建pv监听器，就是要实现PersistentVolumeLister接口的List和Get方法。
            pVCLister:                      pvcInformer.Lister(),                  //创建pv监听器，就是要实现PersistentVolumeClaimLister接口的List方法。
		    serviceLister:                  serviceInformer.Lister(),              //创建service监听器，要实现ServiceLister接口的List方法。
		    controllerLister:               replicationControllerInformer.Lister(),//创建rc监听器，要实现ReplicationControllerLister接口的List方法。
		    replicaSetLister:               replicaSetInformer.Lister(),           //创建ReplicaSet监听器，要实现ReplicaSetLister接口的List方法。
		    statefulSetLister:              statefulSetInformer.Lister(),          //创建StatefulSet监听器，要实现StatefulSetLister接口的List方法。
		    schedulerCache:                 schedulerCache,
		    StopEverything:                 stopEverything,
		    schedulerName:                  schedulerName,
		    hardPodAffinitySymmetricWeight: hardPodAffinitySymmetricWeight,
        }

        c.scheduledPodsHasSynced = podInformer.Informer().HasSynced

        //向podInformer中添加事件处理器，包括过滤器和资源处理器。以调度的pod cache。
        podInformer.Informer().AddEventHandler(
            cache.FilteringResourceEventHandler{
                FilterFunc: func(obj interface{})bool{
                    switch t:=obj.(type){
                    case *v1.Pod:
                        return assignedNonTerminatedPod(t)    //过滤器的功能是判断是否是Pod对象，如果是，则判断如果指定PodSpec.NodeName，则返回false，即调度器会调度到指定node上。再判断pod状态如果是PodSucceded或PodFailed（这两种都是终止状态），也返回false，即调度器不会调度非终止Pod。
                    default:
                        runtime.HandleError(fmt.Errorf("unable to handle object in %T: %T", c, obj))
                        return false
                    }
                },
                //处理器功能：把pod添加到cache，更新cache中的pod，删除cache中的pod。
                Handler: cache.ResourceEventHandlerFuncs{
                    AddFunc: c.addPodToCache,
                    UpdateFunc: c.updatePodInCache,
                    DeleteFunc: c.deletePodFromCache,
                },
            },
        )

        //未调度的pod cache
        podInformer.Informer().AddEventHandler(
            cache.FilteringResourceEventHandler{
                FileterFunc: func(obj interface{}) bool{
                    switch t:=obj.(type){
                    case *v1.Pod:
                        return unassignedNonTerminatedPod(t)              //和上面的一样
                    default:
                        runtime.HandleError(fmt.Errorf("unable to handle object in %T: %T", c, obj))
                        return false
                    }
                },
                Handler: cache.ResourceEventHandlerFuncs{
                    //把pod添加到podQueue中。
                    AddFunc: func(obj interface{}){
                        if err:=c.podQueue.Add(obj); err!=nil{
                            runtime.HandleError(fmt.Errorf("unable to queue %T: %v", obj, err))
                        }
                    },
                    //更新podQueue
                    UpdateFunc: func(oldObj, newObj interface{}){
                        if err:=c.podQueue.Update(newObj); err!=nil{
                            runtime.HandleError(fmt.Errorf("unable to update %T: %v", newObj, err))
                        }
                    },
                    //从podQueue中删除pod
                    DeleteFunc: func(obj interface{}) {
					    if err := c.podQueue.Delete(obj); err != nil {
						    runtime.HandleError(fmt.Errorf("unable to dequeue %T: %v", obj, err))
					    }
                    },
				},
            },
        )

        //调度pod的监听器
        c.scheduledPodLister = assignedPodLister{podInformer.Lister()}

        //为nodeInformer添加事件处理器。
        nodeInformer.Informer().AddEventHandlerWithResyncPeriod(
            cache.ResourceEventHandlerFuncs{
                AddFunc: c.addNodeToCache,             //添加node到缓存
                UpdateFunc: c.updateNodeInCache,       //更新缓存中node
                DeleteFunc: c.deleteNodeFromCache,     //从缓存中删除node
            },
            0,
        )
        c.nodeLister = nodeInformer.Lister()           //设置node监听器

        return c
    }

（a）schedulercache.New

含义：

    自动启动一个goroutine，管理pod的到期失效时间。ttl表示pod多久到期失效。stop是一个channel，表示关闭后台goroutine。

路径：

    k8s.io/kubernetes/plugin/pkg/scheduler/schedulercache/cache.go

定义：

    func New(ttl time.Duration, stop <-chan struct{}) Cache {
        //cleanAssumedPeriod=1s
        cache := newSchedulerCache(ttl, cleanAssumedPeriod, stop)

        //启动一个goroutine，处理pod到期后的操作，例如pod失效了，就清理pod，最后关闭goroutine。
        cache.run()
        return cache
    }

（b）NewFIFO

含义：

    创建一个FIFO队列形式的存储。

路径：

    k8s.io/kubernetes/vendor/k8s.io/client-go/tools/cache/fifo.go

定义：

    func NewFIFO(keyFunc keyFunc) *FIFO{
        f:=&FIFO{
            items: map[string]interface{}{},
            queue: []string{},
            keyFunc: keyFunc,
        }
        f.cond.L = &f.lock
        return f
    }

（2）NewFromConfigurator
含义：

    创建一个调度器。

路径：

    k8s.io/kubernetes/plugin/pkg/scheduler/scheduler.go

定义：

    func NewFromConfigurator(c Configurator, modifiers ...func(c *Config)) (*Scheduler, error){
        //创建配置，就是一个指向配置的指针。
        cfg, err:=c.Create()
        if err!=nil{
            return nil,err
        }

        //允许对配置进行修改，允许修改recorder。
        for _,modifier:=range modifies{
            modifier(cfg)
        }

        //创建调度器
        s:=&Scheduler{
            config: cfg,
        }

        //注册所有metrics。包括：E2eSchedulingLatency、SchedulingAlgorithmLatency、BindingLatency。
        metrics.Register()
        return s, nil
    }

### go startHTTP
含义：

    启动http server。提供profiling、healthz、metric、configz的API。

路径：

    k8s.io/kubernetes/plugin/cmd/kube-scheduler/app/server.go

定义：

    func startHTTP(s *options.SchedulerServer){
        //设置路由
        max:=http.NewServeMux()

        //设置健康检查路由，"/healthz"
        healthz.InstallHandler(mux)

        //设置各种profiling相关路由。
        if s.EnableProfiling{
            mux.HandlerFunc("/debug/pprof/", pprof.Index)
            mux.HandlerFunc("/debug/pprof/profile", pprof.Profile)
            mux.HandlerFunc("/debug/pprof/trace", pprof.Trace)
            if s.EnableContentionProfiling{
                goruntime.SetBlockProfileRate(1)
            }
        }

        //创建componentconfig配置。
        if c,err:=configz.New("componentconfig"); err==nil{
            c.Set(s.KubeSchedulerConfiguration)
        }else{
            glog.Errorf("unable to register configz: %s", err)
        }

        //设置configz路由，"/configz"
        configz.InstallHandler(mux)

        //设置metrics路由，"/metrics"
        mux.Handle("/metrics", prometheus.Handler())

        //创建http server
        server:=&http.Server{
            Addr: net.JoinHostPort(s.Address, strConv.Itoa(int(s.Port))),
            Handler: mux,
        }

        //启动http server
        glog.Fatal(server.ListenAndServe())
    }

### sched.Run
含义：

    运行调度器。启动goroutine立即返回。在调度器执行前必须同步cache。

路径：

    k8s.io/kubernetes/plugin/pkg/scheduler/scheduler.go

定义：

    func (sched *Scheduler)Run(){
        //等待cache同步
        if !sched.config.WaitForCacheSync(){
            return
        }

        //执行pod的调度过程。
        go wait.Until(sched.scheduleOne, 0, sched.config.StopEverything)
    }

（1）sched.scheduleOne
含义：

    调度pod的处理流程。kube-scheduler维护了一个FIFO类型的PodQueue缓存，新创建的Pod会被ConfigFactory监听到，被添加到PodQueue中，每次调度都从该PodQueue中取出Pod，执行调度流程。

路径：

    k8s.io/kubernetes/plugin/pkg/scheduler/scheduler.go

定义：

    func (sched *Scheduler) scheduleOne(){
        //获得pod做下面的处理。NextPod函数是一个阻塞函数，直到一个可用的pod到达才返回该pod。该函数用slice实现队列PodQueue，没有用channel，因为，调度过程很长，channel中pod状态可能会变成旧的。
        pod:=sched.config.NextPod()

        //如果pod被删除，并且设置了优雅关闭时间，则记录器会记录调度失败，并返回。
        if pod.DeletionTimestamp != nil{
            sched.config.Recorder.Eventf(pod, v1.EventTypeWarning, "FailedScheduling", "skip schedule deleting pod: %v/%v", pod.Namespace, pod.Name)
            glog.V(3).Infof("Skip schedule deleting pod: %v/%v", pod.Namespace, pod.Name)
            return
        }
        glog.V(3).Infof("Attempting to schedule pod: %v/%v", pod.Namespace, pod.Name)

        start:=time.Now()

        //执行调度算法，返回合适的host，即node。
        suggestedHost,err:=sched.schedule(pod)

        metrics.SchedulingAlgorithmLatency.Observe(metrics.SinceInMicroseconds(start))
        if err!=nil{
            return
        }

        err=sched.assume(pod, suggestedHost)
        if err!=nil{
            return
        }

        //如果调度成功，则将pod和node进行bind。并访问apiserver，创建Binding对象，记录到etcd中。
        go func(){
            err:=sched.bind(pod, &v1.Binding{
                ObjectMeta: metav1.ObjectMeta{Namespace: pod.Namespace, Name: pod.Name, UID: pod.UID},
                Target: v1.ObjectReference{
                    Kind: "Node",
                    Name: suggestedHost,
                },
            })
            metrics.E2eSchedulingLatency.Observe(metrics.SinceInMicroseconds(start))
            if err!=nil{
                glog.Errorf("Internal error binding pod: (%v)", err)
            }
        }()
    }

（a）sched.schedule
含义：

    执行调度算法，选择合适的node。这个是调度的核心。

路径：

    k8s.io/kubernetes/plugin/pkg/scheduler/scheduler.go

定义：

    func (sched *Scheduler) schedule(pod *v1.Pod)(string, error){
        //ConfigFactory监听到node作为参数传入调度函数。
        host,err:=sched.config.Algorithm.Schedule(pod, sched.config.NodeLister)
        if err!=nil{
            glog.V(1).Infof("Failed to schedule pod: %v/%v", pod.Namespace, pod.Name)
            copied, cerr:=api.Scheme.Copy(pod)
            if err!=nil{
                runtime.HandleError(err)
                return "",err
            }
            pod=copied.(*v1.Pod)
            sched.config.Error(pod,err)
            sched.config.Recorder.Eventf(pod, v1.EventTypeWarning, "FailedScheduling", "%v", err)
            sched.config.PodConditionUpdater.Update(pod, &v1.PodCondition{
                Type: v1.PodScheduled,
                Status: v1.ConditionFalse,
                Reason: v1.PodReasonUnschedulable,
                Message: err.Error(),
            })
            return "",err
        }
        return host,err
    }

**调度算法sched.config.Algorithm.Schedule会执行预选算法和优选算法，这个在后面的章节中介绍。**

_______________________________________________________________________
[[返回kube-scheduler/kube-scheduler.md]](./kube-scheduler.md) 

