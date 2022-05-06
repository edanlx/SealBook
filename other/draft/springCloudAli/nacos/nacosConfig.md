md5判断文件是否发生变化,即磁盘读文件，mysql数据是在启动的时候加载到本地信息，在DumpService中。如果发生变化通过LocalDataChangeEvent事件通知，依赖长轮询机制通知客户端，异步通知其它节点。subscriber.OnEvent
核心类NacosConfigService、NacosConfigController

## 4.com.alibaba.cloud.nacos.client.NacosPropertySourceLocator#locate|配置文件加载顺序
在springBoot启动的时候prepareEnviroment
```java
public PropertySource<?> locate(Environment env) {
	nacosConfigProperties.setEnvironment(env);
	ConfigService configService = nacosConfigManager.getConfigService();

	if (null == configService) {
		log.warn("no instance of config service found, can't load config from nacos");
		return null;
	}
	long timeout = nacosConfigProperties.getTimeout();
	nacosPropertySourceBuilder = new NacosPropertySourceBuilder(configService,
			timeout);
	String name = nacosConfigProperties.getName();

	String dataIdPrefix = nacosConfigProperties.getPrefix();
	if (StringUtils.isEmpty(dataIdPrefix)) {
		dataIdPrefix = name;
	}

	if (StringUtils.isEmpty(dataIdPrefix)) {
		dataIdPrefix = env.getProperty("spring.application.name");
	}

	CompositePropertySource composite = new CompositePropertySource(
			NACOS_PROPERTY_SOURCE_NAME);

	// 加载共享
	loadSharedConfiguration(composite);
	// 加载扩展
	loadExtConfiguration(composite);
	// 加载当前应用(微服务名称->文件扩展名->profile扩展名)
	loadApplicationConfiguration(composite, dataIdPrefix, nacosConfigProperties, env);
	return composite;
}
```

## 5.com.alibaba.nacos.client.config.NacosConfigService#getConfig|获取配置中心配置文件
```java
private String getConfigInner(String tenant, String dataId, String group, long timeoutMs) throws NacosException {
    group = blank2defaultGroup(group);
    ParamUtils.checkKeyParam(dataId, group);
    ConfigResponse cr = new ConfigResponse();
    
    cr.setDataId(dataId);
    cr.setTenant(tenant);
    cr.setGroup(group);
    
    // 优先使用本地配置
    // 容错
    String content = LocalConfigInfoProcessor.getFailover(agent.getName(), dataId, group, tenant);
    if (content != null) {
        LOGGER.warn("[{}] [get-config] get failover ok, dataId={}, group={}, tenant={}, config={}", agent.getName(),
                dataId, group, tenant, ContentUtils.truncateContent(content));
        cr.setContent(content);
        String encryptedDataKey = LocalEncryptedDataKeyProcessor
                .getEncryptDataKeyFailover(agent.getName(), dataId, group, tenant);
        cr.setEncryptedDataKey(encryptedDataKey);
        configFilterChainManager.doFilter(null, cr);
        content = cr.getContent();
        return content;
    }
    
    try {
    	// 获取配置接口，根据客户端负载均衡(轮询算法)
        ConfigResponse response = worker.getServerConfig(dataId, group, tenant, timeoutMs);
        cr.setContent(response.getContent());
        cr.setEncryptedDataKey(response.getEncryptedDataKey());
        
        configFilterChainManager.doFilter(null, cr);
        content = cr.getContent();
        
        return content;
    } catch (NacosException ioe) {
        if (NacosException.NO_RIGHT == ioe.getErrCode()) {
            throw ioe;
        }
        LOGGER.warn("[{}] [get-config] get from server error, dataId={}, group={}, tenant={}, msg={}",
                agent.getName(), dataId, group, tenant, ioe.toString());
    }
    
    LOGGER.warn("[{}] [get-config] get snapshot ok, dataId={}, group={}, tenant={}, config={}", agent.getName(),
            dataId, group, tenant, ContentUtils.truncateContent(content));
    content = LocalConfigInfoProcessor.getSnapshot(agent.getName(), dataId, group, tenant);
    cr.setContent(content);
    String encryptedDataKey = LocalEncryptedDataKeyProcessor
            .getEncryptDataKeyFailover(agent.getName(), dataId, group, tenant);
    cr.setEncryptedDataKey(encryptedDataKey);
    configFilterChainManager.doFilter(null, cr);
    content = cr.getContent();
    return content;
}
```

