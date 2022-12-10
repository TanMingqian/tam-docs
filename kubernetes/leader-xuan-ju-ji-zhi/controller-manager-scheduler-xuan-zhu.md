# Controller Manager & Scheduler选主

## 选举的原因

在k8s的组件中，其中有kube-scheduler和kube-manager-controller两个组件是有leader选举的，这个选举机制是k8s对于这两个组件的高可用保障。

即正常情况下kube-scheduler或kube-manager-controller组件的多个副本只有一个是处于业务逻辑运行状态，其它副本则不断的尝试去获取锁，去竞争leader，直到自己成为leader。

如果正在运行的leader因某种原因导致当前进程退出，或者锁丢失，则由其它副本去竞争新的leader，获取leader继而执行业务逻辑。

## 选举的配置

* leader-elect-resource-namespace：选举过程中用于锁定的资源所在的namespace名称，默认为“kube-system”
* leader-elect-resource-name：选举过程中用于锁定的资源对象名称。
* leader-elect：true为开启选举
* leader-elect-lease-duration：资源锁租约观察时间，如果其它竞争者在该时间间隔过后发现leader没更新获取锁时间，则其它副本可以认为leader已经挂掉不参与工作了，将重新选举leader。
* leader-elect-renew-deadline：选举过程中在停止leading角色之前再次renew的时间间隔，既在该时间内没有更新则失去leader身份。
* leader-elect-retry-period：选举过程中获取leader角色和renew之间的时间间隔，既为其它副本获取锁的时间间隔(竞争leader)和leader更新间隔；默认是2s。
* leader-elect-resource-lock：选根据过程中使用哪种资源对象进行锁定操作。

## 选举的逻辑

所有节点上的组件请求各自apiserver，apiserver从etcd中抢占锁资源，抢到锁的节点组件会将自己标记成为锁的持有者。

&#x20;leader 则可以通过更新RenewTime来确保持续保有该锁。同时其它节点上的组件也会请求各节点上的apiserver，来查询加锁对象的更新时间来判断自己是否成为新的leader。

当leader在配置的时间内未能成功更新锁资源的时间，立即会失去leader身份

### 核心逻辑

源码分析自 kubernetes 1.17 &#x20;

controller-manager组件初始化逻辑

```go
// cmd/kube-controller-manager/app/options/options.go 392
// Config return a controller manager config objective
func (s KubeControllerManagerOptions) Config(allControllers []string, disabledByDefaultControllers []string) (*kubecontrollerconfig.Config, error) {
   if err := s.Validate(allControllers, disabledByDefaultControllers); err != nil {
      return nil, err
   }

   if err := s.SecureServing.MaybeDefaultWithSelfSignedCerts("localhost", nil, []net.IP{net.ParseIP("127.0.0.1")}); err != nil {
      return nil, fmt.Errorf("error creating self-signed certificates: %v", err)
   }

   kubeconfig, err := clientcmd.BuildConfigFromFlags(s.Master, s.Kubeconfig)
   if err != nil {
      return nil, err
   }
   kubeconfig.DisableCompression = true
   kubeconfig.ContentConfig.AcceptContentTypes = s.Generic.ClientConnection.AcceptContentTypes
   kubeconfig.ContentConfig.ContentType = s.Generic.ClientConnection.ContentType
   kubeconfig.QPS = s.Generic.ClientConnection.QPS
   kubeconfig.Burst = int(s.Generic.ClientConnection.Burst)

   client, err := clientset.NewForConfig(restclient.AddUserAgent(kubeconfig, KubeControllerManagerUserAgent))
   if err != nil {
      return nil, err
   }

   // shallow copy, do not modify the kubeconfig.Timeout.
   config := *kubeconfig
   config.Timeout = s.Generic.LeaderElection.RenewDeadline.Duration
   // 通过client-go 创建 用于选举的资源对象 client 
   leaderElectionClient := clientset.NewForConfigOrDie(restclient.AddUserAgent(&config, "leader-election"))

   eventRecorder := createRecorder(client, KubeControllerManagerUserAgent)

   c := &kubecontrollerconfig.Config{
      Client:               client,
      Kubeconfig:           kubeconfig,
      EventRecorder:        eventRecorder,
      LeaderElectionClient: leaderElectionClient,
   }
   if err := s.ApplyTo(c); err != nil {
      return nil, err
   }

   return c, nil
}
```

controller-manager 运行逻辑

调用`leaderelection.RunOrDie` 执行选举逻辑

