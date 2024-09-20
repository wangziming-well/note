# JDBC简述

JDBC是Java平台进行数据库连接的标准API

JDBC的规范接口定义在 `java.sql`和`javax.sql`包下

java本身没有提供这些接口的实现；具体的实现是由各个数据库厂商分别提供的，即他们提供JDBC驱动程序。

这样，即使不同数据库提供的驱动不同，Java程序员仍然可以通过一套API来操作访问它们。

# DriverManager

java.sql包下提供了驱动管理器类`DriverManager`它有两个主要功能：

* 注册和管理各个数据库厂商提供的驱动程序
* 提供对数据库的连接Connection

## 管理驱动

`DriverManager`内部维护了一个驱动器信息(DriverInfo)数组:

~~~JAVA
private final static CopyOnWriteArrayList<DriverInfo> registeredDrivers = new CopyOnWriteArrayList<>();
~~~

`DriverInfo`类内部持有具体的Driver类，具体的Driver类提供具体的`Connection`对象

通过Connection对象，我们可以访问数据库

`DriverManager`提供了一下与驱动注册相关的方法:

~~~java
//注册驱动，实质上是向registeredDrivers数组添加一个持有driver实例的DriverInfo
public static void registerDriver(java.sql.Driver driver)
//取消注册驱动，实质上是将driver移除registeredDrivers数组
public static void deregisterDriver(Driver driver)
~~~

## 获取连接

注册驱动后，可以通过`DriverManager`获取连接，它提供了以下获取连接的重载方法：

~~~java
public static Connection getConnection(String url)

public static Connection getConnection(String url,String user, String password) 
    
public static Connection getConnection(String url,java.util.Properties info)
~~~

以上重载方法实际上调用的是：

~~~java
private static Connection getConnection(
        String url, java.util.Properties info, Class<?> caller)
~~~

该方法的内部逻辑实际上就是循环registeredDrivers数组,通过`DriverInfo`持有的`Driver`的`connect`方法建立一个到指定数据库的连接，并获取连接对应的Connection对象；它会返回获取到的第一个Connection

## 示例

要使用特定的JDBC驱动程序，首先需要获取相应的jar包，以mysql提供的驱动程序为例，它的maven依赖为：

~~~xml
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.28</version>
</dependency>
~~~

这个jar包下`com.mysql.cj.jdbc`包下提供了以下类:

~~~java
public class Driver extends NonRegisteringDriver implements java.sql.Driver {
    static {
        try {
            java.sql.DriverManager.registerDriver(new Driver());
        } catch (SQLException E) {
            throw new RuntimeException("Can't register driver!");
        }
    }

    public Driver() throws SQLException {
        // Required for Class.forName().newInstance()
    }
}
~~~

可以看到它定义的静态代码块中执行了DriverManager.registerDriver()方法将自己注册到驱动管理器中了；

所以想要执行注册，只需要实例化该类就行，可以通过以下方式实例化该类:

~~~java
//直接用new关键字实例化类
new com.mysql.cj.jdbc.Driver();
//使用Class.forName()方法
Class.forName("com.mysql.cj.jdbc.Driver");
~~~

实际上在 JDK 6 及以后的版本中，使用 JDBC 连接数据库时，已经不再需要显式地调用 `Class.forName()` 方法来加载和注册数据库驱动程序了，因为 JDBC 4.0 规范中定义了自动加载和注册驱动程序的机制，只要在类路径下包含了数据库驱动程序的 JAR 包，JDBC 就会自动加载和注册该驱动程序。

# DataSource

DataSource是JDBC 2.0标准之后引入的接口，是统一管理数据库连接的规范

是替代DriverManager工具，获取Connection对象的首选方案

接口定义的最重要的方法是:

~~~java
Connection getConnection();
~~~

用以获取数据库连接

DataSource的具体实现由数据库厂商或者第三方框架提供，按照功能不同，可以使用将DataSource分为以下三类：

* 简单的DataSource实现：

  只提供作为ConnectionFactory角色的基本功能，不会用于正式的生产环境

* 具有连接缓冲池的DataSource实现：

  内部通过数据库连接缓冲池对数据库连接进行管理

