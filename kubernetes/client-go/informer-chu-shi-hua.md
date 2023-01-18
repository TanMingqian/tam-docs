# informer初始化

## SharedInformerFactory的初始化

```go
// k8s.io/client-go/informers/factory.go
type sharedInformerFactory struct {
   // 连接kubernetes client
   client           kubernetes.Interface
   namespace        string
   tweakListOptions internalinterfaces.TweakListOptionsFunc
   lock             sync.Mutex
   defaultResync    time.Duration
   customResync     map[reflect.Type]time.Duration
   
   // key 为 obj 资源类型，value为对应资源的sharedIndexInformer
   informers map[reflect.Type]cache.SharedIndexInformer
   // startedInformers is used for tracking which informers have been started.
   // This allows Start() to be called multiple times safely.
   startedInformers map[reflect.Type]bool
   // wg tracks how many goroutines were started.
   wg sync.WaitGroup
   // shuttingDown is true when Shutdown has been called. It may still be running
   // because it needs to wait for goroutines.
   shuttingDown bool
}
```

## informer初始化

<pre class="language-go"><code class="lang-go">// k8s.io/client-go/tools/cache/shared_informer.go
// 在SharedInformer接口基础上添加add and get Indexers能力
// SharedIndexInformer provides add and get Indexers ability based on SharedInformer.
type SharedIndexInformer interface {
   SharedInformer
   // AddIndexers add indexers to the informer before it starts.
   AddIndexers(indexers Indexers) error
   GetIndexer() Indexer
}

