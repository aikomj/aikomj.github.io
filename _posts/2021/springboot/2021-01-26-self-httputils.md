---
layout: post
title: Http客户端工具类
category: springboot
tags: [springboot]
keywords: springboot
excerpt: hutool已经做了http客户端工具类的封装，引入maven 依赖拿来即用，自己想封装一个，简单轻松的实现get、post、put、delete 以及上传、下载请求，也是可以的。
lock: noneed
---

## 1、介绍

最近在工作中需要在后台调用各种上传、下载、以及第三方服务接口，经过研究决定使用 HttpClient，自己封装了一个 HttpClient 工具类，简单轻松的实现get、post、put、delete 以及上传、下载请求

## 2、实践应用

本文基于 HttpClient4.5.5 版本进行开发，也是现在最新的版本，之所以要提供版本说明，是因为 HttpClient 3 版本和 HttpClient 4 版本 API 差别还是很多大的，你把 HttpClient 3 版本的代码拿到 HttpClient 4 上面运行不起来，会报错的。所以在使用之前，一定要注意 HtppClient 的版本问题。

hutool的httpUtils是基于JDK自带的HttpUrlConnection进行封装的，这是不同点

新建一个Springboot工具类

### 导入HttpClient依赖

```xml
<dependency>
  <groupId>org.apache.httpcomponents</groupId>
  <artifactId>httpclient</artifactId>
  <version>4.5.6</version>
</dependency>
<dependency>
  <groupId>org.apache.httpcomponents</groupId>
  <artifactId>httpcore</artifactId>
  <version>4.4.10</version>
</dependency>
<dependency>
  <groupId>org.apache.httpcomponents</groupId>
  <artifactId>httpmime</artifactId>
  <version>4.5.6</version>
</dependency>
<dependency>
  <groupId>com.alibaba</groupId>
  <artifactId>fastjson</artifactId>
  <version>1.2.68</version>
</dependency>
```

### 编写工具类

本次采用单利模式来初始化客户端，并用线程池来管理，同时支持`http`和`https`协议，项目启动之后，**无需手动关闭`httpClient`客户端**！

