# HttpClient 概述

HTTP协议上当今互联网上最重要的协议之一

Java的java.net包提供了通过HTTP访问资源的基本功能，但不够全面和灵活

HttpClient则提供了更加全面和高效的HTTP功能

HttpClient包实现了HTTP标准和客户端

## 示例

### Get请求

~~~java
try (CloseableHttpClient httpClient = HttpClients.createDefault()){
    ClassicHttpRequest httpGet = ClassicRequestBuilder.get("https://www.baidu.com").build();
    httpClient.execute(httpGet,response -> {
        System.out.println(response.getCode()+" "+response.getReasonPhrase());
        HttpEntity entity = response.getEntity();
        System.out.println(EntityUtils.toString(entity,"utf-8"));
        EntityUtils.consume(entity);
        return null;
    });
} catch (Exception e){
    e.printStackTrace();
} 
~~~

* 底层的HTTP连接由response持有.
* 为了确保正确释放系统资源，用户必须从finally子句调用`CloseableHttpResponse.close()`
* 如果响应内容未被完全使用(`EntityUtils.consume(entity)`)，则底层连接不能被安全地重用，并将被连接管理器关闭和丢弃。

### Post请求

~~~java
try (CloseableHttpClient httpClient = HttpClients.createDefault()){
    ClassicHttpRequest httpPost = ClassicRequestBuilder.post("https://www.baidu.com")
        .setEntity(new UrlEncodedFormEntity(Arrays.asList(
            new BasicNameValuePair("username", "admin"),
            new BasicNameValuePair("password", "123321")
        ))).build();
    httpClient.execute(httpPost,response -> {
        System.out.println(response.getCode()+" "+response.getReasonPhrase());
        HttpEntity entity = response.getEntity();
        System.out.println(EntityUtils.toString(entity,"utf-8"));
        EntityUtils.consume(entity);
        return null;
    });
} catch (Exception e){
    e.printStackTrace();
}
~~~

### Fluent API



