- org.apache.dubbo.config.spring.ServiceBean#exported|入口,旧版是在该类onApplicationEvent,新版是org.apache.dubbo.config.spring.context.DubboBootstrapApplicationListener#onContextRefreshedEvent启动dubboBootstrap
```java
public void exported() {
    super.exported();
    // Publish ServiceBeanExportedEvent
    // 导出完毕
    publishExportEvent();
}
```
## 4.org.apache.dubbo.config.bootstrap.DubboBootstrap#start
旧版本是走checkAndUpdateSubConfigs
```java
public DubboBootstrap start() {
    if (started.compareAndSet(false, true)) {
        startup.set(false);
        initialized.set(false);
        shutdown.set(false);
        awaited.set(false);
        // 初始化配置，包括配置中心刷新本地配置
        // org.apache.dubbo.config.bootstrap.DubboBootstrap#initialize->org.apache.dubbo.config.bootstrap.DubboBootstrap#startConfigCenter->org.apache.dubbo.config.context.ConfigManager#refreshAll
        initialize();
        if (logger.isInfoEnabled()) {
            logger.info(NAME + " is starting...");
        }
        // 1. export Dubbo Services
        exportServices();

        // If register consumer instance or has exported services
        if (isRegisterConsumerInstance() || hasExportedServices()) {
            // 2. export MetadataService
            exportMetadataService();
            // 3. Register the local ServiceInstance if required
            registerServiceInstance();
        }

        referServices();
        if (asyncExportingFutures.size() > 0 || asyncReferringFutures.size() > 0) {
            new Thread(() -> {
                try {
                    this.awaitFinish();
                } catch (Exception e) {
                    logger.warn(NAME + " asynchronous export / refer occurred an exception.");
                }
                startup.set(true);
                if (logger.isInfoEnabled()) {
                    logger.info(NAME + " is ready.");
                }
                onStart();
            }).start();
        } else {
            startup.set(true);
            if (logger.isInfoEnabled()) {
                logger.info(NAME + " is ready.");
            }
            onStart();
        }

        if (logger.isInfoEnabled()) {
            logger.info(NAME + " has started.");
        }
    }
    return this;
}
```
### 4.1org.apache.dubbo.config.bootstrap.DubboBootstrap#exportServices
```java
private void exportServices() {
    for (ServiceConfigBase sc : configManager.getServices()) {
        // TODO, compatible with ServiceConfig.export()
        ServiceConfig<?> serviceConfig = (ServiceConfig<?>) sc;
        serviceConfig.setBootstrap(this);
        if (!serviceConfig.isRefreshed()) {
            serviceConfig.refresh();
        }

        if (sc.shouldExportAsync()) {
            ExecutorService executor = executorRepository.getExportReferExecutor();
            CompletableFuture<Void> future = CompletableFuture.runAsync(() -> {
                try {
                	// 核心方法
                    sc.export();
                    exportedServices.add(sc);
                } catch (Throwable t) {
                    logger.error("export async catch error : " + t.getMessage(), t);
                }
            }, executor);

            asyncExportingFutures.add(future);
        } else {
        	// 核心方法
            sc.export();
            exportedServices.add(sc);
        }
    }
}
```
- org.apache.dubbo.config.ServiceConfig#export
```java
public synchronized void export() {
    if (this.shouldExport() && !this.exported) {
        this.init();

        // check bootstrap state
        if (!bootstrap.isInitialized()) {
            throw new IllegalStateException("DubboBootstrap is not initialized");
        }
        // 初始化配置，包括配置中心刷新本地service相关配置
        if (!this.isRefreshed()) {
            this.refresh();
        }

        if (!shouldExport()) {
            return;
        }

        if (shouldDelay()) {
            DELAY_EXPORT_EXECUTOR.schedule(this::doExport, getDelay(), TimeUnit.MILLISECONDS);
        } else {
        	// 核心方法
            doExport();
        }

        if (this.bootstrap.getTakeoverMode() == BootstrapTakeoverMode.AUTO) {
            this.bootstrap.start();
        }
    }
}
```
- org.apache.dubbo.config.ServiceConfig#doExport
```java
protected synchronized void doExport() {
    if (unexported) {
        throw new IllegalStateException("The service " + interfaceClass.getName() + " has already unexported!");
    }
    if (exported) {
        return;
    }
    exported = true;

    if (StringUtils.isEmpty(path)) {
        path = interfaceName;
    }
    doExportUrls();
    exported();
}
```
#### 4.1.1org.apache.dubbo.config.ServiceConfig#doExportUrlsFor1Protocol
```java
private void doExportUrls() {
    ServiceRepository repository = ApplicationModel.getServiceRepository();
    ServiceDescriptor serviceDescriptor = repository.registerService(getInterfaceClass());
    repository.registerProvider(
            getUniqueServiceName(),
            ref,
            serviceDescriptor,
            this,
            serviceMetadata
    );
    // 在dubbo中注册中心、服务、协议、监控中心都是用URL表示zookeeper://、dubbo://等
    List<URL> registryURLs = ConfigValidationUtils.loadRegistries(this, true);
    // 根据http还是dubbo协议进行构造，如果有多个注册中心也会注册多个
    for (ProtocolConfig protocolConfig : protocols) {
        String pathKey = URL.buildKey(getContextPath(protocolConfig)
                .map(p -> p + "/" + path)
                .orElse(path), group, version);
        // In case user specified path, register service one more time to map it to path.
        repository.registerService(pathKey, interfaceClass);
        doExportUrlsFor1Protocol(protocolConfig, registryURLs);
    }
}
```
##### 4.1.1.1org.apache.dubbo.config.ServiceConfig#doExportUrls
```java
private void doExportUrlsFor1Protocol(ProtocolConfig protocolConfig, List<URL> registryURLs) {
    String name = protocolConfig.getName();
    if (StringUtils.isEmpty(name)) {
        name = DUBBO;
    }

    Map<String, String> map = new HashMap<String, String>();
    map.put(SIDE_KEY, PROVIDER_SIDE);

    ServiceConfig.appendRuntimeParameters(map);
    AbstractConfig.appendParameters(map, getMetrics());
    AbstractConfig.appendParameters(map, getApplication());
    AbstractConfig.appendParameters(map, getModule());
    // remove 'default.' prefix for configs from ProviderConfig
    // appendParameters(map, provider, Constants.DEFAULT_KEY);
    AbstractConfig.appendParameters(map, provider);
    AbstractConfig.appendParameters(map, protocolConfig);
    AbstractConfig.appendParameters(map, this);
    MetadataReportConfig metadataReportConfig = getMetadataReportConfig();
    if (metadataReportConfig != null && metadataReportConfig.isValid()) {
        map.putIfAbsent(METADATA_KEY, REMOTE_METADATA_STORAGE_TYPE);
    }
    if (CollectionUtils.isNotEmpty(getMethods())) {
        //TODO Improve method config processing
        for (MethodConfig method : getMethods()) {
            AbstractConfig.appendParameters(map, method, method.getName());
            String retryKey = method.getName() + ".retry";
            if (map.containsKey(retryKey)) {
                String retryValue = map.remove(retryKey);
                if ("false".equals(retryValue)) {
                    map.put(method.getName() + ".retries", "0");
                }
            }
            List<ArgumentConfig> arguments = method.getArguments();
            if (CollectionUtils.isNotEmpty(arguments)) {
                for (ArgumentConfig argument : arguments) {
                    // convert argument type
                    if (argument.getType() != null && argument.getType().length() > 0) {
                        Method[] methods = interfaceClass.getMethods();
                        // visit all methods
                        if (methods.length > 0) {
                            for (int i = 0; i < methods.length; i++) {
                                String methodName = methods[i].getName();
                                // target the method, and get its signature
                                if (methodName.equals(method.getName())) {
                                    Class<?>[] argtypes = methods[i].getParameterTypes();
                                    // one callback in the method
                                    if (argument.getIndex() != -1) {
                                        if (argtypes[argument.getIndex()].getName().equals(argument.getType())) {
                                            AbstractConfig.appendParameters(map, argument, method.getName() + "." + argument.getIndex());
                                        } else {
                                            throw new IllegalArgumentException("Argument config error : the index attribute and type attribute not match :index :" + argument.getIndex() + ", type:" + argument.getType());
                                        }
                                    } else {
                                        // multiple callbacks in the method
                                        for (int j = 0; j < argtypes.length; j++) {
                                            Class<?> argclazz = argtypes[j];
                                            if (argclazz.getName().equals(argument.getType())) {
                                                AbstractConfig.appendParameters(map, argument, method.getName() + "." + j);
                                                if (argument.getIndex() != -1 && argument.getIndex() != j) {
                                                    throw new IllegalArgumentException("Argument config error : the index attribute and type attribute not match :index :" + argument.getIndex() + ", type:" + argument.getType());
                                                }
                                            }
                                        }
                                    }
                                    break;
                                }
                            }
                        }
                    } else if (argument.getIndex() != -1) {
                        AbstractConfig.appendParameters(map, argument, method.getName() + "." + argument.getIndex());
                    } else {
                        throw new IllegalArgumentException("Argument config must set index or type attribute.eg: <dubbo:argument index='0' .../> or <dubbo:argument type=xxx .../>");
                    }

                }
            }
        } // end of methods for
    }

    if (ProtocolUtils.isGeneric(generic)) {
        map.put(GENERIC_KEY, generic);
        map.put(METHODS_KEY, ANY_VALUE);
    } else {
        String revision = Version.getVersion(interfaceClass, version);
        if (revision != null && revision.length() > 0) {
            map.put(REVISION_KEY, revision);
        }

        String[] methods = Wrapper.getWrapper(interfaceClass).getMethodNames();
        if (methods.length == 0) {
            logger.warn("No method found in service interface " + interfaceClass.getName());
            map.put(METHODS_KEY, ANY_VALUE);
        } else {
            map.put(METHODS_KEY, StringUtils.join(new HashSet<String>(Arrays.asList(methods)), ","));
        }
    }

    /**
     * Here the token value configured by the provider is used to assign the value to ServiceConfig#token
     */
    if (ConfigUtils.isEmpty(token) && provider != null) {
        token = provider.getToken();
    }

    if (!ConfigUtils.isEmpty(token)) {
        if (ConfigUtils.isDefault(token)) {
            map.put(TOKEN_KEY, UUID.randomUUID().toString());
        } else {
            map.put(TOKEN_KEY, token);
        }
    }
    //init serviceMetadata attachments
    serviceMetadata.getAttachments().putAll(map);

    // export service
    String host = findConfigedHosts(protocolConfig, registryURLs, map);
    Integer port = findConfigedPorts(protocolConfig, name, map);
    URL url = new ServiceConfigURL(name, null, null, host, port, getContextPath(protocolConfig).map(p -> p + "/" + path).orElse(path), map);

    // You can customize Configurator to append extra parameters
    if (ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class)
            .hasExtension(url.getProtocol())) {
        url = ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class)
                .getExtension(url.getProtocol()).getConfigurator(url).configure(url);
    }

    String scope = url.getParameter(SCOPE_KEY);
    // don't export when none is configured
    if (!SCOPE_NONE.equalsIgnoreCase(scope)) {

        // export to local if the config is not remote (export to remote only when config is remote)
        if (!SCOPE_REMOTE.equalsIgnoreCase(scope)) {
            exportLocal(url);
        }
        // export to remote if the config is not local (export to local only when config is local)
        if (!SCOPE_LOCAL.equalsIgnoreCase(scope)) {
            if (CollectionUtils.isNotEmpty(registryURLs)) {
            	// 遍历注册中心
                for (URL registryURL : registryURLs) {
                    //if protocol is only injvm ,not register
                    if (LOCAL_PROTOCOL.equalsIgnoreCase(url.getProtocol())) {
                        continue;
                    }
                    // 服务的URL
                    url = url.addParameterIfAbsent(DYNAMIC_KEY, registryURL.getParameter(DYNAMIC_KEY));
                    URL monitorUrl = ConfigValidationUtils.loadMonitor(this, registryURL);
                    if (monitorUrl != null) {
                        url = url.putAttribute(MONITOR_KEY, monitorUrl);
                    }
                    if (logger.isInfoEnabled()) {
                        if (url.getParameter(REGISTER_KEY, true)) {
                            logger.info("Register dubbo service " + interfaceClass.getName() + " url " + url.getServiceKey() + " to registry " + registryURL.getAddress());
                        } else {
                            logger.info("Export dubbo service " + interfaceClass.getName() + " to url " + url.getServiceKey());
                        }
                    }

                    // For providers, this is used to enable custom proxy to generate invoker
                    String proxy = url.getParameter(PROXY_KEY);
                    if (StringUtils.isNotEmpty(proxy)) {
                    	// 注册中心URL
                        registryURL = registryURL.addParameter(PROXY_KEY, proxy);
                    }
                    // 使用代理生成一个Invoker，可执行的服务。被调用时invoker.invoke进行反射执行相应实现类
                    Invoker<?> invoker = PROXY_FACTORY.getInvoker(ref, (Class) interfaceClass, registryURL.putAttribute(EXPORT_KEY, url));
                    // 二次包装，包含全部信息
                    DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);
                   	// 返回导出器，可以注销服务
                    Exporter<?> exporter = PROTOCOL.export(wrapperInvoker);
                    exporters.add(exporter);
                }
            } else {
                if (logger.isInfoEnabled()) {
                    logger.info("Export dubbo service " + interfaceClass.getName() + " to url " + url);
                }
                if (MetadataService.class.getName().equals(url.getServiceInterface())) {
                    MetadataUtils.saveMetadataURL(url);
                }
                Invoker<?> invoker = PROXY_FACTORY.getInvoker(ref, (Class) interfaceClass, url);
                DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);

                Exporter<?> exporter = PROTOCOL.export(wrapperInvoker);
                exporters.add(exporter);
            }

            MetadataUtils.publishServiceDefinition(url);
        }
    }
    this.urls.add(url);
}
```
###### 4.1.1.1.1org.apache.dubbo.registry.integration.RegistryProtocol#export
```java
 public <T> Exporter<T> export(final Invoker<T> originInvoker) throws RpcException {
 	// 注册中心URL
    URL registryUrl = getRegistryUrl(originInvoker);
    // url to export locally
    // 服务URL
    URL providerUrl = getProviderUrl(originInvoker);

    // Subscribe the override data
    // FIXME When the provider subscribes, it will affect the scene : a certain JVM exposes the service and call
    //  the same service. Because the subscribed is cached key with the name of the service, it causes the
    //  subscription information to cover.
    final URL overrideSubscribeUrl = getSubscribedOverrideUrl(providerUrl);
    final OverrideListener overrideSubscribeListener = new OverrideListener(overrideSubscribeUrl, originInvoker);
    overrideListeners.put(overrideSubscribeUrl, overrideSubscribeListener);

    providerUrl = overrideUrlWithConfig(providerUrl, overrideSubscribeListener);
    //export invoker
    // invoker与具体的端口逻辑关联exporterMap,具体可回看大类，以及启动netty
    final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker, providerUrl);

    // url to registry
    // 获取注册中心包装类
    final Registry registry = getRegistry(registryUrl);
    // 简化注册用的URL
    final URL registeredProviderUrl = getUrlToRegistry(providerUrl, registryUrl);

    // decide if we need to delay publish
    boolean register = providerUrl.getParameter(REGISTER_KEY, true);
    if (register) {
    	// 实际注册
    	// zkClient.create
        register(registry, registeredProviderUrl);
    }

    // register stated url on provider model
    registerStatedUrl(registryUrl, registeredProviderUrl, register);


    exporter.setRegisterUrl(registeredProviderUrl);
    exporter.setSubscribeUrl(overrideSubscribeUrl);

    // Deprecated! Subscribe to override rules in 2.6.x or before.
    registry.subscribe(overrideSubscribeUrl, overrideSubscribeListener);

    notifyExport(exporter);
    //Ensure that a new exporter instance is returned every time export
    return new DestroyableExporter<>(exporter);
}
```
## 5.org.apache.dubbo.registry.integration.RegistryProtocol#doLocalExport
```java
private <T> ExporterChangeableWrapper<T> doLocalExport(final Invoker<T> originInvoker, URL providerUrl) {
    String key = getCacheKey(originInvoker);

    return (ExporterChangeableWrapper<T>) bounds.computeIfAbsent(key, s -> {
        Invoker<?> invokerDelegate = new InvokerDelegate<>(originInvoker, providerUrl);
        return new ExporterChangeableWrapper<>((Exporter<T>) protocol.export(invokerDelegate), originInvoker);
    });
}
```
- org.apache.dubbo.rpc.protocol.dubbo.DubboProtocol#export
```java
public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
    URL url = invoker.getUrl();

    // export service.
    String key = serviceKey(url);
    // 包装invoker
    DubboExporter<T> exporter = new DubboExporter<T>(invoker, key, exporterMap);
    exporterMap.put(key, exporter);

    //export an stub service for dispatching event
    Boolean isStubSupportEvent = url.getParameter(STUB_EVENT_KEY, DEFAULT_STUB_EVENT);
    Boolean isCallbackservice = url.getParameter(IS_CALLBACK_SERVICE, false);
    if (isStubSupportEvent && !isCallbackservice) {
        String stubServiceMethods = url.getParameter(STUB_EVENT_METHODS_KEY);
        if (stubServiceMethods == null || stubServiceMethods.length() == 0) {
            if (logger.isWarnEnabled()) {
                logger.warn(new IllegalStateException("consumer [" + url.getParameter(INTERFACE_KEY) +
                        "], has set stubproxy support event ,but no stub methods founded."));
            }

        }
    }
    // 启动nettyServer
    openServer(url);
    optimizeSerialization(url);

    return exporter;
}
```
## 6.org.apache.dubbo.registry.integration.RegistryProtocol.OverrideListener#notify|监听配置中心
```java
public synchronized void notify(List<URL> urls) {
    logger.debug("original override urls: " + urls);

    List<URL> matchedUrls = getMatchedUrls(urls, subscribeUrl.addParameter(CATEGORY_KEY,
            CONFIGURATORS_CATEGORY));
    logger.debug("subscribe url: " + subscribeUrl + ", override urls: " + matchedUrls);

    // No matching results
    if (matchedUrls.isEmpty()) {
        return;
    }

    this.configurators = Configurator.toConfigurators(classifyUrls(matchedUrls, UrlUtils::isConfigurator))
            .orElse(configurators);

    doOverrideIfNecessary();
}
```
```java
public synchronized void doOverrideIfNecessary() {
    final Invoker<?> invoker;
    if (originInvoker instanceof InvokerDelegate) {
        invoker = ((InvokerDelegate<?>) originInvoker).getInvoker();
    } else {
        invoker = originInvoker;
    }
    //The origin invoker
    URL originUrl = RegistryProtocol.this.getProviderUrl(invoker);
    String key = getCacheKey(originInvoker);
    ExporterChangeableWrapper<?> exporter = bounds.get(key);
    if (exporter == null) {
        logger.warn(new IllegalStateException("error state, exporter should not be null"));
        return;
    }
    //The current, may have been merged many times
    URL currentUrl = exporter.getInvoker().getUrl();
    //Merged with this configuration
    URL newUrl = getConfigedInvokerUrl(configurators, currentUrl);
    newUrl = getConfigedInvokerUrl(providerConfigurationListener.getConfigurators(), newUrl);
    newUrl = getConfigedInvokerUrl(serviceConfigurationListeners.get(originUrl.getServiceKey())
            .getConfigurators(), newUrl);
    // 比较配置中心的信息是否有改变
    if (!currentUrl.equals(newUrl)) {
    	// 配置中心则需要重新导出
        RegistryProtocol.this.reExport(originInvoker, newUrl);
        logger.info("exported provider url changed, origin url: " + originUrl +
                ", old export url: " + currentUrl + ", new export url: " + newUrl);
    }
}
```
