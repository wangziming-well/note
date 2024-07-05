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

## Starter

Starter是Springboot提供的一系列开箱即用的依赖，可以在应用程序中导入它们。 通过Starter，可以获得所有需要的Spring和相关技术的一站式服务，免去了需要到处大量复制粘贴依赖的烦恼。例如，想要使用springmvc，只需要引入下面依赖即可：

~~~xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
~~~

而无需导入所有的相关依赖，这是因为在`spring-boot-starter-web`包中已经提前帮我们导入好了相关依赖

@SpringBootApplication

一个 `@SpringBootApplication` 注解就可以用来启用这三个功能，如下。

- `@EnableAutoConfiguration`：启用Spring Boot的自动配置机制。
- `@ComponentScan`：对应用程序所在的包启用 `@Component` 扫描。
- `@SpringBootConfiguration`：允许在Context中注册额外的Bean或导入额外的配置类。这是Spring标准的 `@Configuration` 的替代方案



