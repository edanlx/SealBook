# 【优雅代码】18-反射与序列换联合使用(你没用过的反射技巧-上)
> 欢迎关注b站账号/公众号【六边形战士夏宁】，一个要把各项指标拉满的男人。该文章已在[github目录](https://github.com/edanlx/SealBook)收录。
屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。
* [gitee目录](https://gitee.com/seal_li/SealBook)
* [知乎目录](https://zhuanlan.zhihu.com/p/338222208)
* [csdn目录](https://blog.csdn.net/seal_li/article/details/111415366)
* [博客园目录](https://www.cnblogs.com/sealLee/articles/14748368.html)
* [可直接运行的完整代码](https://github.com/edanlx/TechingCode/tree/master/demoGrace/src/main/java/com/example/demo/lesson/grace/reflect)  
* [视频讲解](https://www.bilibili.com/video/BV1ok4y1q7Be)   
* [上一篇](./17generic.md)当泛型遇上多态
* [下一篇](./18treelist.md)list、tree互转通用工具(你没用过的反射技巧-中)

## 1.背景介绍
在日常代码中有时候近乎避免不了的使用魔法值，但是如果使用传入方法这种方式可以极大的降低魔法值出现的频率并且不用创建静态值。该方法主要参考了mybatisPlus，并在此基础上进行了扩展。  
核心部分(1)IFN,该类java有自带的Function,使用方法一致，可用于各种get方法的传入，如果是set方法则使用Consumer。本质上调用时第一个参数传入对象，和反射的用法非常相似
(2)使用writeReplace，该部分为重写Serializable中的方法

## 2.反射的简单使用
```java
public class EnumReflects {

    @Getter
    @AllArgsConstructor
    private enum TestEnum {
        China("ZH", "中国"),
        English("EN", "英国");
        private String language;
        private String name;
    }

    /**
     * 将enum->转为List<Map<K,V>>形式输出，舍弃constant
     *
     * @param enumClass
     * @param <E>
     * @return
     */
    public static <E extends Enum<E>> List<Map<String, Object>> getListMap(Class<E> enumClass) {
        Field[] declaredFields = enumClass.getDeclaredFields();
        List<Field> listField = Arrays.stream(declaredFields).filter(f -> !Modifier.isStatic(f.getModifiers())).collect(Collectors.toList());
        listField.forEach(l -> l.setAccessible(true));
        List<Map<String, Object>> result = Arrays.stream(enumClass.getEnumConstants())
                .map(en -> listField.stream().collect(Collectors.toMap(Field::getName, field -> ReflectionUtils.getField(field, en))))
                .collect(Collectors.toList());
        return result;
    }

    /**
     * 将enum->Map<K,enum>的形式，k以传入的
     *
     * @param enumClass
     * @param <E>
     * @return
     */
    public static <E extends Enum<E>, T> Map<T, E> enumToMap(Class<E> enumClass, Function<E, T> keyFn) {
        return Arrays.stream(enumClass.getEnumConstants()).collect(Collectors.toMap(keyFn, (l) -> (l)));
    }

    public static void main(String[] args) {
        System.out.println(getListMap(TestEnum.class));
        System.out.println(enumToMap(TestEnum.class, TestEnum::getLanguage));
    }
}
```
输出如下利用反射可以轻松实现enum以想要的格式进行输出，如果这里看不明白的话就要补基础知识了，后面肯定不理解。这个也是我个人日常使用的枚举工具类
```text
[{name=中国, language=ZH}, {name=英国, language=EN}]
{EN=English, ZH=China}
```
## 3.序列化反射相关代码  
* IFn
```java
package com.example.demo.bean;

import java.io.Serializable;

/**
 * F 传入类的类型，T返回类型
 * 类比get方法第一个参数是隐形的this,那么source就是此对象,F就是此对象的类型
 * @author seal
 */
@FunctionalInterface
public interface IFn<F, T> extends Serializable {
    T apply(F source);
}

```

* IFnVoid
```java
package com.example.demo.bean;

import java.io.Serializable;

/**
 * F 传入类的类型，T第一个参数的类型
 * 类比set方法第一个参数是隐形的this,那么source就是此对象,arg就是第一个要传入参数
 * @author seal
 */
@FunctionalInterface
public interface IFnVoid<F,T> extends Serializable {
    void apply(F source,T arg);
}

```

* Reflections
```java
package com.example.demo.bean;

import lombok.extern.slf4j.Slf4j;
import org.springframework.data.mongodb.core.mapping.Field;

import java.beans.Introspector;
import java.lang.invoke.SerializedLambda;
import java.lang.reflect.Method;

/**
 * 传入方法后的实现逻辑
 *
 * @author seal 876651109@qq.com
 * @date 7/12/2020 7:20 PM
 */
@Slf4j
public class Reflections {
    private Reflections() {
    }

    /**
     * 通过get方法获得字段名
     */
    public static <F, T> String fnToFieldName(IFn<F, T> fn) {
        SerializedLambda serializedLambda = getSerializedLambda(fn);
        String getter = serializedLambda.getImplMethodName();
        String fieldName = "";
        if (getter.startsWith("get")) {
            fieldName = Introspector.decapitalize(getter.replace("get", ""));
        }
        return fieldName;
    }

    /**
     * 通过get方法获得方法名
     */
    public static <F, T> String fnToFnName(IFn<F, T> fn) {
        return getSerializedLambda(fn).getImplMethodName();
    }

    /**
     * 通过get方法获得method对象
     */
    public static <F, T> Method fnToMethod(IFn<F, T> fn) {
        SerializedLambda serializedLambda = getSerializedLambda(fn);
        Method method = null;
        try {
            method = getClazz(serializedLambda).getDeclaredMethod(serializedLambda.getImplMethodName());
        } catch (NoSuchMethodException e) {
            log.error("", e);
        }
        return method;
    }

    /**
     * 通过get方法获得Field对象
     */
    public static <F, T> Field fnToField(IFn<F, T> fn) {
        SerializedLambda serializedLambda = getSerializedLambda(fn);
        String fieldName = "";
        String getter = serializedLambda.getImplMethodName();
        if (getter.startsWith("get")) {
            fieldName = Introspector.decapitalize(getter.replace("get", ""));
        }
        Class clazz = getClazz(serializedLambda);
        Field field = null;
        try {
            field = clazz.getDeclaredField(fieldName);
            ReflectionUtils.makeAccessible(field);
        } catch (NoSuchFieldException e) {
            log.error("", e);
        }
        return field;
    }

    /**
     * 通过set方法获得字段名
     */
    public static <F, T> String fnToFieldNameVoid(IFnVoid<F, T> fn) {
        SerializedLambda serializedLambda = getSerializedLambda(fn);
        String getter = serializedLambda.getImplMethodName();
        String fieldName = "";
        if (getter.startsWith("set")) {
            fieldName = Introspector.decapitalize(getter.replace("set", ""));
        }
        return fieldName;
    }

    private static SerializedLambda getSerializedLambda(Object fn) {
        SerializedLambda serializedLambda = null;
        try {
            Method method = fn.getClass().getDeclaredMethod("writeReplace");
            method.setAccessible(Boolean.TRUE);
            serializedLambda = (SerializedLambda) method.invoke(fn);
        } catch (NoSuchMethodException | IllegalAccessException | InvocationTargetException e) {
            log.error("", e);
        }
        return serializedLambda;
    }

    private static Class getClazz(SerializedLambda serializedLambda) {
        Class clazz = null;
        try {
            clazz = Class.forName(serializedLambda.getImplClass().replace("/", "."));
        } catch (ClassNotFoundException e) {
            log.error("", e);
        }
        return clazz;
    }

    public static void main(String[] args) {
        String fieldName = Reflections.fnToFieldName(ValidatedRequestVO::getStr);
        System.out.println("字段名：" + fieldName);
        String fnName = Reflections.fnToFnName(ValidatedRequestVO::getStr);
        System.out.println("方法名：" + fnName);
        String fieldName2 = Reflections.fnToFieldNameVoid(ValidatedRequestVO::setStr);
        System.out.println("字段名：" + fieldName2);
        Method method = Reflections.fnToMethod(ValidatedRequestVO::getStartDate);
        System.out.println("注解：" + Reflections.fnToField(ValidatedRequestVO::getStartDate).getAnnotation(Nullable.class));
    }
}
```
输出如下，可以通过无需硬编码的形式获取字段名等信息
```text
字段名：str
方法名：getStr
字段名：str
注解：@javax.annotation.Nullable()
```

## 4.实际应用  
从main方法可以看到没有使用任何的硬编码，当字段发生改变的时候可以动态的感知到无法编译通过，因为get、set方法变了。进而可以将很多硬编码信息直接写在类上面，动态获取某个字段的注解信息从而进行解耦。