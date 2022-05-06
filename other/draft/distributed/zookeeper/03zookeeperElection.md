# 【JAVA分布式】01-spring扫描BeanDefinition
> 欢迎关注b站账号/公众号【六边形战士夏宁】，一个要把各项指标拉满的男人。该文章已在[github目录](https://github.com/edanlx/SealBook/blob/master/catalogue/wechat.md)收录。
屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。

## 1.背景
ZooKeeperAtomic Broadcast
zookeeper是Apache下Hadoop的子项目，分布式协调框架。用来解决统一命名服务，状态同步服务，集群管理，分布式应用配置的项目管理等  
- zk选举，以两台机器为例，vote格式为(机器id,事务id,选举轮次)
1. zk启动先投自己一票
2. 每台机器获得其它机器投的票，例如机器1，第一轮投票为(1,0,1),收到选票为(2,0,1),比较后进入第二轮先比较事务id，再比较机器id，则机器1第二轮投票(2,0,1)

- 核心pk逻辑
1. 收到的轮次更高
2. 轮次相同时事务id更高
3. 轮次和事务相同时，机器id更高

- 关键易混变量
todo 图
|变量|意义|处理线程|相应线程逻辑|
|--|--|--|--|
|LinkedBlockingQueue\<ToSend\> sendqueue|发送选票队列|WorkerSender|如果是发给自己转入***recvQueue***发给别人转入***queueSendMap***|
|LinkedBlockingQueue\<Notification\> recvqueue|接收选票队列|WorkerReceiver|WorkerReceiver判断当前是否还在选举，如果在选举将收到的数据封装到recvqueue；如果WorkerReceiver判断没有在选举则将leader信息存入sendqueue。如果还在选举会在在lookForLeader中循环取出，本机如果还在选举则PK选票并发送给所有机器表示自己选票有更新，接着判断所有选票能否选出leader能够选出接着把收到的选票依次比较未发生变化则更改本机为leading/following|
|ConcurrentHashMap\<Long, SendWorker\> senderWorkerMap|维护向目标机器发送的socket，key为sid目标机器|||
|ConcurrentHashMap\<Long, BlockingQueue\<ByteBuffer\>\> queueSendMap|发送选票队列,key为sid,v为该sid的队列选票,此处虽然是队列但长度为1用于丢弃旧数据|||

- 事务id
大32位为周期，小32位为自增
```java
public static long makeZxid(long epoch, long counter) {
    return (epoch << 32L) | (counter & 0xffffffffL);
}
```

- 端口(server.1=127.0.0.1:2888:3888)
1. 2181 监听客户端的默认端口(NIO)
2. 2888 监听数据同步端口(BIO)
3. 3888 监听选举投票端口(BIO)
## 2.源码下载
```shell
https://github.com/apache/zookeeper.git
```
如果启动时有XXX包找不到，手动maven编译Jute。如果org.apache.zookeeper.version.Info找不到，则按照路径建立如下辅助类
```java
public interface Info {
    int MAJOR = 1;
    int MINOR = 0;
    int MICRO = 0;
    String QUALIFIER = null;
    int REVISION = -1;
    String REVISION_HASH = "1";
    String BUILD_DATE = "2020‐10‐15";
}
```
* 其它运行时找不到包的
如metrics-core、snappy-java把scope为provided都删除
## 3.源码目录
* bin
启动脚本
* conf
配置文件
* zookeeper-client
客户端源码
* zookeeper-jute
序列化代码
* zookeeper-recipes
示例代码
* zookeeper-server
服务端源码

## 4.zkServer.sh
在strat参数中有如下代码片段
```shell
 nohup "$JAVA" $ZOO_DATADIR_AUTOCREATE "-Dzookeeper.log.dir=${ZOO_LOG_DIR}" \
    "-Dzookeeper.log.file=${ZOO_LOG_FILE}" \
    -XX:+HeapDumpOnOutOfMemoryError -XX:OnOutOfMemoryError='kill -9 %p' \
    -cp "$CLASSPATH" $JVMFLAGS $ZOOMAIN "$ZOOCFG" > "$_ZOO_DAEMON_OUT" 2>&1 < /dev/null &
```
查找ZOOMAIN
```shell
ZOOMAIN="-Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.local.only=$JMXLOCALONLY org.apache.zookeeper.server.quorum.QuorumPeerMain"
```
则有启动类QuorumPeerMain

## 5.服务端启动
QuorumPeerMain.main()->initializeAndRun
```java
protected void initializeAndRun(String[] args) throws ConfigException, IOException, AdminServerException {
    QuorumPeerConfig config = new QuorumPeerConfig();
    // 启动的时候传入的参数即配置文件
    // parse()->parseProperties()
    if (args.length == 1) {
        // 解析配置文件，将配置文件的内容存至java内存
        config.parse(args[0]);
    }

    // Start and schedule the the purge task
    // 定期清理过期文件
    DatadirCleanupManager purgeMgr = new DatadirCleanupManager(
        config.getDataDir(),
        config.getDataLogDir(),
        config.getSnapRetainCount(),
        config.getPurgeInterval());
    purgeMgr.start();

    // 判断集群还是单机,两个方式最终都会进入到runFromConfig方法，判断方式即配置文件中配置机器数量
    // ZooKeeperServerMain.main->initializeAndRun()->runFromConfig()
    if (args.length == 1 && config.isDistributed()) {
        runFromConfig(config);
    } else {
        LOG.warn("Either no config or no quorum defined in config, running in standalone mode");
        // there is only server in the quorum -- run as standalone
        ZooKeeperServerMain.main(args);
    }
}
```

```java
public void runFromConfig(QuorumPeerConfig config) throws IOException, AdminServerException {
    try {
        // 日志
        ManagedUtil.registerLog4jMBeans();
    } catch (JMException e) {
        LOG.warn("Unable to register log4j JMX control", e);
    }

    LOG.info("Starting quorum peer, myid=" + config.getServerId());
    final MetricsProvider metricsProvider;
    try {
        metricsProvider = MetricsProviderBootstrap.startMetricsProvider(
            config.getMetricsProviderClassName(),
            config.getMetricsProviderConfiguration());
    } catch (MetricsProviderLifeCycleException error) {
        throw new IOException("Cannot boot MetricsProvider " + config.getMetricsProviderClassName(), error);
    }
    try {
        // 监控相关
        ServerMetrics.metricsProviderInitialized(metricsProvider);
        // authProvider权限相关
        ProviderRegistry.initialize();
        ServerCnxnFactory cnxnFactory = null;
        ServerCnxnFactory secureCnxnFactory = null;

        if (config.getClientPortAddress() != null) {
            cnxnFactory = ServerCnxnFactory.createFactory();
            // 将端口信息等传入cnxnFactory
            cnxnFactory.configure(config.getClientPortAddress(), config.getMaxClientCnxns(), config.getClientPortListenBacklog(), false);
        }
        // 安全相关
        if (config.getSecureClientPortAddress() != null) {
            secureCnxnFactory = ServerCnxnFactory.createFactory();
            secureCnxnFactory.configure(config.getSecureClientPortAddress(), config.getMaxClientCnxns(), config.getClientPortListenBacklog(), true);
        }

        quorumPeer = getQuorumPeer();
        quorumPeer.setTxnFactory(new FileTxnSnapLog(config.getDataLogDir(), config.getDataDir()));
        quorumPeer.enableLocalSessions(config.areLocalSessionsEnabled());
        quorumPeer.enableLocalSessionsUpgrading(config.isLocalSessionsUpgradingEnabled());
        //quorumPeer.setQuorumPeers(config.getAllMembers());
        // 选举类型，默认3
        quorumPeer.setElectionType(config.getElectionAlg());
        quorumPeer.setMyid(config.getServerId());
        quorumPeer.setTickTime(config.getTickTime());
        quorumPeer.setMinSessionTimeout(config.getMinSessionTimeout());
        quorumPeer.setMaxSessionTimeout(config.getMaxSessionTimeout());
        quorumPeer.setInitLimit(config.getInitLimit());
        quorumPeer.setSyncLimit(config.getSyncLimit());
        quorumPeer.setConnectToLearnerMasterLimit(config.getConnectToLearnerMasterLimit());
        quorumPeer.setObserverMasterPort(config.getObserverMasterPort());
        quorumPeer.setConfigFileName(config.getConfigFilename());
        quorumPeer.setClientPortListenBacklog(config.getClientPortListenBacklog());
        // 初始化基础内存数据结构
        quorumPeer.setZKDatabase(new ZKDatabase(quorumPeer.getTxnFactory()));
        quorumPeer.setQuorumVerifier(config.getQuorumVerifier(), false);
        if (config.getLastSeenQuorumVerifier() != null) {
            quorumPeer.setLastSeenQuorumVerifier(config.getLastSeenQuorumVerifier(), false);
        }
        quorumPeer.initConfigInZKDatabase();
        quorumPeer.setCnxnFactory(cnxnFactory);
        quorumPeer.setSecureCnxnFactory(secureCnxnFactory);
        quorumPeer.setSslQuorum(config.isSslQuorum());
        quorumPeer.setUsePortUnification(config.shouldUsePortUnification());
        quorumPeer.setLearnerType(config.getPeerType());
        quorumPeer.setSyncEnabled(config.getSyncEnabled());
        quorumPeer.setQuorumListenOnAllIPs(config.getQuorumListenOnAllIPs());
        if (config.sslQuorumReloadCertFiles) {
            quorumPeer.getX509Util().enableCertFileReloading();
        }
        quorumPeer.setMultiAddressEnabled(config.isMultiAddressEnabled());
        quorumPeer.setMultiAddressReachabilityCheckEnabled(config.isMultiAddressReachabilityCheckEnabled());
        quorumPeer.setMultiAddressReachabilityCheckTimeoutMs(config.getMultiAddressReachabilityCheckTimeoutMs());

        // sets quorum sasl authentication configurations
        quorumPeer.setQuorumSaslEnabled(config.quorumEnableSasl);
        if (quorumPeer.isQuorumSaslAuthEnabled()) {
            quorumPeer.setQuorumServerSaslRequired(config.quorumServerRequireSasl);
            quorumPeer.setQuorumLearnerSaslRequired(config.quorumLearnerRequireSasl);
            quorumPeer.setQuorumServicePrincipal(config.quorumServicePrincipal);
            quorumPeer.setQuorumServerLoginContext(config.quorumServerLoginContext);
            quorumPeer.setQuorumLearnerLoginContext(config.quorumLearnerLoginContext);
        }
        quorumPeer.setQuorumCnxnThreadsSize(config.quorumCnxnThreadsSize);
        quorumPeer.initialize();

        if (config.jvmPauseMonitorToRun) {
            quorumPeer.setJvmPauseMonitor(new JvmPauseMonitor(config));
        }

        quorumPeer.start();
        ZKAuditProvider.addZKStartStopAuditLog();
        quorumPeer.join();
    } catch (InterruptedException e) {
        // warn, but generally this is ok
        LOG.warn("Quorum Peer interrupted", e);
    } finally {
        try {
            metricsProvider.stop();
        } catch (Throwable error) {
            LOG.warn("Error while stopping metrics", error);
        }
    }
}
```

### 5.1ServerCnxnFactory.createFactory
```java
public static ServerCnxnFactory createFactory() throws IOException {
    // ZOOKEEPER_SERVER_CNXN_FACTORY=zookeeper.serverCnxnFactory
    String serverCnxnFactoryName = System.getProperty(ZOOKEEPER_SERVER_CNXN_FACTORY);
    // 如果没有配置该参数，则默认使用NIOServerCnxnFactory,即默认使用NIO,一般配置成NettyServerCnxn
    if (serverCnxnFactoryName == null) {
        serverCnxnFactoryName = NIOServerCnxnFactory.class.getName();
    }
    try {
        // 反射生成实体类
        ServerCnxnFactory serverCnxnFactory = (ServerCnxnFactory) Class.forName(serverCnxnFactoryName)
                                                                       .getDeclaredConstructor()
                                                                       .newInstance();
        LOG.info("Using {} as server connection factory", serverCnxnFactoryName);
        return serverCnxnFactory;
    } catch (Exception e) {
        IOException ioe = new IOException("Couldn't instantiate " + serverCnxnFactoryName, e);
        throw ioe;
    }
}
```

### 5.2ZKDatabase
ZKDatabase->createDataTree()->DataTree()->new NodeHashMapImpl
```java
// 左边String是路径，右边是对象
private final ConcurrentHashMap<String, DataNode> nodes;
public NodeHashMapImpl(DigestCalculator digestCalculator) {
    this.digestCalculator = digestCalculator;
    nodes = new ConcurrentHashMap<>();
    hash = new AdHash();
    digestEnabled = ZooKeeperServer.isDigestEnabled();
}
```
```java
public class DataNode implements Record {
// the digest value of this node, calculated from path, data and stat
    private volatile long digest;

    // indicate if the digest of this node is up to date or not, used to
    // optimize the performance.
    volatile boolean digestCached;

    /** the data for this datanode */
    // 用户数据
    byte[] data;

    /**
     * the acl map long for this datanode. the datatree has the map
     */
    // 权限
    Long acl;

    /**
     * the stat for this node that is persisted to disk.
     */
    public StatPersisted stat;
    ...
}
```

### 5.3start
```java
public synchronized void start() {
    if (!getView().containsKey(myid)) {
        throw new RuntimeException("My id " + myid + " not in the peer list");
    }
    // loadDataBase()->zkDb.loadDataBase->snapLog.restore()->FileTxnLog txnLog = new FileTxnLog(dataDir);
    // 加载配置文件
    loadDataBase();
    // 启动ServerCnxnFactory
    startServerCnxnFactory();
    try {
        // 服务端状态信息使用jetty容器，默认8080，注意此处直接catch，不影响任何流程
        adminServer.start();
    } catch (AdminServerException e) {
        LOG.warn("Problem starting AdminServer", e);
    }
    // 从名称既可以看到启动选举流程
    startLeaderElection();
    startJvmPauseMonitor();
    // 因为当前继承的Thread,所以就是启动当前类的run
    super.start();
}
```

### 5.3.1start
```java
private void startServerCnxnFactory() {
    // 此处为之前设置的cnx启动
    if (cnxnFactory != null) {
        cnxnFactory.start();
    }
    if (secureCnxnFactory != null) {
        secureCnxnFactory.start();
    }
}
```
### 5.3.1.1NettyServerCnxnFactory.start
```java
public void start() {
    if (listenBacklog != -1) {
        bootstrap.option(ChannelOption.SO_BACKLOG, listenBacklog);
    }
    LOG.info("binding to port {}", localAddress);
    // netty启动服务端监听 2181
    parentChannel = bootstrap.bind(localAddress).syncUninterruptibly().channel();
    // Port changes after bind() if the original port was 0, update
    // localAddress to get the real port.
    localAddress = (InetSocketAddress) parentChannel.localAddress();
    LOG.info("bound to port {}", getLocalPort());
}
```
### 5.3.2super.start
```java
public void run() {
    updateThreadName();

    LOG.debug("Starting quorum peer");
    try {
        // jmx监控
        jmxQuorumBean = new QuorumBean(this);
        MBeanRegistry.getInstance().register(jmxQuorumBean, null);
        for (QuorumServer s : getView().values()) {
            ZKMBeanInfo p;
            if (getId() == s.id) {
                p = jmxLocalPeerBean = new LocalPeerBean(this);
                try {
                    MBeanRegistry.getInstance().register(p, jmxQuorumBean);
                } catch (Exception e) {
                    LOG.warn("Failed to register with JMX", e);
                    jmxLocalPeerBean = null;
                }
            } else {
                RemotePeerBean rBean = new RemotePeerBean(this, s);
                try {
                    MBeanRegistry.getInstance().register(rBean, jmxQuorumBean);
                    jmxRemotePeerBean.put(s.id, rBean);
                } catch (Exception e) {
                    LOG.warn("Failed to register with JMX", e);
                }
            }
        }
    } catch (Exception e) {
        LOG.warn("Failed to register with JMX", e);
        jmxQuorumBean = null;
    }

    try {
        /*
         * Main loop
         */
        while (running) {
            if (unavailableStartTime == 0) {
                unavailableStartTime = Time.currentElapsedTime();
            }

            // 获取当前状态，启动默认LOOKING
            switch (getPeerState()) {
            case LOOKING:
                // 转到大目录LOOKING分支
                LOG.info("LOOKING");
                ServerMetrics.getMetrics().LOOKING_COUNT.add(1);
                // 只读模式，一般不启用
                if (Boolean.getBoolean("readonlymode.enabled")) {
                    ...
                } else {
                    try {
                        // 重置configFlag标记
                        reconfigFlagClear();
                        // 默认false，某些特殊情况监听选票没有开启在此处开启
                        if (shuttingDownLE) {
                            shuttingDownLE = false;
                            startLeaderElection();
                        }
                        // 设置当前投票
                        setCurrentVote(makeLEStrategy().lookForLeader());
                    } catch (Exception e) {
                        LOG.warn("Unexpected exception", e);
                        setPeerState(ServerState.LOOKING);
                    }
                }
                break;
            case OBSERVING:
                try {
                    LOG.info("OBSERVING");
                    setObserver(makeObserver(logFactory));
                    observer.observeLeader();
                } catch (Exception e) {
                    LOG.warn("Unexpected exception", e);
                } finally {
                    observer.shutdown();
                    setObserver(null);
                    updateServerState();

                    // Add delay jitter before we switch to LOOKING
                    // state to reduce the load of ObserverMaster
                    if (isRunning()) {
                        Observer.waitForObserverElectionDelay();
                    }
                }
                break;
            case FOLLOWING:
                try {
                    LOG.info("FOLLOWING");
                    setFollower(makeFollower(logFactory));
                    follower.followLeader();
                } catch (Exception e) {
                    LOG.warn("Unexpected exception", e);
                } finally {
                    follower.shutdown();
                    setFollower(null);
                    updateServerState();
                }
                break;
            case LEADING:
                LOG.info("LEADING");
                try {
                    setLeader(makeLeader(logFactory));
                    leader.lead();
                    setLeader(null);
                } catch (Exception e) {
                    LOG.warn("Unexpected exception", e);
                } finally {
                    if (leader != null) {
                        leader.shutdown("Forcing shutdown");
                        setLeader(null);
                    }
                    updateServerState();
                }
                break;
            }
        }
    } finally {
        LOG.warn("QuorumPeer main thread exited");
        MBeanRegistry instance = MBeanRegistry.getInstance();
        instance.unregister(jmxQuorumBean);
        instance.unregister(jmxLocalPeerBean);

        for (RemotePeerBean remotePeerBean : jmxRemotePeerBean.values()) {
            instance.unregister(remotePeerBean);
        }

        jmxQuorumBean = null;
        jmxLocalPeerBean = null;
        jmxRemotePeerBean = null;
    }
}
```

## 6.startLeaderElection
- ServerState
```java
 public enum ServerState {
    // 正在选举中，启动默认该节点，其它三个分别对应角色
    LOOKING,
    FOLLOWING,
    LEADING,
    OBSERVING
}
```
- startLeaderElection
```java
public synchronized void startLeaderElection() {
    try {
        // 该分支表示此时没有leader需要进行选举
        if (getPeerState() == ServerState.LOOKING) {
            // myid机器id、Zxid事务id、Epoch选举轮次
            // 初始化一个选票
            currentVote = new Vote(myid, getLastLoggedZxid(), getCurrentEpoch());
        }
    } catch (IOException e) {
        RuntimeException re = new RuntimeException(e.getMessage());
        re.setStackTrace(e.getStackTrace());
        throw re;
    }
    // 此处传入默认值3FastLeaderElection
    this.electionAlg = createElectionAlgorithm(electionType);
}
```
- createElectionAlgorithm
```java
protected Election createElectionAlgorithm(int electionAlgorithm) {
    Election le = null;

    //TODO: use a factory rather than a switch
    // 如果是旧版本还会有其它算法可以选，现在没得选
    switch (electionAlgorithm) {
    case 1:
        throw new UnsupportedOperationException("Election Algorithm 1 is not supported.");
    case 2:
        throw new UnsupportedOperationException("Election Algorithm 2 is not supported.");
    case 3:
        QuorumCnxManager qcm = createCnxnManager();
        QuorumCnxManager oldQcm = qcmRef.getAndSet(qcm);
        if (oldQcm != null) {
            LOG.warn("Clobbering already-set QuorumCnxManager (restarting leader election?)");
            oldQcm.halt();
        }
        QuorumCnxManager.Listener listener = qcm.listener;
        if (listener != null) {
            // 此处listener是线程
            listener.start();
            FastLeaderElection fle = new FastLeaderElection(this, qcm);
            fle.start();
            le = fle;
        } else {
            LOG.error("Null listener when initializing cnx manager");
        }
        break;
    default:
        assert false;
    }
    return le;
}
```

## 6.1QuorumCnxManager.Listener.run
```java
public void run() {
    if (!shutdown) {
        LOG.debug("Listener thread started, myId: {}", self.getId());
        Set<InetSocketAddress> addresses;

        // 获取选举监听的端口，注意此处是自己的端口正常也只会获取一个，新版本0.0.0.0/0.0.0.0:3888这种ip端口这个改动应该是来源于容器化的k8s问题
        if (self.getQuorumListenOnAllIPs()) {
            addresses = self.getElectionAddress().getWildcardAddresses();
        } else {
            addresses = self.getElectionAddress().getAllAddresses();
        }

        CountDownLatch latch = new CountDownLatch(addresses.size());
        // 每个端口起一个ListenerHandler
        listenerHandlers = addresses.stream().map(address ->
                        new ListenerHandler(address, self.shouldUsePortUnification(), self.isSslQuorum(), latch))
                .collect(Collectors.toList());

        final ExecutorService executor = Executors.newFixedThreadPool(addresses.size());
        try {
            // 上面创建的每个ListenerHandler执行run方法
            listenerHandlers.forEach(executor::submit);
        } finally {
            // prevent executor's threads to leak after ListenerHandler tasks complete
            executor.shutdown();
        }

        try {
            // 等待所有返回
            latch.await();
        } catch (InterruptedException ie) {
            LOG.error("Interrupted while sleeping. Ignoring exception", ie);
        } finally {
            // Clean up for shutdown.
            for (ListenerHandler handler : listenerHandlers) {
                try {
                    handler.close();
                } catch (IOException ie) {
                    // Don't log an error for shutdown.
                    LOG.debug("Error closing server socket", ie);
                }
            }
        }
    }

    LOG.info("Leaving listener");
    if (!shutdown) {
        LOG.error(
          "As I'm leaving the listener thread, I won't be able to participate in leader election any longer: {}",
          self.getElectionAddress().getAllAddresses().stream()
            .map(NetUtils::formatInetAddr)
            .collect(Collectors.joining("|")));
        if (socketException.get()) {
            // After leaving listener thread, the host cannot join the quorum anymore,
            // this is a severe error that we cannot recover from, so we need to exit
            socketBindErrorHandler.run();
        }
    }
}
```
## 6.1.1ListenerHandler.run
```java
public void run() {
    try {
        Thread.currentThread().setName("ListenerHandler-" + address);
        acceptConnections();
        try {
            close();
        } catch (IOException e) {
            LOG.warn("Exception when shutting down listener: ", e);
        }
    } catch (Exception e) {
        // Output of unexpected exception, should never happen
        LOG.error("Unexpected error ", e);
    } finally {
        latch.countDown();
    }
```
```java
private void acceptConnections() {
    int numRetries = 0;
    Socket client = null;

    while ((!shutdown) && (portBindMaxRetry == 0 || numRetries < portBindMaxRetry)) {
        try {
            // 此处起了一个Socket用于监听选票返回
            serverSocket = createNewServerSocket();
            LOG.info("{} is accepting connections now, my election bind port: {}", QuorumCnxManager.this.mySid, address.toString());
            while (!shutdown) {
                try {
                    // 此处接收到请求
                    client = serverSocket.accept();
                    setSockOpts(client);
                    LOG.info("Received connection request from {}", client.getRemoteSocketAddress());
                    // Receive and handle the connection request
                    // asynchronously if the quorum sasl authentication is
                    // enabled. This is required because sasl server
                    // authentication process may take few seconds to finish,
                    // this may delay next peer connection requests.
                    // 判断是否ssl，默认false
                    if (quorumSaslAuthEnabled) {
                        receiveConnectionAsync(client);
                    } else {
                        receiveConnection(client);
                    }
                    numRetries = 0;
                } catch (SocketTimeoutException e) {
                    LOG.warn("The socket is listening for the election accepted "
                            + "and it timed out unexpectedly, but will retry."
                            + "see ZOOKEEPER-2836");
                }
            }
        } catch (IOException e) {
            if (shutdown) {
                break;
            }

            LOG.error("Exception while listening to address {}", address, e);

            if (e instanceof SocketException) {
                socketException.set(true);
            }

            numRetries++;
            try {
                close();
                Thread.sleep(1000);
            } catch (IOException ie) {
                LOG.error("Error closing server socket", ie);
            } catch (InterruptedException ie) {
                LOG.error("Interrupted while sleeping. Ignoring exception", ie);
            }
            closeSocket(client);
        }
    }
    if (!shutdown) {
        LOG.error(
          "Leaving listener thread for address {} after {} errors. Use {} property to increase retry count.",
          formatInetAddr(address),
          numRetries,
          ELECTION_PORT_BIND_RETRY);
    }
}
```

## 6.1.2QuorumCnxManager#receiveConnection
QuorumCnxManager#receiveConnection->QuorumCnxManager#handleConnection
```java
private void handleConnection(Socket sock, DataInputStream din) throws IOException {
    Long sid = null, protocolVersion = null;
    MultipleAddresses electionAddr = null;

    try {
        protocolVersion = din.readLong();
        if (protocolVersion >= 0) { // this is a server id and not a protocol version
            sid = protocolVersion;
        } else {
            try {
                InitialMessage init = InitialMessage.parse(protocolVersion, din);
                // 获取远端的sid
                sid = init.sid;
                if (!init.electionAddr.isEmpty()) {
                    electionAddr = new MultipleAddresses(init.electionAddr,
                            Duration.ofMillis(self.getMultiAddressReachabilityCheckTimeoutMs()));
                }
                LOG.debug("Initial message parsed by {}: {}", self.getId(), init.toString());
            } catch (InitialMessage.InitialMessageException ex) {
                LOG.error("Initial message parsing error!", ex);
                closeSocket(sock);
                return;
            }
        }

        if (sid == QuorumPeer.OBSERVER_ID) {
            /*
             * Choose identifier at random. We need a value to identify
             * the connection.
             */
            sid = observerCounter.getAndDecrement();
            LOG.info("Setting arbitrary identifier to observer: {}", sid);
        }
    } catch (IOException e) {
        LOG.warn("Exception reading or writing challenge", e);
        closeSocket(sock);
        return;
    }

    // do authenticating learner
    authServer.authenticate(sock, din);
    //If wins the challenge, then close the new connection.
    // 此处注意，为了提升性能，减少建立的连接
    if (sid < self.getId()) {
        /*
         * This replica might still believe that the connection to sid is
         * up, so we have to shut down the workers before trying to open a
         * new connection.
         */
        SendWorker sw = senderWorkerMap.get(sid);
        if (sw != null) {
            sw.finish();
        }

        /*
         * Now we start a new connection
         */
        LOG.debug("Create new connection to server: {}", sid);
        // 如果我方的id大于对方的id则直接关闭，也即只能大连小
        closeSocket(sock);

        if (electionAddr != null) {
            connectOne(sid, electionAddr);
        } else {
            connectOne(sid);
        }

    } else if (sid == self.getId()) {
        // we saw this case in ZOOKEEPER-2164
        LOG.warn("We got a connection request from a server with our own ID. "
                 + "This should be either a configuration error, or a bug.");
    } else { // Otherwise start worker threads to receive data.
        SendWorker sw = new SendWorker(sock, sid);
        RecvWorker rw = new RecvWorker(sock, din, sid, sw);
        sw.setRecv(rw);

        // 结束旧socket，可能连不通，也可能新旧socket一致
        SendWorker vsw = senderWorkerMap.get(sid);

        if (vsw != null) {
            vsw.finish();
        }

        senderWorkerMap.put(sid, sw);
        // 该队列长度是1
        queueSendMap.putIfAbsent(sid, new CircularBlockingQueue<>(SEND_CAPACITY));

        sw.start();
        rw.start();
    }
}
```
## 6.1.2.1QuorumCnxManager.SendWorker#run
```java
public void run() {
    threadCnt.incrementAndGet();
    try {
        /**
         * If there is nothing in the queue to send, then we
         * send the lastMessage to ensure that the last message
         * was received by the peer. The message could be dropped
         * in case self or the peer shutdown their connection
         * (and exit the thread) prior to reading/processing
         * the last message. Duplicate messages are handled correctly
         * by the peer.
         *
         * If the send queue is non-empty, then we have a recent
         * message than that stored in lastMessage. To avoid sending
         * stale message, we should send the message in the send queue.
         */
        BlockingQueue<ByteBuffer> bq = queueSendMap.get(sid);
        if (bq == null || isSendQueueEmpty(bq)) {
            ByteBuffer b = lastMessageSent.get(sid);
            if (b != null) {
                LOG.debug("Attempting to send lastMessage to sid={}", sid);
                send(b);
            }
        }
    } catch (IOException e) {
        LOG.error("Failed to send last message. Shutting down thread.", e);
        this.finish();
    }
    LOG.debug("SendWorker thread started towards {}. myId: {}", sid, QuorumCnxManager.this.mySid);

    try {
        while (running && !shutdown && sock != null) {

            ByteBuffer b = null;
            try {
                BlockingQueue<ByteBuffer> bq = queueSendMap.get(sid);
                if (bq != null) {
                    b = pollSendQueue(bq, 1000, TimeUnit.MILLISECONDS);
                } else {
                    LOG.error("No queue of incoming messages for server {}", sid);
                    break;
                }

                if (b != null) {
                    lastMessageSent.put(sid, b);
                    send(b);
                }
            } catch (InterruptedException e) {
                LOG.warn("Interrupted while waiting for message on queue", e);
            }
        }
    } catch (Exception e) {
        LOG.warn(
            "Exception when using channel: for id {} my id = {}",
            sid ,
            QuorumCnxManager.this.mySid,
            e);
    }
    this.finish();

    LOG.warn("Send worker leaving thread id {} my id = {}", sid, self.getId());
}
```
## 6.1.2.2RecvWorker#run
- RecvWorker#run
```java
public void run() {
    threadCnt.incrementAndGet();
    try {
        LOG.debug("RecvWorker thread towards {} started. myId: {}", sid, QuorumCnxManager.this.mySid);
        while (running && !shutdown && sock != null) {
            /**
             * Reads the first int to determine the length of the
             * message
             */
            int length = din.readInt();
            if (length <= 0 || length > PACKETMAXSIZE) {
                throw new IOException("Received packet with invalid packet: " + length);
            }
            /**
             * Allocates a new ByteBuffer to receive the message
             */
            final byte[] msgArray = new byte[length];
            din.readFully(msgArray, 0, length);
            // 存入recvQueue
            addToRecvQueue(new Message(ByteBuffer.wrap(msgArray), sid));
        }
    } catch (Exception e) {
        LOG.warn(
            "Connection broken for id {}, my id = {}",
            sid,
            QuorumCnxManager.this.mySid,
            e);
    } finally {
        LOG.warn("Interrupting SendWorker thread from RecvWorker. sid: {}. myId: {}", sid, QuorumCnxManager.this.mySid);
        sw.finish();
        closeSocket(sock);
    }
}
```
- QuorumCnxManager#addToRecvQueue
```java
public void addToRecvQueue(final Message msg) {
  // 存入接收选票队列
  final boolean success = this.recvQueue.offer(msg);
  if (!success) {
      throw new RuntimeException("Could not insert into receive queue");
  }
}
```
### 6.2new FastLeaderElection()
- FastLeaderElection
```java
public FastLeaderElection(QuorumPeer self, QuorumCnxManager manager) {
    this.stop = false;
    this.manager = manager;
    starter(self, manager);
}
```
- starter
```java
private void starter(QuorumPeer self, QuorumCnxManager manager) {
    this.self = self;
    proposedLeader = -1;
    proposedZxid = -1;
    // 发送选票队列
    sendqueue = new LinkedBlockingQueue<ToSend>();
    // 接收选票队列
    recvqueue = new LinkedBlockingQueue<Notification>();
    this.messenger = new Messenger(manager);
}
```
### 6.3fle.start
- start
```java
public void start() {
    this.messenger.start();
}
```
- start
```java
void start() {
    // 发送选票队列线程启动
    this.wsThread.start();
    // 接收选票队列线程启动
    this.wrThread.start();
}
```
- wsThread初始化
```java
Messenger(QuorumCnxManager manager) {
    // 关键赋值
    this.ws = new WorkerSender(manager);

    this.wsThread = new Thread(this.ws, "WorkerSender[myid=" + self.getId() + "]");
    this.wsThread.setDaemon(true);

    // 关键赋值
    this.wr = new WorkerReceiver(manager);

    this.wrThread = new Thread(this.wr, "WorkerReceiver[myid=" + self.getId() + "]");
    this.wrThread.setDaemon(true);
}
```
### 6.3.1WorkerSender#run
```java
public void run() {
    while (!stop) {
        try {
            // 此处弹出要发送的选票消息
            ToSend m = sendqueue.poll(3000, TimeUnit.MILLISECONDS);
            if (m == null) {
                continue;
            }

            process(m);
        } catch (InterruptedException e) {
            break;
        }
    }
    LOG.info("WorkerSender is down");
}
```
- WorkerSender#process
```java
void process(ToSend m) {
    ByteBuffer requestBuffer = buildMsg(m.state.ordinal(), m.leader, m.zxid, m.electionEpoch, m.peerEpoch, m.configData);

    manager.toSend(m.sid, requestBuffer);

}
```
- QuorumCnxManager#toSend
```java
public void toSend(Long sid, ByteBuffer b) {
    /*
     * If sending message to myself, then simply enqueue it (loopback).
     */
    if (this.mySid == sid) {
        b.position(0);
        // 如果是发给自己则存入recvQueue
        addToRecvQueue(new Message(b.duplicate(), sid));
        /*
         * Otherwise send to the corresponding thread to send.
         */
    } else {
        /*
         * Start a new connection if doesn't have one already.
         */
        // 如果不是自己则存入queueSendMap中的队列
        BlockingQueue<ByteBuffer> bq = queueSendMap.computeIfAbsent(sid, serverId -> new CircularBlockingQueue<>(SEND_CAPACITY));
        addToSendQueue(bq, b);
        connectOne(sid);
    }
}
```
### 6.3.2WorkerSender.run
```java
public void run() {
    Message response;
    while (!stop) {
        // Sleeps on receive
        try {
            response = manager.pollRecvQueue(3000, TimeUnit.MILLISECONDS);
            if (response == null) {
                continue;
            }

            final int capacity = response.buffer.capacity();

            // The current protocol and two previous generations all send at least 28 bytes
            if (capacity < 28) {
                LOG.error("Got a short response from server {}: {}", response.sid, capacity);
                continue;
            }

            // this is the backwardCompatibility mode in place before ZK-107
            // It is for a version of the protocol in which we didn't send peer epoch
            // With peer epoch and version the message became 40 bytes
            boolean backCompatibility28 = (capacity == 28);

            // this is the backwardCompatibility mode for no version information
            boolean backCompatibility40 = (capacity == 40);

            response.buffer.clear();

            // Instantiate Notification and set its attributes
            Notification n = new Notification();

            int rstate = response.buffer.getInt();
            long rleader = response.buffer.getLong();
            long rzxid = response.buffer.getLong();
            long relectionEpoch = response.buffer.getLong();
            long rpeerepoch;

            int version = 0x0;
            QuorumVerifier rqv = null;

            try {
                if (!backCompatibility28) {
                    rpeerepoch = response.buffer.getLong();
                    if (!backCompatibility40) {
                        /*
                         * Version added in 3.4.6
                         */

                        version = response.buffer.getInt();
                    } else {
                        LOG.info("Backward compatibility mode (36 bits), server id: {}", response.sid);
                    }
                } else {
                    LOG.info("Backward compatibility mode (28 bits), server id: {}", response.sid);
                    rpeerepoch = ZxidUtils.getEpochFromZxid(rzxid);
                }

                // check if we have a version that includes config. If so extract config info from message.
                if (version > 0x1) {
                    int configLength = response.buffer.getInt();

                    // we want to avoid errors caused by the allocation of a byte array with negative length
                    // (causing NegativeArraySizeException) or huge length (causing e.g. OutOfMemoryError)
                    if (configLength < 0 || configLength > capacity) {
                        throw new IOException(String.format("Invalid configLength in notification message! sid=%d, capacity=%d, version=%d, configLength=%d",
                                                            response.sid, capacity, version, configLength));
                    }

                    byte[] b = new byte[configLength];
                    response.buffer.get(b);

                    synchronized (self) {
                        try {
                            rqv = self.configFromString(new String(b, UTF_8));
                            QuorumVerifier curQV = self.getQuorumVerifier();
                            if (rqv.getVersion() > curQV.getVersion()) {
                                LOG.info("{} Received version: {} my version: {}",
                                         self.getId(),
                                         Long.toHexString(rqv.getVersion()),
                                         Long.toHexString(self.getQuorumVerifier().getVersion()));
                                if (self.getPeerState() == ServerState.LOOKING) {
                                    LOG.debug("Invoking processReconfig(), state: {}", self.getServerState());
                                    self.processReconfig(rqv, null, null, false);
                                    if (!rqv.equals(curQV)) {
                                        LOG.info("restarting leader election");
                                        self.shuttingDownLE = true;
                                        self.getElectionAlg().shutdown();

                                        break;
                                    }
                                } else {
                                    LOG.debug("Skip processReconfig(), state: {}", self.getServerState());
                                }
                            }
                        } catch (IOException | ConfigException e) {
                            LOG.error("Something went wrong while processing config received from {}", response.sid);
                        }
                    }
                } else {
                    LOG.info("Backward compatibility mode (before reconfig), server id: {}", response.sid);
                }
            } catch (BufferUnderflowException | IOException e) {
                LOG.warn("Skipping the processing of a partial / malformed response message sent by sid={} (message length: {})",
                         response.sid, capacity, e);
                continue;
            }
            /*
             * If it is from a non-voting server (such as an observer or
             * a non-voting follower), respond right away.
             */
            if (!validVoter(response.sid)) {
                Vote current = self.getCurrentVote();
                QuorumVerifier qv = self.getQuorumVerifier();
                ToSend notmsg = new ToSend(
                    ToSend.mType.notification,
                    current.getId(),
                    current.getZxid(),
                    logicalclock.get(),
                    self.getPeerState(),
                    response.sid,
                    current.getPeerEpoch(),
                    qv.toString().getBytes(UTF_8));

                sendqueue.offer(notmsg);
            } else {
                // Receive new message
                LOG.debug("Receive new notification message. My id = {}", self.getId());

                // State of peer that sent this message
                QuorumPeer.ServerState ackstate = QuorumPeer.ServerState.LOOKING;
                switch (rstate) {
                case 0:
                    ackstate = QuorumPeer.ServerState.LOOKING;
                    break;
                case 1:
                    ackstate = QuorumPeer.ServerState.FOLLOWING;
                    break;
                case 2:
                    ackstate = QuorumPeer.ServerState.LEADING;
                    break;
                case 3:
                    ackstate = QuorumPeer.ServerState.OBSERVING;
                    break;
                default:
                    continue;
                }

                n.leader = rleader;
                n.zxid = rzxid;
                n.electionEpoch = relectionEpoch;
                n.state = ackstate;
                n.sid = response.sid;
                n.peerEpoch = rpeerepoch;
                n.version = version;
                n.qv = rqv;
                /*
                 * Print notification info
                 */
                LOG.info(
                    "Notification: my state:{}; n.sid:{}, n.state:{}, n.leader:{}, n.round:0x{}, "
                        + "n.peerEpoch:0x{}, n.zxid:0x{}, message format version:0x{}, n.config version:0x{}",
                    self.getPeerState(),
                    n.sid,
                    n.state,
                    n.leader,
                    Long.toHexString(n.electionEpoch),
                    Long.toHexString(n.peerEpoch),
                    Long.toHexString(n.zxid),
                    Long.toHexString(n.version),
                    (n.qv != null ? (Long.toHexString(n.qv.getVersion())) : "0"));

                /*
                 * If this server is looking, then send proposed leader
                 */

                if (self.getPeerState() == QuorumPeer.ServerState.LOOKING) {
                    recvqueue.offer(n);

                    /*
                     * Send a notification back if the peer that sent this
                     * message is also looking and its logical clock is
                     * lagging behind.
                     */
                    if ((ackstate == QuorumPeer.ServerState.LOOKING)
                        && (n.electionEpoch < logicalclock.get())) {
                        Vote v = getVote();
                        QuorumVerifier qv = self.getQuorumVerifier();
                        ToSend notmsg = new ToSend(
                            ToSend.mType.notification,
                            v.getId(),
                            v.getZxid(),
                            logicalclock.get(),
                            self.getPeerState(),
                            response.sid,
                            v.getPeerEpoch(),
                            qv.toString().getBytes());
                        sendqueue.offer(notmsg);
                    }
                } else {
                    /*
                     * If this server is not looking, but the one that sent the ack
                     * is looking, then send back what it believes to be the leader.
                     */
                    Vote current = self.getCurrentVote();
                    // 如果对方还是LOOKING，而我方不是，则通知对方已经有leader了
                    if (ackstate == QuorumPeer.ServerState.LOOKING) {
                        if (self.leader != null) {
                            if (leadingVoteSet != null) {
                                self.leader.setLeadingVoteSet(leadingVoteSet);
                                leadingVoteSet = null;
                            }
                            self.leader.reportLookingSid(response.sid);
                        }


                        LOG.debug(
                            "Sending new notification. My id ={} recipient={} zxid=0x{} leader={} config version = {}",
                            self.getId(),
                            response.sid,
                            Long.toHexString(current.getZxid()),
                            current.getId(),
                            Long.toHexString(self.getQuorumVerifier().getVersion()));

                        QuorumVerifier qv = self.getQuorumVerifier();
                        ToSend notmsg = new ToSend(
                            ToSend.mType.notification,
                            current.getId(),
                            current.getZxid(),
                            current.getElectionEpoch(),
                            self.getPeerState(),
                            response.sid,
                            current.getPeerEpoch(),
                            qv.toString().getBytes());
                        sendqueue.offer(notmsg);
                    }
                }
            }
        } catch (InterruptedException e) {
            LOG.warn("Interrupted Exception while waiting for new message", e);
        }
    }
    LOG.info("WorkerReceiver is down");
}
```
## 7.QuorumPeer#run-LOOKING
```java
case LOOKING:
    LOG.info("LOOKING");
    ServerMetrics.getMetrics().LOOKING_COUNT.add(1);
    // 只读模式，一般不启用
    if (Boolean.getBoolean("readonlymode.enabled")) {
        ...
    } else {
        try {
            // 重置configFlag标记
            reconfigFlagClear();
            // 默认false，某些特殊情况监听选票没有开启在此处开启
            if (shuttingDownLE) {
                shuttingDownLE = false;
                startLeaderElection();
            }
            // 设置当前投票
            setCurrentVote(makeLEStrategy().lookForLeader());
        } catch (Exception e) {
            LOG.warn("Unexpected exception", e);
            setPeerState(ServerState.LOOKING);
        }
    }
    break;
```

### 7.1makeLEStrategy|获取选票策略
返回选票策略
```java
protected Election makeLEStrategy() {
    LOG.debug("Initializing leader election protocol...");
    return electionAlg;
}
```
### 7.2lookForLeader|寻找leader
```java
public Vote lookForLeader() throws InterruptedException {
    try {
        self.jmxLeaderElectionBean = new LeaderElectionBean();
        MBeanRegistry.getInstance().register(self.jmxLeaderElectionBean, self.jmxLocalPeerBean);
    } catch (Exception e) {
        LOG.warn("Failed to register with JMX", e);
        self.jmxLeaderElectionBean = null;
    }

    self.start_fle = Time.currentElapsedTime();
    try {
        /*
         * The votes from the current leader election are stored in recvset. In other words, a vote v is in recvset
         * if v.electionEpoch == logicalclock. The current participant uses recvset to deduce on whether a majority
         * of participants has voted for it.
         */
        Map<Long, Vote> recvset = new HashMap<Long, Vote>();

        /*
         * The votes from previous leader elections, as well as the votes from the current leader election are
         * stored in outofelection. Note that notifications in a LOOKING state are not stored in outofelection.
         * Only FOLLOWING or LEADING notifications are stored in outofelection. The current participant could use
         * outofelection to learn which participant is the leader if it arrives late (i.e., higher logicalclock than
         * the electionEpoch of the received notifications) in a leader election.
         */
        Map<Long, Vote> outofelection = new HashMap<Long, Vote>();

        int notTimeout = minNotificationInterval;

        synchronized (this) {
            // 选举周期+1,内层是个循环除非状态改变
            logicalclock.incrementAndGet();
            /* 
            getInitId()|获取初始化id正常情况就是自己
            getInitLastLoggedZxid()|最后的事务id
            getPeerEpoch()|选票轮次
            */
            // 更新提议
            updateProposal(getInitId(), getInitLastLoggedZxid(), getPeerEpoch());
        }

        LOG.info(
            "New election. My id = {}, proposed zxid=0x{}",
            self.getId(),
            Long.toHexString(proposedZxid));
        // 存入sendQueue队列
        sendNotifications();

        SyncedLearnerTracker voteSet = null;

        /*
         * Loop in which we exchange notifications until we find a leader
         */
        // 自旋直到选出一个leader后自身角色改变
        while ((self.getPeerState() == ServerState.LOOKING) && (!stop)) {
            /*
             * Remove next notification from queue, times out after 2 times
             * the termination time
             */
            // 接收选票
            Notification n = recvqueue.poll(notTimeout, TimeUnit.MILLISECONDS);

            /*
             * Sends more notifications if haven't received enough.
             * Otherwise processes new notification.
             */
            if (n == null) {
                // 如果没有收到选票比如刚启动则走该分支

                
                /**
                 * Check if all queues are empty, indicating that all messages have been delivered.
                 */
                // 官方注释：判断是不是所有消息都投递了,true为都投递了，根据queueSendMap进行判断
                if (manager.haveDelivered()) {
                    sendNotifications();
                } else {
                    /**
                     * Try to establish a connection with each server if one doesn't exist.
                    */
                    // 官方注释:与每个服务器建立连接
                    // 注意此处会放入senderWorkerMap
                    // connectAll()->connectOne->asyncValidateIfSocketIsStillReachable->connectOne->initiateConnectionAsync
                    manager.connectAll();
                }

                /*
                 * Exponential backoff
                 */
                notTimeout = Math.min(notTimeout << 1, maxNotificationInterval);

                /*
                 * When a leader failure happens on a master, the backup will be supposed to receive the honour from
                 * Oracle and become a leader, but the honour is likely to be delay. We do a re-check once timeout happens
                 *
                 * The leader election algorithm does not provide the ability of electing a leader from a single instance
                 * which is in a configuration of 2 instances.
                 * */
                if (self.getQuorumVerifier() instanceof QuorumOracleMaj
                        && self.getQuorumVerifier().revalidateVoteset(voteSet, notTimeout != minNotificationInterval)) {
                    setPeerState(proposedLeader, voteSet);
                    Vote endVote = new Vote(proposedLeader, proposedZxid, logicalclock.get(), proposedEpoch);
                    leaveInstance(endVote);
                    return endVote;
                }

                LOG.info("Notification time out: {} ms", notTimeout);

            } else if (validVoter(n.sid) && validVoter(n.leader)) {
                // 校验选票所选机器和目标机器都是在列表中的
                /*
                 * Only proceed if the vote comes from a replica in the current or next
                 * voting view for a replica in the current or next voting view.
                 */
                // 判断其它机器的状态
                // totalOrderPredicate选票PK核心逻辑
                // updateProposal更新提议,最初的第一轮推举的是自己,其他情况根据判断条件进行推举
                // sendNotifications向其他机器发送新提议
                switch (n.state) {
                case LOOKING:
                    if (getInitLastLoggedZxid() == -1) {
                        LOG.debug("Ignoring notification as our zxid is -1");
                        break;
                    }
                    if (n.zxid == -1) {
                        LOG.debug("Ignoring notification from member with -1 zxid {}", n.sid);
                        break;
                    }
                    // If notification > current, replace and send messages out
                    if (n.electionEpoch > logicalclock.get()) {
                        logicalclock.set(n.electionEpoch);
                        recvset.clear();
                        if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch, getInitId(), getInitLastLoggedZxid(), getPeerEpoch())) {
                            updateProposal(n.leader, n.zxid, n.peerEpoch);
                        } else {
                            updateProposal(getInitId(), getInitLastLoggedZxid(), getPeerEpoch());
                        }
                        sendNotifications();
                    } else if (n.electionEpoch < logicalclock.get()) {
                            LOG.debug(
                                "Notification election epoch is smaller than logicalclock. n.electionEpoch = 0x{}, logicalclock=0x{}",
                                Long.toHexString(n.electionEpoch),
                                Long.toHexString(logicalclock.get()));
                        break;
                    } else if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch, proposedLeader, proposedZxid, proposedEpoch)) {
                        updateProposal(n.leader, n.zxid, n.peerEpoch);
                        sendNotifications();
                    }

                    LOG.debug(
                        "Adding vote: from={}, proposed leader={}, proposed zxid=0x{}, proposed election epoch=0x{}",
                        n.sid,
                        n.leader,
                        Long.toHexString(n.zxid),
                        Long.toHexString(n.electionEpoch));

                    // don't care about the version if it's in LOOKING state
                    recvset.put(n.sid, new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch));

                    voteSet = getVoteTracker(recvset, new Vote(proposedLeader, proposedZxid, logicalclock.get(), proposedEpoch));

                    if (voteSet.hasAllQuorums()) {

                        // Verify if there is any change in the proposed leader
                        while ((n = recvqueue.poll(finalizeWait, TimeUnit.MILLISECONDS)) != null) {
                            if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch, proposedLeader, proposedZxid, proposedEpoch)) {
                                recvqueue.put(n);
                                break;
                            }
                        }

                        /*
                         * This predicate is true once we don't read any new
                         * relevant message from the reception queue
                         */
                        if (n == null) {
                            setPeerState(proposedLeader, voteSet);
                            Vote endVote = new Vote(proposedLeader, proposedZxid, logicalclock.get(), proposedEpoch);
                            leaveInstance(endVote);
                            return endVote;
                        }
                    }
                    break;
                case OBSERVING:
                    LOG.debug("Notification from observer: {}", n.sid);
                    break;

                    /*
                    * In ZOOKEEPER-3922, we separate the behaviors of FOLLOWING and LEADING.
                    * To avoid the duplication of codes, we create a method called followingBehavior which was used to
                    * shared by FOLLOWING and LEADING. This method returns a Vote. When the returned Vote is null, it follows
                    * the original idea to break swtich statement; otherwise, a valid returned Vote indicates, a leader
                    * is generated.
                    *
                    * The reason why we need to separate these behaviors is to make the algorithm runnable for 2-node
                    * setting. An extra condition for generating leader is needed. Due to the majority rule, only when
                    * there is a majority in the voteset, a leader will be generated. However, in a configuration of 2 nodes,
                    * the number to achieve the majority remains 2, which means a recovered node cannot generate a leader which is
                    * the existed leader. Therefore, we need the Oracle to kick in this situation. In a two-node configuration, the Oracle
                    * only grants the permission to maintain the progress to one node. The oracle either grants the permission to the
                    * remained node and makes it a new leader when there is a faulty machine, which is the case to maintain the progress.
                    * Otherwise, the oracle does not grant the permission to the remained node, which further causes a service down.
                    *
                    * In the former case, when a failed server recovers and participate in the leader election, it would not locate a
                    * new leader because there does not exist a majority in the voteset. It fails on the containAllQuorum() infinitely due to
                    * two facts. First one is the fact that it does do not have a majority in the voteset. The other fact is the fact that
                    * the oracle would not give the permission since the oracle already gave the permission to the existed leader, the healthy machine.
                    * Logically, when the oracle replies with negative, it implies the existed leader which is LEADING notification comes from is a valid leader.
                    * To threat this negative replies as a permission to generate the leader is the purpose to separate these two behaviors.
                    *
                    *
                    * */
                case FOLLOWING:
                    /*
                    * To avoid duplicate codes
                    * */
                    Vote resultFN = receivedFollowingNotification(recvset, outofelection, voteSet, n);
                    if (resultFN == null) {
                        break;
                    } else {
                        return resultFN;
                    }
                case LEADING:
                    /*
                    * In leadingBehavior(), it performs followingBehvior() first. When followingBehavior() returns
                    * a null pointer, ask Oracle whether to follow this leader.
                    * */
                    Vote resultLN = receivedLeadingNotification(recvset, outofelection, voteSet, n);
                    if (resultLN == null) {
                        break;
                    } else {
                        return resultLN;
                    }
                default:
                    LOG.warn("Notification state unrecognized: {} (n.state), {}(n.sid)", n.state, n.sid);
                    break;
                }
            } else {
                if (!validVoter(n.leader)) {
                    LOG.warn("Ignoring notification for non-cluster member sid {} from sid {}", n.leader, n.sid);
                }
                if (!validVoter(n.sid)) {
                    LOG.warn("Ignoring notification for sid {} from non-quorum member sid {}", n.leader, n.sid);
                }
            }
        }
        return null;
    } finally {
        try {
            if (self.jmxLeaderElectionBean != null) {
                MBeanRegistry.getInstance().unregister(self.jmxLeaderElectionBean);
            }
        } catch (Exception e) {
            LOG.warn("Failed to unregister with JMX", e);
        }
        self.jmxLeaderElectionBean = null;
        LOG.debug("Number of connection processing threads: {}", manager.getConnectionThreadCount());
    }
}
```
### 7.2.1updateProposal|更新提议
```java
synchronized void updateProposal(long leader, long zxid, long epoch) {
    LOG.debug(
        "Updating proposal: {} (newleader), 0x{} (newzxid), {} (oldleader), 0x{} (oldzxid)",
        leader,
        Long.toHexString(zxid),
        proposedLeader,
        Long.toHexString(proposedZxid));

    proposedLeader = leader;
    proposedZxid = zxid;
    proposedEpoch = epoch;
}
```
### 7.2.2sendNotifications|发送投票
```java
private void sendNotifications() {
    // sid 是目标机器id,循环发送
    for (long sid : self.getCurrentAndNextConfigVoters()) {
        QuorumVerifier qv = self.getQuorumVerifier();
        ToSend notmsg = new ToSend(
            ToSend.mType.notification,
            proposedLeader,
            proposedZxid,
            logicalclock.get(),
            QuorumPeer.ServerState.LOOKING,
            sid,
            proposedEpoch,
            qv.toString().getBytes(UTF_8));

        LOG.debug(
            "Sending Notification: {} (n.leader), 0x{} (n.zxid), 0x{} (n.round), {} (recipient),"
                + " {} (myid), 0x{} (n.peerEpoch) ",
            proposedLeader,
            Long.toHexString(proposedZxid),
            Long.toHexString(logicalclock.get()),
            sid,
            self.getId(),
            Long.toHexString(proposedEpoch));
        // 将刚才的投票存入sendqueue队列
        sendqueue.offer(notmsg);
    }
}
```
### 7.2.3connectAll
connectAll()->connectOne->asyncValidateIfSocketIsStillReachable->connectOne->initiateConnectionAsync  
注意此处因为目标可能有多个监听端口
```java
public boolean initiateConnectionAsync(final MultipleAddresses electionAddr, final Long sid) {
    if (!inprogressConnections.add(sid)) {
        // simply return as there is a connection request to
        // server 'sid' already in progress.
        LOG.debug("Connection request to server id: {} is already in progress, so skipping this request", sid);
        return true;
    }
    try {
        // 线程池执行与目标机器连接
        connectionExecutor.execute(new QuorumConnectionReqThread(electionAddr, sid));
        connectionThreadCnt.incrementAndGet();
    } catch (Throwable e) {
        // Imp: Safer side catching all type of exceptions and remove 'sid'
        // from inprogress connections. This is to avoid blocking further
        // connection requests from this 'sid' in case of errors.
        inprogressConnections.remove(sid);
        LOG.error("Exception while submitting quorum connection request", e);
        return false;
    }
    return true;
}
```
### 7.2.3.1QuorumConnectionReqThread.run  
QuorumConnectionReqThread.run->initiateConnection  
与目标服务器建立socket用于我方发送选票，目标服务器接收选票
```java
public void initiateConnection(final MultipleAddresses electionAddr, final Long sid) {
    Socket sock = null;
    try {
        LOG.debug("Opening channel to server {}", sid);
        if (self.isSslQuorum()) {
            sock = self.getX509Util().createSSLSocket();
        } else {
            sock = SOCKET_FACTORY.get();
        }
        setSockOpts(sock);
        sock.connect(electionAddr.getReachableOrOne(), cnxTO);
        if (sock instanceof SSLSocket) {
            SSLSocket sslSock = (SSLSocket) sock;
            sslSock.startHandshake();
            LOG.info("SSL handshake complete with {} - {} - {}",
                     sslSock.getRemoteSocketAddress(),
                     sslSock.getSession().getProtocol(),
                     sslSock.getSession().getCipherSuite());
        }

        LOG.debug("Connected to server {} using election address: {}:{}",
                  sid, sock.getInetAddress(), sock.getPort());
    } catch (X509Exception e) {
        LOG.warn("Cannot open secure channel to {} at election address {}", sid, electionAddr, e);
        closeSocket(sock);
        return;
    } catch (UnresolvedAddressException | IOException e) {
        LOG.warn("Cannot open channel to {} at election address {}", sid, electionAddr, e);
        closeSocket(sock);
        return;
    }

    try {
        // 存入queueSendMap以及senderWorkerMap
        startConnection(sock, sid);
    } catch (IOException e) {
        LOG.error(
          "Exception while connecting, id: {}, addr: {}, closing learner connection",
          sid,
          sock.getRemoteSocketAddress(),
          e);
        closeSocket(sock);
    }
}
```
### 7.2.4FastLeaderElection#totalOrderPredicate|选票PK核心逻辑
```java
protected boolean totalOrderPredicate(long newId, long newZxid, long newEpoch, long curId, long curZxid, long curEpoch) {
    LOG.debug(
        "id: {}, proposed id: {}, zxid: 0x{}, proposed zxid: 0x{}",
        newId,
        curId,
        Long.toHexString(newZxid),
        Long.toHexString(curZxid));

    if (self.getQuorumVerifier().getWeight(newId) == 0) {
        return false;
    }

    /*
     * We return true if one of the following three cases hold:
     * 1- New epoch is higher
     * 2- New epoch is the same as current epoch, but new zxid is higher
     * 3- New epoch is the same as current epoch, new zxid is the same
     *  as current zxid, but server id is higher.
     */

     /*
     * 官方注释
     * 1- 收到的轮次更高
     * 2- 轮次相同时事务id更高
     * 3- 轮次和事务相同时，机器id更高
     *  as current zxid, but server id is higher.
     */
    return ((newEpoch > curEpoch)
            || ((newEpoch == curEpoch)
                && ((newZxid > curZxid)
                    || ((newZxid == curZxid)
                        && (newId > curId)))));
}
```
### 7.2.5FastLeaderElection#receivedLeadingNotification|避免重复选举
```java
private Vote receivedLeadingNotification(Map<Long, Vote> recvset, Map<Long, Vote> outofelection, SyncedLearnerTracker voteSet, Notification n) {
    /*
    *
    * In a two-node configuration, a recovery nodes cannot locate a leader because of the lack of the majority in the voteset.
    * Therefore, it is the time for Oracle to take place as a tight breaker.
    *
    * */
    // 比较最后投的选票和自己的选票
    Vote result = receivedFollowingNotification(recvset, outofelection, voteSet, n);
    if (result == null) {
        /*
        * Ask Oracle to see if it is okay to follow this leader.
        *
        * We don't need the CheckLeader() because itself cannot be a leader candidate
        * */
        if (self.getQuorumVerifier().getNeedOracle() && !self.getQuorumVerifier().askOracle()) {
            LOG.info("Oracle indicates to follow");
            // 储存leader同时将自己设置成LEADING/FOLLOWING
            setPeerState(n.leader, voteSet);
            Vote endVote = new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch);
            leaveInstance(endVote);
            return endVote;
        } else {
            LOG.info("Oracle indicates not to follow");
            return null;
        }
    } else {
        return result;
    }
}
```
```java
private Vote receivedFollowingNotification(Map<Long, Vote> recvset, Map<Long, Vote> outofelection, SyncedLearnerTracker voteSet, Notification n) {
    /*
     * Consider all notifications from the same epoch
     * together.
     */
    if (n.electionEpoch == logicalclock.get()) {
        recvset.put(n.sid, new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch, n.state));
        // 统计选票
        voteSet = getVoteTracker(recvset, new Vote(n.version, n.leader, n.zxid, n.electionEpoch, n.peerEpoch, n.state));
        // 判断是否有过半的选票
        if (voteSet.hasAllQuorums() && checkLeader(recvset, n.leader, n.electionEpoch)) {
            setPeerState(n.leader, voteSet);
            // 返回收到的leader
            Vote endVote = new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch);
            leaveInstance(endVote);
            return endVote;
        }
    }

    /*
     * Before joining an established ensemble, verify that
     * a majority are following the same leader.
     *
     * Note that the outofelection map also stores votes from the current leader election.
     * See ZOOKEEPER-1732 for more information.
     */
    outofelection.put(n.sid, new Vote(n.version, n.leader, n.zxid, n.electionEpoch, n.peerEpoch, n.state));
    voteSet = getVoteTracker(outofelection, new Vote(n.version, n.leader, n.zxid, n.electionEpoch, n.peerEpoch, n.state));

    if (voteSet.hasAllQuorums() && checkLeader(outofelection, n.leader, n.electionEpoch)) {
        synchronized (this) {
            logicalclock.set(n.electionEpoch);
            setPeerState(n.leader, voteSet);
        }
        Vote endVote = new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch);
        leaveInstance(endVote);
        return endVote;
    }

    return null;
}
```
### 7.2.5.1SyncedLearnerTracker#hasAllQuorums|判断选票是否过半
- SyncedLearnerTracker#hasAllQuorums
```java
public boolean hasAllQuorums() {
    for (QuorumVerifierAcksetPair qvAckset : qvAcksetPairs) {
        if (!qvAckset.getQuorumVerifier().containsQuorum(qvAckset.getAckset())) {
            return false;
        }
    }
    return true;
}
```
- QuorumMaj#containsQuorum
```java
public boolean containsQuorum(Set<Long> ackSet) {
    return (ackSet.size() > half);
}
```
## 8.QuorumPeer#run-LEADING
当自身成为leader
```java
case LEADING:
    LOG.info("LEADING");
    try {
        setLeader(makeLeader(logFactory));
        leader.lead();
        setLeader(null);
    } catch (Exception e) {
        LOG.warn("Unexpected exception", e);
    } finally {
        if (leader != null) {
            leader.shutdown("Forcing shutdown");
            setLeader(null);
        }
        updateServerState();
    }
    break;
}
```