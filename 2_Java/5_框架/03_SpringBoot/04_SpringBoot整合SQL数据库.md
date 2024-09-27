# 内嵌数据库

使用内存中的嵌入式数据库在开发时会很方便。显然，内存数据库不支持持久化存储。所以需要在应用启动时填充数据库，并且需要认识到这些数据会在应用关闭时丢失。

Springboot可以自动配置内嵌的H2、HSQL和Derby数据库。只需要在项目中包含对应的内嵌数据库依赖，springboot就会自动发现并使用它们。

如果当前类路径中由多个内嵌数据库，那么可以通过`spring.datasource.embedded-database-connection`配置属性控制启用指定的内嵌数据库。该属性为`none`时将禁用内嵌数据库的自动配置。

例如在项目中包含下面依赖，将自动配置内嵌数据库：

~~~xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
	<groupId>org.hsqldb</groupId>
	<artifactId>hsqldb</artifactId>
	<scope>runtime</scope>
</dependency>
~~~

# 连接生产数据库

spring使用池化的`DataSource`自动配置了生产数据库的连接

## 配置

通过外部化配置`spring.datasource.*`来控制DataSource配置。例如：

~~~properties
spring.datasource.url=jdbc:mysql://localhost/test
spring.datasource.username=dbuser
spring.datasource.password=dbpass
~~~

需要至少设置 `spring.datasource.url` 来指定URL。否则，springboot将自动配置一个内嵌数据库。

springboot可以根据URL推断出大多数数据库的JDBC驱动类。如果需要指定一个特定类，可以使用`spring.datasource.driver-class-name` 属性。



完整的配置属性参见`org.springframework.boot.autoconfigure.jdbc.DataSourceProperties`。这些配置属性是标准选项。对任何数据库连接实现都生效。

也可以使用实现的前缀来调整特定实现的配置。例如 (`spring.datasource.hikari.*`, `spring.datasource.tomcat.*`, `spring.datasource.dbcp2.*`, and `spring.datasource.oracleucp.*`)，例如如果使用的是Tomcat练级吃，那么可以定制格外的设置，如下所示：

~~~properties
spring.datasource.tomcat.max-wait=10000
spring.datasource.tomcat.max-active=50
spring.datasource.tomcat.test-on-borrow=true
~~~

## 连接池

springboot使用下述规则来选择一个连接池的特定实现：

* 如果`HikariCP`连接池可用，那么springboot会选择它
* 否则，使用Tomcat池化`DataSource`
* 否则，使用`Commons DBCP2`
* 否则，使用`Oracle UCP`

 `spring-boot-starter-jdbc` 和`spring-boot-starter-data-jpa` 的starter中会自动获取`HikariCP`依赖。

如果需要，可以通过设置`spring.datasource.type`属性来指定连接池的具体实现。

# 事务自动配置

springboot通过`org.springframework.boot.autoconfigure.transaction.TransactionAutoConfiguration`自动配置类，自动配置了spring的事务管理。它:

* 根据项目依赖选择自动配置`TranactionManager`
  * 如果存在 `spring-boot-starter-data-jpa`，会自动配置一个 `JpaTransactionManager`
  * 如果没有引入 JPA 但引入了 `spring-jdbc`，会自动配置一个 `DataSourceTransactionManager`。

* 隐式启用了`@EnableTransactionManagement`注解

所以在springboot环境中，默认情况下不需要任何配置就可以使用`@Transactional`来控制事务行为。

可以通过`spring.transaction.*`属性来配置全局的事务行为。(见`TransactionProperties`属性类)，例如：

~~~properties
spring.transaction.default-timeout=30 # 设置事务超时时间为30秒
spring.transaction.rollback-on-commit-failure=true # 事务提交失败时回滚
~~~



如果有更复杂的事务管理需求，也可以手动配置`TransactionManager`，例如：

~~~java
@Configuration
public class TransactionManagerConfig {

    @Bean
    public PlatformTransactionManager transactionManager(EntityManagerFactory entityManagerFactory) {
        return new JpaTransactionManager(entityManagerFactory);
    }
}
~~~