<strong>type SharedInformer interface {
</strong>	// AddEventHandler adds an event handler to the shared informer using the shared informer's resync
	// period.  Events to a single handler are delivered sequentially, but there is no coordination
	// between different handlers.
	// It returns a registration handle for the handler that can be used to remove
	// the handler again.
	AddEventHandler(handler ResourceEventHandler) (ResourceEventHandlerRegistration, error)
	// AddEventHandlerWithResyncPeriod adds an event handler to the
	// shared informer with the requested resync period; zero means
	// this handler does not care about resyncs.  The resync operation
	// consists of delivering to the handler an update notification
	// for every object in the informer's local cache; it does not add
	// any interactions with the authoritative storage.  Some
	// informers do no resyncs at all, not even for handlers added
	// with a non-zero resyncPeriod.  For an informer that does
	// resyncs, and for each handler that requests resyncs, that
	// informer develops a nominal resync period that is no shorter
	// than the requested period but may be longer.  The actual time
	// between any two resyncs may be longer than the nominal period
	// because the implementation takes time to do work and there may
	// be competing load and scheduling noise.
	// It returns a registration handle for the handler that can be used to remove
	// the handler again and an error if the handler cannot be added.
	AddEventHandlerWithResyncPeriod(handler ResourceEventHandler, resyncPeriod time.Duration) (ResourceEventHandlerRegistration, error)
	// RemoveEventHandler removes a formerly added event handler given by
	// its registration handle.
	// This function is guaranteed to be idempotent, and thread-safe.
	RemoveEventHandler(handle ResourceEventHandlerRegistration) error
	// GetStore returns the informer's local cache as a Store.
	GetStore() Store
	// GetController is deprecated, it does nothing useful
	GetController() Controller
	// Run starts and runs the shared informer, returning after it stops.
	// The informer will be stopped when stopCh is closed.
	Run(stopCh &#x3C;-chan struct{})
	// HasSynced returns true if the shared informer's store has been
	// informed by at least one full LIST of the authoritative state
	// of the informer's object collection.  This is unrelated to "resync".
	HasSynced() bool
	// LastSyncResourceVersion is the resource version observed when last synced with the underlying
	// store. The value returned is not synchronized with access to the underlying store and is not
	// thread-safe.
	LastSyncResourceVersion() string

	// The WatchErrorHandler is called whenever ListAndWatch drops the
	// connection with an error. After calling this handler, the informer
	// will backoff and retry.
	//
	// The default implementation looks at the error type and tries to log
	// the error message at an appropriate level.
	//
	// There's only one handler, so if you call this multiple times, last one
	// wins; calling after the informer has been started returns an error.
	//
	// The handler is intended for visibility, not to e.g. pause the consumers.
	// The handler should return quickly - any expensive processing should be
	// offloaded.
	SetWatchErrorHandler(handler WatchErrorHandler) error

	// The TransformFunc is called for each object which is about to be stored.
	//
	// This function is intended for you to take the opportunity to
	// remove, transform, or normalize fields. One use case is to strip unused
	// metadata fields out of objects to save on RAM cost.
	//
	// Must be set before starting the informer.
	//
	// Note: Since the object given to the handler may be already shared with
	//	other goroutines, it is advisable to copy the object being
	//  transform before mutating it at all and returning the copy to prevent
	//	data races.
	SetTransform(handler TransformFunc) error

	// IsStopped reports whether the informer has already been stopped.
	// Adding event handlers to already stopped informers is not possible.
	// An informer already stopped will never be started again.
	IsStopped() bool
}

// `*sharedIndexInformer` implements SharedIndexInformer and has three
// main components.  One is an indexed local cache, `indexer Indexer`.
// The second main component is a Controller that pulls
// objects/notifications using the ListerWatcher and pushes them into
// a DeltaFIFO --- whose knownObjects is the informer's local cache
// --- while concurrently Popping Deltas values from that fifo and
// processing them with `sharedIndexInformer::HandleDeltas`.  Each
// invocation of HandleDeltas, which is done with the fifo's lock
// held, processes each Delta in turn.  For each Delta this both
// updates the local cache and stuffs the relevant notification into
// the sharedProcessor.  The third main component is that
// sharedProcessor, which is responsible for relaying those
// notifications to each of the informer's clients.
type sharedIndexInformer struct {
	// Indexer本地缓存k8s资源对象，通过缓存获取资源对象，减少对apiserver的压力
	indexer    Indexer
	// Controller 从DeltaFIFO 中 pop Deltas出来处理，根据对象的变化更新Indexer中的本地缓存，并通知processor
	controller Controller
	// Processor根据对象的变化事件类型，调用相应的ResourceEventHandler来处理对象的变化
	processor             *sharedProcessor

	cacheMutationDetector MutationDetector
	// 从api-server获取资源对象，保存到DeltaFIFO中
	listerWatcher ListerWatcher

	// objectType is an example object of the type this informer is
	// expected to handle.  Only the type needs to be right, except
	// that when that is `unstructured.Unstructured` the object's
	// `"apiVersion"` and `"kind"` must also be right.
	objectType runtime.Object

	// resyncCheckPeriod is how often we want the reflector's resync timer to fire so it can call
	// shouldResync to check if any of our listeners need a resync.
	resyncCheckPeriod time.Duration
	// defaultEventHandlerResyncPeriod is the default resync period for any handlers added via
	// AddEventHandler (i.e. they don't specify one and just want to use the shared informer's default
	// value).
	defaultEventHandlerResyncPeriod time.Duration
	// clock allows for testability
	clock clock.Clock

	started, stopped bool
	startedLock      sync.Mutex

	// blockDeltas gives a way to stop all event distribution so that a late event handler
	// can safely join the shared informer.
	blockDeltas sync.Mutex

	// Called whenever the ListAndWatch drops the connection with an error.
	watchErrorHandler WatchErrorHandler

	transform TransformFunc
}

// 初始化SharedIndexInformer
func NewSharedIndexInformer(lw ListerWatcher, exampleObject runtime.Object, defaultEventHandlerResyncPeriod time.Duration, indexers Indexers) SharedIndexInformer {
	realClock := &#x26;clock.RealClock{}
	sharedIndexInformer := &#x26;sharedIndexInformer{
		processor:                       &#x26;sharedProcessor{clock: realClock},
		indexer:                         NewIndexer(DeletionHandlingMetaNamespaceKeyFunc, indexers),
		listerWatcher:                   lw,
		objectType:                      exampleObject,
		resyncCheckPeriod:               defaultEventHandlerResyncPeriod,
		defaultEventHandlerResyncPeriod: defaultEventHandlerResyncPeriod,
		cacheMutationDetector:           NewCacheMutationDetector(fmt.Sprintf("%T", exampleObject)),
		clock:                           realClock,
	}
	return sharedIndexInformer
}

// k8s.io/client-go/informers/core/v1/pod.go 
// 每个obj资源对象，通过cache包的NewSharedIndexInformer方法创建自己的informer
// NewPodInformer constructs a new informer for Pod type.
// Always prefer using an informer factory to get a shared informer instead of getting an independent
// one. This reduces memory footprint and number of connections to the server.
func NewPodInformer(client kubernetes.Interface, namespace string, resyncPeriod time.Duration, indexers cache.Indexers) cache.SharedIndexInformer {
	return NewFilteredPodInformer(client, namespace, resyncPeriod, indexers, nil)
}

