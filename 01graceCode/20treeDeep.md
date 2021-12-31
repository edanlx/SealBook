# 【优雅代码】20-复杂树的回调通用工具(你没用过的反射技巧-下)
> 欢迎关注b站账号/公众号【六边形战士夏宁】，一个要把各项指标拉满的男人。该文章已在[github目录](https://github.com/edanlx/SealBook/blob/master/catalogue/wechat.md)收录。
屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。
* [可直接运行的完整代码](https://github.com/edanlx/TechingCode/tree/master/demoGrace/src/main/java/com/example/demo/lesson/grace/reflect)  
* [视频讲解](https://www.bilibili.com/video/BV1ok4y1q7Be)   
* [上一篇](./19treelist.md)list、tree互转通用工具(你没用过的反射技巧-中
* [下一章](../02jvm/01classloader.md)双亲委派都会说，破坏双亲委派你会吗

## 1.背景介绍
类似省市区的树结构在日常开发中经常遇到，如果结构一致则可以利用上一篇介绍的工具类进行循环，但难免会遇到虽然有层级结构，但是结构不一致的，嵌套循环又很深的
## 2.相关代码
```java
public class MultiList {

	public static class TestMultObjectParent {
	    private List<TestMultObject> list;
	}

	public static class TestMultObject {
	    private List<TestMultObjectChild> list;
	}

	public static class TestMultObjectChild {
	    private int id;
	}
	/**
	 * 按照树结构和传参要求自行创建
	 */
    public static class Handle {
        public static void handle(TestMultObjectParent obj1, TestMultObject obj2, TestMultObjectChild obj3, String a, int b) {
            System.out.println(obj3);
        }
    }

    /**
     * @param data       数据
     * @param methodList 按顺序封装每一层获取下一层子集的方式
     * @param clazz      最终回调的类需要按固定格式写唯一一个方法用于回调，前N个参数分别是每一层的对象，最后传入extend扩展对象
     * @param obj        反射方法执行对象，用不上但需要传
     * @param extend     需要参与底层调用的参数,可以将需要的返回值传入等等操作
     */
    public static void handleMultiList(List data, List<Method> methodList, Class clazz, Object obj, List extend) {
        Method invoke = clazz.getMethods()[0];
        List<List> objects = new ArrayList<>();
        handleMultiList(data, methodList, 0, objects);
        for (List object : objects) {
            object.addAll(extend);
            ReflectionUtils.invokeMethod(invoke, obj, object.toArray());
            object.clear();
        }
    }

    private static List<Integer> handleMultiList(List data, List<Method> methodList, int step, List<List> result) {
        List<Integer> intList = new ArrayList<>();
        if (CollectionUtils.isEmpty(data)) {
            return intList;
        }
        data.forEach(l -> {
            if (l == null) {
                return;
            }
            if (step < methodList.size()) {
                List newData = (List) ReflectionUtils.invokeMethod(methodList.get(step), l);
                List<Integer> integers = handleMultiList(newData, methodList, step + 1, result);
                for (Integer integer : integers) {
                    result.get(integer).add(0, l);
                }
                intList.addAll(integers);
            } else {
                result.add(Stream.of(l).collect(Collectors.toList()));
                intList.add(result.size() - 1);
            }
        });
        return intList;
    }
}
```
## 3.核心调用
```java
public static void main(String[] args) {
    List<TestMultObjectParent> data = new ArrayList<TestMultObjectParent>() {{
        add(TestMultObjectParent.builder().list(new ArrayList<TestMultObject>() {{
            add(TestMultObject.builder().list(new ArrayList<TestMultObjectChild>() {{
                add(new TestMultObjectChild(1));
                add(new TestMultObjectChild(2));
            }}).build());
            add(TestMultObject.builder().list(new ArrayList<TestMultObjectChild>() {{
                add(new TestMultObjectChild(4));
                add(new TestMultObjectChild(5));
            }}).build());
            add(TestMultObject.builder().list(new ArrayList<TestMultObjectChild>() {{
                add(new TestMultObjectChild(7));
                add(new TestMultObjectChild(8));
            }}).build());
            add(null);
        }}).build());
        add(TestMultObjectParent.builder().list(new ArrayList<TestMultObject>() {{
            add(TestMultObject.builder().list(new ArrayList<TestMultObjectChild>() {{
                add(new TestMultObjectChild(11));
                add(new TestMultObjectChild(12));
            }}).build());
            add(TestMultObject.builder().list(new ArrayList<TestMultObjectChild>() {{
                add(new TestMultObjectChild(14));
                add(new TestMultObjectChild(15));
            }}).build());
            add(TestMultObject.builder().list(new ArrayList<TestMultObjectChild>() {{
                add(new TestMultObjectChild(17));
                add(new TestMultObjectChild(18));
            }}).build());
        }}).build());
    }};
    List<Method> methodList = new ArrayList<Method>() {{
        // 这个是18节介绍过自己封装的工具类，方法返回method对象
        add(Reflections.fnToMethod(TestMultObjectParent::getList));
        add(Reflections.fnToMethod(TestMultObject::getList));
    }};
    List<Object> extend = new ArrayList<Object>() {{
        add("1");
        add(1);
    }};
    handleMultiList(data, methodList, Handle.class, null, extend);
}
```
输出如下，可以看到虽然每一层结构不一样，但也做到了通用返回
```text
MultiList.TestMultObjectChild(super=com.example.demo.lesson.grace.reflect.MultiList$TestMultObjectChild@3c, id=1)
MultiList.TestMultObjectChild(super=com.example.demo.lesson.grace.reflect.MultiList$TestMultObjectChild@3d, id=2)
MultiList.TestMultObjectChild(super=com.example.demo.lesson.grace.reflect.MultiList$TestMultObjectChild@3f, id=4)
MultiList.TestMultObjectChild(super=com.example.demo.lesson.grace.reflect.MultiList$TestMultObjectChild@40, id=5)
MultiList.TestMultObjectChild(super=com.example.demo.lesson.grace.reflect.MultiList$TestMultObjectChild@42, id=7)
MultiList.TestMultObjectChild(super=com.example.demo.lesson.grace.reflect.MultiList$TestMultObjectChild@43, id=8)
MultiList.TestMultObjectChild(super=com.example.demo.lesson.grace.reflect.MultiList$TestMultObjectChild@46, id=11)
MultiList.TestMultObjectChild(super=com.example.demo.lesson.grace.reflect.MultiList$TestMultObjectChild@47, id=12)
MultiList.TestMultObjectChild(super=com.example.demo.lesson.grace.reflect.MultiList$TestMultObjectChild@49, id=14)
MultiList.TestMultObjectChild(super=com.example.demo.lesson.grace.reflect.MultiList$TestMultObjectChild@4a, id=15)
MultiList.TestMultObjectChild(super=com.example.demo.lesson.grace.reflect.MultiList$TestMultObjectChild@4c, id=17)
MultiList.TestMultObjectChild(super=com.example.demo.lesson.grace.reflect.MultiList$TestMultObjectChild@4d, id=18)
```