# 【优雅代码】12更快的序列化
> 欢迎关注b站账号/公众号【六边形战士夏宁】，一个要把各项指标拉满的男人。该文章已在[github目录](https://github.com/edanlx/SealBook)收录。
屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。
* [gitee目录](https://gitee.com/seal_li/SealBook)
* [知乎目录](https://zhuanlan.zhihu.com/p/338222208)
* [csdn目录](https://blog.csdn.net/seal_li/article/details/111415366)
* [博客园目录](https://www.cnblogs.com/sealLee/articles/14748368.html)
* [上一篇](./11stream.md)
* [下一篇](./13listSpeed.md)

## 1.背景
平常我们在使用rpc调用或者将其持久化到数据库的时候则需要将对象或者文件或者图片等数据将其转为二进制字节数据。
## 2.常见的序列化方式
<!-- https://segmentfault.com/a/1190000021701653 -->
java自带的序列化,平常用的最多的json序列化(也可以叫http数据传输的序列化),dubbo默认的序列化hessian2,目前公认稳定且最快的序列化方式Kryo
### 2.1
## 3.比较