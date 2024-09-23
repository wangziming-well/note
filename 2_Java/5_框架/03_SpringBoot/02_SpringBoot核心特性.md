# SpringApplication

通过 `SpringApplication` 类，可以从 `main()` 方法中启动Spring应用程序。 在许多情况下，可以直接调动 `SpringApplication.run` 静态方法，例如：

~~~java
@SpringBootApplication
public class MyApplication {

    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }

}
~~~

## 懒初始化

`SpringApplication` 允许应用程序被懒初始化。 当启用懒初始化时，Bean在需要时被创建，而不是在应用程序启动时。 因此，懒初始化可以减少应用程序的启动时间。 在一个Web应用程序中，启用懒初始化后将导致许多与Web相关的Bean在收到HTTP请求之后才会进行初始化。

懒初始化的一个缺点是它会延迟发现应用程序的问题。 如果一个配置错误的Bean被懒初始化了，那么在启动过程中就不会再出现故障，问题只有在Bean被初始化时才会显现出来。 还必须注意确保JVM有足够的内存来容纳应用程序的所有Bean，而不仅仅是那些在启动期间被初始化的Bean。 由于这些原因，默认情况下不启用懒初始化，建议在启用懒初始化之前，对JVM的堆大小进行微调。

可以使用 `SpringApplicationBuilder` 的 `lazyInitialization` 方法或 `SpringApplication` 的 `setLazyInitialization` 方法以编程方式启用懒初始化。 另外，也可以使用 `spring.main.lazy-initialization` 属性来启用，如下例所示：

~~~Properties
spring.main.lazy-initialization=true
~~~

## 自定义 Banner

启动时打印的Banner可以通过在 classpath 中添加 `banner.txt` 文件或通过将 `spring.banner.location` 属性设置为该文件的位置来自定义。 如果该文件的编码不是UTF-8，可以通过 `spring.banner.charset` 属性设置其字符编码

在 `banner.txt` 文件中，可以使用 `Environment` 中的任何key，以及以下任何占位符

| 变量                                                         | 介绍                                                         |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| `${application.version}`                                     | 应用程序的版本号，也就是 `MANIFEST.MF` 中声明的。 例如，`Implementation-Version: 1.0` 被打印为 `1.0`。 |
| `${application.formatted-version}`                           | 应用程序的版本号，如在`MANIFEST.MF`中声明的那样，并以格式化显示（用括号包围，以 `v` 为前缀）。 例如 `(v1.0)`。 |
| `${spring-boot.version}`                                     | 使用的Spring Boot版本。 例如 `3.2.0-SNAPSHOT` 。             |
| `${spring-boot.formatted-version}`                           | 正在使用的Spring Boot版本，格式化显示（用大括号包围并以 `v` 为前缀）。 例如 `(v3.2.0-SNAPSHOT)`。 |
| `${Ansi.NAME}` (或 `${AnsiColor.NAME}`, `${AnsiBackground.NAME}`, `${AnsiStyle.NAME}`) | 其中 `NAME` 是一个ANSI转义代码的名称                         |
| `${application.title}`                                       | 应用程序的标题，正如在 `MANIFEST.MF` 中声明的那样。 例如， `Implementation-Title: MyApp` 被打印成 `MyApp`。 |

如果想以编程方式生成一个Banner，可以使用 `SpringApplication.setBanner(..)` 方法。 实现 `org.springframework.boot.Banner` 接口并实现自己的 `printBanner()` 方法

可以使用 `spring.main.banner-mode` 属性来决定Beann打印模式。例如打印到 `System.out`（`console`）上，发送到配置的logger（`log`），或者根本不打印（`off`）

打印的Banner被注册为一个单例Bean，名字是：`springBootBanner`

## 配置SpringApplication

如果 `SpringApplication` 的默认值不符合的需求，可以创建一个实例并对其进行自定义。 例如，要关闭Banner，可以：

~~~java
@SpringBootApplication
public class MyApplication {

    public static void main(String[] args) {
        SpringApplication application = new SpringApplication(MyApplication.class);
        application.setBannerMode(Banner.Mode.OFF);
        application.run(args);
    }

}
~~~

