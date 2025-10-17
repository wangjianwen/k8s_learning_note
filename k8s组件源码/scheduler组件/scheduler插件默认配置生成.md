# 概述

在 Kubernetes v1.23 版本之前，可以使用调度策略来指定 predicates 和 priorities 进程。 例如，可以通过运行 kube-scheduler --policy-config-file <filename> 或者 kube-scheduler --policy-configmap <ConfigMap> 设置调度策略。
但是从 Kubernetes v1.23 版本开始，不再支持这种调度策略。 同样地也不支持相关的 policy-config-file、policy-configmap、policy-configmap-namespace 和 use-legacy-policy-config 标志。 你可以通过使用调度配置来实现类似的行为。

也就是可以通过编写如下配置文件来不同阶段的调度行为：

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
  - plugins:
      score:
        disabled:
        - name: PodTopologySpread
        enabled:
        - name: MyCustomPluginA
          weight: 2
        - name: MyCustomPluginB
          weight: 1
```

比如上述配置文件自定义了 score阶段启用的插件：MyCustomPluginA，MyCustomPluginB；禁止使用的插件：PodTopologySpread 。

scheduler组件把pod调度到某1个节点的整个过程分为如下几个阶段：

```
┌────────────────────────────┐
│         PreEnqueue         │ ← Pod进入调度器队列前
└─────────────┬──────────────┘
↓
┌────────────────────────────┐
│         QueueSort          │ ← 确定调度顺序
└─────────────┬──────────────┘
↓
┌────────────────────────────┐
│         PreFilter          │ ← 调度前准备
└─────────────┬──────────────┘
↓
┌────────────────────────────┐
│          Filter            │ ← 过滤不可用节点
└─────────────┬──────────────┘
↓
┌────────────────────────────┐
│         PostFilter         │ ← 没有可用节点时尝试抢占
└─────────────┬──────────────┘
↓
┌────────────────────────────┐
│         PreScore           │ ← 打分前准备
└─────────────┬──────────────┘
↓
┌────────────────────────────┐
│           Score            │ ← 选出最优节点
└─────────────┬──────────────┘
↓
┌────────────────────────────┐
│          Reserve           │ ← 临时保留资源
└─────────────┬──────────────┘
↓
┌────────────────────────────┐
│           Permit           │ ← 检查或等待许可
└─────────────┬──────────────┘
↓
┌────────────────────────────┐
│          PreBind           │ ← 准备绑定
└─────────────┬──────────────┘
↓
┌────────────────────────────┐
│            Bind            │ ← 执行绑定
└─────────────┬──────────────┘
↓
┌────────────────────────────┐
│          PostBind          │ ← 绑定后处理
└────────────────────────────┘

```

可以通过配置来配置scheduler不同阶段的调度行为，每个阶段都可在一个扩展点中进行扩展，调度插件通过实现一个或多个扩展点，来提供调度行为。
有哪些扩展点呢？

# 问题：scheduler组件如何配置各个阶段的插件及扩展点？

查看scheduler组件的启动命令行：

```yaml
- command:
    - kube-scheduler
    - --authentication-kubeconfig=/etc/kubernetes/scheduler.conf
    - --authorization-kubeconfig=/etc/kubernetes/scheduler.conf
    - --bind-address=127.0.0.1
    - --kubeconfig=/etc/kubernetes/scheduler.conf
    - --leader-elect=true
