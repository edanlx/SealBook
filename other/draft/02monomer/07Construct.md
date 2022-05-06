# 【框架类源码】07-spring推断构造方法
> 欢迎关注b站账号/公众号【六边形战士夏宁】，一个要把各项指标拉满的男人。该文章已在[github目录](https://github.com/edanlx/SealBook/blob/master/catalogue/wechat.md)收录。
屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。

## 1.背景
spring推断构造方法
## 2.整体流程
1. 没有添加autowire注解->有多个构造方法->返回null->调用无参，没有无参则报错
2. 没有添加autowire注解->只有1个有参的构造方法->返回该构造方法->使用
3. 没有添加autowire注解->只有1个无参构造方法->返回null->调用无参
4. 有添加autowire注解->只有1个require方法->返回该构造方法->使用
5. 有添加autowire注解->有多个require为ture->抛异常
6. 有添加autowire注解->有1个require为true多个false->抛异常
7. 有添加autowire注解->全false->返回多个->按照权重、长度等进行排序，public在前，参数多的在前，不用接口的在前
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
### 4.1createBeanInstance
```java
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
	// Make sure bean class is actually resolved at this point.
	Class<?> beanClass = resolveBeanClass(mbd, beanName);

	if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
		throw new BeanCreationException(mbd.getResourceDescription(), beanName,
				"Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
	}
	// 处理beanDefinition的supplier
	Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
	if (instanceSupplier != null) {
		return obtainFromSupplier(instanceSupplier, beanName);
	}

	// @Bean的处理
	if (mbd.getFactoryMethodName() != null) {
		return instantiateUsingFactoryMethod(beanName, mbd, args);
	}

	// Shortcut when re-creating the same bean...
	boolean resolved = false;
	boolean autowireNecessary = false;
	if (args == null) {
		synchronized (mbd.constructorArgumentLock) {
			if (mbd.resolvedConstructorOrFactoryMethod != null) {
				// 缓存
				resolved = true;
				// 构造方法的参数是否已经需要参数且进行缓存
				autowireNecessary = mbd.constructorArgumentsResolved;
			}
		}
	}
	if (resolved) {
		if (autowireNecessary) {
			// 使用缓存过得构造方法构造(多例情况)
			return autowireConstructor(beanName, mbd, null, null);
		}
		else {
			// 直接构造
			return instantiateBean(beanName, mbd);
		}
	}

	// Candidate constructors for autowiring?
	// 推断构造方法，1个，多个，null，public往前排，参数多的往前排，分数小的往前排(宽松模式:父类会分大)
	Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
	if (ctors != null || mbd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR ||
			mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args)) {
		// 执行构造
		return autowireConstructor(beanName, mbd, ctors, args);
	}

	// Preferred constructors for default construction?
	ctors = mbd.getPreferredConstructors();
	if (ctors != null) {
		return autowireConstructor(beanName, mbd, ctors, null);
	}

	// No special handling: simply use no-arg constructor.
	// 无参构造
	return instantiateBean(beanName, mbd);
}
```

### 4.2determineConstructorsFromBeanPostProcessors
```java
protected Constructor<?>[] determineConstructorsFromBeanPostProcessors(@Nullable Class<?> beanClass, String beanName)
			throws BeansException {

	if (beanClass != null && hasInstantiationAwareBeanPostProcessors()) {
		for (BeanPostProcessor bp : getBeanPostProcessors()) {
			if (bp instanceof SmartInstantiationAwareBeanPostProcessor) {
				SmartInstantiationAwareBeanPostProcessor ibp = (SmartInstantiationAwareBeanPostProcessor) bp;
				Constructor<?>[] ctors = ibp.determineCandidateConstructors(beanClass, beanName);
				if (ctors != null) {
					return ctors;
				}
			}
		}
	}
	return null;
}
```


```java
public Constructor<?>[] determineCandidateConstructors(Class<?> beanClass, final String beanName)
			throws BeanCreationException {
	// Let's check for lookup methods here...
	if (!this.lookupMethodsChecked.contains(beanName)) {
		if (AnnotationUtils.isCandidateClass(beanClass, Lookup.class)) {
			try {
				Class<?> targetClass = beanClass;
				do {
					ReflectionUtils.doWithLocalMethods(targetClass, method -> {
						Lookup lookup = method.getAnnotation(Lookup.class);
						if (lookup != null) {
							Assert.state(this.beanFactory != null, "No BeanFactory available");
							LookupOverride override = new LookupOverride(method, lookup.value());
							try {
								RootBeanDefinition mbd = (RootBeanDefinition)
										this.beanFactory.getMergedBeanDefinition(beanName);
								mbd.getMethodOverrides().addOverride(override);
							}
							catch (NoSuchBeanDefinitionException ex) {
								throw new BeanCreationException(beanName,
										"Cannot apply @Lookup to beans without corresponding bean definition");
							}
						}
					});
					targetClass = targetClass.getSuperclass();
				}
				while (targetClass != null && targetClass != Object.class);

			}
			catch (IllegalStateException ex) {
				throw new BeanCreationException(beanName, "Lookup method resolution failed", ex);
			}
		}
		this.lookupMethodsChecked.add(beanName);
	}

	// Quick check on the concurrent map first, with minimal locking.
	Constructor<?>[] candidateConstructors = this.candidateConstructorsCache.get(beanClass);
	if (candidateConstructors == null) {
		// Fully synchronized resolution now...
		synchronized (this.candidateConstructorsCache) {
			candidateConstructors = this.candidateConstructorsCache.get(beanClass);
			if (candidateConstructors == null) {
				Constructor<?>[] rawCandidates;
				try {
					rawCandidates = beanClass.getDeclaredConstructors();
				}
				catch (Throwable ex) {
					throw new BeanCreationException(beanName,
							"Resolution of declared constructors on bean Class [" + beanClass.getName() +
							"] from ClassLoader [" + beanClass.getClassLoader() + "] failed", ex);
				}
				List<Constructor<?>> candidates = new ArrayList<>(rawCandidates.length);
				Constructor<?> requiredConstructor = null;
				Constructor<?> defaultConstructor = null;
				Constructor<?> primaryConstructor = BeanUtils.findPrimaryConstructor(beanClass);
				int nonSyntheticConstructors = 0;
				for (Constructor<?> candidate : rawCandidates) {
					if (!candidate.isSynthetic()) {
						nonSyntheticConstructors++;
					}
					else if (primaryConstructor != null) {
						continue;
					}
					MergedAnnotation<?> ann = findAutowiredAnnotation(candidate);
					if (ann == null) {
						Class<?> userClass = ClassUtils.getUserClass(beanClass);
						if (userClass != beanClass) {
							try {
								Constructor<?> superCtor =
										userClass.getDeclaredConstructor(candidate.getParameterTypes());
								ann = findAutowiredAnnotation(superCtor);
							}
							catch (NoSuchMethodException ex) {
								// Simply proceed, no equivalent superclass constructor found...
							}
						}
					}
					if (ann != null) {
						if (requiredConstructor != null) {
							throw new BeanCreationException(beanName,
									"Invalid autowire-marked constructor: " + candidate +
									". Found constructor with 'required' Autowired annotation already: " +
									requiredConstructor);
						}
						boolean required = determineRequiredStatus(ann);
						if (required) {
							if (!candidates.isEmpty()) {
								throw new BeanCreationException(beanName,
										"Invalid autowire-marked constructors: " + candidates +
										". Found constructor with 'required' Autowired annotation: " +
										candidate);
							}
							// 记录一个require为true的方法
							requiredConstructor = candidate;
						}
						candidates.add(candidate);
					}
					else if (candidate.getParameterCount() == 0) {
						// 记录无参构造
						defaultConstructor = candidate;
					}
				}
				// 可能为空->无auto注解
				// 全是false
				// 一个true
				if (!candidates.isEmpty()) {
					// Add default constructor to list of optional constructors, as fallback.
					if (requiredConstructor == null) {
						if (defaultConstructor != null) {
							candidates.add(defaultConstructor);
						}
						else if (candidates.size() == 1 && logger.isInfoEnabled()) {
							logger.info("Inconsistent constructor declaration on bean with name '" + beanName +
									"': single autowire-marked constructor flagged as optional - " +
									"this constructor is effectively required since there is no " +
									"default constructor to fall back to: " + candidates.get(0));
						}
					}
					candidateConstructors = candidates.toArray(new Constructor<?>[0]);
				}
				// 判断是否只有1个构造方法，判断参数数量
				else if (rawCandidates.length == 1 && rawCandidates[0].getParameterCount() > 0) {
					candidateConstructors = new Constructor<?>[] {rawCandidates[0]};
				}
				else if (nonSyntheticConstructors == 2 && primaryConstructor != null &&
						defaultConstructor != null && !primaryConstructor.equals(defaultConstructor)) {
					candidateConstructors = new Constructor<?>[] {primaryConstructor, defaultConstructor};
				}
				else if (nonSyntheticConstructors == 1 && primaryConstructor != null) {
					candidateConstructors = new Constructor<?>[] {primaryConstructor};
				}
				else {
					candidateConstructors = new Constructor<?>[0];
				}
				this.candidateConstructorsCache.put(beanClass, candidateConstructors);
			}
		}
	}
	return (candidateConstructors.length > 0 ? candidateConstructors : null);
}
```

## 4.3autowireConstructor
```java
protected BeanWrapper autowireConstructor(
		String beanName, RootBeanDefinition mbd, @Nullable Constructor<?>[] ctors, @Nullable Object[] explicitArgs) {

	return new ConstructorResolver(this).autowireConstructor(beanName, mbd, ctors, explicitArgs);
}
```

```java
public BeanWrapper autowireConstructor(String beanName, RootBeanDefinition mbd,
			@Nullable Constructor<?>[] chosenCtors, @Nullable Object[] explicitArgs) {

	BeanWrapperImpl bw = new BeanWrapperImpl();
	this.beanFactory.initBeanWrapper(bw);

	Constructor<?> constructorToUse = null;
	ArgumentsHolder argsHolderToUse = null;
	Object[] argsToUse = null;

	if (explicitArgs != null) {
		// 有外部传参使用外部的
		argsToUse = explicitArgs;
	}
	else {
		Object[] argsToResolve = null;
		synchronized (mbd.constructorArgumentLock) {
			constructorToUse = (Constructor<?>) mbd.resolvedConstructorOrFactoryMethod;
			if (constructorToUse != null && mbd.constructorArgumentsResolved) {
				// Found a cached constructor...
				argsToUse = mbd.resolvedConstructorArguments;
				if (argsToUse == null) {
					argsToResolve = mbd.preparedConstructorArguments;
				}
			}
		}
		if (argsToResolve != null) {
			argsToUse = resolvePreparedArguments(beanName, mbd, bw, constructorToUse, argsToResolve, true);
		}
	}

	if (constructorToUse == null || argsToUse == null) {
		// Take specified constructors, if any.
		Constructor<?>[] candidates = chosenCtors;
		if (candidates == null) {
			Class<?> beanClass = mbd.getBeanClass();
			try {
				candidates = (mbd.isNonPublicAccessAllowed() ?
						beanClass.getDeclaredConstructors() : beanClass.getConstructors());
			}
			catch (Throwable ex) {
				throw new BeanCreationException(mbd.getResourceDescription(), beanName,
						"Resolution of declared constructors on bean Class [" + beanClass.getName() +
						"] from ClassLoader [" + beanClass.getClassLoader() + "] failed", ex);
			}
		}

		// 只有1个候选构造，并且没有指定参数，并且无参
		if (candidates.length == 1 && explicitArgs == null && !mbd.hasConstructorArgumentValues()) {
			Constructor<?> uniqueCandidate = candidates[0];
			if (uniqueCandidate.getParameterCount() == 0) {
				synchronized (mbd.constructorArgumentLock) {
					mbd.resolvedConstructorOrFactoryMethod = uniqueCandidate;
					mbd.constructorArgumentsResolved = true;
					mbd.resolvedConstructorArguments = EMPTY_ARGS;
				}
				bw.setBeanInstance(instantiate(beanName, mbd, uniqueCandidate, EMPTY_ARGS));
				return bw;
			}
		}

		// Need to resolve the constructor.
		boolean autowiring = (chosenCtors != null ||
				mbd.getResolvedAutowireMode() == AutowireCapableBeanFactory.AUTOWIRE_CONSTRUCTOR);
		ConstructorArgumentValues resolvedValues = null;

		// 确定要选择的构造方法的参数个数最小值，后续判断构造方法的参数个数小于minNrOfArgs，直接去掉
		int minNrOfArgs;
		if (explicitArgs != null) {
			minNrOfArgs = explicitArgs.length;
		}
		else {
			// 获取通过Beandefinition指定值
			ConstructorArgumentValues cargs = mbd.getConstructorArgumentValues();
			resolvedValues = new ConstructorArgumentValues();
			// 处理runtimeBeanReference
			minNrOfArgs = resolveConstructorArguments(beanName, mbd, bw, cargs, resolvedValues);
		}

		// 排序
		AutowireUtils.sortConstructors(candidates);
		int minTypeDiffWeight = Integer.MAX_VALUE;
		Set<Constructor<?>> ambiguousConstructors = null;
		LinkedList<UnsatisfiedDependencyException> causes = null;

		for (Constructor<?> candidate : candidates) {

			int parameterCount = candidate.getParameterCount();

			if (constructorToUse != null && argsToUse != null && argsToUse.length > parameterCount) {
				// Already found greedy constructor that can be satisfied ->
				// do not look any further, there are only less greedy constructors left.
				break;
			}
			if (parameterCount < minNrOfArgs) {
				continue;
			}

			ArgumentsHolder argsHolder;
			Class<?>[] paramTypes = candidate.getParameterTypes();
			if (resolvedValues != null) {
				try {
					String[] paramNames = ConstructorPropertiesChecker.evaluate(candidate, parameterCount);
					if (paramNames == null) {
						ParameterNameDiscoverer pnd = this.beanFactory.getParameterNameDiscoverer();
						if (pnd != null) {
							paramNames = pnd.getParameterNames(candidate);
						}
					}
					argsHolder = createArgumentArray(beanName, mbd, resolvedValues, bw, paramTypes, paramNames,
							getUserDeclaredConstructor(candidate), autowiring, candidates.length == 1);
				}
				catch (UnsatisfiedDependencyException ex) {
					if (logger.isTraceEnabled()) {
						logger.trace("Ignoring constructor [" + candidate + "] of bean '" + beanName + "': " + ex);
					}
					// Swallow and try next constructor.
					if (causes == null) {
						causes = new LinkedList<>();
					}
					causes.add(ex);
					continue;
				}
			}
			else {
				// Explicit arguments given -> arguments length must match exactly.
				if (parameterCount != explicitArgs.length) {
					continue;
				}
				argsHolder = new ArgumentsHolder(explicitArgs);
			}

			int typeDiffWeight = (mbd.isLenientConstructorResolution() ?
					argsHolder.getTypeDifferenceWeight(paramTypes) : argsHolder.getAssignabilityWeight(paramTypes));
			// Choose this constructor if it represents the closest match.
			if (typeDiffWeight < minTypeDiffWeight) {
				constructorToUse = candidate;
				argsHolderToUse = argsHolder;
				argsToUse = argsHolder.arguments;
				minTypeDiffWeight = typeDiffWeight;
				ambiguousConstructors = null;
			}
			else if (constructorToUse != null && typeDiffWeight == minTypeDiffWeight) {
				if (ambiguousConstructors == null) {
					ambiguousConstructors = new LinkedHashSet<>();
					ambiguousConstructors.add(constructorToUse);
				}
				ambiguousConstructors.add(candidate);
			}
		}

		if (constructorToUse == null) {
			if (causes != null) {
				UnsatisfiedDependencyException ex = causes.removeLast();
				for (Exception cause : causes) {
					this.beanFactory.onSuppressedException(cause);
				}
				throw ex;
			}
			throw new BeanCreationException(mbd.getResourceDescription(), beanName,
					"Could not resolve matching constructor " +
					"(hint: specify index/type/name arguments for simple parameters to avoid type ambiguities)");
		}
		else if (ambiguousConstructors != null && !mbd.isLenientConstructorResolution()) {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName,
					"Ambiguous constructor matches found in bean '" + beanName + "' " +
					"(hint: specify index/type/name arguments for simple parameters to avoid type ambiguities): " +
					ambiguousConstructors);
		}

		if (explicitArgs == null && argsHolderToUse != null) {
			argsHolderToUse.storeCache(mbd, constructorToUse);
		}
	}

	Assert.state(argsToUse != null, "Unresolved constructor arguments");
	bw.setBeanInstance(instantiate(beanName, mbd, constructorToUse, argsToUse));
	return bw;
}
```

## 4.4instantiateUsingFactoryMethod处理@Bean
```java
public BeanWrapper instantiateUsingFactoryMethod(
			String beanName, RootBeanDefinition mbd, @Nullable Object[] explicitArgs) {

	BeanWrapperImpl bw = new BeanWrapperImpl();
	this.beanFactory.initBeanWrapper(bw);

	Object factoryBean;
	Class<?> factoryClass;
	boolean isStatic;

	String factoryBeanName = mbd.getFactoryBeanName();
	if (factoryBeanName != null) {
		if (factoryBeanName.equals(beanName)) {
			throw new BeanDefinitionStoreException(mbd.getResourceDescription(), beanName,
					"factory-bean reference points back to the same bean definition");
		}
		factoryBean = this.beanFactory.getBean(factoryBeanName);
		if (mbd.isSingleton() && this.beanFactory.containsSingleton(beanName)) {
			throw new ImplicitlyAppearedSingletonException();
		}
		factoryClass = factoryBean.getClass();
		isStatic = false;
	}
	else {
		// It's a static factory method on the bean class.
		if (!mbd.hasBeanClass()) {
			throw new BeanDefinitionStoreException(mbd.getResourceDescription(), beanName,
					"bean definition declares neither a bean class nor a factory-bean reference");
		}
		factoryBean = null;
		factoryClass = mbd.getBeanClass();
		isStatic = true;
	}

	Method factoryMethodToUse = null;
	ArgumentsHolder argsHolderToUse = null;
	Object[] argsToUse = null;

	if (explicitArgs != null) {
		argsToUse = explicitArgs;
	}
	else {
		Object[] argsToResolve = null;
		synchronized (mbd.constructorArgumentLock) {
			factoryMethodToUse = (Method) mbd.resolvedConstructorOrFactoryMethod;
			if (factoryMethodToUse != null && mbd.constructorArgumentsResolved) {
				// Found a cached factory method...
				argsToUse = mbd.resolvedConstructorArguments;
				if (argsToUse == null) {
					argsToResolve = mbd.preparedConstructorArguments;
				}
			}
		}
		if (argsToResolve != null) {
			argsToUse = resolvePreparedArguments(beanName, mbd, bw, factoryMethodToUse, argsToResolve, true);
		}
	}

	if (factoryMethodToUse == null || argsToUse == null) {
		// Need to determine the factory method...
		// Try all methods with this name to see if they match the given arguments.
		factoryClass = ClassUtils.getUserClass(factoryClass);

		List<Method> candidates = null;
		if (mbd.isFactoryMethodUnique) {
			if (factoryMethodToUse == null) {
				factoryMethodToUse = mbd.getResolvedFactoryMethod();
			}
			if (factoryMethodToUse != null) {
				candidates = Collections.singletonList(factoryMethodToUse);
			}
		}
		if (candidates == null) {
			candidates = new ArrayList<>();
			Method[] rawCandidates = getCandidateMethods(factoryClass, mbd);
			for (Method candidate : rawCandidates) {
				if (Modifier.isStatic(candidate.getModifiers()) == isStatic && mbd.isFactoryMethod(candidate)) {
					candidates.add(candidate);
				}
			}
		}

		if (candidates.size() == 1 && explicitArgs == null && !mbd.hasConstructorArgumentValues()) {
			Method uniqueCandidate = candidates.get(0);
			if (uniqueCandidate.getParameterCount() == 0) {
				mbd.factoryMethodToIntrospect = uniqueCandidate;
				synchronized (mbd.constructorArgumentLock) {
					mbd.resolvedConstructorOrFactoryMethod = uniqueCandidate;
					mbd.constructorArgumentsResolved = true;
					mbd.resolvedConstructorArguments = EMPTY_ARGS;
				}
				bw.setBeanInstance(instantiate(beanName, mbd, factoryBean, uniqueCandidate, EMPTY_ARGS));
				return bw;
			}
		}

		if (candidates.size() > 1) {  // explicitly skip immutable singletonList
			candidates.sort(AutowireUtils.EXECUTABLE_COMPARATOR);
		}

		ConstructorArgumentValues resolvedValues = null;
		boolean autowiring = (mbd.getResolvedAutowireMode() == AutowireCapableBeanFactory.AUTOWIRE_CONSTRUCTOR);
		int minTypeDiffWeight = Integer.MAX_VALUE;
		Set<Method> ambiguousFactoryMethods = null;

		int minNrOfArgs;
		if (explicitArgs != null) {
			minNrOfArgs = explicitArgs.length;
		}
		else {
			// We don't have arguments passed in programmatically, so we need to resolve the
			// arguments specified in the constructor arguments held in the bean definition.
			if (mbd.hasConstructorArgumentValues()) {
				ConstructorArgumentValues cargs = mbd.getConstructorArgumentValues();
				resolvedValues = new ConstructorArgumentValues();
				minNrOfArgs = resolveConstructorArguments(beanName, mbd, bw, cargs, resolvedValues);
			}
			else {
				minNrOfArgs = 0;
			}
		}

		LinkedList<UnsatisfiedDependencyException> causes = null;

		for (Method candidate : candidates) {

			int parameterCount = candidate.getParameterCount();
			if (parameterCount >= minNrOfArgs) {
				ArgumentsHolder argsHolder;

				Class<?>[] paramTypes = candidate.getParameterTypes();
				if (explicitArgs != null) {
					// Explicit arguments given -> arguments length must match exactly.
					if (paramTypes.length != explicitArgs.length) {
						continue;
					}
					argsHolder = new ArgumentsHolder(explicitArgs);
				}
				else {
					// Resolved constructor arguments: type conversion and/or autowiring necessary.
					try {
						String[] paramNames = null;
						ParameterNameDiscoverer pnd = this.beanFactory.getParameterNameDiscoverer();
						if (pnd != null) {
							paramNames = pnd.getParameterNames(candidate);
						}
						argsHolder = createArgumentArray(beanName, mbd, resolvedValues, bw,
								paramTypes, paramNames, candidate, autowiring, candidates.size() == 1);
					}
					catch (UnsatisfiedDependencyException ex) {
						if (logger.isTraceEnabled()) {
							logger.trace("Ignoring factory method [" + candidate + "] of bean '" + beanName + "': " + ex);
						}
						// Swallow and try next overloaded factory method.
						if (causes == null) {
							causes = new LinkedList<>();
						}
						causes.add(ex);
						continue;
					}
				}

				int typeDiffWeight = (mbd.isLenientConstructorResolution() ?
						argsHolder.getTypeDifferenceWeight(paramTypes) : argsHolder.getAssignabilityWeight(paramTypes));
				// Choose this factory method if it represents the closest match.
				if (typeDiffWeight < minTypeDiffWeight) {
					factoryMethodToUse = candidate;
					argsHolderToUse = argsHolder;
					argsToUse = argsHolder.arguments;
					minTypeDiffWeight = typeDiffWeight;
					ambiguousFactoryMethods = null;
				}
				// Find out about ambiguity: In case of the same type difference weight
				// for methods with the same number of parameters, collect such candidates
				// and eventually raise an ambiguity exception.
				// However, only perform that check in non-lenient constructor resolution mode,
				// and explicitly ignore overridden methods (with the same parameter signature).
				else if (factoryMethodToUse != null && typeDiffWeight == minTypeDiffWeight &&
						!mbd.isLenientConstructorResolution() &&
						paramTypes.length == factoryMethodToUse.getParameterCount() &&
						!Arrays.equals(paramTypes, factoryMethodToUse.getParameterTypes())) {
					if (ambiguousFactoryMethods == null) {
						ambiguousFactoryMethods = new LinkedHashSet<>();
						ambiguousFactoryMethods.add(factoryMethodToUse);
					}
					ambiguousFactoryMethods.add(candidate);
				}
			}
		}

		if (factoryMethodToUse == null || argsToUse == null) {
			if (causes != null) {
				UnsatisfiedDependencyException ex = causes.removeLast();
				for (Exception cause : causes) {
					this.beanFactory.onSuppressedException(cause);
				}
				throw ex;
			}
			List<String> argTypes = new ArrayList<>(minNrOfArgs);
			if (explicitArgs != null) {
				for (Object arg : explicitArgs) {
					argTypes.add(arg != null ? arg.getClass().getSimpleName() : "null");
				}
			}
			else if (resolvedValues != null) {
				Set<ValueHolder> valueHolders = new LinkedHashSet<>(resolvedValues.getArgumentCount());
				valueHolders.addAll(resolvedValues.getIndexedArgumentValues().values());
				valueHolders.addAll(resolvedValues.getGenericArgumentValues());
				for (ValueHolder value : valueHolders) {
					String argType = (value.getType() != null ? ClassUtils.getShortName(value.getType()) :
							(value.getValue() != null ? value.getValue().getClass().getSimpleName() : "null"));
					argTypes.add(argType);
				}
			}
			String argDesc = StringUtils.collectionToCommaDelimitedString(argTypes);
			throw new BeanCreationException(mbd.getResourceDescription(), beanName,
					"No matching factory method found: " +
					(mbd.getFactoryBeanName() != null ?
						"factory bean '" + mbd.getFactoryBeanName() + "'; " : "") +
					"factory method '" + mbd.getFactoryMethodName() + "(" + argDesc + ")'. " +
					"Check that a method with the specified name " +
					(minNrOfArgs > 0 ? "and arguments " : "") +
					"exists and that it is " +
					(isStatic ? "static" : "non-static") + ".");
		}
		else if (void.class == factoryMethodToUse.getReturnType()) {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName,
					"Invalid factory method '" + mbd.getFactoryMethodName() +
					"': needs to have a non-void return type!");
		}
		else if (ambiguousFactoryMethods != null) {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName,
					"Ambiguous factory method matches found in bean '" + beanName + "' " +
					"(hint: specify index/type/name arguments for simple parameters to avoid type ambiguities): " +
					ambiguousFactoryMethods);
		}

		if (explicitArgs == null && argsHolderToUse != null) {
			mbd.factoryMethodToIntrospect = factoryMethodToUse;
			argsHolderToUse.storeCache(mbd, factoryMethodToUse);
		}
	}

	bw.setBeanInstance(instantiate(beanName, mbd, factoryBean, factoryMethodToUse, argsToUse));
	return bw;
}
```

## 4.5@lookUp
```java
protected BeanWrapper instantiateBean(final String beanName, final RootBeanDefinition mbd) {
	try {
		Object beanInstance;
		final BeanFactory parent = this;
		if (System.getSecurityManager() != null) {
			beanInstance = AccessController.doPrivileged((PrivilegedAction<Object>) () ->
					getInstantiationStrategy().instantiate(mbd, beanName, parent),
					getAccessControlContext());
		}
		else {
			// 默认cglibSubclassingInstantiationStrategy
			beanInstance = getInstantiationStrategy().instantiate(mbd, beanName, parent);
		}
		BeanWrapper bw = new BeanWrapperImpl(beanInstance);
		initBeanWrapper(bw);
		return bw;
	}
	catch (Throwable ex) {
		throw new BeanCreationException(
				mbd.getResourceDescription(), beanName, "Instantiation of bean failed", ex);
	}
}
```
```java
public Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner) {
	// Don't override the class with CGLIB if no overrides.
	// 判断@LookUp
	if (!bd.hasMethodOverrides()) {
		Constructor<?> constructorToUse;
		synchronized (bd.constructorArgumentLock) {
			constructorToUse = (Constructor<?>) bd.resolvedConstructorOrFactoryMethod;
			if (constructorToUse == null) {
				final Class<?> clazz = bd.getBeanClass();
				if (clazz.isInterface()) {
					throw new BeanInstantiationException(clazz, "Specified class is an interface");
				}
				try {
					if (System.getSecurityManager() != null) {
						constructorToUse = AccessController.doPrivileged(
								(PrivilegedExceptionAction<Constructor<?>>) clazz::getDeclaredConstructor);
					}
					else {
						constructorToUse = clazz.getDeclaredConstructor();
					}
					bd.resolvedConstructorOrFactoryMethod = constructorToUse;
				}
				catch (Throwable ex) {
					throw new BeanInstantiationException(clazz, "No default constructor found", ex);
				}
			}
		}
		return BeanUtils.instantiateClass(constructorToUse);
	}
	else {
		// 有@lookUp，是写@lookup所在的那个类生成代理对象
		// Must generate CGLIB subclass.
		return instantiateWithMethodInjection(bd, beanName, owner);
	}
}
```
```java
protected Object instantiateWithMethodInjection(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner) {
	return instantiateWithMethodInjection(bd, beanName, owner, null);
}

protected Object instantiateWithMethodInjection(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner,
			@Nullable Constructor<?> ctor, Object... args) {

	// Must generate CGLIB subclass...
	return new CglibSubclassCreator(bd, owner).instantiate(ctor, args);
}

public Object instantiate(@Nullable Constructor<?> ctor, Object... args) {
	Class<?> subclass = createEnhancedSubclass(this.beanDefinition);
	Object instance;
	if (ctor == null) {
		instance = BeanUtils.instantiateClass(subclass);
	}
	else {
		try {
			Constructor<?> enhancedSubclassConstructor = subclass.getConstructor(ctor.getParameterTypes());
			instance = enhancedSubclassConstructor.newInstance(args);
		}
		catch (Exception ex) {
			throw new BeanInstantiationException(this.beanDefinition.getBeanClass(),
					"Failed to invoke constructor for CGLIB enhanced subclass [" + subclass.getName() + "]", ex);
		}
	}
	// SPR-10785: set callbacks directly on the instance instead of in the
	// enhanced class (via the Enhancer) in order to avoid memory leaks.
	Factory factory = (Factory) instance;
	// 拦截器
	factory.setCallbacks(new Callback[] {NoOp.INSTANCE,
			new LookupOverrideMethodInterceptor(this.beanDefinition, this.owner),
			new ReplaceOverrideMethodInterceptor(this.beanDefinition, this.owner)});
	return instance;
}

@Override
public Object intercept(Object obj, Method method, Object[] args, MethodProxy mp) throws Throwable {
	// Cast is safe, as CallbackFilter filters are used selectively.
	// 如果该代理对象执行lookup注解在修改为执行getBean
	LookupOverride lo = (LookupOverride) getBeanDefinition().getMethodOverrides().getOverride(method);
	Assert.state(lo != null, "LookupOverride not found");
	Object[] argsToUse = (args.length > 0 ? args : null);  // if no-arg, don't insist on args at all
	if (StringUtils.hasText(lo.getBeanName())) {
		return (argsToUse != null ? this.owner.getBean(lo.getBeanName(), argsToUse) :
				this.owner.getBean(lo.getBeanName()));
	}
	else {
		return (argsToUse != null ? this.owner.getBean(method.getReturnType(), argsToUse) :
				this.owner.getBean(method.getReturnType()));
	}
}
```