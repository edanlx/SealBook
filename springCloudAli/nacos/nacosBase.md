
## 4.客户端启动
- 客户端主要类
spring-cloud-starter-alibaba-nacos-discovery-2.2.6.RELEASE.jar!/META-INF/spring.factories->com.alibaba.cloud.nacos.discovery.NacosDiscoveryAutoConfiguration
- com.alibaba.cloud.nacos.discovery.NacosDiscoveryAutoConfiguration|nacos1.x的核心代码
```java
public NacosServiceDiscovery nacosServiceDiscovery(NacosDiscoveryProperties discoveryProperties, NacosServiceManager nacosServiceManager) {
	return new NacosServiceDiscovery(discoveryProperties, nacosServiceManager);
}
```
- com.alibaba.cloud.nacos.registry.NacosServiceRegistryAutoConfiguratio|nacos2.x的核心代码
```java
public NacosAutoServiceRegistration nacosAutoServiceRegistration(
		NacosServiceRegistry registry,
		AutoServiceRegistrationProperties autoServiceRegistrationProperties,
		NacosRegistration registration) {
	return new NacosAutoServiceRegistration(registry,
			autoServiceRegistrationProperties, registration);
}
```
- org.springframework.cloud.client.serviceregistry.AbstractAutoServiceRegistration#onApplicationEvent|上述两种都会进到这个核心类
```java
public void onApplicationEvent(WebServerInitializedEvent event) {
	bind(event);
}
```
- org.springframework.cloud.client.serviceregistry.AbstractAutoServiceRegistration#bind
```java
public void bind(WebServerInitializedEvent event) {
	ApplicationContext context = event.getApplicationContext();
	if (context instanceof ConfigurableWebServerApplicationContext) {
		if ("management".equals(((ConfigurableWebServerApplicationContext) context)
				.getServerNamespace())) {
			return;
		}
	}
	this.port.compareAndSet(0, event.getWebServer().getPort());
	this.start();
}
```
- org.springframework.cloud.client.serviceregistry.AbstractAutoServiceRegistration#register
```java
public void start() {
	if (!isEnabled()) {
		if (logger.isDebugEnabled()) {
			logger.debug("Discovery Lifecycle disabled. Not starting");
		}
		return;
	}

	// only initialize if nonSecurePort is greater than 0 and it isn't already running
	// because of containerPortInitializer below
	if (!this.running.get()) {
		this.context.publishEvent(
				new InstancePreRegisteredEvent(this, getRegistration()));
		// 核心注册
		register();
		if (shouldRegisterManagement()) {
			registerManagement();
		}
		this.context.publishEvent(
				new InstanceRegisteredEvent<>(this, getConfiguration()));
		this.running.compareAndSet(false, true);
	}

}
```
### 4.1org.springframework.cloud.client.serviceregistry.AbstractAutoServiceRegistration#register
```java
protected void register() {
	// spring cloud的标准实现策略模式选择(原先是Eureka)
	this.serviceRegistry.register(getRegistration());
}
```
- com.alibaba.cloud.nacos.registry.NacosServiceRegistry#register
```java
public void register(Registration registration) {

	if (StringUtils.isEmpty(registration.getServiceId())) {
		log.warn("No service to register for nacos client...");
		return;
	}

	NamingService namingService = namingService();
	String serviceId = registration.getServiceId();
	String group = nacosDiscoveryProperties.getGroup();
	// 获取实例
	Instance instance = getNacosInstanceFromRegistration(registration);

	try {
		// 注册
		namingService.registerInstance(serviceId, group, instance);
		log.info("nacos registry, {} {} {}:{} register finished", group, serviceId,
				instance.getIp(), instance.getPort());
	}
	catch (Exception e) {
		if (nacosDiscoveryProperties.isFailFast()) {
			log.error("nacos registry, {} register failed...{},", serviceId,
					registration.toString(), e);
			rethrowRuntimeException(e);
		}
		else {
			log.warn("Failfast is false. {} register failed...{},", serviceId,
					registration.toString(), e);
		}
	}
}
```
- com.alibaba.nacos.client.naming.NacosNamingService#registerInstance(java.lang.String, java.lang.String, com.alibaba.nacos.api.naming.pojo.Instance)
```java
 public void registerInstance(String serviceName, String groupName, Instance instance) throws NacosException {
    NamingUtils.checkInstanceIsLegal(instance);
    String groupedServiceName = NamingUtils.getGroupedName(serviceName, groupName);
    if (instance.isEphemeral()) {
        BeatInfo beatInfo = beatReactor.buildBeatInfo(groupedServiceName, instance);
        // 添加心跳
        beatReactor.addBeatInfo(groupedServiceName, beatInfo);
    }
    serverProxy.registerService(groupedServiceName, groupName, instance);
}
```
#### 4.1.1com.alibaba.nacos.client.naming.net.NamingProxy#registerService
```java
public void registerService(String serviceName, String groupName, Instance instance) throws NacosException {
        
    NAMING_LOGGER.info("[REGISTER-SERVICE] {} registering service {} with instance: {}", namespaceId, serviceName,
            instance);
    
    final Map<String, String> params = new HashMap<String, String>(16);
    params.put(CommonParams.NAMESPACE_ID, namespaceId);
    params.put(CommonParams.SERVICE_NAME, serviceName);
    params.put(CommonParams.GROUP_NAME, groupName);
    params.put(CommonParams.CLUSTER_NAME, instance.getClusterName());
    params.put("ip", instance.getIp());
    params.put("port", String.valueOf(instance.getPort()));
    params.put("weight", String.valueOf(instance.getWeight()));
    params.put("enable", String.valueOf(instance.isEnabled()));
    params.put("healthy", String.valueOf(instance.isHealthy()));
    params.put("ephemeral", String.valueOf(instance.isEphemeral()));
    params.put("metadata", JacksonUtils.toJson(instance.getMetadata()));

    // 向客户端发送请求/nacos/v1/ns/instance
    reqApi(UtilAndComs.nacosUrlInstance, params, HttpMethod.POST);
    
}
```
#### 4.1.2com.alibaba.nacos.client.naming.beat.BeatReactor#addBeatInfo
```java
public void addBeatInfo(String serviceName, BeatInfo beatInfo) {
    NAMING_LOGGER.info("[BEAT] adding beat: {} to beat map.", beatInfo);
    String key = buildKey(serviceName, beatInfo.getIp(), beatInfo.getPort());
    BeatInfo existBeat = null;
    //fix #1733
    if ((existBeat = dom2Beat.remove(key)) != null) {
        existBeat.setStopped(true);
    }
    dom2Beat.put(key, beatInfo);
    executorService.schedule(new BeatTask(beatInfo), beatInfo.getPeriod(), TimeUnit.MILLISECONDS);
    MetricsMonitor.getDom2BeatSizeMonitor().set(dom2Beat.size());
}
```
- com.alibaba.nacos.client.naming.beat.BeatReactor.BeatTask#run
```java
public void run() {
    if (beatInfo.isStopped()) {
        return;
    }
    long nextTime = beatInfo.getPeriod();
    try {
        JsonNode result = serverProxy.sendBeat(beatInfo, BeatReactor.this.lightBeatEnabled);
        long interval = result.get("clientBeatInterval").asLong();
        boolean lightBeatEnabled = false;
        if (result.has(CommonParams.LIGHT_BEAT_ENABLED)) {
            lightBeatEnabled = result.get(CommonParams.LIGHT_BEAT_ENABLED).asBoolean();
        }
        BeatReactor.this.lightBeatEnabled = lightBeatEnabled;
        if (interval > 0) {
            nextTime = interval;
        }
        int code = NamingResponseCode.OK;
        if (result.has(CommonParams.CODE)) {
            code = result.get(CommonParams.CODE).asInt();
        }
        if (code == NamingResponseCode.RESOURCE_NOT_FOUND) {
            Instance instance = new Instance();
            instance.setPort(beatInfo.getPort());
            instance.setIp(beatInfo.getIp());
            instance.setWeight(beatInfo.getWeight());
            instance.setMetadata(beatInfo.getMetadata());
            instance.setClusterName(beatInfo.getCluster());
            instance.setServiceName(beatInfo.getServiceName());
            instance.setInstanceId(instance.getInstanceId());
            instance.setEphemeral(true);
            try {
                serverProxy.registerService(beatInfo.getServiceName(),
                        NamingUtils.getGroupName(beatInfo.getServiceName()), instance);
            } catch (Exception ignore) {
            }
        }
    } catch (NacosException ex) {
        NAMING_LOGGER.error("[CLIENT-BEAT] failed to send beat: {}, code: {}, msg: {}",
                JacksonUtils.toJson(beatInfo), ex.getErrCode(), ex.getErrMsg());

    } catch (Exception unknownEx) {
        NAMING_LOGGER.error("[CLIENT-BEAT] failed to send beat: {}, unknown exception msg: {}",
                JacksonUtils.toJson(beatInfo), unknownEx.getMessage(), unknownEx);
    } finally {
    	// 定时
        executorService.schedule(new BeatTask(beatInfo), nextTime, TimeUnit.MILLISECONDS);
    }
}
```

