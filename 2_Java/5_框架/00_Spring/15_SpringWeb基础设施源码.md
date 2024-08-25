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

# 异步支持

`WebAsyncManager`



# 消息转换

`HttpMessageConverter`

