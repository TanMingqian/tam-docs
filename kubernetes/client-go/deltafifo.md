# DeltaFIFO

DeltaFIFO，是一个FIFO是一个先进先出的队列，而Delta包含资源对象数据本身及其变化类型

Reflector往DeltaFIFO里面写入，Controller从DeltaFIFO从里面读取；

一个对象能算出一个唯一的object key，其对应着一个Deltas，所以一个对象对应着一个Deltas。

Delta有4种Type，分别是: Added、Updated、Deleted、Sync。针对同一个对象，可能有多个不同Type的Delta元素在Deltas中，表示对该对象做了不同的操作

## DeltaFIFO数据结构定义

items：是个map，key根据对象算出，value为Deltas类型；&#x20;

queue：存储对象obj的key的队列；

```go
// Delta is a member of Deltas (a list of Delta objects) which
// in its turn is the type stored by a DeltaFIFO. It tells you what
// change happened, and the object's state after* that change.
//
// [*] Unless the change is a deletion, and then you'll get the final
// state of the object before it was deleted.
type Delta struct {
	Type   DeltaType
	Object interface{}
}
// DeltaType is the type of a change (addition, deletion, etc)
type DeltaType string

// Change type definition
const (
   Added   DeltaType = "Added"
   Updated DeltaType = "Updated"
   Deleted DeltaType = "Deleted"
   // Replaced is emitted when we encountered watch errors and had to do a
   // relist. We don't know if the replaced object has changed.
   //
   // NOTE: Previous versions of DeltaFIFO would use Sync for Replace events
   // as well. Hence, Replaced is only emitted when the option
   // EmitDeltaTypeReplaced is true.
   Replaced DeltaType = "Replaced"
   // Sync is for synthetic events during a periodic resync.
   Sync DeltaType = "Sync"
)


type DeltaFIFO struct {
	// lock/cond protects access to 'items' and 'queue'.
	lock sync.RWMutex
	cond sync.Cond

	// `items` maps a key to a Deltas.
	// Each such Deltas has at least one Delta.
	items map[string]Deltas

	// `queue` maintains FIFO order of keys for consumption in Pop().
	// There are no duplicates in `queue`.
	// A key is in `queue` if and only if it is in `items`.
	queue []string

	// populated is true if the first batch of items inserted by Replace() has been populated
	// or Delete/Add/Update/AddIfNotPresent was called first.
	populated bool
	// initialPopulationCount is the number of items inserted by the first call of Replace()
	initialPopulationCount int

	// keyFunc is used to make the key used for queued item
	// insertion and retrieval, and should be deterministic.
	keyFunc KeyFunc

	// knownObjects list keys that are "known" --- affecting Delete(),
	// Replace(), and Resync()
	knownObjects KeyListerGetter

	// Used to indicate a queue is closed so a control loop can exit when a queue is empty.
	// Currently, not used to gate any of CRUD operations.
	closed bool

	// emitDeltaTypeReplaced is whether to emit the Replaced or Sync
	// DeltaType when Replace() is called (to preserve backwards compat).
	emitDeltaTypeReplaced bool
}
```

## DeltaFIFO初始化

初始化一个items和queue都为空的DeltaFIFO并返回。

```go
// NewDeltaFIFOWithOptions returns a Queue which can be used to process changes to
// items. See also the comment on DeltaFIFO.
func NewDeltaFIFOWithOptions(opts DeltaFIFOOptions) *DeltaFIFO {
   if opts.KeyFunction == nil {
      opts.KeyFunction = MetaNamespaceKeyFunc
   }

   f := &DeltaFIFO{
      items:        map[string]Deltas{},
      queue:        []string{},
      keyFunc:      opts.KeyFunction,
      knownObjects: opts.KnownObjects,

      emitDeltaTypeReplaced: opts.EmitDeltaTypeReplaced,
   }
   f.cond.L = &f.lock
   return f
}
```

## DeltaFIFO核心处理方法

Reflector的核心处理方法里有调用过几个方法，分别是r.store.Replace、r.store.Add、r.store.Update、r.store.Delete，Reflector里的r.store就是DeltaFIFO，方法就是DeltaFIFO的**Replace、Add、Update、Delete**方法



