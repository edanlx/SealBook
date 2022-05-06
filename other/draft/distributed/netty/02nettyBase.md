Reactor近似与Selector
中间发展为主从Selector
主负责响应连接事件(线程池增加分发速度)(此处响应主要为注册读写事件，主要在于数量多因为并非每个都会发读写)，从负责响应读写事件(线程池增加分发速度)(此处响应为耗时间)
按照传统的模型，需要相应连接-响应读写-响应连接的循环，但是如果此时还在分发则会降低其他事件的响应速度
则bossGroup相当于主线Selector，workerGroup相当于从线程(一般情况一主多从就可以了)

注意bind()是异步的

- BootStrap、ServerBootStrap
启动引导类、服务端启动引导类
- ChannelFuture
异步监听回调
- Channel
Netty网络通信组件
1. NioSocketChannel TCP客户端
2. NioServerSocketChannel TCP服务端
3. NioDatagramChannel UDP连接
4. NioSctpChannel
5. NioServerSctpChannel
- NioEventLoopGroup
一组NioEventLoop，和线程池组类似
- NioEventLoop
内部维护一个线程和一个队列，执行相关IO任务，如accept、connect、read、write。由ProcessSelectedKeys触发
- ChannelHandler
处理/拦截IO操作,使用时需要继承ChannelInboundHandler(入站)、ChannelOutboundHandler(出站)、ChannelInboundHandlerAdapter、ChannelOutboundHandlerAdapter
1. channelActive有连接，可以将channel存入channelGroup
2. channelInactive表示有连接断开
3. read0表示有写事件，我方可以读取
- Channelpipeline
pipeline与tomcat的pipeline类似，起拦截器的功能，结构为双向链表，虽然在一个链表内，但是入站只会走入站的方法，出站只会走出站的方法
一般加入
1. 解码器StringDecoder、编码器StringEncoder(其实现ChannelOutboundHandler，用于处理汉字等)、
2. Handler(自己的实现逻辑,可以传输对象需要自己继承MessageToByteEncoder，服务端继承ByteToMessageDecoder)
```
// 客户端
out.writeInt(长度)
out.writeBytes(内容)
```
```
// 服务端
if(in.readableBytes()>=4){
    // 整形4个字节
    if(length == 0) {
        // 如果此时length是初始化的0则进入并读取到长度，同时先读掉4个字节
        length == in.readInt();
    }
    if(in.readableBytes() < length){
        // 可读长度不够
        renturn;
    }
    if(in.readableBytes()>=length){
        in.readBytes(接收容器，只读length这么长new Byte[length]);
        // 处理反序列化业务逻辑。。。
        out.add(XXX序列化完成的对象交给下一个handler)
    }
    length = 0
}
```
3. FixedLengthFrameDecoder(根据长度处理粘包、拆包,实现逻辑是先接收字符串再根据字符串接收相应长度，除了序列化以外的重要编解码器处理粘包拆包，但并不好用，自定义也可以实现同样的功能)
- ByteBuffer
与NIO的ByteBuffer类似但是操作体验完全不用，NIO的需要读写切换flip。
1. readIndex读索引
2. WriteIndex写索引
3. capacity容量
一开始读写索引都是0，当调用WriteByte时WriteIndex写移动，调用ReadByte时读移动，调用getBytes时不动。
由读写索引组成如下三个区域
1. 已读[0.readIndex)
2. 可读[readIndex,WriteIndex)
3. 未写[WriteIndex,capacity)

- 心跳
1. 在服务端增加 pipeline.addLast(new IdleStateHandler(3, 0, 0, TimeUnit.SECONDS));分别是读超时、写超时、读写超时，内部原理为嵌套线程定时任务检测，如示例则是每3秒检测一下该连接，如果有连通则要重新计时，即重启一个线程
2. 自己的handler需要实现userEventTriggered，如果没有及时收到响应，则IdleStateHandler会调用该方法。例如执行socket执行关闭
3. 客户端根据需求发送任意数据即可

- 重连
在客户端检测到服务端断开连接，则进行connect重连即可
1. channelInactive,客户端丢失
2. 启动时没通
- 零拷贝
socket缓冲-直接内存-jvm内存-直接内存-socket缓冲，完整的四次拷贝。零拷贝则为socket缓冲-直接内存-socket缓冲，共2次
- epoll空轮询
netty空轮询，使用rebulidSelector，事件转移

## 3.模型
1. Netty抽象出两组线程池BossGroup和WorkerGroup，BossGroup专门负责接收客户端的连接,WorkerGroup专门负责网络的读写
2. BossGroup和WorkerGroup类型都是NioEventLoopGroup
3. NioEventLoopGroup相当于一个事件循环线程组,这个组中含有多个事件循环线程，每一个事件循环线程是NioEventLoop
4. 每个NioEventLoop都有一个selector,用于监听注册在其上的socketChannel的网络通讯
5. 每个BossNioEventLoop线程内部循环执行的步骤有3步
	1. 处理accept事件,与client建立连接,生成NioSocketChannel
	2. 将NioSocketChannel注册到某个workerNIOEventLoop上的selector
	3. 处理任务队列的任务，即runAllTasks
