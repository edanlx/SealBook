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
一般加入解码器StringDecoder、编码器StringEncoder(其实现ChannelOutboundHandler)、Handler(自己的实现逻辑)
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