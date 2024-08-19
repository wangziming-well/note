# DispatcherServlet

`DispatcherServlet`一方面作为Web容器的一个Servlet来接受请求，另一方面从对应的`WebApplicationContext`中获取MVC组件，将请求处理的任务委托给这些组件。

`DispatcherServlet`作为`Servlet`，其生命周期受Web容器管理：

* 在`Servlet.init()`中，`DispatcherServlet`会初始化`WebApplicationContext`并请求相关的`Bean`组件。
* 在`Servlet.service()`方法中，`DispatcherServlet`处理Web请求。

`DispatcherServlet`继承自`FrameworkServlet`,而`FrameworkServlet`是SpringMVC的基础servlet。在基于JavaBean的整体解决方案中提供与Spring应用程序上下文的集成。

## 初始化

**note:**本节陈述中提到的方法如果没有显示指定，默认为`FrameworkServlet`的方法

`DispatcherServlet`初始化的核心逻辑在`FrameworkServlet.initWebApplicationContext()`方法中。这个方法会初始化`WebApplicationContext`并将其注册到`this.webApplicationContext`字段上。其主要逻辑为：

* 获取root `WebApplicationContext`(一般是由`ContextLoaderListener`在容器初始化时注册到`ServletContext`)

* 如果在构造`DispatcherServlet`时注入了`WebApplicationContext`,就使用它作为`DispatcherServlet`的应用上下文

  如果这个上下文还没调用过`ConfigurableWebApplicationContext.refresh()`(初始化并加载外部化配置)那么：

  * 若`wac`没有父容器，将第一步获取的根上下文注册到`wac`的父容器
  * 调用`configureAndRefreshWebApplicationContext()`配置并初始化容器

* 若构造时没有注入context实例，调用`findWebApplicationContext()`，根据**可配置字段`this.contextAttribute`**指定的属性指尝试从`ServletContext`中获取注册的`WebApplicationContext`

  (该`WebApplicationContext`一般是由`WebApplicationInitializer`在容器初始化时注册到`Servlet`上下文中，并且该`WebApplicationContext`需要用户自己给它设置了父容器和执行任何初始化操作，`DispatcherServlet`不会执行这些操作)

* 如果此时仍然没有一个`WebApplicationContext`实例，那么调用`createWebApplicationContext()`创建一个本地实例

  * 获取**可配置字段`this.contextClass`**指定的本地实例类型(默认为`XmlWebApplicationContext.class`)实例类型必须是`ConfigurableWebApplicationContext`的实现。
  * 根据上一步获取的类型创建一个容器实例。
  * 设置容器的`environment`和`parent`
  * 根据**可配置字段`this.contextConfigLocation`**设置容器的配置文件位置
  * 调用`configureAndRefreshWebApplicationContext()`配置并初始化容器

* 若此时前端控制器还未刷新(一般出现在使用`findWebApplicationContext()`获取到容器上下文的情况下，此时容器的刷新由用户触发),调用`onRefresh()`方法初始化`DispatcherServlet`(`DispatcherServlet`保证在注册到其中之后才初始化的容器上下文，会在初始化刷新容器后紧接着初始化刷新`DispatcherServlet`)
* 若**可配置字段`this.publishContext`**(默认为`true`)为`true`,则将当前`WebApplicationContext`作为属性发布到`ServletContext`中，属性名为`org.springframework.web.servlet.FrameworkServlet.CONTEXT.`+`ServletName`,其中`ServletName`为`DispatcherServlet`在Web容器中的名称。

### 配置并初始化容器上下文

在实例化`WebApplicationContext`  容器后，会调用`FrameworkServlet.configureAndRefreshWebApplicationContext()`对容器上下文进行一些初始化配置并刷新。其主要逻辑如下：

* 配置容器的唯一序列Id
* 配置容器的`ServletContext`、`ServletConfig`、`Namespace`
* 向容器注册一个`FrameworkServlet.ContextRefreshListener`监听器
* 初始化配置容器的`Environment`，向其环境属性源中注册`ServletContext`、`ServletConfig`
* 调用`postProcessWebApplicationContext()`，该方法默认为空，可以继承重写该方法，以对`FrameworkServlet`的容器上下文做一些后处理。

* 调用`applyInitializers()`，这个方法
  * 从`Servlet`中获取`globalInitializerClasses`初始化参数，实例化参数指定的类，将其存储到`contextInitializers`
  * 获取**可配置参数`contextInitializerClasses`**，实例化参数指定的类，将其存储到`contextInitializers`
  * 根据`@Order`、`@Priority`、`Ordered`指定的优先级对`contextInitializers`排序
  * 遍历`contextInitializers`，调用`ApplicationContextInitializer.initialize()`方法以应用初始化器。
* 调用容器的`ConfigurableWebApplicationContext.refresh()`方法初始化刷新容器

**`FrameworkServlet`的初始化刷新**

`FrameworkServlet.ContextRefreshListener`监听一个`ContextRefreshedEvent`事件，这个事件会在容器初始化完成时被发布。`ContextRefreshListener`监听到事件后，会调用`FrameworkServlet.onApplicationEvent()`方法，这个方法会调用`FrameworkServlet.onRefresh()`以初始化刷新`FrameworkServlet`。

所以`ContextRefreshListener`监听器保证了容器初始化后会立即调用`FrameworkServlet`的初始化刷新。

### 初始化`DispatcherServlet`

在之前的分析我们已经知道，在`WebApplicationContext`初始化完成刷新后，会通知`FrameworkServlet`调用`FrameworkServlet.onRefresh()`来初始化刷新`FrameworkServlet`。

`FrameworkServlet.onRefresh()`方法是空的，`DispatcherServlet`重写了`onRefresh()`方法，在其中调用`DispatcherServlet.initStrategies()`方法。这个方法主要从容器中将MVC相关组件bean注册到自身。定义如下：

~~~java
protected void initStrategies(ApplicationContext context) {
    initMultipartResolver(context);
    initLocaleResolver(context);
    initThemeResolver(context);
    initHandlerMappings(context);
    initHandlerAdapters(context);
    initHandlerExceptionResolvers(context);
    initRequestToViewNameTranslator(context);
    initViewResolvers(context);
    initFlashMapManager(context);
}
~~~

* `initMultipartResolver()`:从容器中获取ID为`multipartResolver`的`MultipartResolver` Bean实例，将其注册到`multipartResolver`字段
* `initLocaleResolver()`，注册`localeResolver`字段
  * 从容器中获取ID为`localeResolver`的`LocaleResolver` Bean实例
  * 若容器中没有，读取并实例化`DispatcherServlet.properties`默认配置文件指定的`LocaleResolver`
