# 【jvm】02-手写自己的类加载器
> 欢迎关注b站账号/公众号【六边形战士夏宁】，一个要把各项指标拉满的男人。该文章已在[github目录](https://github.com/edanlx/SealBook)收录。
屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。
* [公众号目录](https://gitee.com/seal_li/SealBook/catalogue/wechat.md)
* [可直接运行的完整代码](https://github.com/edanlx/TechingCode/tree/master/demoGrace/src/main/java/com/example/demo/lesson/jvm/myloader)  
* [视频讲解](https://www.bilibili.com/video/BV1Y54y1274Y/) 
* [上一篇](./01classloader.md)双亲委派都会说，破坏双亲委派你会吗
* [下一篇](./03jv.md)string真的会共用常量池吗

## 1.简单手写自己的类加载器

创建一个类继承ClassLoader,然后重写findClass、loadClass这两个方法

findClass的方法
```java
private static byte[] loadByte(String name) throws Exception {
    name = name.replaceAll("\\.", "/");
    FileInputStream fis = new FileInputStream(classPath + "/" + name
            + ".class");
    int len = fis.available();
    byte[] data = new byte[len];
    fis.read(data);
    fis.close();
    return data;
}

protected Class<?> findClass(String name) throws ClassNotFoundException {
    try {
        byte[] data = loadByte(name);
        //defineClass将一个字节数组转为Class对象，这个字节数组是class文件读取后最终的字节数组。
        return defineClass(name, data, 0, data.length);
    } catch (Exception e) {
        e.printStackTrace();
        throw new ClassNotFoundException();
    }
}
```
loadClass直接复制ClassLoader中的方法然后把错误的行数删除就行了  
重点改动代码如下，让该自己的加载器加载指定包的文件，注意需要提前编译java文件为class
```java
if (name.startsWith("com.example.demo.lesson")) {
    c = findClass(name);
} else {
    c = this.getParent().loadClass(name);
}
```

## 2.加载一个和jdk中同名的类

创建一个java.lang.Byte的类，然后用自己的类加载器去加载，然后就触发了jdk的沙箱机制，报了一个安全错误

# tomcat的类加载器逻辑

如图所示，这里的核心其实就是每一个war包的代码即使含有同名的类也可以加载
做法如下，自己的classloader复制一份然后加载同一个class文件，可以看到启用的是不同的加载器，都被加载了

## 3.完整代码
```java
package com.example.demo.lesson.jvm.myloader;

import java.io.FileInputStream;

/**
 * @author seal email:876651109@qq.com
 * @date 2020/9/1 7:23 PM
 * @description
 */
public class MyClassLoaderDemo {

    public static void main(String[] args) throws ClassNotFoundException {
        // Class clazz1 = new MyClassLoader().loadClass("com.example.demo.lesson.jvm.loader.A",false);
        //Class clazz1 = new MyClassLoader().loadClass("java.lang.Byte",false);
        Class clazz2 = new MyClassLoader2().loadClass("com.example.demo.lesson.jvm.loader.A",false);
        System.out.println(new MyClassLoaderDemo().getClass().getClassLoader());
        // System.out.println(clazz1.getClassLoader());
        System.out.println(clazz2.getClassLoader());
    }

    public static String classPath = "F:\\IdeaProjects\\TechingCode\\demoGrace\\src\\main\\java";
    private static byte[] loadByte(String name) throws Exception {
        name = name.replaceAll("\\.", "/");
        FileInputStream fis = new FileInputStream(classPath + "/" + name
                + ".class");
        int len = fis.available();
        byte[] data = new byte[len];
        fis.read(data);
        fis.close();
        return data;
    }

    static class MyClassLoader extends ClassLoader {

        @Override
        protected Class<?> findClass(String name) throws ClassNotFoundException {
            try {
                byte[] data = loadByte(name);
                //defineClass将一个字节数组转为Class对象，这个字节数组是class文件读取后最终的字节数组。
                return defineClass(name, data, 0, data.length);
            } catch (Exception e) {
                e.printStackTrace();
                throw new ClassNotFoundException();
            }
        }

        @Override
        protected Class<?> loadClass(String name, boolean resolve)
                throws ClassNotFoundException {
            synchronized (getClassLoadingLock(name)) {
                // First, check if the class has already been loaded
                Class<?> c = findLoadedClass(name);
                if (c == null) {
                    long t0 = System.nanoTime();
                    if (name.startsWith("java.lang.Byte")) {
                        c = findClass(name);
                    } else {
                        c = this.getParent().loadClass(name);
                    }
                    if (c == null) {
                        // If still not found, then invoke findClass in order
                        // to find the class.
                        long t1 = System.nanoTime();
                        c = findClass(name);

                        // this is the defining class loader; record the stats
                    }
                }
                if (resolve) {
                    resolveClass(c);
                }
                return c;
            }
        }
    }

    static class MyClassLoader2 extends ClassLoader {

        @Override
        protected Class<?> findClass(String name) throws ClassNotFoundException {
            try {
                byte[] data = loadByte(name);
                //defineClass将一个字节数组转为Class对象，这个字节数组是class文件读取后最终的字节数组。
                return defineClass(name, data, 0, data.length);
            } catch (Exception e) {
                e.printStackTrace();
                throw new ClassNotFoundException();
            }
        }

        @Override
        protected Class<?> loadClass(String name, boolean resolve)
                throws ClassNotFoundException {
            synchronized (getClassLoadingLock(name)) {
                // First, check if the class has already been loaded
                Class<?> c = findLoadedClass(name);
                if (c == null) {
                    long t0 = System.nanoTime();
                    if (name.startsWith("com.example.demo.lesson")) {
                        c = findClass(name);
                    } else {
                        c = this.getParent().loadClass(name);
                    }
                    if (c == null) {
                        // If still not found, then invoke findClass in order
                        // to find the class.
                        long t1 = System.nanoTime();
                        c = findClass(name);

                        // this is the defining class loader; record the stats
                    }
                }
                if (resolve) {
                    resolveClass(c);
                }
                return c;
            }
        }
    }
}
```
## 参考资料
《深入理解Java虚拟机》-周志明