# MySQL概述

Mysql是一个可移植的数据库，几乎在当前所有系统上都能运行。

## 数据库和实例

在数据库领域，数据库和数据库实例是两个不同的概念：

* 数据库：物理操作系统文件或者其他形式文件类型的集合。在MySQL数据库中，数据库文件可以是frm、MYD、MYI、ibd结尾的文件
* 实例：MySQL数据库由后台线程以及一个共享内存区域组成。数据库实例时用来操作数据库文件的。

在MySQL中，实例和数据库的关系通常是一一对应的，即一个实例对应一个数据库，一个数据库对应一个实例。但在集群情况下，可能存在一个数据库被多个数据实例使用的情况。

MySQL是一个单进程多线程架构的数据库，MySQL数据库实例在系统上的表现就是一个进程。

##  配置文件

Mysql的配置文件是`my.cnf`，一个Mysql可以在不同位置有多个配置文件，比如在liunx环境下输入`mysql --help | grep m.cnf`输出为：

~~~cmd
		order of preference, my.cnf, $MYSQL_TCP_PORT,
/etc/my.cnf /etc/mysql/my.cnf /user/etc/my.cnf ~/.my.cnf
~~~

可以看到，MySQL数据库是按照`/etc/my.cnf` 、 `/etc/mysql/my.cnf` 、`/user/etc/my.cnf` 、`~/.my.cnf`的顺序在这几个位置读取配置文件的。如果有几个配置文件都配置了相同的参数，MySQL数据库以最后一个配置文件中的参数为准。在Liunx环境下，配置文件一般放在`/etc/my.cnf`下

# MySQL体系结构

MySQL数据库的体系结构如图：

![MySQL体系结构图](https://gitee.com/wangziming707/note-pic/raw/master/img/MySQL%E4%BD%93%E7%B3%BB%E7%BB%93%E6%9E%84%E5%9B%BE.jpg)

由此可知，MySQL由如下几个部分组成：

* 连接池组件
* 管理服务和工具组件
* SQL接口组件
* 查询分析器组件
* 优化器组件
* 缓冲组件
* 插件式存储引擎
* 物理文件

MySQL数据库区别于其他数据库的重要特点是其插件式的表存储引擎。这个存储引擎是基于表的，而不是数据库。

# MySQL存储引擎

我们已经知道了MySQL结构体系最特殊的是插件式的存储引擎。这样的好处是，每个存储引擎都有各自的特点，可以根据具体的应用建立不同的存储引擎表。接下来简单介绍各种存储引擎。

可以通过`SHOW ENGINES`语句查询当前使用的MySQL数据库支持的存储引擎，也可以通过查找information_schema架构下的`ENGINES`表

## InnoDB存储引擎

InnoDB存储引擎支持事务，其设计目标主要面向在线事务处理的应用。其特点是行锁设计、支持外键，并支持非锁定读，即默认读取操作不会产生锁。

从MySQL5.5.8开始，InnoDB是默认的存储引擎。

InnoDB存储引擎将数据放在一个逻辑的表空间中。表可以单独存放到一个单独的idb文件中。

InnoDB通过使用多版本并发控制(MVCC)来获得高并发性，并实现了SQL标准的4种隔离级别，默认为可重复读级别。同时，使用一种被称为`next-key-locking`的策略来避免幻读现象。此外，InnoDB还提供了插入缓冲，二次写、自适应哈希索引、预读等高性能和高可用的功能。

InnoDB采用聚集的方式来存储表种数据，因此每张表的存储都是按主键的顺序进行存放。如果没有显式地定义表的主键，InnoDB会为每一行生成一个6字节的ROWID作为主键

## MyISAM存储引擎

MyISAM存储引擎不支持事务、表锁设计，支持全文索引，主要面向一些分析型OLAP(Online Analytical Processing)数据库应用。

因为分析性数据库通常不需要删除、修改数据，只需要查询，所以MyISAM不支持事务是核里的。并且它的缓冲池只缓存索引文件，而不缓冲数据文件。

MyISAM存储引擎由MYD和MYI组成，MYD用来存放数据文件，MYI用来存放索引文件。

MySQL5.0后MyISAM默认支持256TB的单表数据。

## NDB存储引擎

NDB存储引擎是一个集群存储引擎，使用share nothing的集群架构。其数据全部存放在内容中，因此主键查找的速度极快，可以通过添加NDB数据存储节点线性地提高数据库性能，是高可用、高性能的集权。

但是NDB的连接操作时在MySQL数据库层完成的，而不是在存储引擎层完成的，这意味着复杂的连接操作需要巨大的网络开销，所以查询速度会很慢。

## Memory存储引擎

Memory存储引擎将表中的数据存放到内存中，如果数据库发送重启或者崩溃，则表中的数据都将消失。它适合用于临时存储数据的临时表，以及数据仓库中的纬度表。

它默认使用哈希索引，而不是B+树索引

Memory因为数据存储在内存中，所以速度非常快，但是它只支持表锁，并发性能较差，并且不支持TEXT和BLOB列类型，因此会 浪费内存。

MySQL数据库使用Memory存储引擎作为临时表来存放查询的中间结果集，如果中间结果集大于Memory表的容量设置，又或者中间结果包含TEXT或BLOB列类型字段，则MySQL数据库会把其转换到MyISAM存储引擎而存放到磁盘中。这种情况下，查询性能会降低。

## Archive存储引擎

Archive存储引擎只支持INSERT和SELECT操作，也支持所以。它使用zlib算法将数据行压缩后存储，压缩比一般可达1：10。正如其名，Archive引擎适合存储归档数据，如日志信息。

Archive存储引擎使用行锁来实现高并发的插入操作，但其本身并不是事务安全的存储引擎，其设计目的时提供告诉的插入和压缩功能。

## Federated存储引擎

Federated存储引擎表本身不存放数据，而是对应一台远程MySQL数据库服务器上的表。

## Maria存储引擎

Maria存储引擎是新开发的引擎，用来取代原有的MyISAM存储引擎。其特点是：支持缓存数据和索引文件，应用行锁设计，提供MVCC功能，支持事务和非事务安全选项，更好的BLOB字符类型的处理性能。
