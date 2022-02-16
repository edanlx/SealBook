# 【JAVA分布式】01-spring扫描BeanDefinition
> 欢迎关注b站账号/公众号【六边形战士夏宁】，一个要把各项指标拉满的男人。该文章已在[github目录](https://github.com/edanlx/SealBook/blob/master/catalogue/wechat.md)收录。
屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。

## 1.背景
zookeeper是Apache下Hadoop的子项目，分布式协调框架。用来解决统一命名服务，状态同步服务，集群管理，分布式应用配置的项目管理等

## 2.基础知识
### 2.1链接方式
Client与Server使用长连接，避免每次发出三次握手，通过ping、pong进行维护，在发送请求后会获得sessionid
### 2.2六种节点(核心特性一)
1. PERSISTENT持久化目录节点、
2. PERSISTENT_SEQUENTIAL持久化顺序编号目录节点(会加顺序后缀)
3. EPHEMERAL临时目录节点 
4. EPHEMERAL_SEQUENTIAL临时顺序编号目录节点
5. Container 节点(没有子节点会被清除，默认60秒扫描一次，分布式锁的实现)
6. TTL 节点(过期删除)
### 2.3监听(核心特性二)
#### 2.3.1数据监听
1. get -w /path 获取数据
2. stat -w /path 查看节点状态
#### 2.3.2目录监听
1. ls -w /path 当前目录收到一次
2. ls -R -w /path 递归，每个目录都会收到一次
#### 2.3配置文件
1. tickTime
最小时间单位，默认2秒
2. initLimit
数据同步时间,默认10个单位，即20秒
3. syncLimit
心跳检测时间，默认5个单位，即10秒
4. dataDir
数据存放位置
5. clientPort
端口
### 2.4get -s
1. cZxid
所有的set请求都会有事务id，创建时的事务id
2. ctime
创建时间
3. mZxid
修改事务id
4. mtime
修改时间
5. pZxid
子节点列表发生变化时的事务id
6. cversion
子节点列表版本
7. dataVersion
乐观锁
8. aclVersion
权限锁
9. XXXOwner
sessionId，连接断开临时节点被销毁，如果不是临时节点该字段为0
9. dataLength
数据字节数
10. numChilder
子节点数量
### 2.5持久化
1. 日志:磁盘预分配固定大小，然后写入  
查看日志:
```
java ‐classpath .:slf4j‐api‐1.7.25.jar:zookeeper‐3.5.8.jar:zookeeper‐jute‐3.5.8.jar org.apache.zookeeper.server.LogFormatter /usr/local/zookeeper/apache‐zookeeper‐3.5.8‐bin/data/version‐2/log.1
```
2. 快照:执行多少次命令形成一个快照
```linux
java ‐classpath .:slf4j‐api‐1.7.25.jar:zookeeper‐3.7.0.jar:zookeeper‐jute‐3.7.0.jar org.apache.zookeeper.server.SnapshotFormatter /usr/local/zookeeper/apac he‐zookeeper‐3.5.8‐bin/data‐dir/version‐2/snapshot.0
```
## 3.综合应用
### 3.1非公平锁
1. 获取锁
2. 当前锁是否右其它事务创建，是的话进入监听等待获取锁
3. 如果没有创建，则自己创建
4. 创建失败则进入监听等待
5. 加锁->释放锁
### 3.2公平锁
1. 在某个节点下创建临时顺序节点
2. 判断自己是不是最小的，是获取锁，不是则监听前一个节点
3. 获取锁后删除最小的即自己的节点
### 3.3读写锁
1. 读操作，如果前面有些请求，则对写请求监听，如果没有写请求直接获取锁
2. 写请求，则直接进入互斥锁
### 3.4选举
利用zk的公平锁进行选举，不是zk自身选举，例如kafka利用了该功能
### 3.5注册中心
消费者都去注册中心的指定路径各自生成自己的临时节点，生产者则去该指定路径获取所有的临时节点即获取到所有的消费者并监听，缓存后在消费者端实现负载均衡。如果是自身实现例如springcloud+zookeeper,负载均衡默认是ribbon，ribbon有自己的心跳检测机制

## 4.start
构造函数如下
```java
public ZooKeeper(
    String connectString,
    int sessionTimeout,
    Watcher watcher,
    boolean canBeReadOnly,
    HostProvider aHostProvider,
    ZKClientConfig clientConfig) throws IOException {
    LOG.info(
        "Initiating client connection, connectString={} sessionTimeout={} watcher={}",
        connectString,
        sessionTimeout,
        watcher);

    if (clientConfig == null) {
        clientConfig = new ZKClientConfig();
    }
    this.clientConfig = clientConfig;
    watchManager = defaultWatchManager();
    watchManager.defaultWatcher = watcher;
    ConnectStringParser connectStringParser = new ConnectStringParser(connectString);
    hostProvider = aHostProvider;

    // 建立客户端连接
    cnxn = createConnection(
        connectStringParser.getChrootPath(),
        hostProvider,
        sessionTimeout,
        this,
        watchManager,
        getClientCnxnSocket(),
        canBeReadOnly);
    // 核心启动
    cnxn.start();
}
```
```java
public void start() {
	// 客户端向服务端发送数据的相应启动
    sendThread.start();
    // 服务端向客户端返回数据或发送数据的相应启动
    eventThread.start();
}
```
### 4.1start