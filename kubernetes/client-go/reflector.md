# Reflector

Reflector的两个核心操作：

1 List\&Watch&#x20;

Reflector从kube-apiserver中list\&watch资源对象，然后将对象的变化包装成Delta并将其存到DeltaFIFO中

2.将对象的变化包装成Delta然后扔进DeltaFIFO&#x20;

Reflector首先通过List操作获取全量的资源对象数据，调用DeltaFIFO的Replace方法全量插入DeltaFIFO，然后后续通过Watch操作根据资源对象的变化类型相应的调用DeltaFIFO的Add、Update、Delete方法，将对象及其变化插入到DeltaFIFO中

### Reflector初始化与启动

**Reflector结构体**

```go
// k8s.io/client-go/tools/cache/reflector.go
// Reflector watches a specified resource and causes all changes to be reflected in the given store.
type Reflector struct {
   // name identifies this reflector. By default it will be a file:line if possible.
   name string

   // The name of the type we expect to place in the store. The name
   // will be the stringification of expectedGVK if provided, and the
   // stringification of expectedType otherwise. It is for display
   // only, and should not be used for parsing or comparison.
   expectedTypeName string
   // An example object of the type we expect to place in the store.
   // Only the type needs to be right, except that when that is
   // `unstructured.Unstructured` the object's `"apiVersion"` and
   // `"kind"` must also be right.
   // 放到Store中（即DeltaFIFO中）的对象类型
   expectedType reflect.Type
   // The GVK of the object we expect to place in the store if unstructured.
   expectedGVK *schema.GroupVersionKind
   // The destination to sync up with the watch source
   // store会赋值为DeltaFIFO
   store Store
   // listerWatcher is used to perform lists and watches.
   // 负责list和watch
   listerWatcher ListerWatcher

   // backoff manages backoff of ListWatch
   backoffManager wait.BackoffManager
   // initConnBackoffManager manages backoff the initial connection with the Watch call of ListAndWatch.
   initConnBackoffManager wait.BackoffManager
   // MaxInternalErrorRetryDuration defines how long we should retry internal errors returned by watch.
   MaxInternalErrorRetryDuration time.Duration

   resyncPeriod time.Duration
   // ShouldResync is invoked periodically and whenever it returns `true` the Store's Resync operation is invoked
   ShouldResync func() bool
   // clock allows tests to manipulate time
   clock clock.Clock
   // paginatedResult defines whether pagination should be forced for list calls.
   // It is set based on the result of the initial list call.
   paginatedResult bool
   // lastSyncResourceVersion is the resource version token last
   // observed when doing a sync with the underlying store
   // it is thread safe, but not synchronized with the underlying store
   lastSyncResourceVersion string
   // isLastSyncResourceVersionUnavailable is true if the previous list or watch request with
   // lastSyncResourceVersion failed with an "expired" or "too large resource version" error.
   isLastSyncResourceVersionUnavailable bool
   // lastSyncResourceVersionMutex guards read/write access to lastSyncResourceVersion
   lastSyncResourceVersionMutex sync.RWMutex
   // WatchListPageSize is the requested chunk size of initial and resync watch lists.
   // If unset, for consistent reads (RV="") or reads that opt-into arbitrarily old data
   // (RV="0") it will default to pager.PageSize, for the rest (RV != "" && RV != "0")
   // it will turn off pagination to allow serving them from watch cache.
   // NOTE: It should be used carefully as paginated lists are always served directly from
   // etcd, which is significantly less efficient and may lead to serious performance and
   // scalability problems.
   WatchListPageSize int64
   // Called whenever the ListAndWatch drops the connection with an error.
   watchErrorHandler WatchErrorHandler
}
```

**Reflector初始化**

<pre class="language-go"><code class="lang-go">// k8s.io/client-go/tools/cache/reflector.go
<strong>func NewReflector(lw ListerWatcher, expectedType interface{}, store Store, resyncPeriod time.Duration) *Reflector {
</strong>   return NewNamedReflector(naming.GetNameFromCallsite(internalPackages...), lw, expectedType, store, resyncPeriod)
}