* `initThemeResolver()`，注册`themeResolver`字段
  * 从容器中获取ID为`themeResolver`的`ThemeResolver` Bean实例
  * 若容器中没有，读取并实例化`DispatcherServlet.properties`默认配置文件指定的`ThemeResolver`
* `initHandlerMappings()`，注册`handlerMappings`字段
  * 若**可配置参数`this.detectAllHandlerMappings`**(默认为`true`)为`true`，获取容器中所有的`HandlerMapping` Bean实例
  * 否则从容器中获取ID为`handlerMapping`的`HandlerMapping` Bean实例
  * 若容器中没有，读取并实例化`DispatcherServlet.properties`默认配置文件指定的`HandlerMapping`

* `initHandlerAdapters()`，注册`handlerAdapters`字段
  * 若**可配置参数`this.detectAllHandlerAdapters`**(默认为`true`)为`true`，获取容器中所有的`HandlerAdapter` Bean实例
  * 否则从容器中获取ID为`handlerAdapter`的`HandlerAdapter` Bean实例
  * 若容器中没有，读取并实例化`DispatcherServlet.properties`默认配置文件指定的`HandlerAdapter`
* `initHandlerExceptionResolvers()`，注册`handlerExceptionResolvers`字段
  * 若**可配置参数`this.detectAllHandlerExceptionResolvers`**(默认为`true`)为`true`，获取容器中所有的`HandlerExceptionResolver` Bean实例
  * 否则从容器中获取ID为`handlerExceptionResolver`的`HandlerExceptionResolver` Bean实例
  * 若容器中没有，读取并实例化`DispatcherServlet.properties`默认配置文件指定的`HandlerExceptionResolver`

* `initRequestToViewNameTranslator()`，注册`viewNameTranslator`字段
  * 从容器中获取ID为`viewNameTranslator`的`RequestToViewNameTranslator` Bean实例
  * 若容器中没有，读取并实例化`DispatcherServlet.properties`默认配置文件指定的`RequestToViewNameTranslator`
* `initViewResolvers()`，注册`viewResolvers`字段
  * 若**可配置参数`this.detectAllViewResolvers`**(默认为`true`)为`true`，获取容器中所有的`ViewResolver` Bean实例
  * 否则从容器中获取ID为`viewResolver`的`ViewResolver` Bean实例
  * 若容器中没有，读取并实例化`DispatcherServlet.properties`默认配置文件指定的`ViewResolver`
* `initFlashMapManager()`，注册`flashMapManager`字段
  * 从容器中获取ID为`flashMapManager`的`FlashMapManager` Bean实例
  * 若容器中没有，读取并实例化`DispatcherServlet.properties`默认配置文件指定的`FlashMapManager`

可以看到，除了`MultipartResolver`组件，其他所有组件都会从容器中获取，如果获取不到，就会加载实例化`DispatcherServlet.properties`配置的组件，这个文件内容如下：

~~~properties
org.springframework.web.servlet.LocaleResolver=org.springframework.web.servlet.i18n.AcceptHeaderLocaleResolver
org.springframework.web.servlet.ThemeResolver=org.springframework.web.servlet.theme.FixedThemeResolver
org.springframework.web.servlet.HandlerMapping=org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping,\
	org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping,\
	org.springframework.web.servlet.function.support.RouterFunctionMapping
org.springframework.web.servlet.HandlerAdapter=org.springframework.web.servlet.mvc.HttpRequestHandlerAdapter,\
	org.springframework.web.servlet.mvc.SimpleControllerHandlerAdapter,\
	org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter,\
	org.springframework.web.servlet.function.support.HandlerFunctionAdapter
org.springframework.web.servlet.HandlerExceptionResolver=org.springframework.web.servlet.mvc.method.annotation.ExceptionHandlerExceptionResolver,\
	org.springframework.web.servlet.mvc.annotation.ResponseStatusExceptionResolver,\
	org.springframework.web.servlet.mvc.support.DefaultHandlerExceptionResolver
org.springframework.web.servlet.RequestToViewNameTranslator=org.springframework.web.servlet.view.DefaultRequestToViewNameTranslator
org.springframework.web.servlet.ViewResolver=org.springframework.web.servlet.view.InternalResourceViewResolver
org.springframework.web.servlet.FlashMapManager=org.springframework.web.servlet.support.SessionFlashMapManager
~~~

## 请求处理

`DispatcherServlet`接受到请求后会调用`FrameworkServlet.processRequest()`方法来处理请求,该方法主要逻辑为：

* 向当前请求线程(对异步请求的异步线程也适用)绑定一些对象：
  * 实例化`LocaleContext`和`RequestAttributes`对象
    * `RequestAttributes`:用于访问与请求关联的属性对象的抽象。支持访问请求范围的属性以及会话范围的属性，并具有“全局会话”的可选概念。可以为任何类型的请求会话机制实现，特别是servlet请求。
  * 将第一步中实例化的对象通过`ThreadLocal`绑定到当前请求线程。这样，在当前请求里可以通过`LocaleContextHolder`和`RequestContextHolder`获取他们。
  * 向`WebAsyncManager`注册一个回调，保证异步请求中另起的线程也会绑定这些对象。
* 调用`FrameworkServlet.doService()`处理请求
* 请求处理完成后重置第一步绑定到线程上的对象(因为请求完成后线程不是销毁，而是重新放入线程池中复用)
* 如果**可配置参数`this.publishEvents`**(默认为`true`)为`true`，向绑定的容器上下文发布一个`ServletRequestHandledEvent`事件。

由此可以看出请求处理的真正逻辑在`FrameworkServlet.doService()`中，这个方法默认为空，`DispatcherServlet`重写了该方法，提供真正的请求处理逻辑。

### `DispatcherServlet.doService()`

`DispatcherServlet.doService()`的主要逻辑如下：

* 创建一个`attributesSnapshot`属性快照，用于include包含请求的属性重置(对于include请求，在servlet处理前就有被包含的请求的request属性，现在保存快照会在请求处理结束后恢复快照)

  * 如果**可配置参数`this.cleanupAfterInclude`**(默认为`true`)为`true` ,将所有request域种属性保存到`attributesSnapshot`
  * 否则只保存那些由`DispatcherServlet`添加的属性(属性名前缀为`org.springframework.web.servlet`)

