# mysql的innodb整体结构
> 欢迎关注b站账号/公众号【六边形战士夏宁】，一个要把各项指标拉满的男人。该文章已在[github目录](https://github.com/edanlx/SealBook/blob/master/catalogue/wechat.md)收录。
屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。
* [gitee目录](https://gitee.com/seal_li/SealBook)
* [知乎目录](https://zhuanlan.zhihu.com/p/338222208)
* [csdn目录](https://blog.csdn.net/seal_li/article/details/111415366)

## 1.整体结构
如图所示在经过语法分析的时候如果无法通过则进行报错
生成语法树
经过优化器
经过执行器
## 2.inndodb聚簇索引(主键索引)
## 3.inndodb非聚簇索引(普通索引、联合索引)