# Redis简介

redis是非关系型数据库。能处理集中高并发操作。数据可以存储在内存中。分布式数据库。

## Redis是单线程的

 Redis的网络请求模块和数据操作模块是单线程的

因为Redis是基于内存的操作，CPU不是Redis的瓶颈，Redis的瓶颈最有可能是机器内存的大小或者网络带宽。既然单线程容易实现，而且CPU不会成为瓶颈，那就顺理成章地采用单线程的方案了。

采用单线程，避免了不必要的上下文切换和竞争条件，也不存在多进程或者多线程导致的切换而消耗 CPU。

## Redis为什么快

### 纯内存操作

内存的访问速度是远远大于硬盘访问速度的

### 高效的数据结构

数据结构一共有6种，分别是，简单动态字符串，双向链表，压缩列表，哈希表，跳表和整数数组

### 合适的线程模型

I/O多路复用模型

单线程避免了 CPU 的上下文切换

## Redis常用应用场景

- 缓存
- 排行榜
- 计数器应用
- 
- 社交网络
- 消息队列

# Liunx的Redis操作

* 启动服务端：
    * `redis-server`前台启动，关闭命令窗口，服务器就关闭
    * `redis-server &`后台启动
    * `redis-server redis.conf &`加载配置启动
* 关闭服务器：
    * `redis-cli shtdown`
    * `kill -9 pid`

* 启动客户端：
    * `redis-cli [-h host] [-p port]`默认地址端口为127.0.0.1:6379

# 基础命令

* `ping`沟通命令，若返回PONG则表示redis服务正常运行
* `dbsize`查看当前数据库中key数目
* `select db`切换库命令
* `flushdb`删除当前库的数据
* `exit|quit`退出客户端
* `info`查看当前服务器的信息
    * all 全部Redis系统状态统计信息。
    * server	获取 server 信息
    * clients	获取 clients 信息，如客户端连接数等
    * memory	获取 server 的内存信息，包括当前内存消耗、内存使用峰值
    * persistence	获取 server 的持久化配置信息
    * stats	获取 server 的一些基本统计信息，如处理过的连接数量等
    * replication	获取 server 的主从配置信息
    * cpu	获取 server 的 CPU 使用信息
    * keyspace	获取 server 中各个 DB 的 key 的数量
    * cluster	获取集群节点信息，仅在开启集群后可见
    * commandstas	获取每种命令的统计信息

# Key命令

* `keys pattern`查找所有符合模式pattern的key
    * `*`匹配0-对个字符
    * `?`匹配单个字符
* `exists key [key...]`判断key是否存在，返回存在的key的数量
* `expire key seconds`设置key的生存时间
* `ttl key`查看key的生存时间
    * -1表示永远存在
    * -2表示不存在
* `type key`查看key的类型 none表示该key不存在
* `del key [key...]`删除指定的key，返回成功删除的key的数量



# 数据类型

## string 

字符串类型时Redis中最基本的数据类型，能存储任何形式的字符串，包括二进制数据，序列化后的数据，JSON化的对象甚至是图片

* `set key value`

    设置字符串数据的KV

    * 若key存在，则覆盖value
    * 若key不存在，创建新数据

* `get key`

    获取指定字符串的key

    * 若key存在，返回对应的value
    * 若key不存在，返回nil

* `incr`

    将key中存储的数字值加1，只能对数值类型的string操作

    * 若key不存在，可以的值会被初始化为0再执行incr操作

* `decr`

    将key中存储的数字值减1，只能对数值类型的string操作

    * 若key不存在，可以的值会被初始化为0再执行decr操作

* `append key value` 

    追加字符串，返回新值的长度

    * 如果key存在，将value追加到旧值的末尾
    * 如果key不存在，则将key设置值为value

* `strlen`

    返回字符串长度，若key不存在返回0

* `getrange key start end`

    获取key中从start开始到end结束的子字符串

    * 负数表示从字符串的末尾开始，-1表示最后一个字符
    * start表示的位置必须在end表示位置之前

* `setrange key offset value`

    用value覆盖key的存储的值，从offset开始，不存在的key做空白字符串。返回修改后的字符串的长度

* `mset key value [key value ...]`

    批量添加字符串

