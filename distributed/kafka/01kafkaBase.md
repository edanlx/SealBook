# 【JAVA分布式】01-spring扫描BeanDefinition
> 欢迎关注b站账号/公众号【六边形战士夏宁】，一个要把各项指标拉满的男人。该文章已在[github目录](https://github.com/edanlx/SealBook/blob/master/catalogue/wechat.md)收录。
屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。

## 1.背景
zookeeper是Apache下Hadoop的子项目，分布式协调框架。用来解决统一命名服务，状态同步服务，集群管理，分布式应用配置的项目管理等

## 2.基础知识
|名称|解释|
|--|--|
|Broker|消息中间件处理节点|
|Producer|生产者|
|Consumer|消费者|
|Topic|Topic|
|ConsumerGroup|一个消息可被多个ConsumerGroup消费，但一个组只有一个Consumer消费|
|Partition|一个topic分为多个Partition，可以配置一个topic有几个分区|

- 选举
    - controller
    会有一个总控制器负责管理与选举与zk交互并通知其它broker，使用zk的选举功能
    - Partition
    从ISR列表中选择第一个即同步数据最多的
- 以命令为例，--describe --group XXXgroup，会议列表呈现
    - Group代表消费者组名
    - Topic代表该组消费消费的TOPIC
    - current-offset消费到了哪个偏移量
    - last-end-offset该topic的偏移量
    - HOST
    - LAG未消费的量
- 选举
kafka的选举是针对分区的，因为正常情况下一台机器是有多个分区的,多份不同的commitLog文件，而rocketmq是一台机器就一个commitLog，所以在多topic的时候kafka有明显性能下降，所以这也是kafaka不需要至少3台机器的集群。和rocketmq一样broker不能超过分区数量，否则无法分配
- 顺序消费
例如ELK，是根据时间戳二次实现的，并发kafka实现
- 负载均衡
默认是用groupId进行取模得到分区
- 持久化文件
与rocketmq基本一致，commitlog也是以offset进行记录，索引文件是有一定间隔值的例如0-4-8等
- offset
consumer与kafka的topic均保存offset情况，方便rebanlance，默认50个分区
- 拉取消息
采用长轮询
- 消费组
不设置会默认一个名称

- rebanlance
    - 触发
        1. 消费组的consumer增加或减少
        2. 动态给topic增加分区
        3. 消费者订阅更多topic
rebanlance中，消费者无法从kafka获取消息
    - 分配策略
    默认range(取模，还有轮询和sticky，sticky保证原有尽可能不变)
    - 步骤
        1. 由分区leader指定消费组的组长GroupCoordinator(最先连接的)进行消费组的分配方案
        2. 每个GroupCoordinator再找到该分区的的leader将分配结果分给每个该组的consumer
- HW
    所有ISR最小的那个，意义在于等待同步完成才能消费
- 高性能
    1. 磁盘顺序读写
    2. 零拷贝
    3. 批量提交
## 3.参数
- ACKS_CONFIG(默认0)
    * 0代表不等待
    * 1代表等待本地持久化
    * -1/all代表副本同步成功(副本数量由min.insync.replices)
- RETRIES_CONFIG(重试次数)
- RETRIES_BACKOFF_MS_CONFIG(重试间隔)
- BATCH_SIZE_CONFIG(一次网络请求会发送多条信息的大小)
- LINGER_MS_CONFIG(等待毫秒数如果未达到上述要求也会发送)
- AUTO_COMMIT_INTERVAL_MS_CONFIG(自动提交定时毫秒数)
- ENABLE_AUTO_COMMIT_CONFIG(是否自动提交，默认true，数据要求高时应改为false)
- MAX_POLL_RECODS_CONFIG(一次拉取最大数量默认500)
- MAX_POLL_INTERVAL_MS_CONFIG(一批消息最长处理时间，能力过弱会被剔除)
- AUTO_OFFSET_REST_CONFIG("earlist"新加入的消费组从头开始而不是从当前时间点开始)
- enable.idempotence(在broker会记录PID，但只能保证消费者的消息幂等)

