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

kube-controller-manager和 kube-scheduler 都是依赖etcd实现分布式锁从而实现leader选举

所有节点上的组件请求各自apiserver，apiserver从etcd中抢占锁资源（configmap或endpoint或lease），抢到锁的节点组件会将自己标记成为锁的持有者。

&#x20;leader 则可以通过更新RenewTime来确保持续保有该锁。同时其它节点上的组件也会请求各节点上的apiserver，来查询加锁对象的更新时间来判断自己是否成为新的leader。

当leader在配置的时间内未能成功更新锁资源的时间，立即会失去leader身份

<figure><img src="../../.gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

### 核心逻辑

源码分析自 kubernetes 1.17 &#x20;

#### controller-manager组件初始化逻辑

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

#### controller-manager 运行逻辑

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

#### 创建 LeaderElector 对象，执行Run 方法开始选举

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
	//竞争锁资源，只有超时才返回fasle，
	if !le.acquire(ctx) {
		return // ctx signalled done
	}
	// 获取到锁后，才允许执行controller 启动逻辑
	ctx, cancel := context.WithCancel(ctx)
	defer cancel()
	go le.config.Callbacks.OnStartedLeading(ctx)
	le.renew(ctx)
}
```

#### acquire

使用golang wait库的 wait.JitterUntil方法，在给定间隔和抖动因子范围内周期执行

抖动因子 ---- 如果大于 0.0 间隔时间变为 duration 到 duration + maxFactor \* duration 的随机值

实现定时任务，在给定时间间隔内执行获取资源锁的 func， 除非收到stopCh信号，否则一直循环

```go
// k8s.io/client-go/leaderelection/leaderelection.go 235
// acquire loops calling tryAcquireOrRenew and returns true immediately when tryAcquireOrRenew succeeds.
// Returns false if ctx signals done.
func (le *LeaderElector) acquire(ctx context.Context) bool {
   ctx, cancel := context.WithCancel(ctx)
   defer cancel()
   succeeded := false
   desc := le.config.Lock.Describe()
   klog.Infof("attempting to acquire leader lease  %v...", desc)
   // 定时任务
   wait.JitterUntil(func() {
      // 获取资源锁
      succeeded = le.tryAcquireOrRenew()
      le.maybeReportTransition()
      // 抢占锁失败则直接返回，等待下个周期
      if !succeeded {
         klog.V(4).Infof("failed to acquire lease %v", desc)
         return
      }
      le.config.Lock.RecordEvent("became leader")
      le.metrics.leaderOn(le.config.Name)
      klog.Infof("successfully acquired lease %v", desc)
      cancel()
      // 间隔周期 默认为2s
      // 抖动因子 如果大于 0.0 间隔时间变为 duration 到 duration + maxFactor * duration 的随机值
      // siling 逻辑的执行时间是否不算入间隔时间
   }, le.config.RetryPeriod, JitterFactor, true, ctx.Done())
   return succeeded
}
```

#### tryAcquireOrRenew

tryAcquireOrRenew 函数尝试获取租约，如果获取不到或者得到的租约已过期则尝试抢占，否则 leader 不变。函数返回 True 说明本 goroutine 已成功抢占到锁，获得租约合同，成为 leader。

<pre class="language-go"><code class="lang-go">// k8s.io/client-go/leaderelection/leaderelection.go 322
<strong>// tryAcquireOrRenew tries to acquire a leader lease if it is not already acquired,
</strong>// else it tries to renew the lease if it has already been acquired. Returns true
// on success else returns false.
func (le *LeaderElector) tryAcquireOrRenew() bool {
   now := metav1.Now()
   leaderElectionRecord := rl.LeaderElectionRecord{
      // 锁持有者（leader）身份标识
      HolderIdentity:       le.config.Lock.Identity(),
      // 租约时长
      LeaseDurationSeconds: int(le.config.LeaseDuration / time.Second),
      RenewTime:            now,
      AcquireTime:          now,
   }

   // 1. obtain or create the ElectionRecord
   // le.config.Lock.Get()通过client-go 获取k8s resource状态（从etcd获取）
   oldLeaderElectionRecord, oldLeaderElectionRawRecord, err := le.config.Lock.Get()
   if err != nil {
      if !errors.IsNotFound(err) {
         klog.Errorf("error retrieving resource lock %v: %v", le.config.Lock.Describe(), err)
         return false
      }
      if err = le.config.Lock.Create(leaderElectionRecord); err != nil {
         klog.Errorf("error initially creating leader election record: %v", err)
         return false
      }
      le.observedRecord = leaderElectionRecord
      le.observedTime = le.clock.Now()
      return true
   }

   // 2. Record obtained, check the Identity &#x26; Time
   // 校验租约是否到期
   if !bytes.Equal(le.observedRawRecord, oldLeaderElectionRawRecord) {
      le.observedRecord = *oldLeaderElectionRecord
      le.observedRawRecord = oldLeaderElectionRawRecord
      le.observedTime = le.clock.Now()
   }
   if len(oldLeaderElectionRecord.HolderIdentity) > 0 &#x26;&#x26;
      le.observedTime.Add(le.config.LeaseDuration).After(now.Time) &#x26;&#x26;
      !le.IsLeader() {
      klog.V(4).Infof("lock is held by %v and has not yet expired", oldLeaderElectionRecord.HolderIdentity)
      return false
   }

   // 3. We're going to try to update. The leaderElectionRecord is set to it's default
   // here. Let's correct it before updating.
   if le.IsLeader() {
      leaderElectionRecord.AcquireTime = oldLeaderElectionRecord.AcquireTime
      leaderElectionRecord.LeaderTransitions = oldLeaderElectionRecord.LeaderTransitions
   } else {
      leaderElectionRecord.LeaderTransitions = oldLeaderElectionRecord.LeaderTransitions + 1
   }

   // update the lock itself
   // 更新资源锁
   if err = le.config.Lock.Update(leaderElectionRecord); err != nil {
      klog.Errorf("Failed to update lock: %v", err)
      return false
   }

   le.observedRecord = leaderElectionRecord
   le.observedTime = le.clock.Now()
   return true
}
</code></pre>

#### le.config.Lock

资源锁接口

接口实现包括 EndpointLock 和 ConfigMapLock

具体实现代码在k8s.io/client-go/leaderelection/resourcelock 包内

创建和更新资源锁，会在configmap或endpoint的annotation 更新 "control-plane.alpha.kubernetes.io/leader" 字段

```go
// Interface offers a common interface for locking on arbitrary
// resources used in leader election.  The Interface is used
// to hide the details on specific implementations in order to allow
// them to change over time.  This interface is strictly for use
// by the leaderelection code.
type Interface interface {
   // Get returns the LeaderElectionRecord
   Get() (*LeaderElectionRecord, []byte, error)

   // Create attempts to create a LeaderElectionRecord
   Create(ler LeaderElectionRecord) error

   // Update will update and existing LeaderElectionRecord
   Update(ler LeaderElectionRecord) error

   // RecordEvent is used to record events
   RecordEvent(string)

   // Identity will return the locks Identity
   Identity() string

   // Describe is used to convert details on current resource lock
   // into a string
   Describe() string
}
```

#### renew

获取到资源锁，成为leader之后，leader通过renew方法来更新租约，维持自身的leader状态

使用wait.Until 定时执行，没有抖动随机因子，因此leader 更新租约的间隔 < 节点获取锁的间隔，保证leader在正常情况下持续更新租约维持leader状态

```go
// k8s.io/client-go/leaderelection/leaderelection.go 259
// renew loops calling tryAcquireOrRenew and returns immediately when tryAcquireOrRenew fails or ctx signals done.
func (le *LeaderElector) renew(ctx context.Context) {
   ctx, cancel := context.WithCancel(ctx)
   defer cancel()
   // 周期执行renew
   wait.Until(func() {
      // 设置renew context 超时时间
      timeoutCtx, timeoutCancel := context.WithTimeout(ctx, le.config.RenewDeadline)
      defer timeoutCancel()
      // 在超时时间范围内周期执行 更新租约
      err := wait.PollImmediateUntil(le.config.RetryPeriod, func() (bool, error) {
         done := make(chan bool, 1)
         go func() {
            defer close(done)
            // 尝试更新租约
            done <- le.tryAcquireOrRenew()
         }()

         select {
         case <-timeoutCtx.Done():
            return false, fmt.Errorf("failed to tryAcquireOrRenew %s", timeoutCtx.Err())
         case result := <-done:
            return result, nil
         }
      }, timeoutCtx.Done())

      le.maybeReportTransition()
      desc := le.config.Lock.Describe()
      if err == nil {
         klog.V(5).Infof("successfully renewed lease %v", desc)
         return
      }
      le.config.Lock.RecordEvent("stopped leading")
      le.metrics.leaderOff(le.config.Name)
      klog.Infof("failed to renew lease %v: %v", desc, err)
      cancel()
   }, le.config.RetryPeriod, ctx.Done())

   // if we hold the lease, give it up
   if le.config.ReleaseOnCancel {
      le.release()
   }
}
```
