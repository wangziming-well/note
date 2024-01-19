# MySQL文件概述

MySQL数据库和InnoDB存储引擎表有各种类型文件。这些文件主要有：

* 参数文件：告诉MySQL实例启动时到哪里可以找到数据库文件，并指定某些初始化参数
* 日志文件：用来记录MySQL实例对某些条件做出响应时写入的文件，如错误日志文件、二进制日志文件、慢查询日志文件、查询日志文件等。
* socket文件：当用UNIX域套接字方式进行连接时需要的文件
* pid文件：MySQL实例的进程ID文件
* MySQL表结构文件：用来存放MySQL表结构定义文件
* 存储引擎文件：每个存储引擎都有自己的文件来保存各种数据。这些存储引擎存储了记录和索引等数据。我们主要介绍InnoDB相关的。

大部分文件都存储在参数`datadir`参数指定的文件夹中，如error log，binlog，slow log，pid文件和InnodB的表空间文件和undo等

# 参数文件

之前已经介绍过，当MySQL实例启动时，数据库会读取一个配置参数文件，用来寻找数据库的各种文件的位置以及指定某些初始化参数。在默认情况下，MySQL实例会按照一定的顺序在指定的位置进行读取，可以通过`mysql --help | grep my.cnf`来寻找

如果MySQL找不到任何参数文件，此时MySQL仍然能正常启动，只是所有的参数值取决于编译MySQL时指定的默认值和源代码中指定的参数的默认值。如果在这些默认值下找不到mysql架构，这是才会启动失败

## 参数

数据库参时可以看作是一个键值对，它们由等号连接，例如`innodb_buffer_pool_size=1G`。

可以通过命令`show variables`查看数据库中的所有参数，也可以用`like`过滤参数名。从MySQL5.1开始，也可以通过`information_schema`架构下的`global_variables`视图来查看所有参数。

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
show variables like 'log_error';
~~~

在默认情况下错误文件的文件名为服务器的`主机名.err`。可以通过`system hostname`查看系统主机名。

## 慢查询日志

慢查询日志(slow log)可以帮助DBA定位可能存在问题的SQL语句，从而进行SQL语句层面的优化。

可以在MySQL启动时设置一个阈值，将运行时间超过该阈值的所有SQL语句都记录到慢查询日志文件中。该阈值可以通过参数`long_query_time`来设置，默认值为10，代表10s

在默认情况下，MySQL数据库不启动慢查询日志，需要将参数`log_slow_queries`设置为ON。

还有一个相关参数`log_queries_not_using_indexes`如果这个参数值为ON，那么运行的SQL没有索引时，数据库同样会将这条SQL语句记录到慢查询日志中

MySQL5.6.5版本开始，新增了参数`log_throttle_quries_not_using_indexes`，用来表示每分钟允许记录到slow log的且未使用索引的SQL语句次数，默认为0，表示没有限制。

当slow log 中的SQL记录非常多时，直接分析文件就显得不那么简单和直观。所以MySQL数据库提供了`mysqldumpslow`命令，可以很好的解决这个问题，例如：

~~~cmd
mysqldumpslow -s al -n 10 david.log #表示查询执行时间最长的10条SQL
~~~

MySQL5.1开始，支持将慢查询日志记录放到表中，可以通过`log-output`指定慢查询输出的格式，默认为`FILE`，可以将其设置为`TABLE`，这样就可以查询mysql框架下的`slow_log`表了。

MySQL的slow log通过运行时间来对SQL语句进行捕获，而InnoSQL版本加强了对SQL语句的捕获方式。在原版MySQL的基础上在slow log中增加了对逻辑读取和物理读取的统计。

这里逻辑读取是指从磁盘进行IO读取的次数，逻辑读取包含所有的读取，不管是从磁盘还是缓冲区。

可以通过额外的参数`long_query_io`将超过指定逻辑IO次数的SQL记录到slow log，默认值为100.

还添加了`slow_query_type`用来表示启用slow log的方式，可选值有：

* 0：表示不将SQL语句记录到 slow log
* 1：表示根据运行时间进行记录
* 2：表示根据逻辑IO次数进行记录
* 3：表示根据运行时间和逻辑IO次数进行记录

## 查询日志

查询日志记录所有对MySQL数据库请求的信息，无论这些请求是否得到了正确的执行。默认文件名为`主机名.log`

同样地，从MySQL5.1开始，可以将查询日志的记录放入`mysql`框架下的`general_log`表中，方法和之前的`slow_log`基本一样。

## 二进制日志

二进制日志(binary log)记录了所有对MySQL数据库执行更改的所有操作。比如update、delete、insert操作。

但是，如果一个update操作没有导致数据库发生变化，这样的操作仍然可能会被写入二进制文件。

binlog主要有以下几种租用：

* 恢复：某些数据的恢复需要用到二进制文件
* 复制：在主从库的分布式系统重，主库通过复制和执行二进制日志和从库的数据进行同步
* 审计：用户可以通过对二进制日志中的信息来进行审计，判断是否有对数据库进行注入的攻击。

