- DTP模型
1. 应用程序(AP):定于事务开始结束
2. 资源管理(RM):访问数据库
3. 事务管理(TM):负责提交事务
4. 通信管理(CRM):控制通信
5. 通信协议(CP)
- 2PC
有XA和TCC实现
1. 阶段1执行sql但不提交
2. TM收集各个RM的数据决定是否提交
- XA
强一致性，依赖数据库操作，除了普通dataBase，还会提供XAdataBase。seataXA做了优化使用seata创建的undoLog
- TCC
预操作，3个阶段都需要自行实现，网络波动问题有数据不一致的可能性，为了解决该问题性能下降严重。替代方案是事务mq。
1. try:请求预减
2. confirm:请求确认
3. cancel:请求取消
例子如下io.seata.rm.tcc.TccActionImpl
- AT
1. 开启事务的时候TM会生成全局事务xid给TC，微服务调用链需要传递该id。但在传递前RM已经提交DB
2. TC返回TM每个阶段的提交情况，如果有提交失败的则每个分支事务都需要反向补偿。根据UDOLOG补偿，其存在数据库，前置镜像与后置镜像。为了处理可能的补偿时会有写入，会将id发送到TC做成全局锁。成功则删除undolog、事务等信息
- SAGA
非常少用的模式
不同于TCC的三个方法，他只有两个方法提交和补偿，非常容易出现脏写
- 其它最终一致性方案
利用延迟消息进行回查

- 主要流程
1. 开启全局事务(写@GlobalTransactional的就是TM)
	1. TM(事务发起者)向TC发起GlobalBegingRequest请求
	2. TC生成全局事务信息
	3. 返回全局事务xid
2. 注册分支事务
	1. 生成前后置镜像执行业务sql
	2. 向TC注册分支事务(全局锁校验等)
	3. 保存undolog到数据库
	4. 本地事务提交或回滚
	5. 提交失败则会上报
3. 提交/回滚
	1. 成功则删除相关日志、锁数据
	2. 失败则TM向TC发送回滚请求，TC向RM发送回滚