* `mget key [key ...]`

    批量获取字符串key对应的值

## hash

redis哈希类型hash是一个string类型的field和value的映射表

* `hset key field value`

    将哈希表key中域field的值设为value

    * 若key不存在，则创建hash表，执行赋值
    * 若field存在，则覆盖vallue

* `hget key field`

    获取哈希表key中的field的值

    * 若key不存在或者field不存在返回nil

* `hmset key field value [field value ...]`

    批量设置哈希表的field-value

    * 若key不存在，创建空的hash表，执行hmset命令
    * 设置成功返回OK否则返回错误信息

* `hmget key field [field...]`

    批量获取哈希表field的值

* `hgetall key`

    获取哈希表中所有的域和值

* `hdel key field [field ...]`

    删除哈希表key中指定的域

* `hkeys key`

    查看哈希表key中所有的field

* `hvals key`

    查看哈希表key中所有的value

* `hexists key field`

    查看哈希表key中给定域field是否存在

    * 如果存在返回1
    * 如果不存在返回0

## list

简单的字符串列表

* `lpush key value [value...]`

    从列表key表头依次插入指定value

* `rpush key value [value...]`

    从列表key队尾依次插入指定value

* `lrange key start stop`

    获取列表key指定范围的值

* `lindex key index`

    获取列表key指定下标的值

* `llen key`

    获取列表key的长度

* `lrem key count value`

    移除列表key中指定value ，count次

    * count为0 全部移除
    * count为正，从表头开始移除
    * count为负，从表未开始移除

* `lset key index value`

    设置列表key指定下标位置的值

* `linsert key before|after pivot value`

    在指定的第一个pivot元素的头或尾插入指定的值

## set

是string类型的无序集合，成员唯一



* `sadd key member [member ...]`

    将一个或者多个member元素添加到集合key中

    * 如果没有key则创建集合
    * 如果member已经存在，则不会再加入
    * 返回添加到集合的新元素的个数

* `smembers key`

    查看指定集合key的所有成员，如果key不存在则视为空集合

* `sismember key member`

    判断指定成员member是否在指定集合key中

    * 如果在，返回1
    * 如果不在，返回0

* `scard key`

    返回集合的长度

* `srem key member [member...]`

    删除集合key的指定成员

    返回成功删除的数量

* `srandmember key [count]`

    随机返回指定数量的集合key的成员，count默认为1

* `spop key [count]`

    随机弹出指定数量的集合key的成员，count默认为1

## zset

zset 的每一个元素都会关联一个分数（分数可重复，必须是数值类型）

redis通过分数来为集合中的成员进行从小到大排序



* `zadd key score member [score member...]`

    将指定的member元素和对应分数scores加入到有序集合key中

    如果key不存在，则创建有序集合

    返回新添加的元素个数

* `zrange key start stop [withscores]`

    顺序查询有序集合key，start，stop指定范围，可以指定是否显示成员分数

* `zrevrange key start stop [withscores]`

    倒序查询有序集合key

* `zrem key member[member]`

    删除指定的有序集合key成员

    返回成功删除的成员数量

* `zcard key`

    返回指定有序集合key的大小

* `zrangebyscore key min max [withscores] [limit offset count]`

    按照分数顺序查询指定范围的有序集合key的成员

    可以指定是否显示分数

    可以指定是否分页

    * min max 默认是闭区间范围
    * 可在数字前加`(`指定开区间
    * `+inf` `-inf`表示最大数和最小数

* `zrevrangebyscore key min max [withscores] [limit offset count]`

    按照分数倒序查询指定范围的有序集合key的成员

* `zcount key min max`

    查询指定分数范围内成员的数量

# Redis事务

* `multi`

    标记一个事务的开始。事务内的多条命令会按照先后顺序被放进一个队列中

* `exec`

    执行所有事务块内的命令，返回事务内所有执行语句的结果，事务被打掉返回nil

* `discard`

    取消事务

* `watch key [key...]`

    监视指定的key，若事务执行之前，监视的key有被其他命令改动， 事务将被打断

* `unwatch`

    取消监视

# 安全

