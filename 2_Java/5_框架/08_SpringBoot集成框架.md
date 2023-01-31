# Springboot集成MyBatis

mybatis对spring进行了集成

## 依赖

~~~xml
<!--MyBatis整合SpringBoot的起步依赖-->
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>2.0.0</version>
</dependency>
<!--MySQL的驱动依赖-->
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
</dependency>
~~~

## 配置

springboot配置

~~~properties
#配置数据库的连接信息
#注意这里的驱动类有变化
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
spring.datasource.url=jdbc:mysql://localhost:3306/springboot?useUnicode=true&characterEncoding=utf8&useSSL=false
spring.datasource.username=root
spring.datasource.password=root
#如果mapper.xml没有与接口放到一起放在mapper包下，而是放在resources下的mapper包中，那么需要在springboot配置中配置：
mybatis.mapper-locations=classpath:mapper/*.xml
~~~

maven配置资源扫描

~~~xml
<resources>
    <resource>
        <directory>src/main/java</directory>
        <includes>
            <include>**/*.xml</include>
        </includes>
    </resource>
</resources>
~~~



## 注解

* `@Mapper`标记当前类为mapper类

* `@MapperScan("com.bjpn.springboot.mapper")`扫描mapper包的注解

    需要再运行主类Application上添加



## 事务

* 在入口类中使用注解 @EnableTransactionManagement 开启事务支持
* 在需要事务支持的Service方法上添加注解 @Transactional 即可

@Transacitonal注解可以设置的参数：

* propagation

    事务传播行为

    * REQUIRED：支持当前事务，如果当前没有事务，就新建一个事务。这是最常见的选择。 
    * SUPPORTS：支持当前事务，如果当前没有事务，就以非事务方式执行。 
    * MANDATORY：支持当前事务，如果当前没有事务，就抛出异常。 
    * REQUIRES_NEW：新建事务，如果当前存在事务，把当前事务挂起。 
    * NOT_SUPPORTED：以非事务方式执行操作，如果当前存在事务，就把当前事务挂起。 
    * NEVER：以非事务方式执行，如果当前存在事务，则抛出异常。 
    * NESTED：支持当前事务，如果当前事务存在，则执行一个嵌套事务，如果当前没有事务，就新建一个事务。 

* timeout 超时时间，默认30秒

* isolation

    事务隔离级别

    MYSQL: 默认为REPEATABLE_READ级别
    SQLSERVER: 默认为READ_COMMITTED

    * Isolation.READ_UNCOMMITTED  : 读取未提交数据(会出现脏读, 不可重复读) 基本不使用

    * Isolation.READ_COMMITTED  : 读取已提交数据(会出现不可重复读和幻读)

    * Isolation.REPEATABLE_READ：可重复读(会出现幻读)
        Isolation.SERIALIZABLE：串行化

* readOnly 

    属性用于设置当前事务是否为只读事务，设置为true表示只读，false则表示可读写，默认值为false。

* rollbackFor

* 该属性用于设置需要进行回滚的异常类数组，当方法中抛出指定异常数组中的异常时，则进行事务回滚

* rollbackForClassName

    该属性用于设置需要进行回滚的异常类名称数组，当方法中抛出指定异常名称数组中的异常时，则进行事务回滚

* noRollbackForClassName

    该属性用于设置不需要进行回滚的异常类名称数组，当方法中抛出指定异常名称数组中的异常时，不进行事务回滚

* noRollbackFor

    该属性用于设置不需要进行回滚的异常类数组，当方法中抛出指定异常数组中的异常时，不进行事务回滚

**注意：**以上四个rollbackFor参数，既可以指定单个参数，也可以用数组指定多个参数,例如：

指定单一异常类：@Transactional(noRollbackFor=RuntimeException.class)

指定多个异常类：@Transactional(noRollbackFor={RuntimeException.class, Exception.class})

### 手动回滚

在有`@Transacitonal`注解的方法下，程序员可以根据业务需求手动设置回滚点和进行回滚

