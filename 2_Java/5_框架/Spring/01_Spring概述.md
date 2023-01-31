# Spring概述

Spring是基于POJO(Plain Old Java Object 简单Java对象)的轻量级的Java开发框架

相较于其他框架，spring没有对开发过程中常用接口进行封装(比如mybatis封装了JDBC)

而是通过core核心模块IoC控制反转容器提供了一套适宜用POJO进行轻量级开发的环境

所以它可以集成其他框架，并简化其他框架的开发：

Spring提供了AOP框架，可以以AOP的形式增强POJO的功能

## 环境搭建

maven依赖：

~~~xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>5.2.5.RELEASE</version>
</dependency>
~~~

核心配置文件的头约束：

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
  http://www.springframework.org/schema/beans/spring-beans-4.1.xsd">
    <!--beans  就是对象容器-->
</beans>
~~~

