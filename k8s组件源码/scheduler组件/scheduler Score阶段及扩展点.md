# 概述

scheduler中Score阶段是pod调度最关键的阶段，它的作用是对所有可调度节点进行打分，pod将被优先调度到打分高的节点。那这一块是怎样实现的呢？
首先看Score插件接口是如何定义的？

```go
type ScorePlugin interface {
	Plugin
	// Score is called on each filtered node. It must return success and an integer
	// indicating the rank of the node. All scoring plugins must return success or
	// the pod will be rejected.
	Score(ctx context.Context, state fwk.CycleState, p *v1.Pod, nodeInfo fwk.NodeInfo) (int64, *fwk.Status)

	// ScoreExtensions returns a ScoreExtensions interface if it implements one, or nil if does not.
	ScoreExtensions() ScoreExtensions
}
```

定义了2个接口： Score，ScoreExtensions。其中  Score 接口参数是 pod、 node，接口返回int64型的分数，即pod调度到当前node上的分数。

在scheduler中，实现了ScorePlugin的有：
- BalancedAllocation
- Fit
- ImageLocality
- InterPodAffinity
- NodeAffinity
- PodTopologySpread
- TaintToleration
- VolumeBinding

2个问题：
- scheduler组件是如何调用这些 ScorePlugin给Node打分？
- 每个插件的具体实现是怎样的？
- 最后这个综合分数是根据什么计算公式得到的？


# scheduler组件是如何调用这些 ScorePlugin给Node打分

入口代码：scheduler 组件从调度队列取出1个未调度的pod，进入调度循环

```go
func (sched *Scheduler) ScheduleOne(ctx context.Context) {
	logger := klog.FromContext(ctx)
	podInfo, err := sched.NextPod(logger)
	... ...

	fwk, err := sched.frameworkForPod(pod)
	... ... 

	// 进入调度循环
	scheduleResult, assumedPodInfo, status := sched.schedulingCycle(schedulingCycleCtx, state, fwk, podInfo, start, podsToActivate)
	if !status.IsSuccess() {
		sched.FailureHandler(schedulingCycleCtx, fwk, assumedPodInfo, status, scheduleResult.nominatingInfo, start)
		return
	}

	// bind the pod to its host asynchronously (we can do this b/c of the assumption step above).
	go func() {
		... ... 

		// 根据调度结果，为pod 绑定1个得分高的节点
		status := sched.bindingCycle(bindingCycleCtx, state, fwk, scheduleResult, assumedPodInfo, start, podsToActivate)
		if !status.IsSuccess() {
			sched.handleBindingCycleError(bindingCycleCtx, state, fwk, assumedPodInfo, start, scheduleResult, status)
			return
		}
	}()
}
```


```go
func (sched *Scheduler) schedulingCycle(
	ctx context.Context,
	state fwk.CycleState,
	schedFramework framework.Framework,
	podInfo *framework.QueuedPodInfo,
	start time.Time,
	podsToActivate *framework.PodsToActivate,
) (ScheduleResult, *framework.QueuedPodInfo, *fwk.Status) {
	logger := klog.FromContext(ctx)
	pod := podInfo.Pod
	scheduleResult, err := sched.SchedulePod(ctx, schedFramework, state, pod)
	... ...
	return scheduleResult, assumedPodInfo, nil
}
```

其中 SchedulePod 函数即如下schedulePod函数：

```go
func (sched *Scheduler) schedulePod(ctx context.Context, fwk framework.Framework, state fwk.CycleState, pod *v1.Pod) (result ScheduleResult, err error) {
	... ... 
	// 选择可行节点
	feasibleNodes, diagnosis, err := sched.findNodesThatFitPod(ctx, fwk, state, pod)
	
	... ... 
	
	// 给可行节点打分
	priorityList, err := prioritizeNodes(ctx, sched.Extenders, fwk, state, pod, feasibleNodes)
	if err != nil {
		return result, err
	}

	// 根据得分，从可行节点列表选择1个节点
	host, _, err := selectHost(priorityList, numberOfHighestScoredNodesToReport)
	trace.Step("Prioritizing done")

	return ScheduleResult{
		SuggestedHost:  host,
		EvaluatedNodes: len(feasibleNodes) + diagnosis.NodeToStatus.Len(),
		FeasibleNodes:  len(feasibleNodes),
	}, err
}
```

本文重点关注score相关的逻辑，所以下面开始看 prioritizeNodes 函数

```go
func prioritizeNodes(
	ctx context.Context,
	extenders []framework.Extender,
	schedFramework framework.Framework,
	state fwk.CycleState,
	pod *v1.Pod,
	nodes []fwk.NodeInfo,
) ([]framework.NodePluginScores, error) {
	... ...
	
	// 运行score插件
	nodesScores, scoreStatus := schedFramework.RunScorePlugins(ctx, state, pod, nodes)
	if !scoreStatus.IsSuccess() {
		return nil, scoreStatus.AsError()
	}

	... ...
}
```