```

可以看出，kube-scheduler命令行在启动时是没有指定配置文件。既然没有配置 KubeSchedulerConfiguration 中profiles，那默认配置到底是什么？带着这个问题去看源码

# scheduler初始化插件配置

从源代码来看，scheduler默认仅配置了 profiles的 plugins.MultiPoint.Enabled 字段。 包括如下插件：
- SchedulingGates
- NodeUnschedulable
- NodeName
- TaintToleration
- NodeAffinity
- NodePorts
- NodeResourcesFit
- VolumeRestrictions
- NodeVolumeLimits
- VolumeZone
- PodTopologySpread
- InterPodAffinity
- DefaultPreemption
- NodeResourcesBalancedAllocation
- ImageLocality
- ImageLocality

下面是 scheduler初始化插件配置整个过程

```go
//  
func New(ctx context.Context,
	client clientset.Interface,
	informerFactory informers.SharedInformerFactory,
	dynInformerFactory dynamicinformer.DynamicSharedInformerFactory,
	recorderFactory profile.RecorderFactory,
	opts ...Option) (*Scheduler, error) {

	... ... 

	options := defaultSchedulerOptions
	for _, opt := range opts {
		opt(&options)
	}

	if options.applyDefaultProfile {
		var versionedCfg configv1.KubeSchedulerConfiguration
		// 调用  configv1.KubeSchedulerConfiguration 注册的typeDefaulterFunc， versionedCfg 的 Profiles 被定义好，特别是profile.plugins被定义了
		scheme.Scheme.Default(&versionedCfg)
		// 把 configv1.KubeSchedulerConfiguration的配置塞进 schedulerapi.KubeSchedulerConfiguration
		cfg := schedulerapi.KubeSchedulerConfiguration{}
		if err := scheme.Scheme.Convert(&versionedCfg, &cfg, nil); err != nil {
			return nil, err
		}
		// options.profiles 也被赋值为 cfg.Profiles
		options.profiles = cfg.Profiles
	}
	... ...
	
}

func (s *Scheme) Default(src Object) {
    if fn, ok := s.defaulterFuncs[reflect.TypeOf(src)]; ok {
        // 这里调用 configv1.KubeSchedulerConfiguration 注册的typeDefaulterFunc
        fn(src)
    }
}

```
通过分析上面代码可以看出，scheduler 在定义 options.profiles，会通过调用scheme为类型configv1.KubeSchedulerConfiguration 注册的typeDefaulterFunc函数，得到options.profiles配置。
下面带着如下2个问题阅读代码：
- 注册的typeDefaulterFunc函数到底是什么
- 注册的typeDefaulterFunc函数实现了什么功能

## typeDefaulterFunc函数的注册

scheduler 是如何注册 typeDefaulterFunc 函数的呢？

pkg/scheduler/apis/v1/register.go

```go

func init() {
	localSchemeBuilder.Register(addDefaultingFuncs)
}

func addDefaultingFuncs(scheme *runtime.Scheme) error {
	return RegisterDefaults(scheme)
}

func RegisterDefaults(scheme *runtime.Scheme) error {
	... ...
	// 这里在scheme中注册 configv1.KubeSchedulerConfiguration 的typeDefaulterFunc 
	scheme.AddTypeDefaultingFunc(&configv1.KubeSchedulerConfiguration{}, func(obj interface{}) {
		SetObjectDefaults_KubeSchedulerConfiguration(obj.(*configv1.KubeSchedulerConfiguration))
	})
	scheme.AddTypeDefaultingFunc(&configv1.NodeResourcesBalancedAllocationArgs{}, func(obj interface{}) {
		SetObjectDefaults_NodeResourcesBalancedAllocationArgs(obj.(*configv1.NodeResourcesBalancedAllocationArgs))
	})
	scheme.AddTypeDefaultingFunc(&configv1.NodeResourcesFitArgs{}, func(obj interface{}) { SetObjectDefaults_NodeResourcesFitArgs(obj.(*configv1.NodeResourcesFitArgs)) })
	scheme.AddTypeDefaultingFunc(&configv1.PodTopologySpreadArgs{}, func(obj interface{}) { SetObjectDefaults_PodTopologySpreadArgs(obj.(*configv1.PodTopologySpreadArgs)) })
	scheme.AddTypeDefaultingFunc(&configv1.VolumeBindingArgs{}, func(obj interface{}) { SetObjectDefaults_VolumeBindingArgs(obj.(*configv1.VolumeBindingArgs)) })
	return nil
}


