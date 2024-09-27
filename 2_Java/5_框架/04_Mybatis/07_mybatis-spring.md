# 简介

mybatis提供MyBatis-Spring 以将MyBatis 代码无缝整合到 Spring 中，它提供下面功能，行为：

* 让mybatis加入到spring的统一事务管理中
* 创建映射器 mapper 和 `SqlSession` 并注入到 bean 中
* 将 Mybatis 的异常转换为 Spring 的 `DataAccessException`
* 让应用代码与框架解耦合

maven依赖为：

~~~xml
<dependency>
  <groupId>org.mybatis</groupId>
  <artifactId>mybatis-spring</artifactId>
  <version>3.0.4</version>
</dependency>
~~~

# 快速开始

## 声明`SqlSessionFactory`

要在spirng中使用mybatis，首先需要在spring容器中定义一个`SqlSessionFactory`。mybatis-spring，提供了 `SqlSessionFactoryBean`来创建 `SqlSessionFactory`，可以用下面xml/注解声明配置它：

~~~xml
<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
  <property name="dataSource" ref="dataSource" />
</bean>
~~~

~~~java
@Configuration
public class MyBatisConfig {
  @Bean
  public SqlSessionFactory sqlSessionFactory() throws Exception {
    SqlSessionFactoryBean factoryBean = new SqlSessionFactoryBean();
    factoryBean.setDataSource(dataSource());
    return factoryBean.getObject();
  }
}
~~~

## 声明`Mapper`接口

可以将`MapperFactoryBean`将mybatis的Mapper接口添加到Spring容器中。

例如，对于下面`Mapper`接口：

~~~java
public interface UserMapper {
  @Select("SELECT * FROM users WHERE id = #{userId}")
  User getUser(@Param("userId") String userId);
}
~~~

通过下面XML/注解声明将其添加到Spring容器：

~~~xml
<bean id="userMapper" class="org.mybatis.spring.mapper.MapperFactoryBean">
  <property name="mapperInterface" value="org.mybatis.spring.sample.mapper.UserMapper" />
  <property name="sqlSessionFactory" ref="sqlSessionFactory" />
</bean>
~~~

~~~java
@Configuration
public class MyBatisConfig {
  @Bean
  public UserMapper userMapper() throws Exception {
    SqlSessionTemplate sqlSessionTemplate = new SqlSessionTemplate(sqlSessionFactory());
    return sqlSessionTemplate.getMapper(UserMapper.class);
  }
}
~~~

这样，就可以像使用Spring中的其他普通bean一样，将mapper注入到需要使用的地方。

`MapperFactoryBean` 将会负责 `SqlSession` 的创建和关闭。 如果使用了 Spring 的事务功能，那么当事务完成时，session 将会被提交或回滚。最终任何异常都会被转换成 Spring 的 `DataAccessException` 异常。

## 使用Mapper实例

上面配置完成后，就可以通过spring注入mapper实例并使用它了：

~~~java
public class FooServiceImpl implements FooService {

  private final UserMapper userMapper;

  public FooServiceImpl(UserMapper userMapper) {
    this.userMapper = userMapper;
  }

  public User doSomeBusinessStuff(String userId) {
    return this.userMapper.getUser(userId);
  }
}
~~~

# `SqlSessionFactoryBean`

在快速开始中，我们知道了在Spring中是通过`SqlSessionFactoryBean`声明`SqlSessionFactory`的。

`SqlSessionFactoryBean`需要一个`DataSource`来配置数据源。

另一个常用的属性是`confiLocation`,它用来指定 MyBatis 的 XML 配置文件路径

需要注意的是，这个配置文件**并不需要**是一个完整的 MyBatis 配置，该配置文件的下面元素会被**忽略**：

* 环境配置（`<environments>`）
* 数据源（`<DataSource>`）
* MyBatis 的事务管理器（`<transactionManager>`）

以上要素会由 `SqlSessionFactoryBean` 管理。



如果 MyBatis 在映射器类对应的路径下找不到与之相对应的映射器 XML 文件，同时仍需要配置文件时，有两种解决办法：

* 在`SqlSessionFactoryBean`指定 的 XML 配置文件中的 `<mappers>` 部分中指定 XML 文件的类路径
* 设置`SqlSessionFactoryBean`的 `mapperLocations` 属性

 `mapperLocations`接受多个Resource location，并且支持Ant风格的通配符。例如：

~~~xml
<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
  <property name="dataSource" ref="dataSource" />
  <property name="mapperLocations" value="classpath*:sample/config/mappers/**/*.xml" />
</bean>
~~~

自 1.3.0 版本开始，新增的 `configuration` 属性能够在没有对应的 MyBatis XML 配置文件的情况下，直接设置 `Configuration` 实例。例如：

```xml
<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
  <property name="dataSource" ref="dataSource" />
  <property name="configuration">
    <bean class="org.apache.ibatis.session.Configuration">
      <property name="mapUnderscoreToCamelCase" value="true"/>
    </bean>
  </property>
</bean>
```

# 加入Spring事务管理

MyBatis-Spring 使用 Spring 中的 `DataSourceTransactionManager` 来实现事务管理。

只需要配置一个 `DataSourceTransactionManager` ，并将  mybatis使用的`DataSource`交给这个事务管理器。

那么之后的事务管理工作就可以全部交给Spring和 `DataSourceTransactionManager`。可以使用Spring声明式事务管理，如`@Transactional`来管理事务。

 `DataSourceTransactionManager`的声明方式如下：

~~~xml
<bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
  <constructor-arg ref="dataSource" />
