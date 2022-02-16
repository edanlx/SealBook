# 【JAVA分布式】01-spring扫描BeanDefinition
> 欢迎关注b站账号/公众号【六边形战士夏宁】，一个要把各项指标拉满的男人。该文章已在[github目录](https://github.com/edanlx/SealBook/blob/master/catalogue/wechat.md)收录。
屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。

## 1.背景
zookeeper是Apache下Hadoop的子项目，分布式协调框架。用来解决统一命名服务，状态同步服务，集群管理，分布式应用配置的项目管理等

## 3.acquire非公平独占锁
```java
interProcessMutex.acquire()
```
```java
public void acquire() throws Exception
{
    if ( !internalLock(-1, null) )
    {
        throw new IOException("Lost connection while trying to acquire lock: " + basePath);
    }
}
```
```java
private boolean internalLock(long time, TimeUnit unit) throws Exception
{
    /*
       Note on concurrency: a given lockData instance
       can be only acted on by a single thread so locking isn't necessary
    */

    Thread currentThread = Thread.currentThread();
    // 查询是否有锁
    LockData lockData = threadData.get(currentThread);
    if ( lockData != null )
    {
        // re-entering
        // 可重入锁
        lockData.lockCount.incrementAndGet();
        return true;
    }
    // 加锁
    String lockPath = internals.attemptLock(time, unit, getLockNodeBytes());
    if ( lockPath != null )
    {
        // 加锁成功
        LockData newLockData = new LockData(currentThread, lockPath);
        threadData.put(currentThread, newLockData);
        return true;
    }

    return false;
}
```
### 3.1acquire
```java
String attemptLock(long time, TimeUnit unit, byte[] lockNodeBytes) throws Exception
{
    final long      startMillis = System.currentTimeMillis();
    final Long      millisToWait = (unit != null) ? unit.toMillis(time) : null;
    final byte[]    localLockNodeBytes = (revocable.get() != null) ? new byte[0] : lockNodeBytes;
    int             retryCount = 0;

    String          ourPath = null;
    boolean         hasTheLock = false;
    boolean         isDone = false;
    while ( !isDone )
    {
        isDone = true;

        try
        {
            // 加锁逻辑,获得创建好的锁的路径
            ourPath = driver.createsTheLock(client, path, localLockNodeBytes);
            hasTheLock = internalLockLoop(startMillis, millisToWait, ourPath);
        }
        catch ( KeeperException.NoNodeException e )
        {
            // gets thrown by StandardLockInternalsDriver when it can't find the lock node
            // this can happen when the session expires, etc. So, if the retry allows, just try it all again
            if ( client.getZookeeperClient().getRetryPolicy().allowRetry(retryCount++, System.currentTimeMillis() - startMillis, RetryLoop.getDefaultRetrySleeper()) )
            {
                isDone = false;
            }
            else
            {
                throw e;
            }
        }
    }

    if ( hasTheLock )
    {
        return ourPath;
    }

    return null;
}
```

### 3.1.1createsTheLock
```java
public String createsTheLock(CuratorFramework client, String path, byte[] lockNodeBytes) throws Exception
{
    String ourPath;
    // withProtection防止创建后服务端与客户端通信失败而重复创建，UUID进行幂等校验
    // creatingParentContainersIfNeeded如果没有则创建容器节点
    // CreateMode.EPHEMERAL_SEQUENTIAL同时创建临时顺序节点
    if ( lockNodeBytes != null )
    {
        ourPath = client.create().creatingParentContainersIfNeeded().withProtection().withMode(CreateMode.EPHEMERAL_SEQUENTIAL).forPath(path, lockNodeBytes);
    }
    else
    {
        ourPath = client.create().creatingParentContainersIfNeeded().withProtection().withMode(CreateMode.EPHEMERAL_SEQUENTIAL).forPath(path);
    }
    return ourPath;
}
```

### 3.1.2internalLockLoop
```java
private boolean internalLockLoop(long startMillis, Long millisToWait, String ourPath) throws Exception
{
    boolean     haveTheLock = false;
    boolean     doDelete = false;
    try
    {
        if ( revocable.get() != null )
        {
            client.getData().usingWatcher(revocableWatcher).forPath(ourPath);
        }

        while ( (client.getState() == CuratorFrameworkState.STARTED) && !haveTheLock )
        {
            // 获取所有子节点并进行排序
            List<String>        children = getSortedChildren();
            // 截取获得子节点名称
            String              sequenceNodeName = ourPath.substring(basePath.length() + 1); // +1 to include the slash
            // 独占锁maxLeases=1
            // 判断是否加锁成功，如果失败则获取前一个的编号
            PredicateResults    predicateResults = driver.getsTheLock(client, children, sequenceNodeName, maxLeases);
            if ( predicateResults.getsTheLock() )
            {
                // 标记成功加锁
                haveTheLock = true;
            }
            else
            {
                // 监听前一个节点
                String  previousSequencePath = basePath + "/" + predicateResults.getPathToWatch();

                synchronized(this)
                {
                    try
                    {
                        // use getData() instead of exists() to avoid leaving unneeded watchers which is a type of resource leak
                        // watcher回调会调用LockInternals.this.notifyAll，然后重新进入该循环
                        client.getData().usingWatcher(watcher).forPath(previousSequencePath);
                        if ( millisToWait != null )
                        {
                            millisToWait -= (System.currentTimeMillis() - startMillis);
                            startMillis = System.currentTimeMillis();
                            if ( millisToWait <= 0 )
                            {
                                doDelete = true;    // timed out - delete our node
                                break;
                            }

                            wait(millisToWait);
                        }
                        else
                        {
                            wait();
                        }
                    }
                    catch ( KeeperException.NoNodeException e )
                    {
                        // it has been deleted (i.e. lock released). Try to acquire again
                    }
                }
            }
        }
    }
    catch ( Exception e )
    {
        ThreadUtils.checkInterrupted(e);
        doDelete = true;
        throw e;
    }
    finally
    {
        if ( doDelete )
        {
            deleteOurPath(ourPath);
        }
    }
    return haveTheLock;
}
```

### 3.1.2.1internalLockLoop
```java
public PredicateResults getsTheLock(CuratorFramework client, List<String> children, String sequenceNodeName, int maxLeases) throws Exception
{
    // 获取当前节点序号
    int             ourIndex = children.indexOf(sequenceNodeName);
    validateOurIndex(sequenceNodeName, ourIndex);
    // 当前节点是否为0，因为独占锁maxLeases=1
    boolean         getsTheLock = ourIndex < maxLeases;
    // 获取需要监听编号
    String          pathToWatch = getsTheLock ? null : children.get(ourIndex - maxLeases);

    return new PredicateResults(pathToWatch, getsTheLock);
}
```
## 4.InterProcessReadWriteLock
## 5.LeaderSelector
```java
public void start()
{
    Preconditions.checkState(state.compareAndSet(State.LATENT, State.STARTED), "Cannot be started more than once");

    Preconditions.checkState(!executorService.isShutdown(), "Already started");
    Preconditions.checkState(!hasLeadership, "Already has leadership");

    client.getConnectionStateListenable().addListener(listener);
    requeue();
}
```
```java
public boolean requeue()
{
    Preconditions.checkState(state.get() == State.STARTED, "close() has already been called");
    return internalRequeue();
}
```
```java
private synchronized boolean internalRequeue()
{
    if ( !isQueued && (state.get() == State.STARTED) )
    {
        isQueued = true;
        // 线程池
        Future<Void> task = executorService.submit(new Callable<Void>()
        {
            @Override
            public Void call() throws Exception
            {
                try
                {
                    // doWorkLoop->doWork->mutex.acquire()获取锁
                    doWorkLoop();
                }
                finally
                {
                    clearIsQueued();
                    if ( autoRequeue.get() )
                    {
                        internalRequeue();
                    }
                }
                return null;
            }
        });
        ourTask.set(task);

        return true;
    }
    return false;
}
```