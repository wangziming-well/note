# 附加功能简介

除了5种数据结构，Redis还提供了很多附加功能：

* 慢查询分析： 通过慢查询分析， 找到有问题的命令进行优化。
* Redis Shell： 功能强大的Redis Shell会有意想不到的实用功能。
* Pipeline： 通过Pipeline（管道或者流水线） 机制有效提高客户端性能。
* 事务与Lua： 制作自己的专属原子命令。
* Bitmaps： 通过在字符串数据结构上使用位操作， 能有效节省内存。
* HyperLogLog： 一种基于概率的新算法， 可以节省内存空间。
* 发布订阅： 基于发布订阅模式的消息通信机制。
* GEO： Redis3.2提供了基于地理位置信息的功能。

# 慢查询分析

Redis客户端执行一条命令分为4个部分：发送命令、命令排队、命令执行、返回结果。而慢查询只统计命令执行的时间。

当命令执行的时间大于`slowlog-log-slower-than`(单位是微秒，默认值为10000)的值时，这条命令就会被记录到慢查询日志中。

Redis使用一个列表来存储慢查询日志，`slowlog-max-len`就是列表的最大长度。当这个列表已满时，一条新命令被插入时会导致最旧的命令从列表中移出。

可以通过修改配置文件来修改配置，也可以通过`config set`命令动态修改，r如果要将`config set`的修改持久化到本地文件，需要执行`config rewrite`命令，例如：

~~~bat
config set slowlog-log-slower-than 20000
config rewrite
~~~

慢查询日志由id、time、duration、command+参数四部分组成。

Redis没有暴露慢查询日志列表的键，而是提供一组命令来实现对慢查询日志的访问和管理：

| 命令              | 说明                                 |
| ----------------- | ------------------------------------ |
| `slowlog get [n]` | 获取慢查询日志 n可以指定条数         |
| `slowlog len`     | 获取慢查询日志列表的当前长度         |
| `slowlog reset`   | 重置慢查询日志，清空慢查询日志的列表 |

# Redis Shell

Redis提供了redis-cli、redis-server、redis-benchmark等Shell工具。

## redis-cli

之前已经介绍过redis-cli的 -h -p参数，接下来我们继续介绍它的重要参数

**1.-r**

-r(repeat)选项表示将命令执行多次，例如：

~~~bat
redis-cli -r 3 ping
PONG
PONG
PONG
~~~

**2.-i**

-i(interval)选项表示每隔几秒执行一次命令，必须和-r配合使用：

~~~bat
redis-cli -r 3 -i 0.5 ping
PONG
PONG
PONG
~~~

**3.-x**

-x选项表示从标准输入(stdin)读取输入作为redis-cli的最后一个参数，例如：

~~~bat
echo "world" | redis-cli -x set hello
OK
~~~

相当于`redis-cli set hello world`

**4.-c**

-c(cluster)选项是连接Redis Cluster节点时需要使用的，可以防止moved和ask异常，后续会介绍

**5.-a**

如果Redis配置了密码， 可以用-a（auth） 选项， 有了这个选项就不需要手动输入auth命令  

**6.--scan和--pattern  **

--scan选项和--pattern选项用于扫描指定模式的键， 相当于使用scan命令    

**7.--slave**

--slave选项是把当前客户端模拟成当前Redis节点的从节点， 可以用来获取当前Redis节点的更新操作  

合理的利用这个选项可以记录当前连接Redis节点的一些更新操作， 这些更新操作很可能是实际开发业务时需要的数据  

**8.--rdb**

--rdb选项会请求Redis实例生成并发送RDB持久化文件， 保存在本地。可使用它做持久化文件的定期备份。   

**9.--pipe**

--pipe选项用于将命令封装成Redis通信协议定义的数据格式， 批量发送给Redis执行， 有关Redis通信协议会在后续介绍

例如下面操作同时执行了set hello world和incr counter两条命令  

~~~bat
echo -en '*3\r\n$3\r\nSET\r\n$5\r\nhello\r\n$5\r\nworld\r\n*2\r\n$4\r\nincr\r\n$7\r\ncounter\r\n' | redis-cli --pipe
~~~

**10.--bigkeys**

--bigkeys选项使用scan命令对Redis的键进行采样，从中找到内存占用比
较大的键值  

**11.--eval**

--eval选项用于执行指定Lua脚本，后续介绍Lua脚本

**12.--latency**

latency有三个选项， 分别是--latency、 --latency-history、 --latency-dist。它们都可以检测网络延迟  

* --latency:测试客户端到目标Redis的网络延迟  
* --latency-history :以分时段的形式了解延迟信息
* --latency-dist:  使用统计图表的形式从控制台输出延迟统计信息  

**13.--stat**

-stat选项可以实时获取Redis的重要统计信息， 虽然info命令中的统计信息更全， 但是能实时看到一些增量的数据（例如requests）对运维很重要