也可以通过使用 `application.properties` 文件来配置 `SpringApplication`

## 事件

除了Spring的ApplicationContent提供的事件发布监听功能，SpringApplication 还会发布一些额外的应用事件。

有些事件实际上是在 `ApplicationContext` 被创建之前触发的，所以不能以 `@Bean` 的形式注册一个监听器。 可以通过 `SpringApplication.addListeners(…)` 方法或 `SpringApplicationBuilder.listeners(…)` 方法注册它们。

如果想让这些监听器自动注册，不管应用程序是如何创建的，可以在项目中添加一个 `META-INF/spring.factories` 文件，并通过 `org.springframework.context.ApplicationListener` 属性来配置监听器，如以下例子所示。

```properties
org.springframework.context.ApplicationListener=com.example.project.MyListener
```

当应用程序运行时，Application event按以下顺序发布。

1. 一个 `ApplicationStartingEvent` 在运行开始时被发布，但在任何处理之前，除了注册监听器和初始化器之外。
2. 当在上下文中使用的 `Environment` 已知，但在创建上下文之前，将发布 `ApplicationEnvironmentPreparedEvent`。
3. 当 `ApplicationContext` 已准备好并且 `ApplicationContextInitializers` 被调用，但在任何Bean定义被加载之前，`ApplicationContextInitializedEvent` 被发布。
4. 一个 `ApplicationPreparedEvent` 将在刷新开始前但在Bean定义加载后被发布。
5. 在上下文被刷新之后，但在任何应用程序和命令行运行程序被调用之前，将发布一个 `ApplicationStartedEvent`。
6. 紧接着发布 `LivenessState.CORRECT` 状态的 `AvailabilityChangeEvent`，表明应用程序被认为是存活的。
7. 在任何ApplicationRunner 和 CommandLineRunner调用后，将发布一个 `ApplicationReadyEvent`。
8. 紧接着发布 `ReadinessState.ACCEPTING_TRAFFIC` 状态的 `AvailabilityChangeEvent`，表明应用程序已经准备好为请求提供服务。
9. 如果启动时出现异常，将发布一个 `ApplicationFailedEvent`。

以上列表仅包括与 `SpringApplication` 相关的 `SpringApplicationEvent`。 除此以外，以下事件也会在 `ApplicationPreparedEvent` 之后和 `ApplicationStartedEvent` 之前发布。

- 在 `WebServer` 准备好后发布 `WebServerInitializedEvent`。 `ServletWebServerInitializedEvent` 和 `ReactiveWebServerInitializedEvent` 分别对应Servlet和reactive的实现。
- 当 `ApplicationContext` 被刷新时，将发布一个 `ContextRefreshedEvent`。

## Web环境

`SpringApplication` 会试图创建正确类型的 `ApplicationContext`。 确定为 `WebApplicationType` 的算法如下。

- 如果Spring MVC存在，就会使用 `AnnotationConfigServletWebServerApplicationContext`。
- 如果Spring MVC不存在而Spring WebFlux存在，则使用 `AnnotationConfigReactiveWebServerApplicationContext`。
- 否则，将使用 `AnnotationConfigApplicationContext`。

这意味着，如果在同一个应用程序中使用Spring MVC和新的 `WebClient`（来自于Spring WebFlux），Spring MVC将被默认使用。 可以通过调用 `setWebApplicationType(WebApplicationType)` 来轻松覆盖。

也可以通过调用 `setApplicationContextFactory(…)` 来完全控制使用的 `ApplicationContext` 类型。

## 访问应用参数

如果需要访问传递给 `SpringApplication.run(..)` 的命令行参数，你可以注入一个 `org.springframework.boot.ApplicationArguments` bean。 通过 `ApplicationArguments` 接口，你可以访问原始的 `String[]` 参数以及经过解析的 `option` 和 `non-option` 参数，例如：

~~~java
@Component
public class MyBean {
    public MyBean(ApplicationArguments args) {
        String[] sourceArgs = args.getSourceArgs();
        System.out.println(Arrays.toString(sourceArgs));
    }
}

~~~

Spring Boot还在Spring的 `Environment` 中注册了一个 `CommandLinePropertySource`。 所以也可以通过使用 `@Value` 注解来注入单个应用参数

