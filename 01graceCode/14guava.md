# 【优雅代码】14guava精选方法
> 欢迎关注b站账号/公众号【六边形战士夏宁】，一个要把各项指标拉满的男人。该文章已在[github目录](https://github.com/edanlx/SealBook)收录。
屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。
* [公众号目录](https://gitee.com/seal_li/SealBook/blob/master/catalogue/wechat.md)
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
```java
public static void newCollections(){
    // 这里只介绍BiMap和Table。Multiset、Multimap这两个就是嵌了list进去
    BiMap<Integer,String> biMap=HashBiMap.create();
    biMap.put(1,"张三");
    biMap.put(2,"李四");
    biMap.put(3,"王五");
    biMap.put(4,"赵六");
    biMap.put(5,"李七");
    biMap.put(6,"小小");
    Integer result = biMap.inverse().get("赵六");
    // 输出结果4
    System.out.println(result);
    // ===========================================================
    // table是个很有意思的数据结构，很有启发性思维，虽然我也不知道这个有啥用
    /*
     *  Company: IBM, Microsoft, TCS
     *  IBM         -> {101:Mahesh, 102:Ramesh, 103:Suresh}
     *  Microsoft     -> {101:Sohan, 102:Mohan, 103:Rohan }
     *  TCS         -> {101:Ram, 102: Shyam, 103: Sunil }
     *
     * */
    //create a table
    Table<String, String, String> employeeTable = HashBasedTable.create();

    //initialize the table with employee details
    employeeTable.put("IBM", "101","Mahesh");
    employeeTable.put("IBM", "102","Ramesh");
    employeeTable.put("IBM", "103","Suresh");

    employeeTable.put("Microsoft", "111","Sohan");
    employeeTable.put("Microsoft", "112","Mohan");
    employeeTable.put("Microsoft", "113","Rohan");

    employeeTable.put("TCS", "121","Ram");
    employeeTable.put("TCS", "102","Shyam");
    employeeTable.put("TCS", "123","Sunil");

    //所有行数据
    System.out.println(employeeTable.cellSet());
    //所有公司
    System.out.println(employeeTable.rowKeySet());
    //所有员工编号
    System.out.println(employeeTable.columnKeySet());
    //所有员工名称
    System.out.println(employeeTable.values());
    //公司中的所有员工和员工编号
    System.out.println(employeeTable.rowMap());
    //员工编号对应的公司和员工名称
    System.out.println(employeeTable.columnMap());
}
```
输出如下
```text
4
[(IBM,101)=Mahesh, (IBM,102)=Ramesh, (IBM,103)=Suresh, (Microsoft,111)=Sohan, (Microsoft,112)=Mohan, (Microsoft,113)=Rohan, (TCS,121)=Ram, (TCS,102)=Shyam, (TCS,123)=Sunil]
[IBM, Microsoft, TCS]
[101, 102, 103, 111, 112, 113, 121, 123]
[Mahesh, Ramesh, Suresh, Sohan, Mohan, Rohan, Ram, Shyam, Sunil]
{IBM={101=Mahesh, 102=Ramesh, 103=Suresh}, Microsoft={111=Sohan, 112=Mohan, 113=Rohan}, TCS={121=Ram, 102=Shyam, 123=Sunil}}
{101={IBM=Mahesh}, 102={IBM=Ramesh, TCS=Shyam}, 103={IBM=Suresh}, 111={Microsoft=Sohan}, 112={Microsoft=Mohan}, 113={Microsoft=Rohan}, 121={TCS=Ram}, 123={TCS=Sunil}}
```
### 3.3集合工具类
```java
public static void Collections() {
    // 个人觉得比较又用的有如下几个方法，前几个可以看做是redis交集、并集的内存实现。后面是数据库笛卡尔积的内存实现
    // 交集、并集、
    Set<Integer> set1 = Stream.of(1, 2, 3, 4).collect(Collectors.toSet());
    Set<Integer> set2 = Stream.of(3, 4, 5, 6).collect(Collectors.toSet());
    // 求set1的差集
    System.out.println(Sets.difference(set1, set2));
    // 求set1和set2差集的并集
    System.out.println(Sets.symmetricDifference(set1, set2));
    // 求交集
    System.out.println(Sets.intersection(set1, set2));
    // 笛卡尔积
    System.out.println(Sets.cartesianProduct(set1, set2));

    // 笛卡尔积
    List<Integer> list1 = Stream.of(1, 2, 3, 4).collect(Collectors.toList());
    List<Integer> list2 = Stream.of(2, 3, 4, 5).collect(Collectors.toList());
    System.out.println(Lists.cartesianProduct(list1, list2));
}
```
输出如下
```text
[1, 2]
[1, 2, 5, 6]
[3, 4]
[[1, 3], [1, 4], [1, 5], [1, 6], [2, 3], [2, 4], [2, 5], [2, 6], [3, 3], [3, 4], [3, 5], [3, 6], [4, 3], [4, 4], [4, 5], [4, 6]]
```

## 4.缓存(重要)
该部分单独在下一篇分享。
## 5.函数式风格Functional idioms
同2一样，在stream的强力作用下不怎么用了
## 6.并发Concurrency
Futrue部分已经有了CompletableFuture(在第4节thread有介绍)，非常好用。Service部分功能是很强，但这东西一般情况是真的用不上。
限流部分在下下篇分享
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
该部分单独在下下一篇分享。
## 12.事件总线EventBus(重要)
超级容易的观察者模式，有用到的可以用这东西，写起来舒心不少。这一块的设计思想个人还是非常喜欢的，觉得有必要展开一下。需要注意的是@Subscribe才会被通知，依赖的是方法参数和投递的对象参数一致。
### 12.1代码使用
1. 创建被观察者
```java
public static class eventBusObject {
    @Subscribe
    public void listenStr1(String str) {
        System.out.println(str + "listenStr1");
    }

    @Subscribe
    public void listenStr2(String str) {
        System.out.println(str + "listenStr2");
    }

    @Subscribe
    public void listenObj(Object str) {
        System.out.println(str + "listenStr1");
    }

    @Subscribe
    public void listenInt1(Integer str) {
        System.out.println(str + "listenInt1");
    }

    public void listenInt2(Integer str) {
        System.out.println(str + "listenInt2");
    }
}
```
2. 方法通知
```java
public static void eventBus() {
    EventBus eventBus = new EventBus("eventBusTest");
    eventBus.register(new eventBusObject());
    eventBus.post(100);
    eventBus.post("我是字符串");
}
```
3. 输出结果
```text
100listenInt1
100listenStr1
我是字符串listenStr1
我是字符串listenStr2
我是字符串listenStr1
```

### 12.2核心源码
1. 注册
```java
// 点击进入register方法如下
public void register(Object object) {
	subscribers.register(object);
}
// 再点击进入register方法如下,这里主要是反射，然后就存起来
void register(Object listener) {
	// 核心方法反射获取@Subscribe该注解的方法，并将传入的class进行分类
    Multimap<Class<?>, Subscriber> listenerMethods = findAllSubscribers(listener);

    for (Entry<Class<?>, Collection<Subscriber>> entry : listenerMethods.asMap().entrySet()) {
      Class<?> eventType = entry.getKey();
      Collection<Subscriber> eventMethodsInListener = entry.getValue();

      CopyOnWriteArraySet<Subscriber> eventSubscribers = subscribers.get(eventType);

      if (eventSubscribers == null) {
        CopyOnWriteArraySet<Subscriber> newSet = new CopyOnWriteArraySet<>();
        eventSubscribers =
            MoreObjects.firstNonNull(subscribers.putIfAbsent(eventType, newSet), newSet);
      }

      eventSubscribers.addAll(eventMethodsInListener);
    }
  }
```

2. 调用
```java
// 主方法调用post
public void post(Object event) {
	// 该方法反符合event的迭代集合
    Iterator<Subscriber> eventSubscribers = subscribers.getSubscribers(event);
    if (eventSubscribers.hasNext()) {
      dispatcher.dispatch(event, eventSubscribers);
    } else if (!(event instanceof DeadEvent)) {
      // the event had no subscribers and was not itself a DeadEvent
      post(new DeadEvent(this, event));
    }
  }

// 主方法调用getSubscribers方法
Iterator<Subscriber> getSubscribers(Object event) {
	// 该方法返回event所有的父级对象，最上级即为Object，下面就是取两者交集进行组装
    ImmutableSet<Class<?>> eventTypes = flattenHierarchy(event.getClass());

    List<Iterator<Subscriber>> subscriberIterators =
        Lists.newArrayListWithCapacity(eventTypes.size());

    for (Class<?> eventType : eventTypes) {
      CopyOnWriteArraySet<Subscriber> eventSubscribers = subscribers.get(eventType);
      if (eventSubscribers != null) {
        // eager no-copy snapshot
        subscriberIterators.add(eventSubscribers.iterator());
      }
    }

    return Iterators.concat(subscriberIterators.iterator());
  } 
// 回到主方法dispatcher.dispatch进行调用，该方法标记为2.1
@Override
void dispatch(Object event, Iterator<Subscriber> subscribers) {
  checkNotNull(event);
  checkNotNull(subscribers);
  Queue<Event> queueForThread = queue.get();
  queueForThread.offer(new Event(event, subscribers));

  if (!dispatching.get()) {
    dispatching.set(true);
    try {
      Event nextEvent;
      while ((nextEvent = queueForThread.poll()) != null) {
        while (nextEvent.subscribers.hasNext()) {
        	//  核心方法调用，标记为2.1.1
          nextEvent.subscribers.next().dispatchEvent(nextEvent.event);
        }
      }
    } finally {
      dispatching.remove();
      queue.remove();
    }
  }
}
// 从2.1.1这里就可以看到把任务直接丢到线程池
final void dispatchEvent(final Object event) {
    executor.execute(
        new Runnable() {
          @Override
          public void run() {
            try {
              invokeSubscriberMethod(event);
            } catch (InvocationTargetException e) {
              bus.handleSubscriberException(e.getCause(), context(event));
            }
          }
        });
  }
```

## 13.数学运算Math
一般情况用不到，apache下也有Math的包，功能基本一致
## 14.反射Reflection(次重要)
spring里面也带了这块的工具类，个人感觉spring的在处理常规情况下已经非常优秀了，这边补充了泛型方向的，用起来也比较简单，不过鉴于平常使用的机会不多就不展开了。