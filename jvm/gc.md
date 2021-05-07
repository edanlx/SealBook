# full gc分析思路
[目录](https://github.com/edanlx/SealBook/blob/master/catalog.md)  
[可直接运行的完整代码](https://github.com/edanlx/TechingCode/tree/master/demoGrace/src/main/java/com/example/demo/lesson/jvm/gc) 
[视频讲解](https://www.bilibili.com/video/BV1Ey4y167HQ/)   
[文字版](https://github.com/edanlx/SealBook/blob/master/jvm/gc.md)

屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。

## 1.命令界面
### 1.1.Jmap堆命令
jmap -histo:live {pid} | head -13  
* num:序号
* instances:实例数量
* bytes:占用空间大小
* class name:类名称，[C is a char[]，[S is a short[]，[I is a int[]，[B is a byte[]，[[I is a int[][]

### 1.2.Jstack线程命令
执行 jstack -l {pid}


### 1.3.Jinfo运行参数命令
jinfo {pid}
### 1.4.Jstat综合命令
垃圾回收统计jstat -gc {pid}

* NGCMN:新生代最小容量 
* NGCMX:新生代最大容量
* NGC:当前新生代容量
* S0C:第一个幸存区大小
* S1C:第二个幸存区的大小
* EC:伊甸园区的大小
* OGCMN:老年代最小容量
* OGCMX:老年代最大容量
* OGC:当前老年代大小 
* OC:当前老年代大小
* MCMN:最小元数据容量
* MCMX:最大元数据容量
* MC:当前元数据空间大小 
* CCSMN:最小压缩类空间大小 
* CCSMX:最大压缩类空间大小 
* CCSC:当前压缩类空间大小 
* YGC:年轻代gc次数 
* FGC:老年代GC次数

## 2.可视化界面
### 2.1.jvisualvm
### 2.2.Arthas
* 下载：curl -O https://arthas.aliyun.com/arthas-boot.jar
* 启动 java -jar arthas-boot.jar
|命令|介绍|
|--|--|
|dashboard|当前系统的实时数据面板|
|thread|查看当前 JVM 的线程堆栈信息|
|watch|方法执行数据观测|
|trace|方法内部调用路径，并输出方法路径上的每个节点上耗时|
|stack|输出当前方法被调用的调用路径|
|tt|方法执行数据的时空隧道，记录下指定方法每次调用的入参和返回信息，并能对这些不同的时间下调用进行观测|
|jvm|查看当前 JVM 信息|
|vmoption|查看，更新 JVM 诊断相关的参数|
|sc|查看 JVM 已加载的类信息|
|sm|查看已加载类的方法信息|
|jad|反编译指定已加载类的源码|
|classloader|查看 classloader 的继承树，urls，类加载信息|
|heapdump|类似 jmap 命令的 heap dump 功能|

## 3.线上参数
堆溢出自动打印日志
‐XX:+HeapDumpOnOutOfMemoryError
打印路径
‐XX:HeapDumpPath=D:\jvm.dump
如果有设置Xms，需要设置Xmx并保持一致防止堆抖动，元空间同理
‐Xms1536M(默认是物理内存的1/64) ‐Xmx1536(默认是物理内存的1/4)
禁止调用system.gc
­XX:+DisableExplicitGC
gc日志相关
‐Xloggc:./gc‐%t.log‐XX:+PrintGCDetails‐XX:+PrintGCDateStamps‐XX:+PrintGCTimeStamps‐XX:+PrintGCCause ‐XX:+UseGCLogFileRotation‐XX:NumberOfGCLogFiles=10‐XX:GCLogFileSize=100M

## 参考资料
《深入理解Java虚拟机》-周志明