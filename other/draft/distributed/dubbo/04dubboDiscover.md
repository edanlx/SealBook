Invoker是Dubbo中的实体域，也就是真实存在的。其他模型都向它靠拢或转换成它，它也就代表一个可执行体，可向它发起invoke调用。在服务提供方，Invoker用于调用服务提供类。在服务消费方，Invoker用于执行远程调用。

## 4.org.apache.dubbo.config.spring.ReferenceBean#getObject
```java
public Object getObject() {
    if (lazyProxy == null) {
        createLazyProxy();
    }
    return lazyProxy;
}
```
- org.apache.dubbo.config.spring.ReferenceBean#createLazyProxy
```java
private void createLazyProxy() {

    //set proxy interfaces
    //see also: org.apache.dubbo.rpc.proxy.AbstractProxyFactory.getProxy(org.apache.dubbo.rpc.Invoker<T>, boolean)
    ProxyFactory proxyFactory = new ProxyFactory();
    proxyFactory.setTargetSource(new DubboReferenceLazyInitTargetSource());
    proxyFactory.addInterface(interfaceClass);
    Class<?>[] internalInterfaces = AbstractProxyFactory.getInternalInterfaces();
    for (Class<?> anInterface : internalInterfaces) {
        proxyFactory.addInterface(anInterface);
    }
    if (!StringUtils.isEquals(interfaceClass.getName(), interfaceName)) {
        //add service interface
        try {
            Class<?> serviceInterface = ClassUtils.forName(interfaceName, beanClassLoader);
            proxyFactory.addInterface(serviceInterface);
        } catch (ClassNotFoundException e) {
            // generic call maybe without service interface class locally
        }
    }

    this.lazyProxy = proxyFactory.getProxy(this.beanClassLoader);
}
```
- org.apache.dubbo.config.spring.ReferenceBean.DubboReferenceLazyInitTargetSource#createObject
```java
protected Object createObject() throws Exception {
    return getCallProxy();
}
```
- org.apache.dubbo.config.spring.ReferenceBean#getCallProxy
```java
private Object getCallProxy() throws Exception {
    if (referenceConfig == null) {
        throw new IllegalStateException("ReferenceBean is not ready yet, please make sure to call reference interface method after dubbo is started.");
    }
    //get reference proxy
    return ReferenceConfigCache.getCache().get(referenceConfig);
}
```
- org.apache.dubbo.config.utils.ReferenceConfigCache#get(org.apache.dubbo.config.ReferenceConfigBase\<T\>)
```java
public <T> T get(ReferenceConfigBase<T> referenceConfig) {
    String key = generator.generateKey(referenceConfig);
    Class<?> type = referenceConfig.getInterfaceClass();

    proxies.computeIfAbsent(type, _t -> new ConcurrentHashMap<>());

    ConcurrentMap<String, Object> proxiesOfType = proxies.get(type);
    proxiesOfType.computeIfAbsent(key, _k -> {
    	// 如果不存在则创建
        Object proxy = referenceConfig.get();
        referredReferences.put(key, referenceConfig);
        return proxy;
    });

    return (T) proxiesOfType.get(key);
}
```
- org.apache.dubbo.config.ReferenceConfig#get
```java
public synchronized T get() {
    if (destroyed) {
        throw new IllegalStateException("The invoker of ReferenceConfig(" + url + ") has already destroyed!");
    }

    if (ref == null) {
        init();
    }

    return ref;
}
```
## 5.org.apache.dubbo.config.ReferenceConfig#init
```java
protected synchronized void init() {
    if (initialized) {
        return;
    }

    // Using DubboBootstrap API will associate bootstrap when registering reference.
    // Loading by Spring context will associate bootstrap in afterPropertiesSet() method.
    // Initializing bootstrap here only for compatible with old API usages.
    if (bootstrap == null) {
        bootstrap = DubboBootstrap.getInstance();
        bootstrap.initialize();
        bootstrap.reference(this);
    }

    // check bootstrap state
    if (!bootstrap.isInitialized()) {
        throw new IllegalStateException("DubboBootstrap is not initialized");
    }

    if (!this.isRefreshed()) {
        this.refresh();
    }

    //init serivceMetadata
    initServiceMetadata(consumer);
    serviceMetadata.setServiceType(getServiceInterfaceClass());
    // TODO, uncomment this line once service key is unified
    serviceMetadata.setServiceKey(URL.buildKey(interfaceName, group, version));

    ServiceRepository repository = ApplicationModel.getServiceRepository();
    ServiceDescriptor serviceDescriptor = repository.registerService(interfaceClass);
    repository.registerConsumer(
            serviceMetadata.getServiceKey(),
            serviceDescriptor,
            this,
            null,
            serviceMetadata);


    Map<String, String> map = new HashMap<String, String>();
    map.put(SIDE_KEY, CONSUMER_SIDE);

    ReferenceConfigBase.appendRuntimeParameters(map);
    if (!ProtocolUtils.isGeneric(generic)) {
        String revision = Version.getVersion(interfaceClass, version);
        if (revision != null && revision.length() > 0) {
            map.put(REVISION_KEY, revision);
        }

        String[] methods = Wrapper.getWrapper(interfaceClass).getMethodNames();
        if (methods.length == 0) {
            logger.warn("No method found in service interface " + interfaceClass.getName());
            map.put(METHODS_KEY, ANY_VALUE);
        } else {
            map.put(METHODS_KEY, StringUtils.join(new HashSet<String>(Arrays.asList(methods)), COMMA_SEPARATOR));
        }
    }
    map.put(INTERFACE_KEY, interfaceName);
    AbstractConfig.appendParameters(map, getMetrics());
    AbstractConfig.appendParameters(map, getApplication());
    AbstractConfig.appendParameters(map, getModule());
    // remove 'default.' prefix for configs from ConsumerConfig
    // appendParameters(map, consumer, Constants.DEFAULT_KEY);
    AbstractConfig.appendParameters(map, consumer);
    AbstractConfig.appendParameters(map, this);
    MetadataReportConfig metadataReportConfig = getMetadataReportConfig();
    if (metadataReportConfig != null && metadataReportConfig.isValid()) {
        map.putIfAbsent(METADATA_KEY, REMOTE_METADATA_STORAGE_TYPE);
    }
    Map<String, AsyncMethodInfo> attributes = null;
    if (CollectionUtils.isNotEmpty(getMethods())) {
        attributes = new HashMap<>();
        for (MethodConfig methodConfig : getMethods()) {
            AbstractConfig.appendParameters(map, methodConfig, methodConfig.getName());
            String retryKey = methodConfig.getName() + ".retry";
            if (map.containsKey(retryKey)) {
                String retryValue = map.remove(retryKey);
                if ("false".equals(retryValue)) {
                    map.put(methodConfig.getName() + ".retries", "0");
                }
            }
            AsyncMethodInfo asyncMethodInfo = AbstractConfig.convertMethodConfig2AsyncInfo(methodConfig);
            if (asyncMethodInfo != null) {
//                    consumerModel.getMethodModel(methodConfig.getName()).addAttribute(ASYNC_KEY, asyncMethodInfo);
                attributes.put(methodConfig.getName(), asyncMethodInfo);
            }
        }
    }

    String hostToRegistry = ConfigUtils.getSystemProperty(DUBBO_IP_TO_REGISTRY);
    if (StringUtils.isEmpty(hostToRegistry)) {
        hostToRegistry = NetUtils.getLocalHost();
    } else if (isInvalidLocalHost(hostToRegistry)) {
        throw new IllegalArgumentException(
                "Specified invalid registry ip from property:" + DUBBO_IP_TO_REGISTRY + ", value:" + hostToRegistry);
    }
    // 当前所引入的服务信息都在map内
    map.put(REGISTER_IP_KEY, hostToRegistry);

    serviceMetadata.getAttachments().putAll(map);
    // 创建代理对象，默认javassist生成invoker
    ref = createProxy(map);

    serviceMetadata.setTarget(ref);
    serviceMetadata.addAttribute(PROXY_CLASS_REF, ref);
    ConsumerModel consumerModel = repository.lookupReferredService(serviceMetadata.getServiceKey());
    consumerModel.setProxyObject(ref);
    consumerModel.init(attributes);

    initialized = true;

    checkInvokerAvailable();
}
```
- org.apache.dubbo.config.ReferenceConfig#createProxy
```java
private T createProxy(Map<String, String> map) {
    if (shouldJvmRefer(map)) {
        URL url = new ServiceConfigURL(LOCAL_PROTOCOL, LOCALHOST_VALUE, 0, interfaceClass.getName()).addParameters(map);
        invoker = REF_PROTOCOL.refer(interfaceClass, url);
        if (logger.isInfoEnabled()) {
            logger.info("Using injvm service " + interfaceClass.getName());
        }
    } else {
        urls.clear();
        if (url != null && url.length() > 0) { // user specified URL, could be peer-to-peer address, or register center's address.
            String[] us = SEMICOLON_SPLIT_PATTERN.split(url);
            if (us != null && us.length > 0) {
                for (String u : us) {
                    URL url = URL.valueOf(u);
                    if (StringUtils.isEmpty(url.getPath())) {
                        url = url.setPath(interfaceName);
                    }
                    if (UrlUtils.isRegistry(url)) {
                        urls.add(url.putAttribute(REFER_KEY, map));
                    } else {
                        URL peerURL = ClusterUtils.mergeUrl(url, map);
                        peerURL = peerURL.putAttribute(PEER_KEY, true);
                        urls.add(peerURL);
                    }
                }
            }
        } else { // assemble URL from register center's configuration
            // if protocols not injvm checkRegistry
            if (!LOCAL_PROTOCOL.equalsIgnoreCase(getProtocol())) {
                checkRegistry();
                // 加载注册中心地址(可以配置多个注册中心)，后续调用的时候按照list顺序选择一个可用的
                List<URL> us = ConfigValidationUtils.loadRegistries(this, false);
                if (CollectionUtils.isNotEmpty(us)) {
                    for (URL u : us) {
                        URL monitorUrl = ConfigValidationUtils.loadMonitor(this, u);
                        if (monitorUrl != null) {
                            u = u.putAttribute(MONITOR_KEY, monitorUrl);
                        }
                        urls.add(u.putAttribute(REFER_KEY, map));
                    }
                }
                if (urls.isEmpty()) {
                    throw new IllegalStateException(
                            "No such any registry to reference " + interfaceName + " on the consumer " + NetUtils.getLocalHost() +
                                    " use dubbo version " + Version.getVersion() +
                                    ", please config <dubbo:registry address=\"...\" /> to your spring config.");
                }
            }
        }

        if (urls.size() == 1) {
        	// 最终是RegistryProtocol.refer以及DubboProtocol.refer,调用链则是mock、listen等包装类，即调用执行或者构建服务目录
            invoker = REF_PROTOCOL.refer(interfaceClass, urls.get(0));
        } else {
            List<Invoker<?>> invokers = new ArrayList<Invoker<?>>();
            URL registryURL = null;
            for (URL url : urls) {
                // For multi-registry scenarios, it is not checked whether each referInvoker is available.
                // Because this invoker may become available later.
                invokers.add(REF_PROTOCOL.refer(interfaceClass, url));

                if (UrlUtils.isRegistry(url)) {
                    registryURL = url; // use last registry url
                }
            }

            if (registryURL != null) { // registry url is available
                // for multi-subscription scenario, use 'zone-aware' policy by default
                String cluster = registryURL.getParameter(CLUSTER_KEY, ZoneAwareCluster.NAME);
                // The invoker wrap sequence would be: ZoneAwareClusterInvoker(StaticDirectory) -> FailoverClusterInvoker(RegistryDirectory, routing happens here) -> Invoker
                invoker = Cluster.getCluster(cluster, false).join(new StaticDirectory(registryURL, invokers));
            } else { // not a registry url, must be direct invoke.
                String cluster = CollectionUtils.isNotEmpty(invokers)
                        ?
                        (invokers.get(0).getUrl() != null ? invokers.get(0).getUrl().getParameter(CLUSTER_KEY, ZoneAwareCluster.NAME) :
                                Cluster.DEFAULT)
                        : Cluster.DEFAULT;
                invoker = Cluster.getCluster(cluster).join(new StaticDirectory(invokers));
            }
        }
    }

    if (logger.isInfoEnabled()) {
        logger.info("Referred dubbo service " + interfaceClass.getName());
    }

    URL consumerURL = new ServiceConfigURL(CONSUMER_PROTOCOL, map.get(REGISTER_IP_KEY), 0, map.get(INTERFACE_KEY), map);
    MetadataUtils.publishServiceDefinition(consumerURL);

    // create service proxy
    return (T) PROXY_FACTORY.getProxy(invoker, ProtocolUtils.isGeneric(generic));
}
```
- org.apache.dubbo.registry.integration.RegistryProtocol#refer
```java
public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
	// type是引入的服务
    url = getRegistryUrl(url);
    // 注册中心registry，例如zk
    Registry registry = registryFactory.getRegistry(url);
    if (RegistryService.class.equals(type)) {
        return proxyFactory.getInvoker((T) registry, type, url);
    }

    // group="a,b" or group="*"
    Map<String, String> qs = (Map<String, String>)url.getAttribute(REFER_KEY);
    String group = qs.get(GROUP_KEY);
    if (group != null && group.length() > 0) {
        if ((COMMA_SPLIT_PATTERN.split(group)).length > 1 || "*".equals(group)) {
            return doRefer(Cluster.getCluster(MergeableCluster.NAME), registry, type, url, qs);
        }
    }

    Cluster cluster = Cluster.getCluster(qs.get(CLUSTER_KEY));
    return doRefer(cluster, registry, type, url, qs);
}
```
- org.apache.dubbo.registry.integration.RegistryProtocol#doRefer
```java
protected <T> Invoker<T> doRefer(Cluster cluster, Registry registry, Class<T> type, URL url, Map<String, String> parameters) {
    Map<String, Object> consumerAttribute = new HashMap<>(url.getAttributes());
    consumerAttribute.remove(REFER_KEY);
    URL consumerUrl = new ServiceConfigURL(parameters.get(PROTOCOL_KEY) == null ? DUBBO : parameters.get(PROTOCOL_KEY),
            null,
            null,
            parameters.get(REGISTER_IP_KEY),
            0, getPath(parameters, type),
            parameters,
            consumerAttribute);
    url = url.putAttribute(CONSUMER_URL_KEY, consumerUrl);
    /*
    protected <T> ClusterInvoker<T> getMigrationInvoker(RegistryProtocol registryProtocol, Cluster cluster, Registry registry, Class<T> type, URL url, URL consumerUrl) {
        return new ServiceDiscoveryMigrationInvoker<T>(registryProtocol, cluster, registry, type, url, consumerUrl);
    }
    */
    // 构建服务目录,doCreateInvoker中监听路由、旧版本、新版的服务提供者信息
    ClusterInvoker<T> migrationInvoker = getMigrationInvoker(this, cluster, registry, type, url, consumerUrl);
    return interceptInvoker(migrationInvoker, url, consumerUrl, url);
}
```