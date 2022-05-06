ZAB原子广播协议
顺序一致性
## 3.ZAB
1. LeaderRequestProcessor
2. PrepRequestProcessor
生成zxid
3. ProposalRequestProcessor
    1. leader
        1. 发送数据
        2. 写本地文件
        3. 给自己送ack
    2. follwoer
        1. 写本地文件
        2. 返回ack
    3. leader
        1. 收到半数以上ack发送commit给其它follower
        2. 发送commit给其它observe
        3. 同步到内存
        4. 返回节点数据变动给客户端
    4. follwoer
        1. 写自己的内存数据
4. CommitProcessor
ack确认提交
5. ToBeAppliedRequestProcess
6. FinalRequestProcessor
应用到内存树  
最终完成后由服务端向客户端发送数据，客户端一直在netty的wait，收到消息后(Selector注册的读事件)通过notify唤醒
## 4.new ZooKeeper|客户端启动
```java
public ZooKeeper(String connectString, int sessionTimeout, Watcher watcher) throws IOException {
    this(connectString, sessionTimeout, watcher, false);
}
```
```java
public ZooKeeper(
    String connectString,
    int sessionTimeout,
    Watcher watcher,
    boolean canBeReadOnly) throws IOException {
    this(connectString, sessionTimeout, watcher, canBeReadOnly, createDefaultHostProvider(connectString));
}
```
```java
public ZooKeeper(
    String connectString,
    int sessionTimeout,
    Watcher watcher,
    boolean canBeReadOnly,
    HostProvider aHostProvider) throws IOException {
    this(connectString, sessionTimeout, watcher, canBeReadOnly, aHostProvider, null);
}
```
```java
public ZooKeeper(
    String connectString,
    int sessionTimeout,
    Watcher watcher,
    boolean canBeReadOnly,
    HostProvider hostProvider,
    ZKClientConfig clientConfig
) throws IOException {
    LOG.info(
        "Initiating client connection, connectString={} sessionTimeout={} watcher={}",
        connectString,
        sessionTimeout,
        watcher);

    this.clientConfig = clientConfig != null ? clientConfig : new ZKClientConfig();
    this.hostProvider = hostProvider;
    ConnectStringParser connectStringParser = new ConnectStringParser(connectString);

    cnxn = createConnection(
        connectStringParser.getChrootPath(),
        hostProvider,
        sessionTimeout,
        this.clientConfig,
        watcher,
        getClientCnxnSocket(),
        canBeReadOnly);
    cnxn.start();
}
```
```java
public void start() {
    sendThread.start();
    eventThread.start();
}
```
### 4.1org.apache.zookeeper.ClientCnxn.SendThread#run
```java
public void run() {
    clientCnxnSocket.introduce(this, sessionId, outgoingQueue);
    clientCnxnSocket.updateNow();
    clientCnxnSocket.updateLastSendAndHeard();
    int to;
    long lastPingRwServer = Time.currentElapsedTime();
    final int MAX_SEND_PING_INTERVAL = 10000; //10 seconds
    InetSocketAddress serverAddress = null;
    while (state.isAlive()) {
        try {
            if (!clientCnxnSocket.isConnected()) {
                // don't re-establish connection if we are closing
                if (closing) {
                    break;
                }
                if (rwServerAddress != null) {
                    serverAddress = rwServerAddress;
                    rwServerAddress = null;
                } else {
                    serverAddress = hostProvider.next(1000);
                }
                onConnecting(serverAddress);
                // 起socket与服务端建立连接
                startConnect(serverAddress);
                // Update now to start the connection timer right after we make a connection attempt
                clientCnxnSocket.updateNow();
                clientCnxnSocket.updateLastSendAndHeard();
            }

            if (state.isConnected()) {
                // determine whether we need to send an AuthFailed event.
                if (zooKeeperSaslClient != null) {
                    boolean sendAuthEvent = false;
                    if (zooKeeperSaslClient.getSaslState() == ZooKeeperSaslClient.SaslState.INITIAL) {
                        try {
                            zooKeeperSaslClient.initialize(ClientCnxn.this);
                        } catch (SaslException e) {
                            LOG.error("SASL authentication with Zookeeper Quorum member failed.", e);
                            changeZkState(States.AUTH_FAILED);
                            sendAuthEvent = true;
                        }
                    }
                    KeeperState authState = zooKeeperSaslClient.getKeeperState();
                    if (authState != null) {
                        if (authState == KeeperState.AuthFailed) {
                            // An authentication error occurred during authentication with the Zookeeper Server.
                            changeZkState(States.AUTH_FAILED);
                            sendAuthEvent = true;
                        } else {
                            if (authState == KeeperState.SaslAuthenticated) {
                                sendAuthEvent = true;
                            }
                        }
                    }

                    if (sendAuthEvent) {
                        eventThread.queueEvent(new WatchedEvent(Watcher.Event.EventType.None, authState, null));
                        if (state == States.AUTH_FAILED) {
                            eventThread.queueEventOfDeath();
                        }
                    }
                }
                to = readTimeout - clientCnxnSocket.getIdleRecv();
            } else {
                to = connectTimeout - clientCnxnSocket.getIdleRecv();
            }

            if (to <= 0) {
                String warnInfo = String.format(
                    "Client session timed out, have not heard from server in %dms for session id 0x%s",
                    clientCnxnSocket.getIdleRecv(),
                    Long.toHexString(sessionId));
                LOG.warn(warnInfo);
                throw new SessionTimeoutException(warnInfo);
            }
            if (state.isConnected()) {
                //1000(1 second) is to prevent race condition missing to send the second ping
                //also make sure not to send too many pings when readTimeout is small
                int timeToNextPing = readTimeout / 2
                                     - clientCnxnSocket.getIdleSend()
                                     - ((clientCnxnSocket.getIdleSend() > 1000) ? 1000 : 0);
                //send a ping request either time is due or no packet sent out within MAX_SEND_PING_INTERVAL
                if (timeToNextPing <= 0 || clientCnxnSocket.getIdleSend() > MAX_SEND_PING_INTERVAL) {
                    sendPing();
                    clientCnxnSocket.updateLastSend();
                } else {
                    if (timeToNextPing < to) {
                        to = timeToNextPing;
                    }
                }
            }

            // If we are in read-only mode, seek for read/write server
            if (state == States.CONNECTEDREADONLY) {
                long now = Time.currentElapsedTime();
                int idlePingRwServer = (int) (now - lastPingRwServer);
                if (idlePingRwServer >= pingRwTimeout) {
                    lastPingRwServer = now;
                    idlePingRwServer = 0;
                    pingRwTimeout = Math.min(2 * pingRwTimeout, maxPingRwTimeout);
                    pingRwServer();
                }
                to = Math.min(to, pingRwTimeout - idlePingRwServer);
            }
            // 处理outgoingQueue，进行doWrite发给服务端
            clientCnxnSocket.doTransport(to, pendingQueue, ClientCnxn.this);
        } catch (Throwable e) {
            if (closing) {
                // closing so this is expected
                LOG.warn(
                    "An exception was thrown while closing send thread for session 0x{}.",
                    Long.toHexString(getSessionId()),
                    e);
                break;
            } else {
                LOG.warn(
                    "Session 0x{} for server {}, Closing socket connection. "
                        + "Attempting reconnect except it is a SessionExpiredException.",
                    Long.toHexString(getSessionId()),
                    serverAddress,
                    e);

                // At this point, there might still be new packets appended to outgoingQueue.
                // they will be handled in next connection or cleared up if closed.
                cleanAndNotifyState();
            }
        }
    }

    synchronized (state) {
        // When it comes to this point, it guarantees that later queued
        // packet to outgoingQueue will be notified of death.
        cleanup();
    }
    clientCnxnSocket.close();
    if (state.isAlive()) {
        eventThread.queueEvent(new WatchedEvent(Event.EventType.None, Event.KeeperState.Disconnected, null));
    }
    eventThread.queueEvent(new WatchedEvent(Event.EventType.None, Event.KeeperState.Closed, null));

    if (zooKeeperSaslClient != null) {
        zooKeeperSaslClient.shutdown();
    }
    ZooTrace.logTraceMessage(
        LOG,
        ZooTrace.getTextTraceLevel(),
        "SendThread exited loop for session: 0x" + Long.toHexString(getSessionId()));
}
```
#### 4.1.1org.apache.zookeeper.ClientCnxn.SendThread#startConnect
```java
private void startConnect(InetSocketAddress addr) throws IOException {
    // initializing it for new connection
    saslLoginFailed = false;
    if (!isFirstConnect) {
        try {
            Thread.sleep(ThreadLocalRandom.current().nextLong(1000));
        } catch (InterruptedException e) {
            LOG.warn("Unexpected exception", e);
        }
    }
    changeZkState(States.CONNECTING);

    String hostPort = addr.getHostString() + ":" + addr.getPort();
    MDC.put("myid", hostPort);
    setName(getName().replaceAll("\\(.*\\)", "(" + hostPort + ")"));
    // 安全校验
    if (clientConfig.isSaslClientEnabled()) {
        try {
            if (zooKeeperSaslClient != null) {
                zooKeeperSaslClient.shutdown();
            }
            zooKeeperSaslClient = new ZooKeeperSaslClient(SaslServerPrincipal.getServerPrincipal(addr, clientConfig), clientConfig);
        } catch (LoginException e) {
            // An authentication error occurred when the SASL client tried to initialize:
            // for Kerberos this means that the client failed to authenticate with the KDC.
            // This is different from an authentication error that occurs during communication
            // with the Zookeeper server, which is handled below.
            LOG.warn(
                "SASL configuration failed. "
                    + "Will continue connection to Zookeeper server without "
                    + "SASL authentication, if Zookeeper server allows it.", e);
            eventThread.queueEvent(new WatchedEvent(Watcher.Event.EventType.None, Watcher.Event.KeeperState.AuthFailed, null));
            saslLoginFailed = true;
        }
    }
    logStartConnect(addr);
    // 建立NIO/netty连接的标准写法
    clientCnxnSocket.connect(addr);
}
```
#### 4.1.2org.apache.zookeeper.ClientCnxnSocketNetty#doTransport
```java
void doTransport(
    int waitTimeOut,
    Queue<Packet> pendingQueue,
    ClientCnxn cnxn) throws IOException, InterruptedException {
    try {
        if (!firstConnect.await(waitTimeOut, TimeUnit.MILLISECONDS)) {
            return;
        }
        Packet head = null;
        if (needSasl.get()) {
            if (!waitSasl.tryAcquire(waitTimeOut, TimeUnit.MILLISECONDS)) {
                return;
            }
        } else {
            head = outgoingQueue.poll(waitTimeOut, TimeUnit.MILLISECONDS);
        }
        // check if being waken up on closing.
        if (!sendThread.getZkState().isAlive()) {
            // adding back the packet to notify of failure in conLossPacket().
            addBack(head);
            return;
        }
        // channel disconnection happened
        if (disconnected.get()) {
            addBack(head);
            throw new EndOfStreamException("channel for sessionid 0x" + Long.toHexString(sessionId) + " is lost");
        }
        if (head != null) {
            doWrite(pendingQueue, head, cnxn);
        }
    } finally {
        updateNow();
    }
}
```
### 4.2org.apache.zookeeper.ClientCnxn.EventThread#run
```java
public void run() {
    try {
        isRunning = true;
        while (true) {
            Object event = waitingEvents.take();
            if (event == eventOfDeath) {
                wasKilled = true;
            } else {
                // 回调自己写的watcher回调
                processEvent(event);
            }
            if (wasKilled) {
                synchronized (waitingEvents) {
                    if (waitingEvents.isEmpty()) {
                        isRunning = false;
                        break;
                    }
                }
            }
        }
    } catch (InterruptedException e) {
        LOG.error("Event thread exiting due to interruption", e);
    }

    LOG.info("EventThread shut down for session: 0x{}", Long.toHexString(getSessionId()));
}
```

