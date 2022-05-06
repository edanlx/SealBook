# 【JAVA框架】02-spring的factrybean懒加载的源码
> 欢迎关注b站账号/公众号【六边形战士夏宁】，一个要把各项指标拉满的男人。该文章已在[github目录](https://github.com/edanlx/SealBook/blob/master/catalogue/wechat.md)收录。
屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。

## 1.背景
在最开始扫描的时候BeanDefinition中并没有扫描factrybean，那么该对象是如何生成的
## 2.整体流程
最开始只会根据BeanDefinition进行创建，在getBean时会根据外界传入的name是否携带&进行判断取哪个bean，原先的在单例池中。factoryBean创建的则在factoryBean缓存池中
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
在系统启动后发现没有打印"实例化UserFacoryBean",发现其为懒加载
## 4.源码过程
AnnotationConfigApplicationContext->refresh->finishBeanFactoryInitialization->preInstantiateSingletons
### 4.1方法1
```java
public void preInstantiateSingletons() throws BeansException {
	if (logger.isTraceEnabled()) {
		logger.trace("Pre-instantiating singletons in " + this);
	}

	// Iterate over a copy to allow for init methods which in turn register new bean definitions.
	// While this may not be part of the regular factory bootstrap, it does otherwise work fine.
	// 获取beanDefinitionName
	List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

	// Trigger initialization of all non-lazy singleton beans...
	for (String beanName : beanNames) {
		// 合并BeanDefinition，主要用于parent属性，该属性可以让继承parent的BeanDefinition
		// 如果有多个context还会递归找父context的属性进行合并
		// 注意这里会生成RootBeanDefinition，后续会用该对象替代原先的BeanDefinitionMap，一直点进去会有overrideFrom核心方法
		RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
		// bd.isAbstract(),抽象属性，非抽象类，标注了抽象属性在这里会跳过不会被实例化，但在上一步可以被合并BeanDefinition，除了该属性
		// isSingleton()、!isLazyInit()单例和非懒加载的进入
		if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
			// 判断是否是factoryBean
			if (isFactoryBean(beanName)) {
				// 如果是factoryBean，则以&+beanName进入getBean方法，但此处注意里面有做二次处理，实际上单例池中还是RootBeanDefinition的beanName,查看方法2
				Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
				if (bean instanceof FactoryBean) {
					final FactoryBean<?> factory = (FactoryBean<?>) bean;
					boolean isEagerInit;
					if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
						isEagerInit = AccessController.doPrivileged((PrivilegedAction<Boolean>)
										((SmartFactoryBean<?>) factory)::isEagerInit,
								getAccessControlContext());
					}
					else {
						isEagerInit = (factory instanceof SmartFactoryBean &&
								((SmartFactoryBean<?>) factory).isEagerInit());
					}
					// 如果实现了SmartFactoryBean并且有EagerInit属性则创建FactoryBean，注意此处虽然传的是RootBeanDefinition的beanName但是其实调用的FactoryBean的实现，和普通的懒加载式FactoryBean的逻辑一致
					if (isEagerInit) {
						getBean(beanName);
					}
				}
			}
			else {
				getBean(beanName);
			}
		}
	}

	// Trigger post-initialization callback for all applicable beans...
	for (String beanName : beanNames) {
		Object singletonInstance = getSingleton(beanName);
		// 判断是否实现了SmartInitializingSingleton，如果有则回调，即所有单例bean生成完毕后的回调
		if (singletonInstance instanceof SmartInitializingSingleton) {
			final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
			if (System.getSecurityManager() != null) {
				AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
					smartSingleton.afterSingletonsInstantiated();
					return null;
				}, getAccessControlContext());
			}
			else {
				smartSingleton.afterSingletonsInstantiated();
			}
		}
	}
}
```
### 4.2方法2
```java
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
			@Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {
	// 如果是以&开头这里会将其去除
	final String beanName = transformedBeanName(name);
	Object sharedInstance = getSingleton(beanName);
	// 如果已经创建过则进入
	if (sharedInstance != null && args == null) {
		if (logger.isTraceEnabled()) {
			if (isSingletonCurrentlyInCreation(beanName)) {
				logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +
						"' that is not fully initialized yet - a consequence of a circular reference");
			}
			else {
				logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
			}
		}
		// 注意此处会将去除&和未去除的&都传入 进入方法3
		bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
	}...
}

```
### 4.3方法3
```java
protected Object getObjectForBeanInstance(
			Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd) {

	// Don't let calling code try to dereference the factory if the bean isn't a factory.
	// 判断name前有没有&符号，如果有则表示要拿factoryBean的原始对象
	if (BeanFactoryUtils.isFactoryDereference(name)) {
		if (beanInstance instanceof NullBean) {
			return beanInstance;
		}
		if (!(beanInstance instanceof FactoryBean)) {
			throw new BeanIsNotAFactoryException(beanName, beanInstance.getClass());
		}
		if (mbd != null) {
			mbd.isFactoryBean = true;
		}
		return beanInstance;
	}

	// Now we have the bean instance, which may be a normal bean or a FactoryBean.
	// If it's a FactoryBean, we use it to create a bean instance, unless the
	// caller actually wants a reference to the factory.
	if (!(beanInstance instanceof FactoryBean)) {
		return beanInstance;
	}

	Object object = null;
	if (mbd != null) {
		mbd.isFactoryBean = true;
	}
	else {
		// 从factoryBeanObjectCache缓存池中拿factoryBean
		object = getCachedObjectForFactoryBean(beanName);
	}
	// 如果没拿到则创建
	if (object == null) {
		// Return bean instance from factory.
		FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
		// Caches object obtained from FactoryBean if it is a singleton.
		if (mbd == null && containsBeanDefinition(beanName)) {
			mbd = getMergedLocalBeanDefinition(beanName);
		}
		boolean synthetic = (mbd != null && mbd.isSynthetic());
		object = getObjectFromFactoryBean(factory, beanName, !synthetic);
	}
	return object;
}
```