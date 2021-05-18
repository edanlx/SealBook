# 【优雅代码】15不用部署中间件的本地缓存
> 欢迎关注b站账号/公众号【六边形战士夏宁】，一个要把各项指标拉满的男人。该文章已在[github目录](https://github.com/edanlx/SealBook)收录。
屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。
* [gitee目录](https://gitee.com/seal_li/SealBook)
* [知乎目录](https://zhuanlan.zhihu.com/p/338222208)
* [csdn目录](https://blog.csdn.net/seal_li/article/details/111415366)
* [博客园目录](https://www.cnblogs.com/sealLee/articles/14748368.html)
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