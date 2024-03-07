# 简介

在web服务器中，高可用是指服务器可以正常访问的时间，衡量的标准是在多长时间内可以提供正常服务。但是在Redis语境中，高可用的含义似乎要宽泛一些，除了保证提供正常服务(如主从分离、快速容灾技术)，还需要考虑数据容量的扩展、数据安全不会丢失等。

在Redis中，实现高可用的技术主要包括持久化、主从复制、哨兵和cluster集群：

* 持久化： 持久化是最简单的高可用方法(有时甚至不被归为高可用的手段)，主要作用是数据备份，即将数据存储在硬盘，保证数据不会因进程退出而丢失。
* 主从复制： 主从复制是高可用Redis的基础，哨兵和集群都是在主从复制基础上实现高可用的。主从复制主要实现了数据的多机备份(和同步)，以及对于读操作的负载均衡和简单的故障恢复。缺陷：故障恢复无法自动化；写操作无法负载均衡；存储能力受到单机的限制。
* 哨兵： 在主从复制的基础上，哨兵实现了自动化的故障恢复。(主挂了，找一个从成为新的主，哨兵节点进行监控)缺陷：写操作无法负载均衡；存储能力受到单机的限制。
* Cluster集群： 通过集群，Redis解决了写操作无法负载均衡，以及存储能力受到单机限制的问题，实现了较为完善的高可用方案。(6台起步，3主3从)

# 持久化

Redis支持RDB和AOF两种持久化机制，持久化功能有效地避免因进程退出造成的数据丢失问题，当下次重启时利用之前持久化的文件即可实现数据恢复。

## RDB

RDB(Redis Database)持久化是当前进程数据生成快照保存到硬盘的过程，触发RDB持久化过程分为手动触发和自动触发

### 触发机制

可以使用save和bgsave命令设置手动触发

* save命令：阻塞当前Redis服务器，直到RDB过程完成。当Redis实例内存很大时，会造成长时间阻塞，线上环境不建议使用
* bgsave命令：Redis进程执行fork操作创建子进程，RDB持久化过程由子进程负责，完成后自动结束。阻塞只发生再fork阶段，一般时间很短。

除了手动触发外，Redis内部还存在自动触发RDB持久化的机制，如下场景会自动触发bgsave：

* 配置文件中使用`save`配置，如`save m n`表示m秒内数据集存在n次修改时
* 如果从节点执行全量复制操作，主节点自动执行bgsave生成RDB文件并发送给从节点
* 执行debug reload命令重新加载Redis时
* 默认情况下执行shutdown命令时，如果没有开启AOF持久化功能则自动执行bgsave

### bgsave流程

bgsave是主流的触发RDB持久化方式，它的工作流程如下：

* 执行bgsave命令，Redis父进程判断当前是否存在正在执行的子进程，如RDB/AOF子进程，如果存在bgsave命令直接返回

* 父进程执行fork操作创建子进程，fork操作过程中父进程会阻塞；通过info stats命令查看latest_fork_usec选项，可以获取最近一个fork操作的时，单位为微秒 
* 父进程fork完成后，bgsave命令返回“Background saving started”信息并不再阻塞父进程，可以继续响应其他命令
* 子进程创建RDB文件，根据父进程内存生成临时快照文件，完成后对原有文件进行原子替换 ；执行lastsave命令可以获取最后一次生成RDB的时间，对应info统计的rdb_last_save_time选项  
* 进程发送信号给父进程表示完成，父进程更新统计信息，具体见info Persistence下的rdb_*相关选项  

### RDB文件的处理

RDB文件保存在`dir`配置指定的目录下，默认为`./`，文件名通过`dbfilename`配置指定

可以通过执行`config set dir newDir`和`config set dbfilename fileName`运行时动态更改配置。当遇到坏盘或者磁盘写满等情况时，可以通过上述命令修改文件路径到可用的磁盘路径。

Redis默认采用LZF算法对生成的RDB文件做压缩处理，压缩后的文件远远小于内存大小，默认开启。可以通过`rdbcompression {yes|no}`配置，也可以通过`config`命令动态修改。

虽然压缩RDB会消耗CPU，但是可以大幅降低文件的体积，方便保存到硬盘和通过网络发送到从节点，因此线上建议开启。

### 优缺点

优点：

* RDB是紧凑压缩的二进制文件，是Redis某个时间点上的数据快照，非常适合用于进行备份、灾备和全量复制的场景。
* Redis加载RDB回复数据远快于AOF的方式

* 是由子进程来处理生成RDB文件的工作的，主进程不进行任何的IO操作，不会影响redis的读写性能

缺点：

* 没有办法做到实时持久化，因为bgsave每次运行fock操作创建子进程属于重量级操作，不能频繁执行；所以最后一次持久化后的数据可能丢失，有一定的数据安全风险
* RDB使用特定二进制格式保存，新老版本Redis之间可能不兼容

## AOF

(Append-only File)AOF持久化：以独立日志的形式来记录每次写命令，重启时再重新执行AOF文件中的命令以达到恢复数据的目的。AOFn能够实时进行数据持久化，是Redis持久化的主流方式

### AOF持久化过程

开启AOF需要设置配置`appendonly yes`，默认不开启，AOF文件名通过`appendfilename`配置设置，默认文件名为`appendonly.aof`。保存路径也是通过`dir`配置指定。

AOF的工作流程如下:

* 客户端的请求写命令会被append追加到aof_buf缓冲区内
* AOF缓冲区根据AOF持久化策略将操作sync同步到磁盘的AOF文件中
* AOF文件大小超过重写策略或手动重写时，会对AOF文件rewrite重写，压缩AOF文件容量
* Redis服务重启时，会重新load加载AOF文件中的写操作达到数据恢复的目的

如果直接将命令写入硬盘，那么整个流程性能完全取决于硬盘性能。所以Redis先将命令写入缓冲区，之后再根据策略同步硬盘，并且Redis提供了多种同步硬盘的策略，再性能和安全方面做出平衡。

### 文件同步策略

在介绍策略之前，我们需要了解Liunx内核关于硬盘写的操作：

Liunx在内核提供页缓冲区来提高硬盘IO性能，系统调用write操作只将数据写入系统缓冲区后就直接返回，不管数据是否真的写入硬盘。而系统页缓冲区的同步策略依赖于系统调度机制，例如：缓冲区页空间写满或者达到特定时间周期。

而fsync针对单个文件操作做强制硬盘同步，fsync将阻塞直到写入硬盘后返回，确保了数据的持久化。

Redis提供了多种缓冲区同步文件的策略，由`appendfsync`配置控制：

* `always`命令写入aof_buf缓冲区后调用系统fsync操作进行同步。这样硬盘IO会成为写操作的性能瓶颈，不建议操作。
* `everysec`命令写入aof_buf后调用系统write操作。fsync的同步操作由专门线程每秒操作一次。是建议的同步策略，也是默认配置，兼顾了性能和安全性。理论上最多丢失1s的数据。
* `no`：命令写入aof_buf后调用系统write操作，不主动调用fsync操作，同步硬盘的步骤由操作系统来完成，通常同步周期最长为30s。虽然提高了性能，但是数据安全性完全无法保证。

### 重写机制

随着命令不断写入AOF，文件会越来越大，为了解决这个问题，Redis引入AOF重写机制压缩文件体积。AOF文件重写是把Redis进程内的数据转化为写命令同步到新AOF文件的过程。

重写后的AOF文件可以变小的原因有：

* 进程内已经超时的数据不再写入文件
* 旧的AOF文件含有无效命令，如`del`、`hdel`、`srem`等删除命令。重写使用进程内数据直接生成，这样新的AOF文件只保留最终数据的写入命令
* 多条写命令可以合并为一个，例如对列表的多次插入，可以合并为一条命令。

更小的的AOF文件不止能节省空间，还可以更快地被Redis加载。

AOF重写过程可以手动触发和自动触发：

* 手动触发，直接调用`bgrewriteaof`命令

* 当满足下面任意条件时，自动触发AOF重写：

  * aof_current_size >=auto-aof-rewrite-min-size

  * aof_base_size/aof_current_size>=auto-aof-rewrite-percentage

  其中auto-aof-rewrite-min-size和auto-aof-rewrite-percentage是配置参数，aof_current_size是当前AOF文件大小，aof_base_size是上一次重写后AOF文件的大小，可以info Persistence统计信息中查看

当触发AOF重写时，进行了一下流程：

* 执行AOF重写请求
  * 如果当前进程正在进行执行AOF重写，则请求不执行
  * 如果当前进程正在执行bgsave，重写命令延迟到bgsave完成之后执行
* 父进程执行fork创建子进程
* 主进程fork完成后，继续响应其他命令。所有修改命令仍然写入AOF缓冲区并根据策略同步到硬盘，保证原有的AOF机制正确执行

* 由于fork操作运用写时复制技术，子进程只能共享fork操作时的内存数据。由于父进程依然响应命令，Redis使用“AOF重写缓冲区”保存这部分新数据，防止新AOF文件生成期间丢失这部分数据  
* 子进程根据内存快照，按照命令合并规则写入到新的AOF文件。每次批量写入硬盘数据量由配置aof-rewrite-incremental-fsync控制，默认为32MB，防止单次刷盘数据过多造成硬盘阻塞
* 新AOF文件写入完成后，子进程发送信号给父进程，父进程更新统计信息，具体见info persistence下的aof_*相关统计
* 父进程把AOF重写缓冲区的数据写入到新的AOF文件
* 使用新AOF文件替换老文件，完成AOF重写

## 重启加载

AOF和RDB都可以用于重启后的数据回复，下面流程表示Redis持久化文件加载流程：

