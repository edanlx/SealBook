# 【框架类源码】07-spring启动整体流程
> 欢迎关注b站账号/公众号【六边形战士夏宁】，一个要把各项指标拉满的男人。该文章已在[github目录](https://github.com/edanlx/SealBook/blob/master/catalogue/wechat.md)收录。
屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。

## 1.背景
springMVC启动。如果是web.xml则需要配置org.springframework.web.context.ContextLoaderListener，其会启动spring容器。org.springframework.web.servlet.DispatcherServlet，其会启动子springMVC容器

## 3.javaConfig
### 加载springMVC的context和despatcherServlet
在sping-webjar包中META-INF/services/javax.servlet.ServletContainerInitializer文件中有org.springframework.web.SpringServletContainerInitializer。该类由tomcat进行SPI实例化并调用onStartup方法传入所有@HandlesTypes(WebApplicationInitializer.class)有该注解的方法(其中包含AbstractDispatcherServletInitializer、AbstractContextLoaderInitializer)
```java
public void onStartup(@Nullable Set<Class<?>> webAppInitializerClasses, ServletContext servletContext)
			throws ServletException {

	List<WebApplicationInitializer> initializers = new LinkedList<>();

	if (webAppInitializerClasses != null) {
		for (Class<?> waiClass : webAppInitializerClasses) {
			// Be defensive: Some servlet containers provide us with invalid classes,
			// no matter what @HandlesTypes says...
			if (!waiClass.isInterface() && !Modifier.isAbstract(waiClass.getModifiers()) &&
					WebApplicationInitializer.class.isAssignableFrom(waiClass)) {
				try {
					initializers.add((WebApplicationInitializer)
							ReflectionUtils.accessibleConstructor(waiClass).newInstance());
				}
				catch (Throwable ex) {
					throw new ServletException("Failed to instantiate WebApplicationInitializer class", ex);
				}
			}
		}
	}

	if (initializers.isEmpty()) {
		servletContext.log("No Spring WebApplicationInitializer types detected on classpath");
		return;
	}

	servletContext.log(initializers.size() + " Spring WebApplicationInitializers detected on classpath");
	AnnotationAwareOrderComparator.sort(initializers);
	for (WebApplicationInitializer initializer : initializers) {
		// 调用其它onStartup方法初始化
		initializer.onStartup(servletContext);
	}
}
```
### 初始化父容器
```java
public void contextInitialized(ServletContextEvent event) {
		initWebApplicationContext(event.getServletContext());
	}
```
```java
public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {
	if (servletContext.getAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE) != null) {
		throw new IllegalStateException(
				"Cannot initialize context because there is already a root application context present - " +
				"check whether you have multiple ContextLoader* definitions in your web.xml!");
	}

	servletContext.log("Initializing Spring root WebApplicationContext");
	Log logger = LogFactory.getLog(ContextLoader.class);
	if (logger.isInfoEnabled()) {
		logger.info("Root WebApplicationContext: initialization started");
	}
	long startTime = System.currentTimeMillis();

	try {
		// Store context in local instance variable, to guarantee that
		// it is available on ServletContext shutdown.
		// xml创建会等于null，SPI创建
		if (this.context == null) {
			this.context = createWebApplicationContext(servletContext);
		}
		if (this.context instanceof ConfigurableWebApplicationContext) {
			ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) this.context;
			if (!cwac.isActive()) {
				// The context has not yet been refreshed -> provide services such as
				// setting the parent context, setting the application context id, etc
				if (cwac.getParent() == null) {
					// The context instance was injected without an explicit parent ->
					// determine parent for root web application context, if any.
					ApplicationContext parent = loadParentContext(servletContext);
					cwac.setParent(parent);
				}
				// SPI创建的调用refresh
				configureAndRefreshWebApplicationContext(cwac, servletContext);
			}
		}
		servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);

		ClassLoader ccl = Thread.currentThread().getContextClassLoader();
		if (ccl == ContextLoader.class.getClassLoader()) {
			currentContext = this.context;
		}
		else if (ccl != null) {
			currentContextPerThread.put(ccl, this.context);
		}

		if (logger.isInfoEnabled()) {
			long elapsedTime = System.currentTimeMillis() - startTime;
			logger.info("Root WebApplicationContext initialized in " + elapsedTime + " ms");
		}

		return this.context;
	}
	catch (RuntimeException | Error ex) {
		logger.error("Context initialization failed", ex);
		servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, ex);
		throw ex;
	}
}
```

```java
protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac, ServletContext sc) {
	if (ObjectUtils.identityToString(wac).equals(wac.getId())) {
		// The application context id is still set to its original default value
		// -> assign a more useful id based on available information
		String idParam = sc.getInitParameter(CONTEXT_ID_PARAM);
		if (idParam != null) {
			wac.setId(idParam);
		}
		else {
			// Generate default id...
			wac.setId(ConfigurableWebApplicationContext.APPLICATION_CONTEXT_ID_PREFIX +
					ObjectUtils.getDisplayString(sc.getContextPath()));
		}
	}
	// 将servletContext放入到springContext中
	wac.setServletContext(sc);
	// 获取web.xml中的信息
	String configLocationParam = sc.getInitParameter(CONFIG_LOCATION_PARAM);
	if (configLocationParam != null) {
		wac.setConfigLocation(configLocationParam);
	}

	// The wac environment's #initPropertySources will be called in any case when the context
	// is refreshed; do it eagerly here to ensure servlet property sources are in place for
	// use in any post-processing or initialization that occurs below prior to #refresh
	ConfigurableEnvironment env = wac.getEnvironment();
	if (env instanceof ConfigurableWebEnvironment) {
		((ConfigurableWebEnvironment) env).initPropertySources(sc, null);
	}

	customizeContext(sc, wac);
	// 调用refresh
	wac.refresh();
}
```

### 初始化子容器
DispatcherServlet继承HttpServletBean，会被tomcat的启动过程调用init
```java
public final void init() throws ServletException {

	// Set bean properties from init parameters.
	PropertyValues pvs = new ServletConfigPropertyValues(getServletConfig(), this.requiredProperties);
	if (!pvs.isEmpty()) {
		try {
			BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
			ResourceLoader resourceLoader = new ServletContextResourceLoader(getServletContext());
			bw.registerCustomEditor(Resource.class, new ResourceEditor(resourceLoader, getEnvironment()));
			initBeanWrapper(bw);
			bw.setPropertyValues(pvs, true);
		}
		catch (BeansException ex) {
			if (logger.isErrorEnabled()) {
				logger.error("Failed to set bean properties on servlet '" + getServletName() + "'", ex);
			}
			throw ex;
		}
	}

	// Let subclasses do whatever initialization they like.
	initServletBean();
}
```

```java
protected final void initServletBean() throws ServletException {
	getServletContext().log("Initializing Spring " + getClass().getSimpleName() + " '" + getServletName() + "'");
	if (logger.isInfoEnabled()) {
		logger.info("Initializing Servlet '" + getServletName() + "'");
	}
	long startTime = System.currentTimeMillis();

	try {
		// 初始化springMVC子容器
		this.webApplicationContext = initWebApplicationContext();
		initFrameworkServlet();
	}
	catch (ServletException | RuntimeException ex) {
		logger.error("Context initialization failed", ex);
		throw ex;
	}

	if (logger.isDebugEnabled()) {
		String value = this.enableLoggingRequestDetails ?
				"shown which may lead to unsafe logging of potentially sensitive data" :
				"masked to prevent unsafe logging of potentially sensitive data";
		logger.debug("enableLoggingRequestDetails='" + this.enableLoggingRequestDetails +
				"': request parameters and headers will be " + value);
	}

	if (logger.isInfoEnabled()) {
		logger.info("Completed initialization in " + (System.currentTimeMillis() - startTime) + " ms");
	}
}
```

### 默认适配器等加载
```java
public void onApplicationEvent(ContextRefreshedEvent event) {
	this.refreshEventReceived = true;
	synchronized (this.onRefreshMonitor) {
		onRefresh(event.getApplicationContext());
	}
}
```
```java
protected void onRefresh(ApplicationContext context) {
		initStrategies(context);
	}
```
```java
protected void initStrategies(ApplicationContext context) {
	initMultipartResolver(context);
	initLocaleResolver(context);
	initThemeResolver(context);
	initHandlerMappings(context);// 从dispatcherServlet.properties加载
	initHandlerAdapters(context);
	initHandlerExceptionResolvers(context);
	initRequestToViewNameTranslator(context);
	initViewResolvers(context);
	initFlashMapManager(context);
}
```

## tomcat
### xml加载相关
spring.handlers加载org.springframework.web.servlet.config.MvcNamespaceHandler用于解析xml。例如<mvc:annotation-driven>和@EnableWebMvc、@Import(DelegatingWebMvcConfiguration.class)一致

## springboot
取消父子容器

