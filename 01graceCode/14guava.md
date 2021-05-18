# 【优雅代码】14guava精选方法
> 欢迎关注b站账号/公众号【六边形战士夏宁】，一个要把各项指标拉满的男人。该文章已在[github目录](https://github.com/edanlx/SealBook)收录。
屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。
* [gitee目录](https://gitee.com/seal_li/SealBook)
* [知乎目录](https://zhuanlan.zhihu.com/p/338222208)
* [csdn目录](https://blog.csdn.net/seal_li/article/details/111415366)
* [博客园目录](https://www.cnblogs.com/sealLee/articles/14748368.html)
* [可直接运行的完整代码](https://github.com/edanlx/TechingCode/tree/master/demoGrace/src/main/java/com/example/demo/lesson/grace/guava)
* [上一篇](./13listSpeed.md)linkedList插入真的比arrayList快么
* [下一篇](./14localeCache.md)不用部署中间件的本地缓存

## 1.背景
google的guava是非常经典的工具类,但是java经过多年的发展，不少方法已经有了优秀的替代方案，以下分享是个人在日常开发的时候依然觉得非常经典的方法。
## 2.基本工具Basic utilities
因为Optional和stream的流行，这个工具类里面的方法基本都用不上了。
## 3.集合Collections(重要)
这块主要介绍guava的集合，工具类虽然是对apache的补充，但是过于花里胡哨。
### 3.1不可变集合
1. 常规用法
```java
public static void immutableOrdinary() {
    // 常规创建
    Set<Integer> set = ImmutableSet.of(1, 2, 3, 1);
    List<Integer> list = ImmutableList.of(1, 2, 3, 1);
    // map的k-v连续的写法还是非常舒服的
    Map<Integer, Integer> map = ImmutableMap.of(1, 2, 3, 1);

    // 循环创建
    ImmutableSet.Builder<Integer> builder = ImmutableSet.<Integer>builder();
    for (int i = 0; i < 10; i++) {
        builder.add(i);
    }
    Set<Integer> build = builder.build();

    // 常规list转不可变,值得注意的是Collections.unmodifiableList()
    List<Integer> collect = Stream.of(1, 2, 3, 4).collect(Collectors.toList());
    List<Integer> integers = ImmutableList.copyOf(collect);
}
```
2. 以近乎同样的方式插入性能对比
```java
public static void effectiveList() {
    StopWatch sw = new StopWatch();
    sw.start("listAdd");
    Stream.Builder<Integer> builder1 = Stream.builder();
    for (int i = 0; i < 10000; i++) {
        builder1.add(i);
    }
    List<Integer> list = builder1.build().collect(Collectors.toList());
    sw.stop();

    sw.start("ImmutableListAdd");
    ImmutableList.Builder<Integer> builder = ImmutableList.builder();
    for (int i = 0; i < 10000; i++) {
        builder.add(i);
    }
    List<Integer> guava = builder.build();
    sw.stop();
    System.out.println(sw.prettyPrint());
}
```
* 可以看到插入速度非常明显的不可变集合要更快
```text
---------------------------------------------
ns         %     Task name
---------------------------------------------
072909616  081%  listAdd
017632120  019%  ImmutableListAdd
```
3. 获取单个元素的速度对比
```java
List<Object> listUn = Collections.unmodifiableList(list);
sw.start("listGet");
list.get(0);
sw.stop();
sw.start("listUnGet");
listUn.get(0);
sw.stop();
sw.start("guavaGet");
guava.get(0);
sw.stop();
```
* list可太慢了，其它两个效率差不多,guava还是要更快一点
```text
---------------------------------------------
000018405  089%  listGet
000001323  006%  listUnGet
000000936  005%  guavaGet
```

4. 循环性能对比
```java
List<Object> listUn = Collections.unmodifiableList(list);
sw.start("listFor");
IntStream.range(0, 10000).boxed().forEach(list::get);
sw.stop();
sw.start("listUnFor");
IntStream.range(0, 10000).boxed().forEach(listUn::get);
sw.stop();
sw.start("guavaFor");
IntStream.range(0, 10000).boxed().forEach(guava::get);
sw.stop();
```
* 和上面的结果一致
```text
---------------------------------------------
007031946  053%  listFor
003760153  028%  listUnFor
002458426  019%  guavaFor
```
如上所述，这就是我非常喜欢guava的原因
5. 区别对比
```text
// 常规list转不可变,值得注意的是Collections.unmodifiableList()如果原list被改变不可变是会被改变的
list.remove(0);
// 即list和unmodifiableList都会少一个
```
### 3.2新集合类型

## 4.缓存(重要)
该部分单独在下一篇分享。
## 5.函数式风格Functional idioms
同2一样，在stream的强力作用下不怎么用了
## 6.并发Concurrency
Futrue部分已经有了CompletableFuture(在第4节thread有介绍)，非常好用。Service部分功能是很强，但这东西一般情况是真的用不上。
限流算法
## 7.字符串处理Strings
apache的也不差，这块都差不多
## 8.原生类型Primitives
功能看起来很棒，但是目前还没找到实际使用场景
## 9.区间Ranges
功能看起来很棒，但是目前还没找到实际使用场景
## 10.I/O
apache的IOUtils用起来比这个要简单
## 11.散列Hash(因业务而异)
这个里面提供的布隆过滤，在有有需要的场景的时候还是可以用的。哈希算法在处理文件一致性校验等也有一席之地。
## 12.事件总线EventBus(重要)
超级容易的观察者模式，有用到的可以用这东西，写起来舒心不少。这一块的设计思想个人还是非常喜欢的，觉得有必要展开一下。
## 13.数学运算Math
一般情况用不到，apache下也有Math的包，功能基本一致
## 14.反射Reflection(次重要)
spring里面也带了这块的工具类，个人感觉spring的在处理常规情况下已经非常优秀了，这边补充了泛型方向的，用起来也比较简单，不过鉴于平常使用的机会不多就不展开了。