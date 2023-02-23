---
description: 反亲和调度--selector反亲和  k8s 1.17
---

# SelectorSpreadPriority

### SelectorSpreadPriority&#x20;

将节点调度到 匹配相同selector 的pod数最少的node上

```go
// kubernetes/pkg/scheduler/algorithm/priorities/priorities.go 
// SelectorSpreadPriority defines the name of prioritizer function that spreads pods by minimizing
// the number of pods (belonging to the same service or replication controller) on the same node.
SelectorSpreadPriority = "SelectorSpreadPriority"
```

注册 SelectorSpreadPriority 工厂到 scheuler上

添加MapReduceFuction 用于计算节点分数

<pre class="language-go"><code class="lang-go"><strong>
</strong><strong>
</strong><strong>scheduler.RegisterPriorityConfigFactory(
</strong>   priorities.SelectorSpreadPriority,
   scheduler.PriorityConfigFactory{
      MapReduceFunction: func(args scheduler.AlgorithmFactoryArgs) (priorities.PriorityMapFunction, priorities.PriorityReduceFunction) {
         serviceLister := args.InformerFactory.Core().V1().Services().Lister()
         controllerLister := args.InformerFactory.Core().V1().ReplicationControllers().Lister()
         replicaSetLister := args.InformerFactory.Apps().V1().ReplicaSets().Lister()
         statefulSetLister := args.InformerFactory.Apps().V1().StatefulSets().Lister()
         return priorities.NewSelectorSpreadPriority(serviceLister, controllerLister, replicaSetLister, statefulSetLister)
      },
      Weight: 1,
   },
)
</code></pre>

创建 SelectorSpreadPriority  实例，传入service relicationController replicaSet statefulSet  的Lister

返回CalculateSpreadPriorityMap CalculateSpreadPriorityReduce两个function

```go
// NewSelectorSpreadPriority creates a SelectorSpread.
func NewSelectorSpreadPriority(
   serviceLister corelisters.ServiceLister,
   controllerLister corelisters.ReplicationControllerLister,
   replicaSetLister appslisters.ReplicaSetLister,
   statefulSetLister appslisters.StatefulSetLister) (PriorityMapFunction, PriorityReduceFunction) {
   selectorSpread := &SelectorSpread{
      serviceLister:     serviceLister,
      controllerLister:  controllerLister,
      replicaSetLister:  replicaSetLister,
      statefulSetLister: statefulSetLister,
   }
   return selectorSpread.CalculateSpreadPriorityMap, selectorSpread.CalculateSpreadPriorityReduce
}
```

CalculateSpreadPriorityMap 用于给指定节点打分，分数为计算本节点匹配selector的pod个数

```go
// CalculateSpreadPriorityMap spreads pods across hosts, considering pods
// belonging to the same service,RC,RS or StatefulSet.
// When a pod is scheduled, it looks for services, RCs,RSs and StatefulSets that match the pod,
// then finds existing pods that match those selectors.
// It favors nodes that have fewer existing matching pods.
// i.e. it pushes the scheduler towards a node where there's the smallest number of
// pods which match the same service, RC,RSs or StatefulSets selectors as the pod being scheduled.
func (s *SelectorSpread) CalculateSpreadPriorityMap(pod *v1.Pod, meta interface{}, nodeInfo *schedulernodeinfo.NodeInfo) (framework.NodeScore, error) {
   var selector labels.Selector
   // 获取node信息
   node := nodeInfo.Node()
   if node == nil {
      return framework.NodeScore{}, fmt.Errorf("node not found")
   }

   priorityMeta, ok := meta.(*priorityMetadata)
   if ok {
      selector = priorityMeta.podSelector
   } else {
      selector = getSelector(pod, s.serviceLister, s.controllerLister, s.replicaSetLister, s.statefulSetLister)
   }
   
   // 计数匹配的pod数量作为 NodeScore 分数
   count := countMatchingPods(pod.Namespace, selector, nodeInfo)
   return framework.NodeScore{
      Name:  node.Name,
      Score: int64(count),
   }, nil
}

// 通过pod.namespace 和 selector 匹配node节点的所有pod
// countMatchingPods counts pods based on namespace and matching all selectors
func countMatchingPods(namespace string, selector labels.Selector, nodeInfo *schedulernodeinfo.NodeInfo) int {
	if nodeInfo.Pods() == nil || len(nodeInfo.Pods()) == 0 || selector.Empty() {
		return 0
	}
	count := 0
	// 遍历pod列表
	for _, pod := range nodeInfo.Pods() {
	        // 如果pod与给定ns相同，且selector 完全匹配，则count ++
		// Ignore pods being deleted for spreading purposes
		// Similar to how it is done for SelectorSpreadPriority
		if namespace == pod.Namespace && pod.DeletionTimestamp == nil {
			if selector.Matches(labels.Set(pod.Labels)) {
				count++
			}
		}
	}
	return count
}
```



