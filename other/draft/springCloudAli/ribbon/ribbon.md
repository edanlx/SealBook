- IclientConfig
客户端配置，默认采用DefaultClientConfigimpl
- IRule
负载均衡策略，ZoneAvoidanceRule，可以自行实现灰度发布、同机群(如果同机没有再把其它的列表放进来)
- IPing
Ribbon的实例检查策略，DummyPing
- ServerList
实例清单的维护ConfigurationBasedServerList，同IRule
- ServerListFilter
服务实例清单过滤机制，默认采用ZonePreferenceServerListFilter
- ILoadBalancer
负载均衡器，默认采用ZoneAwareLoadBalancer

feign主要是增强注解，有日志、鉴权拦截器、重试发送等，但最主要还是以代理对象的方式调用服务，甚至在底层可修改为dubbo
## 4.启动类org.springframework.cloud.client.loadbalancer.LoadBalancerAutoConfiguration
在@LoadBalanced同包下
```java
@Bean
public LoadBalancerInterceptor loadBalancerInterceptor(
		LoadBalancerClient loadBalancerClient,
		LoadBalancerRequestFactory requestFactory) {
	return new LoadBalancerInterceptor(loadBalancerClient, requestFactory);
}

@Bean
public RestTemplateCustomizer restTemplateCustomizer(
				final LoadBalancerInterceptor loadBalancerInterceptor) {
	// 对restTemplate加一个拦截器
	return restTemplate -> {
		List<ClientHttpRequestInterceptor> list = new ArrayList<>(
				restTemplate.getInterceptors());
		list.add(loadBalancerInterceptor);
		restTemplate.setInterceptors(list);
	};
}


// 获取所有的restTemplate
@LoadBalanced
@Autowired(required = false)
private List<RestTemplate> restTemplates = Collections.emptyList();

@Bean
public SmartInitializingSingleton loadBalancedRestTemplateInitializerDeprecated(
			final ObjectProvider<List<RestTemplateCustomizer>> restTemplateCustomizers) {
	// 所有的restTemplate加上拦截器
	return () -> restTemplateCustomizers.ifAvailable(customizers -> {
		for (RestTemplate restTemplate : LoadBalancerAutoConfiguration.this.restTemplates) {
			for (RestTemplateCustomizer customizer : customizers) {
				customizer.customize(restTemplate);
			}
		}
	});
}
```
- org.springframework.cloud.client.loadbalancer.LoadBalancerInterceptor#intercept
```java
public ClientHttpResponse intercept(final HttpRequest request, final byte[] body,
			final ClientHttpRequestExecution execution) throws IOException {
	// 获取URi
	final URI originalUri = request.getURI();
	// 获取host
	String serviceName = originalUri.getHost();
	Assert.state(serviceName != null,
			"Request URI does not contain a valid hostname: " + originalUri);
	return this.loadBalancer.execute(serviceName,
			this.requestFactory.createRequest(request, body, execution));
}
```
- org.springframework.cloud.netflix.ribbon.RibbonLoadBalancerClient#execute(java.lang.String, org.springframework.cloud.client.loadbalancer.LoadBalancerRequest<T>)
```java
public <T> T execute(String serviceId, LoadBalancerRequest<T> request)
			throws IOException {
	return execute(serviceId, request, null);
}
```
- org.springframework.cloud.netflix.ribbon.RibbonLoadBalancerClient#execute(java.lang.String, org.springframework.cloud.client.loadbalancer.LoadBalancerRequest<T>, java.lang.Object)
```java
public <T> T execute(String serviceId, LoadBalancerRequest<T> request, Object hint)
			throws IOException {
	// 负载均衡器
	ILoadBalancer loadBalancer = getLoadBalancer(serviceId);
	Server server = getServer(loadBalancer, hint);
	// 负载均衡
	if (server == null) {
		throw new IllegalStateException("No instances available for " + serviceId);
	}
	RibbonServer ribbonServer = new RibbonServer(serviceId, server,
			isSecure(server, serviceId),
			serverIntrospector(serviceId).getMetadata(server));

	return execute(serviceId, ribbonServer, request);
}
```