6. 每个workerNIOEventLoop线程循环执行的步骤
	1. 轮询注册到自己selector上的所有NioSocketChannel的read,write事件
	2. 处理I/O事件，即read,write事件，在对应NioSocketChannel处理业务
	3. runAllTasks处理任务队列TaskQueue的任务，一些耗时的业务处理一般可以放入TaskQueue中慢慢处理，这样不影响数据在pipeline中的流动处理
7. 每个workerNIOEventLoop处理NioSocketChannel业务时，会使用pipeline(管道)，管道中维护了很多handler处理器用来处理channel中的数据

## 4.NioEventLoopGroup
```java
public NioEventLoopGroup(int nThreads) {
	// 线程池是null
    this(nThreads, (Executor) null);
}
```
```java
public NioEventLoopGroup(int nThreads, Executor executor) {
	// 调用epoll
    this(nThreads, executor, SelectorProvider.provider());
}
```
```java
 public NioEventLoopGroup(
        int nThreads, Executor executor, final SelectorProvider selectorProvider) {
    this(nThreads, executor, selectorProvider, DefaultSelectStrategyFactory.INSTANCE);
}
```
```java
public NioEventLoopGroup(int nThreads, Executor executor, final SelectorProvider selectorProvider,
                         final SelectStrategyFactory selectStrategyFactory) {
    super(nThreads, executor, selectorProvider, selectStrategyFactory, RejectedExecutionHandlers.reject());
}
```
```java
 protected MultithreadEventLoopGroup(int nThreads, Executor executor, Object... args) {
 	// thread如果不传就是2倍核
    super(nThreads == 0 ? DEFAULT_EVENT_LOOP_THREADS : nThreads, executor, args);
}
```
```java
protected MultithreadEventExecutorGroup(int nThreads, Executor executor, Object... args) {
    this(nThreads, executor, DefaultEventExecutorChooserFactory.INSTANCE, args);
}
```
```java
protected MultithreadEventExecutorGroup(int nThreads, Executor executor,
                                            EventExecutorChooserFactory chooserFactory, Object... args) {
    if (nThreads <= 0) {
        throw new IllegalArgumentException(String.format("nThreads: %d (expected: > 0)", nThreads));
    }

    if (executor == null) {
    	// 初始化线程池
        executor = new ThreadPerTaskExecutor(newDefaultThreadFactory());
    }
    // 赋值给children
    children = new EventExecutor[nThreads];

    for (int i = 0; i < nThreads; i ++) {
        boolean success = false;
        try {
        	// 循环给每个children赋值child，即NioEventLoop
            children[i] = newChild(executor, args);
            success = true;
        } catch (Exception e) {
            // TODO: Think about if this is a good exception type
            throw new IllegalStateException("failed to create a child event loop", e);
        } finally {
            if (!success) {
                for (int j = 0; j < i; j ++) {
                    children[j].shutdownGracefully();
                }

                for (int j = 0; j < i; j ++) {
                    EventExecutor e = children[j];
                    try {
                        while (!e.isTerminated()) {
                            e.awaitTermination(Integer.MAX_VALUE, TimeUnit.SECONDS);
                        }
                    } catch (InterruptedException interrupted) {
                        // Let the caller handle the interruption.
                        Thread.currentThread().interrupt();
                        break;
                    }
                }
            }
        }
    }

    chooser = chooserFactory.newChooser(children);

    final FutureListener<Object> terminationListener = new FutureListener<Object>() {
        @Override
        public void operationComplete(Future<Object> future) throws Exception {
            if (terminatedChildren.incrementAndGet() == children.length) {
                terminationFuture.setSuccess(null);
            }
        }
    };

    for (EventExecutor e: children) {
        e.terminationFuture().addListener(terminationListener);
    }

    Set<EventExecutor> childrenSet = new LinkedHashSet<EventExecutor>(children.length);
    Collections.addAll(childrenSet, children);
    readonlyChildren = Collections.unmodifiableSet(childrenSet);
}
```
### 4.1NioEventLoopGroup#newChild
```java
protected EventLoop newChild(Executor executor, Object... args) throws Exception {
    EventLoopTaskQueueFactory queueFactory = args.length == 4 ? (EventLoopTaskQueueFactory) args[3] : null;
    return new NioEventLoop(this, executor, (SelectorProvider) args[0],
        ((SelectStrategyFactory) args[1]).newSelectStrategy(), (RejectedExecutionHandler) args[2], queueFactory);
}
```
### 4.1.1
```java
NioEventLoop(NioEventLoopGroup parent, Executor executor, SelectorProvider selectorProvider,
                 SelectStrategy strategy, RejectedExecutionHandler rejectedExecutionHandler,
                 EventLoopTaskQueueFactory queueFactory) {
    super(parent, executor, false, newTaskQueue(queueFactory), newTaskQueue(queueFactory),
            rejectedExecutionHandler);
    this.provider = ObjectUtil.checkNotNull(selectorProvider, "selectorProvider");
    this.selectStrategy = ObjectUtil.checkNotNull(strategy, "selectStrategy");
    // nio的epoll
    final SelectorTuple selectorTuple = openSelector();
    this.selector = selectorTuple.selector;
    this.unwrappedSelector = selectorTuple.unwrappedSelector;
}
```
### 4.1.1.1
```java
protected SingleThreadEventLoop(EventLoopGroup parent, Executor executor,
                                    boolean addTaskWakesUp, Queue<Runnable> taskQueue, Queue<Runnable> tailTaskQueue,
                                    RejectedExecutionHandler rejectedExecutionHandler) {
    super(parent, executor, addTaskWakesUp, taskQueue, rejectedExecutionHandler);
    tailTasks = ObjectUtil.checkNotNull(tailTaskQueue, "tailTaskQueue");
}
```
```java
protected SingleThreadEventExecutor(EventExecutorGroup parent, Executor executor,
                                        boolean addTaskWakesUp, Queue<Runnable> taskQueue,
                                        RejectedExecutionHandler rejectedHandler) {
    super(parent);
    this.addTaskWakesUp = addTaskWakesUp;
    this.maxPendingTasks = DEFAULT_MAX_PENDING_EXECUTOR_TASKS;
    // 使用外部初始化的线程池
    this.executor = ThreadExecutorMap.apply(executor, this);
    // 核心处理队列
    this.taskQueue = ObjectUtil.checkNotNull(taskQueue, "taskQueue");
    this.rejectedExecutionHandler = ObjectUtil.checkNotNull(rejectedHandler, "rejectedHandler");
}
```
## 5.io.netty.bootstrap.AbstractBootstrap#bind(int)
```java
public ChannelFuture bind(int inetPort) {
    return bind(new InetSocketAddress(inetPort));
}
```
```java
public ChannelFuture bind(SocketAddress localAddress) {
    validate();
    return doBind(ObjectUtil.checkNotNull(localAddress, "localAddress"));
}
```
```java
private ChannelFuture doBind(final SocketAddress localAddress) {
	// 通过反射初始化外部传入的channel，例如NioServerSocketChannel(内部javaChannel()会获得java的原始Channel)
    final ChannelFuture regFuture = initAndRegister();
    final Channel channel = regFuture.channel();
    if (regFuture.cause() != null) {
        return regFuture;
    }
    // 执行完毕则监听回调，最终都会进入doBind0
    if (regFuture.isDone()) {
        // At this point we know that the registration was complete and successful.
        ChannelPromise promise = channel.newPromise();
        doBind0(regFuture, channel, localAddress, promise);
        return promise;
    } else {
        // Registration future is almost always fulfilled already, but just in case it's not.
        final PendingRegistrationPromise promise = new PendingRegistrationPromise(channel);
        regFuture.addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                Throwable cause = future.cause();
                if (cause != null) {
                    // Registration on the EventLoop failed so fail the ChannelPromise directly to not cause an
                    // IllegalStateException once we try to access the EventLoop of the Channel.
                    promise.setFailure(cause);
                } else {
                    // Registration was successful, so set the correct executor to use.
                    // See https://github.com/netty/netty/issues/2586
                    promise.registered();

                    doBind0(regFuture, channel, localAddress, promise);
                }
            }
        });
        return promise;
    }
}
```
- io.netty.bootstrap.AbstractBootstrap#initAndRegister
```java
final ChannelFuture initAndRegister() {
    Channel channel = null;
    try {
        channel = channelFactory.newChannel();
        init(channel);
    } catch (Throwable t) {
        if (channel != null) {
            // channel can be null if newChannel crashed (eg SocketException("too many open files"))
            channel.unsafe().closeForcibly();
            // as the Channel is not registered yet we need to force the usage of the GlobalEventExecutor
            return new DefaultChannelPromise(channel, GlobalEventExecutor.INSTANCE).setFailure(t);
        }
        // as the Channel is not registered yet we need to force the usage of the GlobalEventExecutor
        return new DefaultChannelPromise(new FailedChannel(), GlobalEventExecutor.INSTANCE).setFailure(t);
    }
    // 此处的group即模型中的parent
    ChannelFuture regFuture = config().group().register(channel);
    if (regFuture.cause() != null) {
        if (channel.isRegistered()) {
            channel.close();
        } else {
            channel.unsafe().closeForcibly();
        }
    }

    // If we are here and the promise is not failed, it's one of the following cases:
    // 1) If we attempted registration from the event loop, the registration has been completed at this point.
    //    i.e. It's safe to attempt bind() or connect() now because the channel has been registered.
    // 2) If we attempted registration from the other thread, the registration request has been successfully
    //    added to the event loop's task queue for later execution.
    //    i.e. It's safe to attempt bind() or connect() now:
    //         because bind() or connect() will be executed *after* the scheduled registration task is executed
    //         because register(), bind(), and connect() are all bound to the same thread.

    return regFuture;
}
```
### 5.1io.netty.channel.socket.nio.NioServerSocketChannel#NioServerSocketChannel()
```java
public NioServerSocketChannel() {
    this(newSocket(DEFAULT_SELECTOR_PROVIDER));
}
```
#### 5.1.1io.netty.channel.socket.nio.NioServerSocketChannel#newSocket
```java
private static ServerSocketChannel newSocket(SelectorProvider provider) {
    try {
        /**
         *  Use the {@link SelectorProvider} to open {@link SocketChannel} and so remove condition in
         *  {@link SelectorProvider#provider()} which is called by each ServerSocketChannel.open() otherwise.
         *
         *  See <a href="https://github.com/netty/netty/issues/2308">#2308</a>.
         */
    	// 获取NIO原生ServerSocketChannel
        return provider.openServerSocketChannel();
    } catch (IOException e) {
        throw new ChannelException(
                "Failed to open a server socket.", e);
    }
}
```
#### 5.1.2io.netty.channel.socket.nio.NioServerSocketChannel#NioServerSocketChannel(java.nio.channels.ServerSocketChannel)
```java
public NioServerSocketChannel(ServerSocketChannel channel) {
	// 传入ACCEPT绑定事件
    super(null, channel, SelectionKey.OP_ACCEPT);
    config = new NioServerSocketChannelConfig(this, javaChannel().socket());
}
```
- io.netty.channel.nio.AbstractNioChannel#AbstractNioChannel
```java
protected AbstractNioChannel(Channel parent, SelectableChannel ch, int readInterestOp) {
    super(parent);
    this.ch = ch;
    // 将绑定事件重新赋值
    this.readInterestOp = readInterestOp;
    try {
    	// javaNIO标准非阻塞写法
        ch.configureBlocking(false);
    } catch (IOException e) {
        try {
            ch.close();
        } catch (IOException e2) {
            logger.warn(
                        "Failed to close a partially initialized socket.", e2);
        }

        throw new ChannelException("Failed to enter non-blocking mode.", e);
    }
}
```
- io.netty.channel.AbstractChannel#AbstractChannel(io.netty.channel.Channel)
```java
protected AbstractChannel(Channel parent) {
    this.parent = parent;
    id = newId();
    unsafe = newUnsafe();
    // 初始化pipeline
    pipeline = newChannelPipeline();
}
```
#### 5.1.2.1io.netty.channel.AbstractChannel#newChannelPipeline
```java
protected DefaultChannelPipeline newChannelPipeline() {
    return new DefaultChannelPipeline(this);
}
```
- io.netty.channel.DefaultChannelPipeline#DefaultChannelPipeline
```java
protected DefaultChannelPipeline(Channel channel) {
    this.channel = ObjectUtil.checkNotNull(channel, "channel");
    succeededFuture = new SucceededChannelFuture(channel, null);
    voidPromise =  new VoidChannelPromise(channel, true);
    // 初始的pipeline有head和tail
    tail = new TailContext(this);
    head = new HeadContext(this);

    head.next = tail;
    tail.prev = head;
}
```
### 5.2io.netty.bootstrap.ServerBootstrap#init
```java
void init(Channel channel) {
    setChannelOptions(channel, newOptionsArray(), logger);
    setAttributes(channel, attrs0().entrySet().toArray(EMPTY_ATTRIBUTE_ARRAY));

    ChannelPipeline p = channel.pipeline();

    final EventLoopGroup currentChildGroup = childGroup;
    final ChannelHandler currentChildHandler = childHandler;
    final Entry<ChannelOption<?>, Object>[] currentChildOptions;
    synchronized (childOptions) {
        currentChildOptions = childOptions.entrySet().toArray(EMPTY_OPTION_ARRAY);
    }
    final Entry<AttributeKey<?>, Object>[] currentChildAttrs = childAttrs.entrySet().toArray(EMPTY_ATTRIBUTE_ARRAY);

    // 向pipeline中添加ChannelInitializer(根据注释该类只会执行一次就会被删除)，注意这个addLast是添加到倒数第二个，倒数第一的tail是不动的(此处只是将该实例添加到pipeline，但未执行具体的initChannel)
    p.addLast(new ChannelInitializer<Channel>() {
        @Override
        public void initChannel(final Channel ch) {
            final ChannelPipeline pipeline = ch.pipeline();
            ChannelHandler handler = config.handler();
            if (handler != null) {
                pipeline.addLast(handler);
            }

            ch.eventLoop().execute(new Runnable() {
                @Override
                public void run() {
                	// 添加ServerBootstrapAcceptor核心
                    pipeline.addLast(new ServerBootstrapAcceptor(
                            ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
                }
            });
        }
    });
}
```
### 5.3io.netty.channel.MultithreadEventLoopGroup#register(io.netty.channel.Channel)
```java
public ChannelFuture register(Channel channel) {
    return next().register(channel);
}
```
#### 5.3.1io.netty.channel.MultithreadEventLoopGroup#next
```java
 public EventLoop next() {
 	// 从bossGroup中获取一个EventLoop，默认就是轮询获取一个线程
    return (EventLoop) super.next();
}
```
#### 5.3.2io.netty.channel.SingleThreadEventLoop#register(io.netty.channel.Channel)
```java
public ChannelFuture register(Channel channel) {
    return register(new DefaultChannelPromise(channel, this));
}
```
- io.netty.channel.SingleThreadEventLoop#register(io.netty.channel.ChannelPromise)
```java
public ChannelFuture register(final ChannelPromise promise) {
    ObjectUtil.checkNotNull(promise, "promise");
    promise.channel().unsafe().register(this, promise);
    return promise;
}
```
- io.netty.channel.AbstractChannel.AbstractUnsafe#register
```java
public final void register(EventLoop eventLoop, final ChannelPromise promise) {
        ObjectUtil.checkNotNull(eventLoop, "eventLoop");
        if (isRegistered()) {
            promise.setFailure(new IllegalStateException("registered to an event loop already"));
            return;
        }
        if (!isCompatible(eventLoop)) {
            promise.setFailure(
                    new IllegalStateException("incompatible event loop type: " + eventLoop.getClass().getName()));
            return;
        }

        AbstractChannel.this.eventLoop = eventLoop;

        if (eventLoop.inEventLoop()) {
            register0(promise);
        } else {
            try {
            	// 核心逻辑，异步存入task
                eventLoop.execute(new Runnable() {
                    @Override
                    public void run() {
                        register0(promise);
                    }
                });
            } catch (Throwable t) {
                logger.warn(
                        "Force-closing a channel whose registration task was not accepted by an event loop: {}",
                        AbstractChannel.this, t);
                closeForcibly();
                closeFuture.setClosed();
                safeSetFailure(promise, t);
            }
        }
    }
```
##### 5.3.2.1io.netty.util.concurrent.SingleThreadEventExecutor#execute(java.lang.Runnable)
```java
public void execute(Runnable task) {
    ObjectUtil.checkNotNull(task, "task");
    execute(task, !(task instanceof LazyRunnable) && wakesUpForTask(task));
}
```
- io.netty.util.concurrent.SingleThreadEventExecutor#execute(java.lang.Runnable, boolean)
```java
private void execute(Runnable task, boolean immediate) {
    boolean inEventLoop = inEventLoop();
    // 存入之前的task
    addTask(task);
    if (!inEventLoop) {
        // 此方法最终会异步有个runAllTask执行上面的task
        startThread();
        if (isShutdown()) {
            boolean reject = false;
            try {
                if (removeTask(task)) {
                    reject = true;
                }
            } catch (UnsupportedOperationException e) {
                // The task queue does not support removal so the best thing we can do is to just move on and
                // hope we will be able to pick-up the task before its completely terminated.
                // In worst case we will log on termination.
            }
            if (reject) {
                reject();
            }
        }
    }

    if (!addTaskWakesUp && immediate) {
        wakeup(inEventLoop);
    }
}
```
- io.netty.util.concurrent.SingleThreadEventExecutor#startThread
```java
private void startThread() {
    if (state == ST_NOT_STARTED) {
        if (STATE_UPDATER.compareAndSet(this, ST_NOT_STARTED, ST_STARTED)) {
            boolean success = false;
            try {
                doStartThread();
                success = true;
            } finally {
                if (!success) {
                    STATE_UPDATER.compareAndSet(this, ST_STARTED, ST_NOT_STARTED);
                }
            }
        }
    }
}
```
- io.netty.util.concurrent.SingleThreadEventExecutor#doStartThread
```java
private void doStartThread() {
    assert thread == null;
    executor.execute(new Runnable() {
        @Override
        public void run() {
            thread = Thread.currentThread();
            if (interrupted) {
                thread.interrupt();
            }

            boolean success = false;
            updateLastExecutionTime();
            try {
                // 核心方法
                SingleThreadEventExecutor.this.run();
                success = true;
            } catch (Throwable t) {
                logger.warn("Unexpected exception from an event executor: ", t);
            } finally {
                for (;;) {
                    int oldState = state;
                    if (oldState >= ST_SHUTTING_DOWN || STATE_UPDATER.compareAndSet(
                            SingleThreadEventExecutor.this, oldState, ST_SHUTTING_DOWN)) {
                        break;
                    }
                }

                // Check if confirmShutdown() was called at the end of the loop.
                if (success && gracefulShutdownStartTime == 0) {
                    if (logger.isErrorEnabled()) {
                        logger.error("Buggy " + EventExecutor.class.getSimpleName() + " implementation; " +
                                SingleThreadEventExecutor.class.getSimpleName() + ".confirmShutdown() must " +
                                "be called before run() implementation terminates.");
                    }
                }

                try {
                    // Run all remaining tasks and shutdown hooks. At this point the event loop
                    // is in ST_SHUTTING_DOWN state still accepting tasks which is needed for
                    // graceful shutdown with quietPeriod.
                    for (;;) {
                        if (confirmShutdown()) {
                            break;
                        }
                    }

                    // Now we want to make sure no more tasks can be added from this point. This is
                    // achieved by switching the state. Any new tasks beyond this point will be rejected.
                    for (;;) {
                        int oldState = state;
                        if (oldState >= ST_SHUTDOWN || STATE_UPDATER.compareAndSet(
                                SingleThreadEventExecutor.this, oldState, ST_SHUTDOWN)) {
                            break;
                        }
                    }

                    // We have the final set of tasks in the queue now, no more can be added, run all remaining.
                    // No need to loop here, this is the final pass.
                    confirmShutdown();
                } finally {
                    try {
                        cleanup();
                    } finally {
                        // Lets remove all FastThreadLocals for the Thread as we are about to terminate and notify
                        // the future. The user may block on the future and once it unblocks the JVM may terminate
                        // and start unloading classes.
                        // See https://github.com/netty/netty/issues/6596.
                        FastThreadLocal.removeAll();

                        STATE_UPDATER.set(SingleThreadEventExecutor.this, ST_TERMINATED);
                        threadLock.countDown();
                        int numUserTasks = drainTasks();
                        if (numUserTasks > 0 && logger.isWarnEnabled()) {
                            logger.warn("An event executor terminated with " +
                                    "non-empty task queue (" + numUserTasks + ')');
                        }
                        terminationFuture.setSuccess(null);
                    }
                }
            }
        }
    });
}
```