### 5.org.apache.zookeeper.ZooKeeper#create(java.lang.String, byte[], java.util.List<org.apache.zookeeper.data.ACL>, org.apache.zookeeper.CreateMode)
```java
public String create(
    final String path,
    byte[] data,
    List<ACL> acl,
    CreateMode createMode) throws KeeperException, InterruptedException {
    final String clientPath = path;
    PathUtils.validatePath(clientPath, createMode.isSequential());
    EphemeralType.validateTTL(createMode, -1);
    validateACL(acl);

    final String serverPath = prependChroot(clientPath);

    RequestHeader h = new RequestHeader();
    h.setType(createMode.isContainer() ? ZooDefs.OpCode.createContainer : ZooDefs.OpCode.create);
    CreateRequest request = new CreateRequest();
    CreateResponse response = new CreateResponse();
    request.setData(data);
    request.setFlags(createMode.toFlag());
    request.setPath(serverPath);
    request.setAcl(acl);
    // 存入outgongqueue,在packetAdd中执行selector.wakeup
    ReplyHeader r = cnxn.submitRequest(h, request, response, null);
    if (r.getErr() != 0) {
        throw KeeperException.create(KeeperException.Code.get(r.getErr()), clientPath);
    }
    if (cnxn.chrootPath == null) {
        return response.getPath();
    } else {
        return response.getPath().substring(cnxn.chrootPath.length());
    }
}
```