* 设置密码

    去掉redis.conf 中的requirepass foobared 的注释

    访问有密码的Redis两种方式：

    * 连接到客户端后，使用命令 `auto password`
    * 连接客户端时，`redis-cli -a password`

* 绑定ip

    redis.conf 中的bind字段

* 修改端口

    redis.conf中的port字段

# Jedis

jedis是java连接redis的官方推荐包，几乎所有的方法都与redis命令一致

## 依赖

~~~xml
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
    <version>3.3.0</version>
</dependency>
~~~



## 获取对象

* 通过构造方法：

~~~java
Jedis jedis = new Jedis("192.168.134.128",6379);
~~~

* 通过pool

~~~java
JedisPoolConfig config = new JedisPoolConfig();
//设置连接池最大连接数量
config.setMaxTotal(10);
//设置连接池最大空闲连接数
config.setMaxIdle(3);
//提前检查Jedis对象，让获取的Jedis一定是可用的
config.setTestOnBorrow(true);
//创建Jedis连接池
JedisPool pool = new JedisPool(config, "192.168.134.128", 6379, 6 * 1000, "123456");
Jedis jedis = pool.getResource();
~~~

## 对象方法

### 键方法

* `flushAll() ` 清空所有数据库数据
* `flushDB()`	清空当前数据库数据
* `boolean exists(String key)`	判断某个键是否存在
* `set(String key,String value)`	新增键值对
* `Set<String> keys(String pattern)`	返回匹配的key
* `del(String key)`	删除键为key的数据项	
* `expire(String key,int i)`	设置键为key的过期时间为i秒	
* `int ttl(String key)`	获取键为key数据项的剩余时间（秒）	
* `persist(String key)`	移除键为key属性项的生存时间限制	
* `type(String key)`	查看键为key所对应value的数据类型	



### 字符串类型方法

* `set(String key,String value)`	增加（或覆盖）数据项
* `setnx(String key,String value)`	不覆盖增加数据项（重复的不插入）
* `setex(String ,int t,String value)`	增加数据项并设置有效时间
* `del(String key)`	删除键为key的数据项
* `get(String key)`	获取键为key对应的value
* `append(String key, String s)`	在key对应value 后边扩展字符串 s
* `mset(String k1,String V1,String K2,String V2,…)`	增加多个键值对
* `String[] mget(String K1,String K2,…)`	获取多个key对应的value
* `del(new String[](String K1,String K2,.... ))`	删除多个key对应的数据项
* `String getSet(String key,String value)`	获取key对应value并更新value
* `String getrange(String key , int i, int j)`	获取key对应value第i到j字符 ，从0开始，包头包尾



### 哈希类型方法

* `hmset(String key,Map map)`	添加一个Hash
* `hset(String key , String key, String value)`	向Hash中插入一个元素（K-V）
* `hgetAll(String key)`	获取Hash的所有（K-V） 元素
* `hkeys（String key）`	获取Hash所有元素的key
* `hvals(String key)	`获取Hash所有元素 的value
* `hincrBy(String key , String k, int i)`	把Hash中对应的k元素的值 val+=i
* `hdecrBy(String key,String k, int i)`	把Hash中对应的k元素的值 val-=i
* `hdel(String key , String k1, String k2,…)`	从Hash中删除一个或多个元素
* `hlen(String key)`	获取Hash中元素的个数
* `hexists(String key,String K1)`	判断Hash中是否存在K1对应的元素
* `hmget(String key,String K1,String K2)`	获取Hash中一个或多个元素value



### 列表类型方法

* `lpush(String key, String v1, String v2,....)`	添加一个List , 注意：如果已经有该List对应的key, 则按顺序在左边追加 一个或多个
* `rpush(String key , String vn)`	key对应list右边插入元素
* `lrange(String key,int i,int j)`	获取key对应list区间[i,j]的元素，注：从左边0开始，包头包尾
* `lrem(String key,int n , String val)`	删除list中 n个元素val
* `ltrim(String key,int i,int j)`	删除list区间[i,j] 之外的元素
* `lpop(String key)`	key对应list ,左弹出栈一个元素
* `rpop(String key)`	key对应list ,右弹出栈一个元素
* `lset(String key,int index,String val)	`修改key对应的list指定下标index的元素
* `llen(String key)`	获取key对应list的长度
* `lindex(String key,int index)`	获取key对应list下标为index的元素
* `sort(String key)`	把key对应list里边的元素从小到大排序 

