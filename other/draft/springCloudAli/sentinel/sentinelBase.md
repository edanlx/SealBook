||Sentinel|Hytrix|
|--|--|--|
|隔离策略|信号量隔离|线程池隔离/信号量隔离|
|熔断降级策略|基于响应时间或失败比率|基于失败比率|
|实时指标实现|基于滑动窗口|滑动窗口|
|规则配置|多种数据源|多种数据源|
|扩展性|多个扩展点|插件形式|
|基于注解的支持|支持|支持|
|限流|基于QPS|有限的支持|
|流量整形|支持慢启动、匀速器模式|不支持|
|系统负载保护|支持|不支持|
|控制台|开箱即用|无|

- 插槽链路
* NodeSelectorSlot
负责收集资源的路径，并将这些资源的调用路径，以树状结构存储起来，用于根据调用路径来限流降级
* ClusterBuilderSlot
则用于存储资源的统计信息以及调用者信息，
* StatisticSlot
用于记录、统计不同纬度的runtime指标监控信息；
* FlowSlot
则用于根据预设的限流规则以及前面slot统计的状态，进行流量控制
* AuthoritySlot
则根据配置的黑白名单和调用来源信息，来做黑白名单控制；
* DegradeSlot
则通过统计信息以及预设的规则，来做熔断降级；
* SystemSlot
则通过系统的状态，例如load1等，来控制总的入口流量

- 实现资源
	1. 方式1
```java
Entry entry = null;
// 务必保证 finally会被执行
try{
	//资源名可使用任意有业务语义的字符串
	entry = SphU.entry("自定义资源名");
	//被保护的业务逻辑
	//dosomething...
}
catch (BlockException ex) {
// 资源访问阻止，被限流或被降级
// 进行相应的处理操作
} catch (Exception ex) {
// 若需要配置降级规则，需要通过这种方式记录业务异常
Tracer.traceEntry(ex, entry);
} finally {
// 务必保证exit，务必保证每个entry与exit配对
if(entry!=null){
	entry.exit();
}
```
	2. @SentinelResource
	基于注解(内部是aop,会修改异常类型)
	3. 代码定义/配置文件定义
	4. 控制台(本地端口默认8719与控制台通信)(内部是拦截器)
- 流控规则
	快速失败和排队，虽然都是阈值拒绝，但是排队会在响应超时后拒绝
- 降级规则
	不稳定的链路进行熔断降级暂时切断，防止服务雪崩，熔断器关-半开-开
	1. 慢调用比例
	2. 异常比例
	3. 异常数
- 热点参数
	针对于热点数据特殊控制，不过热点数据一般会单独部署的解决方案。只能使用注解方式
- 系统规则
	ip黑白名单
- 授权规则
	一般在网关那边使用
- 集群规则
	sentinel的核心
		* 集群总体
		* 单机均摊
- restTemplate整合
	@SentinelRestTemplate，其原理与ribbon一致，但注入方式是使用BeanPostProcessor。引入后在簇点链路中可以直接设置
- feign整合
	原理是InvocationHandler。引入后在簇点链路中可以直接设置
- dubbo整合
	原理是dubbo的filter扩展点