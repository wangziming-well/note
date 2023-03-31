# SpringMVC概述

## MVC

 MVC是一种设计模式，常用于开发web应用程序的框架中。MVC的意思是Model-View-Controller（模型-视图-控制器）。它将web应用程序分为三个主要组件：模型、视图和控制器。每个组件都负责不同的功能，但又相互关联，以便应用程序可以正确地工作。

* Model（模型） 模型是应用程序中的数据和逻辑组件，它处理应用程序数据和业务逻辑。模型是独立于用户界面和控制器的，因此可以在不影响应用程序其他部分的情况下修改数据和逻辑。
* View（视图） 视图是应用程序中的用户界面组件，它负责呈现模型中的数据。视图通常是HTML模板，它显示给用户的是模型中的数据。视图不处理数据或逻辑，它只负责呈现它们。
* Controller（控制器） 控制器是应用程序中的中介组件，负责接收用户请求，根据请求通知模型更新数据，并选择合适的视图响应给用户

![mvc设计模式](https://gitee.com/wangziming707/note-pic/raw/master/img/mvc%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F.png)

MVC设计模式是Web开发中的一种重要的设计模式，它能够提供良好的结构化方式，使得Web应用程序的不同部分更易于维护和扩展，提高了应用程序的质量和可靠性。

## SpringMVC组件

SpringMVC是请求驱动的MVC模式的Web框架，使用单一控制器处理web请求，它有以下主要组件：

* DispatcherServlet负责接收并处理所有的Web请求，争对具体的处理逻辑，它会委派给下一级控制器实现，即Controller

* HandlerMapping 负责管理Web请求到具体的处理类之间的映射关系。当请求到达DispatcherServlet后，DispatcherServlet将会寻求具体的HandlerMapping 实例，获取对应当前请求发具体处理类，即Controller
* Controller：Web请求的具体请求者，是DispatcherServlet的次级控制器
* ModelAndView：当Controller的处理方法完成后，将返回它的实例，它包含如下两部分信息：
  * 视图的逻辑名称(或者具体的视图实例)。DispatcherServlet将根据该视图的逻辑名称来决定为用户显示哪个视图。
  * 模型数据。视图渲染过程中需要将这些模型数据并入视图的显示中

#  搭建SpringMVC应用程序

SpringMVC也是基于servlet的架构，所以SpringMVC的项目结构和servlet一样，只是多出了springioc的配置文件

## 依赖

首先需要引入SpringMVC项目所需的依赖：

~~~xml
<dependency>
  <groupId>org.springframework</groupId>
  <artifactId>spring-context</artifactId>
  <version>5.2.5.RELEASE</version>
</dependency>
<dependency>
  <groupId>org.springframework</groupId>
  <artifactId>spring-webmvc</artifactId>
  <version>5.2.5.RELEASE</version>
</dependency>
<dependency>
  <groupId>javax.servlet</groupId>
  <artifactId>javax.servlet-api</artifactId>
  <version>4.0.1</version>
</dependency>
~~~

## 配置web.xml

需要配置web.xml文件，将springMVC的需要的组件在servlet容器初始化时，加载到ServletContext中以供使用

### 注册WebApplicationContext

WebApplicationContext继承ApplicationContext，在Spring IoC的基础上扩展提供了web应用的支持:

* 提供web相关的bean的scope: session和request
* 提供获取当前应用程序的ServletContext的方法

我们可以使用ContextLoaderListener在程序启动时，将WebApplicationContext加载到ServletContext中

ContextLoaderListener继承了ContextLoaderListener，并重写了contextInitialized()方法:

~~~java
public void contextInitialized(ServletContextEvent event) {
    initWebApplicationContext(event.getServletContext());
}
~~~

在servlet容器启动时，将调用`initWebApplicationContext()`，初始化`WebApplicationContext`，该方法主要做了以下事情:

* 创建`WebApplicationContext`实例
* 将当前`ServletContext`绑定到`WebApplicationContext`实例中
* 从`ServletContext`获取初始化参数 :`contextConfigLocation `的值并调用`setConfigLocation()`方法将其设置到`WebApplicationContext`中
  * 后续将调用`getConfigLocation()`方法获取`configLocation`，获取配置文件位置，将配置文件的定义加载成`Bean Definition`
  * 如果`configLocations`为空，将使用默认的`configLocation:/WEB-INF/applicationContext.xml`
* 调用`ServletContext`的`setAttribute()`方法，将`WebApplicationContext`绑定到`ServletContext`中，键为：`WebApplicationContext.class.getName() + ".ROOT"`

从上面过程可以看出，如果我们的springweb容器的ioc配置文件不在默认位置`/WEB-INF/applicationContext.xml`,则需要通过`web.xml`的标签`<context-param>`将contextConfigLocation加载到servletContext中，值应该为springweb容器配置文件的实际位置

综上，要在程序启动时加载`WebApplicationContext`,需要在web.xml文件中配置:

~~~xml
<context-param>
	<param-name>contextConfigLocation</param-name>
    <!--可以配置多个文件，可用以下分隔符分开：",; \t\n"-->
	<param-value>classpath:springmvc.xml</param-value>
</context-param>
<listener>
	<listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
~~~

可以通过`WebApplicationContextUtils`类中的方法获取当前`ServletContext`中绑定的`WebApplicationContext`,

就不需要知道`WebApplicationContext`在`ServletContext`中的key了:

~~~java
public static WebApplicationContext getWebApplicationContext(ServletContext sc)
~~~

### 注册DispatcherServlet

DispatcherServlet时SpringMVC框架Web应用程序的前端控制器，负责几乎所有对应当前Web应用程序的Web请求的处理。

DispatcherServlet使用了外部化的配置文件，用来配置Spring MVC框架在处理Web请求过程中所涉及的各个组件，包括:HandlerMapping的定义、Controller定义、ViewResolver定义等。该配置文件和普通的SpringIoc配置文件一样。

DispatcherServlet在启动后将加载该配置文件，并构建相应的WebApplicationContext，该WebApplicationContext将之前通过ContextLoaderListener加载的顶层WebApplicationContext(ROOT WebApplicationContext)作为父容器。

默认情况下，该配置文件的路径为`/WEB-INF/<servlet-name>-servlet.xml`

其中`<servlet-name>`为DispatcherServlet在servlet容器中的servlet名称，即web.xml中定义DispatcherServlet的`<servlet-name>`的值

当然，我们也可以通过在初始化DispatcherServlet时传入参数`contenxtConfigLocation`来自定义DispatcherServlet对应的配置文件的路径

~~~xml
<servlet>
	<servlet-name>dispatcherServlet</servlet-name>
	<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
	<init-param>
  		<param-name>contextConfigLocation</param-name>
  		<param-value>classpath:dispatcher-servlet.xml</param-value>
	</init-param>
    <!--保证前端控制器第一时间启动 -->
    <load-on-startup>1</load-on-startup>
</servlet>
<servlet-mapping>
	<servlet-name>dispatcherServlet</servlet-name>
    <!--过滤请求uri
      第一种：/  代表过滤所有
      第二种:*.action   *.do   *.jsp    配置具体的后缀
    -->
	<url-pattern>/</url-pattern>
</servlet-mapping>
~~~

## 配置WebApplicationContext配置文件

需要在DispatcherServlet的对应的WebApplicationContext的配置文件中，配置好SpringMVC框架需要的组件，如HandlerMapping、Controller、ViewResolver等。

### 注册HandlerMapping

DispatcherServlet在接收到Web请求后，将寻找相应的HandlerMapping进行Web请求到具体的Controller实现的匹配。

所以需要为DispatcherServlet提供一个具体的HandlerMapping的实现。

SpringMVC框架默认提供了多个HandlerMapping的实现。我们现在先注册一个BeanNameUrlHandlerMapping到WebApplicationContext中

~~~xml
<bean id="handlerMapping" class="org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping"/>
~~~

实际上，如果没有配置任何HandlerMapping，SpringMVC也会默认使用BeanNameUrlHandlerMapping。

BeanNameUrlHandlerMapping会将url的路径映射到对应的beanName 为请求路径的Controller上。

### 注册Controller

在注册Controller之前，我们需要先简单实现一个Controller类:

~~~java
public class DemoController extends AbstractController {
    @Override
    protected ModelAndView handleRequestInternal(HttpServletRequest request, HttpServletResponse response) throws Exception {
        System.out.println("收到请求");
        return null;
    }
}
~~~

继承AbstractController并覆盖其handleRequestInternal方法。

然后注册到对应的ioc容器中:

~~~java
<bean name="/demo" class="com.wzm.spring.controller.DemoController"/>
~~~

假设该web的前缀为:`http://localhost:8080/spring/`

那么这个controller对应的路径就为`http://localhost:8080/spring/demo`

### 注册ViewResover

DispatcherServlet需要ViewResover通过ModelAndView中的逻辑视图名查找相应的视图实现。

虽然现在不需要用到，但至少要注册一个，否则DispatcherServlet会报错

~~~xml
<bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
    <property name="prefix" value="/WEB-INF/jsp/"/>
    <property name="suffix" value=".jsp"/>
</bean>
~~~

定义了视图的前后缀；

这样假设视图的逻辑名称为`demo`，那么视图解析器将查找路径`/WEB-INF/jsp/demo.jsp`对应的视图

# SpringMVC核心组件

SpringMVC提供了许多组件以支持SpringMVC的Web应用程序运行和避免重复开发。

## HandlerMapping

HandlerMapping帮助DispatcherServlet进行Web请求的URL到具体处理类之间的匹配。

其定义如下：

~~~java
public interface HandlerMapping {
	@Nullable
	HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception;

}
~~~

定义非常简单，`getHandler()`方法根据HttpServletRequest 获取 HandlerExecutionChain 以获取处理器

SpringMVC提供了许多HandlerMapping的实现:

*  BeanNameUrlHandlerMapping

* SimpleUrlHandlerMapping
* ControllerClassNameHandlerMapping
* RequestMappingHandlerMapping

### BeanNameUrlHandlerMapping

映射关系：Web的请求路径对应的是Controller在容器中的beanName

它会直接将`http://localhost:8080/spring/demo`映射到`DemoController`上

### SimpleUrlHandlerMapping

通过map管理请求url和handler之间的定义，在创建时，传入Map或者Properties

示例:

~~~xml
<bean id="handlerMapping" class="...SimpleUrlHandlerMapping">
    <property name="mappings">
        <!--prop的key为路径，value为controller的beanId-->
        <props>
            <prop key="demo">demoController</prop>
        </props>
    </property>
</bean>
<bean id="demoController" class="..DemoController"/>
~~~

或者：

~~~xml
<bean id="handlerMapping" class="...SimpleUrlHandlerMapping">
    <property name="urlMap">
        <map>
            <entry key="demo" value-ref="demoController"/>
        </map>
    </property>
</bean>
<bean id="demoController" class="...DemoController"/>
~~~

两种配置都将`http://localhost:8080/spring/demo`映射到`DemoController`上

我们也可以通过表达式，将一组或者多组相似特征的Web请求处理映射给相同Handler处理:

~~~xml
<bean id="handlerMapping" class="...SimpleUrlHandlerMapping">
    <property name="mappings">
        <!--prop的key为路径，value为controller的beanId-->
        <props>
            <!--**匹配多重路径，*匹配任意字符-->
            <prop key="/**/*Demo">demoController</prop>
        </props>
    </property>
</bean>
<bean id="demoController" class="..DemoController"/>
~~~

### HandlerMapping执行序列

在基于SpringMVC的Web应用程序中，我们可以为DispatcherServlet提供多个HandlerMapping供其使用。

收到Web请求时，DispatcherServlet会根据可用的HandlerMapping实例的优先级进行遍历。先调用优先级高的HandlerMapping，直到某个HandlerMapping返回当前的Handler为止。

HandlerMapping的优先级由Spring框架内Ordered接口定义。在定义HandlerMapping实例时，我们可以通过设置其order属性来指定其优先级

order值越低优先级越高。

## Controller

Controller是SpringMVC框架支持的用于处理具体Web情趣的handler类型之一。

在使用SpringMVC进行Web开发时，它时我们接触使用最多的组件，用于实现具体的请求处理逻辑。

它的定义如下：

~~~java
public interface Controller {
	@Nullable
	ModelAndView handleRequest(HttpServletRequest request, HttpServletResponse response) throws Exception;
}
~~~

handleRequest()方法和servlet的service功能相同，都是进行请求的具体处理,该方法将被DispatcherServlet调用

我们可以直接实现Controller接口来处理请求，但这需要我们自己完成更多的细节，比如请求参数的抽取、请求编码的设定、国际化信息的处理等等，实际上这些关注点有很多时所有Controller都需要的

SpringMVC提供了一套Controller实现体系，以复用这些通用的逻辑:

![Controller](https://gitee.com/wangziming707/note-pic/raw/master/img/Controller.png)

### AbstractController

AbstractController是简单的Controller的实现抽象类，使用了模板方法的设计模式，继承它时需要重写指定的方法，它的HandlerRequest()方法如下:

~~~java
@Override
@Nullable
public ModelAndView handleRequest(HttpServletRequest request, HttpServletResponse response)
        throws Exception {
	//如果请求方式时OPTIONS，则通过getAllowHeader()响应当前controller支持的请求方法
    //默认只支持GET、HEAD、POST方法
    if (HttpMethod.OPTIONS.matches(request.getMethod())) {
        response.setHeader("Allow", getAllowHeader());
        return null;
    }
    //检查请求，委派给WebContentGenerator来做，默认检查请求方式是否是支持的，和如果session是必须的，则检查session是否存在
    checkRequest(request);
    //准备响应，委派给WebContentGenerator来做，默认将缓存相关的首部字段添加到Response首部
    prepareResponse(response);

    // 如果需要，同步执行handleRequestInternal方法
    if (this.synchronizeOnSession) {
        HttpSession session = request.getSession(false);
        if (session != null) {
            Object mutex = WebUtils.getSessionMutex(session);
            synchronized (mutex) {
                return handleRequestInternal(request, response);
            }
        }
    }
	// 否则直接执行handleRequestInternal方法
    return handleRequestInternal(request, response);
}
~~~

我们必须重写handleRequestInternal()方法，以处理具体的响应逻辑

通过AbstractController.handleRequest()方法，我们可以了解如何自定义一些配置:

* 自定义controller支持的方法:重写getAllowHeader()
* 自定义请求检查：重写checkRequest()
* 自定义准备响应：重写prepareResponse()
* 自定义是否同步session:响应前调用setSynchronizeOnSession()方法

### ServletWrappingController

一个servlet的包装控制器，将当前应用中的某个已存在的Servlet直接包装为一个Controller

所有到ServletWrappingController的请求实际上是由它内部所包装的这个Servlet 实例来处理的，也就是说内部封装的Servlet实例并不对外开放。

这个Servlet实例不是由servlet容器创建，而是ServletWrappingController自己在内部创建

实际上对该controller的请求由内部封装的Servlet实例进行处理。它通常用于对已存的Servlet的逻辑重用上。

示例：

~~~xml
<bean id="strutsWrappingController" class="org.springframework.web.servlet.mvc.ServletWrappingController">
	<property name="servletClass">
  		<value>org.apache.struts.action.ActionServlet</value>
	</property>
	<property name="servletName">
  		<value>action</value>
	</property>
	<property name="initParameters">
 		<props>
    		<prop key="config">/WEB-INF/struts-config.xml</prop>
  		</props>
	</property>
</bean>
~~~

初始化时需要指定servlet的class和name

### ServletForwardingController

一个servlet的转发控制器，将请求转发给指定的servlet

与ServletWrappingController不同的是，该控制器不会创建相应的实例，如果servlet容器中没有相应的servlet实例，会通知servlet容器让其创建。

web.xml中定义:

~~~xml
<servlet>
	<servlet-name>myServlet</servlet-name>
	<servlet-class>mypackage.TestServlet</servlet-class>
</servlet>
~~~

ioc配置文件中定义:

~~~xml
<bean id="myServletForwardingController" class="org.springframework.web.servlet.mvc.ServletForwardingController">
	<property name="servletName"><value>myServlet</value></property>
</bean>
~~~

### ParameterizableViewController

返回预配置视图并可选地设置响应状态代码的controller。视图和状态可以使用提供的配置属性进行配置。

它的handleRequestInternal方法如下：

~~~java
@Override
protected ModelAndView handleRequestInternal(HttpServletRequest request, HttpServletResponse response)
        throws Exception {

    String viewName = getViewName();
	//如果statusCode不为空，则设置相应状态码
    if (getStatusCode() != null) {
        if (getStatusCode().is3xxRedirection()) {
            request.setAttribute(View.RESPONSE_STATUS_ATTRIBUTE, getStatusCode());
        }
        else {
            response.setStatus(getStatusCode().value());
            if (getStatusCode().equals(HttpStatus.NO_CONTENT) && viewName == null) {
                return null;
            }
        }
    }
	//如果是只响应状态的，直接返回，不返回视图
    if (isStatusOnly()) {
        return null;
    }
	//根据viewName或者View()返回modelAndView
    ModelAndView modelAndView = new ModelAndView();
    modelAndView.addAllObjects(RequestContextUtils.getInputFlashMap(request));
    if (viewName != null) {
        modelAndView.setViewName(viewName);
    }
    else {
        modelAndView.setView(getView());
    }
    return modelAndView;
}
~~~

配置示例：

~~~xml
<bean id="demo" class="org.springframework.web.servlet.mvc.ParameterizableViewController">
    <property name="statusCode" value="OK"/>
    <property name="statusOnly" value="false"/>
    <property name="viewName" value="index"/>
</bean>
~~~

### UrlFilenameViewController

简单控制器实现，将URL的虚拟路径转换为视图名并返回该视图。

转换示例如下:

* "/index" -> "index"
* "/index.html" -> "index"
* "/index.html" + prefix "pre_" and suffix "_suf" -> "pre_index_suf"
* "/products/view.html" -> "products/view"

在设置时，可以配置前后缀:

~~~xml
<bean name="/d" class="org.springframework.web.servlet.mvc.UrlFilenameViewController">
    <property name="prefix" value="in"/>
    <property name="suffix" value="ex"/>
</bean>
~~~

该配置将返回viewName :index

## ModelAndView

Controller在将Web请求处理完成后，会返回一个ModelAndView实例。ModelAndView包含两部分内容：

* 视图内容:可能是视图的逻辑名称，也可以是具体的视图实例
* 模型数据:一个map，视图渲染过程中将会把这些模型数据合并入最终的视图输出。

它内部维护下面字段:

~~~java
private Object view;
private ModelMap model;
private HttpStatus status;
~~~

我们可以在实例化ModelAndView时通过构造函数传入 视图和模型数据

也可以在实例化完成后，通过set方法设置：

~~~java
public ModelAndView(String viewName);
public ModelAndView(View view);
public ModelAndView(String viewName,Map<String, ?> model);
public ModelAndView(View view, Map<String, ?> model) ;
public ModelAndView(String viewName, HttpStatus status);
public ModelAndView(String viewName,Map<String, ?> model,  HttpStatus status);
public ModelAndView(String viewName, String modelName, Object modelObject);
public ModelAndView(View view, String modelName, Object modelObject);

public void setViewName(String viewName);
public void setView(View view);
public void setStatus(HttpStatus status);
public ModelAndView addAllObjects(Map<String, ?> modelMap);
~~~

DispatcherServlet获取到ModelAndView后:

* 先获取View实例:先从ModelAndView中获取viewName

  * 若viewName为空，则从ModelAndView直接获取View实例
  * 若viewName不为空，则委托ViewResolver通过viewName获取具体的View实例

* 再将Model渲染到View中:

  调用view的render方法，将模型数据渲染到view，不同的view实现渲染模型数据的方法不同

## ViewResolver

从上一部分对ModelAndView的介绍，我们已经可以知道ViewResolver的职责:

根据Controller返回的ModelAndView中的逻辑视图名viewName，为DispatcherServlet返回一个可用的View实例

它的定义如下:

~~~java
public interface ViewResolver {
	View resolveViewName(String viewName, Locale locale) throws Exception;
}
~~~

接口的实现类只需要根据viewName 和传入的Locale值来返回相应的视图

传入Locale的目的是在需要的情况下，根据Locale的不同返回不同的视图实例以支持国际化

其继承体系如下:

![ViewResolver](https://gitee.com/wangziming707/note-pic/raw/master/img/ViewResolver.png)

大部分的`ViewResolver`实现，都会直接或者间接实现`AbstractCachingViewResolver`抽象类

因为正对每次请求都重新实例化View将可能为Web应用程序带来性能上的损失，所以`AbstractCachingViewResolver`实现了View实例的缓存功能，而且默认情况下时启用该功能的。在生产环境下，这是个合理的默认值，不过如果在测试或者开发环境下，我们想要立刻反映修改的结果，可以通过`setCache(false)`暂时关闭它的缓存功能

### 面向单一视图类型

面向单一视图类型的ViewResolver类都会直接或间接的继承`UrlBasedViewResolver`

使用该类型的ViewResolver，不需要配置具体的逻辑视图名到具体View的映射关系。通常只要指定以下视图模板所在的位置，这些ViewResolver会按照逻辑视图名，找到相应的模板文件、构造相应的View实例并返回。

`UrlBasedViewResolver`除了提供基本的前后缀映射的支持，还提供了解析转发和重定向URL的支持:

在正式解析viewName前，首先会判断viewName的字符串前缀，如果前缀为:

* `forward:`指示`ViewResolver`进行URL的转发
* `redirect:`指示`ViewResolver`进行URL的重定向

之所以面向叫单一视图类型是因为该类别中，每个具体的ViewResolver实现都只负责一种View类型的映射。它的主要实现类如下:

* `InternalResourceViewResolver`对应`InternalResourceView`的映射，也就是处理JSP模板类型的视图映射DispatcherServlet在初始化时，如果没有其他的ViewResolver，将默认使用该类
* `FreeMarkerViewResolver`：对应`FreeMarkerView`的映射
* `XsltViewResolver`：对应`XsltView`的映射

等等

启用以上的ViewResolver，和使用`InternalResourceViewResolver`一样样，使用`prefix`和`suffix`属性指定前后缀即可

### 面向多视图类型

面向多视图类型的ViewResolver，我们需要通过某种配置方式指定逻辑视图名和具体视图之间的映射关系。这样就可以实现多种视图类型的映射管理。

它有如下的实现:

* ResourceBundleViewResolver

* XmlViewResolver
* BeanNameViewResolver

### ViewResolver优先级

和HandlerMapping一样；我们可以为DispatcherServlet提供多个ViewResolver，ViewResolver的实现都实现了Ordered接口

解析视图名称时，DispatcherServlet会根据可用的ViewResolver实例的优先级进行遍历。先调用优先级高的ViewResolver，直到某个ViewResolver返回当前的View为止。

## View

View是封装了视图渲染逻辑的组件，通过引入该策略抽象接口，我们可以极具灵活性地支持各种视图渲染技术

它的定义如下:
~~~java
public interface View {
	default String getContentType() {
		return null;
	}
	void render(Map<String, ?> model, HttpServletRequest request, HttpServletResponse response)
			throws Exception;
}
~~~

各种View实现类主要职责就是在redner()方法中实现最终的视图渲染工作

* 使用JSP技术的View实现:
  * InternalResourceView
  * JstlView
  * TilesView
  * TilesJstlView
* 使用通用模板技术的View实现:
  * FreeMarkerView
  * VelocityView
* 面向二进制文档格式的View实现:
  * Excel形式的视图:
    * AbstractExcelView
    * AbstractJExcelView
  * PDF形式的视图：
    * AbstractPDFlView

等等

# SpringMVC其他组件

除了之前介绍的核心组件外，SpringMVC还提供了更多的组件以支持Web开发

* `MultipartResolver`负责文件上传
* `HandlerAdaptor`使用不同类型的Handler
* `HandlerInterceptor`处理器拦截器
* `HandlerExceptionResolver`：提供请求时异常的标准处理方式
* `LocaleResolver`：提供更方便的显示国际化视图

## MultipartResolver

HTTP协议的实体类型Context-Type有值`multipart/form-data`格式，支持表单的文件上传；在前端页面HTML页面或者js脚本中，设置表单的属性enctype为`multipart/form-data`以对请求实体内容进行编码

针对这种编码类型的请求进行解析上传的文件，是通用的逻辑，有通用的类库,如:Oreilly、Commons FileUpload类库等。

在实现基于表单的文件上传功能时，SpringMVC框架底层实际上也是使用了以上的几种类库，只是通过MultipartResolver策略接口的抽象，将具体选用哪一种类库的权利给了用户。

MultipartResolver提供了两个可用的实现:

* CommonsMultipartResolver：基于 Apache Commons FileUpload类库实现，使用它需要引入相应依赖
* StandardServletMultipartResolver:基于Servlet 3.0 Part API的标准MultipartResolver实现

MultipartResolver接口的定义如下:

~~~java
public interface MultipartResolver {

	boolean isMultipart(HttpServletRequest request);

	MultipartHttpServletRequest resolveMultipart(HttpServletRequest request) throws MultipartException;

	void cleanupMultipart(MultipartHttpServletRequest request);

}

~~~

核心方法是`resolveMultipart()`,它将传入的HttpServletRequest实例转换为MultipartHttpServletRequest实例

MultipartHttpServletRequest继承MultipartRequest

提供了获取请求上传的文件相关的方法:

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

### DispatcherServlet使用MultipartResolver

#### 初始化MultipartResolver

`DispatcherServlet`作为一个servlet在启动时会被servlet容器调用`init()`方法进行初始化，此时会调用`DispatcherServlet`的`initMultipartResolver()`方法进行`MultipartResolver`的初始化:

从自己的`WebApplicationContext`中获取beanName固定为:`multipartResolver`的`MultipartResolver`实例

#### 使用MultipartResolver

然后在收到Web请求时，首先调用`checkMultipart()`方法进行Multipart校验:

如果`DispatcherServlet`持有的`multipartResolver`不为空且请求的Content-Type是以  `multipart/ `开头的:

则调用`multipartResolver`的`resolveMultipart`，将请求的`HttpServletRequest`实例转化为

`MultipartHttpServletRequest`

由此可以看出:

在`WebApplicationContext`注册`MultipartResolver`供`DispatcherServlet`使用时，beanName必须是`multipartResolver`

### 文件上传实例

现在我们来实现一个完整的文件上传流程:

#### 相关依赖

使用CommonsMultipartResolver需要引入Apache Commons FileUpload类库

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

接受文件上传的请求，将HttpServletRequest转化为 MultipartHttpServletRequest，再接受文件

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
<!--注册MultipartResolver实例-->
<bean id="multipartResolver" class="org.springframework.web.multipart.commons.CommonsMultipartResolver">
    <property name="maxUploadSize" value="#{1024*1024*80}"/>
    <property name="defaultEncoding" value="utf-8"/>
</bean>
~~~

## HandlerAdapter

之前我们说DispatcherServlet调用HandlerMapping后会返回一个Controller

但实现上我们看到HandlerMapping返回的是HandlerExecutionChain对象

是因为SpringMVC充当处理Web请求的Handler处理器对象不止是Controller一种类型。HandlerExecutionChain返回的是Object类型的Handler对象

直接在DispatcherServlet中使用if-else进行Handler对象类型的判断，然后调用不同Handler对象的处理逻辑方法显然是不合适的，大量的if-else既难以维护也不利于拓展

SpringMVC采用了适配器的设计模式，设计提供HandlerAdapter接口

DispatcherServlet将直接调用Handler获取ModelAndView的任务委托给了HandlerAdaptor，由相应的HandlerAdaptor实现来调用不同类型的Handler

这样DispatcherServlet就屏蔽了Handler对象的不同所带来的调用差异。

HandlerAdapter的定义如下:

~~~java
public interface HandlerAdapter {
    //判断当前适配器是否支持传入的handler
	boolean supports(Object handler);
    //执行handler的处理请求的方法，并返回ModelAndView
	ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception;
    //获取响应首部Last-Modified
	long getLastModified(HttpServletRequest request, Object handler);
}
~~~

它的核心方法就是`supports()`和`handle()`

### DispatcherServlet使用HandlerAdapter

#### 初始化HandlerAdapter

`DispatcherServlet`在初始化时，在Servlet的`init()`方法中会调用`initHandlerAdapters()`,默认检测加载所有可用的HandlerAdapter:

获取WebApplicationContext中注册的所有HandlerAdapter实现类实例

如果没有获取到容器中的HandlerAdapter实例，则会读取文件

`DispatcherServlet.properties`中配置的默认HandlerAdapter:

~~~properties
org.springframework.web.servlet.HandlerAdapter=org.springframework.web.servlet.mvc.HttpRequestHandlerAdapter,\
org.springframework.web.servlet.mvc.SimpleControllerHandlerAdapter,\
org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter,\
org.springframework.web.servlet.function.support.HandlerFunctionAdapter
~~~

实例化并加载到DispatcherServlet中。

#### 使用HandlerAdapter

DispatcherServlet调用合适的`HandlerMapping`获取到`HandlerExecutionChain`

调用`HandlerExecutionChain`的`getHandler()`获取具体的处理器

循环调用持有的所有HandlerAdapter实例的supports方法，如果HandlerAdapter实例支持当前handler类型，就使用该HandlerAdapter使用来调用handler来处理Web请求，返回ViewAndModel

### HandlerAdapter实现

SpringMVC提供了几个HandlerAdapter实现以适配不同的Handler处理器:

* SimpleControllerHandlerAdapter：适配Controller类型的Handler
* SimpleServletHandlerAdapter：适配Servlet类型的Handler
* HttpRequestHandlerAdapter：适配HttpRequestHandler类型的Handler
* HandlerFunctionAdapter：适配HandlerFunction类型的Handler
* RequestMappingHandlerAdapter：适配被@RequestMapping注释的HandlerMethods.以支持SpringMVC解析注解使用@Controller的Handler

## 自定义Handler

通过对上面SpringMVC组件:HandlerMapping和HandlerAdapter的学习，我们可以实现自定的Handler了。

自定义的Handler可以是任何样子，不需要继承任何接口，但你需要实现对应的HandlerMapping和HandlerAdapter以让DispatcherServlet可以使用自定义的Handler

### 定义Handler

首先需要自定义一个Handler接口，当然也可以使用注解的方式注释自定义的Handler，但这需要多一层转换的工作

~~~java
public interface MyHandler {
    void handleRequest(HttpServletRequest request,HttpServletResponse response);
}
~~~

再定义一个简单的实现类:

~~~java
public class SimpleMyHandler implements MyHandler{
    @Override
    public void handleRequest(HttpServletRequest request, HttpServletResponse response) {
        System.out.println("收到请求");
    }
}
~~~

### 定义HandlerAdapter

直接实现`HandlerAdapter`并重写`supports()`和`handle()`方法，以支持适配MyHandler类型的处理器

~~~java
public class MyHandlerAdapter implements HandlerAdapter {
    @Override
    public boolean supports(Object handler) {
        return handler instanceof MyHandler;
    }

    @Override
    public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
         ((MyHandler)handler).handleRequest(request,response);
         return null;
    }

    @Override
    public long getLastModified(HttpServletRequest request, Object handler) {
        return 0;
    }
}
~~~

### 注册组件

需要注册相关的组件以通知`DispatcherServlet`可以使用`MyHandler`类型的处理器:

~~~xml
<bean id="myHandler" class="com.wzm.spring.SimpleMyHandler"/>
<bean class="com.wzm.spring.MyHandlerAdapter"/>
<bean class="org.springframework.web.servlet.handler.SimpleUrlHandlerMapping">
    <property name="urlMap">
        <map>
            <entry key="testMyHandler" value-ref="myHandler"/>
        </map>
    </property>
</bean>
~~~

## HandlerInterceptor

前面已经提到，`HandlerMapping`返回的是`HandlerExecutionChain`实例，`DispatcherServlet`从`HandlerExecutionChain`中获取具体的`Handler`对象实例，实际上`HandlerExecutionChain`在除了保存`Handler`对象实例还保存了一组`HandlerInterceptor`

这组`HandlerInterceptor`可以在`Handler`的执行前后对处理流程进行拦截操作

`HandlerInterceptor`的定义如下

~~~java
public interface HandlerInterceptor {
	//在handler处理Web请求之前执行，如果返回false则终止流程
	default boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
			throws Exception {

		return true;
	}
	//在handler处理web请求之后，在视图的解析渲染之前执行
	default void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler,
			@Nullable ModelAndView modelAndView) throws Exception {
	}
	//整个处理流程结束之后，不管是正常结束还是异常终止，都将执行afterCompletion的方法
	default void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler,
			@Nullable Exception ex) throws Exception {
	}

}
~~~