- com.alibaba.nacos.client.naming.net.NamingProxy#sendBeat
```java
public JsonNode sendBeat(BeatInfo beatInfo, boolean lightBeatEnabled) throws NacosException {
        
    if (NAMING_LOGGER.isDebugEnabled()) {
        NAMING_LOGGER.debug("[BEAT] {} sending beat to server: {}", namespaceId, beatInfo.toString());
    }
    Map<String, String> params = new HashMap<String, String>(8);
    Map<String, String> bodyMap = new HashMap<String, String>(2);
    if (!lightBeatEnabled) {
        bodyMap.put("beat", JacksonUtils.toJson(beatInfo));
    }
    params.put(CommonParams.NAMESPACE_ID, namespaceId);
    params.put(CommonParams.SERVICE_NAME, beatInfo.getServiceName());
    params.put(CommonParams.CLUSTER_NAME, beatInfo.getCluster());
    params.put("ip", beatInfo.getIp());
    params.put("port", String.valueOf(beatInfo.getPort()));
    String result = reqApi(UtilAndComs.nacosUrlBase + "/instance/beat", params, bodyMap, HttpMethod.PUT);
    return JacksonUtils.toObj(result);
}
```

## 5.服务端启动
```java
@SpringBootApplication(scanBasePackages = "com.alibaba.nacos")
@ServletComponentScan
@EnableScheduling
public class Nacos {
    
    public static void main(String[] args) {
        SpringApplication.run(Nacos.class, args);
    }
}
```

