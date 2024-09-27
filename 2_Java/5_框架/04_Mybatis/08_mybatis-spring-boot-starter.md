MyBatis-Spring-Boot-Starter 将提供了mybatis对springboot的集成。让我们在springboot应用上更方便地构建mybatis

## 依赖

需要在`pom.xml`文件中天机下面依赖：

~~~xml
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>3.0.2</version>
</dependency>
~~~

## 快速开始

我们已经知道，要在spring中使用mybatis，那么至少需要一个`SqlSessionFactory`和mapper接口。

而MyBatis-Spring-Boot-Starter 会完成下面动作：

* 自动探测已存在的的 `DataSource`
* 使用 `SqlSessionFactoryBean` 创建并注册一个 `SqlSessionFactory` 实例，并将探测到的 `DataSource` 作为数据源
* 创建并注册一个从 `SqlSessionFactory` 中得到的 `SqlSessionTemplate` 的实例
* 自动扫描 mapper，将它们与 `SqlSessionTemplate` 相关联，并将它们注册到Spring 容器中

如果有下面`Mapper`:

~~~java
@Mapper
public interface CityMapper {

  @Select("SELECT * FROM CITY WHERE state = #{state}")
  City findByState(@Param("state") String state);

}
~~~

那么在springboot中就可以直接使用这个mapper:

~~~java
@SpringBootApplication
public class SampleMybatisApplication implements CommandLineRunner {

  private final CityMapper cityMapper;

  public SampleMybatisApplication(CityMapper cityMapper) {
    this.cityMapper = cityMapper;
  }

  public static void main(String[] args) {
    SpringApplication.run(SampleMybatisApplication.class, args);
  }

  @Override
  public void run(String... args) throws Exception {
    System.out.println(this.cityMapper.findByState("CA"));
  }

}
~~~

这就是需要做的所有事情了

## springboot中mapper的自动发现

MyBatis-Spring-Boot-Starter 将默认搜寻带有 `@Mapper` 注解的 mapper 接口。

如果需要指定一个自定义的注解或接口来进行扫描，就必须使用 `@MapperScan` 注解了。

如果Mapper接口需要关联的Mapper XML文件。在springboot中，可以用下面方式关联：

* 让Mapper XML文件和Mapper接口处于同一类路径下，且它们的名字相同。这样mybatis-spring 的`MapperFactoryBean`会将它们自动关联
* 使用`mybatis.mapper-location`指定XMLmapper文件位置
* 使用`config-location`指定mybatis的配置文件，在配置文件中指定mapper配置文件位置

更多配置项后续会详细介绍

## 配置

MyBatis-Spring-Boot-Starter提供了外部配置属性(详见`org.mybatis.spring.boot.autoconfigure.MybatisProperties` )

可以直接在springboot应用的`application.yml`或者`application.properties`中配置

可用的配置项如下，以`mybatis.`作为前缀：

| 配置项（properties）                     | 描述                                                         |
| :--------------------------------------- | :----------------------------------------------------------- |
| `config-location`                        | MyBatis XML 配置文件的路径。                                 |
| `check-config-location`                  | 指定是否对 MyBatis XML 配置文件的存在进行检查。              |
| `mapper-locations`                       | XML 映射文件的路径。                                         |
| `type-aliases-package`                   | 搜索类型别名的包名。（包使用的分隔符是 “`,; \t\n`”）         |
| `type-aliases-super-type`                | 用于过滤类型别名的父类。如果没有指定，MyBatis会将所有从 `type-aliases-package` 搜索到的类作为类型别名处理。 |
| `type-handlers-package`                  | 搜索类型处理器的包名。（包使用的分隔符是 “`,; \t\n`”）       |
| `executor-type`                          | SQL 执行器类型： `SIMPLE`, `REUSE`, `BATCH`                  |
| `default-scripting-language-driver`      | 默认的脚本语言驱动（模板引擎），此功能需要与 mybatis-spring 2.0.2 以上版本一起使用。 |
| `configuration-properties`               | 可在外部配置的 MyBatis 配置项。指定的配置项可以被用作 MyBatis 配置文件和 Mapper 文件的占位符 |
| `lazy-initialization`                    | 是否启用 mapper bean 的延迟初始化。设置 `true` 以启用延迟初始化。此功能需要与 mybatis-spring 2.0.2 以上版本一起使用。 |
| `mapper-default-scope`                   | 通过自动配置扫描的 mapper 组件的默认作用域。该功能需要与 mybatis-spring 2.0.6 以上版本一起使用。 |
| `inject-sql-session-on-mapper-scan`      | 设置是否注入 `SqlSessionTemplate` 或 `SqlSessionFactory` 组件 （如果你想回到 2.2.1 或之前的行为，请指定 `false` ）。如果你和 spring-native 一起使用，应该设置为 `true` （默认）。 |
| `configuration.*`                        | MyBatis Core 提供的`Configuration` 组件的配置项。注：此属性不能与 `config-location` 同时使用。 |
| `scripting-language-driver.thymeleaf.*`  | MyBatis `ThymeleafLanguageDriverConfig` 组件的 properties keys。 |
| `scripting-language-driver.freemarker.*` | MyBatis `FreemarkerLanguageDriverConfig` 组件的 properties keys。 |
| `scripting-language-driver.velocity.*`   | MyBatis `VelocityLanguageDriverConfig` 组件的 properties keys。 |

## Java配置

MyBatis-Spring-Boot-Starter 将自动寻找实现了 `ConfigurationCustomizer` 接口的组件，调用自定义 MyBatis 配置的方法。我们可以用它对mybatis进行java配置

~~~java
@Configuration
public class MyBatisConfig {
  @Bean
  ConfigurationCustomizer mybatisConfigurationCustomizer() {
    return new ConfigurationCustomizer() {
      @Override
      public void customize(Configuration configuration) {
        // customize ...
      }
    };
  }
}
~~~