### 集合类型方法

* `sadd(String key,String v1,String v2,…)`	添加一个set
* `smenbers(String key)`	获取key对应set的所有元素
* `srem(String key,String val)`	删除集合key中值为val的元素
* `srem(String key, Sting v1, String v2,…)`	删除值为v1, v2 , …的元素
* `spop(String key)`	随机弹出栈set里的一个元素
* `scared(String key)`	获取set元素个数
* `smove(String key1, String key2, String val)`	将元素val从集合key1中移到key2中
* `sinter(String key1, String key2)	`获取集合key1和集合key2的交集
* `sunion(String key1, String key2)`	获取集合key1和集合key2的并集
* `sdiff(String key1, String key2)`	获取集合key1和集合key2的差集

### 有序集合方法

* `zadd(String key,Map map)`	添加一个ZSet
* `hset(String key,int score , int val)`	往 ZSet插入一个元素（Score-Val）
* `zrange(String key, int i , int j)`	获取ZSet 里下表[i,j] 区间元素Val
* ` zrangeWithScore(String key,int i , int j)`	获取ZSet 里下表[i,j] 区间元素Score - Val
* `zrangeByScore(String , int i , int j)`	获取ZSet里score[i,j]分数区间的元素（Score-Val）
* `zscore(String key,String value)	`获取ZSet里value元素的Score
* `zrank(String key,String value)`	获取ZSet里value元素的score的排名
* `zrem(String key,String value)`	删除ZSet里的value元素
* `zcard(String key)`	获取ZSet的元素个数
* `zcount(String key , int i ,int j)`	获取ZSet总score在[i,j]区间的元素个数
* `zincrby(String key,int n , String value)`	把ZSet中value元素的score+=n

### 数的方法

* `incr(String key)`	将key对应的value 加1
* `incrBy(String key,int n)`	将key对应的value 加 n
* `decr(String key)`	将key对应的value 减1
* `decrBy(String key , int n)`	将key对应的value 减 n

### 事务方法

* `multi()`开启事务
* `exec()`执行事务

# 过期策略

如果设置了键的生存时间，redis会在指定时间删除过期键

有三种过期策略：

* 定时删除:在设置键的过期时间的同时，创建一个定时器，让定时器在键的过期时间来临时，立即执行对键的删除操作。
    * 优点：对内存友好，证过期的键会尽可能快地被删除，释放所占内存
    * 缺点：对cpu最不友好：生成定时器删除键会消耗cpu资源

* 惰性删除:放任键的过期不管，但是每次从键空间中获取键时，都检查取得的键是否过期，如果过期的话，就删除该键；如果没有过期，就返回该键。
    * 优点：对cpu最友好，既不需要创建大量定时器，也不需要定时扫描资源
    * 缺点：对内存最不友好，如果有大量过期的键没有被访问过，那么这些键就会一直保存在内存中，消耗大量的内存
* 定期删除:每隔一段时间，程序就对数据库进行一次检查，删除里面过期的键。至于要删除多少过期键，以及要检查多少个数据库，则有算法决定。
    * 优点：相较于以上两种方案，兼顾了cpu和内存
    * 缺点：难以确定删除操作执行的时长和频率

Redis采用的是惰性删除和定期删除两种策略：通过配好使用这两种策略，服务器可以很好地在合理使用cpu时间和避免浪费内存空间之间取得平衡。

## 惰性删除实现

过期键的惰性删除删除策略由db.c/expireIfNeeded函数实现，所有读写数据库的Redis命令在执行之前都会调用expireIfNeed函数对输入键进行检查：

- 如果键已经过期，那么expireIfNeeded函数将键删除
- 如果键未过期，那么expireIfNeeded函数不做操作

## 定期删除实现

过期键的定期删除策略由redis.c/activeExpireCycle函数实现，每当Redis的服务器周期性操作redis.c/serverCron函数执行时，activeExpireCycle函数就会被调用，它在规定时间内，分多次遍历服务器中各个数据库。