* 向request域中添加属性,属性名和属性值如下：

  其中属性名为`DispatcherServlet.class.getName() `+属性名后缀

  | 属性名后缀           | 属性值                                                       |
  | -------------------- | ------------------------------------------------------------ |
  | `.CONTEXT`           | `this.webApplicationContext`                                 |
  | `.LOCALE_RESOLVER`   | `this.localeResolver`                                        |
  | `.THEME_RESOLVER`    | `this.themeResolver`                                         |
  | `.THEME_SOURCE`      | 一个`ThemeSource`通常就是容器本身                            |
  | `.INPUT_FLASH_MAP`   | 在转发中，由转发源请求生成的`FlashMap`,里面可能包含转发源设置的属性。 |
  | `.OUTPUT_FLASH_MAP`  | 在转发中，用于发送给转发目标的`FlashMap`,这里是一个空的`FlashMap` |
  | `.FLASH_MAP_MANAGER` | `this.flashMapManager`                                       |

* 如果注册的`HandlerMapping`中有使用路径模式的

  * 在请求域中获取`PATH_ATTRIBUTE = ServletRequestPathUtils.class.getName() + ".PATH"`属性，设置给`previousRequestPath`
  * 向request域中添加属性`ServletRequestPathUtils.class.getName() + ".PATH"`，值为当前请求对应的`RequestPath`

* 调用`DispatcherServlet.doDispatch()`处理请求

* 若请求没有进行异步处理，使用第一步生成的`attributesSnapshot`属性快照将request域重置为快照属性。

* 如果注册的`HandlerMapping`中有使用路径模式的，向request域中添加属性`ServletRequestPathUtils.class.getName() + ".PATH"`，值为`previousRequestPath`

由此可以看出`DispatcherServlet.doService()`主要是针对request域进行一些预处理和后处理，真正处理请求是在`DispatcherServlet.doDispatch()`中

### `HandlerExecutionChain`

在继续分析`doDispatch()`前需要先了解一下`HandlerExecutionChain`类。这个类是处理器执行链的抽象。包含了Handler处理器和对应的处理器拦截器。可以让`DispatcherServlet`方便的执行拦截器的前处理、后处理。`HandlerMapping.getHandler()`返回的就是一个`HandlerExecutionChain`,其核心方法如下：

* `applyPreHandle()`:执行包含的拦截器链的前处理回调：
  * 按优先级从高到底遍历`HandlerExecutionChain.interceptorList`的拦截器链：调用`HandlerInterceptor.preHandle()`进行前处理。
  * 如果上一步某个拦截器返回`false`则表示这个拦截器已经完成了请求的处理(例如抛出了一个异常，或者在拦截器中完成了请求的响应)，那么中断遍历，不再继续调用后续的`HandlerInterceptor.preHandle()`。然后进行请求的后处理：
    * 调用`triggerAfterCompletion()`执行包含的拦截器链的完成回调
    * 然后直接返回`false`

* `applyPostHandle()`:执行包含的拦截器链的后处理回调：按照优先级从低到高遍历`HandlerExecutionChain.interceptorList`的拦截器链,调用`HandlerInterceptor.postHandle()`

* 调用`triggerAfterCompletion()`执行包含的拦截器链的完成回调：按照优先级从低到高遍历`HandlerExecutionChain.interceptorList`的拦截器链，调用`HandlerInterceptor.afterCompletion()`
* `applyAfterConcurrentHandlingStarted()`执行包含的拦截器链的异步完成回调：按照优先级从低到高遍历`HandlerExecutionChain.interceptorList`的拦截器链，调用`AsyncHandlerInterceptor.afterConcurrentHandlingStarted()`完成回调

### `DispatcherServlet.doDispatch()`

**note:**本节陈述中提到的方法如果没有显示指定，默认为`DispatcherServlet`的方法

`DispatcherServlet.doDispatch()`方法主要逻辑如下：

* 调用`checkMultipart()`检查是否需要包装为`multipart`请求：若`this.multipartResolver`不为空且请求是`multipart`的(通过`MultipartResolver.isMultipart()`判断其请求的ContentType前缀是否为`multipart/`)，调用`MultipartResolver.resolveMultipart()`将request包装为`MultipartHttpServletRequest`类型，以供处理器访问文件等。
* 遍历`this.handlerMappings`，调用`HandlerMapping.getHandler()`方法，获取`HandlerExecutionChain`实例。该类型包含了`Handler`和`HandlerInterceptor`的调用链。
    * 如果获取的`HandlerExecutionChain`实例为空，抛出一个`NoHandlerFoundException`异常，并响应一个404错误，最后退出方法
    * 否则继续
* 调用`getHandlerAdapter()`方法，获取一个支持获得的 `Handler`的`HandlerAdapter`
    * 遍历`this.handlerAdapters`，调用`HandlerAdapter.supports()`方法，如果该方法返回`true`，则返回当前的`HandlerAdapter`
* 处理HTTP缓存机制：如果当前HTTP请求类型为`GET`或者`HEAD`,获取当前资源的最后修改时间戳`lastModified`
    * 如果是`GET`请求，并且缓存未失效，则响应一个304状态码并返回
    * 否则，继续
* 调用`HandlerExecutionChain.applyPreHandle()`进行前处理。返回`false`，表示前处理中已经完成了请求的处理，方法直接返回
* 调用`HandlerAdapter.handle()`处理请求，返回一个`ModelAndView`
* 判断异步请求：如果当前请求处于异步模式，并且异步线程正在进行，那么当前方法直接返回。执行`finally`中代码
* 如果当前`ModeAndView`还没有`View`  根据请求获取默认`viewName`，设置到`ModelAndView`中；默认的`viewName`是从`this.viewNameTranslator.getViewName()`方法返回
* 调用`HandlerExecutionChain.applyPostHandle()`进行后处理
* **调用`processDispatchResult()`处理响应的结果**包括异常处理和`View`渲染
* 调用`HandlerExecutionChain.triggerAfterCompletion()`进行拦截器完成回调
* 以下为`finally`代码块内容，即使前边有`return`也会执行
    * 如果请求请求处于异步模式，调用`HandlerExecutionChain.applyAfterConcurrentHandlingStarted()`进行异步拦截器回调
    * 如果第一步包装了`multipart`请求，调用`MultipartResolver.cleanupMultipart()`清除释放资源：最终删除缓存文件


### `DispatcherServlet.processDispatchResult()`