### DispatcherServlet使用HandlerInterceptor

#### 注册HandlerInterceptor

`HandlerExecutionChain`中的`HandlerInterceptor`实例是来自`HandlerMapping`的，而所有的`HandlerMapping`实现都会继承`AbstractHandlerMapping`，它提供了以下方法:

~~~java
public void setInterceptors(Object... interceptors);
~~~

以供我们设置拦截器，所以我们可以在定义HandlerMapping是传入interceptors参数:

~~~xml
<bean class="...SimpleUrlHandlerMapping">
    <property name="interceptors">
        <list>
            <bean class="...WebContentInterceptor"/>
            ......
        </list>
    </property>
	......
</bean>
~~~

#### 使用HandlerInterceptor

`DispatcherServlet`并没有直接调用`HandlerInterceptor`的`preHandle()`等拦截方法

而是交委托HandlerExecutionChain来做,HandlerExecutionChain提供下面方法:

~~~java
boolean applyPreHandle(HttpServletRequest request, HttpServletResponse response);
//遍历调用持有的所有HandlerInterceptor的preHandle方法
void applyPostHandle(HttpServletRequest request, HttpServletResponse response, ModelAndView mv);
//遍历调用持有的所有HandlerInterceptor的postHandle方法
void triggerAfterCompletion(HttpServletRequest request, HttpServletResponse response, Exception ex);
//遍历调用持有的所有HandlerInterceptor的afterCompletion方法
~~~