sharedIndexInformer.Run方法中调用NewDeltaFIFOWithOptions初始化了DeltaFIFO，随后将DeltaFIFO作为参数传入初始化Config

```go
func (s *sharedIndexInformer) Run(stopCh <-chan struct{}) {
   defer utilruntime.HandleCrash()

   if s.HasStarted() {
      klog.Warningf("The sharedIndexInformer has started, run more than once is not allowed")
      return
   }
   fifo := NewDeltaFIFOWithOptions(DeltaFIFOOptions{
      KnownObjects:          s.indexer,
      EmitDeltaTypeReplaced: true,
   })

   cfg := &Config{
      Queue:            fifo,
      ListerWatcher:    s.listerWatcher,
      ObjectType:       s.objectType,
      FullResyncPeriod: s.resyncCheckPeriod,
      RetryOnError:     false,
      ShouldResync:     s.processor.shouldResync,

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
   wg.StartWithChannel(processorStopCh, s.processor.run)

   defer func() {
      s.startedLock.Lock()
      defer s.startedLock.Unlock()
      s.stopped = true // Don't want any new listeners
   }()
   s.controller.Run(stopCh)
}
```

在controller的Run方法中，调用NewReflector初始化Reflector时，将之前的DeltaFIFO传入，赋值给Reflector的store属性，所以Reflector里的r.store其实就是DeltaFIFO

