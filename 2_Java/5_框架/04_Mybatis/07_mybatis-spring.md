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

# 事务

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