`DispatcherServlet`通过调用`HandlerExecutionChain`的上面方法来间接调用`HandlerInterceptor`的拦截方法，`DispatcherServlet`：

* 在调用`HandlerAdapter.handle()`方法处理请求之前，会调用`applyPreHandle()`方法
* 在调用`HandlerAdapter.handle()`方法处理请求之后，在执行`processDispatchResult()`方法解析视图并渲染视图之前，会调用`applyPostHandle()`方法
* 在`doDispatch()`方法流程的最后的finally代码块中，会调用`triggerAfterCompletion()`方法

### 可用的HandlerInterceptor实现

SpringMVC提供了一些可用的HandlerInterceptor实现

* UserRoleAuthorizationInterceptor
* WebContentInterceptor
* LocaleChangeInterceptor
* ThemeChangeInterceptor

等等，我们先只介绍其中两个

#### UserRoleAuthorizationInterceptor

`UserRoleAuthorizationInterceptor`允许我们通过`HttpServletRequest`的``isUserInRole()``方法，使用一组指定的UserRoles对当前请求进行验证:

如果验证不通过，将默认返回HTTP的403状态码forbidden，可以通过覆写 `handleNotAuthorized()`方法改写这种默认行为

只需要在注册UserRoleAuthorizationInterceptor的时候指定authorizedRoles属性来指定允许的UserRoles