Redis 默认每秒进行 10 次过期扫描，过期扫描不会遍历过期字典中所有的 key, 而是采用了一种简单的贪心策略，步骤如下。

* 从过期字典中随机选出 20个 key。
* 删除这 20 个 key 中已经过期的 key。
* 如果过期的 key的比例超过 1/4，那就重复第一个步骤。

同时，为了保证过期扫描不会出现循环过度，导致结程卡死的现象，算法还增加了扫描时间的上限，默认不会超过 25ms。

所以如果有大量key在同一时间过期，会导致activeExpireCycle函数循环多次，导致请求卡顿或超时



# 内存淘汰机制

在配置文件redis.conf 中，可以通过参数` maxmemory <bytes>` 来设定最大内存，当数据内存达到 `maxmemory `时，便会触发redis的内存淘汰策略

内存淘汰策略可以通过maxmemory-policy进行配置，目前Redis提供了以下几种（2个LFU的策略是4.0后出现的）：

* volatile-lru，针对设置了过期时间的key，使用lru算法进行淘汰。
* allkeys-lru，针对所有key使用lru算法进行淘汰。
* volatile-lfu，针对设置了过期时间的key，使用lfu算法进行淘汰。
* allkeys-lfu，针对所有key使用lfu算法进行淘汰。
* volatile-random，从所有设置了过期时间的key中使用随机淘汰的方式进行淘汰。
* allkeys-random，针对所有的key使用随机淘汰机制进行淘汰。
* volatile-ttl，针对设置了过期时间的key，越早过期的越先被淘汰。
* noeviction，（默认策略）不会淘汰任何数据，当使用的内存空间超过 maxmemory 值时，再有写请求来时返回错误。

volatile-前缀的策略代表从设置了过期时间的key中选择键进行清除

allkeys-开头的策略代表从所有key中选择键进行清除

## LRU

Least Recently Used 最近很少使用,也可以理解成最久没有使用

也就是说当内存不够的时候，每次添加一条数据，都需要抛弃一条最久时间没有使用的旧数据

LRU 是基于链表结构实现的，链表中的元素按照操作顺序从前往后排列，最新操作的键会被移动到表头，当需要进行内存淘汰时，只需要删除链表尾部的元素即可

Redis并没有使用标准的LRU实现方法作为LRU淘汰策略的实现方式，这是因为：  

- 要实现LRU，需要将所有数据维护一个链表，这就需额外内存空间来保存链表
- 每当有新数据插入或现有数据被再次访问，都要调整链表中节点的位置，尤其是频繁的操作将会造成巨大的开销

  为了解决这一问题，Redis使用了近似的LRU策略进行了优化，平衡了时间与空间的效率。

## 近似LRU

 近似LRU在执行时，会随机抽取N个key，找出其中最久未被访问的key（通过redisObject中的lru字段计算得出），然后删除这个key。然后再判当前内存是超过限制，如仍超标则继续上述过程。

 随机抽取的个数N可以通过redis.conf的配置进行修改，默认为5。

## LFU

Least Frequently Used  最近最少使用

根据key最近被访问的频率进行淘汰，比较少访问的key优先淘汰，反之则保留

相比LRU算法,LFU增加了访问频率的这样一个维度来统计数据的热点情况

LFU主要使用了两个双向链表去形成一个二维的双向链表，一个用来保存访问频率，另一个用来访问频率相同的所有元素，其内部按照访问时间排序。

## 淘汰策略的选择

* 如果数据呈现幂等分布(即部分数据访问频率较高而其余部分访问频率较低)，建议使用 allkeys-lru或allkeys-lfu。
* 如果数据呈现平等分布(即所有数据访问概率大致相等)，建议使用 allkeys-random。
* 如果需要通过设置不同的ttls来确定数据过期的顺序，建议使用volatile-ttl。
* 如果你想让一些数据长期保存，而一些数据可以消除，建议使用volatile-lru或volatile-random。



# 持久化

将数据备份到硬盘中，防止服务器关机或者重启导致内存redis数据丢失

redis提供了两种持久化策略：RDB和AOF

AOF和RDB同时开启，系统默认取AOF的数据

## RDB

Redis Database 

