# 【优雅代码】13-linkedList插入真的比arrayList快么
> 欢迎关注b站账号/公众号【六边形战士夏宁】，一个要把各项指标拉满的男人。该文章已在[github目录](https://github.com/edanlx/SealBook/blob/master/catalogue/wechat.md)收录。
屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。
* [可直接运行的完整代码](https://github.com/edanlx/TechingCode/tree/master/demoGrace/src/main/java/com/example/demo/lesson/grace/listspeed) 
* [上一篇](./12serialize.md)序列化大对比
* [下一篇](./14localeCache.md)guava精选方法

## 1.背景
在学习中，常规的方法总是先去模仿，硬性的接收知识，等到构建起大体框架后再去探寻实际情况
## 2.addAll比较
个人认为addAll的情况出现主要源于，ArrayList走的是native的复制所以更快
```java
public static void addAllCompare(){
    List<Integer> listArray = IntStream.range(0, 10000).boxed().collect(Collectors.toList());
    List<Integer> listLinked= new LinkedList<>(listArray);

    StopWatch sw = new StopWatch();
    sw.start("arrayListAddAllArray");
    new ArrayList<>(listArray);
    sw.stop();

    sw.start("linkedListAddAllArray");
    new LinkedList<>(listArray);
    sw.stop();

    sw.start("arrayListAddAllLinked");
    new ArrayList<>(listLinked);
    sw.stop();

    sw.start("linkedListAddAllLinked");
    new LinkedList<>(listLinked);
    sw.stop();
    // 个人认为addAll的情况出现主要源于，ArrayList走的是native的复制所以更快
    System.out.println(sw.prettyPrint());
}
```
```text
000037172  002%  arrayListAddAllArray
000811717  041%  linkedListAddAllArray
000237292  012%  arrayListAddAllLinked
000913354  046%  linkedListAddAllLinked
```
## 3.循环速度比较
可以看出不同的方式循环速度上略有差异，总体上链表的确是要更快，但stream的方式数组要更快，而且三种循环，stream的速度是最慢的
```java
public static void foreachCompare() {
    List<Integer> listArray = IntStream.range(0, 10000).boxed().collect(Collectors.toList());
    ArrayList<Integer> arrays = new ArrayList<>(listArray);
    LinkedList<Integer> linked = new LinkedList<>(listArray);
    StopWatch sw = new StopWatch();
    sw.start("arraysStream");
    arrays.forEach(Integer::getClass);
    sw.stop();

    sw.start("linkedStream");
    linked.forEach(Integer::getClass);
    sw.stop();

    sw.start("arraysForEach");
    for (Integer array : arrays) {
        array.getClass();
    }
    sw.stop();
    sw.start("linkedForEach");
    for (Integer array : linked) {
        array.getClass();
    }
    sw.stop();
    sw.start("arraysIterator");
    Iterator<Integer> iteratorArray = arrays.iterator();
    while (iteratorArray.hasNext()){
        iteratorArray.next().getClass();
    }
    sw.stop();
    sw.start("arraysLinked");
    Iterator<Integer> iteratorLinked = arrays.iterator();
    while (iteratorLinked.hasNext()){
        iteratorLinked.next().getClass();
    }
    sw.stop();
    System.out.println(sw.prettyPrint());
}
```
```text
001787856  020%  arraysStream
002100260  023%  linkedStream
001649966  018%  arraysForEach
001189238  013%  linkedForEach
001229047  014%  arraysIterator
001111209  012%  arraysLinked
```
## 4.插入比较
为了避免其它差异，以同样的方式进行循环
```java
public static void addCompare() {
    List<Integer> listArray = IntStream.range(0, 10000).boxed().collect(Collectors.toList());
    // 因为链表数组没有初始大小的所以不创建
    ArrayList<Integer> arraysListHeadNon = new ArrayList<>();
    ArrayList<Integer> arraysListHead = new ArrayList<>(listArray.size());
    ArrayList<Integer> arraysListTailNon = new ArrayList<>();
    ArrayList<Integer> arraysListTailHead = new ArrayList<>(listArray.size());
    LinkedList<Integer> linkedListHeadNon = new LinkedList<>();
    LinkedList<Integer> linkedListTailNon = new LinkedList<>();
    StopWatch sw = new StopWatch();
    sw.start("arraysListHeadNon");
    listArray.stream().forEach(s -> arraysListHeadNon.add(0, s));
    sw.stop();
    sw.start("arraysListHead");
    listArray.stream().forEach(s -> arraysListHead.add(0, s));
    sw.stop();
    sw.start("arraysListTailNon");
    listArray.stream().forEach(s -> arraysListTailNon.add(s));
    sw.stop();
    sw.start("arraysListTailHead");
    listArray.stream().forEach(s -> arraysListTailHead.add(s));
    sw.stop();
    sw.start("linkedListHeadNon");
    listArray.stream().forEach(s -> linkedListHeadNon.addFirst(s));
    sw.stop();
    sw.start("linkedListTailNon");
    listArray.stream().forEach(s -> linkedListTailNon.add(s));
    sw.stop();
    System.out.println(sw.prettyPrint());
}
```
可以看到，头插的arrayList明显要慢一大截，这主要是虽然是动态数组，头插每次都要重新创建
```text
009308259  043%  arraysListHeadNon
006646659  030%  arraysListHead
001572576  007%  arraysListTailNon
001321555  006%  arraysListTailHead
001591243  007%  linkedListHeadNon
001438585  007%  linkedListTailNon

```
去掉前两组坑逼数组可以发现，如果是已经申请好空间的话，arrayList还要更快一丢丢
```text
003019400  031%  arraysListTailNon
002100608  022%  arraysListTailHead
002379269  024%  linkedListHeadNon
002248337  023%  linkedListTailNon
```