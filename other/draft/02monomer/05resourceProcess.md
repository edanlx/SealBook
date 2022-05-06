# 【JAVA框架】05-spring的resource注入流程
> 欢迎关注b站账号/公众号【六边形战士夏宁】，一个要把各项指标拉满的男人。该文章已在[github目录](https://github.com/edanlx/SealBook/blob/master/catalogue/wechat.md)收录。
屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。

## 1.背景
@resource注解作为spring最重要的注解之一，其核心原理如何赋值
## 2.整体流程
1. postProcessMergedBeanDefinition流程先扫描然后将符合条件的@Resource
2. postProcessProperties进行回调，如果没初始化则初始化
3. CommonAnnotationBeanPostProcessor类中进行反射赋值反射赋值
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
CommonAnnotationBeanPostProcessor
### 4.1postProcessMergedBeanDefinition.postProcessMergedBeanDefinition()
```java
public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName) {
	super.postProcessMergedBeanDefinition(beanDefinition, beanType, beanName);
	// 核心方法
	InjectionMetadata metadata = findResourceMetadata(beanName, beanType, null);
	metadata.checkConfigMembers(beanDefinition);
}
```

#### 4.1.1findResourceMetadata
```java
private InjectionMetadata findResourceMetadata(String beanName, final Class<?> clazz, @Nullable PropertyValues pvs) {
	// Fall back to class name as cache key, for backwards compatibility with custom callers.
	String cacheKey = (StringUtils.hasLength(beanName) ? beanName : clazz.getName());
	// Quick check on the concurrent map first, with minimal locking.
	InjectionMetadata metadata = this.injectionMetadataCache.get(cacheKey);
	if (InjectionMetadata.needsRefresh(metadata, clazz)) {
		synchronized (this.injectionMetadataCache) {
			metadata = this.injectionMetadataCache.get(cacheKey);
			if (InjectionMetadata.needsRefresh(metadata, clazz)) {
				if (metadata != null) {
					metadata.clear(pvs);
				}
				// 核心方法
				metadata = buildResourceMetadata(clazz);
				this.injectionMetadataCache.put(cacheKey, metadata);
			}
		}
	}
	return metadata;
}
```

### 4.1.2InjectionMetadata
```java
private InjectionMetadata buildResourceMetadata(final Class<?> clazz) {
	if (!AnnotationUtils.isCandidateClass(clazz, resourceAnnotationTypes)) {
		return InjectionMetadata.EMPTY;
	}

	List<InjectionMetadata.InjectedElement> elements = new ArrayList<>();
	Class<?> targetClass = clazz;

	do {
		final List<InjectionMetadata.InjectedElement> currElements = new ArrayList<>();

		ReflectionUtils.doWithLocalFields(targetClass, field -> {
			if (webServiceRefClass != null && field.isAnnotationPresent(webServiceRefClass)) {
				if (Modifier.isStatic(field.getModifiers())) {
					throw new IllegalStateException("@WebServiceRef annotation is not supported on static fields");
				}
				currElements.add(new WebServiceRefElement(field, field, null));
			}
			else if (ejbClass != null && field.isAnnotationPresent(ejbClass)) {
				if (Modifier.isStatic(field.getModifiers())) {
					throw new IllegalStateException("@EJB annotation is not supported on static fields");
				}
				currElements.add(new EjbRefElement(field, field, null));
			}
			// 此处扫描@Resource
			else if (field.isAnnotationPresent(Resource.class)) {
				if (Modifier.isStatic(field.getModifiers())) {
					throw new IllegalStateException("@Resource annotation is not supported on static fields");
				}
				if (!this.ignoredResourceTypes.contains(field.getType().getName())) {
					currElements.add(new ResourceElement(field, field, null));
				}
			}
		});

		ReflectionUtils.doWithLocalMethods(targetClass, method -> {
			Method bridgedMethod = BridgeMethodResolver.findBridgedMethod(method);
			if (!BridgeMethodResolver.isVisibilityBridgeMethodPair(method, bridgedMethod)) {
				return;
			}
			if (method.equals(ClassUtils.getMostSpecificMethod(method, clazz))) {
				if (webServiceRefClass != null && bridgedMethod.isAnnotationPresent(webServiceRefClass)) {
					if (Modifier.isStatic(method.getModifiers())) {
						throw new IllegalStateException("@WebServiceRef annotation is not supported on static methods");
					}
					if (method.getParameterCount() != 1) {
						throw new IllegalStateException("@WebServiceRef annotation requires a single-arg method: " + method);
					}
					PropertyDescriptor pd = BeanUtils.findPropertyForMethod(bridgedMethod, clazz);
					currElements.add(new WebServiceRefElement(method, bridgedMethod, pd));
				}
				else if (ejbClass != null && bridgedMethod.isAnnotationPresent(ejbClass)) {
					if (Modifier.isStatic(method.getModifiers())) {
						throw new IllegalStateException("@EJB annotation is not supported on static methods");
					}
					if (method.getParameterCount() != 1) {
						throw new IllegalStateException("@EJB annotation requires a single-arg method: " + method);
					}
					PropertyDescriptor pd = BeanUtils.findPropertyForMethod(bridgedMethod, clazz);
					currElements.add(new EjbRefElement(method, bridgedMethod, pd));
				}
				else if (bridgedMethod.isAnnotationPresent(Resource.class)) {
					if (Modifier.isStatic(method.getModifiers())) {
						throw new IllegalStateException("@Resource annotation is not supported on static methods");
					}
					Class<?>[] paramTypes = method.getParameterTypes();
					if (paramTypes.length != 1) {
						throw new IllegalStateException("@Resource annotation requires a single-arg method: " + method);
					}
					if (!this.ignoredResourceTypes.contains(paramTypes[0].getName())) {
						PropertyDescriptor pd = BeanUtils.findPropertyForMethod(bridgedMethod, clazz);
						currElements.add(new ResourceElement(method, bridgedMethod, pd));
					}
				}
			}
		});

		elements.addAll(0, currElements);
		targetClass = targetClass.getSuperclass();
	}
	while (targetClass != null && targetClass != Object.class);

	return InjectionMetadata.forElements(elements, clazz);
}
```
## 4.2postProcessProperties
```java
public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) {
	InjectionMetadata metadata = findResourceMetadata(beanName, bean.getClass(), pvs);
	try {
		// 核心方法
		metadata.inject(bean, beanName, pvs);
	}
	catch (Throwable ex) {
		throw new BeanCreationException(beanName, "Injection of resource dependencies failed", ex);
	}
	return pvs;
}
```

## 4.2.1inject
```java
public void inject(Object target, @Nullable String beanName, @Nullable PropertyValues pvs) throws Throwable {
	Collection<InjectedElement> checkedElements = this.checkedElements;
	Collection<InjectedElement> elementsToIterate =
			(checkedElements != null ? checkedElements : this.injectedElements);
	if (!elementsToIterate.isEmpty()) {
		for (InjectedElement element : elementsToIterate) {
			if (logger.isTraceEnabled()) {
				logger.trace("Processing injected element of bean '" + beanName + "': " + element);
			}
			// 核心方法
			element.inject(target, beanName, pvs);
		}
	}
}
```

## 4.2.2inject
```java
protected void inject(Object target, @Nullable String requestingBeanName, @Nullable PropertyValues pvs)
				throws Throwable {

	if (this.isField) {
		Field field = (Field) this.member;
		ReflectionUtils.makeAccessible(field);
		// 核心方法
		field.set(target, getResourceToInject(target, requestingBeanName));
	}
	else {
		if (checkPropertySkipping(pvs)) {
			return;
		}
		try {
			Method method = (Method) this.member;
			ReflectionUtils.makeAccessible(method);
			method.invoke(target, getResourceToInject(target, requestingBeanName));
		}
		catch (InvocationTargetException ex) {
			throw ex.getTargetException();
		}
	}
}
```

## 4.3getResourceToInject
```java
protected Object getResourceToInject(Object target, @Nullable String requestingBeanName) {
	// 判断懒加载
	return (this.lazyLookup ? buildLazyResourceProxy(this, requestingBeanName) :
			getResource(this, requestingBeanName));
}
```

## 4.4getResource
```java
protected Object getResource(LookupElement element, @Nullable String requestingBeanName)
			throws NoSuchBeanDefinitionException {

	if (StringUtils.hasLength(element.mappedName)) {
		return this.jndiFactory.getBean(element.mappedName, element.lookupType);
	}
	if (this.alwaysUseJndiLookup) {
		return this.jndiFactory.getBean(element.name, element.lookupType);
	}
	if (this.resourceFactory == null) {
		throw new NoSuchBeanDefinitionException(element.lookupType,
				"No resource factory configured - specify the 'resourceFactory' property");
	}
	// 核心方法
	return autowireResource(this.resourceFactory, element, requestingBeanName);
}
```
## 4.5findAutowireCandidates
```java
protected Object autowireResource(BeanFactory factory, LookupElement element, @Nullable String requestingBeanName)
			throws NoSuchBeanDefinitionException {

	Object resource;
	Set<String> autowiredBeanNames;
	// 即@Resource指定的name,如果没有指定就是field的name
	String name = element.name;

	if (factory instanceof AutowireCapableBeanFactory) {
		AutowireCapableBeanFactory beanFactory = (AutowireCapableBeanFactory) factory;
		DependencyDescriptor descriptor = element.getDependencyDescriptor();
		// element.isDefaultName，如果指定了name则该值为false走else，如果没有指定则为true判断是否在工厂中(不在则和autowired一样先type,在则进入else)
		if (this.fallbackToDefaultTypeMatch && element.isDefaultName && !factory.containsBean(name)) {
			autowiredBeanNames = new LinkedHashSet<>();
			resource = beanFactory.resolveDependency(descriptor, requestingBeanName, autowiredBeanNames, null);
			if (resource == null) {
				throw new NoSuchBeanDefinitionException(element.getLookupType(), "No resolvable resource object");
			}
		}
		else {
			resource = beanFactory.resolveBeanByName(name, descriptor);
			autowiredBeanNames = Collections.singleton(name);
		}
	}
	else {
		resource = factory.getBean(name, element.lookupType);
		autowiredBeanNames = Collections.singleton(name);
	}

	if (factory instanceof ConfigurableBeanFactory) {
		ConfigurableBeanFactory beanFactory = (ConfigurableBeanFactory) factory;
		for (String autowiredBeanName : autowiredBeanNames) {
			if (requestingBeanName != null && beanFactory.containsBean(autowiredBeanName)) {
				beanFactory.registerDependentBean(autowiredBeanName, requestingBeanName);
			}
		}
	}

	return resource;
}
```