redis会单独创建一个子进程(使用fork函数)来进行持久化，会先将数据写入到一个临时文件中，待持久化过程都结束了，再用这个临时文件替换上次持久化好的文件。整个过程中，主进程是不进行任何IO操作的，这就确保了极高的性能

将内存中的数据集快照写入磁盘，数据恢复时将快照文件直接读到内存中

自动RDB只需要在redis.conf文件中配置即可，默认配置时启用的：

* `save <seconds><changes>`设置生成RDB文件的时间策略，含义是，数据在指定的时间范围内变动指定次数就保存一次RDB文件
* `dbfilename`设置RDB的文件名，默认为dump.rdb
* `dir`设置RDB文件的存储位置，默认为./当前目录

也可以手动命令保存：

* `save`保存
* `bgsave`后台保存

优点：

* 由于存储的是数据快照文件，RDB文件非常适合用于进行备份和灾难恢复。
* 是由子进程来处理生成RDB文件的工作的，主进程不进行任何的IO操作，不会影响redis 的读写性能
* RDB 在恢复大数据集时的速度比 AOF 的恢复速度要快

缺点：

* 最后一次持久化后的数据可能丢失，有一定的数据安全风险
* 如果生成RDB文件的频率过高，文件过大会影响性能

## AOF

Append-only File Redis

以日志的形式来记录每个写操作，将Redis执行过的所有写指令记录下追加到AOF文件最后

当Redis重启时，它通过执行AOF文件中的所有命令来恢复数据

持久化过程：

* 客户端的请求写命令会被append追加到AOF缓冲区内
* AOF缓冲区根据AOF持久化策略将操作sync同步到磁盘的AOF文件中
* AOF文件大小超过重写策略或手动重写时，会对AOF文件rewrite重写，压缩AOF文件容量
* Redis服务重启时，会重新load加载AOF文件中的写操作达到数据恢复的目的

配置redis.conf:

* `appendonly`是否开启aof持久化，默认为no，改为yes开启

* `appendfilename`指定AOF文件名，默认文件名为appendonly.aof

* `dir`指定RDB和AOF文件存放的目录，默认是./

* `appendfsync`配置aof文件写命令数据的策略：

    * `no`不主动进行同步操作，完全交由操作系统来做(每30s一次)
    * `always`每次执行写入修改都会执行同步
    * `everysec`每秒执行一次同步操作

* `auto-aof-rewrite-min-size`允许重写的最小AOF文件大小，默认为64M

    当aof文件大于64M时，开始整理aop文件，去掉无用的操作命令(只保存对key的最后一次修改操作)

优点：

* 备份机制更稳健，丢失数据概率更低

* 可读的日志文本，通过操作AOF文件，可以处理误操作

缺点：

* 比起RDB占用更多的磁盘空间
* 恢复备份速度要慢
* 每次读写都同步的话，有一定的性能压力。

# Redis高可用

Redis 实现高可用有三种部署模式：主从模式，哨兵模式，Cluster模式

## 主从复制

为了避免单点故障，我们需要将数据复制多份部署在多台不同的服务器上

主从复制的集群模式可以实现读写分离，保证redis高可用的同时降低了主服务器的读压力

redis可以指定保存相同数据的服务器之间的主从关系：

* master主服务器负责写入数据，同时把写入的数据实时同步到从服务器

* slave从服务器只能用于读数据，无法写数据

设置主从服务器方法：

* 如果没有声明指定，默认为主服务器
* 指定从服务器可以的方法：
    * 配置文件中：设置`slaveof <master-ip> <master-port>`
    * 启动服务器时：加上属性`--slaveof <master-ip> <master-port>`
    * 客户端命令：
        * `slaveof no one`提升自己为master
        * `slaveof <master-ip> <master-port> `

### 主从同步

redis会将主服务器的数据同步到从服务器上，以保证主从服务器数据的一致性

同步方式分为全量同步和分量同步：

#### 全量同步

当主从服务器第一次连接，或者重连是不满足增量同步的条件时，会进行全量同步，将主服务器的所有数据全部同步到从服务器，具体过程如下：

