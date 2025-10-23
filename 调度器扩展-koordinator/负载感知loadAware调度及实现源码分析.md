# 基于节点负载的评分算法实现

公式： (可分配的资源 - pod需要使用的资源) * MaxNodeScore / 可分配的资源
其中： MaxNodeScore是一个常数

```go
func (p *Plugin) Score(ctx context.Context, state *framework.CycleState, pod *corev1.Pod, nodeName string) (int64, *framework.Status) {
	
	// 获取可分配的资源列表
	allocatableList, err := p.estimator.EstimateNode(node)
	if err != nil {
		klog.ErrorS(err, "Estimated node allocatable failed!", "node", node.Name)
		return 0, nil
	}
	allocatable := p.vectorizer.ToVec(allocatableList)

	// 是否是生产节点
	prodPod := p.args.ScoreAccordingProdUsage && extension.GetPodPriorityClassWithDefault(pod) == extension.PriorityProd
	var aggDuration metav1.Duration
	var aggType extension.AggregationType
	if agg := p.args.Aggregated; !prodPod && agg != nil && agg.ScoreAggregationType != "" {
		aggDuration, aggType = agg.ScoreAggregatedDuration, agg.ScoreAggregationType
	}

	// 获取节点指标、estimated待调度的pod需分配的资源（默认为空）、指标还未收集到pod列表；其中节点指标来自 koordlet 采集的指标
	nodeMetric, estimated, estimatedPods, err := p.podAssignCache.GetNodeMetricAndEstimatedOfExisting(nodeName, prodPod, aggDuration, aggType, klog.V(6).Enabled())

    if nodeMetric.Status.NodeMetric == nil {
        klog.Warningf("nodeMetrics(%s) should not be nil.", node.Name)
        return 0, nil
    }
	// 根据pod的limit、request评估pod的需分配的资源，并加入estimated
	if err = p.addEstimatedOfIncoming(estimated, state, pod); err != nil {
		klog.ErrorS(err, "Failed to estimate incoming pod usage", "pod", klog.KObj(pod))
		return 0, nil
	}
	if klog.V(6).Enabled() {
		klog.InfoS("Estimate node usage for scoring", "pod", klog.KObj(pod), "node", nodeMetric.Name,
			"estimated", klog.Format(p.vectorizer.ToList(estimated)),
			"estimatedExistingPods", klog.KObjSlice(estimatedPods))
	}
	score := loadAwareSchedulingScorer(p.args.DominantResourceWeight, p.scoreWeights, estimated, allocatable)
	return score, nil
}
```


EstimateNode 评估节点可以分配的资源列表：(1) 从注解"node.koordinator.sh/raw-allocatable"中读取 补充 （2）从node.Status.Allocatable中读取
```go
func (e *DefaultEstimator) EstimateNode(node *corev1.Node) (corev1.ResourceList, error) {
	rawAllocatable, err := extension.GetNodeRawAllocatable(node.Annotations)
	if err != nil {
		return node.Status.Allocatable, nil
	}
	if len(rawAllocatable) == 0 {
		return node.Status.Allocatable, nil
	}
	if quotav1.Equals(rawAllocatable, node.Status.Allocatable) {
		return node.Status.Allocatable, nil
	}
	allocatableCopy := node.Status.Allocatable.DeepCopy()
	if allocatableCopy == nil {
		allocatableCopy = corev1.ResourceList{}
	}
	for k, v := range rawAllocatable {
		allocatableCopy[k] = v
	}
	return allocatableCopy, nil
}
```


```go
// 根据节点上已经分配的资源（allocatable），pod需要使用的资源（used），resToWeightMap 资源权重映射 计算得分
func loadAwareSchedulingScorer(dominantWeight int64, resToWeightMap, used, allocatable ResourceVector) int64 {
	var nodeScore, dominantScore, weightSum int64
	if dominantWeight != 0 {
		dominantScore, weightSum = framework.MaxNodeScore, dominantWeight
	}
	for i, weight := range resToWeightMap {
		score := leastUsedScore(used[i], allocatable[i])
		nodeScore += score * weight
		weightSum += weight
		if dominantScore > score {
			dominantScore = score
		}
	}
	nodeScore += dominantScore * dominantWeight
	if weightSum <= 0 {
		return 0
	}
	return nodeScore / weightSum
}

func leastUsedScore(used, capacity int64) int64 {
	if capacity == 0 {
		return 0
	}
	if used > capacity {
		return 0
	}
    // (可分配的资源 - pod需要使用的资源) * MaxNodeScore / 可分配的资源
	return ((capacity - used) * framework.MaxNodeScore) / capacity
}
```