## 6.服务端接收注册
```java
@CanDistro
@PostMapping
@Secured(action = ActionTypes.WRITE)
// Rest风格，对增删改查按照Mapping的传输类型(POST、GET、PUT、DELETE)进行区分,用同一个路径
public String register(HttpServletRequest request) throws Exception {
    
    final String namespaceId = WebUtils
            .optional(request, CommonParams.NAMESPACE_ID, Constants.DEFAULT_NAMESPACE_ID);
    final String serviceName = WebUtils.required(request, CommonParams.SERVICE_NAME);
    NamingUtils.checkServiceNameFormat(serviceName);
    
    final Instance instance = HttpRequestInstanceBuilder.newBuilder()
            .setDefaultInstanceEphemeral(switchDomain.isDefaultInstanceEphemeral()).setRequest(request).build();
    
    getInstanceOperator().registerInstance(namespaceId, serviceName, instance);
    return "ok";
}
```
- com.alibaba.nacos.naming.core.InstanceOperatorServiceImpl#registerInstance
```java
public void registerInstance(String namespaceId, String serviceName, Instance instance) throws NacosException {
    com.alibaba.nacos.naming.core.Instance coreInstance = parseInstance(instance);
    serviceManager.registerInstance(namespaceId, serviceName, coreInstance);
}
```
- com.alibaba.nacos.naming.core.ServiceManager#registerInstance
```java
 public void registerInstance(String namespaceId, String serviceName, Instance instance) throws NacosException {
    // 构建service结构
    createEmptyService(namespaceId, serviceName, instance.isEphemeral());
    
    Service service = getService(namespaceId, serviceName);
    
    checkServiceIsNull(service, namespaceId, serviceName);
    
    addInstance(namespaceId, serviceName, instance.isEphemeral(), instance);
}
```
### 6.1com.alibaba.nacos.naming.core.ServiceManager#createEmptyService
```java
public void createEmptyService(String namespaceId, String serviceName, boolean local) throws NacosException {
    createServiceIfAbsent(namespaceId, serviceName, local, null);
}
```
- com.alibaba.nacos.naming.core.ServiceManager#createServiceIfAbsent
```java
public void createServiceIfAbsent(String namespaceId, String serviceName, boolean local, Cluster cluster)
            throws NacosException {
    Service service = getService(namespaceId, serviceName);
    if (service == null) {
        
        Loggers.SRV_LOG.info("creating empty service {}:{}", namespaceId, serviceName);
        service = new Service();
        service.setName(serviceName);
        service.setNamespaceId(namespaceId);
        service.setGroupName(NamingUtils.getGroupName(serviceName));
        // now validate the service. if failed, exception will be thrown
        service.setLastModifiedMillis(System.currentTimeMillis());
        service.recalculateChecksum();
        if (cluster != null) {
            cluster.setService(service);
            service.getClusterMap().put(cluster.getName(), cluster);
        }
        service.validate();
        
        putServiceAndInit(service);
        if (!local) {
            addOrReplaceService(service);
        }
    }
}
```
### 6.1.1com.alibaba.nacos.naming.core.ServiceManager#getService
```java
/**
 * 注册表
 * Map(namespace, Map(group::serviceName, Service)).
 * Service 内部 private Map<String, Cluster> clusterMap = new HashMap<>();
 * Cluster 内部 private Set<Instance> ephemeralInstances = new HashSet<>();
 */
private final Map<String, Map<String, Service>> serviceMap = new ConcurrentHashMap<>();
public Service getService(String namespaceId, String serviceName) {
    if (serviceMap.get(namespaceId) == null) {
        return null;
    }
    return chooseServiceMap(namespaceId).get(serviceName);
}
```

