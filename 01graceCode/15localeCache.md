# 【优雅代码】15-不用部署中间件的本地缓存
> 欢迎关注b站账号/公众号【六边形战士夏宁】，一个要把各项指标拉满的男人。该文章已在[github目录](https://github.com/edanlx/SealBook/blob/master/catalogue/wechat.md)收录。
屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。
* [可直接运行的完整代码](https://github.com/edanlx/TechingCode/tree/master/demoGrace/src/main/java/com/example/demo/lesson/grace/guava)
* [上一篇](./14guava.md)guava精选方法
* [下一篇](./16bloomAndRate.md)guava布隆过滤与限流算法

## 1.背景
承接前一篇章的guava精选方法
## 2.cache
这一块的功能设计很巧的是和redis也很像
### 2.1使用
```java
@SneakyThrows
public static void cache() {
    // 注意两个如果一起用有时候会有bug
    Cache<Integer, Integer> accessBuild = CacheBuilder.newBuilder().expireAfterAccess(1, TimeUnit.SECONDS).build();
    Cache<Integer, Integer> writeBuild = CacheBuilder.newBuilder().expireAfterWrite(1, TimeUnit.SECONDS).build();

    accessBuild.put(1, 1);
    accessBuild.put(2, 2);
    writeBuild.put(1, 1);
    writeBuild.put(2, 2);
    // 输出1
    System.out.println(accessBuild.getIfPresent(1));
    // 输出1
    System.out.println(writeBuild.getIfPresent(1));
    Thread.sleep(500);
    // 输出2
    System.out.println(accessBuild.getIfPresent(2));
    Thread.sleep(600);
    // 输出null
    System.out.println(accessBuild.getIfPresent(1));
    // 输出2
    System.out.println(accessBuild.getIfPresent(2));
    // 输出null
    System.out.println(writeBuild.getIfPresent(1));
}
```
输出如下
```text
1
1
2
null
2
null
```
### 2.2核心源码详解
1. 构造方法
```java
// 整体构造链相对简单
// build方法
public <K1 extends K, V1 extends V> Cache<K1, V1> build() {
    checkWeightWithWeigher();
    checkNonLoadingCache();
    return new LocalCache.LocalManualCache<>(this);
  }
// 给expireAfterAccessNanos赋值失效时间
public CacheBuilder<K, V> expireAfterAccess(long duration, TimeUnit unit) {
    checkState(
        expireAfterAccessNanos == UNSET_INT,
        "expireAfterAccess was already set to %s ns",
        expireAfterAccessNanos);
    checkArgument(duration >= 0, "duration cannot be negative: %s %s", duration, unit);
    this.expireAfterAccessNanos = unit.toNanos(duration);
    return this;
  }
// 给expireAfterWriteNanos赋值失效时间
public CacheBuilder<K, V> expireAfterWrite(long duration, TimeUnit unit) {
    checkState(
        expireAfterWriteNanos == UNSET_INT,
        "expireAfterWrite was already set to %s ns",
        expireAfterWriteNanos);
    checkArgument(duration >= 0, "duration cannot be negative: %s %s", duration, unit);
    this.expireAfterWriteNanos = unit.toNanos(duration);
    return this;
  }
```
2. put
```java
// put方法，和老版本的ConcurrentHashMap一样的设计模式，用segment桶
@Override
  public V put(K key, V value) {
    checkNotNull(key);
    checkNotNull(value);
    int hash = hash(key);
    return segmentFor(hash).put(key, hash, value, false);
  }

V put(K key, int hash, V value, boolean onlyIfAbsent) {
  lock();
  try {
    long now = map.ticker.read();
    // **********重点方法:清除过期内容********
    preWriteCleanup(now);

    int newCount = this.count + 1;
    if (newCount > this.threshold) { // ensure capacity
      expand();
      newCount = this.count + 1;
    }

    AtomicReferenceArray<ReferenceEntry<K, V>> table = this.table;
    int index = hash & (table.length() - 1);
    ReferenceEntry<K, V> first = table.get(index);

    // Look for an existing entry.
    // 这里判断一下是不是已经有同样的key了
    for (ReferenceEntry<K, V> e = first; e != null; e = e.getNext()) {
      K entryKey = e.getKey();
      if (e.getHash() == hash
          && entryKey != null
          && map.keyEquivalence.equivalent(key, entryKey)) {
        // We found an existing entry.

        ValueReference<K, V> valueReference = e.getValueReference();
        V entryValue = valueReference.get();

        if (entryValue == null) {
          ++modCount;
          if (valueReference.isActive()) {
            enqueueNotification(
                key, hash, entryValue, valueReference.getWeight(), RemovalCause.COLLECTED);
            setValue(e, key, value, now);
            newCount = this.count; // count remains unchanged
          } else {
            setValue(e, key, value, now);
            newCount = this.count + 1;
          }
          this.count = newCount; // write-volatile
          evictEntries(e);
          return null;
        } else if (onlyIfAbsent) {
          // Mimic
          // "if (!map.containsKey(key)) ...
          // else return map.get(key);
          recordLockedRead(e, now);
          return entryValue;
        } else {
          // clobber existing entry, count remains unchanged
          ++modCount;
          enqueueNotification(
              key, hash, entryValue, valueReference.getWeight(), RemovalCause.REPLACED);
          setValue(e, key, value, now);
          evictEntries(e);
          return entryValue;
        }
      }
    }

    // Create a new entry.
    ++modCount;
    ReferenceEntry<K, V> newEntry = newEntry(key, hash, first);
    // **********重点方法:赋值********
    setValue(newEntry, key, value, now);
    table.set(index, newEntry);
    newCount = this.count + 1;
    this.count = newCount; // write-volatile
    evictEntries(newEntry);
    return null;
  } finally {
    unlock();
     // **********重点方法:调用监听者********
    postWriteCleanup();
  }
}
```
3. setValue
```java
@GuardedBy("this")
void setValue(ReferenceEntry<K, V> entry, K key, V value, long now) {
    // 获取这个包装entry原先的值，如果原先这个key不存在，则获取不到东西
  ValueReference<K, V> previous = entry.getValueReference();
  int weight = map.weigher.weigh(key, value);
  checkState(weight >= 0, "Weights must be non-negative");

  ValueReference<K, V> valueReference =
      map.valueStrength.referenceValue(this, entry, value, weight);
    // 将value写入到entry包装对象中
  entry.setValueReference(valueReference);
  // 核心方法
  recordWrite(entry, weight, now);
  previous.notifyNewValue(value);
}
@GuardedBy("this")
void recordWrite(ReferenceEntry<K, V> entry, int weight, long now) {
  // we are already under lock, so drain the recency queue immediately
  drainRecencyQueue();
  totalWeight += weight;

  if (map.recordsAccess()) {
    // 设置访问时间
    entry.setAccessTime(now);
  }
  if (map.recordsWrite()) {
    // 设置写时间
    entry.setWriteTime(now);
  }
  // **************重点方法***************这个地方将指针存了两份队列到末尾，因为缓存时间是一致的，所以只要判断队列头部就可以了
  accessQueue.add(entry);
  writeQueue.add(entry);
}
```
4. WriteQueue与accessQueue
这两个用的实现类基本一致，这里重写了add方法，重写的目的是如果key一样的entry就进行重排而不是插入
```java
@Override
    public boolean offer(ReferenceEntry<K, V> entry) {
      // unlink
      // 将entry的前一个和后一个进行互指
      connectWriteOrder(entry.getPreviousInWriteQueue(), entry.getNextInWriteQueue());

      // add to tail
      // entry和对头互指，和队尾互指，即添加到队尾
      connectWriteOrder(head.getPreviousInWriteQueue(), entry);
      connectWriteOrder(entry, head);

      return true;
    }
    // 两个对象进行互指
static <K, V> void connectWriteOrder(ReferenceEntry<K, V> previous, ReferenceEntry<K, V> next) {
    previous.setNextInWriteQueue(next);
    next.setPreviousInWriteQueue(previous);
  }
```
5. preWriteCleanup
```java
void preWriteCleanup(long now) {
      runLockedCleanup(now);
    }
 void runLockedCleanup(long now) {
      if (tryLock()) {
        try {
          drainReferenceQueues();
          // 核心方法继续进入
          expireEntries(now); // calls drainRecencyQueue
          readCount.set(0);
        } finally {
          unlock();
        }
      }
    }
void expireEntries(long now) {
      drainRecencyQueue();

      ReferenceEntry<K, V> e;
      // 就两个队列不停的判断头节点是不是失效
      while ((e = writeQueue.peek()) != null && map.isExpired(e, now)) {
        if (!removeEntry(e, e.getHash(), RemovalCause.EXPIRED)) {
          throw new AssertionError();
        }
      }
      while ((e = accessQueue.peek()) != null && map.isExpired(e, now)) {
        if (!removeEntry(e, e.getHash(), RemovalCause.EXPIRED)) {
          throw new AssertionError();
        }
      }
    }
```
6. 监听者模式
```java
void postWriteCleanup() {
    // 核心方法继续进入
      runUnlockedCleanup();
    }

void runUnlockedCleanup() {
      // locked cleanup may generate notifications we can send unlocked
      if (!isHeldByCurrentThread()) {
        // 核心方法继续进入
        map.processPendingNotifications();
      }
    }
void processPendingNotifications() {
    RemovalNotification<K, V> notification;
    // 前面所有涉及notify的操作都会进到相应的queue中，然后在该主方法中进行回调
    while ((notification = removalNotificationQueue.poll()) != null) {
      try {
        // 核心流程，只要实现removalListener，通过构造方法传进来，然后这里就会同步调用实现的回调方法
        // PS第一遍看源码一度卡在这个位置，不知道这玩意儿就一个通知机制怎么就移除元素了
        removalListener.onRemoval(notification);
      } catch (Throwable e) {
        logger.log(Level.WARNING, "Exception thrown by removal listener", e);
      }
    }
  }
```