通过配置参数`log_bin[=name]`可以启动二进制日志。如果不指定name，则默认的二进制日志文件名为主机名。后缀为二进制日志的序列号。二进制日志文件的所在路径是数据库所以的路径。

有一个相关的`.index`文件，是二进制的索引文件，用来存储过往产生的二进制日志序号，通常情况下不建议收工修改这个文件。



下面介绍影响二进制日志记录的信息和行为的配置文件参数

### `binlog_cache_size`

`binlog_cache_size`：当使用事务的表存储引擎时，如InnoDB，所有未提交的二进制日志都会被记录到一个缓存中去，等该事务提交时直接将缓冲中的数据写入二进制文件，而该缓冲的大小就是由这个参数决定的，默认大小为32k。

此外，`binlog_cache_size`是基于会话的，即当一个线程开始一个事务时，MySQL会自动分配一个大小为`binlog_cache_size`的缓存，所以该值的设置需要相当小心，不能设置的过大。

当一个事务的记录大于设定的`binlog_cache_size`时，MySQL就会把缓冲中的日志写入磁盘的临时文件中，所以此时事务的性能就会下降。所以这个值夜不能设置的过小。

通过`SHOW GLOBAL STATUS`查看`binlog_cache_use`和`binlog_cache_disk_use`状态，可以判断当前的值设置的是否合适。

* `binlog_cache_use`记录了使用缓冲写二进制日志的次数
* `binlog_cache_disk_use`记录了使用磁盘临时文件写二进制日志的次数

### `sync_binlog`

`sync_binlog`在默认情况下，二进制日志并不是在每次事务提交的时候同步到磁盘的，所以，当数据库发生故障时，可能会有最后一部分数据没有写入二进制日志文件中，这会给恢复和复制带来问题。参数`sync_binlog=[N]`表示二进制日志文件同步磁盘的策略。

* 将N设置为1，表示采用同步写磁盘的方式来写二进制，即每次日志提交都进行同步磁盘
* 将N设置为0，表示仅当二进制日志的缓冲区满时，或者某些特定操作(关闭服务器)时，才会写入二进制

N为1是固然会带来高可用性，但会对数据库的IO系统带来一定的负担。

但是即使将`sync_binlog`设置为1，还可能出现问题：在一个事务发出COMMIT动作之前，会立即将二进制日志写入磁盘。此时已经写入了binlog，但是提交还未发生，若此时发生宕机，数据库下次启动时，由于COMMIT操作还没有发生，这个事务就会被回滚掉。但是二进制日志已经记录了该事务信息，不能被回滚。这个问题可以通过将参数`innodb_support_xa`设为1来解决，虽然这个参数与XA事务有关，但它也同时保证了二进制日志文件和InnoDB数据文件的同步

### `binlog_format`

`binlog_format`该参数影响记录二进制日志的格式。

这个值很重要，默认情况下，binlog记录的文件格式是基于SQL语句(statement)的，这种格式在主从复制架构下会产生主从数据库数据不一致的问题。例如sql语句中使用了`uuid()`这种两次运行结果不相等的值，这样在从库复制时，和主库的数据就不一样了。

参数有三个可选的值：

* statement ：记录的是日志的逻辑SQL语句
* row：记录表的行更改情况。同时，解决了主从复制场景下数据不一致的问题。
* mixed：默认会采用statement格式进行记录，但在下面情况下会使用row格式：
  * 表的存储引擎为NDB
  * 使用了`UUID()`、`USER()`、`CURRENT_USER()`、`FOUND_ROWS()`、`ROW_COUNT()`等不确定函数
  * 使用了 `INSERT DELAY`语句
  * 使用了用户定义函数(UDF)
  * 使用了临时表

binlog_format是动态参数，可以在数据库运行时进行更改。通常情况下，我们将这个值设置为row，这样为数据库的恢复复制带来更好的可靠性。但是会增加binlog文件的大小。因此在主从复制时也会同样增加网络的开销。

因为binlog的文件格式是二进制，所以无法直接使用cat、tail命令查看，必须使用MySQL提供的工具`mysqlbinlog`查看

### 其他配置参数

* `max_binlog_size`:指定了单个二进制文件的最大值，如果超过该值，就产生新的二进制文件，后缀名+1，并记录到`.index`文件。默认值为1073741824，代表1G
* `binlog-do-db`表示需要写入哪些库的日志，默认为空，表示同步所有库的binlog
* `binlog-ignore-db`表示忽略写入哪些库的binlog
* `log-slave-update`，在主从复制结构中，如果当前数据库是slave角色，则它不会把 从master取得并执行的binlog写入自己的binlog中。如果需要写入，则要设置该参数。在`master->slave->slave`架构中，中间的`slave`必须设置该参数

# InnoDB存储引擎文件

每个存储引擎都有自己独有的文件。InnoDB也同样如此，与InnoDB相关的文件有表空间文件、redo log文件

## 表空间文件

InnoDB采用将存储的数据按表空间(tablespace)进行存放的设计。

默认配置下会有一个初始大小为10MB，名为idbdata1的文件。该文件就是默认的表空间文件，可以通过`innodb_data_file_path`对其进行设置，格式如下：

