# new一个对象到底占了多少内存?
[目录](https://github.com/edanlx/SealBook/blob/master/catalog.md)  
[可直接运行的完整代码](https://github.com/edanlx/TechingCode/tree/master/demoGrace/src/main/java/com/example/demo/lesson/jvm/object)  
[视频讲解](https://www.bilibili.com/video/BV1A54y1k7UW/)   
[文字版](https://github.com/edanlx/SealBook/blob/master/jvm/HotSpotAndObject.md)

如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。

## 1.对象的创建
当Java虚拟机遇到一条字节码new指令时，首先将去检查这个指令的参数是否能在常量池中定位到 一个类的符号引用，并且检查这个符号引用代表的类是否已被加载、解析和初始化过。如果没有，那必须先执行相应的类加载过程。
### 1.1.空间分配
* 指针碰撞
假设Java堆中内存是绝对规整的，所有被使用过的内存都被放在一 边，空闲的内存被放在另一边，中间放着一个指针作为分界点的指示器，那所分配内存就仅仅是把那 个指针向空闲空间方向挪动一段与对象大小相等的距离(Serial、ParNew使用)
* 空间列表
但如果Java堆中的内存并不是规整的，已被使用的内存和空闲的内存相互交错在一起，那 就没有办法简单地进行指针碰撞了，虚拟机就必须维护一个列表，记录上哪些内存块是可用的，在分 配的时候从列表中找到一块足够大的空间划分给对象实例，并更新列表上的记录(CMS使用)

### 1.2.创建时并发
* TLAB（Thread Local Allocation Buffer 线程本地分配缓存）
把内存分配的动作按照线程划分在不同的空间之中进 行，即每个线程在Java堆中预先分配一小块内存（实际上是Eden区中划出的）。（-XX: +UseTLAB默认开启，关闭后则使用CAS）
内存分配完成之后，虚拟机必须将分配到的内存空间（但不包括对象头）都初始化为零值。接着创建设置对象头的信息，最后执行构造方法。
* 栈上分配
因为一旦分配在堆空间中，当方法调用结束，没有了引用指向该对象，该对象就需要被gc回收，而如果存在大量的这种情况，对gc来说无疑是一种负担。（逃逸分析：-XX:+DoEscapeAnalysis、标量替换：-XX:+EliminateAllocations默认开启）  
可通过循环调用某个方法创建对象但堆空间不会发生变化

## 2.对象的内存布局
在HotSpot虚拟机里，对象在堆内存中的存储布局可以划分为三个部分：对象头（Header）、实例数据（Instance Data）和对齐填充（Padding）。
* 对象头
第一部分是用于存储对象自身的运行时数据，如哈希码（HashCode）、GC分代年龄、锁状态标志、线程持有的锁、偏向线程ID、偏向时间戳等，这部分数据的长度在32位和64位的虚拟机（未开启压缩指针）中分别为32个比特和64个比特
|存储内容|标志位|状态|
|--|--|--|
|对象哈希码、对象分代年龄|01|未锁定|
|指向锁记录的指针|00|轻量锁定|
|指向重量级锁的指针|10|膨胀|
|空|11|GC标记|
|偏向锁ID、偏向锁时间戳、对象分代年龄等|01|可偏向|
对象头的另外一部分是类型指针，即对象指向它的类型元数据的指针，Java虚拟机通过这个指针 来确定该对象是哪个类的实例
* 实例数据
即我们在程序代码里面所定义的各种类型的字段内容，无论是从父类继承下来的，还是在子类中定义的字段都必须记录起来。
* 对齐填充
由于HotSpot虚拟机的自动内存管理系统要求对象起始地址必须是8字节的整数倍

## 3.对象的访问定位
* 句柄访问
![句柄访问](http://seal_li.gitee.io/sealbook/pic/jvm_HotSpotAndObject_object1.png)
* 直接指针访问(HotSpot使用,可以减少一次指针开销)
![直接指针](http://seal_li.gitee.io/sealbook/pic/jvm_HotSpotAndObject_object2.png)

## 4.内存分配与回收策略
* 对象优先在Eden分配
大多数情况下，对象在新生代Eden区中分配。当Eden区没有足够空间进行分配时，虚拟机将发起 一次Minor GC。Hotspot默认为8:1:1
* 大对象直接进入老年代
-XX:PretenureSizeThreshold ，默认关闭开启的优势在于避免新生代在使用复制算法时效率降低
* 长期存活的对象将进入老年代
-XX:MaxTenuringThrehold,CMS以前都是15岁，CMS默认6岁
* 动态对象年龄判定
-XX:TargetSurvivorRatio 默认50%，年龄1+年龄2+年龄n的多个年龄对象总和超过了Survivor区域的50%，此时就会 把年龄n(含)以上的对象都放入老年代,会在minar gc后触发
* 空间分配担保
在回收前检查老年代空间确保有足够的空间存放年轻代，如果不够的话则触发full gc。老年代默认使用2/3。

## 5.对象内存回收
* 引用计数法
给对象中添加一个引用计数器，每当有一个地方引用它，计数器就加1;当引用失效，计数器就减1;任何时候计数器为0 的对象就是不可能再被使用的。互相引用则无法确定。Hotspot未采用。
* 可达性分析算法
GC Roots根节点:线程栈的本地变量、静态变量、本地方法栈的变量等等，依次向下寻找 

## 6.扩展知识
* java常见引用类型
java的引用类型一般分为四种:强引用(不可回收，new的时候直接引用)、软引用(SoftReference包裹，收集后还不够则再行回收，可用于缓存)、弱引用(WeakReference包裹，会被直接回收)、虚引用(仅作标示)
* finalize
在被回收前执行该方法，但只会执行一次，开发代码中禁止使用

## 7.补充知识对象大小计算
也可以使用jvisualvm(高版本jdkbin目录下)
代码如下
```java
public static void main(String[] args) {
        ClassLayout layout = ClassLayout.parseInstance(new Object());
        System.out.println("------------------layout------------------");
        System.out.println(layout.toPrintable());
        ClassLayout layout1 = ClassLayout.parseInstance(new Parent());
        System.out.println("------------------layout1------------------");
        System.out.println(layout1.toPrintable());
        Parent parent = new Parent().setChild(new Child());
        ClassLayout layout2 = ClassLayout.parseInstance(parent);
        System.out.println("------------------layout2------------------");
        System.out.println(layout2.toPrintable());
        ClassLayout layout3 = ClassLayout.parseInstance(new Child());
        System.out.println("------------------layout3------------------");
        System.out.println(layout3.toPrintable());
        char c1 = '1';
        ClassLayout layout4 = ClassLayout.parseInstance(c1);
        System.out.println("------------------layout4------------------");
        System.out.println(layout4.toPrintable());
        char c2 = '你';
        ClassLayout layout5 = ClassLayout.parseInstance(c2);
        System.out.println("------------------layout5------------------");
        System.out.println(layout5.toPrintable());
        char[] arrC1 = new char[0];
        System.out.println("------------------layout6------------------");
        ClassLayout layout6 = ClassLayout.parseInstance(arrC1);
        System.out.println(layout6.toPrintable());


        char[] arrC2 = new char[]{'你','好'};
        ClassLayout layout7 = ClassLayout.parseInstance(arrC2);
        System.out.println("------------------layout7------------------");
        System.out.println(layout7.toPrintable());

        char[] arrC3 = new char[]{'你'};
        ClassLayout layout8 = ClassLayout.parseInstance(arrC3);
        System.out.println("------------------layout8------------------");
        System.out.println(layout8.toPrintable());
        while (true){
            parent.toString();
        }
    }
class Child {
    private List<String> list;
    private String str;
}
class Parent {
    private Child child;
}
```
引用jar包
```xml
<dependency>
    <groupId>org.openjdk.jol</groupId>
    <artifactId>jol-core</artifactId>
    <version>0.9</version>
</dependency>
```
执行结果如下:
```txt
------------------layout------------------
java.lang.Object object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           e5 01 00 f8 (11100101 00000001 00000000 11111000) (-134217243)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

------------------layout1------------------
com.example.demo.lesson.grace.optional.Parent object internals:
 OFFSET  SIZE                                           TYPE DESCRIPTION                               VALUE
      0     4                                                (object header)                           05 00 00 00 (00000101 00000000 00000000 00000000) (5)
      4     4                                                (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4                                                (object header)                           c4 f3 00 f8 (11000100 11110011 00000000 11111000) (-134155324)
     12     4   com.example.demo.lesson.grace.optional.Child Parent.child                              null
Instance size: 16 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total

------------------layout2------------------
com.example.demo.lesson.grace.optional.Parent object internals:
 OFFSET  SIZE                                           TYPE DESCRIPTION                               VALUE
      0     4                                                (object header)                           05 00 00 00 (00000101 00000000 00000000 00000000) (5)
      4     4                                                (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4                                                (object header)                           c4 f3 00 f8 (11000100 11110011 00000000 11111000) (-134155324)
     12     4   com.example.demo.lesson.grace.optional.Child Parent.child                              (object)
Instance size: 16 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total

------------------layout3------------------
com.example.demo.lesson.grace.optional.Child object internals:
 OFFSET  SIZE               TYPE DESCRIPTION                               VALUE
      0     4                    (object header)                           05 00 00 00 (00000101 00000000 00000000 00000000) (5)
      4     4                    (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4                    (object header)                           06 f4 00 f8 (00000110 11110100 00000000 11111000) (-134155258)
     12     4     java.util.List Child.list                                null
     16     4   java.lang.String Child.str                                 null
     20     4                    (loss due to the next object alignment)
Instance size: 24 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

------------------layout4------------------
java.lang.Character object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 00 00 00 (00000101 00000000 00000000 00000000) (5)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           c6 20 00 f8 (11000110 00100000 00000000 11111000) (-134209338)
     12     2   char Character.value                           1
     14     2        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 2 bytes external = 2 bytes total

------------------layout5------------------
java.lang.Character object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 00 00 00 (00000101 00000000 00000000 00000000) (5)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           c6 20 00 f8 (11000110 00100000 00000000 11111000) (-134209338)
     12     2   char Character.value                           你
     14     2        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 2 bytes external = 2 bytes total

------------------layout6------------------
[C object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           41 00 00 f8 (01000001 00000000 00000000 11111000) (-134217663)
     12     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
     16     0   char [C.<elements>                             N/A
Instance size: 16 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total

------------------layout7------------------
[C object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           41 00 00 f8 (01000001 00000000 00000000 11111000) (-134217663)
     12     4        (object header)                           02 00 00 00 (00000010 00000000 00000000 00000000) (2)
     16     4   char [C.<elements>                             N/A
     20     4        (loss due to the next object alignment)
Instance size: 24 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

------------------layout8------------------
[C object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           41 00 00 f8 (01000001 00000000 00000000 11111000) (-134217663)
     12     4        (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
     16     2   char [C.<elements>                             N/A
     18     6        (loss due to the next object alignment)
Instance size: 24 bytes
Space losses: 0 bytes internal + 6 bytes external = 6 bytes total
```
即一个空对象为8byte的普通对象头+4byte的kcalss+4byte的补充空间


## 参考资料
《深入理解Java虚拟机》-周志明