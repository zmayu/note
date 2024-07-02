

# secrect、configmap 介绍


## kubelet中对于secrect、configmap代码流程

#### secectManager和configmap的创建策略
```go
switch kubeCfg.ConfigMapAndSecretChangeDetectionStrategy {
    //从apiServer watch增量资源
	case kubeletconfiginternal.WatchChangeDetectionStrategy:
		secretManager = secret.NewWatchingSecretManager(kubeDeps.KubeClient, klet.resyncInterval)
		configMapManager = configmap.NewWatchingConfigMapManager(kubeDeps.KubeClient, klet.resyncInterval)
    // TTL（过期时间）通过缓存过期的策略依据name从apiServer获取资源对象
	case kubeletconfiginternal.TTLCacheChangeDetectionStrategy:
		secretManager = secret.NewCachingSecretManager(
			kubeDeps.KubeClient, manager.GetObjectTTLFromNodeFunc(klet.GetNode))
		configMapManager = configmap.NewCachingConfigMapManager(
			kubeDeps.KubeClient, manager.GetObjectTTLFromNodeFunc(klet.GetNode))
    //仅在需要时直接从apiServer get资源对象。
	case kubeletconfiginternal.GetChangeDetectionStrategy:
		secretManager = secret.NewSimpleSecretManager(kubeDeps.KubeClient)
		configMapManager = configmap.NewSimpleConfigMapManager(kubeDeps.KubeClient)
	default:
		return nil, fmt.Errorf("unknown configmap and secret manager mode: %v", kubeCfg.ConfigMapAndSecretChangeDetectionStrategy)
	}
```





#### 创建secrect、configmap manager(以WatchChangeDetectionStrategy为例)

* 创建watch configmap manager
``` go
func NewWatchingConfigMapManager(kubeClient clientset.Interface, resyncInterval time.Duration) Manager {
	listConfigMap := func(namespace string, opts metav1.ListOptions) (runtime.Object, error) {
		return kubeClient.CoreV1().ConfigMaps(namespace).List(context.TODO(), opts)
	}
	watchConfigMap := func(namespace string, opts metav1.ListOptions) (watch.Interface, error) {
		return kubeClient.CoreV1().ConfigMaps(namespace).Watch(context.TODO(), opts)
	}
	newConfigMap := func() runtime.Object {
		return &v1.ConfigMap{}
	}
	isImmutable := func(object runtime.Object) bool {
		if configMap, ok := object.(*v1.ConfigMap); ok {
			return configMap.Immutable != nil && *configMap.Immutable
		}
		return false
	}
	gr := corev1.Resource("configmap")
	return &configMapManager{
        //todo 关键点：创建的是一个watchManager,watchManager会不断从apiServer观测对应资源的变化
		manager: manager.NewWatchBasedManager(listConfigMap, watchConfigMap, newConfigMap, isImmutable, gr, resyncInterval, getConfigMapNames),
	}
}
```

* 创建watch secret manager
  里面有listWatch的方法，最终的实现方式为WatchBasedManager，通过watchApiserver的方式获取secrect配置
```go 
func NewWatchingSecretManager(kubeClient clientset.Interface, resyncInterval time.Duration) Manager {
	listSecret := func(namespace string, opts metav1.ListOptions) (runtime.Object, error) {
		return kubeClient.CoreV1().Secrets(namespace).List(context.TODO(), opts)
	}
	watchSecret := func(namespace string, opts metav1.ListOptions) (watch.Interface, error) {
		return kubeClient.CoreV1().Secrets(namespace).Watch(context.TODO(), opts)
	}
	newSecret := func() runtime.Object {
		return &v1.Secret{}
	}
	isImmutable := func(object runtime.Object) bool {
		if secret, ok := object.(*v1.Secret); ok {
			return secret.Immutable != nil && *secret.Immutable
		}
		return false
	}
	gr := corev1.Resource("secret")
	return &secretManager{
		manager: manager.NewWatchBasedManager(listSecret, watchSecret, newSecret, isImmutable, gr, resyncInterval, getSecretNames),
	}
}
```


* 在syncPod启动pod流程时，将pod中的secrect和configmap注册到manager中。
``` go
	if !kl.podWorkers.IsPodTerminationRequested(pod.UID) {
		if kl.secretManager != nil {
			kl.secretManager.RegisterPod(pod)
		}
		if kl.configMapManager != nil {
			kl.configMapManager.RegisterPod(pod)
		}
	}
```
#### 注册secret和configmap
* 解析pod中引用的secret、configmap名称，向manager中注册对应资源
```go
func (c *cacheBasedManager) RegisterPod(pod *v1.Pod) {
	names := c.getReferencedObjects(pod)
	c.lock.Lock()
	defer c.lock.Unlock()
	for name := range names {
		c.objectStore.AddReference(pod.Namespace, name)
	}
	var prev *v1.Pod
	key := objectKey{namespace: pod.Namespace, name: pod.Name, uid: pod.UID}
	prev = c.registeredPods[key]
	c.registeredPods[key] = pod
	if prev != nil {
		for name := range c.getReferencedObjects(prev) {
			// On an update, the .Add() call above will have re-incremented the
			// ref count of any existing object, so any objects that are in both
			// names and prev need to have their ref counts decremented. Any that
			// are only in prev need to be completely removed. This unconditional
			// call takes care of both cases.
			c.objectStore.DeleteReference(prev.Namespace, name)
		}
	}
}
```
* reference 通过Reflector管理对应的configmap
  同一个secret、configmap资源对象被多个pod引用时，只会创建一个item对象，但是会通过item.refCount++ 记录当前资源对象引用的次数
