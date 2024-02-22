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