springboot提供了`TransactionStatus`接口通过切面编程，提供事务相关的操作：

~~~java
public interface TransactionStatus extends TransactionExecution, SavepointManager, Flushable {
    //判断是否有回滚点
    boolean hasSavepoint();
	
    void flush();
}
~~~

它继承的关于事务的接口：`TransactionExecution, SavepointManager`

它们的代码如下：

~~~java
public interface TransactionExecution {
    //是否是一个新的事务
    boolean isNewTransaction();

    //将一个事务标识为不可提交的。在调用完setRollbackOnly()后只能被回滚
	//在大多数情况下，事务管理器会检测到这一点，在它发现事务要提交时会立刻结束事务。
	//调用完setRollbackOnly()后，数数据库可以继续执行select，但不允许执行update语句，因为事务只可以进行读取操作，任何修改都不会被提交。
    void setRollbackOnly();
	//判断事务是否是只可回滚的
    boolean isRollbackOnly();

    //判断事务是否已经完成
    boolean isCompleted();
}

public interface SavepointManager {
    //1.创建回滚点
    Object createSavepoint() throws TransactionException;

    //2.回滚到回滚点 
    void rollbackToSavepoint(Object savepoint) throws TransactionException;

    //3.释放回滚点
    void releaseSavepoint(Object savepoint) throws TransactionException;
}

public interface Flushable {

    void flush() throws IOException;
}
~~~

获取`TransactionStatus`

~~~java
TransactionStatus transactionStatus = TransactionAspectSupport.currentTransactionStatus();
~~~



# Springboot下的springMVC

Spring Boot下的Spring MVC和之前的Spring MVC使用是完全一样的，这里主要介绍spring支持的处理器注解

* `@Controller`基本的处理器注解

* `@RestController`

    是@Controller与@ResponseBody的组合注解
    	如果一个Controller类添加了@RestController，那么该Controller类下的所有方法都相当于添加了@ResponseBody注解

* `RequestMapping`设置资源路径支持Get请求，也支持Post请求

* `@GetMapping`

    RequestMapping和Get请求方法的组合
    只支持Get请求
    Get请求主要用于查询操作

    下面的注解类似

* `@PostMapping`Post请求主要用户新增数据

* `@PutMapping`Put通常用于修改数据

* `@DeleteMapping`通常用于删除数据

## 使用拦截器

通过java配置的方式注册拦截器：

springboot提供了WebMvcConfigurer接口，约束注册springMVC组件的规范

实例:

~~~java
@Configuration
public class InterceptorConfig implements WebMvcConfigurer {

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        //定义需要拦截的路径
        String[] addPathPatterns = {"/**"};
        //定义不需要拦截的路径
        String[] excludePathPatterns = {"/login", "/register"};
        registry.addInterceptor(new MyInterceptor())
                .addPathPatterns(addPathPatterns)
                .excludePathPatterns(excludePathPatterns);
    }
}
~~~

**注意：**拦截器只拦截经过前端处理器的请求

## 配置字符编码

~~~properties
#设置请求响应的字符编码
server.servlet.encoding.enabled=true
server.servlet.encoding.force=true
server.servlet.encoding.charset=UTF-8
~~~

## 配置文件虚拟路径映射

文件配置：