* 支持分布式事务的DataSource实现：

  这一类DataSource实现类应该是XADataSource的实现类，可以实现分布式事务

主流的 JDBC 数据库连接池主要包括以下几种：

* HikariCP 是目前最流行的 JDBC 数据库连接池之一，以其高性能和轻量级著称。

  * 优点：

    - 性能优越，特别是在高并发情况下表现突出。

    - 轻量级，开销低，内存占用小。

    - 提供详细的监控和统计功能。

    - 快速启动与连接池填充。

  * 缺点：
    - 配置较为复杂，需要对不同参数进行细调才能达到最佳性能。

  * **使用建议**：HikariCP 适用于大部分场景，特别是高并发、高吞吐量的应用，如 Web 服务和微服务架构。它是性能敏感型项目的首选。

* Apache DBCP 是 Apache Commons 项目的一部分，也是较为流行的 JDBC 连接池之一。

  * 优点：
    * 成熟稳定，历史悠久，广泛使用。
    * 配置较为简单，易于集成。
    * 拥有丰富的功能，适用于各种场景。

  * 缺点：
    - 性能不如 HikariCP，尤其在高并发下。
    - 启动较慢，初始化时间长。
  * **使用建议**：适合中小型项目，或对性能要求不太高的应用。如果是已有项目且需要稳定的连接池，DBCP 是一个安全的选择。

* C3P0 是较早出现的一个 JDBC 连接池，也是很多老项目中常见的选择。

  - 优点：
    - 支持多种配置选项，灵活性高。
    - 配置简单，易于使用。

  - 缺点：
    - 性能较差，尤其是在高并发情况下。
    - 更新较慢，不如 HikariCP 那样活跃维护。

  - **使用建议**：适合小型项目或者已有使用 C3P0 的遗留系统。在性能要求不高的场景下依然可以胜任。

- Tomcat JDBC Pool 是 Tomcat 团队开发的连接池，很多 Tomcat 服务器会自带这个连接池。

  - 优点：
    - 与 Tomcat 深度集成，特别适合使用 Tomcat 作为应用服务器的项目。
    - 提供丰富的监控、日志和调优选项。

  - 缺点：
    - 性能不如 HikariCP，在高负载情况下表现一般。

  - **使用建议**：如果你的项目使用 Tomcat 作为应用服务器，并且对性能要求不是特别极端，Tomcat JDBC Pool 是一个不错的选择。否则可以考虑 HikariCP。

# Connection

与特定数据库的连接（会话），在连接上下文中执行 SQL 语句并返回结果。

Connection类主要有两个功能：

* 创建获取Statement对象以执行具体sql语句。可以创建多个Statement对象
* 进行事务管理

## 创建statement会话

* `Statement createStatement() `
          创建一个 Statement 对象来将 SQL 语句发送到数据库。 

*  `Statement createStatement(int resultSetType, int resultSetConcurrency) `
              创建一个 Statement 对象，该对象将生成具有给定类型和并发性的 ResultSet 对象。 

* `PreparedStatement prepareStatement(String sql) `
              创建一个 PreparedStatement 对象来将参数化的 SQL 语句发送到数据库。 

## 事务管理

事务是数据库中的重要概念，JDBC可以进行事务管理：

~~~java
void setAutoCommit(boolean autoCommit);
//设置连接的自动提交模式，默认为true，即每执行一个更新语句，就会被提交，无法回滚；如果要进行事务管理，需要将该值设为false
void rollback();
//回滚事务，取消在当前事务中进行的所有更改，并释放此 Connection 对象当前持有的所有数据库锁。 建议放入到catch代码中，发生异常时实现回滚。
void commit();
//提交事务，使所有上一次提交/回滚后进行的更改成为持久更改，并释放此 Connection 对象当前持有的所有数据库锁。 

~~~

可以使用savepoint进行更细致的回滚操作

创建一个保存点意味着稍后回滚时可以回滚到指定的保存点，而不是事务开头

~~~java
Savepoint setSavepoint();
Savepoint setSavepoint(String name);
//在当前事务中创建一个保存点 (savepoint)，并返回表示它的新 Savepoint 对象。
void rollback(Savepoint savepoint);
//回滚到指定的保存点
void releaseSavepoint(Savepoint savepoint);
//释放指定的保存点
~~~

