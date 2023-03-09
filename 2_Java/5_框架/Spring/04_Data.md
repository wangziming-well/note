# JdbcTemplate简述

JDBC是Java平台访问关系型数据库的标准API，它主要面向比较底层的数据库操作，提供尽可能多的访问数据库的功能。

但是过于贴近底层的API设计，让JDBC来实际使用过程中比较繁琐，会产生大量重复的代码。并且在使用JDBC的过程中要如果不遵守JDBC的规范，可能会造成严重的程序问题。

所以Spring提供了JdbcTemplate类来帮助我们通过JDBC访问数据库：

* 它封装所有基于JDBC 的数据访问代码，用统一的规范和格式来使用JDBC API
* 对SQLException所提供的异常信息在框架内进行统一转义，简化客户端代码对数据访问异常的处理

JdbcTempate实现了设计模式：模板方法模式(Template Method Pattern)：

这个设计模式主要用于对算法或者行为逻辑进行封装，如果多个类中存在相似的算法逻辑或者行为逻辑，那么可以将这些相似的逻辑提取到模板方法类中实现，然后让相应的子类根据需要实现某些自定义的逻辑

JDBC API就十分适合用模板方法模式封装，不管是谁来进行数据访问，不管数据访问逻辑如何，JDBC API 进行数据访问的代码都有一下的流程：

1. 获取数据库连接

2. 根据Connection创建相应的Statement或者PreparedStatement

3. 向Statement对象传入SQL语句或者参数，进行数据库的增删改查操作

4. 关闭相应的Statement和PreparedStatement

5. 处理相应的数据库异常

6. 关闭数据库连接以避免连接泄露

在第三步时，执行sql语句并返回结果的逻辑必须在模板方法中完成，不能放到模板外，因为我们要在获取到结果后及时关闭链接和资源，

所以可以定义一个Callback接口，实现Callback接口以实现执行sql语句并获取结果

# JdbcTemplate类

## 继承关系

JdbcTemplate类的定义如下：

~~~java
public class JdbcTemplate extends JdbcAccessor implements JdbcOperations
~~~

它继承了JdbcAccessor类并实现了JdbcOperations接口。

* JdbcOperations接口声明了JDBC查询更新等操作集合
* JdbcAccessor是一个抽象类，为子类提供一些公有的属性：
    * DataSource：JDBC获取数据源的统一接口
    * SQLExceptionTranslator: 定义Spring对SQLException进行统一转译的行为。

## 方法

JdbcTemplate提供的模板方法可划分为四类：

* 面向Connection的模板方法：使用ConnectionCallback回调接口所公开的Connection进行数据访问，不需要关心Connection的获取和释放。通常情况下，应避免直接使用面向Connection层面的模板方法进行数据访问。

* 面向Statement的模板方法：主要处理基于静态的SQL的数据访问请求。

  使用StatementCallback回调接口对外公开Statement类型的操作句柄。

* 面向PreparedStatement的模板方法：使用包含查询参数的SQL请求。使用PreparedStatement可以避免sql注入。

  通过PreparedStatementCreator回调接口公开Connection以允许PreparedStatement的创建。PreparedStatement创建后，会公开给PreparedStatementCallback回调接口，以支持其使用PreparedStatement进行数据访问。

* 面向CallableStatement的模板方法。JDBC支持使用CallableStatement进行数据库存储过程的访问。





