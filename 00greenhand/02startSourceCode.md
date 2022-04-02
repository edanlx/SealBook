
## 1.背景介绍
常见源码服务端启动汇总。相关源码下载地址。
* https://github.com/apache/zookeeper
* https://github.com/apache/rocketmq
* https://github.com/alibaba/nacos
* https://github.com/alibaba/sentinel
* https://github.com/alibaba/gateway
## 2.zookeeper
3台机器
### 2.1建立文件持久化路径
建议在项目的根路径建立temp文件夹，底下分别建立zookeeper1、zookeeper2、zookeeper3，同时每个路径下建立myid文件，内部写上1/2/3与server对应上就行。windows环境记得转义。
### 2.2建立配置文件
复制conf下的zoo_sample.cfg,修改如下关键信息,dataDir为刚才创建的文件,clientPort别互相冲突就行,例如2181,2182,2183。
```
dataDir=F:\\IdeaProjects\\zookeeper\\data\\zookeeper1
clientPort=2181
```
最后添加集群信息如下,根据自身配置修改
```
server.1=127.0.0.1:2887:3887
server.2=127.0.0.1:2888:3888
server.3=127.0.0.1:2889:3889
```
![00greenhand_02startSourceCode_zookeeperConfig](F:\git\SealBook\tools\pic\00greenhand_02startSourceCode_zookeeperConfig.png)

### 2.3修改maven文件

metrics-core、snappy-java的scope属性全部删除，否则编译会报错

![00greenhand_02startSourceCode_zookeeperMaven](F:\git\SealBook\tools\pic\00greenhand_02startSourceCode_zookeeperMaven.png)

### 2.4程序包org.apache.zookeeper.data不存在
Jute包编译一下，再整体刷新一下，不然会有类找不到

![00greenhand_02startSourceCode_zookeeperJute](F:\git\SealBook\tools\pic\00greenhand_02startSourceCode_zookeeperJute.png)

### 2.5程序包org.apache.zookeeper.version不存在

```java
package org.apache.zookeeper.version;

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

![00greenhand_02startSourceCode_zookeeperInfo](F:\git\SealBook\tools\pic\00greenhand_02startSourceCode_zookeeperInfo.png)

### 2.6设置启动类
org.apache.zookeeper.server.quorum.QuorumPeerMain
![00greenhand_02startSourceCode_zookeeperInfo](F:\git\SealBook\tools\pic\00greenhand_02startSourceCode_zookeeperInfo.png)

## 3.nacos