## 6.com.alibaba.cloud.nacos.refresh.NacosContextRefresher#registerNacosListenersForApplications|客户端动态修改配置
```java
private void registerNacosListenersForApplications() {
	if (isRefreshEnabled()) {
		for (NacosPropertySource propertySource : NacosPropertySourceRepository
				.getAll()) {
			if (!propertySource.isRefreshable()) {
				continue;
			}
			String dataId = propertySource.getDataId();
			registerNacosListener(propertySource.getGroup(), dataId);
		}
	}
}
```
- com.alibaba.cloud.nacos.refresh.NacosContextRefresher#registerNacosListener
```java
private void registerNacosListener(final String groupKey, final String dataKey) {
	String key = NacosPropertySourceRepository.getMapKey(dataKey, groupKey);
	Listener listener = listenerMap.computeIfAbsent(key,
			lst -> new AbstractSharedListener() {
				@Override
				public void innerReceive(String dataId, String group,
						String configInfo) {
					refreshCountIncrement();
					nacosRefreshHistory.addRefreshRecord(dataId, group, configInfo);
					// todo feature: support single refresh for listening
					applicationContext.publishEvent(
							new RefreshEvent(this, null, "Refresh Nacos config"));
					if (log.isDebugEnabled()) {
						log.debug(String.format(
								"Refresh Nacos config group=%s,dataId=%s,configInfo=%s",
								group, dataId, configInfo));
					}
				}
			});
	try {
		configService.addListener(dataKey, groupKey, listener);
	}
	catch (NacosException e) {
		log.warn(String.format(
				"register fail for nacos listener ,dataId=[%s],group=[%s]", dataKey,
				groupKey), e);
	}
}
```
- org.springframework.cloud.endpoint.event.RefreshEventListener#onApplicationEvent
```java
public void onApplicationEvent(ApplicationEvent event) {
	if (event instanceof ApplicationReadyEvent) {
		handle((ApplicationReadyEvent) event);
	}
	else if (event instanceof RefreshEvent) {
		handle((RefreshEvent) event);
	}
}
```
- org.springframework.cloud.endpoint.event.RefreshEventListener#handle(org.springframework.cloud.endpoint.event.RefreshEvent)
```java
public void handle(RefreshEvent event) {
	if (this.ready.get()) { // don't handle events before app is ready
		log.debug("Event received " + event.getEventDesc());
		Set<String> keys = this.refresh.refresh();
		log.info("Refresh keys changed: " + keys);
	}
}
```
- org.springframework.cloud.context.refresh.ContextRefresher#refresh
```java
public synchronized Set<String> refresh() {
	// spring刷新环境更新bean
	Set<String> keys = refreshEnvironment();
	// springCloud标准实现刷新配置
	this.scope.refreshAll();
	return keys;
}
```

