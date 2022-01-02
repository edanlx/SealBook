# 【优雅代码】11-stream精选示例
> 欢迎关注b站账号/公众号【六边形战士夏宁】，一个要把各项指标拉满的男人。该文章已在[github目录](https://github.com/edanlx/SealBook/blob/master/catalogue/wechat.md)收录。
屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。
* [可直接运行的完整代码](https://github.com/edanlx/TechingCode/tree/master/demoGrace/src/main/java/com/example/demo/lesson/grace/stream) 
* [上一篇](./10front.md)优雅数据校验及转换
* [下一篇](./12serialize.md)更快的序列化

## 1.背景
之前在优雅代码系列的第3节分享过了[optional](./03optional.md)的用法,这边就不再赘述了
## 2.常用部分
该部分包含了个人日常开发中95%以上的使用场景，或者其变种
```java
public static void ordinaryUsed() {
    // 并行流，会乱序
    System.out.println("Step1");
    Stream.of(1, 2, 3, 4).parallel().forEach(System.out::print);
    System.out.println();
    System.out.println("Step2");
    // 并行流+收集保证不会乱序
    Stream.of(1, 2, 3, 4).parallel().collect(Collectors.toList()).forEach(System.out::print);
    System.out.println();
    System.out.println("Step3");
    // 获取唯一数据
    System.out.println(Stream.of(1, 2, 3, 4).filter(s -> s.equals(1)).findAny().get());
    System.out.println("Step4");
    // 条件过滤获取新数据-list/set
    System.out.println(Stream.of(1, 2, 3, 4).filter(s -> s > 3).collect(Collectors.toList()));
    // 条件过滤获取新数据-map
    System.out.println("Step5");
    System.out.println(new HashMap<Integer, Integer>() {{
        put(1, 1);
        put(2, 2);
        put(3, 3);
        put(4, 4);
    }}.entrySet().stream().filter(s -> s.getKey() > 3).collect(Collectors.toMap(Map.Entry::getKey, Map.Entry::getValue)));
    System.out.println("Step6");
    // list做物理分页
    List<Integer> list = Stream.of(1, 2, 3, 4, 5).collect(Collectors.toList());
    int pageNum = 2;
    int pageSize = 3;
    System.out.println(list.stream().skip((pageNum - 1) * pageSize).limit(pageSize).collect(Collectors.toList()));
    // 循环创建连续数字list
    System.out.println("Step7");
    System.out.println(IntStream.range(0, 10).boxed().collect(Collectors.toList()));
    // list循环获取索引下标
    System.out.println("Step8");
    IntStream.range(0, list.size()).forEach(x -> System.out.print(list.get(x)));
    System.out.println();
    // list对象-转map，注意key不能相同，否则会报错
    System.out.println("Step9");
    System.out.println(Stream.of(TestEnum.values()).collect(Collectors.toMap(TestEnum::getCode, s -> s)));
    System.out.println("Step10");
    // list对象-转map嵌list,根据选择的属性进行合并
    System.out.println(Arrays.stream(TestEnum.values()).collect(Collectors.groupingBy(TestEnum::getDesc)));
}
```
* 输出如下
```text
Step1
3421
Step2
1234
Step3
1
Step4
[4]
Step5
{4=4}
Step6
[4, 5]
Step7
[0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
Step8
12345
Step9
{1=TestEnum.EXCEL(super=EXCEL, code=1, desc=excel), 2=TestEnum.PPT(super=PPT, code=2, desc=ppt)}
Step10
{excel=[TestEnum.EXCEL(super=EXCEL, code=1, desc=excel)], ppt=[TestEnum.PPT(super=PPT, code=2, desc=ppt)]}
```

## 3.扩展部分
之前在优雅代码系列的第2节[method](./02method.md)和第4节[threa](./04thread.md)分享过了传方法的两种特定场景的用法,这边补充一下常规用法

```java
/**
 * Consumer ：
 * 1.主要适用于没有返回值的,
 * 2.可以在方法内部抽1个小方法，注意该方法只能传1个参数，且一定要简单，复杂的话还是直接抽方法，仅仅是处理代码重复问题
 * Supplier主要适用于有返回值的
 * predicate
 * 1.和consumer类似，可以在方法内抽小方法，也只接受一个参数
 * 2.接受传参方式的判断，可应用于多个复杂判断拆成小单元时再行组合，应用场景少
 * <p>
 * 虽然只接收一个参数但可以是对象
 *
 * @author 876651109@qq.com
 * @date 2021/2/10 9:25 上午
 */
public class Delay {
    public static <T> T showLog(int level, Supplier<T> method) {
        // 日志级别等于1的时候执行，实际场景为从redis获取数据为空时执行刷新缓存并返回
        if (level == 1) {
            return method.get();
        } else {
            return null;
        }
    }

    public static void showLog2(int level, Consumer method) {
        // 日志街边等于3的时候输出日志
        if (level == 3) {
            method.accept(level);
        }
    }

    public static void main(String[] args) throws InterruptedException {
        // 输出
        showLog(1, () -> {
            System.out.println(1);
            return 1;
        });
        // 不输出
        showLog(2, () -> {
            System.out.println(2);
            return 1;
        });
        // 输出
        showLog2(3, System.out::println);
        // 不输出
        showLog2(4, System.out::println);
    }
}
```

## 4.非常用部分
这部分就不是个人感悟了，网上有各种使用，这里仅列举部分场景
```java
public static void predicate() {
    Predicate<Integer> predicate = x -> x > 7;
    System.out.println(predicate.test(10)); //输出 true
    System.out.println(predicate.test(6));  //输出 fasle
    /**
     * 2、大于7并且
     */
    //在上面大于7的条件下，添加是偶数的条件
    predicate = predicate.and(x -> x % 2 == 0);
    System.out.println(predicate.test(6));  //输出 fasle
    System.out.println(predicate.test(12)); //输出 true
    System.out.println(predicate.test(13)); //输出 fasle
    /**
     * 3、add or 简化写法
     */
    predicate = x -> x > 5 && x < 9;
    System.out.println(predicate.test(10)); //输出 false
    System.out.println(predicate.test(6));  //输出 true
}

public static void reduce() {
    // 从40开始，迭代+2，共20个
    List<Integer> list3 = Stream.iterate(40, n -> n + 2).limit(20).collect(Collectors.toList());
    System.out.println(list3);
    // 从10的基础上全部相加
    int reducedParams = Stream.of(1, 2, 3)
            .reduce(10, (a, b) -> a + b, (a, b) -> {
                return a + b;
            });
    System.out.println(reducedParams);
}
```