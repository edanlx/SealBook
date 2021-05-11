# 【优雅代码】06apache下的优秀工具类
> 欢迎关注b站账号/公众号【六边形战士夏宁】，一个要把各项指标拉满的男人。该文章已在[github目录](https://github.com/edanlx/SealBook)收录。
屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。
* [gitee目录](https://gitee.com/seal_li/SealBook)
* [知乎目录](https://zhuanlan.zhihu.com/p/338222208)
* [csdn目录](https://blog.csdn.net/seal_li/article/details/111415366)
* [博客园目录](https://www.cnblogs.com/sealLee/articles/14748368.html)
* [上一篇](./05symbol.md)

## 1.codec包
### 1.1背景
该包下的工具主要用于编码解码，包括md5，base64等，但加密方式单一，已经不适用于现行大环境
### 1.2使用
```java
public void base64Example(){
        System.out.println("===============base64======================");
        Base64 base64 = new Base64();
        String s = base64.encodeToString("测试22222222222".getBytes());
        System.out.println(s);
        String s1 = new String(base64.decode(s));
        System.out.println(s1);
    }

    public void md5Example(){
        System.out.println("===============MD5======================");
        String result = DigestUtils.md5Hex("测试");
        System.out.println(result);
    }

    public void shaExample(){
        System.out.println("===============sha======================");
        String result = DigestUtils.sha256Hex("测试");
        System.out.println(result);
    }

    public static String encodeHexTest(String str) throws UnsupportedEncodingException {
        return Hex.encodeHexString(str.getBytes(StandardCharsets.UTF_8));
    }

    private static String decodeHexTest(String str) throws DecoderException {
        return new String((byte[])new Hex().decode(str));
    }
```
## 2.collections包(重要)
### 2.1背景
该工具包主要处理集合相关，常用方法为集合判空，以及空集合CollectionUtils、ListUtils、MapUtils、SetUtils、QueueUtils为常用工具类。BagUtils、ClosureUtils、ComparatorUtils、EnumerationUtils、FactoryUtils、IterableUtils、IteratorUtils、MultiMapUtils、MultiSetUtils、PredicateUtils、SplitMapUtils、TransformerUtils、TrieUtils为非常用工具类
### 2.2使用
```java
Collections.emptyList();
Collections.emptyMap();
Collections.emptySet();
Collections.singletonList("1");
Collections.singletonMap("1", "2");
```
## 3.compress包
### 3.1背景
该工具包主要用于压缩和解压，支持多种类型
### 3.2使用
```java
public static void zip() throws Exception
    {
        File zipFile = new File("/Users/seal/Downloads/test.zip");
        ArchiveOutputStream stream = new ZipArchiveOutputStream(zipFile);
        File[] files = new File("/Users/seal/Downloads/UI").listFiles();
        for (File file : files)
        {
            InputStream in = new FileInputStream(file);
            ArchiveEntry entry = new ZipArchiveEntry(file, file.getName());
            // 添加一个条目
            stream.putArchiveEntry(entry);
            IOUtils.copy(in, stream);
            // 结束
            stream.closeArchiveEntry();
            in.close();
        }
        stream.finish();
        stream.close();
    }

    public static void unZip() throws Exception
    {
        InputStream stream = new FileInputStream("/Users/seal/Downloads/jihuoma.zip");
        ArchiveInputStream inputStream = new ZipArchiveInputStream(stream);
        FileUtils.forceMkdir(new File("/Users/seal/Downloads/jihuoma/1/"));
        ArchiveEntry entry = null;
        while ((entry = inputStream.getNextEntry()) != null)
        {
            System.out.println(entry.getName());
            try (FileOutputStream outputStream = new FileOutputStream("/Users/seal/Downloads/jihuoma/1/"+entry.getName())) {
                IOUtils.copy(inputStream, outputStream);
            }
        }
        inputStream.close();
        stream.close();
    }
```
## 4.exec包
### 4.1背景
用于执行命令，比java自带的要多出一些扩展功能，中规中矩
### 4.2使用
```java
private static String execCmdWithResult() {
    try (//接收正常结果流
         ByteArrayOutputStream outputStream = new ByteArrayOutputStream();
         //接收异常结果流
         ByteArrayOutputStream errorStream = new ByteArrayOutputStream();
    ) {
        String command = "ping www.baidu.com";
        CommandLine commandline = CommandLine.parse(command);
        DefaultExecutor exec = new DefaultExecutor();
        exec.setExitValues(null);
        //设置一分钟5秒
        ExecuteWatchdog watchdog = new ExecuteWatchdog(5 * 1000);
        exec.setWatchdog(watchdog);
        PumpStreamHandler streamHandler = new PumpStreamHandler(outputStream, errorStream);
        exec.setStreamHandler(streamHandler);
        exec.execute(commandline);
        //不同操作系统注意编码，否则结果乱码
        String out = outputStream.toString("GBK");
        String error = errorStream.toString("GBK");
        return out + error;
    } catch (Exception e) {
        e.printStackTrace();
        return e.getMessage();
    }
}
```
## 5.io包(重要)
### 5.1背景
非常好用的包，对于文件处理，流处理方便不少，性能优秀。常用工具类FileUtils、IOUtils、FilenameUtils
### 5.2使用
* 格式转换相关
```java
// 获取后缀名
System.out.println(FilenameUtils.getExtension("file.txt"));
//格式化路径
String normalize = FilenameUtils.normalize("D:" + File.separator + "data.txt");
System.out.println(normalize);
//转unix分隔符
System.out.println(FilenameUtils.separatorsToUnix("D:" + File.separator + "data.txt"));
System.out.println(FilenameUtils.separatorsToWindows("D:" + File.separator + "data.txt"));

//目录分隔符
System.out.println(IOUtils.DIR_SEPARATOR);
System.out.println(IOUtils.DIR_SEPARATOR_UNIX);
System.out.println(IOUtils.DIR_SEPARATOR_WINDOWS);
System.out.println(IOUtils.LINE_SEPARATOR);
System.out.println(IOUtils.LINE_SEPARATOR_UNIX);
System.out.println(IOUtils.LINE_SEPARATOR_WINDOWS);
```

## 6.lang包(重要)
### 6.1背景
对平时工具类的补充，优秀的有SerializationUtils，简便的序列化，DateFormatUtils、DateUtils两个非常优秀的日期处理工具类，StringUtils、ObjectUtils这两个可太常用了，处理了各种null的情况，代码写起来更加丝滑。ArrayUtils，补齐了collection的最后一环。
### 6.2使用
* 日期和序列化
```java
System.out.println(SerializationUtils.deserialize(SerializationUtils.serialize(new String("123"))).toString());
System.out.println(NumberUtils.min(1,2));
System.out.println(DateFormatUtils.format(new Date(),"yyyy/MM/dd"));
System.out.println(DateUtils.parseDate("2020/05/29", "",""));
```
* 数组
```java
public static void arrayUtilsExample(){
    int[] a = {1, 5, 6, 8};
    // 01.数组转换成字符串
    String string = ArrayUtils.toString(a);
    // 02.在一个数组中查找某个元素是否存在
    System.out.println("intArray contains '8'?" + ArrayUtils.contains(a, 9));
    System.out.println("intArray index of '8'?" + ArrayUtils.indexOf(a, 9));
    System.out.println("intArray last index of '8'?" + ArrayUtils.contains(a, 9));
    // 03.原始类型转换成包装类
    Integer[] object = ArrayUtils.toObject(a);
    System.out.println(object[2]);
}
```
* 字符串
```java
public static void stringUtilsExample(){
    //实现=======的效果，用于打日志
    StringUtils.repeat("=", 50);
    //实现 %%%%%%%%Customised Header%%%%%%%%效果
    String msg = StringUtils.center(" Customised Header ", 50, "%");

    //将一个array中的String连接起来，用分隔符隔开
    StringUtils.join(msg, ",");
    //相反，把用分隔符隔开的string转为数组
    StringUtils.split(msg, ",");

    //加强代码可读性，减少if判断
    StringUtils.defaultString(msg, msg);
    //缩写一个长string，若不足则不干任何事，否则截断并在末尾添加”…”
    StringUtils.abbreviate(msg,5);
}
```
* 对象
```java
public static void objectUtilsExample(){
    //增强代码可读性，如果obj为null返回defaultObj，这一点在common-lang包中一脉相承
     ObjectUtils.defaultIfNull(null, "");

    //是否相等，等价于obj.equals(obj2)，省略了null判断
     ObjectUtils.equals(null, null);
}
```

## 7.math包
### 7.1背景
眼花缭乱的数学计算，虽然很优秀，但使用场景非常有限
### 7.2使用
```java
public static void main(String[] args) {
    // 如果不涉及统计类或者数学公式计算应该都用不上
    // MathUtils;
    // CombinatoricsUtils
    // ArithmeticUtils
    double[] values = new double[] { 0.33, 1.33,0.27333, 0.3, 0.501,
            0.444, 0.44, 0.34496, 0.33,0.3, 0.292, 0.667 };
    double[] values2 = new double[] { 0.89, 1.51,0.37999, 0.4, 0.701,
            0.484, 0.54, 0.56496, 0.43,0.3, 0.392, 0.567 };

    //计数
    System.out.println("计算样本个数为：" +values.length);
    //mean--算数平均数
    System.out.println("平均数：" + StatUtils.mean(values));
    //sum--和
    System.out.println("所有数据相加结果为：" + StatUtils.sum(values));
    //max--最小值
    System.out.println("最小值：" + StatUtils.min(values));
    //max--最大值
    System.out.println("最大值：" + StatUtils.max(values));
    //范围
    System.out.println("范围是：" + (StatUtils.max(values)-StatUtils.min(values)));
    //标准差
    StandardDeviation standardDeviation =new StandardDeviation();
    System.out.println("一组数据的标准差为：" + standardDeviation.evaluate(values));
    //variance--方差
    System.out.println("一组数据的方差为：" + StatUtils.variance(values));
    //median--中位数
    Median median= new Median();
    System.out.println("中位数：" + median.evaluate(values));
    //mode--众数
    double[] res = StatUtils.mode(values);
    System.out.println("众数：" + res[0]+","+res[1]);
    for(int i = 0;i<res.length;i++){
        System.out.println("第"+(i+1)+"个众数为："+res[i]);
    }
    //geometricMean--几何平均数
    System.out.println("几何平均数为：" +StatUtils.geometricMean(values));
    //meanDifference-- 平均差，平均概率偏差
    System.out.println("平均差为："+StatUtils.meanDifference(values, values2));
    //normalize--标准化
    double[] norm = StatUtils.normalize(values2);
    for(int i = 0;i<res.length;i++){
        System.out.println("第"+(i+1)+"个数据标准化结果为：" + norm[i]);
    }
    //percentile--百分位数
    System.out.println("从小到大排序后位于80%位置的数：" + StatUtils.percentile(values, 70.0));
    //populationVariance--总体方差
    System.out.println("总体方差为：" + StatUtils.populationVariance(values));
    //product--乘积
    System.out.println("所有数据相乘结果为：" + StatUtils.product(values));
    //sumDifference--和差
    System.out.println("两样本数据的和差为：" + StatUtils.sumDifference(values,values2));
    //sumLog--对数求和
    System.out.println("一组数据的对数求和为：" + StatUtils.sumLog(values));
    //sumSq--计算一组数值的平方和
    System.out.println("一组数据的平方和：" + StatUtils.sumSq(values));
    //varianceDifference --方差差异性。
    System.out.println("一组数据的方差差异性为：" + StatUtils.varianceDifference(values,values2,StatUtils.meanDifference(values, values2)));
}
```
## 8.Text包
### 8.1背景
大段落或文章使用的工具，除非特定场景，否则基本都是下位替代，有着很多比它们更优秀的工具
### 8.2使用
* 长文本(单词类)，可我们在中国呀
```java
public static void wordUtilsExample(){
    // 每个单词首字母大写
    System.out.println(WordUtils.capitalize("i am fine"));

    // 每个单词首字母小写
    System.out.println(WordUtils.uncapitalize("I AM FINE"));

    // 取每个单词的首字母
    System.out.println(WordUtils.initials("I AM FINE"));

    // 一行显示X个字符，计算空格
    System.out.println( WordUtils.wrap("i am fine" ,9));

    // 自定义换行符
    System.out.println( WordUtils.wrap("i am fine" ,4,"\n",false));

    // 大小写转换，没什么用
    System.out.println(WordUtils.swapCase("I am Fine"));

    // 是否包含所有单词
    System.out.println(WordUtils.containsAllWords("abc def", "def", "abc"));

    // 没什么用的
    WordUtils.abbreviate("Now is the time for all good men", 0, 40, null);
}
```
* 转义，马马虎虎吧
```java
public static void stringEscapeUtilsExample(){
    System.out.println(StringEscapeUtils.escapeHtml4("<html></html>"));
    System.out.println(StringEscapeUtils.escapeEcmaScript("<script>alert('123')<script>"));
    System.out.println(StringEscapeUtils.escapeXml11("<html></html>"));
    System.out.println(StringEscapeUtils.escapeJava("你好"));
    System.out.println(StringEscapeUtils.escapeJava("<html></html>"));
}
```
* 看似很强但实际上没有在内存这么干的，里面的算法还算不错
```java
private static void jaccardSimilarityExample() {
    //计算jaccard相似系数
    JaccardSimilarity jaccardSimilarity = new JaccardSimilarity();
    double jcdsimilary1 = jaccardSimilarity.apply("hello", "hell");
    System.out.println("jcdsimilary1:"+jcdsimilary1);
    double jcdsimilary2 = jaccardSimilarity.apply("this is an apple", "this is an app");
    System.out.println("jcdsimilary2:"+jcdsimilary2);
    //计算余弦相似度
    CosineSimilarity cosineSimilarity = new CosineSimilarity();
    Map<CharSequence, Integer> leftVector = new HashMap<>();
    Map<CharSequence, Integer> rightVector = new HashMap<>();
    leftVector.put("a", 1);
    leftVector.put("b", 0);
    leftVector.put("c", 1);
    rightVector.put("a", 1);
    rightVector.put("b", 1);
    rightVector.put("c", 0);
    double cosSimilary = cosineSimilarity.cosineSimilarity(leftVector, rightVector);
    System.out.println("cosSimilary:"+cosSimilary);
}
```
* 功能很强，但是java自带的已经基本能覆盖住95&以上的场景了
```java
private static void messageFormatExample() {
    String param1 = String.format("hi,%s, your age is %s", "john", "26");
    System.out.println("-------------------param1=" + param1);
    Object[] object = new Object[]{"john", Longs.tryParse("24")};
    MessageFormat messageFormat = new MessageFormat("{0} now is at the age of {1}");
    String param2 = messageFormat.format(object);
    System.out.println("-------------------param2=" + param2);

    Map<String, String> replaceValue = Maps.newHashMap();
    replaceValue.put("name", "john");
    replaceValue.put("age", "27");
    StrSubstitutor strSubstitutor = new StrSubstitutor(replaceValue);
    String template1 = "${name} is at the age of${age}";
    String param3 = strSubstitutor.replace(template1);
    System.out.println("-------------------param3=" + param3);

    System.out.println(StringSubstitutor.replace("${name} is at the age of${age}",replaceValue));
}
```