# 【框架类源码】07-spring启动整体流程
> 欢迎关注b站账号/公众号【六边形战士夏宁】，一个要把各项指标拉满的男人。该文章已在[github目录](https://github.com/edanlx/SealBook/blob/master/catalogue/wechat.md)收录。
屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。

## 1.背景
Mybaties四大核心对象，executor、statementHandler、ParmeterHandler、ResultHandler。plugin插件为四大核心对象增强
## 2.整体流程

```java
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBUilder.builder()
sqlSessionFactory.openSession();
```
## 3.DefaultSqlSessionFactory.openSession
```java
public SqlSession openSession(boolean autoCommit) {
  // 核心方法
  return openSessionFromDataSource(configuration.getDefaultExecutorType(), null, autoCommit);
}
```
```java
private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
  Transaction tx = null;
  try {
    final Environment environment = configuration.getEnvironment();
    final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
    // 获取事务
    tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
    // 获取执行器
    final Executor executor = configuration.newExecutor(tx, execType);
    return new DefaultSqlSession(configuration, executor, autoCommit);
  } catch (Exception e) {
    closeTransaction(tx); // may have fetched a connection so lets call close()
    throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
  } finally {
    ErrorContext.instance().reset();
  }
}
```
### 3.1newExecutor
```java
public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
    executorType = executorType == null ? defaultExecutorType : executorType;
    executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
    Executor executor;
    // 判断三种执行器类型,其实现都是mysql实现
    if (ExecutorType.BATCH == executorType) {
      executor = new BatchExecutor(this, transaction);
    } else if (ExecutorType.REUSE == executorType) {
      executor = new ReuseExecutor(this, transaction);
    } else {
      executor = new SimpleExecutor(this, transaction);
    }
    // 判断是否开启二级缓存
    if (cacheEnabled) {
      // 包装
      executor = new CachingExecutor(executor);
    }
    // 如果有插件则生成代理对象
    executor = (Executor) interceptorChain.pluginAll(executor);
    return executor;
  }
```
#### 3.1.1pluginAll
```java
public Object pluginAll(Object target) {
  // 递归包装
    for (Interceptor interceptor : interceptors) {
      target = interceptor.plugin(target);
    }
    return target;
  }
```
```java
default Object plugin(Object target) {
    return Plugin.wrap(target, this);
  }
```
```java
public static Object wrap(Object target, Interceptor interceptor) {
  Map<Class<?>, Set<Method>> signatureMap = getSignatureMap(interceptor);
  Class<?> type = target.getClass();
  Class<?>[] interfaces = getAllInterfaces(type, signatureMap);
  // 创建JDK动态代理
  if (interfaces.length > 0) {
    return Proxy.newProxyInstance(
        type.getClassLoader(),
        interfaces,
        new Plugin(target, interceptor, signatureMap));
  }
  return target;
}
```
#### 3.1.1.1invoke
```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      Set<Method> methods = signatureMap.get(method.getDeclaringClass());
      if (methods != null && methods.contains(method)) {
        // 包装为Invocation进行调用
        return interceptor.intercept(new Invocation(target, method, args));
      }
      return method.invoke(target, args);
    } catch (Exception e) {
      throw ExceptionUtil.unwrapThrowable(e);
    }
  }
```

Invocation.proceed执行原方法
```java
public Object proceed() throws InvocationTargetException, IllegalAccessException {
    return method.invoke(target, args);
  }
```

## 4.DefaultSqlSession.selectList
```java
private <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds, ResultHandler handler) {
  try {
    // 根据id获取节点
    MappedStatement ms = configuration.getMappedStatement(statement);
    return executor.query(ms, wrapCollection(parameter), rowBounds, handler);
  } catch (Exception e) {
    throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
  } finally {
    ErrorContext.instance().reset();
  }
}
```
## 4.1CachingExecutor.query
```java
public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
    // 解析动态sql
    BoundSql boundSql = ms.getBoundSql(parameterObject);
    CacheKey key = createCacheKey(ms, parameterObject, rowBounds, boundSql);
    return query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }
```

## 4.1.1query
```java
public BoundSql getBoundSql(Object parameterObject) {
  BoundSql boundSql = sqlSource.getBoundSql(parameterObject);
  List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
  if (parameterMappings == null || parameterMappings.isEmpty()) {
    boundSql = new BoundSql(configuration, boundSql.getSql(), parameterMap.getParameterMappings(), parameterObject);
  }

  // check for nested result maps in parameter mappings (issue #30)
  for (ParameterMapping pm : boundSql.getParameterMappings()) {
    String rmId = pm.getResultMapId();
    if (rmId != null) {
      ResultMap rm = configuration.getResultMap(rmId);
      if (rm != null) {
        hasNestedResultMaps |= rm.hasNestedResultMaps();
      }
    }
  }

  return boundSql;
}
```

## 4.1.1.1DynamicSqlSource.getBoundSql
```java
public BoundSql getBoundSql(Object parameterObject) {
  DynamicContext context = new DynamicContext(configuration, parameterObject);
  // 使用StringJoiner拼接，一般情况最外层是mixNode，其它解析逻辑使用OGNL解析器，值得关注的是trim是where和set的父类，即per是where或set即可
  rootSqlNode.apply(context);
  SqlSourceBuilder sqlSourceParser = new SqlSourceBuilder(configuration);
  Class<?> parameterType = parameterObject == null ? Object.class : parameterObject.getClass();
  SqlSource sqlSource = sqlSourceParser.parse(context.getSql(), parameterType, context.getBindings());
  BoundSql boundSql = sqlSource.getBoundSql(parameterObject);
  context.getBindings().forEach(boundSql::setAdditionalParameter);
  return boundSql;
}
```
MixedSqlNode.apply如下
```java
public boolean apply(DynamicContext context) {
  // 调用各自的解析逻辑进行循环递归解析
  contents.forEach(node -> node.apply(context));
  return true;
}
```

## 4.1.2query
```java
public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
      throws SQLException {
  // 获取缓存
  Cache cache = ms.getCache();
  if (cache != null) {
    // 判断是否需要刷新缓存
    flushCacheIfRequired(ms);
    if (ms.isUseCache() && resultHandler == null) {
      ensureNoOutParams(ms, boundSql);
      @SuppressWarnings("unchecked")
      // 从事务临时缓存中获取数据
      List<E> list = (List<E>) tcm.getObject(cache, key);
      if (list == null) {
        // 由原始executor执行sql
        list = delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
        tcm.putObject(cache, key, list); // issue #578 and #116
      }
      return list;
    }
  }
  return delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
}
```
## 5.plugin
只能增强四大核心对象里面已有的方法
```java
@Interceots({@Signature(type = Executor.class, method="query",arags={MappedStatement.class,Object.class,RowBounds.class,ResultHandler.class})})
```