## 6.org.apache.zookeeper.server.ServerCnxnFactory#createFactory()|服务端启动
```java
public static ServerCnxnFactory createFactory() throws IOException {
    // 创建NettyServerCnxnFactory或者NIO
    String serverCnxnFactoryName = System.getProperty(ZOOKEEPER_SERVER_CNXN_FACTORY);
    if (serverCnxnFactoryName == null) {
        serverCnxnFactoryName = NIOServerCnxnFactory.class.getName();
    }
    try {
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
### 6.1org.apache.zookeeper.server.NettyServerCnxnFactory.CnxnChannelHandler#channelRead|服务端读取消息
```java
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    try {
        if (LOG.isTraceEnabled()) {
            LOG.trace("message received called {}", msg);
        }
        try {
            LOG.debug("New message {} from {}", msg, ctx.channel());
            NettyServerCnxn cnxn = ctx.channel().attr(CONNECTION_ATTRIBUTE).get();
            if (cnxn == null) {
                LOG.error("channelRead() on a closed or closing NettyServerCnxn");
            } else {
                cnxn.processMessage((ByteBuf) msg);
            }
        } catch (Exception ex) {
            LOG.error("Unexpected exception in receive", ex);
            throw ex;
        }
    } finally {
        ReferenceCountUtil.release(msg);
    }
}
```
- org.apache.zookeeper.server.NettyServerCnxn#processMessage
```java
void processMessage(ByteBuf buf) {
    checkIsInEventLoop("processMessage");
    LOG.debug("0x{} queuedBuffer: {}", Long.toHexString(sessionId), queuedBuffer);

    if (LOG.isTraceEnabled()) {
        LOG.trace("0x{} buf {}", Long.toHexString(sessionId), ByteBufUtil.hexDump(buf));
    }

    if (throttled.get()) {
        LOG.debug("Received message while throttled");
        // we are throttled, so we need to queue
        if (queuedBuffer == null) {
            LOG.debug("allocating queue");
            queuedBuffer = channel.alloc().compositeBuffer();
        }
        appendToQueuedBuffer(buf.retainedDuplicate());
        if (LOG.isTraceEnabled()) {
            LOG.trace("0x{} queuedBuffer {}", Long.toHexString(sessionId), ByteBufUtil.hexDump(queuedBuffer));
        }
    } else {
        LOG.debug("not throttled");
        if (queuedBuffer != null) {
            appendToQueuedBuffer(buf.retainedDuplicate());
            processQueuedBuffer();
        } else {
            receiveMessage(buf);
            // Have to check !closingChannel, because an error in
            // receiveMessage() could have led to close() being called.
            if (!closingChannel && buf.isReadable()) {
                if (LOG.isTraceEnabled()) {
                    LOG.trace("Before copy {}", buf);
                }

                if (queuedBuffer == null) {
                    queuedBuffer = channel.alloc().compositeBuffer();
                }
                appendToQueuedBuffer(buf.retainedSlice(buf.readerIndex(), buf.readableBytes()));
                if (LOG.isTraceEnabled()) {
                    LOG.trace("Copy is {}", queuedBuffer);
                    LOG.trace("0x{} queuedBuffer {}", Long.toHexString(sessionId), ByteBufUtil.hexDump(queuedBuffer));
                }
            }
        }
    }
}
```
#### 6.1.1org.apache.zookeeper.server.NettyServerCnxn#receiveMessage
```java
private void receiveMessage(ByteBuf message) {
    checkIsInEventLoop("receiveMessage");
    try {
        while (message.isReadable() && !throttled.get()) {
            if (bb != null) {
                if (LOG.isTraceEnabled()) {
                    LOG.trace("message readable {} bb len {} {}", message.readableBytes(), bb.remaining(), bb);
                    ByteBuffer dat = bb.duplicate();
                    dat.flip();
                    LOG.trace("0x{} bb {}", Long.toHexString(sessionId), ByteBufUtil.hexDump(Unpooled.wrappedBuffer(dat)));
                }

                if (bb.remaining() > message.readableBytes()) {
                    int newLimit = bb.position() + message.readableBytes();
                    bb.limit(newLimit);
                }
                message.readBytes(bb);
                bb.limit(bb.capacity());

                if (LOG.isTraceEnabled()) {
                    LOG.trace("after readBytes message readable {} bb len {} {}", message.readableBytes(), bb.remaining(), bb);
                    ByteBuffer dat = bb.duplicate();
                    dat.flip();
                    LOG.trace("after readbytes 0x{} bb {}",
                              Long.toHexString(sessionId),
                              ByteBufUtil.hexDump(Unpooled.wrappedBuffer(dat)));
                }
                if (bb.remaining() == 0) {
                    bb.flip();
                    packetReceived(4 + bb.remaining());

                    ZooKeeperServer zks = this.zkServer;
                    if (zks == null || !zks.isRunning()) {
                        throw new IOException("ZK down");
                    }
                    if (initialized) {
                        // TODO: if zks.processPacket() is changed to take a ByteBuffer[],
                        // we could implement zero-copy queueing.
                        zks.processPacket(this, bb);
                    } else {
                        LOG.debug("got conn req request from {}", getRemoteSocketAddress());
                        zks.processConnectRequest(this, bb);
                        initialized = true;
                    }
                    bb = null;
                }
            } else {
                if (LOG.isTraceEnabled()) {
                    LOG.trace("message readable {} bblenrem {}", message.readableBytes(), bbLen.remaining());
                    ByteBuffer dat = bbLen.duplicate();
                    dat.flip();
                    LOG.trace("0x{} bbLen {}", Long.toHexString(sessionId), ByteBufUtil.hexDump(Unpooled.wrappedBuffer(dat)));
                }

                if (message.readableBytes() < bbLen.remaining()) {
                    bbLen.limit(bbLen.position() + message.readableBytes());
                }
                message.readBytes(bbLen);
                bbLen.limit(bbLen.capacity());
                if (bbLen.remaining() == 0) {
                    bbLen.flip();

                    if (LOG.isTraceEnabled()) {
                        LOG.trace("0x{} bbLen {}", Long.toHexString(sessionId), ByteBufUtil.hexDump(Unpooled.wrappedBuffer(bbLen)));
                    }
                    int len = bbLen.getInt();
                    if (LOG.isTraceEnabled()) {
                        LOG.trace("0x{} bbLen len is {}", Long.toHexString(sessionId), len);
                    }

                    bbLen.clear();
                    if (!initialized) {
                        if (checkFourLetterWord(channel, message, len)) {
                            return;
                        }
                    }
                    if (len < 0 || len > BinaryInputArchive.maxBuffer) {
                        throw new IOException("Len error " + len);
                    }
                    ZooKeeperServer zks = this.zkServer;
                    if (zks == null || !zks.isRunning()) {
                        LOG.info("Closing connection to {} because the server is not ready",
                                getRemoteSocketAddress());
                        close(DisconnectReason.IO_EXCEPTION);
                        return;
                    }
                    // checkRequestSize will throw IOException if request is rejected
                    zks.checkRequestSizeWhenReceivingMessage(len);
                    bb = ByteBuffer.allocate(len);
                }
            }
        }
    } catch (IOException e) {
        LOG.warn("Closing connection to {}", getRemoteSocketAddress(), e);
        close(DisconnectReason.IO_EXCEPTION);
    } catch (ClientCnxnLimitException e) {
        // Common case exception, print at debug level
        ServerMetrics.getMetrics().CONNECTION_REJECTED.add(1);

        LOG.debug("Closing connection to {}", getRemoteSocketAddress(), e);
        close(DisconnectReason.CLIENT_RATE_LIMIT);
    }
}
```
- org.apache.zookeeper.server.ZooKeeperServer#processPacket
```java
public void processPacket(ServerCnxn cnxn, ByteBuffer incomingBuffer) throws IOException {
    // We have the request, now process and setup for next
    InputStream bais = new ByteBufferInputStream(incomingBuffer);
    BinaryInputArchive bia = BinaryInputArchive.getArchive(bais);
    RequestHeader h = new RequestHeader();
    // 反序列化
    h.deserialize(bia, "header");

    // Need to increase the outstanding request count first, otherwise
    // there might be a race condition that it enabled recv after
    // processing request and then disabled when check throttling.
    //
    // Be aware that we're actually checking the global outstanding
    // request before this request.
    //
    // It's fine if the IOException thrown before we decrease the count
    // in cnxn, since it will close the cnxn anyway.
    cnxn.incrOutstandingAndCheckThrottle(h);

    // Through the magic of byte buffers, txn will not be
    // pointing
    // to the start of the txn
    incomingBuffer = incomingBuffer.slice();
    if (h.getType() == OpCode.auth) {
        LOG.info("got auth packet {}", cnxn.getRemoteSocketAddress());
        AuthPacket authPacket = new AuthPacket();
        ByteBufferInputStream.byteBuffer2Record(incomingBuffer, authPacket);
        String scheme = authPacket.getScheme();
        ServerAuthenticationProvider ap = ProviderRegistry.getServerProvider(scheme);
        Code authReturn = KeeperException.Code.AUTHFAILED;
        if (ap != null) {
            try {
                // handleAuthentication may close the connection, to allow the client to choose
                // a different server to connect to.
                authReturn = ap.handleAuthentication(
                    new ServerAuthenticationProvider.ServerObjs(this, cnxn),
                    authPacket.getAuth());
            } catch (RuntimeException e) {
                LOG.warn("Caught runtime exception from AuthenticationProvider: {}", scheme, e);
                authReturn = KeeperException.Code.AUTHFAILED;
            }
        }
        if (authReturn == KeeperException.Code.OK) {
            LOG.info("Session 0x{}: auth success for scheme {} and address {}",
                    Long.toHexString(cnxn.getSessionId()), scheme,
                    cnxn.getRemoteSocketAddress());
            ReplyHeader rh = new ReplyHeader(h.getXid(), 0, KeeperException.Code.OK.intValue());
            cnxn.sendResponse(rh, null, null);
        } else {
            if (ap == null) {
                LOG.warn(
                    "No authentication provider for scheme: {} has {}",
                    scheme,
                    ProviderRegistry.listProviders());
            } else {
                LOG.warn("Authentication failed for scheme: {}", scheme);
            }
            // send a response...
            ReplyHeader rh = new ReplyHeader(h.getXid(), 0, KeeperException.Code.AUTHFAILED.intValue());
            cnxn.sendResponse(rh, null, null);
            // ... and close connection
            cnxn.sendBuffer(ServerCnxnFactory.closeConn);
            cnxn.disableRecv();
        }
        return;
    } else if (h.getType() == OpCode.sasl) {
        processSasl(incomingBuffer, cnxn, h);
    } else {
        if (!authHelper.enforceAuthentication(cnxn, h.getXid())) {
            // Authentication enforcement is failed
            // Already sent response to user about failure and closed the session, lets return
            return;
        } else {
            Request si = new Request(cnxn, cnxn.getSessionId(), h.getXid(), h.getType(), incomingBuffer, cnxn.getAuthInfo());
            int length = incomingBuffer.limit();
            if (isLargeRequest(length)) {
                // checkRequestSize will throw IOException if request is rejected
                checkRequestSizeWhenMessageReceived(length);
                si.setLargeRequestSize(length);
            }
            si.setOwner(ServerCnxn.me);
            // 存入submittedRequests队列
            submitRequest(si);
        }
    }
}
```
## 7.setFailCreate|处理submittedRequests
```java
public static void setFailCreate(boolean b) {
        failCreate = b;
}
@Override
public void run() {
    LOG.info(String.format("PrepRequestProcessor (sid:%d) started, reconfigEnabled=%s", zks.getServerId(), zks.reconfigEnabled));
    try {
        while (true) {
            ServerMetrics.getMetrics().PREP_PROCESSOR_QUEUE_SIZE.add(submittedRequests.size());
            Request request = submittedRequests.take();
            ServerMetrics.getMetrics().PREP_PROCESSOR_QUEUE_TIME
                .add(Time.currentElapsedTime() - request.prepQueueStartTime);
            if (LOG.isTraceEnabled()) {
                long traceMask = ZooTrace.CLIENT_REQUEST_TRACE_MASK;
                if (request.type == OpCode.ping) {
                    traceMask = ZooTrace.CLIENT_PING_TRACE_MASK;
                }
                ZooTrace.logRequest(LOG, traceMask, 'P', request, "");
            }
            if (Request.requestOfDeath == request) {
                break;
            }

            request.prepStartTime = Time.currentElapsedTime();
            pRequest(request);
        }
    } catch (Exception e) {
        handleException(this.getName(), e);
    }
    LOG.info("PrepRequestProcessor exited loop!");
}
```
- org.apache.zookeeper.server.PrepRequestProcessor#pRequest
```java
protected void pRequest(Request request) throws RequestProcessorException {
    // LOG.info("Prep>>> cxid = " + request.cxid + " type = " +
    // request.type + " id = 0x" + Long.toHexString(request.sessionId));
    request.setHdr(null);
    request.setTxn(null);

    if (!request.isThrottled()) {
        // 核心处理
      pRequestHelper(request);
    }

    request.zxid = zks.getZxid();
    long timeFinishedPrepare = Time.currentElapsedTime();
    ServerMetrics.getMetrics().PREP_PROCESS_TIME.add(timeFinishedPrepare - request.prepStartTime);
    // TODO 链式处理,里面会给其它follower发送数据包,以及实例化本地数据
    nextProcessor.processRequest(request);
    ServerMetrics.getMetrics().PROPOSAL_PROCESS_TIME.add(Time.currentElapsedTime() - timeFinishedPrepare);
}
```
- org.apache.zookeeper.server.PrepRequestProcessor#pRequestHelper
```java
private void pRequestHelper(Request request) throws RequestProcessorException {
    try {
        switch (request.type) {
        case OpCode.createContainer:
        case OpCode.create:
        case OpCode.create2:
            CreateRequest create2Request = new CreateRequest();
            /**
             *     private final AtomicLong hzxid = new AtomicLong(0);
             * // 此处是低32位的事务id
             *long getNextZxid() {
                return hzxid.incrementAndGet();
            } 
             */
            pRequest2Txn(request.type, zks.getNextZxid(), request, create2Request, true);
            break;
        case OpCode.createTTL:
            CreateTTLRequest createTtlRequest = new CreateTTLRequest();
            pRequest2Txn(request.type, zks.getNextZxid(), request, createTtlRequest, true);
            break;
        case OpCode.deleteContainer:
        case OpCode.delete:
            DeleteRequest deleteRequest = new DeleteRequest();
            pRequest2Txn(request.type, zks.getNextZxid(), request, deleteRequest, true);
            break;
        case OpCode.setData:
            SetDataRequest setDataRequest = new SetDataRequest();
            pRequest2Txn(request.type, zks.getNextZxid(), request, setDataRequest, true);
            break;
        case OpCode.reconfig:
            ReconfigRequest reconfigRequest = new ReconfigRequest();
            ByteBufferInputStream.byteBuffer2Record(request.request, reconfigRequest);
            pRequest2Txn(request.type, zks.getNextZxid(), request, reconfigRequest, true);
            break;
        case OpCode.setACL:
            SetACLRequest setAclRequest = new SetACLRequest();
            pRequest2Txn(request.type, zks.getNextZxid(), request, setAclRequest, true);
            break;
        case OpCode.check:
            CheckVersionRequest checkRequest = new CheckVersionRequest();
            pRequest2Txn(request.type, zks.getNextZxid(), request, checkRequest, true);
            break;
        case OpCode.multi:
            MultiOperationRecord multiRequest = new MultiOperationRecord();
            try {
                ByteBufferInputStream.byteBuffer2Record(request.request, multiRequest);
            } catch (IOException e) {
                request.setHdr(new TxnHeader(request.sessionId, request.cxid, zks.getNextZxid(), Time.currentWallTime(), OpCode.multi));
                throw e;
            }
            List<Txn> txns = new ArrayList<Txn>();
            //Each op in a multi-op must have the same zxid!
            long zxid = zks.getNextZxid();
            KeeperException ke = null;

            //Store off current pending change records in case we need to rollback
            Map<String, ChangeRecord> pendingChanges = getPendingChanges(multiRequest);
            request.setHdr(new TxnHeader(request.sessionId, request.cxid, zxid,
                    Time.currentWallTime(), request.type));

            for (Op op : multiRequest) {
                Record subrequest = op.toRequestRecord();
                int type;
                Record txn;

                /* If we've already failed one of the ops, don't bother
                 * trying the rest as we know it's going to fail and it
                 * would be confusing in the logfiles.
                 */
                if (ke != null) {
                    type = OpCode.error;
                    txn = new ErrorTxn(Code.RUNTIMEINCONSISTENCY.intValue());
                } else {
                    /* Prep the request and convert to a Txn */
                    try {
                        pRequest2Txn(op.getType(), zxid, request, subrequest, false);
                        type = op.getType();
                        txn = request.getTxn();
                    } catch (KeeperException e) {
                        ke = e;
                        type = OpCode.error;
                        txn = new ErrorTxn(e.code().intValue());

                        if (e.code().intValue() > Code.APIERROR.intValue()) {
                            LOG.info("Got user-level KeeperException when processing {} aborting"
                                     + " remaining multi ops. Error Path:{} Error:{}",
                                     request.toString(),
                                     e.getPath(),
                                     e.getMessage());
                        }

                        request.setException(e);

                        /* Rollback change records from failed multi-op */
                        rollbackPendingChanges(zxid, pendingChanges);
                    }
                }

                // TODO: I don't want to have to serialize it here and then
                //       immediately deserialize in next processor. But I'm
                //       not sure how else to get the txn stored into our list.
                try (ByteArrayOutputStream baos = new ByteArrayOutputStream()) {
                    BinaryOutputArchive boa = BinaryOutputArchive.getArchive(baos);
                    txn.serialize(boa, "request");
                    ByteBuffer bb = ByteBuffer.wrap(baos.toByteArray());
                    txns.add(new Txn(type, bb.array()));
                }
            }

            request.setTxn(new MultiTxn(txns));
            if (digestEnabled) {
                setTxnDigest(request);
            }

            break;

        //create/close session don't require request record
        case OpCode.createSession:
        case OpCode.closeSession:
            if (!request.isLocalSession()) {
                pRequest2Txn(request.type, zks.getNextZxid(), request, null, true);
            }
            break;

        //All the rest don't need to create a Txn - just verify session
        case OpCode.sync:
        case OpCode.exists:
        case OpCode.getData:
        case OpCode.getACL:
        case OpCode.getChildren:
        case OpCode.getAllChildrenNumber:
        case OpCode.getChildren2:
        case OpCode.ping:
        case OpCode.setWatches:
        case OpCode.setWatches2:
        case OpCode.checkWatches:
        case OpCode.removeWatches:
        case OpCode.getEphemerals:
        case OpCode.multiRead:
        case OpCode.addWatch:
        case OpCode.whoAmI:
            zks.sessionTracker.checkSession(request.sessionId, request.getOwner());
            break;
        default:
            LOG.warn("unknown type {}", request.type);
            break;
        }
    } catch (KeeperException e) {
        if (request.getHdr() != null) {
            request.getHdr().setType(OpCode.error);
            request.setTxn(new ErrorTxn(e.code().intValue()));
        }

        if (e.code().intValue() > Code.APIERROR.intValue()) {
            LOG.info(
                "Got user-level KeeperException when processing {} Error Path:{} Error:{}",
                request.toString(),
                e.getPath(),
                e.getMessage());
        }
        request.setException(e);
    } catch (Exception e) {
        // log at error level as we are returning a marshalling
        // error to the user
        LOG.error("Failed to process {}", request, e);

        StringBuilder sb = new StringBuilder();
        ByteBuffer bb = request.request;
        if (bb != null) {
            bb.rewind();
            while (bb.hasRemaining()) {
                sb.append(String.format("%02x", (0xff & bb.get())));
            }
        } else {
            sb.append("request buffer is null");
        }

        LOG.error("Dumping request buffer for request type {}: 0x{}", Request.op2String(request.type), sb);
        if (request.getHdr() != null) {
            request.getHdr().setType(OpCode.error);
            request.setTxn(new ErrorTxn(Code.MARSHALLINGERROR.intValue()));
        }
    }
}
```
## 8.org.apache.zookeeper.server.SyncRequestProcessor#processRequest|同步数据
```java
public void processRequest(final Request request) {
    Objects.requireNonNull(request, "Request cannot be null");

    request.syncQueueStartTime = Time.currentElapsedTime();
    queuedRequests.add(request);
    ServerMetrics.getMetrics().SYNC_PROCESSOR_QUEUED.add(1);
}
```
- org.apache.zookeeper.server.SyncRequestProcessor#run
```java
public void run() {
    try {
        // we do this in an attempt to ensure that not all of the servers
        // in the ensemble take a snapshot at the same time
        resetSnapshotStats();
        lastFlushTime = Time.currentElapsedTime();
        while (true) {
            ServerMetrics.getMetrics().SYNC_PROCESSOR_QUEUE_SIZE.add(queuedRequests.size());

            long pollTime = Math.min(zks.getMaxWriteQueuePollTime(), getRemainingDelay());
            Request si = queuedRequests.poll(pollTime, TimeUnit.MILLISECONDS);
            if (si == null) {
                /* We timed out looking for more writes to batch, go ahead and flush immediately */
                flush();
                si = queuedRequests.take();
            }

            if (si == REQUEST_OF_DEATH) {
                break;
            }

            long startProcessTime = Time.currentElapsedTime();
            ServerMetrics.getMetrics().SYNC_PROCESSOR_QUEUE_TIME.add(startProcessTime - si.syncQueueStartTime);

            // track the number of records written to the log
            if (!si.isThrottled() && zks.getZKDatabase().append(si)) {
                if (shouldSnapshot()) {
                    resetSnapshotStats();
                    // roll the log
                    zks.getZKDatabase().rollLog();
                    // take a snapshot
                    if (!snapThreadMutex.tryAcquire()) {
                        LOG.warn("Too busy to snap, skipping");
                    } else {
                        new ZooKeeperThread("Snapshot Thread") {
                            public void run() {
                                try {
                                    zks.takeSnapshot();
                                } catch (Exception e) {
                                    LOG.warn("Unexpected exception", e);
                                } finally {
                                    snapThreadMutex.release();
                                }
                            }
                        }.start();
                    }
                }
            } else if (toFlush.isEmpty()) {
                // optimization for read heavy workloads
                // iff this is a read or a throttled request(which doesn't need to be written to the disk),
                // and there are no pending flushes (writes), then just pass this to the next processor
                if (nextProcessor != null) {
                    nextProcessor.processRequest(si);
                    if (nextProcessor instanceof Flushable) {
                        ((Flushable) nextProcessor).flush();
                    }
                }
                continue;
            }
            toFlush.add(si);
            if (shouldFlush()) {
                flush();
            }
            ServerMetrics.getMetrics().SYNC_PROCESS_TIME.add(Time.currentElapsedTime() - startProcessTime);
        }
    } catch (Throwable t) {
        handleException(this.getName(), t);
    }
    LOG.info("SyncRequestProcessor exited!");
}
```
## 9.org.apache.zookeeper.server.quorum.AckRequestProcessor#processRequest|接收flower返回ack
```java
public void processRequest(Request request) {
    QuorumPeer self = leader.self;
    if (self != null) {
        request.logLatency(ServerMetrics.getMetrics().PROPOSAL_ACK_CREATION_LATENCY);
        leader.processAck(self.getId(), request.zxid, null);
    } else {
        LOG.error("Null QuorumPeer");
    }
}
```
- org.apache.zookeeper.server.quorum.Leader#processAck
```java
public synchronized void processAck(long sid, long zxid, SocketAddress followerAddr) {
    if (!allowedToCommit) {
        return; // last op committed was a leader change - from now on
    }
    // the new leader should commit
    if (LOG.isTraceEnabled()) {
        LOG.trace("Ack zxid: 0x{}", Long.toHexString(zxid));
        for (Proposal p : outstandingProposals.values()) {
            long packetZxid = p.packet.getZxid();
            LOG.trace("outstanding proposal: 0x{}", Long.toHexString(packetZxid));
        }
        LOG.trace("outstanding proposals all");
    }

    if ((zxid & 0xffffffffL) == 0) {
        /*
         * We no longer process NEWLEADER ack with this method. However,
         * the learner sends an ack back to the leader after it gets
         * UPTODATE, so we just ignore the message.
         */
        return;
    }

    if (outstandingProposals.size() == 0) {
        LOG.debug("outstanding is 0");
        return;
    }
    if (lastCommitted >= zxid) {
        LOG.debug(
            "proposal has already been committed, pzxid: 0x{} zxid: 0x{}",
            Long.toHexString(lastCommitted),
            Long.toHexString(zxid));
        // The proposal has already been committed
        return;
    }
    Proposal p = outstandingProposals.get(zxid);
    if (p == null) {
        LOG.warn("Trying to commit future proposal: zxid 0x{} from {}", Long.toHexString(zxid), followerAddr);
        return;
    }

    if (ackLoggingFrequency > 0 && (zxid % ackLoggingFrequency == 0)) {
        p.request.logLatency(ServerMetrics.getMetrics().ACK_LATENCY, Long.toString(sid));
    }

    p.addAck(sid);
    // 判断是否可以提交(半数以上),在能够提交后(一阶段结束)会再发送commit包给每个fowller，最后发给observe
    boolean hasCommitted = tryToCommit(p, zxid, followerAddr);

    // If p is a reconfiguration, multiple other operations may be ready to be committed,
    // since operations wait for different sets of acks.
    // Currently we only permit one outstanding reconfiguration at a time
    // such that the reconfiguration and subsequent outstanding ops proposed while the reconfig is
    // pending all wait for a quorum of old and new config, so its not possible to get enough acks
    // for an operation without getting enough acks for preceding ops. But in the future if multiple
    // concurrent reconfigs are allowed, this can happen and then we need to check whether some pending
    // ops may already have enough acks and can be committed, which is what this code does.

    if (hasCommitted && p.request != null && p.request.getHdr().getType() == OpCode.reconfig) {
        long curZxid = zxid;
        while (allowedToCommit && hasCommitted && p != null) {
            curZxid++;
            p = outstandingProposals.get(curZxid);
            if (p != null) {
                hasCommitted = tryToCommit(p, curZxid, null);
            }
        }
    }
}
```
### 9.1org.apache.zookeeper.server.quorum.Leader#tryToCommit
```java
public synchronized boolean tryToCommit(Proposal p, long zxid, SocketAddress followerAddr) {
    // make sure that ops are committed in order. With reconfigurations it is now possible
    // that different operations wait for different sets of acks, and we still want to enforce
    // that they are committed in order. Currently we only permit one outstanding reconfiguration
    // such that the reconfiguration and subsequent outstanding ops proposed while the reconfig is
    // pending all wait for a quorum of old and new config, so it's not possible to get enough acks
    // for an operation without getting enough acks for preceding ops. But in the future if multiple
    // concurrent reconfigs are allowed, this can happen.
    if (outstandingProposals.containsKey(zxid - 1)) {
        return false;
    }

    // in order to be committed, a proposal must be accepted by a quorum.
    //
    // getting a quorum from all necessary configurations.
    // ack是否大于一半
    if (!p.hasAllQuorums()) {
        return false;
    }

    // commit proposals in order
    if (zxid != lastCommitted + 1) {
        LOG.warn(
            "Commiting zxid 0x{} from {} not first!",
            Long.toHexString(zxid),
            followerAddr);
        LOG.warn("First is 0x{}", Long.toHexString(lastCommitted + 1));
    }

    outstandingProposals.remove(zxid);

    if (p.request != null) {
        toBeApplied.add(p);
    }

    if (p.request == null) {
        LOG.warn("Going to commit null: {}", p);
    } else if (p.request.getHdr().getType() == OpCode.reconfig) {
        LOG.debug("Committing a reconfiguration! {}", outstandingProposals.size());

        //if this server is voter in new config with the same quorum address,
        //then it will remain the leader
        //otherwise an up-to-date follower will be designated as leader. This saves
        //leader election time, unless the designated leader fails
        Long designatedLeader = getDesignatedLeader(p, zxid);

        QuorumVerifier newQV = p.qvAcksetPairs.get(p.qvAcksetPairs.size() - 1).getQuorumVerifier();

        self.processReconfig(newQV, designatedLeader, zk.getZxid(), true);

        if (designatedLeader != self.getId()) {
            LOG.info(String.format("Committing a reconfiguration (reconfigEnabled=%s); this leader is not the designated "
                    + "leader anymore, setting allowedToCommit=false", self.isReconfigEnabled()));
            allowedToCommit = false;
        }

        // we're sending the designated leader, and if the leader is changing the followers are
        // responsible for closing the connection - this way we are sure that at least a majority of them
        // receive the commit message.
        // 提交给其它的follower
        commitAndActivate(zxid, designatedLeader);
        // 提交给observe
        informAndActivate(p, designatedLeader);
    } else {
        p.request.logLatency(ServerMetrics.getMetrics().QUORUM_ACK_LATENCY);
        commit(zxid);
        inform(p);
    }
    // learder提交给内存数据
    zk.commitProcessor.commit(p.request);
    if (pendingSyncs.containsKey(zxid)) {
        for (LearnerSyncRequest r : pendingSyncs.remove(zxid)) {
            sendSync(r);
        }
    }

    return true;
}
```

### 10.org.apache.zookeeper.server.quorum.Follower#processPacket-Leader.PROPOSAL|follower接收数据包
```java
case Leader.PROPOSAL:
    ServerMetrics.getMetrics().LEARNER_PROPOSAL_RECEIVED_COUNT.add(1);
    TxnLogEntry logEntry = SerializeUtils.deserializeTxn(qp.getData());
    TxnHeader hdr = logEntry.getHeader();
    Record txn = logEntry.getTxn();
    TxnDigest digest = logEntry.getDigest();
    if (hdr.getZxid() != lastQueued + 1) {
        LOG.warn(
            "Got zxid 0x{} expected 0x{}",
            Long.toHexString(hdr.getZxid()),
            Long.toHexString(lastQueued + 1));
    }
    lastQueued = hdr.getZxid();

    if (hdr.getType() == OpCode.reconfig) {
        SetDataTxn setDataTxn = (SetDataTxn) txn;
        QuorumVerifier qv = self.configFromString(new String(setDataTxn.getData(), UTF_8));
        self.setLastSeenQuorumVerifier(qv, true);
    }
    // 写文件
    fzk.logRequest(hdr, txn, digest);
    if (hdr != null) {
        /*
         * Request header is created only by the leader, so this is only set
         * for quorum packets. If there is a clock drift, the latency may be
         * negative. Headers use wall time, not CLOCK_MONOTONIC.
         */
        long now = Time.currentWallTime();
        long latency = now - hdr.getTime();
        if (latency >= 0) {
            ServerMetrics.getMetrics().PROPOSAL_LATENCY.add(latency);
        }
    }
    if (om != null) {
        final long startTime = Time.currentElapsedTime();
        om.proposalReceived(qp);
        ServerMetrics.getMetrics().OM_PROPOSAL_PROCESS_TIME.add(Time.currentElapsedTime() - startTime);
    }
    break;
```