# Statement

用于执行静态 SQL 语句并返回它所生成结果的对象。 

### 单次处理

* ResultSet executeQuery(String sql) 
  执行给定的 SQL 查询语句，该语句返回单个 ResultSet 对象。 

* int executeUpdate(String sql) 
   执行给定 SQL 更新(insert,update,delte)语句，返回受影响的行数。 

* ResultSet getGeneratedKeys()
  执行`executeUpdate`语句后，可以调用该方法获取影响行的主键。
  
* boolean execute(String sql) 
          执行给定的 SQL 语句，该语句可能返回多个结果。 如果为查询语句，返回true，如果为操作语句，返回false；执行该语句后可以通过getResultSet()和getUpdateCount()来获取执行结果
      
* ResultSet getResultSet() 
  
  返回前一条查询语句的结果集。如果前一条语句为产生结果集，则返回null
  对于每一条执行过的语句，该方法只能被调用一次
  
* boolean getMoreResults() 

     如果有下个结果集，将移动到下个结果集，并返回true,通过再次调用getResultSet，获取下个结果集。

     如果没有下个结果集，则返回false

* int getUpdateCount() 
          以更新计数的形式获取当前结果；如果结果为 ResultSet 对象或没有更多结果，则返回 -1。 
          对于每一条执行过的语句，该方法只能被调用一次

### 批处理

* void addBatch(String sql) 
          将给定的 SQL 命令添加到此 Statement 对象的当前命令列表中。 

* int[] executeBatch() 
          将一批命令提交给数据库来执行，如果全部命令执行成功，则返回更新计数组成的数组。 

*  void clearBatch() 
          清空此 Statement 对象的当前 SQL 命令列表。 

# PreparedStatement

PreparedStatement 继承了Statement，有Statement的所有功能。

表示预编译的 SQL 语句的对象。 

可以执行带占位符`?`的sql语句，每个`?`都代表一个变量

在执行预编译的SQL语句前，需为所有的`?`赋值

## 方法

为`?`参数赋值:

* `void setXxx(int parameterIndex, Xxx x) `
  设置第 parameterIndex个参数值为x

  (Xxx 指int、String、Date等数据类型)

* `void clearParameters()`

  消除预备语句中所有的当前参数

执行sql语句:

* `ResultSet executeQuery() `
         在此 PreparedStatement 对象中执行 SQL 查询语句，并返回该查询生成的 ResultSet 对象。 
* `int executeUpdate() `
         在此 PreparedStatement 对象中执行 SQL 更新语句，

# ResultSet

表示数据库结果集的数据表，通常通过执行查询数据库的语句生成。 

在使用ResultSet结果集时，必须保持和数据库的连接。

在关闭了statement或者connection后，使用ResultSet对象，将报错。

## 字段

在创建statement是，可以传入两个参数:`Statement createStatement(int resultSetType, int resultSetConcurrency) `用于控制语句产生的结果集的类型和并发模式。

resultSetType可以取ResultSet的静态常量:

~~~java
int TYPE_FORWARD_ONLY = 1003;
//表示结果集游标只能向前移动，不可滚动(默认值)
int TYPE_SCROLL_INSENSITIVE = 1004;
//表示结果集是可滚动的并且Resultset中数据不随数据库中数据变化而变化响
int TYPE_SCROLL_SENSITIVE = 1005;
//表示结果集可滚动并且受Resultset中数据随着数据库中数据变化而变化
~~~

resultSetConcurrency可取ResultSet的静态常量:

~~~java
int CONCUR_READ_ONLY = 1007;
//结果集不可用于更新数据库(默认值)
int CONCUR_UPDATABLE = 1008;
//结果集可以用于更新数据库:编辑结果集中的数据，将结果集上数据的变更同步到数据库中
~~~

**注意：**

* 并非所有的JDBC驱动都支持可滚动和可更新的结果集

  可以使用DatabaseMetaData中的`supportsResultSetType()`和`supportsResultSetConcurrency`来判断当前驱动支持的结果集类型和并发模式

* 并且对于特定的查询，即使驱动支持，返回的结果集也可能是无法更新或者滚动的

  比如一个复杂查询的结果集很可能无法更新。

  可以调用具体的ResultSet的`getType()`和`getConcurrency()`方法确认当前结果集的类型

