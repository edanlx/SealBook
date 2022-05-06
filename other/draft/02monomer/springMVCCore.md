# 【框架类源码】07-spring启动整体流程
> 欢迎关注b站账号/公众号【六边形战士夏宁】，一个要把各项指标拉满的男人。该文章已在[github目录](https://github.com/edanlx/SealBook/blob/master/catalogue/wechat.md)收录。
屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。

## 1.背景
1. 找到能够映射的handler,正常是RequestMappingHandlerMapping,返回handleMethod
2. 找到能够支持的adapter,正常是RequestMappingHandlerAdapter,在AbstractHandlerMethodAdapter.supports()中解析判断为true
3. 前置拦截器生命周期
4. 执行方法并获得modelAndView，解析入参和封装出参如果都是json都会使用messageConvert
5. 后置拦截器生命周期
6. 渲染视图

## 3.DispatcherServlet.doDispatch
DispatcherServlet继承FrameworkServlet继承HttpServletBean继承HttpServlet，在DispatcherServlet中的doService会调用doDispatch
```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
	HttpServletRequest processedRequest = request;
	HandlerExecutionChain mappedHandler = null;
	boolean multipartRequestParsed = false;

	WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

	try {
		ModelAndView mv = null;
		Exception dispatchException = null;

		try {
			processedRequest = checkMultipart(request);
			multipartRequestParsed = (processedRequest != request);

			// Determine handler for the current request.
			// 进行映射
			mappedHandler = getHandler(processedRequest);
			if (mappedHandler == null) {
				noHandlerFound(processedRequest, response);
				return;
			}

			// Determine handler adapter for the current request.
			// 获取适配器
			HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

			// Process last-modified header, if supported by the handler.
			String method = request.getMethod();
			boolean isGet = "GET".equals(method);
			if (isGet || "HEAD".equals(method)) {
				long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
				if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
					return;
				}
			}

			// 调用拦截器PreHandle生命周期
			if (!mappedHandler.applyPreHandle(processedRequest, response)) {
				return;
			}

			// Actually invoke the handler.
			// 解析参数，执行方法,并获得modelandview
			mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

			if (asyncManager.isConcurrentHandlingStarted()) {
				return;
			}

			applyDefaultViewName(processedRequest, mv);
			// 后置拦截器生命周期
			mappedHandler.applyPostHandle(processedRequest, response, mv);
		}
		catch (Exception ex) {
			dispatchException = ex;
		}
		catch (Throwable err) {
			// As of 4.3, we're processing Errors thrown from handler methods as well,
			// making them available for @ExceptionHandler methods and other scenarios.
			dispatchException = new NestedServletException("Handler dispatch failed", err);
		}
		// 渲染视图
		processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
	}
	catch (Exception ex) {
		// 拦截器AfterCompletion
		triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
	}
	catch (Throwable err) {
		triggerAfterCompletion(processedRequest, response, mappedHandler,
				new NestedServletException("Handler processing failed", err));
	}
	finally {
		if (asyncManager.isConcurrentHandlingStarted()) {
			// Instead of postHandle and afterCompletion
			if (mappedHandler != null) {
				mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
			}
		}
		else {
			// Clean up any resources used by a multipart request.
			if (multipartRequestParsed) {
				cleanupMultipart(processedRequest);
			}
		}
	}
}
```

### 3.1getHandler
有多个映射器，一般只会用RequestMappingHandlerMapping，即解析@RequestMapping注解的handler
```java
protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
	if (this.handlerMappings != null) {
		for (HandlerMapping mapping : this.handlerMappings) {
			HandlerExecutionChain handler = mapping.getHandler(request);
			if (handler != null) {
				return handler;
			}
		}
	}
	return null;
}
```

```java
public final HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
	Object handler = getHandlerInternal(request);
	if (handler == null) {
		handler = getDefaultHandler();
	}
	if (handler == null) {
		return null;
	}
	// Bean name or resolved handler?
	if (handler instanceof String) {
		String handlerName = (String) handler;
		handler = obtainApplicationContext().getBean(handlerName);
	}

	HandlerExecutionChain executionChain = getHandlerExecutionChain(handler, request);

	if (logger.isTraceEnabled()) {
		logger.trace("Mapped to " + handler);
	}
	else if (logger.isDebugEnabled() && !request.getDispatcherType().equals(DispatcherType.ASYNC)) {
		logger.debug("Mapped to " + executionChain.getHandler());
	}

	if (hasCorsConfigurationSource(handler) || CorsUtils.isPreFlightRequest(request)) {
		CorsConfiguration config = (this.corsConfigurationSource != null ? this.corsConfigurationSource.getCorsConfiguration(request) : null);
		CorsConfiguration handlerConfig = getCorsConfiguration(handler, request);
		config = (config != null ? config.combine(handlerConfig) : handlerConfig);
		executionChain = getCorsHandlerExecutionChain(request, executionChain, config);
	}

	return executionChain;
}
```

### 3.1.1getHandlerInternal
```java
protected HandlerMethod getHandlerInternal(HttpServletRequest request) throws Exception {
	String lookupPath = getUrlPathHelper().getLookupPathForRequest(request);
	request.setAttribute(LOOKUP_PATH, lookupPath);
	this.mappingRegistry.acquireReadLock();
	try {
		// 核心方法
		HandlerMethod handlerMethod = lookupHandlerMethod(lookupPath, request);
		return (handlerMethod != null ? handlerMethod.createWithResolvedBean() : null);
	}
	finally {
		this.mappingRegistry.releaseReadLock();
	}
}
```
接着进入lookupHandlerMethod
```java
protected HandlerMethod lookupHandlerMethod(String lookupPath, HttpServletRequest request) throws Exception {
	List<Match> matches = new ArrayList<>();
	// 直接匹配
	List<T> directPathMatches = this.mappingRegistry.getMappingsByUrl(lookupPath);
	if (directPathMatches != null) {
		// 根据注解的过滤条件进行过滤
		addMatchingMappings(directPathMatches, matches, request);
	}
	// 没找到则使用循环匹配利用ant表达式匹配
	if (matches.isEmpty()) {
		// No choice but to go through all mappings...
		addMatchingMappings(this.mappingRegistry.getMappings().keySet(), matches, request);
	}

	if (!matches.isEmpty()) {
		Match bestMatch = matches.get(0);
		if (matches.size() > 1) {
			Comparator<Match> comparator = new MatchComparator(getMappingComparator(request));
			// 匹配到多个进行排序，?>*>{}>**
			matches.sort(comparator);
			bestMatch = matches.get(0);
			if (logger.isTraceEnabled()) {
				logger.trace(matches.size() + " matching mappings: " + matches);
			}
			if (CorsUtils.isPreFlightRequest(request)) {
				return PREFLIGHT_AMBIGUOUS_MATCH;
			}
			Match secondBestMatch = matches.get(1);
			if (comparator.compare(bestMatch, secondBestMatch) == 0) {
				Method m1 = bestMatch.handlerMethod.getMethod();
				Method m2 = secondBestMatch.handlerMethod.getMethod();
				String uri = request.getRequestURI();
				throw new IllegalStateException(
						"Ambiguous handler methods mapped for '" + uri + "': {" + m1 + ", " + m2 + "}");
			}
		}
		request.setAttribute(BEST_MATCHING_HANDLER_ATTRIBUTE, bestMatch.handlerMethod);
		handleMatch(bestMatch.mapping, lookupPath, request);
		return bestMatch.handlerMethod;
	}
	else {
		return handleNoMatch(this.mappingRegistry.getMappings().keySet(), lookupPath, request);
	}
}
```
### 3.1.1.1getMappingsByUrl
直接匹配
```java
public List<T> getMappingsByUrl(String urlPath) {
		return this.urlLookup.get(urlPath);
	}
```

### 3.2getHandlerAdapter
有多个适配器，如果上一步是RequestMappingHandlerMapping返回的handleMethod，这一步会返回RequestMappingHandlerAdapter适配器(AbstractHandlerMethodAdapter)
```java
protected HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
	if (this.handlerAdapters != null) {
		for (HandlerAdapter adapter : this.handlerAdapters) {
			if (adapter.supports(handler)) {
				return adapter;
			}
		}
	}
	throw new ServletException("No adapter for handler [" + handler +
			"]: The DispatcherServlet configuration needs to include a HandlerAdapter that supports this handler");
}
```

### 3.3handle
```java
public final ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
		throws Exception {
	return handleInternal(request, response, (HandlerMethod) handler);
}
```

```java
protected ModelAndView handleInternal(HttpServletRequest request,
			HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

	ModelAndView mav;
	// 校验supportedMethods，默认不设，一般也不用，校验requireSession，一般也不配
	checkRequest(request);

	// Execute invokeHandlerMethod in synchronized block if required.
	// 是否顺序执行，一般走下面的并发逻辑
	if (this.synchronizeOnSession) {
		HttpSession session = request.getSession(false);
		if (session != null) {
			Object mutex = WebUtils.getSessionMutex(session);
			synchronized (mutex) {
				mav = invokeHandlerMethod(request, response, handlerMethod);
			}
		}
		else {
			// No HttpSession available -> no mutex necessary
			mav = invokeHandlerMethod(request, response, handlerMethod);
		}
	}
	else {
		// No synchronization on session demanded at all...
		// 获得modelAndView，如果是@ResponseBody则为null
		mav = invokeHandlerMethod(request, response, handlerMethod);
	}

	if (!response.containsHeader(HEADER_CACHE_CONTROL)) {
		if (getSessionAttributesHandler(handlerMethod).hasSessionAttributes()) {
			applyCacheSeconds(response, this.cacheSecondsForSessionAttributeHandlers);
		}
		else {
			prepareResponse(response);
		}
	}

	return mav;
}
```

### 3.3.1handle
```java
protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
			HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

	// 包装req、rep
	ServletWebRequest webRequest = new ServletWebRequest(request, response);
	try {
		// 配置InintBinder
		WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
		// modelAttribute
		ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);

		ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
		if (this.argumentResolvers != null) {
			// 参数解析器
			invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
		}
		if (this.returnValueHandlers != null) {
			// 返回值解析器
			invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
		}
		invocableMethod.setDataBinderFactory(binderFactory);
		invocableMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);

		ModelAndViewContainer mavContainer = new ModelAndViewContainer();
		mavContainer.addAllAttributes(RequestContextUtils.getInputFlashMap(request));
		// 调用@modelAttribute
		modelFactory.initModel(webRequest, mavContainer, invocableMethod);
		mavContainer.setIgnoreDefaultModelOnRedirect(this.ignoreDefaultModelOnRedirect);

		AsyncWebRequest asyncWebRequest = WebAsyncUtils.createAsyncWebRequest(request, response);
		asyncWebRequest.setTimeout(this.asyncRequestTimeout);

		WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
		asyncManager.setTaskExecutor(this.taskExecutor);
		asyncManager.setAsyncWebRequest(asyncWebRequest);
		asyncManager.registerCallableInterceptors(this.callableInterceptors);
		asyncManager.registerDeferredResultInterceptors(this.deferredResultInterceptors);

		if (asyncManager.hasConcurrentResult()) {
			Object result = asyncManager.getConcurrentResult();
			mavContainer = (ModelAndViewContainer) asyncManager.getConcurrentResultContext()[0];
			asyncManager.clearConcurrentResult();
			LogFormatUtils.traceDebug(logger, traceOn -> {
				String formatted = LogFormatUtils.formatValue(result, !traceOn);
				return "Resume with async result [" + formatted + "]";
			});
			invocableMethod = invocableMethod.wrapConcurrentResult(result);
		}

		// 调用目标方法
		invocableMethod.invokeAndHandle(webRequest, mavContainer);
		if (asyncManager.isConcurrentHandlingStarted()) {
			return null;
		}
		// 返回modelAndView
		return getModelAndView(mavContainer, modelFactory, webRequest);
	}
	finally {
		webRequest.requestCompleted();
	}
}
```
#### 3.3.1.1invokeAndHandle
```java
public void invokeAndHandle(ServletWebRequest webRequest, ModelAndViewContainer mavContainer,
			Object... providedArgs) throws Exception {
	// 解析参数并执行获得返回值
	Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);
	setResponseStatus(webRequest);

	if (returnValue == null) {
		if (isRequestNotModified(webRequest) || getResponseStatus() != null || mavContainer.isRequestHandled()) {
			disableContentCachingIfNecessary(webRequest);
			mavContainer.setRequestHandled(true);
			return;
		}
	}
	else if (StringUtils.hasText(getResponseStatusReason())) {
		mavContainer.setRequestHandled(true);
		return;
	}

	mavContainer.setRequestHandled(false);
	Assert.state(this.returnValueHandlers != null, "No return value handlers");
	try {
		// 处理返回值
		this.returnValueHandlers.handleReturnValue(
				returnValue, getReturnValueType(returnValue), mavContainer, webRequest);
	}
	catch (Exception ex) {
		if (logger.isTraceEnabled()) {
			logger.trace(formatErrorForReturnValue(returnValue), ex);
		}
		throw ex;
	}
}
```

```java
public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
			ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {
	// 此处如果是json就是RequestResponseBodyMethodProcessor，根据是否有responsebody注解
	HandlerMethodReturnValueHandler handler = selectHandler(returnValue, returnType);
	if (handler == null) {
		throw new IllegalArgumentException("Unknown return value type: " + returnType.getParameterType().getName());
	}
	// 调用所有的messageConvert.canWrite进行判断，然后就使用response.write进行写
	handler.handleReturnValue(returnValue, returnType, mavContainer, webRequest);
}
```

### 3.4processDispatchResult
```java
private void processDispatchResult(HttpServletRequest request, HttpServletResponse response,
			@Nullable HandlerExecutionChain mappedHandler, @Nullable ModelAndView mv,
			@Nullable Exception exception) throws Exception {

		boolean errorView = false;

	if (exception != null) {
		if (exception instanceof ModelAndViewDefiningException) {
			logger.debug("ModelAndViewDefiningException encountered", exception);
			mv = ((ModelAndViewDefiningException) exception).getModelAndView();
		}
		else {
			Object handler = (mappedHandler != null ? mappedHandler.getHandler() : null);
			mv = processHandlerException(request, response, handler, exception);
			errorView = (mv != null);
		}
	}

	// Did the handler return a view to render?
	if (mv != null && !mv.wasCleared()) {
		// 核心解析
		render(mv, request, response);
		if (errorView) {
			WebUtils.clearErrorRequestAttributes(request);
		}
	}
	else {
		if (logger.isTraceEnabled()) {
			logger.trace("No view rendering, null ModelAndView returned.");
		}
	}

	if (WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
		// Concurrent handling started during a forward
		return;
	}

	if (mappedHandler != null) {
		// Exception (if any) is already handled..
		mappedHandler.triggerAfterCompletion(request, response, null);
	}
}
```

#### 3.4.1render
```java
protected void render(ModelAndView mv, HttpServletRequest request, HttpServletResponse response) throws Exception {
		// Determine locale for request and apply it to the response.
		Locale locale =
				(this.localeResolver != null ? this.localeResolver.resolveLocale(request) : request.getLocale());
		response.setLocale(locale);

		View view;
		String viewName = mv.getViewName();
		if (viewName != null) {
			// We need to resolve the view name.
			// 使用resolve解析器拼接前后缀
			view = resolveViewName(viewName, mv.getModelInternal(), locale, request);
			if (view == null) {
				throw new ServletException("Could not resolve view with name '" + mv.getViewName() +
						"' in servlet with name '" + getServletName() + "'");
			}
		}
		else {
			// No need to lookup: the ModelAndView object contains the actual View object.
			view = mv.getView();
			if (view == null) {
				throw new ServletException("ModelAndView [" + mv + "] neither contains a view name nor a " +
						"View object in servlet with name '" + getServletName() + "'");
			}
		}

		// Delegate to the View object for rendering.
		if (logger.isTraceEnabled()) {
			logger.trace("Rendering view [" + view + "] ");
		}
		try {
			if (mv.getStatus() != null) {
				response.setStatus(mv.getStatus().value());
			}
			// 核心渲染
			view.render(mv.getModelInternal(), request, response);
		}
		catch (Exception ex) {
			if (logger.isDebugEnabled()) {
				logger.debug("Error rendering view [" + view + "]", ex);
			}
			throw ex;
		}
	}
```

##### 3.4.1.1AbstractView.render
```java
public void render(@Nullable Map<String, ?> model, HttpServletRequest request,
			HttpServletResponse response) throws Exception {

	if (logger.isDebugEnabled()) {
		logger.debug("View " + formatViewName() +
				", model " + (model != null ? model : Collections.emptyMap()) +
				(this.staticAttributes.isEmpty() ? "" : ", static attributes " + this.staticAttributes));
	}

	Map<String, Object> mergedModel = createMergedOutputModel(model, request, response);
	prepareResponse(request, response);
	// 核心方法
	renderMergedOutputModel(mergedModel, getRequestToExpose(request), response);
}
```
InternalResourceView.renderMergedOutputModel
```java
protected void renderMergedOutputModel(
			Map<String, Object> model, HttpServletRequest request, HttpServletResponse response) throws Exception {

	// Expose the model object as request attributes.
	// 循环set将model的值设置到request的attribute中
	exposeModelAsRequestAttributes(model, request);

	// Expose helpers as request attributes, if any.
	// 国际化资源
	exposeHelpers(request);

	// Determine the path for the request dispatcher.
	// 
	String dispatcherPath = prepareForRendering(request, response);

	// Obtain a RequestDispatcher for the target resource (typically a JSP).
	// request.getRequestDispatcher请求转发至jsp
	RequestDispatcher rd = getRequestDispatcher(request, dispatcherPath);
	if (rd == null) {
		throw new ServletException("Could not get RequestDispatcher for [" + getUrl() +
				"]: Check that the corresponding file exists within your web application archive!");
	}

	// If already included or response already committed, perform include, else forward.
	if (useInclude(request, response)) {
		response.setContentType(getContentType());
		if (logger.isDebugEnabled()) {
			logger.debug("Including [" + getUrl() + "]");
		}
		rd.include(request, response);
	}

	else {
		// Note: The forwarded resource is supposed to determine the content type itself.
		if (logger.isDebugEnabled()) {
			logger.debug("Forwarding to [" + getUrl() + "]");
		}
		rd.forward(request, response);
	}
}
```

## 4.RequestMappingHandlerMapping
### 4.1afterPropertiesSet
```java
public void afterPropertiesSet() {
		this.config = new RequestMappingInfo.BuilderConfiguration();
		this.config.setUrlPathHelper(getUrlPathHelper());
		this.config.setPathMatcher(getPathMatcher());
		this.config.setSuffixPatternMatch(useSuffixPatternMatch());
		this.config.setTrailingSlashMatch(useTrailingSlashMatch());
		this.config.setRegisteredSuffixPatternMatch(useRegisteredSuffixPatternMatch());
		this.config.setContentNegotiationManager(getContentNegotiationManager());

		super.afterPropertiesSet();
	}
```
```java
public void afterPropertiesSet() {
	initHandlerMethods();
}
```
```java
protected void initHandlerMethods() {
	// 获取所有beanName
	for (String beanName : getCandidateBeanNames()) {
		if (!beanName.startsWith(SCOPED_TARGET_NAME_PREFIX)) {
			processCandidateBean(beanName);
		}
	}
	handlerMethodsInitialized(getHandlerMethods());
}
```
#### 4.1.1processCandidateBean
```java
protected void processCandidateBean(String beanName) {
	Class<?> beanType = null;
	try {
		beanType = obtainApplicationContext().getType(beanName);
	}
	catch (Throwable ex) {
		// An unresolvable bean type, probably from a lazy bean - let's ignore it.
		if (logger.isTraceEnabled()) {
			logger.trace("Could not resolve type for bean '" + beanName + "'", ex);
		}
	}
	// 判断是否为handler即@Controller和@RequestMapping
	if (beanType != null && isHandler(beanType)) {
		// 解析
		detectHandlerMethods(beanName);
	}
}
```

##### 4.1.1.1isHandler
```java
protected boolean isHandler(Class<?> beanType) {
	return (AnnotatedElementUtils.hasAnnotation(beanType, Controller.class) ||
			AnnotatedElementUtils.hasAnnotation(beanType, RequestMapping.class));
}
```
##### 4.1.1.2detectHandlerMethods
```java
protected void detectHandlerMethods(Object handler) {
		Class<?> handlerType = (handler instanceof String ?
				obtainApplicationContext().getType((String) handler) : handler.getClass());

	if (handlerType != null) {
		Class<?> userType = ClassUtils.getUserClass(handlerType);
		// 循环所有方法过滤
		Map<Method, T> methods = MethodIntrospector.selectMethods(userType,
				(MethodIntrospector.MetadataLookup<T>) method -> {
					try {
						return getMappingForMethod(method, userType);
					}
					catch (Throwable ex) {
						throw new IllegalStateException("Invalid mapping on handler class [" +
								userType.getName() + "]: " + method, ex);
					}
				});
		if (logger.isTraceEnabled()) {
			logger.trace(formatMappings(userType, methods));
		}
		// 再次循环，解析
		methods.forEach((method, mapping) -> {
			Method invocableMethod = AopUtils.selectInvocableMethod(method, userType);
			registerHandlerMethod(handler, invocableMethod, mapping);
		});
	}
}
```

###### 4.1.1.2.1detectHandlerMethods
```java
protected RequestMappingInfo getMappingForMethod(Method method, Class<?> handlerType) {
	// 获取@RequestMapping并封装成对象，内部有多种condition
	/*
	method:get\set
	parmas:参数
	header:头部
	consumes:提交的contentType。json\text
	prodeces:返回的contentType
	patthern:@requestmapping填写的值
	*/
	RequestMappingInfo info = createRequestMappingInfo(method);
	if (info != null) {
		RequestMappingInfo typeInfo = createRequestMappingInfo(handlerType);
		if (typeInfo != null) {
			info = typeInfo.combine(info);
		}
		String prefix = getPathPrefix(handlerType);
		if (prefix != null) {
			info = RequestMappingInfo.paths(prefix).options(this.config).build().combine(info);
		}
	}
	return info;
}
```
###### 4.1.1.2.2registerHandlerMethod
```java
protected void registerHandlerMethod(Object handler, Method method, T mapping) {
	this.mappingRegistry.register(mapping, handler, method);
}
```
```java
public void register(T mapping, Object handler, Method method) {
		// Assert that the handler method is not a suspending one.
		if (KotlinDetector.isKotlinType(method.getDeclaringClass())) {
			Class<?>[] parameterTypes = method.getParameterTypes();
			if ((parameterTypes.length > 0) && "kotlin.coroutines.Continuation".equals(parameterTypes[parameterTypes.length - 1].getName())) {
				throw new IllegalStateException("Unsupported suspending handler method detected: " + method);
			}
		}
		this.readWriteLock.writeLock().lock();
		try {
			HandlerMethod handlerMethod = createHandlerMethod(handler, method);
			validateMethodMapping(handlerMethod, mapping);
			// key是@requestMapping封装对象，v是handlerMethod对象用于反射调用
			this.mappingLookup.put(mapping, handlerMethod);

			List<String> directUrls = getDirectUrls(mapping);
			// k是原始url直接匹配，v是@requestMapping封装对象
			for (String url : directUrls) {
				this.urlLookup.add(url, mapping);
			}

			String name = null;
			if (getNamingStrategy() != null) {
				name = getNamingStrategy().getName(handlerMethod, mapping);
				addMappingName(name, handlerMethod);
			}

			CorsConfiguration corsConfig = initCorsConfiguration(handler, method, mapping);
			if (corsConfig != null) {
				this.corsLookup.put(handlerMethod, corsConfig);
			}
			// key是@requestMapping封装对象，v是上述其它对象封装
			this.registry.put(mapping, new MappingRegistration<>(mapping, handlerMethod, directUrls, name));
		}
		finally {
			this.readWriteLock.writeLock().unlock();
		}
	}
```
