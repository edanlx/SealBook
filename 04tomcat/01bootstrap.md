# 【tomcat】01-tomcat启动主要流程
> 欢迎关注b站账号/公众号【六边形战士夏宁】，一个要把各项指标拉满的男人。该文章已在[github目录](https://github.com/edanlx/SealBook)收录。
屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。
* [gitee目录](https://gitee.com/seal_li/SealBook)
* [知乎目录](https://zhuanlan.zhihu.com/p/338222208)
* [csdn目录](https://blog.csdn.net/seal_li/article/details/111415366)
* [博客园目录](https://www.cnblogs.com/sealLee/articles/14748368.html)
* [视频讲解](https://www.bilibili.com/video/BV1GK41137LQ/)   

**注意以下流程每个tomcat版本略有不同**

## 1.整体启动
* 打开startup.sh可以看到启动了catalina.sh
* 打开catalina.sh可以看到启动了bootstrap.jar
* 于是找到Bootstrap.java主函数执行main方法/如果是spring boot则直接使用tomcat.java的main
* 找到start命令发现执行Bootstrap.java的的load、start方法
* 反射执行Catalina.java的load、start方法

## 2.server.xml加载
* 执行createStartDigester加载反射类
* 进入load方法执行digester.parse(inputSource)方法加载server.xml
* 回到start方法执行getServer().start()开始以startInternal()进行责任链调用
* 执行StandardServer.java的startInternal()方法
* 执行service.start()方法接着执行StandardService的startInternal()方法
* 执行engine.start()方法接着执行StandardEngine的startInternal()方法
* 执行super.startInternal();即ContainerBase的startInternal()方法
* 执行results.add(startStopExecutor.submit(new StartChild(child)));
* 注意此时就有了变化child由上面的findChildren()方法获得
* 分别执行四个容器的start

## 3.nio线程加载
* 回到StandardService.java的startInternal()方法
* 执行connector.start();即Connector的startInternal方法
* 执行protocolHandler.start();注意ProtocolHandler即为接口类，通常由Http11NioProtocol实现加载具体则看配置,创建代码为p = ProtocolHandler.create(protocol, apr);不同版本略有不同，搜索关键字"HTTP/1.1"即可
* 注意看 return "http-nio";这行代码，打开jvisualvm可以看到有若干个nio线程

## 4.web.xml加载
* 回到StandardContext.java的startInternal可以看到使用resource和web.xml(加载为ContextConfig.java,由ContextRuleSet进行注册而它由Catalina注册)，接着对listener、manager、filter等按顺序初始化

## 5.classes
由WebApp加载器加载
![tomcat类加载器](http://seal_li.gitee.io/sealbook/pic/tomcat_bootstrap_TomcatClassLoader.jpg)