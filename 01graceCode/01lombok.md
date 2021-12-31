# 【优雅代码】01-lombok精选注解及原理
> 欢迎关注b站账号/公众号【六边形战士夏宁】，一个要把各项指标拉满的男人。该文章已在[github目录](https://github.com/edanlx/SealBook)收录。
屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。
* [公众号目录](https://gitee.com/seal_li/SealBook/catalogue/wechat.md)
* [可直接运行的完整代码](https://github.com/edanlx/TechingCode/tree/master/demoGrace/src/main/java/com/example/demo/lesson/grace/lombok) 
* [视频讲解](https://www.bilibili.com/video/BV13m4y1Q7GD/)
* [下一篇](./02junit.md)自动化工具合集介绍

## 1.背景介绍
在日常开发中免不了进行一些繁琐的代码自动生成，虽然ide的功能已然非常强大但是并不能够做到动态，lombok可以非常好的解决这个问题。它会在生成class文件时将其进行编译成平常所写的代码,这里介绍一些我个人觉得比较好用的注解
## 2.lombok
先上[官网](https://projectlombok.org/)地址。如果想了解更多注解可以去[https://projectlombok.org/features/all](https://projectlombok.org/features/all)
### 2.1.get/set注解(重要)
此部分注解有@Data、@Getter、@Setter,一般普通Bean对象会使用@Data注解(里面已经包含另外两个注解)，如果是enum则使用@Getter注解
```java
@Data
static class DataBean {
    private String name;
}
// 使用方法如下
public static void DataBeanExample() {
    log.info(new DataBean().getName());
}
```
***生成核心代码如下***
```java
static class DataBean {
    private String name;

    public DataBean() {
    }

    public String getName() {
        return this.name;
    }

    public void setName(final String name) {
        this.name = name;
    }
    ...
}
```
### 2.2.常规构造方法注解(重要)
此部分注解包含@NoArgsConstructor无参构造、@AllArgsConstructor所有参构造、@EqualsAndHashCode、@ToString
```java
@NoArgsConstructor
@AllArgsConstructor
@EqualsAndHashCode
@ToString
static class ConstructorBean {
    private String name1;
    private String name2;
}
// 使用方法如下
public static void ConstructorExample() {
    log.info(new ConstructorBean("1", "2").toString());
}
```
***生成核心代码如下***
```java
static class ConstructorBean {
    private String name1;
    private String name2;

    public ConstructorBean() {
    }

    public ConstructorBean(final String name1, final String name2) {
        this.name1 = name1;
        this.name2 = name2;
    }

    public boolean equals(final Object o) {
        //....
    }

    protected boolean canEqual(final Object other) {
        return other instanceof LombokExample.ConstructorBean;
    }

    public int hashCode() {
        int PRIME = true;
        int result = 1;
        Object $name1 = this.name1;
        int result = result * 59 + ($name1 == null ? 43 : $name1.hashCode());
        Object $name2 = this.name2;
        result = result * 59 + ($name2 == null ? 43 : $name2.hashCode());
        return result;
    }

    public String toString() {
        return "LombokExample.ConstructorBean(name1=" + this.name1 + ", name2=" + this.name2 + ")";
    }
}
```
### 2.4.build构造方法(重要)
此部分注解包含@Builder
```java
@Builder
@ToString
static class BuilderBean {
    private String name1;
    private String name2;
}
// 使用方式如下
public static void BuilderExample() {
    log.info(BuilderBean.builder().name1("1").name2("2").build().toString());
}
```
***生成核心代码如下***
```java
static class BuilderBean {
    private String name1;
    private String name2;

    BuilderBean(final String name1, final String name2) {
        this.name1 = name1;
        this.name2 = name2;
    }

    public static LombokExample.BuilderBean.BuilderBeanBuilder builder() {
        return new LombokExample.BuilderBean.BuilderBeanBuilder();
    }

    public String toString() {
        return "LombokExample.BuilderBean(name1=" + this.name1 + ", name2=" + this.name2 + ")";
    }

    public static class BuilderBeanBuilder {
        private String name1;
        private String name2;

        BuilderBeanBuilder() {
        }

        public LombokExample.BuilderBean.BuilderBeanBuilder name1(final String name1) {
            this.name1 = name1;
            return this;
        }

        public LombokExample.BuilderBean.BuilderBeanBuilder name2(final String name2) {
            this.name2 = name2;
            return this;
        }

        public LombokExample.BuilderBean build() {
            return new LombokExample.BuilderBean(this.name1, this.name2);
        }

        public String toString() {
            return "LombokExample.BuilderBean.BuilderBeanBuilder(name1=" + this.name1 + ", name2=" + this.name2 + ")";
        }
    }
}
```
### 2.5.链式构造方法
此部分注解包含@Accessors,但是因为改写了set方法的返回值，有些时候会和其它bean工具类不兼容，一般不建议使用
```java
@Accessors(chain = true)
@Setter
@ToString
static class ChainBean {
    private String name1;
    private String name2;
}
// 使用方式如下
public static void ChainExample() {
    log.info(new ChainBean().setName1("1").setName2("2").toString());
}
```
***生成核心代码如下***
```java
static class ChainBean {
    private String name1;
    private String name2;

    ChainBean() {
    }

    public LombokExample.ChainBean setName1(final String name1) {
        this.name1 = name1;
        return this;
    }

    public LombokExample.ChainBean setName2(final String name2) {
        this.name2 = name2;
        return this;
    }

    ...
}
```
### 2.6.日志注解(重要)
此部分注解包含@Slf4j，其它的注解都不重要，这个会自动根据引入包进行选择
```java
@Slf4j
public class LombokExample {
	public static void main(String[] args) {
        log.info("123");
    }
}
```
***生成核心代码如下***
```java
private static final Logger log = LoggerFactory.getLogger(LombokExample.class);
```
### 2.7.关闭流
@Cleanup,这个可以帮助关闭流，需要注意的是需要对其捕获IO异常。虽然不错，但是有了trywith的写法以后就用的不多了。
```java
public static void CloseExample() throws FileNotFoundException {
    try {
        @Cleanup FileInputStream fileInputStream = new FileInputStream("");
    } catch (IOException e) {

    }
}
```
***生成核心代码如下***
可以看到会帮我们进行close。不过并没有在finally里面不建议使用
```java
public static void CloseExample() throws FileNotFoundException {
    try {
        FileInputStream fileInputStream = new FileInputStream("");
        if (Collections.singletonList(fileInputStream).get(0) != null) {
            fileInputStream.close();
        }
    } catch (IOException var1) {
    }

}
```
这里推荐try-with-resources写法
```java
public static void CloseExample2() {
    try (FileInputStream fileInputStream = new FileInputStream("")) {
        log.info("123");
    } catch (IOException e) {

    }
}
```
***生成核心代码如下***
可以看到是在finally里面关闭了流，并且各种判断非常全面
```java
public static void CloseExample2() {
    try {
        FileInputStream fileInputStream = new FileInputStream("");
        Throwable var1 = null;

        try {
            log.info("123");
        } catch (Throwable var11) {
            var1 = var11;
            throw var11;
        } finally {
            if (fileInputStream != null) {
                if (var1 != null) {
                    try {
                        fileInputStream.close();
                    } catch (Throwable var10) {
                        var1.addSuppressed(var10);
                    }
                } else {
                    fileInputStream.close();
                }
            }

        }
    } catch (IOException var13) {
    }

}
```
### 2.8.异常注解
在捕获编译时异常的时候比较好用，但是现在越来越多的工具类都对编译异常捕获了，它的出场机会并不多
```java
@SneakyThrows({IOException.class})
public static void ExceptionExample() {
    CloseExample();
}
```
***生成核心代码如下***
唯一的功能就是捕获编译时异常，不用手动去写
```java
public static void ExceptionExample() {
    try {
        CloseExample();
    } catch (IOException var1) {
        throw var1;
    }
}
```
## 3.delombok
如果后面不想用注解则需要使用delombok[官网](https://projectlombok.org/features/delombok)
* 方法一
在新版的idea中refactor->delombok已经集成了delombok，可以直接还原算是比较方便的
![delombok](http://seal_li.gitee.io/sealbook/pic/grace_lombok01.png)
* 方法二

```sh
java -jar lombok.jar delombok src -d src-delomboked
```
注意替换包名
