# 【框架类源码】07-spring启动整体流程
> 欢迎关注b站账号/公众号【六边形战士夏宁】，一个要把各项指标拉满的男人。该文章已在[github目录](https://github.com/edanlx/SealBook/blob/master/catalogue/wechat.md)收录。
屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。

## 1.背景
springboot启动笔记，spring.factories使用了与SPI一样的思想，但是其并非接口

## 3.run	
```java
public static ConfigurableApplicationContext run(Class<?> primarySource, String... args) {
	return run(new Class<?>[] { primarySource }, args);
}
```
```java
public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
	return new SpringApplication(primarySources).run(args);
}
```

### 3.1SpringApplication
```java
public SpringApplication(Class<?>... primarySources) {
		this(null, primarySources);
	}
```
```java
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
	this.resourceLoader = resourceLoader;
	Assert.notNull(primarySources, "PrimarySources must not be null");
	// 将启动类存起来，挡在@Configuration启动类使用
	this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
	// 推算当前webflux还是servlet
	this.webApplicationType = WebApplicationType.deduceFromClasspath();
	// 读取spring.factories,获取ApplicationContextInitializer
	setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
	// 读取spring.factories,获取ApplicationListener
	setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
	// 获取启动类
	this.mainApplicationClass = deduceMainApplicationClass();
}
```

### 3.2run
```java
public ConfigurableApplicationContext run(String... args) {
	StopWatch stopWatch = new StopWatch();
	stopWatch.start();
	ConfigurableApplicationContext context = null;
	Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
	configureHeadlessProperty();
	SpringApplicationRunListeners listeners = getRunListeners(args);
	// 执行监听器相应回调ApplicationStartingEvent
	listeners.starting();
	try {
		ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
		// 通过监听器扩展点读取springBoot配置文件(ConfigFileApplicationListener)
		ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
		configureIgnoreBeanInfo(environment);
		// 打印banner
		Banner printedBanner = printBanner(environment);
		// 创建spring上下文AnnotationConfigServletWebServerApplicationContext
		context = createApplicationContext();
		// 保存分析报告节点，用于出错时打印信息
		exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
				new Class[] { ConfigurableApplicationContext.class }, context);
		// 即将启动类转化成beanDefinition相当于手动注入@Configuration
		prepareContext(context, environment, listeners, applicationArguments, printedBanner);
		// 调用spring容器核心refresh，进入spring本身的主流程
		refreshContext(context);
		afterRefresh(context, applicationArguments);
		stopWatch.stop();
		if (this.logStartupInfo) {
			new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
		}
		listeners.started(context);
		callRunners(context, applicationArguments);
	}
	catch (Throwable ex) {
		handleRunFailure(context, ex, exceptionReporters, listeners);
		throw new IllegalStateException(ex);
	}

	try {
		listeners.running(context);
	}
	catch (Throwable ex) {
		handleRunFailure(context, ex, exceptionReporters, null);
		throw new IllegalStateException(ex);
	}
	return context;
}
```

#### 3.2.1prepareEnvironment
```java
private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners,
			ApplicationArguments applicationArguments) {
	// Create and configure the environment
	ConfigurableEnvironment environment = getOrCreateEnvironment();
	configureEnvironment(environment, applicationArguments.getSourceArgs());
	ConfigurationPropertySources.attach(environment);
	// ApplicationEnvironmentPreparedEvent监听回调
	listeners.environmentPrepared(environment);
	bindToSpringApplication(environment);
	if (!this.isCustomEnvironment) {
		environment = new EnvironmentConverter(getClassLoader()).convertEnvironmentIfNecessary(environment,
				deduceEnvironmentClass());
	}
	ConfigurationPropertySources.attach(environment);
	return environment;
}
```

#### 3.2.2prepareContext
prepareContext->load(context, sources.toArray(new Object[0]));->loader.load();->load(source)
```java
private int load(Object source) {
	// 一般启动都是走class分支
	Assert.notNull(source, "Source must not be null");
	if (source instanceof Class<?>) {
		return load((Class<?>) source);
	}
	if (source instanceof Resource) {
		return load((Resource) source);
	}
	if (source instanceof Package) {
		return load((Package) source);
	}
	if (source instanceof CharSequence) {
		return load((CharSequence) source);
	}
	throw new IllegalArgumentException("Invalid source type " + source.getClass());
}
```
```java
private int load(Class<?> source) {
	if (isGroovyPresent() && GroovyBeanDefinitionSource.class.isAssignableFrom(source)) {
		// Any GroovyLoaders added in beans{} DSL can contribute beans here
		GroovyBeanDefinitionSource loader = BeanUtils.instantiateClass(source, GroovyBeanDefinitionSource.class);
		load(loader);
	}
	if (isComponent(source)) {
		// 此处将启动类当做@Configureation进行注册
		this.annotatedReader.register(source);
		return 1;
	}
	return 0;
}
```

#### 3.2.3onRefresh
从spring的refresh()->onRefresh
```java
protected void onRefresh() {
		super.onRefresh();
		try {
			// 核心方法
			createWebServer();
		}
		catch (Throwable ex) {
			throw new ApplicationContextException("Unable to start web server", ex);
		}
	}
```

```java
private void createWebServer() {
	WebServer webServer = this.webServer;
	ServletContext servletContext = getServletContext();
	// 判断是内置还是外置tomcat，内置tomcat才需要创建
	if (webServer == null && servletContext == null) {
		ServletWebServerFactory factory = getWebServerFactory();
		this.webServer = factory.getWebServer(getSelfInitializer());
		getBeanFactory().registerSingleton("webServerGracefulShutdown",
				new WebServerGracefulShutdownLifecycle(this.webServer));
		getBeanFactory().registerSingleton("webServerStartStop",
				new WebServerStartStopLifecycle(this, this.webServer));
	}
	else if (servletContext != null) {
		try {
			getSelfInitializer().onStartup(servletContext);
		}
		catch (ServletException ex) {
			throw new ApplicationContextException("Cannot initialize servlet context", ex);
		}
	}
	initPropertySources();
}
```
#### 3.2.3.1getSelfInitializer
这里返回一个函数指针传入getWebServer
```java
private org.springframework.boot.web.servlet.ServletContextInitializer getSelfInitializer() {
		return this::selfInitialize;
	}

private void selfInitialize(ServletContext servletContext) throws ServletException {
	prepareWebApplicationContext(servletContext);
	registerApplicationScope(servletContext);
	WebApplicationContextUtils.registerEnvironmentBeans(getBeanFactory(), servletContext);
	// 这些bean由AutoConfiguration提供经，例如DispatcherServletAutoConfiguration，向ServletRegistrationBean进行注册
	// ServletRegistrationBean.onStartup->DynamicRegistrationBean.register(description, servletContext)->ServletRegistrationBean.addRegistration(description, servletContext)->servletContext.addServlet(name, this.servlet)
	for (ServletContextInitializer beans : getServletContextInitializerBeans()) {
		// 向tomcat中注册onStartUp等待回调，包括DispatcherServletRegistrationBean
		beans.onStartup(servletContext);
	}
}
```
#### 3.2.3.2getWebServer
```java
public WebServer getWebServer(ServletContextInitializer... initializers) {
	if (this.disableMBeanRegistry) {
		Registry.disableRegistry();
	}
	// 建立tomcat对象
	Tomcat tomcat = new Tomcat();
	File baseDir = (this.baseDirectory != null) ? this.baseDirectory : createTempDir("tomcat");
	tomcat.setBaseDir(baseDir.getAbsolutePath());
	Connector connector = new Connector(this.protocol);
	connector.setThrowOnFailure(true);
	tomcat.getService().addConnector(connector);
	customizeConnector(connector);
	tomcat.setConnector(connector);
	tomcat.getHost().setAutoDeploy(false);
	configureEngine(tomcat.getEngine());
	for (Connector additionalConnector : this.additionalTomcatConnectors) {
		tomcat.getService().addConnector(additionalConnector);
	}
	prepareContext(tomcat.getHost(), initializers);
	// 核心启动流程
	return getTomcatWebServer(tomcat);
}
```

#### 3.2.3.2.1getTomcatWebServer
getTomcatWebServer()->new TomcatWebServer()->initialize()
```java
private void initialize() throws WebServerException {
	logger.info("Tomcat initialized with port(s): " + getPortsDescription(false));
	synchronized (this.monitor) {
		try {
			addInstanceIdToEngineName();

			Context context = findContext();
			context.addLifecycleListener((event) -> {
				if (context.equals(event.getSource()) && Lifecycle.START_EVENT.equals(event.getType())) {
					// Remove service connectors so that protocol binding doesn't
					// happen when the service is started.
					removeServiceConnectors();
				}
			});

			// Start the server to trigger initialization listeners
			// tomcat启动
			this.tomcat.start();

			// We can re-throw failure exception directly in the main thread
			rethrowDeferredStartupExceptions();

			try {
				ContextBindings.bindClassLoader(context, context.getNamingToken(), getClass().getClassLoader());
			}
			catch (NamingException ex) {
				// Naming is not enabled. Continue
			}

			// Unlike Jetty, all Tomcat threads are daemon threads. We create a
			// blocking non-daemon to stop immediate shutdown
			// tomcat进行await挂起成为服务端
			startDaemonAwaitThread();
		}
		catch (Exception ex) {
			stopSilently();
			destroySilently();
			throw new WebServerException("Unable to start embedded Tomcat", ex);
		}
	}
}
```

## 4.@SpringBootApplication自动装配
@import的三种主要使用方式，springboot使用的是DeferredImportSelector
1. import直接导入bean
2. import导入的bean实现importSelector(再定义要导入的ben)->DeferredImportSelector(在将@Component加载成beanDefinition之后，组内顺序)
3. import导入的bean实现importBeanDefinitionRegister(使用beanDefinition注册)
### 4.1spring.factories
文件读取路径  
@SpringBootApplication->@EnableAutoConfiguration->AutoConfigurationImportSelector.class->getImportGroup()->AutoConfigurationGroup.class->process()->getAutoConfigurationEntry()->getCandidateConfigurations()->loadFactoryNames()->loadSpringFactories()->FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories"


### 4.2EnableAutoConfiguration
@SpringBootApplication->@EnableAutoConfiguration->AutoConfigurationImportSelector.class->getImportGroup()->AutoConfigurationGroup.class->process()->getAutoConfigurationEntry()->getCandidateConfigurations()->getSpringFactoriesLoaderFactoryClass()->EnableAutoConfiguration.class

```java
protected AutoConfigurationEntry getAutoConfigurationEntry(AnnotationMetadata annotationMetadata) {
	if (!isEnabled(annotationMetadata)) {
		return EMPTY_ENTRY;
	}
	AnnotationAttributes attributes = getAttributes(annotationMetadata);
	// 拿到自动配置类
	List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);
	// 去除重复
	configurations = removeDuplicates(configurations);
	// EnableAutoConfiguration中配置的exclude、spring.autoconfigure.exclude
	Set<String> exclusions = getExclusions(annotationMetadata, attributes);
	checkExcludedClasses(configurations, exclusions);
	configurations.removeAll(exclusions);
	// AutoConfigurationImportFilters进行过滤
	configurations = getConfigurationClassFilter().filter(configurations);
	fireAutoConfigurationImportEvents(configurations, exclusions);
	return new AutoConfigurationEntry(configurations, exclusions);
}
```
在该方法内进行分组排序，保证其实例化在最后以检测如果有实现相同接口的类则不易springboot为准，以自己实现的为准
```java
public Iterable<Entry> selectImports() {
		if (this.autoConfigurationEntries.isEmpty()) {
			return Collections.emptyList();
		}
		Set<String> allExclusions = this.autoConfigurationEntries.stream()
				.map(AutoConfigurationEntry::getExclusions).flatMap(Collection::stream).collect(Collectors.toSet());
		Set<String> processedConfigurations = this.autoConfigurationEntries.stream()
				.map(AutoConfigurationEntry::getConfigurations).flatMap(Collection::stream)
				.collect(Collectors.toCollection(LinkedHashSet::new));
		processedConfigurations.removeAll(allExclusions);

		return sortAutoConfigurations(processedConfigurations, getAutoConfigurationMetadata()).stream()
				.map((importClassName) -> new Entry(this.entries.get(importClassName), importClassName))
				.collect(Collectors.toList());
	}
```

## 5.springboot启动
### 5.1内置tomcat
1. 通过springboot-starter-plugin生成MANIFEST.MF
2. 所有依赖的jar文件都在BOOT-INF/lib中。class在BOOT-INF/classes
3. 其反射调用MANIFEST.MF中生成的Start-Class反射执行,org.springframework.boot.loader.JarLauncher。classloader不能加载jar包中的jar,所以是springboot自行加载
idea调试:idea启动选择jarApplication选择jar包
### 5.2外置tomcat
因为外置tomcat所以上文中自动装配给tomcat回调的onStartUp不会生效,此时则与SSM版本的javaConfig配置tomcat一致。在sping-webjar包中META-INF/services/javax.servlet.ServletContainerInitializer文件中有org.springframework.web.SpringServletContainerInitializer。该类由tomcat进行SPI实例化并调用onStartup。
***SpringBootServletInitializer***和***AbstractDispatcherServletInitializer***分别是springboot启动，和dispatcher向tomcat中接入servlet。
