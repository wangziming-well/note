# Redis特性

Redis是一种基于键值对的NoSQL数据库，Redis中的值可以是由string、hash、list、set、zset、Bitmaps、HyperLogLog、GEO等多种数据结构和算法组成，所以它可以满足多种应用常见；并且Redis所有数据都存放在内存中，它的读写性能很好；而且对内存中的数据做了持久化，以防止故障导致的数据丢失。除此之外，Redis还提供了键过期、发布订阅、事务、流水线、Lua脚本等附加功能。

下面是关于Redis的重要特性

## 速度快

Redis执行命令的速度非常快，官方给出的读写性能可以达到10万/秒。它速度快的因素可以归纳为以下四点：

* Redis的所有数据都放在内存中，内存的IO性能比硬盘高上好多数量级。内存的一次IO可能只需要100纳秒、而硬盘则需要一千万纳秒。
* Redis是用C语言实现的。C语言相对于其他语言更接近操作系统，执行速度相对更快
* Redis使用单线程架构，避免多线程可能产生的竞争问题
* Redis 的源代码优化很好

## 基于键值对的数据结构服务器

于很多键值对数据库不同的是，Redis中的值不仅可以是字符串，还可以是具体的数据结构，这即便于在许多应用场景的开发，同时也能提高开发效率。

Redis全称为Remote Dictionnary Server，它主要提供了5种数据结构：字符串、哈希、列表、集合、有序集合。同时在字符串的基础之上演变出了位图（Bitmaps） 和HyperLogLog两种神奇的“数据结构”， 并且随着LBS（Location Based Service， 基于位置服务） 的不断发展， Redis3.2版本中加入有关GEO（地理信息定位） 的功能  

## 丰富的功能

除了5种数据结构， Redis还提供了许多额外的功能：

* 提供了键过期功能， 可以用来实现缓存  

* 提供了发布订阅功能， 可以用来实现消息系统。
* 支持Lua脚本功能， 可以利用Lua创造出新的Redis命令。
* 提供了简单的事务功能， 能在一定程度上保证事务特性。
* 提供了流水线（Pipeline） 功能， 这样客户端能将一批命令一次性传到Redis， 减少了网络的开销  

## 简单稳定

Redis的简单主要体现在三个方面：

* Redis源码很少，早期版本的代码只有2w行左右，3.0添加了集群特性，代码增至5w行左右，相对于其他NoSQL数据库来说代码量相对少很多
* Redis使用单线程模型，使Redis服务端处理模型变得简单，而且也使客户端开发变得简单
* Redis不依赖操作系统中的类库，自己实现了事务处理相关的功能

Redis虽然简单，但是它仍然很稳定

## 客户端语言多

Redis提供了简单的TCP通信协议，很多编程语言可以很方便地接入到Redis，支持Redis的客户端语言几乎涵盖了主流的编程语言：Java、 PHP、Python、 C、 C++、 Nodejs  等。

## 其他特性

* 持久化：放在内存中的数据通常是不安全的，一旦系统发送故障，这些数据就可能丢失。因此Redis提供了两种持久化方式RDB和AOF，可以使用这两种策略将内存中的数据保存到硬盘中

* 主从复制：Redis提供了复制功能，可以实现多个相同数据的Redis副本，复制功能是分布式Redis的基础

* 高可用和分布式：Redis从2.8版本正式提供了高可用实现Redis Sentinel，它可以保证Redis节点的故障发现和故障自动转移。Redis从3.0版本正式提供了分布式实现Redis Cluster，提供了高可用，读写和容量的扩展性

# Redis使用场景

Redis的典型应用场景如下：

* 缓存：缓存机制在几乎所有大型网站都有使用，合理使用缓存可以加快数据的访问速度，并且可以有效降低后端数据源压力。Redis提供了键过期事件设置，也提供了灵活控制最大内存和内存溢出后的淘汰策略
* 排行榜系统：Redis提供了列表和有序集合数据结构，可以方便地构建排行榜系统
* 计数器应用：Redis天然支持计数器功能并且性能也非常好
* 社交网络：点赞/踩、共同好友、推送、下拉刷新是社交网络的必备功能，Redis提供的数据结构可以相对容易的实现这些功能。
* 消息队列系统：消息队列具有业务解构、削峰等特性。Redis提供了发布订阅功能和阻塞队列功能，可以满足基本的消息队列需求。
* 在分布式系统中，Redis也可以作为共享Session和分布式锁的实现。

# Redis安装

再Liunx安装Redis可以通过下面命令：

~~~bash
$ wget http://download.redis.io/releases/redis-3.0.7.tar.gz #下载Redis指定版本的源码压缩包到当前目录
$ tar -xzf redis-3.0.7.tar.gz #解压缩Redis源码压缩包
$ ln -s redis-3.0.7 redis # 建立一个redis目录的软连接，指向当前redis
$ cd redis # 进入redis
$ make # 编译(确保已经安装了gcc)
$ make install # 安装(将Redis相关运行文件放到/usr/local/bin下，这样就可以在任意目录下执行Redis的命令)
~~~

安装完成后可以在任意目录下运行Redis：