## 5.com.netflix.loadbalancer.DynamicServerListLoadBalancer#enableAndInitLearnNewServersFeature
```java
public void enableAndInitLearnNewServersFeature() {
    LOGGER.info("Using serverListUpdater {}", serverListUpdater.getClass().getSimpleName());
    // 定时调用updateAction，定时从nacos的客户端server列表
    serverListUpdater.start(updateAction);
}
```
```java
protected final ServerListUpdater.UpdateAction updateAction = new ServerListUpdater.UpdateAction() {
    @Override
    public void doUpdate() {
        updateListOfServers();
    }
};
```
- com.netflix.loadbalancer.DynamicServerListLoadBalancer#updateListOfServers
```java
 public void updateListOfServers() {
    List<T> servers = new ArrayList<T>();
    if (serverListImpl != null) {
    	// 获取列表，整合nacos则从nacos拿
        servers = serverListImpl.getUpdatedListOfServers();
        LOGGER.debug("List of Servers for {} obtained from Discovery client: {}",
                getIdentifier(), servers);

        if (filter != null) {
            servers = filter.getFilteredListOfServers(servers);
            LOGGER.debug("Filtered List of Servers for {} obtained from Discovery client: {}",
                    getIdentifier(), servers);
        }
    }
    updateAllServerList(servers);
}
```

## 6.@EnableFeignClients
@Import(FeignClientsRegistrar.class)
```java
public void registerBeanDefinitions(AnnotationMetadata metadata,
		BeanDefinitionRegistry registry) {
	registerDefaultConfiguration(metadata, registry);
	// 核心方法扫描@FeignClient放入spring
	registerFeignClients(metadata, registry);
}
```
- org.springframework.cloud.openfeign.FeignClientsRegistrar#registerFeignClients
```java
public void registerFeignClients(AnnotationMetadata metadata,
			BeanDefinitionRegistry registry) {

	LinkedHashSet<BeanDefinition> candidateComponents = new LinkedHashSet<>();
	Map<String, Object> attrs = metadata
			.getAnnotationAttributes(EnableFeignClients.class.getName());
	AnnotationTypeFilter annotationTypeFilter = new AnnotationTypeFilter(
			FeignClient.class);
	final Class<?>[] clients = attrs == null ? null
			: (Class<?>[]) attrs.get("clients");
	if (clients == null || clients.length == 0) {
		ClassPathScanningCandidateComponentProvider scanner = getScanner();
		scanner.setResourceLoader(this.resourceLoader);
		scanner.addIncludeFilter(new AnnotationTypeFilter(FeignClient.class));
		Set<String> basePackages = getBasePackages(metadata);
		for (String basePackage : basePackages) {
			candidateComponents.addAll(scanner.findCandidateComponents(basePackage));
		}
	}
	else {
		for (Class<?> clazz : clients) {
			candidateComponents.add(new AnnotatedGenericBeanDefinition(clazz));
		}
	}

	for (BeanDefinition candidateComponent : candidateComponents) {
		if (candidateComponent instanceof AnnotatedBeanDefinition) {
			// verify annotated class is an interface
			AnnotatedBeanDefinition beanDefinition = (AnnotatedBeanDefinition) candidateComponent;
			AnnotationMetadata annotationMetadata = beanDefinition.getMetadata();
			Assert.isTrue(annotationMetadata.isInterface(),
					"@FeignClient can only be specified on an interface");

			Map<String, Object> attributes = annotationMetadata
					.getAnnotationAttributes(FeignClient.class.getCanonicalName());

			String name = getClientName(attributes);
			registerClientConfiguration(registry, name,
					attributes.get("configuration"));
			// 核心方法
			registerFeignClient(registry, annotationMetadata, attributes);
		}
	}
}
```
- org.springframework.cloud.openfeign.FeignClientsRegistrar#registerFeignClient
```java
private void registerFeignClient(BeanDefinitionRegistry registry,
			AnnotationMetadata annotationMetadata, Map<String, Object> attributes) {
	String className = annotationMetadata.getClassName();
	// 由FeignClientFactoryBean进行初始化
	BeanDefinitionBuilder definition = BeanDefinitionBuilder
			.genericBeanDefinition(FeignClientFactoryBean.class);
	validate(attributes);
	definition.addPropertyValue("url", getUrl(attributes));
	definition.addPropertyValue("path", getPath(attributes));
	String name = getName(attributes);
	definition.addPropertyValue("name", name);
	String contextId = getContextId(attributes);
	definition.addPropertyValue("contextId", contextId);
	definition.addPropertyValue("type", className);
	definition.addPropertyValue("decode404", attributes.get("decode404"));
	definition.addPropertyValue("fallback", attributes.get("fallback"));
	definition.addPropertyValue("fallbackFactory", attributes.get("fallbackFactory"));
	definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);

	String alias = contextId + "FeignClient";
	AbstractBeanDefinition beanDefinition = definition.getBeanDefinition();
	beanDefinition.setAttribute(FactoryBean.OBJECT_TYPE_ATTRIBUTE, className);

	// has a default, won't be null
	boolean primary = (Boolean) attributes.get("primary");

	beanDefinition.setPrimary(primary);

	String qualifier = getQualifier(attributes);
	if (StringUtils.hasText(qualifier)) {
		alias = qualifier;
	}

	BeanDefinitionHolder holder = new BeanDefinitionHolder(beanDefinition, className,
			new String[] { alias });
	BeanDefinitionReaderUtils.registerBeanDefinition(holder, registry);
}
```