// NewNamedReflector same as NewReflector, but with a specified name for logging
func NewNamedReflector(name string, lw ListerWatcher, expectedType interface{}, store Store, resyncPeriod time.Duration) *Reflector {
   realClock := &#x26;clock.RealClock{}
   r := &#x26;Reflector{
      name:          name,
      listerWatcher: lw,
      store:         store,
      // We used to make the call every 1sec (1 QPS), the goal here is to achieve ~98% traffic reduction when
      // API server is not healthy. With these parameters, backoff will stop at [30,60) sec interval which is
      // 0.22 QPS. If we don't backoff for 2min, assume API server is healthy and we reset the backoff.
      backoffManager:         wait.NewExponentialBackoffManager(800*time.Millisecond, 30*time.Second, 2*time.Minute, 2.0, 1.0, realClock),
      initConnBackoffManager: wait.NewExponentialBackoffManager(800*time.Millisecond, 30*time.Second, 2*time.Minute, 2.0, 1.0, realClock),
      resyncPeriod:           resyncPeriod,
      clock:                  realClock,
      watchErrorHandler:      WatchErrorHandler(DefaultWatchErrorHandler),
   }
   r.setExpectedType(expectedType)
   return r
}
</code></pre>

**ListerWatcher**

ListerWatcher 定义了Reflector最核心的两个方法，List和Watch，用于全量获取资源对象以及监控资源对象的变化

```go
// k8s.io/client-go/tools/cache/listwatch.go
type Lister interface {
    // List should return a list type object; the Items field will be extracted, and the
    // ResourceVersion field will be used to start the watch in the right place.
    List(options metav1.ListOptions) (runtime.Object, error)
}

type Watcher interface {
    // Watch should begin a watch at the specified version.
    Watch(options metav1.ListOptions) (watch.Interface, error)
}

type ListerWatcher interface {
    Lister
    Watcher
}
```

**ListWatch**

ListWatch实现了 ListerWatcher interface

```go
// k8s.io/client-go/tools/cache/listwatch.go
type ListFunc func(options metav1.ListOptions) (runtime.Object, error)

type WatchFunc func(options metav1.ListOptions) (watch.Interface, error)

type ListWatch struct {
    ListFunc  ListFunc
    WatchFunc WatchFunc
    // DisableChunking requests no chunking for this list watcher.
    DisableChunking bool
}
```

**ListWatch的初始化**\
在`NewDeploymentInformer`初始化Deployment对象的informer中，会初始化ListWatch并定义其ListFunc与WatchFunc

```go

// k8s.io/client-go/informers/apps/v1/deployment.go
// NewDeploymentInformer constructs a new informer for Deployment type.
// Always prefer using an informer factory to get a shared informer instead of getting an independent
// one. This reduces memory footprint and number of connections to the server.
func NewDeploymentInformer(client kubernetes.Interface, namespace string, resyncPeriod time.Duration, indexers cache.Indexers) cache.SharedIndexInformer {
   return NewFilteredDeploymentInformer(client, namespace, resyncPeriod, indexers, nil)
}

// NewFilteredDeploymentInformer constructs a new informer for Deployment type.
// Always prefer using an informer factory to get a shared informer instead of getting an independent
// one. This reduces memory footprint and number of connections to the server.
func NewFilteredDeploymentInformer(client kubernetes.Interface, namespace string, resyncPeriod time.Duration, indexers cache.Indexers, tweakListOptions internalinterfaces.TweakListOptionsFunc) cache.SharedIndexInformer {
   return cache.NewSharedIndexInformer(
      &cache.ListWatch{
         ListFunc: func(options metav1.ListOptions) (runtime.Object, error) {
            if tweakListOptions != nil {
               tweakListOptions(&options)
            }
            return client.AppsV1().Deployments(namespace).List(context.TODO(), options)
         },
         WatchFunc: func(options metav1.ListOptions) (watch.Interface, error) {
            if tweakListOptions != nil {
               tweakListOptions(&options)
            }
            return client.AppsV1().Deployments(namespace).Watch(context.TODO(), options)
         },
      },
      &appsv1.Deployment{},
      resyncPeriod,
      indexers,
   )
}
```

**Reflector启动**

其主要是循环调用r.ListAndWatch

```go
// k8s.io/client-go/tools/cache/reflector.go
func (r *Reflector) Run(stopCh <-chan struct{}) {
	klog.V(3).Infof("Starting reflector %s (%s) from %s", r.expectedTypeName, r.resyncPeriod, r.name)
	wait.BackoffUntil(func() {
		if err := r.ListAndWatch(stopCh); err != nil {
			r.watchErrorHandler(r, err)
		}
	}, r.backoffManager, true, stopCh)
	klog.V(3).Infof("Stopping reflector %s (%s) from %s", r.expectedTypeName, r.resyncPeriod, r.name)
}
```

### &#x20;Reflector核心处理方法

ListAndWatch的主要逻辑分为三大块：

