## 2.扩展点汇总
1. spingBean生命周期
2. 创建bean的扩展点比如init,FacoryBean等
3. import注册
4. Aware
5. listen
6. lifecycle
7. SmartInitializingSingleton
8. HandlerInterceptor(spingMVC拦截器)
9. MethodInterceptor(aop扩展点)
## 3.nacos
- 自动注册|onApplicationEvent
AbstractAutoServiceRegistration#onApplicationEvent->NacosServiceRegistry#register
- 接收实例变更|SmartLifecycle
NacosWatch#start->NamingService#subscribe
## 4.feign
- factoryBean
生成代理类
## 5.ribbon
- SmartInitializingSingleton
循环对其添加拦截器
## 6.sentinel
- HandlerInterceptor
资源保护
- factoryBean
注册数据源进行持久化
## 7.seata
- AbstractAutoProxyCreator
springAOP代替原先事务