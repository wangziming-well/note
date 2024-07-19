# Springboot简介

SpringBoot是Spring家族中的一员，它是用来简化Spring应用程序的创建和开发过程

它通过采用大量默认配置的方式简化之前框架中繁琐配置

它还可以集成许多优秀的框架

## 特性

* 快速创建基于Spring的应用程序
* 直接使用java main方法启动内嵌的Tomcat服务器运行Spring Boot程序，不需要部署war包
* 提供约定的starter POM来简化Maven配置
* 采用注解配置，Java配置来替代了XML文件配置

## 核心

* 自动配置：针对很多Spring应用程序和常见的应用功能，Spring Boot能自动提供相关配置
* 起步依赖：告诉Spring Boot需要什么功能模块，它能够引入需要的依赖库
* Actuator：让你能够深入运行中的Spring Boot应用程序，一探Spring Boot程序的内部信息
* 命令行界面：这是Spring Boot的可选特性，主要针对Groovy语言使用

# 创建一个springboot项目

## 自动构建

可以手动创建springboot项目，但通过现代编译器，可以很方便的自动创建springboot项目，以IDEA为例，创建新项目时，选择Spring Initializr 的Generators，可以很方便的构建springboot项目：

![idea创建springboot项目](https://gitee.com/wangziming707/note-pic/raw/master/img/idea%E5%88%9B%E5%BB%BAspringboot%E9%A1%B9%E7%9B%AE.png)

在下一步可以很方便的选择需要的启动依赖项：

![springboot启动依赖项](https://gitee.com/wangziming707/note-pic/raw/master/img/springboot%E5%90%AF%E5%8A%A8%E4%BE%9D%E8%B5%96%E9%A1%B9.png)



## 手动构建

需要使用maven或者Gradle这样的项目构建工具来构建，以maven为例，需要使用下面的pom文件：

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>myproject</artifactId>
    <version>0.0.1-SNAPSHOT</version>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0-SNAPSHOT</version>
    </parent>

    <!-- 将在此添加其他行... -->

    <!-- ((只有当你使用 milestone 或 snapshot 版本时，你才需要这个。)) -->
    <repositories>
        <repository>
            <id>spring-snapshots</id>
            <url>https://repo.spring.io/snapshot</url>
            <snapshots><enabled>true</enabled></snapshots>
        </repository>
        <repository>
            <id>spring-milestones</id>
            <url>https://repo.spring.io/milestone</url>
        </repository>
    </repositories>
    <pluginRepositories>
        <pluginRepository>
            <id>spring-snapshots</id>
            <url>https://repo.spring.io/snapshot</url>
        </pluginRepository>
        <pluginRepository>
            <id>spring-milestones</id>
            <url>https://repo.spring.io/milestone</url>
        </pluginRepository>
    </pluginRepositories>
</project>
~~~

# Springboot核心机制

## Starter

Starter是Springboot提供的一系列开箱即用的依赖，可以在应用程序中导入它们。 通过Starter，可以获得所有需要的Spring和相关技术的一站式服务，免去了需要到处大量复制粘贴依赖的烦恼。例如，想要使用springmvc，只需要引入下面依赖即可：

~~~xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
~~~

而无需导入所有的相关依赖，这是因为在`spring-boot-starter-web`包中已经提前帮我们导入好了相关依赖

## Configuration 类

Spring Boot倾向于通过Java代码来进行配置的定义。 虽然也可以使用XML来配置 `SpringApplication` ，但还是建议你通过 `@Configuration` 类来进行配置。 通常，可以把启动类是作为主要的 `@Configuration` 类

### 导入额外的 Configuration 类

不需要把所有的 `@Configuration` 放在一个类中。 `@Import` 注解可以用来导入额外的配置类。 另外，你可以使用 `@ComponentScan` 来自动扫描加载所有Spring组件，包括 `@Configuration` 类

### 导入 XML Configuration

如果你确实需要使用基于XML的配置，我们建议你仍然从 `@Configuration` 类开始，然后通过 `@ImportResource` 注解来加载XML配置文件。

## 自动装配

Spring Boot的自动装配机制会试图根据你所添加的依赖来自动配置你的Spring应用程序。 例如，如果你添加了 `HSQLDB` 依赖，而且你没有手动配置任何DataSource Bean，那么Spring Boot就会自动配置内存数据库。

需要将 `@EnableAutoConfiguration` 或 `@SpringBootApplication` 注解添加到你的 `@Configuration` 类中，从而开启自动配置功能。

### 非侵入性

自动配置是非侵入性的。 在任何时候，你都可以开始定义你自己的配置来取代自动配置的特定部分。 例如，如果你添加了你自己的 `DataSource` bean，默认的嵌入式数据库支持就会“退步”从而让你的自定义配置生效。

如果你想知道在应用中使用了哪些自动配置，你可以在启动命令后添加 `--debug` 参数。 这个参数会为核心的logger开启debug级别的日志，会在控制台输出自动装配项目以及触发自动装配的条件。

### 禁用指定的自动装配类

如果你想禁用掉项目中某些自动装配类，你可以在 `@SpringBootApplication` 注解的 `exclude` 属性中指定，例如：

~~~java
@SpringBootApplication(exclude = { DataSourceAutoConfiguration.class })
public class MyApplication {

}
~~~

如果要禁用的自动装配类不在classpath上（没有导入），那么你可以在注解的 `excludeName` 属性中指定类的全路径名称。 `exclude` 和 `excludeName` 属性在 `@EnableAutoConfiguration` 中也可以使用。 最后，你也可以在配置文件中通过 `spring.autoconfigure.exclude[]` 配置来定义要禁用的自动配置类列表

## Spring Bean 和 依赖注入

你可以使用任何标准的Spring技术来定义你的Bean以及依赖注入关系。 推荐使用构造函数注入，并使用 `@ComponentScan` 注解来扫描Bean。

如果你按照上面的建议构造你的代码（将你的启动类定位在顶级包中），你可以在启动类添加 `@ComponentScan` 注解，也不需要定义它任何参数， 你的所有应用组件（`@Component`、`@Service`、`@Repository`、`@Controller` 和其他）都会自动注册为Spring Bean。

也可以直接使用 `@SpringBootApplication` 注解（该注解已经包含了 `@ComponentScan`）。

## `@SpringBootApplication`

一个 `@SpringBootApplication` 注解就可以用来启用这三个功能，如下。

- `@EnableAutoConfiguration`：启用Spring Boot的自动装配机制
- `@ComponentScan`：对应用程序所在的包启用 `@Component` 扫描。
- `@SpringBootConfiguration`：允许在Context中注册额外的Bean或导入额外的配置类。这是Spring标准的 `@Configuration` 的替代方案