`processDispatchResult()`方法处理异常并渲染视图返回结果，方法入参的`exception`为`doDispatch()`中抛出的异常。其主要逻辑为：

* 处理异常：如果有抛出异常(`exception != null`):
  * 若异常类型为`ModelAndViewDefiningException`,调用`ModelAndViewDefiningException.getModelAndView()`覆盖`ModelAndView`
  * 否则，调用`processHandlerException()`方法覆盖`ModelAndView`，该方法：
    * 遍历`this.handlerExceptionResolvers`调用`HandlerExceptionResolver.resolveException()`生成错误响应视图`ModelAndView`
    * 如果生成的错误`ModelAndView`中的`view`为空，根据请求获取默认`viewName`，设置到`ModelAndView`中；默认的`viewName`是从`this.viewNameTranslator.getViewName()`方法返回
* 渲染模型：如果`ModelAndView`不为空,**调用`render()`渲染模型**
  * 调用`this.localeResolver.resolveLocale()`获取`Locale`实例
  * 如果`ModeAndView`中`viewName`不为空
    * 遍历`this.viewResolvers`,调用`ViewResolver.resolveViewName()`直到获取`View`实例
    * 如果遍历完成后仍然没有获取到`View`实例，抛出一个`ServletException`异常
  * 如果`ModeAndView`中`viewName`为空，调用`ModelAndView.getView()`获取`View`实例，如果获取不到，抛出一个`ServletException`异常
    * 调用`View.render()`渲染视图

* 如果请求是异步模式(可能是在拦截器的后处理程序中被设置)，方法直接返回
* 调用`HandlerExecutionChain.triggerAfterCompletion()`进行拦截器完成回调

# 路径匹配

在继续分析前，需要先了解下面Spring 提供的进行路径匹配的Util工具类：

* `org.springframework.util.PathMatcher`
* `org.springframework.web.util.pattern.PathPattern`

这两个工具类都是用于路径匹配，主要功能基本一致，`PathPattern`是Spring5新增的路径匹配器。旨在取代`PathMatcher`，性能也更强。

## `PathMatcher`

`PathMatcher`只有一个实现类`AntPathMatcher`，主要用来判断给定的模式`pattern`和路径`path`是否匹配。

路径由路径分隔符分割成若干个路径段。

其中`pattern`模式的匹配规则如下：

* `?`匹配一个任意字符（不包括路径分隔符）
* `*`匹配零个或多个任意字符（不包括路径分隔符）
* `**`匹配路径中零个或者多个目录
* `{var:regex}`:匹配一个满足`regex`代表的正则表达式路径段，并将匹配到的值赋值给变量名`var`，可以省略正则表达式，此时匹配任意路径段

注意：模式和路径要么都是相对的，要么都是绝对的，否则无法进行匹配(匹配结果为false)

### `match()`

`match()`方法对路径进行完全匹配：

~~~java
public boolean match(String pattern, String path);
~~~

使用示例：

~~~java
AntPathMatcher matcher = new AntPathMatcher();

matcher.match("/test/a?c", "/test/abc"); //true
matcher.match("/test/a?c", "/test/abbc"); //false

matcher.match("/test/*.html", "/test/index.html"); //true
matcher.match("/test/*.html", "/test/index");

matcher.match("/test/*", "/test/index"); //true
matcher.match("/test/*", "/test/v1/index"); //false

matcher.match("/test/**/index.jsp", "/test/v1/v2/index.jsp");//true
matcher.match("/test/**/index.jsp", "/test/index.jsp");//true

matcher.match("test/**", "/test/v1/v2/index.jsp"); //false
matcher.match("test/**", "test/index.jsp"); //true
~~~

### `matchStart()`

进行不完全匹配，路径只需要完全匹配模式中前面部分的若干个路径段，例如：

~~~java
matcher.matchStart("/test/path/**", "/test"); //true
matcher.matchStart("/test/path/**", "/test/abc"); //false
~~~

### `extractPathWithinPattern()`

找出通配符`*`、`**`、`?`所在的路径段，并返回该路径段和其所有子路径，例如：

~~~java
matcher.extractPathWithinPattern("/test/**/*.html", "/test/p/index.jsp"); //  p/index.jsp
matcher.extractPathWithinPattern("/test/**/*.html", "/test/p/index.html"); // p/index.html
matcher.extractPathWithinPattern("/test/a?c/*", "/test/abc/v"); // abc/v
~~~

### `extractUriTemplateVariables()`

返回模板变量，即`{var:regex}`匹配的变量，例如：

~~~java
matcher.extractUriTemplateVariables("/test/{v1:\\w+}/{v2}.html", "/test/p/index.html");//  {v1=p, v2=index}
~~~

### `combine()`

`combine(String pattern1, String pattern2)`方法合并两个模式，主要逻辑如下：

* 如果两个模式其中一个为空(空字符串或者`null`)，直接返回另一个模式

* 如果`pattern1`和`pattern2`不相同，且`pattern1`不包含路径模板变量，且将`pattern2`作为路径时，`pattern1`完全匹配`pattern2`,直接返回`pattern2`

* 如果`pattern1`以`/*`结尾,去掉`pattern1`的后缀的`*`，然后拼接两个模式并返回

* 如果`pattern1`以`/**`结尾,直接拼接两个模式并返回

* 如果`pattern1`包含路径模板变量，或者`pattern1`没有匹配任意文件名(`pattern1`中不包含`*.`),或者路径分隔符为`.`,那么直接拼接两个模式并返回

* 走到这一步，说明`pattern1`不以`/*`、`/**`为后缀，不包含路径模板变量，会匹配任意文件名(包含`*.`),并且路径分隔符不为`.`

* 如果`pattern1`和`pattern2`都不匹配所有文件拓展名(`pattern`的中包含`.`，且后缀不为`.*`)，抛出一个`IllegalArgumentException`

  例如，对于`/test/*.jsp`和`/test/key.jsp`将抛出异常

* 最后，`pattern2`去掉文件后缀(去掉`.xxx`的后缀)，获得`pattern2` 的文件名`file2`(包含路径)

  * 如果`pattern1`匹配所有文件拓展名(以`*.*`作为后缀)，直接返回`pattern2`
  * 否则返回`file2`拼接上`pattern1`的文件拓展名


注意：两个模式的拼接会自动判断是否添加路径分隔符。

例如：