## 使用 ApplicationRunner 或 CommandLineRunner

如果需要在SpringApplication启动后运行一些特定的代码，那么可以在容器中注册 `ApplicationRunner` 或 `CommandLineRunner` 接口的实现。 这两个接口以相同的方式工作，并提供一个单一的 `run` 方法，该方法在 `SpringApplication.run(…)` 执行完毕之前被调用。

`CommandLineRunner` 接口以字符串数组形式提供了对应用程序参数（启动参数）的访问。而 `ApplicationRunner` 使用前面讨论的 `ApplicationArguments` 接口。下面以`CommandLineRunner`为例：

~~~java
@Component
public class MyCommandLineRunner implements CommandLineRunner {

    @Override
    public void run(String... args) {
        System.out.println(Arrays.toString(args));
    }

}
~~~

## 程序退出

每个 `SpringApplication` 都向JVM注册了一个shutdown hook，以确保 `ApplicationContext` 在退出时优雅地关闭。除了可以使用所有标准的Spring生命周期回调（如 `DisposableBean` 接口或 `@PreDestroy` 注解）外，如果Bean希望在调用 `SpringApplication.exit()` 时返回特定的退出代码，可以实现 `org.springframework.boot.ExitCodeGenerator` 接口。 然后，这个退出代码可以被传递给 `System.exit()` ，将其作为状态代码返回，例如：

~~~java
@SpringBootApplication
public class StartApplication {

    @Bean
    public ExitCodeGenerator exitCodeGenerator() {
        return () -> 42;
    }

	public static void main(String[] args) {
        System.exit(SpringApplication.exit(SpringApplication.run(StartApplication.class,args)));
    }

}
~~~

另外，`ExitCodeGenerator` 接口可以由异常（Exception）实现。 当遇到这种异常时，Spring Boot会返回由实现的 `getExitCode()` 方法提供的退出代码

如果有多个 `ExitCodeGenerator` ，则使用第一个生成的非零退出代码。 要控制生成器（Generator）的调用顺序，你可以实现 `org.springframework.core.Ordered` 接口或使用 `org.springframework.core.annotation.Order` 注解

# 外部化配置

Spring Boot可以让配置外部化，这样就可以在不同的环境中使用相同的应用程序代码。 可以使用各种外部配置源，包括Java properties 文件、YAML文件、环境变量和命令行参数。

属性值可以通过使用 `@Value` 注解直接注入到Bean，也可以通过Spring 的 `Environment` 访问，或者通过 `@ConfigurationProperties`绑定到对象

Spring Boot 使用一个非常特别的 `PropertySource` 顺序，旨在允许合理地重写值。 后面的 property source 可以覆盖前面属性源中定义的值。 按以下顺序考虑：

1. 默认属性（通过 `SpringApplication.setDefaultProperties` 指定）。
2. `@Configuration` 类上的 `@PropertySource` 注解。请注意，这样的属性源直到application context被刷新时才会被添加到环境中。这对于配置某些属性来说已经太晚了，比如 `logging.*` 和 `spring.main.*` ，它们在刷新开始前就已经被读取了。
3. 配置数据（如 `application.properties` 文件）。
4. `RandomValuePropertySource`，它只有 `random.*` 属性。
5. 操作系统环境变量
6. Java System properties (`System.getProperties()`).
7. `java:comp/env` 中的 JNDI 属性。
8. `ServletContext` init parameters.
9. `ServletConfig` init parameters.
10. 来自 `SPRING_APPLICATION_JSON` 的属性（嵌入环境变量或系统属性中的内联JSON）。
11. 命令行参数
12. 你在测试中的 `properties` 属性。在 `@SpringBootTest` 和测试注解中可用，用于测试你的应用程序的一个特定片断
13. `@DynamicPropertySource`注解在你的测试中。
14. 你测试中的`@TestPropertySource`]注解.
15. 当devtools处于活动状态时，`$HOME/.config/spring-boot` 目录下的Devtools全局设置属性

第三的配置数据文件按照以下顺序读取：

