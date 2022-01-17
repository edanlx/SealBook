# 【JAVA框架】04-spring的autowired注入流程
> 欢迎关注b站账号/公众号【六边形战士夏宁】，一个要把各项指标拉满的男人。该文章已在[github目录](https://github.com/edanlx/SealBook/blob/master/catalogue/wechat.md)收录。
屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。

## 1.背景
@autowired注解作为spring最重要的注解之一，其核心原理如何赋值
## 2.整体流程
1. postProcessMergedBeanDefinition流程先扫描然后将符合条件的@Value、@Autowired字段及方法缓存
2. postProcessProperties进行回调，如果是@Value直接解析从环境变量中获取结果，如果是@Autowired则进行6次过滤获得最终结果(如果是注入自己则会优先注入其他对象)，如果没初始化则初始化
3. 反射赋值
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
	@Autowired
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
AutowiredAnnotationBeanPostProcessor
### 4.1postProcessMergedBeanDefinition.postProcessMergedBeanDefinition()
```java
public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName) {
    InjectionMetadata metadata = this.findAutowiringMetadata(beanName, beanType, (PropertyValues)null);
    metadata.checkConfigMembers(beanDefinition);
}
```

#### 4.1.1InjectionMetadata
```java
private InjectionMetadata findAutowiringMetadata(String beanName, Class<?> clazz, @Nullable PropertyValues pvs) {
    String cacheKey = StringUtils.hasLength(beanName) ? beanName : clazz.getName();
    // 通过缓存去拿
    InjectionMetadata metadata = (InjectionMetadata)this.injectionMetadataCache.get(cacheKey);
    if (InjectionMetadata.needsRefresh(metadata, clazz)) {
        synchronized(this.injectionMetadataCache) {
            metadata = (InjectionMetadata)this.injectionMetadataCache.get(cacheKey);
            if (InjectionMetadata.needsRefresh(metadata, clazz)) {
                if (metadata != null) {
                    metadata.clear(pvs);
                }
                // 获取metadata
                metadata = this.buildAutowiringMetadata(clazz);
                // 存入缓存
                this.injectionMetadataCache.put(cacheKey, metadata);
            }
        }
    }

    return metadata;
}
```

### 4.1.2InjectionMetadata
```java
private InjectionMetadata buildAutowiringMetadata(Class<?> clazz) {
	/**
	 *public AutowiredAnnotationBeanPostProcessor() {
        this.autowiredAnnotationTypes.add(Autowired.class);
        this.autowiredAnnotationTypes.add(Value.class);

        try {
            this.autowiredAnnotationTypes.add(ClassUtils.forName("javax.inject.Inject", AutowiredAnnotationBeanPostProcessor.class.getClassLoader()));
            this.logger.trace("JSR-330 'javax.inject.Inject' annotation found and supported for autowiring");
        } catch (ClassNotFoundException var2) {
        }

    } 
	 */
	// 不是java.开头的则进入else
    if (!AnnotationUtils.isCandidateClass(clazz, this.autowiredAnnotationTypes)) {
        return InjectionMetadata.EMPTY;
    } else {
        List<InjectedElement> elements = new ArrayList();
        Class targetClass = clazz;

        do {
            List<InjectedElement> currElements = new ArrayList();
            // 循环目标类所有字段
            ReflectionUtils.doWithLocalFields(targetClass, (field) -> {
            	// 查找该字段是否有autowiredAnnotationTypes中的某一个注解,根据其构造方法可知@Autowired、@Value、@Inject
                MergedAnnotation<?> ann = this.findAutowiredAnnotation(field);
                if (ann != null) {
                    if (Modifier.isStatic(field.getModifiers())) {
                        if (this.logger.isInfoEnabled()) {
                            this.logger.info("Autowired annotation is not supported on static fields: " + field);
                        }

                        return;
                    }

                    boolean required = this.determineRequiredStatus(ann);
                    // 将该字段存入集合，后续返回
                    currElements.add(new AutowiredAnnotationBeanPostProcessor.AutowiredFieldElement(field, required));
                }

            });
            // 循环目标类所有方法
            ReflectionUtils.doWithLocalMethods(targetClass, (method) -> {
                Method bridgedMethod = BridgeMethodResolver.findBridgedMethod(method);
                if (BridgeMethodResolver.isVisibilityBridgeMethodPair(method, bridgedMethod)) {
                    MergedAnnotation<?> ann = this.findAutowiredAnnotation(bridgedMethod);
                    if (ann != null && method.equals(ClassUtils.getMostSpecificMethod(method, clazz))) {
                        if (Modifier.isStatic(method.getModifiers())) {
                            if (this.logger.isInfoEnabled()) {
                                this.logger.info("Autowired annotation is not supported on static methods: " + method);
                            }

                            return;
                        }

                        if (method.getParameterCount() == 0 && this.logger.isInfoEnabled()) {
                            this.logger.info("Autowired annotation should only be used on methods with parameters: " + method);
                        }

                        boolean required = this.determineRequiredStatus(ann);
                        PropertyDescriptor pd = BeanUtils.findPropertyForMethod(bridgedMethod, clazz);
                        currElements.add(new AutowiredAnnotationBeanPostProcessor.AutowiredMethodElement(method, required, pd));
                    }

                }
            });
            elements.addAll(0, currElements);
            targetClass = targetClass.getSuperclass();
            // 循环遍历父类
        } while(targetClass != null && targetClass != Object.class);

        return InjectionMetadata.forElements(elements, clazz);
    }
}
```
## 4.2postProcessProperties
```java
public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) {
	// 因为之前扫描过，这里可以直接从缓存中拿到值
    InjectionMetadata metadata = this.findAutowiringMetadata(beanName, bean.getClass(), pvs);

    try {
        metadata.inject(bean, beanName, pvs);
        return pvs;
    } catch (BeanCreationException var6) {
        throw var6;
    } catch (Throwable var7) {
        throw new BeanCreationException(beanName, "Injection of autowired dependencies failed", var7);
    }
}
```

## 4.2.1inject
```java
public void inject(Object target, @Nullable String beanName, @Nullable PropertyValues pvs) throws Throwable {
	Collection<InjectedElement> checkedElements = this.checkedElements;
	Collection<InjectedElement> elementsToIterate =
			(checkedElements != null ? checkedElements : this.injectedElements);
	if (!elementsToIterate.isEmpty()) {
		// 循环
		for (InjectedElement element : elementsToIterate) {
			if (logger.isTraceEnabled()) {
				logger.trace("Processing injected element of bean '" + beanName + "': " + element);
			}
			// 核心方法，注意，这里element会不一样，要找自己的，比如字段就是AnnotatedFieldElement
			element.inject(target, beanName, pvs);
		}
	}
}
```

## 4.2.2inject
```java
@Override
protected void inject(Object bean, @Nullable String beanName, @Nullable PropertyValues pvs) throws Throwable {
	Field field = (Field) this.member;
	Object value;
	if (this.cached) {
		value = resolvedCachedArgument(beanName, this.cachedFieldValue);
	}
	else {
		// 将field和其它属性封装成统一对象，method也用这个对象
		DependencyDescriptor desc = new DependencyDescriptor(field, this.required);
		desc.setContainingClass(bean.getClass());
		Set<String> autowiredBeanNames = new LinkedHashSet<>(1);
		Assert.state(beanFactory != null, "No BeanFactory available");
		TypeConverter typeConverter = beanFactory.getTypeConverter();
		try {
			// 核心方法
			value = beanFactory.resolveDependency(desc, beanName, autowiredBeanNames, typeConverter);
		}
		catch (BeansException ex) {
			throw new UnsatisfiedDependencyException(null, beanName, new InjectionPoint(field), ex);
		}
		synchronized (this) {
			if (!this.cached) {
				if (value != null || this.required) {
					this.cachedFieldValue = desc;
					registerDependentBeans(beanName, autowiredBeanNames);
					if (autowiredBeanNames.size() == 1) {
						String autowiredBeanName = autowiredBeanNames.iterator().next();
						if (beanFactory.containsBean(autowiredBeanName) &&
								beanFactory.isTypeMatch(autowiredBeanName, field.getType())) {
							this.cachedFieldValue = new ShortcutDependencyDescriptor(
									desc, autowiredBeanName, field.getType());
						}
					}
				}
				else {
					this.cachedFieldValue = null;
				}
				this.cached = true;
			}
		}
	}
	if (value != null) {
		ReflectionUtils.makeAccessible(field);
		field.set(bean, value);
	}
}
```

## 4.3resolveDependency
```java
public Object resolveDependency(DependencyDescriptor descriptor, @Nullable String requestingBeanName,
			@Nullable Set<String> autowiredBeanNames, @Nullable TypeConverter typeConverter) throws BeansException {

	// 注意该方法传进来的有可能是field也有可能是method
	// 获取method参数名称的发现器
	descriptor.initParameterNameDiscovery(getParameterNameDiscoverer());
	// 处理optional的特殊分支
	if (Optional.class == descriptor.getDependencyType()) {
		return createOptionalDependency(descriptor, requestingBeanName);
	}
	// 处理ObjectFactory的特殊分支
	else if (ObjectFactory.class == descriptor.getDependencyType() ||
			ObjectProvider.class == descriptor.getDependencyType()) {
		return new DependencyObjectProvider(descriptor, requestingBeanName);
	}
	// 处理javaxInjectProviderClass的特殊分支
	else if (javaxInjectProviderClass == descriptor.getDependencyType()) {
		return new Jsr330Factory().createDependencyProvider(descriptor, requestingBeanName);
	}
	else {
		// @lazy分支返回一个代理对象
		Object result = getAutowireCandidateResolver().getLazyResolutionProxyIfNecessary(
				descriptor, requestingBeanName);
		// 正常分支
		if (result == null) {
			result = doResolveDependency(descriptor, requestingBeanName, autowiredBeanNames, typeConverter);
		}
		return result;
	}
}
```

## 4.4doResolveDependency
```java
public Object doResolveDependency(DependencyDescriptor descriptor, @Nullable String beanName,
			@Nullable Set<String> autowiredBeanNames, @Nullable TypeConverter typeConverter) throws BeansException {

	InjectionPoint previousInjectionPoint = ConstructorResolver.setCurrentInjectionPoint(descriptor);
	try {
		// 获取缓存
		Object shortcut = descriptor.resolveShortcut(this);
		if (shortcut != null) {
			return shortcut;
		}

		Class<?> type = descriptor.getDependencyType();
		// 获取@Value
		Object value = getAutowireCandidateResolver().getSuggestedValue(descriptor);
		if (value != null) {
			if (value instanceof String) {
				String strVal = resolveEmbeddedValue((String) value);
				BeanDefinition bd = (beanName != null && containsBean(beanName) ?
						getMergedBeanDefinition(beanName) : null);
				// 解析SPEL
				value = evaluateBeanDefinitionString(strVal, bd);
			}
			// 调用类型转换器
			TypeConverter converter = (typeConverter != null ? typeConverter : getTypeConverter());
			try {
				return converter.convertIfNecessary(value, type, descriptor.getTypeDescriptor());
			}
			catch (UnsupportedOperationException ex) {
				// A custom TypeConverter which does not support TypeDescriptor resolution...
				return (descriptor.getField() != null ?
						converter.convertIfNecessary(value, type, descriptor.getField()) :
						converter.convertIfNecessary(value, type, descriptor.getMethodParameter()));
			}
		}

		// 处理List、Map这种collections的对象，例如List<User>,map的key只能是string
		Object multipleBeans = resolveMultipleBeans(descriptor, beanName, autowiredBeanNames, typeConverter);
		if (multipleBeans != null) {
			return multipleBeans;
		}

		// 根据参数找bean,如果该bean没有被实例化则会返回class
		Map<String, Object> matchingBeans = findAutowireCandidates(beanName, type, descriptor);
		if (matchingBeans.isEmpty()) {
			if (isRequired(descriptor)) {
				// 如果没找到则报错
				raiseNoMatchingBeanFound(type, descriptor.getResolvableType(), descriptor);
			}
			return null;
		}

		String autowiredBeanName;
		Object instanceCandidate;

		if (matchingBeans.size() > 1) {
			// 如果找到多个，会根据byName去匹配(会先判断@primary注解、@Priority注解)
			autowiredBeanName = determineAutowireCandidate(matchingBeans, descriptor);
			if (autowiredBeanName == null) {
				if (isRequired(descriptor) || !indicatesMultipleBeans(type)) {
					return descriptor.resolveNotUnique(descriptor.getResolvableType(), matchingBeans);
				}
				else {
					// In case of an optional Collection/Map, silently ignore a non-unique case:
					// possibly it was meant to be an empty collection of multiple regular beans
					// (before 4.3 in particular when we didn't even look for collection beans).
					return null;
				}
			}
			instanceCandidate = matchingBeans.get(autowiredBeanName);
		}
		else {
			// 如果只找到一个
			// We have exactly one match.
			Map.Entry<String, Object> entry = matchingBeans.entrySet().iterator().next();
			autowiredBeanName = entry.getKey();
			instanceCandidate = entry.getValue();
		}

		if (autowiredBeanNames != null) {
			autowiredBeanNames.add(autowiredBeanName);
		}
		// 如果是class即表示还没创建，则进行创建
		if (instanceCandidate instanceof Class) {
			instanceCandidate = descriptor.resolveCandidate(autowiredBeanName, type, this);
		}
		Object result = instanceCandidate;
		// @Bean返回null的情况，会被创建成NullBean，接着会报错
		if (result instanceof NullBean) {
			if (isRequired(descriptor)) {
				raiseNoMatchingBeanFound(type, descriptor.getResolvableType(), descriptor);
			}
			result = null;
		}
		if (!ClassUtils.isAssignableValue(type, result)) {
			throw new BeanNotOfRequiredTypeException(autowiredBeanName, type, instanceCandidate.getClass());
		}
		return result;
	}
	finally {
		ConstructorResolver.setCurrentInjectionPoint(previousInjectionPoint);
	}
}
```
## 4.5findAutowireCandidates
```java
protected Map<String, Object> findAutowireCandidates(
			@Nullable String beanName, Class<?> requiredType, DependencyDescriptor descriptor) {

	// 根据类型找名字
	String[] candidateNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
			this, requiredType, true, descriptor.isEager());
	Map<String, Object> result = new LinkedHashMap<>(candidateNames.length);
	// spring自己的对象非用户定义的返回
	for (Map.Entry<Class<?>, Object> classObjectEntry : this.resolvableDependencies.entrySet()) {
		Class<?> autowiringType = classObjectEntry.getKey();
		if (autowiringType.isAssignableFrom(requiredType)) {
			Object autowiringValue = classObjectEntry.getValue();
			autowiringValue = AutowireUtils.resolveAutowiringValue(autowiringValue, requiredType);
			if (requiredType.isInstance(autowiringValue)) {
				result.put(ObjectUtils.identityToString(autowiringValue), autowiringValue);
				break;
			}
		}
	}
	for (String candidate : candidateNames) {
		// 除去自己，判断autowireCandicate=false，beanClass是否匹配，@Qualifer是否匹配
		if (!isSelfReference(beanName, candidate) && isAutowireCandidate(candidate, descriptor)) {
			addCandidateEntry(result, candidate, descriptor, requiredType);
		}
	}
	if (result.isEmpty()) {
		// 如果没匹配到
		boolean multiple = indicatesMultipleBeans(requiredType);
		// Consider fallback matches if the first pass failed to find anything...
		DependencyDescriptor fallbackDescriptor = descriptor.forFallbackMatch();
		for (String candidate : candidateNames) {
			if (!isSelfReference(beanName, candidate) && isAutowireCandidate(candidate, fallbackDescriptor) &&
					(!multiple || getAutowireCandidateResolver().hasQualifier(descriptor))) {
				addCandidateEntry(result, candidate, descriptor, requiredType);
			}
		}
		// 看一下是不是自己注入自己的情况
		if (result.isEmpty() && !multiple) {
			// Consider self references as a final pass...
			// but in the case of a dependency collection, not the very same bean itself.
			for (String candidate : candidateNames) {
				if (isSelfReference(beanName, candidate) &&
						(!(descriptor instanceof MultiElementDescriptor) || !beanName.equals(candidate)) &&
						isAutowireCandidate(candidate, fallbackDescriptor)) {
					addCandidateEntry(result, candidate, descriptor, requiredType);
				}
			}
		}
	}
	return result;
}
```

需要注意的是
MergedBeanDefinitionPostProcessor.postProcessMergedBeanDefinition()虽然先执行但是后赋值，所以会覆盖autowired注解的值