**14.--raw和--no-raw**

--no-raw选项是要求命令的返回结果必须是原始的格式， --raw恰恰相反， 返回格式化后的结果。

例如，设置一个中文值：

~~~bat
redis-cli set hello "你好"
OK
~~~

然后正常执行get或者使用`--no-raw`选项，返回结果是二进制格式：

~~~bat
redis-cli get hello
"\xc4\xe3\xba\xc3"
~~~

而使用了`--raw`选项，则返回中文：

~~~bat
redis-cli --raw get hello
你好
~~~

## redis-server

redis-server除了启动Redis之外，还有一个`--test-memory`选项，用来检测当前操作系统是否稳定地分配指定容量的内存给Redis，这种检测可以有效避免因内存问题造成Redis崩溃，例如：

~~~bat
redis-server --test-memory 1024 #检测操作系统是否提供1G的内存给Redis
~~~

##  redis-benchmark

redis-benchmark可以为Redis做基准性能测试，它提供的选项如下

**-c**

-c(clients)选项表示客户端的并发数量，默认50

**-n\<requests>**

-n(num)表示客户端请求总量，默认100000

例如：下面命令表示100个客户端同时请求Redis，一共执行20000次：

~~~bat
redis-benchmark -c 100 -n 20000
......
====== GET ======
  20000 requests completed in 0.31 seconds
  100 parallel clients
  3 bytes payload
  keep alive: 1

25.42% <= 1 milliseconds
99.83% <= 2 milliseconds
100.00% <= 2 milliseconds
64516.13 requests per second
......
~~~

命令会输出redis的各种命令的测试结果

**-q**

-q选项表示仅仅显示redis-benchmark的requests per second信息，例如：

~~~bat
redis-benchmark -c 100 -n 20000 -q
PING_INLINE: 63291.14 requests per second
PING_BULK: 63897.77 requests per second
SET: 60240.96 requests per second
......
~~~

**-r**

使用redis-benchmark只会生成下面三个键：

~~~bat
1) "mylist"
2) "counter:__rand_int__"
3) "key:__rand_int__"
~~~

可以使用-r(random)生成更多的键

~~~bat
redis-benchmark -c100 -n 2000 -r 10000
~~~

-r选项会在key、counter键上加一个12位的后缀， -r10000代表只对后四位做随机处理(不表示随机数的个数)

**-P**

表示每个请求pipeline的数据量，默认为1

**-k\<boolean>**

代表客户端是否使用keepalive，1为使用，0为不使用；默认为1

**-t**

可以对指定命令进行基准测试

~~~bat
redis-benchmark -t get,set -q
SET: 62034.74 requests per second
GET: 65231.57 requests per second
~~~

**-csv**

表示将结果按照csv格式输出，便于后续处理，如导入到Excel等

~~~bat
redis-benchmark -t get,set --csv
"SET","64143.68"
"GET","65616.80"
~~~

# Pipeline

Redis客户端执行一条命令分为如下四个过程：

* 发送命令
* 命令排队
* 命令执行
* 返回结果

其中发送命令和返回结果的总时间称为 Round Trip Time(RTT,往返时间)

Redis提供了批量操作命令，如mget、mset，有效地减少RTT。但是大部分命令是不支持批量操作的。先对于Redis命令排队和执行，网络可以成为Redis的性能瓶颈。

Pipeline(流水线)机制能改善上述问题，它将一组Redis命令进行组装，通过一次RTT传输给Redis，再将这组Redis命令的执行结果按顺序返回给客户端。

客户端和服务端网络延时越大，Pipeline的效果越明显

redis-cli的--pipe选项实际上就是使用Pipeline机制，下面操作将set hello world 和incr counter 两条命令组装

~~~bat
echo -en '*3\r\n$3\r\nSET\r\n$5\r\nhello\r\n$5\r\nworld\r\n*2\r\n$4\r\nincr\r\n$7\r\ncounter\r\n' | redis-cli --pipe
~~~

生产中我们一般在高级语言客户端中使用Pipeline

## 和原生批量命令相比

可以使用Pipeline模拟出批量操作的效果，它们两者有如下区别：

* 原生批量命令是原子的，而Pipeline不是原子的
* 原生批量命令是一个命令对应对各key，Pipeline支持多个命令
* 原生批量命令是Redis服务端支持实现的，而Pipeline需要服务端和客户端共同实现。



# 事务

Redis提供了简单的事务功能，将一组需要一起执行的命令放到multi和exec两个命令之间。multi表示事务开始，exec表示事务结束。它们之间的命令是原子顺序执行的。

 例如：

~~~bat
127.0.0.1:6379> multi
OK
127.0.0.1:6379> sadd user:a:follow user:b
QUEUED
127.0.0.1:6379> sadd user:b:fans user:a
QUEUED
127.0.0.1:6379> exec
1) (integer) 1
2) (integer) 1
~~~

