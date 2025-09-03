# k8s调用cri创建容器
## kubelet源码

kubelet会不断循环SyncPod，其主流程如下：

代码文件：kuberuntime_manager.go

```go
// SyncPod syncs the running pod into the desired pod by executing following steps:
//
//  1. Compute sandbox and container changes.
//  2. Kill pod sandbox if necessary.
//  3. Kill any containers that should not be running.
//  4. Create sandbox if necessary.
//  5. Create ephemeral containers.
//  6. Create init containers.
//  7. Resize running containers (if InPlacePodVerticalScaling==true)
//  8. Create normal containers.
func (m *kubeGenericRuntimeManager) SyncPod(ctx context.Context, pod *v1.Pod, podStatus *kubecontainer.PodStatus, pullSecrets []v1.Secret, backOff *flowcontrol.Backoff) (result kubecontainer.PodSyncResult) {
  // Step 1: Compute sandbox and container changes.
	podContainerChanges := m.computePodActions(ctx, pod, podStatus)
  ... ...
  start := func(ctx context.Context, typeName, metricLabel string, spec *startSpec) error {
		startContainerResult := kubecontainer.NewSyncResult(kubecontainer.StartContainer, spec.container.Name)
		result.AddSyncResult(startContainerResult)

		... ... 省略其他步骤
		klog.V(4).InfoS("Creating container in pod", "containerType", typeName, "container", spec.container, "pod", klog.KObj(pod))
		// NOTE (aramase) podIPs are populated for single stack and dual stack clusters. Send only podIPs.
		if msg, err := m.startContainer(ctx, podSandboxID, podSandboxConfig, spec, pod, podStatus, pullSecrets, podIP, podIPs); err != nil {
			... ...
			return err
		}

		return nil
	}

  // Step 8: start containers in podContainerChanges.ContainersToStart.
	for _, idx := range podContainerChanges.ContainersToStart {
		start(ctx, "container", metrics.Container, containerStartSpec(&pod.Spec.Containers[idx]))
	}
}
```

在看 startContainer 函数的具体实现

代码文件：kuberuntime_container.go

```go
func (m *kubeGenericRuntimeManager) startContainer(ctx context.Context, podSandboxID string, podSandboxConfig *runtimeapi.PodSandboxConfig, spec *startSpec, pod *v1.Pod, podStatus *kubecontainer.PodStatus, pullSecrets []v1.Secret, podIP string, podIPs []string) (string, error) {
  container := spec.container

	// Step 1: pull the image.
	imageRef, msg, err := m.imagePuller.EnsureImageExists(ctx, pod, container, pullSecrets, podSandboxConfig)
  ... ...

  // Step 2: create the container.
	// For a new container, the RestartCount should be 0
  ... ...
  // 产生容器的配置
  containerConfig, cleanupAction, err := m.generateContainerConfig(ctx, container, pod, restartCount, podIP, imageRef, podIPs, target)
	... ...
  // 调用容器运行时创建容器
	containerID, err := m.runtimeService.CreateContainer(ctx, podSandboxID, containerConfig, podSandboxConfig)
	... ...

	// Step 3: start the container. 调用容器运行时启动容器
	err = m.runtimeService.StartContainer(ctx, containerID)

}
```

这里仅详细分析如何生成容器的配置，因为本人对容器、cgroup之前的底层实现感兴趣。首先看
- 如何生成容器的配置
代码文件：kuberuntime_container.go
```go
func (m *kubeGenericRuntimeManager) generateContainerConfig(ctx context.Context, container *v1.Container, pod *v1.Pod, restartCount int, podIP, imageRef string, podIPs []string, nsTarget *kubecontainer.ContainerID) (*runtimeapi.ContainerConfig, func(), error) {
  opts, cleanupAction, err := m.runtimeHelper.GenerateRunContainerOptions(ctx, pod, container, podIP, podIPs)
  ... ...
  containerLogsPath := buildContainerLogsPath(container.Name, restartCount)
	restartCountUint32 := uint32(restartCount)
  // 此处生成容器的大部分配置
	config := &runtimeapi.ContainerConfig{
		Metadata: &runtimeapi.ContainerMetadata{
			Name:    container.Name,
			Attempt: restartCountUint32,
		},
		Image:       &runtimeapi.ImageSpec{Image: imageRef, UserSpecifiedImage: container.Image},
		Command:     command,
		Args:        args,
		WorkingDir:  container.WorkingDir,
		Labels:      newContainerLabels(container, pod),
		Annotations: newContainerAnnotations(container, pod, restartCount, opts),
		Devices:     makeDevices(opts),
		CDIDevices:  makeCDIDevices(opts),
		Mounts:      m.makeMounts(opts, container),
		LogPath:     containerLogsPath,
		Stdin:       container.Stdin,
		StdinOnce:   container.StdinOnce,
		Tty:         container.TTY,
	}

	// set platform specific configurations. 这里会设置cgroup相关的配置，重点关注
	if err := m.applyPlatformSpecificContainerConfig(config, container, pod, uid, username, nsTarget); err != nil {
		return nil, cleanupAction, err
	}
  ... ...
  return config, cleanupAction, nil
```

代码文件：kuberuntime_container_linux.go

