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

该方法的内部逻辑实际上就是循环registeredDrivers数组,通过`DriverInfo`持有的`Driver`的`connect`方法获取连接；它会返回获取到的第一个Connection

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

# Connection

与特定数据库的连接（会话）。在连接上下文中执行 SQL 语句并返回结果。

## 方法

### 创建statement会话

* Statement createStatement() 
              创建一个 Statement 对象来将 SQL 语句发送到数据库。 

*  Statement createStatement(int resultSetType, int resultSetConcurrency) 
              创建一个 Statement 对象，该对象将生成具有给定类型和并发性的 ResultSet 对象。 

* PreparedStatement prepareStatement(String sql) 
              创建一个 PreparedStatement 对象来将参数化的 SQL 语句发送到数据库。 

### 事务管理

*  void setAutoCommit(boolean autoCommit) 
              将此连接的自动提交模式设置为给定状态。 false表示开启事务

* void rollback() 
              取消在当前事务中进行的所有更改，并释放此 Connection 对象当前持有的所有数据库锁。 建议放入到catch代码中，发生异常时实现回滚。

*  void commit() 
              使所有上一次提交/回滚后进行的更改成为持久更改，并释放此 Connection 对象当前持有的所有数据库锁。 

* Savepoint setSavepoint() 
              在当前事务中创建一个未命名的保存点 (savepoint)，并返回表示它的新 Savepoint 对象。 

* Savepoint setSavepoint(String name) 
              在当前事务中创建一个具有给定名称的保存点，并返回表示它的新 Savepoint 对象。 



# Statement

## 描述

用于执行静态 SQL 语句并返回它所生成结果的对象。 

## 方法

### 单次处理

* ResultSet executeQuery(String sql) 
              执行给定的 SQL 查询语句，该语句返回单个 ResultSet 对象。 

* int executeUpdate(String sql) 
              执行给定 SQL 操作语句，返回操作数据的个数。 

* boolean execute(String sql) 
              执行给定的 SQL 语句，该语句可能返回多个结果。 如果为查询语句，返回true，如果为操作语句，返回false；执行该语句后可以通过getResultSet()和getUpdateCount()来获取执行结果
* ResultSet getResultSet() 
              以 ResultSet 对象的形式获取当前结果。 

* int getUpdateCount() 
              以更新计数的形式获取当前结果；如果结果为 ResultSet 对象或没有更多结果，则返回 -1。 

### 批处理

* void addBatch(String sql) 
              将给定的 SQL 命令添加到此 Statement 对象的当前命令列表中。 

* int[] executeBatch() 
              将一批命令提交给数据库来执行，如果全部命令执行成功，则返回更新计数组成的数组。 

*  void clearBatch() 
              清空此 Statement 对象的当前 SQL 命令列表。 

# PreparedStatement

## 描述

表示预编译的 SQL 语句的对象。 

## 方法

### 给SQL框架传值

 void setString(int parameterIndex, String x) 
          将指定参数设置为给定 Java String 值 

 void setInt(int parameterIndex, int x) 
          将指定参数设置为给定 Java int 值。 

### 执行语句

* ResultSet executeQuery() 
              在此 PreparedStatement 对象中执行 SQL 查询语句，并返回该查询生成的 ResultSet 对象。 
*  int executeUpdate() 
              在此 PreparedStatement 对象中执行 SQL 更新语句，



# ResultSet

## 描述

表示数据库结果集的数据表，通常通过执行查询数据库的语句生成。 

## 字段

* static int TYPE_FORWARD_ONLY 
              该常量指示光标只能向前移动的 ResultSet 对象的类型。 
* static int TYPE_SCROLL_INSENSITIVE 
              该常量指示可滚动但通常不受 ResultSet 底层数据更改影响的 ResultSet 对象的类型。 
* static int TYPE_SCROLL_SENSITIVE 
              该常量指示可滚动并且通常受 ResultSet 底层数据更改影响的 ResultSet 对象的类型。 
* static int CONCUR_READ_ONLY 
              该常量指示不可以更新的 ResultSet 对象的并发模式。 (只读)
* static int CONCUR_UPDATABLE 
              该常量指示可以更新的 ResultSet 对象的并发模式。 （可修改数据）



字段常用的三种组合方式：

* ResultSet.TYPE_FORWARD_ONLY 和 ResultSet.CONCUR_READ_ONLY ：（默认）只读不支持向回滚动
* ResultSet.TYPE_SCROLL_INSENSITIVE 和 ResultSet.CONCUR_READ_ONLY  ：只读，支持向回滚动
* ResultSet.TYPE_SCROLL_SENSITIVE 和 ResultSet.CONCUR_UPDATABLE ：支持向回滚动，支持对数据修改 

## 方法

### 控制光标

**注意：**默认的 ResultSet 对象不可更新，仅有一个向前移动的光标。因此，只能迭代它一次，并且只能按从第一行到最后一行的顺序进行。

通过一下代码可以生成可滚动和/或可更新的 `ResultSet` 对象。

~~~java
Statement stmt = con.createStatement(
 ResultSet.TYPE_SCROLL_INSENSITIVE,ResultSet.CONCUR_UPDATABLE);
ResultSet rs = stmt.executeQuery("SELECT a, b FROM TABLE2");
~~~

* boolean absolute(int row) 
    将光标移动到此 ResultSet 对象的给定行编号。 

* boolean next() 
          将光标从当前位置向前移一行。 

* boolean previous() 
          将光标移动到此 ResultSet 对象的上一行。 

* boolean first() 
          将光标移动到此 ResultSet 对象的第一行。 

*  boolean last() 
          将光标移动到此 ResultSet 对象的最后一行。 

*  void beforeFirst() 
          将光标移动到此 ResultSet 对象的开头，正好位于第一行之前。 

* void afterLast() 
     将光标移动到此 ResultSet 对象的末尾，正好位于最后一行之后。 

### 读取数据

* int getRow() 
              获取当前行编号。 

* String getString(int columnIndex) 
              以 Java 编程语言中 String 的形式获取此 ResultSet 对象的当前行中指定列的值。 

* String getString(String columnLabel) 
              以 Java 编程语言中 String 的形式获取此 ResultSet 对象的当前行中指定列的值。 

* int getInt(int columnIndex) 
              以 Java 编程语言中 int 的形式获取此 ResultSet 对象的当前行中指定列的值。 
     int getInt(String columnLabel) 
              以 Java 编程语言中 int 的形式获取此 ResultSet 对象的当前行中指定列的值。 

* Date getDate(int columnIndex) 
              以 Java 编程语言中 java.sql.Date 对象的形式获取此 ResultSet 对象的当前行中指定列的值。 

*  Date getDate(String columnLabel) 
              以 Java 编程语言中的 java.sql.Date 对象的形式获取此 ResultSet 对象的当前行中指定列的值。 

### 更新数据

**注意**：在生成resultSet时需要指定参数ResultSet.CONCUR_UPDATABLE

* void updateString(int columnIndex, String x) 
              用 String 值更新指定列。 
* void updateString(String columnLabel, String x) 
              用 String 值更新指定列。 

* void updateInt(int columnIndex, int x) 
              用 int 值更新指定列。 
* void updateInt(String columnLabel, int x) 
              用 int 值更新指定列。 

上述介绍的修改方法都只是在内存中修改，并没有实际修改数据库中的方法，如果想要修改数据库中的数据，需要对应的方法实现。

* void updateRow() 
              用此 ResultSet 对象的当前行的新内容更新底层数据库。 
