# 启用 MVC 配置

MVC Java配置和MVC XML命名空间提供了适合大多数应用程序的默认配置，并提供了一个配置API来定制它

在Java配置中，可以使用 `@EnableWebMvc` 注解来启用MVC配置，如下例所示：

~~~java
@Configuration
@EnableWebMvc
public class WebConfig {
}
~~~

在XML配置中，可以使用 `<mvc:annotation-driven>` 元素来启用MVC配置：

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:mvc="http://www.springframework.org/schema/mvc"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/mvc
        https://www.springframework.org/schema/mvc/spring-mvc.xsd">

    <mvc:annotation-driven/>

</beans>
~~~

在Java配置中，你可以实现 `WebMvcConfigurer` 接口:

~~~java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    // Implement configuration methods...
}
~~~

在XML中，你可以检查 `<mvc:annotation-driven/>` 的属性和子元素。

# 类型转换

默认情况下， 已安装各种数字和日期类型的 formatter，同时还支持通过字段上的 `@NumberFormat` 和 `@DateTimeFormat` 。

要使用Java配置注册自定义的 formatter 和 converter，可以：

~~~java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addFormatters(FormatterRegistry registry) {
        // ...
    }
}
~~~

在XML配置中，可以：

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:mvc="http://www.springframework.org/schema/mvc"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/mvc
        https://www.springframework.org/schema/mvc/spring-mvc.xsd">

    <mvc:annotation-driven conversion-service="conversionService"/>

    <bean id="conversionService"
            class="org.springframework.format.support.FormattingConversionServiceFactoryBean">
        <property name="converters">
            <set>
                <bean class="org.example.MyConverter"/>
            </set>
        </property>
        <property name="formatters">
            <set>
                <bean class="org.example.MyFormatter"/>
                <bean class="org.example.MyAnnotationFormatterFactory"/>
            </set>
        </property>
        <property name="formatterRegistrars">
            <set>
                <bean class="org.example.MyFormatterRegistrar"/>
            </set>
        </property>
    </bean>
</beans>

~~~

# 校验

默认情况下，如果 Bean Validation 在classpath上存在（例如Hibernate Validator），`LocalValidatorFactoryBean` 会被注册为全局 Validator，用于校验 `@Valid` 和 `Validated` 的 controller 方法参数。

Java配置自定义全局Validator：

~~~java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public Validator getValidator() {
        // ...
    }
}
~~~

XML配置：

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:mvc="http://www.springframework.org/schema/mvc"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/mvc
        https://www.springframework.org/schema/mvc/spring-mvc.xsd">

    <mvc:annotation-driven validator="globalValidator"/>

</beans>

~~~

也可以注册本地局部的validator：

~~~java
@Controller
public class MyController {

    @InitBinder
    protected void initBinder(WebDataBinder binder) {
        binder.addValidators(new FooValidator());
    }
}
~~~

# 拦截器

使用Java配置拦截器：

~~~java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new LocaleChangeInterceptor());
        registry.addInterceptor(new ThemeChangeInterceptor()).addPathPatterns("/**").excludePathPatterns("/admin/**");
    }
}
~~~

在XML配置中：

~~~xml
<mvc:interceptors>
    <bean class="org.springframework.web.servlet.i18n.LocaleChangeInterceptor"/>
    <mvc:interceptor>
        <mvc:mapping path="/**"/>
        <mvc:exclude-mapping path="/admin/**"/>
        <bean class="org.springframework.web.servlet.theme.ThemeChangeInterceptor"/>
    </mvc:interceptor>
</mvc:interceptors>
~~~

# Content Type

可以配置Spring MVC如何从请求中确定所请求的媒体类型（例如，`Accept` 头、URL路径扩展、查询参数，以及其他）

Java配置中，你可以自定义请求的 content type 解析：

~~~java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void configureContentNegotiation(ContentNegotiationConfigurer configurer) {
        configurer.mediaType("json", MediaType.APPLICATION_JSON);
        configurer.mediaType("xml", MediaType.APPLICATION_XML);
    }
}
~~~

XML配置：

~~~xml
<mvc:annotation-driven content-negotiation-manager="contentNegotiationManager"/>

<bean id="contentNegotiationManager" class="org.springframework.web.accept.ContentNegotiationManagerFactoryBean">
    <property name="mediaTypes">
        <value>
            json=application/json
            xml=application/xml
        </value>
    </property>
</bean>
~~~

# HttpMessageConverter

`HttpMessageConverter`能帮助`RequestMappingHandlerAdapter`：

* 将读取到的Web请求体中的请求体信息转换为对象传递给Handler Method的参数
* 将Handler Method返回的对象转换为响应体信息以写入Web响应体

`@RequestBody`、`@ResponseBody`、`@RequestPart`、`HttpEntity`、`ResponseEntity`背后的实现都需要`HttpMessageConverter`的支持，以实现类型转换和数据绑定

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

你可以通过写 `configureMessageConverters()`（替换Spring MVC创建的默认转换器）或覆写 `extendMessageConverters()`（自定义默认转换器或为默认转换器添加额外的转换器）在Java配置中定制 `HttpMessageConverter`。

下面的例子添加了XML和Jackson JSON转换器，用一个定制的 `ObjectMapper` 代替默认的：

~~~java
@Configuration
@EnableWebMvc
public class WebConfiguration implements WebMvcConfigurer {