1 List操作（只执行一次）：\
（1）调用r.listerWatcher.List方法，执行list操作，即获取全量的资源对象；\
（2）调用r.syncWith，调用r.store.Replace处理DeltaFIFO；

2 Resync操作（隔一段时间执行一次）；\
（1）判断是否需要执行Resync操作，即重新同步；\
（2）需要则调用r.store.Resync处理DeltaFIFO；

3 Watch操作\
（1）调用r.listerWatcher.Watch，开始监听操作；\
（2）调用r.watchHandler，处理watch操作返回来的chan；

```go
// k8s.io/client-go/tools/cache/reflector.go
// ListAndWatch first lists all items and get the resource version at the moment of call,
// and then use the resource version to watch.
// It returns error if ListAndWatch didn't even try to initialize watch.
func (r *Reflector) ListAndWatch(stopCh <-chan struct{}) error {
   klog.V(3).Infof("Listing and watching %v from %s", r.expectedTypeName, r.name)
   // 调用list方法，获取全量的资源对象
   err := r.list(stopCh)
   if err != nil {
      return err
   }
   // Resync操作，隔一段时间执行一次
   resyncerrc := make(chan error, 1)
   cancelCh := make(chan struct{})
   defer close(cancelCh)
   go func() {
      resyncCh, cleanup := r.resyncChan()
      defer func() {
         cleanup() // Call the last one written into cleanup
      }()
      for {
         select {
         case <-resyncCh:
         case <-stopCh:
            return
         case <-cancelCh:
            return
         }
         // 是否需要执行Resync操作，重新同步
         if r.ShouldResync == nil || r.ShouldResync() {
            klog.V(4).Infof("%s: forcing resync", r.name)
            // 调用r.store.Resync操作DeltaFIFO
            if err := r.store.Resync(); err != nil {
               resyncerrc <- err
               return
            }
         }
         cleanup()
         resyncCh, cleanup = r.resyncChan()
      }
   }()
   // watch操作
   retry := NewRetryWithDeadline(r.MaxInternalErrorRetryDuration, time.Minute, apierrors.IsInternalError, r.clock)
   for {
      // give the stopCh a chance to stop the loop, even in case of continue statements further down on errors
      select {
      case <-stopCh:
         return nil
      default:
      }

      timeoutSeconds := int64(minWatchTimeout.Seconds() * (rand.Float64() + 1.0))
      options := metav1.ListOptions{
         ResourceVersion: r.LastSyncResourceVersion(),
         // We want to avoid situations of hanging watchers. Stop any watchers that do not
         // receive any events within the timeout window.
         TimeoutSeconds: &timeoutSeconds,
         // To reduce load on kube-apiserver on watch restarts, you may enable watch bookmarks.
         // Reflector doesn't assume bookmarks are returned at all (if the server do not support
         // watch bookmarks, it will ignore this field).
         AllowWatchBookmarks: true,
      }

      // start the clock before sending the request, since some proxies won't flush headers until after the first watch event is sent
      start := r.clock.Now()
      // 调用r.listerWatcher.Watch 开始监听资源对象
      w, err := r.listerWatcher.Watch(options)
      if err != nil {
         // If this is "connection refused" error, it means that most likely apiserver is not responsive.
         // It doesn't make sense to re-list all objects because most likely we will be able to restart
         // watch where we ended.
         // If that's the case begin exponentially backing off and resend watch request.
         // Do the same for "429" errors.
         if utilnet.IsConnectionRefused(err) || apierrors.IsTooManyRequests(err) {
            <-r.initConnBackoffManager.Backoff().C()
            continue
         }
         return err
      }
      // 调用r.watchHandler,处理watch操作返回的channel，操作DeltaFIFO
      err = watchHandler(start, w, r.store, r.expectedType, r.expectedGVK, r.name, r.expectedTypeName, r.setLastSyncResourceVersion, r.clock, resyncerrc, stopCh)
      retry.After(err)
      if err != nil {
         if err != errorStopRequested {
            switch {
            case isExpiredError(err):
               // Don't set LastSyncResourceVersionUnavailable - LIST call with ResourceVersion=RV already
               // has a semantic that it returns data at least as fresh as provided RV.
               // So first try to LIST with setting RV to resource version of last observed object.
               klog.V(4).Infof("%s: watch of %v closed with: %v", r.name, r.expectedTypeName, err)
            case apierrors.IsTooManyRequests(err):
               klog.V(2).Infof("%s: watch of %v returned 429 - backing off", r.name, r.expectedTypeName)
               <-r.initConnBackoffManager.Backoff().C()
               continue
            case apierrors.IsInternalError(err) && retry.ShouldRetry():
               klog.V(2).Infof("%s: retrying watch of %v internal error: %v", r.name, r.expectedTypeName, err)
               continue
            default:
               klog.Warningf("%s: watch of %v ended with: %v", r.name, r.expectedTypeName, err)
            }
         }
         return nil
      }
   }
}
```

