# 【优雅代码】15guava布隆过滤与限流算法
> 欢迎关注b站账号/公众号【六边形战士夏宁】，一个要把各项指标拉满的男人。该文章已在[github目录](https://github.com/edanlx/SealBook)收录。
屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。
* [gitee目录](https://gitee.com/seal_li/SealBook)
* [知乎目录](https://zhuanlan.zhihu.com/p/338222208)
* [csdn目录](https://blog.csdn.net/seal_li/article/details/111415366)
* [博客园目录](https://www.cnblogs.com/sealLee/articles/14748368.html)
* [上一篇](./15localeCache.md)不用部署中间件的本地缓存
* [下一章](../02jvm/01classloader.md)双亲委派都会说，破坏双亲委派你会吗

## 1.背景
承接[guava精选方法]((./14guava.md))
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
    // 只有-10、-7、-5、-2,而上面用2000个的长度存了一万个数据，有一定的误报率，当然你如果存10000个就会全都检测出来了，但也失去了布隆过滤的意义
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