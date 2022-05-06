# 【框架类源码】07-spring启动整体流程
> 欢迎关注b站账号/公众号【六边形战士夏宁】，一个要把各项指标拉满的男人。该文章已在[github目录](https://github.com/edanlx/SealBook/blob/master/catalogue/wechat.md)收录。
屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。

## 1.背景
Mybaties启动大体流程
## 2.整体流程

## 3.@MapperScan
```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(MapperScannerRegistrar.class)
@Repeatable(MapperScans.class)
public @interface MapperScan {
}
```
```java
public class MapperScannerRegistrar implements ImportBeanDefinitionRegistrar, ResourceLoaderAware {
	...
}
```
```java
@Override
  public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
    AnnotationAttributes mapperScanAttrs = AnnotationAttributes
        .fromMap(importingClassMetadata.getAnnotationAttributes(MapperScan.class.getName()));
    if (mapperScanAttrs != null) {
    // 如果是早期1.x版本此处为直接scanner扫描不是注册成BeanDefinition再扫描
    // 所以在2.x版本可以使用@Bean的方式生成MapperScannerConfigurer进行扫描不写@MapperScan的另一种写法
      registerBeanDefinitions(importingClassMetadata, mapperScanAttrs, registry,
          generateBaseBeanName(importingClassMetadata, 0));
    }
  }

void registerBeanDefinitions(AnnotationMetadata annoMeta, AnnotationAttributes annoAttrs,
  BeanDefinitionRegistry registry, String beanName) {

BeanDefinitionBuilder builder = BeanDefinitionBuilder.genericBeanDefinition(MapperScannerConfigurer.class);
builder.addPropertyValue("processPropertyPlaceHolders", true);

Class<? extends Annotation> annotationClass = annoAttrs.getClass("annotationClass");
if (!Annotation.class.equals(annotationClass)) {
  builder.addPropertyValue("annotationClass", annotationClass);
}

Class<?> markerInterface = annoAttrs.getClass("markerInterface");
if (!Class.class.equals(markerInterface)) {
  builder.addPropertyValue("markerInterface", markerInterface);
}

Class<? extends BeanNameGenerator> generatorClass = annoAttrs.getClass("nameGenerator");
if (!BeanNameGenerator.class.equals(generatorClass)) {
  builder.addPropertyValue("nameGenerator", BeanUtils.instantiateClass(generatorClass));
}

Class<? extends MapperFactoryBean> mapperFactoryBeanClass = annoAttrs.getClass("factoryBean");
if (!MapperFactoryBean.class.equals(mapperFactoryBeanClass)) {
  builder.addPropertyValue("mapperFactoryBeanClass", mapperFactoryBeanClass);
}

String sqlSessionTemplateRef = annoAttrs.getString("sqlSessionTemplateRef");
if (StringUtils.hasText(sqlSessionTemplateRef)) {
  builder.addPropertyValue("sqlSessionTemplateBeanName", annoAttrs.getString("sqlSessionTemplateRef"));
}

String sqlSessionFactoryRef = annoAttrs.getString("sqlSessionFactoryRef");
if (StringUtils.hasText(sqlSessionFactoryRef)) {
  builder.addPropertyValue("sqlSessionFactoryBeanName", annoAttrs.getString("sqlSessionFactoryRef"));
}

List<String> basePackages = new ArrayList<>();
basePackages.addAll(
    Arrays.stream(annoAttrs.getStringArray("value")).filter(StringUtils::hasText).collect(Collectors.toList()));

basePackages.addAll(Arrays.stream(annoAttrs.getStringArray("basePackages")).filter(StringUtils::hasText)
    .collect(Collectors.toList()));

basePackages.addAll(Arrays.stream(annoAttrs.getClassArray("basePackageClasses")).map(ClassUtils::getPackageName)
    .collect(Collectors.toList()));

if (basePackages.isEmpty()) {
  basePackages.add(getDefaultBasePackage(annoMeta));
}

String lazyInitialization = annoAttrs.getString("lazyInitialization");
if (StringUtils.hasText(lazyInitialization)) {
  builder.addPropertyValue("lazyInitialization", lazyInitialization);
}

String defaultScope = annoAttrs.getString("defaultScope");
if (!AbstractBeanDefinition.SCOPE_DEFAULT.equals(defaultScope)) {
  builder.addPropertyValue("defaultScope", defaultScope);
}

builder.addPropertyValue("basePackage", StringUtils.collectionToCommaDelimitedString(basePackages));
// 将写MapperScannerConfigurer注册成BeanDefinition里面填写了prpertyValue为写@MapperScan
registry.registerBeanDefinition(beanName, builder.getBeanDefinition());

}
```
## 3.1MapperScannerConfigurer.postProcessBeanDefinitionRegistry
```java
@Override
public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
if (this.processPropertyPlaceHolders) {
  processPropertyPlaceHolders();
}

ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);
scanner.setAddToConfig(this.addToConfig);
scanner.setAnnotationClass(this.annotationClass);
scanner.setMarkerInterface(this.markerInterface);
scanner.setSqlSessionFactory(this.sqlSessionFactory);
scanner.setSqlSessionTemplate(this.sqlSessionTemplate);
scanner.setSqlSessionFactoryBeanName(this.sqlSessionFactoryBeanName);
scanner.setSqlSessionTemplateBeanName(this.sqlSessionTemplateBeanName);
scanner.setResourceLoader(this.applicationContext);
scanner.setBeanNameGenerator(this.nameGenerator);
scanner.setMapperFactoryBeanClass(this.mapperFactoryBeanClass);
if (StringUtils.hasText(lazyInitialization)) {
  scanner.setLazyInitialization(Boolean.valueOf(lazyInitialization));
}
if (StringUtils.hasText(defaultScope)) {
  scanner.setDefaultScope(defaultScope);
}
// 注册自己的includeFilters，因为本身的扫描是不过接口的
scanner.registerFilters();
// 核心方法
scanner.scan(
    StringUtils.tokenizeToStringArray(this.basePackage, ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS));
}
```
## 3.2scanner.scan
```java
public int scan(String... basePackages) {
	int beanCountAtScanStart = this.registry.getBeanDefinitionCount();
	// 核心方法
	doScan(basePackages);

	// Register annotation config processors, if necessary.
	if (this.includeAnnotationConfig) {
		AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
	}

	return (this.registry.getBeanDefinitionCount() - beanCountAtScanStart);
}
```
## 3.3doScan(注意此处为ClassPathMapperScanner实现)
```java
@Override
public Set<BeanDefinitionHolder> doScan(String... basePackages) {
// 调用父类方法的doScan
Set<BeanDefinitionHolder> beanDefinitions = super.doScan(basePackages);

if (beanDefinitions.isEmpty()) {
  LOGGER.warn(() -> "No MyBatis mapper was found in '" + Arrays.toString(basePackages)
      + "' package. Please check your configuration.");
} else {
//核心方法
  processBeanDefinitions(beanDefinitions);
}

return beanDefinitions;
}
```
### 3.3.1isCandidateComponent
```java
// 修改其中一个扫描的判断逻辑
protected boolean isCandidateComponent(AnnotatedBeanDefinition beanDefinition) {
    return beanDefinition.getMetadata().isInterface() && beanDefinition.getMetadata().isIndependent();
}
```
### 3.4processBeanDefinitions
```java
private void processBeanDefinitions(Set<BeanDefinitionHolder> beanDefinitions) {
    GenericBeanDefinition definition;
    // for循环生成beanDefinition
    for (BeanDefinitionHolder holder : beanDefinitions) {
      definition = (GenericBeanDefinition) holder.getBeanDefinition();
      String beanClassName = definition.getBeanClassName();
      LOGGER.debug(() -> "Creating MapperFactoryBean with name '" + holder.getBeanName() + "' and '" + beanClassName
          + "' mapperInterface");

      // the mapper interface is the original class of the bean
      // but, the actual class of the bean is MapperFactoryBean
      // 给构造函数方法传参
      definition.getConstructorArgumentValues().addGenericArgumentValue(beanClassName); // issue #59
      // 构造该bean的class
      definition.setBeanClass(this.mapperFactoryBeanClass);

      definition.getPropertyValues().add("addToConfig", this.addToConfig);

      boolean explicitFactoryUsed = false;
      if (StringUtils.hasText(this.sqlSessionFactoryBeanName)) {
        definition.getPropertyValues().add("sqlSessionFactory",
            new RuntimeBeanReference(this.sqlSessionFactoryBeanName));
        explicitFactoryUsed = true;
      } else if (this.sqlSessionFactory != null) {
        definition.getPropertyValues().add("sqlSessionFactory", this.sqlSessionFactory);
        explicitFactoryUsed = true;
      }

      if (StringUtils.hasText(this.sqlSessionTemplateBeanName)) {
        if (explicitFactoryUsed) {
          LOGGER.warn(
              () -> "Cannot use both: sqlSessionTemplate and sqlSessionFactory together. sqlSessionFactory is ignored.");
        }
        definition.getPropertyValues().add("sqlSessionTemplate",
            new RuntimeBeanReference(this.sqlSessionTemplateBeanName));
        explicitFactoryUsed = true;
      } else if (this.sqlSessionTemplate != null) {
        if (explicitFactoryUsed) {
          LOGGER.warn(
              () -> "Cannot use both: sqlSessionTemplate and sqlSessionFactory together. sqlSessionFactory is ignored.");
        }
        definition.getPropertyValues().add("sqlSessionTemplate", this.sqlSessionTemplate);
        explicitFactoryUsed = true;
      }

      if (!explicitFactoryUsed) {
        LOGGER.debug(() -> "Enabling autowire by type for MapperFactoryBean with name '" + holder.getBeanName() + "'.");
        // 注入sqlSession时替代@Autowirde注解
        definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);
      }
      definition.setLazyInit(lazyInitialization);
    }
 }
```
## 4. MapperFactoryBean
```java
public class MapperFactoryBean<T> extends SqlSessionDaoSupport implements FactoryBean<T> {
	// 接收保存前面步骤传进来的参数
	public MapperFactoryBean(Class<T> mapperInterface) {
    this.mapperInterface = mapperInterface;
  }

  // 在创建bean后获取其替代对象时调用
  @Override
  public T getObject() throws Exception {
  	// 获取mybaties自己生成的代理对象
    return getSqlSession().getMapper(this.mapperInterface);
  }
}
```
### 4.1 getSqlSession
```java
// 注意此处的sqlSessionTemplate是spring-mybaties包的其继承的SqlSession是原生mybaties,由defaultSqlSession执行
private SqlSessionTemplate sqlSessionTemplate;
public SqlSession getSqlSession() {
    return this.sqlSessionTemplate;
  }
// SqlSessionFactory生成(由SqlSessionFactoryBean生成,其xml路径已经被spingBootAuto全部获取在spring-mybaties中调用mybaties循环加载xml)
public void setSqlSessionFactory(SqlSessionFactory sqlSessionFactory) {
    if (this.sqlSessionTemplate == null || sqlSessionFactory != this.sqlSessionTemplate.getSqlSessionFactory()) {
      this.sqlSessionTemplate = createSqlSessionTemplate(sqlSessionFactory);
    }
  }
@Override
// 注意这里面的所有方法都是由sqlSessionProxy进行执行
  public <K, V> Map<K, V> selectMap(String statement, String mapKey) {
    return this.sqlSessionProxy.selectMap(statement, mapKey);
  }
 ...
```
#### 4.1.1sqlSessionProxy
```java
public SqlSessionTemplate(SqlSessionFactory sqlSessionFactory, ExecutorType executorType,
      PersistenceExceptionTranslator exceptionTranslator) {

    notNull(sqlSessionFactory, "Property 'sqlSessionFactory' is required");
    notNull(executorType, "Property 'executorType' is required");

    this.sqlSessionFactory = sqlSessionFactory;
    this.executorType = executorType;
    this.exceptionTranslator = exceptionTranslator;
    // JDK动态代理,sqlSessionTemplate->sqlSessionFactory->sqlSessionProxy->defaultSqlSession
    this.sqlSessionProxy = (SqlSession) newProxyInstance(SqlSessionFactory.class.getClassLoader(),
        new Class[] { SqlSession.class }, new SqlSessionInterceptor());
  }
```
#### 4.1.2SqlSessionInterceptor.invoke
```java
@Override
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
  // 核心逻辑
  SqlSession sqlSession = getSqlSession(SqlSessionTemplate.this.sqlSessionFactory,
      SqlSessionTemplate.this.executorType, SqlSessionTemplate.this.exceptionTranslator);
  try {
    Object result = method.invoke(sqlSession, args);
    if (!isSqlSessionTransactional(sqlSession, SqlSessionTemplate.this.sqlSessionFactory)) {
      // force commit even on non-dirty sessions because some databases require
      // a commit/rollback before calling close()
      sqlSession.commit(true);
    }
    return result;
  } catch (Throwable t) {
    Throwable unwrapped = unwrapThrowable(t);
    if (SqlSessionTemplate.this.exceptionTranslator != null && unwrapped instanceof PersistenceException) {
      // release the connection to avoid a deadlock if the translator is no loaded. See issue #22
      closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
      sqlSession = null;
      Throwable translated = SqlSessionTemplate.this.exceptionTranslator
          .translateExceptionIfPossible((PersistenceException) unwrapped);
      if (translated != null) {
        unwrapped = translated;
      }
    }
    throw unwrapped;
  } finally {
    if (sqlSession != null) {
      closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
    }
  }
}
```
```java
public static SqlSession getSqlSession(SqlSessionFactory sessionFactory, ExecutorType executorType,
      PersistenceExceptionTranslator exceptionTranslator) {
	notNull(sessionFactory, NO_SQL_SESSION_FACTORY_SPECIFIED);
    notNull(executorType, NO_EXECUTOR_TYPE_SPECIFIED);

    // spring事务、以及ThreadLoacale处理线程安全(原生mybaties不是线程安全的)(同时会导致一级缓存失效，启用则开启事务使用同一个sqlSession即可)
    SqlSessionHolder holder = (SqlSessionHolder) TransactionSynchronizationManager.getResource(sessionFactory);
   	
    SqlSession session = sessionHolder(executorType, holder);
    if (session != null) {
      return session;
    }
	// 获取mybaties原生的defaultSqlSession
    session = sessionFactory.openSession(executorType);

    registerSessionHolder(sessionFactory, executorType, exceptionTranslator, session);

    return session;
  }
```

### 4.2 getMapper
```java
@Override
  public <T> T getMapper(Class<T> type) {
  	// 核心方法
    return configuration.getMapper(type, this);
  }
```
```java
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
	// 核心方法
    return mapperRegistry.getMapper(type, sqlSession);
  }
```
```java
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
    if (mapperProxyFactory == null) {
      throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
    }
    try {
    	// 核心方法
      return mapperProxyFactory.newInstance(sqlSession);
    } catch (Exception e) {
      throw new BindingException("Error getting mapper instance. Cause: " + e, e);
    }
  }
```
```java
public T newInstance(SqlSession sqlSession) {
    final MapperProxy<T> mapperProxy = new MapperProxy<>(sqlSession, mapperInterface, methodCache);
    return newInstance(mapperProxy);
  }
```
```java
protected T newInstance(MapperProxy<T> mapperProxy) {
	// JDK动态代理
    return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
  }

  public T newInstance(SqlSession sqlSession) {
    final MapperProxy<T> mapperProxy = new MapperProxy<>(sqlSession, mapperInterface, methodCache);
    // 核心方法
    return newInstance(mapperProxy);
  }
```
#### 4.2.1MapperProxy
```java
@Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      if (Object.class.equals(method.getDeclaringClass())) {
        return method.invoke(this, args);
      } else {
        return cachedInvoker(method).invoke(proxy, method, args, sqlSession);
      }
    } catch (Throwable t) {
      throw ExceptionUtil.unwrapThrowable(t);
    }
  }
```