    @Override
    public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
        Jackson2ObjectMapperBuilder builder = new Jackson2ObjectMapperBuilder()
                .indentOutput(true)
                .dateFormat(new SimpleDateFormat("yyyy-MM-dd"))
                .modulesToInstall(new ParameterNamesModule());
        converters.add(new MappingJackson2HttpMessageConverter(builder.build()));
        converters.add(new MappingJackson2XmlHttpMessageConverter(builder.createXmlMapper(true).build()));
    }
}
~~~

在前面的例子中， `Jackson2ObjectMapperBuilder`]被用来为 `MappingJackson2HttpMessageConverter` 和 `MappingJackson2XmlHttpMessageConverter` 创建一个共同的配置，其中启用了缩进，自定义了日期格式，并注册了 `jackson-module-parameter-names`，这增加了对访问参数名称的支持（Java 8中增加的一个功能）。

XML配置：

~~~xml
<mvc:annotation-driven>
    <mvc:message-converters>
        <bean class="org.springframework.http.converter.json.MappingJackson2HttpMessageConverter">
            <property name="objectMapper" ref="objectMapper"/>
        </bean>
        <bean class="org.springframework.http.converter.xml.MappingJackson2XmlHttpMessageConverter">
            <property name="objectMapper" ref="xmlMapper"/>
        </bean>
    </mvc:message-converters>
</mvc:annotation-driven>

<bean id="objectMapper" class="org.springframework.http.converter.json.Jackson2ObjectMapperFactoryBean"
      p:indentOutput="true"
      p:simpleDateFormat="yyyy-MM-dd"
      p:modulesToInstall="com.fasterxml.jackson.module.paramnames.ParameterNamesModule"/>

<bean id="xmlMapper" parent="objectMapper" p:createXmlMapper="true"/>
~~~

# View Controller

 `ParameterizableViewController` 可以方便地对请求转发一个视图。MVC配置提供一个方便注册它的配置。

Java配置：

~~~java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addViewControllers(ViewControllerRegistry registry) {
        registry.addViewController("/").setViewName("home");
    }
}
~~~

XML配置：

~~~xml
<mvc:view-controller path="/" view-name="home"/>
~~~

# ViewResolver

MVC配置简化了视图解析器的注册。

下面的Java配置例子通过使用JSP和Jackson作为JSON渲染的默认 `View` 来配置内容协商的视图解析：

~~~java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void configureViewResolvers(ViewResolverRegistry registry) {
        registry.enableContentNegotiation(new MappingJackson2JsonView());
        registry.jsp();
    }
}
~~~

XML配置：

~~~xml
<mvc:view-resolvers>
    <mvc:content-negotiation>
        <mvc:default-views>
            <bean class="org.springframework.web.servlet.view.json.MappingJackson2JsonView"/>
        </mvc:default-views>
    </mvc:content-negotiation>
    <mvc:jsp/>
</mvc:view-resolvers>
~~~

# 静态资源

MVC配置提供基于 Resource 的位置列表中提供静态资源的方法。

在下一个例子中，给定一个以 `/resources` 开头的请求，相对路径被用来寻找和提供相对于Web应用root下的 `/public` 或 `/static` 下classpath的静态资源。这些资源将在浏览器的缓存1年，减少浏览器的HTTP请求：

~~~java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/resources/**")
                .addResourceLocations("/public", "classpath:/static/")
                .setCacheControl(CacheControl.maxAge(Duration.ofDays(365)));
    }
}
~~~

XML配置：

~~~xml
<mvc:resources mapping="/resources/**"
    location="/public, classpath:/static/"
    cache-period="31556926" />

~~~

# Default Servlet

Spring MVC允许将 `DispatcherServlet` 映射到 `/`（从而覆盖容器的默认Servlet的映射），同时仍然允许静态资源请求由容器的默认Servlet处理。它配置了一个 `DefaultServletHttpRequestHandler`，其URL映射为 `/**`，相对于其他URL映射，其优先级最低。

这个 handler 将所有请求转发到默认的Servlet。因此，它必须在所有其他URL `HandlerMappings` 的顺序中保持最后。

Java配置：

~~~java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void configureDefaultServletHandling(DefaultServletHandlerConfigurer configurer) {
        configurer.enable();
    }
}
~~~

XML配置：

~~~xml
<mvc:default-servlet-handler/>
~~~

# 高级Java配置

`@EnableWebMvc` 导入 `DelegatingWebMvcConfiguration`，它：

- 为Spring MVC应用程序提供默认的Spring配置。
- 检测并委托给 `WebMvcConfigurer` 实现来定制该配置。

可以去掉 `@EnableWebMvc`，直接从 `DelegatingWebMvcConfiguration` 扩展，而不是实现 `WebMvcConfigurer`：

~~~java
@Configuration
public class WebConfig extends DelegatingWebMvcConfiguration {

    // ...
}


~~~

# 高级 XML 配置

MVC命名空间没有高级模式。如果你需要在Bean上定制一个你无法以其他方式改变的属性，你可以使用Spring `ApplicationContext` 的 `BeanPostProcessor` 生命周期钩子：

~~~java
@Component
public class MyPostProcessor implements BeanPostProcessor {

    public Object postProcessBeforeInitialization(Object bean, String name) throws BeansException {
        // ...
    }
}
~~~
