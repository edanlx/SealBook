# 【框架类源码】01-spring扫描BeanDefinition
> 欢迎关注b站账号/公众号【六边形战士夏宁】，一个要把各项指标拉满的男人。该文章已在[github目录](https://github.com/edanlx/SealBook/blob/master/catalogue/wechat.md)收录。
屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。

## 1.背景
spring扫描包注册成BeanDefinition过程
## 2.扫描整体流程
![pic](http://seal_li.gitee.io/sealbook/pic/02monmoer_01canBean.png)
## 3.源码过程 
### 方法1
```java
public int scan(String... basePackages) {
	int beanCountAtScanStart = this.registry.getBeanDefinitionCount();
	// 该方法为主方法，转入方法2
	doScan(basePackages);

	// Register annotation config processors, if necessary.
	if (this.includeAnnotationConfig) {
		AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
	}

	return (this.registry.getBeanDefinitionCount() - beanCountAtScanStart);
}
```

### 方法2
```java
protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
	Assert.notEmpty(basePackages, "At least one base package must be specified");
	Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();
	// 循环包的扫描路径
	for (String basePackage : basePackages) {
		// 注意此处返回BeanDefinition为核心方法，转入方法3
		Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
		// 遍历BeanDefinition开始解析其中的属性
		for (BeanDefinition candidate : candidates) {
			// 解析除了beanName外的数据，例如是否单例，是否懒加载
			ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
			candidate.setScope(scopeMetadata.getScopeName());
			// 生成bean的名字，component重写名字或自动生成名字
			String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
			if (candidate instanceof AbstractBeanDefinition) {
				// 赋默认值，比如懒加载等
				postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
			}
			if (candidate instanceof AnnotatedBeanDefinition) {
				// 根据注解重写上述默认值@Lazy、@Primary、@DependsOn、@Role、@Description
				AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
			}
			// 检查spring中是否有重复beanName,同时根据beanDefinitionMap检查是否是重复扫描已经注册过了
			if (checkCandidate(beanName, candidate)) {
				BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
				definitionHolder =
						AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
				beanDefinitions.add(definitionHolder);
				// 注册BeanDefinition进入方法10
				registerBeanDefinition(definitionHolder, this.registry);
			}
		}
	}
	return beanDefinitions;
}
```
### 方法3
```java
public Set<BeanDefinition> findCandidateComponents(String basePackage) {
	// 根据spring.components的定义进行扫描加速，一般用不到，除非文件特别多
	if (this.componentsIndex != null && indexSupportsIncludeFilters()) {
		return addCandidateComponentsFromIndex(this.componentsIndex, basePackage);
	}
	else {
		// 主方法进入方法4
		return scanCandidateComponents(basePackage);
	}
}
```
### 方法4
```java
private Set<BeanDefinition> scanCandidateComponents(String basePackage) {
	Set<BeanDefinition> candidates = new LinkedHashSet<>();
	try {
		String packageSearchPath = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX +
				resolveBasePackage(basePackage) + '/' + this.resourcePattern;
		// spring的工具类以ant格式加载该表达式下所有的方法
		Resource[] resources = getResourcePatternResolver().getResources(packageSearchPath);
		boolean traceEnabled = logger.isTraceEnabled();
		boolean debugEnabled = logger.isDebugEnabled();
		for (Resource resource : resources) {
			if (traceEnabled) {
				logger.trace("Scanning " + resource);
			}
			if (resource.isReadable()) {
				try {
					// 获取元数据
					MetadataReader metadataReader = getMetadataReaderFactory().getMetadataReader(resource);
					// 判断该元数据是否纳入最终bean定义 进入方法5
					if (isCandidateComponent(metadataReader)) {
						// 构造BeanDefinition 进入方法9
						ScannedGenericBeanDefinition sbd = new ScannedGenericBeanDefinition(metadataReader);
						sbd.setSource(resource);
						// 判断lookup注解、该类是否为接口、抽象类、子类等
						if (isCandidateComponent(sbd)) {
							if (debugEnabled) {
								logger.debug("Identified candidate component class: " + resource);
							}
							candidates.add(sbd);
						}
						else {
							if (debugEnabled) {
								logger.debug("Ignored because not a concrete top-level class: " + resource);
							}
						}
					}
					else {
						if (traceEnabled) {
							logger.trace("Ignored because not matching any filter: " + resource);
						}
					}
				}
				catch (Throwable ex) {
					throw new BeanDefinitionStoreException(
							"Failed to read candidate component class: " + resource, ex);
				}
			}
			else {
				if (traceEnabled) {
					logger.trace("Ignored because not readable: " + resource);
				}
			}
		}
	}
	catch (IOException ex) {
		throw new BeanDefinitionStoreException("I/O failure during classpath scanning", ex);
	}
	return candidates;
}
```

### 方法5
```java
protected boolean isCandidateComponent(MetadataReader metadataReader) throws IOException {
	// 判断是不是被exclude排除
	for (TypeFilter tf : this.excludeFilters) {
		if (tf.match(metadataReader, getMetadataReaderFactory())) {
			return false;
		}
	}
	// 判断是不是在include中，此处注意会有默认的include去处理@Component，具体看方法6
	for (TypeFilter tf : this.includeFilters) {
		if (tf.match(metadataReader, getMetadataReaderFactory())) {
			// 此处进行二次判断，具体看方法7
			return isConditionMatch(metadataReader);
		}
	}
	return false;
}
```

### 方法6
```java
// 方法6结束后返回方法5进入方法7
// 从ClassPathBeanDefinitionScanner的构造方法一直进入
public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry, boolean useDefaultFilters,
			Environment environment, @Nullable ResourceLoader resourceLoader) {

	Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
	this.registry = registry;

	if (useDefaultFilters) {
		registerDefaultFilters();
	}
	setEnvironment(environment);
	setResourceLoader(resourceLoader);
}

protected void registerDefaultFilters() {
	// 此处注册Component注解
	this.includeFilters.add(new AnnotationTypeFilter(Component.class));
	ClassLoader cl = ClassPathScanningCandidateComponentProvider.class.getClassLoader();
	try {
		this.includeFilters.add(new AnnotationTypeFilter(
				((Class<? extends Annotation>) ClassUtils.forName("javax.annotation.ManagedBean", cl)), false));
		logger.trace("JSR-250 'javax.annotation.ManagedBean' found and supported for component scanning");
	}
	catch (ClassNotFoundException ex) {
		// JSR-250 1.1 API (as included in Java EE 6) not available - simply skip.
	}
	try {
		this.includeFilters.add(new AnnotationTypeFilter(
				((Class<? extends Annotation>) ClassUtils.forName("javax.inject.Named", cl)), false));
		logger.trace("JSR-330 'javax.inject.Named' annotation found and supported for component scanning");
	}
	catch (ClassNotFoundException ex) {
		// JSR-330 API not available - simply skip.
	}
}
```

### 方法7
```java
private boolean isConditionMatch(MetadataReader metadataReader) {
	if (this.conditionEvaluator == null) {
		this.conditionEvaluator =
				new ConditionEvaluator(getRegistry(), this.environment, this.resourcePatternResolver);
	}
	// 进入方法8
	return !this.conditionEvaluator.shouldSkip(metadataReader.getAnnotationMetadata());
}
```

### 方法8
```java
// 方法8结束后回到方法4进入方法9
public boolean shouldSkip(@Nullable AnnotatedTypeMetadata metadata, @Nullable ConfigurationPhase phase) {
	// 此处判断是否有Conditional注解
	if (metadata == null || !metadata.isAnnotated(Conditional.class.getName())) {
		return false;
	}

	if (phase == null) {
		if (metadata instanceof AnnotationMetadata &&
				ConfigurationClassUtils.isConfigurationCandidate((AnnotationMetadata) metadata)) {
			return shouldSkip(metadata, ConfigurationPhase.PARSE_CONFIGURATION);
		}
		return shouldSkip(metadata, ConfigurationPhase.REGISTER_BEAN);
	}

	List<Condition> conditions = new ArrayList<>();
	for (String[] conditionClasses : getConditionClasses(metadata)) {
		for (String conditionClass : conditionClasses) {
			Condition condition = getCondition(conditionClass, this.context.getClassLoader());
			conditions.add(condition);
		}
	}

	AnnotationAwareOrderComparator.sort(conditions);

	for (Condition condition : conditions) {
		ConfigurationPhase requiredPhase = null;
		if (condition instanceof ConfigurationCondition) {
			requiredPhase = ((ConfigurationCondition) condition).getConfigurationPhase();
		}
		if ((requiredPhase == null || requiredPhase == phase) && !condition.matches(this.context, metadata)) {
			return true;
		}
	}

	return false;
}
```

### 方法9
```java
// 方法9结束后回到方法2
public ScannedGenericBeanDefinition(MetadataReader metadataReader) {
	Assert.notNull(metadataReader, "MetadataReader must not be null");
	this.metadata = metadataReader.getAnnotationMetadata();
	// 将BeanClassNmae设置
	setBeanClassName(this.metadata.getClassName());
	setResource(metadataReader.getResource());
}
```

### 方法10
```java
public static void registerBeanDefinition(
		BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
		throws BeanDefinitionStoreException {

	// Register bean definition under primary name.
	String beanName = definitionHolder.getBeanName();
	// 进入方法11
	registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

	// Register aliases for bean name, if any.
	String[] aliases = definitionHolder.getAliases();
	if (aliases != null) {
		// 注册别名
		for (String alias : aliases) {
			registry.registerAlias(beanName, alias);
		}
	}
}
```

### 方法11
```java
public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition) throws BeanDefinitionStoreException {
    Assert.hasText(beanName, "Bean name must not be empty");
    Assert.notNull(beanDefinition, "BeanDefinition must not be null");
    if (beanDefinition instanceof AbstractBeanDefinition) {
        try {
            ((AbstractBeanDefinition)beanDefinition).validate();
        } catch (BeanDefinitionValidationException var8) {
            throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName, "Validation of bean definition failed", var8);
        }
    }

    BeanDefinition existingDefinition = (BeanDefinition)this.beanDefinitionMap.get(beanName);
    if (existingDefinition != null) {
        if (!this.isAllowBeanDefinitionOverriding()) {
            throw new BeanDefinitionOverrideException(beanName, beanDefinition, existingDefinition);
        }

        if (existingDefinition.getRole() < beanDefinition.getRole()) {
            if (this.logger.isInfoEnabled()) {
                this.logger.info("Overriding user-defined bean definition for bean '" + beanName + "' with a framework-generated bean definition: replacing [" + existingDefinition + "] with [" + beanDefinition + "]");
            }
        } else if (!beanDefinition.equals(existingDefinition)) {
            if (this.logger.isDebugEnabled()) {
                this.logger.debug("Overriding bean definition for bean '" + beanName + "' with a different definition: replacing [" + existingDefinition + "] with [" + beanDefinition + "]");
            }
        } else if (this.logger.isTraceEnabled()) {
            this.logger.trace("Overriding bean definition for bean '" + beanName + "' with an equivalent definition: replacing [" + existingDefinition + "] with [" + beanDefinition + "]");
        }
        // 塞值
        this.beanDefinitionMap.put(beanName, beanDefinition);
    } else {
        if (this.hasBeanCreationStarted()) {
            synchronized(this.beanDefinitionMap) {
            	// 塞值
                this.beanDefinitionMap.put(beanName, beanDefinition);
                List<String> updatedDefinitions = new ArrayList(this.beanDefinitionNames.size() + 1);
                updatedDefinitions.addAll(this.beanDefinitionNames);
                updatedDefinitions.add(beanName);
                this.beanDefinitionNames = updatedDefinitions;
                this.removeManualSingletonName(beanName);
            }
        } else {
        	// 塞值
            this.beanDefinitionMap.put(beanName, beanDefinition);
            this.beanDefinitionNames.add(beanName);
            this.removeManualSingletonName(beanName);
        }

        this.frozenBeanDefinitionNames = null;
    }

    if (existingDefinition == null && !this.containsSingleton(beanName)) {
        if (this.isConfigurationFrozen()) {
            this.clearByTypeCache();
        }
    } else {
        this.resetBeanDefinition(beanName);
    }

}
```