## 方法

### 控制游标

可使用下面方法控制游标移动：

~~~java
boolean absolute(int row) ;
//将游标移动到结果集的给定行编号。
boolean relative( int rows )
//将游标向前或向后移动指定行，rows为正向前移动，rows为负向后移动
boolean next() 
//将游标从当前位置向前移一行。如果已在最后一行，则返回false
boolean previous() 
//将游标移动到结果集的上一行，如果已在第一行，则返回false
boolean first()
//将游标移动到结果集的第一行。 
boolean last();
//将游标移动到结果集的最后一行。 
void beforeFirst();
//将游标移动到结果集的开头，正好位于第一行之前。 
void afterLast();
//将游标移动到结果集的末尾，正好位于最后一行之后。 
~~~

可使用以下方法判断游标是否在特殊位置：

~~~java
boolean isBeforeFirst();
boolean isAfterLast();
boolean isFirst();
boolean isLast();
~~~

**注意：**

* 默认的 ResultSet 对象不可滚动，仅有一个向前移动的光标。如果要获取可滚动的结果集，需要在创建`statement`时，将`resultSetType`设置为`TYPE_SCROLL_INSENSITIVE`或者`TYPE_SCROLL_SENSITIVE`
* 初始的结果集游标在第一行的前面第0行

### 读取行数据

获取游标所在行的数据：

~~~java
int getRow();
//获取当前行编号。 
Xxx  getXxx(int columnIndex);
//根据列下标，返回指定列的值
//Xxx指Java中的数据类型，例如String，int，Date，Blob,Clob等类型
Xxx getXxx(String columnLabel);
//根据列的列名，返回指定列的值
<T> T getObject(int columnIndex, Class<T> type);
//根据列下标，返回指定列的值,返回值类型由传入的type决定
<T> T getObject(String columnLabel, Class<T> type);
//根据列的列名，返回指定列的值,返回值类型由传入的type决定
ResultSetMetaData getMetaData();
//获取ResultSetMetaData对象，该对象可用于获取关于ResultSet对象中列的类型和属性的信息，比如可获取有多少列
~~~

### 更新行数据

~~~java
void updateXxx(int columnIndex, Xxx x) ;
//根据列下标，更新结果集中当前行的指定列
void updateXxx(String columnLabel, Xxx x) ;
//根据列名，更新结果集中当前行的指定类 
void updateRow() ;
//将当前行的数据更新，同步到数据库中
void cancelRowUpdates();
//取消结果集中对当前行的更新，该方法一般在调用updateXxx之后，和调用updateRow之前执行
//如果该方法被调用时，当前行没有调用过updateXxx或者已经调用过updateRow，则该方法不会做任何事情
~~~

**注意**：

* 默认的 ResultSet 对象不可更新；如果要获取可更新的结果集，需要在创建`statement`时，将`resultSetConcurrency`设置为`CONCUR_UPDATABLE`
* `updateXxx`方法只更新了结果集中的行值，而非数据库中的值。必须调用`updateRow`方法将当前行中的所有更新同步到数据库
* 如果没有调用`updateRow`方法就将游标移动到其他行上，那么对此行中的所有更新将被丢弃，并且不会被传递到数据库

### 插入行数据

 ~~~java
 void moveToInsertRow();
 //将游标移动到插入行，插入行是与可更新结果集关联的特殊行，用于执行插入操作
 //游标移动到插入行前，会记住当前的位置，执行完插入操作后，需要调用moveToCurrentRow()方法从插入行回到当前行
 //当游标自动到插入行后，只能执行updateXxx() 方法 和 insertRow()方法
 void insertRow();
 //将插入行的内容插入到该ResultSet对象和数据库中。调用此方法时，游标必须位于插入行上。
 void moveToCurrentRow();
 //将游标移动到记住的游标位置，通常是当前行。如果游标不在插入行上，则此方法无效。
 ~~~

通过ResultSet进行数据库插入操作示例：

