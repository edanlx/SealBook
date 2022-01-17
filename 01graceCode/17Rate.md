# 【优雅代码】17-guava限流源码解析
> 欢迎关注b站账号/公众号【六边形战士夏宁】，一个要把各项指标拉满的男人。该文章已在[github目录](https://github.com/edanlx/SealBook/blob/master/catalogue/wechat.md)收录。
屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。
* [可直接运行的完整代码](https://github.com/edanlx/TechingCode/tree/master/demoGrace/src/main/java/com/example/demo/lesson/grace/guava)
* [上一篇](https://github.com/edanlx/SealBook/blob/master/01graceCode/16bloom.md)guava布隆过滤与限流算法源码解析
* [下一篇](https://github.com/edanlx/SealBook/blob/master/01graceCode/18treelist.md)利用function实现list、tree互转工具
## 1.背景
承接前一篇章的guava精选方法

## 2.限流
这部分和其它限流算法的令牌桶算法基本一致
### 2.1使用
```java
@SneakyThrows
public static void rate() {
    // 这里直接设置的就QPS(每秒查询率)
    RateLimiter rateLimiter = RateLimiter.create(1);
    while (true){
        System.out.println(rateLimiter.tryAcquire());
        Thread.sleep(300);
    }
}
```
```text
true
false
false
false
true
```
### 2.2核心源码
1. 构造方法
```java
// 主流程方法,保留了部分说明注释，这里创建了一个计时器后面用于计算时间的
public static RateLimiter create(double permitsPerSecond) {
    /*
     *
     * T0 at 0 seconds
     * T1 at 1.05 seconds
     * T2 at 2 seconds
     * T3 at 3 seconds
     *
     * Due to the slight delay of T1, T2 would have to sleep till 2.05 seconds, and T3 would also
     * have to sleep till 3.05 seconds.
     */
    return create(permitsPerSecond, SleepingStopwatch.createFromSystemTimer());
  }
// 继续主流程往下走，然后默认构造了平均的令牌桶限流策略
static RateLimiter create(double permitsPerSecond, SleepingStopwatch stopwatch) {
    RateLimiter rateLimiter = new SmoothBursty(stopwatch, 1.0 /* maxBurstSeconds */);
    // 该方法标记为主流程方法2，设置速率，非常重要的方法
    rateLimiter.setRate(permitsPerSecond);
    return rateLimiter;
  }
// 该方法标记为主流程方法2
public final void setRate(double permitsPerSecond) {
    synchronized (mutex()) {
    	// 标记为方法主流程方法3，后面那个参数就是获取当前时间的
      doSetRate(permitsPerSecond, stopwatch.readMicros());
    }
  }
// 该标记为方法主流程方法3,注意会有多个doSetRate，但是传参不一样
final void doSetRate(double permitsPerSecond, long nowMicros) {
	// 用于刷新当前令牌数，在构造方法不重要，获取令牌时会再次调用
    resync(nowMicros);
    // ****中间这个赋值很重要***，其它地方会在调用这个结果，这里本身传入的是QPS
    // 即原先为每秒可以获取X个令牌，通过下列公式获得，每微妙可获得的令牌数,也可以从字面意思理解，获得令牌的微秒间隔
    double stableIntervalMicros = SECONDS.toMicros(1L) / permitsPerSecond;
    this.stableIntervalMicros = stableIntervalMicros;
    // 下面这个方法不重要
    doSetRate(permitsPerSecond, stableIntervalMicros);
  }
```
2. 获取令牌
```java
public boolean tryAcquire() {
	// 直接不等待，走快速失败的逻辑
    return tryAcquire(1, 0, MICROSECONDS);
  }

public boolean tryAcquire(int permits, long timeout, TimeUnit unit) {
    long timeoutMicros = max(unit.toMicros(timeout), 0);
    checkPermits(permits);
    long microsToWait;
    // 此处并发控制，每次只进入一个
    synchronized (mutex()) {
    	// 获取当前时间
      long nowMicros = stopwatch.readMicros();
      	// 计算有没有可用令牌，根据nextFreeTicketMicros
      if (!canAcquire(nowMicros, timeoutMicros)) {
        return false;
      } else {
      	// 获取令牌并返回等待时间为0，进入该方法，主流程2
        microsToWait = reserveAndGetWaitLength(permits, nowMicros);
      }
    }
    stopwatch.sleepMicrosUninterruptibly(microsToWait);
    return true;
  }
 // 主流程2从上一步进入
final long reserveAndGetWaitLength(int permits, long nowMicros) {
	// 进入该方法，主流程3
    long momentAvailable = reserveEarliestAvailable(permits, nowMicros);
    // 因为now就是最大的，所以max是0
    return max(momentAvailable - nowMicros, 0);
  }
 // 主流程3从上一步进入。这里涉及令牌桶的核心算法
@Override
  final long reserveEarliestAvailable(int requiredPermits, long nowMicros) {
  	// 主流程4
    resync(nowMicros);
    long returnValue = nextFreeTicketMicros;
    // 返回要消耗掉的令牌数，这里需要先看主流程4的逻辑，如果够消耗则返回1，不够消耗只有零点几
    double storedPermitsToSpend = min(requiredPermits, this.storedPermits);
    // 这里有两种情况，如果令牌有余额，则1-1=0，如果令牌不足则获得一个小于1的数
    double freshPermits = requiredPermits - storedPermitsToSpend;
    // 这里左边固定是0，就不贴源码了
    // 根据上面的情况，这里有可能是0即有余令牌，那么再往下走两行，下次刷新时间则不动。如果小于1则令牌不够，将差额和间隔进行相乘获得还需要下个令牌的时间
    long waitMicros =
        storedPermitsToWaitTime(this.storedPermits, storedPermitsToSpend)
            + (long) (freshPermits * stableIntervalMicros);
     // 这里赋值下次的时间，其实就是把两个参数加起来，然后就得到了下次的时间，这也是为什么示例中的2.05的下次就是3.05
    this.nextFreeTicketMicros = LongMath.saturatedAdd(nextFreeTicketMicros, waitMicros);
    // 这里减掉消耗掉的令牌数
    this.storedPermits -= storedPermitsToSpend;
    return returnValue;
  }

// 主流程4
 void resync(long nowMicros) {
    // if nextFreeTicket is in the past, resync to now
    // 如果当前时间大于下次获取时间则进入，正常情况是一定大于的，否则就没有多余的令牌可以发
    if (nowMicros > nextFreeTicketMicros) {
    	// coolDownIntervalMicros()这个就是返回stableIntervalMicros
    	// 前面存的这玩意儿现在再除回去进行还原，即可以获得的令牌数,注意这里是可以返回零点几的
      double newPermits = (nowMicros - nextFreeTicketMicros) / coolDownIntervalMicros();
      // 相加得到现在的令牌数，根据上一行，这里通常是几点几
      storedPermits = min(maxPermits, storedPermits + newPermits);
      // 标记获取令牌时间
      nextFreeTicketMicros = nowMicros;
    }
  }
```