# 【优雅代码】01-最简单的简化代码注解
> 欢迎关注b站账号/公众号【六边形战士夏宁】，一个要把各项指标拉满的男人。该文章已在[github目录](https://github.com/edanlx/SealBook)收录。
屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。
* [gitee目录](https://gitee.com/seal_li/SealBook)
* [知乎目录](https://zhuanlan.zhihu.com/p/338222208)
* [csdn目录](https://blog.csdn.net/seal_li/article/details/111415366)
* [博客园目录](https://www.cnblogs.com/sealLee/articles/14748368.html)
* [可直接运行的完整代码](https://github.com/edanlx/TechingCode/tree/master/demoGrace/src/main/java/com/example/demo/lesson/grace/lombok) 
* [视频讲解](https://www.bilibili.com/video/BV1yC4y1877R/)
* [下一篇](./02method.md)java传递方法

## 1.背景介绍
在日常开发中免不了进行一些繁琐的代码自动生成，虽然ide的功能已然非常强大但是并不能够做到动态，lombok可以非常好的解决这个问题。它会在生成class文件时将其进行编译成平常所写的代码,这里介绍一些我个人觉得比较好用的注解
## 2.lombok
先上[官网](https://projectlombok.org/)地址。如果想了解更多注解可以去[https://projectlombok.org/features/all](https://projectlombok.org/features/all)
### 2.1.get/set注解(重要)
此部分注解有@Data、@Getter、@Setter,一般普通Bean对象会使用@Data注解(里面已经包含另外两个注解)，如果是enum则使用@Getter注解
```java
@Data
static class DataExample{
    private String name;
}
```
### 2.2.构造方法注解(重要)
此部分注解包含@NoArgsConstructor无参构造、@AllArgsConstructor所有参构造、@Builder构造链
```java
@NoArgsConstructor
@Builder
@AllArgsConstructor
static class DataExample{
    private String name;
}
// 使用方法如下
new DataExample();
new DataExample("");
DataExample.builder().name("123").build();
```
### 2.3.日志注解(重要)
此部分注解包含@Slf4j，其它的注解都不重要，这个会自动根据引入包进行选择
```java
@Slf4j
public class LombokExample {
	public static void main(String[] args) {
        log.info("123");
    }
}
```
### 2.4.Bean辅助注解(重要)
众所周知，比较两个对象是否相等是要用equals。所以有如下注解@EqualsAndHashCode、@ToString。值得注意的是如果有继承父类需要填写callSuper = true
### 2.5.链式调用
@Accessors,该注解一度觉得很好用，后来发现它和一些拷贝等不兼容就放弃了
### 2.6.关闭流
@Cleanup,这个可以帮助关闭流，需要注意的是需要对其捕获IO异常。虽然不错，但是有了trywith的写法以后就用的不多了。
```java
try {
    @Cleanup FileInputStream fileInputStream = new FileInputStream("");
} catch (IOException e) {
    e.printStackTrace();
}
```
### 2.7.异常注解
在捕获编译时异常的时候比较好用，但是现在越来越多的工具类都对编译异常捕获了，它的出场机会并不多
```java
@SneakyThrows({Exception.class})
    public static void main(String[] args) {
}
```
## 3.delombok
如果后面不想用注解则需要使用delombok[官网](https://projectlombok.org/features/delombok)
* 方法一

```sh
java -jar lombok.jar delombok src -d src-delomboked
```
注意替换包名
* 方法二
在新版的idea中refactor->delombok已经集成了delombok，可以直接还原算是比较方便的