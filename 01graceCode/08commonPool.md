# 【优雅代码】08构建自己的连接池
> 欢迎关注b站账号/公众号【六边形战士夏宁】，一个要把各项指标拉满的男人。该文章已在[github目录](https://github.com/edanlx/SealBook)收录。
屏幕前的**大帅比**和**大漂亮**如果有帮助到你的话请顺手点个赞、加个收藏这对我真的很重要。别下次一定了，都不关注上哪下次一定。
* [gitee目录](https://gitee.com/seal_li/SealBook)
* [知乎目录](https://zhuanlan.zhihu.com/p/338222208)
* [csdn目录](https://blog.csdn.net/seal_li/article/details/111415366)
* [博客园目录](https://www.cnblogs.com/sealLee/articles/14748368.html)
* [可直接运行的完整代码](https://github.com/edanlx/TechingCode/tree/master/demoGrace/src/main/java/com/example/demo/lesson/grace/commonpool) 
* [上一篇](./07springUtils.md)

## 1.背景
线程池的优势自不比多说，连接池和线程池有着众多想通之处，比较常见的连接池有druid、jedis等，但若是某些自研数据库等该如何构建自己的连接池就成问题。笔者使用http这一工具进行构建，可以对比效率差异。核心包为common-pool2
## 2.构建对象工厂
```java
public class HttpCoonFactory extends BasePooledObjectFactory<HttpClient> {

    @Override
    public HttpClient create() throws Exception {
        // 和线程池一样的设计思路创建对象
        return HttpClients.createDefault();
    }

    @Override
    public PooledObject<HttpClient> wrap(HttpClient httpClient) {
        // 和线程池一样的设计思路，包装对象
        return new DefaultPooledObject<HttpClient>(httpClient);
    }
}
```
## 3.定制化参数
```java
public class HttpPoolConfig extends GenericObjectPoolConfig {

    public HttpPoolConfig() {
        // 这里的配置和其它连接池基本一致，一脉相承的设计思路
        setMinIdle(5);
        setTestOnBorrow(true);
        setMaxTotal(50);
    }
}
```
## 4.构建连接池
```java
public class HttpPoolManager extends GenericObjectPool<HttpClient> {
    private static HttpPoolManager httpPoolManager = new HttpPoolManager();

    public static HttpPoolManager getInstance() {
        // 将单例暴露出去
        return httpPoolManager;
    }
    private HttpPoolManager() {
        // 将配置注入到连接池内
        super(new HttpCoonFactory(), new HttpPoolConfig());
    }
}
```
## 5.构建工具类
```java
@Slf4j
public class HttpUtil {

    public static String sendGet(String url) {
        HttpGet httpGet = new HttpGet(url);
        httpGet.setConfig(RequestConfig.custom()
                .setConnectTimeout(3000)
                .setConnectionRequestTimeout(3000)
                .setSocketTimeout(3000)
                .build());
        CloseableHttpClient httpClient = null;
        try {
            httpClient = (CloseableHttpClient) HttpPoolManager.getInstance().borrowObject();
            try {
                @Cleanup CloseableHttpResponse response = httpClient.execute(httpGet);
                HttpEntity entity = response.getEntity();
                if (entity != null) {
                    return EntityUtils.toString(entity);
                }
            } catch (IOException e) {
                log.warn(String.format("%s:%s",
                        Thread.currentThread().getStackTrace()[1].getMethodName(),
                        e.getMessage()), e);
            }
        } catch (Exception e) {
            log.warn(String.format("%s:%s",
                    Thread.currentThread().getStackTrace()[1].getMethodName(),
                    e.getMessage()), e);
        }finally {
            HttpPoolManager.getInstance().returnObject(httpClient);
        }
        return "";
    }

    public static String sendPost(String url, Map<String,String> paramsMap,Map<String, String> headMap){
        List<NameValuePair> formParams = new ArrayList<>();
        if(MapUtils.isNotEmpty(paramsMap)){
            for (Map.Entry<String, String> entry : paramsMap.entrySet()) {
                formParams.add(new BasicNameValuePair(entry.getKey(), entry.getValue()));
            }
        }
        HttpPost httpPost = new HttpPost(url);
        httpPost.setConfig(RequestConfig.custom()
                .setConnectTimeout(3000)
                .setConnectionRequestTimeout(3000)
                .setSocketTimeout(3000)
                .build());
        if(MapUtils.isNotEmpty(headMap)){
            for (Map.Entry<String, String> entry : headMap.entrySet()) {
                httpPost.setHeader(entry.getKey(), entry.getValue());
            }
        }
        try {
            httpPost.setEntity(new UrlEncodedFormEntity(formParams, "utf-8"));
            @Cleanup CloseableHttpClient httpClient = HttpClients.createDefault();
            @Cleanup CloseableHttpResponse response = httpClient.execute(httpPost);
            HttpEntity entity = response.getEntity();
            if (entity != null) {
                return EntityUtils.toString(entity);
            }
        } catch (IOException e) {
            log.warn(String.format("%s:%s",
                    Thread.currentThread().getStackTrace()[1].getMethodName(),
                    e.getMessage()), e);
        }
        return "";
    }

    public static String sendPostJson(String url,String param,Map<String, String> headMap){
        StringEntity entity = new StringEntity(param,"utf-8");
        entity.setContentType(MediaType.APPLICATION_JSON_VALUE);
        entity.setContentEncoding("utf-8");
        HttpPost httpPost = new HttpPost(url);
        httpPost.setConfig(RequestConfig.custom()
                .setConnectTimeout(3000)
                .setConnectionRequestTimeout(3000)
                .setSocketTimeout(3000)
                .build());
        if(MapUtils.isNotEmpty(headMap)){
            for (Map.Entry<String, String> entry : headMap.entrySet()) {
                httpPost.setHeader(entry.getKey(), entry.getValue());
            }
        }
        try {
            httpPost.setEntity(entity);
            @Cleanup CloseableHttpClient httpClient = HttpClients.createDefault();
            @Cleanup CloseableHttpResponse response = httpClient.execute(httpPost);
            HttpEntity entityResukt = response.getEntity();
            if (entityResukt != null) {
                return EntityUtils.toString(entityResukt);
            }
        } catch (IOException e) {
            log.warn(String.format("%s:%s",
                    Thread.currentThread().getStackTrace()[1].getMethodName(),
                    e.getMessage()), e);
        }
        return "";
    }

    /**
     *
     * @author seal 876651109@qq.com
     * @date 2020/6/4 7:23 PM
     */
    public static String postFile(InputStream stream,String fileName,String requestUrl){
        try {
            URL url = new URL(requestUrl);
            HttpURLConnection conn = (HttpURLConnection) url.openConnection();
            conn.setRequestMethod("POST");
            conn.setDoInput(true);
            conn.setDoOutput(true);
            conn.setUseCaches(true);
            conn.setChunkedStreamingMode(1024 * 10000);
            conn.setRequestProperty("Content-Type", MediaType.MULTIPART_FORM_DATA_VALUE);
            @Cleanup OutputStream out = new DataOutputStream(conn.getOutputStream());
            IOUtils.copy(stream,out);
        } catch (ProtocolException e) {
            e.printStackTrace();
        } catch (MalformedURLException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
        return "";
    }
}
```
## 6.使用及对比
```java
public static void main(String[] args) {
        String url = "http://www.baidu.com";
        StopWatch stopWatch = new StopWatch();
        stopWatch.start("pool");
        for (int i = 0; i < 100; i++) {
            sendGet(url);
        }
        stopWatch.stop();
        stopWatch.start("common");
        for (int i = 0; i < 100; i++) {
            HttpGet httpGet = new HttpGet(url);
            httpGet.setConfig(RequestConfig.custom()
                    .setConnectTimeout(3000)
                    .setConnectionRequestTimeout(3000)
                    .setSocketTimeout(3000)
                    .build());
            try (CloseableHttpClient httpclient = HttpClients.createDefault()) {
                httpclient.execute(httpGet).getEntity();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        stopWatch.stop();
        System.out.println(stopWatch.prettyPrint());
    }
```
* 结果如下，用连接池快了一倍，好处大大地
```text
---------------------------------------------
ns         %     Task name
---------------------------------------------
1698114024  031%  pool
3696532228  069%  common
```