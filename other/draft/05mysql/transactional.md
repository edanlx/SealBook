# mysql的innodb整体结构
> 欢迎关注b站账号/公众号【六边形战士夏宁】，一个要把各项指标拉满的男人。该文章已在[github目录](https://github.com/edanlx/SealBook/blob/master/catalogue/wechat.md)收录。
屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。
* [gitee目录](https://gitee.com/seal_li/SealBook)
* [知乎目录](https://zhuanlan.zhihu.com/p/338222208)
* [csdn目录](https://blog.csdn.net/seal_li/article/details/111415366)

## 1.事务基本概念
* **原子性(Atomicity)**:事务是一个原子操作单元,其对数据的修改,要么全都执行,要么全都不执行。
* **一致性(Consistent)**:在事务开始和完成时,数据都必须保持一致状态。这意味着所有相关的数据规
则都必须应用于事务的修改,以保持数据的完整性。
* **隔离性(Isolation)**:数据库系统提供一定的隔离机制,保证事务在不受外部并发操作影响的“独
立”环境执行。这意味着事务处理过程中的中间状态对外部是不可见的,反之亦然。
* **持久性(Durable)**:事务完成之后,它对于数据的修改是永久性的,即使出现系统故障也能够保持。

## 2.并发事务的隔离级别
* **脏写(Lost Update)**:最后的更新覆盖了由其他事务所做的更新。
* **脏读(Dirty Reads)**:事务A读取到了事务B已经修改但尚未提交的数据。
* **不可重读(Non-Repeatable Reads)**:事务A内部的相同查询语句在不同时刻读出的结果不一致，不符合隔离性
* **幻读(Phantom Reads)**:事务A读取到了事务B提交的新增数据，不符合隔离性
|隔离级别|脏读|不可重复读|幻读|
|--|--|--|--|
|读未提交|可能|可能|可能|
|读已提交|不可能|可能|可能|
|可重复读|不可能|不可能|可能|
|可串行化|不可能|不可能|不可能|

## 3.锁分类
* 从性能上分为**乐观锁**与**悲观锁**
* 从数据库操作上分为**共享锁(读锁)**与**排它锁(写锁)**
* 从数据库粒度上分为**表锁**与**行锁**

## 4.可重复读事务的实现————间隙锁
* **间隙锁**:Innodb在可重复读提交下为了解决幻读问题时引入的锁机制。当进行范围查找时会出现幻读问题