```go
// cmd/kube-controller-manager/app/controller-manager.go 159
// Run runs the KubeControllerManagerOptions.  This should never exit.

import "k8s.io/client-go/tools/leaderelection"
func Run(c *config.CompletedConfig, stopCh <-chan struct{}) error {
   // ......
   // 核心启动函数，用于启动所有k8s资源对象的controller
   run := func(ctx context.Context) {
		rootClientBuilder := controller.SimpleControllerClientBuilder{
			ClientConfig: c.Kubeconfig,
		}
		var clientBuilder controller.ControllerClientBuilder
		if c.ComponentConfig.KubeCloudShared.UseServiceAccountCredentials {
			if len(c.ComponentConfig.SAController.ServiceAccountKeyFile) == 0 {
				// It's possible another controller process is creating the tokens for us.
				// If one isn't, we'll timeout and exit when our client builder is unable to create the tokens.
				klog.Warningf("--use-service-account-credentials was specified without providing a --service-account-private-key-file")
			}

			if shouldTurnOnDynamicClient(c.Client) {
				klog.V(1).Infof("using dynamic client builder")
				//Dynamic builder will use TokenRequest feature and refresh service account token periodically
				clientBuilder = controller.NewDynamicClientBuilder(
					restclient.AnonymousClientConfig(c.Kubeconfig),
					c.Client.CoreV1(),
					"kube-system")
			} else {
				klog.V(1).Infof("using legacy client builder")
				clientBuilder = controller.SAControllerClientBuilder{
					ClientConfig:         restclient.AnonymousClientConfig(c.Kubeconfig),
					CoreClient:           c.Client.CoreV1(),
					AuthenticationClient: c.Client.AuthenticationV1(),
					Namespace:            "kube-system",
				}
			}
		} else {
			clientBuilder = rootClientBuilder
		}
		controllerContext, err := CreateControllerContext(c, rootClientBuilder, clientBuilder, ctx.Done())
		if err != nil {
			klog.Fatalf("error building controller context: %v", err)
		}
		saTokenControllerInitFunc := serviceAccountTokenControllerStarter{rootClientBuilder: rootClientBuilder}.startServiceAccountTokenController

		if err := StartControllers(controllerContext, saTokenControllerInitFunc, NewControllerInitializers(controllerContext.LoopMode), unsecuredMux); err != nil {
			klog.Fatalf("error starting controllers: %v", err)
		}

		controllerContext.InformerFactory.Start(controllerContext.Stop)
		controllerContext.ObjectOrMetadataInformerFactory.Start(controllerContext.Stop)
		close(controllerContext.InformersStarted)

		select {}
	}
   // 引用client-go 库的leaderelection 执行选举逻辑，被选举为主节点后才真正执行启动controller
   leaderelection.RunOrDie(context.TODO(), leaderelection.LeaderElectionConfig{
      Lock:          rl,
      LeaseDuration: c.ComponentConfig.Generic.LeaderElection.LeaseDuration.Duration,
      RenewDeadline: c.ComponentConfig.Generic.LeaderElection.RenewDeadline.Duration,
      RetryPeriod:   c.ComponentConfig.Generic.LeaderElection.RetryPeriod.Duration,
      Callbacks: leaderelection.LeaderCallbacks{
         OnStartedLeading: run,
         OnStoppedLeading: func() {
            klog.Fatalf("leaderelection lost")
         },
      },
      WatchDog: electionChecker,
      Name:     "kube-controller-manager",
   })
   panic("unreachable")
}
```



```go
// k8s.io/client-go/leaderelection/leaderelection.go 213
// RunOrDie starts a client with the provided config or panics if the config
// fails to validate.
func RunOrDie(ctx context.Context, lec LeaderElectionConfig) {
   // 创建 LeaderElector 对象，校验参数
   le, err := NewLeaderElector(lec)
   if err != nil {
      panic(err)
   }
   if lec.WatchDog != nil {
      lec.WatchDog.SetLeaderElection(le)
   }
   le.Run(ctx)
}

// Run starts the leader election loop
func (le *LeaderElector) Run(ctx context.Context) {
	defer func() {
		runtime.HandleCrash()
		le.config.Callbacks.OnStoppedLeading()
	}()
	if !le.acquire(ctx) {
		return // ctx signalled done
	}
	ctx, cancel := context.WithCancel(ctx)
	defer cancel()
	go le.config.Callbacks.OnStartedLeading(ctx)
	le.renew(ctx)
}
```

\