```java
public class HttpUtils {

    private static final Logger log = LoggerFactory.getLogger(HttpUtils.class);

    private HttpUtils() {}

    //多线程共享实例
    private static CloseableHttpClient httpClient;

    static {
        SSLContext sslContext = createSSLContext();
        SSLConnectionSocketFactory sslsf = new SSLConnectionSocketFactory(sslContext, NoopHostnameVerifier.INSTANCE);
        // 注册http套接字工厂和https套接字工厂
        Registry<ConnectionSocketFactory> socketFactoryRegistry = RegistryBuilder.<ConnectionSocketFactory> create()
                .register("http", PlainConnectionSocketFactory.INSTANCE)
                .register("https", sslsf)
                .build();
        // 连接池管理器
        PoolingHttpClientConnectionManager connMgr = new PoolingHttpClientConnectionManager(socketFactoryRegistry);
        connMgr.setMaxTotal(300);//连接池最大连接数
        connMgr.setDefaultMaxPerRoute(300);//每个路由最大连接数，设置的过小，无法支持大并发
        connMgr.setValidateAfterInactivity(5 * 1000); //在从连接池获取连接时，连接不活跃多长时间后需要进行一次验证
        // 请求参数配置管理器
        RequestConfig requestConfig = RequestConfig.custom()
                .setConnectTimeout(60000)
                .setSocketTimeout(60000)
                .setConnectionRequestTimeout(60000)
                .build();
        // 获取httpClient客户端
        httpClient = HttpClients.custom()
                .setConnectionManager(connMgr)
                .setDefaultRequestConfig(requestConfig)
                .build();
    }

    /**
     * GET请求
     * @param url
     * @return
     */
    public static String getUrl(String url) {
        return sendHttp(HttpMethod.GET, url, null, null);
    }

    /**
     * GET请求/带头部的信息
     * @param url
     * @param header
     * @return
     */
    public static String getUrl(String url, Map<String, String> header) {
        return sendHttp(HttpMethod.GET, url, header, null);
    }

    /**
     * POST请求/无参数
     * @param url
     * @return
     */
    public static String postJson(String url) {
        return postJson(url, null, null);
    }

    /**
     * POST请求/有参数
     * @param url
     * @param param
     * @return
     */
    public static String postJson(String url, String param) {
        return postJson(url, null, param);
    }

    /**
     * POST请求/无参数带头部
     * @param url
     * @param header
     * @return
     */
    public static String postJson(String url, Map<String, String> header) {
        return postJson(url, header, null);
    }

    /**
     * POST请求/有参数带头部
     * @param url
     * @param header
     * @param params
     * @return
     */
    public static String postJson(String url, Map<String, String> header, String params) {
        return sendHttp(HttpMethod.POST, url, header, params);
    }

    /**
     * 上传文件流
     * @param url
     * @param header
     * @param param
     * @param fileName
     * @param inputStream
     * @return
     */
    public static RequestResult postUploadFileStream(String url, Map<String, String> header, Map<String, String> param, String fileName, InputStream inputStream) {
        byte[] stream = inputStream2byte(inputStream);
        return postUploadFileStream(url, header, param, fileName, stream);
    }

    /**
     * 上传文件
     * @param url 上传地址
     * @param header 请求头部
     * @param param 请求表单
     * @param fileName 文件名称
     * @param stream 文件流
     * @return
     */
    public static RequestResult postUploadFileStream(String url, Map<String, String> header, Map<String, String> param, String fileName, byte[] stream) {
        String infoMessage =  new StringBuilder().append("request postUploadFileStream，url:").append(url).append("，header:").append(header.toString()).append("，param:").append(JSONObject.toJSONString(param)).append("，fileName:").append(fileName).toString();
        log.info(infoMessage);
        RequestResult result = new RequestResult();
        if(StringUtils.isBlank(fileName)){
            log.warn("上传文件名称为空");
            throw new RuntimeException("上传文件名称为空");
        }
        if(Objects.isNull(stream)){
            log.warn("上传文件流为空");
            throw new RuntimeException("上传文件流为空");
        }
        try {
            ContentType contentType = ContentType.MULTIPART_FORM_DATA.withCharset("UTF-8");
            HttpPost httpPost = new HttpPost(url);
            if (Objects.nonNull(header) && !header.isEmpty()) {
                for (Map.Entry<String, String> entry : header.entrySet()) {
                    httpPost.setHeader(entry.getKey(), entry.getValue());
                    if(log.isDebugEnabled()){
                        log.debug(entry.getKey() + ":" + entry.getValue());
                    }
                }
            }
            MultipartEntityBuilder builder = MultipartEntityBuilder.create();
            builder.setCharset(Charset.forName("UTF-8"));//使用UTF-8
            builder.setMode(HttpMultipartMode.BROWSER_COMPATIBLE);//设置浏览器兼容模式
            if (Objects.nonNull(param) && !param.isEmpty()) {
                for (Map.Entry<String, String> entry : param.entrySet()) {
                    if (log.isDebugEnabled()) {
                        log.debug(entry.getKey() + ":" + entry.getValue());
                    }
                    builder.addPart(entry.getKey(), new StringBody(entry.getValue(), contentType));
                }
            }
            builder.addBinaryBody("file",  stream, contentType, fileName);//封装上传文件
            httpPost.setEntity(builder.build());
            //请求执行，获取返回对象
            CloseableHttpResponse response = httpClient.execute(httpPost);
            log.info("postUploadFileStream response status:{}", response.getStatusLine());
            int statusCode = response.getStatusLine().getStatusCode();
            if (statusCode == HttpStatus.SC_OK || statusCode == HttpStatus.SC_NO_CONTENT) {
                result.setSuccess(true);
            }
            HttpEntity httpEntity = response.getEntity();
            if (Objects.nonNull(httpEntity)) {
                String content = EntityUtils.toString(httpEntity, "UTF-8");
                log.info("postUploadFileStream response body:{}", content);
                result.setMsg(content);
            }
            HttpClientUtils.closeQuietly(response);
        } catch (Exception e) {
            log.error(infoMessage + " failure", e);
            result.setMsg("请求异常");
        }
        return result;
    }

    /**
     * 从下载地址获取文件流(如果链接出现双斜杠，请用OKHttp)
     * @param url
     * @return
     */
    public static ByteArrayOutputStream getDownloadFileStream(String url) {
        String infoMessage = new StringBuilder().append("request getDownloadFileStream，url:").append(url).toString();
        log.info(infoMessage);
        ByteArrayOutputStream byteOutStream = null;
        try {
            CloseableHttpResponse response = httpClient.execute(new HttpGet(url));
            log.info("getDownloadFileStream response status:{}", response.getStatusLine());
            int statusCode = response.getStatusLine().getStatusCode();
            if (statusCode == HttpStatus.SC_OK) {
                //请求成功
                HttpEntity entity = response.getEntity();
                if (entity != null && entity.getContent() != null) {
                    //复制输入流
                    byteOutStream = cloneInputStream(entity.getContent());
                }
            }
            HttpClientUtils.closeQuietly(response);
        } catch (Exception e) {
            log.error(infoMessage + " failure", e);
        }
        return byteOutStream;
    }


    /**
     * 发送http请求(通用方法)
     * @param httpMethod 请求方式（GET、POST、PUT、DELETE）
     * @param url        请求路径
     * @param header     请求头
     * @param params     请求body（json数据）
     * @return 响应文本
     */
    public static String sendHttp(HttpMethod httpMethod, String url, Map<String, String> header, String params) {
        String infoMessage = new StringBuilder().append("request sendHttp，url:").append(url).append("，method:").append(httpMethod.name()).append("，header:").append(JSONObject.toJSONString(header)).append("，param:").append(params).toString();
        log.info(infoMessage);
        //返回结果
        String result = null;
        long beginTime = System.currentTimeMillis();
        try {
            ContentType contentType = ContentType.APPLICATION_JSON.withCharset("UTF-8");
            HttpRequestBase request = buildHttpMethod(httpMethod, url);
            if (Objects.nonNull(header) && !header.isEmpty()) {
                for (Map.Entry<String, String> entry : header.entrySet()) {
                    //打印头部信息
                    if(log.isDebugEnabled()){
                        log.debug(entry.getKey() + ":" + entry.getValue());
                    }
                    request.setHeader(entry.getKey(), entry.getValue());
                }
            }
            if (StringUtils.isNotEmpty(params)) {
                if(HttpMethod.POST.equals(httpMethod) || HttpMethod.PUT.equals(httpMethod)){
                    ((HttpEntityEnclosingRequest) request).setEntity(new StringEntity(params, contentType));
                }
            }
            CloseableHttpResponse response = httpClient.execute(request);
            HttpEntity httpEntity = response.getEntity();
            log.info("sendHttp response status:{}", response.getStatusLine());
            if (Objects.nonNull(httpEntity)) {
                result = EntityUtils.toString(httpEntity, "UTF-8");
                log.info("sendHttp response body:{}", result);
            }
            HttpClientUtils.closeQuietly(response);//关闭返回对象
        } catch (Exception e) {
            log.error(infoMessage + " failure", e);
        }
        long endTime = System.currentTimeMillis();
        log.info("request sendHttp response time cost:" + (endTime - beginTime) + " ms");
        return result;
    }

    /**
     * 请求方法（全大些）
     */
    public enum HttpMethod {
        GET, POST, PUT, DELETE
    }

    /**
     * 上传返回结果封装
     */
    public static class RequestResult{
        private boolean isSuccess;//是否成功
        private String msg;//消息

        public boolean isSuccess() {
            return isSuccess;
        }

        public RequestResult setSuccess(boolean success) {
            isSuccess = success;
            return this;
        }

        public String getMsg() {
            return msg;
        }

        public RequestResult setMsg(String msg) {
            this.msg = msg;
            return this;
        }

        public RequestResult() {
            this.isSuccess = false;
        }
    }

    /**
     * 构建请求方法
     * @param method
     * @param url
     * @return
     */
    private static HttpRequestBase buildHttpMethod(HttpMethod method, String url) {
        if (HttpMethod.GET.equals(method)) {
            return new HttpGet(url);
        } else if (HttpMethod.POST.equals(method)) {
            return new HttpPost(url);
        } else if (HttpMethod.PUT.equals(method)) {
            return new HttpPut(url);
        } else if (HttpMethod.DELETE.equals(method)) {
            return new HttpDelete(url);
        } else {
            return null;
        }
    }

    /**
     * 配置证书
     * @return
     */
    private static SSLContext createSSLContext(){
        try {
            //信任所有,支持导入ssl证书
            TrustStrategy acceptingTrustStrategy = (cert, authType) -> true;
            SSLContext sslContext = SSLContexts.custom().loadTrustMaterial(null, acceptingTrustStrategy).build();
            return sslContext;
        } catch (Exception e) {
            log.error("初始化ssl配置失败", e);
            throw new RuntimeException("初始化ssl配置失败");
        }
    }

    /**
     * 复制文件流
     * @param input
     * @return
     */
    private static ByteArrayOutputStream cloneInputStream(InputStream input) {
        try {
            ByteArrayOutputStream byteOutStream = new ByteArrayOutputStream();
            byte[] buffer = new byte[1024];
            int len;
            while ((len = input.read(buffer)) > -1) {
                byteOutStream.write(buffer, 0, len);
            }
            byteOutStream.flush();
            return byteOutStream;
        } catch (IOException e) {
            log.warn("copy InputStream error，{}", ExceptionUtils.getStackTrace(e));
        }
        return null;
    }

    /**
     * 输入流转字节流
     * @param in
     * @return
     */
    private static byte[] inputStream2byte(InputStream in) {
        ByteArrayOutputStream bos = null;
        try {
            bos = new ByteArrayOutputStream();
            byte[] b = new byte[1024];
            int n;
            while ((n = in.read(b)) != -1) {
                bos.write(b, 0, n);
            }
            in.close();
            bos.close();
            byte[] buffer = bos.toByteArray();
            return buffer;
        } catch (IOException e) {
            log.warn("inputStream transfer byte error，{}", ExceptionUtils.getStackTrace(e));
        } finally {
            try {
                if (in != null) {
                    in.close();
                }
            } catch (IOException e) {
                log.error("clone inputStream error", e);
            }
            try {
                if (bos != null) {
                    bos.close();
                }
            } catch (IOException e) {
                log.error("clone outputStream error", e);
            }
        }
        return null;
    }
}
```

除了上传、下载请求之外，默认封装的请求参数格式都是`application/json`，如果不够，可以根据自己的业务场景进行封装处理！

其中`sendHttp`是一个支持`GET`、`POST`、`PUT`、`DELETE`请求的通用方法，上面介绍的`getUrl`、`postJosn`等方法，最终都会调用到这个方法！

### 接口请求示例

工具包封装完成之后，在代码中使用起来也非常简单，直接采用工具类方法就可以直接使用，例如下面以`post`方式请求某个接口！

```java
public static void main(String[] args) throws Exception {
    String url = "https://101.231.204.80:5000/gateway/api/queryTrans.do";
    String result= HttpUtils.postJson(url);
    System.out.println(result);
}
```

### 小结

在编写工具类的时候，需要注意的地方是，尽可能保证`httpClient`客户端全局唯一，也就是采用单利模式，如果我们每次请求都初始化一个客户端，结束之后又将其关闭，在高并发的接口请求场景下，性能效率急剧下降！

`HttpClients`客户端的初始化参数配置非常丰富，本文默认初始化的线程池为`300`，在实际的业务开发中，大家还可以结合自己的业务场景进行调优，具体的配置可以参考官网文档