可以看到，在事务中的命令结果返回QUEUE，表示命令没有真正执行，而是暂存在Redis中，只有当exec执行后，事务中的所有命令才算完成exec的返回结果是事务中命令的所有结果

如果需要取消事务，可以使用`discard`命令代替`exec`命令即可

## 错误处理

关于事务中的错误处理：

* 命令错误，如果事务中的命令有语法错误，那么exec将失败，整个事务无法执行
* 运行时错误，如果没有命令错误，而是执行逻辑有误，事务将正确执行，不支持回滚。开发人员需要自己修复这类问题。

在事务运行中，一般需要保证事务访问的键不被其他客户端修改，Redis提供了watch命令，以提供类似乐观锁的机制来解决这个问题

## 监控key

在事务开启前，可以用`watch key`命令监视key，在事务提交前，如果监控的key被其他事务/客户端修改了，那么事务提交将失败。

`unwatch key`用于取消监视。

# Redis中的Lua脚本

在Redis中执行Lua脚本由两种方法eval和evalsha。

Lua脚本为Redis开发带来下面好处：

* Lua脚本在Redis中是原子执行的，执行过程中不会插入其他命令
* Lua脚本可以帮助用户定制复合命令，并可以将这些命令常驻在Redis内存中，实现复用效果
* Lua脚本可以将多条命令一次性打包，有效地减少网络开销

## eval

~~~bat
eval 脚本内容 key个数 key列表 参数列表
~~~

例如：

~~~bat
127.0.0.1:6379> eval 'return "hello " .. KEYS[1] .. ARGV[1]' 1 redis world
"hello redisworld"
~~~

此时 KEYS[1] = "redis",ARGV[1] ="world"，所以最终返回结果为 "hello redisworld"

如果Lua脚本较长，可以使用`redis-cli --eval`直接执行文件，eval命令和--eval参数本质一样，都是将脚本作为字符串发送给服务端，服务端执行脚本，将结果返回给客户端。

格式如下：

~~~bat
redis-cli –eval [lua 脚本] [key…]空格,空格[args…]
~~~

例如：

~~~bat
redis-cli --eval ./test.lua  redis , world
"hello redisworld"
~~~

## evalsha

evalsha同样用来执行Lua脚本。

与eval不同的是，需要通过`script load`命令首先将Lua脚本加载到Redis服务端，得到该脚本的SHA1校验和，

然后再evalsha命令使用SHA1作为参数可以直接执行Lua脚本。

这在需要多次执行同一个脚本的情况下可以避免每次发送Lua脚本的开销。让脚本得到复用。

例如：

在Window系统下使用script load命令将脚本内容加载到Redis内存中：

~~~bat
D:\Data\Temporary> redis-cli -x script load < ./test.lua
"7413dc2440db1fea7c0a0bde841fa68eefaf149c"
~~~

然后开打客户端使用这串SHA1加载调用lua脚本：

~~~bat
127.0.0.1:6379> evalsha 7413dc2440db1fea7c0a0bde841fa68eefaf149c 1 redis world
"hello redisworld"
~~~

如果使用liunx系统，可以使用下面命令加载脚本：

~~~bat
redis-cli script load "$(cat test.lua)"
~~~

或者使用`pipe`：

~~~bat
cat test.lua | redis-cli script load --pipe
~~~

## Lua的Redis API

Lua可以使用`redis.call`函数实现对Redis的访问，例如下面代码使用redis.call调用了Redis的set和get：

~~~lua
redis.call("set","hello","world")
redis.call("get","hello")
~~~

用Redis调用的执行效果如下：

~~~bat
127.0.0.1:6379> eval 'return redis.call("get",KEYS[1])' 1 hello
"world"
~~~

也可以使用`redis.pcall()`函数实现对Redis的调用，`redis.call`与`redis.pcall`的不同之处在于，如果`redis.call`执行失败，那么脚本执行结束会直接返回错误，而`redis.pcall`会忽略错误继续执行脚本。

`redis.log`可以向Redis日志文件输出。

## Redis管理Lua脚本

Redis提供了4个命令实现对Lua脚本的管理：

**script load**

~~~bat
script load scriptStr
~~~

用于将Lua脚本加载到Redis内存中，并且返回对应的sha1

**script exists**

~~~bat
scripts exists sha1 [sha1 ...]
~~~

用于判断sha1是否已经加载到Redis内存中

返回结果代表指定的sha1中被加载到Redis内存中的个数

**script flush**

~~~bat
script flush
~~~

用于清除Redis内存中所有已经加载的Lua脚本

**script kill**

~~~bat
script kill
~~~

用于杀掉正在执行的Lua脚本。如果Lua脚本耗时很多，就会阻塞Redis，可以用这个命令强制杀死运行的脚本。

# Bitmaps