func (s *Scheme) AddTypeDefaultingFunc(srcType Object, fn func(interface{})) {
	s.defaulterFuncs[reflect.TypeOf(srcType)] = fn
}
```

这里最终在scheme的defaulterFuncs map中注册了 configv1.KubeSchedulerConfiguration的处理函数为：`SetObjectDefaults_KubeSchedulerConfiguration`

## typeDefaulterFunc函数的实现

再看 SetObjectDefaults_KubeSchedulerConfiguration 的实现，其实质是定义 prof.Plugins 

```go
func SetDefaults_KubeSchedulerConfiguration(obj *configv1.KubeSchedulerConfiguration) {
	logger := klog.TODO() // called by generated code that doesn't pass a logger. See #115724
	if obj.Parallelism == nil {
		obj.Parallelism = ptr.To[int32](16)
	}

	if len(obj.Profiles) == 0 {
		// 插入默认 profile
		obj.Profiles = append(obj.Profiles, configv1.KubeSchedulerProfile{})
	}
	// Only apply a default scheduler name when there is a single profile.
	// Validation will ensure that every profile has a non-empty unique name.
	if len(obj.Profiles) == 1 && obj.Profiles[0].SchedulerName == nil {
		obj.Profiles[0].SchedulerName = ptr.To(v1.DefaultSchedulerName)
	}

	// Add the default set of plugins and apply the configuration.
	for i := range obj.Profiles {
		prof := &obj.Profiles[i]
		// 初始化默认插件 
		setDefaults_KubeSchedulerProfile(logger, prof)
	}

	... ...
}

func setDefaults_KubeSchedulerProfile(logger klog.Logger, prof *configv1.KubeSchedulerProfile) {
	// 定义prof的插件
	prof.Plugins = mergePlugins(logger, getDefaultPlugins(), prof.Plugins)
	... ...
}

func getDefaultPlugins() *v1.Plugins {
	plugins := &v1.Plugins{
		// 仅定义 MultiPoint 启用的插件名称
		MultiPoint: v1.PluginSet{
			Enabled: []v1.Plugin{
				{Name: names.SchedulingGates},
				{Name: names.PrioritySort},
				{Name: names.NodeUnschedulable},
				{Name: names.NodeName},
				{Name: names.TaintToleration, Weight: ptr.To[int32](3)},
				{Name: names.NodeAffinity, Weight: ptr.To[int32](2)},
				{Name: names.NodePorts},
				{Name: names.NodeResourcesFit, Weight: ptr.To[int32](1)},
				{Name: names.VolumeRestrictions},
				{Name: names.NodeVolumeLimits},
				{Name: names.VolumeBinding},
				{Name: names.VolumeZone},
				{Name: names.PodTopologySpread, Weight: ptr.To[int32](2)},
				{Name: names.InterPodAffinity, Weight: ptr.To[int32](2)},
				{Name: names.DefaultPreemption},
				{Name: names.NodeResourcesBalancedAllocation, Weight: ptr.To[int32](1)},
				{Name: names.ImageLocality, Weight: ptr.To[int32](1)},
				{Name: names.DefaultBinder},
			},
		},
	}
	applyFeatureGates(plugins)

	return plugins
}
```

最终可以看出，typeDefaulterFunc函数的实现中定义了应用于多个扩展点的插件名称及权重，包括： BalancedAllocation，NodeAffinity等插件。
那多个扩展点（multi）的插件 如何最终转换为scheduler已定义的阶段的插件呢？也就是最终转换为类似下面的配置呢？

```yaml
kind: KubeSchedulerConfiguration
profiles:
  - plugins:
      preFilter:
        enabled: 
          ... 
      filter:
          ...
      score:
        disabled:
        - name: PodTopologySpread
        enabled:
        - name: MyCustomPluginA
          weight: 2
        - name: MyCustomPluginB
          weight: 1
