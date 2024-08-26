# 内容协商

在 HTTP 中，内容协商是一种用于在同一 URL 上提供资源的不同表示形式的机制。内容协商机制是指客户端和服务器端就响应的资源内容进行交涉，然后提供给客户端最为适合的资源。

客户端发起请求时会告诉服务端自己期望一个什么样的资源，服务端接收到请求后会据此检查自己支持的响应类型，并返回一个合适的响应。

SpringMVC提供对HTTP内容协商提供了全面的支持:

* `ContentNegotiationStrategy`是内容协商的策略接口，不同的实现对应不同的内容协商策略

* `MediaTypeFileExtensionResolver`：将`MediaType`解析为文件拓展名的接口

* `ContentNegotiationManager` ：内容协商的管理器。

## `MediaTypeFileExtensionResolver`

`MediaTypeFileExtensionResolver`，将`MediaType`解析为文件拓展名。其定义如下：

~~~java
public interface MediaTypeFileExtensionResolver {

	List<String> resolveFileExtensions(MediaType mediaType); //将MediaType解析为对应的文件拓展名列表，例如："application/json" 解析为 "json"

	List<String> getAllFileExtensions(); //获取所有注册的文件拓展名
}

~~~

想要实现这样的功能，可以在内部维护一个`mediaType`和文件拓展名的映射。实际上它的主要实现类`MappingMediaTypeFileExtensionResolver`就是这样做的。

`MediaTypeFileExtensionResolver`的其他实现类并不是为了拓展不同的接口的实现，而是：通过继承或者委托`MappingMediaTypeFileExtensionResolver`获得将`MediaType`解析为文件拓展名的能力。

### `MappingMediaTypeFileExtensionResolver`

`MappingMediaTypeFileExtensionResolver`内部维护下面三个重要字段：

~~~java
private final ConcurrentMap<String, MediaType> mediaTypes = new ConcurrentHashMap<>(64);
//文件拓展名到MediaType的映射
private final ConcurrentMap<MediaType, List<String>> fileExtensions = new ConcurrentHashMap<>(64);
//MediaType到文件拓展名列表的映射
private final List<String> allFileExtensions = new CopyOnWriteArrayList<>();
//注册的所有文件拓展名
~~~

可以看到`MediaType`和文件拓展名是一对多的关系

在创建`MappingMediaTypeFileExtensionResolver`时需要提供一个文件拓展名到`MediaType`的映射：`Map<String, MediaType> `通过这个映射可以初始化上面三个字段。

只需要直接查找`this.mediaTypes`即可实现`resolveFileExtensions()`方法。

另外它还提供下面几个`protected`方法，以供子类使用：

~~~java
protected MediaType lookupMediaType(String extension); //根据文件拓展名查找MediaType
protected void addMapping(String extension, MediaType mediaType); //添加一个扩展名到MediaType的映射，忽略映射已存在的情况
protected List<MediaType> getAllMediaTypes() //获取注册的所有MediaType
~~~

## `ContentNegotiationStrategy`

`ContentNegotiationStrategy`是内容协商的策略接口，它提供一下可用策略实现：

* `FixedContentNegotiationStrategy`:返回固定的`MediaType`列表
* `HeaderContentNegotiationStrategy`:根据请求的`Accept`头解析请求期望的`Content-Type`
* `ParameterContentNegotiationStrategy`:根据指定的请求参数(默认为`format`)解析请求期望的`Content-Type`

### `ParameterContentNegotiationStrategy`

`ParameterContentNegotiationStrategy`继承了`AbstractMappingContentNegotiationStrategy`

`AbstractMappingContentNegotiationStrategy`即继承了`MappingMediaTypeFileExtensionResolver`，也实现了`ContentNegotiationStrategy`

创建`ParameterContentNegotiationStrategy`是需要提供一组`Map<String, MediaType>`文件拓展名到`MediaType`的映射。

这样该策略会取出指定的请求参数值(默认为`format`)，通过该值检索映射关系找到对应的`MediaType`列表

## `ContentNegotiationManager`

`ContentNegotiationManager`是实现内容协商的中心类。它实现了`ContentNegotiationStrategy`和`MediaTypeFileExtensionResolver`。

`ContentNegotiationManager`将内容协商任务委托给注册的`ContentNegotiationStrategy`实现。

`ContentNegotiationManager`将解析`MediaType`的任务委托给注册的`MediaTypeFileExtensionResolver`实现。

它内部维护下面两个字段：

~~~java
private final List<ContentNegotiationStrategy> strategies = new ArrayList<>();
private final Set<MediaTypeFileExtensionResolver> resolvers = new LinkedHashSet<>();
~~~

创建`ContentNegotiationManager`是需要提供`ContentNegotiationStrategy[]`列表以注册到`this.strategies`字段中。

`this.resolvers`字段的注册有下面两个途径：

* 如果构造器提供的`ContentNegotiationStrategy`同时也是`MediaTypeFileExtensionResolver`(这里只有`ContentNegotiationManager`本身和`ParameterContentNegotiationStrategy`)，那么会将其注册到`this.resolvers`字段。

