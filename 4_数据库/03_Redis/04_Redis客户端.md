# Jedis

jedis是java连接redis的官方推荐包，几乎所有的方法都与redis命令一致

## 获取Jedis

可以通过maven、gradle等将Jedis目标版本配置加入到项目：

~~~xml
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
    <version>3.3.0</version>
</dependency>
~~~

## 基本使用方法

可以使用下面代码使用Jedis：

~~~java
Jedis jedis = new Jedis("127.0.0.1", 6379);
jedis.set("hello", "world");
String result = jedis.get("hello");
~~~

初始化Jedis需要两个参数，Redis实例的IP和端口，除了这两个参数外，还有一个包含4个参数的构造函数：

~~~java
public Jedis(final String host, final int port, final int connectionTimeout, final int soTimeout)
~~~

参数说明：

* host：Redis实例的所在机器的IP
* port：Redis实例的端口
* connectionTimeout：客户端连接超时
* soTimeout：客户端读写超时  

在实际使用时推荐使用`try catch finally`的形式来使用jedis资源，在使用完毕时无论是否异常都应该在finally代码块中关闭客户端释放资源。

~~~java
Jedis jedis = null;
try {
    jedis = new Jedis("127.0.0.1", 6379);
    jedis.get("hello");
} catch (Exception e) {
    System.err.println(e.getMessage());
    e.printStackTrace();
} finally {
    if (jedis != null) {
        jedis.close();
    }
}
~~~

## Jedis连接池

上面介绍的是Jedis的直连方式， 所谓直连是指Jedis每次都会新建TCP连接，使用后再断开连接，对于频繁访问Redis的场景显然不是高效的使用方式。会重复建立TCP网络连接。

因此生产环境中一般使用连接池的方式对Jedis进行管理，所有Jedis对象预先放在JedisPool中，每次要连接Redis就从池子中申请，使用完后再归还给池子。

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
//获取Jedis对象
Jedis jedis = pool.getResource();
....
//用完后释放对象,当使用JedisPool时，close()方法将不是关闭连接，而是代表归还连接池。
jedis.close();
~~~

其中JedisPoolConfig对连接池的行为进行配置，它是`GenericObjectPoolConfig`的子类，其重要属性如下：

