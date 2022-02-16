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
|LinkedBlockingQueue\<Notification\> recvqueue|接收选票队列|WorkerReceiver|WorkerReceiver|WorkerReceiver判断当前是否还在选举，如果在选举将收到的数据封装到recvqueue；如果WorkerReceiver判断没有在选举则将leader信息存入sendqueue。如果还在选举会在在lookForLeader中循环取出，本机如果还在选举则PK选票并发送给所有机器表示自己选票有更新，接着判断所有选票能否选出leader能够选出接着把收到的选票依次比较未发生变化则更改本机为leading/following|
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
1. 2181 监听客户端的默认端口
2. 2888 监听数据同步端口
3. 3888 监听选举投票端口
## 2.源码下载
## 3.QuorumPeer#run-LEADING
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
        // 如果出现异常或其它情况(比如内部私吞异常但自旋已经被破坏)此处重置当前机器状态，并把leader标记为null
        if (leader != null) {
            leader.shutdown("Forcing shutdown");
            setLeader(null);
        }
        updateServerState();
    }
    break;
```
### 3.1QuorumPeer#makeLeader
```java
protected Leader makeLeader(FileTxnSnapLog logFactory) throws IOException, X509Exception {
    return new Leader(this, new LeaderZooKeeperServer(logFactory, this, this.zkDb));
}
```
### 3.1.1Leader#Leader
```java
public Leader(QuorumPeer self, LeaderZooKeeperServer zk) throws IOException {
    this.self = self;
    this.proposalStats = new BufferStats();

    Set<InetSocketAddress> addresses;
    if (self.getQuorumListenOnAllIPs()) {
        addresses = self.getQuorumAddress().getWildcardAddresses();
    } else {
        addresses = self.getQuorumAddress().getAllAddresses();
    }

    addresses.stream()
    // 创建监听数据同步端口2888
      .map(address -> createServerSocket(address, self.shouldUsePortUnification(), self.isSslQuorum()))
      .filter(Optional::isPresent)
      .map(Optional::get)
      .forEach(serverSockets::add);

    if (serverSockets.isEmpty()) {
        throw new IOException("Leader failed to initialize any of the following sockets: " + addresses);
    }

    this.zk = zk;
}
```
### 3.2Leader#lead
```java
void lead() throws IOException, InterruptedException {
    self.end_fle = Time.currentElapsedTime();
    long electionTimeTaken = self.end_fle - self.start_fle;
    self.setElectionTimeTaken(electionTimeTaken);
    ServerMetrics.getMetrics().ELECTION_TIME.add(electionTimeTaken);
    LOG.info("LEADING - LEADER ELECTION TOOK - {} {}", electionTimeTaken, QuorumPeer.FLE_TIME_UNIT);
    self.start_fle = 0;
    self.end_fle = 0;

    zk.registerJMX(new LeaderBean(this, zk), self.jmxLocalPeerBean);

    try {
        self.setZabState(QuorumPeer.ZabState.DISCOVERY);
        self.tick.set(0);
        // 加载内存数据
        zk.loadData();

        leaderStateSummary = new StateSummary(self.getCurrentEpoch(), zk.getLastProcessedZxid());

        // Start thread that waits for connection requests from
        // new followers.
        // 官方注释，起一个线程等待follower连接
        cnxAcceptor = new LearnerCnxAcceptor();
        cnxAcceptor.start();

        long epoch = getEpochToPropose(self.getId(), self.getAcceptedEpoch());

        zk.setZxid(ZxidUtils.makeZxid(epoch, 0));

        synchronized (this) {
            lastProposed = zk.getZxid();
        }

        newLeaderProposal.packet = new QuorumPacket(NEWLEADER, zk.getZxid(), null, null);

        if ((newLeaderProposal.packet.getZxid() & 0xffffffffL) != 0) {
            LOG.info("NEWLEADER proposal has Zxid of {}", Long.toHexString(newLeaderProposal.packet.getZxid()));
        }

        QuorumVerifier lastSeenQV = self.getLastSeenQuorumVerifier();
        QuorumVerifier curQV = self.getQuorumVerifier();
        if (curQV.getVersion() == 0 && curQV.getVersion() == lastSeenQV.getVersion()) {
            // This was added in ZOOKEEPER-1783. The initial config has version 0 (not explicitly
            // specified by the user; the lack of version in a config file is interpreted as version=0).
            // As soon as a config is established we would like to increase its version so that it
            // takes presedence over other initial configs that were not established (such as a config
            // of a server trying to join the ensemble, which may be a partial view of the system, not the full config).
            // We chose to set the new version to the one of the NEWLEADER message. However, before we can do that
            // there must be agreement on the new version, so we can only change the version when sending/receiving UPTODATE,
            // not when sending/receiving NEWLEADER. In other words, we can't change curQV here since its the committed quorum verifier,
            // and there's still no agreement on the new version that we'd like to use. Instead, we use
            // lastSeenQuorumVerifier which is being sent with NEWLEADER message
            // so its a good way to let followers know about the new version. (The original reason for sending
            // lastSeenQuorumVerifier with NEWLEADER is so that the leader completes any potentially uncommitted reconfigs
            // that it finds before starting to propose operations. Here we're reusing the same code path for
            // reaching consensus on the new version number.)

            // It is important that this is done before the leader executes waitForEpochAck,
            // so before LearnerHandlers return from their waitForEpochAck
            // hence before they construct the NEWLEADER message containing
            // the last-seen-quorumverifier of the leader, which we change below
            try {
                LOG.debug(String.format("set lastSeenQuorumVerifier to currentQuorumVerifier (%s)", curQV.toString()));
                QuorumVerifier newQV = self.configFromString(curQV.toString());
                newQV.setVersion(zk.getZxid());
                self.setLastSeenQuorumVerifier(newQV, true);
            } catch (Exception e) {
                throw new IOException(e);
            }
        }

        newLeaderProposal.addQuorumVerifier(self.getQuorumVerifier());
        if (self.getLastSeenQuorumVerifier().getVersion() > self.getQuorumVerifier().getVersion()) {
            newLeaderProposal.addQuorumVerifier(self.getLastSeenQuorumVerifier());
        }

        // We have to get at least a majority of servers in sync with
        // us. We do this by waiting for the NEWLEADER packet to get
        // acknowledged

        waitForEpochAck(self.getId(), leaderStateSummary);
        self.setCurrentEpoch(epoch);
        self.setLeaderAddressAndId(self.getQuorumAddress(), self.getId());
        self.setZabState(QuorumPeer.ZabState.SYNCHRONIZATION);

        try {
            waitForNewLeaderAck(self.getId(), zk.getZxid());
        } catch (InterruptedException e) {
            shutdown("Waiting for a quorum of followers, only synced with sids: [ "
                     + newLeaderProposal.ackSetsToString()
                     + " ]");
            HashSet<Long> followerSet = new HashSet<Long>();

            for (LearnerHandler f : getLearners()) {
                if (self.getQuorumVerifier().getVotingMembers().containsKey(f.getSid())) {
                    followerSet.add(f.getSid());
                }
            }
            boolean initTicksShouldBeIncreased = true;
            for (Proposal.QuorumVerifierAcksetPair qvAckset : newLeaderProposal.qvAcksetPairs) {
                if (!qvAckset.getQuorumVerifier().containsQuorum(followerSet)) {
                    initTicksShouldBeIncreased = false;
                    break;
                }
            }
            if (initTicksShouldBeIncreased) {
                LOG.warn("Enough followers present. Perhaps the initTicks need to be increased.");
            }
            return;
        }

        startZkServer();

        /**
         * WARNING: do not use this for anything other than QA testing
         * on a real cluster. Specifically to enable verification that quorum
         * can handle the lower 32bit roll-over issue identified in
         * ZOOKEEPER-1277. Without this option it would take a very long
         * time (on order of a month say) to see the 4 billion writes
         * necessary to cause the roll-over to occur.
         *
         * This field allows you to override the zxid of the server. Typically
         * you'll want to set it to something like 0xfffffff0 and then
         * start the quorum, run some operations and see the re-election.
         */
        String initialZxid = System.getProperty("zookeeper.testingonly.initialZxid");
        if (initialZxid != null) {
            long zxid = Long.parseLong(initialZxid);
            zk.setZxid((zk.getZxid() & 0xffffffff00000000L) | zxid);
        }

        if (!System.getProperty("zookeeper.leaderServes", "yes").equals("no")) {
            self.setZooKeeperServer(zk);
        }

        self.setZabState(QuorumPeer.ZabState.BROADCAST);
        self.adminServer.setZooKeeperServer(zk);

        // We ping twice a tick, so we only update the tick every other
        // iteration
        boolean tickSkip = true;
        // If not null then shutdown this leader
        String shutdownMessage = null;

        while (true) {
            synchronized (this) {
                // 每隔一段时间构建一次心跳
                long start = Time.currentElapsedTime();
                long cur = start;
                long end = start + self.tickTime / 2;
                while (cur < end) {
                    wait(end - cur);
                    cur = Time.currentElapsedTime();
                }

                if (!tickSkip) {
                    self.tick.incrementAndGet();
                }

                // We use an instance of SyncedLearnerTracker to
                // track synced learners to make sure we still have a
                // quorum of current (and potentially next pending) view.
                SyncedLearnerTracker syncedAckSet = new SyncedLearnerTracker();
                syncedAckSet.addQuorumVerifier(self.getQuorumVerifier());
                if (self.getLastSeenQuorumVerifier() != null
                    && self.getLastSeenQuorumVerifier().getVersion() > self.getQuorumVerifier().getVersion()) {
                    syncedAckSet.addQuorumVerifier(self.getLastSeenQuorumVerifier());
                }

                syncedAckSet.addAck(self.getId());

                for (LearnerHandler f : getLearners()) {
                    if (f.synced()) {
                        syncedAckSet.addAck(f.getSid());
                    }
                }

                // check leader running status
                if (!this.isRunning()) {
                    // set shutdown flag
                    shutdownMessage = "Unexpected internal error";
                    break;
                }

                /*
                 *
                 * We will need to re-validate the outstandingProposal to maintain the progress of ZooKeeper.
                 * It is likely a proposal is waiting for enough ACKs to be committed. The proposals are sent out, but the
                 * only follower goes away which makes the proposals will not be committed until the follower recovers back.
                 * An earlier proposal which is not committed will block any further proposals. So, We need to re-validate those
                 * outstanding proposal with the help from Oracle. A key point in the process of re-validation is that the proposals
                 * need to be processed in order.
                 *
                 * We make the whole method blocking to avoid any possible race condition on outstandingProposal and lastCommitted
                 * as well as to avoid nested synchronization.
                 *
                 * As a more generic approach, we pass the object of forwardingFollowers to QuorumOracleMaj to determine if we need
                 * the help from Oracle.
                 *
                 *
                 * the size of outstandingProposals can be 1. The only one outstanding proposal is the one waiting for the ACK from
                 * the leader itself.
                 * */
                if (!tickSkip && !syncedAckSet.hasAllQuorums()
                    && !(self.getQuorumVerifier().overrideQuorumDecision(getForwardingFollowers()) && self.getQuorumVerifier().revalidateOutstandingProp(this, new ArrayList<>(outstandingProposals.values()), lastCommitted))) {
                    // Lost quorum of last committed and/or last proposed
                    // config, set shutdown flag
                    shutdownMessage = "Not sufficient followers synced, only synced with sids: [ "
                                      + syncedAckSet.ackSetsToString()
                                      + " ]";
                    break;
                }
                tickSkip = !tickSkip;
            }
            // 官方注释：learners包含所有的followers and observers
            for (LearnerHandler f : getLearners()) {
                // 进行ping
                f.ping();
            }
        }
        if (shutdownMessage != null) {
            shutdown(shutdownMessage);
            // leader goes in looking state
        }
    } finally {
        zk.unregisterJMX(this);
    }
}
```

### 3.2.1LearnerCnxAcceptor#run
```java
public void run() {
    if (!stop.get() && !serverSockets.isEmpty()) {
        ExecutorService executor = Executors.newFixedThreadPool(serverSockets.size());
        CountDownLatch latch = new CountDownLatch(serverSockets.size());

        serverSockets.forEach(serverSocket ->
                executor.submit(new LearnerCnxAcceptorHandler(serverSocket, latch)));

        try {
            latch.await();
        } catch (InterruptedException ie) {
            LOG.error("Interrupted while sleeping in LearnerCnxAcceptor.", ie);
        } finally {
            closeSockets();
            executor.shutdown();
            try {
                if (!executor.awaitTermination(1, TimeUnit.SECONDS)) {
                    LOG.error("not all the LearnerCnxAcceptorHandler terminated properly");
                }
            } catch (InterruptedException ie) {
                LOG.error("Interrupted while terminating LearnerCnxAcceptor.", ie);
            }
        }
    }
}
```

### 3.2.2Leader#getLearners
```java
// list of all the learners, including followers and observers
// 官方注释：learners包含所有的followers and observers
private final HashSet<LearnerHandler> learners = new HashSet<LearnerHandler>();
public List<LearnerHandler> getLearners() {
    synchronized (learners) {
        return new ArrayList<LearnerHandler>(learners);
    }
}
```
### 3.2.3LearnerHandler#ping
```java
public void ping() {
    // If learner hasn't sync properly yet, don't send ping packet
    // otherwise, the learner will crash
    if (!sendingThreadStarted) {
        return;
    }
    long id;
    if (syncLimitCheck.check(System.nanoTime())) {
        id = learnerMaster.getLastProposed();
        // 构建类型为PING，数据为null的数据
        QuorumPacket ping = new QuorumPacket(Leader.PING, id, null, null);
        // 存入queuedPackets队列    
        queuePacket(ping);
    } else {
        LOG.warn("Closing connection to peer due to transaction timeout.");
        shutdown();
    }
}
```
## 4.QuorumPeer#run-FOLLOWING
```java
 case FOLLOWING:
    try {
        LOG.info("FOLLOWING");
        // 包装对象
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
```

### 4.1Follower#followLeader
```java
void followLeader() throws InterruptedException {
    self.end_fle = Time.currentElapsedTime();
    long electionTimeTaken = self.end_fle - self.start_fle;
    self.setElectionTimeTaken(electionTimeTaken);
    ServerMetrics.getMetrics().ELECTION_TIME.add(electionTimeTaken);
    LOG.info("FOLLOWING - LEADER ELECTION TOOK - {} {}", electionTimeTaken, QuorumPeer.FLE_TIME_UNIT);
    self.start_fle = 0;
    self.end_fle = 0;
    fzk.registerJMX(new FollowerBean(this, zk), self.jmxLocalPeerBean);

    long connectionTime = 0;
    boolean completedSync = false;

    try {
        self.setZabState(QuorumPeer.ZabState.DISCOVERY);
        // 核心代码寻找leader,获取leader的地址
        QuorumServer leaderServer = findLeader();
        try {
            // 创建socket连接leader
            connectToLeader(leaderServer.addr, leaderServer.hostname);
            connectionTime = System.currentTimeMillis();
            long newEpochZxid = registerWithLeader(Leader.FOLLOWERINFO);
            if (self.isReconfigStateChange()) {
                throw new Exception("learned about role change");
            }
            //check to see if the leader zxid is lower than ours
            //this should never happen but is just a safety check
            long newEpoch = ZxidUtils.getEpochFromZxid(newEpochZxid);
            if (newEpoch < self.getAcceptedEpoch()) {
                LOG.error("Proposed leader epoch "
                          + ZxidUtils.zxidToString(newEpochZxid)
                          + " is less than our accepted epoch "
                          + ZxidUtils.zxidToString(self.getAcceptedEpoch()));
                throw new IOException("Error: Epoch of leader is lower");
            }
            long startTime = Time.currentElapsedTime();
            self.setLeaderAddressAndId(leaderServer.addr, leaderServer.getId());
            self.setZabState(QuorumPeer.ZabState.SYNCHRONIZATION);
            syncWithLeader(newEpochZxid);
            self.setZabState(QuorumPeer.ZabState.BROADCAST);
            completedSync = true;
            long syncTime = Time.currentElapsedTime() - startTime;
            ServerMetrics.getMetrics().FOLLOWER_SYNC_TIME.add(syncTime);
            if (self.getObserverMasterPort() > 0) {
                LOG.info("Starting ObserverMaster");

                om = new ObserverMaster(self, fzk, self.getObserverMasterPort());
                om.start();
            } else {
                om = null;
            }
            // create a reusable packet to reduce gc impact
            QuorumPacket qp = new QuorumPacket();
            // 自旋进行数据同步
            while (this.isRunning()) {
                readPacket(qp);
                processPacket(qp);
            }
        } catch (Exception e) {
            // 如果从socket读数据出现异常则打印日志关闭socket
            LOG.warn("Exception when following the leader", e);
            closeSocket();

            // clear pending revalidations
            pendingRevalidations.clear();
        }
    } finally {
        if (om != null) {
            om.stop();
        }
        zk.unregisterJMX(this);

        if (connectionTime != 0) {
            long connectionDuration = System.currentTimeMillis() - connectionTime;
            LOG.info(
                "Disconnected from leader (with address: {}). Was connected for {}ms. Sync state: {}",
                leaderAddr,
                connectionDuration,
                completedSync);
            messageTracker.dumpToLog(leaderAddr.toString());
        }
    }
}
```
#### 4.1.1Learner#readPacket
```java
void readPacket(QuorumPacket pp) throws IOException {
    synchronized (leaderIs) {
        leaderIs.readRecord(pp, "packet");
        messageTracker.trackReceived(pp.getType());
    }
    if (LOG.isTraceEnabled()) {
        final long traceMask =
            (pp.getType() == Leader.PING) ? ZooTrace.SERVER_PING_TRACE_MASK
                : ZooTrace.SERVER_PACKET_TRACE_MASK;

        ZooTrace.logQuorumPacket(LOG, traceMask, 'i', pp);
    }
}
```
#### 4.1.1.1BinaryInputArchive#readRecord
- BinaryInputArchive#readRecord
```java
public void readRecord(Record r, String tag) throws IOException {
        r.deserialize(this, tag);
    }
```
- QuorumPacket#deserialize
```java
public void deserialize(InputArchive a_, String tag) throws java.io.IOException {
    a_.startRecord(tag);
    // 以readInt为例in.readInt()，in就是input流，即与leader建立的socket
    type=a_.readInt("type");
    zxid=a_.readLong("zxid");
    data=a_.readBuffer("data");
    {
      Index vidx1 = a_.startVector("authinfo");
      if (vidx1!= null) {          authinfo=new java.util.ArrayList<org.apache.zookeeper.data.Id>();
          for (; !vidx1.done(); vidx1.incr()) {
    org.apache.zookeeper.data.Id e1;
    e1= new org.apache.zookeeper.data.Id();
    a_.readRecord(e1,"e1");
            authinfo.add(e1);
          }
      }
    a_.endVector("authinfo");
    }
    a_.endRecord(tag);
}
```
