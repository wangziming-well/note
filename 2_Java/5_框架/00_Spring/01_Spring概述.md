# Spring概述

Spring 框架是一个功能强大的 Java 应用程序框架，旨在提供高效且可扩展的开发环境。它结合了轻量级的容器和依赖注入功能，提供了一种使用 POJO (Plain Old Java Object 简单Java对象)进行容器配置和面向切面的编程的简单方法，以及一组用于AOP的模块。此外，它还提供了对事务管理、对象/关系映射、JavaBeans、JDBC、JMS 和其他技术的支持，从而确保高效开发。

## 设计理念

下面是Spring框架的指导原则。

- 在每个层面上提供选择。Spring让你尽可能晚地推迟设计决策。例如，你可以通过配置来切换持久化供应商，而不需要改变你的代码。对于许多其他基础设施问题和与第三方API的集成也是如此。
- 适应不同的观点。Spring拥抱灵活性，对事情应该如何做不持意见。它支持具有不同视角的广泛的应用需求。
- 保持强大的后向兼容性。Spring的演进是经过精心管理的，在不同的版本之间几乎不存在破坏性的变化。Spring支持一系列精心选择的JDK版本和第三方库，以方便维护依赖Spring的应用程序和库。
- 关心API的设计。Spring团队花了很多心思和时间来制作直观的API，并且在很多版本和很多年中都能保持良好的效果。
- 为代码质量设定高标准。Spring框架非常强调有意义的、最新的和准确的javadoc。它是为数不多的可以宣称代码结构干净、包与包之间没有循环依赖关系的项目之一。

# Maven依赖

## 核心库

- spring-core
  这个jar 文件包含Spring 框架基本的核心工具类。Spring 其它组件要都要使用到这个包里的类，是其它组件的基本核心，当然你也可以在自己的应用系统中使用这些工具类。
  外部依赖Commons Logging， (Log4J)。

```xml
<dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-core</artifactId>
        <version>4.3.7.RELEASE</version>
</dependency>
```

- spring-beans
  这个jar 文件是所有应用都要用到的，它包含访问配置文件、创建和管理bean 以及进行Inversion ofControl / Dependency Injection（IoC/DI）操作相关的所有类。如果应用只需基本的IoC/DI 支持，引入spring-core.jar 及spring-beans.jar 文件就可以了。
  外部依赖spring-core，(CGLIB)。

```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-beans</artifactId>
    <version>4.3.7.RELEASE</version>
</dependency>
```

- spring-context
  这个jar 文件为Spring 核心提供了大量扩展。可以找到使用Spring ApplicationContext特性时所需的全部类，JDNI 所需的全部类，instrumentation组件以及校验Validation 方面的相关类。
  外部依赖spring-beans, (spring-aop)。

```xml
<dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-context</artifactId>
        <version>4.3.7.RELEASE</version>
 </dependency>
```

- spring-context-support
  包含支持缓存Cache（ehcache）、JCA、JMX、 邮件服务（Java Mail、COS Mail）、任务计划Scheduling（Timer、Quartz）方面的类。
  以前的版本中应该是这个：spring-support.jar这个jar 文件包含支持UI模版（Velocity，FreeMarker，JasperReports），邮件服务，脚本服务(JRuby)，缓存Cache（EHCache），任务计划Scheduling（uartz）方面的类。
  外部依赖spring-context, (spring-jdbc, Velocity,FreeMarker, JasperReports, BSH, Groovy,JRuby, Quartz, EHCache)

```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context-support</artifactId>
    <version>4.3.7.RELEASE</version>
</dependency>
```

- spring-expression
  模块提供了一个强大的表达式语言，用于在运行时查询和处理对象图。该语言支持设置和获取属性值；属性赋值，方法调用，访问数组的内容，收集和索引器，逻辑和算术运算，命名变量，并从Spring的IOC容器的名字对象检索，它也支持列表选择和投影以及常见的列表聚合。

```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-expression</artifactId>
    <version>4.3.7.RELEASE</version>
</dependency>
```

## AOP

- Spring-aop(必须)
  这个jar 文件包含在应用中使用Spring 的AOP 特性时所需的类和源码级元数据支持。
  使用基于AOP 的Spring特性，如声明型事务管理（Declarative Transaction Management），也要在应用里包含这个jar包。
  外部依赖spring-core， (spring-beans，AOP Alliance， CGLIB，Commons Attributes)

```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-aop</artifactId>
    <version>4.3.7.RELEASE</version>
</dependency>
```

aspectj的runtime包(必须)

```xml
<dependency>
    <groupId>org.aspectj</groupId>
    <artifactId>aspectjrt</artifactId>
    <version>1.9.1</version>
</dependency>
```

aspectjweaver是aspectj的织入包(必须)

```xml
<dependency>
    <groupId>org.aspectj</groupId>
    <artifactId>aspectjweaver</artifactId>
    <version>1.9.1</version>
</dependency>
```

- Spring-aspects
  提供对AspectJ的支持，以便可以方便的将面向方面的功能集成进IDE中，比如Eclipse AJDT。
  外部依赖。

```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-aspects</artifactId>
    <version>4.3.7.RELEASE</version>
</dependency>
```

## Data Access/Intergration

- spring-jdbc
  这个jar 文件包含对Spring 对JDBC 数据访问进行封装的所有类。
  外部依赖spring-beans，spring-dao。

```xml
<dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-jdbc</artifactId>
        <version>4.3.7.RELEASE</version>
</dependency>
```

- spring-tx
  以前是在这里org.springframework.transaction
  为JDBC、Hibernate、JDO、JPA、Beans等提供的一致的声明式和编程式事务管理支持。

```xml
 <dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-tx</artifactId>
    <version>4.3.7.RELEASE</version>
</dependency>
```

- spring-orm
  包含Spring对DAO特性集进行了扩展，使其支持iBATIS、JDO、OJB、TopLink， 因为Hibernate已经独立成包了，现在不包含在这个包里了。这个jar文件里大部分的类都要依赖spring-dao.jar里的类，用这个包时你需要同时包含spring-dao.jar包。

```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-orm</artifactId>
    <version>4.3.7.RELEASE</version>
</dependency>
```

## web

- spring-web
  这个jar 文件包含Web 应用开发时，用到Spring 框架时所需的核心类，包括自动载入Web ApplicationContext 特性的类、Struts 与JSF 集成类、文件上传的支持类、Filter 类和大量工具辅助类。
  外部依赖spring-context, Servlet API, (JSP API, JSTL,Commons FileUpload, COS)。

```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-web</artifactId>
    <version>4.3.7.RELEASE</version>
</dependency>
```

- spring-webmvc
  这个jar 文件包含Spring MVC 框架相关的所有类。包括框架的Servlets，Web MVC框架，控制器和视图支持。当然，如果你的应用使用了独立的MVC 框架，则无需这个JAR 文件里的任何类。
  外部依赖spring-web, (spring-support，Tiles，iText，POI)。

```xml
<dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-webmvc</artifactId>
        <version>4.3.7.RELEASE</version>
</dependency>
```

