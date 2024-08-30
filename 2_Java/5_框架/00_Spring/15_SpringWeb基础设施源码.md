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

`AsyncWebRequest`是对异步Web请求的抽象，它拓展自`NativeWebRequest`,提供下面方法：

~~~java
public interface AsyncWebRequest extends NativeWebRequest {

	void setTimeout(@Nullable Long timeout); 
    //设置异步超时时间
	void addTimeoutHandler(Runnable runnable);
	//设置并发请求超时时的回调
	void addErrorHandler(Consumer<Throwable> exceptionHandler);
	//设置并发请求发送错误时的回调
	void addCompletionHandler(Runnable runnable);
	//设置并发请求发送完成时的回调
	void startAsync();
	//开启异步请求模式，以便请求线程退出后，仍能进行响应。实际会调用ServletRequest.startAsync()
	boolean isAsyncStarted();
	//判断当前请求是否处于异步状态。实际调用ServletRequest.isAsyncStarted()
	void dispatch();
	//将当前请求再次分派到Web容器，生成一个新请求，异步的结果会在这个新请求中被处理响应。
	boolean isAsyncComplete();
	//判断异步处是否已经完成
}
~~~

## `DeferredResult`

`DeferredResult`表示异步处理的结果。它维护一组异步处理的回调和一个异步处理的结果。主要字段如下：

~~~java
private final Long timeoutValue; //异步处理的超时时间 
private final Supplier<?> timeoutResult; // 异步处理超时时，需要提供的结果
private Runnable timeoutCallback; //异步处理超时回调
private Consumer<Throwable> errorCallback; //异步处理错误回调
private Runnable completionCallback; //异步处理完成回调
private volatile Object result = RESULT_NONE; //异步处理的结果
private DeferredResultHandler resultHandler; //结果处理器，当结果被设置时，会调用该处理器来处理结果
~~~

其中`timeoutCallback`、`errorCallback`、`completionCallback`可以通过它们对应的`setter`方法设置。设置好这些值后，可以调用`getInterceptor()`方法获取一个`DeferredResultProcessingInterceptor`，这个拦截器会使用这三个回调来完成拦截

通过`setResultHandler()`方法设置结果处理器，设置结果处理器时，如果结果已经被设置了，那么会直接调用该结果处理器。保证即使结果提前设置也会被处理。

通过`setResult()`方法设置异步处理的结果，结果会被赋值到`this.result`字段，紧接着会调用`this.resultHandler.handleResult()`处理该结果。

使用`DeferredResult`时，`WebAsyncManager`会设置好`resultHandler`，这样当用户在异步线程中调用`setResult()`时，`WebAsyncManager`就会负责处理该结果。

## `WebAsyncTask`

`WebAsyncTask`同样可以用于异步请求处理，它持有一个代表异步任务的`Callable`回调，一个可以用于执行异步任务的`AsyncTaskExecutor`,以及异步请求处理过程中的回调：

~~~java
private final Callable<V> callable;
private final Long timeout;
private final AsyncTaskExecutor executor;
private final String executorName;
private BeanFactory beanFactory;
private Callable<V> timeoutCallback;
private Callable<V> errorCallback;
private Runnable completionCallback;
~~~

可以通过对应的`on`方法设置`timeoutCallback`、`errorCallback`、`completionCallback`.字段，可以通过`getInterceptor()`方法获取对应的`CallableProcessingInterceptor`拦截器，该拦截器使用这三个回调进行拦截。



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
private AsyncWebRequest asyncWebRequest; //当前异步请求管理器对应的异步请求
private AsyncTaskExecutor taskExecutor = DEFAULT_TASK_EXECUTOR; //用于提供异步线程的线程池
private volatile Object concurrentResult = RESULT_NONE; //异步处理的结果
private volatile Object[] concurrentResultContext; //并发结果的上下文，一般为一个ModelAndViewContainer
private final AtomicReference<State> state = new AtomicReference<>(State.NOT_STARTED); //当前异步请求的状态
private final Map<Object, CallableProcessingInterceptor> callableInterceptors = new LinkedHashMap<>(); //callable异步处理的拦截器，通过对应的register方法设置
private final Map<Object, DeferredResultProcessingInterceptor> deferredResultInterceptors = new LinkedHashMap<>(); //deferredResult异步处理的拦截器，通过对应的register方法设置
~~~



### `setConcurrentResultAndDispatch()`

`WebAsyncManager`通过`setConcurrentResultAndDispatch()`私有方法来设置并发处理结果，并分派到一个新请求，会在新请求中处理该结果。该方法主要逻辑如下：

* 将当前管理器的状态由`ASYNC_PROCESSING`改为`RESULT_SET`
* 将提供的`result`值设置到`this.concurrentResult`字段
* 此时如果当前异步模式已经结束，方法直接返回，否则下一步
* 调用`AsyncWebRequest.dispatch()`将当前请求派发到Web容器，Web容器会生成一个新请求，在这个新请求中，会对`this.concurrentResult`进行处理

### 开启异步处理

可以通过`WebAsyncManager`的下面三个方法开启异步处理：

~~~java
public void startCallableProcessing(Callable<?> callable, Object... processingContext);
public void startCallableProcessing(final WebAsyncTask<?> webAsyncTask, Object... processingContext);
public void startDeferredResultProcessing(final DeferredResult<?> deferredResult, Object... processingContext);
~~~

