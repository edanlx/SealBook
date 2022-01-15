# 【优雅代码】16-guava布隆过滤源码解析
> 欢迎关注b站账号/公众号【六边形战士夏宁】，一个要把各项指标拉满的男人。该文章已在[github目录](https://github.com/edanlx/SealBook/blob/master/catalogue/wechat.md)收录。
屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。
* [可直接运行的完整代码](https://github.com/edanlx/TechingCode/tree/master/demoGrace/src/main/java/com/example/demo/lesson/grace/guava)
* [上一篇](https://github.com/edanlx/SealBook/blob/master/01graceCode/15localeCache.md)guavaCache本地缓存使用及源码解析
* [下一篇](https://github.com/edanlx/SealBook/blob/master/01graceCode/17rate.md)guava限流源码解析

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

### 2.3布隆算法的状态压缩
```java
boolean set(long bitIndex) {
  if (get(bitIndex)) {
    return false;
  }
  // 注意此处long转int有精度丢失导致不同的bitIndex落在同一个longIndex上
  int longIndex = (int) (bitIndex >>> LONG_ADDRESSABLE_BITS);
  long mask = 1L << bitIndex; // only cares about low 6 bits of bitIndex

  long oldValue;
  long newValue;
  do {
    // 此时第一个数进来假设mask是00100，而此时oldValue是00000，则得到newValue是00100
    // 此时第二个数进来假设mask是01000，而此时oldValue是00100，则得到newValue是01100
    oldValue = data.get(longIndex);
    newValue = oldValue | mask;
    if (oldValue == newValue) {
      return false;
    }
  } while (!data.compareAndSet(longIndex, oldValue, newValue));

  // We turned the bit on, so increment bitCount.
  bitCount.increment();
  return true;
}
boolean get(long bitIndex) {
    // 此时第一个数进来假设mask是00100，而此时数据内容是01100，则得到00100,!=0返回true,代表能拿到值
    // 此时第二个数进来假设mask是00010，而此时数据内容是01100，则得到00000,==0返回false,代表拿不到值
  return (data.get((int) (bitIndex >>> LONG_ADDRESSABLE_BITS)) & (1L << bitIndex)) != 0;
}
```