#### WebContentInterceptor

WebContentInterceptor主要做以下几件事情:

* 检查请求方法类型是否在支持方法之列
* 检查必要的Session实例
* 检查缓存时间并通过设置响应HTTP首部控制缓存行为

我们可以通过设置WebContentInterceptor的以下字段来控制上面的检查行为:

~~~java
private Set<String> supportedMethods;
//设置请求支持的Method
private boolean requireSession = false;
//设置请求是否必须有session，默认false
private int cacheSeconds = -1;
//设置缓存时间，默认不缓存
~~~

### HandlerInterceptor和Filter

Servlet组件中的Filter和HandlerInterceptor功能类似，都提供请求拦截功能，但它们之间也存在差别

DispatcherServlet也是一个Servlet

Filter对请求的拦截是在请求进入DispatcherServlet之前和DispatcherServlet处理完请求之后的，在SpringMVC的框架下，Filter是针对DispatcherServlet的执行进行拦截

而HandlerInterceptor对请求的拦截都是在DispatcherServlet处理流程内部的，针对Handler的执行进行拦截

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

然后设计了HandlerExceptionResolver组件，对抛出的异常进行统一处理。其定义如下：

~~~java
public interface HandlerExceptionResolver {
	ModelAndView resolveException(HttpServletRequest request, HttpServletResponse response, @Nullable Object handler, Exception ex);
}
~~~

处理异常，并返回相应的视图

### DispatcherServlet使用HandlerExceptionResolver

#### 初始化HandlerExceptionResolver

在DispatcherServlet初始化调用`init()`方法时，会调用`initHandlerExceptionResolvers()`初始化HandlerExceptionResolver：

默认检测获取DispatcherServlet的WebApplicationContext中的所有的HandlerExceptionResolver实例

如果容器中没有HandlerExceptionResolver实例，则读取DispatcherServlet.properties配置文件:

~~~properties
org.springframework.web.servlet.HandlerExceptionResolver=org.springframework.web.servlet.mvc.method.annotation.ExceptionHandlerExceptionResolver,\
org.springframework.web.servlet.mvc.annotation.ResponseStatusExceptionResolver,\
org.springframework.web.servlet.mvc.support.DefaultHandlerExceptionResolver
~~~

加载配置文件中指定的HandlerExceptionResolver实例到DispatcherServlet

#### 使用HandlerExceptionResolver

实际上HandlerExceptionResolver处理的异常不止是Handler处理Web请求抛出的，它负责处理的异常范围更大:

从MultipartResolver处理multipart请求开始到处理请求到调用HandlerInterceptor的后处理为止，抛出的异常都由它处理:

如果有抛出异常，将调用processHandlerException()方法进行异常处理:

遍历持有的HandlerExceptionResolver，调用其resolveException()方法，如果有返回ModelAndView实例，则退出遍历

### 可用的HandlerExceptionResolver实现

HandlerExceptionResolver的继承体系如下：

