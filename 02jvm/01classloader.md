# 【jvm】01-双亲委派都会说，破坏双亲委派你会吗
> 欢迎关注b站账号/公众号【六边形战士夏宁】，一个要把各项指标拉满的男人。该文章已在[github目录](https://github.com/edanlx/SealBook/blob/master/catalogue/wechat.md)收录。
屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。
* [可直接运行的完整代码](https://github.com/edanlx/TechingCode/tree/master/demoGrace/src/main/java/com/example/demo/lesson/jvm/loader)  
* [视频讲解](https://www.bilibili.com/video/BV1Sz4y1f7FB/)  
* [上一章](../01graceCode/01lombok.md)lombok精选注解
* [下一篇](./02myclassLoader.md)手写自己的类加载器

## 1.类的生命周期
![类的生命周期](http://seal_li.gitee.io/sealbook/pic/jvm_classloader_classLifeCircle.jpg)
首先可以从图中明确类的生命周期  
1）遇到new、getstatic、putstatic或invokestatic这四条字节码指令时，如果类型没有进行过初始化，则需要先触发其初始化阶段。能够生成这四条指令的典型Java代码场景有： 使用new关键字实例化对象的时候、读取或设置一个类型的静态字段的时候、调用一个类型的静态方法的时候。    
2）使用java.lang.reflect包的方法对类型进行反射调用的时候，如果类型没有进行过初始化，则需要先触发其初始化。  
3）当初始化类的时候，如果发现其父类还没有进行过初始化，则需要先触发其父类的初始化。  
4）当虚拟机启动时，用户需要指定一个要执行的主类（包含main()方法的那个类），虚拟机会先 初始化这个主类。  
5）当使用JDK7新加入的动态语言支持时，如果一个java.lang.invoke.MethodHandle实例最后的解析结果为REF_getStatic、REF_putStatic、REF_invokeStatic、REF_newInvokeSpecial四种类型的方法句 柄，并且这个方法句柄对应的类没有进行过初始化，则需要先触发其初始化。  
6）当一个接口中定义了JDK 8新加入的默认方法（被default关键字修饰的接口方法）时，如果有这个接口的实现类发生了初始化，那该接口要在其之前被初始化。

 1. 加载
  1）通过一个类的全限定名来获取定义此类的二进制字节流。  
  2）将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构。   
  3）在内存中生成一个代表这个类的java.lang.Class对象，作为方法区这个类的各种数据的访问入口。
 2. 验证
  1）文件格式验证  
  2）元数据验证  
  3）字节码验证  
  4）符号引用验证
 3. 准备  
    为静态变量分配内存并设置类变量初始值(null)   
    注意这里有个概念为方法区，是逻辑概念  
    jdk8以前使用永久代实现方法区(有专门的永久代区域)，在后面使用了元空间实现方法区，但实例变量物理存储地址其实是放到了堆中。所以说静态变量、字符串等存在堆中也对，存在方法区也对。
 4. 解析  
    解析动作主要针对类或接口、字段、类方法、接口方法、方法类型、方法句柄和调用点限定符这7 
    类符号引用进行，分别对应于常量池的CONSTANT_Class_info、CON-STANT_Fieldref_info、 
    CONSTANT_Methodref_info、CONSTANT_InterfaceMethodref_info、 
    CONSTANT_MethodType_info、CONSTANT_MethodHandle_info、CONSTANT_Dyna-mic_info和 CONSTANT_InvokeDynamic_info 8种常量类型。  
    主要有以下4种解析过程。  
    1）类或接口的解析  
    2）字段解析   
    3）方法解析 
    4）接口方法解析  
 5. 初始化  
 6. 使用  
 7. 卸载
当类被加载后就进入连接阶段，这一阶段包括验证、准备（为静态变量分配内存并设置默认的初始值）和解析（将符号引用替换为直接引用）三个步骤。最后 JVM 对类进行初始化
## 2.jdk8双亲委派模型

![JDK8类加载器](http://seal_li.gitee.io/sealbook/pic/jvm_classloader_JDK8ClassLoaderModel.jpg)

双亲委派核心代码package sun.misc.Launcher;

```java
public Class<?> loadClass(String var1, boolean var2) throws ClassNotFoundException {
            int var3 = var1.lastIndexOf(46);
            if (var3 != -1) {
                SecurityManager var4 = System.getSecurityManager();
                if (var4 != null) {
                    var4.checkPackageAccess(var1.substring(0, var3));
                }
            }

            if (this.ucp.knownToNotExist(var1)) {
                Class var5 = this.findLoadedClass(var1);
                if (var5 != null) {
                    if (var2) {
                        this.resolveClass(var5);
                    }

                    return var5;
                } else {
                    throw new ClassNotFoundException(var1);
                }
            } else {
                return super.loadClass(var1, var2);
            }
        }
```

```java
// 演示的时候漏了一段核心代码 先是一层一层往上抛，如果父加载器找不到则自己加载
protected Class<?> loadClass(String name, boolean resolve)
    throws ClassNotFoundException
{
    synchronized (getClassLoadingLock(name)) {
        // 检查当前类加载器是否已经加载了该类
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            long t0 = System.nanoTime();
            try {
                if (parent != null) {  //如果当前加载器父加载器不为空则委托父加载器加载该类
                    c = parent.loadClass(name, false);
                } else {  //如果当前加载器父加载器为空则委托引导类加载器加载该类
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
                // ClassNotFoundException thrown if class not found
                // from the non-null parent class loader
            }

            if (c == null) {
                // If still not found, then invoke findClass in order
                // to find the class.
                long t1 = System.nanoTime();
                //都会调用URLClassLoader的findClass方法在加载器的类路径里查找并加载该类
                c = findClass(name);

                // this is the defining class loader; record the stats
                sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                sun.misc.PerfCounter.getFindClasses().increment();
            }
        }
        if (resolve) {  //不会执行
            resolveClass(c);
        }
        return c;
    }
}
```



从代码和图中可知所谓双亲委派就是一层一层往上找，每层加载的代码不同避免了重复加载，通常用户所写的代码在系统类加载器



演示代码

```java
package com.example.demo.lesson.jvm.loader;

import sun.misc.Launcher;

import java.net.URL;

public class ClassLoaderExe {
    public static void main(String[] args) {
        // 核心rt.jar中的类加载器 是C++加载的，因此这里为null
        System.out.println(String.class.getClassLoader());
        // 扩展包的加载器 ExtClassLoader
        System.out.println(com.sun.crypto.provider.DESKeyFactory.class.getClassLoader());
        // 应用加载器 AppClassLoader
        System.out.println(ClassLoaderExe.class.getClassLoader());


        System.out.println("");

        // 获取系统ClassLoader
        ClassLoader appClassLoader = ClassLoader.getSystemClassLoader();
        // appClassLoader的父加载器
        ClassLoader extClassLoader = appClassLoader.getParent();
        // extClassLoader的父加载器
        ClassLoader boostrapClassLoader = extClassLoader.getParent();

        System.out.println("the bootstrapLoader : " + boostrapClassLoader);
        System.out.println("the extClassloader : " + extClassLoader);
        System.out.println("the appClassLoader : "+ appClassLoader);

        System.out.println("==============bootstrapLoader加载的文件====================");


        URL[] urLs = Launcher.getBootstrapClassPath().getURLs();
        for (int i = 0; i < urLs.length; i++) {
            System.out.println(urLs[i]);
        }
        System.out.println("");


        System.out.println("==============extClassloader加载的文件====================");
        System.out.println(System.getProperty("java.ext.dirs"));

        System.out.println("");


        System.out.println("==============appClassLoader 加载的文件====================");
        System.out.println(System.getProperty("java.class.path"));
    }
}

```



## 3.jdk9破坏双亲委派模型

![JDK9类加载器](http://seal_li.gitee.io/sealbook/pic/jvm_classloader_JDK9ClassLoaderModel.jpg)

核心代码在jdk11进行了迁移jdk.internal.loader.ClassLoaders

```java
	private static final BootClassLoader BOOT_LOADER;
    private static final PlatformClassLoader PLATFORM_LOADER;
    private static final AppClassLoader APP_LOADER;

    // Creates the built-in class loaders.
    static {
        // -Xbootclasspath/a or -javaagent with Boot-Class-Path attribute
        String append = VM.getSavedProperty("jdk.boot.class.path.append");
        BOOT_LOADER =
            new BootClassLoader((append != null && !append.isEmpty())
                ? new URLClassPath(append, true)
                : null);
        PLATFORM_LOADER = new PlatformClassLoader(BOOT_LOADER);

        // A class path is required when no initial module is specified.
        // In this case the class path defaults to "", meaning the current
        // working directory.  When an initial module is specified, on the
        // contrary, we drop this historic interpretation of the empty
        // string and instead treat it as unspecified.
        String cp = System.getProperty("java.class.path");
        if (cp == null || cp.isEmpty()) {
            String initialModuleName = System.getProperty("jdk.module.main");
            cp = (initialModuleName == null) ? "" : null;
        }
        URLClassPath ucp = new URLClassPath(cp, false);
        APP_LOADER = new AppClassLoader(PLATFORM_LOADER, ucp);
    }
```

从图和代码中可知在java模块化后通过判断该模块由哪个类加载器加载就直接加载不用再一层层往上找



演示代码

```java
package com.example.demo.lesson.jvm.loader;


import java.net.URL;

public class ClassLoaderPlant {
    public static void main(String[] args) {
        System.out.println(String.class.getClassLoader());
        System.out.println(ClassLoaderPlant.class.getClassLoader());


        System.out.println("");

        // 获取系统ClassLoader
        ClassLoader appClassLoader = ClassLoader.getSystemClassLoader();
        // appClassLoader的父加载器
        ClassLoader platformClassLoader = appClassLoader.getParent();
        // platformClassLoader的父加载器
        ClassLoader boostrapClassLoader = platformClassLoader.getParent();

        System.out.println("the bootstrapLoader : " + boostrapClassLoader);
        System.out.println("the extClassLoader : "+ platformClassLoader);
        System.out.println("the appClassLoader : "+ appClassLoader);
    }
}

```

## 参考资料
《深入理解Java虚拟机》-周志明