# 写了final就是常量池了么
[目录](https://github.com/edanlx/SealBook/blob/master/catalog.md)  
[可直接运行的完整代码](https://github.com/edanlx/TechingCode/tree/master/demoGrace/src/main/java/com/example/demo/lesson/jvm/myloader)  
[视频讲解](https://www.bilibili.com/video/BV1Sz4y1f7FB/)   
[文字版](https://github.com/edanlx/SealBook/blob/master/jvm/jv.md)

如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。

## 1.jre和jdk的关系
![jre和jdk的关系](http://seal_li.gitee.io/sealbook/pic/JDKwithJRE.png)
## 2.运行时数据区域
![运行时数据区域](http://seal_li.gitee.io/sealbook/pic/runtiemDataArea.jpg)
### 2.1.程序计数器
它是程序控制流的指示器，分支、循环、跳转、异常处理、线程恢复等基础功能都需要依赖这个计数器来完成
### 2.2.java虚拟机栈
每个方法被执行的时候，Java虚拟机都 会同步创建一个栈帧（Stack Frame）用于存储局部变量表、操作数栈、动态连接、方法出口等信息  
局部变量表存放了编译期可知的各种Java虚拟机基本数据类型（boolean、byte、char、short、int、 float、long、double）、对象引用
### 2.3.本地方法栈
本地方法栈是为虚拟机使用到的本地（Native） 方法服务
### 2.4.java堆
在《Java虚拟机规范》中对Java堆的描述是：“所有 的对象实例以及数组都应当在堆上分配”。Java堆是垃圾收集器管理的内存区域，因此一些资料中它也被称作“GC堆”。
注：由于现代垃圾收集器大部分都是基于分代收集理论设计的，实际上堆内并没有明确的区分。到G1的全代垃圾回收已经有明显废除分代收集的倾向，ZGC则直接摒弃。
### 2.5.方法区
JDK8以前为永久代实现，以后则为元空间实现，且元空间采用了本地内存不归jvm管理。该区域虽然为非堆但HotSpot虚拟机依然使用了堆的内存管理方式对方法区内的“运行时常量池进行管理”。注意符号引用等会转入常量池不会在元空间，元空间基本只剩下方法的字节码、注解、方法计数器了。顺带一提classloader加载的class对象也会进入堆管理。这里依然要区分堆和非堆的概念。
<!--https://www.cnblogs.com/duanxz/p/3728737.html-->
### 2.6.运行时常量池
运行池常量池是方法区内特殊的一部分。编译后的常量池会归入到运行时常量池区域。详情可查看class文件章节的内容，有明确标明class文件的常量池主要有哪些。
### 2.7.直接内存
它可以使用Native函数库直接分配堆外内存，然后通过一个存储在Java堆里面的 DirectByteBuffer对象作为这块内存的引用进行操作。这样能在一些场景中显著提高性能，因为避免了在Java堆和Native堆中来回复制数据。 

## 3.常量池详解
### 3.1.字面量及符号引用
符号引用包含如下信息：
·被模块导出或者开放的包（Package） 
·类和接口的全限定名（Fully Qualified Name）
·字段的名称和描述符（Descriptor）
·方法的名称和描述符   
·方法句柄和方法类型（Method Handle、Method Type、Invoke Dynamic）  
·动态调用点和动态常量（Dynamically-Computed Call Site、Dynamically-Computed Constant）  

### 3.2.官方给出的存放至常量池的内容
![常量池](http://seal_li.gitee.io/sealbook/pic/ConstantPoolType.jpg)
### 3.3.javap命令实际查看Constant Pool到底存放哪些东西
## 4.运行时栈桢结构
每一个方法从调用开始至执行结束的过程，都对应着一个栈帧在虚拟机栈里面从入栈到出栈的过程。一个线程中的方法调用链可能会很长。而对于执行引擎来讲，在活动线程中，只有位于栈顶的方 法才是在运行的，只有位于栈顶的栈帧才是生效的，其被称为“当前栈帧”（Current Stack Frame），与这个栈帧所关联的方法被称为“当前方法”（Current Method）
![运行时栈桢结构](http://seal_li.gitee.io/sealbook/pic/stackFrame.jpg)
### 4.1.局部变量表
存放boolean、 byte、char、short、int、float、reference、returnAddress占一个slot
long、double占两个个slot
### 4.2.操作数栈
后入先出（Last In First Out，LIFO） 
当一个方法刚刚开始执行的时候，这个方法的操作数栈是空的，在方法的执行过程中，会有各种 字节码指令往操作数栈中写入和提取内容，也就是出栈和入栈操作。譬如在做算术运算的时候是通过 将运算涉及的操作数栈压入栈顶后调用运算指令来进行的，又譬如在调用其他方法的时候是通过操作数栈来进行方法参数的传递。
![操作栈共享区域](http://seal_li.gitee.io/sealbook/pic/StackFrameDataShare.jpg)

### 4.3.动态连接
每个栈帧都包含一个指向运行时常量池，目的是找到要运行方法的直接引用
### 4.4.方法返回地址
当一个方法开始执行后，只有两种方式退出这个方法  
1)正常调用完成(Normal Method Invocation Completion)  
2)异常调用完成(Abrupt Method Invocation Completion)  
### 4.5附加信息
java虚拟机并未规范该部分内容，由各个虚拟机厂商自行实现，通常为性能和调试相关信息
## 6.GC垃圾回收
因GC垃圾回收版本众多，主要采用的均为分代垃圾回收，主要涉及算法标记-清除，标记-整理，标记-复制三种算法  
详情可观看垃圾回收章节