~~~properties
spring.mvc.static-path-pattern=/**
spring.web.resources.static-locations=classpath:/META-INF/resources/, classpath:/resources/, classpath:/static/, classpath:/public/,file:D:\\Data\\upload
~~~

注解配置：

~~~java
@Configuration
public class MvcConfig implements WebMvcConfigurer {
    //设置文件虚拟路径映射
    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/test/**").addResourceLocations("file:H:\\AFile\\");
    }
}
~~~

# Springboot实现RESTful

一种互联网软件架构设计的风格，提出了一组客户端和服务器交互时的架构理念和设计原则：

访问资源：请求资源，然后按照请求的方式进行处理，如果说get方式，查询操作，如果put 更新操作，如果是delete方式 删除资源，如果是post方式 添加资源

## 实现

Springboot为rest风格的编程提供了一下注解：

* `@PathVariable`获取url中的路径变量

使用之前介绍的`@PostMapping,@GetMapping,@DeleteMapping,@PutMapping`

并在路径中设置路径变量,通过中括号：

`@PostMapping("/springBoot/student/{name}/{age}")`

案例：

~~~java
@PostMapping(value = "/springBoot/student/{name}/{age}")
public Object addStudent(@PathVariable("name") String name,
                             @PathVariable("age") Integer age) {
        Map<String,Object> retMap = new HashMap<String, Object>();
        retMap.put("name",name);
        retMap.put("age",age);
        return retMap;
    }
~~~

## RESTful原则

* 增post请求、删delete请求、改put请求、查get请求
* 请求路径不要出现动词
* 分页、排序等操作，不需要使用斜杠传参数

# Springboot集成Redis

## 依赖

~~~xml
<!-- 加载spring boot redis包 -->
<dependency>
   <groupId>org.springframework.boot</groupId>
   <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
~~~

## 配置

~~~yaml
spring:
	redis:
		host: 192.168.92.134
		port: 6379
		password: 123456
~~~

## RedisTemplate

springboot提供了RedisTemplate类，针对redis的操作进行了薄层封装

### 键

RedisTemplate针对redis的key操作进行封装：

* `void rename(K oldKey, K newKey)`

    修改 redis 中 key 的名称

* `Boolean renameIfAbsent(K oldKey, K newKey)`

    如果旧值 key 存在时，将旧值改为新值

* `Boolean hasKey(K key)`

    判断是否有 key 所对应的值，有则返回 true，没有则返回 false

* `Boolean delete(K key)`

    删除指定的key

* `Long delete(Collection<K> keys)`

    批量删除 key

* `Boolean expire(K key, long timeout, TimeUnit unit)`
    设置过期时间

* `Boolean expireAt(K key, Date date)`

    设置过期时间

* `Long getExpire(K key)`

    返回当前 key 所对应的剩余过期时间

* `Long getExpire(K key, TimeUnit timeUnit)`

    返回剩余过期时间并且指定时间单位

* `Set<K>	keys(K pattern)`

    查找匹配的 key 值，返回一个 Set 集合类型

* `Boolean persist(K key)`

    将 key 持久化保存

* `Boolean move(K key, int dbIndex)`

    将当前数据库的 key 移动到指定 redis 中数据库当中

### 值

RedisTemplate针对redis值的五种数据类型，提供不同的封装类：

非绑定key操作：

* `ValueOperations<K, V> opsForValue()`
* `<HK, HV> HashOperations<K, HK, HV> opsForHash()`
* `ListOperations<K, V> opsForList()`
* `SetOperations<K, V> opsForSet()`
* `ZSetOperations<K, V> opsForZSet()`

绑定key操作：

* `BoundValueOperations<K, V> boundValueOps(K key)`
* `<HK, HV> BoundHashOperations<K, HK, HV> boundHashOps(K key)`
* `BoundListOperations<K, V> boundListOps(K key)`
* `BoundSetOperations<K, V> boundSetOps(K key)`
* `BoundZSetOperations<K, V> boundZSetOps(K key)`

Operations的方法与Jedis类似

### RedisTemplate泛型

springboot提供的redisTemplate自动配置类如下：

~~~java
package org.springframework.boot.autoconfigure.data.redis;

@AutoConfiguration
@ConditionalOnClass({RedisOperations.class})
@EnableConfigurationProperties({RedisProperties.class})
@Import({LettuceConnectionConfiguration.class, JedisConnectionConfiguration.class})
public class RedisAutoConfiguration {
    public RedisAutoConfiguration() {
    }

    @Bean
    @ConditionalOnMissingBean(
        name = {"redisTemplate"}
    )
    @ConditionalOnSingleCandidate(RedisConnectionFactory.class)
    public RedisTemplate<Object, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory) {
        RedisTemplate<Object, Object> template = new RedisTemplate();
        template.setConnectionFactory(redisConnectionFactory);
        return template;
    }

    @Bean
    @ConditionalOnMissingBean
    @ConditionalOnSingleCandidate(RedisConnectionFactory.class)
    public StringRedisTemplate stringRedisTemplate(RedisConnectionFactory redisConnectionFactory) {
        return new StringRedisTemplate(redisConnectionFactory);
    }
}
~~~

可以看到按照默认配置，spring容器中只有两个redisTemplate实例： 

`RedisTemplate<Object, Object>`和`StringRedisTemplate`(也就是`RedisTemplate<String, String>` )

所以在使用redisTemplate时，无法自定义泛型，想要自定义泛型的redisTemplate，必须自己配置

### RedisTemplate序列化

redisTemplate需要将Java对象 K,V 传递给redis，就需要将K,V序列化

springboot提供了许多序列化方案：

* `JdkSerializationRedisSerializer    `使用 jdk 序列化机制，将对象通过 `ObjectInputStream/ObjectOutputStream` 方法进行序列化操作，最终 Redis 中将存储字节序列，是默认的序列化策略。限制：被序列化 Class 需要实现 Serializable 接口，而且 Redis 数据库中数据很不直观，为16进制或乱码。

* `StringRedisSerializer    `在 key 或 value 为字符串的场景中，根据指定的字符集对数据的字节序列编码成 String，是`new String(bytes, charset)`和`string.getbytes(charset)`的直接封装，是最轻量级和高效的策略。

* `Jackson2JsonRedisSerializer    `使用 Jackson 和 Jackson Databind ObjectMapper 读取和写入 JSON 的序列化策略。此序列化器可用于绑定到类型化的 Bean 或非类型化的 HashMap 实例。注意：空对象被序列化为空数组，反之亦然。

* `GenericJackson2JsonRedisSerializer    `使用 Jackson 实现的序列化器。使用 GenericJackson2JsonRedisSerializer 序列化时，会保存序列化的对象的包名和类名，反序列化时以这个作为标示就可以反序列化成指定的对象。

* `GenericToStringSerializer    `将通用字符串转换为 byte[] (和反向) 序列化程序。依赖于 Spring ConversionService 将对象转换为字符串，反之亦然。使用指定的字符集 (默认情况下为 UTF-8) 将字符串转换为字节，反之亦然。注意：如果将类定义为 Spring Bean，则转换服务初始化会自动发生。不以任何特殊方式处理空值，将所有内容委托给容器。

* `OxmSerializer    `提供了将 JavaBean 与 XML 之间的转换能力，目前可用的三方支持包括 jaxb，apache-xmlbeans。Redis 存储的数据将是 xml 工具。不过使用此策略，编程将会有些难度，而且效率最低；不建议使用。注意：需要 spring-oxm 模块的支持

其他序列化器：

* `GenericFastJsonRedisSerializer    `由 FastJson 库实现的序列化器，将数组反序列化为 JSONArray ，查看 JSONArray类的定义就可以看出其是一种 List 集合，所以需要注意其强转的数据类型。

这些方案都实现了`RedisSerializer<T>`接口，这个接口提供了一些静态方法，以方便的获取序列化方案对象：

~~~java
static RedisSerializer<Object> java() {
    return java((ClassLoader)null);
}

static RedisSerializer<Object> java(@Nullable ClassLoader classLoader) {
    return new JdkSerializationRedisSerializer(classLoader);
}

static RedisSerializer<Object> json() {
    return new GenericJackson2JsonRedisSerializer();
}

static RedisSerializer<String> string() {
    return StringRedisSerializer.UTF_8;
}

static RedisSerializer<byte[]> byteArray() {
    return ByteArrayRedisSerializer.INSTANCE;
}
~~~

在RedisTemplate源码中：

~~~java
public void afterPropertiesSet() {
    super.afterPropertiesSet();
    boolean defaultUsed = false;
    if (this.defaultSerializer == null) {
        this.defaultSerializer = new JdkSerializationRedisSerializer(this.classLoader != null ? this.classLoader : this.getClass().getClassLoader());
    }

    if (this.enableDefaultSerializer) {
        if (this.keySerializer == null) {
            this.keySerializer = this.defaultSerializer;
            defaultUsed = true;
        }

        if (this.valueSerializer == null) {
            this.valueSerializer = this.defaultSerializer;
            defaultUsed = true;
        }

        if (this.hashKeySerializer == null) {
            this.hashKeySerializer = this.defaultSerializer;
            defaultUsed = true;
        }

        if (this.hashValueSerializer == null) {
            this.hashValueSerializer = this.defaultSerializer;
            defaultUsed = true;
        }
    }

    if (this.enableDefaultSerializer && defaultUsed) {
        Assert.notNull(this.defaultSerializer, "default serializer null and not all serializers initialized");
    }

    if (this.scriptExecutor == null) {
        this.scriptExecutor = new DefaultScriptExecutor(this);
    }

    this.initialized = true;
}
~~~

如果不设置`defaultSerializer` `keySerializer` 或 `valueSerializer`等那么会默认使用`JdkSerializationRedisSerializer`

如果想要配置指定的序列化方案，必须自己配置

### 配置RedisTemplate

如果redis提供的redisTemplate的默认泛型和序列化方案不满足当前需求，可以自己配置：

~~~java
@Configuration
public class RedisConfig {
    @Bean
    public RedisTemplate<String, Object> redisTemplate( RedisConnectionFactory connectionFactory) {
        // 创建 RedisTemplate 对象
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        // 设置连接工厂
        template.setConnectionFactory(connectionFactory);
        // 设置 key 的序列化
        template.setKeySerializer(RedisSerializer.string());
        template.setHashKeySerializer(RedisSerializer.string());
        // 设置 value 的序列化
        template.setValueSerializer(RedisSerializer.json());
        template.setHashValueSerializer(RedisSerializer.json());
        // 返回
        return template;
    }
}
~~~



### Json序列化导致Long类型退化为Integer

如果配置了redisTemplate json序列化方案，并且泛型是Object时，向redis存入一个Long类型数据

取出时，如果该数据值小于等于`Integer.MAX_VALUE`那么反序列化时，会自动将数值转换为Integer类型

因为并没有指定具体的数据类型，所以反序列化时，将按照默认的类型进行转换

所以如果没有具体的类型信息，可能会导致类型信息不一致。

解决方案：

- 避免使用通用的Object类型。
- 使用Number类型，因为Integer和Long都是Number的子类，通过Number.longValue()可以获取我们想要的类型和数值。
- 统一使用String类型进行数值传递，适当处理字符串和数值之间的转换即可。

## redis作为数据库缓存

### 设计思路

* 接受到查询数据的请求
* 首先访问redis查询
    * 查询到数据，缓存命中，返回数据
    * 没有查询到数据
        * 查询数据库，将结果存入redis，并返回给用户

### 代码实现

```java
public Long queryAllStudentCount() {
    //设置redisTemplate对象key的序列化方式
    redisTemplate.setKeySerializer(new StringRedisSerializer());
    //从redis缓存中获取总人数
    Long allStudentCount = (Long) redisTemplate.opsForValue().get("allStudentCount");
    //判断是否为空
    if (null == allStudentCount) {
        //去数据库查询，并存放到redis缓存中
        System.out.println("数据库查询");
        allStudentCount = studentMapper.selectAllStudentCount();
        redisTemplate.opsForValue().set("allStudentCount",allStudentCount,30, TimeUnit.SECONDS);
    }else{
        System.out.println("缓存命中");
    }
    return allStudentCount;
}
```

### 缓存击穿

上面的代码在并发环境下会出现缓存击穿，缓存中已经有数据，但请求仍然会访问数据库

需要改进代码，进行同步，进行两次请求两次判断

~~~java
public Long queryAllStudentCount() {
    //设置redisTemplate对象key的序列化方式
    redisTemplate.setKeySerializer(new StringRedisSerializer());
    //从redis缓存中获取总人数
    Long allStudentCount = (Long) redisTemplate.opsForValue().get("allStudentCount");
    if(null == allStudentCount){
        synchronized (this){
            allStudentCount = (Long) redisTemplate.opsForValue().get("allStudentCount");
            //判断是否为空
            if (null == allStudentCount) {
                //去数据库查询，并存放到redis缓存中
                System.out.println("-----数据库查询------");
                allStudentCount = studentMapper.selectAllStudentCount();
                redisTemplate.opsForValue().set("allStudentCount",allStudentCount,30, TimeUnit.SECONDS);
            }else{
                System.out.println("------缓存命中-------");
            }
        }
    }else{
        System.out.println("------缓存命中-------");
    }
    return allStudentCount;
}
~~~



# Springboot集成Dubbo

## 依赖

~~~xml
<!--Spring Boot集成Dubbo的起步依赖-->
<dependency>
   <groupId>com.alibaba.spring.boot</groupId>
   <artifactId>dubbo-spring-boot-starter</artifactId>
   <version>2.0.0</version>
</dependency>
<!--ZooKeeper注册中心依赖-->
<dependency>
   <groupId>com.101tec</groupId>
   <artifactId>zkclient</artifactId>
   <version>0.10</version>
    <!--处理Zookeeper包中slf4j-log4j12和log4j的冲突-->
    <exclusions>
        <exclusion>
            <groupId>log4j</groupId>
            <artifactId>log4j</artifactId>
        </exclusion>
        <exclusion>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-log4j12</artifactId>
        </exclusion>
    </exclusions>
</dependency>
~~~

## 配置

~~~yaml
spring:
	application:
		#配置Dubbo应用名称，必须有且唯一
		name:
	dubbo:
		#配置是服务提供者，可省略
		server: true
		#配置注册中心地址
		registry: zookeeper://192.168.92.134:2181
~~~

## 注解

* 开启Dubbo支持

~~~java
@SpringBootApplication
@EnableDubboConfiguration//开启Dubbo配置支持
public class Application {
   public static void main(String[] args) {
      SpringApplication.run(Application.class, args);
   }
}

~~~

* 服务提供者

~~~java
mport com.alibaba.dubbo.config.annotation.Service;
import org.springframework.stereotype.Component;
//@Service是由阿里提供的注解，不是spring的注册
//@Service相当于 <dubbo:service interface="" ref="" version="" timeout=""/>
//如果提供者指定version,那么消费者也得指定version
@Service(interfaceClass = UserService.class,version = "1.0.0",timeout = 15000)
@Component  //将接口实现类交给spring容器进行管理
public class UserServiceImpl implements UserService {
    @Override
    public String say(String name) {
        return "Hello SpringBoot!";
    }
}
~~~

* 服务消费者

~~~java
@RestController
public class UserController {
    //@Reference注解 相当于<dubbo:reference id="" interface="" version="" check="false"/>
    //通过远程调用注入服务提供者
    @Reference(interfaceClass = UserService.class,version = "1.0.0",timeout = 15000,check=false)
    private UserService userService;

    @RequestMapping(value = "/springBoot/hello")
    public Object hello(HttpServletRequest request) {
        String sayHello = userService.say("SpringBoot");
        return sayHello;
    }
}
~~~



# SpringBoot集成Thymeleaf

Thymeleaf时采用Java开发的模板引擎（跨领域跨平台）

它是基于HTML的，以HTML标签为载体，采用标签属性的形式，实现数据的动态展示

## 依赖

~~~xml
<!--SpringBoot集成Thymeleaf的起步依赖，可在创建项目时勾选-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
~~~

## 配置

springboot核心配置：

~~~yaml
spring:
	thymeleaf:
		#thymeleaf页面的缓存开关，默认true开启缓存
		#建议在开发阶段关闭thymeleaf页面缓存，目的实时看到页面
		cache: false
		#thymeleaf模版前后缀,默认可以不写
		prefix: classpath:/templates/
		suffix: .html
~~~

html页面配置：

~~~html
<!--设置thymeleaf命名空间-->
<html lang="en" xmlns:th="http://www.thymeleaf.org">
~~~

## Thymeleaf表达式

* `${}`标准变量表达式

    用于访问容器（tomcat）上下文环境中的变量，功能和EL中的 ${} 相同。

* `@{}`URL表达式

    主要用于链接、地址的展示

    可用于 

    `<script src="...">、<link href="...">、<a href="...">、<form action="...">、<img src="">`等，可以在URL路径中动态获取数据

    Thymeleaf提供了路径变量，参数变量的方式，替代简化了url字符串拼接：

    * 路径变量`th:href="@{/admin/{id}/delete(id=${user.id})}"`
    * 参数变量`th:href="@{/admin(name=${user.name},age=${user.age})}"`

* `*{}`选择变量表达式，也叫星号变量表达式

    首先使用th:object来绑定后台传来的User对象，然后使用 * 来代表这个对象，后面 {} 中的值是此对象中的属性。

## Thymeleaf常用属性

 thymeleaf的大部分属性是在html属性前加上th:前缀

这样的thymeleaf属性在保证html属性的基础功能上，能在属性值中使用thymeleaf的表达式，来达到动态数据的效果，例如：

`th:action` `th:method` `th:href` `th:src` `th:id` `th:name` `th:value` `th:text`

`th:onclick` `th:style `等，不需多做介绍

还有一些thymeleaf独有的属性：

* `th:attr` 可以设置当前HTML的属性属性值：

    ~~~html
    <span th:attr="zhangsan=${user.id}"></span>
    ~~~

* `th:object`用于数据对象绑定，通常用于选择变量表达式（星号表达式）

    ~~~html
    <div th:object="${user}">
        <span th:text="*{id}"></span><br/>
        <span th:text="*{address}"></span><br/>
    </div>
    ~~~

* `th:each`遍历上下文中的容器对象

    格式：`th:each="item[,iterStatu]:${container}"`

    * `item`:迭代变量当次遍历的元素

    * `iterStat`当次循环的信息对象，有如下属性：

        * `index`当前迭代对象的index（从0开始计算）
        * `current`当前迭代对象的个数（从1开始计算）
        * `even/odd`布尔值，当前循环是否是偶数/奇数（从0开始计算）
        * `first`布尔值，当前循环是否是第一个
        * `last`布尔值，当前循环是否是最后一个

        **注意：**循环体信息interStat可省略，默认采用迭代变量加上Stat后缀

    **注意：**th:each在遍历map对象时，获取item是以entrySet的形式，也就是说此时的item有key和value两个属性供操作

    实例：

    ~~~html
    <div th:each="user, iterStat : ${userlist}">
        <span th:text="${userStat.index}"></span>
        <span th:text="${user.id}"></span>
        <span th:text="${user.phone}"></span>
    </div>
    ~~~

* `th:if` 属性值为条件表达式; 如果属性值为true显示标签，否则不显示

* `th:unless`s是th:if的一个相反操作,如果属性值为false显示标签，否则不显示

* `th:switch/th:case`选择语句

    ```html
    <div th:switch="${sex}">
        <p th:case="1">性别：男</p>
        <p th:case="2">性别：女</p>
        <p th:case="*">性别：未知</p>
    </div>
    ```

    `*`表示默认的case，前面的case都不匹配时候，执行默认的case

* `th:inline`内联属性，可以在所属标签范围内使用Thymeleaf内联表达式，而不仅限于属性

    内联表达式：`[[${表达式}]]`

    它有三个属性值：

    * `text`内敛文本 在标签文本中使用内联表达式
    * `javascript`内敛脚本 在js代码中使用内联表达式
    * `none`禁用内联

## Thymeleaf表达式内

Thymeleaf属性值除了支持表达式，还支持：

* 字面量

    * 数字
    * 字符串，用`''`表示
    * boolean值 `true` `false`
    * `null`

* 运算符

    * 算术运算符：`+ , - , * , / , %`
    * 关系运算符: `> , < , >= , <= ( gt , lt , ge , le )`
    * 逻辑运算符：`== , != ( eq , ne )`

* 字符串拼接：

    * `+`拼接

    ~~~html
    <span th:text="'当前是第'+${sex}+'页 ,共'+${sex}+'页'"></span>
    ~~~

    * `|  |`在之间直接拼接

    ~~~html
    <span th:text="|当前是第${sex}页,共${sex}页|"></span>
    ~~~

## Thymeleaf内置对象

模板引擎提供了一组内置的对象，这些内置的对象可以直接在模板中使用，这些对象由#号开始引用

域对象

* `#request`相当于httpServletRequest 对象
* `#session`相当于HttpSession 对象

功能对象

* `#dates`: java.util.Date对象的实用方法
* `#calendars`: 和dates类似, 但是 java.util.Calendar 对象
* `#numbers`: 格式化数字对象的实用方法
* `#strings`: 字符串对象的实用方法
* `#objects`: 对objects操作的实用方法
* `#bools`: 对布尔值求值的实用方法
* `#arrays`: 数组的实用方法
* `#lists`: list的实用方法
* `#sets`: set的实用方法
* `#maps`: map的实用方法
* `#aggregates`: 对数组或集合创建聚合的实用方法



# Springboot集成logback日志

* 依赖

~~~xml
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <version>1.18.2</version>
</dependency>
~~~

* 日志配置

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<!-- 日志级别从低到高分为 TRACE < DEBUG < INFO < WARN < ERROR < FATAL，如果
设置为 WARN，则低于 WARN 的信息都不会输出 -->
<!-- scan:当此属性设置为 true 时，配置文件如果发生改变，将会被重新加载，默认值为
true -->
<!-- scanPeriod:设置监测配置文件是否有修改的时间间隔，如果没有给出时间单位，默认
单位是毫秒。当 scan 为 true 时，此属性生效。默认的时间间隔为 1 分钟。 -->
<!-- debug:当此属性设置为 true 时，将打印出 logback 内部日志信息，实时查看 logback
运行状态。默认值为 false。通常不打印 -->
<configuration scan="true" scanPeriod="10 seconds">
    <!--输出到控制台-->
    <appender name="CONSOLE"
              class="ch.qos.logback.core.ConsoleAppender">
        <!--此日志 appender 是为开发使用，只配置最底级别，控制台输出的日志级别是大
       于或等于此级别的日志信息-->
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <level>debug</level>
        </filter>
        <encoder>
            <Pattern>%date [%-5p] [%thread] %logger{60}
                [%file : %line] %msg%n</Pattern>
            <!-- 设置字符集 -->
            <charset>UTF-8</charset>
        </encoder>
    </appender>
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <!--<File>/home/log/stdout.log</File>-->
        <File>D:/log/stdout.log</File>
        <encoder>
            <pattern>%date [%-5p] %thread %logger{60}
                [%file : %line] %msg%n</pattern>
        </encoder>
        <rollingPolicy
                class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!-- 添加.gz 历史日志会启用压缩 大大缩小日志文件所占空间 -->

            <fileNamePattern>/home/log/stdout.log.%d{yyyy-MM-dd}.log</fileNamePattern>
            <maxHistory>30</maxHistory><!-- 保留 30 天日志 -->

        </rollingPolicy>
    </appender> <logger name="com.abc.springboot.servlet" level="DEBUG" />
    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
        <appender-ref ref="FILE"/>
    </root>
</configuration>
~~~

* springboot配置

~~~yaml
logging:
    config: classpath:logback-spring.xml
~~~

