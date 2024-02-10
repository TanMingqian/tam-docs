# Indexer

Indexer是资源对象的本地缓存，依赖于threadSafeMap的items，indexers，indices items：items是一个map，本地存储所有资源对象，key为资源对象的namespace/name组成，value为资源对象本身 indexers，indices：索引

<figure><img src="../../.gitbook/assets/image (1).png" alt=""><figcaption></figcaption></figure>

### Indexer的结构定义分析

**Store**

Store接口本身，定义了Add、Update、Delete、List、Get等一些对象增删改查的方法声明，用于操作informer的本地缓存items

```go
// staging/src/k8s.io/client-go/tools/cache/store.go
type Store interface {
    Add(obj interface{}) error
    Update(obj interface{}) error
    Delete(obj interface{}) error
    List() []interface{}
    ListKeys() []string
    Get(obj interface{}) (item interface{}, exists bool, err error)
    GetByKey(key string) (item interface{}, exists bool, err error)

    Replace([]interface{}, string) error
    Resync() error
}
```

**Indexer**\
Indexer接口继承了一个Store接口（实现本地缓存），以及包含几个index索引相关的方法声明（实现索引功能）

```go
// staging/src/k8s.io/client-go/tools/cache/index.go
type Indexer interface {
    Store

    Index(indexName string, obj interface{}) ([]interface{}, error)

    IndexKeys(indexName, indexedValue string) ([]string, error)

    ListIndexFuncValues(indexName string) []string

    ByIndex(indexName, indexedValue string) ([]interface{}, error)

    GetIndexers() Indexers

    AddIndexers(newIndexers Indexers) error
}
```

**cache**

cache是Indexer接口的一个实现，cache包含一个ThreadSafeStore接口的实现，以及一个计算object key的函数KeyFunc

cache会根据keyFunc生成某个obj对象对应的一个唯一key, 然后调用ThreadSafeStore接口中的方法来操作本地缓存中的对象

```go
// staging/src/k8s.io/client-go/tools/cache/store.go
type cache struct {
    cacheStorage ThreadSafeStore
    keyFunc KeyFunc
}
```

**ThreadSafeStore**

ThreadSafeStore接口包含了操作本地缓存的增删改查方法以及索引功能的相关方法

```go
// staging/src/k8s.io/client-go/tools/cache/thread_safe_store.go
type ThreadSafeStore interface {
    Add(key string, obj interface{})
    Update(key string, obj interface{})
    Delete(key string)
    Get(key string) (item interface{}, exists bool)
    List() []interface{}
    ListKeys() []string
    Replace(map[string]interface{}, string)

    Index(indexName string, obj interface{}) ([]interface{}, error)
    IndexKeys(indexName, indexKey string) ([]string, error)
    ListIndexFuncValues(name string) []string
    ByIndex(indexName, indexKey string) ([]interface{}, error)
    GetIndexers() Indexers

    AddIndexers(newIndexers Indexers) error
    Resync() error
}
```

**threadSafeMap**

threadSafeMap是ThreadSafeStore接口的一个实现，资源对象都存在items这个map中，key是根据资源对象来算出，value为资源对象本身，这里的items即为informer的本地缓存了，而indexers与indices属性则与索引功能有关

```go
// staging/src/k8s.io/client-go/tools/cache/thread_safe_store.go
type threadSafeMap struct {
    lock  sync.RWMutex
  // 资源对象本地存储，key为对obj计算的key，value为obj本身
    items map[string]interface{}

    // indexers maps a name to an IndexFunc
    indexers Indexers
    // indices maps a name to an Index
    indices Indices
}
```

### Indexer的索引功能

在threadSafeMap中，与索引功能有关的是indexers与indices属性

```go
// staging/src/k8s.io/client-go/tools/cache/thread_safe_store.go
type threadSafeMap struct {
    lock  sync.RWMutex
    items map[string]interface{}

    // indexers maps a name to an IndexFunc
    indexers Indexers
    // indices maps a name to an Index
    indices Indices
}
```

```go
// staging/src/k8s.io/client-go/tools/cache/index.go
const (
    NamespaceIndex string = "namespace"
)

type IndexFunc func(obj interface{}) ([]string, error)

type Indexers map[string]IndexFunc

type Indices map[string]Index

type Index map[string]sets.String
```

**Indexers**\
Indexers包含了所有索引器(索引分类)及其索引器函数IndexFunc，IndexFunc计算obj索引键

```go
Indexers: {  
  "索引器名称1": 索引函数1,
  "索引器名称2": 索引函数2,
}
```

```go
Indexers: {  
  "namespace": MetaNamespaceIndexFunc, #根据namespace建立索引，MetaNamespaceIndexFunc计算obj的索引值
}
```



**Indices**\
Indices包含了所有索引器及其所有的索引数据Index；而Index则包含了索引键以及索引键下的所有对象键的列表

```go
Indices: {
 "索引器名称1": {  
  "索引键1": ["对象键1", "对象键2"],  
  "索引键2": ["对象键3"],   
 },
 "索引器名称2": {  
  "索引键3": ["对象键1"],  
  "索引键4": ["对象键2", "对象键3"],  
 }
}
```

```go
Indices: {
 "namespace": {  
  "default": ["default/pod-1", "default/pod-2"],  #通过命名空间default索引映射到所有本地缓存的key
  "kube-system": ["kube-system/pod-3"],   #通过命名空间kube-system索引映射到所有本地缓存的key
 }
}
```

当items中添加obj时，MetaNamespaceIndexFunc计算出obj的namespace(default)，将obj的key(default/pod-2) 当获取namespace(default)下资源对象时，返回相应key列表，并从items中根据key获取obj