* 主从服务器连接后，slave nod 发送 sync命令到 master node
* master接收到sync命令后，会异步开启一个线程开始执行bgsave命令，生成RDB文件，并且在缓冲区中记录之后所有的记录操作
* RDB文件生成后，master会向从服务器发送该文件，并在此阶段内继续在缓冲区内写操作
* salve接受到文件后会将文件保存到磁盘上，然后加载到内存中

#### 增量同步

Redis全量同步后，新的数据会以增量同步的方式同步到从服务器

增量同步是通过将缓冲器去的写操作复制到从服务器实现的

## Sentinel

sentinel哨兵提供主从复制模式的故障转移的的自动化处理

### 原理

Redis Sentinel是运行在特殊模式下的Redis服务器

有三个主要任务：

* 监控：Sentinel不断检查主服务器和从服务器是否按照预期正常工作，

    每隔固定时间sentinel会请求主服务器，如果没有收到主服务的正常响应，则报告异常

* 提醒：被监视的Redis出现问题时，Sentinel会通知管理员或其他应用程序

* 自动故障转移：监控的主Redis不能正常工作，Sentinel会开始进行故障迁移操作。

    将一个从服务器升级新的主服务器。让其他从服务器挂到新的主服务器。同时向客户端提供新的主服务器地址

Sentinel分布式系统：

* 如果只有一个Sentinel，那么Sentinel出现问题就无法监控。所以需要多个哨兵，组成Sentinel网络。一个健康的sentinel至少有三个Sentinel应用。彼此在独立的物理机器或虚拟机
* 监控同一个Master的Sentinel会自动连接，组成一个分布式的网络，互相通信并彼此交换关于被监控服务器的信息
* 当一个 Sentinel 认为被监控的服务器已经下线时，它会向网络中的其它 Sentinel 进行确认，判断该服务器是否真的已经下线 
* 如果下线的服务器为主服务器，那么 Sentinel 网络将对下线主服务器进行自动故障转移，通过将下线主服务器的某个从服务器提升为新的主服务器，并让其从服务器转移到新的主服务器下，以此来让系统重新回到正常状态 
* 下线的旧主服务器重新上线，Sentinel 会让它成为从，挂到新的主服务器下 

### 实现

配置sentinel.conf文件：

~~~python
#sentinel自己的接口:
port <port>
#sentinel要监视的master:
sentinel monitor <name> <masterIP> <masterPort> <Quorum>
# quorum 表示投票数，当有超过该数量的哨兵认为服务器已经下线，才会判断下线
~~~

启动sentinel服务器：

~~~python
redis-sentinel sentinel.conf
~~~

## Cluster

Sentinel 集群方案中只有一个主服务器，只提高了redis 的可用性，并没有提高redis 的性能，如果业务对redis性能有高要求，需要搭建多台主服务器，来提高redis性能，这就是cluster集群

为了实现高可用，最好一个集群有3个master节点，每个master节点至少有1个子节

哈希槽：

cluster集群的设计是去中性化的

它引入了哈希槽的概念，Redis集群有16384个哈希槽

在集群中的每个主节点会被分配相等个数的哈希槽

在进行set操作时，每个key会通过 CRC16 算法得出当前key对应的哈希槽，这样就能知道这个key应该去往的master节点

gossip协议：

redis集权通过gossip协议进行通讯，保证所有节点都会知道整个集群完整的信息

# 缓存问题

redis作为缓存可以显著提高数据库的性能，但也带来了许多问题

## 数据一致性

redis作为缓存时，需要保证缓存数据与数据库数据的一致性

即数据库数据和缓存数据在同一时间是一致的

针对不同程度的一致性，我们可以划分：

* 强一致性：数据库更新操作与缓存更新操作是原子性的，缓存与数据库的数据在任何时刻都是一致的，这是最难实现的一致性。
* 弱一致性：当数据更新后，缓存中的数据可能是更新前的值，也可能是更新后的值，因为这种更新是异步的。
* 最终一致性：一种特殊的弱一致性，在一定时间后，数据会达到一致的状态。最终一致性是弱一致性的理想状态，也是分布式系统的数据一致性解决方案上比较推崇的。

要保证一致性，就必须在数据库修改数据的同时，修改缓存的数据，而修改缓存可以是直接更新缓存，也可以是删除缓存数据，当下一次请求数据时，再访问数据库，将新数据加载入缓存，而修改或删除缓存的时机可以是更新数据库前，也可以是更新数据库后，这样组合后有四种方案来保证数据的一致性，

