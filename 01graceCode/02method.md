# 【优雅代码】02-java传个方法你会吗,不是Method对象
> 欢迎关注b站账号/公众号【六边形战士夏宁】，一个要把各项指标拉满的男人。该文章已在[github目录](https://github.com/edanlx/SealBook)收录。
屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。
* [gitee目录](https://gitee.com/seal_li/SealBook)
* [知乎目录](https://zhuanlan.zhihu.com/p/338222208)
* [csdn目录](https://blog.csdn.net/seal_li/article/details/111415366)
* [博客园目录](https://www.cnblogs.com/sealLee/articles/14748368.html)
* [可直接运行的完整代码](https://github.com/edanlx/TechingCode/tree/master/demoGrace/src/main/java/com/example/demo/lesson/grace/reflect)  
* [视频讲解](https://www.bilibili.com/video/BV1ok4y1q7Be)   
* [上一篇](./01lombok.md)lombok精选注解
* [下一篇](./03optional.md)optional杜绝空指针异常

## 1.背景介绍
在日常代码中有时候近乎避免不了的使用魔法值，但是如果使用传入方法这种方式可以极大的降低魔法值出现的频率并且不用创建静态值。该方法主要参考了mybatisPlus，并在此基础上进行了扩展。  
核心部分(1)IFN,该类java有自带的Function,使用方法一致，可用于各种get方法的传入，如果是set方法则使用Consumer。本质上调用时第一个参数传入对象，和反射的用法非常相似
(2)使用writeReplace，该部分为重写Serializable中的方法
## 2.相关代码  
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

    public static <F, T> String fnToFieldName(IFn<F, T> fn) {
        SerializedLambda serializedLambda = getSerializedLambda(fn);
        String getter = serializedLambda.getImplMethodName();
        String fieldName = "";
        if (getter.startsWith("get")) {
            fieldName = Introspector.decapitalize(getter.replace("get", ""));
        }
        return fieldName;
    }

    public static <F, T> String fnToFnName(IFn<F, T> fn) {
        return getSerializedLambda(fn).getImplMethodName();
    }

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
        System.out.println("注解：" + Reflections.fnToField(ValidatedRequestVO::getStartDate).getAnnotation(org.springframework.data.mongodb.core.mapping.Field.class).value());
    }
}

```

## 3.举例应用  
1. 如视频中所展现的可以取方法/字段的注解
2. 利用反射实现伪代理
3. 最普遍的应用即mybatisPlus的应用，可以动态传入需要的字段和不需要的字段而不用改动sql，以此优化性能  

平铺转树与树转平铺的个人工具类

```java
public class TreeUtils {
    public static void main(String[] args) {
        List<TestTreeObj> list = new ArrayList<TestTreeObj>() {{
            add(TestTreeObj.builder().id(1).build());
            add(TestTreeObj.builder().id(11).pid(1).build());
            add(TestTreeObj.builder().id(12).pid(1).build());
            add(TestTreeObj.builder().id(111).pid(11).build());
            add(TestTreeObj.builder().id(112).pid(11).build());
            add(TestTreeObj.builder().id(121).pid(12).build());
            add(TestTreeObj.builder().id(122).pid(12).build());
            add(TestTreeObj.builder().id(2).build());
            add(TestTreeObj.builder().id(21).pid(2).build());
            add(TestTreeObj.builder().id(22).pid(2).build());
            add(TestTreeObj.builder().id(211).pid(21).build());
            add(TestTreeObj.builder().id(212).pid(21).build());
            add(TestTreeObj.builder().id(221).pid(22).build());
            add(TestTreeObj.builder().id(222).pid(22).build());
        }};
        List<TestTreeObj> treeResult = listToTree(list, TestTreeObj::setTestTreeObj, TestTreeObj::getId, TestTreeObj::getPid, (l) -> l.getPid() == 0);

        List<TestTreeObj> testTreeObjs = new ArrayList<TestTreeObj>() {{
            add(TestTreeObj.builder().id(1).testTreeObj(new ArrayList<TestTreeObj>() {{
                add(TestTreeObj.builder().id(11).testTreeObj(new ArrayList<TestTreeObj>() {{
                    add(TestTreeObj.builder().id(111).build());
                    add(TestTreeObj.builder().id(112).build());
                }}).build());
            }}).build());
        }};
        List<TestTreeObj> result = new ArrayList<>();
        treeToListDeep(testTreeObjs, result, TestTreeObj::getTestTreeObj, (l) -> l.getTestTreeObj() == null);
        List<TestTreeObj> result2 = new ArrayList<>();
        treeToListDeep(testTreeObjs, result2, TestTreeObj::getTestTreeObj, (l) -> l.getPid() == 0);
        System.out.println(result2);
    }

    /**
     * 树转平铺
     * treeToListDeep(testTreeObjs, result, TestTreeObj::getTestTreeObj, (l) -> l.getTestTreeObj() == null);
     *
     * @param source 源数据
     * @param target 目标容器
     * @param childListFn 递归调用方法
     * @param addTargetCondition 添加到容器的判断方法
     * @author 876651109@qq.com
     * @date 2021/3/1 8:19 下午
     */
    public static <F> void treeToListDeep(List<F> source, List<F> target, Function<F, List<F>> childListFn, Predicate<F> addTargetCondition) {
        if (CollectionUtils.isEmpty(source)) {
            return;
        }
        for (F f : source) {
            if (addTargetCondition.test(f)) {
                target.add(f);
            }
            treeToListDeep(childListFn.apply(f), target, childListFn, addTargetCondition);
        }
    }

    /**
     * List<TestTreeObj> treeResult = listToTree(list, TestTreeObj::setTestTreeObj, TestTreeObj::getId, TestTreeObj::getPid, (l) -> l.getPid() == 0);
     *
     * @param source 源数据
     * @param childListFn 设置递归的方法
     * @param idFn 获取id的方法
     * @param pidFn 获取父id的方法
     * @param getRootCondition 获取根节点的提哦啊见
     * @return {@link List<F>}
     * @author 876651109@qq.com
     * @date 2021/3/1 8:18 下午
     */
    public static <F, T> List<F> listToTree(List<F> source, BiConsumer<F, List<F>> childListFn, Function<F, T> idFn, Function<F, T> pidFn, Predicate<F> getRootCondition) {
        List<F> tree = new ArrayList<>();
        Map<T, List<F>> map = new HashMap<>();
        for (F f : source) {
            if (getRootCondition.test(f)) {
                tree.add(f);
            } else {
                List<F> tempList = map.getOrDefault(pidFn.apply(f), new ArrayList<>());
                tempList.add(f);
                map.put(pidFn.apply(f), tempList);
            }
        }
        tree.forEach(l -> assembleTree(l, map, childListFn, idFn));
        return tree;
    }

    private static <F, T> void assembleTree(F current, Map<T, List<F>> map, BiConsumer<F, List<F>> childListFn, Function<F, T> idFn) {
        List<F> fs = map.get(idFn.apply(current));
        if (CollectionUtils.isEmpty(fs)) {
            return;
        }
        childListFn.accept(current, fs);
        fs.forEach(l -> assembleTree(l, map, childListFn, idFn));
    }
}
```