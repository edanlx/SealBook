# 【框架类源码】06-spring三级缓存(循环依赖)
> 欢迎关注b站账号/公众号【六边形战士夏宁】，一个要把各项指标拉满的男人。该文章已在[github目录](https://github.com/edanlx/SealBook/blob/master/catalogue/wechat.md)收录。
屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。

## 1.背景
spring三级缓存如何处理循环依赖
## 2.整体流程
### 2.1涉及对象
1. singletonObjects：缓存经过了完整生命周期的bean
2. earlySingletonObjects:正在创建时出现了循环依赖，将对象放入其中如果有AOP则放入代理对象
3. singletonFactories:获取原始bean(虽然这里是原始bean但是该bean贯穿整个生命周期就会变成完整bean)或者代理对象的匿名函数
4. earlyProxyReferences:AOP缓存
### 2.2整体流程
Aservice自动注入Bservice时，Bservice又引用Aservice  
Bservice->单例池中查找Aservice(查到返回)->二级缓存查找Aservice(查到返回)->三级缓存查找Aservice->查到后创建二级缓存同时删除三级缓存保证AOP对象一致
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
### 4.1singletonObjects.put
#### 4.1.1调用addSingleton
```java
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
	Assert.notNull(beanName, "Bean name must not be null");
	synchronized (this.singletonObjects) {
		Object singletonObject = this.singletonObjects.get(beanName);
		if (singletonObject == null) {
			if (this.singletonsCurrentlyInDestruction) {
				throw new BeanCreationNotAllowedException(beanName,
						"Singleton bean creation not allowed while singletons of this factory are in destruction " +
						"(Do not request a bean from a BeanFactory in a destroy method implementation!)");
			}
			if (logger.isDebugEnabled()) {
				logger.debug("Creating shared instance of singleton bean '" + beanName + "'");
			}
			beforeSingletonCreation(beanName);
			boolean newSingleton = false;
			boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
			if (recordSuppressedExceptions) {
				this.suppressedExceptions = new LinkedHashSet<>();
			}
			try {
				singletonObject = singletonFactory.getObject();
				newSingleton = true;
			}
			catch (IllegalStateException ex) {
				// Has the singleton object implicitly appeared in the meantime ->
				// if yes, proceed with it since the exception indicates that state.
				singletonObject = this.singletonObjects.get(beanName);
				if (singletonObject == null) {
					throw ex;
				}
			}
			catch (BeanCreationException ex) {
				if (recordSuppressedExceptions) {
					for (Exception suppressedException : this.suppressedExceptions) {
						ex.addRelatedCause(suppressedException);
					}
				}
				throw ex;
			}
			finally {
				if (recordSuppressedExceptions) {
					this.suppressedExceptions = null;
				}
				afterSingletonCreation(beanName);
			}
			if (newSingleton) {
				// 核心方法
				addSingleton(beanName, singletonObject);
			}
		}
		return singletonObject;
	}
}
```
#### 4.1.2addSingleton核心
```java
protected void addSingleton(String beanName, Object singletonObject) {
	synchronized (this.singletonObjects) {
		this.singletonObjects.put(beanName, singletonObject);
		this.singletonFactories.remove(beanName);
		this.earlySingletonObjects.remove(beanName);
		this.registeredSingletons.add(beanName);
	}
}
```
### 4.2earlySingletonObjects.put()
```java
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
	Object singletonObject = this.singletonObjects.get(beanName);
	if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
		synchronized (this.singletonObjects) {
			singletonObject = this.earlySingletonObjects.get(beanName);
			if (singletonObject == null && allowEarlyReference) {
				ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
				if (singletonFactory != null) {
					singletonObject = singletonFactory.getObject();
					this.earlySingletonObjects.put(beanName, singletonObject);
					this.singletonFactories.remove(beanName);
				}
			}
		}
	}
	return singletonObject;
}
```


### 4.3singletonFactories.put
#### 4.3.1singletonFactories核心方法
```java
protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {
	// 前置方法addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
	Assert.notNull(singletonFactory, "Singleton factory must not be null");
	synchronized (this.singletonObjects) {
		if (!this.singletonObjects.containsKey(beanName)) {
			this.singletonFactories.put(beanName, singletonFactory);
			this.earlySingletonObjects.remove(beanName);
			this.registeredSingletons.add(beanName);
		}
	}
}
```

#### 4.3.2getEarlyBeanReference
```java
protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {
	Object exposedObject = bean;
	if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
		for (BeanPostProcessor bp : getBeanPostProcessors()) {
			// 判断该bean是否存在aop代理，如果有则返回代理对象
			if (bp instanceof SmartInstantiationAwareBeanPostProcessor) {
				SmartInstantiationAwareBeanPostProcessor ibp = (SmartInstantiationAwareBeanPostProcessor) bp;
				exposedObject = ibp.getEarlyBeanReference(exposedObject, beanName);
			}
		}
	}
	return exposedObject;
}
```

#### 4.3.3getEarlyBeanReference
```java
public Object getEarlyBeanReference(Object bean, String beanName) {
	Object cacheKey = getCacheKey(bean.getClass(), beanName);
	this.earlyProxyReferences.put(cacheKey, bean);
	return wrapIfNecessary(bean, beanName, cacheKey);
}
```

### 4.4二级缓存获取aop对象核心
```java
// true支持循环依赖
if (earlySingletonExposure) {
	// 有二级则返回二级缓存，没有则返回一级
	Object earlySingletonReference = getSingleton(beanName, false);
	if (earlySingletonReference != null) {
		// exposedObject初始化后回调，正常为原始对象与bean一致，特殊情况为在某个环节Bean被改变
		if (exposedObject == bean) {
			// 赋值最终对象
			exposedObject = earlySingletonReference;
		}
		else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
			String[] dependentBeans = getDependentBeans(beanName);
			Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
			for (String dependentBean : dependentBeans) {
				if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
					actualDependentBeans.add(dependentBean);
				}
			}
			if (!actualDependentBeans.isEmpty()) {
				throw new BeanCurrentlyInCreationException(beanName,
						"Bean with name '" + beanName + "' has been injected into other beans [" +
						StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
						"] in its raw version as part of a circular reference, but has eventually been " +
						"wrapped. This means that said other beans do not use the final version of the " +
						"bean. This is often the result of over-eager type matching - consider using " +
						"'getBeanNamesForType' with the 'allowEagerInit' flag turned off, for example.");
			}
		}
	}
}
```