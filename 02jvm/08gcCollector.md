# 【jvm】08-垃圾回收器那么多傻傻分不清?
> 欢迎关注b站账号/公众号【六边形战士夏宁】，一个要把各项指标拉满的男人。该文章已在[github目录](https://github.com/edanlx/SealBook)收录。
屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。
* [gitee目录](https://gitee.com/seal_li/SealBook)
* [知乎目录](https://zhuanlan.zhihu.com/p/338222208)
* [csdn目录](https://blog.csdn.net/seal_li/article/details/111415366)
* [博客园目录](https://www.cnblogs.com/sealLee/articles/14748368.html)
* [视频讲解](https://www.bilibili.com/video/BV1S5411V74U/) 
* [上一篇](./07concurrence.md)
* [下一篇](./09gc.md)

## 1.垃圾收集算法
### 1.1.标记-复制算法
该算法将内存分为2块均等的，当该区域使用完毕后就一次性复制到另一块区域。在Hotspot中实现即为s0与s1,只不过做了优化吧伊甸园分了出去，目前年轻代的垃圾收集器都是用该算法。标记-复制算法极度浪费空间，在新生代有95%的对象都会被回收掉自然还好，但是在老年代就行不通了于是演变出了后面两种算法。
### 1.2.标记-清除算法
顾名思义就是将需要回收的垃圾标记后清除缺点为空间不连续无法存放大对象。需要另做处理。
### 1.3.标记-整理算法
将垃圾清理后排列整齐，缺点为内存地址变动，需要重新修正。
## 2.垃圾收集器
### 2.1.Serial收集器
历史悠久的垃圾收集器，收集时必须"Stop The World" ，早期会与Serial Old联用，目前仅剩下单线程服务器才有使用价值。
### 2.2.Parallel Scavenge收集器
即Serial的多线程版本，早期会与Parallel Old联用，除非内存过小，否则已无使用价值
### 2.3.ParNew收集器
Parallel Scavenge的优化版本，真正的多线程收集器。自面世后在JDK9以前一直与CMS联用霸占了相当长时间，目前也是优秀的选择。
### 2.4.Serial Old收集器
早期单线程老年代回收，使用标记整理算法，在CMS并发失败后会默认调用
### 2.5.Parallel Old收集器
多线程老年代回收，使用标记整理算法，JDK默认使用但一般都会选择CMS
### 2.6.CMS收集器
Concurrent Mark Sweep跨时代的老年代回收器，后面的垃圾回收器都以CMS为蓝本开发。使用了标记清除，但可以开启整理算法。核心思想采用三色标记思想(白色为可回收，黑色为不可回收，灰色为未处理完毕)。等待所有灰色处理完毕则完成一次迭代。

* **初始标记(STW)** ：记录下gc roots直接能引用的对象，速度很快
* **并发标记** ：沿着root向下寻找，时间最长
* **重新标记** ：写屏障 + 增量更新(保存写后数据)。对并发过程中有重新赋值的进行重新标记触发STW，
* **并发清理** ：对白色的垃圾进行清理
* **并发重置** ：重置标记进入下一轮循环

1. -XX:+UseConcMarkSweepGC:启用cms
2. -XX:ConcGCThreads:并发的GC线程数
3. -XX:+UseCMSCompactAtFullCollection:FullGC之后做压缩整理(减少碎片)
4. -XX:CMSFullGCsBeforeCompaction:多少次FullGC之后压缩一次，默认是0，代表每次FullGC后都会压缩一 次
5. -XX:CMSInitiatingOccupancyFraction: 当老年代使用达到该比例时会触发FullGC(默认是92，这是百分比) 6. -XX:+UseCMSInitiatingOccupancyOnly:只使用设定的回收阈值(-XX:CMSInitiatingOccupancyFraction设 定的值)，如果不指定，JVM仅在第一次使用设定值，后续则会自动调整
7. -XX:+CMSScavengeBeforeRemark:在CMS GC前启动一次minor gc，目的在于减少老年代对年轻代的引 用，降低CMS GC的标记阶段时的开销，一般CMS的GC耗时 80%都在标记阶段
8. -XX:+CMSParallellnitialMarkEnabled:表示在初始标记的时候多线程执行，缩短STW
9. -XX:+CMSParallelRemarkEnabled:在重新标记的时候多线程执行，缩短STW;

### 2.7.G1收集器
JDK9默认使用收集器，保留了分代收集理论但将内存分为2048个格子，每个格子可以在不同的时机充当不同的区域。主要分为四区：伊甸园，s区，老年区，大对象区。G1的过程与CMS基本一致，不同的是它是全代收集且算法更为复杂。整体上标记整理算法，小区上为标记复制
写屏障 + 原始快照(SATB)(保存写前数据)

1. -XX:+UseG1GC:使用G1收集器
2. -XX:ParallelGCThreads:指定GC工作的线程数量 -XX:G1HeapRegionSize:指定分区大小(1MB~32MB，且必须是2的N次幂)，默认将整堆划分为2048个分区 -XX:MaxGCPauseMillis:目标暂停时间(默认200ms)
3. -XX:G1NewSizePercent:新生代内存初始空间(默认整堆5%)
4. -XX:G1MaxNewSizePercent:新生代内存最大空间 -XX:TargetSurvivorRatio:Survivor区的填充容量(默认50%)，Survivor区域里的一批对象(年龄1+年龄2+年龄n的多个年龄对象)总和超过了Survivor区域的50%，此时就会把年龄n(含)以上的对象都放入老年代 -XX:MaxTenuringThreshold:最大年龄阈值(默认15) -XX:InitiatingHeapOccupancyPercent:老年代占用空间达到整堆内存阈值(默认45%)，则执行新生代和老年代的混合收集(MixedGC)，比如我们之前说的堆默认有2048个region，如果有接近1000个region都是老年代的region，则可能 就要触发MixedGC了
5. -XX:G1MixedGCLiveThresholdPercent(默认85%) region中的存活对象低于这个值时才会回收该region，如果超过这 个值，存活对象过多，回收的的意义不大。
6. -XX:G1MixedGCCountTarget:在一次回收过程中指定做几次筛选回收(默认8次)，在最后一个筛选回收阶段可以回收一 会，然后暂停回收，恢复系统运行，一会再开始回收，这样可以让系统不至于单次停顿时间过长。
7. -XX:G1HeapWastePercent(默认5%):gc过程中空出来的region是否充足阈值，在混合回收的时候，对Region回收都是基于复制算法进行的，都是把要回收的Region里的存活对象放入其他Region，然后这个Region中的垃圾对象全部清理掉，这样的话在回收过程就会不断空出来新的Region，一旦空闲出来的Region数量达到了堆内存的5%，此时就会立 即停止混合回收，意味着本次混合回收就结束了。

### 2.8.ZGC收集器
核心为使用读屏障，缺点为占用内存开销较大。真正意义上的全代收集，目前还在完善中但已可以使用
## 3.补充知识
### 3.1.垃圾收集器的选择
JDK 1.8默认使用 Parallel(年轻代和老年代都是) JDK 1.9默认使用 G1。4G以下可以用parallel，4-8G可以用ParNew+CMS，8G以上可以用G1，几百G以上用ZGC。
### 3.2.记忆集与卡表
Hotspot使用写屏障维护卡表状态，记忆集为抽象概念
### 3.3.安全点与安全区域
为保证某些操作的原子性所以设置安全点，在安全点的时候才可以进行STW，常见于方法前后。安全区域：在安全点周围未发生赋值等语句则为安全区域。

## 参考资料
《深入理解Java虚拟机》-周志明