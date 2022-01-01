# 【优雅代码】04-1行代码完成多线程，别再写runnable了
> 欢迎关注b站账号/公众号【六边形战士夏宁】，一个要把各项指标拉满的男人。该文章已在[github目录](https://github.com/edanlx/SealBook/blob/master/catalogue/wechat.md)收录。
屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。
* [可直接运行的完整代码](https://github.com/edanlx/TechingCode/tree/master/demoGrace/src/main/java/com/example/demo/lesson/grace/thread)  
* [视频讲解](https://www.bilibili.com/video/BV1jr4y1w7SH/)  
* [上一篇](./03optional.md)optional杜绝空指针异常
* [下一篇](./05symbol.md)从hashMap源码介绍位运算符

## 1.背景介绍
java8提供的CompletableFuture以及匿名函数可以让我们一行代码完成多线程
## 2.建立相关类
### 2.1.ThreadEntity
用于多线程测试的实体类
```java
public class ThreadEntity {
    private int num;
    private int price;
    public int countPrice(){
        price = RandomUtils.nextInt();
        try {
            System.out.println(num);
            // 随机等待1~10秒
            Thread.sleep(RandomUtils.nextInt(1, 10) * 1000);
            System.out.println(num);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return price;
    }
}
```
### 2.2.ThreadPoolManager
```java
/**
 * tasks 每秒的任务数，默认200,依据访问量及使用线程池的地方进行计算
 * taskCost：每个任务花费时间，默认0.1s
 * responseTime：最大响应时间，默认为1s，一般用户最大忍受时间为3秒
 *
 * @author seal email:876651109@qq.com
 * @date 2020/5/30 10:08 AM
 */
@Data
@Slf4j
@Configuration
public class ThreadPoolManager {
    /**
     * 平均响应时间默认2秒
     */
    private static final float ALL_COST_AVG = 2F;
    /**
     * 平均IO时间默认1.5秒
     */
    private static final float IO_COST_AVG = 1.5F;
    /**
     * 服务器核数
     */
    private static final int SIZE_PROCESSOR = Runtime.getRuntime().availableProcessors();
    /**
     * https://www.cnblogs.com/dennyzhangdd/p/6909771.html?utm_source=itdadao&utm_medium=referral
     * 阻塞系数=阻塞时间/（阻塞时间+计算时间）
     * 线程数=核心数/(1-阻塞系数)
     * 等同于CPU核心数*cpu使用率*(1+等待时间与计算时间的比率)
     * N+1通常为最优效率
     * <p>
     * https://blog.51cto.com/13527416/2056080
     */
    private static final int SIZE_CORE_POOL = SIZE_PROCESSOR + 1;

    /**
     * 线程池维护最大数量，默认会与核心线程数一致无意义，保守情况取2cpu
     * 或者使用简单计算 线程池大小 = （（线程 IO time + 线程 CPU time ）/线程 CPU time ） CPU数目**
     * 请求所消耗的时间 /(请求所消耗的时间-DB处理)*CPU数目,重点在于cpu等待时间，通常为数据库DB时间
     * 按照通常2秒展示界面，数据库运算1.5秒则(2/0.5)*n,其实就是优化等待时间
     * <p>
     * 默认4n即8核32线程
     */
    private static final int SIZE_MAX_POOL = (int) (SIZE_PROCESSOR * (ALL_COST_AVG / (ALL_COST_AVG - IO_COST_AVG)));
    /**
     * 线程池队列长度，默认为integer最大值,Dubbo使用1000，无限大会引起用户用户的任务一直排队，应选择适当性丢弃，
     * 可忍受时间6其它的则抛弃
     * SIZE_MAX_POOL/IO_COST_AVG=每秒可处理任务数,默认为
     * 可忍受时间6*每秒可处理任务数=X队列数
     */
    private static final int SIZE_QUEUE = (int) (6 * (SIZE_MAX_POOL / IO_COST_AVG));
    /**
     * 线程池具体类
     * LinkedBlockingDeque常用于固定线程，SynchronousQueue常用于cache线程池
     * Executors.newCachedThreadPool()常用于短期任务
     * <p>
     * 线程工厂选择,区别不大
     * 有spring的CustomizableThreadFactory,new CustomizableThreadFactory("springThread-pool-")
     * guava的ThreadFactoryBuilder,new ThreadFactoryBuilder().setNameFormat("retryClient-pool-").build();
     * apache-lang的BasicThreadFactory,new BasicThreadFactory.Builder().namingPattern("basicThreadFactory-").build()
     * <p>
     * 队列满了的策略默认AbortPolicy
     */
    private static ThreadPoolManager threadPoolManager = new ThreadPoolManager();

    private final ThreadPoolExecutor pool = new ThreadPoolExecutor(
            SIZE_CORE_POOL,
            SIZE_MAX_POOL,
            30L, TimeUnit.SECONDS, new LinkedBlockingDeque<>(SIZE_QUEUE),
            new CustomizableThreadFactory("springThread-pool-"),
            new ThreadPoolExecutor.AbortPolicy()
    );

    private void prepare() {
        if (pool.isShutdown() && !pool.prestartCoreThread()) {
            int coreSize = pool.prestartAllCoreThreads();
            System.out.println("当前线程池");
        }
    }


    public static ThreadPoolManager getInstance() {
        if (threadPoolManager != null) {
            ThreadPoolExecutor pool = threadPoolManager.pool;
        }
        return threadPoolManager;
    }
}
```
## 3.核心代码
### 3.1.并行流
parallel是并行核心可以发现内部是多线程运行，但是经过collect以后会排好序所以不用担心，小项目可以使用，大项目的话建议老老实实用自己的线程池，JDK自带的fork/join并不贴合业务
```java
System.out.println(Stream.of(1, 2, 3, 4, 5, 6).parallel().map(l -> {
    System.out.println(l);
    return l;
}).collect(Collectors.toList()));
```
输出如下，因为多线程所以随机输出，但因为使用collect收集则最终结果并未发生改变
```text
2
6
4
5
3
1
[1, 2, 3, 4, 5, 6]
```
### 3.2.同步代码
这个可以不用再去实现线程的接口，不过还是要考虑一下队列满了的丢弃情况
```java
List<ThreadEntity> listEntity = IntStream.range(0, 10).mapToObj(x -> ThreadEntity.builder().num(x).build()).collect(Collectors.toList());
List<CompletableFuture<Integer>> listCompletableFuture = listEntity.stream().map(x -> {
    try {
        // 此处ThreadPoolManager.getInstance().getPool()如果不传该参数则使用默认commonPool，无特殊需求的话trycatch一般不写
        return CompletableFuture.supplyAsync(() -> x.countPrice(),
                ThreadPoolManager.getInstance().getPool());
    } catch (RejectedExecutionException e) {
        System.out.println("reject" + x);
        log.error("", e);
        return null;
    }
}).collect(Collectors.toList());
List<Integer> result = listCompletableFuture.stream().map(CompletableFuture::join).collect(Collectors.toList());
System.out.println(result);
System.out.println(listEntity);
```
输出如下可以看到运行是以多线程的方式进行，但是结果和原先是保持一致的
```text
start6
start9
start0
start3
start2
start1
start8
start5
start4
start7
end3
end8
end5
end7
end9
end1
end2
end6
end0
end4
[131523688, 1491605535, 222657954, 132274662, 1134597171, 2057763841, 1168687436, 1842194861, 1264173480, 56446450]
[ThreadEntity(super=com.example.demo.lesson.grace.thread.ThreadEntity@7d6f201, num=0, price=131523688), ThreadEntity(super=com.example.demo.lesson.grace.thread.ThreadEntity@58e825f3, num=1, price=1491605535), ThreadEntity(super=com.example.demo.lesson.grace.thread.ThreadEntity@d458bb1, num=2, price=222657954), ThreadEntity(super=com.example.demo.lesson.grace.thread.ThreadEntity@7e26830, num=3, price=132274662), ThreadEntity(super=com.example.demo.lesson.grace.thread.ThreadEntity@43a0a2b8, num=4, price=1134597171), ThreadEntity(super=com.example.demo.lesson.grace.thread.ThreadEntity@7aa70ac1, num=5, price=2057763841), ThreadEntity(super=com.example.demo.lesson.grace.thread.ThreadEntity@45a8d047, num=6, price=1168687436), ThreadEntity(super=com.example.demo.lesson.grace.thread.ThreadEntity@6dcdb8e3, num=7, price=1842194861), ThreadEntity(super=com.example.demo.lesson.grace.thread.ThreadEntity@4b59d119, num=8, price=1264173480), ThreadEntity(super=com.example.demo.lesson.grace.thread.ThreadEntity@35d5d9e, num=9, price=56446450)]
```
如果IntStream.range(0, 10)改成(0, 1000)则会有如下拒绝报错
```text
java.util.concurrent.RejectedExecutionException: Task java.util.concurrent.CompletableFuture$AsyncSupply@5af97850 rejected from java.util.concurrent.ThreadPoolExecutor@491666ad[Running, pool size = 64, active threads = 64, queued tasks = 256, completed tasks = 0]
    at java.util.concurrent.ThreadPoolExecutor$AbortPolicy.rejectedExecution(ThreadPoolExecutor.java:2063)
    at java.util.concurrent.ThreadPoolExecutor.reject(ThreadPoolExecutor.java:830)
    at java.util.concurrent.ThreadPoolExecutor.execute(ThreadPoolExecutor.java:1379)
    at java.util.concurrent.CompletableFuture.asyncSupplyStage(CompletableFuture.java:1618)
    at java.util.concurrent.CompletableFuture.supplyAsync(CompletableFuture.java:1843)
    at com.example.demo.lesson.grace.thread.TestMain.lambda$threadEx1$2(TestMain.java:34)
    at java.util.stream.ReferencePipeline$3$1.accept(ReferencePipeline.java:193)
    at java.util.ArrayList$ArrayListSpliterator.forEachRemaining(ArrayList.java:1384)
    at java.util.stream.AbstractPipeline.copyInto(AbstractPipeline.java:482)
    at java.util.stream.AbstractPipeline.wrapAndCopyInto(AbstractPipeline.java:472)
    at java.util.stream.ReduceOps$ReduceOp.evaluateSequential(ReduceOps.java:708)
    at java.util.stream.AbstractPipeline.evaluate(AbstractPipeline.java:234)
    at java.util.stream.ReferencePipeline.collect(ReferencePipeline.java:499)
    at com.example.demo.lesson.grace.thread.TestMain.threadEx1(TestMain.java:41)
    at com.example.demo.lesson.grace.thread.TestMain.main(TestMain.java:26)
rejectThreadEntity(super=com.example.demo.lesson.grace.thread.ThreadEntity@1a9, num=366)
```
### 3.3.异步代码
以下代码可以直接简写成一行，在处理异步任务变得异常方便  
CompletableFuture.runAsync(() -> fun())  

```java
List<ThreadEntity> listEntity = IntStream.range(0, 500).mapToObj(x -> ThreadEntity.builder().num(x).build()).collect(Collectors.toList());
List<CompletableFuture> listCompletableFuture = listEntity.stream().map(x -> {
    try {
        // 此处ThreadPoolManager.getInstance().getPool()如果不传该参数则使用默认commonPool，无特殊需求的话trycatch一般不写
        return CompletableFuture.runAsync(() -> x.countPrice(), ThreadPoolManager.getInstance().getPool());
    } catch (RejectedExecutionException e) {
        System.out.println("reject" + x);
        return null;
    }
}).collect(Collectors.toList());
listCompletableFuture.stream().map(CompletableFuture::join);
System.out.println("1234");
// 一行多线程异步执行写法
CompletableFuture.runAsync(() -> System.out.println(1));
```
输出如下，可以看到主线程已经结束了其它子线程才在输出，完全没有等待的多线程
```text
1234
1
start7
start0
start6
start5
start4
start2
start8
start1
start9
start3
end8
end4
end9
end6
end2
end0
end1
end3
end5
end7
```