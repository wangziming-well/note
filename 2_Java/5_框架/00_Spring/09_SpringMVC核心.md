

# MVC设计模式

 MVC是一种设计模式，常用于开发web应用程序的框架中。MVC的意思是Model-View-Controller（模型-视图-控制器）。它将web应用程序分为三个主要组件：模型、视图和控制器。每个组件都负责不同的功能，但又相互关联，以便应用程序可以正确地工作。

* Model（模型） 模型是应用程序中的数据和逻辑组件，它处理应用程序数据和业务逻辑。模型是独立于用户界面和控制器的，因此可以在不影响应用程序其他部分的情况下修改数据和逻辑。
* View（视图） 视图是应用程序中的用户界面组件，它负责呈现模型中的数据。视图通常是HTML模板，它显示给用户的是模型中的数据。视图不处理数据或逻辑，它只负责呈现它们。
* Controller（控制器） 控制器是应用程序中的中介组件，负责接收用户请求，根据请求通知模型更新数据，并选择合适的视图响应给用户

![mvc设计模式](https://gitee.com/wangziming707/note-pic/raw/master/img/mvc%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F.png)

MVC设计模式是Web开发中的一种重要的设计模式，它能够提供良好的结构化方式，使得Web应用程序的不同部分更易于维护和扩展，提高了应用程序的质量和可靠性。

# DispatcherServlet

Spring MVC和其他许多Web框架一样，是围绕前端控制器模式设计。Spring MVC的前端控制器就是 `DispatcherServlet`，它接受所有请求，并为请求处理提供共享算法，实际的请求处理工作由 `DispatcherServlet`委派给可配置的委托组件执行。

一方面`DispatcherServlet` 和其他Servlet一样，遵循 `Servlet` 规范，使用Java配置或在 `web.xml` 中进行声明和映射。

另一方面`DispatcherServlet `使用Spring配置来发现它在请求映射、视图解析、异常处理等方面需要的委托组件。这些组件由`WebApplicationContext`容器管理。

`WebApplicationContext`扩展了`ApplicationContext`，在其基础上提供了web应用的支持:

* 支持web相关的bean的scope: session和request
* 提供获取当前应用程序的`ServletContext`的方法

所以想要SpringMVC可用，需要向servlet注册一个`DispatcherServlet` ，并且为`DispatcherServlet`创建并绑定一个`WebApplicationContext`。

并且在默认情况下，`DispatcherServlet`会将与之关联的`WebApplicationContext`绑定到所在的`ServletContext`中。注册的属性名为`FrameworkServlet.SERVLET_CONTEXT_PREFIX`前缀拼接上`DispatcherServlet`的servlet名称

下面介绍通过`web.xml`配置和Java配置 来注册`DispatcherServlet` 

## web.xml配置

可以直接在Servlet的 `web.xml` 配置文件中注册和初始化`DispatcherServlet`:

~~~xml
<web-app>
	<!--向servlet注册DispatcherServlet -->
    <servlet>
        <servlet-name>app</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <!--指定与DispatcherServlet绑定的WebApplicationContext的xml配置文件位置 -->
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>classpath:springmvc.xml</param-value>
        </init-param>
        <!--保证前端控制器第一时间启动 -->
        <load-on-startup>1</load-on-startup>
    </servlet>
    <servlet-mapping>
        <servlet-name>app</servlet-name>
        <url-pattern>/app/*</url-pattern>
    </servlet-mapping>
</web-app>
~~~

## Java配置

可以用下面Java代码配置注册并初始化 `DispatcherServlet`，这些代码会在Servlet容器启动时并发现并执行。

~~~java
public class MyWebApplicationInitializer implements WebApplicationInitializer {

    @Override
    public void onStartup(ServletContext servletContext) {

        // 加载WebApplicationContext
        AnnotationConfigWebApplicationContext context = new AnnotationConfigWebApplicationContext();
        context.register(AppConfig.class);

        // 创建并注册DispatcherServlet
        DispatcherServlet servlet = new DispatcherServlet(context);
        ServletRegistration.Dynamic registration = servletContext.addServlet("app", servlet);
        registration.setLoadOnStartup(1);
        registration.addMapping("/app/*");
    }
}
~~~

### 发现Java配置的原理

web能够容器发现并执行`WebApplicationInitializer`的实现是因为servlet3.0规范提供了方便第三方框架在容器启动时做一些初始化动作的机制：

* Servlet容器启动时会扫描应用中每个jar包的`META-INF/services/javax.servlet.ServletContainerInitializer`文件，这个文件会指定 `ServletContainerInitializer`的实现类
* Servlet容器会执行指定 `ServletContainerInitializer`的实现类的`onStartup()`方法
*  `ServletContainerInitializer`的实现类可以被` @HandlesTypes`的注解注释，该注解声明一些类型，这些类型会作为入参被传入上一步的`onStartup()`方法

在spring-web.jar包中，就存在文件`META-INF/services/javax.servlet.ServletContainerInitializer`，该文件内容为：

~~~java
org.springframework.web.SpringServletContainerInitializer
~~~

指定了`SpringServletContainerInitializer`实现，这个实现类简略声明如下：

~~~java
@HandlesTypes(WebApplicationInitializer.class)
public class SpringServletContainerInitializer implements ServletContainerInitializer {
    	@Override
	public void onStartup(@Nullable Set<Class<?>> webAppInitializerClasses, ServletContext servletContext)
			throws ServletException {
		//webAppInitializerClasses就是指定的WebApplicationInitializer类型，方法体主要逻辑就是执行WebApplicationInitializer的onStartup()方法
	}
~~~

所以我们能够直接声明`WebApplicationInitializer`的实现，不需要主动注册它到容器，而是等待它被容器扫描发现执行。

### WebApplicationInitializer抽象实现

我们可以直接实现`WebApplicationInitializer`接口来完成mvc的Java配置，但实际上Spring MVC提供了现成的`WebApplicationInitializer`抽象实现，能够帮助我们完成一些通用逻辑。

如果使用基于Java代码的Spring配置的应用程序，可以实现`AbstractAnnotationConfigDispatcherServletInitializer`:

~~~java
public class MyWebAppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {

    @Override
    protected Class<?>[] getRootConfigClasses() {
        return null;
    }

    @Override
    protected Class<?>[] getServletConfigClasses() {
        return new Class<?>[] { MyWebConfig.class };
    }

    @Override
    protected String[] getServletMappings() {
        return new String[] { "/" };
    }
}
~~~

如果使用基于XML的Spring配置，可以直接从 `AbstractDispatcherServletInitializer` 扩展：

~~~java
public class MyWebAppInitializer extends AbstractDispatcherServletInitializer {

    @Override
    protected WebApplicationContext createRootApplicationContext() {
        return null;
    }

    @Override
    protected WebApplicationContext createServletApplicationContext() {
        XmlWebApplicationContext cxt = new XmlWebApplicationContext();
        cxt.setConfigLocation("/WEB-INF/spring/dispatcher-config.xml");
        return cxt;
    }

    @Override
    protected String[] getServletMappings() {
        return new String[] { "/" };
    }
}
~~~

`AbstractDispatcherServletInitializer `提供了一种方便的方法来添加 `Filter` 实例，并让它们自动映射到 `DispatcherServlet`:

~~~java
public class MyWebAppInitializer extends AbstractDispatcherServletInitializer {

    // ...

    @Override
    protected Filter[] getServletFilters() {
        return new Filter[] {
            new HiddenHttpMethodFilter(), new CharacterEncodingFilter() };
    }
}
~~~

每个 filter 都根据其具体类型添加了一个默认名称（name），并自动映射到 `DispatcherServlet`。

`AbstractDispatcherServletInitializer` 的 `isAsyncSupported` protected 方法提供了一个单一的地方来启用对 `DispatcherServlet` 和所有映射到它的 filter 的异步支持。默认情况下，这个标志被设置为 `true`。

如果需要进一步定制 `DispatcherServlet` 本身，可以重写 `AbstractDispatcherServletInitializer.createDispatcherServlet()` 方法

## Context层次结构

一般情况下，有一个单一的`DispatcherServlet`和与之绑定的单一的  `WebApplicationContext`就足够了。 

但是应用程序中在多个`DispatcherServlet`(（或其他 `Servlet`）)时，此时就会有多个对应的 `WebApplicationContext`。一些作为基础设置的Bean可能每个`DispatcherServlet`都需要，如果向每个`WebApplicationContext`中注册，那么不仅麻烦，而且浪费。那么定义一个公共的父 `WebApplicationContext`是最佳的选择。

Root `WebApplicationContext`就是这样的父容器。 它通常包含基础设施Bean，例如需要在多个 `Servlet` 实例中共享的数据存储库和业务服务。这些Bean会被子`WebApplicationContext`继承以供不同的`Servlet`使用。下图显示了这种关系：

![mvc-context-hierarchy](https://gitee.com/wangziming707/note-pic/raw/master/img/mvc-context-hierarchy.png)

### ContextLoaderListener

可以通过`ContextLoaderListener`初始化并注册根`WebApplicationContext`。

`ContextLoaderListener`作为Servlet的标准监听器，会在Web容器初始化时创建并注册根`WebApplicationContext`

如果创建`ContextLoaderListener`时没有通过构造器提供`WebApplicationContext`实例。那么`ContextLoaderListener`默认会自己创建一个`XmlWebApplicationContext`,并通过ServletContext的初始化参数`contextConfigLocation`指定的位置获取XML配置文件加载到`XmlWebApplicationContext`中

### web.xml配置

所以，如果需要根`WebApplicationContext`，只需要在servlet的web.xml配置中注册一个`ContextLoaderListener`并指定初始化参数`contextConfigLocation`即可：

~~~xml
<web-app>
    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>

    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>/WEB-INF/root-context.xml</param-value>
    </context-param>

    <servlet>
        <servlet-name>app1</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>/WEB-INF/app1-context.xml</param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>
    <servlet-mapping>
        <servlet-name>app1</servlet-name>
        <url-pattern>/app1/*</url-pattern>
    </servlet-mapping>

</web-app>
~~~

### Java配置

我们可以直接实现`WebApplicationInitializer`接口来完成mvc的Java配置，但实际上Spring MVC提供了现成的`WebApplicationInitializer`抽象实现，能够帮助我们完成一些通用逻辑。其中包括了创建并设置根 `WebApplicationContext` 

下面的例子配置了一个 `WebApplicationContext` 的层次结构：

~~~java
public class MyWebAppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {

    @Override
    protected Class<?>[] getRootConfigClasses() {
        return new Class<?>[] { RootConfig.class };
    }

    @Override
    protected Class<?>[] getServletConfigClasses() {
        return new Class<?>[] { App1Config.class };
    }

    @Override
    protected String[] getServletMappings() {
        return new String[] { "/app1/*" };
    }
}
~~~

`AbstractAnnotationConfigDispatcherServletInitializer`会在其`onStartup()`方法(该方法会在容器创建时被servlet调用)中为创建注册根 `WebApplicationContext` 做了下面几个动作：

* 如果`getRootConfigClasses()`方法返回不为空，那么创建一个`AnnotationConfigWebApplicationContext`,并注册配置类
* 创建一个`ContextLoaderListener`并在构造器中传递上一步创建的`WebApplicationContext`
* 将`ContextLoaderListener`注册到`ServletContext`中

所以如果不想注册根 `WebApplicationContext` ，只需要重写`getRootConfigClasses() `方法返回值为null。

# SpringMVC组件

我们已经知道`DispatcherServlet`在收到请求后会将请求委托给特殊的Bean组件来处理请求。这些特殊的Bean组件会被`WebApplicationContext`管理，并被对应的`DispatcherServlet`检测发现。这些组件会实现SpringMVC框架约定的标准接口：

| Bean 类型                                 | 说明                                                         |
| :---------------------------------------- | :----------------------------------------------------------- |
| `HandlerMapping`                          | 将一个请求和一个用于前后处理的拦截器链一起映射到一个处理程序（handler）。 |
| `HandlerAdapter`                          | 帮助 `DispatcherServlet` 调用映射到请求的处理程序（handler），不管处理程序实际上是如何被调用的。例如，调用一个有注解的controller需要解析注解的问题。`HandlerAdapter` 的主要目的是将 `DispatcherServlet` 从这些细节中屏蔽掉。 |
| `HandlerExceptionResolver`                | 解决异常的策略，可能将它们映射到处理程序、HTML error 视图或其他目标。 |
| `ViewResolver`                            | 将处理程序返回的基于 `String` 的逻辑视图名称解析为实际的 `View`（视图），并将其渲染到响应。 |
| `LocaleResolver`, `LocaleContextResolver` | 解析客户端使用的 `Locale`，可能还有他们的时区，以便能够提供国际化的视图。 |
| `ThemeResolver`                           | 解析你的web应用可以使用的主题（theme）--例如，提供个性化的布局。 |
| `MultipartResolver`                       | 在一些 multipart 解析库的帮助下，解析一个 multipart 请求（例如，浏览器表单文件上传）的抽象。 |
| `FlashMapManager`                         | 存储和检索 "输入" 和 "输出" `FlashMap`，可用于将属性从一个请求传递到另一个请求，通常跨越重定向。 |

可以声明这些组件到`WebApplicationContext `中。 `DispatcherServlet `会检查 `WebApplicationContext `中的组件。如果没有匹配的Bean类型，它将使用` DispatcherServlet.properties` 中所指示的的默认类型。

通常情况下可以通过Java或XML声明所需的Bean。

## 请求处理流程

`DispatcherServlet` 接受到请求的处理方式如下：

- 将与之绑定的`WebApplicationContext` 作为一个属性（attribute）绑定在请求（request）中，controller和进程中的其他元素可以使用。默认的属性名是`DispatcherServlet.WEB_APPLICATION_CONTEXT_ATTRIBUTE` 。
- 绑定locale 解析器到request上(如果有)，以便让流程中的元素在处理请求（渲染视图、准备数据等）时解析要使用的 locale。
- 绑定theme 解析器到request上(如果有)，以让诸如视图等元素决定使用哪个主题。
- 如果指定了multipart file 解析器，如果是multipart请求，该请求将被包裹在一个 `MultipartHttpServletRequest` 中，以便由流程中的其他元素进一步处理。
- handler mapping会匹配一个handler处理器。如果能够匹配到处理器，将运行与该处理器相关的执行链（预处理程序、后处理程序和 controller），以准备渲染的模型（model）。
- 如果有 model 返回，就会渲染View。如果没有返回 model（也许是由于预处理器或后处理器拦截了请求，也许是出于安全原因），就不会渲染视图，因为请求可能不需要视图。

请求中抛出的异常由 `WebApplicationContext` 中声明的 `HandlerExceptionResolver` Bean来处理

## HandlerMapping

HandlerMapping帮助DispatcherServlet进行Web请求的URL到具体处理类之间的匹配。

其定义如下：

~~~java
public interface HandlerMapping {
	@Nullable
	HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception;

}
~~~

定义非常简单，`getHandler()`方法根据`HttpServletRequest `获取 `HandlerExecutionChain `以获取处理器

SpringMVC提供了许多`HandlerMapping`的实现:

*  `BeanNameUrlHandlerMapping`：Web的请求路径对应的是Controller在容器中的beanName；它会直接将`http://localhost:8080/spring/demo`映射到`DemoController`上

* `SimpleUrlHandlerMapping`：通过map管理请求url和handler之间的定义
* `ControllerClassNameHandlerMapping`
* `RequestMappingHandlerMapping`:支持 `@RequestMapping` 注解的方法

### HandlerMapping执行顺序

容器中可以有多个`HandlerMapping`。

收到Web请求时，`DispatcherServlet`会根据可用的`HandlerMapping`实例的优先级进行遍历。先调用优先级高的`HandlerMapping`，直到某个`HandlerMapping`匹配到`Handler`为止。

`HandlerMapping`的优先级由Spring框架内`Ordered`接口定义。在定义`HandlerMapping`实例时，我们可以通过设置其order属性来指定其优先级

order值越低优先级越高。

## HandlerInterceptor

handler拦截器必须实现  `HandlerInterceptor`，它提供三个方法，用来进行预处理和后处理：

- `preHandle(..)`: 在实际 handler 运行之前，该方法返回一个布尔值，来指示中断或继续执行链的处理：
  - 返回 `true` 时，handler 执行链继续进行
  - 返回 `false` 时， `DispatcherServlet` 认为拦截器本身已经处理了请求（例如，渲染了一个适当的视图），并且不继续执行其他拦截器和执行链中的实际 handler
- `postHandle(..)`: handler 运行后
- `afterCompletion(..)`: 在整个请求完成后

**注意**：`postHandle` 方法在 `@ResponseBody` 和 `ResponseEntity` 方法中用处不大，因为这些方法的响应是在 `HandlerAdapter` 中和 `postHandle` 之前写入和提交的。对于这种情况，可以实现 `ResponseBodyAdvice`，并把它声明为一个 Controller Advice Bean，或者直接在 `RequestMappingHandlerAdapter `上配置它。

### HandlerInterceptor&Filter

Servlet组件中的`Filter`和`HandlerInterceptor`功能类似，都提供请求拦截功能，但它们之间也存在差别，`DispatcherServlet`也是一个Servlet

`Filter`对请求的拦截是在请求进入`DispatcherServlet`之前和`DispatcherServlet`处理完请求之后的，在SpringMVC的框架下，`Filter`是针对`DispatcherServlet`的执行进行拦截

而`HandlerInterceptor`对请求的拦截都是在`DispatcherServlet`处理流程内部的，针对`Handler`的执行进行拦截

## HandlerExceptionResolver

在我们Controller的定义中:

~~~java
public interface Controller {
	ModelAndView handleRequest(HttpServletRequest request, HttpServletResponse response) throws Exception;
}
~~~

我们发现处理请求的方法`handleRequest`直接将所有异常抛出,这实际上违反了我们处理异常的方法:

对于可能抛出多个异常的方法，我们需要分别抛出，而不是直接用所有可能抛出的异常的父类作为抛出异常。

但实际上这中异常设计是不得已而为之的:

处理Web请求时，可能的用到的逻辑和抛出的异常无法预测也无法限定，所以SpringMVC框架直接将其全部抛出

`DispatcherServlet`会将抛出的异常委托给`HandlerExceptionResolver`链去统一处理。其定义如下：

~~~java
public interface HandlerExceptionResolver {
	ModelAndView resolveException(HttpServletRequest request, HttpServletResponse response, @Nullable Object handler, Exception ex);
}
~~~

处理异常，并返回相应的视图

SpringMVC提供了一些可用的 `HandlerExceptionResolver` 实现

* `ResponseStatusExceptionResolver`：解析带有 `@ResponseStatus` 注解的异常，并根据注解中的值将其映射到HTTP状态码。

* `ExceptionHandlerExceptionResolver`：通过调用 `@Controller` 或 `@ControllerAdvice` 类中的 `@ExceptionHandler` 方法来解析异常。

* `SimpleMappingExceptionResolver`：异常类名称和错误视图名称之间的映射。对于在浏览器应用程序中渲染错误页面非常有用

* `DefaultHandlerExceptionResolver`：解析由Spring MVC引发的异常，并将其映射到HTTP状态码。例如：
  * `NoHandlerFoundException `-->  404 (SC_NOT_FOUND)
  * `ServletRequestBindingException `--> 400 (SC_BAD_REQUEST)


### 调用顺序

可以配置多个`HandlerExceptionResolver`,形成一个异常解析器链,Spring根据它们的 `order` 属性指定优先级，`order`值越低，优先级越高，越早被调用。

按照框架约定，`HandlerExceptionResolver`处理异常后可以返回：

- 一个指向错误视图的 `ModelAndView`。
- 如果异常在解析器中被处理，则是一个空（empty）的 `ModelAndView`。
- 如果异常仍未被解决，则为 `null`，供后续的解析器尝试，如果所有解析器都返回null，则允许异常冒泡到Servlet容器中。

SpringMVC的配置自动为默认的Spring MVC异常、`@ResponseStatus` 注解的异常以及 `@ExceptionHandler` 方法的支持声明了内置解析器

### 未处理异常

上一节中我们知道如果一个异常没能被任何一个解析器处理，那么这个异常会抛给Servlet容器，Servlet容器可以在HTML中渲染一个默认的错误页面。可以在 `web.xml` 中声明一个错误页面映射。以定制错误页面：

~~~xml
<error-page>
    <location>/error</location>
</error-page>
~~~

 **注意**：Servlet API并没有提供在Java中创建错误页面映射的方法。可以同时使用 `WebApplicationInitializer` 和一个最小的 `web.xml`

我们可以使用 `DispatcherServlet` 来处理这个最终的错误页面，比如将`/error`映射到一个`@Controller`上：

~~~java
@RestController
public class ErrorController {

    @RequestMapping(path = "/error")
    public Map<String, Object> handle(HttpServletRequest request) {
        Map<String, Object> map = new HashMap<>();
        map.put("status", request.getAttribute("jakarta.servlet.error.status_code"));
        map.put("reason", request.getAttribute("jakarta.servlet.error.message"));
        return map;
    }
}
~~~

## ViewResolver

`ViewResolver` 和 `View` ，用来在浏览器中渲染模型，而不需要绑定到特定的视图技术。`ViewResolver` 提供了视图名称viewName和实际视图之间的映射。`View` 解决了在移交给特定视图技术之前的数据准备问题。

`ViewResolver` 的继承体系如下:

![ViewResolver](https://gitee.com/wangziming707/note-pic/raw/master/img/ViewResolver.png)

大部分的`ViewResolver`实现，都会直接或者间接实现`AbstractCachingViewResolver`抽象类

因为正对每次请求都重新实例化View将可能为Web应用程序带来性能上的损失，所以`AbstractCachingViewResolver`实现了View实例的缓存功能，而且默认情况下时启用该功能的。在生产环境下，这是个合理的默认值，不过如果在测试或者开发环境下，我们想要立刻反映修改的结果，可以通过将 `cache` 属性设置为 `false` 来关闭缓存功能。

### 面向单一视图类型

面向单一视图类型的ViewResolver类都会直接或间接的继承`UrlBasedViewResolver`

使用该类型的ViewResolver，不需要配置具体的逻辑视图名到具体View的映射关系。通常只要指定视图模板所在的位置，这些ViewResolver会按照逻辑视图名，找到相应的模板文件、构造相应的View实例并返回。

`UrlBasedViewResolver`除了提供基本的前后缀映射的支持，还提供了解析转发和重定向URL的支持:

在正式解析viewName前，首先会判断viewName的字符串前缀，如果前缀为:

* `forward:`指示`ViewResolver`进行URL的转发
* `redirect:`指示`ViewResolver`进行URL的重定向

之所以面向叫单一视图类型是因为该类别中，每个具体的`ViewResolver`实现都只负责一种View类型的映射。它的主要实现类如下:

* `InternalResourceViewResolver`对应`InternalResourceView`的映射，也就是处理JSP模板类型的视图映射DispatcherServlet在初始化时，如果没有其他的ViewResolver，将默认使用该类
* `FreeMarkerViewResolver`：对应`FreeMarkerView`的映射
* `XsltViewResolver`：对应`XsltView`的映射

等等

启用以上的ViewResolver，和使用`InternalResourceViewResolver`一样样，使用`prefix`和`suffix`属性指定前后缀即可

### 面向多视图类型

面向多视图类型的ViewResolver，我们需要通过某种配置方式指定逻辑视图名和具体视图之间的映射关系。这样就可以实现多种视图类型的映射管理。

它有如下的实现:

* `ResourceBundleViewResolver`

* `XmlViewResolver`
* `BeanNameViewResolver`

### ViewResolver优先级

和`HandlerMapping`一样；我们可以为`DispatcherServlet`提供多个`ViewResolver`，`ViewResolver`的实现都实现了`Ordered`接口

解析视图名称时，`DispatcherServlet`会根据可用的`ViewResolver`实例的优先级进行遍历。先调用优先级高的`ViewResolver`，直到某个`ViewResolver`返回当前的`View`为止。

## LocaleResolver

SpringMVC支持国际化， `DispatcherServlet` 委托`LocaleResolver`解析请求，根据策略获取当前请求对应的`Locale `,这个`Locale`会在后续渲染`View`时发挥作用完成国际化渲染。

SpringMVC使用`LocaleResolver`接口对可能的`Locale`值的获取解析方式进行统一的策略抽象，定义如下:

~~~java
public interface LocaleResolver {
	Locale resolveLocale(HttpServletRequest request);
    //根据当前Locale解析策略获取当前请求对应的Locale值        
	void setLocale(HttpServletRequest request, HttpServletResponse response, Locale locale);
    //如果当前策略支持Locale的更改，可以通过该方法对当前策略默认取得的Locale值进行更改
}
~~~



### 可用的LocaleResolver实现

根据Locale获取策略，SpringMVC为LocaleResolver提供了相应的可用实现类:

LocaleResolver的继承体系如下:

![LocaleResolver](https://gitee.com/wangziming707/note-pic/raw/master/img/LocaleResolver.png)

* `AbstractLocaleResolver`:`LocaleResolver`的抽象类，为`LocaleResolver`实现提供一个设置defaultLocale的功能

* `LocaleContextResolver`:`LocaleResolver`的拓展接口，提供了一个`LocaleContext`(富Locale上下文，可能包含了Locale和时区信息)支持

  通常该`LocaleContext`会绑定的当前线程供其他组件使用，比如`LocaleChangeInterceptor`

* `AceptHeaderLocaleResolver`：根据HTTP的`Accept-Language`请求首部来分析并返回当前请求对应的Locale值，如果没有获取到Locale值，或者获取的值不在`supportedLocales`中(如果supportedLocales不为空)，则使用默认的defaultLocale

* `FixedLocaleResolver`:对于所有的请求，总是返回固定的Locale，该Locale是当前JVM默认的locale

* `SessionLocaleResolver`:根据指定键值从Session中获取相应的Locale，这需要提前在Session中设置该值

* `CookieLocaleResolver`:根据指定键值从Cookie中获取相应的Locale，这需要提前在Cookies中设置该值

### LocaleChangeInterceptor

在一些场景下，有提供给用户自己选择语言的需求，这需要使用`LocaleResolver`解析request的Locale前可以改变request的标志Locale的键；

显然要实现这种改变，只能使用`CookieLocaleResolver`和`SessionLocaleResolver`才能实现，因为JVM默认的Locale和HTTP的`Accept-Language`都是固定的无法改变的。只有cookie，session中存储的locale键值才能随意改变。

这就需要在请求处理前能够提前处理，locale的键值。`LocaleChangeInterceptor`就是这样的拦截器:

* 首先解析requset请求域，根据给定的参数名称获取域中的locale(默认为locale)
* 获取`DispatcherServlet`再收到请求时添加到request域中的LocaleResolver实例
* 调用`LocaleResolver`的`setLocale()`方法将之前解析出的locale设置到session或cookie中(根据`LocaleResolver`的不同)覆盖原本的locale

这样当处理完请求调用`LocaleResolver`来获取locale时，将获取到`LocaleChangeInterceptor`设置的locale而不是原本的

例如如果使用下面声明:

~~~xml
<bean id="localeChangeInterceptor"
        class="org.springframework.web.servlet.i18n.LocaleChangeInterceptor">
    <property name="paramName" value="lang"/>
</bean>

<bean id="localeResolver"
        class="org.springframework.web.servlet.i18n.CookieLocaleResolver"/>

<bean id="urlMapping"
        class="org.springframework.web.servlet.handler.SimpleUrlHandlerMapping">
    <property name="interceptors">
        <list>
            <ref bean="localeChangeInterceptor"/>
        </list>
    </property>
    <property name="mappings">
        <value>/**/*.view=someController</value>
    </property>
</bean>

~~~

那么客户端URL请求`https://www.sf.net/home.view?lang=zh_CN`就可以将网站改为中午

## MultipartResolver

HTTP协议的实体类型Context-Type有值`multipart/form-data`格式，支持表单的文件上传；在前端页面HTML页面或者js脚本中，设置表单的属性enctype为`multipart/form-data`以对请求实体内容进行编码，以发起multipart请求。

SpringMVC委托`MultipartResolver`解析multipart请求。

解析multipart请求有通用的类库,如:Oreilly、Commons FileUpload类库等。`MultipartResolver`的具体实现就是基于这些类库，提供了两个可用的实现:

* `CommonsMultipartResolver`：基于 Apache Commons FileUpload类库实现，使用它需要引入相应依赖(已过时，Spring Framework 6.0有新的Servlet 5.0+基线)。
* `StandardServletMultipartResolver`:基于Servlet 3.0 Part API的标准MultipartResolver实现。

想要启用 multipart 处理，需要在 `DispatcherServlet` 的Spring配置中声明一个`MultipartResolver`Bean实例，名称必须为 `multipartResolver`。

`DispatcherServlet` 会检测到它并将其应用于传入的请求。当收到一个内容类型为 `multipart/form-data` 的POST时，它会调用`MultipartResolver`的核心方法`resolveMultipart()`,将`HttpServletRequest`包裹为`MultipartHttpServletRequest`实例。

`MultipartHttpServletRequest`继承了`MultipartRequest`；提供对解析文件的访问:

~~~java
public interface MultipartRequest {
	Iterator<String> getFileNames();
	MultipartFile getFile(String name);
	List<MultipartFile> getFiles(String name);
	Map<String, MultipartFile> getFileMap();
	MultiValueMap<String, MultipartFile> getMultiFileMap();
	String getMultipartContentType(String paramOrFileName);
}
~~~

### Servlet配置

multipart 解析支持需要Servlet容器配置来启用：

- 对Java配置，需要在Servlet注册上设置一个 `MultipartConfigElement`。
- 对 `web.xml` 配置，需要在servlet声明中添加一个 `"<multipart-config>"` 部分

一个Java配置的示例：

~~~java
public class AppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {
    // ...
    @Override
    protected void customizeRegistration(ServletRegistration.Dynamic registration) {

        // Optionally also set maxFileSize, maxRequestSize, fileSizeThreshold
        registration.setMultipartConfig(new MultipartConfigElement("/tmp"));
    }

}
~~~

一个`web.xml`配置示例：

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_4_0.xsd"
         version="4.0">

    <!-- 配置SpringMVC核心控制器：DispatcherServlet主要负责流程的控制。-->
    <servlet>
        <servlet-name>SpringMVC</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>classpath:spring-*.xml</param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
        <multipart-config>
            <!--上传文件最大多少-->
            <max-file-size>20848820</max-file-size>
            <!--最大请求大小-->
            <max-request-size>418018841</max-request-size>
            <!--多大以上的文件可以上传-->
            <file-size-threshold>1048576</file-size-threshold>
        </multipart-config>
    </servlet>
    <servlet-mapping>
        <servlet-name>SpringMVC</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>
</web-app>
~~~

### 文件上传实例

现在我们来实现一个完整的文件上传流程:

#### 相关依赖

如果使用的是`CommonsMultipartResolver`需要引入Apache Commons FileUpload类库

~~~xml
<dependency>
    <groupId>commons-fileupload</groupId>
    <artifactId>commons-fileupload</artifactId>
    <version>1.4</version>
</dependency>
~~~

#### 前端页面

post表单需要设置`enctype`值为`multipart/form-data`

~~~html
<form method="post" enctype="multipart/form-data" action="${pageContext.request.contextPath}/fileUpload">
    选择上传的文件:<input name="picture" type="file"/><br/>
    <input name="submit" type="submit" value="提交">
</form>
~~~

#### Controller

接受文件上传的请求，将`HttpServletRequest`转化为 `MultipartHttpServletRequest`，再接受文件

~~~java
public class FileUploadController extends AbstractController {
    @Override
    protected ModelAndView handleRequestInternal(HttpServletRequest request, HttpServletResponse response) throws Exception {
        System.out.println("接受到文件上传请求");
        MultipartHttpServletRequest msr = (MultipartHttpServletRequest) request;
        File file = new File("D:\\Data\\Temporary\\picture.png");
        msr.getFile("picture").transferTo(file);
        System.out.println("上传文件成功");
        return null;
    }
}
~~~

#### 注册组件

~~~xml
<!--注册文件上传Controller-->
<bean name="/fileUpload" class="com.wzm.spring.controller.FileUploadController"/>
<!--注册MultipartResolver实例1-->
<bean id="multipartResolver" class="org.springframework.web.multipart.commons.CommonsMultipartResolver">
    <property name="maxUploadSize" value="#{1024*1024*80}"/>
    <property name="defaultEncoding" value="utf-8"/>
</bean>
<!--注册MultipartResolver实例2-->
<bean id="multipartResolver" class="org.springframework.web.multipart.support.StandardServletMultipartResolver">
</bean>
~~~

# CharacterEncodingFilter

SpringMVC内置了了编码过滤器，可以解决乱码问题，直接在web.xml中配置即可：

~~~xml
<!--解决post中文乱码问题-->
<filter>
    <filter-name>CharacterEncodingFilter</filter-name>
    <filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
    <!--指定编码格式-->
    <init-param>
        <param-name>encoding</param-name>
        <param-value>utf-8</param-value>
    </init-param>
</filter>
<filter-mapping>
    <filter-name>CharacterEncodingFilter</filter-name>
    <url-pattern>/*</url-pattern>
</filter-mapping>
~~~

或者通过Java方式：

~~~java
public class MyWebAppInitializer extends AbstractDispatcherServletInitializer {

    // ...

    @Override
    protected Filter[] getServletFilters() {
        return new Filter[] {
            new HiddenHttpMethodFilter(), new CharacterEncodingFilter() };
    }
}
~~~