![GenericObjectPoolConfig重要属性](https://gitee.com/wangziming707/note-pic/raw/master/img/GenericObjectPoolConfig%E9%87%8D%E8%A6%81%E5%B1%9E%E6%80%A7.png)

## Jedis使用Pipeline

之前介绍过了Pipeline的基本原理，Jedis就支持Pipeline特性 ，下面方法使用pipeline执行批量删除操作：

~~~java
public void mdel(List<String> keys) {
    Jedis jedis = new Jedis("127.0.0.1");
    //生成pipeline对象
    Pipeline pipeline = jedis.pipelined();
    //pipeline执行命令， 注意此时命令并未真正执行
    for (String key : keys) {
        pipeline.del(key);
    }// 执行命令
    pipeline.sync();
}
~~~

除了`pipeline.sync()` ， 还可以使用`pipeline.syncAndReturnAll() `将pipeline的命令进行返回  

## Jedis的Lua脚本

Jedis中执行Lua脚本和redis-cli十分类似，提供接口方法如下：

~~~java
public interface ScriptingCommands {
  Object eval(String script, int keyCount, String... params);
  Object eval(String script, List<String> keys, List<String> args);
  Object eval(String script);
  Object evalsha(String sha1);
  Object evalsha(String sha1, List<String> keys, List<String> args);
  Object evalsha(String sha1, int keyCount, String... params);
  Boolean scriptExists(String sha1);
  List<Boolean> scriptExists(String... sha1);
  String scriptLoad(String script);
}
~~~

eval函数参数说明：

* script： Lua脚本内容。
* keyCount： 键的个数。
* params： 相关参数KEYS和ARGV。  

scriptLoad将脚本加载到Redis中，获取对应的sha1值，然后使用evalsha函数传入对应的sha1值调用脚本。

# 客户端管理

Redis提供了客户端相关API对其状态进行监控和管理

## 客户端API

### client list

client list命令能列出与Redis服务端相连的所有客户端连接信息，例如：

~~~bat 
127.0.0.1:6379> client list
id=6247 addr=127.0.0.1:63472 fd=35 name= age=411 idle=35 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=0 qbuf-free=0 obl=0 oll=0 omem=0 events=r cmd=client
id=6248 addr=127.0.0.1:63223 fd=88 name= age=6 idle=0 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=0 qbuf-free=32768 obl=0 oll=0 omem=0 events=r cmd=client
~~~

输出结果的每一行代表一个客户端的信息， 可以看到每行包含了十几个属性， 它们是每个客户端的一些执行状态 ，下面选择几个重要的属性进行说明：

#### 客户端标识

* id： 客户端连接的唯一标识， 这个id是随着Redis的连接自增的， 重启Redis后会重置为0
* addr： 客户端连接的ip和端口
* fd： socket的文件描述符， 与lsof命令结果中的fd是同一个， 如果fd=-1代表当前客户端不是外部客户端， 而是Redis内部的伪装客户端  
* name： 客户端的名字，用client setName命令设置客户端的名字

#### 输入缓冲区

Redis为每个客户端分配了输入缓冲区，它的作用是将客户端发送的命令临时保存，同时Redis会从输入缓冲区拉去命令并执行，输入缓冲区为客户端发送命令到Redis执行命令提供了缓冲功能 。输入缓冲区会根据输入内容大小的不同动态调整， 只是要求每个客户端缓冲区的大小不能超过1G， 超过后客户端将被关闭，输入缓冲使用不当会产生两个问题：

* 一旦某个客户端的输入缓冲区超过1G， 客户端将会被关闭  
* 输入缓冲区不受maxmemory控制，如果多个客户端的输入缓冲区很大，可能挤占Redis内存中的其他数据，如会产生数据丢失、 键值淘汰、 OOM

输入缓冲区过大的原因可能有：

* Redis的处理速度跟不上输入缓冲区的输入速度， 并且每次进入输入缓冲区的命令包含了大量bigkey
* Redis发生了阻塞， 短期内不能处理命令， 造成客户端输入的命令积压在了输入缓冲区

可以通过下面方法监控输入缓冲区的异常：

* 通过定期执行client list命令， 收集qbuf和qbuf-free找到异常的连接记录  
* 通过info命令的info clients模块， 找到最大的输入缓冲区  

qbuf和qbuf-free分别代表这个缓冲区的总容量和剩余容量

#### 输出缓冲区

Redis为每个客户端分配了输出缓冲区， 它的作用是保存命令执行的结果返回给客户端

按照客户端输出缓冲区分为三种：普通客户端、 发布订阅客户端、 slave客户端    

输出缓冲区的容量可以通过参数client-output-buffer-limit  进行设置，其配置规则如下：

~~~config
client-output-buffer-limit <class> <hard limit> <soft limit> <soft seconds>
~~~

* `<class>`： 客户端类型， 分为三种。
  * normal：普通客户端
  * slave：slave客户端， 用于复制
  * pubsub：发布订阅客户端 

* `<hard limit>`： 如果客户端使用的输出缓冲区大于`<hard limit>`， 客户端会被立即关闭  

* `<soft limit>`和`<soft seconds>`： 如果客户端使用的输出缓冲区超过了`<soft
  limit>`并且持续了`<soft limit>`秒， 客户端会被立即关闭  

Redis的默认配置是： 

~~~config
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit slave 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60
~~~

和输入缓冲区相同的是， 输出缓冲区也不会受到maxmemory的限制， 如果使用不当同样会造成maxmemory用满产生的数据丢失、 键值淘汰、 OOM等情况  

实际上输出缓冲区由两部分组成： 固定缓冲区（16KB） 和动态缓冲区， 其中固定缓冲区返回比较小的执行结果， 而动态缓冲区返回比较大的结果， 例如大的字符串、hgetall、 smembers命令的结果等  

固定缓冲区使用的是字节数组， 动态缓冲区使用的是列表。 当固定缓冲区存满后会将Redis新的返回结果存放在动态缓冲区的队列中， 队列中的每个对象就是每个返回结果  

client list中的obl代表固定缓冲区的长度， oll代表动态缓冲区列表的长度， omem代表使用的字节数

监控输出缓冲区的方法依然有两种 :

* 通过定期执行client list命令， 收集obl、 oll、 omem找到异常的连接记录并分析， 最终找到可能出问题的客户端。
* 通过info命令的info clients模块， 找到输出缓冲区列表最大对象数

预防输出缓冲区过大的方法如下：

* 监控输出缓冲区大小
* 限制普通客户端输出缓冲区的大小
* 适当增大slave的输出缓冲区的大小

* 限制容易让输出缓冲区增大的命令， 例如monitor命令  

* 及时监控内存， 一旦发现内存抖动频繁， 可能就是输出缓冲区过大  

#### 客户端存活状态

client list中的age和idle分别代表当前客户端已经连接的时间和最近一次的空闲时间  

#### 客户端限制

Redis提供了maxclients参数来限制最大客户端连接数，一旦连接数超过maxclients， 新的连接将被拒绝  

maxclients默认值是10000， 可以通过infoclients来查询当前Redis的连接数  

可以通过config set maxclients对最大客户端连接数进行动态设置：

Redis提供了timeout（ 单位为秒） 参数来限制连接的最大空闲时间， 一旦客户端连接的idle时间超过了timeout， 连接将会被关闭  

Redis的默认配置给出的timeout=0 ,这是基于对客户端开发的一种保护。在实际开发和运维中， 需要将timeout设置成大于0  

#### 客户端类型

client list中的flag是用于标识当前客户端的类型，其可能值如下表：

| 客户端类型 | 说明                                                |
| ---------- | --------------------------------------------------- |
| N          | 普通客户端                                          |
| M          | 当前客户端是master节点                              |
| S          | 当前客户端是slave节点                               |
| O          | 当前客户端正在执行monitor命令                       |
| x          | 当前客户端正在执行事务                              |
| b          | 当前客户端正在等待阻塞事件                          |
| i          | 当前客户端正在等待VMI/O，但是此状态目前已经废弃不用 |
| d          | 个受监视的键已被修改，EXEC 命令将失败               |
| u          | 客户端未被阻塞                                      |
| c          | 回复完整输出后，关闭连接                            |
| A          | 尽可能快地关闭连接                                  |

### client setName/getName

~~~bat
client setName xx
client getName
~~~

client setName用于给当前客户端设置名字，样比较容易标识出客户端的来源 

使用client getName命令 ，直接查看当前客户端的name  

在Redis只有一个应用方使用的情况下， IP和端口作为标识会更加清晰。 

当多个应用方共同使用一个Redis， 那么此时client setName可以作为标识客户端的一个依据  

### client kill

~~~bat
client kill ip:port
~~~

用于杀死指定IP地址和端口的客户端

由于一些原因（例如设置timeout=0时产生的长时间idle的客户端） ， 需要手动杀掉客户端连接时， 可以使用client kill命令  

### client pause  

~~~bat
client pause timeout(毫秒)
~~~

该命令用于阻塞客户端timeout毫秒数， 在此期间客户端连接将被阻塞  

该命令在如下场景起到作用：

* client pause只对普通和发布订阅客户端有效， 对于主从复制（从节点内部伪装了一个客户端） 是无效的， 也就是此期间主从复制是正常进行的，所以此命令可以用来让主从复制保持一致  
* client pause可以用一种可控的方式将客户端连接从一个Redis节点切换到另一个Redis节点  

注意在生产环境中， 暂停客户端成本非常高  

### monitor

monitor命令用于监控Redis正在执行的命令，并记录了详细的时间戳   

monitor能监听到所有的命令， 一旦Redis的并发量过大，monitor客户端的输出缓冲会暴涨， 可能瞬间会占用大量内存  