~~~java
matcher.combine("/test/**", "/test/*");  // /test/*
matcher.combine("/test/**", "test/*");  // /test/**/test/*
matcher.combine("/path1/*.*","/test/var.h");  // /test/var.h
matcher.combine("/path1/*.jsp","/test/var");  // /test/var.jsp
~~~

### `getPatternComparator()`

这个方法获取一个模式比较器，基于一个路径比较模式的通用性。通用性强的模式大于通用性弱的模式。

注意：模式的通用性是基于路径的，可能有两个模式A，B，对于路径P1来说A更通用，但对于路径P2来说B更通用。

在多个模式同时匹配上同一个路径的场景下，可以用模式比较器选出一个最佳/最优先(最不通用)的模式。比如一个URL请求能够匹配多个`Controller`时,可以选出一个最优先的`Controller`。

对于`pattern1`和`pattern2`和路径`path`；比较器按照下面顺序判断通用性，下面情况中`pattern1`更通用：

* `pattern1`为`null`或者等于`/**`  （都为`null`或者`/**`则通用性相同）
* `pattern2`与`path`相等（都与`path`相同则通用性相同）
* `pattern1`以`/**`为后缀且`pattern2`中没有`**`通配符
* `pattern1`和`pattern2`都以`/**`结尾，`pattern1`长度更短(模板变量长度算1)（长度相同相同则通用性相同）
* `pattern1`中有更多的通配符(`**`算两个通配符，`{}`和`*`算一个通配符)
* `pattern1`长度更短(模板变量长度算1)
* `pattern1`中单`*`通配符更多
* `pattern1`中模板变量更多
* 以上判断都通过则通用性相同

例如：

~~~java
AntPathMatcher matcher = new AntPathMatcher();
Comparator<String> comparator = matcher.getPatternComparator("/test/path/inde");
comparator.compare("/**", "/test/**"); //1
comparator.compare("/test/path/*.*", "/test/path/index.jsp"); // 1
comparator.compare("/test/path/**", "/test/**/inde"); //0
~~~

## 新路径匹配器

不同与`PathMatcher`把所有功能集中在一个类中,新的路径匹配围绕`PathPattern`拥有要给体系，在设计上更模块化，更加面向对象。涉及的几个类：

* `PathElement`：路径节点的抽象。一个路径可以拆分成多个路径节点对象。
* `PathContainer`：路径的结构化抽象。
* `PathPattern`：模式的抽象。路径模式匹配器的最核心API

* `PathPatternParser`：将一个String类型的模式解析为`PathPattern`实例，这是创建`PathPattern`实例的唯一方式

`PathPattern`提供的路径匹配功能基本和`AntPathMatcher`一样，不同之处在于：

* `**`通配符只能在模式最后
* 除了通配符：`?`、`*`、`**`、`{var:regex}`外，新提供一个`{*var} `用于捕获剩余所有路径段到变量`var`上

### `PathElement`

`PathElement`代表一个路径节点。例如路径段、路径分隔符。

`PathElement`中的`next`和`prev`指示当前节点的前后节点。所有的`PathElement`之间形成链状结构，构成一个完整的路径模板。

它不是一个公开的类，我们不需要仔细了解其实现细节，只需要了解一下它的不同实现即可。

它有8个实现类，代表不同类型的路径节点：

* `SeparatorPathElement`:路径分隔符
* `WildcardPathElement`：仅有`*`通配符的路径段
* `SingleCharWildcardedPathElement`：带有`?`通配符的路径段
* `WildcardTheRestPathElement`：表示路径剩余部分的路径段，例如在`/foo/**`中的`/**`
* `CaptureVariablePathElement`表示捕获路径变量的路径段，例如`/foo/{id}`的`{id}`
* `CaptureTheRestPathElement`表示捕获剩余所有路径的路径段，例如 `/foo/{*foobar}`中的`/{*foobar}`
* `LiteralPathElement`表示字面路径段，例如`/foo/bar`中的`foo`和`bar`
* `RegexPathElement`表示正则表达式路径段，例如`/foo/*_*/*_{foobar}`中的`*_*`和`*_{foobar}`

### `PathContainer`

是URI路径的结构化表示，它有如下内部接口/类：

* `PathContainer.Element`:表示路径元素，它有如下子接口：
  * `PathContainer.Separator`:表示路径分隔符
  * `PathContainer.PathSegment`:表示路径段
* `Options`路径解析的可选项，可以设置路径分隔符和是否解码解析路径段

主要方法如下：

* `value()`返回路径对应的字符串
* `elements()`返回路径元素的有序列表

* `subPath()`截取返回子路径
* `static parsePath()`根据字符串解析返回一个`PathContainer`

### `PathPattern`

`PathPattern`代表一个模式，用于和代表路径的`PathContainer`进行匹配

`PathPattern`只有由`PathPatternParser`创建，例如：

~~~java
String patternStr = ....;
PathPattern parse = PathPatternParser.defaultInstance.parse(patternStr);
~~~

其主要方法如下：

#### `matches()`

和指定的路径进行完全匹配。例如：

~~~java
PathPattern pattern = PathPatternParser.defaultInstance.parse("/test/*/*.jsp");
PathContainer path = PathContainer.parsePath("/test/v1/index.jsp");
boolean matches = pattern.matches(path); // true
~~~

#### `matchAndExtract()`

匹配并捕获路径变量和路径参数。

如果不匹配，返回null，否则它返回一个`PathPattern.PathMatchInfo`, 模式匹配路径时捕获的路径变量和路径参数，它持有如下参数：

* `Map<String, String>  uriVariables`:由模式中`{var}`和`{*var}`捕获的路径变量，
* `Map<String, MultiValueMap<String, String>> matrixVariables`：被捕获的路径参数(如果没有被模板捕获，那么即使路径有参数，该字段也为空)，例如，针对路径`/test/v1;a=1;b=2,3/index.html`:
    * 使用模式`/test/{version}/index.html`进行匹配，该参数为：`{version={a=[1], b=[2, 3]}}`
    * 使用模式`/test/v1/{filename}`进行匹配，该参数为`{}`

使用示例：

~~~java
PathPattern pattern = PathPatternParser.defaultInstance.parse("/test/{version}/{filename}");
PathContainer path = PathContainer.parsePath("/test/v1;a=1;b=2,3/index.html");
PathPattern.PathMatchInfo pathMatchInfo = pattern.matchAndExtract(path);
System.out.println(pathMatchInfo); //PathMatchInfo[uriVariables={filename=index.html, version=v1}, matrixVariables={version={a=[1], b=[2, 3]}}]

~~~

#### `matchStartOfPath()`