<pre class="language-go"><code class="lang-go">func (c *controller) Run(stopCh &#x3C;-chan struct{}) {
   defer utilruntime.HandleCrash()
   go func() {
      &#x3C;-stopCh
      c.config.Queue.Close()
   }()
   r := NewReflector(
      c.config.ListerWatcher,
      c.config.ObjectType,
      // 使用DeltaFIFO 作为 Reflector 的store参数
    <a data-footnote-ref href="#user-content-fn-1">  c.config.Queue,</a>
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

   wg.StartWithChannel(stopCh, r.Run)

   wait.Until(c.processLoop, time.Second, stopCh)
   wg.Wait()
}
</code></pre>

```go
func NewReflector(lw ListerWatcher, expectedType interface{}, store Store, resyncPeriod time.Duration) *Reflector {
   return NewNamedReflector(naming.GetNameFromCallsite(internalPackages...), lw, expectedType, store, resyncPeriod)
}

// NewNamedReflector same as NewReflector, but with a specified name for logging
func NewNamedReflector(name string, lw ListerWatcher, expectedType interface{}, store Store, resyncPeriod time.Duration) *Reflector {
   realClock := &clock.RealClock{}
   r := &Reflector{
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
```

## DeltaFIFO.Add

```go
// Add inserts an item, and puts it in the queue. The item is only enqueued
// if it doesn't already exist in the set.
func (f *DeltaFIFO) Add(obj interface{}) error {
   f.lock.Lock()
   defer f.lock.Unlock()
   f.populated = true
   /*
    调用f.queueActionLocked，
    操作DeltaFIFO中的queue与Deltas，
    根据对象key构造Added类型的新Delta追加到相应的Deltas中
   */
   return f.queueActionLocked(Added, obj)
}
// queueActionLocked负责操作DeltaFIFO中的queue与items，根据对象key构造新的Delta追加到对应的Deltas中
// queueActionLocked appends to the delta list for the object.
// Caller must lock first.
func (f *DeltaFIFO) queueActionLocked(actionType DeltaType, obj interface{}) error {
	// 计算对象的key
	id, err := f.KeyOf(obj)
	if err != nil {
		return KeyError{obj, err}
	}
	oldDeltas := f.items[id]
	// 构建新的Delta，追加到Deltas末尾
	newDeltas := append(oldDeltas, Delta{actionType, obj})
	// Delta 去重，去除重复的event
	newDeltas = dedupDeltas(newDeltas)

	if len(newDeltas) > 0 {
	        // 判断对象的key是否在queue中，不在则添加入queue
		if _, exists := f.items[id]; !exists {
			f.queue = append(f.queue, id)
		}
		// 更新items中的Deltas
		f.items[id] = newDeltas
		// 通知所有消费者解除阻塞
		f.cond.Broadcast()
	} else {
		// This never happens, because dedupDeltas never returns an empty list
		// when given a non-empty list (as it is here).
		// If somehow it happens anyway, deal with it but complain.
		if oldDeltas == nil {
			klog.Errorf("Impossible dedupDeltas for id=%q: oldDeltas=%#+v, obj=%#+v; ignoring", id, oldDeltas, obj)
			return nil
		}
		klog.Errorf("Impossible dedupDeltas for id=%q: oldDeltas=%#+v, obj=%#+v; breaking invariant by storing empty Deltas", id, oldDeltas, obj)
		f.items[id] = newDeltas
		return fmt.Errorf("Impossible dedupDeltas for id=%q: oldDeltas=%#+v, obj=%#+v; broke DeltaFIFO invariant by storing empty Deltas", id, oldDeltas, obj)
	}
	return nil
}
```

## DeltaFIFO.Upate

```go
func (f *DeltaFIFO) Update(obj interface{}) error {
    f.lock.Lock()
    defer f.lock.Unlock()
    f.populated = true
    return f.queueActionLocked(Updated, obj)
}
```

## **DeltaFIFO.Delete**

```go
func (f *DeltaFIFO) Delete(obj interface{}) error {
   id, err := f.KeyOf(obj)
   if err != nil {
      return KeyError{obj, err}
   }
   f.lock.Lock()
   defer f.lock.Unlock()
   f.populated = true
   if f.knownObjects == nil {
      if _, exists := f.items[id]; !exists {
         // items 中不存在的对象，直接return
         // Presumably, this was deleted when a relist happened.
         // Don't provide a second report of the same deletion.
         return nil
      }
   } else {
      // We only want to skip the "deletion" action if the object doesn't
      // exist in knownObjects and it doesn't have corresponding item in items.
      // Note that even if there is a "deletion" action in items, we can ignore it,
      // because it will be deduped automatically in "queueActionLocked"
      _, exists, err := f.knownObjects.GetByKey(id)
      _, itemsExist := f.items[id]
      if err == nil && !exists && !itemsExist {
         // Presumably, this was deleted when a relist happened.
         // Don't provide a second report of the same deletion.
         return nil
      }
   }

   // exist in items and/or KnownObjects
   return f.queueActionLocked(Deleted, obj)
}
```

## DeltaFIFO.Replace

```go
func (f *DeltaFIFO) Replace(list []interface{}, _ string) error {
   f.lock.Lock()
   defer f.lock.Unlock()
   keys := make(sets.String, len(list))

   // keep backwards compat for old clients
   action := Sync
   if f.emitDeltaTypeReplaced {
      action = Replaced
   }

   // Add Sync/Replaced action for each new item.
   for _, item := range list {
      key, err := f.KeyOf(item)
      if err != nil {
         return KeyError{item, err}
      }
      keys.Insert(key)
      if err := f.queueActionLocked(action, item); err != nil {
         return fmt.Errorf("couldn't enqueue object: %v", err)
      }
   }

   if f.knownObjects == nil {
      // Do deletion detection against our own list.
      queuedDeletions := 0
      for k, oldItem := range f.items {
         if keys.Has(k) {
            continue
         }
         // Delete pre-existing items not in the new list.
         // This could happen if watch deletion event was missed while
         // disconnected from apiserver.
         var deletedObj interface{}
         if n := oldItem.Newest(); n != nil {
            deletedObj = n.Object
         }
         queuedDeletions++
         if err := f.queueActionLocked(Deleted, DeletedFinalStateUnknown{k, deletedObj}); err != nil {
            return err
         }
      }

      if !f.populated {
         f.populated = true
         // While there shouldn't be any queued deletions in the initial
         // population of the queue, it's better to be on the safe side.
         f.initialPopulationCount = keys.Len() + queuedDeletions
      }

      return nil
   }

   // Detect deletions not already in the queue.
   knownKeys := f.knownObjects.ListKeys()
   queuedDeletions := 0
   for _, k := range knownKeys {
      if keys.Has(k) {
         continue
      }

      deletedObj, exists, err := f.knownObjects.GetByKey(k)
      if err != nil {
         deletedObj = nil
         klog.Errorf("Unexpected error %v during lookup of key %v, placing DeleteFinalStateUnknown marker without object", err, k)
      } else if !exists {
         deletedObj = nil
         klog.Infof("Key %v does not exist in known objects store, placing DeleteFinalStateUnknown marker without object", k)
      }
      queuedDeletions++
      if err := f.queueActionLocked(Deleted, DeletedFinalStateUnknown{k, deletedObj}); err != nil {
         return err
      }
   }

   if !f.populated {
      f.populated = true
      f.initialPopulationCount = keys.Len() + queuedDeletions
   }

   return nil
}
```

DeltaFIFO.Pop

DeltaFIFO的Pop操作，从items中弹出Deltas给回调函数PopProcessFunc进行处理，此处回调函数是sharedIndexInformer.HandleDeltas

```go
func (f *DeltaFIFO) Pop(process PopProcessFunc) (interface{}, error) {
   f.lock.Lock()
   defer f.lock.Unlock()
   // 1.循环判断queue的长度是否为0，为0则阻塞住，调用f.cond.Wait()，等待通知（与queueActionLocked方法中的f.cond.Broadcast()相对应，即queue中有对象key则发起通知）
   for {
      for len(f.queue) == 0 {
         // When the queue is empty, invocation of Pop() is blocked until new item is enqueued.
         // When Close() is called, the f.closed is set and the condition is broadcasted.
         // Which causes this loop to continue and return from the Pop().
         if f.closed {
            return nil, ErrFIFOClosed
         }

         f.cond.Wait()
      }
      // 2.取出queue的队头对象key
      id := f.queue[0]
      // 3.更新queue，把queue中所有的对象key前移，相当于把第一个对象key给pop出去
      f.queue = f.queue[1:]
      depth := len(f.queue)
      // 4.initialPopulationCount变量减1，当减到0时则说明initialPopulationCount代表第一次调用Replace方法加入DeltaFIFO中的对象key已经被pop完成
      if f.initialPopulationCount > 0 {
         f.initialPopulationCount--
      }
      // 5.根据对象key从items中获取对象
      item, ok := f.items[id]
      if !ok {
         // This should never happen
         klog.Errorf("Inconceivable! %q was in f.queue but not f.items; ignoring.", id)
         continue
      }
      // 6.把对象从items中删除
      delete(f.items, id)
      // Only log traces if the queue depth is greater than 10 and it takes more than
      // 100 milliseconds to process one item from the queue.
      // Queue depth never goes high because processing an item is locking the queue,
      // and new items can't be added until processing finish.
      // https://github.com/kubernetes/kubernetes/issues/103789
      if depth > 10 {
         trace := utiltrace.New("DeltaFIFO Pop Process",
            utiltrace.Field{Key: "ID", Value: id},
            utiltrace.Field{Key: "Depth", Value: depth},
            utiltrace.Field{Key: "Reason", Value: "slow event handlers blocking the queue"})
         defer trace.LogIfLong(100 * time.Millisecond)
      }
      // 7.调用PopProcessFunc处理pop出来的对象
      err := process(item)
      if e, ok := err.(ErrRequeue); ok {
         f.addIfNotPresent(id, item)
         err = e.Err
      }
      // Don't need to copyDeltas here, because we're transferring
      // ownership to the caller.
      return item, err
   }
}
```

## **DeltaFIFO.HasSynced**

HasSynced指第一次从kube-apiserver中获取到的全量的对象是否全部从DeltaFIFO中pop完成，全部pop完成，说明list回来的对象已经全部同步到了Indexer缓存中去了。

①populated在第一次调用DeltaFIFO的Replace方法中将其设置为true。

②initialPopulationCount的值在第一次调用DeltaFIFO的Replace方法中设置值为加入到items中的Deltas的数量，然后每pop一个Delta，则initialPopulationCount的值减1，pop完成时值则为0。

```go
// staging/src/k8s.io/client-go/tools/cache/delta_fifo.go
func (f *DeltaFIFO) HasSynced() bool {
    f.lock.Lock()
    defer f.lock.Unlock()
    return f.populated && f.initialPopulationCount == 0
}
```

\




[^1]: DeltaFIFO