## 4.io.seata.spring.boot.autoconfigure.SeataAutoConfiguration|扫描核心类
```java
public class SeataAutoConfiguration {
    private static final Logger LOGGER = LoggerFactory.getLogger(SeataAutoConfiguration.class);

    @Bean(BEAN_NAME_FAILURE_HANDLER)
    @ConditionalOnMissingBean(FailureHandler.class)
    public FailureHandler failureHandler() {
        return new DefaultFailureHandlerImpl();
    }

    @Bean
    @DependsOn({BEAN_NAME_SPRING_APPLICATION_CONTEXT_PROVIDER, BEAN_NAME_FAILURE_HANDLER})
    @ConditionalOnMissingBean(GlobalTransactionScanner.class)
    public GlobalTransactionScanner globalTransactionScanner(SeataProperties seataProperties, FailureHandler failureHandler) {
        if (LOGGER.isInfoEnabled()) {
            LOGGER.info("Automatically configure Seata");
        }
       	// 扫描类且继承AbstractAutoProxyCreator，spring自身的事务也继承AbstractAutoProxyCreator
        return new GlobalTransactionScanner(seataProperties.getApplicationId(), seataProperties.getTxServiceGroup(), failureHandler);
    }


}
```
## 5.io.seata.spring.annotation.GlobalTransactionScanner#wrapIfNecessary|AOP
```java
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
    try {
        synchronized (PROXYED_SET) {
            if (PROXYED_SET.contains(beanName)) {
                return bean;
            }
            interceptor = null;
            //check TCC proxy
            if (TCCBeanParserUtils.isTccAutoProxy(bean, beanName, applicationContext)) {
                //TCC interceptor, proxy bean of sofa:reference/dubbo:reference, and LocalTCC
                interceptor = new TccActionInterceptor(TCCBeanParserUtils.getRemotingDesc(beanName));
                ConfigurationCache.addConfigListener(ConfigurationKeys.DISABLE_GLOBAL_TRANSACTION,
                    (ConfigurationChangeListener)interceptor);
            } else {
                Class<?> serviceInterface = SpringProxyUtils.findTargetClass(bean);
                Class<?>[] interfacesIfJdk = SpringProxyUtils.findInterfaces(bean);

                if (!existsAnnotation(new Class[]{serviceInterface})
                    && !existsAnnotation(interfacesIfJdk)) {
                    return bean;
                }

                if (globalTransactionalInterceptor == null) {
                	// 方法拦截器，执行的时候调用其invoke。与spring事务TransactionInterceptor起类似作用
                    globalTransactionalInterceptor = new GlobalTransactionalInterceptor(failureHandlerHook);
                    ConfigurationCache.addConfigListener(
                        ConfigurationKeys.DISABLE_GLOBAL_TRANSACTION,
                        (ConfigurationChangeListener)globalTransactionalInterceptor);
                }
                interceptor = globalTransactionalInterceptor;
            }

            LOGGER.info("Bean[{}] with name [{}] would use interceptor [{}]", bean.getClass().getName(), beanName, interceptor.getClass().getName());
            if (!AopUtils.isAopProxy(bean)) {
                bean = super.wrapIfNecessary(bean, beanName, cacheKey);
            } else {
                AdvisedSupport advised = SpringProxyUtils.getAdvisedSupport(bean);
                Advisor[] advisor = buildAdvisors(beanName, getAdvicesAndAdvisorsForBean(null, null, null));
                for (Advisor avr : advisor) {
                    advised.addAdvisor(0, avr);
                }
            }
            PROXYED_SET.add(beanName);
            return bean;
        }
    } catch (Exception exx) {
        throw new RuntimeException(exx);
    }
}
```
## 6.io.seata.spring.annotation.GlobalTransactionalInterceptor#invoke
```java
public Object invoke(final MethodInvocation methodInvocation) throws Throwable {
    Class<?> targetClass =
        methodInvocation.getThis() != null ? AopUtils.getTargetClass(methodInvocation.getThis()) : null;
    Method specificMethod = ClassUtils.getMostSpecificMethod(methodInvocation.getMethod(), targetClass);
    if (specificMethod != null && !specificMethod.getDeclaringClass().equals(Object.class)) {
        final Method method = BridgeMethodResolver.findBridgedMethod(specificMethod);
        final GlobalTransactional globalTransactionalAnnotation =
            getAnnotation(method, targetClass, GlobalTransactional.class);
        final GlobalLock globalLockAnnotation = getAnnotation(method, targetClass, GlobalLock.class);
        boolean localDisable = disable || (degradeCheck && degradeNum >= degradeCheckAllowTimes);
        if (!localDisable) {
            if (globalTransactionalAnnotation != null) {
            	// 全局事务核心逻辑
                return handleGlobalTransaction(methodInvocation, globalTransactionalAnnotation);
            } else if (globalLockAnnotation != null) {
                return handleGlobalLock(methodInvocation, globalLockAnnotation);
            }
        }
    }
    return methodInvocation.proceed();
}
```
### 6.1io.seata.spring.annotation.GlobalTransactionalInterceptor#handleGlobalTransaction
```java
Object handleGlobalTransaction(final MethodInvocation methodInvocation,
    final GlobalTransactional globalTrxAnno) throws Throwable {
    boolean succeed = true;
    try {
    	// 核心流程
        return transactionalTemplate.execute(new TransactionalExecutor() {
            @Override
            public Object execute() throws Throwable {
                return methodInvocation.proceed();
            }

            public String name() {
                String name = globalTrxAnno.name();
                if (!StringUtils.isNullOrEmpty(name)) {
                    return name;
                }
                return formatMethod(methodInvocation.getMethod());
            }

            @Override
            public TransactionInfo getTransactionInfo() {
                // reset the value of timeout
                int timeout = globalTrxAnno.timeoutMills();
                if (timeout <= 0 || timeout == DEFAULT_GLOBAL_TRANSACTION_TIMEOUT) {
                    timeout = defaultGlobalTransactionTimeout;
                }

                TransactionInfo transactionInfo = new TransactionInfo();
                transactionInfo.setTimeOut(timeout);
                transactionInfo.setName(name());
                transactionInfo.setPropagation(globalTrxAnno.propagation());
                transactionInfo.setLockRetryInternal(globalTrxAnno.lockRetryInternal());
                transactionInfo.setLockRetryTimes(globalTrxAnno.lockRetryTimes());
                Set<RollbackRule> rollbackRules = new LinkedHashSet<>();
                for (Class<?> rbRule : globalTrxAnno.rollbackFor()) {
                    rollbackRules.add(new RollbackRule(rbRule));
                }
                for (String rbRule : globalTrxAnno.rollbackForClassName()) {
                    rollbackRules.add(new RollbackRule(rbRule));
                }
                for (Class<?> rbRule : globalTrxAnno.noRollbackFor()) {
                    rollbackRules.add(new NoRollbackRule(rbRule));
                }
                for (String rbRule : globalTrxAnno.noRollbackForClassName()) {
                    rollbackRules.add(new NoRollbackRule(rbRule));
                }
                transactionInfo.setRollbackRules(rollbackRules);
                return transactionInfo;
            }
        });
    } catch (TransactionalExecutor.ExecutionException e) {
        TransactionalExecutor.Code code = e.getCode();
        switch (code) {
            case RollbackDone:
                throw e.getOriginalException();
            case BeginFailure:
                succeed = false;
                failureHandler.onBeginFailure(e.getTransaction(), e.getCause());
                throw e.getCause();
            case CommitFailure:
                succeed = false;
                failureHandler.onCommitFailure(e.getTransaction(), e.getCause());
                throw e.getCause();
            case RollbackFailure:
                failureHandler.onRollbackFailure(e.getTransaction(), e.getOriginalException());
                throw e.getOriginalException();
            case RollbackRetrying:
                failureHandler.onRollbackRetrying(e.getTransaction(), e.getOriginalException());
                throw e.getOriginalException();
            default:
                throw new ShouldNeverHappenException(String.format("Unknown TransactionalExecutor.Code: %s", code));
        }
    } finally {
        if (degradeCheck) {
            EVENT_BUS.post(new DegradeCheckEvent(succeed));
        }
    }
}
```
#### 6.1.1io.seata.tm.api.TransactionalTemplate#execute
```java
public Object execute(TransactionalExecutor business) throws Throwable {
    // 1. Get transactionInfo
    TransactionInfo txInfo = business.getTransactionInfo();
    if (txInfo == null) {
        throw new ShouldNeverHappenException("transactionInfo does not exist");
    }
    // 1.1 Get current transaction, if not null, the tx role is 'GlobalTransactionRole.Participant'.
    GlobalTransaction tx = GlobalTransactionContext.getCurrent();

    // 1.2 Handle the transaction propagation.
    Propagation propagation = txInfo.getPropagation();
    SuspendedResourcesHolder suspendedResourcesHolder = null;
    try {
    	// 事务级别
        switch (propagation) {
            case NOT_SUPPORTED:
                // If transaction is existing, suspend it.
                if (existingTransaction(tx)) {
                    suspendedResourcesHolder = tx.suspend();
                }
                // Execute without transaction and return.
                return business.execute();
            case REQUIRES_NEW:
                // If transaction is existing, suspend it, and then begin new transaction.
                if (existingTransaction(tx)) {
                    suspendedResourcesHolder = tx.suspend();
                    tx = GlobalTransactionContext.createNew();
                }
                // Continue and execute with new transaction
                break;
            case SUPPORTS:
                // If transaction is not existing, execute without transaction.
                if (notExistingTransaction(tx)) {
                    return business.execute();
                }
                // Continue and execute with new transaction
                break;
            case REQUIRED:
                // If current transaction is existing, execute with current transaction,
                // else continue and execute with new transaction.
                break;
            case NEVER:
                // If transaction is existing, throw exception.
                if (existingTransaction(tx)) {
                    throw new TransactionException(
                        String.format("Existing transaction found for transaction marked with propagation 'never', xid = %s"
                                , tx.getXid()));
                } else {
                    // Execute without transaction and return.
                    return business.execute();
                }
            case MANDATORY:
                // If transaction is not existing, throw exception.
                if (notExistingTransaction(tx)) {
                    throw new TransactionException("No existing transaction found for transaction marked with propagation 'mandatory'");
                }
                // Continue and execute with current transaction.
                break;
            default:
                throw new TransactionException("Not Supported Propagation:" + propagation);
        }

        // 1.3 If null, create new transaction with role 'GlobalTransactionRole.Launcher'.
        if (tx == null) {
            tx = GlobalTransactionContext.createNew();
        }

        // set current tx config to holder
        GlobalLockConfig previousConfig = replaceGlobalLockConfig(txInfo);

        try {
            // 2. If the tx role is 'GlobalTransactionRole.Launcher', send the request of beginTransaction to TC,
            //    else do nothing. Of course, the hooks will still be triggered.
            // 核心逻辑开启事务
            beginTransaction(txInfo, tx);

            Object rs;
            try {
                // Do Your Business
                rs = business.execute();
            } catch (Throwable ex) {
                // 3. The needed business exception to rollback.
                completeTransactionAfterThrowing(txInfo, tx, ex);
                throw ex;
            }

            // 4. everything is fine, commit.
            commitTransaction(tx);

            return rs;
        } finally {
            //5. clear
            resumeGlobalLockConfig(previousConfig);
            triggerAfterCompletion();
            cleanUp();
        }
    } finally {
        // If the transaction is suspended, resume it.
        if (suspendedResourcesHolder != null) {
            tx.resume(suspendedResourcesHolder);
        }
    }
}
```
## 7.io.seata.tm.api.TransactionalTemplate#beginTransaction|TM提交事务
```java
private void beginTransaction(TransactionInfo txInfo, GlobalTransaction tx) throws TransactionalExecutor.ExecutionException {
    try {
        triggerBeforeBegin();
        tx.begin(txInfo.getTimeOut(), txInfo.getName());
        triggerAfterBegin();
    } catch (TransactionException txe) {
        throw new TransactionalExecutor.ExecutionException(tx, txe,
            TransactionalExecutor.Code.BeginFailure);

    }
}
```
- io.seata.tm.api.DefaultGlobalTransaction#begin(int, java.lang.String)
```java
public void begin(int timeout, String name) throws TransactionException {
	// 如果事务不等于发起者即如果等于参与者，校验xid不等于null
    if (role != GlobalTransactionRole.Launcher) {
        assertXIDNotNull();
        if (LOGGER.isDebugEnabled()) {
            LOGGER.debug("Ignore Begin(): just involved in global transaction [{}]", xid);
        }
        return;
    }
    assertXIDNull();
    // feign在com.alibaba.cloud.seata.feign.SeataFeignClient#getModifyRequest中通过header传递
    String currentXid = RootContext.getXID();
    if (currentXid != null) {
        throw new IllegalStateException("Global transaction already exists," +
            " can't begin a new global transaction, currentXid = " + currentXid);
    }
    // 获取xid
    xid = transactionManager.begin(null, null, name, timeout);
    status = GlobalStatus.Begin;
    RootContext.bind(xid);
    if (LOGGER.isInfoEnabled()) {
        LOGGER.info("Begin new global transaction [{}]", xid);
    }
}
```
- io.seata.tm.DefaultTransactionManager#begin
```java
public String begin(String applicationId, String transactionServiceGroup, String name, int timeout)
    throws TransactionException {
    GlobalBeginRequest request = new GlobalBeginRequest();
    request.setTransactionName(name);
    request.setTimeout(timeout);
    GlobalBeginResponse response = (GlobalBeginResponse) syncCall(request);
    if (response.getResultCode() == ResultCode.Failed) {
        throw new TmTransactionException(TransactionExceptionCode.BeginFailed, response.getMsg());
    }
    return response.getXid();
}
```
- io.seata.tm.DefaultTransactionManager#syncCall
```java
private AbstractTransactionResponse syncCall(AbstractTransactionRequest request) throws TransactionException {
    try {
    	// 通过netty调用
        return (AbstractTransactionResponse) TmNettyRemotingClient.getInstance().sendSyncRequest(request);
    } catch (TimeoutException toe) {
        throw new TmTransactionException(TransactionExceptionCode.IO, "RPC timeout", toe);
    }
}
```
## 8.io.seata.server.coordinator.DefaultCoordinator#doGlobalBegin|返回信息同时持久化
io.seata.core.model.TransactionManager有多种实现类进行持久化
```java
protected void doGlobalBegin(GlobalBeginRequest request, GlobalBeginResponse response, RpcContext rpcContext)
        throws TransactionException {
        	// xid是雪花算法
    response.setXid(core.begin(rpcContext.getApplicationId(), rpcContext.getTransactionServiceGroup(),
            request.getTransactionName(), request.getTimeout()));
    if (LOGGER.isInfoEnabled()) {
        LOGGER.info("Begin new global transaction applicationId: {},transactionServiceGroup: {}, transactionName: {},timeout:{},xid:{}",
                rpcContext.getApplicationId(), rpcContext.getTransactionServiceGroup(), request.getTransactionName(), request.getTimeout(), response.getXid());
    }
}
```
## 9.io.seata.rm.datasource.ConnectionProxy#commit|代理connection分支逻辑提交(本地也会调)
```java
public void commit() throws SQLException {
    try {
    	// 锁冲突策略会自旋30次再抛异常
        LOCK_RETRY_POLICY.execute(() -> {
            doCommit();
            return null;
        });
    } catch (SQLException e) {
        if (targetConnection != null && !getAutoCommit() && !getContext().isAutoCommitChanged()) {
            rollback();
        }
        throw e;
    } catch (Exception e) {
        throw new SQLException(e);
    }
}
```
- io.seata.rm.datasource.ConnectionProxy#doCommit
```java
private void doCommit() throws SQLException {
    if (context.inGlobalTransaction()) {
        processGlobalTransactionCommit();
    } else if (context.isGlobalLockRequire()) {
        processLocalCommitWithGlobalLocks();
    } else {
        targetConnection.commit();
    }
}
```
### 9.1io.seata.rm.datasource.ConnectionProxy#processGlobalTransactionCommit
```java
private void processGlobalTransactionCommit() throws SQLException {
    try {
        register();
    } catch (TransactionException e) {
        recognizeLockKeyConflictException(e, context.buildLockKeys());
    }
    try {
    	// 插入undolog日志
        UndoLogManagerFactory.getUndoLogManager(this.getDbType()).flushUndoLogs(this);
        targetConnection.commit();
    } catch (Throwable ex) {
        LOGGER.error("process connectionProxy commit error: {}", ex.getMessage(), ex);
        report(false);
        throw new SQLException(ex);
    }
    if (IS_REPORT_SUCCESS_ENABLE) {
        report(true);
    }
    context.reset();
}
```
- io.seata.rm.datasource.ConnectionProxy#register
```java
private void register() throws TransactionException {
    if (!context.hasUndoLog() || !context.hasLockKey()) {
        return;
    }
    // 分支事务注册，向分支事务表插入信息
    Long branchId = DefaultResourceManager.get().branchRegister(BranchType.AT, getDataSourceProxy().getResourceId(),
        null, context.getXid(), null, context.buildLockKeys());
    context.setBranchId(branchId);
}
```
### 10.io.seata.rm.datasource.PreparedStatementProxy#execute|代理PreparedStatement
- io.seata.rm.datasource.exec.ExecuteTemplate#execute(java.util.List<io.seata.sqlparser.SQLRecognizer>, io.seata.rm.datasource.StatementProxy<S>, io.seata.rm.datasource.exec.StatementCallback<T,S>, java.lang.Object...)
```java
public static <T, S extends Statement> T execute(List<SQLRecognizer> sqlRecognizers,
                                                     StatementProxy<S> statementProxy,
                                                     StatementCallback<T, S> statementCallback,
                                                     Object... args) throws SQLException {
	// 判断全局事务
    if (!RootContext.requireGlobalLock() && BranchType.AT != RootContext.getBranchType()) {
        // Just work as original statement
        // 普通逻辑
        return statementCallback.execute(statementProxy.getTargetStatement(), args);
    }

    String dbType = statementProxy.getConnectionProxy().getDbType();
    if (CollectionUtils.isEmpty(sqlRecognizers)) {
        sqlRecognizers = SQLVisitorFactory.get(
                statementProxy.getTargetSQL(),
                dbType);
    }
    Executor<T> executor;
    if (CollectionUtils.isEmpty(sqlRecognizers)) {
        executor = new PlainExecutor<>(statementProxy, statementCallback);
    } else {
        if (sqlRecognizers.size() == 1) {
        	// sql识别器
            SQLRecognizer sqlRecognizer = sqlRecognizers.get(0);
            switch (sqlRecognizer.getSQLType()) {
                case INSERT:
                    executor = EnhancedServiceLoader.load(InsertExecutor.class, dbType,
                            new Class[]{StatementProxy.class, StatementCallback.class, SQLRecognizer.class},
                            new Object[]{statementProxy, statementCallback, sqlRecognizer});
                    break;
                case UPDATE:
                    executor = new UpdateExecutor<>(statementProxy, statementCallback, sqlRecognizer);
                    break;
                case DELETE:
                    executor = new DeleteExecutor<>(statementProxy, statementCallback, sqlRecognizer);
                    break;
                case SELECT_FOR_UPDATE:
                    executor = new SelectForUpdateExecutor<>(statementProxy, statementCallback, sqlRecognizer);
                    break;
                default:
                    executor = new PlainExecutor<>(statementProxy, statementCallback);
                    break;
            }
        } else {
            executor = new MultiExecutor<>(statementProxy, statementCallback, sqlRecognizers);
        }
    }
    T rs;
    try {
        rs = executor.execute(args);
    } catch (Throwable ex) {
        if (!(ex instanceof SQLException)) {
            // Turn other exception into SQLException
            ex = new SQLException(ex);
        }
        throw (SQLException) ex;
    }
    return rs;
}
```
- io.seata.rm.datasource.exec.BaseTransactionalExecutor#execute
```java
public T execute(Object... args) throws Throwable {
    String xid = RootContext.getXID();
    if (xid != null) {
        statementProxy.getConnectionProxy().bind(xid);
    }

    statementProxy.getConnectionProxy().setGlobalLockRequire(RootContext.requireGlobalLock());
    return doExecute(args);
}
```
- io.seata.rm.datasource.exec.AbstractDMLBaseExecutor#doExecute
```java
public T doExecute(Object... args) throws Throwable {
    AbstractConnectionProxy connectionProxy = statementProxy.getConnectionProxy();
    if (connectionProxy.getAutoCommit()) {
    	// 内部会取消自动提交再走executeAutoCommitFalse
        return executeAutoCommitTrue(args);
    } else {
        return executeAutoCommitFalse(args);
    }
}
```
- io.seata.rm.datasource.exec.AbstractDMLBaseExecutor#executeAutoCommitFalse
```java
protected T executeAutoCommitFalse(Object[] args) throws Exception {
    if (!JdbcConstants.MYSQL.equalsIgnoreCase(getDbType()) && isMultiPk()) {
        throw new NotSupportYetException("multi pk only support mysql!");
    }
    // 前置镜像，根据insert、update、delete不同的方式组装出查询sql，并得到结果(如果是update则会生成 for update保证本地事务行锁)
    TableRecords beforeImage = beforeImage();
    T result = statementCallback.execute(statementProxy.getTargetStatement(), args);
    // 后置镜像，读未提交
    TableRecords afterImage = afterImage(beforeImage);
    prepareUndoLog(beforeImage, afterImage);
    return result;
}
```
## 11.io.seata.server.coordinator.DefaultCore#branchRegister|全局锁
```java
public Long branchRegister(BranchType branchType, String resourceId, String clientId, String xid,
                           String applicationData, String lockKeys) throws TransactionException {
    return getCore(branchType).branchRegister(branchType, resourceId, clientId, xid,
        applicationData, lockKeys);
}
```
- io.seata.server.coordinator.AbstractCore#branchRegister
```java
public Long branchRegister(BranchType branchType, String resourceId, String clientId, String xid,
                               String applicationData, String lockKeys) throws TransactionException {
    GlobalSession globalSession = assertGlobalSessionNotNull(xid, false);
    return SessionHolder.lockAndExecute(globalSession, () -> {
    	// 检查全局事务状态
        globalSessionStatusCheck(globalSession);
        // 增加监听
        globalSession.addSessionLifecycleListener(SessionHolder.getRootSessionManager());
        // 获取一个分支事务信息
        BranchSession branchSession = SessionHelper.newBranchByGlobal(globalSession, branchType, resourceId,
                applicationData, lockKeys, clientId);
        MDC.put(RootContext.MDC_KEY_BRANCH_ID, String.valueOf(branchSession.getBranchId()));
        // 加锁
        branchSessionLock(globalSession, branchSession);
        try {
            globalSession.addBranch(branchSession);
        } catch (RuntimeException ex) {
            branchSessionUnlock(branchSession);
            throw new BranchTransactionException(FailedToAddBranch, String
                    .format("Failed to store branch xid = %s branchId = %s", globalSession.getXid(),
                            branchSession.getBranchId()), ex);
        }
        if (LOGGER.isInfoEnabled()) {
            LOGGER.info("Register branch successfully, xid = {}, branchId = {}, resourceId = {} ,lockKeys = {}",
                globalSession.getXid(), branchSession.getBranchId(), resourceId, lockKeys);
        }
        return branchSession.getBranchId();
    });
}
```
- branchSession.lock(autoCommit, skipCheckLock)->io.seata.server.session.BranchSession#lock(boolean, boolean)
```java
public boolean lock(boolean autoCommit, boolean skipCheckLock) throws TransactionException {
    if (this.getBranchType().equals(BranchType.AT)) {
        return LockerManagerFactory.getLockManager().acquireLock(this, autoCommit, skipCheckLock);
    }
    return true;
}
```
- io.seata.server.lock.AbstractLockManager#acquireLock(io.seata.server.session.BranchSession, boolean, boolean)行锁收集
```java
public boolean acquireLock(BranchSession branchSession, boolean autoCommit, boolean skipCheckLock) throws TransactionException {
    if (branchSession == null) {
        throw new IllegalArgumentException("branchSession can't be null for memory/file locker.");
    }
    String lockKey = branchSession.getLockKey();
    if (StringUtils.isNullOrEmpty(lockKey)) {
        // no lock
        return true;
    }
    // get locks of branch
    // 行锁收集，如果是update多行就会生成多行的行锁
    List<RowLock> locks = collectRowLocks(branchSession);
    if (CollectionUtils.isEmpty(locks)) {
        // no lock
        return true;
    }
    return getLocker(branchSession).acquireLock(locks, autoCommit, skipCheckLock);
}
```