## watchHandler

r.watchHandler主要是处理watch操作返回来的结果，其主要逻辑为循环做以下操作，直至event事件处理完毕：&#x20;

（1）从watch操作返回来的chan中获取event事件；&#x20;

（2）chan关闭则退出loop；&#x20;

（3）区分watch.Added、watch.Modified、watch.Deleted三种类型的event事件，分别调用r.store.Add、r.store.Update、r.store.Delete做处理，具体关于r.store.xxx的方法分析，在后续对DeltaFIFO进行分析时再做具体的分析；&#x20;

（4）调用setLastSyncResourceVersion，为Reflector更新已被处理的最新资源对象的resourceVersion值；

```go
// watchHandler watches w and sets setLastSyncResourceVersion
func watchHandler(start time.Time,
   w watch.Interface,
   store Store,
   expectedType reflect.Type,
   expectedGVK *schema.GroupVersionKind,
   name string,
   expectedTypeName string,
   setLastSyncResourceVersion func(string),
   clock clock.Clock,
   errc chan error,
   stopCh <-chan struct{},
) error {
   eventCount := 0

   // Stopping the watcher should be idempotent and if we return from this function there's no way
   // we're coming back in with the same watch interface.
   defer w.Stop()

loop:
   for {
      select {
      case <-stopCh:
         return errorStopRequested
      case err := <-errc:
         return err
      case event, ok := <-w.ResultChan():
         if !ok {
            break loop
         }
         if event.Type == watch.Error {
            return apierrors.FromObject(event.Object)
         }
         if expectedType != nil {
            if e, a := expectedType, reflect.TypeOf(event.Object); e != a {
               utilruntime.HandleError(fmt.Errorf("%s: expected type %v, but watch event object had type %v", name, e, a))
               continue
            }
         }
         if expectedGVK != nil {
            if e, a := *expectedGVK, event.Object.GetObjectKind().GroupVersionKind(); e != a {
               utilruntime.HandleError(fmt.Errorf("%s: expected gvk %v, but watch event object had gvk %v", name, e, a))
               continue
            }
         }
         meta, err := meta.Accessor(event.Object)
         if err != nil {
            utilruntime.HandleError(fmt.Errorf("%s: unable to understand watch event %#v", name, event))
            continue
         }
         resourceVersion := meta.GetResourceVersion()
         // 根据watch channel里的事件类型，更新本地缓存（DeltaFIFO）
         switch event.Type {
         case watch.Added:
            err := store.Add(event.Object)
            if err != nil {
               utilruntime.HandleError(fmt.Errorf("%s: unable to add watch event object (%#v) to store: %v", name, event.Object, err))
            }
         case watch.Modified:
            err := store.Update(event.Object)
            if err != nil {
               utilruntime.HandleError(fmt.Errorf("%s: unable to update watch event object (%#v) to store: %v", name, event.Object, err))
            }
         case watch.Deleted:
            // TODO: Will any consumers need access to the "last known
            // state", which is passed in event.Object? If so, may need
            // to change this.
            err := store.Delete(event.Object)
            if err != nil {
               utilruntime.HandleError(fmt.Errorf("%s: unable to delete watch event object (%#v) from store: %v", name, event.Object, err))
            }
         case watch.Bookmark:
            // A `Bookmark` means watch has synced here, just update the resourceVersion
         default:
            utilruntime.HandleError(fmt.Errorf("%s: unable to understand watch event %#v", name, event))
         }
         更新RV值
         setLastSyncResourceVersion(resourceVersion)
         if rvu, ok := store.(ResourceVersionUpdater); ok {
            rvu.UpdateResourceVersion(resourceVersion)
         }
         eventCount++
      }
   }

   watchDuration := clock.Since(start)
   if watchDuration < 1*time.Second && eventCount == 0 {
      return fmt.Errorf("very short watch: %s: Unexpected watch close - watch lasted less than a second and no items received", name)
   }
   klog.V(4).Infof("%s: Watch close - %v total %v items received", name, expectedTypeName, eventCount)
   return nil
}
```

