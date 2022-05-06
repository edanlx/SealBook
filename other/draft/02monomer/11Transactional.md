# 【框架类源码】07-spring启动整体流程
> 欢迎关注b站账号/公众号【六边形战士夏宁】，一个要把各项指标拉满的男人。该文章已在[github目录](https://github.com/edanlx/SealBook/blob/master/catalogue/wechat.md)收录。
屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。

## 1.背景

## 3.@EnableTransactionManagement
TransactionManagementConfigurationSelector,其实现ImportSelector，即会被调用selectImports
```java
protected String[] selectImports(AdviceMode adviceMode) {
	switch (adviceMode) {
		case PROXY:
			// 默认
			// 导入InfrastructureAdvisorAutoProxyCreator，即开启aop(但该类不解析aspectj注解)
			return new String[] {AutoProxyRegistrar.class.getName(),
			// 注册三个bean
			// BeanFactoryTransactionAttributeSourceAdvisor即核心advisor
			// TransactionAttributeSource用于解析@Transactional注解
			// TransactionInterceptor 事务拦截器
					ProxyTransactionManagementConfiguration.class.getName()};
		case ASPECTJ:
			return new String[] {determineTransactionAspectClass()};
		default:
			return null;
	}
}
```
## 3.1AutoProxyRegistrar.registerBeanDefinitions
```java
public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
	boolean candidateFound = false;
	Set<String> annTypes = importingClassMetadata.getAnnotationTypes();
	for (String annType : annTypes) {
		AnnotationAttributes candidate = AnnotationConfigUtils.attributesFor(importingClassMetadata, annType);
		if (candidate == null) {
			continue;
		}
		Object mode = candidate.get("mode");
		Object proxyTargetClass = candidate.get("proxyTargetClass");
		if (mode != null && proxyTargetClass != null && AdviceMode.class == mode.getClass() &&
				Boolean.class == proxyTargetClass.getClass()) {
			candidateFound = true;
			if (mode == AdviceMode.PROXY) {
				// 核心注册方法
				AopConfigUtils.registerAutoProxyCreatorIfNecessary(registry);
				if ((Boolean) proxyTargetClass) {
					AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
					return;
				}
			}
		}
	}
	if (!candidateFound && logger.isInfoEnabled()) {
		String name = getClass().getSimpleName();
		logger.info(String.format("%s was imported but no annotations were found " +
				"having both 'mode' and 'proxyTargetClass' attributes of type " +
				"AdviceMode and boolean respectively. This means that auto proxy " +
				"creator registration and configuration may not have occurred as " +
				"intended, and components may not be proxied as expected. Check to " +
				"ensure that %s has been @Import'ed on the same class where these " +
				"annotations are declared; otherwise remove the import of %s " +
				"altogether.", name, name, name));
	}
}
```

```java
public static BeanDefinition registerAutoProxyCreatorIfNecessary(BeanDefinitionRegistry registry) {
	return registerAutoProxyCreatorIfNecessary(registry, null);
}
```
```java
public static BeanDefinition registerAutoProxyCreatorIfNecessary(
		BeanDefinitionRegistry registry, @Nullable Object source) {
	// 注册核心类InfrastructureAdvisorAutoProxyCreator，其继承AbstractAutoProxyCreator
	return registerOrEscalateApcAsRequired(InfrastructureAdvisorAutoProxyCreator.class, registry, source);
}
```

## 4.BeanFactoryTransactionAttributeSourceAdvisor
```java
private final TransactionAttributeSourcePointcut pointcut = new TransactionAttributeSourcePointcut()
```
```java
protected TransactionAttributeSourcePointcut() {
	setClassFilter(new TransactionAttributeSourceClassFilter());
}

@Override
public boolean matches(Method method, Class<?> targetClass) {
	TransactionAttributeSource tas = getTransactionAttributeSource();
	return (tas == null || tas.getTransactionAttribute(method, targetClass) != null);
}
```
### 4.1TransactionAttributeSourceClassFilter
```java
private class TransactionAttributeSourceClassFilter implements ClassFilter {

	@Override
	public boolean matches(Class<?> clazz) {
		// 判断是否是事务相关类，如果是则直接返回false
		if (TransactionalProxy.class.isAssignableFrom(clazz) ||
				PlatformTransactionManager.class.isAssignableFrom(clazz) ||
				PersistenceExceptionTranslator.class.isAssignableFrom(clazz)) {
			return false;
		}
		// 获得前面注入的TransactionAttributeSource
		TransactionAttributeSource tas = getTransactionAttributeSource();
		// 判断该类上是否有注解@Transactional
		return (tas == null || tas.isCandidateClass(clazz));
	}
}
```

### 4.1.1AnnotationTransactionAttributeSource.isCandidateClass
```java
public boolean isCandidateClass(Class<?> targetClass) {
	for (TransactionAnnotationParser parser : this.annotationParsers) {
		if (parser.isCandidateClass(targetClass)) {
			return true;
		}
	}
	return false;
}
```

```java
public boolean isCandidateClass(Class<?> targetClass) {
	return AnnotationUtils.isCandidateClass(targetClass, Transactional.class);
}
```

### 4.2TransactionAttributeSourcePointcut.matches
```java
public boolean matches(Method method, Class<?> targetClass) {
	TransactionAttributeSource tas = getTransactionAttributeSource();
	return (tas == null || tas.getTransactionAttribute(method, targetClass) != null);
}
```

### 4.2.1AbstractFallbackTransactionAttributeSource.getTransactionAttribute
```java
public TransactionAttribute getTransactionAttribute(Method method, @Nullable Class<?> targetClass) {
	if (method.getDeclaringClass() == Object.class) {
		return null;
	}

	// First, see if we have a cached value.
	Object cacheKey = getCacheKey(method, targetClass);
	TransactionAttribute cached = this.attributeCache.get(cacheKey);
	// 获取缓存
	if (cached != null) {
		// Value will either be canonical value indicating there is no transaction attribute,
		// or an actual transaction attribute.
		if (cached == NULL_TRANSACTION_ATTRIBUTE) {
			return null;
		}
		else {
			return cached;
		}
	}
	else {
		// We need to work it out.
		// 解析@Transactional注解->RuleBasedTransactionAttribute对象
		TransactionAttribute txAttr = computeTransactionAttribute(method, targetClass);
		// Put it in the cache.
		if (txAttr == null) {
			this.attributeCache.put(cacheKey, NULL_TRANSACTION_ATTRIBUTE);
		}
		else {
			String methodIdentification = ClassUtils.getQualifiedMethodName(method, targetClass);
			if (txAttr instanceof DefaultTransactionAttribute) {
				((DefaultTransactionAttribute) txAttr).setDescriptor(methodIdentification);
			}
			if (logger.isTraceEnabled()) {
				logger.trace("Adding transactional method '" + methodIdentification + "' with attribute: " + txAttr);
			}
			this.attributeCache.put(cacheKey, txAttr);
		}
		return txAttr;
	}
}
```

### 4.2.2parseTransactionAnnotation
computeTransactionAttribute->determineTransactionAttribute->SpringTransactionAnnotationParser.parseTransactionAnnotation
```java
public TransactionAttribute parseTransactionAnnotation(AnnotatedElement element) {
	// 使用spring工具类获取相关信息
	AnnotationAttributes attributes = AnnotatedElementUtils.findMergedAnnotationAttributes(
			element, Transactional.class, false, false);
	if (attributes != null) {
		return parseTransactionAnnotation(attributes);
	}
	else {
		return null;
	}
}
```

### 4.2.3parseTransactionAnnotation
将注解相关值封装到RuleBasedTransactionAttribute对象
```java
protected TransactionAttribute parseTransactionAnnotation(AnnotationAttributes attributes) {
	RuleBasedTransactionAttribute rbta = new RuleBasedTransactionAttribute();

	Propagation propagation = attributes.getEnum("propagation");
	rbta.setPropagationBehavior(propagation.value());
	Isolation isolation = attributes.getEnum("isolation");
	rbta.setIsolationLevel(isolation.value());
	rbta.setTimeout(attributes.getNumber("timeout").intValue());
	rbta.setReadOnly(attributes.getBoolean("readOnly"));
	rbta.setQualifier(attributes.getString("value"));

	List<RollbackRuleAttribute> rollbackRules = new ArrayList<>();
	for (Class<?> rbRule : attributes.getClassArray("rollbackFor")) {
		rollbackRules.add(new RollbackRuleAttribute(rbRule));
	}
	for (String rbRule : attributes.getStringArray("rollbackForClassName")) {
		rollbackRules.add(new RollbackRuleAttribute(rbRule));
	}
	for (Class<?> rbRule : attributes.getClassArray("noRollbackFor")) {
		rollbackRules.add(new NoRollbackRuleAttribute(rbRule));
	}
	for (String rbRule : attributes.getStringArray("noRollbackForClassName")) {
		rollbackRules.add(new NoRollbackRuleAttribute(rbRule));
	}
	rbta.setRollbackRules(rollbackRules);

	return rbta;
}
```

## 5.TransactionInterceptor.invoke
```java
public Object invoke(MethodInvocation invocation) throws Throwable {
	// Work out the target class: may be {@code null}.
	// The TransactionAttributeSource should be passed the target class
	// as well as the method, which may be from an interface.
	Class<?> targetClass = (invocation.getThis() != null ? AopUtils.getTargetClass(invocation.getThis()) : null);

	// Adapt to TransactionAspectSupport's invokeWithinTransaction...
	// invocation::proceed递归回调
	return invokeWithinTransaction(invocation.getMethod(), targetClass, invocation::proceed);
}
```
延续调用invokeWithinTransaction
```java
protected Object invokeWithinTransaction(Method method, @Nullable Class<?> targetClass,
			final InvocationCallback invocation) throws Throwable {

	// If the transaction attribute is null, the method is non-transactional.
	TransactionAttributeSource tas = getTransactionAttributeSource();
	// 获取注解相关信息
	final TransactionAttribute txAttr = (tas != null ? tas.getTransactionAttribute(method, targetClass) : null);
	// 获取事务管理器
	final TransactionManager tm = determineTransactionManager(txAttr);

	if (this.reactiveAdapterRegistry != null && tm instanceof ReactiveTransactionManager) {
		// 分支逻辑在@Transactional中配置一般不走
		ReactiveTransactionSupport txSupport = this.transactionSupportCache.computeIfAbsent(method, key -> {
			if (KotlinDetector.isKotlinType(method.getDeclaringClass()) && KotlinDelegate.isSuspend(method)) {
				throw new TransactionUsageException(
						"Unsupported annotated transaction on suspending function detected: " + method +
						". Use TransactionalOperator.transactional extensions instead.");
			}
			ReactiveAdapter adapter = this.reactiveAdapterRegistry.getAdapter(method.getReturnType());
			if (adapter == null) {
				throw new IllegalStateException("Cannot apply reactive transaction to non-reactive return type: " +
						method.getReturnType());
			}
			return new ReactiveTransactionSupport(adapter);
		});
		return txSupport.invokeWithinTransaction(
				method, targetClass, invocation, txAttr, (ReactiveTransactionManager) tm);
	}

	// 将事务管理器强转为PlatformTransactionManager
	PlatformTransactionManager ptm = asPlatformTransactionManager(tm);
	// 获取jionpoint标识名称
	final String joinpointIdentification = methodIdentification(method, targetClass, txAttr);

	// 正常是走该逻辑
	if (txAttr == null || !(ptm instanceof CallbackPreferringPlatformTransactionManager)) {
		// Standard transaction demarcation with getTransaction and commit/rollback calls.
		// 处理事务传播机制
		TransactionInfo txInfo = createTransactionIfNecessary(ptm, txAttr, joinpointIdentification);

		Object retVal;
		try {
			// This is an around advice: Invoke the next interceptor in the chain.
			// This will normally result in a target object being invoked.
			// 执行process递归回调即下一个Intercept
			retVal = invocation.proceedWithInvocation();
		}
		catch (Throwable ex) {
			// target invocation exception
			// 回滚事务
			completeTransactionAfterThrowing(txInfo, ex);
			throw ex;
		}
		finally {
			cleanupTransactionInfo(txInfo);
		}

		if (vavrPresent && VavrDelegate.isVavrTry(retVal)) {
			// Set rollback-only in case of Vavr failure matching our rollback rules...
			TransactionStatus status = txInfo.getTransactionStatus();
			if (status != null && txAttr != null) {
				retVal = VavrDelegate.evaluateTryFailure(retVal, txAttr, status);
			}
		}
		// 执行提交
		commitTransactionAfterReturning(txInfo);
		return retVal;
	}

	else {
		final ThrowableHolder throwableHolder = new ThrowableHolder();

		// It's a CallbackPreferringPlatformTransactionManager: pass a TransactionCallback in.
		try {
			Object result = ((CallbackPreferringPlatformTransactionManager) ptm).execute(txAttr, status -> {
				TransactionInfo txInfo = prepareTransactionInfo(ptm, txAttr, joinpointIdentification, status);
				try {
					// 执行process递归回调即下一个Intercept
					Object retVal = invocation.proceedWithInvocation();
					if (vavrPresent && VavrDelegate.isVavrTry(retVal)) {
						// Set rollback-only in case of Vavr failure matching our rollback rules...
						retVal = VavrDelegate.evaluateTryFailure(retVal, txAttr, status);
					}
					return retVal;
				}
				catch (Throwable ex) {
					if (txAttr.rollbackOn(ex)) {
						// A RuntimeException: will lead to a rollback.
						if (ex instanceof RuntimeException) {
							throw (RuntimeException) ex;
						}
						else {
							throw new ThrowableHolderException(ex);
						}
					}
					else {
						// A normal return value: will lead to a commit.
						throwableHolder.throwable = ex;
						return null;
					}
				}
				finally {
					cleanupTransactionInfo(txInfo);
				}
			});

			// Check result state: It might indicate a Throwable to rethrow.
			if (throwableHolder.throwable != null) {
				throw throwableHolder.throwable;
			}
			return result;
		}
		catch (ThrowableHolderException ex) {
			throw ex.getCause();
		}
		catch (TransactionSystemException ex2) {
			if (throwableHolder.throwable != null) {
				logger.error("Application exception overridden by commit exception", throwableHolder.throwable);
				ex2.initApplicationException(throwableHolder.throwable);
			}
			throw ex2;
		}
		catch (Throwable ex2) {
			if (throwableHolder.throwable != null) {
				logger.error("Application exception overridden by commit exception", throwableHolder.throwable);
			}
			throw ex2;
		}
	}
}
```

## 5.1createTransactionIfNecessary
```java
protected TransactionInfo createTransactionIfNecessary(@Nullable PlatformTransactionManager tm,
			@Nullable TransactionAttribute txAttr, final String joinpointIdentification) {

	// If no name specified, apply method identification as transaction name.
	// 如果没有name则将joinpointIdentification赋值
	// txAttr是@transitional注解对象
	if (txAttr != null && txAttr.getName() == null) {
		txAttr = new DelegatingTransactionAttribute(txAttr) {
			@Override
			public String getName() {
				return joinpointIdentification;
			}
		};
	}

	TransactionStatus status = null;
	if (txAttr != null) {
		if (tm != null) {
			// 获取事务
			status = tm.getTransaction(txAttr);
		}
		else {
			if (logger.isDebugEnabled()) {
				logger.debug("Skipping transactional joinpoint [" + joinpointIdentification +
						"] because no transaction manager has been configured");
			}
		}
	}
	return prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);
}
```

## 5.1.1getTransaction
```java
public final TransactionStatus getTransaction(@Nullable TransactionDefinition definition)
			throws TransactionException {

	// Use defaults if no transaction definition given.
	TransactionDefinition def = (definition != null ? definition : TransactionDefinition.withDefaults());

	// 获取一个事务工具对象，如果之前有已经开启的事务则会包含已经开启的事务
	Object transaction = doGetTransaction();
	boolean debugEnabled = logger.isDebugEnabled();

	// 判断是否存在一个事务
	if (isExistingTransaction(transaction)) {
		// Existing transaction found -> check propagation behavior to find out how to behave.
		return handleExistingTransaction(def, transaction, debugEnabled);
	}

	// Check definition settings for new transaction.
	// 判断执行sql是否超时
	if (def.getTimeout() < TransactionDefinition.TIMEOUT_DEFAULT) {
		throw new InvalidTimeoutException("Invalid transaction timeout", def.getTimeout());
	}

	// No existing transaction found -> check propagation behavior to find out how to proceed.
	// 没有找到事务，判断传播机制
	if (def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_MANDATORY) {
		throw new IllegalTransactionStateException(
				"No existing transaction found for transaction marked with propagation 'mandatory'");
	}
	else if (def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRED ||
			def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW ||
			def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
		// 如果是以上三种事务，则不挂起任何事务
		SuspendedResourcesHolder suspendedResources = suspend(null);
		if (debugEnabled) {
			logger.debug("Creating new transaction with name [" + def.getName() + "]: " + def);
		}
		try {
			// 开启一个事务
			return startTransaction(def, transaction, debugEnabled, suspendedResources);
		}
		catch (RuntimeException | Error ex) {
			resume(null, suspendedResources);
			throw ex;
		}
	}
	else {
		// Create "empty" transaction: no actual transaction, but potentially synchronization.
		if (def.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT && logger.isWarnEnabled()) {
			logger.warn("Custom isolation level specified but no actual transaction initiated; " +
					"isolation level will effectively be ignored: " + def);
		}
		boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
		return prepareTransactionStatus(def, null, true, newSynchronization, debugEnabled, null);
	}
}
```
# 6~9均为getTransaction中的分支方法

## 6.DataSourceTransactionManager.doGetTransaction
```java
protected Object doGetTransaction() {
	DataSourceTransactionObject txObject = new DataSourceTransactionObject();
	txObject.setSavepointAllowed(isNestedTransactionAllowed());
	// obtainDataSource获取datasource
	// 根据datasource从ThreadLocale中获取ConnectionHolder
	ConnectionHolder conHolder =
			(ConnectionHolder) TransactionSynchronizationManager.getResource(obtainDataSource());
	// 如果能拿到则放入txObject并表示非新建
	txObject.setConnectionHolder(conHolder, false);
	return txObject;
}
```
### 6.1obtainDataSource
```java
protected DataSource obtainDataSource() {
		// 获取datasource
		DataSource dataSource = getDataSource();
		Assert.state(dataSource != null, "No DataSource set");
		return dataSource;
	}
```

### 6.2TransactionSynchronizationManager.getResource
```java
public static Object getResource(Object key) {
	Object actualKey = TransactionSynchronizationUtils.unwrapResourceIfNecessary(key);
	Object value = doGetResource(actualKey);
	if (value != null && logger.isTraceEnabled()) {
		logger.trace("Retrieved value [" + value + "] for key [" + actualKey + "] bound to thread [" +
				Thread.currentThread().getName() + "]");
	}
	return value;
}
```

#### 6.2.1.1doGetResource
```java
private static Object doGetResource(Object actualKey) {
	// 点击resources可以看到是ThreadLocal，key是datasource对象，value是ConnectionaHolder对象	(包装connection对象)
	Map<Object, Object> map = resources.get();
	if (map == null) {
		return null;
	}
	Object value = map.get(actualKey);
	// Transparently remove ResourceHolder that was marked as void...
	if (value instanceof ResourceHolder && ((ResourceHolder) value).isVoid()) {
		map.remove(actualKey);
		// Remove entire ThreadLocal if empty...
		if (map.isEmpty()) {
			resources.remove();
		}
		value = null;
	}
	return value;
}
```



## 7.DataSourceTransactionManager.isExistingTransaction
```java
protected boolean isExistingTransaction(Object transaction) {
		DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;
		// 判断是否包含ConnectionHolder
		return (txObject.hasConnectionHolder() && txObject.getConnectionHolder().isTransactionActive());
	}
```
## 8.handleExistingTransaction
```java
private TransactionStatus handleExistingTransaction(
			TransactionDefinition definition, Object transaction, boolean debugEnabled)
			throws TransactionException {
	// 根据不同的传播机制做不同的处理
	if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NEVER) {
		throw new IllegalTransactionStateException(
				"Existing transaction found for transaction marked with propagation 'never'");
	}

	if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NOT_SUPPORTED) {
		if (debugEnabled) {
			logger.debug("Suspending current transaction");
		}
		Object suspendedResources = suspend(transaction);
		boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
		return prepareTransactionStatus(
				definition, null, false, newSynchronization, debugEnabled, suspendedResources);
	}

	if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW) {
		if (debugEnabled) {
			logger.debug("Suspending current transaction, creating new transaction with name [" +
					definition.getName() + "]");
		}
		SuspendedResourcesHolder suspendedResources = suspend(transaction);
		try {
			// 这里会将挂起资源传入封装后返回
			return startTransaction(definition, transaction, debugEnabled, suspendedResources);
		}
		catch (RuntimeException | Error beginEx) {
			resumeAfterBeginException(transaction, suspendedResources, beginEx);
			throw beginEx;
		}
	}

	if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
		if (!isNestedTransactionAllowed()) {
			throw new NestedTransactionNotSupportedException(
					"Transaction manager does not allow nested transactions by default - " +
					"specify 'nestedTransactionAllowed' property with value 'true'");
		}
		if (debugEnabled) {
			logger.debug("Creating nested transaction with name [" + definition.getName() + "]");
		}
		if (useSavepointForNestedTransaction()) {
			// Create savepoint within existing Spring-managed transaction,
			// through the SavepointManager API implemented by TransactionStatus.
			// Usually uses JDBC 3.0 savepoints. Never activates Spring synchronization.
			DefaultTransactionStatus status =
					prepareTransactionStatus(definition, transaction, false, false, debugEnabled, null);
			status.createAndHoldSavepoint();
			return status;
		}
		else {
			// Nested transaction through nested begin and commit/rollback calls.
			// Usually only for JTA: Spring synchronization might get activated here
			// in case of a pre-existing JTA transaction.
			return startTransaction(definition, transaction, debugEnabled, null);
		}
	}

	// Assumably PROPAGATION_SUPPORTS or PROPAGATION_REQUIRED.
	if (debugEnabled) {
		logger.debug("Participating in existing transaction");
	}
	if (isValidateExistingTransaction()) {
		if (definition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT) {
			Integer currentIsolationLevel = TransactionSynchronizationManager.getCurrentTransactionIsolationLevel();
			if (currentIsolationLevel == null || currentIsolationLevel != definition.getIsolationLevel()) {
				Constants isoConstants = DefaultTransactionDefinition.constants;
				throw new IllegalTransactionStateException("Participating transaction with definition [" +
						definition + "] specifies isolation level which is incompatible with existing transaction: " +
						(currentIsolationLevel != null ?
								isoConstants.toCode(currentIsolationLevel, DefaultTransactionDefinition.PREFIX_ISOLATION) :
								"(unknown)"));
			}
		}
		if (!definition.isReadOnly()) {
			if (TransactionSynchronizationManager.isCurrentTransactionReadOnly()) {
				throw new IllegalTransactionStateException("Participating transaction with definition [" +
						definition + "] is not marked as read-only but existing transaction is");
			}
		}
	}
	boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
	return prepareTransactionStatus(definition, transaction, false, newSynchronization, debugEnabled, null);
}
```

## 9.suspend
```java
protected final SuspendedResourcesHolder suspend(@Nullable Object transaction) throws TransactionException {
	if (TransactionSynchronizationManager.isSynchronizationActive()) {
		// 挂起TransactionSynchronization,用于事务生命周期回调如下使用
		// TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronization())
		List<TransactionSynchronization> suspendedSynchronizations = doSuspendSynchronization();
		try {
			Object suspendedResources = null;
			if (transaction != null) {
				// 挂起数据库资源并返回连接对象，方便后续放入挂起池中
				suspendedResources = doSuspend(transaction);
			}
			// 挂起名字
			String name = TransactionSynchronizationManager.getCurrentTransactionName();
			TransactionSynchronizationManager.setCurrentTransactionName(null);
			// 挂起readonly
			boolean readOnly = TransactionSynchronizationManager.isCurrentTransactionReadOnly();
			TransactionSynchronizationManager.setCurrentTransactionReadOnly(false);
			// 挂起隔离级别
			Integer isolationLevel = TransactionSynchronizationManager.getCurrentTransactionIsolationLevel();
			TransactionSynchronizationManager.setCurrentTransactionIsolationLevel(null);
			// 是否活跃
			boolean wasActive = TransactionSynchronizationManager.isActualTransactionActive();
			TransactionSynchronizationManager.setActualTransactionActive(false);
			return new SuspendedResourcesHolder(
					suspendedResources, suspendedSynchronizations, name, readOnly, isolationLevel, wasActive);
		}
		catch (RuntimeException | Error ex) {
			// doSuspend failed - original transaction is still active...
			doResumeSynchronization(suspendedSynchronizations);
			throw ex;
		}
	}
	else if (transaction != null) {
		// Transaction active but no synchronization active.
		Object suspendedResources = doSuspend(transaction);
		return new SuspendedResourcesHolder(suspendedResources);
	}
	else {
		// Neither transaction nor synchronization active.
		return null;
	}
}
```

### 9.1DataSourceTransactionManager.doSuspend
以最重要的事务挂起为例
```java
protected Object doSuspend(Object transaction) {
	DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;
	// conn设置为null
	txObject.setConnectionHolder(null);
	return TransactionSynchronizationManager.unbindResource(obtainDataSource());
}
```
### 9.1.1doUnbindResource
将ThreadLocale的map进行remove
```java
private static Object doUnbindResource(Object actualKey) {
		Map<Object, Object> map = resources.get();
	if (map == null) {
		return null;
	}
	Object value = map.remove(actualKey);
	// Remove entire ThreadLocal if empty...
	if (map.isEmpty()) {
		resources.remove();
	}
	// Transparently suppress a ResourceHolder that was marked as void...
	if (value instanceof ResourceHolder && ((ResourceHolder) value).isVoid()) {
		value = null;
	}
	if (value != null && logger.isTraceEnabled()) {
		logger.trace("Removed value [" + value + "] for key [" + actualKey + "] from thread [" +
				Thread.currentThread().getName() + "]");
	}
	return value;
}
```

## 9.getTransactionstartTransaction
处理开启事务核心方法，延续5.1.1getTransaction
```java
private TransactionStatus startTransaction(TransactionDefinition definition, Object transaction,
			boolean debugEnabled, @Nullable SuspendedResourcesHolder suspendedResources) {

	boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
	DefaultTransactionStatus status = newTransactionStatus(
			definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
	// 开启事务
	doBegin(transaction, definition);
	// 将隔离级别、名称等进行设置
	prepareSynchronization(status, definition);
	return status;
}
```

### 9.1DataSourceTransactionManager.doBegin
```java
protected void doBegin(Object transaction, TransactionDefinition definition) {
	DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;
	Connection con = null;

	try {
		if (!txObject.hasConnectionHolder() ||
				txObject.getConnectionHolder().isSynchronizedWithTransaction()) {
			// 从datasource中获取数据库连接
			Connection newCon = obtainDataSource().getConnection();
			if (logger.isDebugEnabled()) {
				logger.debug("Acquired Connection [" + newCon + "] for JDBC transaction");
			}
			// 将该连接放入txObject中并以新建表示
			txObject.setConnectionHolder(new ConnectionHolder(newCon), true);
		}

		txObject.getConnectionHolder().setSynchronizedWithTransaction(true);
		con = txObject.getConnectionHolder().getConnection();

		// 获取隔离级别，并设置
		Integer previousIsolationLevel = DataSourceUtils.prepareConnectionForTransaction(con, definition);
		txObject.setPreviousIsolationLevel(previousIsolationLevel);
		txObject.setReadOnly(definition.isReadOnly());

		// Switch to manual commit if necessary. This is very expensive in some JDBC drivers,
		// so we don't want to do it unnecessarily (for example if we've explicitly
		// configured the connection pool to set it already).
		// 设置autoCommit为false
		if (con.getAutoCommit()) {
			txObject.setMustRestoreAutoCommit(true);
			if (logger.isDebugEnabled()) {
				logger.debug("Switching JDBC Connection [" + con + "] to manual commit");
			}
			con.setAutoCommit(false);
		}

		prepareTransactionalConnection(con, definition);
		txObject.getConnectionHolder().setTransactionActive(true);

		// 设置超时时间
		int timeout = determineTimeout(definition);
		if (timeout != TransactionDefinition.TIMEOUT_DEFAULT) {
			txObject.getConnectionHolder().setTimeoutInSeconds(timeout);
		}

		// Bind the connection holder to the thread.
		// 如果该连接时新建的，则存入ThreadLocale
		if (txObject.isNewConnectionHolder()) {
			TransactionSynchronizationManager.bindResource(obtainDataSource(), txObject.getConnectionHolder());
		}
	}

	catch (Throwable ex) {
		if (txObject.isNewConnectionHolder()) {
			DataSourceUtils.releaseConnection(con, obtainDataSource());
			txObject.setConnectionHolder(null, false);
		}
		throw new CannotCreateTransactionException("Could not open JDBC Connection for transaction", ex);
	}
}
```

### 9.1.1prepareConnectionForTransaction
```java
public static Integer prepareConnectionForTransaction(Connection con, @Nullable TransactionDefinition definition)
			throws SQLException {

	Assert.notNull(con, "No Connection specified");

	boolean debugEnabled = logger.isDebugEnabled();
	// Set read-only flag.
	// 设置readonly
	if (definition != null && definition.isReadOnly()) {
		try {
			if (debugEnabled) {
				logger.debug("Setting JDBC Connection [" + con + "] read-only");
			}
			con.setReadOnly(true);
		}
		catch (SQLException | RuntimeException ex) {
			Throwable exToCheck = ex;
			while (exToCheck != null) {
				if (exToCheck.getClass().getSimpleName().contains("Timeout")) {
					// Assume it's a connection timeout that would otherwise get lost: e.g. from JDBC 4.0
					throw ex;
				}
				exToCheck = exToCheck.getCause();
			}
			// "read-only not supported" SQLException -> ignore, it's just a hint anyway
			logger.debug("Could not set JDBC Connection read-only", ex);
		}
	}

	// Apply specific isolation level, if any.
	// 设置数据库隔离级别
	Integer previousIsolationLevel = null;
	if (definition != null && definition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT) {
		if (debugEnabled) {
			logger.debug("Changing isolation level of JDBC Connection [" + con + "] to " +
					definition.getIsolationLevel());
		}
		int currentIsolation = con.getTransactionIsolation();
		if (currentIsolation != definition.getIsolationLevel()) {
			previousIsolationLevel = currentIsolation;
			con.setTransactionIsolation(definition.getIsolationLevel());
		}
	}

	return previousIsolationLevel;
}
```

## 10.commitTransactionAfterReturning
```java
protected void commitTransactionAfterReturning(@Nullable TransactionInfo txInfo) {
	if (txInfo != null && txInfo.getTransactionStatus() != null) {
		if (logger.isTraceEnabled()) {
			logger.trace("Completing transaction for [" + txInfo.getJoinpointIdentification() + "]");
		}
		txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
	}
}
```
commit核心方法继续进入
```java
public final void commit(TransactionStatus status) throws TransactionException {
	if (status.isCompleted()) {
		throw new IllegalTransactionStateException(
				"Transaction is already completed - do not call commit or rollback more than once per transaction");
	}

	DefaultTransactionStatus defStatus = (DefaultTransactionStatus) status;
	// TransactionAspectSuppert.currentTransactionStatus().setRollbackOnly()判断，用于局部错误，但是在catch后进行回滚，强制回滚
	if (defStatus.isLocalRollbackOnly()) {
		if (defStatus.isDebug()) {
			logger.debug("Transactional code has requested rollback");
		}
		processRollback(defStatus, false);
		return;
	}

	// 嵌套方法a嵌套b，b出现异常，但是a将b()trycatch后，还是会整体回滚,如果有过修改且b方法发生异常，则只有a会插入记录出现事务不一致
	if (!shouldCommitOnGlobalRollbackOnly() && defStatus.isGlobalRollbackOnly()) {
		if (defStatus.isDebug()) {
			logger.debug("Global transaction is marked as rollback-only but transactional code requested commit");
		}
		processRollback(defStatus, true);
		return;
	}

	processCommit(defStatus);
}
```
## 10.1processCommit
```java
private void processCommit(DefaultTransactionStatus status) throws TransactionException {
	try {
		boolean beforeCompletionInvoked = false;

		try {
			boolean unexpectedRollback = false;
			prepareForCommit(status);
			// 单个事务生命周期回调，rollback不会掉该方法
			triggerBeforeCommit(status);
			triggerBeforeCompletion(status);
			beforeCompletionInvoked = true;

			if (status.hasSavepoint()) {
				// nested
				if (status.isDebug()) {
					logger.debug("Releasing transaction savepoint");
				}
				unexpectedRollback = status.isGlobalRollbackOnly();
				status.releaseHeldSavepoint();
			}
			else if (status.isNewTransaction()) {
				// 只有当前methodnew 的才可以提交
				if (status.isDebug()) {
					logger.debug("Initiating transaction commit");
				}
				unexpectedRollback = status.isGlobalRollbackOnly();
				// 提交
				doCommit(status);
			}
			else if (isFailEarlyOnGlobalRollbackOnly()) {
				unexpectedRollback = status.isGlobalRollbackOnly();
			}

			// Throw UnexpectedRollbackException if we have a global rollback-only
			// marker but still didn't get a corresponding exception from commit.
			if (unexpectedRollback) {
				throw new UnexpectedRollbackException(
						"Transaction silently rolled back because it has been marked as rollback-only");
			}
		}
		catch (UnexpectedRollbackException ex) {
			// can only be caused by doCommit
			triggerAfterCompletion(status, TransactionSynchronization.STATUS_ROLLED_BACK);
			throw ex;
		}
		catch (TransactionException ex) {
			// can only be caused by doCommit
			if (isRollbackOnCommitFailure()) {
				doRollbackOnCommitException(status, ex);
			}
			else {
				triggerAfterCompletion(status, TransactionSynchronization.STATUS_UNKNOWN);
			}
			throw ex;
		}
		catch (RuntimeException | Error ex) {
			if (!beforeCompletionInvoked) {
				triggerBeforeCompletion(status);
			}
			doRollbackOnCommitException(status, ex);
			throw ex;
		}

		// Trigger afterCommit callbacks, with an exception thrown there
		// propagated to callers but the transaction still considered as committed.
		try {
			triggerAfterCommit(status);
		}
		finally {
			triggerAfterCompletion(status, TransactionSynchronization.STATUS_COMMITTED);
		}

	}
	finally {
		// 恢复被挂起的事务
		cleanupAfterCompletion(status);
	}
}
```

## 10.1.1cleanupAfterCompletion
```java
private void cleanupAfterCompletion(DefaultTransactionStatus status) {
	status.setCompleted();
	if (status.isNewSynchronization()) {
		TransactionSynchronizationManager.clear();
	}
	if (status.isNewTransaction()) {
		// 关闭数据库连接，如果是当前正在执行的method建的事务比如a嵌b，a会执行关闭，b不执行
		doCleanupAfterCompletion(status.getTransaction());
	}
	if (status.getSuspendedResources() != null) {
		if (status.isDebug()) {
			logger.debug("Resuming suspended transaction after completion of inner transaction");
		}
		Object transaction = (status.hasTransaction() ? status.getTransaction() : null);
		// 恢复
		resume(transaction, (SuspendedResourcesHolder) status.getSuspendedResources());
	}
}
```

## 11.completeTransactionAfterThrowing
回滚事务
```java
protected void completeTransactionAfterThrowing(@Nullable TransactionInfo txInfo, Throwable ex) {
	if (txInfo != null && txInfo.getTransactionStatus() != null) {
		if (logger.isTraceEnabled()) {
			logger.trace("Completing transaction for [" + txInfo.getJoinpointIdentification() +
					"] after exception: " + ex);
		}
		// 判断rollback是否符合当前异常
		if (txInfo.transactionAttribute != null && txInfo.transactionAttribute.rollbackOn(ex)) {
			try {
				txInfo.getTransactionManager().rollback(txInfo.getTransactionStatus());
			}
			catch (TransactionSystemException ex2) {
				logger.error("Application exception overridden by rollback exception", ex);
				ex2.initApplicationException(ex);
				throw ex2;
			}
			catch (RuntimeException | Error ex2) {
				logger.error("Application exception overridden by rollback exception", ex);
				throw ex2;
			}
		}
		else {
			// We don't roll back on this exception.
			// Will still roll back if TransactionStatus.isRollbackOnly() is true.
			try {
				txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
			}
			catch (TransactionSystemException ex2) {
				logger.error("Application exception overridden by commit exception", ex);
				ex2.initApplicationException(ex);
				throw ex2;
			}
			catch (RuntimeException | Error ex2) {
				logger.error("Application exception overridden by commit exception", ex);
				throw ex2;
			}
		}
	}
}
```

## 11.1rollback
```java
public final void rollback(TransactionStatus status) throws TransactionException {
	if (status.isCompleted()) {
		throw new IllegalTransactionStateException(
				"Transaction is already completed - do not call commit or rollback more than once per transaction");
	}

	DefaultTransactionStatus defStatus = (DefaultTransactionStatus) status;
	processRollback(defStatus, false);
}
```

```java
private void processRollback(DefaultTransactionStatus status, boolean unexpected) {
try {
	boolean unexpectedRollback = unexpected;

	try {
		triggerBeforeCompletion(status);

		if (status.hasSavepoint()) {
			// 判断savepoint
			if (status.isDebug()) {
				logger.debug("Rolling back transaction to savepoint");
			}
			status.rollbackToHeldSavepoint();
		}
		else if (status.isNewTransaction()) {
			// 如果是当前新建直接回滚
			if (status.isDebug()) {
				logger.debug("Initiating transaction rollback");
			}
			doRollback(status);
		}
		else {
			// Participating in larger transaction
			if (status.hasTransaction()) {
				// 如果是延续某个事务则打上RollbackOnly标记，方便上层回滚
				if (status.isLocalRollbackOnly() || isGlobalRollbackOnParticipationFailure()) {
					if (status.isDebug()) {
						logger.debug("Participating transaction failed - marking existing transaction as rollback-only");
					}
					doSetRollbackOnly(status);
				}
				else {
					if (status.isDebug()) {
						logger.debug("Participating transaction failed - letting transaction originator decide on rollback");
					}
				}
			}
			else {
				logger.debug("Should roll back transaction but cannot - no transaction available");
			}
			// Unexpected rollback only matters here if we're asked to fail early
			if (!isFailEarlyOnGlobalRollbackOnly()) {
				unexpectedRollback = false;
			}
		}
	}
	catch (RuntimeException | Error ex) {
		triggerAfterCompletion(status, TransactionSynchronization.STATUS_UNKNOWN);
		throw ex;
	}

	triggerAfterCompletion(status, TransactionSynchronization.STATUS_ROLLED_BACK);

	// Raise UnexpectedRollbackException if we had a global rollback-only marker
	if (unexpectedRollback) {
		throw new UnexpectedRollbackException(
				"Transaction rolled back because it has been marked as rollback-only");
	}
}
finally {
	cleanupAfterCompletion(status);
}
}
```