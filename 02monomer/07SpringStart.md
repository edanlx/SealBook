# 【框架类源码】07-spring推断构造方法
> 欢迎关注b站账号/公众号【六边形战士夏宁】，一个要把各项指标拉满的男人。该文章已在[github目录](https://github.com/edanlx/SealBook/blob/master/catalogue/wechat.md)收录。
屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。

## 1.背景
spring推断构造方法
## 2.整体流程
1. 实例化AnnotationConfigApplicationContext->同时自动调用父类GenericApplicationContext构造方法this.beanFactory = new DefaultListableBeanFactory()->DefaultListableBeanFactory自动调用父类构造方法使用cglib策略同时ingnore(BeanNameAware、BeanFactoryAware、BeanClassLoaderAware)
2. 实例化reader->实例化环境变量对象、实例化@Conditional的解析器ConditionEvaluator->setDependencyComparator比较器等
3. 实例化scaaner
## 3.初始代码
```java
@ComponentScan("com.example.demo.lesson.spring")
public class AppConfig {
    @Bean
    public UserBean userBean() {
        return new UserBean();
    }
}
@Component
public class UserEntity implements FactoryBean<UserFacoryBean> {
	@Resource
    private UserBean userBean;
    @Override
    public UserFacoryBean getObject()  {
        System.out.println("实例化UserFacoryBean");
        return new UserFacoryBean();
    }

    @Override
    public Class<?> getObjectType() {
        return UserFacoryBean.class;
    }
}
```
## 4.源码过程
### 4.1refresh
```java
public void refresh() throws BeansException, IllegalStateException {
	synchronized (this.startupShutdownMonitor) {
		// Prepare this context for refreshing.
		// 准备环境变量
		prepareRefresh();

		// Tell the subclass to refresh the internal bean factory.
		// contetx.refresh()两种实现，一种重复刷新报错，另一种刷新bean，将bean销毁重新创建
		ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

		// Prepare the bean factory for use in this context.
		// 初始化classloader，以及多种默认类型转化器，包括ApplicationContextAware，处理aware回调的
		prepareBeanFactory(beanFactory);

		try {
			// Allows post-processing of the bean factory in context subclasses.
			// 模板方法
			postProcessBeanFactory(beanFactory);

			// Invoke factory processors registered as beans in the context.
			// 扫描
			invokeBeanFactoryPostProcessors(beanFactory);

			// Register bean processors that intercept bean creation.
			// 将自己实现的BeanPostProcessors放入集合
			registerBeanPostProcessors(beanFactory);

			// Initialize message source for this context.
			// 国际化
			initMessageSource();

			// Initialize event multicaster for this context.
			// 初始化事件发布器
			initApplicationEventMulticaster();

			// Initialize other special beans in specific context subclasses.
			// 模板方法
			onRefresh();

			// Check for listener beans and register them.
			// 注册事件监听器
			registerListeners();

			// Instantiate all remaining (non-lazy-init) singletons.
			// 初始化bean
			finishBeanFactoryInitialization(beanFactory);

			// Last step: publish corresponding event.
			finishRefresh();
		}

		catch (BeansException ex) {
			if (logger.isWarnEnabled()) {
				logger.warn("Exception encountered during context initialization - " +
						"cancelling refresh attempt: " + ex);
			}

			// Destroy already created singletons to avoid dangling resources.
			destroyBeans();

			// Reset 'active' flag.
			cancelRefresh(ex);

			// Propagate exception to caller.
			throw ex;
		}

		finally {
			// Reset common introspection caches in Spring's core, since we
			// might not ever need metadata for singleton beans anymore...
			resetCommonCaches();
		}
	}
}
```

#### 4.1.1prepareRefresh
```java
protected void prepareRefresh() {
	// Switch to active.
	this.startupDate = System.currentTimeMillis();
	this.closed.set(false);
	this.active.set(true);

	if (logger.isDebugEnabled()) {
		if (logger.isTraceEnabled()) {
			logger.trace("Refreshing " + this);
		}
		else {
			logger.debug("Refreshing " + getDisplayName());
		}
	}

	// Initialize any placeholder property sources in the context environment.
	/**
	 protected void initPropertySources() {
		ConfigurableEnvironment env = getEnvironment();
		if (env instanceof ConfigurableWebEnvironment) {
			((ConfigurableWebEnvironment) env).initPropertySources(this.servletContext, this.servletConfig);
		}
	}
	 * */
	// 将额外的环境变量如tomcat中的变量加入到spring容器中
	initPropertySources();

	// Validate that all properties marked as required are resolvable:
	// see ConfigurablePropertyResolver#setRequiredProperties
	// 验证环境变量必须key，可以自定义注册必须值
	getEnvironment().validateRequiredProperties();

	// Store pre-refresh ApplicationListeners...
	if (this.earlyApplicationListeners == null) {
		this.earlyApplicationListeners = new LinkedHashSet<>(this.applicationListeners);
	}
	else {
		// Reset local application listeners to pre-refresh state.
		this.applicationListeners.clear();
		this.applicationListeners.addAll(this.earlyApplicationListeners);
	}

	// Allow for the collection of early ApplicationEvents,
	// to be published once the multicaster is available...
	this.earlyApplicationEvents = new LinkedHashSet<>();
}
```

#### 4.1.2prepareBeanFactory
```java
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
	// Tell the internal bean factory to use the context's class loader etc.
	// 默认classloader
	beanFactory.setBeanClassLoader(getClassLoader());
	beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
	beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

	// Configure the bean factory with context callbacks.
	// aware回调
	beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
	// 如果set方法有该实现则默认执行
	beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
	beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
	beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
	beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
	beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
	beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

	// BeanFactory interface not registered as resolvable type in a plain factory.
	// MessageSource registered (and found for autowiring) as a bean.
	// 建立映射关系方便autowired回调
	beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
	beanFactory.registerResolvableDependency(ResourceLoader.class, this);
	beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
	beanFactory.registerResolvableDependency(ApplicationContext.class, this);

	// Register early post-processor for detecting inner beans as ApplicationListeners.
	// 负责将接口实现的listen进行注册,eventListenerMethodProcessor负责注解实现
	beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

	// Detect a LoadTimeWeaver and prepare for weaving, if found.
	if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
		beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
		// Set a temporary ClassLoader for type matching.
		beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
	}

	// Register default environment beans.
	if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
		beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
	}
	if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
		beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
	}
	if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
		beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
	}
}
```
#### 4.1.3finishBeanFactoryInitialization
```java
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
	// Initialize conversion service for this context.
	// 设置转换器
	if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
			beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
		beanFactory.setConversionService(
				beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
	}

	// Register a default embedded value resolver if no bean post-processor
	// (such as a PropertyPlaceholderConfigurer bean) registered any before:
	// at this point, primarily for resolution in annotation attribute values.
	// 设置占位符解析器
	if (!beanFactory.hasEmbeddedValueResolver()) {
		beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
	}

	// Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
	// 
	String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
	for (String weaverAwareName : weaverAwareNames) {
		getBean(weaverAwareName);
	}

	// Stop using the temporary ClassLoader for type matching.
	beanFactory.setTempClassLoader(null);

	// Allow for caching all bean definition metadata, not expecting further changes.
	beanFactory.freezeConfiguration();

	// Instantiate all remaining (non-lazy-init) singletons.
	// 实例化非懒加载bean
	beanFactory.preInstantiateSingletons();
}
```
#### 4.1.4finishRefresh
```java
protected void finishRefresh() {
	// Clear context-level resource caches (such as ASM metadata from scanning).
	// 清除ASM缓存
	clearResourceCaches();

	// Initialize lifecycle processor for this context.
	// 初始化spring容器生命周期(回调和事件发布类似)
	initLifecycleProcessor();

	// Propagate refresh to lifecycle processor first.
	// 回调Lifecycle.start()
	getLifecycleProcessor().onRefresh();

	// Publish the final event.
	// 发布事件
	publishEvent(new ContextRefreshedEvent(this));

	// Participate in LiveBeansView MBean, if active.
	LiveBeansView.registerApplicationContext(this);
}
```

### 4.2invokeBeanFactoryPostProcessors
```java
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
	// 先调postProcessBeanDefinitionRegistry，再回调beanFactory后置处理器(postProcessBeanFactory)
	PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());

	// Detect a LoadTimeWeaver and prepare for weaving, if found in the meantime
	// (e.g. through an @Bean method registered by ConfigurationClassPostProcessor)
	if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
		beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
		beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
	}
}
```
#### 4.2.1invokeBeanFactoryPostProcessors
```java
public static void invokeBeanFactoryPostProcessors(
			ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {
	// Invoke BeanDefinitionRegistryPostProcessors first, if any.
	Set<String> processedBeans = new HashSet<>();

	// 此处只有直接通过context添加的，如果通过接口写的此时还没生成对象
	if (beanFactory instanceof BeanDefinitionRegistry) {
		BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;
		List<BeanFactoryPostProcessor> regularPostProcessors = new ArrayList<>();
		List<BeanDefinitionRegistryPostProcessor> registryProcessors = new ArrayList<>();

		for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
			if (postProcessor instanceof BeanDefinitionRegistryPostProcessor) {
				BeanDefinitionRegistryPostProcessor registryProcessor =
						(BeanDefinitionRegistryPostProcessor) postProcessor;
				// 执行回调BeanDefinitionRegistryPostProcessor.postProcessBeanDefinitionRegistry()
				registryProcessor.postProcessBeanDefinitionRegistry(registry);
				registryProcessors.add(registryProcessor);
			}
			else {
				regularPostProcessors.add(postProcessor);
			}
		}

		// Do not initialize FactoryBeans here: We need to leave all regular beans
		// uninitialized to let the bean factory post-processors apply to them!
		// Separate between BeanDefinitionRegistryPostProcessors that implement
		// PriorityOrdered, Ordered, and the rest.
		List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<>();

		// First, invoke the BeanDefinitionRegistryPostProcessors that implement PriorityOrdered.
		// 执行实现PriorityOrdered.postProcessBeanDefinitionRegistry()接口的类
		String[] postProcessorNames =
				beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
		for (String ppName : postProcessorNames) {
			if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
				// 将BeanDefinition生成对象
				currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
				processedBeans.add(ppName);
			}
		}
		sortPostProcessors(currentRegistryProcessors, beanFactory);
		registryProcessors.addAll(currentRegistryProcessors);
		invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
		currentRegistryProcessors.clear();

		// Next, invoke the BeanDefinitionRegistryPostProcessors that implement Ordered.
		// 执行实现Ordered.postProcessBeanDefinitionRegistry()接口的类
		postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
		for (String ppName : postProcessorNames) {
			// 执行实现
			if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {
				currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
				processedBeans.add(ppName);
			}
		}
		sortPostProcessors(currentRegistryProcessors, beanFactory);
		registryProcessors.addAll(currentRegistryProcessors);
		invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
		currentRegistryProcessors.clear();

		// Finally, invoke all other BeanDefinitionRegistryPostProcessors until no further ones appear.
		boolean reiterate = true;
		// 重复循环，防止在beanFactory中定义beandefinition中beanFactory
		while (reiterate) {
			reiterate = false;
			// 执行前面没有实例化的BeanFactory
			postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
			for (String ppName : postProcessorNames) {
				if (!processedBeans.contains(ppName)) {
					currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
					processedBeans.add(ppName);
					reiterate = true;
				}
			}
			sortPostProcessors(currentRegistryProcessors, beanFactory);
			registryProcessors.addAll(currentRegistryProcessors);
			invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
			currentRegistryProcessors.clear();
		}

		// Now, invoke the postProcessBeanFactory callback of all processors handled so far.
		// 先执行实现子接口的BeanFactory后置处理器
		invokeBeanFactoryPostProcessors(registryProcessors, beanFactory);
		// 手动添加的
		invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
	}

	else {
		// Invoke factory processors registered with the context instance.
		invokeBeanFactoryPostProcessors(beanFactoryPostProcessors, beanFactory);
	}

	// Do not initialize FactoryBeans here: We need to leave all regular beans
	// uninitialized to let the bean factory post-processors apply to them!
	String[] postProcessorNames =
			beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);

	// Separate between BeanFactoryPostProcessors that implement PriorityOrdered,
	// Ordered, and the rest.
	List<BeanFactoryPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
	List<String> orderedPostProcessorNames = new ArrayList<>();
	List<String> nonOrderedPostProcessorNames = new ArrayList<>();
	for (String ppName : postProcessorNames) {
		if (processedBeans.contains(ppName)) {
			// skip - already processed in first phase above
		}
		else if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
			priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
		}
		else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
			orderedPostProcessorNames.add(ppName);
		}
		else {
			nonOrderedPostProcessorNames.add(ppName);
		}
	}

	// First, invoke the BeanFactoryPostProcessors that implement PriorityOrdered.
	sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
	invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);

	// Next, invoke the BeanFactoryPostProcessors that implement Ordered.
	List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList<>(orderedPostProcessorNames.size());
	for (String postProcessorName : orderedPostProcessorNames) {
		orderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
	}
	sortPostProcessors(orderedPostProcessors, beanFactory);
	invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);

	// Finally, invoke all other BeanFactoryPostProcessors.
	List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList<>(nonOrderedPostProcessorNames.size());
	for (String postProcessorName : nonOrderedPostProcessorNames) {
		nonOrderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
	}
	invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);

	// Clear cached merged bean definitions since the post-processors might have
	// modified the original metadata, e.g. replacing placeholders in values...
	beanFactory.clearMetadataCache();
}
```

### 4.3ConfigurationClassPostProcessor
#### 4.3.1postProcessBeanDefinitionRegistry
```java
public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
	int registryId = System.identityHashCode(registry);
	if (this.registriesPostProcessed.contains(registryId)) {
		throw new IllegalStateException(
				"postProcessBeanDefinitionRegistry already called on this post-processor against " + registry);
	}
	if (this.factoriesPostProcessed.contains(registryId)) {
		throw new IllegalStateException(
				"postProcessBeanFactory already called on this post-processor against " + registry);
	}
	this.registriesPostProcessed.add(registryId);
	// 寻找配置类并解析
	processConfigBeanDefinitions(registry);
}
```

#### 4.3.1.1postProcessBeanDefinitionRegistry
```java
public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
	List<BeanDefinitionHolder> configCandidates = new ArrayList<>();
	String[] candidateNames = registry.getBeanDefinitionNames();

	for (String beanName : candidateNames) {
		BeanDefinition beanDef = registry.getBeanDefinition(beanName);
		if (beanDef.getAttribute(ConfigurationClassUtils.CONFIGURATION_CLASS_ATTRIBUTE) != null) {
			if (logger.isDebugEnabled()) {
				logger.debug("Bean definition has already been processed as a configuration class: " + beanDef);
			}
		}
		// 检查配置类
		else if (ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory)) {
			configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));
		}
	}

	// Return immediately if no @Configuration classes were found
	if (configCandidates.isEmpty()) {
		return;
	}

	// Sort by previously determined @Order value, if applicable
	configCandidates.sort((bd1, bd2) -> {
		int i1 = ConfigurationClassUtils.getOrder(bd1.getBeanDefinition());
		int i2 = ConfigurationClassUtils.getOrder(bd2.getBeanDefinition());
		return Integer.compare(i1, i2);
	});

	// Detect any custom bean name generation strategy supplied through the enclosing application context
	SingletonBeanRegistry sbr = null;
	if (registry instanceof SingletonBeanRegistry) {
		sbr = (SingletonBeanRegistry) registry;
		if (!this.localBeanNameGeneratorSet) {
			BeanNameGenerator generator = (BeanNameGenerator) sbr.getSingleton(
					AnnotationConfigUtils.CONFIGURATION_BEAN_NAME_GENERATOR);
			if (generator != null) {
				this.componentScanBeanNameGenerator = generator;
				this.importBeanNameGenerator = generator;
			}
		}
	}

	if (this.environment == null) {
		this.environment = new StandardEnvironment();
	}

	// Parse each @Configuration class
	// 解析配置类，包括@bean、scan等
	ConfigurationClassParser parser = new ConfigurationClassParser(
			this.metadataReaderFactory, this.problemReporter, this.environment,
			this.resourceLoader, this.componentScanBeanNameGenerator, registry);

	Set<BeanDefinitionHolder> candidates = new LinkedHashSet<>(configCandidates);
	Set<ConfigurationClass> alreadyParsed = new HashSet<>(configCandidates.size());
	do {
		// 解析成相应对象 例如：@Bean->BeanMethods,properties->环境变量等
		parser.parse(candidates);
		parser.validate();

		Set<ConfigurationClass> configClasses = new LinkedHashSet<>(parser.getConfigurationClasses());
		configClasses.removeAll(alreadyParsed);

		// Read the model and create bean definitions based on its content
		if (this.reader == null) {
			this.reader = new ConfigurationClassBeanDefinitionReader(
					registry, this.sourceExtractor, this.resourceLoader, this.environment,
					this.importBeanNameGenerator, parser.getImportRegistry());
		}
		// 将前面的解析的对象进行注册等，例如BeanMethods->BeanDefinitions
		this.reader.loadBeanDefinitions(configClasses);
		alreadyParsed.addAll(configClasses);

		candidates.clear();BeanDefinition
		// 如果有新增的，则进行循环
		if (registry.getBeanDefinitionCount() > candidateNames.length) {
			String[] newCandidateNames = registry.getBeanDefinitionNames();
			Set<String> oldCandidateNames = new HashSet<>(Arrays.asList(candidateNames));
			Set<String> alreadyParsedClasses = new HashSet<>();
			for (ConfigurationClass configurationClass : alreadyParsed) {
				alreadyParsedClasses.add(configurationClass.getMetadata().getClassName());
			}
			for (String candidateName : newCandidateNames) {
				if (!oldCandidateNames.contains(candidateName)) {
					BeanDefinition bd = registry.getBeanDefinition(candidateName);
					if (ConfigurationClassUtils.checkConfigurationClassCandidate(bd, this.metadataReaderFactory) &&
							!alreadyParsedClasses.contains(bd.getBeanClassName())) {
						candidates.add(new BeanDefinitionHolder(bd, candidateName));
					}
				}
			}
			candidateNames = newCandidateNames;
		}
	}
	while (!candidates.isEmpty());

	// Register the ImportRegistry as a bean in order to support ImportAware @Configuration classes
	if (sbr != null && !sbr.containsSingleton(IMPORT_REGISTRY_BEAN_NAME)) {
		sbr.registerSingleton(IMPORT_REGISTRY_BEAN_NAME, parser.getImportRegistry());
	}

	if (this.metadataReaderFactory instanceof CachingMetadataReaderFactory) {
		// Clear cache in externally provided MetadataReaderFactory; this is a no-op
		// for a shared cache since it'll be cleared by the ApplicationContext.
		((CachingMetadataReaderFactory) this.metadataReaderFactory).clearCache();
	}
}
```

##### 4.3.1.1.1checkConfigurationClassCandidate
```java
public static boolean checkConfigurationClassCandidate(
			BeanDefinition beanDef, MetadataReaderFactory metadataReaderFactory) {

	String className = beanDef.getBeanClassName();
	if (className == null || beanDef.getFactoryMethodName() != null) {
		return false;
	}

	// 获取注解元数据
	AnnotationMetadata metadata;
	if (beanDef instanceof AnnotatedBeanDefinition &&
			className.equals(((AnnotatedBeanDefinition) beanDef).getMetadata().getClassName())) {
		// Can reuse the pre-parsed metadata from the given BeanDefinition...
		metadata = ((AnnotatedBeanDefinition) beanDef).getMetadata();
	}
	else if (beanDef instanceof AbstractBeanDefinition && ((AbstractBeanDefinition) beanDef).hasBeanClass()) {
		// Check already loaded Class if present...
		// since we possibly can't even load the class file for this Class.
		Class<?> beanClass = ((AbstractBeanDefinition) beanDef).getBeanClass();
		if (BeanFactoryPostProcessor.class.isAssignableFrom(beanClass) ||
				BeanPostProcessor.class.isAssignableFrom(beanClass) ||
				AopInfrastructureBean.class.isAssignableFrom(beanClass) ||
				EventListenerFactory.class.isAssignableFrom(beanClass)) {
			return false;
		}
		metadata = AnnotationMetadata.introspect(beanClass);
	}
	else {
		try {
			MetadataReader metadataReader = metadataReaderFactory.getMetadataReader(className);
			metadata = metadataReader.getAnnotationMetadata();
		}
		catch (IOException ex) {
			if (logger.isDebugEnabled()) {
				logger.debug("Could not find class file for introspecting configuration annotations: " +
						className, ex);
			}
			return false;
		}
	}
	// 获取@Configuration
	Map<String, Object> config = metadata.getAnnotationAttributes(Configuration.class.getName());
	// full配置类
	if (config != null && !Boolean.FALSE.equals(config.get("proxyBeanMethods"))) {
		beanDef.setAttribute(CONFIGURATION_CLASS_ATTRIBUTE, CONFIGURATION_CLASS_FULL);
	}
	// lite配置类,有@Component、@ComponentScan、@Import、@ImportResource或者有@Bean方法
	else if (config != null || isConfigurationCandidate(metadata)) {
		/**
		 * for (String indicator : candidateIndicators) {
			if (metadata.isAnnotated(indicator)) {
				return true;
			}
			metadata.hasAnnotatedMethods(Bean.class.getName());
		}
		 * */
		beanDef.setAttribute(CONFIGURATION_CLASS_ATTRIBUTE, CONFIGURATION_CLASS_LITE);
	}
	else {
		return false;
	}

	// It's a full or lite configuration candidate... Let's determine the order value, if any.
	Integer order = getOrder(metadata);
	if (order != null) {
		beanDef.setAttribute(ORDER_ATTRIBUTE, order);
	}

	return true;
}
```

##### 4.3.1.1.2parser.parse()
```java
public void parse(Set<BeanDefinitionHolder> configCandidates) {
	for (BeanDefinitionHolder holder : configCandidates) {
		BeanDefinition bd = holder.getBeanDefinition();
		try {
			if (bd instanceof AnnotatedBeanDefinition) {
				// 核心方法
				parse(((AnnotatedBeanDefinition) bd).getMetadata(), holder.getBeanName());
			}
			else if (bd instanceof AbstractBeanDefinition && ((AbstractBeanDefinition) bd).hasBeanClass()) {
				// 核心方法
				parse(((AbstractBeanDefinition) bd).getBeanClass(), holder.getBeanName());
			}
			else {
				// 核心方法
				parse(bd.getBeanClassName(), holder.getBeanName());
			}
		}
		catch (BeanDefinitionStoreException ex) {
			throw ex;
		}
		catch (Throwable ex) {
			throw new BeanDefinitionStoreException(
					"Failed to parse configuration class [" + bd.getBeanClassName() + "]", ex);
		}
	}

	this.deferredImportSelectorHandler.process();
}

protected void processConfigurationClass(ConfigurationClass configClass, Predicate<String> filter) throws IOException {
	// 是否有@Conditional注解
	if (this.conditionEvaluator.shouldSkip(configClass.getMetadata(), ConfigurationPhase.PARSE_CONFIGURATION)) {
		return;
	}

	ConfigurationClass existingClass = this.configurationClasses.get(configClass);
	if (existingClass != null) {
		if (configClass.isImported()) {
			if (existingClass.isImported()) {
				existingClass.mergeImportedBy(configClass);
			}
			// Otherwise ignore new imported config class; existing non-imported class overrides it.
			return;
		}
		else {
			// Explicit bean definition found, probably replacing an import.
			// Let's remove the old one and go with the new one.
			this.configurationClasses.remove(configClass);
			this.knownSuperclasses.values().removeIf(configClass::equals);
		}
	}

	// Recursively process the configuration class and its superclass hierarchy.
	SourceClass sourceClass = asSourceClass(configClass, filter);
	do {
		// 核心方法
		sourceClass = doProcessConfigurationClass(configClass, sourceClass, filter);
	}
	while (sourceClass != null);

	this.configurationClasses.put(configClass, configClass);
}

protected final SourceClass doProcessConfigurationClass(
			ConfigurationClass configClass, SourceClass sourceClass, Predicate<String> filter)
			throws IOException {

	if (configClass.getMetadata().isAnnotated(Component.class.getName())) {
		// Recursively process any member (nested) classes first
		processMemberClasses(configClass, sourceClass, filter);
	}

	// Process any @PropertySource annotations
	for (AnnotationAttributes propertySource : AnnotationConfigUtils.attributesForRepeatable(
			sourceClass.getMetadata(), PropertySources.class,
			org.springframework.context.annotation.PropertySource.class)) {
		if (this.environment instanceof ConfigurableEnvironment) {
			processPropertySource(propertySource);
		}
		else {
			logger.info("Ignoring @PropertySource annotation on [" + sourceClass.getMetadata().getClassName() +
					"]. Reason: Environment must implement ConfigurableEnvironment");
		}
	}

	// Process any @ComponentScan annotations
	Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(
			sourceClass.getMetadata(), ComponentScans.class, ComponentScan.class);
	if (!componentScans.isEmpty() &&
			!this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {
		for (AnnotationAttributes componentScan : componentScans) {
			// The config class is annotated with @ComponentScan -> perform the scan immediately
			// 扫描
			Set<BeanDefinitionHolder> scannedBeanDefinitions =
					this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
			// Check the set of scanned definitions for any further config classes and parse recursively if needed
			for (BeanDefinitionHolder holder : scannedBeanDefinitions) {
				BeanDefinition bdCand = holder.getBeanDefinition().getOriginatingBeanDefinition();
				if (bdCand == null) {
					bdCand = holder.getBeanDefinition();
				}
				if (ConfigurationClassUtils.checkConfigurationClassCandidate(bdCand, this.metadataReaderFactory)) {
					parse(bdCand.getBeanClassName(), holder.getBeanName());
				}
			}
		}
	}

	// Process any @Import annotations
	processImports(configClass, sourceClass, getImports(sourceClass), filter, true);

	// Process any @ImportResource annotations
	AnnotationAttributes importResource =
			AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ImportResource.class);
	if (importResource != null) {
		String[] resources = importResource.getStringArray("locations");
		Class<? extends BeanDefinitionReader> readerClass = importResource.getClass("reader");
		for (String resource : resources) {
			String resolvedResource = this.environment.resolveRequiredPlaceholders(resource);
			configClass.addImportedResource(resolvedResource, readerClass);
		}
	}

	// Process individual @Bean methods
	Set<MethodMetadata> beanMethods = retrieveBeanMethodMetadata(sourceClass);
	for (MethodMetadata methodMetadata : beanMethods) {
		configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
	}

	// Process default methods on interfaces
	processInterfaces(configClass, sourceClass);

	// Process superclass, if any
	if (sourceClass.getMetadata().hasSuperClass()) {
		String superclass = sourceClass.getMetadata().getSuperClassName();
		if (superclass != null && !superclass.startsWith("java") &&
				!this.knownSuperclasses.containsKey(superclass)) {
			this.knownSuperclasses.put(superclass, configClass);
			// Superclass found, return its annotation metadata and recurse
			return sourceClass.getSuperClass();
		}
	}

	// No superclass -> processing is complete
	return null;
}
```