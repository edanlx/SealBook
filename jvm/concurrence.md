# 偏向锁、轻量锁、重量锁到底是啥?
[目录](https://github.com/edanlx/SealBook/blob/master/catalog.md)  
[可直接运行的完整代码](https://github.com/edanlx/TechingCode/tree/master/demoGrace/src/main/java/com/example/demo/lesson/jvm/concurrence) 
[视频讲解](https://www.bilibili.com/video/BV1LV411a7u7/)   
[文字版](https://github.com/edanlx/SealBook/blob/master/jvm/concurrence.md)

如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。

## 1.概述
多任务处理在现代计算机操作系统中几乎已是一项必备的功能了。在许多场景下，让计算机同时 去做几件事情，不仅是因为计算机的运算能力强大了，还有一个很重要的原因是计算机的运算速度与  它的存储和通信子系统的速度差距太大，大量的时间都花费在磁盘I/O、网络通信或者数据库访问上。这个在 [一行代码完成多线程](https://github.com/edanlx/SealBook/blob/master/graceCode/thread.md)有写过如何分配线程，原理基本一致。  
由于计算机 的存储设备与处理器的运算速度有着几个数量级的差距，所以现代计算机系统都不得不加入一层或多 层读写速度尽可能接近处理器运算速度的高速缓存(Cache)  来作为内存与处理器之间的缓冲，但同时引入新的问题即缓存一致性Java虚拟机的即时编译器中也有指令重排序 (Instruction Reorder)优化([详见](https://github.com/edanlx/SealBook/blob/master/jvm/compile.md))。

## 2.java内存模型
### 2.1.主内存与工作内存
主内存直接对应于物理硬件的内存，而为了获取更好的运行速度，虚拟机(或者是硬件、操作系统本身的优化措施)可能会让工作内存优先存储于寄存器和高速缓存中，因为程序运行时主要访问的是工作内存。
### 2.2.内存间交互操作
Java内存模型中定义了以下8种操作来完成:lock(锁定)\unlock(解锁)\read(从主内存读取到工作内存中)\load(放入工作内存副本)\use(载入执行引擎)\assign(赋值)\store(传递至主内存)\write(写入主内存)。即不允许read和load、store和write操作之一单独出现。对一个变量执行unlock操作之前，必须先把此变量同步回主内存中(执行store、write操作)等
### 2.3.volatile
可以被认为是轻量级同步，仅保证可见性,如下代码则会发现线程2永远不会打印hello，但是当加入volatile就可以打印了
```java
/**
 * @author seal email:876651109@qq.com
 * @date 2020/11/22 3:20 PM
 * @description
 */
public class Concurrence {
    public  boolean v = false;
    // public volatile boolean v = false;
    public static void main(String[] args) {
        Concurrence c = new Concurrence();
        Thread thread1 = new Thread(() -> {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            c.v = true;
            System.out.println(c.v);
        });
        Thread thread2 = new Thread(() -> {
            while (true){
                if (c.v) {
                    System.out.println("hello");
                }
            }
        });
        thread2.start();
        thread1.start();
        while (true) {

        }
    }
}
```
### 2.4.long和double型变量的特殊规则
因为过去对于这两个值特别的存储方式使其并非原子性，但在JDK9以后可以使用-XX:+AlwaysAtomicAccesses开启
### 2.5.原子性、可见性与有序性
原子性：java内存模型中的除lock和unlock外其他6个均具备原子性，synchronized为大范围原子性操作。  
可见性:除了volatile之外，Java还有两个关键字能实现可见性，它们是synchronized和final.  
有序性:volatile和synchronized

## 3.java与线程
### 3.1.内核线程
程序一般不会直接使用内核线程，而是使用内核线程的一种高级接口——轻量级进程(Light Weight Process，LWP)，轻量级进程就是我们通常意义上所讲的线程
### 3.2.线程调度
调度主要方式有两种，分别是协同式(Cooperative Threads-Scheduling)线程调度和抢占式(Preemptive Threads-Scheduling)线程调度。但前者并不稳定。
### 3.3.状态转换
6种线程状态:新建(new)\运行(start)\无限期等待(需被显式唤醒，主要有wait、join未设置参数)\限期等待(sleep、wait、join等)、阻塞(synchronized)、结束(shutdown)

## 4.java与协程
### 4.1.用户线程
用户线程需要自行维护不如内核线程方便使用，但这部分可以交由虚拟机完成，目前仍在fork/join中研发。研发成熟可以让线程切换消耗更低的资源，而且更加轻量仅占1MB
## 5.线程安全
### 5.1.Java语言中的线程安全
#### 5.1.1.不可变
String、final以及基本数据类型
#### 5.1.2.绝对线程安全
synchronized修饰
#### 5.1.3.相对线程安全
例如vector、HashTable等使用同步手段
### 5.2.线程安全的实现方法
#### 5.2.1.互斥同步
synchronized以及JDK5后新起的java.util.concurrent.locks.Lock
#### 5.2.2.非阻塞同步
即CAS等实现
## 6.锁优化(AQS)
### 6.1.自旋锁与自适应自旋
即有限次数的while(true)，优点是避免线程切换
### 6.2.锁消除
例如某个同步方法中全程使用线程安全方法
### 6.3.锁粗化
例如在循环中不停加锁解锁，则直接会移到循环外面
### 6.4.轻量级锁
如果整个过程jvm判断只有无竞争但有多个线程可能会使用时进行替换，由CAS实现，如果竞争过多则变为重量锁
### 6.5.偏向锁
如果整个过程jvm判断没有竞争关系，则进行锁消除处理，在该锁被其它线程获取时依据轻量锁标记判断退化为轻量锁还是重量锁

![偏向锁和轻量锁的关系](http://seal_li.gitee.io/sealbook/pic/jvm_concurrence_lock.png)

## 参考资料
《深入理解Java虚拟机》-周志明