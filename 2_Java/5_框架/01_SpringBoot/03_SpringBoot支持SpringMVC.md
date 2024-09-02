# SpringMVC自动配置

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

因为`@EnableWebMvc`注解会引入`DelegatingWebMvcConfiguration`类来完成默认配置。但是Springboot的自动配置已经引入了`DelegatingWebMvcConfiguration`类。除非想要在Springboot中完全控制MVC，否则不需要使用`@EnableWebMvc`注解。

## `HttpMessageConverter`