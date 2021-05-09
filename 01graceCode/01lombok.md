# 【优雅代码】01-最简单的简化代码注解
> 欢迎关注b站账号/公众号【六边形战士夏宁】，一个要把各项指标拉满的男人。该文章已在[github目录](https://github.com/edanlx/SealBook)收录。
屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。
* [gitee目录](https://gitee.com/seal_li/SealBook)
* [知乎目录](https://zhuanlan.zhihu.com/p/338222208)
* [csdn目录](https://blog.csdn.net/seal_li/article/details/111415366)
* [可直接运行的完整代码](https://github.com/edanlx/TechingCode/tree/master/demoGrace/src/main/java/com/example/demo/entity)
* [视频讲解](https://www.bilibili.com/video/BV1yC4y1877R/)
* [下一篇](./02method.md)

## 1.lombok
先上[官网](https://projectlombok.org/)地址  
[官方教程](https://projectlombok.org/features/EqualsAndHashCode)  注意替换域名后缀
### 1.1.相关注解
* 方法使用的注解
@SneakyThrows该注解后面可以跟异常，一般会使用编译时异常，因为编译时异常正常都是不会报错的，比如类找不到等情况，使用该注解可以让代码变得更易读
* 实体类使用的注解
@EqualsAndHashCode(equals和hashcode相关)、@AllArgsConstructor（构造方法相关）、@NoArgsConstructor（构造方法相关）、@Data(get、set类)、@Builder(创建类的时候可以直接使用build强力推荐)。推荐使用以上五个注解，至于@Accessors注解虽然很好用但是和bean拷贝会有冲突，所以不太推荐使用。注意这里部分注解在使用的时候要把父类实现带上，否则会有隐藏bug。
* 日志使用的注解
@Slf4j该注解反编译会自动注入class算是比较方便的
* 其它注解
这些注解使用场景并不多
@Log, @Log4j, @Log4j2,  @XSlf4j, @CommonsLog, @JBossLog, @Flogger, @CustomLog(日志相关)  
@Data(get、set方法)  
@SuperBuilder(构造链)  
@Singular(默认值)  
@Delegate(委托方法，用于避免超类出错)  
@Value(私有构造方法，并开放一个公共静态构造)  
@Accessors(链式调用)  
@Wither(已并入value)  
@With(使用with构造)  
@val,@var(没有不会js的吧)  
@clean

## 2.delombok
[官网](https://projectlombok.org/features/delombok)
* 方法一

```
java -jar lombok.jar delombok src -d src-delomboked
```
注意替换包名
* 方法二
在新版的idea中refactor->delombok已经集成了delombok，可以直接还原算是比较方便的