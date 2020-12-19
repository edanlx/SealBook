# mysql的innodb整体结构
[目录](https://github.com/edanlx/SealBook/blob/master/catalog.md)  
[视频讲解](https://www.bilibili.com/video/BV1Ey4y167HQ/)   
[文字版](https://github.com/edanlx/SealBook/blob/master/mysql/structure.md)

屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。

## 1.整体结构
如图所示在经过语法分析的时候如果无法通过则进行报错
生成语法树
经过优化器
经过执行器
## 2.inndodb聚簇索引(主键索引)
## 3.inndodb非聚簇索引(普通索引、联合索引)