如果不匹配，方法返回`null`,否则返回`PathPattern.PathRemainingMatchInfo`它持有如下参数：

* ` PathContainer pathMatched`:与模式匹配的路径前缀
* `PathContainer pathRemaining`：不匹配的剩余路径
* `PathMatchInfo pathMatchInfo`：匹配的捕获参数

示例：

~~~java
PathPattern pattern = PathPatternParser.defaultInstance.parse("/test/{version}");
PathContainer path = PathContainer.parsePath("/test/v1;a=1;b=2,3/index.html");
PathPattern.PathRemainingMatchInfo result = pattern.matchStartOfPath(path);
System.out.println(result.getPathMatched()); // "/test/v1;a=1;b=2,3"
System.out.println(result.getPathRemaining()); // "/index.html"
System.out.println(result.getUriVariables()); // {version=v1}
System.out.println(result.getMatrixVariables()); // {version={a=[1], b=[2, 3]}}
~~~

#### `extractPathWithinPattern()`

找出通配符`*`、`**`、`?`所在的路径段，并返回该路径段和其所有子路径，例如：

~~~java
PathPattern pattern = PathPatternParser.defaultInstance.parse("/test/*/*.html");
PathContainer path = PathContainer.parsePath("/test/v1/index.html");
PathContainer result = pattern.extractPathWithinPattern(path); // v1/index.html
~~~

#### `combine()`

合并两个模式，合并的逻辑和`AntPathMatcher`一样，例如：

~~~java
PathPattern pattern = PathPatternParser.defaultInstance.parse("/test/*.html");
PathPattern toCombine = PathPatternParser.defaultInstance.parse("/path.*");
PathPattern combine = pattern.combine(toCombine); // /path.html
~~~

#### `compareTo()`

比较模式的通用性，通用性强的`PathPattern`更大，注意，和`AntPathMatcher`不同，`PathPattern`的通用性比较不需要基于某个路径。其通用性按照判断顺序考虑以下因素：

* `null`实例最通用
* 模式包含`/**`和`/{*ver}`这样的全匹配路径段(`CaptureTheRestPathElement`和`WildcardTheRestPathElement`)更通用
    * 如果两个路径都是全匹配路径段，模式长度(`normalizedLength`字段)越短越通用
* 模式中`*`通配符更多的更通用
    * 如果模式中`*`通配符一样多，`?`通配符更多的更通用
* 模式长度(`normalizedLength`字段)越短越通用
* 以上都相等时，直接比较模式代表的字符串字典顺序大小`String.compareTo()`

注意：模式长度(`normalizedLength`字段)计算方式如下：

* 匹配剩余路径段的模式段算长度1，例如`/**`和`/{*var}`
* 模板变量算长度算1,五变量名称无关，例如`{var}`和`{fooo}`长度都为1
* 其他字母长度都算1，例如`/`、`*`、`?`和普通字母

一个示例：

~~~java
String str1 = "/**";
String str2 = "/{*var}";
PathPattern pattern1 = PathPatternParser.defaultInstance.parse(str1);
PathPattern pattern2 = PathPatternParser.defaultInstance.parse(str2);
System.out.println(pattern1.compareTo(pattern2)); // -81
System.out.println(str1.compareTo(str2)); // -81
~~~

### `PathPatternParser`

用于解析字符串创建`PathPattern`,提供如下可配置字段以配置创建的`PathPattern`:

* `caseSensitive`：路径模式匹配是否区分大小写
* `pathOptions`解析URL路径的可选项

并且提供一个全局的静态实例：

~~~java
public static final PathPatternParser defaultInstance;
~~~

通过`parse()`方法创建实例：

~~~java
String patternStr = ....;
PathPattern parse = PathPatternParser.defaultInstance.parse(patternStr);
~~~

# SpringMVC重要组件

## `HandlerMapping`

`HandlereMapping`的主要类层次结构如下：

