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
格式转换相关
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
### 6.2使用
## 7.math包
### 7.1背景
### 7.2使用
## 8.Text包
### 8.1背景
### 8.2使用
## 9.总结
以上只是总结了common下的，还有诸如httpClient等则单独分享