1. 在你的jar中打包的Application properties（application.properties 和 YAML）。
2. 在你的jar中打包的 特定的 Profile application properties（`application-{profile}.properties` 和 YAML）。
3. 在你打包的jar之外的Application properties性（application.properties和YAML）。
4. 在你打包的jar之外的特定的 Profile application properties（ `application-{profile}.properties` 和YAML）

建议在整个应用程序中坚持使用一种格式。如果同时有 `.properties` 和YAML格式的配置文件，那么 `.properties` 优先

## 访问命令行属性

默认情况下，`SpringApplication` 会将任何命令行选项参数（即以 `--` 开头的参数，如 `--server.port=9000` ）转换为 `property` 并将其添加到Spring `Environment` 中。 如前所述，命令行属性总是优先于基于文件的属性源。

如果不希望命令行属性被添加到 `Environment` 中，你可以通过 `SpringApplication.setAddCommandLineProperties(false)` 禁用它们。

## 核心配置文件

当你的应用程序启动时，Spring Boot会自动从以下位置找到并加载 `application.properties` 和 `application.yaml` 文件。

1. classpath
   1. classpath 根路径
   2. classpath 下的 `/config` 包
2. 当前目录(`System.getProperty("user.dir")`)
   1. 当前目录下
   2. 当前目录下的 `config/` 子目录
   3. `config/` 子目录的直接子目录

列表按优先级排序（较晚项目的值覆盖较早项目的值）。加载的文件被作为 `PropertySources` 添加到Spring的 `Environment` 中

`application.properties` 和 `application.yaml` 是默认的配置文件名，可以通过指定 `spring.config.name` 环境属性设置自定义的配置文件名称。如：

~~~shell
$ java -jar myproject.jar --spring.config.name=myproject
~~~

也可以通过使用 `spring.config.location` 环境属性来引用一个明确的位置。 该属性接受一个逗号分隔的列表，其中包含一个或多个要检查的位置：

~~~shell
$ java -jar myproject.jar --spring.config.location=\
    optional:classpath:/default.properties,\
    optional:classpath:/override.properties
~~~

`spring.config.location` 属性需要注意以下几点：

*  `optional:` 前缀表示配置文件是可选的，并且可以不存在

* 如果 `spring.config.location` 包含目录（而不是文件），它们应该以 `/` 结尾。 在运行时，它们将被附加上由 `spring.config.name` 生成的名称，然后被加载。 在 `spring.config.location` 中指定的文件被直接导入。
* 目录和文件位置值也被扩展，以检查特定的配置文件。例如，如果 `spring.config.location` 是 `classpath:myconfig.properties`，那么适当的 `classpath:myconfig-<profile>.properties` 文件被加载
* 添加的每个 `spring.config.location` 项将引用一个文件或目录。 位置是按照它们被定义的顺序来处理的，后面的位置可以覆盖前面的位置的值
* `spring.config.location` 配置的位置会取代默认位置，如果只是想添加额外的位置而不是取代默认位置，可以设置 `spring.config.extra-location` 

`spring.config.name`, `spring.config.location`, 和 `spring.config.extra-location` 很早就用来确定哪些文件必须被加载。 它们必须被定义为环境属性（通常是操作系统环境变量，系统属性，或命令行参数）

### 特定文件(profiles)

除了`application`属性文件外，SpringBoot还会尝试使用 `application-{profile}` 的命名惯例加载profile特定的文件。

例如当前应用程序激活了名为`prod`的配置文件(`spring.profiles.active=prod`),那么`application-prod.properties`也会被加载。

特定文件(`profiles`)的属性与标准的 `application.properties` 的位置相同，特定文件总是优先于非特定文件。

 如果指定了几个配置文件，则采用最后胜出的策略。 例如对于 `spring.profiles.active=prod,live` ，那么`application-prod.properties` 中的值可以被 `application-live.properties` 中的值所覆盖。

`Environment` 有一组默认的配置文件（默认为 `[default]` ），如果没有设置活动的配置文件，就会使用这些配置文件。 换句话说，如果没有明确激活的配置文件，那么就会考虑来自 `application-default` 的属性。

### 特定配置下加载

现在我们可以使用`spring.profiles.active`激活特定的配置文件，这样可以用来隔离应用程序配置，使其仅在特定的环境中可以用。

