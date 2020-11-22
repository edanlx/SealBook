<center>为什么你写的代码有时候和预期不一致</center>

# 一.友情链接
[目录](https://github.com/edanlx/SealBook/blob/master/catalog.md)  
[视频讲解](https://www.bilibili.com/video/BV11i4y1L7BX/)   
[文字版](https://github.com/edanlx/SealBook/blob/master/jvm/compile.md)

如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。

# 二.前端编译
## 2.1javac编译器
即一般所说的编译，从javac代码的总体结构来看，编译过程大致可以分为1个准备过程和3个处理过程，它们分为  
1)准备过程：初始化插入式注解处理器  
2)解析与填充符号表过程，构造抽象语法树  
3)插入式注解处理器的注解处理过程  
4)分析与字节码生成过程：语法检查->控制流分析->解析语法糖->字节码生成  

语法糖主要有以下几种：泛型、自动拆装箱遍历循环、条件编译  
类型擦除(编译前)

```java
public static void main(String[] args) {
    Map<String,String> map = new HashMap<String,String>(16);
    map.put("name","csx-mg");
    String name = map.get("name");
    System.out.println(name);
}
```
类型擦除(编译后)
```java
public static void main(String[] args) {
    //类型擦除
    Map map = new HashMap();
    map.put("name", "csx-mg");
    //强制转换
    String name = (String)map.get("name");
    System.out.println(name);
}
```
例如如下代码不可编译
```java

public static void method(List<String> list){
    }

    public static void method(List<Integer> list){

    }
```

自动拆装箱
```java
Integer a= 200;
Integer b = 200;
System.out.println(a==b);
```
输出为false 

# 三.后端编译
## 3.1即时编译器
首先主流的java虚拟机同时包含解释器与编译器，这样可以保证同时使用两者的优点。  
热点代码会被编译成本地代码，主要针对方法和被多次执行的循环体，这里还会触发一个有意思的现象栈上替换  
主要有两种热点代码采集方式：基于采样的热点探测、基于计数的热点探测。Hotspot采用了第二种，使用了方法调用计数器(默认10000次)和回边计数器(默认10700)  
## 3.2提前编译器
目的是改善java的启动时间，演变为现在所熟知的JIT
目前也在积极开发更多更优秀的提前编译器，用于匹配新出的一些垃圾回收器
## 3.3编译优化
1.最重要的优化：方法内联：把目标方法的代码原封不动地“复 制”到发起调用的方法之中，避免发生真实的方法调用。  
2.最前沿的优化：逃逸分析：栈上分配(对象往栈上分配)、标量替换(大对象拆成基础数据类型)、同步消除(消除无用的线程同步)  
3.最经典的优化：公共子表达式消除(例如 a=1和b=1先后执行顺序并没有区别，可以先执行初始化a，然后所有用到b的地方都替换为a)  
4.最显著的优化：无用代码消除  
5.java经典优化：数组边界消除检查  
1)数组下标是一个常量，如foo[3]，只要在编译期根据数据流分析来确定foo.length的值，并判断下标“3”没有越界，执行的时候就无须判断了。  
2)数组访问发生在循环之中，并且使用循环变量来进行数组的访问。如果编译器只要通过数据流分析就可以判定循环变量的取值范围永远在区间[0，foo.length)之内，那么在循环中就可以把整个数组的上下界检查消除掉。  
```java
if(foo!=null){
	return foo.value;
}else{
	throw new NullPointException
}
```
3)如果foo几乎不会发生NPE，那么上述方法无疑是浪费了一次判断性能，就会被直接优化成trycatch  
其它优化策略：编译器策略、基于性能监控的优化技术、基于证据的优化技术、数据流敏感重写、语言相关的优化技术、内存及代码位置变换、循环变换、全局代码调整、控制流图变换等  


