# 【优雅代码】16-guava布隆过滤与限流算法
> 欢迎关注b站账号/公众号【六边形战士夏宁】，一个要把各项指标拉满的男人。该文章已在[github目录](https://github.com/edanlx/SealBook/blob/master/catalogue/wechat.md)收录。
屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。
* [可直接运行的完整代码](https://github.com/edanlx/TechingCode/tree/master/demoGrace/src/main/java/com/example/demo/lesson/grace/guava)
* [上一篇](./15localeCache.md)不用部署中间件的本地缓存
* [下一章](./17generic.md)双当泛型遇上多态

## 1.背景
承接前一篇章的guava精选方法
## 2.布隆过滤
这部分和redis的BitMap思路基本一致
### 2.1使用
```java
public static void hash() {
    // 存储格式，大小，误报率(如果判断出来不存在则一定不存在，如果判断出来存在则有可能存在有可能不存在，因为其机制和hash非常相似存在多个值对一个hash的情况)
    // 一般会根据订单号，如果不行的话可以使用byte数组
    BloomFilter<Integer> bloomFilter = BloomFilter.create(Funnels.integerFunnel(), 2000, 0.0001);
    IntStream.range(0, 10000).forEach(bloomFilter::put);
    System.out.println(bloomFilter.mightContain(1));
    // 只有-10、-7、-5、-2,因为这里的预期数据与实际相差比较大，所以布隆过滤并不完全
    IntStream.range(-10, 0).forEach(s -> {
        if (!bloomFilter.mightContain(s)) {
            System.out.println(s);
        }
    });
}
```
可以看到有部分已经被过滤掉了
```text
true
-10
-7
-5
-2
```
### 2.2核心源码
1. 构造参数流程
```java
// 这里主流程方法自动选择了策略BloomFilterStrategies.MURMUR128_MITZ_64
public static <T> BloomFilter<T> create(
      Funnel<? super T> funnel, long expectedInsertions, double fpp) {
    return create(funnel, expectedInsertions, fpp, BloomFilterStrategies.MURMUR128_MITZ_64);
 }
// 主流程方法1
static <T> BloomFilter<T> create(
      Funnel<? super T> funnel, long expectedInsertions, double fpp, Strategy strategy) {
    // ....省略中间的各种校验
    // 这里numBits即底下LockFreeBitArray位数组的长度，可以看到计算方式就是外部传入的期待数和容错率
    // 该方法标记为1-1
    long numBits = optimalNumOfBits(expectedInsertions, fpp);
    // 该方法标记为1-2
    int numHashFunctions = optimalNumOfHashFunctions(expectedInsertions, numBits);
    try {
    	// 主流程方法2
      return new BloomFilter<T>(new LockFreeBitArray(numBits), numHashFunctions, funnel, strategy);
    } catch (IllegalArgumentException e) {
      throw new IllegalArgumentException("Could not create BloomFilter of " + numBits + " bits", e);
    }
  }
// 方法1-1
static long optimalNumOfBits(long n, double p) {
    if (p == 0) {
      p = Double.MIN_VALUE;
    }
    // Math.log是求e为底数的对数，我们传入的p在0-1之间所以所得结果必然为负数
    return (long) (-n * Math.log(p) / (Math.log(2) * Math.log(2)));
  }
 // 方法1-2,后续的循环次数
static int optimalNumOfHashFunctions(long n, long m) {
    // (m / n) * log(2), but avoid truncation due to division!
    return Math.max(1, (int) Math.round((double) m / n * Math.log(2)));
  } 
// 主流程方法2
private BloomFilter(
      LockFreeBitArray bits, int numHashFunctions, Funnel<? super T> funnel, Strategy strategy) {
    checkArgument(numHashFunctions > 0, "numHashFunctions (%s) must be > 0", numHashFunctions);
    checkArgument(
        numHashFunctions <= 255, "numHashFunctions (%s) must be <= 255", numHashFunctions);
    this.bits = checkNotNull(bits);
    this.numHashFunctions = numHashFunctions;
    this.funnel = checkNotNull(funnel);
    this.strategy = checkNotNull(strategy);
  }
```
2. put流程
```java
// 主流程进入可以看到将之前构造参数四个值全都拿过来了
public boolean put(T object) {
    return strategy.put(object, funnel, numHashFunctions, bits);
  }
// 接着找到构造参数策略中的put方法
public <T> boolean put(T object, Funnel<? super T> funnel, int numHashFunctions, BloomFilterStrategies.LockFreeBitArray bits) {
    long bitSize = bits.bitSize();
    // 进行hash
    byte[] bytes = Hashing.murmur3_128().hashObject(object, funnel).getBytesInternal();
    // 获得低位的长度进行拼接
    long hash1 = this.lowerEight(bytes);
    // 获得高位的长度进行拼接
    long hash2 = this.upperEight(bytes);
    boolean bitsChanged = false;
    long combinedHash = hash1;

    // 构造方法中计算的循环次数在这里进行循环标记，以保证尽可能不与其它数重复
    for(int i = 0; i < numHashFunctions; ++i) {
    	// 将计算得到的位置进行标记
        bitsChanged |= bits.set((combinedHash & 9223372036854775807L) % bitSize);
        combinedHash += hash2;
    }

    return bitsChanged;
}
private long lowerEight(byte[] bytes) {
            return Longs.fromBytes(bytes[7], bytes[6], bytes[5], bytes[4], bytes[3], bytes[2], bytes[1], bytes[0]);
        }

private long upperEight(byte[] bytes) {
    return Longs.fromBytes(bytes[15], bytes[14], bytes[13], bytes[12], bytes[11], bytes[10], bytes[9], bytes[8]);
}
```
3. 校验流程
```java
// 和put流程基本一致
public <T> boolean mightContain(T object, Funnel<? super T> funnel, int numHashFunctions, BloomFilterStrategies.LockFreeBitArray bits) {
    long bitSize = bits.bitSize();
    byte[] bytes = Hashing.murmur3_128().hashObject(object, funnel).getBytesInternal();
    long hash1 = this.lowerEight(bytes);
    long hash2 = this.upperEight(bytes);
    long combinedHash = hash1;

    for(int i = 0; i < numHashFunctions; ++i) {
        if (!bits.get((combinedHash & 9223372036854775807L) % bitSize)) {
            return false;
        }

        combinedHash += hash2;
    }

    return true;
}
```

## 3.限流
这部分和其它限流算法的令牌桶算法基本一致
### 3.1使用
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
### 3.2核心源码
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
// 该标记为方法主流程方法3
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