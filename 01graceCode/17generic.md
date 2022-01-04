# 【优雅代码】17-当泛型遇上多态
> 欢迎关注b站账号/公众号【六边形战士夏宁】，一个要把各项指标拉满的男人。该文章已在[github目录](https://github.com/edanlx/SealBook/blob/master/catalogue/wechat.md)收录。
屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。
* [可直接运行的完整代码](https://github.com/edanlx/TechingCode/tree/master/demoGrace/src/main/java/com/example/demo/lesson/grace/generic)
* [上一篇](./16bloomAndRate.md)guava布隆过滤与限流算法源码解析
* [下一篇](./16method.md)反射与序列换联合使用(你没用过的反射技巧-上)


## 1.背景
日常开发中单独用泛型没有问题，单独用多态没有问题，但是当两者相遇就会变成神奇的java
## 2.泛型
### 2.1泛型概念
java在一开始是不支持泛型的，是后天支持的，为了支持它做了两件事，在编译阶段用泛型概念进行框定，在编译完成后会进行泛型擦除，在获取时进行强转。如下列伪代码。
```java
// 编译前
List<Integer> list = new ArrayList<>();
Integer integer = list.get(0);
// ===================
// 编译后
List list = new ArrayList();
Integer integer = (Integer) list.get(0);
// 无论list的泛型填什么在获取class的时候都是java.util.ArrayList
System.out.println(list.getClass());
```
### 2.2泛型类
```java
// 直接贴List的代码，使用上就是在创建对象时加上尖括号
public interface List<E> {}
```
### 2.3泛型方法
```java
// 写一个泛型方法如下，非常的简单
public static<F extends String>  void testMethod(List<F> obj) {
}
// 值得注意的是普通方法调用和泛型方法调用传泛型是一样的方式，如下方法本身不是泛型方法但依然可以送
new StringBuilder().<String, Object>append("1");
```
### 2.4泛型与通配符
PECS原则
1. 如果要从集合中读取类型T的数据，并且不能写入，可以使用 ? extends 通配符；(Producer Extends)
2. 如果要从集合中写入类型T的数据，并且不需要读取，可以使用 ? super 通配符；(Consumer Super)
3. 如果既要存又要取，那么就不要使用任何通配符。
从使用上看通配符好像只能修饰形参，但没有找到相关介绍

### 2.5获取泛型类型
目前方式都是通过固定类，固定属性进行反射，这样可以获得List<String>中的String，但如果原始类就是List<T>这样就无法获取到了。类似如下的处理方式
```java
Field stringListField = Test.class.getDeclaredField("stringList");
ParameterizedType stringListType = (ParameterizedType) stringListField.getGenericType();
Class<?> stringListClass = (Class<?>) stringListType.getActualTypeArguments()[0];
System.out.println(stringListClass); // class java.lang.String.
```

## 3.重写
### 3.1重写概念
方法覆盖必须满足下列条件
1. 子类的方法的名称及参数必须和所覆盖的方法相同
2. 子类的方法返回类型必须和所覆盖的方法相同(JDK1.5以后可以返回该类型的子类，比如父类返回Object，子类可以返回Integer)
3. 子类方法不能缩小所覆盖方法的访问权限
4. 子类方法不能抛出比所覆盖方法更多的异常
```java
// 如父类有如下方法
public Object testMethod2(Object obj){
        System.out.println("Test4" + obj);
        return obj;
    }
// 子类按如下方式是可以通过校验的
@Override
    public  Integer testMethod2(Object obj){
        System.out.println("Test3" + obj);
        return (Integer) obj;
    }
```

### 3.2泛型与重写
```java
// 如父类有如下方法，以下两种互相不兼容，具体看重载部分
public <T> T testMethod(T obj){
      System.out.println("Test4" + obj);
      return obj;
  }
// 或者如下方法
public Object testMethod(Object obj){
      System.out.println("Test4" + obj);
      return obj;
  }
// 那么子类重写就可以有两种方式写法，当然也可以写Object的子类
  @Override
  public Object testMethod(Object obj){
      System.out.println("Test4" + obj);
      return obj;
  }
// 或者如下
  @Override
public <T> T testMethod(T obj){
      System.out.println("Test4" + obj);
      return obj;
  }
```

## 4.重载
### 4.1重载概念
1. 方法名必须相同
2. 方法的参数签名必须相同
3. 方法的返回类型和方法的修饰符可以不相同
* 根据阿里巴巴java规约，不要去交换形参来重载

### 4.2泛型与重载
```java
// 因为类型擦除，所以下面两种编译结果是一模一样的，所以编译不会通过
public <T> T testMethod(T obj){
      System.out.println("Test4" + obj);
      return obj;
  }
public Object testMethod(Object obj){
      System.out.println("Test4" + obj);
      return obj;
  }
```
## 5.参考资料
《java编程思想第四版》