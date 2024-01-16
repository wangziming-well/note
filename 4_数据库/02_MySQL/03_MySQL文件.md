# MySQL文件概述

MySQL数据库和InnoDB存储引擎表有各种类型文件。这些文件主要有：

* 参数文件：告诉MySQL实例启动时到哪里可以找到数据库文件，并指定某些初始化参数
* 日志文件：用来记录MySQL实例对某些条件做出响应时写入的文件，如错误日志文件、二进制日志文件、慢查询日志文件、查询日志文件等。
* socket文件：当用UNIX域套接字方式进行连接时需要的文件
* pid文件：MySQL实例的进程ID文件
* MySQL表结构文件：用来存放MySQL表结构定义文件
* 存储引擎文件：每个存储引擎都有自己的文件来保存各种数据。这些存储引擎存储了记录和索引等数据。我们主要介绍InnoDB相关的。

# 参数文件

之前已经介绍过，当MySQL实例启动时，数据库会读取一个配置参数文件，用来寻找数据库的各种文件的位置以及指定某些初始化参数。在默认情况下，MySQL实例会按照一定的顺序在指定的位置进行读取，可以通过`mysql --help | grep my.cnf`来寻找

如果MySQL找不到任何参数文件，此时MySQL仍然能正常启动，只是所有的参数值取决于编译MySQL时指定的默认值和源代码中指定的参数的默认值。如果在这些默认值下找不到mysql架构，这是才会启动失败

## 参数

数据库参时可以看作是一个键值对，它们由等号连接，例如`innodb_buffer_pool_size=1G`。

可以通过命令`show variables`查看数据库中登所有参数，也可以用`like`过滤参数名。从MySQL5.1开始，也可以通过`information_schema`架构下的`global_variables`视图来查看所有参数。

## 参数类型

MySQL数据库中的参数可以分为两类：

* 动态(dynamic)参数：在MySQL实例运行期间可以进行更改
* 静态(static)参数：在整个实例生命周期内不得进行更改。

可以通过set命令对动态的参数值进行修改，其语法如下：

~~~mysql
SET
| [globle | session] system_var_name = expr
| [@@globle. | @@session. | @@]system_var_name = expr
~~~

可以看到`session`和`globle`关键字，他们表明参数的修改是只在当前会话生效还是整个实例的生命周期生效。

有些参数只能在会话中修改，如`autocommit`，而有些参数修改完后，在整个实例生命周期中都会生效，如`binlog_cache_size`；而有些参数即可以在会话中又可以在整个实例的生命周期内生效，如`read_buffer_size`。例如：

~~~mysql
set read_buffer_size = 524288;
set @@session.read_buffer_size=1048576;
~~~

需要注意，不管是`session`和`globle`关键字，都不会修改参数文件。下次重新启动MySQL实例时，还是会读取参数文件。所以若想让MySQL的参数值永久修改，就必须修改参数文件。

# 日志文件

日志文件记录了影响MySQL数据库的各种类型活动。MySQL数据库中常见的日志文件有：

* error log错误日志
* binlog 二进制日志
* slow query log 慢查询日志
* log 查询日志

这些日志文件可以帮助DBA对MySQL数据库的运行状态进行诊断

## 错误日志

错误日志文件对MySQL的启动、运行、关闭过程进行了记录。该文件不仅记录了所有的的错误信息，也记录一些警告和正确的信息。

可以通过下面命令定位到该文件：

~~~mysql
show variables like 'log_error'\G;
~~~

在默认情况下错误文件的文件名为服务器的主机名。可以通过`system hostname`查看系统主机名。

## 慢查询日志

慢查询日志(slow log)可以帮助DBA定位可能存在问题的SQL语句，从而进行SQL语句层面的优化。

可以在MySQL启动时设置一个阈值，将运行时间超过该阈值的所有SQL语句都记录到慢查询日志文件中。该阈值可以通过参数`long_query_time`来设置，默认值为10，代表10s

在默认情况下，MySQL数据库不启动慢查询日志，需要将参数`log_slow_queries`设置为ON。



