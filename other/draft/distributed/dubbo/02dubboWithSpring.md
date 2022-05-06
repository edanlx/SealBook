@DubboService
会生成两个bean对象，第一个是org.apache.dubbo.config.spring.ServiceBean,里面父类包含dubbo的所有相关属性，另一个是XXXimpl正常对象



@EnableDubbo里面有@EnableDubboConfig和@DubboComponentScan，分别@Imoort了DubboConfigConfigurationRegistrar、DubboComponentScanRegistrar

旧版Config的是先是使用registBeanDefinition。然后postProcessBeforeInitialization中使用beanName.equals进行判断并生成相应的bean，在afterP中赋值其他属性


## 4.org.apache.dubbo.config.spring.context.annotation.DubboComponentScanRegistrar#registerBeanDefinitions
```java
public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {

    // @since 2.7.6 Register the common beans
    // 此处会注册一个关键bean,ReferenceAnnotationBeanPostProcessor，用于处理@DubboService的注入，其实现BeanFactoryPostProcessor会在spring的doScan时回调
    // 如果是旧版该bean会在此处直接注册
    registerCommonBeans(registry);

    // 获取扫描路径，如果没有设置则把当前importingClassMetadata的包路径作为扫描路径
    Set<String> packagesToScan = getPackagesToScan(importingClassMetadata);

    // 处理注解
    registerServiceAnnotationPostProcessor(packagesToScan, registry);
}
```
## 4.1org.apache.dubbo.config.spring.context.annotation.DubboComponentScanRegistrar#registerServiceAnnotationPostProcessor
```java
private void registerServiceAnnotationPostProcessor(Set<String> packagesToScan, BeanDefinitionRegistry registry) {
    // 获取一个bean工厂的后置处理器，名称有误。即实现BeanDefinitionRegistryPostProcessor
    BeanDefinitionBuilder builder = rootBeanDefinition(ServiceAnnotationPostProcessor.class);
    builder.addConstructorArgValue(packagesToScan);
    builder.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
    AbstractBeanDefinition beanDefinition = builder.getBeanDefinition();
    BeanDefinitionReaderUtils.registerWithGeneratedName(beanDefinition, registry);

}
```
#### 4.1.1org.apache.dubbo.config.spring.beans.factory.annotation.ServiceAnnotationPostProcessor#postProcessBeanDefinitionRegistry
```java
 public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
    this.registry = registry;
    // 将对象转为字符串
    Set<String> resolvedPackagesToScan = resolvePackagesToScan(packagesToScan);

    if (!CollectionUtils.isEmpty(resolvedPackagesToScan)) {
        scanServiceBeans(resolvedPackagesToScan, registry);
    } else {
        if (logger.isWarnEnabled()) {
            logger.warn("packagesToScan is empty , ServiceBean registry will be ignored!");
        }
    }

}
```
- org.apache.dubbo.config.spring.beans.factory.annotation.ServiceAnnotationPostProcessor#scanServiceBeans
```java
private void scanServiceBeans(Set<String> packagesToScan, BeanDefinitionRegistry registry) {

    DubboClassPathBeanDefinitionScanner scanner =
            new DubboClassPathBeanDefinitionScanner(registry, environment, resourceLoader);

    BeanNameGenerator beanNameGenerator = resolveBeanNameGenerator(registry);
    scanner.setBeanNameGenerator(beanNameGenerator);
    for (Class<? extends Annotation> annotationType : serviceAnnotationTypes) {
        // DubboService,dubbo下的service,apache下的service
        scanner.addIncludeFilter(new AnnotationTypeFilter(annotationType));
    }

    ScanExcludeFilter scanExcludeFilter = new ScanExcludeFilter();
    scanner.addExcludeFilter(scanExcludeFilter);

    for (String packageToScan : packagesToScan) {

        // avoid duplicated scans
        if (servicePackagesHolder.isPackageScanned(packageToScan)) {
            if (logger.isInfoEnabled()) {
                logger.info("Ignore package who has already bean scanned: " + packageToScan);
            }
            continue;
        }

        // Registers @Service Bean first
        scanner.scan(packageToScan);

        // Finds all BeanDefinitionHolders of @Service whether @ComponentScan scans or not.
        // 获取扫描到的类，和BeanDefinition格式类似,同时已经注册到spring容器中(和mybaties不同这里是有具体实现类可直接注册)
        Set<BeanDefinitionHolder> beanDefinitionHolders =
                findServiceBeanDefinitionHolders(scanner, packageToScan, registry, beanNameGenerator);

        if (!CollectionUtils.isEmpty(beanDefinitionHolders)) {
            if (logger.isInfoEnabled()) {
                List<String> serviceClasses = new ArrayList<>(beanDefinitionHolders.size());
                for (BeanDefinitionHolder beanDefinitionHolder : beanDefinitionHolders) {
                    serviceClasses.add(beanDefinitionHolder.getBeanDefinition().getBeanClassName());
                }
                logger.info("Found " + beanDefinitionHolders.size() + " classes annotated by Dubbo @Service under package [" + packageToScan + "]: " + serviceClasses);
            }

            for (BeanDefinitionHolder beanDefinitionHolder : beanDefinitionHolders) {
                processScannedBeanDefinition(beanDefinitionHolder, registry, scanner);
                servicePackagesHolder.addScannedClass(beanDefinitionHolder.getBeanDefinition().getBeanClassName());
            }
        } else {
            if (logger.isWarnEnabled()) {
                logger.warn("No class annotated by Dubbo @Service was found under package ["
                        + packageToScan + "], ignore re-scanned classes: " + scanExcludeFilter.getExcludedCount());
            }
        }

        servicePackagesHolder.addScannedPackage(packageToScan);
    }
}
```
##### 4.1.1.1org.apache.dubbo.config.spring.beans.factory.annotation.ServiceAnnotationPostProcessor#processScannedBeanDefinition
```java
private void processScannedBeanDefinition(BeanDefinitionHolder beanDefinitionHolder, BeanDefinitionRegistry registry,
                                              DubboClassPathBeanDefinitionScanner scanner) {

    Class<?> beanClass = resolveClass(beanDefinitionHolder);

    // 获取@DubboService注解信息
    Annotation service = findServiceAnnotation(beanClass);

    // The attributes of @Service annotation
    Map<String, Object> serviceAnnotationAttributes = AnnotationUtils.getAttributes(service, true);

    // 服务实现类对应的接口
    Class<?> interfaceClass = resolveServiceInterfaceClass(serviceAnnotationAttributes, beanClass);

    String annotatedServiceBeanName = beanDefinitionHolder.getBeanName();

    // ServiceBean Bean name
    // 生成serviceBean的名称
    String beanName = generateServiceBeanName(serviceAnnotationAttributes, interfaceClass);

    // 构建BeanDefinition
    AbstractBeanDefinition serviceBeanDefinition =
            buildServiceBeanDefinition(serviceAnnotationAttributes, interfaceClass, annotatedServiceBeanName);
    // 注册BeanDefinition
    registerServiceBeanDefinition(beanName, serviceBeanDefinition, interfaceClass);

}
```
- org.apache.dubbo.config.spring.beans.factory.annotation.ServiceAnnotationPostProcessor#buildServiceBeanDefinition
```java
private AbstractBeanDefinition buildServiceBeanDefinition(Map<String, Object> serviceAnnotationAttributes,
                                                              Class<?> interfaceClass,
                                                              String refServiceBeanName) {
    // 生成ServiceBean的BeanDefinition
    BeanDefinitionBuilder builder = rootBeanDefinition(ServiceBean.class);

    AbstractBeanDefinition beanDefinition = builder.getBeanDefinition();
    // 将注解上的信息赋值到ServiceBean中
    MutablePropertyValues propertyValues = beanDefinition.getPropertyValues();

    String[] ignoreAttributeNames = of("provider", "monitor", "application", "module", "registry", "protocol",
            "interface", "interfaceName", "parameters");
    propertyValues.addPropertyValues(new AnnotationPropertyValuesAdapter(serviceAnnotationAttributes, environment, ignoreAttributeNames));

    //set config id, for ConfigManager cache key
    //builder.addPropertyValue("id", beanName);
    // References "ref" property to annotated-@Service Bean
    // 将ref属性关联到写@DubboService生成的实现类
    addPropertyReference(builder, "ref", refServiceBeanName);
    // Set interface
    builder.addPropertyValue("interface", interfaceClass.getName());
    // Convert parameters into map
    builder.addPropertyValue("parameters", convertParameters((String[]) serviceAnnotationAttributes.get("parameters")));
    // Add methods parameters
    List<MethodConfig> methodConfigs = convertMethodConfigs(serviceAnnotationAttributes.get("methods"));
    if (!methodConfigs.isEmpty()) {
        builder.addPropertyValue("methods", methodConfigs);
    }

    // convert provider to providerIds
    String providerConfigId = (String) serviceAnnotationAttributes.get("provider");
    if (StringUtils.hasText(providerConfigId)) {
        addPropertyValue(builder, "providerIds", providerConfigId);
    }

    // Convert registry[] to registryIds
    String[] registryConfigIds = (String[]) serviceAnnotationAttributes.get("registry");
    if (registryConfigIds != null && registryConfigIds.length > 0) {
        resolveStringArray(registryConfigIds);
        builder.addPropertyValue("registryIds", StringUtils.join(registryConfigIds, ','));
    }

    // Convert protocol[] to protocolIds
    String[] protocolConfigIds = (String[]) serviceAnnotationAttributes.get("protocol");
    if (protocolConfigIds != null && protocolConfigIds.length > 0) {
        resolveStringArray(protocolConfigIds);
        builder.addPropertyValue("protocolIds", StringUtils.join(protocolConfigIds, ','));
    }

    // TODO Could we ignore these attributes: applicatin/monitor/module ? Use global config
    // monitor reference
    String monitorConfigId = (String) serviceAnnotationAttributes.get("monitor");
    if (StringUtils.hasText(monitorConfigId)) {
        addPropertyReference(builder, "monitor", monitorConfigId);
    }

    // application reference
    String applicationConfigId = (String) serviceAnnotationAttributes.get("application");
    if (StringUtils.hasText(applicationConfigId)) {
        addPropertyReference(builder, "application", applicationConfigId);
    }

    // module reference
    String moduleConfigId = (String) serviceAnnotationAttributes.get("module");
    if (StringUtils.hasText(moduleConfigId)) {
        addPropertyReference(builder, "module", moduleConfigId);
    }

    return builder.getBeanDefinition();

}
```
## 5.@DubboReference
### 5.1@DubboReference生成
ReferenceAnnotationBeanPostProcessor extends AbstractAnnotationBeanPostProcessor,而AbstractAnnotationBeanPostProcessor implements MergedBeanDefinitionPostProcessor
- org.apache.dubbo.config.spring.beans.factory.annotation.AbstractAnnotationBeanPostProcessor#postProcessMergedBeanDefinition
```java
public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName) {
    if (beanType != null) {
        AnnotatedInjectionMetadata metadata = findInjectionMetadata(beanName, beanType, null);
        metadata.checkConfigMembers(beanDefinition);
        try {
            // 核心方法
        /**
         * List<String> registeredReferenceBeanNames = referenceBeanManager.getByKey(referenceKey);
         * 判断bean中是否有
         * if (beanDefinitionRegistry.containsBeanDefinition(referenceBeanName)) {
                beanDefinition.setBeanClassName(ReferenceBean.class.getName());
            }
         * 
         * */
        // 生成相应的bean ，如果是旧版则在doGetInjectedBean中直接生成然后再缓存
            prepareInjection(metadata);
        } catch (Exception e) {
            logger.error("Prepare injection of @"+getAnnotationType().getSimpleName()+" failed", e);
        }
    }
}
```
#### 5.1.1org.apache.dubbo.config.spring.beans.factory.annotation.AbstractAnnotationBeanPostProcessor#postProcessMergedBeanDefinition
```java
protected void prepareInjection(AnnotatedInjectionMetadata metadata) throws BeansException {
    try {
        //find and registry bean definition for @DubboReference/@Reference
        // 循环属性是否有@DubboReference相关注解
        for (AnnotatedFieldElement fieldElement : metadata.getFieldElements()) {
            if (fieldElement.injectedObject != null) {
                continue;
            }
            Class<?> injectedType = fieldElement.field.getType();
            AnnotationAttributes attributes = fieldElement.attributes;
            // 生成相应的bean->beanDefinition.setBeanClassName(ReferenceBean.class.getName());
            String referenceBeanName = registerReferenceBean(fieldElement.getPropertyName(), injectedType, attributes, fieldElement.field);

            //associate fieldElement and reference bean
            fieldElement.injectedObject = referenceBeanName;
            injectedFieldReferenceBeanCache.put(fieldElement, referenceBeanName);

        }

        for (AnnotatedMethodElement methodElement : metadata.getMethodElements()) {
            if (methodElement.injectedObject != null) {
                continue;
            }
            Class<?> injectedType = methodElement.getInjectedType();
            AnnotationAttributes attributes = methodElement.attributes;
            String referenceBeanName = registerReferenceBean(methodElement.getPropertyName(), injectedType, attributes, methodElement.method);

            //associate fieldElement and reference bean
            methodElement.injectedObject = referenceBeanName;
            injectedMethodReferenceBeanCache.put(methodElement, referenceBeanName);
        }
    } catch (ClassNotFoundException e) {
        throw new BeanCreationException("prepare reference annotation failed", e);
    }
}
```
#### 5.1.2org.apache.dubbo.config.spring.ReferenceBean#getObject
```java
public Object getObject() {
    if (lazyProxy == null) {
        createLazyProxy();
    }
    return lazyProxy;
}
```
- org.apache.dubbo.config.spring.ReferenceBean#createLazyProxy
```java
private void createLazyProxy() {
    //set proxy interfaces
    //see also: org.apache.dubbo.rpc.proxy.AbstractProxyFactory.getProxy(org.apache.dubbo.rpc.Invoker<T>, boolean)
    ProxyFactory proxyFactory = new ProxyFactory();
    // 原始类来源
    proxyFactory.setTargetSource(new DubboReferenceLazyInitTargetSource());
    proxyFactory.addInterface(interfaceClass);
    Class<?>[] internalInterfaces = AbstractProxyFactory.getInternalInterfaces();
    for (Class<?> anInterface : internalInterfaces) {
        proxyFactory.addInterface(anInterface);
    }
    if (!StringUtils.isEquals(interfaceClass.getName(), interfaceName)) {
        //add service interface
        try {
            Class<?> serviceInterface = ClassUtils.forName(interfaceName, beanClassLoader);
            proxyFactory.addInterface(serviceInterface);
        } catch (ClassNotFoundException e) {
            // generic call maybe without service interface class locally
        }
    }

    this.lazyProxy = proxyFactory.getProxy(this.beanClassLoader);
}
```
- org.apache.dubbo.config.spring.ReferenceBean.DubboReferenceLazyInitTargetSource#createObject
```java
protected Object createObject() throws Exception {
    return getCallProxy();
}
```
```java
private Object getCallProxy() throws Exception {
    if (referenceConfig == null) {
        throw new IllegalStateException("ReferenceBean is not ready yet, please make sure to call reference interface method after dubbo is started.");
    }
    //get reference proxy
    // ReferenceConfigCache是dubbo包里的了
    return ReferenceConfigCache.getCache().get(referenceConfig);
}
```
- org.apache.dubbo.config.utils.ReferenceConfigCache#get(org.apache.dubbo.config.ReferenceConfigBase<T>)
```java
public <T> T get(ReferenceConfigBase<T> referenceConfig) {
    String key = generator.generateKey(referenceConfig);
    Class<?> type = referenceConfig.getInterfaceClass();

    proxies.computeIfAbsent(type, _t -> new ConcurrentHashMap<>());

    ConcurrentMap<String, Object> proxiesOfType = proxies.get(type);
    // 如果没有则创建
    proxiesOfType.computeIfAbsent(key, _k -> {
        Object proxy = referenceConfig.get();
        referredReferences.put(key, referenceConfig);
        return proxy;
    });

    return (T) proxiesOfType.get(key);
}
```
### 5.2@DubboReference赋值
AbstractAnnotationBeanPostProcessor extends InstantiationAwareBeanPostProcessorAdapter。而InstantiationAwareBeanPostProcessorAdapter一直往上找则有InstantiationAwareBeanPostProcessor。spring会在属性赋值时调用postProcessPropertyValues
- org.apache.dubbo.config.spring.beans.factory.annotation.AbstractAnnotationBeanPostProcessor#postProcessPropertyValues
```java
public PropertyValues postProcessPropertyValues(
        PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName) throws BeansException {

    try {
        // 查找该bean的所有@DubboRefference等的注入点
        AnnotatedInjectionMetadata metadata = findInjectionMetadata(beanName, bean.getClass(), pvs);
        prepareInjection(metadata);
        // 反射赋值
        metadata.inject(bean, beanName, pvs);
    } catch (BeansException ex) {
        throw ex;
    } catch (Throwable ex) {
        throw new BeanCreationException(beanName, "Injection of @" + getAnnotationType().getSimpleName()
                + " dependencies is failed", ex);
    }
    return pvs;
}
```
### 5.2.1org.springframework.beans.factory.annotation.InjectionMetadata#inject
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
            element.inject(target, beanName, pvs);
        }
    }
}
```
- org.apache.dubbo.config.spring.beans.factory.annotation.AbstractAnnotationBeanPostProcessor.AnnotatedInjectElement#inject
```java
protected void inject(Object bean, String beanName, PropertyValues pvs) throws Throwable {
    Object injectedObject = getInjectedObject(attributes, bean, beanName, getInjectedType(), this);

    if (member instanceof Field) {
        Field field = (Field) member;
        ReflectionUtils.makeAccessible(field);
        field.set(bean, injectedObject);
    } else if (member instanceof Method) {
        Method method = (Method) member;
        ReflectionUtils.makeAccessible(method);
        method.invoke(bean, injectedObject);
    }
}
```
- org.apache.dubbo.config.spring.beans.factory.annotation.AbstractAnnotationBeanPostProcessor#getInjectedObject
```java
protected Object getInjectedObject(AnnotationAttributes attributes, Object bean, String beanName, Class<?> injectedType,
                                       AnnotatedInjectElement injectedElement) throws Exception {
        return doGetInjectedBean(attributes, bean, beanName, injectedType, injectedElement);
    }
```
- org.apache.dubbo.config.spring.beans.factory.annotation.ReferenceAnnotationBeanPostProcessor#doGetInjectedBean
```java
protected Object doGetInjectedBean(AnnotationAttributes attributes, Object bean, String beanName, Class<?> injectedType,
                                       AnnotatedInjectElement injectedElement) throws Exception {

    if (injectedElement.injectedObject == null) {
        throw new IllegalStateException("The AnnotatedInjectElement of @DubboReference should be inited before injection");
    }
    // 从factoryBean中获取对象
    return getBeanFactory().getBean((String) injectedElement.injectedObject);
}
```