其中前两个是重载的，第一个方法会将`callable`参数包装成一个`WebAsyncTask`然后调用第二个。

接下来详细了解一下第二三个方法的主要逻辑：

#### `startCallableProcessing()`

* 将`WebAsyncManager`的状态从`NOT_STARTED`改变为`ASYNC_PROCESSING`
* 设置超时时间：将`WebAsyncTask`的超时时间设置到`this.asyncWebRequest`中
* 获取执行任务的线程池：
  * 尝试从`WebAsyncTask`中获取`AsyncTaskExecutor`
  * 如果上一步获取的`AsyncTaskExecutor`不为`null`，那么将其设置到`this.taskExecutor`
* 生成`CallableProcessingInterceptor`的调用链：
  * 从`webAsyncTask`获取`CallableProcessingInterceptor`
  * 从`this.callableInterceptors`字段获取`CallableProcessingInterceptor`
  * 获取一个`TimeoutCallableProcessingInterceptor()`
  * 将上面获取到的所有拦截器放入一个`CallableInterceptorChain`实例，名为`interceptorChain`

* 为异步请求添加回调：
  * 调用`this.asyncWebRequest.addTimeoutHandler()`添加超时回调。在该回调中：
    * 调用`interceptorChain.triggerAfterTimeout()`触发拦截链的超时拦截，生成一个并发结果
    * 调用`setConcurrentResultAndDispatch()`设置并发处理结果并分派请求
  * 调用`this.asyncWebRequest.addErrorHandler()`添加错误回调，在该回调中：
    * 调用`interceptorChain.triggerAfterError()`触发拦截链的错误拦截，生成一个并发结果
    * 调用`setConcurrentResultAndDispatch()`设置并发处理结果并分派请求
  * 调用`this.asyncWebRequest.addCompletionHandler()`添加完成回调，在该回调中：
    * 调用`interceptorChain.triggerAfterCompletion()`触发拦截链的完成拦截
* 调用`interceptorChain.applyBeforeConcurrentHandling()`触发并发处理前拦截
* 调用`this.taskExecutor.submit()`提交任务并执行，在该任务中：
  * 调用`interceptorChain.applyPreProcess()`触发前处理拦截
  * 调用`WebAsyncTask`中的`Callable`实例的`call()`方法执行异步处理逻辑，生成一个`result`结果
  * 如果执行过程中抛出异常，将异常赋值给`result`
  * 调用`interceptorChain.applyPostProcess()`触发后处理拦截
  * 调用`setConcurrentResultAndDispatch()`设置并发处理结果并分派请求

* y如果上一步提交任务的过程中抛出异常：
  * 调用`interceptorChain.applyPostProcess()`触发后处理拦截
  * 调用`setConcurrentResultAndDispatch()`设置并发处理结果并分派请求

#### `startDeferredResultProcessing()`

`startDeferredResultProcessing()`和`startCallableProcessing()`的处理逻辑基本一致，不同之处在于：

* 对`DeferredResult`的处理不需要线程池，异步线程由用户自己提供并处理

* 拦截器类型不同，使用的是`DeferredResultProcessingInterceptor`

* 不需要提交任务到线程池，只需要添加一个结果处理器到`DeferredResult`中即可，即调用`DeferredResult.setResultHandler()`方法添加一个结果处理器，在该结果处理器中：

  * 调用`interceptorChain.applyPostProcess()`触发拦截器后处理
  * 调用`setConcurrentResultAndDispatch()`设置并发处理结果并分派请求

  这样，当用户在异步线程中调用`DeferredResult.setResult()`是，就会触发该结果处理器以处理结果。

### 与请求绑定

一个`WebAsyncManager`一般对应请求，并且会绑定到请求域中，`WebAsyncUtils`负责处理这样的行为

其重要静态方法为`getAsyncManager()`，其主要逻辑为：

* 从`requset`域中获取名为`WebAsyncUtils.WEB_ASYNC_MANAGER_ATTRIBUTE`的属性,该属性为一个`WebAsyncManager`
* 如果该属性为`null`，那么新建一个`WebAsyncManager`实例，并绑定到`WebAsyncUtils.WEB_ASYNC_MANAGER_ATTRIBUTE`请求域属性上



#  Web数据绑定





Web场景下有数据绑定的需求：从Web请求到JavaBean对象的数据绑定，所以SpringMVC提供了`WebDataBinder`专门用于Web环境的数据绑定，它拓展自`DataBinder`，在绑定时会对字段进行一些请求相关的预处理

`WebDataBinder`重写了父类的`doBind()`方法，在调用`super.doBind()`前新增了三个相关的逻辑，对传入的`MutablePropertyValues`做了一些预处理：

* `checkFieldDefaults()`检查字段默认值，如果传入的`PropertyValue`的字段名以`!`(可配置)开头，那么该`PropertyValue`的字段值将为默认值，可以被其他`PropertyValue`覆盖。

  例如：传入`!city=Shanghai`,则默认值被设置，如果没有其他`PropertyValue`设置`city`字段，那么`city`字段取默认值`Shanghai`。

  用于

* `checkFieldMarkers()`检查字段标记，如果字段名以`_`(可配置)开头，那么该字段将被设置一个空值(如果字段类型为`boolean`，空值为`false`，如果是集合、数组、映射，空值为一个空集合、数组、映射)。

* `adaptEmptyArrayIndices()`处理空数组，如果传入的字段以`[]`结尾，将删除`[]`











