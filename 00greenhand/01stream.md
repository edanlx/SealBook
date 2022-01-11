# 【触手可及】01-stream的基础使用及基本原理
> 欢迎关注b站账号/公众号【六边形战士夏宁】，一个要把各项指标拉满的男人。该文章已在[github目录](https://github.com/edanlx/SealBook/blob/master/catalogue/wechat.md)收录。
屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。

* [可直接运行的完整代码](https://github.com/edanlx/TechingCode/tree/master/demoGrace/src/main/java/com/example/demo/lesson/greenhand/stream) 
* [视频讲解](https://www.bilibili.com/video/BV1BL4y1b7S5/)

## 1.背景介绍
stream基础向，如果已经很熟了就忽略这篇文章，可以看[stream进阶使用](https://github.com/edanlx/SealBook/blob/master/01graceCode/11stream.md)
## 2.基础概念及基础原理介绍
操作分类
        中间操作(有状态的需要临时迭代再进行下一步，无状态不需要)
            无状态(整体链表结构保证其只循环一次)：unordered()、filter()、 map()、 mapToInt() 、mapToLong() 、mapToDouble()、 flatMap() 、flatMapToInt()、 flatMapToLong()、 flatMapToDouble()、 peek()
            有状态：distinct()、 sorted() 、sorted()、 limit()、 skip()
        结束操作
            非短路操作(循环全部)：forEach() 、forEachOrdered() 、toArray()、 reduce()、 collect()、 max() 、min() 、count()
            短路操作(匹配到直接结束流)：anyMatch() 、allMatch() 、noneMatch()、findFirst()、findAny()

## 3.拉姆达表达式和匿名内部类
以相对熟悉的runnable接口为例
```java
Thread thread = new Thread(new Runnable() {
    @Override
    public void run() {
        System.out.println(1);
    }
});
```
可以简写成如下一行代码，其关键在于Runnable下只有一个方法，所以jvm可以推断出找的就是run方法
```java
Thread thread = new Thread(() -> System.out.println(1));
```
## 4.::写法和拉姆达表达式
|使用方法|对应lambda表达式|
|--|--|
|object::instanceMethod|(a,b,.....)->特定对象.实例方法(a,b....)|
|Class::staticMethod|(a,b,.....)->类名.类方法(a,b....)|
|Class::instanceMethod|(a,b,.....)->a.实例方法(b....)|
|Class::new|(a,b,.....)->new 类名(a,b....)|

### 4.1::情景1
```java
StreamExample streamExample = new StreamExample();
Stream.of("1", "2", "3", "4").forEach(s -> streamExample.test(s));
```
等同于如下方法
```java
StreamExample streamExample = new StreamExample();
Stream.of("1", "2", "3", "4").forEach(streamExample::test);
```
### 4.2::情景2
```java
Stream.of(1, 2, 3, 4).forEach(s -> StreamExample.staticTest(s));
```
等同于如下方法
```java
Stream.of(1, 2, 3, 4).forEach(StreamExample::staticTest);
```
### 4.3::情景3
```java
Stream.of(new StreamExample(), new StreamExample()).forEach(s -> s.test());
```
等同于
```java
Stream.of(new StreamExample(), new StreamExample()).forEach(StreamExample::test);
```
### 4.4::情景4
```java
Stream.of("1", "2", "3", "4").forEach(s -> new String(s));
```
等同于
```java
Stream.of("1", "2", "3", "4").forEach(String::new);
```
## 5.stream的基础写法
```java
static void mainExample() {
    List<StreamTestExample> collectStreamTestExample = IntStream.range(0, 6).mapToObj(s -> new StreamTestExample(s, String.valueOf(s))).collect(Collectors.toList());
    System.out.println("filter");
    System.out.println(Stream.of(1, 2, 3, 4, 5).filter(s -> s > 3).count());
    System.out.println(collectStreamTestExample.stream().filter(s -> s.getAge() > 3).collect(Collectors.toList()));
    System.out.println("distinct");
    System.out.println(Stream.of(1, 1, 2).distinct().collect(Collectors.toList()));
    System.out.println("sorted");
    System.out.println(Stream.of(1, 2, 12, 7, 9).sorted().collect(Collectors.toList()));
    System.out.println(Stream.of(1, 2, 12, 7, 9).sorted((a, b) -> b - a).collect(Collectors.toList()));
    System.out.println("skip&limit");
    List<Integer> list = Stream.of(1, 2, 3, 4, 5).collect(Collectors.toList());
    int pageNum = 2;
    int pageSize = 3;
    System.out.println(list.stream().skip((pageNum - 1) * pageSize).limit(pageSize).collect(Collectors.toList()));

    // anyMatch 判断集合中的元素是否至少有一个满足某个条件
    System.out.println("anyMatch");
    System.out.println(Stream.of(1, 2, 1, 3, 2, 5).anyMatch(s -> s > 3));
    System.out.println(Stream.of(1, 2, 1, 3, 2, 5).anyMatch(s -> s > 10));

    // allMatch 判断集合中的元素是否都满足某个条件
    System.out.println("allMatch");
    System.out.println(Stream.of(1, 2, 1, 3, 2, 5).allMatch(s -> s > 3));
    System.out.println(Stream.of(1, 2, 1, 3, 2, 5).allMatch(s -> s > 0));

    // noneMatch 判断集合中的元素是否都不满足某个条件
    System.out.println("noneMatch");
    System.out.println(Stream.of(1, 2, 1, 3, 2, 5).noneMatch(s -> s > 3));
    System.out.println(Stream.of(1, 2, 1, 3, 2, 5).noneMatch(s -> s < 0));

    // findAny 快速获取一个通常第一个
    System.out.println("findAny");
    System.out.println(Stream.of(1, 2, 1, 3, 2, 5).findAny().get());

    System.out.println("findFirst");
    System.out.println(Stream.of(1, 2, 1, 3, 2, 5).findFirst().get());

    //对numbers中的元素求和 16
    System.out.println(Arrays.asList(1, 2, 1, 3, 3, 2, 4)
            .stream()
            .reduce(0, Integer::sum));
    //求集合中的最大值
    Arrays.asList(1,2,1,3,2,5)
            .stream()
            .reduce(Integer::max)
            .ifPresent(System.out::println);
}
```
```text
filter
2
[StreamExample.StreamTestExample(super=com.example.demo.lesson.greenhand.StreamExample$StreamTestExample@eb9, age=4, name=4), StreamExample.StreamTestExample(super=com.example.demo.lesson.greenhand.StreamExample$StreamTestExample@ef5, age=5, name=5)]
distinct
[1, 2]
sorted
[1, 2, 7, 9, 12]
[12, 9, 7, 2, 1]
skip&limit
[4, 5]
anyMatch
true
false
allMatch
false
true
noneMatch
false
true
findAny
1
findFirst
1
16
5

```