![Redis加载持久化文件流程](https://gitee.com/wangziming707/note-pic/raw/master/img/Redis%E5%8A%A0%E8%BD%BD%E6%8C%81%E4%B9%85%E5%8C%96%E6%96%87%E4%BB%B6%E6%B5%81%E7%A8%8B.png)

可以看到，AOF持久化开启并且存在AOF文件时，优先加载AOF文件。否则才会加载RDB文件。

## 问题定位和优化

Redis持久化功能一直是影响Redis性能的高发地，我们介绍常见的持久化问题进行分析定位和优化

### fork操作

当Redis做RDB或AOF重写时，一个必不可少的操作就是执行fork操作创建子进程，对于大多数操作系统来说fork是个重量级错误。虽然fork创建的子进程不需要拷贝父进程的物理内存空间，但是会复制父进程的空间内存页表。例如对于10GB的Redis进程，需要复制大约20MB的内存页表，因此fork操作耗时跟进程总内存量息息相关  

**问题定位**:对于高流量的Redis实例OPS可达5万以上，如果fork操作耗时在秒级别将拖慢Redis几万条命令执行，对线上应用延迟影响非常明显正常情况下fork耗时应该是每GB消耗20毫秒左右。可以在info stats统计中查latest_fork_usec指标获取最近一次fork操作耗时，单位微秒   

fork操作耗时优化：

* 优先使用物理机或者高效支持fork操作的虚拟化技术，避免使用Xen

* 控制Redis实例最大可用内存，fork耗时跟内存量成正比，线上建议每个Redis实例内存控制在10GB以内
* 合理配置Linux内存分配策略，避免物理内存不足导致fork失败
* 降低fork操作的频率，如适度放宽AOF自动触发时机，避免不必要的全量复制等

### 子进程开销

子进程负责AOF或者RDB文件的重写，它的运行过程主要涉及CPU、 内存、 硬盘三部分的消耗

#### CPU

子进程负责把进程内的数据分批写入文件，这个过程属于CPU密集操作，通常子进程对单核CPU利用率接近90%

优化：

* Redis是CPU密集型服务，不要做绑定单核CPU操作。由于子进程非常消耗CPU，会和父进程产生单核资源竞争
* 不要和其他CPU密集型服务部署在一起，造成CPU过度竞争
* 如果部署多个Redis实例，尽量保证同一时刻只有一个子进程执行重写工作

#### 内存

子进程通过fork操作产生，占用内存大小等同于父进程，理论上需要两倍的内存来完成持久化操作，但Linux有写时复制机制(copy-on-write)。父子进程会共享相同的物理内存页，当父进程处理写请求时会把要修改的页创建副本，而子进程在fork操作过程中共享整个父进程内存快照。

在RDB和AOF重写时，Redis日志会输出相关内存占用信息。

优化：

* 同CPU优化一样，如果部署多个Redis实例，尽量保证同一时刻只有一个子进程在工作

* 避免在大量写入时做子进程重写操作，这样将导致父进程维护大量页副本，造成内存消耗

#### 硬盘

子进程主要职责是把AOF或者RDB文件写入硬盘持久化。势必造成硬盘写入压力

优化：

* 不要和其他高硬盘负载的服务部署在一起。如： 存储服务、 消息队列服务等
* AOF重写时会消耗大量硬盘IO，可以开启配置no-appendfsync-onrewrite，默认闭。表示在AOF重写期间不做fsync操作
* 当开启AOF功能的Redis用于高流量写入场景时，如果使用普通机械磁盘，写入吞吐一般在100MB/s左右，这时Redis实例的瓶颈主要在AOF同步硬盘上
* 对于单机配置多个Redis实例的情况，可以配置不同实例分盘存储AOF文件，分摊硬盘写入压力

### AOF追加阻塞

当开启AOF持久化时，常用的同步硬盘的策略是everysec，用于平衡性能和数据安全性。对于这种方式，Redis使用另一条线程每秒执行fsync同步硬盘。当系统硬盘资源繁忙时，会造成Redis主线程阻塞。

当同步线程同步磁盘因为硬盘IO繁忙阻塞时，主线程会对比上次fsync时间，如果大于2s，那么主线程会阻塞等待fsync操作完成才会继续处理命令。图示如下：

![AOF追加阻塞示意图](https://gitee.com/wangziming707/note-pic/raw/master/img/AOF%E8%BF%BD%E5%8A%A0%E9%98%BB%E5%A1%9E%E7%A4%BA%E6%84%8F%E5%9B%BE.png)

所以我们发现：

* everysec配置最多可能丢失2秒数据，不是1秒
* 如果系统fsync缓慢，将会导致Redis主线程阻塞影响效率

当发生这样的AOF阻塞时，Redis会输出一下日志：

~~~log
Asynchronous AOF fsync is taking too long (disk is busy). Writing the AOF buffer without waiting for fsync to complete, this may slow down Redis
~~~

每当发生AOF追加阻塞事件发生时，在info Persistence统计中，aof_delayed_fsync指标会累加，查看这个指标方便定位AOF阻塞问题  

# 复制

在分布式系统中为了解决单点故障，通常会把数据复制多个副本部署到其他及其，满足故障恢复和负载均衡等要求。Redis也是如厕，它为我们提供了复制功能，实现了相同数据的多个Redis副本。复制功能是高可用Redis的基础。

## 配置

### 建立复制

参与复制的Redis实例划分为主节点(master)和从节点(slave)。默认情况下，Redis都是主节点。每个从节点只能有一个主节点，而主节点可以有多个从节点。复制的数据流是单向的，只能由主节点复制到从节点。

将当前Redis实例设置成从节点有如下三种方式：

* 配置文件中：设置`slaveof <master-ip> <master-port>`。随Redis启动生效
* 在redis-server启动命令后加上`--slaveof <master-ip> <master-port>`生效
* 客户端直接使用命令：`slaveof <master-ip> <master-port> `

建立主从结构后可以通过`info replication`命令查看role信息确认当前节点的角色。

### 断开复制

在从节点上执行`slaveof no one`断开于主节点的复制关系。从节点晋升为主节点。

从节点断开复制后不会抛弃原有数据，只是无法再获取主节点上的数据变化

通过slaveof命令还可以实现切主操作，所谓切主是指把当前从节点对主节点的复制切换到另一个主节点。执行`slaveof{newMasterIp}{newMasterPort}`命令即可  

切换主节点后从节点会清空之前所有数据，需要小心操作。

### 安全性

对于数据比较重要的节点，主节点会通过设置`requirepass`参数进行密码验证，这时所有的客户端访问必须使用auth命令进行校验。从节点和主节点的复制连接是通过一个特殊标识的客户端来完成的，因此需要配置从节点的`masterauth`参数于主节点密码保持一致，这样从节点才能正确地连接到主节点。

### 只读

默认情况下，从节点使用`slave-read-only=yes`配置为只读模式。由于复制只能是主节点到从节点，如果从节点有任何改动，主节点无法感知同步，会造成主从不一致，所以建议不要修改改配置。

### 传输延迟

主从节点一般部署在不同的机器上，复制时的网络延迟是需要考虑的问题。Redis提供了`repl-disable-tcp-nodelay`参数用于控制是否关闭tcp_nodelay，默认关闭，说明如下：

* 当关闭时，主节点产生的命令数据无论大小都会及时发送给从节点，这样主从之间的延迟会变小，但增加了网络带宽的消耗。适用于主从之间网络环境良好、或者同机架同机房部署的场景。
* 当开启时，主节点会合并比较小的TCP数据包从而节省带宽。默认发送时间间隔取决于Liunx的内核，一般默认为40毫秒。这种配置节省了带宽但增大主从之间的延迟。适用于主从网络环境复杂或者带宽紧张的场景。

## 拓扑

Redis的复制拓扑结构可以支持单层或者多层复制关系

### 一主一从

一主一从结构是最简单的复制拓扑结构，用于主节点出现宕机时从节点提供故障转移支持当应用写命令并发量较高且需要持久化时，可以只在从节点上开启AOF，这样既保证数据安全性同时也避免了持久化对主节点的性能干扰。但需要注意的是，当主节点关闭持久化功能时，如果主节点脱机要避免自动重启操作。因为主节点之前没有开启持久化功能自动重启后数据集为空，这时从节点如果继续复制主节点会导致从节点数据也被清空的情况，丧失了持久化的意义。安全的做法是在从节点上执行slaveof no one断开与主节点的复制关系，再重启主节点从而避免这一问题。

### 一主多从

一主多从结构使得应用端可以利用多个从节点实现读写分离。

对于读占比较大的场景，可以把读命令发送到从节点来分担主节点压力。同时在日常开发中如果需要执行一些比较耗时的读命令，如： keys、 sort等，可以在其中一台从节点上执行，防止慢查询对主节点造成阻塞从而影响线上服务的稳定性。

对于写并发量较高的场景，多个从节点会导致主节点写命令的多次发送从而过度消耗网络带宽，同时也加重了主节点的负载影响服务稳定性。

### 树状主从结构

树状主从结构使得从节点不但可以复制主节点数据，同时可以作为其他从节点的主节点继续向下层复制。

通过引入复制中间层，可以有效降低主节点负载和需要传送给从节点的数据量。

在这样的结构中数据实现了一层一层的向下复制。当主节点需要挂载多个从节点时为了避免对主节点的性能干扰，可以采用树状主从结构降低主节点压力。

## 原理

### 复制过程

从节点执行slaveof命令后，复制过程便开始运作，下面是完整流程：

* 保存主节点信息，在建立连接前，首先将主节点的host和port信息保存

* 从节点(slave)内部通过每秒运行的定时任务维护复制相关逻辑，当定时任务发现存在新的主节点后，会尝试与该节点建立网络连接：从节点会建立一个socket套接字，专门用于接收主节点发送的复制命令。

  如果从节点无法建立连接，定时任务会无限重试直到连接成功或者执行`slaveof no one`取消复制。可以通过info replication查看master_link_down_since_seconds指标  ，它记录与主节点连接失败的系统时间。从节点连接主节点失败时也会每秒打印日志。

* 连接建立成功后从节点发送ping请求进行首次通信，ping请求主要目的如下：

  * 检测主从之间socket是否可用
  * 检测主节点当前是否可以接收处理命令

  如果发送ping命令后，从节点没有收到主节点pong回复或者超时，从节点会断开复制连接，下次定时任务会发起重连。

* 权限验证。如果主节点设置了requirepass参数，从节点必须配置masterauth参数保证与主节点相同的密码才能通过验证，如果验证失败复制将终止。

* 同步数据集。主从复制连接正常通信后，对于首次建立复制的场景，主节点会把持有的数据全部发送给从节点，这部分操作时耗时很长，将异步执行

* 命令持续复制。当主节点把当前数据同步给从节点后，便完成了复制的建立流程。接下来主节点会持续地把写命令发送给从节点，保证主从数据一致性

### 数据同步

Redis使用psync命令完成主从数据同步，同步分为全量复制和部分复制

* 全量复制：一般用于初次复制场景，它会把主节点全部数据一次性发送给从节点，当数据量较大时，会对主从节点和网络造成很大的开销
* 部分复制：用于处理在主从复制中因网络闪断等原因造成的数据丢失场景，当从节点再次连上主节点后，如果条件允许，主节点会补发丢失数据给从节点。因为补发的数据远远小于全量数据，可以有效避免全量复制的过高开销

psync命令需要以下组件支持：

* 主从节点各自复制偏移量
* 主节点复制积压缓冲区
* 主节点运行id

#### 复制偏移量

参与复制的主从节点都会维护自身复制偏移量，主节点(master)在处理完写入命令后，会把命令的字节长度做累加记录，统计信息在info relication中的master_repl_offset指标中。

从节点每秒上报自身的复制偏移量给主节点，因此主节点也会保存从节点的复制偏移量，可以在`info relication`中查看

从节点在接收到主节点发送的命令后，也会累加记录自身的偏移量。统计信息在info relication中的slave_repl_offset指标中。

#### 复制积压缓冲区

复制积压缓冲区是保存在主节点上的一个固定长度的队列， 默认大小为1MB

当主节点有连接的从节点时被创建， 这时主节点响应写命令时， 不但会把命令发送给从节点， 还会写入复制积压缓冲区

由于缓冲区本质上是先进先出的定长队列， 所以能实现保存最近已复制数据的功能， 用于部分复制和复制命令丢失的数据补救。

复制缓冲区相关统计信息保存在主节点的`info replication`中：

~~~bat
127.0.0.1:6379> info replication
# Replication
role:master
...
repl_backlog_active:1 // 开启复制缓冲区
repl_backlog_size:1048576 // 缓冲区最大长度
repl_backlog_first_byte_offset:7479 // 起始偏移量， 计算当前缓冲区可用范围
repl_backlog_histlen:1048576 // 已保存数据的有效长度
~~~

#### 主节点运行ID

每个Redis节点启动后都会动态分配一个40位的16进制字符串作为运行ID。运行ID的主要作用是用来唯一识别Redis节点， 比如从节点保存主节点的运行ID识别自己正在复制的是哪个主节点。

如果只使用ip+port的方式识别主节点， 那么主节点重启变更了整体数据集(如替换RDB/AOF文件)从节点再基于偏移量复制数据将是不安全的， 因此当运行ID变化后从节点将做全量复制

可以运行info server命令查看当前节点的运行ID。需要注意的是Redis关闭再启动后， 运行ID会随之改变

可以使用debug reload命令重新加载RDB并保持运行ID不变，这在做一些需要重启redis操作时会很有用，如调优一些内存相关配置，需要Redis重新加载才能优化已存在的数据

**注意：**debug reload命令会阻塞当前Redis节点主线程， 阻塞期间会生成本地RDB快照并清空数据之后再加载RDB文件。 因此对于大数据量的主节点和无法容忍阻塞的应用场景， 谨慎使用

#### psync

从节点使用psync命令完成部分复制和全量复制，命令格式为：`psync {runId} {offset}`，参数含义如下：

* runId:从节点所复制主节点的运行ID
* offset：当前从节点已复制的数据偏移量

psync命令，的整体流程如下：

* 从节点发送psync命令给主节点
  * 参数runId时当前从节点保存的主节点运行ID，如果没有则默认值为？
  * 参数offset时当前从节点保存的复制偏移量，如果是第一次参与复制则默认值为-1
* 主节点根据psync参数与自身数据情况决定响应结果：
  * 回复`+FULLRESYNC {runId} {offset}`，那从节点将触发全量复制流程
  * 回复`+CONTINUE`，从节点将触发部分复制流程
  * 回复`+ERR`说明主节点版本过低无法识别psync命令，从节点将发送旧版的sync命令触发全量复制流程

### 全量复制

全量复制是主从第一次建立复制时必须经历的阶段。通过psync发起全量复制，流程如下：

* 发送psync命令进行数据同步，第一次复制时发送`psync -1`

* 主节点根据psync-1解析当前为全量复制，回复`+FULLRESYNC`响应

* 从节点接收主节点的响应数据保存运行ID和偏移量offset

* 主节点执行bgsave保存RDB文件到本地

* 主节点发送RDB文件给从节点，从节点把接收的RDB文件保存在本地并直接作为从节点的数据文件。

  如果数据量比较打，可能耗时会很久，当耗时超过`repl-timeout`(默认60s)时全量复制会失败。所以此时可以调高`repl-timeout`的值

  * **无盘复制**：为了降低主节点的磁盘开销，Redis支持无盘复制，生成的RDB文件不保存到硬盘而是直接通过网络发送给从节点，通过`repl-diskless-sync`参数控制，默认关闭。无盘复制适用于主节点磁盘性能较差但网络带宽充裕的场景。

* 从节点开始接收RDB快照到接收完成期间，主节点仍然响应读写命令，因此主节点会把这期间写命令数据保存在复制客户端缓冲区内，当从节点接在完RDB文件后，主节点再把缓冲区内的数据发送给从节点，从而保证主从一致。如果传输RDB的时间过长，对于高流量写入场景容易造成主节点复制客户端缓冲区溢出。

  默认配置为`clientoutput-buffer-limit slave256MB64MB60`， 如果60秒内缓冲区消耗持续大于64MB或者直接超过256MB时， 主节点将直接关闭复制客户端连接， 造成全量同步失败.所以要根据情况配置该值。

* 从节点接收完主节点传送过来的全部数据后会清空自身旧数据

* 从节点清空完成后开始加载RDB文件，当文件较大时，这一步骤比较耗时

  在做读写分离的场景下，此时从节点仍然可以响应读命令。但是因为正处于全量复制货主额复制终端阶段，从节点响应读命令可能拿到过期或错误的数据。Redis提供了`slave-serve-stale-data `参数，设置为no时，将不能响应除了info和slaveof之外的所有命令。如果对一致性要求很高，可以在复制期间关闭该参数。

* 从节点成功加载完RDB后，如果从节点开启了AOF，它会立刻作bgrewriteaof进行aof重写。

可以发现全量复制在某些步骤非常耗时，如主节点bgsave、RDB文件网络传输、从节点加载RDB、AOF重写等。

所以除了第一次复制时采用全量复制外，其他场景都应该规避全量复制的发生。

### 部分复制

部分复制主要是Redis针对全量复制的过高开销做出的一种优化措施，使用`psync{runId}{offset}`命令实现。 当从节点正在复制主节点时， 如果出现网络闪断或者命令丢失等异常情况时， 从节点会向主节点要求补发丢失的命令数据， 如果主节点的复制积压缓冲区内存在这部分数据则直接发送给从节点， 这样就可以保持主从节点复制的一致性。 补发的这部分数据一般远远小于全量数据， 所以开销很小。  

流程说明：

* 当主从节点之间网络出现中断时， 如果超过repl-timeout时间 主节点会认为从节点故障并中断复制连接。
* 主从连接中断期间主节点依然响应命令， 但因复制连接中断命令无法发送给从节点， 不过主节点内部存在的复制积压缓冲区， 依然可以保存最近一段时间的写命令数据， 默认最大缓存1MB
* 当主从连接恢复后， 由于从节点之前保存了自身已复制的偏移量和主节点的运行ID。 因此会把它们当作psync参数发送给主节点， 要求进行部分复制操作
* 主节点接到psync命令后首先核对参数runId是否与自身一致， 如果一致， 说明之前复制的是当前主节点； 之后根据参数offset在自身复制积压缓冲区查找， 如果偏移量之后的数据存在缓冲区中，则对从节点发送`+CONTINUE`响应， 表示可以进行部分复制  
* 主节点根据偏移量把复制积压缓冲区里的数据发送给从节点， 保证主从复制进入正常状态。

### 心跳

主从节点建立复制后，它们之间维护长连接并彼此发送心跳命令。其判断机制如下：

* 主从节点之间都有心跳检测机制，各自模拟成对方的客户端进行通信

  通过`client list`查看客户端信息，主节点模拟的客户端`flags=M`，从节点模拟客户端的`flags=S`

* 主节点默认每隔10秒对从节点发送ping命令，判断从节点的存活性和连接状态，可通过`repl-ping-slave-period`控制发送频率

* 从节点在主线程中每隔1秒发送`replconf ack {offset}`命令，给主节点上报当前自生的复制偏移量，该作用有：

  * 实时检测主从节点网络状态
  * 上报自生复制偏移量，检查复制数据是否丢失
  * 保证从节点的数量和延迟性功能

### 异步复制

主节点不但负责数据读写，还负责把命令同步给从节点。写命令的发送过程是异步完成的。也就是说主节点处理完写命令后立即返回给客户端，不等待从节点复制完成。

主节点复制过程：

* 主节点接收处理命令
* 命令处理完成后立即返回响应结果
* 对于修改命令异步发送给从节点，从节点在主线程中执行复制的命令

因为主从复制过程是异步的，所以会造成从节点的数据相对主节点存在延迟。可以通过`info replication`命令查看相关主从节点的offset判断。主从节点的offset差值就是它们延迟量。

# 哨兵

Redis的主从复制模式下， 一旦主节点由于故障不能提供服务， 需要人工将从节点晋升为主节点， 同时还要通知应用方更新主节点地址， 对于很多应用场景这种故障处理的方式是无法接受的。

Redis Sentinel架构则解决了这个问题。

**主从复制的问题**

Redis的主从复制模式可以将主节点的数据改变同步给从节点， 这样从
节点就可以起到两个作用：一是作为主节点的一个备份，二是从节点可以扩展主节点的读能力

但是主从复制也带来了以下问题：

* 一旦主节点出现故障， 需要手动将一个从节点晋升为主节点， 同时需要修改应用方的主节点地址， 还需要命令其他从节点去复制新的主节点， 整个过程都需要人工干预。所以整个架构不是高可用的。
* 主节点的写能力受到单机的限制
* 主节点的存储能力受到单机的限制

**Sentinel的高可用**

当主节点出现故障时， Redis Sentinel能自动完成故障发现和故障转移，并通知应用方， 从而实现真正的高可用。

Redis Sentinel是一个分布式架构， 其中包含若干个Sentinel节点和Redis数据节点， 每个Sentinel节点会对数据节点和其余Sentinel节点进行监控， 当它发现节点不可达时， 会对节点做下线标识。 如果被标识的是主节点， 它还会和其他Sentinel节点进行“协商”， 当大多数Sentinel节点都认为主节点不可达时， 它们会选举出一个Sentinel节点来完成自动故障转移的工作， 同时会将这个变化实时通知给Redis应用方。 整个过程完全是自动的， 不需要人工来介入， 所以这套方案很有效地解决了Redis的高可用问题。

Redis Sentinel具有以下几个功能：

* 监控： Sentinel节点会定期检测Redis数据节点、 其余Sentinel节点是否可达。
* 通知： Sentinel节点会将故障转移的结果通知给应用方。
* 主节点故障转移： 实现从节点晋升为主节点并维护后续正确的主从关系。
* 配置提供者： 在Redis Sentinel结构中， 客户端在初始化的时候连接的是Sentinel节点集合， 从中获取主节点信息。 

## 安装部署

我们接下来部署一个一主两从的节点并部署三个Sentinel节点来监视他们，其拓扑结构如图：

![RedisSentinel安装示例拓扑图](https://gitee.com/wangziming707/note-pic/raw/master/img/RedisSentinel%E5%AE%89%E8%A3%85%E7%A4%BA%E4%BE%8B%E6%8B%93%E6%89%91%E5%9B%BE.png)

### 部署Redis数据节点

**启动主节点**

首先配置并部署主节点mater:

* 配置redis-679.conf：

~~~conf
port 6379
daemonize yes
logfile "6379.log"
dbfilename "dump-6379.rdb"
dir "/opt/soft/redis/data/"
~~~

* 启动主节点：

~~~bash
redis-server redis-6379.conf
~~~

* 确认是否启动：

~~~bash
redis-cli ping
PONG
~~~

**启动两个从节点**

两个从节点配置完全一样，我们以一个从节点为例，和主节点的配置不同的是添加了slaveof配置：

* 配置redis-3680.conf：

~~~conf
port 6380
daemonize yes
logfile "6380.log"
dbfilename "dump-6380.rdb"
dir "/opt/soft/redis/data/"
slaveof 127.0.0.1 6379
~~~

* 启动两个从节点：

~~~bash
redis-server redis-6380.conf
redis-server redis-6381.conf
~~~

* 验证：

~~~bash
$ redis-cli -p 6380 ping
PONG
$ redis-cli -p 6381 ping
PONG
~~~

**确认主从关系**

查看主节点的复制信息，可以看到有两个从节点：

~~~~bash
$ redis-cli -p 6379 info replication
# Replication
role:master
connected_slaves:2
slave0:ip=127.0.0.1,port=6380,state=online,offset=182,lag=1
slave1:ip=127.0.0.1,port=6381,state=online,offset=182,lag=1
......
~~~~

查看从节点的复制信息，可以看到它的主节点：

~~~bash
$ redis-cli -p 6380 info replication
# Replication
role:slave
master_host:127.0.0.1
master_port:6379
......
~~~

此时的拓扑结构如图：

![Redis一主两从的拓扑图](https://gitee.com/wangziming707/note-pic/raw/master/img/Redis%E4%B8%80%E4%B8%BB%E4%B8%A4%E4%BB%8E%E7%9A%84%E6%8B%93%E6%89%91%E5%9B%BE.png)

### 部署Sentinel节点

除了端口不同，三个Sentinel节点的部署方法完全一致，我们以一个sentinel节点的部署为例子进行说明

**配置Sentinel节点**

redis-sentinel-26379.conf

~~~confi
port 26379
daemonize yes
logfile "26379.log"
dir /opt/soft/redis/data
sentinel monitor mymaster 127.0.0.1 6379 2
sentinel down-after-milliseconds mymaster 30000
sentinel parallel-syncs mymaster 1
sentinel failover-timeout mymaster 180000
~~~

* Sentinel节点的默认端口是26379。
* sentinel monitor mymaster 127.0.0.1 6379 2配置代表sentinel-1节点需要监
  控127.0.0.1:6379这个主节点， 2代表判断主节点失败至少需要2个Sentinel节
  点同意， mymaster是主节点的别名， 其余Sentinel配置将在下一节进行详细说
  明  

**启动Sentinel节点**

启动Sentinel节点有两种方法：

* 使用redis-sentinel命令：

  ~~~bash
  redis-sentinel redis-sentinel-26379.conf
  ~~~

* 使用redis-server命令加`--sentinel`参数

  ~~~bash
  redis-server redis-sentinel-26379.conf --sentinel
  ~~~

两种方法本质上是一样的

**确认**

Sentinel节点本质上是一个特殊的Redis节点， 所以也可以通过info命令来查询它的相关信息：

~~~bash
$ redis-cli -p 26379 info Sentinel
# Sentinel
sentinel_masters:1
sentinel_tilt:0
sentinel_running_scripts:0
sentinel_scripts_queue_length:0
sentinel_simulate_failure_flags:0
master0:name=mymaster,status=ok,address=127.0.0.1:6379,slaves=2,sentinels=3
~~~

从上面info的Sentinel片段来看：

Sentinel节点找到了主节点127.0.0.1:6379， 发现了它的两个从节点， 同时发现Redis Sentinel一共有3个Sentinel节点。

这样三个Sentinel节点启动后，整个拓扑图就如下所示。

![RedisSentinel最终拓扑结构](https://gitee.com/wangziming707/note-pic/raw/master/img/RedisSentinel%E6%9C%80%E7%BB%88%E6%8B%93%E6%89%91%E7%BB%93%E6%9E%84.png)

有以下两点需要注意：

* 生产环境中建议Redis Sentinel的所有节点应该分布在不同的物理机上。
* Redis Sentinel中的数据节点和普通的Redis数据节点在配置上没有任何区别， 只不过是添加了一些Sentinel节点对它们进行监控  

## Sentinal配置

Redis安装目录下有一个`sentinel.conf`,是默认的Sentinel节点配置文件，其内容为：

~~~bash
port 26379
daemonize no
pidfile /var/run/redis-sentinel.pid
logfile ""
# sentinel announce-ip <ip>
# sentinel announce-port <port>
dir /tmp
# sentinel monitor <master-name> <ip> <redis-port> <quorum>
sentinel monitor mymaster 127.0.0.1 6379 2
# sentinel auth-pass <master-name> <password>
# sentinel auth-user <master-name> <username>
sentinel down-after-milliseconds mymaster 30000
acllog-max-len 128
# requirepass <password>
# sentinel sentinel-user <username>
# sentinel sentinel-pass <password>
# sentinel parallel-syncs <master-name> <numreplicas>
sentinel parallel-syncs mymaster 1
# sentinel failover-timeout <master-name> <milliseconds>
sentinel failover-timeout mymaster 180000
# sentinel notification-script <master-name> <script-path>
# sentinel client-reconfig-script <master-name> <script-path>
sentinel deny-scripts-reconfig yes
SENTINEL resolve-hostnames no
SENTINEL announce-hostnames no
~~~

port和dir分别代表Sentinel节点的端口和工作目录， 下面重点对sentinel相关配置进行详细说明

### 主要配置

**sentinel monitor**

配置如下：

~~~conf
sentinel monitor <master-name> <ip> <port> <quorum>
~~~

Sentinel节点会定期监控主节点， 所以从配置上必然也会有所体现， 本配置说明Sentinel节点要监控的是一个名字叫做`<master-name>`， ip地址和端口为`<ip><port>`的主节点。 `<quorum>`代表要判定主节点最终不可达所需要的票数。 

实际上Sentinel节点会对所有节点进行监控,但是在Sentinel节点的配置中没有看到有关从节点和其余Sentinel节点的配置， 那是因为Sentinel节点会从主节点中获取有关从节点以及其余Sentinel节点的相关信息，有关实现我们后续介绍。

例如某个sentinel节点的初始配置如下：

~~~bash
port 26379
daemonize yes
logfile "26379.log"
dir /opt/soft/redis/data
sentinel monitor mymaster 127.0.0.1 6379 2
sentinel down-after-milliseconds mymaster 30000
sentinel parallel-syncs mymaster 1
sentinel failover-timeout mymaster 180000
~~~

但所有节点启动后，配置文件中的内容发生了变化，体现在三个方面：

* Sentinel节点自动发现了从节点、其余Sentinel节点
* 去掉了默认配置，例如parallel-syncs、failover-timeout参数
* 添加了配置纪元相关参数

启动后配置文件变化为：

~~~bash
port 26379
daemonize yes
logfile "26379.log"
dir "/opt/soft/redis/data"
sentinel monitor mymaster 127.0.0.1 6379 2

# Generated by CONFIG REWRITE
protected-mode no
pidfile "/var/run/redis.pid"
user default on nopass ~* &* +@all
sentinel myid 34483900c21262e4518567112607eecbaff383af
sentinel config-epoch mymaster 0
sentinel leader-epoch mymaster 0
sentinel current-epoch 0
sentinel known-replica mymaster 127.0.0.1 6381
sentinel known-replica mymaster 127.0.0.1 6380
sentinel known-sentinel mymaster 127.0.0.1 26381 7b57b9666471f98fe26990fcaa96a037cecc006f
sentinel known-sentinel mymaster 127.0.0.1 26380 df9d9dc6a9f0878070b297aaf8387c63f58cbd33
~~~

`<quorum>`参数用于故障发现和判定， 例如将quorum配置为2,代表当至少有2个Sentinel节点认为主节点故障/不可达，这个判定才会生效。一般建议将其设置为Sentinel节点的一半加1

同时`<quorum>`还与Sentinel节点的领导者选举有关， 至少要有
$max(quorum,num(sentinels)/2+1)$个Sentinel节点参与选举， 才能选出领导者Sentinel， 从而完成故障转移。

例如有5个Sentinel节点, quorum=4，那么至少要有4个在线Sentinel节点才可以进行领导者选举

**sentinel down-after-milliseconds**

配置如下：

~~~bash
sentinel down-after-milliseconds <master-name> <times>
~~~

每个Sentinel节点都要通过定期发送ping命令来判断Redis数据节点和其余Sentinel节点是否可达， 如果超过了down-after-milliseconds配置的时间且没有有效的回复， 则判定节点不可达， `<times>`(单位为毫秒)就是超时时间。 这个配置是对节点失败判定的重要依据  

down-after-milliseconds虽然以`<master-name>`为参数， 但实际上对Sentinel节点、 主节点、 从节点的失败判定同时有效  

`down-after-milliseconds`越大， 代表Sentinel节点对于节点不可达的条件越宽松， 反之越严格。 条件宽松有可能带来的问题是节点确实不可达了， 那么应用方需要等待故障转移的时间越长， 也就意味着应用方故障时间可能越长。 条件严格虽然可以及时发现故障完成故障转移， 但是也存在一定的误判率。

**sentinel parallel-syncs**

~~~bash
sentinel parallel-syncs <master-name> <nums>
~~~

当Sentinel节点集合对主节点故障判定达成一致时， Sentinel领导者节点会做故障转移操作， 选出新的主节点， 原来的从节点会向新的主节点发起复制操作， parallel-syncs就是用来限制在一次故障转移之后， 每次向新的主节点发起复制操作的从节点个数。 

如果这个参数配置的比较大， 那么多个从节点会向新的主节点同时发起复制操作， 尽管复制操作通常不会阻塞主节点，但是同时向主节点发起复制， 必然会对主节点所在的机器造成一定的网络和磁盘IO开销。

**sentinel failover-timeout**

~~~bash
sentinel failover-timeout <master-name> <times>
~~~

failover-timeout通常被解释成故障转移超时时间， 但实际上它作用于故
障转移的各个阶段：

* 选出合适从节点。
* 晋升选出的从节点为主节点
*  命令其余从节点复制新的主节点。
* 等待原主节点恢复后命令它去复制新的主节点

failover-timeout的作用具体体现在四个方面：

* 如果Redis Sentinel对一个主节点故障转移失败， 那么下次再对该主节点做故障转移的起始时间是failover-timeout的2倍
* 在晋升选出的从节点为主节点时， 如果要晋升的从节点执行  slaveof no one一直失败， 当此过程超过failover-timeout时， 则故障转移失败。
* 如果晋升主节点成功， Sentinel节点还会执行info命令来确认选出来的节点确实晋升为主节点， 如果此过程执行时间超过failovertimeout时， 则故障转移失败。
* 如果命令其余从节点复制新的主节点阶段执行时间超过了failover-timeout(不包含复制时间)，则故障转移失败。 注意即使超过了这个时间， Sentinel节点也会最终配置从
  节点去同步最新的主节点

**sentinel auth-pass**

~~~bash
sentinel auth-pass <master-name> <password>
~~~

如果Sentinel监控的主节点配置了密码， sentinel auth-pass配置通过添加主节点的密码， 防止Sentinel节点对主节点无法监控

**sentinel notification-script**

~~~bash
sentinel notification-script <master-name> <script-path>
~~~

该参数的作用是在故障转移期间， 当一些警告级别的Sentinel事件发生(指重要事件， 例如-sdown： 客观下线、 -odown： 主观下线)时， 会触发对应路径的脚本， 并向脚本发送相应的事件参数  

**sentinel client-reconfig-script**

~~~bash
sentinel client-reconfig-script <master-name> <script-path>
~~~

sentinel client-reconfig-script的作用是在故障转移结束后， 会触发对应路径的脚本， 并向脚本发送故障转移结果的相关参数

发送给脚本的具体参数如下：

~~~bash
<master-name> <role> <state> <from-ip> <from-port> <to-ip> <to-port>
~~~

* `<master-name>`： 主节点名。
* `<role>`： Sentinel节点的角色， 分别是leader和observer， leader代表当前Sentinel节点是领导者， 是它进行的故障转移； observer是其余Sentinel节点。
* `<from-ip>`： 原主节点的ip地址
* `<from-port>`： 原主节点的端口
* `<to-ip>`： 新主节点的ip地址
* `<to-port>`： 新主节点的端口  

### 监控多个主节点

Redis Sentinel可以同时监控多个主节点，如下图：

![RedisSentinel监控多个主节点拓扑图](https://gitee.com/wangziming707/note-pic/raw/master/img/RedisSentinel%E7%9B%91%E6%8E%A7%E5%A4%9A%E4%B8%AA%E4%B8%BB%E8%8A%82%E7%82%B9%E6%8B%93%E6%89%91%E5%9B%BE.png)

只需要指定多个masterName来区分不同的主节点即可，例如：

~~~conf
sentinel monitor master-business-1 10.10.xx.1 6379 2
sentinel down-after-milliseconds master-business-1 60000
sentinel failover-timeout master-business-1 180000
sentinel parallel-syncs master-business-1 1
sentinel monitor master-business-2 10.16.xx.2 6380 2
sentinel down-after-milliseconds master-business-2 10000
sentinel failover-timeout master-business-2 180000
sentinel parallel-syncs master-business-2 1
~~~

### 调整配置

和普通的Redis数据节点一样， Sentinel节点也支持动态地设置参数， 而且和普通的Redis数据节点一样并不是支持所有的参数， 具体使用方法如下：

~~~bash
sentinel set <param> <value>
~~~

下面是它支持的常用命令：

![sentinelSet支持的命令](https://gitee.com/wangziming707/note-pic/raw/master/img/sentinelSet%E6%94%AF%E6%8C%81%E7%9A%84%E5%91%BD%E4%BB%A4.png)

需要注意：

* sentinel set命令只对当前Sentinel节点有效
* sentinel set命令如果执行成功会立即刷新配置文件
* 建议所有Sentinel节点的配置尽可能一致， 这样在故障发现和转移时比较容易达成一致
* Sentinel对外不支持config命令

## API

Sentinel节点是一个特殊的Redis节点， 它有自己专属的API

* `sentinel masters`:展示所有被监控的主节点状态以及相关的统计信息  

* `sentinel master<master name>  `:展示指定`<master name>`的主节点状态以及相关的统计信息
* `sentinel slaves<master name>  `:展示指定`<master name>`的从节点状态以及相关的统计信息  

* `sentinel sentinels<master name>  `:展示指定`<master name>`的Sentinel节点集合,不包含当前Sentinel节点
*  `sentinel get-master-addr-by-name<master name>  `:返回指定`<master name>`主节点的IP地址和端口  

* `sentinel reset<pattern>  `:当前Sentinel节点对符合`<pattern>`(通配符风格)主节点的配置进行重置， 包含清除主节点的相关状态(例如故障转移)， 重新发现从节点和Sentinel节点
* `sentinel failover<master name>  `:对指定`<master name>`主节点进行强制故障转移(没有和其他Sentinel节点“协商”)， 当故障转移完成后， 其他Sentinel节点按照故障转移的结果更新自身配置
* `sentinel ckquorum<master name>  `:检测当前可达的Sentinel节点总数是否达到`<quorum>`的个数
* `sentinel flushconfig`将Sentinel节点的配置强制刷到磁盘上
* `sentinel remove<master name>  `:取消当前Sentinel节点对于指定`<master name>`主节点的监控
* `sentinel monitor<master name><ip><port><quorum>  `:这个命令和配置文件中的含义是完全一样的， 只不过是通过命令的形式来完成Sentinel节点对主节点的监控
* `sentinel set<master name>`动态修改Sentinel节点配置选项,之前已经介绍了
* `sentinel is-master-down-by-addr`:Sentinel节点之间用来交换对主节点是否下线的判断， 根据参数的不同， 还可以作为Sentinel领导者选举的通信方式,具体方式后续再介绍

## 客户端连接

Sentinel在完成故障转移后需要通知应用方新的主节点的host和port，这需要客户端对Redis Sentinel进行显式支持才能完成。

### Sentinel客户端原理

Sentinel节点集合具备了监控、 通知、 自动故障转移、 配置提供者若干功能， 也就是说实际上最了解主节点信息的就是Sentinel节点集合， 而各个主节点可以通过`<master-name>`进行标识的， 所以， 无论是哪种编程语言的客户端， 如果需要正确地连接Redis Sentinel， 必须有Sentinel节点集合和masterName两个参数  

实现一个Redis Sentinel客户端的基本步骤如下：

* 遍历Sentinel节点集合获取一个可用的Sentinel节点， 后面会介绍Sentinel节点之间可以共享数据， 所以从任意一个Sentinel节点获取主节点信息都是可以的
* 通过sentinel get-master-addr-by-name master-name这个API来获取对应主节点的相关信息
* 验证当前获取的“主节点”是真正的主节点， 这样做的目的是为了防止故障转移期间主节点的变化
* 保持和Sentinel节点集合的“联系”， 时刻获取关于主节点的相关“信息”

### Jedis操作Sentinel

Jedis能够很好地支持Redis Sentinel， 并且使用Jedis连接Redis Sentinel也很简单， 按照Redis Sentinel的原理， 需要有masterName和Sentinel节点集合两个参数。

Jedis针对Redis Sentinel给出了一个JedisSentinelPool，Jedis给出很多构造方法 ，其中最全的是：

~~~java
public JedisSentinelPool(String masterName, Set<String> sentinels,
      final GenericObjectPoolConfig poolConfig, final int connectionTimeout, final int soTimeout,
      final String user, final String password, final int database, final String clientName)
~~~

具体参数含义如下：

* masterName——主节点名
* sentinels——Sentinel节点集合
* poolConfig——common-pool连接池配置
* connectTimeout——连接超时
* soTimeout——读写超时
* password——主节点密码
* database——当前数据库索引  

* clientName——客户端名

JedisSentinelPool的使用和JedisPool非常类似：

从JedisSentinelPool获取的Jedis实例可以操作Sentinel节点。

~~~java
HashSet<String> sentinelSet = new HashSet<>();
sentinelSet.add("127.0.0.1:26379");
sentinelSet.add("127.0.0.1:26380");
sentinelSet.add("127.0.0.1:26381");
JedisSentinelPool pool = new JedisSentinelPool("mymaster", sentinelSet);
Jedis jedis = pool.getResource();
List<String> mymaster = jedis.sentinelGetMasterAddrByName("mymaster");

jedis.close();
~~~

## 原理

Redis Sentinel的基本实现原理包含以下几个方面：Redis Sentinel的三个定时任务、 主观下线和客观下线、 Sentinel领导者选举、故障转移

### 三个定时任务

一套合理的监控机制是Sentinel节点判定节点不可达的重要保证， RedisSentinel通过三个定时监控任务完成对各个节点发现和监控：

* 每隔10秒， 每个Sentinel节点会向主节点和从节点发送info命令获取最新的拓扑结构，该任务有三个作用：
  * 通过向主节点执行info命令， 获取从节点的信息， 这也是为什么
    Sentinel节点不需要显式配置监控从节点。
  * 当有新的从节点加入时都可以立刻感知出来。
  * 节点不可达或者故障转移后， 可以通过info命令实时更新节点拓扑信
    息  

![Sentinel节点定时执行info命令  ](https://gitee.com/wangziming707/note-pic/raw/master/img/Sentinel%E8%8A%82%E7%82%B9%E5%AE%9A%E6%97%B6%E6%89%A7%E8%A1%8Cinfo%E5%91%BD%E4%BB%A4.png)





* 每隔2秒， 每个Sentinel节点会向Redis数据节点的`__sentinel__:hello`频道上发送该Sentinel节点对于主节点的判断以及当前Sentinel节点的信息,同时每个Sentinel节点也会订阅该频道， 来了解其他Sentinel节点以及它们对主节点的判断。所以该定时任务可以完成以下两个工作
  * 发现新的Sentinel节点： 通过订阅主节点的`__sentinel__: hello`了解其他的Sentinel节点信息， 如果是新加入的Sentinel节点， 将该Sentinel节点信息保存起来， 并与该Sentinel节点创建连接
  * Sentinel节点之间交换主节点的状态作为后面客观下线以及领导者选举的依据

![Sentinel节点发布和订阅__sentinel__hello频道](https://gitee.com/wangziming707/note-pic/raw/master/img/Sentinel%E8%8A%82%E7%82%B9%E5%8F%91%E5%B8%83%E5%92%8C%E8%AE%A2%E9%98%85__sentinel__hello%E9%A2%91%E9%81%93.png)

* 每隔1秒， 每个Sentinel节点会向主节点、 从节点、 其余Sentinel节点发送一条ping命令做一次心跳检测， 来确认这些节点当前是否可达。通过该定时任务， Sentinel节点对主节点、 从节点、 其余Sentinel节点都建立起连接， 实现了对每个节点的监控， 这个定时任务是节点失败判定的重要依据

![Sentinel节点向其余节点发送ping命令](https://gitee.com/wangziming707/note-pic/raw/master/img/Sentinel%E8%8A%82%E7%82%B9%E5%90%91%E5%85%B6%E4%BD%99%E8%8A%82%E7%82%B9%E5%8F%91%E9%80%81ping%E5%91%BD%E4%BB%A4.png)

### 下线

#### 主观下线

在Sentinel节点的定时任务中，每个Sentinel节点会每隔1秒对主节
点、 从节点、 其他Sentinel节点发送ping命令做心跳检测， 当这些节点超过down-after-milliseconds没有进行有效回复， Sentinel节点就会对该节点做失败判定， 这个行为叫做主观下线。

这个行为是单个Sentinel节点的判断，存在误判的可能。

#### 客观下线

当Sentinel主观下线的节点是主节点时， 该Sentinel节点会通过sentinel ismaster-down-by-addr命令向其他Sentinel节点询问对主节点的判断， 当超过`<quorum>`个数， Sentinel节点认为主节点确实有问题， 这时该Sentinel节点会做出客观下线的决定， 这样客观下线的含义是比较明显了， 也就是大部分Sentinel节点都对主节点的下线做了同意的判定， 那么这个判定就是客观的  

sentinel is-master-down-by-addr命令如下：

~~~bash
sentinel is-master-down-by-addr <ip> <port> <current_epoch> <runid>
~~~

* ip： 主节点IP。
* port： 主节点端口。
* current_epoch： 当前配置纪元。
* runid： 此参数有两种类型， 不同类型决定了此API作用的不同
  * 当runid等于`*`时， 作用是Sentinel节点直接交换对主节点下线的判定
  * 当runid等于当前Sentinel节点的runid时， 作用是当前Sentinel节点希望目标Sentinel节点同意自己成为领导者的请求

### 领导者选举

当Sentinel节点集群对主节点做了客观下线后，就需要进行故障转移。但是故障转移只需要一个Sentinel节点来完成即可，所以Sentinel节点之间会进行一个领导者选举的工作，选出一个Sentinel节点作为领导者来进行故障转移。

Redis使用Raft算法实现领导者选举，大致的思路是：

* 每个在线的Sentinel节点都有资格成为领导者， 当它确认主节点主观下线时候， 会向其他Sentinel节点发送sentinel is-master-down-by-addr命令，要求将自己设置为领导者。
* 收到命令的Sentinel节点， 如果没有同意过其他Sentinel节点的sentinel is-master-down-by-addr命令， 将同意该请求， 否则拒绝。
* 如果该Sentinel节点发现自己的票数已经大于等于$max(quorum，num(sentinels)/2+1)$ ， 那么它将成为领导者。
* 如果此过程没有选举出领导者， 将进入下一次选举  

实际上选举的过程非常快， 基本上谁先完成客观下线， 谁就是领导者。

### 故障转移

领导者选举出的Sentinel节点负责故障转移， 具体步骤如下：

* 在从节点列表中选出一个节点作为新的主节点， 选择方法如下：
  * 过滤： “不健康”(主观下线、 断线)、 5秒内没有回复过Sentinel节点ping响应、 与主节点失联超过down-after-milliseconds*10秒。
  * 选择slave-priority(从节点优先级)最高的从节点列表， 如果存在则返回， 不存在则继续。
  * 选择复制偏移量最大的从节点(复制的最完整)， 如果存在则返回， 不存在则继续。
  * 选择runid最小的从节点

* Sentinel领导者节点会对第一步选出来的从节点执行slaveof no one命令让其成为主节点  

* Sentinel领导者节点会向剩余的从节点发送命令， 让它们成为新主节点的从节点， 复制规则和parallel-syncs参数有关
* Sentinel节点集合会将原来的主节点更新为从节点， 并保持着对其关注， 当其恢复后命令它去复制新的主节点

# 集群

Redis Cluster是Redis的分布式解决方案，当遇到单机内存、 并发、 流量等瓶颈时可以采用Cluster架构方案达到负载均衡的目的 

## 数据分布

分布式数据库首先需要将整个数据集按照分区规则映射到多个节点的问题， 即把数据集划分到多个节点上， 每个节点负责整体数据的一个子集

### 数据分布理论

常见的分区规则有哈希分区和顺序分区两种：

* 哈希分区：离散度好、数据分布与业务无关、无法顺序访问
* 顺序分区：离散度易倾斜、数据分布与业务无关、可顺序访问

Redis Cluster采用哈希分区规则， 这里我们重点讨论哈希分区：

#### 节点取余分区

使用特定的数据， 如Redis的键或用户ID， 再根据节点数量N使用公式：$hash(key)\%N$计算出哈希值， 用来决定数据映射到哪一个节点上  

但是当节点数量变化时， 如扩容或收缩节点， 数据节点映射关系需要重新计算， 会导致数据的重新迁移

![节点取余分区示意图](https://gitee.com/wangziming707/note-pic/raw/master/img/%E8%8A%82%E7%82%B9%E5%8F%96%E4%BD%99%E5%88%86%E5%8C%BA%E7%A4%BA%E6%84%8F%E5%9B%BE.png)





这种方式的突出优点是简单性， 常用于数据库的分库分表规则， 一般采用预分区的方式， 提前根据数据量规划好分区数， 比如划分为512或1024张表， 保证可支撑未来一段时间的数据量， 再根据负载情况将表迁移到其他数据库中。 扩容时通常采用翻倍扩容， 避免数据映射全部被打乱导致全量迁移的情况

#### 一致性哈希分区

一致性哈希分区（Distributed Hash Table） 实现思路是为系统中每个节点分配一个token， 范围一般在0~232， 这些token构成一个哈希环。 数据读写执行节点查找操作时， 先根据key计算hash值， 然后顺时针找到第一个大于等于该哈希值的token节点

![一致性哈希分区示意图](https://gitee.com/wangziming707/note-pic/raw/master/img/%E4%B8%80%E8%87%B4%E6%80%A7%E5%93%88%E5%B8%8C%E5%88%86%E5%8C%BA%E7%A4%BA%E6%84%8F%E5%9B%BE.png)



这种方式相比节点取余最大的好处在于加入和删除节点只影响哈希环中相邻的节点， 对其他节点无影响

但是也存在以下问题：

* 加减节点会造成哈希环中部分数据无法命中， 需要手动处理或者忽略这部分数据， 因此一致性哈希常用于缓存场景

* 当使用少量节点时， 节点变化将大范围影响哈希环中数据映射， 因此这种方式不适合少量数据节点的分布式方案
* 普通的一致性哈希分区在增减节点时需要增加一倍或减去一半节点才能保证数据和负载的均衡

#### 虚拟槽分区

虚拟槽分区巧妙地使用了哈希空间， 使用分散度良好的哈希函数把所有数据映射到一个固定范围的整数集合中， 整数定义为槽（slot） 。 这个范围一般远远大于节点数， 比如Redis Cluster槽范围是0~16383。 槽是集群内数据管理和迁移的基本单位。 采用大范围槽的主要目的是为了方便数据拆分和集群扩展。 每个节点会负责一定数量的槽

Redis Cluster就是采用虚拟槽分区

### Redis数据分区

Redis Cluser采用虚拟槽分区， 所有的键根据哈希函数映射到0~16383整数槽内， 计算公式： $slot=CRC16(key) \&16383$。每个节点负责维护一部分槽以及槽所映射的键值数据。如图所示：

![RedisCluster槽集合与节点关系](https://gitee.com/wangziming707/note-pic/raw/master/img/RedisCluster%E6%A7%BD%E9%9B%86%E5%90%88%E4%B8%8E%E8%8A%82%E7%82%B9%E5%85%B3%E7%B3%BB.png)

Redis虚拟槽分区的特点：

* 解耦数据和节点之间的关系， 简化了节点扩容和收缩难度
* 节点自身维护槽的映射关系， 不需要客户端或者代理服务维护槽分区元数据
* 支持节点、 槽、 键之间的映射查询， 用于数据路由、 在线伸缩等场景

数据分区是分布式存储的核心，理解和灵活运用数据分区规则对于掌握Redis Cluster非常有帮助

### 集群功能限制

Redis集群相对单机在功能上存在一些限制：

* key批量操作支持有限。 如mset、 mget， 目前只支持具有相同slot值的key执行批量操作。 对于映射为不同slot值的key由于执行mget、 mget等操作可能存在于多个节点上因此不被支持
* key事务操作支持有限。 同理只支持多key在同一节点上的事务操作， 当多个key分布在不同的节点上时无法使用事务功能
* key作为数据分区的最小粒度， 因此不能将一个大的键值对象如hash、 list等映射到不同的节点
* 不支持多数据库空间。 单机下的Redis可以支持16个数据库， 集群模式下只能使用一个数据库空间， 即db0
* 复制结构只支持一层， 从节点只能复制主节点， 不支持嵌套树状复制结构

## 搭建集群

接下来我们来搭建一个Redis集群作为示例

### 准备节点

Redis集群一般由多个节点组成， 节点数量至少为6个才能保证组成完整
高可用的集群。 每个节点需要开启配置cluster-enabled yes， 让Redis运行在集
群模式下。一般节点数量为6个可以组成完成的高可用集群，配置如下，其他配置修改端口号即可：

redis-6379.conf:

~~~bash
port 6379
daemonize yes
logfile "6379.log"
dbfilename "dump-6379.rdb"
dir "/opt/soft/redis/data"
# 开启集群模式
cluster-enabled yes 
# 节点超时时间， 单位毫秒
cluster-node-timeout 15000 
# 集群内部配置文件
cluster-config-file "nodes-6379.conf" 
~~~

准备好配置后启动所有节点:

~~~bash
redis-server redis-6379.conf 
redis-server redis-6380.conf 
redis-server redis-6381.conf 
redis-server redis-6382.conf 
redis-server redis-6383.conf 
redis-server redis-6384.conf
~~~

节点启动成功后，第一次启动如果没有集群配置文件，它会自动创建一份，文件名通过`cluster-config-file`参数控制，建议采用`node-{port}.conf`格式定义，通过端口号区分不同节点。如果启动时存在集群配置文件，节点会使用配置文件内容初始化集群信息。

集群模式的Redis除了原有的配置文件之外又加了一份集群配置文件。当集群内节点信息发生变化， 如添加节点、 节点下线、 故障转移等。 节点会自动保存集群状态到配置文件中。  

如节点6379首次启动后生成集群配置如下：

~~~bash
cat nodes-6379.conf 
a3b8b1245e5602f3759c83bf7a92537eaec0f30c :0@0 myself,master - 0 0 0 connected
vars currentEpoch 0 lastVoteEpoch 0
~~~

记录了集群初始状态，其中最重要的是节点ID，它是一个40位16进制字符串，用于唯一标识集群内一个节点。

节点ID不同于运行ID。 节点ID在集群初始化时只创建一次， 节点重启时会加载集群配置文件进行重用， 而Redis的运行ID每次重启都会变化  

也可以通过执行`cluster nodes`命令获取当前集群节点状态：

~~~bash
127.0.0.1:6380> cluster nodes
ddc23f07b96e1f6adac80649ee281344e8f728a3 :6380@16380 myself,master - 0 0 0 connected
~~~

目前我们只启动了6个节点，每个节点彼此不知道对方的存在，下面通过节点握手让6个节点彼此建立联系从而组成一个集群

### 节点握手

节点握手是指一批运行在集群模式下的节点通过Gossip协议彼此通信，达到感知对方的过程。

通过客户端发起命令：` cluster meet{ip}{port}`发起节点握手操作，该命令让当前节点与指定节点握手。

例如，在6379节点，发起命令` cluster meet 127.0.0.1 6380`会进行握手通信：

* 节点6379本地创建6380节点信息对象， 并发送meet消息。
* 节点6380接受到meet消息后， 保存6379节点信息并回复pong消息。
* 之后节点6379和6380彼此定期通过ping/pong消息进行正常的节点通信。

这样执行完成后，在6379和6380节点分别执行cluster nodes命令，可以看到他们彼此已经感知到对方的存在：

~~~bash
$ cat nodes-6379.conf 
ddc23f07b96e1f6adac80649ee281344e8f728a3 127.0.0.1:6380@16380 master - 0 1709703365358 0 connected
a3b8b1245e5602f3759c83bf7a92537eaec0f30c 127.0.0.1:6379@16379 myself,master - 0 0 1 connected
vars currentEpoch 1 lastVoteEpoch 0
$ cat nodes-6380.conf 
a3b8b1245e5602f3759c83bf7a92537eaec0f30c 127.0.0.1:6379@16379 master - 0 1709703365459 1 connected
ddc23f07b96e1f6adac80649ee281344e8f728a3 127.0.0.1:6380@16380 myself,master - 0 0 0 connected
vars currentEpoch 1 lastVoteEpoch 0
~~~

下面分别执行meet命令让其他节点加入到集群中：

~~~bash
127.0.0.1:6379> cluster meet 127.0.0.1 6381
OK
127.0.0.1:6379> cluster meet 127.0.0.1 6382
OK
127.0.0.1:6379> cluster meet 127.0.0.1 6383
OK
127.0.0.1:6379> cluster meet 127.0.0.1 6384
OK
~~~

我们只需要在集群内任意节点上执行cluster meet命令加入新节点， 握手状态会通过消息在集群内传播， 这样其他节点会自动发现新节点并发起握手流程。 最后执行cluster nodes命令确认6个节点都彼此感知并组成集群:

~~~bash
127.0.0.1:6379> cluster nodes
ddc23f07b96e1f6adac80649ee281344e8f728a3 127.0.0.1:6380@16380 master - 0 1709703551482 2 connected
37b01feee2ce3c8be540cb8a21d1a7f4cc7b4885 127.0.0.1:6384@16384 master - 0 1709703552485 5 connected
3c47191f6d501c6ec47cea38023aeca9ba21bd80 127.0.0.1:6383@16383 master - 0 1709703552000 4 connected
96ca10fd3b2fee86df6b72c67efc4d20c46a55de 127.0.0.1:6382@16382 master - 0 1709703551000 3 connected
a3b8b1245e5602f3759c83bf7a92537eaec0f30c 127.0.0.1:6379@16379 myself,master - 0 1709703548000 1 connected
eaf9bdc4cf636784cf3b20e19ed27bc17e9d9cf8 127.0.0.1:6381@16381 master - 0 1709703553488 0 connected
~~~

节点握手后集群还不能正常工作，此时集群处于下线状态，所有数据读写都被禁止，例如：

~~~bash
127.0.0.1:6379> set hello world
(error) CLUSTERDOWN Hash slot not served
~~~

可以看到哈希槽还没有分配，通过cluster info 命令可以获取集群当前状态：

~~~bash
127.0.0.1:6379> cluster info
cluster_state:fail
cluster_slots_assigned:0
cluster_slots_ok:0
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:6
cluster_size:0
......
~~~

从输出内容可以看到， 被分配的槽（cluster_slots_assigned） 是0， 由于目前所有的槽没有分配到节点， 因此集群无法完成槽到节点的映射。 只有当16384个槽全部分配给节点后， 集群才进入在线状态。

### 分配槽

Redis集群把所有的数据映射到16384个槽中。 每个key会映射为一个固定的槽， 只有当节点分配了槽， 才能响应和这些槽关联的键命令。 通过cluster addslots命令为节点分配槽。 这里利用bash特性批量设置槽(slots)，命令如下：  

~~~bash
redis-cli -p 6379 cluster addslots {0..5461}
redis-cli -p 6380 cluster addslots {5462..10922}
redis-cli -p 6381 cluster addslots {10923..16383}
~~~

把16384个slot平均分配给6379、 6380、 6381三个节点。 执行cluster info查看集群状态:

~~~bash
127.0.0.1:6379> cluster info
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:6
cluster_size:3
cluster_current_epoch:5
cluster_my_epoch:1
cluster_stats_messages_ping_sent:1385
cluster_stats_messages_pong_sent:1282
cluster_stats_messages_meet_sent:6
cluster_stats_messages_sent:2673
cluster_stats_messages_ping_received:1282
cluster_stats_messages_pong_received:1391
cluster_stats_messages_received:2673
~~~

当前集群状态是OK， 集群进入在线状态。 所有的槽都已经分配给节点

执行cluster nodes命令可以看到节点和槽的分配关系：

~~~bash
127.0.0.1:6379> cluster nodes
ddc23f07b96e1f6adac80649ee281344e8f728a3 127.0.0.1:6380@16380 master - 0 1709704807000 2 connected 5462-10922
37b01feee2ce3c8be540cb8a21d1a7f4cc7b4885 127.0.0.1:6384@16384 master - 0 1709704807000 5 connected
3c47191f6d501c6ec47cea38023aeca9ba21bd80 127.0.0.1:6383@16383 master - 0 1709704805000 4 connected
96ca10fd3b2fee86df6b72c67efc4d20c46a55de 127.0.0.1:6382@16382 master - 0 1709704806000 3 connected
a3b8b1245e5602f3759c83bf7a92537eaec0f30c 127.0.0.1:6379@16379 myself,master - 0 1709704805000 1 connected 0-5461
eaf9bdc4cf636784cf3b20e19ed27bc17e9d9cf8 127.0.0.1:6381@16381 master - 0 1709704807998 0 connected 10923-16383
~~~

目前还有三个节点没有使用， 作为一个完整的集群， 每个负责处理槽的节点应该具有从节点， 保证当它出现故障时可以自动进行故障转移。 集群模式下， Redis节点角色分为主节点和从节点。 

首次启动的节点和被分配槽的节点都是主节点， 从节点负责复制主节点槽信息和相关的数据。 使用`cluster replicate{nodeId}`命令让一个节点成为从节点。 其中命令执行必须在对应的从节点上执行， nodeId是要复制主节点的节点ID:

~~~bash
$ redis-cli -p 6382 cluster replicate a3b8b1245e5602f3759c83bf7a92537eaec0f30c
$ redis-cli -p 6383 cluster replicate ddc23f07b96e1f6adac80649ee281344e8f728a3
$ redis-cli -p 6384 cluster replicate eaf9bdc4cf636784cf3b20e19ed27bc17e9d9cf8
~~~

Redis集群模式下的主从复制流程就是之前介绍的复制流程，支持全量和部分复制。复制完成后，整个集群的结构如图：

![RedisCluster集群完整结构示意图](https://gitee.com/wangziming707/note-pic/raw/master/img/RedisCluster%E9%9B%86%E7%BE%A4%E5%AE%8C%E6%95%B4%E7%BB%93%E6%9E%84%E7%A4%BA%E6%84%8F%E5%9B%BE.png)

通过cluster nodes命令查看集群状态和复制关系， 如下所示：

~~~bash
127.0.0.1:6379> cluster nodes
ddc23f07b96e1f6adac80649ee281344e8f728a3 127.0.0.1:6380@16380 master - 0 1709707178000 2 connected 5462-10922
37b01feee2ce3c8be540cb8a21d1a7f4cc7b4885 127.0.0.1:6384@16384 slave eaf9bdc4cf636784cf3b20e19ed27bc17e9d9cf8 0 1709707178565 0 connected
3c47191f6d501c6ec47cea38023aeca9ba21bd80 127.0.0.1:6383@16383 slave ddc23f07b96e1f6adac80649ee281344e8f728a3 0 1709707176000 2 connected
96ca10fd3b2fee86df6b72c67efc4d20c46a55de 127.0.0.1:6382@16382 slave a3b8b1245e5602f3759c83bf7a92537eaec0f30c 0 1709707179568 1 connected
a3b8b1245e5602f3759c83bf7a92537eaec0f30c 127.0.0.1:6379@16379 myself,master - 0 1709707178000 1 connected 0-5461
eaf9bdc4cf636784cf3b20e19ed27bc17e9d9cf8 127.0.0.1:6381@16381 master - 0 1709707177560 0 connected 10923-16383
~~~

这样，我们按照Redis协议手动搭建了一个集权，由6个节点构成，三个主节点负责处理槽和相关数据，三个从节点负责故障转移。

### redis-cli搭建集群

可以看到手动搭建需要很多步骤，当集群节点众多时，会增大搭建的复杂度和运营成本

redis5.0版本以上的redis-cli客户端支持`--cluster create`命令来快速搭建集群

首先仍然要完成准备节点的工作，创建配置文件并启动redis节点，此时节点间相互独立，没有组成集群。而`--cluster create`命令可以自动快速完成节点握手和分配槽的流程：

~~~bash
# 其中可选参数cluster-replicas用于指定集群中每个主节点的从节点数量，
# --cluster-replicas 1 即指明一主一从
$ redis-cli --cluster create 127.0.0.1:6379 127.0.0.1:6380 127.0.0.1:6381 127.0.0.1:6382 127.0.0.1:6383 127.0.0.1:6384 --cluster-replicas 1

>>> Performing hash slots allocation on 6 nodes...
Master[0] -> Slots 0 - 5460
Master[1] -> Slots 5461 - 10922
Master[2] -> Slots 10923 - 16383
Adding replica 127.0.0.1:6383 to 127.0.0.1:6379
Adding replica 127.0.0.1:6384 to 127.0.0.1:6380
Adding replica 127.0.0.1:6382 to 127.0.0.1:6381
.......
~~~

此时执行`cluster nodes`命令，可以看到节点直接已经建立通信、形成集群并分配号槽：

~~~bash
127.0.0.1:6379> cluster nodes
986ed98c1603ca82b25a5f28b8a575b97bfbe7a1 127.0.0.1:6382@16382 slave 7dd0459a12e82fd42104e60496049d76d8e2da4f 0 1709708253000 1 connected
3b65631a962382c982e5324913b55468a02beecf 127.0.0.1:6381@16381 master - 0 1709708254000 3 connected 10923-16383
1151a31159c344f93ab585095a0e161f6fd5d776 127.0.0.1:6384@16384 slave 3b65631a962382c982e5324913b55468a02beecf 0 1709708255512 3 connected
7dd0459a12e82fd42104e60496049d76d8e2da4f 127.0.0.1:6379@16379 myself,master - 0 1709708256000 1 connected 0-5460
ea6863f8a22f74616af5757c1bf77fe8b781ea1c 127.0.0.1:6380@16380 master - 0 1709708256515 2 connected 5461-10922
f6fb46edc185372da78ea9250bda31b7cab02193 127.0.0.1:6383@16383 slave ea6863f8a22f74616af5757c1bf77fe8b781ea1c 0 1709708254508 2 connected
~~~

## 节点通信

### 通信流程

在分布式存储中需要提供维护节点元数据信息的机制， 所谓元数据是指： 节点负责哪些数据， 是否出现故障等状态信息。 常见的元数据维护方式分为： 集中式和P2P方式。 

Redis集群采用P2P的Gossip(流言)协议，Gossip协议工作原理就是节点彼此不断通信交换信息， 一段时间后所有的节点都会知道集群完整的信息， 这种方式类似流言传播。

Gossip协议通信过程：

* 集群中的每个节点都会单独开辟一个TCP通道， 用于节点之间彼此通信， 通信端口号在基础端口上加10000
* 每个节点在固定周期内通过特定规则选择几个节点发送ping消息
* 接收到ping消息的节点用pong消息作为响应

集群中每个节点通过一定规则挑选要通信的节点， 每个节点可能知道全部节点， 也可能仅知道部分节点， 只要这些节点彼此可以正常通信， 最终它们会达到一致的状态。 

当节点出故障、 新节点加入、 主从角色变化、 槽信息变更等事件发生时， 通过不断的ping/pong消息通信， 经过一段时间后所有的节点都会知道整个集群全部节点的最新状态， 从而达到集群状态同步的目的。

### Gossip消息

Gossip协议的主要职责就是信息交换。 信息交换的载体就是节点彼此发送的Gossip消息

常用的Gossip消息可分为： ping消息、 pong消息、 meet消息、 fail消息等

* meet消息：用于通知新节点加入。 消息发送者通知接收者加入到当前集群， meet消息通信正常完成后， 接收节点会加入到集群中并进行周期性的ping、 pong消息交换
* ping消息： 集群内交换最频繁的消息， 集群内每个节点每秒向多个其他节点发送ping消息， 用于检测节点是否在线和交换彼此状态信息。 ping消息发送封装了自身节点和部分其他节点的状态数据。
* pong消息： 当接收到ping、 meet消息时， 作为响应消息回复给发送方确认消息正常通信。 pong消息内部封装了自身状态数据。 节点也可以向集群内广播自身的pong消息来通知整个集群对自身状态进行更新。
* fail消息： 当节点判定集群内另一个节点下线时， 会向集群内广播一个fail消息， 其他节点接收到fail消息之后把对应节点更新为下线状态

### 节点选择

虽然Gossip协议的信息交换机制具有天然的分布式特性， 但它是有成本的。

由于内部需要频繁地进行节点信息交换， 而ping/pong消息会携带当前节点和部分其他节点的状态数据， 势必会加重带宽和计算的负担。 Redis集群内节点通信采用固定频率(定时任务每秒执行10次)。 因此节点每次选择需要通信的节点列表变得非常重要:

* 通信节点选择过多虽然可以做到信息及时交换但成本过高。
*  节点选择过少会降低集群内所有节点彼此信息交换频率，从而影响故障判定、 新节点发现等需求的速度。

因此Redis集群的Gossip协议需要兼顾信息交换实时性和成本开销，其选择的规则如图：

![Redis集群Gossip协议节点选择](https://gitee.com/wangziming707/note-pic/raw/master/img/Redis%E9%9B%86%E7%BE%A4Gossip%E5%8D%8F%E8%AE%AE%E8%8A%82%E7%82%B9%E9%80%89%E6%8B%A9.png)

集群内每个节点维护定时任务默认每秒执行10次， 每秒会随机选取5个节点找出最久没有通信的节点发送ping消息， 用于保证Gossip信息交换的随机性。 

每100毫秒都会扫描本地节点列表， 如果发现节点最近一次接受pong消息的时间大于cluster_node_timeout/2， 则立刻发送ping消息， 防止该节点信息太长时间未更新。

根据以上规则得出每个节点每秒需要发送ping消息的数量=1+10*num(node.pong_received > cluster_node_timeout / 2) ， 因此cluster_node_timeout参数对消息发送的节点数量影响非常大，该参数值越大，带宽消耗越低，但是消息交换会降低，从而影响故障转移、 槽信息更新、 新节点发现的速度

因此需要根据业务容忍度和资源消耗进行平衡，同时整个集群消息总交换量也跟节点数成正比。 

## 集群伸缩

Redis集群提供了灵活的节点扩容和收缩方案。 在不影响集群对外服务的情况下， 可以为集群添加节点进行扩容也可以下线部分节点进行缩容。

Redis集群实现对节点的灵活上下线控制的原理可抽象为槽和对应数据在不同节点之间灵活移动。

例如当一个集群加入新节点后，原节点会向新节点迁移一部分负责的槽和对应的数据。

### 扩容集群

Redis集群扩容可以分为如下步骤：

* 准备新节点
* 加入集群
* 迁移槽和数据

#### 准备新节点

需要提前准备好新节点并运行在集群模式下， 新节点建议跟集群内的节点配置保持一致， 便于管理统一 。

准备好两个新节点配置并启动新节点：

~~~bash
$ redis-server redis-6385.conf
$ redis-server redis-6386.conf
~~~

此时两个新节点还没有加入集群与其他节点通信

#### 加入集群

可以通过`cluster meet`命令加入到现有集群，也可以使用 `redis-cli --cluster `命令

我们以 `--cluster add-node`为例，其参数有：

~~~bash
add-node  new_host:new_port  		#新加入集群的ip和port
	existing_host:existing_port     #集群中任一节点的ip和port
	--cluster-slave					#新节点作为从节点，默认随机一个主节点
	--cluster-master-id <arg> 	 	#给新节点指定主节点,值为节点的节点ID
~~~

那么可以执行下面命令：

~~~bash
redis-cli --cluster add-node 127.0.0.1:6385 127.0.0.1:6379
redis-cli --cluster add-node 127.0.0.1:6386 127.0.0.1:6379 --cluster-slave --cluster-master-id d7368f2323c2bd079cf31b5efe64827826d6bbb1 #为6385的节点ID
~~~

这样将6385作为主节点，6386作为其从节点加入到集群，然后执行`cluster nodes`可以看到如下信息：

~~~bash
$ redis-cli cluster nodes
986ed98c1603ca82b25a5f28b8a575b97bfbe7a1 127.0.0.1:6382@16382 slave 7dd0459a12e82fd42104e60496049d76d8e2da4f 0 1709711926058 1 connected
d7368f2323c2bd079cf31b5efe64827826d6bbb1 127.0.0.1:6385@16385 master - 0 1709711925000 0 connected
3b65631a962382c982e5324913b55468a02beecf 127.0.0.1:6381@16381 master - 0 1709711924052 3 connected 10923-16383
8192d66507511b3aee8a3cc16a639df7549a7714 127.0.0.1:6386@16386 slave d7368f2323c2bd079cf31b5efe64827826d6bbb1 0 1709711925055 0 connected
1151a31159c344f93ab585095a0e161f6fd5d776 127.0.0.1:6384@16384 slave 3b65631a962382c982e5324913b55468a02beecf 0 1709711925000 3 connected
7dd0459a12e82fd42104e60496049d76d8e2da4f 127.0.0.1:6379@16379 myself,master - 0 1709711925000 1 connected 0-5460
ea6863f8a22f74616af5757c1bf77fe8b781ea1c 127.0.0.1:6380@16380 master - 0 1709711928064 2 connected 5461-10922
f6fb46edc185372da78ea9250bda31b7cab02193 127.0.0.1:6383@16383 slave ea6863f8a22f74616af5757c1bf77fe8b781ea1c 0 1709711927061 2 connected
~~~

此时6385作为主节点还没有分配槽位

#### 迁移槽和数据

可以通过`--cluster rehard`命令进行槽位的迁移，其命令参数如下：

~~~bash
reshard	host:port
	--cluster-from <arg>     #槽位来源的节点节点ID，多个用,分割，all表示全部节点
	--cluster-to <arg>       #目标节点的节点ID，只允一个
	--cluster-slots <arg>    #迁移的槽位数
	--cluster-yes            #是否默认同意集群内部的迁移计划（默认同意就可以）
	--cluster-timeout <arg>  #迁移命令（migrate）的超时时间
	--cluster-pipeline <arg> #迁移key时，一次取出 的key数量，默认10
	--cluster-replace        #是否直接replace到目标节点
~~~

那么就可以执行如下命令将槽位迁移到新节点：

~~~bash
redis-cli --cluster reshard 127.0.0.1:6379 --cluster-from all --cluster-to d7368f2323c2bd079cf31b5efe64827826d6bbb1 --cluster-slots 4096 
~~~

迁移完成后重新查看cluster信息：

~~~bash
986ed98c1603ca82b25a5f28b8a575b97bfbe7a1 127.0.0.1:6382@16382 slave 7dd0459a12e82fd42104e60496049d76d8e2da4f 0 1709712686307 1 connected
d7368f2323c2bd079cf31b5efe64827826d6bbb1 127.0.0.1:6385@16385 master - 0 1709712688312 8 connected 0-1364 5461-6826 10923-12287
3b65631a962382c982e5324913b55468a02beecf 127.0.0.1:6381@16381 master - 0 1709712685303 3 connected 12288-16383
8192d66507511b3aee8a3cc16a639df7549a7714 127.0.0.1:6386@16386 slave d7368f2323c2bd079cf31b5efe64827826d6bbb1 0 1709712685000 8 connected
1151a31159c344f93ab585095a0e161f6fd5d776 127.0.0.1:6384@16384 slave 3b65631a962382c982e5324913b55468a02beecf 0 1709712685000 3 connected
7dd0459a12e82fd42104e60496049d76d8e2da4f 127.0.0.1:6379@16379 myself,master - 0 1709712685000 1 connected 1365-5460
ea6863f8a22f74616af5757c1bf77fe8b781ea1c 127.0.0.1:6380@16380 master - 0 1709712687000 2 connected 6827-10922
f6fb46edc185372da78ea9250bda31b7cab02193 127.0.0.1:6383@16383 slave ea6863f8a22f74616af5757c1bf77fe8b781ea1c 0 1709712687310 2 connected
~~~

可以看到新节点被分配了 0-1364 5461-6826 10923-12287 这些槽位

### 收缩集群

收缩集群需要下线部分节点。首先需要将要下线的节点的槽分配给其他节点，然后才能从集群中安全下线。

#### 迁移槽

为了保证集群之间的平衡，我们需要保证迁移后的各个节点持有的槽的个数大致相同。

为此我们不必使用多次`--cluster rehard`将槽分多次迁移给其他节点。而是可以直接使用一次`--cluster rehard`将所有槽位一次迁移出去，等安全下线后，再使用`--cluster rebalance`来平衡集群。

那么首先迁移槽位：

~~~bash
redis-cli --cluster reshard 127.0.0.1:6379 --cluster-from d7368f2323c2bd079cf31b5efe64827826d6bbb1 --cluster-to 7dd0459a12e82fd42104e60496049d76d8e2da4f --cluster-slots 4096
~~~

迁移完成后，重新查看cluster信息：

~~~bash
$ redis-cli cluster nodes
986ed98c1603ca82b25a5f28b8a575b97bfbe7a1 127.0.0.1:6382@16382 slave 7dd0459a12e82fd42104e60496049d76d8e2da4f 0 1709713708000 9 connected
d7368f2323c2bd079cf31b5efe64827826d6bbb1 127.0.0.1:6385@16385 master - 0 1709713709382 8 connected
3b65631a962382c982e5324913b55468a02beecf 127.0.0.1:6381@16381 master - 0 1709713706000 3 connected 12288-16383
8192d66507511b3aee8a3cc16a639df7549a7714 127.0.0.1:6386@16386 slave 7dd0459a12e82fd42104e60496049d76d8e2da4f 0 1709713706372 9 connected
1151a31159c344f93ab585095a0e161f6fd5d776 127.0.0.1:6384@16384 slave 3b65631a962382c982e5324913b55468a02beecf 0 1709713708378 3 connected
7dd0459a12e82fd42104e60496049d76d8e2da4f 127.0.0.1:6379@16379 myself,master - 0 1709713705000 9 connected 0-6826 10923-12287
ea6863f8a22f74616af5757c1bf77fe8b781ea1c 127.0.0.1:6380@16380 master - 0 1709713707000 2 connected 6827-10922
f6fb46edc185372da78ea9250bda31b7cab02193 127.0.0.1:6383@16383 slave ea6863f8a22f74616af5757c1bf77fe8b781ea1c 0 1709713710384 2 connected
~~~

可以看到3685节点已经没有槽位了。

#### 删除节点

可以使用`--cluster del-node`删除集群中的节点。其参数如下：

~~~bash
del-node host:port node_id
~~~

为了防止全量复制，建议先删除从节点，再删除主节点

~~~bash
redis-cli --cluster del-node 127.0.0.1:6379 8192d66507511b3aee8a3cc16a639df7549a7714
redis-cli --cluster del-node 127.0.0.1:6379 d7368f2323c2bd079cf31b5efe64827826d6bbb1
~~~

我们再确认当前集群的状态:

~~~bash
redis-cli cluster nodes
986ed98c1603ca82b25a5f28b8a575b97bfbe7a1 127.0.0.1:6382@16382 slave 7dd0459a12e82fd42104e60496049d76d8e2da4f 0 1709714267000 9 connected
3b65631a962382c982e5324913b55468a02beecf 127.0.0.1:6381@16381 master - 0 1709714265100 3 connected 12288-16383
1151a31159c344f93ab585095a0e161f6fd5d776 127.0.0.1:6384@16384 slave 3b65631a962382c982e5324913b55468a02beecf 0 1709714266000 3 connected
7dd0459a12e82fd42104e60496049d76d8e2da4f 127.0.0.1:6379@16379 myself,master - 0 1709714266000 9 connected 0-6826 10923-12287
ea6863f8a22f74616af5757c1bf77fe8b781ea1c 127.0.0.1:6380@16380 master - 0 1709714268109 2 connected 6827-10922
f6fb46edc185372da78ea9250bda31b7cab02193 127.0.0.1:6383@16383 slave ea6863f8a22f74616af5757c1bf77fe8b781ea1c 0 1709714267106 2 connected
~~~

可以看到两个节点确实已经下线

#### 再平衡集群

我们为了方便，将要下线的节点的所有槽位都转移给了一个节点，这造成了集群槽位的不平衡。如下：

~~~bash
redis-cli --cluster info 127.0.0.1:6379
127.0.0.1:6379 (7dd0459a...) -> 0 keys | 8192 slots | 1 slaves.
127.0.0.1:6381 (3b65631a...) -> 0 keys | 4096 slots | 1 slaves.
127.0.0.1:6380 (ea6863f8...) -> 0 keys | 4096 slots | 1 slaves.
~~~

可以使用`--cluster rebalance`命令再平衡集群，其参数如下：

~~~bash
rebalance	host:port
	--cluster-weight <node1=w1...nodeN=wN>#槽位权重（浮点型）比值，例如（这里使用端口号代替运行id）： 7001=1.0,7002=1.0,7003=2.0 则表示，总权重为4.0， 7001和7002将分配到 (16384/4)*1  个槽位，7003则是(16384/4)*2个槽位
	--cluster-use-empty-masters#再平衡时将空主节点也考虑进去
	--cluster-timeout <arg>    #迁移命令（migrate）的超时时间
	--cluster-simulate         # 模拟rebalance操作，不会真正执行迁移操作
	--cluster-pipeline <arg>   #定义 getkeysinslot命令一次取出的key数量，默认值为10
	--cluster-threshold <arg>  #平衡触发的阈值条件，默认为2.00%。例如上一步，7001和7002应该有4096个槽位，如果7001的槽位数不够4096个，且超过 (4096*0.02 约等于)82个及以上；或者7001的槽位数比4096少于82个及以上，则会触发自平衡
	--cluster-replace          #是否直接replace到目标节点
~~~

那么可以执行如下命令：

~~~bash
redis-cli --cluster rebalance 127.0.0.1:6379
~~~

再看当前集群状态：

~~~bash
$ redis-cli --cluster info 127.0.0.1:6379
127.0.0.1:6379 (7dd0459a...) -> 0 keys | 5461 slots | 1 slaves.
127.0.0.1:6381 (3b65631a...) -> 0 keys | 5462 slots | 1 slaves.
127.0.0.1:6380 (ea6863f8...) -> 0 keys | 5461 slots | 1 slaves.
~~~

可以看到槽位已经平衡

## 请求路由

Redis集群对客户端做了比较大的修改，为了追求性能最大化， 并没有采用代理的方式而是采用客户端直连节点的方式。 因此对于希望从单机切换到集群环境的应用需要修改客户端代码。

### 请求重定向

在集群模式下， Redis接收任何键相关命令时首先计算键对应的槽， 再根据槽找出所对应的节点， 如果节点是自身， 则处理键命令； 否则回复MOVED重定向错误， 通知客户端请求正确的节点。 这个过程称为MOVED重定向。

如：

~~~bash
127.0.0.1:6379> set hello world
(error) MOVED 866 127.0.0.1:6381
~~~

**注意:**节点对于不属于它的键命令只回复重定向响应， 并不负责转发  

![RedisCluster客户端请求重定向流程](https://gitee.com/wangziming707/note-pic/raw/master/img/RedisCluster%E5%AE%A2%E6%88%B7%E7%AB%AF%E8%AF%B7%E6%B1%82%E9%87%8D%E5%AE%9A%E5%90%91%E6%B5%81%E7%A8%8B.png)

可以使用`cluster keyslot {key}`返回key对应的槽

使用`redis-cli`命令时，可以加入参数`-c`支持自动重定向，例如：

~~~bash
$ redis-cli -c
127.0.0.1:6379> set hello world
-> Redirected to slot [866] located at 127.0.0.1:6381
OK
127.0.0.1:6381> #接下来客户端就连接到了6381
~~~

redis-cli自动帮我们连接到正确的节点执行命令，这个过程是在`client-cli`内部维护的，实质上时client收到MOEVD消息后再次发起请求，而不是在Redis节点中完成请求转发。

### Smart客户端

可以看到这种请求重定向的方式对客户端协议影响很小，但是弊端很明显：每次执行键命令前都要到Redis上进行重定向才能找到要执行命令的节点， 额外增加了IO开销

所以通常集群客户端都采用另一种实现： Smart(智能)客户端

#### 原理

Smart客户端通过在内部维护slot->node的映射关系，本地久可实现键到节点的查找，从而保证IO效率的最大化，而MOEVD重定向负责协助Smart客户端更新slot->node映射。

我们以Java的Jedis为例，说明Smart客户端操作集群的流程：

* 首先在JedisCluster初始化时会选择一个运行节点， 初始化槽和节点映射关系， 使用cluster slots命令完成：

  ~~~bash
  cluster slots
  1) 1) (integer) 0 #开始槽范围
     2) (integer) 1365 #结束槽范围
     3) 1) "127.0.0.1" #主节点IP
        2) (integer) 6381 #主节点端口
        3) "3b65631a962382c982e5324913b55468a02beecf" #主节点ID
     4) 1) "127.0.0.1" #从节点IP
        2) (integer) 6384 #从节点端口
        3) "1151a31159c344f93ab585095a0e161f6fd5d776" #从节点ID
  ......
  ~~~

* JedisCluster解析cluster slots结果缓存在本地， 并为每个节点创建唯 一的JedisPool连接池。 映射关系在JedisClusterInfoCache类中

* JedisCluster执行键命令

  * 计算slot并根据slots缓存获取目标节点连接， 发送命令
  * 如果出现连接错误， 使用随机连接重新执行键命令， 每次命令重试对redi-rections参数减1
  * 捕获到MOVED重定向错误， 使用cluster slots命令更新slots缓存
  * 重复执行以上三步，直到命令执行成功， 或者当`redirections<=0`时抛出Jedis ClusterMaxRedirectionsException异常

####  JedisCluster

Jedis为Redis Cluster提供了Smart客户端， 对应的类是JedisCluster， 它的常用初始化方法如下：

~~~java
public JedisCluster(Set<HostAndPort> jedisClusterNode, int connectionTimeout, int soTimeout,
      int maxAttempts, final GenericObjectPoolConfig poolConfig)
~~~

* `jedisClusterNode`： 所有Redis Cluster节点信息(也可以是一部分， 因为客户端可以通过cluster slots自动发现)
* connectionTimeout连接超时

* soTimeout：读写超时
* maxAttempts：重试次数

* poolConfig：连接池参数  JedisCluster会为Redis Cluster的每个节点创建连接池  

对于JedisCluster的使用需要注意以下几点：

* JedisCluster包含了所有节点的连接池(JedisPool) ， 所以建议JedisCluster使用单例
* JedisCluster每次操作完成后， 不需要管理连接池的借还， 它在内部已经完成
* JedisCluster一般不要执行close() 操作， 它会将所有JedisPool执行destroy操作

**多节点指令和操作**

Redis Cluster虽然提供了分布式的特性， 但是有些命令或者操作， 诸如keys、 flushall、 删除指定模式的键， 需要遍历所有节点才可以完成，以删除指定模式的键为例，过程如下：

* 通过jedisCluster.getClusterNodes()获取所有节点的连接池
* 使用info replication筛选第一步中的主节点。
* 遍历主节点， 使用scan命令找到指定模式的key， 使用Pipeline机制删除

**批量操作**

Redis Cluster中， 由于key分布到各个节点上， 会造成无法实现mget、mset等功能。 但是可以利用CRC16算法计算出key对应的slot， 以及Smart客户端保存了slot和节点对应关系的特性， 将属于同一个Redis节点的key进行归档， 然后分别对每个节点对应的子key列表执行mget或者pipeline操作

**使用Lua、事务特性**

Lua和事务需要所操作的key， 必须在一个节点上，不过Redis Cluster提供了hashtag， 如果开发人员确实要使用Lua或者事务， 可以将所要操作的key使用一个hashtag

### ASK重定向

Redis集群支持在线迁移槽（slot） 和数据来完成水平伸缩， 当slot对应的数据从源节点到目标节点迁移过程中， 客户端需要做到智能识别， 保证键命令可正常执行。 例如当一个slot数据从源节点迁移到目标节点时， 期间可能出现一部分数据在源节点， 而另一部分在目标节点。

当出现上述情况时， 客户端键命令执行流程将发生变化：

* 客户端根据本地slots缓存发送命令到源节点， 如果存在键对象则直接执行并返回结果给客户端
* 如果键对象不存在， 则可能存在于目标节点， 这时源节点会回复ASK重定向异常。 格式如下： `(error) ASK{slot}{targetIP}:{targetPort}  `
* 客户端从ASK重定向异常提取出目标节点信息， 发送asking命令到目标节点打开客户端连接标识， 再执行键命令。 如果存在则执行， 不存在则返回不存在信息

ASK与MOVED虽然都是对客户端的重定向控制， 但是有着本质区别。ASK重定向说明集群正在进行slot数据迁移， 客户端无法知道什么时候迁移完成， 因此只能是临时性的重定向， 客户端不会更新slots缓存。 但是MOVED重定向说明键对应的槽已经明确指定到新的节点， 因此需要更新slots缓存  

## 故障转移

Redis集群本身实现了高可用。高可用首先需要能够实现故障转移

### 故障发现

当集群内某个节点出现问题时， 需要通过一种健壮的方式保证识别出节点是否发生了故障。

Redis集群内节点通过ping/pong消息实现节点通信， 消息不但可以传播节点槽信息， 还可以传播其他状态如： 主从状态、 节点故障等。 因此故障发现也是通过消息传播机制实现的主要环节包括主观下线和客观下线：

* 主观下线： 指某个节点认为另一个节点不可用， 即下线状态， 这个状态并不是最终的故障判定， 只能代表一个节点的意见， 可能存在误判情况。
* 客观下线： 指标记一个节点真正的下线， 集群内多个节点都认为该节点不可用， 从而达成共识的结果。 如果是持有槽的主节点故障， 需要为该节点进行故障转移

#### 主观下线

集群中每个节点都会定期向其他节点发送ping消息， 接收节点回复pong消息作为响应。 如果在cluster-node-timeout时间内通信一直失败， 则发送节点会认为接收节点存在故障， 把接收节点标记为主观下线（pfail） 状态。

一个节点判定的主观下线并不能让集群真正认定节点已经下线，可能存在只有两个节点间通信存在问题，而与其他节点通信正常的情况。

#### 客观下线

当某个节点判断另一个节点主观下线后， 相应的节点状态会跟随消息在集群内传播

当接受节点发现消息体中含有主观下线的节点状态时，会将其保存在本地。

通过Gossip消息传播， 集群内节点不断收集到故障节点的下线报告。 当半数以上持有槽的主节点都标记某个节点是主观下线时。触发客观下线流程。

这里有两个问题：

* 为什么必须是负责槽的主节点参与故障发现决策？ 因为集群模式下只有处理槽的主节点才负责读写请求和集群槽等关键信息维护， 而从节点只进行主节点数据和状态信息的复制。
* 为什么半数以上处理槽的主节点？ 必须半数以上是为了应对网络分区等原因造成的集群分割情况， 被分割的小集群因为无法完成从主观下线到客观下线这一关键过程， 从而防止小集群完成故障转移之后继续对外提供服务

客观下线流程如下：

* 首先统计有效的下线报告数量， 如果小于集群内持有槽的主节点总数的一半则退出
* 当下线报告大于槽主节点数量一半时， 标记对应故障节点为客观下线状态
* 向集群广播一条fail消息， 通知所有的节点将故障节点标记为客观下线， fail消息的消息体只包含故障节点的ID，这是客观下线的最后一步， 它承担着非常重要的职责：
  * 通知集群内所有的节点标记故障节点为客观下线状态并立刻生效。
  * 通知故障节点的从节点触发故障转移流程  

### 故障恢复

故障节点变为客观下线后， 如果下线节点是持有槽的主节点则需要在它的从节点中选出一个替换它， 从而保证集群的高可用。

下线主节点的所有从节点承担故障恢复的义务， 当从节点通过内部定时任务发现自身复制的主节点进入客观下线时， 将会触发故障恢复流程：

* **资格检查**：每个从节点都要检查最后与主节点断线时间， 判断是否有资格替换故障的主节点 

  如果从节点与主节点断线时间超过  cluster-node-time*cluster-slave-validity-factor，则当前从节点不具备故障转移资格。 参数cluster-slavevalidity-factor用于从节点的有效因子， 默认为10

* **准备选举时间**：当从节点符合故障转移资格后， 更新触发故障选举的时间， 只有到达该时间后才能执行后续流程。复制偏移量越大说明从节点延迟越低， 那么它应该具有更高的优先级来替换故障主节点，所有的从节点中复制偏移量最大的将提前触发故障选举流程

* **发起选举**：从节点到达选举时间后将发起选举流程，流程如下：

  * 更新配置纪元：配置纪元是一个只增不减的整数， 每个主节点自身维护一个配置纪元
    (clusterNode.configEpoch)标示当前主节点的版本， 所有主节点的配置纪元都不相等， 从节点会复制主节点的配置纪元。 整个集群又维护一个全局的配置纪元(clusterState.current Epoch) ， 用于记录集群内所有主节点配置纪元的最大版本.

    配置纪元的主要作用是：

    * 标示集群内每个主节点的不同版本和当前集群最大的版本
    * 每次集群发生重要事件时， 这里的重要事件指出现新的主节点(新加入的或者由从节点转换而来),从节点竞争选举。 都会递增集群全局的配置纪元并赋值给相关主节点， 用于记录这一关键事件
    * 主节点具有更大的配置纪元代表了更新的集群状态， 因此当节点间进行ping/pong消息交换时， 如出现slots等关键信息不一致时， 以配置纪元更大的一方为准， 防止过时的消息状态污染集群

  * 广播选举消息：在集群内广播选举消息（FAILOVER_AUTH_REQUEST） ， 并记录已发  送过消息的状态， 保证该从节点在一个配置纪元内只能发起一次选举。

  * 选举投票：只有持有槽的主节点才会处理故障选举消息，因为每个持有槽的节点在一个配置纪元内都有唯一的一张选票， 当接到第一个请求投票的从节点消息时回复FAILOVER_AUTH_ACK消息作为投票， 之后相同配置纪元内其他从节点的选举消息将忽略

* **替换主节点**：当从节点收集到足够的选票之后， 触发替换主节点操作:

  * 当前从节点取消复制变为主节点。
  * 执行clusterDelSlot操作撤销故障主节点负责的槽， 并执行clusterAddSlot把这些槽委派给自己。
  * 向集群广播自己的pong消息， 通知集群内所有的节点当前从节点变为主节点并接管了故障主节点的槽信息



