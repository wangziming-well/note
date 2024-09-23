## SpringMVC自动配置

Spring Boot为Spring MVC提供了自动配置功能，自动配置在Spring的默认值基础上增加了以下功能。

- 包含了 `ContentNegotiatingViewResolver` 和 `BeanNameViewResolver` Bean。
- 支持为静态资源提供服务，包括对WebJars的支持
- 自动注册 `Converter`、`GenericConverter` 和 `Formatter` Bean。
- 支持 `HttpMessageConverters`
- 自动注册 `MessageCodesResolver`
- 支持静态的 `index.html`。
- 自动使用 `ConfigurableWebBindingInitializer` bean

在原生SpringMVC中，配置MVC需要使用类型为 `WebMvcConfigurer`的`@Configuration`类，并且使用`@EnableWebMvc`注解来开启MVC配置。

但在Springboot中，配置MVC只需要使用类型为 `WebMvcConfigurer`的`@Configuration`类即可，不需要使用`@EnableWebMvc`注解。

因为`@EnableWebMvc`注解会引入`DelegatingWebMvcConfiguration`类来完成默认配置。但是Springboot的自动配置已经引入了`DelegatingWebMvcConfiguration`类。所以在Springboot项目中使用`@EnableWebMvc`配置会覆盖springboot的mvc自动配置。  除非想要在Springboot中完全控制MVC，否则不建议使用`@EnableWebMvc`注解。

## `HttpMessageConverter`

Spring MVC使用 `HttpMessageConverter` 接口来转换HTTP请求和响应。

可以向容器中注册`HttpMessageConverters`类，以添加定制转换器，例如：

~~~java
@Configuration(proxyBeanMethods = false)
public class MyHttpMessageConvertersConfiguration {

    @Bean
    public HttpMessageConverters customConverters() {
        HttpMessageConverter<?> additional = new AdditionalHttpMessageConverter();
        HttpMessageConverter<?> another = new AnotherHttpMessageConverter();
        return new HttpMessageConverters(additional, another);
    }

}
~~~

容器中的`HttpMessageConverters`会被MVC自动配置发现并添加到可用`HttpMessageConverter`列表中

## `MessageCodesResolver`

SpringMVC使用`MessageCodesResolver`生成错误码，可以设置属性 `spring.mvc.message-codes-resolver-format` 为 `PREFIX_ERROR_CODE` 或 `POSTFIX_ERROR_CODE`来设置默认吗的实例，（见 `DefaultMessageCodesResolver.Format`的枚举）

## 静态资源映射

默认情况下，Springboot将从`spring.web.resources.staticLocations`默认配置的下面路径中提供静态资源：

* `classpath:/resources/`
* `classpath:/META-INF/resources/`

* `classpath:/static/`
* `classpath:/public/`

这些静态资源默认将被映射到`/**`路径下(由`spring.mvc.static-path-pattern`属性决定)

例如对于项目中的静态资源`/static/demo.png`，可以使用URI：上下路径+`/demo.png`进行访问。

如果设置属性为：

~~~properties
spring.mvc.static-path-pattern = /resources/**
~~~

那么静态资源`/static/demo.png`就需要使用URI：上下路径+`/resources/demo.png`进行访问。

如果spring无法处理某个请求，那么这个请求会由web容器的default servlet来处理。可以设置下面属性为springboot的嵌入式容器注册一个default servlet 来作为静态资源处理的后备：

~~~properties
server.servlet.register-default-servlet = true
~~~

## 欢迎页面

Springboot通过注册一个`WelcomePageHandlerMapping`到`DispatcherServelt`提供了欢迎页面的支持。

当客户端访问根路径时，`WelcomePageHandlerMapping`会将该请求映射到一个`ParameterizableViewController`处理器，该处理器返回一个`forward:index.html`或者`index`的视图名。

所以静态资源中的`index.html`或者`index`模板将作为欢迎页面。

## 自定义 Favicon

Spring Boot检查配置的静态内容位置中是否有 `favicon.ico`。 如果存在这样的文件，它就会自动作为应用程序的favicon。

## 内容协商

可以通过下面配置内容协商：

~~~properties
spring.mvc.contentnegotiation.favor-parameter=true
spring.mvc.contentnegotiation.parameter-name=myparam
spring.mvc.contentnegotiation.media-types.markdown=text/markdown
~~~

以上配置相当于下面代码：

~~~java
@Configuration
public class AppConfig implements WebMvcConfigurer {
    @Override
    public void configureContentNegotiation(ContentNegotiationConfigurer configurer) {
        configurer.favorParameter(true)
                .parameterName("myparam")
                .useRegisteredExtensionsOnly(true)
            .mediaType("markdown", MediaType.TEXT_MARKDOWN);
    }
}
~~~

## `ConfigurableWebBindingInitializer`

Spring MVC 的`WebDataBinderFactory`使用`WebBindingInitializer`为特定请求初始化`WebDataBinder`.

Springboot会扫描容器中的``ConfigurableWebBindingInitializer` ,并使用它。

## 模板引擎

Spring MVC支持各种模板技术，包括Thymeleaf、FreeMarker和JSP。 此外，许多其他模板引擎也包括它们自己的Spring MVC集成。

Spring Boot包括对以下模板引擎的自动配置支持。

* FreeMarker

* Groovy

* Thymeleaf

* Mustache

使用这些模板引擎的默认配置时，模板会自动从 `src/main/resources/templates `中获取

## Error处理

当`DispatcherServlet`抛出未经处理的异常时，例如404，或者其他运行时异常，这个异常最终会由Web容器处理。

### web容器的默认ErrorPage

springboot会向所在的嵌入式web容器中注册一个errorPage，这个errorPage会在请求响应错误码时，将请求重定向到指定的错误处理页面，springboot相当于进行了下面`web.xml`配置：

~~~xml
<error-page>
    <location>${server.error.path}</location>
</error-page>
~~~

其中`server.error.path`为springboot配置属性，默认为`/error`

所有类型的错误码和异常都会被重定向到`/error`

### `BasicErrorController`

同时springboot还会注册一个`BasicErrorController`，`server.error.path`指定的路径会映射到该`Controller`，用于处理全局异常：

* 对于机器客户端，它产生一个JSON响应，包含错误的细节、HTTP状态和异常消息。 
* 对于浏览器客户端，有一个 “whitelabel” error view，以HTML格式显示相同的数据

### 在DispacherServlet内捕获所有异常

可以使用`@ControllerAdvice`类的 `@ExceptionHandler`方法捕获所有异常，在其中进行全局异常处理，这样错误的请求就会被直接处理，而不需要触发到`/error`的重定向。例如：

~~~java
@ControllerAdvice(basePackageClasses = SomeController.class)
public class MyControllerAdvice extends ResponseEntityExceptionHandler {

    @ResponseBody
    @ExceptionHandler(MyException.class)
    public ResponseEntity<?> handleControllerException(HttpServletRequest request, Throwable ex) {
        HttpStatus status = getStatus(request);
        return new ResponseEntity<>(new MyErrorBody(status.value(), ex.getMessage()), status);
    }

    private HttpStatus getStatus(HttpServletRequest request) {
        Integer code = (Integer) request.getAttribute(RequestDispatcher.ERROR_STATUS_CODE);
        HttpStatus status = HttpStatus.resolve(code);
        return (status != null) ? status : HttpStatus.INTERNAL_SERVER_ERROR;
    }

}
~~~

如果 `MyException` 被定义在与 `SomeController` 相同的包中的控制器抛出，那么就会使用 `MyErrorBody` POJO的JSON表示



# 