### 6.1.2com.alibaba.nacos.naming.core.ServiceManager#putServiceAndInit
```java
private void putServiceAndInit(Service service) throws NacosException {
    putService(service);
    service = getService(service.getNamespaceId(), service.getName());
    service.init();
    consistencyService
            .listen(KeyBuilder.buildInstanceListKey(service.getNamespaceId(), service.getName(), true), service);
    consistencyService
            .listen(KeyBuilder.buildInstanceListKey(service.getNamespaceId(), service.getName(), false), service);
    Loggers.SRV_LOG.info("[NEW-SERVICE] {}", service.toJson());
}
```
#### 6.1.2.1com.alibaba.nacos.naming.core.ServiceManager#putService
```java
public void putService(Service service) {
    if (!serviceMap.containsKey(service.getNamespaceId())) {
        serviceMap.putIfAbsent(service.getNamespaceId(), new ConcurrentSkipListMap<>());
    }
    serviceMap.get(service.getNamespaceId()).putIfAbsent(service.getName(), service);
}
```
#### 6.1.2.2com.alibaba.nacos.naming.core.Service#init
```java
public void init() {
	// 心跳检查任务，每5秒执行一次。只有1台机子会进行检查，同时在ServerStatusReporter.init()中会起一个线程让所有服务端给其它服务端发送心跳。如果检查有服务端断连通过ServiceManager.init()进行通知
    HealthCheckReactor.scheduleCheck(clientBeatCheckTask);
    for (Map.Entry<String, Cluster> entry : clusterMap.entrySet()) {
        entry.getValue().setService(this);
        entry.getValue().init();
    }
}
```
- com.alibaba.nacos.naming.healthcheck.ClientBeatCheckTask#run
```java
public void run() {
    try {
        // If upgrade to 2.0.X stop health check with v1
        if (ApplicationUtils.getBean(UpgradeJudgement.class).isUseGrpcFeatures()) {
            return;
        }
        if (!getDistroMapper().responsible(service.getName())) {
            return;
        }
        
        if (!getSwitchDomain().isHealthCheckEnabled()) {
            return;
        }
        // 获取所有实例
        List<Instance> instances = service.allIPs(true);
        
        // first set health status of instances:
        for (Instance instance : instances) {
        	// 间隔时间大于15秒
            if (System.currentTimeMillis() - instance.getLastBeat() > instance.getInstanceHeartBeatTimeOut()) {
                if (!instance.isMarked()) {
                    if (instance.isHealthy()) {
                    	// 设置不健康
                        instance.setHealthy(false);
                        Loggers.EVT_LOG
                                .info("{POS} {IP-DISABLED} valid: {}:{}@{}@{}, region: {}, msg: client timeout after {}, last beat: {}",
                                        instance.getIp(), instance.getPort(), instance.getClusterName(),
                                        service.getName(), UtilsAndCommons.LOCALHOST_SITE,
                                        instance.getInstanceHeartBeatTimeOut(), instance.getLastBeat());
                        getPushService().serviceChanged(service);
                    }
                }
            }
        }
        
        if (!getGlobalConfig().isExpireInstance()) {
            return;
        }
        
        // then remove obsolete instances:
        for (Instance instance : instances) {
            
            if (instance.isMarked()) {
                continue;
            }
            
            if (System.currentTimeMillis() - instance.getLastBeat() > instance.getIpDeleteTimeout()) {
                // delete instance
                Loggers.SRV_LOG.info("[AUTO-DELETE-IP] service: {}, ip: {}", service.getName(),
                        JacksonUtils.toJson(instance));
                // 删除实例
                deleteIp(instance);
            }
        }
        
    } catch (Exception e) {
        Loggers.SRV_LOG.warn("Exception while processing client beat time out.", e);
    }
    
}
```
## 6.2com.alibaba.nacos.naming.core.ServiceManager#addInstance
```java
public void addInstance(String namespaceId, String serviceName, boolean ephemeral, Instance... ips)
            throws NacosException {
    // ephemeral默认临时实例true，存入instanceMap
    String key = KeyBuilder.buildInstanceListKey(namespaceId, serviceName, ephemeral);
    
    Service service = getService(namespaceId, serviceName);
    
    synchronized (service) {
        List<Instance> instanceList = addIpAddresses(service, ephemeral, ips);
        
        Instances instances = new Instances();
        instances.setInstanceList(instanceList);
        
        consistencyService.put(key, instances);
    }
}
```
### 6.2.1com.alibaba.nacos.naming.consistency.DelegateConsistencyServiceImpl#put
```java
public void put(String key, Record value) throws NacosException {
	// 根据ephemeral判断临时还是持久(AP/CP)
    mapConsistencyService(key).put(key, value);
}
```
- com.alibaba.nacos.naming.consistency.ephemeral.distro.DistroConsistencyServiceImpl#put
Distro是阿里自己实现的分布式数据同步
```java
public void put(String key, Record value) throws NacosException {
    onPut(key, value);
    // If upgrade to 2.0.X, do not sync for v1.
    if (ApplicationUtils.getBean(UpgradeJudgement.class).isUseGrpcFeatures()) {
        return;
    }
    // 同步，存入task，失败则重试，同时会提供一个全量数据接口用于新加入的机器
    distroProtocol.sync(new DistroKey(key, KeyBuilder.INSTANCE_LIST_KEY_PREFIX), DataOperation.CHANGE,
            DistroConfig.getInstance().getSyncDelayMillis());
}
```
- com.alibaba.nacos.naming.consistency.ephemeral.distro.DistroConsistencyServiceImpl#onPut
```java
public void onPut(String key, Record value) {
        
    if (KeyBuilder.matchEphemeralInstanceListKey(key)) {
        Datum<Instances> datum = new Datum<>();
        datum.value = (Instances) value;
        datum.key = key;
        datum.timestamp.incrementAndGet();
        dataStore.put(key, datum);
    }
    
    if (!listeners.containsKey(key)) {
        return;
    }

    notifier.addTask(key, DataOperation.CHANGE);
}
```
- com.alibaba.nacos.naming.consistency.ephemeral.distro.DistroConsistencyServiceImpl.Notifier#addTask
```java
private BlockingQueue<Pair<String, DataOperation>> tasks = new ArrayBlockingQueue<>(1024 * 1024);
public void addTask(String datumKey, DataOperation action) {
            
    if (services.containsKey(datumKey) && action == DataOperation.CHANGE) {
        return;
    }
    if (action == DataOperation.CHANGE) {
        services.put(datumKey, StringUtils.EMPTY);
    }
    tasks.offer(Pair.with(datumKey, action));
}
```