```

# scheduler插件配置最终生成

```go
func New(ctx context.Context,
	client clientset.Interface,
	informerFactory informers.SharedInformerFactory,
	dynInformerFactory dynamicinformer.DynamicSharedInformerFactory,
	recorderFactory profile.RecorderFactory,
	opts ...Option) (*Scheduler, error) {

	... ...
	// 定义
	registry := frameworkplugins.NewInTreeRegistry()
	
	... ...


	profiles, err := profile.NewMap(ctx, options.profiles, registry, recorderFactory,
		frameworkruntime.WithComponentConfigVersion(options.componentConfigVersion),
		frameworkruntime.WithClientSet(client),
		frameworkruntime.WithKubeConfig(options.kubeConfig),
		frameworkruntime.WithInformerFactory(informerFactory),
		frameworkruntime.WithSharedDRAManager(draManager),
		frameworkruntime.WithSnapshotSharedLister(snapshot),
		frameworkruntime.WithCaptureProfile(frameworkruntime.CaptureProfile(options.frameworkCapturer)),
		frameworkruntime.WithParallelism(int(options.parallelism)),
		frameworkruntime.WithExtenders(extenders),
		frameworkruntime.WithMetricsRecorder(metricsRecorder),
		frameworkruntime.WithWaitingPods(waitingPods),
		frameworkruntime.WithAPIDispatcher(apiDispatcher),
	)
	... ...

	sched := &Scheduler{
		Cache:                                  schedulerCache,
		client:                                 client,
		nodeInfoSnapshot:                       snapshot,
		percentageOfNodesToScore:               options.percentageOfNodesToScore,
		Extenders:                              extenders,
		StopEverything:                         stopEverything,
		SchedulingQueue:                        podQueue,
		Profiles:                               profiles,
		logger:                                 logger,
		APIDispatcher:                          apiDispatcher,
		nominatedNodeNameForExpectationEnabled: feature.DefaultFeatureGate.Enabled(features.NominatedNodeNameForExpectation),
	}
	... ...

	return sched, nil
}
```

首先看 registry 的定义，定义插件名字及创建插件的函数

```go
func NewInTreeRegistry() runtime.Registry {
	fts := plfeature.NewSchedulerFeaturesFromGates(feature.DefaultFeatureGate)
	registry := runtime.Registry{
		dynamicresources.Name:                runtime.FactoryAdapter(fts, dynamicresources.New),
		imagelocality.Name:                   imagelocality.New,
		tainttoleration.Name:                 runtime.FactoryAdapter(fts, tainttoleration.New),
		nodename.Name:                        runtime.FactoryAdapter(fts, nodename.New),
		nodeports.Name:                       runtime.FactoryAdapter(fts, nodeports.New),
		nodeaffinity.Name:                    runtime.FactoryAdapter(fts, nodeaffinity.New),
		podtopologyspread.Name:               runtime.FactoryAdapter(fts, podtopologyspread.New),
		nodeunschedulable.Name:               runtime.FactoryAdapter(fts, nodeunschedulable.New),
		noderesources.Name:                   runtime.FactoryAdapter(fts, noderesources.NewFit),
		noderesources.BalancedAllocationName: runtime.FactoryAdapter(fts, noderesources.NewBalancedAllocation),
		volumebinding.Name:                   runtime.FactoryAdapter(fts, volumebinding.New),
		volumerestrictions.Name:              runtime.FactoryAdapter(fts, volumerestrictions.New),
		volumezone.Name:                      runtime.FactoryAdapter(fts, volumezone.New),
		nodevolumelimits.CSIName:             runtime.FactoryAdapter(fts, nodevolumelimits.NewCSI),
		interpodaffinity.Name:                runtime.FactoryAdapter(fts, interpodaffinity.New),
		queuesort.Name:                       queuesort.New,
		defaultbinder.Name:                   defaultbinder.New,
		defaultpreemption.Name:               runtime.FactoryAdapter(fts, defaultpreemption.New),
		schedulinggates.Name:                 runtime.FactoryAdapter(fts, schedulinggates.New),
	}

	return registry
}
```

再看 NewMap 函数

```go
func NewMap(ctx context.Context, cfgs []config.KubeSchedulerProfile, r frameworkruntime.Registry, recorderFact RecorderFactory,
	opts ...frameworkruntime.Option) (Map, error) {
	m := make(Map)
	v := cfgValidator{m: m}

	for _, cfg := range cfgs {
		// 为每个调度定义profile，本文仅关注默认调度器的profile
		p, err := newProfile(ctx, cfg, r, recorderFact, opts...)
		if err != nil {
			return nil, fmt.Errorf("creating profile for scheduler name %s: %v", cfg.SchedulerName, err)
		}
		if err := v.validate(cfg, p); err != nil {
			return nil, err
		}
		m[cfg.SchedulerName] = p
	}
	return m, nil
}