![HandlerExceptionResolver](https://gitee.com/wangziming707/note-pic/raw/master/img/HandlerExceptionResolver.png)

其中的一些我们不做探究:

* `ResponseStatusExceptionResolver`：用` @ResponseStatus`注释的方法表示的异常映射来处理异常

* `ExceptionHandlerExceptionResolver`：它通过`@ExceptionHandler`注释方法来处理异常。

* `HandlerExceptionResolverComposite`:复合HandlerExceptionResolver的实现，将异常处理委托给它持有的一组HandlerExceptionResolver

接下来详细介绍以下实现：

#### AbstractHandlerExceptionResolver

几乎所有的实现都会直接或间接继承`AbstractHandlerExceptionResolver`,它为`HandlerExceptionResolver`的其他实现提供了基础设施

`AbstractHandlerExceptionResolver`也实现了Spring框架下的Ordered接口，`DispatcherServlet`在使用`HandlerExceptionResolver`时，也会按照优先级进行遍历。

* `mappedHandlers&mappedHandlerClasses`：可以通过设置这两个值来让`HandlerExceptionResolver`的实现只捕获指定Handler抛出的异常，如果未指定该类，默认将处理所有的异常
* `preventResponseCaching`：指定是否阻止此异常解析器解析的任何视图的HTTP响应缓存。默认为false。为了自动生成抑制响应缓存的HTTP响应标头，可以将其切换为true。

#### DefaultHandlerExceptionResolver

`HandlerExceptionResolver`的默认实现，将SpringMVC异常转换为响应码,如：

* NoHandlerFoundException   -->  404 (SC_NOT_FOUND)
* ServletRequestBindingException --> 400 (SC_BAD_REQUEST)

等等.....

#### SimpleMappingExceptionResolver

将Exception名称映射到viewName,在注册该异常处理器时，可以设置以下属性，以控制它的映射行为：

* `exceptionMappings`:Properties类型的值，设置异常类和viewName之间的映射关系；SimpleMappingExceptionResolver会将当前的抛出的异常类型与exceptionMappings中相应映射进行匹配。采用的不是类型匹配，而是根据类名字符串进行局部匹配

  使用的是`String`的`indexOf()`进行匹配,这样匹配显然是不合理的。这会导致最终匹配的不是我们想要的映射。为了避免这种匹配方式的不准确性，我们在指定mappings的异常时，最好使用类的全限定名

* `defalutErrorView`：用于指定一个默认的错误信息页面对应的逻辑视图名。当抛出的异常无法在exceptionMappings中查到可用的视图名时，defalutErrorView指定的视图名将被返回

* `defalutStatusCode`:指定异常情况下默认返回给客户端的HTTP状态码

* `exceptionAttribute`:如果想要在错误页面对抛出的异常进行访问，可以设置exceptionAttribute的值，该值将作为前端获取异常的键

  该属性的默认值为exception，如果不想让前端访问到异常，可以将该值设置为null

## LocaleResolver

在`ViewResolver`根据逻辑视图名解析视图的时候，`ViewResolver`的`resolveViewName(viewName,locale)`方法除了接受要解析的逻辑视图名作为参数外，还同时接收一个Locale类型对象；这样ViewResolver就可以根据Locale的不同返回针对不同Locale的视图实例。`ResourceBundleViewResolver`实现就是如此。

但是有一个问题需要解决:Locale实例从何而来，怎样获取用户对应的Locale实例，这就是LocaleResolver的工作:

SpringMVC使用LocaleResolver接口对可能的Locale值的获取解析方式进行统一的策略抽象，定义如下:

~~~java
public interface LocaleResolver {
	Locale resolveLocale(HttpServletRequest request);
    //根据当前Locale解析策略获取当前请求对应的Locale值        
	void setLocale(HttpServletRequest request, HttpServletResponse response, Locale locale);
    //如果当前策略支持Locale的更改，可以通过该方法对当前策略默认取得的Locale值进行更改
}
~~~

### DispatcherServlet使用LocaleResolver

#### 初始化LocaleResolver

DispatcherServlet作为一个Servlet，在Servlet对其初始化，调用其`init()`方法时，会调用

`initLocaleResolver()`方法初始化加载LocaleResolver实例：

* 先从DispatcherServlet持有的WebApplicationContext容器中获取beanName为`localeResolver`的LocaleResolver

* 如果没有获取到对应实例，将加载默认实例：读取`DispatcherServlet.properties`文件配置:

  ~~~properties
  org.springframework.web.servlet.LocaleResolver=org.springframework.web.servlet.i18n.AcceptHeaderLocaleResolver
  ~~~

  加载配置文件中设置的默认LocaleResolver

加载完成LocaleResolver实例实例后，如果接收到Web请求，在处理请求之前，会先将持有的LocaleResolver实例设置到该请求的request域对象中，方便后续组件获取到LocaleResolver实例并使用(如LocaleChangeInterceptor)

**注意:**从上面初始化过程可以看出，如果要指定LocaleResolver，需要注册beanName为localeResolver的LocaleResolver，不能用其他的beanName

#### 使用LocaleResolver

在handler工作完成返回ModelAndView，ViewResolver解析viewName渲染View之前

Dispatcher会调用LocaleResolver的`resolveLocale()`方法获取请求对应的Locale

然后ViewResolver会接收viewName和Locale返回合适的View实例

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

在一些场景下，有提供给用户自己选择语言的需求，这需要使用LocaleResolver解析request的Locale前可以改变request的标志Locale的键；

显然要实现这种改变，只能使用`CookieLocaleResolver`和`SessionLocaleResolver`才能实现，因为JVM默认的Locale和HTTP的`Accept-Language`都是固定的无法改变的。只有cookie，session中存储的locale键值才能随意改变。

这就需要在请求处理前能够提前处理，locale的键值。这就是在介绍HandlerInterceptor时，提到的它的实现LocaleChangeInterceptor，它就是用来这样干的:

* 首先解析requset请求域，根据给定的参数名称获取域中的locale(默认为locale)
* 获取DispatcherServlet再收到请求时添加到request域中的LocaleResolver实例
* 调用LocaleResolver的setLocale方法将之前解析出的locale设置到session或cookie中(更具LocaleResolver的不同)覆盖原本的locale

这样当处理完请求调用LocaleResolver来获取locale时，将获取到LocaleChangeInterceptor设置的locale而不是原本的

示例:

~~~xml
<bean class="org.springframework.web.servlet.handler.SimpleUrlHandlerMapping">
    <property name="urlMap">
        <map>
            <entry key="testMyHandler" value-ref="myHandler"/>
        </map>
    </property>
    <property name="interceptors">
        <list>
            <bean class="org.springframework.web.servlet.i18n.LocaleChangeInterceptor">
                <property name="paramName" value="lang"/>
            </bean>
        </list>
    </property>
</bean>
~~~

这样设置后，可以在请求的时候加入类似参数`lang=zh_CN`来指定国际化语言



# 基于注解的Controller

在之前对SpringMVC组件的讨论中，Handler处理器可以是任何形式的，只要为自定义的Handler配置对应的HandlerMapping和HandlerAdapter，就能够将其集成到SpringMVC的工作流程中。

SpringMVC提供的基于注解的Controller也是这样，提供了相应的基于注解的HandlerMapping和HandlerAdapter实现，就能为用户提供相应的注解开发支持,在之前的讨论中，我们已经提到过这样的实现:

* RequestMappingHandlerMapping
* RequestMappingHandlerAdapter

基于以上组件，SpringMVC提供了注解`@Controller`、`@RequestMapping`，通过注释来声明基于注解的Controller,告知SpringMVC框架该类是注释的Handler:

~~~java
@Controller
public class AnnotatedController {
    @RequestMapping( "/accept")
    public String accept(){
        System.out.println("收到请求");
        return "index";
    }
}
~~~

并配置以下内容:

~~~xml
<mvc:annotation-driven/>
<context:component-scan base-package="com.wzm.spring"/>
~~~

启动mvc的注解驱动，并将刚刚注释的类交给ioc容器管理即可

这样，一个`@Controller`方法注释的类中的每个`@RequestMapping`注释的方法都对应一个handler，叫做处理器方法(Handler Method)

## 基本Handler Method

我们可以使用`@Controller`和`@RequestMapping`注解注释以声明一个基于注解的Controller

因为一个`@RequestMapping`注释的方法在SpringMVC就对应一个Controller，所以被`@RequestMapping`注释的方法也可以叫处理器方法或者处理方法(Handler Method)

### `@Controller`

想要让一个普通的pojo类成为SpringMVC框架下的handler处理器，必须使用`@Controller`注解注释该类

`@Controller`的定义如下:

~~~java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface Controller {

	@AliasFor(annotation = Component.class)
	String value() default "";

}

~~~

可以看到它被`@Component`注解注释，所有被注释了`@Controller`注解的类，同样可以被扫描管理到Spring的ioc容器中，而不需要额外使用其他注解

除此之外，`RequestMappingHandlerMapping`在收到Web请求时，匹配的就是所有容器中被`@Controller`注释的对象

### `@RequsetMapping`

只拥有一个`@Controller`注解，不能让`RequestMappingHandlerMapping`知道应该将Web请求映射到哪个Controller上，需要`@RequsetMapping`提供必要的映射信息。

`@RequsetMapping`可以应用到方法级别和类级别上，定义如下:

~~~java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Mapping
public @interface RequestMapping {
	String name() default "";
	@AliasFor("path")
	String[] value() default {};
	@AliasFor("value")
	String[] path() default {};
	RequestMethod[] method() default {};
	String[] params() default {};
	String[] headers() default {};
	String[] consumes() default {};
	String[] produces() default {};
}
~~~

* `value/path`：指定匹配映射的url路径:

  * 即支持完全匹配，也支持ant风格的路径匹配(`*`、`**`、`?`通配符)

  * 类级别的`@RequsetMapping`是类中所有方法级别的`@RequestMapping`的主映射，方法级别的映射路径前会加上类级别的映射路径，例如:

    ~~~java
    @Controller
    @RequestMapping("/annotation")
    public class AnnotatedController {
        @RequestMapping( "/accept")
        public String accept(){
            System.out.println("收到请求");
            return "index";
        }
    }
    ~~~

    accept()方法对应的映射为`/annotation/accept`

  * 在方法级别时，`path`支持使用相对路径，在刚才的路径中，`accept`和`/accept`效果一样

* `method`:指定匹配映射的HTTP请求方法：
  * 值为`RequestMethod`枚举:`GET, POST, HEAD, OPTIONS, PUT, PATCH, DELETE, TRACE.`
  * 当在类级别设置`method`时，该类方法级别的`method`值将继承类级别上的`method`值，在方法级别上设置`method`值将覆盖继承的值
* `params`：指定匹配映射的HTTP请求参数:
  * 值为形式为`myParam=myValue`表达式的字符串表示匹配指定的键值
  * 表达式可以用`!=`表示不匹配指定的键值对
  * 表达式可以是`myParam`或者`!myParam`表示匹配或者不匹配指定的键
  * 当在类级别上设置`params`属性时，该类所有的方法级别的`params`将自动继承它，可以通过在方法上指定`params`值来覆盖继承的值

* `headers`:指定匹配映射的HTTP请求首部:
  * 值为形式为`My-Header=myValue`表达式的字符串表示匹配指定请求首部的键值
  * 和`params`一样，可以用`!=` 和`My-Header`表示不匹配指定的键值，或者匹配指定的键
  * 支持通配符`*`

* `consumes`:指定匹配映射的HTTP请求`Content-Type`值
  * 值可以为MediaType中指定的值
  * 支持使用`!`表示不匹配指定的`Content-Type`值
* `produces`：指定匹配映射HTTP请求的`Accept`值
  
  * 值可以为MediaType中指定的值
  * 支持使用`!`表示不匹配指定的`Accept`值

另外SpringMVC提供了子注解:

- `@GetMapping`
- `@PostMapping`
- `@PutMapping`
- `@DeleteMapping`
- `@PatchMapping`

他们与`@RequsetMapping`不同之处只在`method`值已经预先固定

## Handler Method参数

@RequestMapping注释的方法参数可以有很多选择，

* 可以使用特定类型的参数，RequestMappingHandlerAdapter会对特定的参数类型进行赋值或者其他操作，
* 可以使用注解注释参数，指示通知RequestMappingHandlerAdapter对该参数进行特定操作

以供Handler Method内部使

可以作为Handler Method参数的类型和可以注释Handler Method方法参数的注解如下表:

| 方法参数                                                     | 说明                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `WebRequest`, `NativeWebRequest`                             | SpringMVC提供的对request参数、request、session属性的通用访问，是对ServletAPI的封装 |
| `ServletRequest`, <br />`ServletResponse`                    | ServletAPI，也可以使用具体的实现如:`HttpServletRequest `, `MultipartRequest`, `MultipartHttpServletRequest`等 |
| `HttpSession`、`PushBuilder`、`Principal`、`HttpMethod`、`Locale`、`TimeZone`、`ZoneId`、`InputStream`、`Reader`、`OutputStream`、`Writer` | RequestMappingHandlerAdapter会从requestAPI获取对应的对象     |
| `@PathVariable`                                              | 用以访问URI的模板变量                                        |
| `@RequestParam`                                              | 用该注解注释以绑定Servlet的request域中的参数(请求参数或者form表单中的参数) |
| `@RequestHeader`                                             | 用以访问请求的请求首部信息                                   |
| `@CookieValue`                                               | 用以访问请求的Cookie信息                                     |
| `@RequestBody`                                               | 用以访问请求的请求体信息使用 `HttpMessageConverter` 实现     |
| `HttpEntity<B>`                                              | 用以访问请求头和请求体                                       |
| `@RequestPart`                                               | 用以访问格式为 `multipart/form-data`的请求的请求体的Part, 通过`HttpMessageConverter`来实现转化。 |
| `Map`, `Model`, `ModelMap`                                   | 用以访问视图信息                                             |
| `RedirectAttributes`                                         | 重定向时传递参数，让参数不会出现在URL中                      |
| `@ModelAttribute`                                            | 用以访问Model中存在的属性，如果不存在，则实例化并进行数据绑定 |
| `Errors`, `BindingResult`                                    | 访问`@ModelAtrribute`的数据绑定和校验的错误信息；或者`@RequestBody`和`@RequestPart`参数的检验的错误信息，在使用`@Valide`校验参数后必须声明`Errors`、`BindingResult` |
| `SessionStatus` + class-level `@SessionAttributes`           | 在`@Controller`类上标记了`@SessionAttributes`后，可以在方法级别的参数上声明`SessionStatus`用以清空`@SessionAttributes`声明的attributes |
| `@SessionAttribute`                                          | 绑定session属性到方法参数上                                  |
| `@RequestAttribute`                                          | 绑定request属性到方法参数上                                  |
| 其他参数                                                     | 如果一个方法参数于表中以上参数都不匹配，并且它是一个简单类型(由BeanUtils#isSimpleProperty定义)，那么这个参数将被视为一个`@RequestParam`参数，否则他将被是为一个`@ModelAttribute`参数 |

## Handler Method返回值

@RequestMapping注释的方法返回值同样有以下两种：

* 可以使用特定类型的参数
* 可以使用注解注释参数，指示通知RequestMappingHandlerAdapter对该参数进行特定操作

以帮助`RequestMappingHandlerAdapter`完成最终的ModelAndView

| 方法返回值                                                   | 说明                                                         |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| `@ResponseBody`                                              | 方法返回值将通过`HttpMessageConverter`转化并写入响应体。     |
| `HttpEntity<B>`, `ResponseEntity<B>`                         | 指定完整的响应(包括响应头和响应状态)的返回值。通过`HttpMessageConverter`转化 |
| `HttpHeaders`                                                | 返回一个只有响应头没有响应体的响应                           |
| `String`                                                     | 返回一个字符串，将被是为逻辑视图名，供`ViewResolver`解析生成视图，并渲染隐式的模型(由`@ModelAttribute`提供，或者方法参数声明的Model) |
| `View`                                                       | 直接返回一个 `View`实例，会被隐式的模型渲染(由`@ModelAttribute`提供，或者方法参数声明的Model) |
| `Map`, `Model`                                               | 返回一个将被添加到隐式Model的Map或者Model。此时viewName由 `RequestToViewNameTranslator`确定。 |
| `@ModelAttribute`                                            | 表示返回值将被添加到model中，因为没有指定view或者view的逻辑视图名，将使用`RequestToViewNameTranslator` 来隐式返回视图名 |
| `ModelAndView`                                               | 返回要使用的视图和模型属性以及(可选的)响应状态。             |
| `void`                                                       | 一个方法如果返回值是`void`类型或者返回了`null`值，那么它需要在方法内部对response进行了处理，比如通过方法参数:`ServletResponse`、`OutputStream`或者注释`@ResponseStatus`.如果控制器已经做了一个正的`ETag `或`lastModified` 时间戳检查，同样是可以的。如果以上都不成立，`void`返回类型也可以表示REST控制器的“无响应体”或HTML控制器的默认视图名称选择。 |
| `DeferredResult<V>`                                          | 从其他线程异步返回结果                                       |
| `Callable<V>`                                                | 在SpringMVC的管理线程上异步生成返回值                        |
| `ListenableFuture<V>`, `CompletionStage<V>`, `CompletableFuture<V>` | `DeferredResult`的替代对象                                   |
| `ResponseBodyEmitter`, `SseEmitter`                          | 异步响应多个值。                                             |
| `StreamingResponseBody`                                      | 可以通过`StreamingResponseBody`来异步响应输出流              |
| Other return values                                          | 如果返回值用以上方式都无法解析，那么它将被是为一个Model属性。如果返回值是一个简单类型(BeanUtils#isSimpleProperty)，那么将无法解析。 |

##  Handler Method可用注解

### `@RequestParam`

用该注解注释以绑定Servlet的request域中的参数(请求参数或者form表单中的参数)

如果取的是post请求的请求体参数，要求请求体的格式为`application/x-www-form-urlencoded`或者`multipart/form-data`

~~~java
@RequestMapping("/requestParam/demo?param=test")
public void requestParam(@RequestParam("param") String param){
    // ...
}
~~~

* 可以注释在类型为Array或者List的参数上，以解析同一参数名称的多个参数值。

  ~~~java
  // url: /requestParam/demo1?list=1,2,3,4,5
  @RequestMapping("/requestParam/demo1")
  public void requestParam(@RequestParam("list") List<String> strings){
      System.out.println(strings.size());
  }
  ~~~
  
* 如果注释的参数类型不是String，而是其他简单类型(由BeanUtils#isSimpleProperty定义)，SpringMVC将自动尝试使用类型转换

  * 简单类型包括：基本数据类型及其包装类、Enum、CharSequence、Number、Date、Temporal、URI、URL、Locale、Class

  ~~~java
  @RequestMapping("/demo")
  public void handle(@RequestParam("date") @DateTimeFormat(pattern="yyyy-MM-dd") Date date){
      System.out.println(date);
  }
  ~~~

* 如果启用了`MultipartResolver`，它会将格式为`multipart/form-data`的请求体解析为普通的请求参数

  这样`@RequestParam`就可以注释`MultipartFile`类型的参数以接收上传的文件:

  ~~~java
  @PostMapping("/requestParam/demo2")
  public void requestParam(@RequestParam("file")MultipartFile file,
                           @RequestParam("username") String username){
      System.out.println(file.getSize());
      System.out.println(username);
  }
  ~~~

  也可以将参数声明为`List<MultipartFile>`以接受同一参数名的多个文件值

  ~~~java
  @PostMapping("/requestParam/demo3")
  public void requestParams(@RequestParam("file") List<MultipartFile> files){
      for (MultipartFile file :files){
          System.out.println(file.getSize());
      }
  }
  ~~~

* 如果该注解指定的参数为` Map<String, String> `或者` MultiValueMap<String, String> `,并且注解没有指定参数名，那么参数将接受所有的request域中的键值对:

  ~~~java
  @PostMapping("/requestParam/demo4")
  public void requestParams(@RequestParam Map<String,String> maps){
      for (Map.Entry<String,String> entry:maps.entrySet()){
          System.out.println(entry.getKey()+"---"+entry.getValue());
      }
  }
  ~~~

**注意：**使用`@RequestParam`注解是可选的，对于任何简单类型的参数，如果没有任何相关的注解，那么默认将采用`@RequestParam`声明的绑定方式，从ServletRequest域中尝试绑定对应参数名的键

### `@RequestBody`

使用`HttpMessageConverter`将请求参数序列化为注释的参数类型:

将参数类型进行实例化，并用set方法进行参数绑定

~~~java
@PostMapping( "/requestBody/demo0")
public void requestBody(@RequestBody User user){
    System.out.println(user);
}
~~~

其中User类定义如下:

~~~java
@Data
public class User {
    private String username;
    private String password;
}
~~~

**注意:**作为`@RequestBody`注释的方法类型对象，必须有无参构造，而且对象内想要被映射的参数必须实现对应的set方法，否则无法映射

### `@RequestPart`

适用于请求体格式为`multipart/form-data`的复杂请求，可以同时解析对象和二进制文件:

~~~java
@PostMapping("/requestPart/demo0")
public void requestPart(@RequestPart("user") User user,
                        @RequestPart("file") MultipartFile file){
    System.out.println(user);
    System.out.println(file.getSize());
}
~~~

**注意:**`@RequestParam`也同样支持`multipart/form-data`的请求，它们之间的区别是:

`@RequestParam`使用Converter进行类型转换，只能转换简单类型

@RequestPart使用HttpMessageConverters 进行类型转换，支持转换复杂的参数类型

### `@PathVariable`

用以访问URI的模板变量

~~~java
@GetMapping("/owners/{ownerId}/pets/{petId}")
public Pet findPet(@PathVariable Long ownerId, @PathVariable Long petId) {
    // ...
}
~~~

如果注释的参数类型是` Map<String, String> `，则将填充所有的路径变量到该Map中

~~~java
@GetMapping("/demo/{version}/{id}")
public void handle(@PathVariable Map<String,String> maps){
    // ...
}
~~~

### `@RequestHeader`

用以访问HTTP请求的请求首部

~~~java
@GetMapping("/demo")
public void handle(
        @RequestHeader("Accept-Encoding") String encoding, 
        @RequestHeader("Keep-Alive") long keepAlive) { 
    //...
}
~~~

除了直接指定请求首部的值，还可以绑定如下类型参数以访问所有的请求首部参数:

 `Map<String, String>`、`MultiValueMap<String, String>`、 `HttpHeaders`,如:

~~~java
@GetMapping("/demo")
public void handle(@RequestHeader HttpHeaders headers){
    for (Map.Entry<String, List<String>> entry :headers.entrySet()){
        System.out.println(entry.getKey()+"---"+entry.getValue());
    }
}
~~~

### `@CookieValue`

用以访问HTTP请求的Cookie

~~~java
@GetMapping("/demo")
public void handle(@CookieValue("JSESSIONID") String cookie) { 
    //...
}
~~~

也可以直接绑定`Cookie`类型的参数:

~~~java
@GetMapping("/demo")
public void handle(@CookieValue("JSESSIONID") Cookie cookie) { 
    //...
}
~~~

### `@ModelAttribute`

`@ModelAttribute`可以注释在方法参数和方法体上

#### 注释在方法参数上

`@ModelAttribute`注释在方法入参时:

可以解析`form-data`或者`x-www-form-urlencoded`格式的请求体中参数

可以从model中访问或者创建一个对象，然后通过`WebDataBinder`将该对象和请求参数进行绑定

* 先从Model中获取指定的参数类型，由`@ModelAttribute`注解的value/name值指定绑定的model属性名，如果不指定，默认由参数类型获取:
  * 如User类默认绑定的属性名是user

  * 如`List<User>`类默认绑定的属性名是userList

* 如果获取不到，则实例化参数类型

* 从ServletRequest域中获取对应的字段，绑定到参数实例中。这被称为参数绑定。

* 最后将参数绑定后的参数实例也添加到Model中暴露给视图

~~~java
@PostMapping("/modelAttribute/demo")
public void modelAttribute(@ModelAttribute User user, Model model){
    System.out.println(user);
    User u =(User) model.getAttribute("user");
    System.out.println(u);
}
~~~

在实例化参数类型后，会应用参数绑定，将ServletRequest参数名匹配到目标参数类型的字段名

参数绑定可能会出现异常，为了在方法内部处理异常，可以同时定义`BindingResult`类型的参数，在方法内部访问绑定结果:

~~~java
@PostMapping("/modelAttribute/demo1")
public void modelAttribute(@ModelAttribute User user, BindingResult bindingResult){
    System.out.println(user);
    System.out.println(bindingResult.hasErrors());
}
~~~

如果只想要访问Model中的属性，而不想进行参数绑定，可以指定`@ModelAttribute`注解的属性`binding=false`。

~~~java
@ModelAttribute
public User setModel(){
    return new User("test","test");
}

@PostMapping("/modelAttribute/demo2")
public void modelAttribute(@ModelAttribute(binding = false) User user){
    System.out.println(user);
}
~~~

**注意:**使用`@ModelAttribute`是可选的，默认请求下，任何不是简单类型( 由BeanUtils#isSimpleProperty定义)的参数，并且该参数没有被其他参数处理器处理过，那么该参数就会被视为被`@ModelAttribute`注解注释了。

#### 注释在方法体上

单独注释`@Controller`类中的方法，为该类中其他所有`@RequestMapping`方法初始化model

一个`@Controller`控制器中可以有任意数量的`@ModelAttribute`方法。接收到Web请求后，所有的这些`@ModelAttribute`方法将先于`@RequestMapping`方法被调用，为其初始化model

`@ModelAttribute`方法可以有多种结构:

* 有返回值，会将返回值添加为model属性:

  ~~~java
  @ModelAttribute
  public User setModel(){
      return new User("test","test");
  }
  ~~~

* 无返回值，在方法体内添加任意model属性:

  ~~~java
  @ModelAttribute
  public void setModel(Model model) {
      model.addAttribute(new User("test","test"));
      ......
  }
  ~~~

当然，`@ModelAttribute`方法和`@RequestMapping`方法一样，可以定义特定类型的方法参数或者为方法参数注释特定注解，以访问请求的信息

除了单独注释`@Controller`类中的方法，`@ModelAttribute`还可以于`@RequestMapping`注解组合使用，以注释方法的返回值是`model`中的属性。通常情况下这种使用方式是不必要的，因为这是controller的默认行为，除非你的返回值是String，而不想让该String被解释为是view 的逻辑名:

~~~java
@PostMapping("/modelAttribute/demo3")
@ModelAttribute("key")
public String modelAttribute(){
    return "value";
}
~~~

### `@SessionAttributes`

`@SessionAttributes`用于在请求之间的HTTP Servlet Session中存储model属性：

~~~java
@Controller
@SessionAttributes("user")
@RequestMapping("/user")
public class SessionAttributesController {
    @PostMapping("/add")
    public void addUser(User user){
        System.out.println(user);
        System.out.println("已添加");
    }
    @GetMapping("/get")
    @ResponseBody
    public User getUser(Model model){
        return (User) model.getAttribute("user");
    }
    @GetMapping("/delete")
    public void deleteUser( SessionStatus status){
        status.setComplete();
    }
}
~~~

### ` @SessionAttribute`

从Servlet的session域中获取已经存在的属性(之前的请求创建的，或者Filter和HandlerIntercepter创建的)

~~~java
@GetMapping("/sessionAttribute/demo")
public void sessionAttribute(@SessionAttribute("user") User user){
    System.out.println(user);
}
~~~

### `@RequestAttribute`

从Servlet的request域中获取提前创建的已经存在的属性(Filter和HandlerIntercepter创建的)

~~~java
@GetMapping("/requestAttribute/demo")
public void requestAttribute(@RequestAttribute("user") User user){
    System.out.println(user);
}
~~~

### `@ResponseBody`

```java
@RequestMapping("/getUser")
@ResponseBody
public User user(){
    return new User("test","test");
}
```

该注解也可和`@Controller`组合使用，表示该`@Controller`类中所有方法都隐士注释了`@ResponseBody`

spring4.0提供了`@RestController`表示`@Controller`和`@ResponseBody`的组合注解



## Handler Method可用实体类

SpringMVC允许和提供了一些对象，以声明在Handler Method的参数和返回值上

### `WebRequest`

`WebRequest`、`NativeWebRequest`:SpringMVC提供的对request参数、request、session属性的通用访问，是对ServletAPI的封装

示例：

~~~java
@RequestMapping("accept")
public void accept(WebRequest request){
	......
}
~~~

### `ServletRequest`&`ServletResponse`

`ServletRequest`、`ServletResponse`:ServletAPI，也可以使用具体的实现如:`HttpServletRequest `, `MultipartRequest`, `MultipartHttpServletRequest`等

示例：

~~~java
@RequestMapping("accept")
public void accept(HttpServletRequest request, HttpServletResponse response){
    // ...
}
~~~

### ServletRequestAPI对象

可以直接指定下面类型的对象，RequestMappingHandlerAdapter会通过Servlet RequestAPI从当前请求获取对应的对象:

`HttpSession`、`PushBuilder`、`Principal`、`HttpMethod`、`Locale`、`TimeZone`等

~~~java
@RequestMapping("accept")
public void accept(HttpSession session,HttpMethod method){
	// ...
}
~~~

**注意:**对于`HttpSession`，该对象不是线程安全的，如果有多个Controller使用同一个`HttpSession`对象的情况，需要将`RequestMappingHandlerAdapter`实例的`synchronizeOnSession`设置为true

### `Model`

SpringMVC在调用处理器方法之前，会创建一个隐式的Model对象，作为模型数据的存储容器

如果Handler Method的入参有声明`Map`、`Model`或者其子类(如`ModelMap`),则SpringMVC会将事先创造的Model的引用传递给入参，或者将其中的属性填充到map里供方法访问model属性。

可以在处理方法内访问model属性，也可以像model中添加model

~~~java
@RequestMapping("/accept")
public String accept(Model model){
	// ...
}
~~~





### `RedirectAttributes`

默认情况下，所有model属性都被认为是作为重定向URL中的URI模板变量公开的。在其余属性中，那些基本类型或基本类型的集合或数组将自动作为查询参数追加。

~~~java
@Controller
public class RedirectController {
    @GetMapping("/redirect")
    public String demo(Model model){
        model.addAttribute("param","test");
        model.addAttribute("number",123);
        return "redirect:/target/{param}";
    }

    @GetMapping("/target/{param}")
    public void redirect(@PathVariable("param") String param, int number, HttpServletRequest request){
        System.out.println(request.getRequestURI());
        Map<String, String[]> parameterMap = request.getParameterMap();
        for (Map.Entry<String, String[]> entry: parameterMap.entrySet()){
            System.out.println(entry.getKey()+"---"+ Arrays.toString(entry.getValue()));
        }
        System.out.println(param);
        System.out.println(number);
    }
}
~~~

但是有时候我们不希望有些参数出现在URI中，就可以使用Model的子接口:`RedirectAttributes`,同样可以传递参数，且不会出现在URI中:

~~~java
@Controller
public class RedirectController {
    @GetMapping("/redirect")
    public String demo(RedirectAttributes redirectAttributes) {
        redirectAttributes.addAttribute("param","test");
        return "redirect:/target";
    }
    @GetMapping("/target")
    public void redirect(String param) {
        System.out.println(param);
    }
}
~~~

### `HttpEntity`

与`@RequestBody`作用类似，解析请求域中键值对，将其序列化为指定的实体类:

~~~java
@PostMapping("demo7")
public void handle(HttpEntity<User> httpEntity){
    System.out.println(httpEntity.getBody());
}
~~~

同时也可以作为方法的返回值，被写入响应体

~~~java
@RequestMapping("/getUserByEntity")
public HttpEntity<User> getUserByHttpEntity(){
    return new HttpEntity<>(new User("test","test"));
}
~~~

### `ResponseEntity`

作为方法的返回值，和`@ResponseBody`类似，但会返回响应状态和响应头:

~~~java
@RequestMapping("/getUserByResponseEntity")
public ResponseEntity<User> getUserByResponseEntity(){
    return ResponseEntity.ok(new User("test","test"));
}
~~~

## 处理Handler Method出入参

我们知道，Web请求和响应的正文都是不同媒体格式的字符串类型，而我们定义的HandlerMethod出入参一般是Java对象，`RequestMappingHandlerAdapter`需要一些组件的支持，以实现Java对象和Web请求响应体的转换

### HttpMessageConverter

在前面介绍MehtodHandler方法的返回值和入参时，已经多次提到了`HttpMessageConverter`,它能帮助`RequestMappingHandlerAdapter`：

* 将读取到的Web请求体中的请求体信息转换为对象传递给Handler Method的参数
* 将Handler Method返回的对象转换为响应体信息以写入Web响应体

`@RequestBody`、`@ResponseBody`、`@RequestPart`、`HttpEntity`、`ResponseEntity`背后的实现都需要`HttpMessageConverter`的支持

以实现类型转换和数据绑定

其定义如下:

~~~java
public interface HttpMessageConverter<T> {
	//指示给定的类型是否是可读的，同时指示支持的请求体Content-Type
	boolean canRead(Class<?> clazz, MediaType mediaType);
	//指示可写的对象类型，同时指示支持的响应体Content-Type
	boolean canWrite(Class<?> clazz, MediaType mediaType);
	//返回当前转换器支持的媒体类型
	List<MediaType> getSupportedMediaTypes();
	//将请求信息流转换为T类型对象
	T read(Class<? extends T> clazz, HttpInputMessage inputMessage);
	//请T类型对象实例转换为输出信息流
	void write(T t, MediaType contentType, HttpOutputMessage outputMessage);
}
~~~

Spring为`HttpMessageConverter`提供了众多的实现类，以支持不同场景的类型转换:

| `HttpMessageConverter`实现                | `<T>` Type                         | Supported Read<br />Content-Type    | Supported Write <br />Content-Type                           |
| ----------------------------------------- | ---------------------------------- | ----------------------------------- | ------------------------------------------------------------ |
| `StringHttpMessageConverter`              | `String`                           | `*/*`                               | `text/plain`                                                 |
| `FormHttpMessageConverter`                | `MultiValueMap`<br />`<String, ?>` | `application/x-www-form-urlencoded` | `application/x-www-form-urlencoded`<br />`multipart/form-data` |
| `AllEncompassingFormHttpMessageConverter` | `MultiValueMap`<br />`<String, ?>` | `application/x-www-form-urlencoded` | `application/x-www-form-urlencoded`<br />`multipart/form-data` |
| `ResourceHttpMessageConverter`            | `Resource`                         | `*/*`                               | 由写入的`Resource`决定                                       |
| `BufferedImageHttpMessageConverter`       | `BufferedImage`                    | 由注册的` image readers`决定        | `BuferedImage`对应的类型                                     |
| `ByteArrayHttpMessageConverter`           | `byte[]`                           | `*/*`                               | `application/octet-stream`                                   |
| `SourceHttpMessageConverter`              | `T extends Source`                 | `text/xml`、`application/xml`       | `text/xml`、`application/xml`                                |
| `MappingJackson2HttpMessageConverter`     | `Object`                           | `application/json`                  | `application/json`                                           |
| `RssChannelHttpMessageConverter`          | `Channel`                          | `application/rss+xml`               | `application/rss+xml`                                        |
| ······                                    |                                    |                                     |                                                              |

`RequestMappingHandlerAdapter`默认已经装配了下面`HttpMessageConverter`:

* `StringHttpMessageConverter`
* `ByteArrayHttpMessageConverter`
* `SourceHttpMessageConverter`
* `AllEncompassingFormHttpMessageConverter`

如果想要使用其他的`HttpMessageConverter`，需要在SpringWeb容器中显示定义`RequestMappingHandlerAdapter`并指定`HttpMessageConverter`

### DataBinder

 `@RequestParam`, `@RequestHeader`, `@PathVariable`, `@CookieValue`、`@ModelAttribute`等注解指示的方法参数的传递，需要`DataBinder`来实现数据绑定，绑定过程包括数据的转换、格式化、校验等

其中`@ModelAttribute`绑定的是参数对象中的字段

![DataBinder工作流程](https://gitee.com/wangziming707/note-pic/raw/master/img/DataBinder%E5%B7%A5%E4%BD%9C%E6%B5%81%E7%A8%8B.jpeg)

SpringMVC将ServletRequest对象以及处理方法的入参对象实例传递给`DataBinder`

* `DataBinder`首先调用装配在SpringWeb上下文的ConverionService组件进行数据转换、数据格式化的工作，将ServletRequest中的消息填充到入参对象中
* 然后调用`Validator`组件对已经绑定了请求消息数据的入参对象进行数据合法性校验，最终生成数据绑定结果BindingResult对象

#### 数据转换



#### 数据格式化



#### 数据校验













## 异步请求

servlet和SpringMVC支持对请求进行异步响应。接收到Web请求后，可以在保持Web连接的情况下结束Servlet容器线程，在其他线程进行响应处理，要开启异步支持，首先需要设置Servlet和所有的Filter支持异步`<async-supported>`:

~~~xml
<servlet>
    <servlet-name>dispatcherServlet</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath:dispatcher-servlet-annotation.xml</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
    <async-supported>true</async-supported>
</servlet>

<filter>
    <filter-name>encodingFilter</filter-name>
    <filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
    <async-supported>true</async-supported>
</filter>
~~~

并且可以设置mvc以控制异步行为:
~~~xml
<mvc:annotation-driven enable-matrix-variables="true" >
    <mvc:async-support default-timeout="100000"/>
</mvc:annotation-driven>
~~~

然后可以声明下面的Handler Method返回值以实现异步请求

### `DeferredResult`

当一个请求到达API接口，如果该API接口的return返回值是DeferredResult，在没有超时或者DeferredResult对象设置setResult时，接口不会返回，但是Servlet容器线程会结束，`DeferredResult`另起线程来进行结果处理(即这种操作提升了服务短时间的吞吐能力)，并`setResult()`，如此以来这个请求不会占用服务连接池太久，如果超时或设置`setResult()`，接口会立即返回。

~~~java
@Controller
@RequestMapping("/async")
public class AsynchronousController {
    @GetMapping("/demo1")
    @ResponseBody
    public DeferredResult<String> demo1(){
        DeferredResult<String> deferredResult = new DeferredResult<>();
        MyThread myThread = new MyThread(deferredResult);
        myThread.start();
        return deferredResult;
    }
}
~~~

最终的结果在MyThread的run方法中处理：

~~~java
class MyThread extends Thread{
    private final DeferredResult<String> deferredResult;
    public MyThread(DeferredResult<String> deferredResult){
        this.deferredResult = deferredResult;
    }
    @SneakyThrows
    @Override
    public void run() {
        Thread.sleep(3000);
        deferredResult.setResult("123");
    }
}
~~~

### `Callable`

作用和DeferredResult类型，不过让SpringMVC线程来异步执行callable任务:
~~~java
@GetMapping("/demo")
@ResponseBody
public Callable<String> demo2(){
    return () ->{
        Thread.sleep(3000);
        return "132";
    };
}
~~~

如果不设置，默认使用`SimpleAsyncTaskExecutor`来执行异步任务

### `ResponseBodyEmitter`

使用`DeferredResult`和`Callable`只能异步返回一个值。

使用`ResponseBodyEmitter`可以异步返回多个值。

```java
@GetMapping("/demo3")
public ResponseBodyEmitter demo3(){
    ResponseBodyEmitter emitter = new ResponseBodyEmitter();
    EmitterThread thread = new EmitterThread(emitter);
    thread.start();
    return emitter;
}
```

其中EmitterThread为:

~~~java
class EmitterThread extends Thread{
    private final ResponseBodyEmitter emitter;
    public EmitterThread(ResponseBodyEmitter emitter){
        this.emitter = emitter;
    }
    @SneakyThrows
    @Override
    public void run() {
        emitter.send("123");
        Thread.sleep(1000);
        emitter.send("456");
        Thread.sleep(1000);
        emitter.send("789");
        emitter.complete();
    }
}
~~~

也可以将`ResponseBodyEmitter`作为`ResponseEntity`的body，以设置响应头和状态码。

可以用fetchAPI接收响应流：

~~~javascript
function demo() {
    let url = "....../async/demo3"
    fetch(url).then((response) => push(response.body.getReader()));
}


function push(reader) {
    reader.read().then(({done, value}) => {
        if (done) {
            return;
        }
        console.log(Uint8ArrayToString(value));
        $("#content").append(Uint8ArrayToString(value))
        push(reader);
    });
}

function Uint8ArrayToString(fileData){
    let dataString = "";
    for (let i = 0; i < fileData.length; i++) {
        dataString += String.fromCharCode(fileData[i]);
    }
    return dataString
}
~~~

### `SseEmitter`

`SseEmitter `(`ResponseBodyEmitter`的子类)提供了对服务器发送事件的支持，其中从服务器发送的事件按照W3C SSE(Server-Sent Events)规范格式化。

~~~java
@GetMapping("/demo3")
public SseEmitter demo4(){
    SseEmitter emitter = new SseEmitter();
    EmitterThread thread = new EmitterThread(emitter);
    thread.start();
    return emitter;
}
~~~

其中`EmitterThread`和上一个示例一样

需要用sse规定的EventSourceAPI来处理响应:

~~~javascript
function demo() {
    let url = "....../async/demo4"
    const evtSource = new EventSource(url);
    evtSource.onmessage = function(event) {
        $("#content").append(event.data)
    }
    evtSource.onerror = function (event) {
        console.log("error")
        evtSource.close();
    }
}
~~~

### `StreamingResponseBody`

可以通过`StreamingResponseBody`来异步响应输出流

~~~java
@GetMapping("/demo5")
public StreamingResponseBody demo5(){
    return outputStream -> {
        for (int i = 0; i < 10; i++) {
            outputStream.write(Integer.toString(i).getBytes(StandardCharsets.UTF_8));
            try {
                Thread.sleep(500);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        }
    };
}
~~~









# 其他

## SpringMVC解决中文乱码问题

SpringMVC提供了编码过滤器，直接在web.xml中配置即可：

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
    <!--这两个可以不写-->
    <!--强制开启request字符集-->
    <init-param>
        <param-name>forceRequestEncoding</param-name>
        <param-value>true</param-value>
    </init-param>
    <!--强制开启response字符集-->
    <init-param>
        <param-name>forceResponseEncoding</param-name>
        <param-value>true</param-value>
    </init-param>
</filter>
<filter-mapping>
    <filter-name>CharacterEncodingFilter</filter-name>
    <url-pattern>/*</url-pattern>
</filter-mapping>
~~~

## SpringMVC读取静态资源

在配置SpringMVC的中央控制器DispatcherServlet时，我们设置的url-pattern是`/`,这意味着浏览器的所有请求都会被中央控制器拦截处理，包括动态资源和静态资源的请求

但中央处理器只能处理类似.jsp .actiond 的动态资源，无法处理 .html .js .css 的静态资源，这导致了tomcat无法读取静态资源给浏览器

有三个方法方案：

1. 使用tomcat自带的default Servlet处理
2. 在SpringMVC中配置静态资源路径
3. 在SpringMVC中设置静态资源的处理方式：交给default Servlet

### 使用Defalut Servlet

直接在web.xml文件中配置默认servlet路径:

~~~xml
  <!--配置默认servlet-->
  <servlet-mapping>
    <servlet-name>default</servlet-name>
    <url-pattern>/images/*</url-pattern>
    <url-pattern>/static/*</url-pattern>
    <url-pattern>/photo/*</url-pattern>
    <url-pattern>/laydate/*</url-pattern>
    <url-pattern>*.js</url-pattern>
    <url-pattern>*.css</url-pattern>
  </servlet-mapping>
~~~

tomcat会优先处理更具体精确的路径，所以tomcat收到请求后，会先匹配default的路径，如果是default路径指定的url pattern 则会交给default处理，如果在指定的路径范围，才会再交给DispatcherServlet处理

### 在SpringM配置静态资源路径

在SpringMVC的核心配置文件中：

~~~xml
<!-- 以下路径不会被当控制器拦截，当静态资源处理 -->
<mvc:resources mapping="/images/*" location="/images/" />
<mvc:resources mapping="/css/*" location="/css/" />
<mvc:resources mapping="/js/*" location="/js/" />
~~~

### SpringMVC交还给default Servlet处理

在SpringMVC核心配置文件中：

~~~xml
<!-- 由springmvc对请求进行分类，如果是静态资源，则交给DefaultServlet处理 -->
<mvc:default-servlet-handler/>
~~~