## 7.com.alibaba.nacos.config.server.controller.ConfigController#getConfig|获取配置信息
```java
@GetMapping
@Secured(action = ActionTypes.READ, signType = SignType.CONFIG)
public void getConfig(HttpServletRequest request, HttpServletResponse response,
        @RequestParam("dataId") String dataId, @RequestParam("group") String group,
        @RequestParam(value = "tenant", required = false, defaultValue = StringUtils.EMPTY) String tenant,
        @RequestParam(value = "tag", required = false) String tag)
        throws IOException, ServletException, NacosException {
    // check tenant
    ParamUtils.checkTenant(tenant);
    tenant = NamespaceUtil.processNamespaceParameter(tenant);
    // check params
    ParamUtils.checkParam(dataId, group, "datumId", "content");
    ParamUtils.checkParam(tag);
    
    final String clientIp = RequestUtil.getRemoteIp(request);
    String isNotify = request.getHeader("notify");
    inner.doGetConfig(request, response, dataId, group, tenant, tag, isNotify, clientIp);
}
```
```java
public String doGetConfig(HttpServletRequest request, HttpServletResponse response, String dataId, String group,
            String tenant, String tag, String isNotify, String clientIp) throws IOException, ServletException {
        
    boolean notify = false;
    if (StringUtils.isNotBlank(isNotify)) {
        notify = Boolean.valueOf(isNotify);
    }
    
    final String groupKey = GroupKey2.getKey(dataId, group, tenant);
    String autoTag = request.getHeader("Vipserver-Tag");
    
    String requestIpApp = RequestUtil.getAppName(request);
    int lockResult = tryConfigReadLock(groupKey);
    
    final String requestIp = RequestUtil.getRemoteIp(request);
    boolean isBeta = false;
    boolean isSli = false;
    if (lockResult > 0) {
        // LockResult > 0 means cacheItem is not null and other thread can`t delete this cacheItem
        FileInputStream fis = null;
        try {
            String md5 = Constants.NULL;
            long lastModified = 0L;
            CacheItem cacheItem = ConfigCacheService.getContentCache(groupKey);
            if (cacheItem.isBeta() && cacheItem.getIps4Beta().contains(clientIp)) {
                isBeta = true;
            }
            
            final String configType =
                    (null != cacheItem.getType()) ? cacheItem.getType() : FileTypeEnum.TEXT.getFileType();
            response.setHeader("Config-Type", configType);
            FileTypeEnum fileTypeEnum = FileTypeEnum.getFileTypeEnumByFileExtensionOrFileType(configType);
            String contentTypeHeader = fileTypeEnum.getContentType();
            response.setHeader(HttpHeaderConsts.CONTENT_TYPE, contentTypeHeader);
            
            File file = null;
            ConfigInfoBase configInfoBase = null;
            PrintWriter out = null;
            if (isBeta) {
            	// md5判断文件是否发生变化,即磁盘读文件，mysql数据是在启动的时候加载到本地信息，在DumpService中。如果发生变化通过LocalDataChangeEvent事件通知，依赖长轮询机制
                md5 = cacheItem.getMd54Beta();
                lastModified = cacheItem.getLastModifiedTs4Beta();
                if (PropertyUtil.isDirectRead()) {
                    configInfoBase = persistService.findConfigInfo4Beta(dataId, group, tenant);
                } else {
                    file = DiskUtil.targetBetaFile(dataId, group, tenant);
                }
                response.setHeader("isBeta", "true");
            } else {
                if (StringUtils.isBlank(tag)) {
                    if (isUseTag(cacheItem, autoTag)) {
                        if (cacheItem.tagMd5 != null) {
                            md5 = cacheItem.tagMd5.get(autoTag);
                        }
                        if (cacheItem.tagLastModifiedTs != null) {
                            lastModified = cacheItem.tagLastModifiedTs.get(autoTag);
                        }
                        if (PropertyUtil.isDirectRead()) {
                            configInfoBase = persistService.findConfigInfo4Tag(dataId, group, tenant, autoTag);
                        } else {
                            file = DiskUtil.targetTagFile(dataId, group, tenant, autoTag);
                        }
                        
                        response.setHeader(com.alibaba.nacos.api.common.Constants.VIPSERVER_TAG,
                                URLEncoder.encode(autoTag, StandardCharsets.UTF_8.displayName()));
                    } else {
                        md5 = cacheItem.getMd5();
                        lastModified = cacheItem.getLastModifiedTs();
                        if (PropertyUtil.isDirectRead()) {
                            configInfoBase = persistService.findConfigInfo(dataId, group, tenant);
                        } else {
                            file = DiskUtil.targetFile(dataId, group, tenant);
                        }
                        if (configInfoBase == null && fileNotExist(file)) {
                            // FIXME CacheItem
                            // No longer exists. It is impossible to simply calculate the push delayed. Here, simply record it as - 1.
                            ConfigTraceService.logPullEvent(dataId, group, tenant, requestIpApp, -1,
                                    ConfigTraceService.PULL_EVENT_NOTFOUND, -1, requestIp, notify);
                            
                            // pullLog.info("[client-get] clientIp={}, {},
                            // no data",
                            // new Object[]{clientIp, groupKey});
                            
                            return get404Result(response);
                        }
                        isSli = true;
                    }
                } else {
                    if (cacheItem.tagMd5 != null) {
                        md5 = cacheItem.tagMd5.get(tag);
                    }
                    if (cacheItem.tagLastModifiedTs != null) {
                        Long lm = cacheItem.tagLastModifiedTs.get(tag);
                        if (lm != null) {
                            lastModified = lm;
                        }
                    }
                    if (PropertyUtil.isDirectRead()) {
                        configInfoBase = persistService.findConfigInfo4Tag(dataId, group, tenant, tag);
                    } else {
                        file = DiskUtil.targetTagFile(dataId, group, tenant, tag);
                    }
                    if (configInfoBase == null && fileNotExist(file)) {
                        // FIXME CacheItem
                        // No longer exists. It is impossible to simply calculate the push delayed. Here, simply record it as - 1.
                        ConfigTraceService.logPullEvent(dataId, group, tenant, requestIpApp, -1,
                                ConfigTraceService.PULL_EVENT_NOTFOUND, -1, requestIp, notify && isSli);
                        
                        // pullLog.info("[client-get] clientIp={}, {},
                        // no data",
                        // new Object[]{clientIp, groupKey});
                        
                        response.setStatus(HttpServletResponse.SC_NOT_FOUND);
                        response.getWriter().println("config data not exist");
                        return HttpServletResponse.SC_NOT_FOUND + "";
                    }
                }
            }
            
            response.setHeader(Constants.CONTENT_MD5, md5);
            
            // Disable cache.
            response.setHeader("Pragma", "no-cache");
            response.setDateHeader("Expires", 0);
            response.setHeader("Cache-Control", "no-cache,no-store");
            if (PropertyUtil.isDirectRead()) {
                response.setDateHeader("Last-Modified", lastModified);
            } else {
                fis = new FileInputStream(file);
                response.setDateHeader("Last-Modified", file.lastModified());
            }
            
            if (PropertyUtil.isDirectRead()) {
                out = response.getWriter();
                out.print(configInfoBase.getContent());
                out.flush();
                out.close();
            } else {
                fis.getChannel()
                        .transferTo(0L, fis.getChannel().size(), Channels.newChannel(response.getOutputStream()));
            }
            
            LogUtil.PULL_CHECK_LOG.warn("{}|{}|{}|{}", groupKey, requestIp, md5, TimeUtils.getCurrentTimeStr());
            
            final long delayed = System.currentTimeMillis() - lastModified;
            
            // TODO distinguish pull-get && push-get
            /*
             Otherwise, delayed cannot be used as the basis of push delay directly,
             because the delayed value of active get requests is very large.
             */
            ConfigTraceService.logPullEvent(dataId, group, tenant, requestIpApp, lastModified,
                    ConfigTraceService.PULL_EVENT_OK, delayed, requestIp, notify && isSli);
            
        } finally {
            releaseConfigReadLock(groupKey);
            IoUtils.closeQuietly(fis);
        }
    } else if (lockResult == 0) {
        
        // FIXME CacheItem No longer exists. It is impossible to simply calculate the push delayed. Here, simply record it as - 1.
        ConfigTraceService
                .logPullEvent(dataId, group, tenant, requestIpApp, -1, ConfigTraceService.PULL_EVENT_NOTFOUND, -1,
                        requestIp, notify && isSli);
        
        return get404Result(response);
        
    } else {
        
        PULL_LOG.info("[client-get] clientIp={}, {}, get data during dump", clientIp, groupKey);
        
        response.setStatus(HttpServletResponse.SC_CONFLICT);
        response.getWriter().println("requested file is being modified, please try later.");
        return HttpServletResponse.SC_CONFLICT + "";
        
    }
    
    return HttpServletResponse.SC_OK + "";
}
```