# leader选举机制

### 背景

选主（Leader election）就是在分布式系统内抉择出一个主节点来负责一些特定的工作。在执行了选主过程后，集群中每个节点都会识别出一个特定的、唯一的节点作为leader。

####

<figure><img src="../../.gitbook/assets/image (41).png" alt=""><figcaption></figcaption></figure>

Master HA 实现没有单点故障的目标，需要为:etcd、kube-apiserver、kube-controller-manager、kube-scheduler组件建立高可用方案

etcd 通常是集群化的，自有一套选主机制

kube-apiserver 本身是无状态服务，实现高可用的主要思路就是负载均衡，具体实现有外部负载均衡、网络层负载均衡及Node节点上使用反向代理对多个Master做负载均衡等方案

kube-controller-manager和kube-scheduler 服务是Master节点的一部分，启动参数\`--leader-elect=true\`，告知组件以高可用方式启动，获取到锁(apiserver中的Endpoint)的实例成为leader，其他候选者通过对比锁跟新时间和持有者来判断自己是否能成为新的leader，leader通过更新RenewTime来确保持续持有锁并保持leadership