###### 5.3.2.1.1io.netty.channel.nio.NioEventLoop#run
```java
protected void run() {
    int selectCnt = 0;
    // 死循环要么进入监听要么处理任务
    for (;;) {
        try {
            int strategy;
            try {
                strategy = selectStrategy.calculateStrategy(selectNowSupplier, hasTasks());
                switch (strategy) {
                case SelectStrategy.CONTINUE:
                    continue;

                case SelectStrategy.BUSY_WAIT:
                    // fall-through to SELECT since the busy-wait is not supported with NIO

                case SelectStrategy.SELECT:
                    // 核心分支，用于select()阻塞
                    long curDeadlineNanos = nextScheduledTaskDeadlineNanos();
                    if (curDeadlineNanos == -1L) {
                        curDeadlineNanos = NONE; // nothing on the calendar
                    }
                    nextWakeupNanos.set(curDeadlineNanos);
                    try {
                        if (!hasTasks()) {
                        	// 如果没有任务则进入监听
                            strategy = select(curDeadlineNanos);
                        }
                    } finally {
                        // This update is just to help block unnecessary selector wakeups
                        // so use of lazySet is ok (no race condition)
                        nextWakeupNanos.lazySet(AWAKE);
                    }
                    // fall through
                default:
                }
            } catch (IOException e) {
                // If we receive an IOException here its because the Selector is messed up. Let's rebuild
                // the selector and retry. https://github.com/netty/netty/issues/8566
                rebuildSelector0();
                selectCnt = 0;
                handleLoopException(e);
                continue;
            }

            selectCnt++;
            cancelledKeys = 0;
            needsToSelectAgain = false;
            final int ioRatio = this.ioRatio;
            boolean ranTasks;
            if (ioRatio == 100) {
                try {
                    if (strategy > 0) {
                    	// 处理对应的事件，连接、读写等unsafe.read()，unsafe是AbstractNioChannel.NioUnsafe的内部类
                        // 在接收连接事件后会调用accept得到channel同时包装成netty的channel，接着调用parent的pipeline，里面的ServerBootstrapAcceptor会在此时发挥作用child.pipeline().addLast(childHandler);将用户写的handler加入到child里面,接着进行regist到workerGroup的其中一个event(selector)中和parent一样的结构以及循环
                        // 也即存到task后再runAllTask处理任务，回调pipeline
                        processSelectedKeys();
                    }
                } finally {
                    // Ensure we always run tasks.
                    // 执行队列里的task
                    ranTasks = runAllTasks();
                }
            } else if (strategy > 0) {
                final long ioStartTime = System.nanoTime();
                try {
                    processSelectedKeys();
                } finally {
                    // Ensure we always run tasks.
                    final long ioTime = System.nanoTime() - ioStartTime;
                    ranTasks = runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
                }
            } else {
                ranTasks = runAllTasks(0); // This will run the minimum number of tasks
            }

            if (ranTasks || strategy > 0) {
                if (selectCnt > MIN_PREMATURE_SELECTOR_RETURNS && logger.isDebugEnabled()) {
                    logger.debug("Selector.select() returned prematurely {} times in a row for Selector {}.",
                            selectCnt - 1, selector);
                }
                selectCnt = 0;
            } else if (unexpectedSelectorWakeup(selectCnt)) { // Unexpected wakeup (unusual case)
                selectCnt = 0;
            }
        } catch (CancelledKeyException e) {
            // Harmless exception - log anyway
            if (logger.isDebugEnabled()) {
                logger.debug(CancelledKeyException.class.getSimpleName() + " raised by a Selector {} - JDK bug?",
                        selector, e);
            }
        } catch (Throwable t) {
            handleLoopException(t);
        }
        // Always handle shutdown even if the loop processing threw an exception.
        try {
            if (isShuttingDown()) {
                closeAll();
                if (confirmShutdown()) {
                    return;
                }
            }
        } catch (Throwable t) {
            handleLoopException(t);
        }
    }
}
```
## 6.io.netty.channel.AbstractChannel.AbstractUnsafe#register0
```java
private void register0(ChannelPromise promise) {
    try {
        // check if the channel is still open as it could be closed in the mean time when the register
        // call was outside of the eventLoop
        if (!promise.setUncancellable() || !ensureOpen(promise)) {
            return;
        }
        boolean firstRegistration = neverRegistered;
        // selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);nio注册的同样写法
        doRegister();
        neverRegistered = false;
        registered = true;

        // Ensure we call handlerAdded(...) before we actually notify the promise. This is needed as the
        // user may already fire events through the pipeline in the ChannelFutureListener.
        pipeline.invokeHandlerAddedIfNeeded();

        safeSetSuccess(promise);
        // 责任链调用
        pipeline.fireChannelRegistered();
        // Only fire a channelActive if the channel has never been registered. This prevents firing
        // multiple channel actives if the channel is deregistered and re-registered.
        if (isActive()) {
            if (firstRegistration) {

                pipeline.fireChannelActive();
            } else if (config().isAutoRead()) {
                // This channel was registered before and autoRead() is set. This means we need to begin read
                // again so that we process inbound data.
                //
                // See https://github.com/netty/netty/issues/4805
                beginRead();
            }
        }
    } catch (Throwable t) {
        // Close the channel directly to avoid FD leak.
        closeForcibly();
        closeFuture.setClosed();
        safeSetFailure(promise, t);
    }
}
```
```java
protected void doRegister() throws Exception {
    boolean selected = false;
    for (;;) {
        try {
        	// 即将serverSocket注册上ACCEPT监听
            selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);
            return;
        } catch (CancelledKeyException e) {
            if (!selected) {
                // Force the Selector to select now as the "canceled" SelectionKey may still be
                // cached and not removed because no Select.select(..) operation was called yet.
                eventLoop().selectNow();
                selected = true;
            } else {
                // We forced a select operation on the selector before but the SelectionKey is still cached
                // for whatever reason. JDK bug ?
                throw e;
            }
        }
    }
}
```
## 7.io.netty.channel.nio.NioEventLoop#processSelectedKeys|当有客户端连接时
io.netty.channel.nio.NioEventLoop#processSelectedKeysOptimized->io.netty.channel.nio.NioEventLoop#processSelectedKey(java.nio.channels.SelectionKey, io.netty.channel.nio.AbstractNioChannel)
```java
private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
    final AbstractNioChannel.NioUnsafe unsafe = ch.unsafe();
    if (!k.isValid()) {
        final EventLoop eventLoop;
        try {
            eventLoop = ch.eventLoop();
        } catch (Throwable ignored) {
            // If the channel implementation throws an exception because there is no event loop, we ignore this
            // because we are only trying to determine if ch is registered to this event loop and thus has authority
            // to close ch.
            return;
        }
        // Only close ch if ch is still registered to this EventLoop. ch could have deregistered from the event loop
        // and thus the SelectionKey could be cancelled as part of the deregistration process, but the channel is
        // still healthy and should not be closed.
        // See https://github.com/netty/netty/issues/5125
        if (eventLoop == this) {
            // close the channel if the key is not valid anymore
            unsafe.close(unsafe.voidPromise());
        }
        return;
    }

    try {
        int readyOps = k.readyOps();
        // We first need to call finishConnect() before try to trigger a read(...) or write(...) as otherwise
        // the NIO JDK channel implementation may throw a NotYetConnectedException.
        if ((readyOps & SelectionKey.OP_CONNECT) != 0) {
            // remove OP_CONNECT as otherwise Selector.select(..) will always return without blocking
            // See https://github.com/netty/netty/issues/924
            int ops = k.interestOps();
            ops &= ~SelectionKey.OP_CONNECT;
            k.interestOps(ops);

            unsafe.finishConnect();
        }

        // Process OP_WRITE first as we may be able to write some queued buffers and so free memory.
        if ((readyOps & SelectionKey.OP_WRITE) != 0) {
            // Call forceFlush which will also take care of clear the OP_WRITE once there is nothing left to write
            ch.unsafe().forceFlush();
        }

        // Also check for readOps of 0 to workaround possible JDK bug which may otherwise lead
        // to a spin loop
        // 判断事件
        if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
            unsafe.read();
        }
    } catch (CancelledKeyException ignored) {
        unsafe.close(unsafe.voidPromise());
    }
}
```
### 7.1io.netty.channel.nio.AbstractNioMessageChannel.NioMessageUnsafe#read
```java
public void read() {
    assert eventLoop().inEventLoop();
    final ChannelConfig config = config();
    final ChannelPipeline pipeline = pipeline();
    final RecvByteBufAllocator.Handle allocHandle = unsafe().recvBufAllocHandle();
    allocHandle.reset(config);

    boolean closed = false;
    Throwable exception = null;
    try {
        try {
            do {
                int localRead = doReadMessages(readBuf);
                if (localRead == 0) {
                    break;
                }
                if (localRead < 0) {
                    closed = true;
                    break;
                }

                allocHandle.incMessagesRead(localRead);
            } while (allocHandle.continueReading());
        } catch (Throwable t) {
            exception = t;
        }

        // doReadMessages往里面塞了值
        int size = readBuf.size();
        for (int i = 0; i < size; i ++) {
            readPending = false;
            // 调用ServerBootstrapAcceptor中会将读事件进行注册
            pipeline.fireChannelRead(readBuf.get(i));
        }
        readBuf.clear();
        allocHandle.readComplete();
        pipeline.fireChannelReadComplete();

        if (exception != null) {
            closed = closeOnReadError(exception);

            pipeline.fireExceptionCaught(exception);
        }

        if (closed) {
            inputShutdown = true;
            if (isOpen()) {
                close(voidPromise());
            }
        }
    } finally {
        // Check if there is a readPending which was not processed yet.
        // This could be for two reasons:
        // * The user called Channel.read() or ChannelHandlerContext.read() in channelRead(...) method
        // * The user called Channel.read() or ChannelHandlerContext.read() in channelReadComplete(...) method
        //
        // See https://github.com/netty/netty/issues/2254
        if (!readPending && !config.isAutoRead()) {
            removeReadOp();
        }
    }
}
```
#### 7.1.1io.netty.channel.socket.nio.NioServerSocketChannel#doReadMessages
```java
protected int doReadMessages(List<Object> buf) throws Exception {
	// 获取连接事件通道，后续用于注册读事件
    SocketChannel ch = SocketUtils.accept(javaChannel());

    try {
        if (ch != null) {
            buf.add(new NioSocketChannel(this, ch));
            return 1;
        }
    } catch (Throwable t) {
        logger.warn("Failed to create a new channel from an accepted socket.", t);

        try {
            ch.close();
        } catch (Throwable t2) {
            logger.warn("Failed to close a socket.", t2);
        }
    }

    return 0;
}
```
- io.netty.channel.socket.nio.NioSocketChannel#NioSocketChannel(io.netty.channel.Channel, java.nio.channels.SocketChannel)
```java
public NioSocketChannel(Channel parent, SocketChannel socket) {
    super(parent, socket);
    config = new NioSocketChannelConfig(this, socket.socket());
}
```
- io.netty.channel.nio.AbstractNioByteChannel#AbstractNioByteChannel
```java
protected AbstractNioByteChannel(Channel parent, SelectableChannel ch) {
	// 读事件后续用于注册
    super(parent, ch, SelectionKey.OP_READ);
}
```
## 8.io.netty.bootstrap.ServerBootstrap.ServerBootstrapAcceptor#channelRead
```java
public void channelRead(ChannelHandlerContext ctx, Object msg) {
	// msg是后面创建的socket即worker的，不是boss的
    final Channel child = (Channel) msg;

    child.pipeline().addLast(childHandler);

    setChannelOptions(child, childOptions, logger);
    setAttributes(child, childAttrs);

    try {
    	// 将workerGroup与childSocket进行关联，此处的注册与bossGroup的注册逻辑一致，从线程池中出一个线程跑逻辑把用户自己的handler加入pipeline，然后等待读事件进入
        childGroup.register(child).addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                if (!future.isSuccess()) {
                    forceClose(child, future.cause());
                }
            }
        });
    } catch (Throwable t) {
        forceClose(child, t);
    }
}
```
### 8.1io.netty.channel.nio.AbstractNioByteChannel.NioByteUnsafe#read|读事件进入
```java
public final void read() {
    final ChannelConfig config = config();
    if (shouldBreakReadReady(config)) {
        clearReadPending();
        return;
    }
    final ChannelPipeline pipeline = pipeline();
    final ByteBufAllocator allocator = config.getAllocator();
    final RecvByteBufAllocator.Handle allocHandle = recvBufAllocHandle();
    allocHandle.reset(config);

    ByteBuf byteBuf = null;
    boolean close = false;
    try {
        do {
        	// 生成缓冲区
            byteBuf = allocHandle.allocate(allocator);
            // 将socket中的数据写入缓冲区byteBuf
            allocHandle.lastBytesRead(doReadBytes(byteBuf));
            if (allocHandle.lastBytesRead() <= 0) {
                // nothing was read. release the buffer.
                byteBuf.release();
                byteBuf = null;
                close = allocHandle.lastBytesRead() < 0;
                if (close) {
                    // There is nothing left to read as we received an EOF.
                    readPending = false;
                }
                break;
            }

            allocHandle.incMessagesRead(1);
            readPending = false;
            pipeline.fireChannelRead(byteBuf);
            byteBuf = null;
        } while (allocHandle.continueReading());

        allocHandle.readComplete();
        pipeline.fireChannelReadComplete();

        if (close) {
            closeOnRead(pipeline);
        }
    } catch (Throwable t) {
        handleReadException(pipeline, byteBuf, t, close, allocHandle);
    } finally {
        // Check if there is a readPending which was not processed yet.
        // This could be for two reasons:
        // * The user called Channel.read() or ChannelHandlerContext.read() in channelRead(...) method
        // * The user called Channel.read() or ChannelHandlerContext.read() in channelReadComplete(...) method
        //
        // See https://github.com/netty/netty/issues/2254
        if (!readPending && !config.isAutoRead()) {
            removeReadOp();
        }
    }
}
```