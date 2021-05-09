# 【优雅代码】03-optional杜绝空指针异常
> 欢迎关注b站账号/公众号【六边形战士夏宁】，一个要把各项指标拉满的男人。该文章已在[github目录](https://github.com/edanlx/SealBook)收录。
屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。
* [gitee目录](https://gitee.com/seal_li/SealBook)
* [知乎目录](https://zhuanlan.zhihu.com/p/338222208)
* [csdn目录](https://blog.csdn.net/seal_li/article/details/111415366)
* [可直接运行的完整代码](https://github.com/edanlx/TechingCode/tree/master/demoGrace/src/main/java/com/example/demo/lesson/grace/optional)  
* [视频讲解](https://www.bilibili.com/video/BV1oy4y1r7r1/)   
* [上一篇](./02method.md)
* [下一篇](./04thread.md)

## 1.建立Child、Parent、GrandParent的层级

* Child

```java
@Data
@AllArgsConstructor
@ToString(callSuper = true)
@NoArgsConstructor
@Accessors(chain = true)
public class Child {
    private List<String> list;
    private String str;
}
```

* GrandParent
```java
@Data
@AllArgsConstructor
@ToString(callSuper = true)
@NoArgsConstructor
@Accessors(chain = true)
public class GrandParent {
    private Parent parent;
}
```

* Parent
```java
@Data
@AllArgsConstructor
@ToString(callSuper = true)
@NoArgsConstructor
@Accessors(chain = true)
public class Parent {
    private Child child;
}

```
* 测试类
```java
	GrandParent opt1 = null;
        String opt1Str =
                Optional.ofNullable(opt1).map(o1 -> o1.getParent())
                        .map(o2 -> o2.getChild()).map(l->l.getStr()).orElse(null);
        System.out.println(String.format("%s:%s", "opt1Object", opt1Str));

        GrandParent opt2 = null;
        List<String> opt2list =
                Optional.ofNullable(opt2).map(o1 -> o1.getParent())
                        .map(o2 -> o2.getChild()).map(l->l.getList()).orElse(null);
        System.out.println(String.format("%s:%s", "opt2list", opt2list));

        GrandParent opt3 = new GrandParent().setParent(new Parent().setChild(new Child().setStr("ssss").setList(Stream.of("1", "2").collect(Collectors.toList()))));
        List<String> opt3list =
                Optional.ofNullable(opt3).map(o1 -> o1.getParent())
                        .map(o2 -> o2.getChild()).map(l->l.getList()).orElse(null);
        String opt3Str =
                Optional.ofNullable(opt3).map(o1 -> o1.getParent())
                        .map(o2 -> o2.getChild()).map(l->l.getStr()).orElse(null);
        System.out.println(String.format("%s:%s", "opt3list", opt3list));
        System.out.println(String.format("%s:%s", "opt3Str", opt3Str));
```

* 输出结果
```
opt1Object:null
opt2list:null
opt3list:[1, 2]
opt3Str:ssss
```
可以发现嵌套类无论是string还是list，中间任何一个类为null都会直接返回null，而不用去多层嵌套if这种很蠢的做法

## 2.非常规复杂optional用法
当然平常时候总会遇到一些奇奇怪怪的结果，例如查询数据库会返回List<Map<String, String>>这样的结构，也是可以用Optinoal做的
```java
        List<Map<String, String>> listR = null;
        String result = Optional.ofNullable(listR).flatMap(l -> l.stream().findAny())
                .flatMap(l -> l.keySet().stream().findAny()).orElse(null);
        System.out.println(result);
        listR = new ArrayList<Map<String, String>>();
        result = Optional.ofNullable(listR).flatMap(l -> l.stream().findAny())
                .flatMap(l -> l.keySet().stream().findAny()).orElse(null);
        System.out.println(result);
        listR = new ArrayList<Map<String, String>>() {{
            add(new HashMap<String, String>());
        }};
        result = Optional.ofNullable(listR).flatMap(l -> l.stream().findAny())
                .flatMap(l -> l.keySet().stream().findAny()).orElse(null);
        System.out.println(result);
        listR = new ArrayList<Map<String, String>>() {{
            add(new HashMap<String, String>() {{
                put("C", "0");
            }});
        }};
        result = Optional.ofNullable(listR).flatMap(l -> l.stream().findAny())
                .flatMap(l -> l.keySet().stream().findAny()).orElse(null);
        System.out.println(result);
```
输出结果如下：
```
null
null
null
C
```
## 3.其它优秀杜绝空指针异常的优秀方法
首先是lombok的@NonNull，这个可以作用于方法参数上，如果传入空则直接抛异常，并在日志精准打印出异常位置及情况，非常适合校验参数，增加代码简洁性

当然刚才写的类还是返回了null，但是没关系，可以用以下工具类，在各种情况下都可以抛出自定义异常或直接return出去
```java
System.out.println(String.format("%s:%s", "StringUtils", StringUtils.isBlank(null)));
System.out.println(String.format("%s:%s", "defaultIfNull", ObjectUtils.defaultIfNull(null, "defaultIfNull")));
List list = null;
Map map = null;
Set set = null;
String[] arr = null;
System.out.println(String.format("%s:%s", "list", CollectionUtils.isEmpty(list)));
System.out.println(String.format("%s:%s", "map", CollectionUtils.isEmpty(map)));
System.out.println(String.format("%s:%s", "set", CollectionUtils.isEmpty(set)));
System.out.println(String.format("%s:%s", "arr", ArrayUtils.isEmpty(arr)));
```

输出结果如下：
```
StringUtils:true
defaultIfNull:defaultIfNull
list:true
map:true
set:true
arr:true
```