## 7.com.alibaba.nacos.naming.consistency.ephemeral.distro.DistroConsistencyServiceImpl.Notifier#run|异步注册
```java
public void run() {
    Loggers.DISTRO.info("distro notifier started");
    
    for (; ; ) {
        try {
            Pair<String, DataOperation> pair = tasks.take();
            handle(pair);
        } catch (Throwable e) {
            Loggers.DISTRO.error("[NACOS-DISTRO] Error while handling notifying task", e);
        }
    }
}
```
- com.alibaba.nacos.naming.consistency.ephemeral.distro.DistroConsistencyServiceImpl.Notifier#handle
```java
private void handle(Pair<String, DataOperation> pair) {
    try {
        String datumKey = pair.getValue0();
        DataOperation action = pair.getValue1();
        
        services.remove(datumKey);
        
        int count = 0;
        
        if (!listeners.containsKey(datumKey)) {
            return;
        }
        
        for (RecordListener listener : listeners.get(datumKey)) {
            
            count++;
            
            try {
                if (action == DataOperation.CHANGE) {
                    listener.onChange(datumKey, dataStore.get(datumKey).value);
                    continue;
                }
                
                if (action == DataOperation.DELETE) {
                    listener.onDelete(datumKey);
                    continue;
                }
            } catch (Throwable e) {
                Loggers.DISTRO.error("[NACOS-DISTRO] error while notifying listener of key: {}", datumKey, e);
            }
        }
        
        if (Loggers.DISTRO.isDebugEnabled()) {
            Loggers.DISTRO
                    .debug("[NACOS-DISTRO] datum change notified, key: {}, listener count: {}, action: {}",
                            datumKey, count, action.name());
        }
    } catch (Throwable e) {
        Loggers.DISTRO.error("[NACOS-DISTRO] Error while handling notifying task", e);
    }
}
```
- com.alibaba.nacos.naming.core.Service#onChange
```java
public void onChange(String key, Instances value) throws Exception {
        
    Loggers.SRV_LOG.info("[NACOS-RAFT] datum is changed, key: {}, value: {}", key, value);
    
    for (Instance instance : value.getInstanceList()) {
        
        if (instance == null) {
            // Reject this abnormal instance list:
            throw new RuntimeException("got null instance " + key);
        }
        
        if (instance.getWeight() > 10000.0D) {
            instance.setWeight(10000.0D);
        }
        
        if (instance.getWeight() < 0.01D && instance.getWeight() > 0.0D) {
            instance.setWeight(0.01D);
        }
    }
    
    updateIPs(value.getInstanceList(), KeyBuilder.matchEphemeralInstanceListKey(key));
    
    recalculateChecksum();
}
```
- com.alibaba.nacos.naming.core.Service#updateIPs
```java
public void updateIPs(Collection<Instance> instances, boolean ephemeral) {
    Map<String, List<Instance>> ipMap = new HashMap<>(clusterMap.size());
    for (String clusterName : clusterMap.keySet()) {
        ipMap.put(clusterName, new ArrayList<>());
    }
    
    for (Instance instance : instances) {
        try {
            if (instance == null) {
                Loggers.SRV_LOG.error("[NACOS-DOM] received malformed ip: null");
                continue;
            }
            
            if (StringUtils.isEmpty(instance.getClusterName())) {
                instance.setClusterName(UtilsAndCommons.DEFAULT_CLUSTER_NAME);
            }
            
            if (!clusterMap.containsKey(instance.getClusterName())) {
                Loggers.SRV_LOG
                        .warn("cluster: {} not found, ip: {}, will create new cluster with default configuration.",
                                instance.getClusterName(), instance.toJson());
                Cluster cluster = new Cluster(instance.getClusterName(), this);
                cluster.init();
                getClusterMap().put(instance.getClusterName(), cluster);
            }
            
            List<Instance> clusterIPs = ipMap.get(instance.getClusterName());
            if (clusterIPs == null) {
                clusterIPs = new LinkedList<>();
                ipMap.put(instance.getClusterName(), clusterIPs);
            }
            
            clusterIPs.add(instance);
        } catch (Exception e) {
            Loggers.SRV_LOG.error("[NACOS-DOM] failed to process ip: " + instance, e);
        }
    }
    
    for (Map.Entry<String, List<Instance>> entry : ipMap.entrySet()) {
        //make every ip mine
        List<Instance> entryIPs = entry.getValue();
        clusterMap.get(entry.getKey()).updateIps(entryIPs, ephemeral);
    }
    
    setLastModifiedMillis(System.currentTimeMillis());
    // 事件发布
    getPushService().serviceChanged(this);
    ApplicationUtils.getBean(DoubleWriteEventListener.class).doubleWriteToV2(this, ephemeral);
    StringBuilder stringBuilder = new StringBuilder();
    
    for (Instance instance : allIPs()) {
        stringBuilder.append(instance.toIpAddr()).append('_').append(instance.isHealthy()).append(',');
    }
    
    Loggers.EVT_LOG.info("[IP-UPDATED] namespace: {}, service: {}, ips: {}", getNamespaceId(), getName(),
            stringBuilder.toString());
    
}
```
### 7.1com.alibaba.nacos.naming.core.Cluster#updateIps
```java
public void updateIps(List<Instance> ips, boolean ephemeral) {

    Set<Instance> toUpdateInstances = ephemeral ? ephemeralInstances : persistentInstances;
    
    HashMap<String, Instance> oldIpMap = new HashMap<>(toUpdateInstances.size());
    // 复制一份数据(修改的逻辑是先删除再新增，并非对引用直接修改)
    for (Instance ip : toUpdateInstances) {
        oldIpMap.put(ip.getDatumKey(), ip);
    }
    
    List<Instance> updatedIps = updatedIps(ips, oldIpMap.values());
    if (updatedIps.size() > 0) {
        for (Instance ip : updatedIps) {
            Instance oldIP = oldIpMap.get(ip.getDatumKey());
            
            // do not update the ip validation status of updated ips
            // because the checker has the most precise result
            // Only when ip is not marked, don't we update the health status of IP:
            if (!ip.isMarked()) {
                ip.setHealthy(oldIP.isHealthy());
            }
            
            if (ip.isHealthy() != oldIP.isHealthy()) {
                // ip validation status updated
                Loggers.EVT_LOG.info("{} {SYNC} IP-{} {}:{}@{}", getService().getName(),
                        (ip.isHealthy() ? "ENABLED" : "DISABLED"), ip.getIp(), ip.getPort(), getName());
            }
            
            if (ip.getWeight() != oldIP.getWeight()) {
                // ip validation status updated
                Loggers.EVT_LOG.info("{} {SYNC} {IP-UPDATED} {}->{}", getService().getName(), oldIP, ip);
            }
        }
    }
    
    List<Instance> newIPs = subtract(ips, oldIpMap.values());
    if (newIPs.size() > 0) {
        Loggers.EVT_LOG
                .info("{} {SYNC} {IP-NEW} cluster: {}, new ips size: {}, content: {}", getService().getName(),
                        getName(), newIPs.size(), newIPs);
        
        for (Instance ip : newIPs) {
            HealthCheckStatus.reset(ip);
        }
    }
    
    List<Instance> deadIPs = subtract(oldIpMap.values(), ips);
    
    if (deadIPs.size() > 0) {
        Loggers.EVT_LOG
                .info("{} {SYNC} {IP-DEAD} cluster: {}, dead ips size: {}, content: {}", getService().getName(),
                        getName(), deadIPs.size(), deadIPs);
        
        for (Instance ip : deadIPs) {
            HealthCheckStatus.remv(ip);
        }
    }
    
    toUpdateInstances = new HashSet<>(ips);
    // 将复制的instance再赋值回去
    if (ephemeral) {
        ephemeralInstances = toUpdateInstances;
    } else {
        persistentInstances = toUpdateInstances;
    }
}
```
### 7.2com.alibaba.nacos.naming.push.UdpPushService#serviceChanged
```java
public void serviceChanged(Service service) {
	// 发布ServiceChangeEvent
    this.applicationContext.publishEvent(new ServiceChangeEvent(this, service));
}
```
- com.alibaba.nacos.naming.push.UdpPushService#onApplicationEvent
```java
public void onApplicationEvent(ServiceChangeEvent event) {
    // If upgrade to 2.0.X, do not push for v1.
    if (ApplicationUtils.getBean(UpgradeJudgement.class).isUseGrpcFeatures()) {
        return;
    }
    Service service = event.getService();
    String serviceName = service.getName();
    String namespaceId = service.getNamespaceId();
    //merge some change events to reduce the push frequency:
    if (futureMap.containsKey(UtilsAndCommons.assembleFullServiceName(namespaceId, serviceName))) {
        return;
    }
    Future future = GlobalExecutor.scheduleUdpSender(() -> {
        try {
            Loggers.PUSH.info(serviceName + " is changed, add it to push queue.");
            ConcurrentMap<String, PushClient> clients = subscriberServiceV1.getClientMap()
                    .get(UtilsAndCommons.assembleFullServiceName(namespaceId, serviceName));
            if (MapUtils.isEmpty(clients)) {
                return;
            }
            
            Map<String, Object> cache = new HashMap<>(16);
            long lastRefTime = System.nanoTime();
            for (PushClient client : clients.values()) {
                if (client.zombie()) {
                    Loggers.PUSH.debug("client is zombie: " + client);
                    clients.remove(client.toString());
                    Loggers.PUSH.debug("client is zombie: " + client);
                    continue;
                }
                
                AckEntry ackEntry;
                Loggers.PUSH.debug("push serviceName: {} to client: {}", serviceName, client);
                String key = getPushCacheKey(serviceName, client.getIp(), client.getAgent());
                byte[] compressData = null;
                Map<String, Object> data = null;
                if (switchDomain.getDefaultPushCacheMillis() >= 20000 && cache.containsKey(key)) {
                    org.javatuples.Pair pair = (org.javatuples.Pair) cache.get(key);
                    compressData = (byte[]) (pair.getValue0());
                    data = (Map<String, Object>) pair.getValue1();
                    
                    Loggers.PUSH.debug("[PUSH-CACHE] cache hit: {}:{}", serviceName, client.getAddrStr());
                }
                
                if (compressData != null) {
                    ackEntry = prepareAckEntry(client, compressData, data, lastRefTime);
                } else {
                    ackEntry = prepareAckEntry(client, prepareHostsData(client), lastRefTime);
                    if (ackEntry != null) {
                        cache.put(key,
                                new org.javatuples.Pair<>(ackEntry.getOrigin().getData(), ackEntry.getData()));
                    }
                }
                
                Loggers.PUSH.info("serviceName: {} changed, schedule push for: {}, agent: {}, key: {}",
                        client.getServiceName(), client.getAddrStr(), client.getAgent(),
                        (ackEntry == null ? null : ackEntry.getKey()));
                
                udpPush(ackEntry);
            }
        } catch (Exception e) {
            Loggers.PUSH.error("[NACOS-PUSH] failed to push serviceName: {} to client, error: {}", serviceName, e);
            
        } finally {
            futureMap.remove(UtilsAndCommons.assembleFullServiceName(namespaceId, serviceName));
        }
        
    }, 1000, TimeUnit.MILLISECONDS);
    
    futureMap.put(UtilsAndCommons.assembleFullServiceName(namespaceId, serviceName), future);
    
}
```
- com.alibaba.nacos.naming.push.UdpPushService#udpPush
```java
private static AckEntry udpPush(AckEntry ackEntry) {
    if (ackEntry == null) {
        Loggers.PUSH.error("[NACOS-PUSH] ackEntry is null.");
        return null;
    }
    
    if (ackEntry.getRetryTimes() > Constants.UDP_MAX_RETRY_TIMES) {
        Loggers.PUSH.warn("max re-push times reached, retry times {}, key: {}", ackEntry.getRetryTimes(),
                ackEntry.getKey());
        ackMap.remove(ackEntry.getKey());
        udpSendTimeMap.remove(ackEntry.getKey());
        MetricsMonitor.incrementFailPush();
        return ackEntry;
    }
    
    try {
        if (!ackMap.containsKey(ackEntry.getKey())) {
            MetricsMonitor.incrementPush();
        }
        ackMap.put(ackEntry.getKey(), ackEntry);
        udpSendTimeMap.put(ackEntry.getKey(), System.currentTimeMillis());
        
        Loggers.PUSH.info("send udp packet: " + ackEntry.getKey());
        udpSocket.send(ackEntry.getOrigin());
        
        ackEntry.increaseRetryTime();
        
        GlobalExecutor.scheduleRetransmitter(new Retransmitter(ackEntry),
                TimeUnit.NANOSECONDS.toMillis(Constants.ACK_TIMEOUT_NANOS), TimeUnit.MILLISECONDS);
        
        return ackEntry;
    } catch (Exception e) {
        Loggers.PUSH.error("[NACOS-PUSH] failed to push data: {} to client: {}, error: {}", ackEntry.getData(),
                ackEntry.getOrigin().getAddress().getHostAddress(), e);
        ackMap.remove(ackEntry.getKey());
        udpSendTimeMap.remove(ackEntry.getKey());
        MetricsMonitor.incrementFailPush();
        
        return null;
    }
}
```