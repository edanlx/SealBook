# 【优雅代码】13linkedList插入真的比arrayList快么
> 欢迎关注b站账号/公众号【六边形战士夏宁】，一个要把各项指标拉满的男人。该文章已在[github目录](https://github.com/edanlx/SealBook)收录。
屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。
* [gitee目录](https://gitee.com/seal_li/SealBook)
* [知乎目录](https://zhuanlan.zhihu.com/p/338222208)
* [csdn目录](https://blog.csdn.net/seal_li/article/details/111415366)
* [博客园目录](https://www.cnblogs.com/sealLee/articles/14748368.html)
* [上一篇](./12serialize.md)
* [下一篇](./14localeCache.md)

## 1.背景
在目前开发中，前后端分离已然成为主流，而后端则有三件事要做，1.不完全信任前端的数据，2.减轻前后端压力3.将接口文档暴露给前端并确保其能看得懂，
## 2.注解边界值
### 2.1官方注解