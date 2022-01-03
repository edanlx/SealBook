# 【优雅代码】13-linkedList插入真的比arrayList快么
> 欢迎关注b站账号/公众号【六边形战士夏宁】，一个要把各项指标拉满的男人。该文章已在[github目录](https://github.com/edanlx/SealBook/blob/master/catalogue/wechat.md)收录。
屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。
* [可直接运行的完整代码](https://github.com/edanlx/TechingCode/tree/master/demoGrace/src/main/java/com/example/demo/lesson/grace/listspeed) 
* [上一篇](./12serialize.md)hessian、kryo、json序列化大对比
* [下一篇](./14localeCache.md)guava精选方法

## 1.背景
在学习中，常规的方法总是先去模仿，硬性的接收知识，但是实际情况往往出人意料，等到构建起大体框架后再去探寻实际情况

## 1.插入比较
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
    sw.start("不带初始化大小arrayList的尾插");
    listArray.stream().forEach(s -> arraysListTailNon.add(s));
    sw.stop();
    sw.start("带初始化大小arrayList的尾插");
    listArray.stream().forEach(s -> arraysListTailHead.add(s));
    sw.stop();

    sw.start("链表头插");
    listArray.stream().forEach(s -> linkedListHeadNon.addFirst(s));
    sw.stop();
    sw.start("链表尾插");
    listArray.stream().forEach(s -> linkedListTailNon.add(s));
    sw.stop();
    System.out.println(sw.prettyPrint());
}
```
可以看到，头插的arrayList太慢了直接去掉，然后发现带初始化的arrayList还是要快一点
```text
001411100  035%  不带初始化大小arrayList的尾插
000776200  020%  带初始化大小arrayList的尾插
000952100  024%  链表头插
000840500  021%  链表尾插

```
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
可以看出不同的方式循环速度上略有差异，总体上链表的确是要更快，但随着数据量的增加arrayStream的方式优势越来越明显最终最快。  
而在循环方式上迭代器基本都是最优的
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
```1000000的输出情况
006974600  012%  arraysStream
011748900  019%  linkedStream
015315300  025%  arraysForEach
009445800  016%  linkedForEach
008187400  014%  arraysIterator
008792100  015%  arraysLinked
```
10000的输出情况
```text
001787856  020%  arraysStream
002100260  023%  linkedStream
001649966  018%  arraysForEach
001189238  013%  linkedForEach
001229047  014%  arraysIterator
001111209  012%  arraysLinked
```
100的输出情况
```text
000372500  050%  arraysStream
000265300  035%  linkedStream
000037700  005%  arraysForEach
000027500  004%  linkedForEach
000024800  003%  arraysIterator
000023100  003%  arraysLinked
```