而`@Profile`注解可以注释在任意 `@Component`、`@Configuration` 或 `@ConfigurationProperties` 类上，用来指示当前类仅在特定的profiles活动时才被加载，例如：

~~~java
@Configuration(proxyBeanMethods = false)
@Profile("production")
public class ProductionConfiguration {

    // ...

}
~~~

上边的`ProductionConfiguration`类只有在`spring.profiles.active`属性中有production时才会被加载

**注意**：如果 `@ConfigurationProperties` Bean是通过 `@EnableConfigurationProperties` 注册的，而不是自动扫描，则需要在具有 `@EnableConfigurationProperties` 注解的 `@Configuration` 类上指定 `@Profile` 注解。 在 `@ConfigurationProperties` 被扫描的情况下，`@Profile` 可以在 `@ConfigurationProperties` 类本身指定。



### 导入额外配置

application properties 中可以使用 `spring.config.import` 属性从其他地方导入更多的配置数据。 导入在被发现时被处理，并被视为紧接着声明导入的文件下面插入的额外文件。

例如，在 classpath `application.properties` 文件中有以下内容：

~~~properties
spring.application.name=myapp
spring.config.import=optional:file:./dev.properties
~~~

这将触发导入当前目录下的 `dev.properties` 文件（如果存在这样的文件）。 导入的 `dev.properties` 中的值将优先于触发导入的文件。 在上面的例子中，`dev.properties` 可以将 `spring.application.name` 重新定义为一个不同的值。

一个导入只会被导入一次，无论它被声明多少次。 一个导入在properties/yaml文件内的单个文件中被定义的顺序并不重要。 例如，下面的两个例子产生相同的结果。

~~~properties
spring.config.import=my.properties
my.property=value
~~~

和：

~~~properties
my.property=value
spring.config.import=my.properties
~~~

在一个单一的 `spring.config.import` 属性下可以指定多个位置。 位置将按照它们被定义的顺序被处理，后来的导入将被优先处理。

### 占位符

`application.properties` 和 `application.yaml` 中的值在使用时通过现有的 `Environment` 过滤，所以可以通过占位符引用以前定义的值（例如，来自系统属性或环境变量）。 标准的 `${name}` 属性占位符语法可以用在一个值的任何地方。 属性占位符也可以指定一个默认值，使用 `:` 来分隔默认值和属性名称，例如 `${name:default}`

例如：

~~~properties
app.name=MyApp
app.description=${app.name} is a Spring Boot application written by ${username:Unknown}
~~~

### 配置随机值

`RandomValuePropertySource`可以在配置的时候注入随机值。它可以产生Integer、Long、UUID，或String，例如：

~~~properties
my.secret=${random.value}
my.number=${random.int}
my.bignumber=${random.long}
my.uuid=${random.uuid}
my.number-less-than-ten=${random.int(10)}
my.number-in-range=${random.int[1024,65536]}
~~~

## 类型安全的配置属性绑定

可以使用 `@Value("${property}")` 注解来注入配置属性，但是当要处理多个属性或者数据是分层的时候，使用`@Value`绑定数据会很繁琐。

Springboot提供了一种替代方法，让强类型的Bean管理和验证你的应用程序的配置。

### JavaBean 属性绑定

可以使用`@ConfigurationProperties`注释类，以让配置属性绑定到bean上，例如：

~~~java
@Data
@ConfigurationProperties("my.service")
public class MyProperties {

    private boolean enabled;

    private InetAddress remoteAddress;

    private final Security security = new Security();

    @Data
    public static class Security {

        private String username;

        private String password;

        private List<String> roles = new ArrayList<>(Collections.singleton("USER"));

    }
}
~~~

这样的POJO能够绑定下面配置属性：

- `my.service.enabled`，默认值为`false`。
- `my.service.remote-address`，其类型可由`String`强制提供。
- `my.service.security`:security属性是一个对象而非简单类型，所以这种数据绑定会再扫描嵌套的Security对象：
  - `my.service.security.username`
  - `my.service.security.password`
  - `my.service.security.role`，有一个 `String` 的集合，默认为 `USER`。

