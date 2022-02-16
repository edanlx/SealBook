# 【框架类源码】07-spring启动整体流程
> 欢迎关注b站账号/公众号【六边形战士夏宁】，一个要把各项指标拉满的男人。该文章已在[github目录](https://github.com/edanlx/SealBook/blob/master/catalogue/wechat.md)收录。
屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。

## 1.背景
Advisor
	pointcut
		classFilter
		MethodMathcer
	Advice
```java
public class FactoryExample {
    public static void main(String[] args) {
        FactoryUser target = new FactoryUser();
        ProxyFactory proxyFactory = new ProxyFactory();
        proxyFactory.setTarget(target);
        // 使用如下代码会从cglib动态代理改为jdk动态代理
        // proxyFactory.setInterfaces();
        proxyFactory.addAdvice(new MethodInterceptor() {
            @Override
            public Object invoke(MethodInvocation invocation) throws Throwable {
                System.out.println("before...");
                Object result = invocation.proceed();
                System.out.println("after...");
                return result;
            }
        });
        FactoryUser userService = (FactoryUser) proxyFactory.getProxy();
        userService.test();
    }
}
```
## 2.setTarget
会将target包装为targetSource，后续过程中调用getTarget获取被代理对象，以proxyFactory为例
```java
public void setTarget(Object target) {
	setTargetSource(new SingletonTargetSource(target));
}
```
```java
public void setTargetSource(@Nullable TargetSource targetSource) {
	this.targetSource = (targetSource != null ? targetSource : EMPTY_TARGET_SOURCE);
}
```
## 3.getProxy
```java
public Object getProxy() {
	return createAopProxy().getProxy();
}
```
### 3.1createAopProxy
判断使用哪种方式进行代理
```java
protected final synchronized AopProxy createAopProxy() {
	if (!this.active) {
		activate();
	}
	return getAopProxyFactory().createAopProxy(this);
}
```

```java
public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
	// 此处根据设置代理方式以及是否有接口进行判断，分别为是否优化(优化进入cglibe)，是否设置proxyTargetClass(true进入cglib)，是否设置ProxyInterface(没添加进入cglib)
	if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
		Class<?> targetClass = config.getTargetClass();
		if (targetClass == null) {
			throw new AopConfigException("TargetSource cannot determine target class: " +
					"Either an interface or a target is required for proxy creation.");
		}
		// 被代理的类如果是接口则使用jdk，是否是jdk代理类。一般不使用该分支逻辑
		if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
			return new JdkDynamicAopProxy(config);
		}
		return new ObjenesisCglibAopProxy(config);
	}
	else {
		return new JdkDynamicAopProxy(config);
	}
}
```

### 3.2JDKgetProxy
JDK动态代理标准写法
```java
public Object getProxy() {
	return getProxy(ClassUtils.getDefaultClassLoader());
}
```
```java
public Object getProxy(@Nullable ClassLoader classLoader) {
	if (logger.isTraceEnabled()) {
		logger.trace("Creating JDK dynamic proxy: " + this.advised.getTargetSource());
	}
	Class<?>[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised, true);
	findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
	return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
}
```
### 3.3JDKinvoke
JDK代理，如果被执行则调用invoke方法
```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
	Object oldProxy = null;
	boolean setProxyContext = false;

	TargetSource targetSource = this.advised.targetSource;
	Object target = null;

	try {
		// equals、hashcode等直接返回，不代理
		if (!this.equalsDefined && AopUtils.isEqualsMethod(method)) {
			// The target does not implement the equals(Object) method itself.
			return equals(args[0]);
		}
		else if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)) {
			// The target does not implement the hashCode() method itself.
			return hashCode();
		}
		else if (method.getDeclaringClass() == DecoratingProxy.class) {
			// There is only getDecoratedClass() declared -> dispatch to proxy config.
			// 得到代理类型的类型，而不是其原实现接口
			return AopProxyUtils.ultimateTargetClass(this.advised);
		}
		else if (!this.advised.opaque && method.getDeclaringClass().isInterface() &&
				method.getDeclaringClass().isAssignableFrom(Advised.class)) {
			// Service invocations on ProxyConfig with the proxy config...
			// 
			return AopUtils.invokeJoinpointUsingReflection(this.advised, method, args);
		}

		Object retVal;

		if (this.advised.exposeProxy) {
			// Make invocation available if necessary.
			// 该代理对象为ThreadLocale(AopContext.currentProxy())
			oldProxy = AopContext.setCurrentProxy(proxy);
			setProxyContext = true;
		}

		// Get as late as possible to minimize the time we "own" the target,
		// in case it comes from a pool.
		target = targetSource.getTarget();
		Class<?> targetClass = (target != null ? target.getClass() : null);

		// Get the interception chain for this method.
		// 根据当前方法去匹配advise，返回advise链
		List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

		// Check whether we have any advice. If we don't, we can fallback on direct
		// reflective invocation of the target, and avoid creating a MethodInvocation.
		if (chain.isEmpty()) {
			// We can skip creating a MethodInvocation: just invoke the target directly
			// Note that the final invoker must be an InvokerInterceptor so we know it does
			// nothing but a reflective operation on the target, and no hot swapping or fancy proxying.
			// 直接执行方法
			Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
			retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
		}
		else {
			// We need to create a method invocation...
			// 执行advise链
			MethodInvocation invocation =
					new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
			// Proceed to the joinpoint through the interceptor chain.
			retVal = invocation.proceed();
		}

		// Massage return value if necessary.
		Class<?> returnType = method.getReturnType();
		if (retVal != null && retVal == target &&
				returnType != Object.class && returnType.isInstance(proxy) &&
				!RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {
			// Special case: it returned "this" and the return type of the method
			// is type-compatible. Note that we can't help if the target sets
			// a reference to itself in another returned object.
			retVal = proxy;
		}
		else if (retVal == null && returnType != Void.TYPE && returnType.isPrimitive()) {
			throw new AopInvocationException(
					"Null return value from advice does not match primitive return type for: " + method);
		}
		return retVal;
	}
	finally {
		if (target != null && !targetSource.isStatic()) {
			// Must have come from TargetSource.
			targetSource.releaseTarget(target);
		}
		if (setProxyContext) {
			// Restore old proxy.
			AopContext.setCurrentProxy(oldProxy);
		}
	}
}
```

#### 3.3.1getInterceptorsAndDynamicInterceptionAdvice
```java
public List<Object> getInterceptorsAndDynamicInterceptionAdvice(Method method, @Nullable Class<?> targetClass) {
	// 获取缓存
	MethodCacheKey cacheKey = new MethodCacheKey(method);
	List<Object> cached = this.methodCache.get(cacheKey);
	if (cached == null) {
		cached = this.advisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice(
				this, method, targetClass);
		this.methodCache.put(cacheKey, cached);
	}
	return cached;
}
```
```java
public List<Object> getInterceptorsAndDynamicInterceptionAdvice(
		Advised config, Method method, @Nullable Class<?> targetClass) {

	// This is somewhat tricky... We have to process introductions first,
	// but we need to preserve order in the ultimate list.
	AdvisorAdapterRegistry registry = GlobalAdvisorAdapterRegistry.getInstance();
	// 获取advisor(advice会转为advisor)
	Advisor[] advisors = config.getAdvisors();
	List<Object> interceptorList = new ArrayList<>(advisors.length);
	Class<?> actualClass = (targetClass != null ? targetClass : method.getDeclaringClass());
	Boolean hasIntroductions = null;

	for (Advisor advisor : advisors) {
		if (advisor instanceof PointcutAdvisor) {
			// Add it conditionally.
			PointcutAdvisor pointcutAdvisor = (PointcutAdvisor) advisor;
			// 匹配类
			if (config.isPreFiltered() || pointcutAdvisor.getPointcut().getClassFilter().matches(actualClass)) {
				MethodMatcher mm = pointcutAdvisor.getPointcut().getMethodMatcher();
				boolean match;
				// 匹配方法进行过滤，此处调用两参的匹配逻辑((无args进行判断))
				if (mm instanceof IntroductionAwareMethodMatcher) {
					if (hasIntroductions == null) {
						hasIntroductions = hasMatchingIntroductions(advisors, actualClass);
					}
					match = ((IntroductionAwareMethodMatcher) mm).matches(method, actualClass, hasIntroductions);
				}
				else {
					match = mm.matches(method, actualClass);
				}
				// 如果匹配成功，将其转为MethodInterceptor
				if (match) {
					MethodInterceptor[] interceptors = registry.getInterceptors(advisor);
					// 如果isRunTime为True，则会包装为InterceptorAndDynamicMethodMatcher
					if (mm.isRuntime()) {
						// Creating a new object instance in the getInterceptors() method
						// isn't a problem as we normally cache created chains.
						for (MethodInterceptor interceptor : interceptors) {
							interceptorList.add(new InterceptorAndDynamicMethodMatcher(interceptor, mm));
						}
					}
					else {
						interceptorList.addAll(Arrays.asList(interceptors));
					}
				}
			}
		}
		else if (advisor instanceof IntroductionAdvisor) {
			IntroductionAdvisor ia = (IntroductionAdvisor) advisor;
			if (config.isPreFiltered() || ia.getClassFilter().matches(actualClass)) {
				Interceptor[] interceptors = registry.getInterceptors(advisor);
				interceptorList.addAll(Arrays.asList(interceptors));
			}
		}
		else {
			Interceptor[] interceptors = registry.getInterceptors(advisor);
			interceptorList.addAll(Arrays.asList(interceptors));
		}
	}

	return interceptorList;
}
```
##### 3.3.1.1getInterceptors
```java
public MethodInterceptor[] getInterceptors(Advisor advisor) throws UnknownAdviceTypeException {
	List<MethodInterceptor> interceptors = new ArrayList<>(3);
	Advice advice = advisor.getAdvice();
	// 如果是MethodInterceptor则直接加入
	if (advice instanceof MethodInterceptor) {
		interceptors.add((MethodInterceptor) advice);
	}
	/*
	其内部的invoke方法会执行proceed()
	registerAdvisorAdapter(new MethodBeforeAdviceAdapter());->new MethodBeforeAdviceInterceptor(advice)
	registerAdvisorAdapter(new AfterReturningAdviceAdapter());->new AfterReturningAdviceInterceptor(advice)
	registerAdvisorAdapter(new ThrowsAdviceAdapter());->new ThrowsAdviceInterceptor(advisor.getAdvice())->其内部会获取实现方法的最后一个参数即异常类型并存起来
	*/
	// 三种适配器进行适配，将其转为Interceptor
	for (AdvisorAdapter adapter : this.adapters) {
		if (adapter.supportsAdvice(advice)) {
			interceptors.add(adapter.getInterceptor(advisor));
		}
	}
	if (interceptors.isEmpty()) {
		throw new UnknownAdviceTypeException(advisor.getAdvice());
	}
	return interceptors.toArray(new MethodInterceptor[0]);
}
```
##### 3.3.2JDK，invocation.proceed();
```java
public Object proceed() throws Throwable {
	// We start with an index of -1 and increment early.
	if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
		// 指针一直移动到最后一个链后，执行被代理类的方法
		return invokeJoinpoint();
	}

	Object interceptorOrInterceptionAdvice =
			this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
	if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
		// Evaluate dynamic method matcher here: static part will already have
		// been evaluated and found to match.
		InterceptorAndDynamicMethodMatcher dm =
				(InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
		Class<?> targetClass = (this.targetClass != null ? this.targetClass : this.method.getDeclaringClass());
		if (dm.methodMatcher.matches(this.method, targetClass, this.arguments)) {
			// 如果是InterceptorAndDynamicMethodMatcher且符合matches三参的判断(args进行判断)
			return dm.interceptor.invoke(this);
		}
		else {
			// Dynamic matching failed.
			// Skip this interceptor and invoke the next in the chain.
			return proceed();
		}
	}
	else {
		// It's an interceptor, so we just invoke it: The pointcut will have
		// been evaluated statically before this object was constructed.
		// 此处进行递归，因为所有的Interceptor都会调用proceed()方法，责任链模式
		return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
	}
}
```
### 3.4cglibgetProxy
```java
public Object getProxy() {
	return getProxy(null);
}
```

```java
public Object getProxy(@Nullable ClassLoader classLoader) {
	if (logger.isTraceEnabled()) {
		logger.trace("Creating CGLIB proxy: " + this.advised.getTargetSource());
	}

	try {
		Class<?> rootClass = this.advised.getTargetClass();
		Assert.state(rootClass != null, "Target class must be available for creating a CGLIB proxy");

		Class<?> proxySuperClass = rootClass;
		if (rootClass.getName().contains(ClassUtils.CGLIB_CLASS_SEPARATOR)) {
			proxySuperClass = rootClass.getSuperclass();
			Class<?>[] additionalInterfaces = rootClass.getInterfaces();
			for (Class<?> additionalInterface : additionalInterfaces) {
				this.advised.addInterface(additionalInterface);
			}
		}

		// Validate the class, writing log messages as necessary.
		validateClassIfNecessary(proxySuperClass, classLoader);

		// Configure CGLIB Enhancer...
		Enhancer enhancer = createEnhancer();
		if (classLoader != null) {
			enhancer.setClassLoader(classLoader);
			if (classLoader instanceof SmartClassLoader &&
					((SmartClassLoader) classLoader).isClassReloadable(proxySuperClass)) {
				enhancer.setUseCache(false);
			}
		}
		enhancer.setSuperclass(proxySuperClass);
		enhancer.setInterfaces(AopProxyUtils.completeProxiedInterfaces(this.advised));
		enhancer.setNamingPolicy(SpringNamingPolicy.INSTANCE);
		enhancer.setStrategy(new ClassLoaderAwareGeneratorStrategy(classLoader));

		// 核心逻辑获取过滤后的callback，注意cglib的执行就是直接调用callback(JDK是invoke)，调用时逻辑则与jdk一致
		Callback[] callbacks = getCallbacks(rootClass);
		Class<?>[] types = new Class<?>[callbacks.length];
		for (int x = 0; x < types.length; x++) {
			types[x] = callbacks[x].getClass();
		}
		// fixedInterceptorMap only populated at this point, after getCallbacks call above
		enhancer.setCallbackFilter(new ProxyCallbackFilter(
				this.advised.getConfigurationOnlyCopy(), this.fixedInterceptorMap, this.fixedInterceptorOffset));
		enhancer.setCallbackTypes(types);

		// Generate the proxy class and create a proxy instance.
		return createProxyClassAndInstance(enhancer, callbacks);
	}
	catch (CodeGenerationException | IllegalArgumentException ex) {
		throw new AopConfigException("Could not generate CGLIB subclass of " + this.advised.getTargetClass() +
				": Common causes of this problem include using a final class or a non-visible class",
				ex);
	}
	catch (Throwable ex) {
		// TargetSource.getTarget() failed
		throw new AopConfigException("Unexpected AOP exception", ex);
	}
}
```
### 3.4.1 getCallbacks
```java
private Callback[] getCallbacks(Class<?> rootClass) throws Exception {
	// Parameters used for optimization choices...
	boolean exposeProxy = this.advised.isExposeProxy();
	boolean isFrozen = this.advised.isFrozen();
	boolean isStatic = this.advised.getTargetSource().isStatic();

	// Choose an "aop" interceptor (used for AOP calls).
	Callback aopInterceptor = new DynamicAdvisedInterceptor(this.advised);

	// Choose a "straight to target" interceptor. (used for calls that are
	// unadvised but can return this). May be required to expose the proxy.
	Callback targetInterceptor;
	if (exposeProxy) {
		targetInterceptor = (isStatic ?
				new StaticUnadvisedExposedInterceptor(this.advised.getTargetSource().getTarget()) :
				new DynamicUnadvisedExposedInterceptor(this.advised.getTargetSource()));
	}
	else {
		targetInterceptor = (isStatic ?
				new StaticUnadvisedInterceptor(this.advised.getTargetSource().getTarget()) :
				new DynamicUnadvisedInterceptor(this.advised.getTargetSource()));
	}

	// Choose a "direct to target" dispatcher (used for
	// unadvised calls to static targets that cannot return this).
	Callback targetDispatcher = (isStatic ?
			new StaticDispatcher(this.advised.getTargetSource().getTarget()) : new SerializableNoOp());

	Callback[] mainCallbacks = new Callback[] {
			aopInterceptor,  // for normal advice
			targetInterceptor,  // invoke target without considering advice, if optimized
			new SerializableNoOp(),  // no override for methods mapped to this
			targetDispatcher, this.advisedDispatcher,
			new EqualsInterceptor(this.advised),
			new HashCodeInterceptor(this.advised)
	};

	Callback[] callbacks;

	// If the target is a static one and the advice chain is frozen,
	// then we can make some optimizations by sending the AOP calls
	// direct to the target using the fixed chain for that method.
	if (isStatic && isFrozen) {
		Method[] methods = rootClass.getMethods();
		Callback[] fixedCallbacks = new Callback[methods.length];
		this.fixedInterceptorMap = new HashMap<>(methods.length);

		// TODO: small memory optimization here (can skip creation for methods with no advice)
		for (int x = 0; x < methods.length; x++) {
			Method method = methods[x];
			// 与JDK一样的逻辑
			List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, rootClass);
			fixedCallbacks[x] = new FixedChainStaticTargetInterceptor(
					chain, this.advised.getTargetSource().getTarget(), this.advised.getTargetClass());
			this.fixedInterceptorMap.put(method, x);
		}

		// Now copy both the callbacks from mainCallbacks
		// and fixedCallbacks into the callbacks array.
		callbacks = new Callback[mainCallbacks.length + fixedCallbacks.length];
		System.arraycopy(mainCallbacks, 0, callbacks, 0, mainCallbacks.length);
		System.arraycopy(fixedCallbacks, 0, callbacks, mainCallbacks.length, fixedCallbacks.length);
		this.fixedInterceptorOffset = mainCallbacks.length;
	}
	else {
		callbacks = mainCallbacks;
	}
	return callbacks;
}

```

## 4.@EnableAspectJAutoProxy
```java
@Import(AspectJAutoProxyRegistrar.class)
```

```java
public void registerBeanDefinitions(
			AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
	// 注册beandefinition
	AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry);

	AnnotationAttributes enableAspectJAutoProxy =
			AnnotationConfigUtils.attributesFor(importingClassMetadata, EnableAspectJAutoProxy.class);
	if (enableAspectJAutoProxy != null) {
		// 将注解上的值设置到beandefinition
		if (enableAspectJAutoProxy.getBoolean("proxyTargetClass")) {
			AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
		}
		if (enableAspectJAutoProxy.getBoolean("exposeProxy")) {
			AopConfigUtils.forceAutoProxyCreatorToExposeProxy(registry);
		}
	}
}
```

```java
public static BeanDefinition registerAspectJAnnotationAutoProxyCreatorIfNecessary(BeanDefinitionRegistry registry) {
		return registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry, null);
	}
```

```java
@Nullable
public static BeanDefinition registerAspectJAnnotationAutoProxyCreatorIfNecessary(
		BeanDefinitionRegistry registry, @Nullable Object source) {
	// 注册AnnotationAwareAspectJAutoProxyCreator，该类有多个扩展点，关键为postProcessAfterInitialization
	return registerOrEscalateApcAsRequired(AnnotationAwareAspectJAutoProxyCreator.class, registry, source);
}
```

### 4.1AbstractAutoProxyCreator.postProcessAfterInitialization
```java
public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
		if (bean != null) {
			Object cacheKey = getCacheKey(bean.getClass(), beanName);
			if (this.earlyProxyReferences.remove(cacheKey) != bean) {
				// 核心方法
				return wrapIfNecessary(bean, beanName, cacheKey);
			}
		}
		return bean;
	}
```

```java
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
	if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
		return bean;
	}
	// 缓存
	if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
		return bean;
	}
	// 当前bean如果是aop相关的类不生成代理
	if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
		this.advisedBeans.put(cacheKey, Boolean.FALSE);
		return bean;
	}

	// Create proxy if we have advice.
	// 判断是否需要进行aop
	Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
	if (specificInterceptors != DO_NOT_PROXY) {
		this.advisedBeans.put(cacheKey, Boolean.TRUE);
		// 创建proxy
		Object proxy = createProxy(
				bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
		this.proxyTypes.put(cacheKey, proxy.getClass());
		return proxy;
	}

	this.advisedBeans.put(cacheKey, Boolean.FALSE);
	return bean;
}
```

### 4.2getAdvicesAndAdvisorsForBean
```java
protected Object[] getAdvicesAndAdvisorsForBean(
			Class<?> beanClass, String beanName, @Nullable TargetSource targetSource) {

	List<Advisor> advisors = findEligibleAdvisors(beanClass, beanName);
	if (advisors.isEmpty()) {
		return DO_NOT_PROXY;
	}
	return advisors.toArray();
}
```

```java
protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName) {
	// 获取所有advise
	List<Advisor> candidateAdvisors = findCandidateAdvisors();
	// 筛选
	List<Advisor> eligibleAdvisors = findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);
	extendAdvisors(eligibleAdvisors);
	if (!eligibleAdvisors.isEmpty()) {
		// 排序
		eligibleAdvisors = sortAdvisors(eligibleAdvisors);
	}
	return eligibleAdvisors;
}
```

#### 4.2.1findCandidateAdvisors
```java
protected List<Advisor> findCandidateAdvisors() {
	// Add all the Spring advisors found according to superclass rules.
	List<Advisor> advisors = super.findCandidateAdvisors();
	// Build Advisors for all AspectJ aspects in the bean factory.
	if (this.aspectJAdvisorsBuilder != null) {
		// 解析注解@Around、@Before、@After、@AfterReturning、@AfterThrowing将其形成InstantiationModelAwarePointcutAdvisorImpl(buildAspectJAdvisors->advisorFactory.getAdvisors->getAdvisor)
		// 调用时InstantiationModelAwarePointcutAdvisorImpl.getAdvice()->instantiateAdvice()->getAdvice该方法内部将method对应成advise
		advisors.addAll(this.aspectJAdvisorsBuilder.buildAspectJAdvisors());
	}
	return advisors;
}
```

```java
protected List<Advisor> findCandidateAdvisors() {
	Assert.state(this.advisorRetrievalHelper != null, "No BeanFactoryAdvisorRetrievalHelper available");
	return this.advisorRetrievalHelper.findAdvisorBeans();
}
```

```java
public List<Advisor> findAdvisorBeans() {
	// Determine list of advisor bean names, if not cached already.
	String[] advisorNames = this.cachedAdvisorBeanNames;
	if (advisorNames == null) {
		// Do not initialize FactoryBeans here: We need to leave all regular beans
		// uninitialized to let the auto-proxy creator apply to them!
		// 从bean工厂中获取所有Advisor类型的bean
		advisorNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
				this.beanFactory, Advisor.class, true, false);
		this.cachedAdvisorBeanNames = advisorNames;
	}
	if (advisorNames.length == 0) {
		return new ArrayList<>();
	}

	List<Advisor> advisors = new ArrayList<>();
	for (String name : advisorNames) {
		if (isEligibleBean(name)) {
			if (this.beanFactory.isCurrentlyInCreation(name)) {
				if (logger.isTraceEnabled()) {
					logger.trace("Skipping currently created advisor '" + name + "'");
				}
			}
			else {
				try {
					advisors.add(this.beanFactory.getBean(name, Advisor.class));
				}
				catch (BeanCreationException ex) {
					Throwable rootCause = ex.getMostSpecificCause();
					if (rootCause instanceof BeanCurrentlyInCreationException) {
						BeanCreationException bce = (BeanCreationException) rootCause;
						String bceBeanName = bce.getBeanName();
						if (bceBeanName != null && this.beanFactory.isCurrentlyInCreation(bceBeanName)) {
							if (logger.isTraceEnabled()) {
								logger.trace("Skipping advisor '" + name +
										"' with dependency on currently created bean: " + ex.getMessage());
							}
							// Ignore: indicates a reference back to the bean we're trying to advise.
							// We want to find advisors other than the currently created bean itself.
							continue;
						}
					}
					throw ex;
				}
			}
		}
	}
	return advisors;
}
```

### 4.3createProxy
```java
protected Object createProxy(Class<?> beanClass, @Nullable String beanName,
			@Nullable Object[] specificInterceptors, TargetSource targetSource) {

	if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
		AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
	}
	// 此处生成proxyFactory
	ProxyFactory proxyFactory = new ProxyFactory();
	proxyFactory.copyFrom(this);

	if (!proxyFactory.isProxyTargetClass()) {
		if (shouldProxyTargetClass(beanClass, beanName)) {
			proxyFactory.setProxyTargetClass(true);
		}
		else {
			evaluateProxyInterfaces(beanClass, proxyFactory);
		}
	}
	// 此处构造advisor
	Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
	proxyFactory.addAdvisors(advisors);
	proxyFactory.setTargetSource(targetSource);
	customizeProxyFactory(proxyFactory);

	proxyFactory.setFrozen(this.freezeProxy);
	if (advisorsPreFiltered()) {
		proxyFactory.setPreFiltered(true);
	}

	return proxyFactory.getProxy(getProxyClassLoader());
}
```

## 5.getEarlyBeanReference
在二级缓存中doCreateBean方法中有addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
```java
protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {
	Object exposedObject = bean;
	if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
		for (BeanPostProcessor bp : getBeanPostProcessors()) {
			if (bp instanceof SmartInstantiationAwareBeanPostProcessor) {
				SmartInstantiationAwareBeanPostProcessor ibp = (SmartInstantiationAwareBeanPostProcessor) bp;
				exposedObject = ibp.getEarlyBeanReference(exposedObject, beanName);
			}
		}
	}
	return exposedObject;
}
```

```java
public Object getEarlyBeanReference(Object bean, String beanName) {
	Object cacheKey = getCacheKey(bean.getClass(), beanName);
	this.earlyProxyReferences.put(cacheKey, bean);
	// 核心方法，与上述一致
	return wrapIfNecessary(bean, beanName, cacheKey);
}
```