![HandlerMapping](https://gitee.com/wangziming707/note-pic/raw/master/img/HandlerMapping.png)

可以看到所有的`HandlerMapping`实现都拓展自`AbstractHandlerMapping`抽象类，它为其他实现提供公用的逻辑。

在具体了解`AbstractHandlerMapping`前，我们需要先粗略了解下面类/接口：

* `RequestPath`，扩展了`PathContainer`，表示一个请求路径。有如下关键方法：
    * `contextPath()`:获取上下文路径
    * `pathWithinApplication()`:获取app中的相对路径。
* `CorsConfiguration`：cors请求的配置类。例如设置允许的`cors`请求源，允许的cors HTTP方法等。
* `CorsConfigurationSource`：根据请求生成`CorsConfiguration`实例。
* `AbstractHandlerMapping.CorsInterceptor`：cors拦截器，委托`CorsProcessor`对cors请求进行预处理，
* `CorsProcessor`:`cors`请求的处理类，根据`CorsConfiguration`配置处理cors请求。例如如果当前请求源不在配置范围内，就拒绝请求；针对预检请求生成响应等。
* `AbstractHandlerMapping.PreFlightHandler`：预检请求的处理器，对于预检请求，不需要实际处理请求。
* `MappedInterceptor`:在SpringMVC配置中，如果配置`Interceptor`是，有配置`includePatterns`或者，`excludePatterns`,那么在注册这个拦截器是，会自动将其包装成一个`MappedInterceptor`

### `AbstractHandlerMapping`

`AbstractHandlerMapping`实现了`WebApplicationObjectSupport`，以提供容器上下文的访问支持，可以通过`WebApplicationObjectSupport.obtainApplicationContext()`方法获得对应的容器实例。

`AbstractHandlerMapping`实现`HandlerMapping.getHandler()`核心方法，其主要逻辑如下：

* 调用`getHandlerInternal()`获取`handler`（该方法是抽象方法，由`AbstractHandlerMapping`的实现类来具体实现）
* 如果上一步获取的`handler`为`null`，使用**可配置字段`this.defaultHandler` **作为`handler`，如果仍未`null`，方法直接返回`null`
* 如果`handler`类型为`String`，将该字符串视为处理器在容器中的Bean Id，尝试从`ApplicationContext`中获取对应handler Bean实例
    * 如果容器中没有对应的`handler`，抛出一个`NoSuchBeanDefinitionException `异常
* 调用`initLookupPath()`方法向`request`域中缓存一个`lookupPath`，用以路径匹配：
    * 如果当前`HandlerMapping.usesPathPatterns()`为`true`，则表示`HandlerMapping`会使用`PathPattern`进行路径匹配，该路径匹配器期望一个`PathContainer`作为路径。
        * 调用`ServletRequestPath.parse()`方法解析请求，获取一个`RequestPath`作为`lookupPath`
        * `lookupPath`在`request`域中的属性名是`ServletRequestPathUtils.PATH_ATTRIBUTE`
    * 否则,会使用`AntPathMatcher`进行路径匹配，该路径匹配器期望一个`String`作为路径。
        * 调用`UrlPathHelper.getLookupPathForRequest()`方法解析请求，获取一个`String`作为`lookupPath`
        * `lookupPath`在`request`域中的属性名是`UrlPathHelper.PATH_ATTRIBUTE`
* 调用`getHandlerExecutionChain()`将`handler`包装成一个`HandlerExecutionChain`并设置与请求匹配的拦截器
    * 根据`handler`创建一个`HandlerExecutionChain`实例
    * 遍历`this.adaptedInterceptors`
        * 如果`interceptor`类型是`MappedInterceptor`，调用`MappedInterceptor.matches()`判断当前拦截器是否匹配请求，如果是，则将其注册到`HandlerExecutionChain`中
        * 否则，说明`interceptor`匹配任意请求，直接将其注册到`HandlerExecutionChain`中
    * 返回`HandlerExecutionChain`实例
* 处理cors跨源请求：
    * 判断是否需要进行`cors`预处理，满足下面条件之一的才会进行cors处理，否则直接跳过下面步骤。
        * 如果当前请求是非简单请求的预检请求(请求类型是`OPTIONS`，并且请求头中的`Origin`和`Access-Control-Request-Method`字段不为空)
        * 能获得`CorsConfigurationSource`实例(当前的`handler`本身就是一个`CorsConfigurationSource`，或者可**配置字段`this.corsConfigurationSource`**不为空)
    * 根据`handler`获取一个本地`CorsConfiguration`实例
    * 根据`this.corsConfigurationSource`获取一个全局`CorsConfiguration`实例
    * 合并全局和本地`CorsConfiguration`实例
    * 处理`cors`请求：
        * 如果是非简单请求的预检请求，创建一个`PreFlightHandler`处理器，它同时也是一个`CorsInterceptor`拦截器。根据它创建`HandlerExecutionChain`,替代之前生成的`HandlerExecutionChain`
        * 否则，为`HandlerExecutionChain`添加一个`CorsInterceptor`即可

* 返回之前`HandlerExecutionChain`实例

**`this.adaptedInterceptors`初始化：**

在`getHandler()`中`AbstractHandlerMapping`会遍历`this.adaptedInterceptors`，将匹配的拦截器添加到`HandlerExecutionChain`中，现在我们来看一下它是如何初始化该字段的：

* `AbstractHandlerMapping`有**可配置字段`this.interceptors`**，该字段允许配置`HandlerInterceptor`和`WebRequestInterceptor`的拦截器

* 请求`HandlerMapping`所在`ApplicationContext`容器上下文中所有的`MappedInterceptor`类型的Bean实例，添加到`this.adaptedInterceptors`中
* 将`this.interceptors`添加到`this.adaptedInterceptors`中。

# 注解式`Controller`的处理器

`org.springframework.web.method.HandlerMathod`是注解式`Controller`中处理器的抽象，对应一个`@Controller`类的`@RequestMapping`方法

`HandlerMethod`处理器方法一定有注解(方法体和方法参数上),并且框架需要根据这些注解声明来控制处理器行为，所以`HandlerMethod`很自然地继承了`org.springframework.core.annotation.AnnotatedMethod`，它是一个注解方法的抽象。

和方法紧密相关的一个概念就是方法的参数和返回值。`org.springframework.core.MethodParameter`就是对方法参数和返回值的抽象

## `MethodParameter`

`MethodParameter`是Spring对方法参数的抽象，它封装了方法参数的各种信息：如参数类型、参数名、参数位置、参数注解等

它有如下重要字段：

~~~java
private final Executable executable; //参数所在的方法(普通方法/构造器)
private final int parameterIndex; //参数在方法中的位置: -1表示返回值，0表示第一个入参
private int nestingLevel; //嵌套层数
Map<Integer, Integer> typeIndexesPerLevel; //嵌套索引映射
private volatile ParameterNameDiscoverer parameterNameDiscoverer; // 获取参数名parameterName需要用到该字段
private volatile Class<?> containingClass; 
//下面字段使用懒加载，在第一次调用对应的getter方法时初始化字段值。
private volatile Parameter parameter; //当前方法参数对应的 Parameter 实例。
private volatile Class<?> parameterType; //参数类型
private volatile Type genericParameterType; //参数类型
private volatile Annotation[] parameterAnnotations; //参数上的注解
volatile String parameterName; //参数名
private volatile MethodParameter nestedMethodParameter; // 嵌套方法参数
~~~

可以用对应的`getter`方法访问这些字段。

## `AnnotatedMethod`

`AnnotatedMethod`是对注解方法的抽象，它封装了方法、方法参数、方法注解信息，可以通过它提供的如下方法访问：

~~~java
public final Method getMethod();//获取当前方法对应的Method实例
protected final Method getBridgedMethod(); // 获取当前方法的桥方法的Method实例(在继承重写方法时，如果重写的方法返回值与父类方法不同，那么编译器会自动生成一个桥方法)
public final MethodParameter[] getMethodParameters(); // 获取入参的方法参数列表
public MethodParameter getReturnType(); //获取返回值对应的方法参数
public boolean isVoid(); //返回值是否是 void
public <A extends Annotation> A getMethodAnnotation(Class<A> annotationType); //获取方法注解
public <A extends Annotation> boolean hasMethodAnnotation(Class<A> annotationType); //判断方法上是否存在指定注解
~~~

## `HandlerMethod`

创建一个`HandlerMethod`至少需要一个`@RequestMapping`方法对应的`Method`实例，和`@RequestMapping`方法所在类的Bean实例，即如下构造器：

~~~java
public HandlerMethod(Object bean, Method method);
~~~

`HandlerMethod`的如下字段和`bean`有关：

~~~java
private final Object bean;   //@RequestMapping方法所在类的实例
private final Class<?> beanType; //@RequestMapping方法所在类的类型
~~~

可以通过对应的`getter`方法访问它们。

### 解析`bean`

也可以通过提供一个`beanName`和`BeanFactory`来指定`@RequestMapping`所在类的实例：

~~~java
public HandlerMethod(String beanName, BeanFactory beanFactory, Method method);
~~~

此时`beanName`字符串会被绑定到`this.bean`上，`beanFactory`会绑定到`this.beanFactory`上

可以看到通过这个构造器创建的实例，其`this.bean`字段上绑定的不是真正的`bean`实例。而是一个`beanName`字符串。

所以它不能对外提供真正的`bean`实例，需要解析当前的`bean`。

此时可以通过`createWithResolvedBean()`方法解析`this.bean`字段，返回一个新的`HandlerMethod`，其逻辑如下：

* 如果当前实例的`this.bean`字符类型是字符串，那么调用`this.beanFactory.getBean(beanName)`获取`handler`实例，否则直接使用`this.bean`作为`handler`
* 创建一个新的`HandlerMethod`实例并返回，新实例的`this.resolvedFromHandlerMethod`字段为当前`HandlerMethod`实例

所以如果一个`HandlerMethod`是通过解析产生的，那么它的`this.resolvedFromHandlerMethod`字段就不为空，并且指向其源`HandlerMethod`

### 处理是否参数校验

`HandlerMethod`提供了两个方法，告诉客户端当前处理器方法是否需要进行参数校验：

~~~java
public boolean shouldValidateArguments();  //需要校验方法入参
public boolean shouldValidateReturnValue(); //需要检验方法返回值
~~~

这对应`HandlerMethod`的如下字段：

~~~java
private final boolean validateArguments;
private final boolean validateReturnValue;
~~~

直接使用构造器创建的`HandlerMethod`的，这两个字段的值都默认为`false`,也就是说默认情况下，`HandlerMethod`不需要进行参数校验。

可以调用`createWithValidateFlags()`创建一个新的，允许参数校验的`HandlerMethod`实例，该主要逻辑如下：

* 创建一个新的`HandlerMethod`
* 新的`HandlerMethod`的`this.resolvedFromHandlerMethod`为当前`HandlerMethod`
* 新的`HandlerMethod`的`this.validateArguments`值为方法`MethodValidationInitializer.checkArguments()`返回值，该方法：
  * 如果当前项目中不存在`jakarta.validation.Validator`类，说明无法进行校验，返回`false`
  * 如果`HandlerMethod`所在的类上有`Validated`注释，那么方法验证由AOP代理进行，返回`false`
  * 遍历方法的参数，如果参数满足下面条件之一，说明需要进行参数验证，返回`true`,否则返回`false`
    * 参数上有`jakarta.validation.Constraint`注解
    * 参数上有`jakarta.validation.Valid`注解，并且当前参数类型是基于键值或者索引的容器(参数类型为`List`,`Map`或者数组)
    * 参数类型是容器类型，如果容器中元素的类型有注解`@Constraint`或者`@Valid`
* 新的`HandlerMethod`的`this.validateReturnValue`值为方法`MethodValidationInitializer.checkReturnValue()`返回值，该方法：
  * 如果当前项目中不存在`jakarta.validation.Validator`类，说明无法进行校验，返回`false`
  * 如果`HandlerMethod`所在的类上有`Validated`注释，那么方法验证由AOP代理进行，返回`false`
  * 如果返回值有注解`@Constraint`或者`@Valid`，返回`true`,否则返回`false`

### 处理`ResponseStatus`

如果当前处理器方法上有`@ResponseStatus`注释，那么：

* `this.responseStatus`字段会设置为`@ResponseStatus`的`code`值
* `this.responseStatusReason`会设置为`@ResponseStatus`的`reason`值
  * 如果`this.messageSource`不为空，那么`reason`会进行国际化处理

可以通过对应的`getter`方法方法这两个字段。

# 注解式`Controller`的`HandlerMapping`

`RequestMappingHandlerMapping`负责维护`HandlerMathod`和请求之间的映射关系。根据继承关系我们知道它有以下主要父类：

* `AbstractHandlerMethodMapping`
* `RequestMappingInfoHandlerMapping`

我们从父类开始依次分析：

## `AbstractHandlerMethodMapping`

`AbstractHandlerMethodMapping`是定义请求和`HandlerMethod`之间的映射的抽象基类。

它的泛型`T`类型表示`HandlerMethod`的`mapping`映射，`T`包含请求与`HandlerMethod`的进行匹配时需要的条件信息。如映射到某个`HandlerMethod`需要的请求的路径/类型/请求头 等信息。每个`HandlerMethod`都需要绑定一个`T`用于请求映射。

`AbstractHandlerMethodMapping`定义下面内部类来辅助它提供处理器映射器的相关功能：

* `MappingRegistration`:一个`T`映射的注册项，主要包含一对`HandlerMethod`和对应的`T`映射实例。除此之外还有如下属性
    * `mappingName`：根据`HandlerMethodMappingNamingStrategy`生成的name
    * `directPaths`：映射T的直接路径集合
    * `corsConfig`:当前处理器是否启用了cors
* `Match`：对`T`映射和对应的`HandlerMathod`的浅包装，主要为了通过比较器在请求上下文中找到最佳匹配。
* `MatchComparator`:可以客户端提供的比较器对`Match`进行比较

* `MappingRegistry`:一个`T`映射的注册表，维护`HandlerMethod`的所有映射关系。并提供给相应的查找方法。

### `MappingRegistry`

它维护以下`Map`:

~~~java
private final Map<T, MappingRegistration<T>> registry = new HashMap<>(); 
// T 到 MappingRegistration的映射
private final MultiValueMap<String, T> pathLookup = new LinkedMultiValueMap<>();
//路径字符串到List<T>的映射 因为一个path可能匹配多个处理器
private final Map<String, List<HandlerMethod>> nameLookup = new ConcurrentHashMap<>();
//name到List<HandlerMethod>的映射。因为name
private final Map<HandlerMethod, CorsConfiguration> corsLookup = new ConcurrentHashMap<>();
//HandlerMethod到CorsConfiguration的映射
~~~

其核心方法是`MappingRegistry.register()`，其方法签名如下：

~~~java
public void register(T mapping, Object handler, Method method);
~~~

主要逻辑为：

* 根据提供的`handler`和`method`，生成一个`HandlerMethod`实例
* 检查是否已经注册，如果`this.registry`中已经存在了同样的`mapping`和`HandlerMethod`对，抛出一个`IllegalStateException`异常
* 调用`HandlerMethod.createWithValidateFlags()`启用其方法校验
* 



# 注解式`Controller`的`HandlerAdapter`