~~~properties
innodb_data_file=datafile_specl[;datafile_spec2]...
~~~

可以通过多个文件组成一个表空间，同时指定文件的属性，例如：

~~~properties
innodb_data_file= /db/ibdata1:2000M; /dr2/db/ibdata2:2000M:autoextend
~~~

用两个文件组成表空间。若这两个文件在不同的磁盘上，磁盘的负载可能被平均，因此可以提高数据库的整体性能。

这两个文件后面也更了路径，2000M表示ibdata1文件的大小，autoextend表示用完了2000M后，该文件可以自动增长

所有基于InnoDB的表的数据都会记录到ibdata的共享表空间中。

若设置了参数`innodb_file_per_table=ON`，则每个InnoDB表都会产生一个独立表空间文件。这个文件的命名规则是`表名.ibd`，这个单独的表空间文件仅存储该表的数据、索引和插入缓冲BITMAP等信息，其余信息仍然存储在默认的表空间。

InnoDB对表的存储方式如下图：

![image-20240117212619737](https://gitee.com/wangziming707/note-pic/raw/master/img/InnoDB%E8%A1%A8%E5%AD%98%E5%82%A8%E5%BC%95%E6%93%8E%E6%96%87%E4%BB%B6.png)

## redo log文件

redo log 用来保证事务的原子性和数据的完整性：InnoDB的事务失败后回滚是通过redo log的，数据库宕机重启恢复是通过redo log恢复到故障前的状态的。

每个InnoDB至少有一个redo log 文件组，每个文件组下至少有两个redo log文件。为了得到更高的可靠性，可以设置多个镜像日志组，将不同的文件组放在不同的磁盘上，以提高redo log的高可用性。

在日志组中的每个redo log文件的大小一致，并以循环写入的方式运行：InnoDB先写redo log文件1，写满后会切换到redo log文件2，文件2也被写满后，会再切回到redo log 文件1中。

下面参数影响重做日志文件的属性：

**`innodb_log_file_size`**

该参数指定redo log 文件大小，在InnoDB1.2.x版本之前，redo log大小不等大于等于4GB，1.2.x版本将该限制扩大为了512G。

redo log 文件不能设置的太大，否则恢复时可能需要很长时间；但是也不能设置得太小，否则可能导致一个事务的日志需要多次切换redo log文件。也可能导致频繁发生async checkpoint，导致性能抖动。

**`innodb_log_files_in_group`**

指定日志文件组种重做日志文件的数量，默认为2

**`innodb_mirrored_log_groups`**

指定日志镜像文件组的数量，默认为1，表示只有一个日志文件组，没有高可用。

如果磁盘本身已经做了高可用方案，如RAID磁盘阵列，那么可以不开启这个功能。

**`Innodb_log_group_home_dir`**

指定日志文件组所在路径，默认为`./`，表示在数据库的data目录下。

#### redo log 结构

在InnoDB种，不同的操作有着不同的redo log格式，到InnoDB1.2.x为止，共定义了51种redo log 类型。这些类型都有着基本的格式，下面显示了redo log 条目的结构：

![redologEntryStructure](https://gitee.com/wangziming707/note-pic/raw/master/img/redologEntryStructure.png)

可以看到redo log条目由4部分组成：

* redo_log_type 占一字节，表示redo log的类型
* space表示表空间ID，采用压缩的方式存储，占用的空间可能小于4字节
* page_no，表示页的偏移量，同样采用压缩的方式
* redo_log_body表示每个redo log条目的数据部分，恢复时需要调用相应的函数进行解析

我们已经知道了redo log文件的操作不是直接写，而是先写入一个 redo log buffer 种，然后按照一定的触发条件顺序写入日志文件。

这一定的触发条件第一个是主线程每秒都会将redo logb 缓冲刷入磁盘，不论事务是否由提交。

而另一个触发条件是由参数`innodb_flush_log_at_trx_commit`控制，表示在提交时，处理redo log的方式，参数有以下值：

* 0:提交事务时，不冲刷redo log buffer
* 1：在执行commit前同步冲刷redo log buffer，即执行commit时，必定冲刷了缓冲
* 2：在执行commit时，异步执行冲刷redo log buffer的操作，也就是执行commit时，不保证已经写入了redo log文件，只是有这个动作发生。

为了保证事务的持久性，必须将`innodb_flush_log_at_trx_commit`参数设置为1

# 其他文件

## Socket文件

在UNIX系统下本地连接MySQL可以采用UNIX Socket方式，这需要一个Socket文件。文件位置由参数`socket`控制。一般在`/tmp/mysql.sock`

## pid文件

当MySQL实例启动时，会将自己的进程ID写入一个文件中，该文件即为pid文件。

文件位置由参数`pid_file`控制，默认位于数据库目录下，文件名为`主机名.pid`

## 表结构定义文件

MySQL数据的存储是根据表进行的，每个表都有与之对应的文件。不论表采用何种存储引擎，MySQL都有一个frm后缀的文件，这个文件记录了该表的表结构定义。注意，创建的视图也有对应的frm文件

这些frm文件通常位于数据库目录的data文件夹下。

