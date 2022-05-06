其虽然在web页面配了数据，但会推送至客户端，实际运行都在客户端。  
在客户端启动后会将自己注册到服务端，并发心跳保活,使用netty进行通信
## 4.启动类
spring-cloud-starter-alibaba-sentinel-2.2.6.RELEASE.jar!/META-INF/spring.factories->com.alibaba.cloud.sentinel.custom.SentinelAutoConfiguration
```java
// 注解aop
@Bean
@ConditionalOnMissingBean
public SentinelResourceAspect sentinelResourceAspect() {
	return new SentinelResourceAspect();
}
```
```java
@Aspect
public class SentinelResourceAspect extends AbstractSentinelAspectSupport {

@Pointcut("@annotation(com.alibaba.csp.sentinel.annotation.SentinelResource)")
public void sentinelResourceAnnotationPointcut() {
}

@Around("sentinelResourceAnnotationPointcut()")
public Object invokeResourceWithSentinel(ProceedingJoinPoint pjp) throws Throwable {
    Method originMethod = resolveMethod(pjp);

    SentinelResource annotation = originMethod.getAnnotation(SentinelResource.class);
    if (annotation == null) {
        // Should not go through here.
        throw new IllegalStateException("Wrong state for SentinelResource annotation");
    }
    // 获取注解信息
    String resourceName = getResourceName(annotation.value(), originMethod);
    EntryType entryType = annotation.entryType();
    int resourceType = annotation.resourceType();
    Entry entry = null;
    try {
    	// 核心代码
        entry = SphU.entry(resourceName, resourceType, entryType, pjp.getArgs());
        Object result = pjp.proceed();
        return result;
    } catch (BlockException ex) {
        return handleBlockException(pjp, annotation, ex);
    } catch (Throwable ex) {
        Class<? extends Throwable>[] exceptionsToIgnore = annotation.exceptionsToIgnore();
        // The ignore list will be checked first.
        if (exceptionsToIgnore.length > 0 && exceptionBelongsTo(ex, exceptionsToIgnore)) {
            throw ex;
        }
        if (exceptionBelongsTo(ex, annotation.exceptionsToTrace())) {
            traceException(ex);
            return handleFallback(pjp, annotation, ex);
        }

        // No fallback function can handle the exception, so throw it out.
        throw ex;
    } finally {
        if (entry != null) {
        	// 全链条的收尾逻辑
            entry.exit(1, pjp.getArgs());
        }
    }
}
}
```
### 4.1com.alibaba.csp.sentinel.SphU#entry(java.lang.String, int, com.alibaba.csp.sentinel.EntryType, java.lang.Object[])
```java
public static Entry entry(String name, int resourceType, EntryType trafficType, Object[] args)
    throws BlockException {
    return Env.sph.entryWithType(name, resourceType, trafficType, 1, args);
}
```
```java
public Entry entryWithType(String name, int resourceType, EntryType entryType, int count, Object[] args)
    throws BlockException {
    return entryWithType(name, resourceType, entryType, count, false, args);
}
public Entry entryWithType(String name, int resourceType, EntryType entryType, int count, boolean prioritized,
                               Object[] args) throws BlockException {
    StringResourceWrapper resource = new StringResourceWrapper(name, entryType, resourceType);
    return entryWithPriority(resource, count, prioritized, args);
}
```
- com.alibaba.csp.sentinel.CtSph#entryWithPriority(com.alibaba.csp.sentinel.slotchain.ResourceWrapper, int, boolean, java.lang.Object...)
```java
private Entry entryWithPriority(ResourceWrapper resourceWrapper, int count, boolean prioritized, Object... args)
    throws BlockException {
    Context context = ContextUtil.getContext();
    if (context instanceof NullContext) {
        // The {@link NullContext} indicates that the amount of context has exceeded the threshold,
        // so here init the entry only. No rule checking will be done.
        return new CtEntry(resourceWrapper, null, context);
    }

    if (context == null) {
        // Using default context.
        context = InternalContextUtil.internalEnter(Constants.CONTEXT_DEFAULT_NAME);
    }

    // Global switch is close, no rule checking will do.
    if (!Constants.ON) {
        return new CtEntry(resourceWrapper, null, context);
    }
    // 责任链
    ProcessorSlot<Object> chain = lookProcessChain(resourceWrapper);

    /*
     * Means amount of resources (slot chain) exceeds {@link Constants.MAX_SLOT_CHAIN_SIZE},
     * so no rule checking will be done.
     */
    if (chain == null) {
        return new CtEntry(resourceWrapper, null, context);
    }

    Entry e = new CtEntry(resourceWrapper, chain, context);
    try {
    	// 调用责任链
        chain.entry(context, resourceWrapper, null, count, prioritized, args);
    } catch (BlockException e1) {
        e.exit(count, args);
        throw e1;
    } catch (Throwable e1) {
        // This should not happen, unless there are errors existing in Sentinel internal.
        RecordLog.info("Sentinel unexpected exception", e1);
    }
    return e;
}
```
### 4.1com.alibaba.csp.sentinel.CtSph#lookProcessChain
```java
ProcessorSlot<Object> lookProcessChain(ResourceWrapper resourceWrapper) {
    ProcessorSlotChain chain = chainMap.get(resourceWrapper);
    if (chain == null) {
        synchronized (LOCK) {
            chain = chainMap.get(resourceWrapper);
            if (chain == null) {
                // Entry size limit.
                if (chainMap.size() >= Constants.MAX_SLOT_CHAIN_SIZE) {
                    return null;
                }
                // 初始化链条
                chain = SlotChainProvider.newSlotChain();
                Map<ResourceWrapper, ProcessorSlotChain> newMap = new HashMap<ResourceWrapper, ProcessorSlotChain>(
                    chainMap.size() + 1);
                newMap.putAll(chainMap);
                newMap.put(resourceWrapper, chain);
                chainMap = newMap;
            }
        }
    }
    return chain;
}
```
- com.alibaba.csp.sentinel.slotchain.ProcessorSlotChain
```java
public static ProcessorSlotChain newSlotChain() {
    if (slotChainBuilder != null) {
        return slotChainBuilder.build();
    }

    // Resolve the slot chain builder SPI.
    slotChainBuilder = SpiLoader.of(SlotChainBuilder.class).loadFirstInstanceOrDefault();

    if (slotChainBuilder == null) {
        // Should not go through here.
        RecordLog.warn("[SlotChainProvider] Wrong state when resolving slot chain builder, using default");
        slotChainBuilder = new DefaultSlotChainBuilder();
    } else {
        RecordLog.info("[SlotChainProvider] Global slot chain builder resolved: {}",
            slotChainBuilder.getClass().getCanonicalName());
    }
    return slotChainBuilder.build();
}
```
- com.alibaba.csp.sentinel.slots.DefaultSlotChainBuilder#build
```java
public ProcessorSlotChain build() {
    ProcessorSlotChain chain = new DefaultProcessorSlotChain();
    // 根据spi加载
    /**
     * 类上面有@SpiOrder
     *com.alibaba.csp.sentinel.slots.nodeselector.NodeSelectorSlot
	com.alibaba.csp.sentinel.slots.clusterbuilder.ClusterBuilderSlot
	com.alibaba.csp.sentinel.slots.logger.LogSlot
	com.alibaba.csp.sentinel.slots.statistic.StatisticSlot
	com.alibaba.csp.sentinel.slots.block.authority.AuthoritySlot
	com.alibaba.csp.sentinel.slots.system.SystemSlot
	com.alibaba.csp.sentinel.slots.block.flow.FlowSlot
	com.alibaba.csp.sentinel.slots.block.degrade.DegradeSlot 
	 */
    List<ProcessorSlot> sortedSlotList = SpiLoader.of(ProcessorSlot.class).loadInstanceListSorted();
    for (ProcessorSlot slot : sortedSlotList) {
        if (!(slot instanceof AbstractLinkedProcessorSlot)) {
            RecordLog.warn("The ProcessorSlot(" + slot.getClass().getCanonicalName() + ") is not an instance of AbstractLinkedProcessorSlot, can't be added into ProcessorSlotChain");
            continue;
        }

        chain.addLast((AbstractLinkedProcessorSlot<?>) slot);
    }

    return chain;
}
```
## 5.com.alibaba.csp.sentinel.slots.nodeselector.NodeSelectorSlot#entry|构建聚簇节点链条
```java
// 构建聚簇节点链条
DefaultNode node = map.get(context.getName());
        if (node == null) {
            synchronized (this) {
                node = map.get(context.getName());
                if (node == null) {
                    node = new DefaultNode(resourceWrapper, null);
                    HashMap<String, DefaultNode> cacheMap = new HashMap<String, DefaultNode>(map.size());
                    cacheMap.putAll(map);
                    cacheMap.put(context.getName(), node);
                    map = cacheMap;
                    // Build invocation tree
                    ((DefaultNode) context.getLastNode()).addChild(node);
                }

            }
        }

context.setCurNode(node);
fireEntry(context, resourceWrapper, node, count, prioritized, args);
```
## 6.com.alibaba.csp.sentinel.slots.clusterbuilder.ClusterBuilderSlot#entry|构建clusterNode
```java
public void entry(Context context, ResourceWrapper resourceWrapper, DefaultNode node, int count,
                      boolean prioritized, Object... args)
    throws Throwable {
    if (clusterNode == null) {
        synchronized (lock) {
            if (clusterNode == null) {
                // Create the cluster node.
                clusterNode = new ClusterNode(resourceWrapper.getName(), resourceWrapper.getResourceType());
                HashMap<ResourceWrapper, ClusterNode> newMap = new HashMap<>(Math.max(clusterNodeMap.size(), 16));
                newMap.putAll(clusterNodeMap);
                newMap.put(node.getId(), clusterNode);

                clusterNodeMap = newMap;
            }
        }
    }
    node.setClusterNode(clusterNode);

    /*
     * if context origin is set, we should get or create a new {@link Node} of
     * the specific origin.
     */
    if (!"".equals(context.getOrigin())) {
        Node originNode = node.getClusterNode().getOrCreateOriginNode(context.getOrigin());
        context.getCurEntry().setOriginNode(originNode);
    }

    fireEntry(context, resourceWrapper, node, count, prioritized, args);
}
```
## 7.com.alibaba.csp.sentinel.slots.logger.LogSlot#entry|记录日志
```java
public void entry(Context context, ResourceWrapper resourceWrapper, DefaultNode obj, int count, boolean prioritized, Object... args)
    throws Throwable {
    try {
        fireEntry(context, resourceWrapper, obj, count, prioritized, args);
    } catch (BlockException e) {
        EagleEyeLogUtil.log(resourceWrapper.getName(), e.getClass().getSimpleName(), e.getRuleLimitApp(),
            context.getOrigin(), count);
        throw e;
    } catch (Throwable e) {
        RecordLog.warn("Unexpected entry exception", e);
    }

}
```
## 8.com.alibaba.csp.sentinel.slots.statistic.StatisticSlot#entry|统计qps等
```java
public void entry(Context context, ResourceWrapper resourceWrapper, DefaultNode node, int count,
                      boolean prioritized, Object... args) throws Throwable {
    try {
        // Do some checking.
        fireEntry(context, resourceWrapper, node, count, prioritized, args);

        // Request passed, add thread count and pass count.
        node.increaseThreadNum();
        node.addPassRequest(count);

        if (context.getCurEntry().getOriginNode() != null) {
            // Add count for origin node.
            context.getCurEntry().getOriginNode().increaseThreadNum();
            context.getCurEntry().getOriginNode().addPassRequest(count);
        }

        if (resourceWrapper.getEntryType() == EntryType.IN) {
            // Add count for global inbound entry node for global statistics.
            Constants.ENTRY_NODE.increaseThreadNum();
            Constants.ENTRY_NODE.addPassRequest(count);
        }

        // Handle pass event with registered entry callback handlers.
        for (ProcessorSlotEntryCallback<DefaultNode> handler : StatisticSlotCallbackRegistry.getEntryCallbacks()) {
            handler.onPass(context, resourceWrapper, node, count, args);
        }
    } catch (PriorityWaitException ex) {
        node.increaseThreadNum();
        if (context.getCurEntry().getOriginNode() != null) {
            // Add count for origin node.
            context.getCurEntry().getOriginNode().increaseThreadNum();
        }

        if (resourceWrapper.getEntryType() == EntryType.IN) {
            // Add count for global inbound entry node for global statistics.
            Constants.ENTRY_NODE.increaseThreadNum();
        }
        // Handle pass event with registered entry callback handlers.
        for (ProcessorSlotEntryCallback<DefaultNode> handler : StatisticSlotCallbackRegistry.getEntryCallbacks()) {
            handler.onPass(context, resourceWrapper, node, count, args);
        }
    } catch (BlockException e) {
        // Blocked, set block exception to current entry.
        context.getCurEntry().setBlockError(e);

        // Add block count.
        // 增加一次被阻塞的qps
        node.increaseBlockQps(count);
        if (context.getCurEntry().getOriginNode() != null) {
            context.getCurEntry().getOriginNode().increaseBlockQps(count);
        }

        if (resourceWrapper.getEntryType() == EntryType.IN) {
            // Add count for global inbound entry node for global statistics.
            Constants.ENTRY_NODE.increaseBlockQps(count);
        }

        // Handle block event with registered entry callback handlers.
        for (ProcessorSlotEntryCallback<DefaultNode> handler : StatisticSlotCallbackRegistry.getEntryCallbacks()) {
            handler.onBlocked(e, context, resourceWrapper, node, count, args);
        }

        throw e;
    } catch (Throwable e) {
        // Unexpected internal error, set error to current entry.
        context.getCurEntry().setError(e);

        throw e;
    }
}
```
## 9.com.alibaba.csp.sentinel.slots.block.authority.AuthoritySlot#entry|黑白名单校验
```java
public void entry(Context context, ResourceWrapper resourceWrapper, DefaultNode node, int count, boolean prioritized, Object... args)
    throws Throwable {
    checkBlackWhiteAuthority(resourceWrapper, context);
    fireEntry(context, resourceWrapper, node, count, prioritized, args);
}
```
## 10.com.alibaba.csp.sentinel.slots.system.SystemSlot#entry|系统规则校验
```java
public void entry(Context context, ResourceWrapper resourceWrapper, DefaultNode node, int count,
                      boolean prioritized, Object... args) throws Throwable {
    SystemRuleManager.checkSystem(resourceWrapper);
    fireEntry(context, resourceWrapper, node, count, prioritized, args);
}
```
## 11.com.alibaba.csp.sentinel.slots.block.flow.FlowSlot#entry|限流
```java
public void entry(Context context, ResourceWrapper resourceWrapper, DefaultNode node, int count,
                      boolean prioritized, Object... args) throws Throwable {
    checkFlow(resourceWrapper, context, node, count, prioritized);

    fireEntry(context, resourceWrapper, node, count, prioritized, args);
}
```
- com.alibaba.csp.sentinel.slots.block.flow.FlowSlot#checkFlow
```java
void checkFlow(ResourceWrapper resource, Context context, DefaultNode node, int count, boolean prioritized)
    throws BlockException {
    checker.checkFlow(ruleProvider, resource, context, node, count, prioritized);
}
```
- com.alibaba.csp.sentinel.slots.block.flow.FlowRuleChecker#checkFlow
```java
public void checkFlow(Function<String, Collection<FlowRule>> ruleProvider, ResourceWrapper resource,
                          Context context, DefaultNode node, int count, boolean prioritized) throws BlockException {
    if (ruleProvider == null || resource == null) {
        return;
    }
    // 构建服务端配置的规则
    Collection<FlowRule> rules = ruleProvider.apply(resource.getName());
    if (rules != null) {
        for (FlowRule rule : rules) {
            if (!canPassCheck(rule, context, node, count, prioritized)) {
                throw new FlowException(rule.getLimitApp(), rule);
            }
        }
    }
}
```
- com.alibaba.csp.sentinel.slots.block.flow.FlowRuleChecker#canPassCheck(com.alibaba.csp.sentinel.slots.block.flow.FlowRule, com.alibaba.csp.sentinel.context.Context, com.alibaba.csp.sentinel.node.DefaultNode, int, boolean)
```java
public boolean canPassCheck(/*@NonNull*/ FlowRule rule, Context context, DefaultNode node, int acquireCount,
                                                    boolean prioritized) {
    String limitApp = rule.getLimitApp();
    if (limitApp == null) {
        return true;
    }

    if (rule.isClusterMode()) {
        return passClusterCheck(rule, context, node, acquireCount, prioritized);
    }

    return passLocalCheck(rule, context, node, acquireCount, prioritized);
}
```
- com.alibaba.csp.sentinel.slots.block.flow.FlowRuleChecker#passLocalCheck
```java
private static boolean passLocalCheck(FlowRule rule, Context context, DefaultNode node, int acquireCount,
                                          boolean prioritized) {
    Node selectedNode = selectNodeByRequesterAndStrategy(rule, context, node);
    if (selectedNode == null) {
        return true;
    }

    return rule.getRater().canPass(selectedNode, acquireCount, prioritized);
}
```
### 11.1com.alibaba.csp.sentinel.slots.block.flow.controller.DefaultController#canPass(com.alibaba.csp.sentinel.node.Node, int, boolean)
```java
public boolean canPass(Node node, int acquireCount, boolean prioritized) {
	// 当前qps次数
    int curCount = avgUsedTokens(node);
    // acquireCount=1，判断是否大于阈值
    if (curCount + acquireCount > count) {
        if (prioritized && grade == RuleConstant.FLOW_GRADE_QPS) {
            long currentTime;
            long waitInMs;
            currentTime = TimeUtil.currentTimeMillis();
            waitInMs = node.tryOccupyNext(currentTime, acquireCount, count);
            if (waitInMs < OccupyTimeoutProperty.getOccupyTimeout()) {
                node.addWaitingRequest(currentTime + waitInMs, acquireCount);
                node.addOccupiedPass(acquireCount);
                sleep(waitInMs);

                // PriorityWaitException indicates that the request will pass after waiting for {@link @waitInMs}.
                throw new PriorityWaitException(waitInMs);
            }
        }
        return false;
    }
    return true;
}
```
#### 11.1.1com.alibaba.csp.sentinel.slots.block.flow.controller.DefaultController#avgUsedTokens
```java
private int avgUsedTokens(Node node) {
    if (node == null) {
        return DEFAULT_AVG_USED_TOKENS;
    }
    // 选择qps还是线程，根据统计信息，以滑动窗口的形式计算出已经通过的qps
    return grade == RuleConstant.FLOW_GRADE_THREAD ? node.curThreadNum() : (int)(node.passQps());
}
```
## 12.com.alibaba.csp.sentinel.slots.block.degrade.DegradeSlot#entry|熔断降级
```java
public void entry(Context context, ResourceWrapper resourceWrapper, DefaultNode node, int count,
                      boolean prioritized, Object... args) throws Throwable {
    performChecking(context, resourceWrapper);

    fireEntry(context, resourceWrapper, node, count, prioritized, args);
}
```
- com.alibaba.csp.sentinel.slots.block.degrade.DegradeSlot#performChecking
```java
void performChecking(Context context, ResourceWrapper r) throws BlockException {
	// 获取页面配置的断路器
    List<CircuitBreaker> circuitBreakers = DegradeRuleManager.getCircuitBreakers(r.getName());
    if (circuitBreakers == null || circuitBreakers.isEmpty()) {
        return;
    }
    for (CircuitBreaker cb : circuitBreakers) {
        if (!cb.tryPass(context)) {
            throw new DegradeException(cb.getRule().getLimitApp(), cb.getRule());
        }
    }
}
```
```java
public boolean tryPass(Context context) {
    // Template implementation.
    // 判断断路器是否关闭
    if (currentState.get() == State.CLOSED) {
        return true;
    }
    if (currentState.get() == State.OPEN) {
    	// 已经触发熔断规则，判断是否到达半开时间
        // For half-open state we allow a request for probing.
        return retryTimeoutArrived() && fromOpenToHalfOpen(context);
    }
    // 半开状态直接失败
    return false;
}
```
### 12.1com.alibaba.csp.sentinel.slots.block.degrade.circuitbreaker.AbstractCircuitBreaker#retryTimeoutArrived
```java
protected boolean retryTimeoutArrived() {
	// 下次可重试连接时间
    return TimeUtil.currentTimeMillis() >= nextRetryTimestamp;
}
```
- com.alibaba.csp.sentinel.slots.block.degrade.circuitbreaker.AbstractCircuitBreaker#fromOpenToHalfOpen|开改为半开返回true
```java
protected boolean fromOpenToHalfOpen(Context context) {
    if (currentState.compareAndSet(State.OPEN, State.HALF_OPEN)) {
        notifyObservers(State.OPEN, State.HALF_OPEN, null);
        Entry entry = context.getCurEntry();
        entry.whenTerminate(new BiConsumer<Context, Entry>() {
            @Override
            public void accept(Context context, Entry entry) {
                // Note: This works as a temporary workaround for https://github.com/alibaba/Sentinel/issues/1638
                // Without the hook, the circuit breaker won't recover from half-open state in some circumstances
                // when the request is actually blocked by upcoming rules (not only degrade rules).
                if (entry.getBlockError() != null) {
                    // Fallback to OPEN due to detecting request is blocked
                    currentState.compareAndSet(State.HALF_OPEN, State.OPEN);
                    notifyObservers(State.HALF_OPEN, State.OPEN, 1.0d);
                }
            }
        });
        return true;
    }
    return false;
}
```
### 12.2com.alibaba.csp.sentinel.slots.block.degrade.DegradeSlot#exit
```java
public void exit(Context context, ResourceWrapper r, int count, Object... args) {
    Entry curEntry = context.getCurEntry();
    if (curEntry.getBlockError() != null) {
        fireExit(context, r, count, args);
        return;
    }
    List<CircuitBreaker> circuitBreakers = DegradeRuleManager.getCircuitBreakers(r.getName());
    if (circuitBreakers == null || circuitBreakers.isEmpty()) {
        fireExit(context, r, count, args);
        return;
    }

    if (curEntry.getBlockError() == null) {
        // passed request
        for (CircuitBreaker circuitBreaker : circuitBreakers) {
            circuitBreaker.onRequestComplete(context);
        }
    }

    fireExit(context, r, count, args);
}
```
#### 12.2.1com.alibaba.csp.sentinel.slots.block.degrade.circuitbreaker.ResponseTimeCircuitBreaker#onRequestComplete
```java
public void onRequestComplete(Context context) {
    SlowRequestCounter counter = slidingCounter.currentWindow().value();
    Entry entry = context.getCurEntry();
    if (entry == null) {
        return;
    }
    long completeTime = entry.getCompleteTimestamp();
    if (completeTime <= 0) {
        completeTime = TimeUtil.currentTimeMillis();
    }
    // rt=responseTime
    long rt = completeTime - entry.getCreateTimestamp();
    if (rt > maxAllowedRt) {
        counter.slowCount.add(1);
    }
    counter.totalCount.add(1);

    handleStateChangeWhenThresholdExceeded(rt);
}
```
- com.alibaba.csp.sentinel.slots.block.degrade.circuitbreaker.ResponseTimeCircuitBreaker#handleStateChangeWhenThresholdExceeded
```java
private void handleStateChangeWhenThresholdExceeded(long rt) {
    if (currentState.get() == State.OPEN) {
        return;
    }
   // 如果是半开状态则重新计算
    if (currentState.get() == State.HALF_OPEN) {
        // In detecting request
        // TODO: improve logic for half-open recovery
        if (rt > maxAllowedRt) {
        	// 半开到开启
            fromHalfOpenToOpen(1.0d);
        } else {
        	// 半开到关闭
            fromHalfOpenToClose();
        }
        return;
    }
    // close变open
    List<SlowRequestCounter> counters = slidingCounter.values();
    long slowCount = 0;
    long totalCount = 0;
    for (SlowRequestCounter counter : counters) {
        slowCount += counter.slowCount.sum();
        totalCount += counter.totalCount.sum();
    }
    if (totalCount < minRequestAmount) {
        return;
    }
    double currentRatio = slowCount * 1.0d / totalCount;
    if (currentRatio > maxSlowRequestRatio) {
        transformToOpen(currentRatio);
    }
    if (Double.compare(currentRatio, maxSlowRequestRatio) == 0 &&
            Double.compare(maxSlowRequestRatio, SLOW_REQUEST_RATIO_MAX_VALUE) == 0) {
        transformToOpen(currentRatio);
    }
}
```