```go
func (m *kubeGenericRuntimeManager) applyPlatformSpecificContainerConfig(config *runtimeapi.ContainerConfig, container *v1.Container, pod *v1.Pod, uid *int64, username string, nsTarget *kubecontainer.ContainerID) error {
	enforceMemoryQoS := false
	// Set memory.min and memory.high if MemoryQoS enabled with cgroups v2
	if utilfeature.DefaultFeatureGate.Enabled(kubefeatures.MemoryQoS) &&
		isCgroup2UnifiedMode() {
		enforceMemoryQoS = true
	}
	cl, err := m.generateLinuxContainerConfig(container, pod, uid, username, nsTarget, enforceMemoryQoS)
	if err != nil {
		return err
	}
	config.Linux = cl

	if utilfeature.DefaultFeatureGate.Enabled(kubefeatures.UserNamespacesSupport) {
		if cl.SecurityContext.NamespaceOptions.UsernsOptions != nil {
			for _, mount := range config.Mounts {
				mount.UidMappings = cl.SecurityContext.NamespaceOptions.UsernsOptions.Uids
				mount.GidMappings = cl.SecurityContext.NamespaceOptions.UsernsOptions.Gids
			}
		}
	}
	return nil
}

func (m *kubeGenericRuntimeManager) generateLinuxContainerConfig(container *v1.Container, pod *v1.Pod, uid *int64, username string, nsTarget *kubecontainer.ContainerID, enforceMemoryQoS bool) (*runtimeapi.LinuxContainerConfig, error) {
	sc, err := m.determineEffectiveSecurityContext(pod, container, uid, username)
	if err != nil {
		return nil, err
	}
	lc := &runtimeapi.LinuxContainerConfig{
		Resources:       m.generateLinuxContainerResources(pod, container, enforceMemoryQoS),
		SecurityContext: sc,
	}

	if nsTarget != nil && lc.SecurityContext.NamespaceOptions.Pid == runtimeapi.NamespaceMode_CONTAINER {
		lc.SecurityContext.NamespaceOptions.Pid = runtimeapi.NamespaceMode_TARGET
		lc.SecurityContext.NamespaceOptions.TargetId = nsTarget.ID
	}

	return lc, nil
}

func (m *kubeGenericRuntimeManager) generateLinuxContainerResources(pod *v1.Pod, container *v1.Container, enforceMemoryQoS bool) *runtimeapi.LinuxContainerResources {
	// set linux container resources
	var cpuRequest *resource.Quantity
	if _, cpuRequestExists := container.Resources.Requests[v1.ResourceCPU]; cpuRequestExists {
		cpuRequest = container.Resources.Requests.Cpu()
	}
	lcr := m.calculateLinuxResources(cpuRequest, container.Resources.Limits.Cpu(), container.Resources.Limits.Memory())

	lcr.OomScoreAdj = int64(qos.GetContainerOOMScoreAdjust(pod, container,
		int64(m.machineInfo.MemoryCapacity)))

	lcr.HugepageLimits = GetHugepageLimitsFromResources(container.Resources)

	// Configure swap for the container
	m.configureContainerSwapResources(lcr, pod, container)
  ... ...
return lcr
}

func (m *kubeGenericRuntimeManager) calculateLinuxResources(cpuRequest, cpuLimit, memoryLimit *resource.Quantity) *runtimeapi.LinuxContainerResources {
	resources := runtimeapi.LinuxContainerResources{}
	var cpuShares int64

	memLimit := memoryLimit.Value()

	// If request is not specified, but limit is, we want request to default to limit.
	// API server does this for new containers, but we repeat this logic in Kubelet
	// for containers running on existing Kubernetes clusters.
	if cpuRequest == nil && cpuLimit != nil {
		cpuShares = int64(cm.MilliCPUToShares(cpuLimit.MilliValue()))
	} else {
		// if cpuRequest.Amount is nil, then MilliCPUToShares will return the minimal number
		// of CPU shares.
		cpuShares = int64(cm.MilliCPUToShares(cpuRequest.MilliValue()))
	}
  // 设置cpu限制
	resources.CpuShares = cpuShares
	if memLimit != 0 {
    // 设置内存限制
		resources.MemoryLimitInBytes = memLimit
	}

	// 是否启用CFS
	if m.cpuCFSQuota {
		// if cpuLimit.Amount is nil, then the appropriate default value is returned
		// to allow full usage of cpu resource.
		cpuPeriod := int64(quotaPeriod)
		if utilfeature.DefaultFeatureGate.Enabled(kubefeatures.CPUCFSQuotaPeriod) {
			// kubeGenericRuntimeManager.cpuCFSQuotaPeriod is provided in time.Duration,
			// but we need to convert it to number of microseconds which is used by kernel.
			cpuPeriod = int64(m.cpuCFSQuotaPeriod.Duration / time.Microsecond)
		}
		cpuQuota := milliCPUToQuota(cpuLimit.MilliValue(), cpuPeriod)
		//在一个周期内允许使用的 CPU 时间，默认 -1 表示无限制。
		resources.CpuQuota = cpuQuota
		// 周期，默认 100ms (100000 µs)。
		resources.CpuPeriod = cpuPeriod
	}

	... ...
	return &resources
}


```