~~~java
resultSet.moveToInsertRow();
resultSet.updateString("user_name","王梓铭");
resultSet.updateString("password","123456");
resultSet.updateTimestamp("register_time",new Timestamp(new java.util.Date().getTime()));
resultSet.insertRow();
resultSet.moveToCurrentRow();
~~~

### 删除数据

~~~java
void deleteRow();
//从结果集和底层数据库中删除当前行。当游标位于插入行上时，无法调用此方法。
~~~

# RowSet

可滚动的结果集在使用过程中需要始终保持与数据库的连接。

而RowSet无需始终保持与数据库的连接。

RowSet继承自ResultSet，拓展了它的功能

RowSet下还有子接口，细分了它的功能用途：

* CachedRowSet：允许在断开连接的状态下执行相关操作。
* WebRowSet：代表一个被缓存的行集，该行集可以保存为XML文件。该文件可以移动到Web应用的其他层中，用另外一个WebRowSet对象重新打开该文件即可
* FilteredRowSet&JoinRowSet：执行对行集的轻量级操作，对行集进行过滤和合并，相当于SQL语句中的select和join操作
* JdbcRowSet：是ResultSet接口的一个瘦包装器

## 获取行集

可以通过行集工厂对象获取行集，以CachedRowSet为例：

~~~java
RowSetFactory factory = RowSetProvider.newFactory();
CachedRowSet cachedRowSet = factory.createCachedRowSet();
~~~

## CachedRowSet

许在断开连接的状态下执行相关操作。也可以主动连接数据库

## 获取行数据

* 通过ResultSet获取数据：

~~~java
public void populate(ResultSet data);
//将data中的数据填充到CachedRowSet
~~~

* 通过连接数据库获取数据：

~~~java
void setUrl(String url);
//设置行集用DriverManager创建Connection所需要的url
void setUsername(String name);
//设置行集用DriverManager创建Connection所需要的username
void setPassword(String password);
//设置行集用DriverManager创建Connection所需要的password
void setCommand(String cmd);
//设置查询Sql语句
void setXxx(int parameterIndex, Xxx x);
//设置command命令中给定的参数，即带?占位符的sql参数
//parameterIndex从1开始
public void execute(Connection conn);
//会创建数据库连接、执行查询操作、填充行集、最后断开连接
//因为传入的Connect对象，调用前只需要设置command参数
void execute();
//需要行集先设置必要的参数:url username password command等;
~~~

* 分页查询

~~~java
public void setPageSize(int size);
//设置行集的分页大小，当行集执行execute()或者populate()时，会将数据填充到额外的分页上，获取数据将从分页获取
public boolean nextPage();
//让行集加载分页的下一页，如果加载的页存在，则返回true
public boolean previousPage();
//让行集加载分页的上一页，如果加载的页存在，则返回true
~~~

* 同步修改

~~~java
public void acceptChanges(Connection con);
//调用该方法将行集数据的修改同步到数据库
public void acceptChanges();
//使用该方法前，必须设置要连接数据库的参数:url、username、password
public void setTableName(String tabName);
//如果行集的数据是通过结果集填充的，那么行集就不知道需要同步的表的名称，必须先调用该方法设置表名
~~~

示例:

~~~java
RowSetFactory factory = RowSetProvider.newFactory();
CachedRowSet rowSet = factory.createCachedRowSet();
rowSet.setUrl("jdbc:mysql://localhost:3306/test");
rowSet.setUsername("root");
rowSet.setPassword("123456");
rowSet.setCommand("select * from user where user_id = ?");
rowSet.setInt(1,6);
rowSet.execute();
rowSet.next();
System.out.println(rowSet.getString("user_name"));
~~~

# 管理JDBC对象

每个Connection都可以创建一个或者多个Statement对象。同一个Statement对象可以用于多个不相关的命令和查询。但是一个Statement对象最多只能有一个打开的结果集。

使用完ResultSet、Statement或Connection对象后，应立刻调用close方法。

因为这些对象都使用了规模较大的数据结构，会占用数据库连接和内存资源。

如果Statement对象上有一个打开的结果集，那么调用Statement对象的close方法，将自动关闭该结果集。

同样的，调用Connection类的close方法将关闭该连接上的所有语句。

反过来，在Statement上调用closeOnCompletion方式时，当所有的结果集都被关闭后，该语句将被立刻自动关闭