CalculateSpreadPriorityReduce 用于对CalculateSpreadPriorityMap 方法计算出的每个节点的scores进行加工运算，计算出最后的节点score

遍历NodeScoreList，获取节点最高pod匹配计数maxCountByNodeName

&#x20;节点分数 = （节点最高pod匹配计数 - 本节点pod匹配计数）/ 节点最高计数 \* 100

例：

节点最高pod匹配计数 = 10&#x20;

A节点pod匹配数 = 5 &#x20;

B节点pod匹配数 = 2

A节点最终分数 = （10 - 5）/10 \* 100 = 50

B节点最终分数 = （10 - 2）/10 \* 100 = 80

```go
// CalculateSpreadPriorityReduce calculates the source of each node
// based on the number of existing matching pods on the node
// where zone information is included on the nodes, it favors nodes
// in zones with fewer existing matching pods.
func (s *SelectorSpread) CalculateSpreadPriorityReduce(pod *v1.Pod, meta interface{}, sharedLister schedulerlisters.SharedLister, result framework.NodeScoreList) error {
   countsByZone := make(map[string]int64, 10)
   maxCountByZone := int64(0)
   maxCountByNodeName := int64(0)

   for i := range result {
      if result[i].Score > maxCountByNodeName {
         maxCountByNodeName = result[i].Score
      }
      nodeInfo, err := sharedLister.NodeInfos().Get(result[i].Name)
      if err != nil {
         return err
      }
      zoneID := utilnode.GetZoneKey(nodeInfo.Node())
      if zoneID == "" {
         continue
      }
      countsByZone[zoneID] += result[i].Score
   }

   for zoneID := range countsByZone {
      if countsByZone[zoneID] > maxCountByZone {
         maxCountByZone = countsByZone[zoneID]
      }
   }

   haveZones := len(countsByZone) != 0

   maxCountByNodeNameFloat64 := float64(maxCountByNodeName)
   maxCountByZoneFloat64 := float64(maxCountByZone)
   MaxNodeScoreFloat64 := float64(framework.MaxNodeScore)

   for i := range result {
      // initializing to the default/max node score of maxPriority
      fScore := MaxNodeScoreFloat64
      if maxCountByNodeName > 0 {
         fScore = MaxNodeScoreFloat64 * (float64(maxCountByNodeName-result[i].Score) / maxCountByNodeNameFloat64)
      }
      // If there is zone information present, incorporate it
      if haveZones {
         nodeInfo, err := sharedLister.NodeInfos().Get(result[i].Name)
         if err != nil {
            return err
         }

         zoneID := utilnode.GetZoneKey(nodeInfo.Node())
         if zoneID != "" {
            zoneScore := MaxNodeScoreFloat64
            if maxCountByZone > 0 {
               zoneScore = MaxNodeScoreFloat64 * (float64(maxCountByZone-countsByZone[zoneID]) / maxCountByZoneFloat64)
            }
            fScore = (fScore * (1.0 - zoneWeighting)) + (zoneWeighting * zoneScore)
         }
      }
      result[i].Score = int64(fScore)
      if klog.V(10) {
         klog.Infof(
            "%v -> %v: SelectorSpreadPriority, Score: (%d)", pod.Name, result[i].Name, int64(fScore),
         )
      }
   }
   return nil
}
```

