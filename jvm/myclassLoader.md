<center>手写自己的类加载器</center>

# 友情链接
[目录](https://github.com/edanlx/SealBook/blob/master/catalog.md)  
[可直接运行的完整代码](https://github.com/edanlx/TechingCode/tree/master/demoGrace/src/main/java/com/example/demo/lesson/jvm/myloader)  
[视频讲解](https://www.bilibili.com/video/BV1Y54y1274Y/)   
[文字版](https://github.com/edanlx/SealBook/blob/master/jvm/myclassLoader.md)

如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。

# 简单手写自己的类加载器

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

# 加载一个和jdk中同名的类

创建一个java.lang.Byte的类，然后用自己的类加载器去加载，然后就触发了jdk的沙箱机制，报了一个安全错误

# tomcat的类加载器逻辑

如图所示，这里的核心其实就是每一个war包的代码即使含有同名的类也可以加载
做法如下，自己的classloader复制一份然后加载同一个class文件，可以看到启用的是不同的加载器，都被加载了

# 完整代码
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

# ConstantPool
```
Constant pool:
    #1 = Methodref          #2.#3         // java/lang/Object."<init>":()V
    #2 = Class              #4            // java/lang/Object
    #3 = NameAndType        #5:#6         // "<init>":()V
    #4 = Utf8               java/lang/Object
    #5 = Utf8               <init>
    #6 = Utf8               ()V
    #7 = Fieldref           #8.#9         // com/example/demo/lesson/jvm/construction/Master.master2:Lcom/example/demo/lesson/jvm/construction/Master;
    #8 = Class              #10           // com/example/demo/lesson/jvm/construction/Master
    #9 = NameAndType        #11:#12       // master2:Lcom/example/demo/lesson/jvm/construction/Master;
   #10 = Utf8               com/example/demo/lesson/jvm/construction/Master
   #11 = Utf8               master2
   #12 = Utf8               Lcom/example/demo/lesson/jvm/construction/Master;
   #13 = Fieldref           #8.#14        // com/example/demo/lesson/jvm/construction/Master.master4:Lcom/example/demo/lesson/jvm/construction/Master;
   #14 = NameAndType        #15:#12       // master4:Lcom/example/demo/lesson/jvm/construction/Master;
   #15 = Utf8               master4
   #16 = Fieldref           #8.#17        // com/example/demo/lesson/jvm/construction/Master.int2:I
   #17 = NameAndType        #18:#19       // int2:I
   #18 = Utf8               int2
   #19 = Utf8               I
   #20 = Fieldref           #8.#21        // com/example/demo/lesson/jvm/construction/Master.int4:I
   #21 = NameAndType        #22:#19       // int4:I
   #22 = Utf8               int4
   #23 = Methodref          #24.#25       // java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
   #24 = Class              #26           // java/lang/Integer
   #25 = NameAndType        #27:#28       // valueOf:(I)Ljava/lang/Integer;
   #26 = Utf8               java/lang/Integer
   #27 = Utf8               valueOf
   #28 = Utf8               (I)Ljava/lang/Integer;
   #29 = Fieldref           #8.#30        // com/example/demo/lesson/jvm/construction/Master.integerMax2:Ljava/lang/Integer;
   #30 = NameAndType        #31:#32       // integerMax2:Ljava/lang/Integer;
   #31 = Utf8               integerMax2
   #32 = Utf8               Ljava/lang/Integer;
   #33 = Fieldref           #8.#34        // com/example/demo/lesson/jvm/construction/Master.integerMax4:Ljava/lang/Integer;
   #34 = NameAndType        #35:#32       // integerMax4:Ljava/lang/Integer;
   #35 = Utf8               integerMax4
   #36 = Fieldref           #8.#37        // com/example/demo/lesson/jvm/construction/Master.integerMin2:Ljava/lang/Integer;
   #37 = NameAndType        #38:#32       // integerMin2:Ljava/lang/Integer;
   #38 = Utf8               integerMin2
   #39 = Fieldref           #8.#40        // com/example/demo/lesson/jvm/construction/Master.integerMin4:Ljava/lang/Integer;
   #40 = NameAndType        #41:#32       // integerMin4:Ljava/lang/Integer;
   #41 = Utf8               integerMin4
   #42 = String             #43           // str25
   #43 = Utf8               str25
   #44 = Fieldref           #8.#45        // com/example/demo/lesson/jvm/construction/Master.str2:Ljava/lang/String;
   #45 = NameAndType        #46:#47       // str2:Ljava/lang/String;
   #46 = Utf8               str2
   #47 = Utf8               Ljava/lang/String;
   #48 = String             #49           // str27
   #49 = Utf8               str27
   #50 = Fieldref           #8.#51        // com/example/demo/lesson/jvm/construction/Master.str4:Ljava/lang/String;
   #51 = NameAndType        #52:#47       // str4:Ljava/lang/String;
   #52 = Utf8               str4
   #53 = Class              #54           // java/lang/String
   #54 = Utf8               java/lang/String
   #55 = String             #56           // str30
   #56 = Utf8               str30
   #57 = Methodref          #53.#58       // java/lang/String."<init>":(Ljava/lang/String;)V
   #58 = NameAndType        #5:#59        // "<init>":(Ljava/lang/String;)V
   #59 = Utf8               (Ljava/lang/String;)V
   #60 = Fieldref           #8.#61        // com/example/demo/lesson/jvm/construction/Master.strN2:Ljava/lang/String;
   #61 = NameAndType        #62:#47       // strN2:Ljava/lang/String;
   #62 = Utf8               strN2
   #63 = String             #64           // str32
   #64 = Utf8               str32
   #65 = Fieldref           #8.#66        // com/example/demo/lesson/jvm/construction/Master.strN4:Ljava/lang/String;
   #66 = NameAndType        #67:#47       // strN4:Ljava/lang/String;
   #67 = Utf8               strN4
   #68 = String             #69           // str35
   #69 = Utf8               str35
   #70 = Methodref          #53.#71       // java/lang/String.intern:()Ljava/lang/String;
   #71 = NameAndType        #72:#73       // intern:()Ljava/lang/String;
   #72 = Utf8               intern
   #73 = Utf8               ()Ljava/lang/String;
   #74 = Fieldref           #8.#75        // com/example/demo/lesson/jvm/construction/Master.strI2:Ljava/lang/String;
   #75 = NameAndType        #76:#47       // strI2:Ljava/lang/String;
   #76 = Utf8               strI2
   #77 = String             #78           // str37
   #78 = Utf8               str37
   #79 = Fieldref           #8.#80        // com/example/demo/lesson/jvm/construction/Master.strI4:Ljava/lang/String;
   #80 = NameAndType        #81:#47       // strI4:Ljava/lang/String;
   #81 = Utf8               strI4
   #82 = String             #83           // str54
   #83 = Utf8               str54
   #84 = String             #85           // str56
   #85 = Utf8               str56
   #86 = String             #87           // str57
   #87 = Utf8               str57
   #88 = String             #89           // str59
   #89 = Utf8               str59
   #90 = String             #91           // str60
   #91 = Utf8               str60
   #92 = Fieldref           #93.#94       // java/lang/System.out:Ljava/io/PrintStream;
   #93 = Class              #95           // java/lang/System
   #94 = NameAndType        #96:#97       // out:Ljava/io/PrintStream;
   #95 = Utf8               java/lang/System
   #96 = Utf8               out
   #97 = Utf8               Ljava/io/PrintStream;
   #98 = Fieldref           #8.#99        // com/example/demo/lesson/jvm/construction/Master.integerMin1:Ljava/lang/Integer;
   #99 = NameAndType        #100:#32      // integerMin1:Ljava/lang/Integer;
  #100 = Utf8               integerMin1
  #101 = Methodref          #102.#103     // java/io/PrintStream.println:(Z)V
  #102 = Class              #104          // java/io/PrintStream
  #103 = NameAndType        #105:#106     // println:(Z)V
  #104 = Utf8               java/io/PrintStream
  #105 = Utf8               println
  #106 = Utf8               (Z)V
  #107 = Fieldref           #8.#108       // com/example/demo/lesson/jvm/construction/Master.integerMax1:Ljava/lang/Integer;
  #108 = NameAndType        #109:#32      // integerMax1:Ljava/lang/Integer;
  #109 = Utf8               integerMax1
  #110 = Fieldref           #8.#111       // com/example/demo/lesson/jvm/construction/Master.strN1:Ljava/lang/String;
  #111 = NameAndType        #112:#47      // strN1:Ljava/lang/String;
  #112 = Utf8               strN1
  #113 = String             #114          // str29
  #114 = Utf8               str29
  #115 = String             #116          // abc
  #116 = Utf8               abc
  #117 = Fieldref           #8.#118       // com/example/demo/lesson/jvm/construction/Master.master1:Lcom/example/demo/lesson/jvm/construction/Master;
  #118 = NameAndType        #119:#12      // master1:Lcom/example/demo/lesson/jvm/construction/Master;
  #119 = Utf8               master1
  #120 = Fieldref           #8.#121       // com/example/demo/lesson/jvm/construction/Master.master3:Lcom/example/demo/lesson/jvm/construction/Master;
  #121 = NameAndType        #122:#12      // master3:Lcom/example/demo/lesson/jvm/construction/Master;
  #122 = Utf8               master3
  #123 = Fieldref           #8.#124       // com/example/demo/lesson/jvm/construction/Master.int3:I
  #124 = NameAndType        #125:#19      // int3:I
  #125 = Utf8               int3
  #126 = Fieldref           #8.#127       // com/example/demo/lesson/jvm/construction/Master.integerMax3:Ljava/lang/Integer;
  #127 = NameAndType        #128:#32      // integerMax3:Ljava/lang/Integer;
  #128 = Utf8               integerMax3
  #129 = Fieldref           #8.#130       // com/example/demo/lesson/jvm/construction/Master.integerMin3:Ljava/lang/Integer;
  #130 = NameAndType        #131:#32      // integerMin3:Ljava/lang/Integer;
  #131 = Utf8               integerMin3
  #132 = String             #133          // str26
  #133 = Utf8               str26
  #134 = Fieldref           #8.#135       // com/example/demo/lesson/jvm/construction/Master.str3:Ljava/lang/String;
  #135 = NameAndType        #136:#47      // str3:Ljava/lang/String;
  #136 = Utf8               str3
  #137 = String             #138          // str31
  #138 = Utf8               str31
  #139 = Fieldref           #8.#140       // com/example/demo/lesson/jvm/construction/Master.strN3:Ljava/lang/String;
  #140 = NameAndType        #141:#47      // strN3:Ljava/lang/String;
  #141 = Utf8               strN3
  #142 = String             #143          // str34
  #143 = Utf8               str34
  #144 = Fieldref           #8.#145       // com/example/demo/lesson/jvm/construction/Master.strI1:Ljava/lang/String;
  #145 = NameAndType        #146:#47      // strI1:Ljava/lang/String;
  #146 = Utf8               strI1
  #147 = String             #148          // str36
  #148 = Utf8               str36
  #149 = Fieldref           #8.#150       // com/example/demo/lesson/jvm/construction/Master.strI3:Ljava/lang/String;
  #150 = NameAndType        #151:#47      // strI3:Ljava/lang/String;
  #151 = Utf8               strI3
  #152 = Utf8               int1
  #153 = Utf8               ConstantValue
  #154 = Integer            9
  #155 = Integer            10
  #156 = Utf8               str1
  #157 = String             #158          // str24
  #158 = Utf8               str24
  #159 = Utf8               Code
  #160 = Utf8               LineNumberTable
  #161 = Utf8               main
  #162 = Utf8               ([Ljava/lang/String;)V
  #163 = Utf8               StackMapTable
  #164 = Class              #165          // "[Ljava/lang/String;"
  #165 = Utf8               [Ljava/lang/String;
  #166 = Utf8               <clinit>
  #167 = Utf8               SourceFile
  #168 = Utf8               Master.java
```