func newProfile(ctx context.Context, cfg config.KubeSchedulerProfile, r frameworkruntime.Registry, recorderFact RecorderFactory,
	opts ...frameworkruntime.Option) (framework.Framework, error) {
	recorder := recorderFact(cfg.SchedulerName)
	opts = append(opts, frameworkruntime.WithEventRecorder(recorder))
	return frameworkruntime.NewFramework(ctx, r, &cfg, opts...)
}
```

```go
unc NewFramework(ctx context.Context, r Registry, profile *config.KubeSchedulerProfile, opts ...Option) (framework.Framework, error) {
	options := defaultFrameworkOptions(ctx.Done())
	... ...
	f := &frameworkImpl{
		registry:             r,
		snapshotSharedLister: options.snapshotSharedLister,
		scorePluginWeight:    make(map[string]int),
		waitingPods:          options.waitingPods,
		clientSet:            options.clientSet,
		kubeConfig:           options.kubeConfig,
		eventRecorder:        options.eventRecorder,
		informerFactory:      options.informerFactory,
		sharedDRAManager:     options.sharedDRAManager,
		metricsRecorder:      options.metricsRecorder,
		extenders:            options.extenders,
		PodNominator:         options.podNominator,
		PodActivator:         options.podActivator,
		apiDispatcher:        options.apiDispatcher,
		parallelizer:         options.parallelizer,
		logger:               logger,
	}

	... ...

	// get needed plugins from config
	pg := f.pluginsNeeded(profile.Plugins)

	... ... 
	// 根据 registry 定义的插件列表，初始化 pluginsMap （插件名 -> 真正的插件）
	f.pluginsMap = make(map[string]framework.Plugin)
	for name, factory := range r {
		// initialize only needed plugins.
		if !pg.Has(name) {
			continue
		}

		args := pluginConfig[name]
		if args != nil {
			outputProfile.PluginConfig = append(outputProfile.PluginConfig, config.PluginConfig{
				Name: name,
				Args: args,
			})
		}
		// 创建插件
		p, err := factory(ctx, args, f)
		if err != nil {
			return nil, fmt.Errorf("initializing plugin %q: %w", name, err)
		}
		// 塞进pluginsMap： 插件名 -> 真正的插件
		f.pluginsMap[name] = p

		f.fillEnqueueExtensions(p)
	}

	// 初始化插件扩展点，默认调度器插件扩展点在初始时为空
	for _, e := range f.getExtensionPoints(profile.Plugins) {
		if err := updatePluginList(e.slicePtr, *e.plugins, f.pluginsMap); err != nil {
			return nil, err
		}
	}

	// 把多个扩展点的插件按其实现进行扩展，比如 BalancedAllocation 扩展了 score， preScore 
	if len(profile.Plugins.MultiPoint.Enabled) > 0 {
		if err := f.expandMultiPointPlugins(logger, profile); err != nil {
			return nil, err
		}
	}

	... ... 
	return f, nil
}
```

expandMultiPointPlugins 是默认调度器配置profile的核心实现，其中 getExtensionPoints 包含：

```go
return []extensionPoint{
    {&plugins.PreFilter, &f.preFilterPlugins},
    {&plugins.Filter, &f.filterPlugins},
    {&plugins.PostFilter, &f.postFilterPlugins},
    {&plugins.Reserve, &f.reservePlugins},
    {&plugins.PreScore, &f.preScorePlugins},
    {&plugins.Score, &f.scorePlugins},
    {&plugins.PreBind, &f.preBindPlugins},
    {&plugins.Bind, &f.bindPlugins},
    {&plugins.PostBind, &f.postBindPlugins},
    {&plugins.Permit, &f.permitPlugins},
    {&plugins.PreEnqueue, &f.preEnqueuePlugins},
    {&plugins.QueueSort, &f.queueSortPlugins},
}

