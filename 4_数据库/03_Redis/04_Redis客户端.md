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
Jedis jedis = new Jedis("192.168.134.128",6379);
~~~









通过pool

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

