# InnoDB存储引擎概述

InnoDB是事务安全的MySQL存储引擎。通常来说，InnoDB引擎是OLTP应用中核心表的首选存储引擎。它也是目前MySQL的默认存储引擎。

MySQL数据库的发行版本中本身就包含InnoDB存储引擎。早期其版本随MySQL数据库的更新而更新。从MySQL5.1版本开始，MySQL数据库允许存储引擎允许开发商以动态方式加载引擎，这样存储引擎的更新就可以不受MySQL数据库版本的限制。

所以现在MySQL可以支持两个版本的InnoDB，一个是静态编译的InnoDB版本，可以将其视为老版本的InnoDB；另一个是动态加载的InnoDB版本，官方称为InnoDB Plugin可以将其视为InnoDB1.x.x版本。下面是各个版本支持的功能：

* 老版本InnoDB：支持ACID，行锁，MVCC
* InnoDB1.0.x:新增 compress和dynamic页格式
* InnoDB1.1.x:新增了Liunx AIO、多段回滚
* InnoDB1.2.x:新增全文索引支持、在线索引添加

# InnoDB体系架构

以下是InnoDB的简单架构：

![image-20240111145257488](https://gitee.com/wangziming707/note-pic/raw/master/img/InnoDB%E6%9E%B6%E6%9E%84.png)

由图可知，InnoDB有多个内存块，这些内存块组成了一个更大的内存池，负责如下工作：

* 维护所有进程/线程需要访问的多个内部数据结构
* 缓存磁盘上的数据，方便快速读取
* 重做日志缓冲

等。

后台线程的主要作用是负责刷新内存池中的数据，保证缓冲池中的内存缓存时最新的数据。此外将已修改的数据文件刷到磁盘文件，同时保证数据库发送异常时InnoDB能恢复正常。

## 后台线程

InnoDB时多线程的，有多个不同的后台线程，负责处理不同的任务。

* Master Thread：核心后台线程，主要负责将缓冲池中的数据异步刷新到磁盘，保证数据一致性，包括脏页的刷新、合并插入缓冲、UNDO页的回收等

* IO Thread：InnoDB大量使用AIO(Async IO)来处理写IO请求，这样可以极大提高数据库的性能。而IO Thread 主要负责这些IO请求的回调处理。有四种IO Thread，分别是write、read、insert buffer和lob IO thread。

  可以通过`SHOW ENGINE INNODB STATUS`来观察InnoDB中的IO Thread

* Purge Thread：事务被提交后，其undolog可能不再需要，所以需要PurgeThread来回收已经使用并分配的undo页。

  在InnoDB1.1版本之前，purge操作仅在MasterThread中完成。在1.1之后，purge操作可以独立到单独的线程进行，以减轻Master Thread 的压力。可以在配置文件中添加如下命令来启用独立的Purge Thread：

  ~~~nginx
  [mysqlId]
  innodb_purge_threads=1
  ~~~

  在InnoDB1.1中，innodb_purge_threads只能是1，否则将报错，在InnoDB1.2后，InnoDB支持多个Purge Thread，以加快undo页的回收。此时可以设置多个innodb_purge_threads了

* Page Clear Thread：这是在InnoDB1.2.x版本中引入的，用于将之前版本中脏页的刷新操作放入单独的线程中来完成。目的是为了减轻原Master Thread的工作量和对于用户查询线程的阻塞，进一步提高InnoDB的性能。

## 内存

### 缓冲池

InnoDB是基于磁盘存储的，将其中的记录按照页的方式进行管理。但是由于CPU速度和磁盘速度的鸿沟，基于磁盘的数据库系统通常引入缓冲池技术来提高数据库整体性能。缓冲池是一块内存区域，通过内存的速度来弥补磁盘速度较慢对数据库性能的影响。

在数据库中进行读取页的操作时，首先将从磁盘读到的页放到缓冲池中，这个过程称为将页“FIX”在缓冲池中。下一次再读相同的页时，会先判断该页是否在缓冲池中。如果还在，则缓存命中，直接在缓冲池中读取该页，否则，读取磁盘上的页。

在数据库中进行页的修改操作时，首先修改在缓冲池中的页，然后再以一定的频率刷新到磁盘上。需要注意，页从缓冲池刷新回磁盘的操作不是再每次页发送更新时触发，而是通过Checkpoint机制刷新回磁盘。这同样是为了提高数据库的整体性能。

对InnoDB来说，其缓冲池的配置通过参数`innodb_buffer_pool_size`来设置。

具体来看，缓冲池中缓存的数据页类型有：索引页、数据页、undo页、插入缓冲、自适应哈希索引、InnoDB存储的锁信息、数据字典信息等。如图所示：

![image-20240111153820307](https://gitee.com/wangziming707/note-pic/raw/master/img/InnoDB%E7%BC%93%E5%86%B2%E6%B1%A0%E5%86%85%E5%AE%B9.png)

从InnoDB1.0.x版本开始，允许有多个缓冲池实例。每个页根据哈希值平均分配到不同缓冲池实例中。这样可以减少数据库内部的资源竞争，增加数据库的并发能力。缓冲池实例个数可以通过参数`innodb_buffer_pool_instances`来配置，默认为1

### LRU List

InnoDB需要对缓冲池的内存区域进行管理。

通常来说，数据库中的缓冲池通过LRU(Lasted Recent Used)算法来进行管理。即最频繁使用的页再LRU列表前端，最少使用的页再LRU列表的尾端。当缓冲池不再存放新读取到的页时，首先释放LRU列表中尾端的页。

再InnoDB中，缓冲池中页的大小默认为16KB,同样使用LRU算法对缓冲池进行管理。但是在此基础上进行了一些优化。

LRU列表中还加入了midpoint位置。新读取到的页，最然是最新访问的页，但是不是直接放入到LRU列表的首部，而是放入到midpoint位置。这个算法再InnoDB存储引擎下称为`midpoint insertion strategy`。再默认配置下，该位置再LRU列表长度的5/8处。midpoint位置可以由`innodb_old_blocks_pct`控制，其值为两位整数，比如37标识插入到LRU列表尾端的37%位置。

在InnoDB中，把midpoint之后的表称为old表，之前的表称为new列表。new表中的页都是最为活跃的热点数据。

这样将新访问的页插入到LRU列表的中间而不是首部，是为了防止一些SQL操作将缓冲池中的热点数据刷新出去，从而影响缓冲池的效率。例如索引或数据的扫描操作，这需要访问许多页。这些页通常是仅仅当次查询操作中需要，不是热点数据。但如果这些页被插入LRU列表的首部，那么很有可能将热点数据页从LRU列表中移除。那么数据库在反复问这些热点数据页时，需要再次访问磁盘。

为了解决这个问题，InnoDB引入了另一个参数来进一步管理LRU列表：`innodb_old_blocks_time`，用于表示页读取到mid位置后需要等待多久才会被加入到LRU列表的热端。也就是说，第一次新读取的页会被放入old表首部，只有在`innodb_old_blocks_time`时间内再次被访问，这个页才会被移动到new表首部，否则这个页将一直留在old表中。这个值默认为1000ms

当页从LRU列表的old部分加入到new部分时，此时发送的操作为page made young，而因`innodb_old_blocks_time`的设置而导致页没有从old部分移动到new部分的操作称为page not made young

可以通过命令`SHOW ENGINE INNODB STATUS`来观察LRU列表和Free列表的使用情况和运行状态

从InnoDB1.2版本开始，也可以通过`infomation_schema.INNODB_BUFFER_POOL_STATS`表来观察缓冲池的状态

### Free List & Flush List

LRU列表用来管理已经读取的页，但当数据库刚启动时，LRU列表是空的，没有任何页。此时所有页都存放在Free List中。当需要从缓冲池中分页时，首先从Free列表中查找是否有可用的空闲页，若有则将该页从Free列表删除，放入LRU列表中。否则，根据LRU算法，淘汰LRU列表末尾的页，将该内存空间分配给新的页。

在LRU列表中的页被修改后，该页就变成了脏页，即缓冲池中页和磁盘中的页的数据产生了不一致。这时数据库会通过checkpoint机制将脏页刷新回磁盘，而Flush列表中的页就是脏页列表。需要注意，脏页即存在于LRU列表，也存在于Flush列表。两个列表互不影响

### redo log buffer

从缓冲的结构图可以看到，InnoDB的内存区域除了有缓冲池外，还有重做日志缓冲。