```go
func (c *objectCache) AddReference(namespace, name string) {
	key := objectKey{namespace: namespace, name: name}

	// AddReference is called from RegisterPod thus it needs to be efficient.
	// Thus, it is only increasing refCount and in case of first registration
	// of a given object it starts corresponding reflector.
	// It's responsibility of the first Get operation to wait until the
	// reflector propagated the store.
	c.lock.Lock()
	defer c.lock.Unlock()
	item, exists := c.items[key]
	if !exists {
		item = c.newReflector(namespace, name)
		c.items[key] = item
	}
	item.refCount++
} 
```

* 创建reflector，对于每一个configmap、secrect都将会创建一个协程，由reflector启动listWatch，将watch到的增量资源存储到缓存中。
```go
func (c *objectCache) newReflector(namespace, name string) *objectCacheItem {
	fieldSelector := fields.Set{"metadata.name": name}.AsSelector().String()
	listFunc := func(options metav1.ListOptions) (runtime.Object, error) {
		options.FieldSelector = fieldSelector
		return c.listObject(namespace, options)
	}
	watchFunc := func(options metav1.ListOptions) (watch.Interface, error) {
		options.FieldSelector = fieldSelector
		return c.watchObject(namespace, options)
	}
	store := c.newStore()
	reflector := cache.NewNamedReflector(
		fmt.Sprintf("object-%q/%q", namespace, name),
		&cache.ListWatch{ListFunc: listFunc, WatchFunc: watchFunc},
		c.newObject(),
		store,
		0,
	)
	item := &objectCacheItem{
		refCount:  0,
		store:     store,
		reflector: reflector,
		hasSynced: func() (bool, error) { return store.hasSynced(), nil },
		stopCh:    make(chan struct{}),
	}
	go item.startReflector()
	return item
}

```


#### 获取secrect、configmap对象
*  在创建容器时，生成容器的环境变量时会获取容器依赖的secret和configmap
*  拉取镜像时是需要先获取镜像仓库的密钥。
```go 
func makeEnvironmentVariables(){
    configMap, err = kl.configMapManager.GetConfigMap(pod.Namespace, name)
    secret, err = kl.secretManager.GetSecret(pod.Namespace, name)
}

func getPullSecretsForPod(){
    secret, err := kl.secretManager.GetSecret(pod.Namespace, secretRef.Name)
}
```

* 从watch_based_manager cache中获取资源对象
```go 
func (c *objectCache) Get(namespace, name string) (runtime.Object, error) {
	key := objectKey{namespace: namespace, name: name}

	c.lock.RLock()
	item, exists := c.items[key]
	c.lock.RUnlock()

	if !exists {
		return nil, fmt.Errorf("object %q/%q not registered", namespace, name)
	}
	item.setLastAccessTime(c.clock.Now())

	item.restartReflectorIfNeeded()
	if err := wait.PollImmediate(10*time.Millisecond, time.Second, item.hasSynced); err != nil {
		return nil, fmt.Errorf("failed to sync %s cache: %v", c.groupResource.String(), err)
	}
	obj, exists, err := item.store.GetByKey(c.key(namespace, name))
	if err != nil {
		return nil, err
	}
	if !exists {
		return nil, apierrors.NewNotFound(c.groupResource, name)
	}
	if object, ok := obj.(runtime.Object); ok {
		if c.isImmutable(object) {
			item.setImmutable()
			if item.stop() {
				klog.V(4).InfoS("Stopped watching for changes - object is immutable", "obj", klog.KRef(namespace, name))
			}
		}
		return object, nil
	}
	return nil, fmt.Errorf("unexpected object type: %v", obj)
}
```

#### 取消注册secret和configmap
* 在syncTerminatedPod同步终止pod时会执行secret和configmap的清理工作
```go 
    syncTerminatedPod(){
        if kl.secretManager != nil {
		    kl.secretManager.UnregisterPod(pod)
	    }
	    if kl.configMapManager != nil {
		    kl.configMapManager.UnregisterPod(pod)
	    }       
    }
```

* 删除reference，即删除secrect、configmap对象。
  当pod资源终止时，就要将对对应资源的引用资源减一，如果item.refCount引用次数为0，那么就说明当前node节点上的所有pod对该资源应用没有任何引用了，就可以删除对应的item
```go 
func (c *objectCache) DeleteReference(namespace, name string) {
	key := objectKey{namespace: namespace, name: name}

	c.lock.Lock()
	defer c.lock.Unlock()
	if item, ok := c.items[key]; ok {
		item.refCount--
		if item.refCount == 0 {
			// Stop the underlying reflector.
			item.stop()
			delete(c.items, key)
		}
	}
}
```

