# 【优雅代码】12-hessian、kryo、json序列化大对比
> 欢迎关注b站账号/公众号【六边形战士夏宁】，一个要把各项指标拉满的男人。该文章已在[github目录](https://github.com/edanlx/SealBook/blob/master/catalogue/wechat.md)收录。
屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。
* [上一篇](https://github.com/edanlx/SealBook/blob/master/01graceCode/11stream.md)stream精选/@functional懒加载示例
* [下一篇](https://github.com/edanlx/SealBook/blob/master/01graceCode/13listSpeed.md)linkedList插入真的比arrayList快么

## 1.背景
平常我们在使用rpc调用或者将其持久化到数据库的时候则需要将对象或者文件或者图片等数据将其转为二进制字节数据，那么各自的优劣是什么呢。
## 2.常见的序列化方式
java自带的序列化,平常用的最多的json序列化(也可以叫http数据传输的序列化),dubbo默认的序列化hessian2,其优势在于跨语言(不过跨语言序列化一般还是json和xml范围更广些),目前公认稳定且最快的序列化方式Kryo
### 2.1 Hessian使用
#### 2.1导包
```xml
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>hessian-lite</artifactId>
    <version>3.2.9</version>
</dependency>
```
#### 2.2工具类
```java
/**
 * JavaBean序列化.
 *
 * @param javaBean Java对象.
 * @throws Exception 异常信息.
 */
public static <T> byte[] serialize(T javaBean) {
    try {
        @Cleanup ByteArrayOutputStream baos = new ByteArrayOutputStream();
        @Cleanup Hessian2Output ho = new Hessian2Output(baos);
        ho.writeObject(javaBean);
        ho.flush();
        return baos.toByteArray();
    } catch (IOException e) {
        throw new RuntimeException(e);
    }
}

/**
 * JavaBean反序列化.
 *
 * @param serializeData 序列化数据.
 * @throws Exception 异常信息.
 */
public static <T> T deserialize(byte[] serializeData) {
    try {
        @Cleanup ByteArrayInputStream bais = new ByteArrayInputStream(serializeData);
        @Cleanup Hessian2Input hi = new Hessian2Input(bais);
        return (T) hi.readObject();
    } catch (IOException e) {
        throw new RuntimeException(e);
    }
}
```
### 2.2kryo
#### 2.2.1导包
```xml
<dependency>
    <groupId>com.esotericsoftware</groupId>
    <artifactId>kryo-shaded</artifactId>
    <version>4.0.2</version>
</dependency>
```
#### 2.2.2工具类
```java
// 这个东西线程不安全
private static final ThreadLocal<Kryo> KRYO = ThreadLocal.withInitial(() -> {
    Kryo kryo = new Kryo();
    kryo.setInstantiatorStrategy(new Kryo.DefaultInstantiatorStrategy());
    return kryo;
});

//    newKryoPool() {
//        return new KryoPool.Builder(() -> {
//            final Kryo kryo = new Kryo();
//            kryo.setInstantiatorStrategy(new Kryo.DefaultInstantiatorStrategy(
//                    new StdInstantiatorStrategy()));
//            return kryo;
//        }).softReferences().build();
//    }
public static <T extends Serializable> byte[] serialization(T obj) {
    KRYO.get().register(obj.getClass());
    ByteArrayOutputStream baos = new ByteArrayOutputStream();
    Output output = new Output(baos);
    // KRYO.get().writeClassAndObject(output, obj);
    KRYO.get().writeObject(output, obj);
    output.close();
    return baos.toByteArray();
}

public static <T extends Serializable> T deserialization(byte[] b, Class<T> clazz) {
    KRYO.get().register(clazz);
    ByteArrayInputStream bais = new ByteArrayInputStream(b);
    Input input = new Input(bais);
    // return (T) KRYO.get().readClassAndObject(input);
    return (T) KRYO.get().readObject(input, clazz);
}
```
## 3.比较
### 3.1长度比较
```java
public static void compareLength() {
    // 输出129，88，4，12,可以看出json格式的优秀，还有Kryo针对java格式的优秀
    SerializeTestObject serializeTestObject = new SerializeTestObject();
    serializeTestObject.name = "1";
    byte[] serialize = SerializationUtils.serialize(serializeTestObject);
    SerializeTestObject deserialize = SerializationUtils.<SerializeTestObject>deserialize(serialize);
    System.out.println("deserialize");
    System.out.println(serialize.length);

    byte[] serializeHe = Hessian2Utils.serialize(serializeTestObject);
    SerializeTestObject deserializeHe = Hessian2Utils.<SerializeTestObject>deserialize(serializeHe);
    System.out.println("deserializeHe");
    System.out.println(serializeHe.length);

    byte[] serializeHeKryo = KryoUtils.serialization(serializeTestObject);
    SerializeTestObject deserializeKryo = KryoUtils.deserialization(serializeHeKryo, SerializeTestObject.class);
    System.out.println("deserializeKryo");
    System.out.println(serializeHeKryo.length);
    System.out.println("json");
    System.out.println(JSON.toJSONString(serializeTestObject).getBytes(StandardCharsets.UTF_8).length);
}
```
### 3.2序列化速度比较
```java
public static void compareSerialize() {
    SerializeTestObject serializeTestObject = new SerializeTestObject();
    serializeTestObject.name = "1";
    StopWatch sw = new StopWatch();
    sw.start("serialize");
    for (int i = 0; i < 10000; i++) {
        SerializationUtils.serialize(serializeTestObject);
    }
    sw.stop();
    sw.start("He");
    for (int i = 0; i < 10000; i++) {
        Hessian2Utils.serialize(serializeTestObject);
    }
    sw.stop();
    sw.start("Kryo");
    for (int i = 0; i < 10000; i++) {
        KryoUtils.serialization(serializeTestObject);
    }
    sw.stop();
    sw.start("fastJson");
    for (int i = 0; i < 10000; i++) {
        JSON.toJSONString(serializeTestObject);
    }
    sw.stop();
    sw.start("jackson");
    // 线程安全
    ObjectMapper objectMapper = new ObjectMapper();
    for (int i = 0; i < 10000; i++) {
        objectMapper.writeValueAsString(serializeTestObject);
    }
    sw.stop();
    // 线程安全
    Gson gson = new Gson();
    sw.start("Gson");
    for (int i = 0; i < 10000; i++) {
        gson.toJson(serializeTestObject);
    }
    sw.stop();
    System.out.println(sw.prettyPrint());

}
```
输出信息如下，fastjson比GSON慢，bug还多些，除了Hessian2明显比较慢，其它的速度都差不多
```text
098443444  004%  serialize
1897660738  075%  He
107569186  004%  Kryo
159348852  006%  fastJson
240632483  010%  jackson
028160603  001%  Gson
```
### 3.3反序列化速度比较
```java
private static void compareDeSerialize() {
    SerializeTestObject serializeTestObject = new SerializeTestObject();
    serializeTestObject.name = "1";
    String jsonStr = JSON.toJSONString(serializeTestObject);
    byte[] serialize = SerializationUtils.serialize(serializeTestObject);
    byte[] serializeHe = Hessian2Utils.serialize(serializeTestObject);
    byte[] serializeHeKryo = KryoUtils.serialization(serializeTestObject);
    // 这里统一采用复杂json的处理方式
    TypeReference<SerializeTestObject> fastJsonType = new TypeReference<SerializeTestObject>() {
    };
    com.fasterxml.jackson.core.type.TypeReference<SerializeTestObject> jackJsonType = new com.fasterxml.jackson.core.type.TypeReference<SerializeTestObject>() {
    };
    Type gsonType = new TypeToken<SerializeTestObject>() {
    }.getType();
    StopWatch sw = new StopWatch();
    sw.start("serialize");
    for (int i = 0; i < 10000; i++) {
        SerializeTestObject deserialize = SerializationUtils.<SerializeTestObject>deserialize(serialize);
    }
    sw.stop();
    sw.start("He");
    for (int i = 0; i < 10000; i++) {
        SerializeTestObject deserializeHe = Hessian2Utils.<SerializeTestObject>deserialize(serializeHe);
    }
    sw.stop();
    sw.start("Kryo");
    for (int i = 0; i < 10000; i++) {
        SerializeTestObject deserializeKryo = KryoUtils.deserialization(serializeHeKryo, SerializeTestObject.class);
    }
    sw.stop();
    sw.start("fastJson");
    for (int i = 0; i < 10000; i++) {
        SerializeTestObject fastJsonObject = JSON.parseObject(jsonStr, fastJsonType);
    }
    sw.stop();
    sw.start("jackson");
    // 线程安全
    ObjectMapper objectMapper = new ObjectMapper();
    for (int i = 0; i < 10000; i++) {
        SerializeTestObject jacksonObject = objectMapper.readValue(jsonStr, jackJsonType);
    }
    sw.stop();
    // 线程安全
    Gson gson = new Gson();
    sw.start("Gson");
    for (int i = 0; i < 10000; i++) {
        SerializeTestObject gsonObject = gson.fromJson(jsonStr, gsonType);
    }
    sw.stop();
    System.out.println(sw.prettyPrint());
}
```
输出结果如下,可以看到Kryo、fastJson、Gson摇摇领先
```text
143722925  021%  serialize
128508821  018%  He
046400438  007%  Kryo
052540898  008%  fastJson
292601294  042%  jackson
032188332  005%  Gson
```