```java
public Object getObject() throws Exception {
	// JDK代理bean
		return getTarget();
	}

	/**
	 * @param <T> the target type of the Feign client
	 * @return a {@link Feign} client created with the specified data and the context
	 * information
	 */
<T> T getTarget() {
	FeignContext context = applicationContext.getBean(FeignContext.class);
	Feign.Builder builder = feign(context);

	if (!StringUtils.hasText(url)) {
		if (!name.startsWith("http")) {
			url = "http://" + name;
		}
		else {
			url = name;
		}
		url += cleanPath();
		return (T) loadBalance(builder, context,
				new HardCodedTarget<>(type, name, url));
	}
	if (StringUtils.hasText(url) && !url.startsWith("http")) {
		url = "http://" + url;
	}
	String url = this.url + cleanPath();
	Client client = getOptional(context, Client.class);
	if (client != null) {
		if (client instanceof LoadBalancerFeignClient) {
			// not load balancing because we have a url,
			// but ribbon is on the classpath, so unwrap
			client = ((LoadBalancerFeignClient) client).getDelegate();
		}
		if (client instanceof FeignBlockingLoadBalancerClient) {
			// not load balancing because we have a url,
			// but Spring Cloud LoadBalancer is on the classpath, so unwrap
			client = ((FeignBlockingLoadBalancerClient) client).getDelegate();
		}
		builder.client(client);
	}
	Targeter targeter = get(context, Targeter.class);
	return (T) targeter.target(this, builder, context,
			new HardCodedTarget<>(type, name, url));
}
```
## 7.org.springframework.cloud.openfeign.ribbon.LoadBalancerFeignClient#execute|Feign对Ribbon的核心调用
```java
public Response execute(Request request, Request.Options options) throws IOException {
	try {
		URI asUri = URI.create(request.url());
		String clientName = asUri.getHost();
		URI uriWithoutHost = cleanUrl(request.url(), clientName);
		// 核心调用
		FeignLoadBalancer.RibbonRequest ribbonRequest = new FeignLoadBalancer.RibbonRequest(
				this.delegate, request, uriWithoutHost);

		IClientConfig requestConfig = getClientConfig(options, clientName);
		return lbClient(clientName)
				.executeWithLoadBalancer(ribbonRequest, requestConfig).toResponse();
	}
	catch (ClientException e) {
		IOException io = findIOException(e);
		if (io != null) {
			throw io;
		}
		throw new RuntimeException(e);
	}
}
```
