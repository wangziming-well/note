# MVC

 MVC是一种设计模式，常用于开发web应用程序的框架中。MVC的意思是Model-View-Controller（模型-视图-控制器）。它将web应用程序分为三个主要组件：模型、视图和控制器。每个组件都负责不同的功能，但又相互关联，以便应用程序可以正确地工作。

* Model（模型） 模型是应用程序中的数据和逻辑组件，它处理应用程序数据和业务逻辑。模型是独立于用户界面和控制器的，因此可以在不影响应用程序其他部分的情况下修改数据和逻辑。
* View（视图） 视图是应用程序中的用户界面组件，它负责呈现模型中的数据。视图通常是HTML模板，它显示给用户的是模型中的数据。视图不处理数据或逻辑，它只负责呈现它们。
* Controller（控制器） 控制器是应用程序中的中介组件，负责接收用户请求，根据请求通知模型更新数据，并选择合适的视图响应给用户

![mvc设计模式](https://gitee.com/wangziming707/note-pic/raw/master/img/mvc%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F.png)

MVC设计模式是Web开发中的一种重要的设计模式，它能够提供良好的结构化方式，使得Web应用程序的不同部分更易于维护和扩展，提高了应用程序的质量和可靠性。

# SpringMVC简介

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
* DefaultAnnotationHandlerMapping

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

面向单一视图类型的ViewResolver类都会直接或间接的继承UrlBasedViewResolver

使用该类型的ViewResolver，不需要配置具体的逻辑视图名到具体View的映射关系。通常只要指定以下视图模板所在的位置，这些ViewResolver会按照逻辑视图名，找到相应的模板文件、构造相应的View实例并返回。

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
* `HandlerInterceptor`处理器拦截器
* `HandlerAdaptor`使用不同类型的Handler
* `HandlerExceptionResolver`：提供请求时异常的标准处理方式

* `LocaleResolver`：提供更方便的显示国际化视图
* `ThemeResolver`:让用户选择不同的主题

## MultipartResolver

HTTP协议的实体类型Context-Type有值`multipart/form-data`格式，支持表单的文件上传；在前端页面HTML页面或者js脚本中，设置表单的属性enctype为`multipart/form-data`以对请求实体内容进行编码

针对这种编码类型的请求进行解析上传的文件，是通用的逻辑，有通用的类库,如:Oreilly、Commons FileUpload类库等。

在实现基于表单的文件上传功能时，SpringMVC框架底层实际上也是使用了以上的几种类库，只是通过MultipartResolver策略接口的抽象，将具体选用哪一种类库的权利给了用户。

MultipartResolver提供了两个可用的实现:

* CommonsMultipartResolver：基于 Apache Commons FileUpload类库实现，使用它需要引入相应依赖
* StandardServletMultipartResolver:基于Servlet 3.0 Part API的标准MultipartResolver实现

### DispatcherServlet使用MultipartResolver

`DispatcherServlet`作为一个servlet在启动时会被servlet容器调用init()方法进行初始化，此时会调用DispatcherServlet的initMultipartResolver()方法进行MultipartResolver的初始化:

从自己的WebApplicationContext中获取beanName固定为:`multipartResolver`的MultipartResolver实例

然后在收到Web请求时，







# 核心配置文件

## 注解版

* springmvc.xml

~~~xml
<!--使用注解  简化配置  和SpringMVC的使用-->
<!--自动扫描   三大件   处理器映射器  处理器适配器  视图解析器-->
<mvc:annotation-driven/>
<!--处理器  注解扫描   自动扫描 @Controller-->
<context:component-scan base-package="com.bjpn.controllers"/>
~~~

* controller

~~~java
@Controller
public class SecondController {
    @RequestMapping("/secondController.action")
    public String controller(){
        return "/index.jsp";
    }
}
~~~



# SpringMVC常用注解

* `@Controller`标识当前类是处理器类

* `@RequestMapping`映射url路径，可以出现在类和方法上

    在类上表示当前处理方法的父路径，在方法上表示当前处理方法的路径

    参数：

    * value:表示请求路径，值为路径字符串数组
    * method:表示请求方式，只能在方法上时用，如GET，POST

* `@PathVariable`接收动态参数  参数值写在请求中
* `@requestParam`         配置不同名参数
* `@ResponseBody`     异步ajax的json格式
* `@RequestBody`      接收前端异步请求参数

# SpringMVC处理器

表示处理器的方法有三种，分别是返回值为：

* `ModelAndView`返回视图，渲染后响应跳转页面资源显示给用户
* `String`返回要跳转的页面路径字符串，工作中常用
* `void`不跳转页面，常用于简单的异步

演示：

~~~java
@Controller
public class SecondController {
    @RequestMapping("/methodDemo1.action")
    public String methodDemo1(){
        return "/index.jsp";
    }
    @RequestMapping("/methodDemo2.action")
    public ModelAndView methodDemo2(){
        ModelAndView modelAndView = new ModelAndView();
        modelAndView.setViewName("/index.jsp");
        return modelAndView;
    }
    @RequestMapping("/methodDemo3.action")
    public void methodDemo3(HttpServletRequest request, HttpServletResponse response) throws IOException {
        PrintWriter writer = response.getWriter();
        writer.write("响应成功");
    }
}
~~~



## 转发与重定向

SpringMVC处理器，不管返回值是ModelAndView还是String，默认调用转发

可以通过在表示资源路径的字符串中指定跳转类型来指定跳转方式：

* forward：表示转发
* redirect：表示重定向

演示：

~~~java
@Controller
public class ThirdController {
    @RequestMapping("/jumpByForward.action")
    public ModelAndView jumpByForward(){
        ModelAndView modelAndView = new ModelAndView();
        modelAndView.setViewName("forward:/index.jsp");
        return modelAndView;
    }
    @RequestMapping("/jumpByRedirect.action")
    public String jumpByRedirect(){
        return "redirect:/index.jsp";
    }
}
~~~



# SpringMVC处理器适配器

处理器适配器封装了从前端接受参数的过程，以及创建对象的过程

处理器适配器能给对应的处理器提供

* 当前工程中的对象
* 前端传递的参数

## 提供对象

适配器能为处理器提供当前工程中所有的有无参构造的类对象，

只需要处理器在形参列表中声明：

示例：

~~~java
@RequestMapping("objectSupport.action")
public ModelAndView objectSupport(ModelAndView modelAndView , User user){
    modelAndView.setViewName("redirect:/main.jsp");
    System.out.println(user);
    return modelAndView;
}
~~~



## 提供参数

适配器可以给处理器提供前端传递的参数，处理器可以用多种方式接收参数

### 通过同名参数接收

适配器会将前端的参数传递给与处理器形参名相同的位置，并自动进行类型转换

示例：

前端：

~~~jsp
 <a href="${pageContext.request.contextPath}/demo/getParamDemo1.action?name=张三&age=18">通过同名参数接收参数</a>
~~~

处理器：

~~~java
@RequestMapping("/getParamDemo1.action")
public void getParamDemo1(String name ,int age){
    System.out.println("name:"+name +"---age:"+(age+2));
}
~~~

### 通过对象接收

适配器会扫描处理器形参上的对象属性，如果前端参数与属性名相同，也会自动将参数值赋值给对象

前端：

~~~jsp
<a href="${pageContext.request.contextPath}/demo/getParamDemo2.action?name=张三">通过对象接收参数</a>
~~~

处理器：

~~~java
@RequestMapping("/getParamDemo2.action")
public void getParamDemo2(User user){
    System.out.println(user);
}
~~~

### 通过不同名对象接收

可以通过`@RequestParam`注解，指定处理器形参要接收的参数名

前端：

~~~jsp
<a href="${pageContext.request.contextPath}/demo/getParamDemo3.action?name=张三">接收不同名参数</a>
~~~

处理器：

~~~java
@RequestMapping("/getParamDemo3.action")
public void getParamDemo3(@RequestParam("name") String uname){
    System.out.println("uname:"+uname);
}
~~~

### restful风格接收

可以直接将请求写在参数中，接收请求参数需要在请求路径中用`{}`且需要在要接收该参数的形参位置加上注解`@PathVariable()`

springmvc在接收到前端请求后，会先扫描一般请求路径，如果没有匹配的，再匹配restful风格的请求路径，如果格式匹配，则会分发给对应处理器

前端：

~~~jsp
<a href="${pageContext.request.contextPath}/demo/张三.action">restful风格接收参数</a>
~~~

后端：

~~~java
@RequestMapping("/{name}.action")
public void getParamDemo4(@PathVariable("name") String name){
    System.out.println(name);
}
~~~



# SpringMVC提供的域对象

在servlet中我们使用request，session，servletContext(application)域对象来向前端传递参数

在SpringMVC中：

* request，session对象可以由处理器适配器直接提供给处理器

* servletContext对象因为没有无参构造，所以只能由servlet-api中的方法获取

除此之外SpringMVC还自己封装了域对象，以供传值：

## ModelAndView

ModelAndView除了提供视图外，还能够起到域对象的作用

生命周期与request相同，但是在重定向时，modelAndView会将key-value拼接到请求中，多用于服务器内跳转

示例：

~~~java
//ModelAndView  转发时类似request域
@RequestMapping("/returnParamDemo3.action")
public ModelAndView returnParamDemo3(ModelAndView modelAndView){
    modelAndView.addObject("mavKey","这是mav传值");
    modelAndView.setViewName("forward:/page/paramSuccess.jsp");
    return modelAndView;
}
//ModelAndView  重定向时会把key-value拼接在请求中  多用在跳转其它处理器
@RequestMapping("/returnParamDemo4.action")
public ModelAndView returnParamDemo4(ModelAndView modelAndView){
    modelAndView.addObject("mavKey","这是mav传值");
    modelAndView.setViewName("redirect:/page/paramSuccess.jsp");
    return modelAndView;
}
~~~

## Model

与ModelAndView类似，为了在String返回值类型的处理器中传值

示例：

~~~java
//使用Model
@RequestMapping("/returnParamDemo5.action")
public String returnParamDemo5(Model model){
    model.addAttribute("modelKey", "这是model传值");
    return "forward:/page/paramSuccess.jsp";
}
//重定向时会把key-value拼接在请求中  多用在跳转其它处理器
@RequestMapping("/returnParamDemo6.action")
public String returnParamDemo6(Model model){
    model.addAttribute("modelKey", "这是model传值");
    return "redirect:/page/paramSuccess.jsp";
}
~~~

# SpringMVC解决中文乱码问题

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



# SpringMVC读取静态资源

在配置SpringMVC的中央控制器DispatcherServlet时，我们设置的url-pattern是`/`,这意味着浏览器的所有请求都会被中央控制器拦截处理，包括动态资源和静态资源的请求

但中央处理器只能处理类似.jsp .actiond 的动态资源，无法处理 .html .js .css 的静态资源，这导致了tomcat无法读取静态资源给浏览器

有三个方法方案：

1. 使用tomcat自带的default Servlet处理
2. 在SpringMVC中配置静态资源路径
3. 在SpringMVC中设置静态资源的处理方式：交给default Servlet



## 使用Defalut Servlet

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



## 在SpringM配置静态资源路径

在SpringMVC的核心配置文件中：

~~~xml
<!-- 以下路径不会被当控制器拦截，当静态资源处理 -->
<mvc:resources mapping="/images/*" location="/images/" />
<mvc:resources mapping="/css/*" location="/css/" />
<mvc:resources mapping="/js/*" location="/js/" />
~~~



## SpringMVC交还给default Servlet处理

在SpringMVC核心配置文件中：

~~~xml
<!-- 由springmvc对请求进行分类，如果是静态资源，则交给DefaultServlet处理 -->
<mvc:default-servlet-handler/>
~~~



# Jackson处理异步数据

springMVC提供了jackson以处理json数据

* 引入依赖：

    需要导入core detabind annotations

    ~~~xml
    <!--导入jackson-->
    <dependency>
      <groupId>com.fasterxml.jackson.core</groupId>
      <artifactId>jackson-core</artifactId>
      <version>2.9.0</version>
    </dependency>
    <dependency>
      <groupId>com.fasterxml.jackson.core</groupId>
      <artifactId>jackson-databind</artifactId>
      <version>2.9.0</version>
    </dependency>
    ~~~

* 注解`@ResponseBody`

    该注解可以在方法上和方法返回值上，可以自动将处理器返回值对象转换成json字符串，并响应给请求

    ~~~java
    @RequestMapping("/login.action")
    public @ResponseBody Admin login() throws IOException {
        Admin  admin= new Admin(1, "123", "123");
        return admin;
    }
    ~~~

    前端js：

    ~~~js
    $.post(rootPath+ "/admin/login.action" ,data,function(message){
        alert(message.number+"---"+message.username+"---"+message.password)
    })
    ~~~

可以通过这种方法来响应异步请求

* 注解`@RequestBody`(了解)

    在SpringMVC低版本时，接收前台传递的json数据时，必须在处理器的形参上加上该注解，而且形参必须不能是对象

    现在高版本，不需要该注解也可以进行json解析，并自动将参数值赋给对象属性

**注意1：** 返回值对象中，要传递给前端的属性，必须实现get方法

**注意2：** 返回值对象是枚举时，对应的枚举类对象必须实现`@JsonFormat(shape = JsonFormat.Shape.OBJECT)`注解

# 配置前置后置路径

​		如果直接将`.jsp`文件放在webapp文件夹下，用户就可以直接通过浏览器地址栏访问到它，这会造成安全风险；同时这样直接访问，请求不会走处理器映射器和处理器适配器，这意味着SpringMVC的拦截器不会对该资源生效。

​		为了避免这种情况，可以把`.jsp`文件放到WEB-INF下，WEB-INF不接受直接请求，然后让所有对`.jsp`页面的访问通过处理器转发到。这样用户就只能通过访问处理器，间接访问`.jsp`文件了。

为了方便我们设置处理器的跳转路径，我们可以设置视图解析器的前置后置路径，这样通过视图解析器访问的路径，会自动拼装设置的前缀后缀：

~~~xml
<bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
    <property name="prefix" value="/WEB-INF/"/>
    <property name="suffix" value=".jsp"/>
</bean>
~~~

这样返回字符串或者ModelAndView对象的处理器，返回视图时，视图路径将自动拼装设置的前后缀：

~~~java
//index.jsp在项目下的路径为 webapp/WEB-INF/index.jsp
@RequestMapping("/toIndex.action")
public String toIndex(){
    return "index";
}
//该处理器返回的实际视图路径为 /WEB-INF/index.jsp
~~~

# 文件传输

## 文件上传

在`servlet-api`中，我们学习了文件上传的三要素：

* 依赖包：`commons-fileupload.jar`,`commons-io.jar`
* 前端请求格式：`method="post"`,`enctype="multipart/form-data"`
* servlet配置：需要添加注解`@MultipartConfig`以开启识别Multipart数据结构的配置

现在可以通过maven依赖导入依赖包：

~~~xml
<dependency>
      <groupId>commons-fileupload</groupId>
      <artifactId>commons-fileupload</artifactId>
      <version>1.3.1</version>
    </dependency>
~~~

在SpringMVC框架下，前两个要素不变，servlet 的文件传输配置由SpringMVC框架封装，只需要在核心配置文件中配置文件上传解析器：

**！！注意：**id必须是multipartResolver

~~~xml
<!--配置文件上传解析器-->
<!--id必须是multipartResolver-->
<bean id="multipartResolver" class="org.springframework.web.multipart.commons.CommonsMultipartResolver">
    <property name="maxUploadSize" value="#{1024*1024*80}"/>
    <property name="defaultEncoding" value="utf-8"/>
</bean>
~~~

SpringMVC还封装的servlet 的part对象，提供了MultipartFile文件对象

处理器可以直接通过处理器适配器获取该对象，可以通过该对象更方便地写出上传的文件

前端表单：

~~~html
<form id="file-upload">
    姓名：<input type="text" name="username"><br/>
    <input type="file" name="fileUpload"><br/>
    <input type="button" value="提交" id="submit">
</form>
~~~

js：

~~~js
$("#submit").click(function () {
    var data = new FormData();
    data.append("username",$("input[name='username']").val());
    data.append("fileUpload",$("input[name='fileUpload']")[0].files[0]);
    $.ajax({
        type:"post",
        url:"${pageContext.request.contextPath}/fileUpload.action",
        contentType:false,
        processData:false,
        enctype:"multipart/form-data",
        data:data,
        dataType:"json",
        success:function (message) {
            alert(message.messageStr)
        }
    })
})
~~~

处理器：

~~~java
// 文件上传处理器
@RequestMapping("/fileUpload.action")
@ResponseBody
public Message fileUpload(HttpServletRequest request, String username, MultipartFile fileUpload, Message message) throws IOException {
    System.out.println(username);
    String oldFileName = fileUpload.getOriginalFilename();
    String fileType = oldFileName.substring(oldFileName.lastIndexOf("."));
    String fileName = UUID.randomUUID().toString().replace("-","") +fileType;
    String path = request.getServletContext().getRealPath("/file");
    fileUpload.transferTo(new File(path + "/" + fileName));
    message.setMessageStr("上传成功");
    return message;
}
~~~

**注意：**用上下文获取真实路径时，需要更改项目的output directory到webapp下；

或者配置虚拟路径

## 文件下载

~~~java
@Controller
public class DownController {
    @RequestMapping("down")
    public ResponseEntity<byte[]> down() throws Exception{
        File file = new File("C:\\yf\\test.png");
        //设置响应头为下载
        HttpHeaders headers = new HttpHeaders();
        headers.setContentDispositionFormData("attachment",new String("测试下载.png".getBytes("GBK"),"ISO-8859-1"));
        byte[] bytes = FileUtils.readFileToByteArray(file);
        return new ResponseEntity<byte[]>(bytes, headers, HttpStatus.OK);
    }
}
~~~



# 拦截器

SpringMVC提供了类似Servlet的过滤器的拦截器，拦截器的作用时间在处理器适配器执行处理器之前

配置拦截器：

~~~xml
<!-- 拦截器配置 -->
<mvc:interceptors>
    <!-- 多个拦截器将顺序执行 -->
    <mvc:interceptor>
        <mvc:mapping path="/**"/> <!-- 拦截路径  **代表着多层路径 -->
        <!--<mvc:exclude-mapping path="/login"/>--><!-- 不拦截路径 -->
        <bean class="com.bjpn.inter.LoginInter"></bean>
    </mvc:interceptor>
</mvc:interceptors>
~~~

拦截器：

~~~java
public class LoginInter implements HandlerInterceptor {
    //访问前连接拦截  请求到达拦截器  判断请求是否合理  放行 true   不放行是false
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        System.out.println("这是访问前 ");
        StringBuffer path = request.getRequestURL();
        System.out.println("path:"+path);
        return false;
    }
    //访问后拦截
    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
        //处理器处理完毕  要返回值到前端控制器    可以进行拦截  还可以查看携带的值
    }
   //消亡
    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {

    }
}
~~~





