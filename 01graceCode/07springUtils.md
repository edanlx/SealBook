# 【优雅代码】07spring下的优秀工具类
> 欢迎关注b站账号/公众号【六边形战士夏宁】，一个要把各项指标拉满的男人。该文章已在[github目录](https://github.com/edanlx/SealBook)收录。
屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。
* [公众号目录](https://gitee.com/seal_li/SealBook/blob/master/catalogue/wechat.md)
* [可直接运行的完整代码](https://github.com/edanlx/TechingCode/tree/master/demoGrace/src/main/java/com/example/demo/lesson/grace/spring)  
* [上一篇](./06apacheUtils.md)apache下的优秀工具类
* [下一篇](./08commonPool.md)构建自己的连接池

## 1.反射相关(重要)
### 1.1背景
ReflectionUtils和AnnotationUtils，各种姿势进行反射，最关键是不用抛异常出去，相当舒心
### 1.2使用
```java
@Nullable
private static void reflectionExample() {
    // 各种反射姿势，可以躲过代码检查
    Method reflectionExample = ReflectionUtils.findMethod(SpringExample.class, "reflectionExample");
    System.out.println(reflectionExample);
    Field testName = ReflectionUtils.findField(SpringExample.class, "testName");
    ReflectionUtils.makeAccessible(testName);
    System.out.println(testName);
    Nullable annotation = AnnotationUtils.findAnnotation(reflectionExample, Nullable.class);
    System.out.println(annotation);
}
```

## 2.cglib相关(重要)
### 2.1背景
cglib作为直接生成字节码，速度上不言而喻，而作为spring引入不用额外引入再次加分，以下介绍个人觉得用起来非常舒服的姿势(代理模式不介绍，直接用springAop即可)
### 2.2使用
```java
/private static void cglibExample() {
    // 注意cglib是对字节码操作，代理模式就不在这里介绍了，spring aop非常好用了，不过这个是spring带的cglib实际上不是spring的东西

    // 创建不可变bean，简直太好用了，避免缓存被别人瞎改
    SpringExample bean = new SpringExample();
    bean.setTestName("hello");
    SpringExample immutableBean = (SpringExample) ImmutableBean.create(bean);
    // 下面这步会直接报错
    // immutableBean.setTestName("123");

    // 对象复制，目前最快的复制,第一个source,第二个atrget,如果要复制list需要自行循环
    BeanCopier copier = BeanCopier.create(SpringExample.class, SpringExample.class, false);
    SpringExample sourceBean = new SpringExample();
    SpringExample targetBean = new SpringExample();
    sourceBean.setTestName("123");
    targetBean.setTestName("223");
    copier.copy(sourceBean, targetBean, null);
    System.out.println(targetBean);
    // 注意第一步可以static缓存起来，BulkBean虽然可以处理复杂逻辑，但是个人认为复杂逻辑就老实写代码实现，用这个反而累赘

    // 使用转换器实现属性合并，也是相当给力
    BeanCopier copier2 = BeanCopier.create(SpringExample.class, SpringExample.class, true);
    SpringExample sourceBean2 = new SpringExample();
    SpringExample targetBean2 = new SpringExample();
    targetBean2.setTestName("223");
    copier2.copy(sourceBean2, targetBean2, (source, aClass, target) -> ObjectUtils.defaultIfNull(source, target));
    System.out.println(sourceBean2);
    System.out.println(targetBean2);

    // 对象转map，可以重新封装，也可以直接用
    Map<String, Object> map = new HashMap<>();
    map.putAll(BeanMap.create(targetBean));
    Map<String, Object> beanMap = BeanMap.create(targetBean);
    System.out.println(map);
    System.out.println(beanMap);

    // map转对象
    SpringExample springExampleFinal = new SpringExample();
    BeanMap.create(springExampleFinal).putAll(map);
    System.out.println(springExampleFinal);
}
```

## 3.spring相关(重要)
### 3.1背景
列了4个比较好用的工具类，request非常棒，扩展的话可以在任意位置获取到用户，StopWatch在流程繁杂的方法中可以直观输出速度消耗的百分比，花板子,但是我很喜欢
### 3.2使用
```java
private static void springExample() {
    // 获取request
    HttpServletRequest request = ((ServletRequestAttributes) RequestContextHolder.getRequestAttributes()).getRequest();
    // 获取cookie
    Cookie cookie = WebUtils.getCookie(request, "hello");
    // 转义url
    UriUtils.decode("", StandardCharsets.UTF_8);
    UriUtils.encode("", StandardCharsets.UTF_8);
    // 记录时间戳
    StopWatch sw = new StopWatch("startTest");
    sw.start("step 1");
    sw.stop();
    sw.start("step 2");
    sw.stop();
    System.out.println(sw.prettyPrint());
}
```

## 4.bean相关(重要)
### 4.1背景
列了3个使用姿势，应该是使用评率最高的3个
### 4.2使用
```java
private void beanExample(){
    // 获取bean
    SpringExample bean = ac.getBean(SpringExample.class);
    // 根据继承或实现获取bean
    Map<String, SpringExample> beansOfType = ac.getBeansOfType(SpringExample.class);
    // 获取当前代理对象，service层常用
    AopContext.currentProxy();
}
```

## 5.其它
### 5.1背景
其它spring中的东西，基本都是下位替代,没其它姿势的时候可以勉强用一下
### 5.2使用
```java
private void otherExample(){
    // 其下有各种转义，用处有限
    System.out.println(StringEscapeUtils.class);
    // 资源加载工具类，但是不如springBoot注解好用
    System.out.println(ResourceUtils.class);
    // 读取properties，马马虎虎的东西，java自带的也不差
    System.out.println(LocalizedResourceHelper.class);
    // apache的IO包可太好用了,以及很多其它和apache重复的就不介绍了
    System.out.println(FileCopyUtils.class);
}
```