```go
func (f *frameworkImpl) RunScorePlugins(ctx context.Context, state fwk.CycleState, pod *v1.Pod, nodes []fwk.NodeInfo) (ns []framework.NodePluginScores, status *fwk.Status) {
	... ...
	allNodePluginScores := make([]framework.NodePluginScores, len(nodes))
	numPlugins := len(f.scorePlugins)
	plugins := make([]framework.ScorePlugin, 0, numPlugins)
	
	// n个插件
	pluginToNodeScores := make(map[string]framework.NodeScoreList, numPlugins)
	for _, pl := range f.scorePlugins {
		if state.GetSkipScorePlugins().Has(pl.Name()) {
			continue
		}
		plugins = append(plugins, pl)
		// 每个插件对应m个节点
		pluginToNodeScores[pl.Name()] = make(framework.NodeScoreList, len(nodes))
	}
	ctx, cancel := context.WithCancel(ctx)
	defer cancel()
	errCh := parallelize.NewErrorChannel()

	if len(plugins) > 0 {
		... ...
		// 并行的为所有可行节点中的每个节点
		f.Parallelizer().Until(ctx, len(nodes), func(index int) {
			nodeInfo := nodes[index]
			nodeName := nodeInfo.Node().Name
			... ... 
			
			for _, pl := range plugins {
				... ...
				// 每个插件对每个节点打分
				s, status := f.runScorePlugin(ctx, pl, state, pod, nodeInfo)
				if !status.IsSuccess() {
					err := fmt.Errorf("plugin %q failed with: %w", pl.Name(), status.AsError())
					errCh.SendErrorWithCancel(err, cancel)
					return
				}
				// 每个插件、每个节点的索引 对应的节点名、分数
				pluginToNodeScores[pl.Name()][index] = framework.NodeScore{
					Name:  nodeName,
					Score: s,
				}
			}
		}, metrics.Score)
		if err := errCh.ReceiveError(); err != nil {
			return nil, fwk.AsStatus(fmt.Errorf("running Score plugins: %w", err))
		}
	}

	// 对Score插件打分的结果进行正则化
	f.Parallelizer().Until(ctx, len(plugins), func(index int) {
		pl := plugins[index]
		if pl.ScoreExtensions() == nil {
			return
		}
		nodeScoreList := pluginToNodeScores[pl.Name()]
		status := f.runScoreExtension(ctx, pl, state, pod, nodeScoreList)
		if !status.IsSuccess() {
			err := fmt.Errorf("plugin %q failed with: %w", pl.Name(), status.AsError())
			errCh.SendErrorWithCancel(err, cancel)
			return
		}
	}, metrics.Score)
	if err := errCh.ReceiveError(); err != nil {
		return nil, fwk.AsStatus(fmt.Errorf("running Normalize on Score plugins: %w", err))
	}

	// 对于每个节点
	f.Parallelizer().Until(ctx, len(nodes), func(index int) {
		nodePluginScores := framework.NodePluginScores{
			Name:   nodes[index].Node().Name,
			Scores: make([]framework.PluginScore, len(plugins)),
		}

		// 对应所有的Score插件，按插件权重聚合所有Score插件得分，得到每个节点的最终得分
		for i, pl := range plugins {
			weight := f.scorePluginWeight[pl.Name()]
			nodeScoreList := pluginToNodeScores[pl.Name()]
			score := nodeScoreList[index].Score

			if score > framework.MaxNodeScore || score < framework.MinNodeScore {
				err := fmt.Errorf("plugin %q returns an invalid score %v, it should in the range of [%v, %v] after normalizing", pl.Name(), score, framework.MinNodeScore, framework.MaxNodeScore)
				errCh.SendErrorWithCancel(err, cancel)
				return
			}
			weightedScore := score * int64(weight)
			nodePluginScores.Scores[i] = framework.PluginScore{
				Name:  pl.Name(),
				Score: weightedScore,
			}
			nodePluginScores.TotalScore += weightedScore
		}
		allNodePluginScores[index] = nodePluginScores
	}, metrics.Score)
	if err := errCh.ReceiveError(); err != nil {
		return nil, fwk.AsStatus(fmt.Errorf("applying score defaultWeights on Score plugins: %w", err))
	}

	return allNodePluginScores, nil
}

func (f *frameworkImpl) runScorePlugin(ctx context.Context, pl framework.ScorePlugin, state fwk.CycleState, pod *v1.Pod, nodeInfo fwk.NodeInfo) (int64, *fwk.Status) {
	... ...
	// 调用真正的插件
	s, status := pl.Score(ctx, state, pod, nodeInfo)
	... ...
	return s, status
}
```

# BalancedAllocation插件

先看 Score 函数

```go
// Score invoked at the score extension point.
func (ba *BalancedAllocation) Score(ctx context.Context, state fwk.CycleState, pod *v1.Pod, nodeInfo fwk.NodeInfo) (int64, *fwk.Status) {
	s, err := getBalancedAllocationPreScoreState(state)
	if err != nil {
		s = &balancedAllocationPreScoreState{podRequests: ba.calculatePodResourceRequestList(pod, ba.resources)}
		if ba.isBestEffortPod(s.podRequests) {
			return 0, nil
		}
	}

	// ba.score favors nodes with balanced resource usage rate.
	// It calculates the standard deviation for those resources and prioritizes the node based on how close the usage of those resources is to each other.
	// Detail: score = (1 - std) * MaxNodeScore, where std is calculated by the root square of Σ((fraction(i)-mean)^2)/len(resources)
	// The algorithm is partly inspired by:
	// "Wei Huang et al. An Energy Efficient Virtual Machine Placement Algorithm with Balanced Resource Utilization"
	return ba.score(ctx, pod, nodeInfo, s.podRequests)
}
```