* 直接调用`addFileExtensionResolvers()`方法添加

在`ContentNegotiationManager.resolveMediaTypes()`通过方法进行内容协商时，会按照`this.strategies`从头到尾的顺序进行遍历，只要其中有一个策略能返回有效的结果，直接结束遍历，返回该策略。

所以在向`ContentNegotiationManager`注册内容协商策略时需要格外注意注册策略的顺序。

# `HttpMessageConverter`

`HttpMessageConverter`消息转换器是HTTP请求/响应体和Java类之间进行转换的策略接口，其定义如下：

~~~java
public interface HttpMessageConverter<T> { // T为转换的对象类型

	boolean canRead(Class<?> clazz,MediaType mediaType); 
    //判断指定的mediaType类型的请求体是否能够转换为 clazz指定的类型
	boolean canWrite(Class<?> clazz,MediaType mediaType);
    //判断clazz指定的类型是否能够写入mediaType指定媒体类型的响应体
	List<MediaType> getSupportedMediaTypes();
    //获取当前消息转换器支持的所有媒体类型
	default List<MediaType> getSupportedMediaTypes(Class<?> clazz);
    //获取当前消息转换器支持的指定clazz类型的所有媒体类型
	T read(Class<? extends T> clazz, HttpInputMessage inputMessage);
    //读取请求体并转换为指定类型
	void write(T t, MediaType contentType, HttpOutputMessage outputMessage);
    //将T实例转换并写入指定媒体类型的响应体
}
~~~

其中：

* `HttpInputMessage`代表HTTP输入消息，由请求头和代表请求体的`InputStream`组成
* `HttpOutputMessage`代表HTTP输出消息，由响应头和代表响应体的`OutputStream`组成

一些常用的消息转换器如下：

| 类名                                                         | 支持Java类型     | 支持读取request类型                 | 支持写入的response类型                                       |
| ------------------------------------------------------------ | ---------------- | ----------------------------------- | ------------------------------------------------------------ |
| `ByteArrayHttpMessageConverter`                              | `byte[]`         | `*/*`                               | `application/octet-stream`                                   |
| `StringHttpMessageConverter`                                 | `String`         | `*/*`                               | `text/plain`                                                 |
| `ResourceHttpMessageConverter`                               | `Resources`      | `*/*`                               | 由` MediaTypeFactory`决定                                    |
| `ResourceRegionHttpMessageConverter`                         | `ResourceRegion` | 不支持写入请求                      | `*/*`                                                        |
| `FormHttpMessageConverter`<br />`AllEncompassingFormHttpMessageConverter` | `MultiValueMap`  | `application/x-www-form-urlencoded` | `application/x-www-form-urlencoded`<br />`multipart/form-data`<br />`multipart/mixed` |

* 对于`FormHttpMessageConverter`，如果要写入的响应体是`multipart`类型的，它支持将以下Java类型转换为`mutipart`响应的`part`部分：`byte[]`、`String`、`Resources`。
  * 其子类`AllEncompassingFormHttpMessageConverter`拓展了可写入`part`类型，支持基于 XML 和JSON的`part`类型

此外还有一些消息转换器需要其他类库的支持才能正常使用，下面是常用的：

| 类名                                  | 需要类库                                          |
| ------------------------------------- | ------------------------------------------------- |
| `MappingJackson2HttpMessageConverter` | `com.fasterxml.jackson.databind:jackson-databind` |
| `GsonHttpMessageConverter`            | `com.google.code.gson:gson`                       |
| ........                              |                                                   |

以上两个消息转换器都支持将Java类型读写到`application/json`或者`application/*+json `类型的请求和响应。



# 异步请求



## `AsyncWebRequest`



## `DeferredResult`



## `WebAsyncManager`

`WebAsyncManager`是管理异步请求的核心类。

使用`WebAsyncManager`管理的异步场景涉及两个请求，三个线程，其中两个线程为请求线程，另一个是异步线程。

一个请求线程可以调用`WebAsyncManager` 的`startCallableProcessing()`或`startDeferredResultProcessing()`方法来启动异步请求处理。结果会在第二个异步线程(由用户自己指定或者使用`WebAsyncManager`配置的线程池)中产生。

结果会被保存，并且`WebAsyncManager`会将当前请求委派(调用`AsyncWebRequest.dispatch()`)给当前路径的另一个请求处理线程，在这个新的请求处理线程中，会获取保存的结果并进行响应。

`WebAsyncManager`有三个状态，

* `NOT_STARTED`:没有正在进行的异步处理
* `ASYNC_PROCESSING`：异步处理已启动，但尚未设置结果
* `RESULT_SET`：结果被设置并会发起第二个请求

其状态机如下:



![WebAsyncManager.State](https://gitee.com/wangziming707/note-pic/raw/master/img/WebAsyncManager.State.png)

`WebAsyncManager`有如下重要字段：

~~~java
~~~





