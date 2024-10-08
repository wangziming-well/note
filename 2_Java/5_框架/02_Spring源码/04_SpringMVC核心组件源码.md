# DispatcherServlet

`DispatcherServlet`一方面作为Web容器的一个Servlet来接受请求，另一方面从对应的`WebApplicationContext`中获取MVC组件，将请求处理的任务委托给这些组件。

`DispatcherServlet`作为`Servlet`，其生命周期受Web容器管理：

* 在`Servlet.init()`中，`DispatcherServlet`会初始化`WebApplicationContext`并请求相关的`Bean`组件。
* 在`Servlet.service()`方法中，`DispatcherServlet`处理Web请求。

`DispatcherServlet`继承自`FrameworkServlet`,而`FrameworkServlet`是SpringMVC的基础servlet。在基于JavaBean的整体解决方案中提供与Spring应用程序上下文的集成。

## 初始化

**note:**本节陈述中提到的方法如果没有显示指定，默认为`FrameworkServlet`声明的方法

`DispatcherServlet`初始化的核心逻辑在 `FrameworkServlet.initWebApplicationContext()`方法中。这个方法会初始化`WebApplicationContext`并将其注册到`this.webApplicationContext`字段上。其主要逻辑为：

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



# `HandlerMapping`

`HandlereMapping`的主要类层次结构如下：

![HandlerMapping](https://gitee.com/wangziming707/note-pic/raw/master/img/HandlerMapping.png)

可以看到所有的`HandlerMapping`实现都拓展自`AbstractHandlerMapping`抽象类，它为其他实现提供公用的逻辑。简单概述一下其主要实现类：

* `SimpleUrlHandlerMapping`：维护一个`url`到`handler`实例的映射关系(创建实例时提供该映射，包括`handler`实例)。
* `BeanNameUrlHandlerMapping`:提供从`url`到`beanName`的简单映射，只有`beanName`或者别名以`/`为前缀的Bean才会被加入到该映射关系。所以想要使用该映射，需要将Bean命名为类型`/foo`这样的形式。
* `RequestMappingHandlerMapping`:注解式`Controller`的处理器映射
* `RouterFunctionMapping`:根据容器中的`RouterFunction`定义的映射关系，根据请求获取一个`HandlerFunction`作为`handler`

## `AbstractHandlerMapping`涉及接口

在具体了解`AbstractHandlerMapping`前，我们需要先粗略了解下面类/接口：

* `RequestPath`，扩展了`PathContainer`，表示一个请求路径。有如下关键方法：
    * `contextPath()`:获取上下文路径
    * `pathWithinApplication()`:获取上下文路径后的请求路径。
* `CorsConfiguration`：cors请求的配置类。例如设置允许的`cors`请求源，允许的cors HTTP方法等。
* `CorsConfigurationSource`：根据请求生成`CorsConfiguration`实例。
* `AbstractHandlerMapping.CorsInterceptor`：cors拦截器，委托`CorsProcessor`对cors请求进行预处理，
* `CorsProcessor`:`cors`请求的处理类，根据`CorsConfiguration`配置处理cors请求。例如如果当前请求源不在配置范围内，就拒绝请求；针对预检请求生成响应等。
* `AbstractHandlerMapping.PreFlightHandler`：预检请求的处理器，对于预检请求，不需要实际处理请求。
* `MappedInterceptor`:在SpringMVC配置中，如果配置`Interceptor`时，有配置`includePatterns`或者，`excludePatterns`,那么在注册这个拦截器是，会自动将其包装成一个`MappedInterceptor`

## `AbstractHandlerMapping`

`AbstractHandlerMapping`主要提供将`handler`对象和拦截器包装成`HandlerExecutionChain`的逻辑

`AbstractHandlerMapping`实现了`WebApplicationObjectSupport`，以提供容器上下文的访问支持，可以通过`WebApplicationObjectSupport.obtainApplicationContext()`方法获得对应的容器实例。

`AbstractHandlerMapping`实现`HandlerMapping.getHandler()`核心方法，其主要逻辑如下：

* **调用`getHandlerInternal()`**获取`handler`（该方法是抽象方法，由`AbstractHandlerMapping`的实现类来具体实现）
* 如果上一步获取的`handler`为`null`，使用**可配置字段`this.defaultHandler` **作为`handler`，如果仍未`null`，方法直接返回`null`
* 如果`handler`类型为`String`，将该字符串视为处理器在容器中的Bean Id，尝试从`ApplicationContext`中获取对应handler Bean实例
    * 如果容器中没有对应的`handler`，抛出一个`NoSuchBeanDefinitionException `异常
* 调用`initLookupPath()`方法向`request`域中缓存一个`lookupPath`，用以路径匹配，并返回对应的相对路径：
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

# HandlerAdapter

`HandlerAdapter`的主要类层次结构如下：

![HandlerAdapter](https://gitee.com/wangziming707/note-pic/raw/master/img/HandlerAdapter.png)

其主要实现类为：

* `HandlerFunctionAdapter`:适配`HandlerFunction`类型的`handler`(由`RouterFunctionMapping`提供)
* `HttpRequestHandlerAdapter`：适配`HttpRequestHandler`类型的`handler`
* `RequestMappingHandlerAdapter`:适配`HandlerMethod`类型的`handler`
* `SimpleControllerHandlerAdapter`：适配`Controller`类型的`handler`
* `SimpleServletHandlerAdapter`:设配`Servlet`类型的`handler`