// NewFilteredPodInformer constructs a new informer for Pod type.
// Always prefer using an informer factory to get a shared informer instead of getting an independent
// one. This reduces memory footprint and number of connections to the server.
func NewFilteredPodInformer(client kubernetes.Interface, namespace string, resyncPeriod time.Duration, indexers cache.Indexers, tweakListOptions internalinterfaces.TweakListOptionsFunc) cache.SharedIndexInformer {
	return cache.NewSharedIndexInformer(
		&#x26;cache.ListWatch{
			ListFunc: func(options metav1.ListOptions) (runtime.Object, error) {
				if tweakListOptions != nil {
					tweakListOptions(&#x26;options)
				}
				return client.CoreV1().Pods(namespace).List(context.TODO(), options)
			},
			WatchFunc: func(options metav1.ListOptions) (watch.Interface, error) {
				if tweakListOptions != nil {
					tweakListOptions(&#x26;options)
				}
				return client.CoreV1().Pods(namespace).Watch(context.TODO(), options)
			},
		},
		&#x26;corev1.Pod{},
		resyncPeriod,
		indexers,
	)
}
</code></pre>

## 启动sharedInformerFactory

sharedInformerFactory.Start方法启动里面所有informer

```go
// k8s.io/client-go/informers/factory.go
func (f *sharedInformerFactory) Start(stopCh <-chan struct{}) {
   f.lock.Lock()
   defer f.lock.Unlock()

   if f.shuttingDown {
      return
   }

   for informerType, informer := range f.informers {
      if !f.startedInformers[informerType] {
         f.wg.Add(1)
         // We need a new variable in each loop iteration,
         // otherwise the goroutine would use the loop variable
         // and that keeps changing.
         informer := informer
         go func() {
            defer f.wg.Done()
            informer.Run(stopCh)
         }()
         f.startedInformers[informerType] = true
      }
   }
}
```

## sharedIndexInformer.Run(核心)

sharedIndexInformer.Run用于启动informer，

主要逻辑为：&#x20;

（1）调用NewDeltaFIFO，初始化DeltaFIFO；&#x20;

（2）构建Config结构体

（3）调用New，利用Config结构体来初始化controller；&#x20;

（4）调用s.processor.run，启动processor；&#x20;

（5）调用s.controller.Run，启动controller；

```go
func (s *sharedIndexInformer) Run(stopCh <-chan struct{}) {
   defer utilruntime.HandleCrash()

   if s.HasStarted() {
      klog.Warningf("The sharedIndexInformer has started, run more than once is not allowed")
      return
   }
   // 初始化DelatFIFO
   fifo := NewDeltaFIFOWithOptions(DeltaFIFOOptions{
      KnownObjects:          s.indexer,
      EmitDeltaTypeReplaced: true,
   })
   
   // 构建Config结构体
   cfg := &Config{
      Queue:            fifo,
      ListerWatcher:    s.listerWatcher,
      ObjectType:       s.objectType,
      FullResyncPeriod: s.resyncCheckPeriod,
      RetryOnError:     false,
      ShouldResync:     s.processor.shouldResync,
      // pop 回调函数
      Process:           s.HandleDeltas,
      WatchErrorHandler: s.watchErrorHandler,
   }

   func() {
      s.startedLock.Lock()
      defer s.startedLock.Unlock()

      s.controller = New(cfg)
      s.controller.(*controller).clock = s.clock
      s.started = true
   }()

   // Separate stop channel because Processor should be stopped strictly after controller
   processorStopCh := make(chan struct{})
   var wg wait.Group
   defer wg.Wait()              // Wait for Processor to stop
   defer close(processorStopCh) // Tell Processor to stop
   wg.StartWithChannel(processorStopCh, s.cacheMutationDetector.Run)
   // 启动processor，启动listener.run与调用listener.pop
   wg.StartWithChannel(processorStopCh, s.processor.run)

   defer func() {
      s.startedLock.Lock()
      defer s.startedLock.Unlock()
      s.stopped = true // Don't want any new listeners
   }()
   // 启动controller，初始化并启动Reflector,调用c.processLoop,开始controller的核心处理
   s.controller.Run(stopCh)
}
```

## **controller启动**

①初始化并启动Reflector&#x20;

②c.processLoop是controller的核心处理，从DeltaFIFO中pop Deltas出来处理，根据对象的变化更新Indexer本地缓存，并通知Processor相关对象有变化事件发生；

```go
// Run begins processing items, and will continue until a value is sent down stopCh or it is closed.
// It's an error to call Run more than once.
// Run blocks; call via go.
func (c *controller) Run(stopCh <-chan struct{}) {
   defer utilruntime.HandleCrash()
   go func() {
      <-stopCh
      c.config.Queue.Close() 
      
   }()
   // 初始化Reflector
   r := NewReflector(
      c.config.ListerWatcher,
      c.config.ObjectType,
      c.config.Queue,
      c.config.FullResyncPeriod,
   )
   r.ShouldResync = c.config.ShouldResync
   r.WatchListPageSize = c.config.WatchListPageSize
   r.clock = c.clock
   if c.config.WatchErrorHandler != nil {
      r.watchErrorHandler = c.config.WatchErrorHandler
   }

   c.reflectorMutex.Lock()
   c.reflector = r
   c.reflectorMutex.Unlock()

   var wg wait.Group
   // 启动Reflector
   wg.StartWithChannel(stopCh, r.Run)
   // c.processLoop是controller的核心处理，从DeltaFIFO中pop Deltas出来处理，根据对象的变化更新Indexer本地缓存，并通知Processor相关对象有变化事件发生
   wait.Until(c.processLoop, time.Second, stopCh)
   wg.Wait()
}
```