```

```go
func (f *frameworkImpl) expandMultiPointPlugins(logger klog.Logger, profile *config.KubeSchedulerProfile) error {
	// initialize MultiPoint plugins
	// 对应所有的插件扩展点：包括 PreFilter，Filter, PostFilter，PreScore，Score 等
	for _, e := range f.getExtensionPoints(profile.Plugins) {
		// 通过反射获取到frameworkImpl字段的 preFilterPlugins 等slice指针对应的变量
		plugins := reflect.ValueOf(e.slicePtr).Elem()
		// 获取到插件类型
		pluginType := plugins.Type().Elem()
		// build enabledSet of plugins already registered via normal extension points
		// to check double registration
		enabledSet := newOrderedSet()
		for _, plugin := range e.plugins.Enabled {
			enabledSet.insert(plugin.Name)
		}

		disabledSet := sets.New[string]()
		for _, disabledPlugin := range e.plugins.Disabled {
			disabledSet.Insert(disabledPlugin.Name)
		}
		if disabledSet.Has("*") {
			logger.V(4).Info("Skipped MultiPoint expansion because all plugins are disabled for extension point", "extension", pluginType)
			continue
		}

		// track plugins enabled via multipoint separately from those enabled by specific extensions,
		// so that we can distinguish between double-registration and explicit overrides
		multiPointEnabled := newOrderedSet()
		overridePlugins := newOrderedSet()
		// 获取 profile.Plugins.MultiPoint启用的插件列表
		for _, ep := range profile.Plugins.MultiPoint.Enabled {
			pg, ok := f.pluginsMap[ep.Name]
			if !ok {
				return fmt.Errorf("%s %q does not exist", pluginType.Name(), ep.Name)
			}

			// 如果名字为ep.Name插件实现了当前插件类型
			if !reflect.TypeOf(pg).Implements(pluginType) {
				continue
			}

			// a plugin that's enabled via MultiPoint can still be disabled for specific extension points
			if disabledSet.Has(ep.Name) {
				logger.V(4).Info("Skipped disabled plugin for extension point", "plugin", ep.Name, "extension", pluginType)
				continue
			}

			// if this plugin has already been enabled by the specific extension point,
			// the user intent is to override the default plugin or make some other explicit setting.
			// Either way, discard the MultiPoint value for this plugin.
			// This maintains expected behavior for overriding default plugins (see https://github.com/kubernetes/kubernetes/pull/99582)
			if enabledSet.has(ep.Name) {
				overridePlugins.insert(ep.Name)
				logger.Info("MultiPoint plugin is explicitly re-configured; overriding", "plugin", ep.Name)
				continue
			}

			// if this plugin is already registered via MultiPoint, then this is
			// a double registration and an error in the config.
			if multiPointEnabled.has(ep.Name) {
				return fmt.Errorf("plugin %q already registered as %q", ep.Name, pluginType.Name())
			}

			// 加入 multiPointEnabled
			multiPointEnabled.insert(ep.Name)
		}

		// Reorder plugins. Here is the expected order:
		// - part 1: overridePlugins. Their order stay intact as how they're specified in regular extension point.
		// - part 2: multiPointEnabled - i.e., plugin defined in multipoint but not in regular extension point.
		// - part 3: other plugins (excluded by part 1 & 2) in regular extension point.
		// newPlugins 使用反射创建 frameworkImpl的比如preFilterPlugins slice字段
		newPlugins := reflect.New(reflect.TypeOf(e.slicePtr).Elem()).Elem()
		// part 1
		for _, name := range slice.CopyStrings(enabledSet.list) {
			if overridePlugins.has(name) {
				newPlugins = reflect.Append(newPlugins, reflect.ValueOf(f.pluginsMap[name]))
				enabledSet.delete(name)
			}
		}
		// part 2
		for _, name := range multiPointEnabled.list {
			newPlugins = reflect.Append(newPlugins, reflect.ValueOf(f.pluginsMap[name]))
		}
		// part 3
		for _, name := range enabledSet.list {
			newPlugins = reflect.Append(newPlugins, reflect.ValueOf(f.pluginsMap[name]))
		}
		// // 通过反射设置 frameworkImpl字段的 preFilterPlugins 等slice 的值
		plugins.Set(newPlugins)
	}
	return nil
}
```

# 小结

scheduler插件默认配置生成包括如下几个过程：
- 定义每个profile.plugins.MultiPoint.enabled，也就是定义启用了哪些多点插件（插件名称、权重）
- 定义好每个profile的registry（插件名称、插件new函数）
- 根据 registry 定义的插件列表，初始化 pluginsMap（插件名称->使用插件new函数初始化的插件）
- 根据 profile.Plugins.MultiPoint.Enabled、pluginsMap， 利用反射，生成每个profile的plugins