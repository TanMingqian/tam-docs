# Client-go

<figure><img src="../.gitbook/assets/image (2) (3).png" alt=""><figcaption></figcaption></figure>

从图中可以看出，k8s informer主要包括以下几个部分：

**1.Reflector**

（1）Reflector调用kube-apiserver的list接口获取资源对象列表，然后调用DeltaFIFO的Replace方法将object包装成Sync/Deleted类型的Delta丢进DeltaFIFO中；

（2）Reflector调用kube-apiserver的watch接口获取资源对象的变化，然后调用DeltaFIFO的Add/Update/Delete方法将object包装成Added/Updated/Deleted类型的Delta丢到DeltaFIFO中；

**2.DeltaFIFO**

DeltaFIFO中存储着一个map和一个queue；

（1）其中queue可以看成是一个先进先出队列，一个object进入DeltaFIFO中，会判断queue中是否已经存在该object key，不存在则添加到队尾；

（2）map即map\[object key]Deltas，是object key和Deltas的映射，Deltas是Delta的切片类型，Delta中存储着DeltaType和object；

DeltaType有4种，分别是Added、Updated、Deleted、Sync

**3.Controller**

Controller从DeltaFIFO的queue中pop一个object key出来，并从DeltaFIFO的map中获取其对应的 Deltas出来进行处理，遍历Deltas，根据object的变化类型更新Indexer本地缓存，并通知Processor相关对象有变化事件发生：

（1）如果DeltaType是Deleted，则调用Indexer的Delete方法，将Indexer本地缓存中的object删除，并构造deleteNotification，通知Processor做处理；

（2）如果DeltaType是Added/Updated/Sync，调用Indexer的Get方法从Indexer本地缓存中获取该对象，存在则调用Indexer的Update方法来更新Indexer缓存中的该对象，随后构造updateNotification，通知Processor做处理；如果Indexer中不存在该对象，则调用Indexer的Add方法将该对象存入本地缓存中，并构造addNotification，通知Processor做处理；

**4.Processor**

Processor根据Controller的通知，即根据对象的变化事件类型（addNotification、updateNotification、deleteNotification），调用相应的ResourceEventHandler（addFunc、updateFunc、deleteFunc）将object key加入工作队列workqueue给custom controller处理。

**5.Indexer**

Indexer是资源对象的本地缓存，依赖于threadSafeMap的items，indexers，indices\
items：items是一个map，本地存储所有资源对象，key为资源对象的namespace/name组成，value为资源对象本身\
indexers，indices：索引

**6.ResourceEventHandler**

调用相应的ResourceEventHandler（addFunc、updateFunc、deleteFunc）将object key加入工作队列workqueue给custom controller处理



<figure><img src="../.gitbook/assets/image (7).png" alt=""><figcaption></figcaption></figure>