</bean>
~~~

~~~java
@Configuration
public class DataSourceConfig {
  @Bean
  public DataSourceTransactionManager transactionManager() {
    return new DataSourceTransactionManager(dataSource());
  }
}
~~~

**注意**：为事务管理器指定的 `DataSource` **必须**和用来创建 `SqlSessionFactoryBean` 的是同一个数据源，否则事务管理器就无法工作了。

# 注册映射器

来快速开始中我们使用`MapperFactoryBean`将映射器注册到了Spring中，这里再次展示一遍：

~~~xml
<bean id="userMapper" class="org.mybatis.spring.mapper.MapperFactoryBean">
  <property name="mapperInterface" value="org.mybatis.spring.sample.mapper.UserMapper" />
  <property name="sqlSessionFactory" ref="sqlSessionFactory" />
</bean>
~~~

对应的Java配置：

~~~java
@Configuration
public class MyBatisConfig {
  @Bean
  public MapperFactoryBean<UserMapper> userMapper() throws Exception {
    MapperFactoryBean<UserMapper> factoryBean = new MapperFactoryBean<>(UserMapper.class);
    factoryBean.setSqlSessionFactory(sqlSessionFactory());
    return factoryBean;
  }
}
~~~

接下来我们需要了解`MapperFactoryBean` 的一个**重要细节**：

如果注册的Mapper接口在相同的类路径下有对应的 XML映射器配置文件，并且该XML文件名称与接口文件类名相同。那么该配置文件将被`MapperFactoryBean`自动解析。不需要再显式配置映射器文件位置。

# 自动注册映射器

在快速开始中，我们使用`MapperFactoryBean`显式地声明注册了需要使用的Mapper接口。

为了方便注册映射器，mybatis-spring提供映射器类路径扫描机制来自动发现和注册映射器。提供以下几种方式：

* 使用 `@MapperScan` 注解
* 使用 `<mybatis:scan/>` 元素
* 在经典 Spring XML 配置文件中注册一个 `MapperScannerConfigurer`

需要注意到，上面前两种方式底层仍然使用的第三种方式，而`MapperScannerConfigurer`最终还是通过`MapperFactoryBean` 来实现映射器的注册的。所以：使用映射器自动扫描时，仍然可以让XML Mapper配置文件和Mapper接口放在同一类路径下，这样就可以避免显式声明Mapper配置文件位置。

## `<mybatis:scan>`

如果spring使用的是xml文件配置，那么可以使用`<mybatis:scan>`

`<mybatis:scan/>` 元素会发现映射器，它发现映射器的方法与 Spring 内建的 `<context:component-scan/>` 发现 bean 的方法非常类似。示例：

~~~xml
<beans xmlns="http://www.springframework.org/schema/beans"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xmlns:mybatis="http://mybatis.org/schema/mybatis-spring"
  xsi:schemaLocation="
  http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
  http://mybatis.org/schema/mybatis-spring http://mybatis.org/schema/mybatis-spring.xsd">

  <mybatis:scan base-package="org.mybatis.spring.sample.mapper" />

  <!-- ... -->

</beans>
~~~

`base-package` 属性允许设置映射器接口文件的基础包。通过使用逗号或分号分隔，你可以设置多个包。并且会在你所指定的包中递归搜索映射器。

使用`<mybatis:scan>`时不需要指定`SqlSessionFactory`或 `SqlSessionTemplate`,因为该元素会使用能够被自动注入的 `MapperFactoryBean`。

所以当使用多个数据源时这种自动注入行为就不适用了。可以通过`<mybatis:scan>`元素的 `factory-ref` 或 `template-ref` 属性指定你想使用的 bean 名称

`<mybatis:scan/>` 支持基于标记接口或注解的过滤操作:

*  `annotation` 属性，指定映射器必须拥有特定注解。
*  `marker-interface` 属性，可以指定映射器必须继承的父接口。

默认情况下，这两个属性为空，因此在基础包中的所有接口都会被作为映射器被发现。

被发现的映射器会按照 Spring 对自动发现组件的默认命名策略进行命名 。也就是说，如果没有使用注解显式指定名称，将会使用映射器的首字母小写非全限定类名作为名称。但如果发现映射器具有 `@Component` 或 JSR-330 标准中 `@Named` 注解，会使用注解中的名称作为名称。 

## `@MapperScan`

如果spring使用的是java配置，那么可以在`@Configuraiton`类上使用`@MapperScan`来实现映射器自动注册：

~~~java
@Configuration
@MapperScan("org.mybatis.spring.sample.mapper")
public class AppConfig {
  // ...
}
~~~

这个注解与的 `<mybatis:scan/>` 的工作方式一样。它也可以通过 `markerInterface` 和 `annotationClass` 属性设置标记接口或注解类。 通过配置 `sqlSessionFactory` 和 `sqlSessionTemplate` 属性，你还能指定一个 `SqlSessionFactory` 或 `SqlSessionTemplate`。

**NOTE** 如果 `basePackageClasses` 或 `basePackages` 没有定义， 扫描将基于声明这个注解的类所在的包。

## `MapperScannerConfigurer`

`MapperScannerConfigurer `是一个 `BeanDefinitionRegistryPostProcessor`,可以在Spring容器启动时进行类路径扫描并将Mapper注册到容器中。实际上上面两个方式底层还是使用的`MapperScannerConfigurer `

它的使用示例：

~~~xml
<bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
  <property name="basePackage" value="org.mybatis.spring.sample.mapper" />
</bean>
~~~