但在并发情况的某些场景下，仍然无法保证数据的一致性，我们来分析具体的情况：

* 先更新缓存，再更新数据库
    * 双写场景：两个线程同时更新同一个数据，会出现一个线程覆盖另一个线程对数据库的更新操作，造成数据库与缓存的不一致

    * 读写场景：如果读线程的缓存未命中，那么仍然要操作数据库，同样会出现更新覆盖的情况，造成数据库和缓存的不一致

* 先更新数据库，再更新缓存
    * 双写场景：会出现一个线程覆盖另一个线程对缓存数据的更新操作，曹成数据库和缓存的不一致
    * 读写场景：如果读线程的缓存未命中，那么同样会出现数据不一致的情况
* 先删除缓存，再更新数据库的
    * 双写场景：数据是一致的
    * 读写场景：在读线程缓存为命中时，有可能出现数据不一致的情况
* 先更新数据库，在删除缓存
    * 双写场景：数据是一致的
    * 读写场景：缓存未命中时，有可能数据会不一致

分析上面的结果，我们得到，更新缓存无法保证读写和双写的数据一致性，删除缓存只能保证双写的数据一致性，读写场景在缓存为命中时仍然有可能出现数据不一致的情况

以上策略都无法保证数据的一致性，所以改进了删除缓存的策略，出现了：

* 延迟双删：先删除缓存，再更新数据库，延迟一段时间后再删除缓存

该方法可以保证最终一致性

## 缓存穿透

查询了缓存和数据库中都不存在的数据，因为缓存中没有，数据库中也没有，因为数据库中没有改数据，所以不会加载到缓存中，所以所有的请求都会打向数据库。

如果有大量访问不存在的数据的请求，这些请求就会打向数据库，导致数据库崩溃

解决方案：

* 前端验证：对接口的请求参数进行校验，只有合格的请求才会被发起(仍然不安全，比如请求并不是通过前端)
* 缓存空值：当数据库返回空时，将这些空值对应的key缓存到Redis中，并设置过期时间，这样下一次相同的请求来时就不会打到数据库中(仍然不安全，比如有大量随机的请求)

* 布隆过滤器：

    * 类似哈希表，但保存的是二进制向量，并且有多个哈希函数，在时间和空间上都很有优势，常用来判断是否存在和去重

        对于一个值：

        只有对应的所有哈希值对应的二进制向量点都存在在布隆过滤器中时，该值才有可能存在(会发生误判)

        只要有一个哈希值不在布隆过滤器中，那么该值一定不在布隆过滤器中

        * 优势：占用空间极小，插入和查询速度极快
        * 缺点：误算率随着元素的增加而增加，一般情况下无法删除元素

    * 维护一个布隆过滤器，该数据结构保存数据库中所有存在的数据(Guava框架 或者 Redisson)

        当缓存未命中时，Redis会先访问布隆过滤器，只有在布隆过滤器中存在该key，才会继续访问数据库

        这样能保证所有未命中但数据库中存在的数据访问数据库，但会有很少一部分访问了数据库中没有的数据的请求访问到数据库，但这样的请求是很少的



## 缓存击穿

当某个热点数据key在Redis中过期时，有大量该key的并发请求，在某个线程首先将数据库中数据加载到Redis之前，这些请求会一起访问数据库，增加数据库压力

解决方案：

* 互斥锁：当缓存未命中时，对接下来的操作进行同步，并再查询一次缓存，若此时再未命中，才会查询数据库，这样能保证当热点key失效时，只有一个线程能访问到数据库

    该方案会降低业务系统的性能

* 设置缓存数据永不过期，(实现：在数据快过期时，使用异步线程刷新数据) 可能会存在数据不一致的情况

* 设置接口限流与熔断，降级。



## 缓存雪崩

大量热点数据同时过期，造成大量请求对不同过期的key进行访问，引起雪崩的现象

解决方案：

* 优化缓存过期时间，过期时间加上随机值
* 搭建集群，保证Redis的高可用
* 加锁同步
* 限流和降级组件
