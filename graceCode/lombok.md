@[TOC](lombok与delombok)
# lombok
先上[官网](https://projectlombok.org/)地址  
[官方教程](https://projectlombok.org/features/EqualsAndHashCode)  注意替换域名后缀
## 相关注解
@EqualsAndHashCode(equals和hashcode相关)
@AllArgsConstructor, @RequiredArgsConstructor ,@NoArgsConstructor（构造方法相关）  
@Log, @Log4j, @Log4j2, @Slf4j, @XSlf4j, @CommonsLog, @JBossLog, @Flogger, @CustomLog(日志相关)  
@Data(get、set方法)  
@Builder,@SuperBuilder(构造链)  
@Singular(默认值)  
@Delegate(委托方法，用于避免超类出错)  
@Value(私有构造方法，并开放一个公共静态构造)  
@Accessors(链式调用)  
@Wither(已并入value)  
@With(使用with构造)  
@SneakyThrows(异常相关)  
@val,@var(没有不会js的吧)  
# delombok
[官网](https://projectlombok.org/features/delombok)

```
java -jar lombok.jar delombok src -d src-delomboked
```
注意替换包名
# 友情链接
[视频讲解](https://www.bilibili.com/video/BV1yC4y1877R/)  
[github](https://github.com/edanlx/SealBook)

如果有帮助到你的话请顺手点个赞，这对我真的很重要