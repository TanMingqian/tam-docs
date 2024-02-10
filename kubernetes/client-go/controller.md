# Controller

Controller从DeltaFIFO中pop Deltas出来处理，根据对象的变化更新Indexer本地缓存，并通知Processor相关对象有变化事件发生：&#x20;

（1）如果是Added、Updated、Sync类型，则从indexer中获取该对象，存在则调用s.indexer.Update来更新indexer中的该对象，随后构造updateNotification struct，并通知Processor；如果indexer中不存在该对象，则调用s.indexer.Add来往indexer中添加该对象，随后构造addNotification struct，并通知Processor；&#x20;

（2）如果是Deleted类型，则调用s.indexer.Delete来将indexer中的该对象删除，随后构造deleteNotification struct，并通知Processor；

### Controller初始化与启动分析 <a href="#id-2controller-chu-shi-hua-yu-qi-dong-fen-xi" id="id-2controller-chu-shi-hua-yu-qi-dong-fen-xi"></a>

**Cotroller初始化-New**

New用于初始化Controller，方法比较简单

```go
// staging/src/k8s.io/client-go/tools/cache/controller.go
func New(c *Config) Controller {
	ctlr := &controller{
		config: *c,
		clock:  &clock.RealClock{},
	}
	return ctlr
}Controller启动-controller.RunController启动-controller.Run
```

**Controller启动-controller.Run**

controller.Run为controller的启动方法，这里主要看到几个点：&#x20;

（1）调用NewReflector，初始化Reflector；&#x20;

（2）调用r.Run，实际上是调用了Reflector的启动方法来启动Reflector（Reflector相关的分析前面已经分析过了，这里不再重复）；&#x20;

（3）调用c.processLoop，开始controller的核心处理

```go
// staging/src/k8s.io/client-go/tools/cache/controller.go
func (c *controller) Run(stopCh <-chan struct{}) {
	defer utilruntime.HandleCrash()
	go func() {
		<-stopCh
		c.config.Queue.Close()
	}()
	r := NewReflector(
		c.config.ListerWatcher,
		c.config.ObjectType,
		c.config.Queue,
		c.config.FullResyncPeriod,
	)
	r.ShouldResync = c.config.ShouldResync
	r.clock = c.clock

	c.reflectorMutex.Lock()
	c.reflector = r
	c.reflectorMutex.Unlock()

	var wg wait.Group
	defer wg.Wait()

	wg.StartWithChannel(stopCh, r.Run)

	wait.Until(c.processLoop, time.Second, stopCh)
}
```

### controller核心处理方法分析 <a href="#id-3controller-he-xin-chu-li-fang-fa-fen-xi" id="id-3controller-he-xin-chu-li-fang-fa-fen-xi"></a>

**controller.processLoop**

controller的核心处理方法processLoop中，最重要的逻辑是循环调用c.config.Queue.Pop将DeltaFIFO中的队头元素给pop出来（实际上pop出来的是Deltas，是Delta的切片类型），然后调用`c.config.Process`方法来做处理，当处理出错时，再调用`c.config.Queue.AddIfNotPresent`将对象重新加入到DeltaFIFO中去

```go
func (c *controller) processLoop() {
	for {
		obj, err := c.config.Queue.Pop(PopProcessFunc(c.config.Process))
		if err != nil {
			if err == ErrFIFOClosed {
				return
			}
			if c.config.RetryOnError {
				// This is the safe way to re-enqueue.
				c.config.Queue.AddIfNotPresent(obj)
			}
		}
	}
}

```

**c.config.Process/s.HandleDeltas**

根据前面分析知道HandleDeltas要处理的是Deltas，是Delta的切片类型。

再来看到HandleDeltas方法的主要逻辑：\
（1）循环遍历Deltas，拿到单个Delta；\
（2）判断Delta的类型；\
（3）如果是Added、Updated、Sync类型，则从indexer中获取该对象，存在则调用s.indexer.Update来更新indexer中的该对象，随后构造updateNotification struct，并调用s.processor.distribute方法；如果indexer中不存在该对象，则调用s.indexer.Add来往indexer中添加该对象，随后构造addNotification struct，并调用s.processor.distribute方法；\
（4）如果是Deleted类型，则调用s.indexer.Delete来将indexer中的该对象删除，随后构造deleteNotification struct，并调用s.processor.distribute方法；

```go
// staging/src/k8s.io/client-go/tools/cache/shared_informer.go
func (s *sharedIndexInformer) HandleDeltas(obj interface{}) error {
	s.blockDeltas.Lock()
	defer s.blockDeltas.Unlock()

	// from oldest to newest
	for _, d := range obj.(Deltas) {
		switch d.Type {
		case Sync, Added, Updated:
			isSync := d.Type == Sync
			s.cacheMutationDetector.AddObject(d.Object)
			if old, exists, err := s.indexer.Get(d.Object); err == nil && exists {
				if err := s.indexer.Update(d.Object); err != nil {
					return err
				}
				s.processor.distribute(updateNotification{oldObj: old, newObj: d.Object}, isSync)
			} else {
				if err := s.indexer.Add(d.Object); err != nil {
					return err
				}
				s.processor.distribute(addNotification{newObj: d.Object}, isSync)
			}
		case Deleted:
			if err := s.indexer.Delete(d.Object); err != nil {
				return err
			}
			s.processor.distribute(deleteNotification{oldObj: d.Object}, false)
		}
	}
	return nil
}

type updateNotification struct {
	oldObj interface{}
	newObj interface{}
}

type addNotification struct {
	newObj interface{}
}

type deleteNotification struct {
	oldObj interface{}
}

```







