# SpringMVC Config

对SpringMVC进行配置，以控制SpringMVC的流程行为

SpringMVC提供了两种配置方式

* 使用MVC XML命名空间的进行配置
* 使用MVC Java编码配置

他们首先提供了一套适用于大部分应用程序的默认配置，并允许我们进行自定义配置以覆盖这些默认配置

## MVC XML Configuration

可以使用`<mvc:annotation-driven>`标签以启用xml配置:

~~~xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:mvc="http://www.springframework.org/schema/mvc"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/mvc
       http://www.springframework.org/schema/mvc/spring-mvc.xsd 
       http://www.springframework.org/schema/context 
       https://www.springframework.org/schema/context/spring-context.xsd">
    <mvc:annotation-driven/>
</beans>
~~~

如果不指定`<mvc:annotation-driven>`的属性和子标签，那么SpringMVC将使用默认配置，我们可以指定`mvc:annotation-driven`的子标签和属性以改变默认行为。

`<mvc:annotation-driven>`属性：

* `enable-matrix-variables`:启用解析URL的矩阵变量
* `validator`:指定`DataBinder`在进行数据校验时使用的 `Validator`实例
* `conversion-service`:指定`DataBinder`在进行数据转化和格式化时使用的`ConversionService`
* `content-negotiation-manager`:指定SpringMVC用来确定请求的媒体类型的`ContentNegotiationManager`
* `ignore-default-model-on-redirect`:指定重定向场景中是否启用默认model
* `message-codes-resolver`：指定`DataBinder`在生成`BindingResult`时解析error code所使用的`MessageCodesResolver`

`<mvc:annotation-driven>`子标签：

* `<mvc:argument-resolvers>`指定用于解析自定义参数的`HandlerMethodArgumentResolver`，该配置不会覆盖默认的参数解析器
* `<mvc:async-support>`：异步请求配置
* `<mvc:message-converters>`：配置自定义的` HttpMessageConverter` 
* `<mvc:path-matching>`：自定义与路径匹配和URL处理相关的行为
* `<mvc:return-value-handlers>`：自定义返回值处理器

## MVC Java Configuration

使用`@EnableWebMvc`注解，并实现`WebMvcConfigurer`的普通类，可以实现配置SpringMVC

~~~java
@EnableWebMvc
@Configuration
public class WebConfig implements WebMvcConfigurer {
    //实现WebMvcConfigurer方法以配置SpringMVC
}
~~~

WebMvcConfigurer可配置的方法如下:

~~~java
public interface WebMvcConfigurer {
	//配置PathHelper和PathMatcher
	default void configurePathMatch(PathMatchConfigurer configurer) ;
	//配置ContentNegotiationManager
	default void configureContentNegotiation(ContentNegotiationConfigurer configurer);
	//配置异步支持
	default void configureAsyncSupport(AsyncSupportConfigurer configurer);
	//配置默认Servlet Handler
	default void configureDefaultServletHandling(DefaultServletHandlerConfigurer configurer) ;

	default void addFormatters(FormatterRegistry registry);

	default void addInterceptors(InterceptorRegistry registry) ;

	default void addResourceHandlers(ResourceHandlerRegistry registry);


	default void addCorsMappings(CorsRegistry registry);


	default void addViewControllers(ViewControllerRegistry registry) ;


	default void configureViewResolvers(ViewResolverRegistry registry);


	default void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers);


	default void addReturnValueHandlers(List<HandlerMethodReturnValueHandler> handlers);

	default void configureMessageConverters(List<HttpMessageConverter<?>> converters);

	default void extendMessageConverters(List<HttpMessageConverter<?>> converters);

	default void configureHandlerExceptionResolvers(List<HandlerExceptionResolver> resolvers) ;

	default void extendHandlerExceptionResolvers(List<HandlerExceptionResolver> resolvers);


	@Nullable
	default Validator getValidator();

	default MessageCodesResolver getMessageCodesResolver();
}
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