~~~bash
$ redis-cli -v
reids-cli 3.0.7
~~~

# Redis的操作和配置

Redis安装之后，`src`和`/usr/local/bin`目录下多了几个以redis开头可执行文件，我们称之为Redis Shell。下面是这些可执行文件的说明：

| 可执行文件       | 作用                              |
| ---------------- | --------------------------------- |
| redis-server     | 启动Redis                         |
| redis-cli        | Redis命令行客户端                 |
| redis-benchmark  | Redis基准测试工具                 |
| redis-check-aof  | Redis AOF持久化文件检测和修复工具 |
| redis-check-dump | Redis RDB持久化文件检测和修复工具 |
| reids-sentinel   | 启动 Redis Sentinel               |

## 启动Redis

有三种方法启动Reids：默认配置、运行配置、配置文件启动：

* 默认配置，redis将以默认配置运行：

  ~~~bash
  $ redis-server
  ~~~

* 运行配置：redis-server加上要修改配置名和键：

  ~~~bash
  $ redis-server --configKey1 configValue1 --configKey2 configValue2 ...
  ~~~

  例如：

  ~~~bash
  $ redis-server --port 6380
  ~~~

* 配置文件启动，例如我们将配置写到了`/opt/redis/reids.conf`中，那么可以通过下面命令启动redis

  ~~~bash
  $ redis-server /opt/redis/reids.conf
  ~~~

Redis目录下有一个redis.conf文件，里面是redis的默认配置，一般我们以这个文件为模板，进行修改配置。

可以在命令行后加`&`表示后台启动，如：

~~~bash
$ redis-server /opt/redis/reids.conf &
~~~

## Redis命令行客户端

redis-cli可以通过两种方式连接Redis服务器：

* 交互式方式，通过`redis-cli -h{host} -p{port}`的方式连接到Redis(其中， -h和-p参数可以省略，省略后默认为127.0.0.1和6379)，之后的所有操作都通过交互的方式实现，例如：

  ~~~bash
  $ redis-cli -h 127.0.0.1 -p 6379
  127.0.0.1:6379> set Hello World
  OK
  127.0.0.1:6379> get Hello
  "World"
  ~~~

* 命令方式，用`redis-cli -h{host} -p{port} {command}`直接得到命令的返回结果：

  ~~~bash
  $ redis-cli get Hello
  "World"
  ~~~

## 停止Redis服务

通过`redis-cli`客户端，可以通过`shutdown`命令关闭指定的redis服务端，例如：

~~~bash
$ redis-cli shutdown #将关闭 1270.0.1:6379上的redis服务端 
~~~

还可以使用`kill`进程的方式关闭Redis，但是不要使用`kill-9`，否则Redis可能不会按照正常关闭流程，进行持久化操作、释放缓冲区资源。

`shutdown`还有一个参数，代表是否在关闭Redis前，生成持久化文件:

~~~bash
redis-cli shutdown nosave|save
~~~

# Redis重大版本

Redis借鉴Liunx操作系统对版本号的命名规则：版本号第二位如果是奇数，则为非稳定版本，如果是偶数，则为稳定版本。当前奇数版本是下一个稳定版本的开发版本。生产环境通常选取偶数版本的Redis

Redis发展过程中有如下重要版本和特性：

## Redis 2.6

- **Lua脚本支持**：通过支持Lua脚本，Redis增加了服务器端脚本执行的能力，这意味着可以在服务器端执行复杂的操作，减少网络开销并保证操作的原子性。
- **新的命令**：引入了更多的命令，如`BITOP`和`SCRIPT LOAD`，增强了Redis的功能性。

## Redis 3.0

- **Redis Cluster**：引入了Redis Cluster，支持自动分片和提供高可用性，这是Redis在可伸缩性和高可用性方面迈出的重要一步。

## Redis 4.0

- **模块系统**：引入了模块系统，使得开发者可以扩展Redis的功能，比如创建新的数据类型或引入新的命令。
- **混合持久化（PSYNC2）**：优化了复制和持久化机制，允许同时使用AOF和RDB进行数据持久化，以达到更好的数据安全性和性能平衡。

## Redis 5.0

- **Stream数据类型**：新增了Stream数据类型，为消息队列和流处理场景提供了原生支持，这是一个针对时间序列数据处理的重要特性。
- **改进的集群管理**：对Redis Cluster进行了优化，提高了其管理和运维的便利性。

## Redis 6.0

- **多线程IO**：引入了多线程来处理网络IO，显著提高了Redis处理大量并发连接的能力。
- **访问控制列表（ACL）**：引入了ACL，提供了更细粒度的权限控制，增强了安全性。
- **客户端缓存支持**：通过RESP3协议支持客户端缓存，减轻了服务器的负担，提高了效率。

## Redis 7.0

- **持久化的优化**：进一步优化了AOF和RDB的持久化策略，包括增量RDB持久化，以及更灵活的AOF重写策略。
- **新的数据结构**：引入了更多的数据结构和命令，为开发者提供了更多的工具和选项。
- **集群和安全性的改进**：包括对Redis Cluster的优化和对ACL功能的增强，提高了Redis的整体安全性和稳定性。

