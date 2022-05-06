# 【JAVA分布式】02-spring扫描BeanDefinition
> 欢迎关注b站账号/公众号【六边形战士夏宁】，一个要把各项指标拉满的男人。该文章已在[github目录](https://github.com/edanlx/SealBook/blob/master/catalogue/wechat.md)收录。
屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。

## 1.背景
zookeeper是Apache下Hadoop的子项目，分布式协调框架。用来解决统一命名服务，状态同步服务，集群管理，分布式应用配置的项目管理等

## 2.消息存储
1. CommitLog消息存储
2. config配置
3. consumerqueue消息队列
4. indeix索引
5. abort正常关闭检测
6. checkpoint文件检车点

- phyOffset
消息存储在commitlog的偏移量。
- storeTimestamp
消息存入commitlog的时间戳。

DLedger基于Raft协议进行多副本同步，会分为两个阶段，一个是uncommitted阶段，一个是commited阶段。首先，Leader Broker上的DLedger收到一条数据之后，会标记为uncommitted状态，然后它通过自己的DledgerServer组件把这个uncommitted数据发送给Follower Broker的DledgerServer。如是是非高可用也没有选举过程，是指定的
1. 同步数据
2. 选举
3. 由DLedger提供api写数据
- rebalance
    - 危害
        1. 消费暂停
        2. 重复消费
        3. 消费突刺
```java
public CompletableFuture<PutMessageResult> asyncPutMessage(MessageExtBrokerInner msg) {
    PutMessageStatus checkStoreStatus = this.checkStoreStatus();
    if (checkStoreStatus != PutMessageStatus.PUT_OK) {
        return CompletableFuture.completedFuture(new PutMessageResult(checkStoreStatus, null));
    }

    PutMessageStatus msgCheckStatus = this.checkMessage(msg);
    if (msgCheckStatus == PutMessageStatus.MESSAGE_ILLEGAL) {
        return CompletableFuture.completedFuture(new PutMessageResult(msgCheckStatus, null));
    }

    PutMessageStatus lmqMsgCheckStatus = this.checkLmqMessage(msg);
    if (msgCheckStatus == PutMessageStatus.LMQ_CONSUME_QUEUE_NUM_EXCEEDED) {
        return CompletableFuture.completedFuture(new PutMessageResult(lmqMsgCheckStatus, null));
    }


    long beginTime = this.getSystemClock().now();
    // 多线程持久化(以前是单线程)
    CompletableFuture<PutMessageResult> putResultFuture = this.commitLog.asyncPutMessage(msg);

    putResultFuture.thenAccept((result) -> {
        long elapsedTime = this.getSystemClock().now() - beginTime;
        if (elapsedTime > 500) {
            log.warn("putMessage not in lock elapsed time(ms)={}, bodyLength={}", elapsedTime, msg.getBody().length);
        }
        this.storeStatsService.setPutMessageEntireTimeMax(elapsedTime);

        if (null == result || !result.isOk()) {
            this.storeStatsService.getPutMessageFailedTimes().add(1);
        }
    });

    return putResultFuture;
}
```
### 7.1CommitLog#asyncPutMessage|消息写盘以及同步
```java
public CompletableFuture<PutMessageResult> asyncPutMessage(final MessageExtBrokerInner msg) {
    // Set the storage time
    msg.setStoreTimestamp(System.currentTimeMillis());
    // Set the message body BODY CRC (consider the most appropriate setting
    // on the client)
    msg.setBodyCRC(UtilAll.crc32(msg.getBody()));
    // Back to Results
    AppendMessageResult result = null;

    StoreStatsService storeStatsService = this.defaultMessageStore.getStoreStatsService();

    String topic = msg.getTopic();
//        int queueId msg.getQueueId();
    final int tranType = MessageSysFlag.getTransactionValue(msg.getSysFlag());
    /*
    public final static int COMPRESSED_FLAG = 0x1;
    public final static int MULTI_TAGS_FLAG = 0x1 << 1;
    public final static int TRANSACTION_NOT_TYPE = 0;
    public final static int TRANSACTION_PREPARED_TYPE = 0x1 << 2;
    public final static int TRANSACTION_COMMIT_TYPE = 0x2 << 2;
    public final static int TRANSACTION_ROLLBACK_TYPE = 0x3 << 2;
    public final static int BORNHOST_V6_FLAG = 0x1 << 4;
    public final static int STOREHOSTADDRESS_V6_FLAG = 0x1 << 5;
    */
    // 判断消息类型，非事务的进入
    if (tranType == MessageSysFlag.TRANSACTION_NOT_TYPE
            || tranType == MessageSysFlag.TRANSACTION_COMMIT_TYPE) {
        // Delay Delivery
        // 如果是延迟消息
        if (msg.getDelayTimeLevel() > 0) {
            if (msg.getDelayTimeLevel() > this.defaultMessageStore.getScheduleMessageService().getMaxDelayLevel()) {
                msg.setDelayTimeLevel(this.defaultMessageStore.getScheduleMessageService().getMaxDelayLevel());
            }

            topic = TopicValidator.RMQ_SYS_SCHEDULE_TOPIC;
            int queueId = ScheduleMessageService.delayLevel2QueueId(msg.getDelayTimeLevel());

            // Backup real topic, queueId
            MessageAccessor.putProperty(msg, MessageConst.PROPERTY_REAL_TOPIC, msg.getTopic());
            MessageAccessor.putProperty(msg, MessageConst.PROPERTY_REAL_QUEUE_ID, String.valueOf(msg.getQueueId()));
            msg.setPropertiesString(MessageDecoder.messageProperties2String(msg.getProperties()));
            // 修改topic这样broker暂时就拿不到了，后续通过其它定时线程再改回来org.apache.rocketmq.store.schedule.ScheduleMessageService#start
            msg.setTopic(topic);
            msg.setQueueId(queueId);
        }
    }

    InetSocketAddress bornSocketAddress = (InetSocketAddress) msg.getBornHost();
    if (bornSocketAddress.getAddress() instanceof Inet6Address) {
        msg.setBornHostV6Flag();
    }

    InetSocketAddress storeSocketAddress = (InetSocketAddress) msg.getStoreHost();
    if (storeSocketAddress.getAddress() instanceof Inet6Address) {
        msg.setStoreHostAddressV6Flag();
    }

    PutMessageThreadLocal putMessageThreadLocal = this.putMessageThreadLocal.get();
    PutMessageResult encodeResult = putMessageThreadLocal.getEncoder().encode(msg);
    if (encodeResult != null) {
        return CompletableFuture.completedFuture(encodeResult);
    }
    msg.setEncodedBuff(putMessageThreadLocal.getEncoder().encoderBuffer);
    PutMessageContext putMessageContext = new PutMessageContext(generateKey(putMessageThreadLocal.getKeyBuilder(), msg));

    long elapsedTimeInLock = 0;
    MappedFile unlockMappedFile = null;

    // 线程锁
    putMessageLock.lock(); //spin or ReentrantLock ,depending on store config
    try {
        // mmap零拷贝关键
        MappedFile mappedFile = this.mappedFileQueue.getLastMappedFile();
        long beginLockTimestamp = this.defaultMessageStore.getSystemClock().now();
        this.beginTimeInLock = beginLockTimestamp;

        // Here settings are stored timestamp, in order to ensure an orderly
        // global
        msg.setStoreTimestamp(beginLockTimestamp);

        if (null == mappedFile || mappedFile.isFull()) {
            mappedFile = this.mappedFileQueue.getLastMappedFile(0); // Mark: NewFile may be cause noise
        }
        if (null == mappedFile) {
            log.error("create mapped file1 error, topic: " + msg.getTopic() + " clientAddr: " + msg.getBornHostString());
            return CompletableFuture.completedFuture(new PutMessageResult(PutMessageStatus.CREATE_MAPEDFILE_FAILED, null));
        }
        // 顺序写入
        result = mappedFile.appendMessage(msg, this.appendMessageCallback, putMessageContext);
        // 判断文件是否写满
        switch (result.getStatus()) {
            case PUT_OK:
                break;
            case END_OF_FILE:
                // 创建新文件
                unlockMappedFile = mappedFile;
                // Create a new file, re-write the message
                mappedFile = this.mappedFileQueue.getLastMappedFile(0);
                if (null == mappedFile) {
                    // XXX: warn and notify me
                    log.error("create mapped file2 error, topic: " + msg.getTopic() + " clientAddr: " + msg.getBornHostString());
                    return CompletableFuture.completedFuture(new PutMessageResult(PutMessageStatus.CREATE_MAPEDFILE_FAILED, result));
                }
                result = mappedFile.appendMessage(msg, this.appendMessageCallback, putMessageContext);
                break;
                // 消息过大等其他错误
            case MESSAGE_SIZE_EXCEEDED:
            case PROPERTIES_SIZE_EXCEEDED:
                return CompletableFuture.completedFuture(new PutMessageResult(PutMessageStatus.MESSAGE_ILLEGAL, result));
            case UNKNOWN_ERROR:
                return CompletableFuture.completedFuture(new PutMessageResult(PutMessageStatus.UNKNOWN_ERROR, result));
            default:
                return CompletableFuture.completedFuture(new PutMessageResult(PutMessageStatus.UNKNOWN_ERROR, result));
        }

        elapsedTimeInLock = this.defaultMessageStore.getSystemClock().now() - beginLockTimestamp;
    } finally {
        beginTimeInLock = 0;
        putMessageLock.unlock();
    }

    if (elapsedTimeInLock > 500) {
        log.warn("[NOTIFYME]putMessage in lock cost time(ms)={}, bodyLength={} AppendMessageResult={}", elapsedTimeInLock, msg.getBody().length, result);
    }

    if (null != unlockMappedFile && this.defaultMessageStore.getMessageStoreConfig().isWarmMapedFileEnable()) {
        this.defaultMessageStore.unlockMappedFile(unlockMappedFile);
    }

    PutMessageResult putMessageResult = new PutMessageResult(PutMessageStatus.PUT_OK, result);

    // Statistics
    storeStatsService.getSinglePutMessageTopicTimesTotal(msg.getTopic()).add(1);
    storeStatsService.getSinglePutMessageTopicSizeTotal(topic).add(result.getWroteBytes());
    // 多线程文件刷盘
    CompletableFuture<PutMessageStatus> flushResultFuture = submitFlushRequest(result, msg);
    // 多线程主从同步
    CompletableFuture<PutMessageStatus> replicaResultFuture = submitReplicaRequest(result, msg);
    return flushResultFuture.thenCombine(replicaResultFuture, (flushStatus, replicaStatus) -> {
        if (flushStatus != PutMessageStatus.PUT_OK) {
            putMessageResult.setPutMessageStatus(flushStatus);
        }
        if (replicaStatus != PutMessageStatus.PUT_OK) {
            putMessageResult.setPutMessageStatus(replicaStatus);
            if (replicaStatus == PutMessageStatus.FLUSH_SLAVE_TIMEOUT) {
                log.error("do sync transfer other node, wait return, but failed, topic: {} tags: {} client address: {}",
                        msg.getTopic(), msg.getTags(), msg.getBornHostNameString());
            }
        }
        return putMessageResult;
    });
}
```
## 3.org.apache.rocketmq.store.DefaultMessageStore#start|分发index、删除过期文件
```java
public void start() throws Exception {

    lock = lockFile.getChannel().tryLock(0, 1, false);
    if (lock == null || lock.isShared() || !lock.isValid()) {
        throw new RuntimeException("Lock failed,MQ already started");
    }

    lockFile.getChannel().write(ByteBuffer.wrap("lock".getBytes()));
    lockFile.getChannel().force(true);
    {
        /**
         * 1. Make sure the fast-forward messages to be truncated during the recovering according to the max physical offset of the commitlog;
         * 2. DLedger committedPos may be missing, so the maxPhysicalPosInLogicQueue maybe bigger that maxOffset returned by DLedgerCommitLog, just let it go;
         * 3. Calculate the reput offset according to the consume queue;
         * 4. Make sure the fall-behind messages to be dispatched before starting the commitlog, especially when the broker role are automatically changed.
         */
        long maxPhysicalPosInLogicQueue = commitLog.getMinOffset();
        for (ConcurrentMap<Integer, ConsumeQueue> maps : this.consumeQueueTable.values()) {
            for (ConsumeQueue logic : maps.values()) {
                if (logic.getMaxPhysicOffset() > maxPhysicalPosInLogicQueue) {
                    maxPhysicalPosInLogicQueue = logic.getMaxPhysicOffset();
                }
            }
        }
        if (maxPhysicalPosInLogicQueue < 0) {
            maxPhysicalPosInLogicQueue = 0;
        }
        if (maxPhysicalPosInLogicQueue < this.commitLog.getMinOffset()) {
            maxPhysicalPosInLogicQueue = this.commitLog.getMinOffset();
            /**
             * This happens in following conditions:
             * 1. If someone removes all the consumequeue files or the disk get damaged.
             * 2. Launch a new broker, and copy the commitlog from other brokers.
             *
             * All the conditions has the same in common that the maxPhysicalPosInLogicQueue should be 0.
             * If the maxPhysicalPosInLogicQueue is gt 0, there maybe something wrong.
             */
            log.warn("[TooSmallCqOffset] maxPhysicalPosInLogicQueue={} clMinOffset={}", maxPhysicalPosInLogicQueue, this.commitLog.getMinOffset());
        }
        log.info("[SetReputOffset] maxPhysicalPosInLogicQueue={} clMinOffset={} clMaxOffset={} clConfirmedOffset={}",
            maxPhysicalPosInLogicQueue, this.commitLog.getMinOffset(), this.commitLog.getMaxOffset(), this.commitLog.getConfirmOffset());
        this.reputMessageService.setReputFromOffset(maxPhysicalPosInLogicQueue);
        // 分发线程
        this.reputMessageService.start();

        /**
         *  1. Finish dispatching the messages fall behind, then to start other services.
         *  2. DLedger committedPos may be missing, so here just require dispatchBehindBytes <= 0
         */
        while (true) {
            if (dispatchBehindBytes() <= 0) {
                break;
            }
            Thread.sleep(1000);
            log.info("Try to finish doing reput the messages fall behind during the starting, reputOffset={} maxOffset={} behind={}", this.reputMessageService.getReputFromOffset(), this.getMaxPhyOffset(), this.dispatchBehindBytes());
        }
        this.recoverTopicQueueTable();
    }

    if (!messageStoreConfig.isEnableDLegerCommitLog()) {
        this.haService.start();
        this.handleScheduleMessageService(messageStoreConfig.getBrokerRole());
    }

    this.flushConsumeQueueService.start();
    this.commitLog.start();
    this.storeStatsService.start();

    this.createTempFile();
    // 添加多个schedule其中有删除过期文件
    this.addScheduleTask();
    this.shutdown = false;
}
```
### 3.1ReputMessageService#run
```java
public void run() {
        DefaultMessageStore.log.info(this.getServiceName() + " service started");

        while (!this.isStopped()) {
            try {
                Thread.sleep(1);
                this.doReput();
            } catch (Exception e) {
                DefaultMessageStore.log.warn(this.getServiceName() + " service has exception. ", e);
            }
        }

        DefaultMessageStore.log.info(this.getServiceName() + " service end");
    }
```
- ReputMessageService#doReput
```java
private void doReput() {
    if (this.reputFromOffset < DefaultMessageStore.this.commitLog.getMinOffset()) {
        log.warn("The reputFromOffset={} is smaller than minPyOffset={}, this usually indicate that the dispatch behind too much and the commitlog has expired.",
            this.reputFromOffset, DefaultMessageStore.this.commitLog.getMinOffset());
        this.reputFromOffset = DefaultMessageStore.this.commitLog.getMinOffset();
    }
    // 判断是否有新的消息
    for (boolean doNext = true; this.isCommitLogAvailable() && doNext; ) {

        if (DefaultMessageStore.this.getMessageStoreConfig().isDuplicationEnable()
            && this.reputFromOffset >= DefaultMessageStore.this.getConfirmOffset()) {
            break;
        }

        SelectMappedBufferResult result = DefaultMessageStore.this.commitLog.getData(reputFromOffset);
        if (result != null) {
            try {
                this.reputFromOffset = result.getStartOffset();

                for (int readSize = 0; readSize < result.getSize() && doNext; ) {
                    DispatchRequest dispatchRequest =
                        DefaultMessageStore.this.commitLog.checkMessageAndReturnSize(result.getByteBuffer(), false, false);
                    int size = dispatchRequest.getBufferSize() == -1 ? dispatchRequest.getMsgSize() : dispatchRequest.getBufferSize();

                    if (dispatchRequest.isSuccess()) {
                        if (size > 0) {
                            /*
                            // CommitLogDispatcherBuildConsumeQueue
                            // CommitLogDispatcherBuildIndex
                            // CommitLogDispatcherCalcBitMap
                            public void doDispatch(DispatchRequest req) {
                                for (CommitLogDispatcher dispatcher : this.dispatcherList) {
                                    dispatcher.dispatch(req);
                                }
                            }
                            */
                            DefaultMessageStore.this.doDispatch(dispatchRequest);

                            if (BrokerRole.SLAVE != DefaultMessageStore.this.getMessageStoreConfig().getBrokerRole()
                                    && DefaultMessageStore.this.brokerConfig.isLongPollingEnable()
                                    && DefaultMessageStore.this.messageArrivingListener != null) {
                                DefaultMessageStore.this.messageArrivingListener.arriving(dispatchRequest.getTopic(),
                                    dispatchRequest.getQueueId(), dispatchRequest.getConsumeQueueOffset() + 1,
                                    dispatchRequest.getTagsCode(), dispatchRequest.getStoreTimestamp(),
                                    dispatchRequest.getBitMap(), dispatchRequest.getPropertiesMap());
                                notifyMessageArrive4MultiQueue(dispatchRequest);
                            }

                            this.reputFromOffset += size;
                            readSize += size;
                            if (DefaultMessageStore.this.getMessageStoreConfig().getBrokerRole() == BrokerRole.SLAVE) {
                                DefaultMessageStore.this.storeStatsService
                                    .getSinglePutMessageTopicTimesTotal(dispatchRequest.getTopic()).add(1);
                                DefaultMessageStore.this.storeStatsService
                                    .getSinglePutMessageTopicSizeTotal(dispatchRequest.getTopic())
                                    .add(dispatchRequest.getMsgSize());
                            }
                        } else if (size == 0) {
                            this.reputFromOffset = DefaultMessageStore.this.commitLog.rollNextFile(this.reputFromOffset);
                            readSize = result.getSize();
                        }
                    } else if (!dispatchRequest.isSuccess()) {

                        if (size > 0) {
                            log.error("[BUG]read total count not equals msg total size. reputFromOffset={}", reputFromOffset);
                            this.reputFromOffset += size;
                        } else {
                            doNext = false;
                            // If user open the dledger pattern or the broker is master node,
                            // it will not ignore the exception and fix the reputFromOffset variable
                            if (DefaultMessageStore.this.getMessageStoreConfig().isEnableDLegerCommitLog() ||
                                DefaultMessageStore.this.brokerConfig.getBrokerId() == MixAll.MASTER_ID) {
                                log.error("[BUG]dispatch message to consume queue error, COMMITLOG OFFSET: {}",
                                    this.reputFromOffset);
                                this.reputFromOffset += result.getSize() - readSize;
                            }
                        }
                    }
                }
            } finally {
                result.release();
            }
        } else {
            doNext = false;
        }
    }
}
```
### 3.2DefaultMessageStore#addScheduleTask
```java
private void addScheduleTask() {
    // 删除过期文件
    this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
        @Override
        public void run() {
            DefaultMessageStore.this.cleanFilesPeriodically();
        }
    }, 1000 * 60, this.messageStoreConfig.getCleanResourceInterval(), TimeUnit.MILLISECONDS);

    this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
        @Override
        public void run() {
            DefaultMessageStore.this.checkSelf();
        }
    }, 1, 10, TimeUnit.MINUTES);

    this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
        @Override
        public void run() {
            if (DefaultMessageStore.this.getMessageStoreConfig().isDebugLockEnable()) {
                try {
                    if (DefaultMessageStore.this.commitLog.getBeginTimeInLock() != 0) {
                        long lockTime = System.currentTimeMillis() - DefaultMessageStore.this.commitLog.getBeginTimeInLock();
                        if (lockTime > 1000 && lockTime < 10000000) {

                            String stack = UtilAll.jstack();
                            final String fileName = System.getProperty("user.home") + File.separator + "debug/lock/stack-"
                                + DefaultMessageStore.this.commitLog.getBeginTimeInLock() + "-" + lockTime;
                            MixAll.string2FileNotSafe(stack, fileName);
                        }
                    }
                } catch (Exception e) {
                }
            }
        }
    }, 1, 1, TimeUnit.SECONDS);

    // this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
    // @Override
    // public void run() {
    // DefaultMessageStore.this.cleanExpiredConsumerQueue();
    // }
    // }, 1, 1, TimeUnit.HOURS);
    this.diskCheckScheduledExecutorService.scheduleAtFixedRate(new Runnable() {
        public void run() {
            DefaultMessageStore.this.cleanCommitLogService.isSpaceFull();
        }
    }, 1000L, 10000L, TimeUnit.MILLISECONDS);
}
```
## 4.org.apache.rocketmq.example.quickstart.Consumer#main|消费者
- org.apache.rocketmq.client.consumer.DefaultMQPushConsumer#start
```java
public void start() throws MQClientException {
    // 设置消费者组
    setConsumerGroup(NamespaceUtil.wrapNamespace(this.getNamespace(), this.consumerGroup));
    this.defaultMQPushConsumerImpl.start();
    if (null != traceDispatcher) {
        try {
            // 消息轨迹
            traceDispatcher.start(this.getNamesrvAddr(), this.getAccessChannel());
        } catch (MQClientException e) {
            log.warn("trace dispatcher start failed ", e);
        }
    }
}
```
- org.apache.rocketmq.client.impl.consumer.DefaultMQPushConsumerImpl#start
```java
public synchronized void start() throws MQClientException {
    switch (this.serviceState) {
        case CREATE_JUST:
            // 服务刚创建
            log.info("the consumer [{}] start beginning. messageModel={}, isUnitMode={}", this.defaultMQPushConsumer.getConsumerGroup(),
                this.defaultMQPushConsumer.getMessageModel(), this.defaultMQPushConsumer.isUnitMode());
            this.serviceState = ServiceState.START_FAILED;
            // 检查配置
            this.checkConfig();
            // 订阅情况
            this.copySubscription();

            if (this.defaultMQPushConsumer.getMessageModel() == MessageModel.CLUSTERING) {
                this.defaultMQPushConsumer.changeInstanceNameToPID();
            }
            // 获取核心对象mQClientFactory
            this.mQClientFactory = MQClientManager.getInstance().getOrCreateMQClientInstance(this.defaultMQPushConsumer, this.rpcHook);
            // 配置负载均衡
            this.rebalanceImpl.setConsumerGroup(this.defaultMQPushConsumer.getConsumerGroup());
            this.rebalanceImpl.setMessageModel(this.defaultMQPushConsumer.getMessageModel());
            // AllocateMessageQueueAveragely默认平均算法
            this.rebalanceImpl.setAllocateMessageQueueStrategy(this.defaultMQPushConsumer.getAllocateMessageQueueStrategy());
            this.rebalanceImpl.setmQClientFactory(this.mQClientFactory);

            this.pullAPIWrapper = new PullAPIWrapper(
                mQClientFactory,
                this.defaultMQPushConsumer.getConsumerGroup(), isUnitMode());
            this.pullAPIWrapper.registerFilterMessageHook(filterMessageHookList);

            if (this.defaultMQPushConsumer.getOffsetStore() != null) {
                this.offsetStore = this.defaultMQPushConsumer.getOffsetStore();
            } else {
                switch (this.defaultMQPushConsumer.getMessageModel()) {
                    // 广播模式使用本地文件
                    case BROADCASTING:
                        this.offsetStore = new LocalFileOffsetStore(this.mQClientFactory, this.defaultMQPushConsumer.getConsumerGroup());
                        break;
                    // 集群模式使用broker
                    case CLUSTERING:
                        this.offsetStore = new RemoteBrokerOffsetStore(this.mQClientFactory, this.defaultMQPushConsumer.getConsumerGroup());
                        break;
                    default:
                        break;
                }
                this.defaultMQPushConsumer.setOffsetStore(this.offsetStore);
            }
            this.offsetStore.load();

            if (this.getMessageListenerInner() instanceof MessageListenerOrderly) {
                this.consumeOrderly = true;
                this.consumeMessageService =
                    new ConsumeMessageOrderlyService(this, (MessageListenerOrderly) this.getMessageListenerInner());
            } else if (this.getMessageListenerInner() instanceof MessageListenerConcurrently) {
                this.consumeOrderly = false;
                this.consumeMessageService =
                    new ConsumeMessageConcurrentlyService(this, (MessageListenerConcurrently) this.getMessageListenerInner());
            }

            this.consumeMessageService.start();

            boolean registerOK = mQClientFactory.registerConsumer(this.defaultMQPushConsumer.getConsumerGroup(), this);
            if (!registerOK) {
                this.serviceState = ServiceState.CREATE_JUST;
                this.consumeMessageService.shutdown(defaultMQPushConsumer.getAwaitTerminationMillisWhenShutdown());
                throw new MQClientException("The consumer group[" + this.defaultMQPushConsumer.getConsumerGroup()
                    + "] has been created before, specify another name please." + FAQUrl.suggestTodo(FAQUrl.GROUP_NAME_DUPLICATE_URL),
                    null);
            }

            // 核心启动
            mQClientFactory.start();
            log.info("the consumer [{}] start OK.", this.defaultMQPushConsumer.getConsumerGroup());
            this.serviceState = ServiceState.RUNNING;
            break;
        case RUNNING:
        case START_FAILED:
        case SHUTDOWN_ALREADY:
            throw new MQClientException("The PushConsumer service state not OK, maybe started once, "
                + this.serviceState
                + FAQUrl.suggestTodo(FAQUrl.CLIENT_SERVICE_NOT_OK),
                null);
        default:
            break;
    }

    this.updateTopicSubscribeInfoWhenSubscriptionChanged();
    this.mQClientFactory.checkClientInBroker();
    this.mQClientFactory.sendHeartbeatToAllBrokerWithLock();
    this.mQClientFactory.rebalanceImmediately();
}
```
org.apache.rocketmq.client.impl.factory.MQClientInstance#start
```java
public void start() throws MQClientException {

    synchronized (this) {
        switch (this.serviceState) {
            case CREATE_JUST:
                this.serviceState = ServiceState.START_FAILED;
                // If not specified,looking address from name server
                if (null == this.clientConfig.getNamesrvAddr()) {
                    this.mQClientAPIImpl.fetchNameServerAddr();
                }
                // Start request-response channel
                this.mQClientAPIImpl.start();
                // Start various schedule tasks
                this.startScheduledTask();
                // Start pull service
                // 内部会有流控，从数量、大小等方面，然后进行数据拉取,ConsumeMessageService中的两个实现类分别规定了普通消息(分页)和顺序消息的拉取
                this.pullMessageService.start();
                // Start rebalance service
                // 根据推模式还是拉模式进行负载均衡(其中的push也是基于pull实现)，也会真正执行前面设置的负载均衡策略
                this.rebalanceService.start();
                // Start push service
                // 已弃用
                this.defaultMQProducer.getDefaultMQProducerImpl().start(false);
                log.info("the client factory [{}] start OK", this.clientId);
                this.serviceState = ServiceState.RUNNING;
                break;
            case START_FAILED:
                throw new MQClientException("The Factory object[" + this.getClientId() + "] has been created before, and failed.", null);
            default:
                break;
        }
    }
}
```

## 9.延迟消息