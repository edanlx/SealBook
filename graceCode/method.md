<center>java传个方法你会吗,不是Method对象</center>

# 友情链接
[目录](https://github.com/edanlx/SealBook/blob/master/catalog.md) 
[可直接运行的完整代码](https://github.com/edanlx/TechingCode/tree/master/demoGrace/src/main/java/com/example/demo/bean)  
[视频讲解](https://www.bilibili.com/video/BV1ok4y1q7Be)   
[文字版](https://github.com/edanlx/SealBook/blob/master/graceCode/method.md)

如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。

# 背景介绍
在日常代码中有时候近乎避免不了的使用魔法值，但是如果使用传入方法这种方式可以极大的降低魔法值出现的频率并且不用创建静态值。该方法主要参考了mybatisPlus，并在此基础上进行了扩展。
# 相关代码  
FnConverter
```java
package com.example.demo.bean;

import com.example.demo.entity.ValidatedRequestVO;

/**
 * @author seal 876651109@qq.com
 * @description
 * @date 7/12/2020 7:20 PM
 */
public class FnConverter<F, T> {
    /**
     * 传入方法返回字段名
     *
     * @param fn 方法
     * @return 字段名
     * @author seal 876651109@qq.com
     * @date 7/12/2020 7:32 PM
     */
    public String fnToFieldName(IFn<F, T> fn) {
        return Reflections.fnToFieldName(fn);
    }

    /**
     * 传入方法返回方法名
     *
     * @param fn 方法
     * @return 方法名
     * @author seal 876651109@qq.com
     * @date 7/12/2020 7:32 PM
     */
    public String fnToFnName(IFn<F, T> fn) {
        return Reflections.fnToFnName(fn);
    }

    /**
     * 传入方法返回注解
     *
     * @param fn 方法
     * @return mongo注解
     * @author seal 876651109@qq.com
     * @date 7/12/2020 7:32 PM
     */
    public String fnToMongoName(IFn<F, T> fn) {
        return Reflections.fnToMongoName(fn);
    }

    public static void main(String[] args) {
        FnConverter<ValidatedRequestVO, Object> fnConverter = new FnConverter();
        String fieldName = fnConverter.fnToFieldName(ValidatedRequestVO::getStr);
        System.out.println("字段名：" + fieldName);
        String fnName = fnConverter.fnToFnName(ValidatedRequestVO::getStr);
        System.out.println("方法名：" + fnName);

        FnConverter<String, Object> fnConverter2 = new FnConverter();
        String fieldName2 = fnConverter2.fnToFieldName(new ValidatedRequestVO()::setStr);
        System.out.println("字段名：" + fieldName2);
        String fnName2 = fnConverter2.fnToFnName(new ValidatedRequestVO()::setStr);
        System.out.println("方法名：" + fnName2);

        FnConverter<ValidatedRequestVO, Object> fnConverter3 = new FnConverter();
        String fieldName3 = fnConverter3.fnToMongoName(ValidatedRequestVO::getStartDate);
        System.out.println("字段名：" + fieldName3);
    }
}

```

IFn
```java
package com.example.demo.bean;

import java.io.Serializable;

/**
 * F 传入类型，T返回类型
 * @author seal
 */
@FunctionalInterface
public interface IFn<F, T> extends Serializable {
    T apply(F source);
}

```

Reflections
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

    public static String fnToFieldName(IFn fn) {
        try {
            Method method = fn.getClass().getDeclaredMethod("writeReplace");
            method.setAccessible(Boolean.TRUE);
            SerializedLambda serializedLambda = (SerializedLambda) method.invoke(fn);
            String getter = serializedLambda.getImplMethodName();
            String fieldName = "";
            if (getter.startsWith("get")) {
                fieldName = Introspector.decapitalize(getter.replace("get", ""));
            } else {
                fieldName = Introspector.decapitalize(getter.replace("set", ""));
            }
            return fieldName;
        } catch (ReflectiveOperationException e) {
            log.warn(String.format("%s:%s",
                    Thread.currentThread().getStackTrace()[1].getMethodName(), e.getMessage()), e);
        }
        return "";
    }

    public static String fnToFnName(IFn fn) {
        try {
            Method method = fn.getClass().getDeclaredMethod("writeReplace");
            method.setAccessible(Boolean.TRUE);
            SerializedLambda serializedLambda = (SerializedLambda) method.invoke(fn);
            return serializedLambda.getImplMethodName();
        } catch (ReflectiveOperationException e) {
            log.warn(String.format("%s:%s",
                    Thread.currentThread().getStackTrace()[1].getMethodName(), e.getMessage()), e);
        }
        return "";
    }

    public static String fnToMongoName(IFn fn) {
        try {
            Method method = fn.getClass().getDeclaredMethod("writeReplace");
            method.setAccessible(Boolean.TRUE);
            SerializedLambda serializedLambda = (SerializedLambda) method.invoke(fn);
            String getter = serializedLambda.getImplMethodName();
            String fieldName = "";
            if (getter.startsWith("get")) {
                fieldName = Introspector.decapitalize(getter.replace("get", ""));
            } else {
                fieldName = Introspector.decapitalize(getter.replace("set", ""));
            }
            Field field = Class.forName(serializedLambda.getImplClass().replace("/", ".")).getDeclaredField(fieldName).getAnnotation(Field.class);
            return field == null ? fieldName : field.value();
        } catch (ReflectiveOperationException e) {
            log.warn(String.format("%s:%s",
                    Thread.currentThread().getStackTrace()[1].getMethodName(), e.getMessage()), e);
        }
        return "";
    }
}

```
# 举例应用  
1. 如视频中所展现的可以取方法/字段的注解
2. 利用反射实现伪代理
3. 最普遍的应用即mybatisPlus的应用，可以动态传入需要的字段和不需要的字段而不用改动sql，以此优化性能