<figure><img src="../../.gitbook/assets/image (44).png" alt=""><figcaption></figcaption></figure>

我们按照命名空间来进行索引，可以简单的理解为这个 Indexer 就是简单的把相同命名空间的对象放在一个集合中，然后基于命名空间来查找对象

\
Indexer本地缓存
-----------

前面对informer-Controller的分析中（代码如下），提到的s.indexer.Add、s.indexer.Update、s.indexer.Delete、s.indexer.Get等方法其实最终就是调用的threadSafeMap.Add、threadSafeMap.Update、threadSafeMap.Delete、threadSafeMap.Get等

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
```

**threadSafeMap.Add**\
调用链：s.indexer.Add --> cache.Add --> threadSafeMap.Add

threadSafeMap.Add方法将`key：object`存入items中，并调用`updateIndices`方法更新索引Indices

```go
// staging/src/k8s.io/client-go/tools/cache/thread_safe_store.go
func (c *threadSafeMap) Add(key string, obj interface{}) {
    c.lock.Lock()
    defer c.lock.Unlock()
    oldObject := c.items[key]
    c.items[key] = obj
    c.updateIndices(oldObject, obj, key)
}
```

也可以看到对threadSafeMap进行操作的方法，基本都会先获取锁，然后方法执行完毕释放锁，所以是并发安全的\
**threadSafeMap.Delete**

调用链：s.indexer.Delete --> cache.Delete --> threadSafeMap.Delete

threadSafeMap.Delete方法中，先判断本地缓存items中是否存在该key，存在则调用`deleteFromIndices`删除相关索引，然后删除items中的key及其对应object

```go
// staging/src/k8s.io/client-go/tools/cache/thread_safe_store.go
func (c *threadSafeMap) Delete(key string) {
    c.lock.Lock()
    defer c.lock.Unlock()
    if obj, exists := c.items[key]; exists {
        c.deleteFromIndices(obj, key)
        delete(c.items, key)
    }
}
```

**threadSafeMap.Get**\
调用链：s.indexer.Get --> cache.Get --> threadSafeMap.Get

threadSafeMap.Get方法逻辑相对简单，没有索引的相关操作，而是直接从items中通过key获取对应的object并返回；

```go
// staging/src/k8s.io/client-go/tools/cache/thread_safe_store.go
func (c *threadSafeMap) Get(key string) (item interface{}, exists bool) {
    c.lock.RLock()
    defer c.lock.RUnlock()
    item, exists = c.items[key]
    return item, exists
}
```

\
**threadSafeMap.Index**

threadSafeMap.Index方法查询indexName索引器下符合obj条件的值，返回实际的对象列表

```go
// Index returns a list of items that match on the index function
// Index is thread-safe so long as you treat all items as immutable
func (c *threadSafeMap) Index(indexName string, obj interface{}) ([]interface{}, error) {
   c.lock.RLock()
   defer c.lock.RUnlock()

   indexFunc := c.indexers[indexName]
   if indexFunc == nil {
      return nil, fmt.Errorf("Index with name %s does not exist", indexName)
   }

   indexKeys, err := indexFunc(obj)
   if err != nil {
      return nil, err
   }
   index := c.indices[indexName]

   var returnKeySet sets.String
   if len(indexKeys) == 1 {
      // In majority of cases, there is exactly one value matching.
      // Optimize the most common path - deduping is not needed here.
      returnKeySet = index[indexKeys[0]]
   } else {
      // Need to de-dupe the return list.
      // Since multiple keys are allowed, this can happen.
      returnKeySet = sets.String{}
      for _, indexKey := range indexKeys {
         for key := range index[indexKey] {
            returnKeySet.Insert(key)
         }
      }
   }

   list := make([]interface{}, 0, returnKeySet.Len())
   for absoluteKey := range returnKeySet {
      list = append(list, c.items[absoluteKey])
   }
   return list, nil
}
```



**threadSafeMap.ByIndex**

&#x20;传入索引器名称indexName，以及索引键名称indexKey，方法寻找该索引器下，索引键对应的对象键列表，然后根据对象键列表，到Indexer缓存（即threadSafeMap中的items）中获取出相应的对象列表

```go
// ByIndex returns a list of items that match an exact value on the index function
func (c *threadSafeMap) ByIndex(indexName, indexKey string) ([]interface{}, error) {
   c.lock.RLock()
   defer c.lock.RUnlock()

   indexFunc := c.indexers[indexName]
   if indexFunc == nil {
      return nil, fmt.Errorf("Index with name %s does not exist", indexName)
   }

   index := c.indices[indexName]

   set := index[indexKey]
   list := make([]interface{}, 0, set.Len())
   for key := range set {
      list = append(list, c.items[key])
   }

   return list, nil
}
```

**threadSafeMap.IndexKeys**

IndexKeys方法与ByIndex方法类似，只不过只返回对象键列表，不会根据对象键列表，到Indexer缓存（即threadSafeMap中的items属性）中获取出相应的对象列表

```go
// IndexKeys returns a list of keys that match on the index function.
// IndexKeys is thread-safe so long as you treat all items as immutable.
func (c *threadSafeMap) IndexKeys(indexName, indexKey string) ([]string, error) {
   c.lock.RLock()
   defer c.lock.RUnlock()

   indexFunc := c.indexers[indexName]
   if indexFunc == nil {
      return nil, fmt.Errorf("Index with name %s does not exist", indexName)
   }

   index := c.indices[indexName]

   set := index[indexKey]
   return set.List(), nil
}
```