需要注意Springboot是通过getter/setter来实现数据绑定的，所以被`@ConfigurationProperties`注释的类的属性通常必须有对应的getter/setter，以保证能够正常绑定。

### 构造函数绑定

上一个例子可以重写为：

~~~java
@ConfigurationProperties("my.service")
public class MyProperties {

    // fields...

    public MyProperties(boolean enabled, InetAddress remoteAddress, Security security) {
        this.enabled = enabled;
        this.remoteAddress = remoteAddress;
        this.security = security;
    }

    // getters...

    public static class Security {

        // fields...

        public Security(String username, String password, @DefaultValue("USER") List<String> roles) {
            this.username = username;
            this.password = password;
            this.roles = roles;
        }

        // getters...

    }

}
~~~

此时SpringBoot会使用提供的构造参数来进行配置属性的注入。

如果有多个构造函数，可以使用 `@ConstructorBinding` 注解来指定使用哪个构造函数进行构造函数绑定

构造函数绑定类的嵌套成员（如上面例子中的 `Security`）也将通过其构造函数被绑定。

### 启用` @ConfigurationProperties `类

可以看到`@ConfigurationProperties`注解并没有类似`@Component`或者`@Configuration`的注解，这意味着只有`@ConfiurationProperties`注解的类并不会被注册为容器的bean。

所以需要再逐个类的记住上启用配置属性，或者启用属性配置扫描。

#### `@EnableConfigurationProperties`

`@EnableConfigurationProperties`注解可以注释在任意的 `@Configuration` 类上，它可以启用指定的要处理的`@ConfigurationProperties`类，例如：

~~~java
@Configuration(proxyBeanMethods = false)
@EnableConfigurationProperties(SomeProperties.class)
public class MyConfiguration {

}
~~~

以上代码可以启用下面配置属性类：

~~~java
@ConfigurationProperties("some.properties")
public class SomeProperties {

}
~~~

####  `@ConfigurationPropertiesScan` 

如果要扫描启用多个配置属性类，可以在任意 `@Configuration` 类上添加 `@ConfigurationPropertiesScan` 注解，但是通常我们在`@SpringBootApplication`类上使用它。

默认情况下，注解会注解所在的包，你如果想自定义扫描其他包，可以参考如下代码：

~~~java
@SpringBootApplication
@ConfigurationPropertiesScan({ "com.example.app", "com.example.another" })
public class MyApplication {

}
~~~

### 第三方配置

 `@ConfigurationProperties`除了可以注释一个类，还可以注释在公共的 `@Bean` 方法上。 当你想把属性绑定到你控制之外的第三方组件时，这样做特别有用。

要从 `Environment` 属性中配置一个Bean，可以在其Bean注册方法上添加 `@ConfigurationProperties` ，如下例所示。

~~~java
@Configuration(proxyBeanMethods = false)
public class ThirdPartyConfiguration {

    @Bean
    @ConfigurationProperties(prefix = "another")
    public AnotherComponent anotherComponent() {
        return new AnotherComponent();
    }

}
~~~

### 宽松绑定

Spring Boot在将 `Environment` 属性绑定到 `@ConfigurationProperties` bean时使用了一些宽松的规则，因此 `Environment` 属性名称和bean属性名称之间不需要完全匹配。这样`context-path`可以绑定到`contextPath`，`PORT`可以绑定到`port`

考虑以下 `@ConfigurationProperties` 类：

~~~java
@ConfigurationProperties(prefix = "my.main-project.person")
@Data
public class MyPersonProperties {

    private String firstName;

}
~~~

对以上代码来说，下面的配置属性名称都是可用的：

| 属性名                              | 说明                                                         |
| :---------------------------------- | :----------------------------------------------------------- |
| `my.main-project.person.first-name` | Kebab 风格（短横线隔开），建议在 `.properties` 和 YAML 文件中使用。 |
| `my.main-project.person.firstName`  | 标准的驼峰语法。                                             |
| `my.main-project.person.first_name` | 下划线，这是一种用于 `.properties` 和 YAML 文件的替代格式。  |
| `MY_MAINPROJECT_PERSON_FIRSTNAME`   | 大写格式，在使用系统环境变量时建议使用大写格式。             |



