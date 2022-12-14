# Rancher 选主

rancher server 同样也是以controller 的形式运行在 k8s 上，底层选举逻辑与kube-controller-manager和kube-scheduler相同

```go
// github.com/rancher/rke2/pkg/rke2/rke2.go 51
func Server(clx *cli.Context, cfg Config) error {
   //....

   var leaderControllers rawServer.CustomControllers
   // .....
   return server.RunWithControllers(clx, leaderControllers, rawServer.CustomControllers{})
}
```

```go
// github.com/rancher/k3s/pkg/cli/server/server.go 45
func RunWithControllers(app *cli.Context, leaderControllers server.CustomControllers, controllers server.CustomControllers) error {
   if err := cmds.InitLogging(); err != nil {
      return err
   }
   return run(app, &cmds.ServerConfig, leaderControllers, controllers)
}

func run(app *cli.Context, cfg *cmds.Server, leaderControllers server.CustomControllers, controllers server.CustomControllers) error {
   // ....

   if err := server.StartServer(ctx, &serverConfig); err != nil {
      return err
   }

  // .....
   return agent.Run(ctx, agentConfig)
}
```



```go
// github.com/rancher/k3s/pkg/server/server.go 53
func StartServer(ctx context.Context, config *Config) error {
   if err := setupDataDirAndChdir(&config.ControlConfig); err != nil {
      return err
   }

   if err := setNoProxyEnv(&config.ControlConfig); err != nil {
      return err
   }

   if err := control.Server(ctx, &config.ControlConfig); err != nil {
      return errors.Wrap(err, "starting kubernetes")
   }

   config.ControlConfig.Runtime.Handler = router(ctx, config)

   if config.ControlConfig.DisableAPIServer {
      go setETCDLabelsAndAnnotations(ctx, config)
   } else {
      // 使用 k8s API Server
      go startOnAPIServerReady(ctx, config)
   }

   for _, hook := range config.StartupHooks {
      if err := hook(ctx, config.ControlConfig.Runtime.APIServerReady, config.ControlConfig.Runtime.KubeConfigAdmin); err != nil {
         return errors.Wrap(err, "startup hook")
      }
   }

   ip := net2.ParseIP(config.ControlConfig.BindAddress)
   if ip == nil {
      hostIP, err := net.ChooseHostInterface()
      if err == nil {
         ip = hostIP
      } else {
         ip = net2.ParseIP("127.0.0.1")
      }
   }

   if err := printTokens(ip.String(), &config.ControlConfig); err != nil {
      return err
   }

   return writeKubeConfig(config.ControlConfig.Runtime.ServerCA, config)
}

func startOnAPIServerReady(ctx context.Context, config *Config) {
   select {
   case <-ctx.Done():
      return
   case <-config.ControlConfig.Runtime.APIServerReady:
      // 启动rancher controllers
      if err := runControllers(ctx, config); err != nil {
         logrus.Fatalf("failed to start controllers: %v", err)
      }
   }
}

func runControllers(ctx context.Context, config *Config) error {
   controlConfig := &config.ControlConfig

   sc, err := NewContext(ctx, controlConfig.Runtime.KubeConfigAdmin)
   if err != nil {
      return err
   }

   if err := stageFiles(ctx, sc, controlConfig); err != nil {
      return err
   }

   // run migration before we set controlConfig.Runtime.Core
   if err := nodepassword.MigrateFile(
      sc.Core.Core().V1().Secret(),
      sc.Core.Core().V1().Node(),
      controlConfig.Runtime.NodePasswdFile); err != nil {
      logrus.Warn(errors.Wrapf(err, "error migrating node-password file"))
   }
   controlConfig.Runtime.Core = sc.Core

   if controlConfig.Runtime.ClusterControllerStart != nil {
      if err := controlConfig.Runtime.ClusterControllerStart(ctx); err != nil {
         return errors.Wrapf(err, "starting cluster controllers")
      }
   }

   for _, controller := range config.Controllers {
      if err := controller(ctx, sc); err != nil {
         return errors.Wrap(err, "controller")
      }
   }

   if err := sc.Start(ctx); err != nil {
      return err
   }

   start := func(ctx context.Context) {
      if err := coreControllers(ctx, sc, config); err != nil {
         panic(err)
      }
      for _, controller := range config.LeaderControllers {
         if err := controller(ctx, sc); err != nil {
            panic(errors.Wrap(err, "leader controller"))
         }
      }
      if err := sc.Start(ctx); err != nil {
         panic(err)
      }
   }

   go setControlPlaneRoleLabel(ctx, sc.Core.Core().V1().Node(), config)

   go setClusterDNSConfig(ctx, config, sc.Core.Core().V1().ConfigMap())

   if controlConfig.NoLeaderElect {
      go func() {
         start(ctx)
         <-ctx.Done()
         logrus.Fatal("controllers exited")
      }()
   } else {
      // 启动rancher选举
      go leader.RunOrDie(ctx, "", version.Program, sc.K8s, start)
   }

   return nil
}
```

在kube-system 命名空间创建configmap作为选举资源锁

引用client-go的leaderelection工具包，具体选举逻辑与controller-manager相同

```go
// github.com/rancher/wrangler/pkg/leader/leader.go 16
func RunOrDie(ctx context.Context, namespace, name string, client kubernetes.Interface, cb Callback) {
   if namespace == "" {
      namespace = "kube-system"
   }

   err := run(ctx, namespace, name, client, cb)
   if err != nil {
      logrus.Fatalf("Failed to start leader election for %s", name)
   }
   panic("Failed to start leader election for " + name)
}

func run(ctx context.Context, namespace, name string, client kubernetes.Interface, cb Callback) error {
   id, err := os.Hostname()
   if err != nil {
      return err
   }
   // 使用configmap 作为资源锁对象
   rl, err := resourcelock.New(resourcelock.ConfigMapsResourceLock,
      namespace,
      name,
      client.CoreV1(),
      client.CoordinationV1(),
      resourcelock.ResourceLockConfig{
         Identity: id,
      })
   if err != nil {
      logrus.Fatalf("error creating leader lock for %s: %v", name, err)
   }

   t := time.Second
   if dl := os.Getenv("CATTLE_DEV_MODE"); dl != "" {
      t = time.Hour
   }

   // 调用client-go 提供的leaderelcetion工具包
   leaderelection.RunOrDie(ctx, leaderelection.LeaderElectionConfig{
      Lock:          rl,
      LeaseDuration: 45 * t,
      RenewDeadline: 30 * t,
      RetryPeriod:   2 * t,
      Callbacks: leaderelection.LeaderCallbacks{
         OnStartedLeading: func(ctx context.Context) {
            go cb(ctx)
         },
         OnStoppedLeading: func() {
            logrus.Fatalf("leaderelection lost for %s", name)
         },
      },
      ReleaseOnCancel: true,
   })
   panic("unreachable")
}
```
