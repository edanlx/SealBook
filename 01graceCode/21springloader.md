# 【优雅代码】21-spring下的优秀工具类(进阶)
> 欢迎关注b站账号/公众号【六边形战士夏宁】，一个要把各项指标拉满的男人。该文章已在[github目录](https://github.com/edanlx/SealBook/blob/master/catalogue/wechat.md)收录。
> 屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。
>
> * [可直接运行的完整代码](https://github.com/edanlx/TechingCode/tree/master/demoGrace/src/main/java/com/example/demo/lesson/grace/springloader)  
* [上一篇](https://github.com/edanlx/SealBook/blob/master/01graceCode/20treeDeep.md)20-复杂树的回调通用工具(你没用过的反射技巧-下)

## 1.背景介绍
spring中有很多神奇的匹配、加载，它用到的工具类会在该篇章中介绍
## 2.不通过反射获取类信息
### 2.1介绍
class文件是懒加载，而spring是如何不在jvm加载前就获取到类信息的。
### 2.2使用
```java
@SneakyThrows
public static void getForClass() {
    // 和cglib一样基于ASM技术的
    // spring获取类的元数据
    SimpleMetadataReaderFactory simpleMetadataReaderFactory = new SimpleMetadataReaderFactory();
    MetadataReader metadataReader = simpleMetadataReaderFactory.getMetadataReader("com.example.demo.lesson.grace.springloader.SpringLoaderExampleTest");
    ClassMetadata classMetadata = metadataReader.getClassMetadata();
    // 获取类名
    System.out.println(classMetadata.getClassName());
    // 获取子类
    System.out.println(classMetadata.getMemberClassNames()[0]);
    // 获取外部类
    MetadataReader metadataReader2 = simpleMetadataReaderFactory.getMetadataReader("com.example.demo.lesson.grace.springloader.SpringLoaderExampleTest.SpringLoaderExampleTestMember");
    System.out.println(metadataReader2.getClassMetadata().getEnclosingClassName());
    // 获取注解
    AnnotationMetadata annotationMetadata = metadataReader.getAnnotationMetadata();
    for (String annotationType : annotationMetadata.getAnnotationTypes()) {
        System.out.println(annotationType);
    }
    // 判断注解的注解
    System.out.println(annotationMetadata.hasMetaAnnotation(Component.class.getName()));
    // 判断是否有注解
    System.out.println(annotationMetadata.hasAnnotation(Service.class.getName()));
    // 获取有该注解的方法
    Set<MethodMetadata> annotatedMethods = annotationMetadata.getAnnotatedMethods(Valid.class.getName());
    System.out.println();
}
```
## 3.通过匹配符获取文件
### 3.1介绍
spring读取相关文件时使用匹配符使用该功能
### 3.1使用
```java
@SneakyThrows
public static void matchResource() {
    // spring下匹配文件
    ResourcePatternResolver resourcePatternResolver = new PathMatchingResourcePatternResolver();
    // 获取resource下文件
    Resource[] resources = resourcePatternResolver.getResources("com/example/demo/lesson/grace/springloader/*.class");
    System.out.println(resources);
    // 获取jar包文件
    Resource[] resources2 = resourcePatternResolver.getResources("classpath*:org/springframework/core/io/*.class");
    System.out.println(resources2);
}
```
## 4.通过匹配符判断是否符合
### 4.1介绍
springMVC经典使用。不仅如此可以适用范围极广
### 4.2使用
```java
public static void pathMatch() {
    // springMVC路径判断
    AntPathMatcher matcherDot = new AntPathMatcher(".");
    String pathDot = "com.example.demo.lesson.grace.springloader.SpringLoaderExampleTest";
    // true
    System.out.println("step1");
    System.out.println(matcherDot.match("com.example.demo.lesson.grace.springloader.*", pathDot));
    // false
    System.out.println("step2");
    System.out.println(matcherDot.match("com.example.demo.lesson.grace.springloader2.*", pathDot));
    // true
    System.out.println("step3");
    System.out.println(matcherDot.match("com.example.demo.lesson.grace.springloader.S?????LoaderExampleTest", pathDot));

    AntPathMatcher matcherSep = new AntPathMatcher("/");
    String pathSep = "com/example/demo/lesson/grace/springloader/SpringLoaderExampleTest";
    // true
    System.out.println("step4");
    System.out.println(matcherSep.match("com/example/demo/lesson/grace/**", pathSep));
    System.out.println("step5");
    // true
    System.out.println(matcherSep.match("com/example/demo/lesson/grace/springloader/{sp}", pathSep));

}
```
## 5.匹配符替换字符串
### 5.1介绍
spring相关配置文件经典使用工具
### 5.2使用
```java
public static void matchProperty() {
    PropertyPlaceholderHelper helper = new PropertyPlaceholderHelper("${", "}");
    String text = "user=${user},name=${name}";
    Properties props = new Properties();
    props.setProperty("user", "siri");
    props.setProperty("name", "apple");
    // user=siri,name=apple
    System.out.println(helper.replacePlaceholders(text, props));
}
```

## 6.泛型工具类
### 6.1介绍
spring简化获取泛型类型的操作
### 6.2使用
```java
public static void getType() {
    Class<?> aClass = GenericTypeResolver.resolveTypeArgument(SpringLoaderExampleTest.class, ISpringLoaderExampleTest.class);
    // class java.lang.String
    System.out.println(aClass);
}
```