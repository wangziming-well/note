# 分布式系统

随着互联网的发展，为了应对互联网大数据、高并发、快响应的需求web系统由以前的单机系统发展为多机器协调的系统

我们把这种多台机器相互协作完成企业业务功能的系统，称为分布式系统。

相比于单机系统，分布式系统实现了：

- **高性能：** 因为大量请求被合理地分摊到各个节点，使每一台 Web 服务器的压力减小，并且多个请求可以使用多台机器处理，所以能处理更多的请求和数据，性能更高。如此，便解决了互联网系统的 3 个关键问题——大数据、高并发和快响应。
- **高可用：**如果某个节点出现故障，系统会自动发现这个故障，不再向这个节点转发请求，系统仍旧可以工作。自动避开存在故障的节点，继续对外提供服务。
- **可伸缩性：**当现有机器的性能不能满足业务的发展时，我们需要更多的机器提供服务。只要改造路由算法，就能够路由到新的机器，从而将更多的机器容纳到系统中，继续满足大数据、高并发和快响应的要求。从另一个方面来说，如果现有的机器已经大大超出所需，则可以减少机器，从而节省成本。
- **可维护性：**如果设备当中有一台机器因某种原因不能对外提供服务，如机器出现故障，此时只需要停止那些出现故障的节点，对其进行处理，然后重新上线即可。
- **灵活性：**例如，当需要更新系统时，只需要在非高峰期，停用部分节点，将这些节点更新为最新版本，然后再通过路由算法将请求路由到这些更新后的节点，最后更新那些旧版本的节点，就可以让网站在更新系统时不间断地对外提供服务了。

分布式虽然解决了互联网大数据、高并发、快响应的需求，但也带来了新的问题：

- **异构的机器与网络：**在分布式系统中，机器的配置、架构、性能、系统等都是不一样的。在不同的网络之间，通信带宽、延时、丢包率也是不一样的。那么在多机的分布式系统中，如何才能让所有的机器齐头并进，为同一个业务目标服务，这是一个相当复杂的问题。
- **普遍的节点故障：**在分布式系统中，存在很多机器因为某些原因（如断电、磁盘损坏等）不能继续工作。分布式系统怎么去发现它们，并且自动将它们剔除出去，将请求分配到能够正常工作的节点，以保证系统能够持续提供服务，这也是需要面对的问题。
- **不可靠的网络和机器：**多机器之间的交互是通过网络进行的，而网络传输必然发生分隔、延时、乱序、丢包等问题。机器也会因为请求量的增加而降低处理能力。

因为网络和机器的众多不确定性，注定了分布式的难点在于，如何让多个节点之间保持一致性，服务于企业实际业务。因为数据一致性是分布式的核心问题之一

## 分布式的衡量标准

衡量一个分布式架构的优良：

- **透明性：**所谓透明性，就是指一个分布式系统对外来说如同一个单机系统，使用者不需要知道其内部的实现，只需要知道其参数、功能和返回结果即可。
- **可伸缩性：**当分布式系统的全部现有节点都无法满足业务膨胀的需求时，可以根据需要加入新的节点来应对业务数据的增加。当业务缩减时，又可以根据需要减少节点来达到节省资源的效果。
- **可用性：**一般来说，分布式系统可全天候不间断地提供服务，即使在出现故障的情况下，也尽可能对外提供服务。因而，可以通过正常服务时间和不可用时间的比值来衡量其可用性。
- **可靠性：**可靠性，主要是针对数据来说的，数据要计算正确且不丢失地存储。
- **高性能：**因为有多个节点分摊请求，所以能更快地处理请求。再加上每一个节点都可以高性能地处理请求，所以分布式系统的性能比单机性能高得多。
- **一致性：**因为分布式系统采用了多个节点，所以在一个业务处理中，需要多台机器协作处理数据。然而，网络延迟、丢包、不稳定性或者协作时序错乱，会造成数据的不一致或者丢失。对于一些重要的数据，如账户金额和产品库存等参数，是不允许发生错误和丢失的，所以如何保证数据的一致性和防止丢失是分布式系统的一个重要的衡量标准。

## SOA

Service-Oriented Architecture 面向服务的分布式架构

一种设计方法，其中包含多个服务，服务之间通过相互依赖最终提供一系列的功能，一个服务通常以独立的形式存在于操作系统进程中。各个服务之间通过网络调用。

使用企业服务总线（ESB）作为协调和控制这些服务的手段。

各服务通过ESB进行交互，解决异构系统之间的连通性，通过协议转换、消息解析、消息路由把服务提供者的数据传送到服务消费者。很重但是有一定的逻辑，可以解决一些公用逻辑的问题。

将重复公用的功能抽取为服务组件，以服务的方式为各个系统提供服务

优点：

* 将重复的功能抽取为服务，提高开发效率，提高系统的可重用性、可维护性。
* 可以针对不同服务的特点制定集群及优化方案。

* 采用ESB减少系统中的接口耦合

缺点：

* 系统与服务的界限模糊，不利于开发及维护

* 虽然使用了ESB，但是服务的接口协议不固定，种类繁多，不利于系统维护
* 抽取的服务的粒度过大，系统与服务之间耦合性高

## 微服务

微服务架构和分布式一样，也是面向服务的：将自己的业务能力封装并对外提供服务

但不同的是微服务架构中的每个服务都是单一职责的，一个微服务只解决一个业务问题

与SOA相比，微服务强调业务需要彻底的组件化和服务化，去中心化

# SpringCloud简介

SpringCloud是实现微服务架构的框架，将一系列优秀的组件进行了整合。基于springboot构建。以组件的形式提供了：配置管理，服务发现，智能路由，负载均衡，熔断器，控制总线，集群状态等等功能

所以对SpringCloud的学习就是对其组件的学习

## 项目构建

SpringCloud下模块需要用到其组件依赖，所以需要让项目的父模块也继承springCloud

但一般父模块已经继承了springboot，所以可以通过指定依赖的type和scope来实现继承的效果：

~~~xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>Hoxton.SR1</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
~~~

这样子模块可以直接声明SpringCloud集成的组件依赖来实现依赖

## 组件

需要学习的基础组件：

* Eureka：注册中心
* Zuul：服务网关
* Ribbon：负载均衡
* Feign：服务调用
* Hystix：熔断器

# Eureka

Eureka注册中心实现服务的自动注册、发现、状态监控

服务提供方与Eureka之间通过“心跳”机制进行监控，当某个服务提供方出现问题，Eureka自然会把它从服务列表中剔除。

Eureka采用C-S架构，分为Eureka-client和Eureka-server

需要被Eureka注册管理的服务需要安装Eureka-client

## 安装配置

**服务端**

* 依赖：

~~~xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
~~~

* 注解：在springboot启动类上加上注解：
    * `@EnableEurekaServer`

* 配置springboot配置文件：

~~~yaml
#eureka-server内置了eureka-client，在eurreka服务启动时，客户端会先向下面指定的地址注册服务
#如果构建eureka集群，eureka之间需要互相注册
#否则只需指定自己的地址，向自己注册即可
eureka:
    client:
        service-url:
            defaultZone: http://localhost:10086/eureka
~~~

**客户端**

* 依赖：

~~~xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
~~~

* 注解：在springboot启动类上加上注解:
    * `@EnableDiscoveryClient`适合于springboot支持的所有的注册中心.
    * `@EnableEurekaClient`只能作用在eureka注册中心,

* 配置springboot配置文件：

~~~yaml
eureka:
    client:
        service-url:
            defaultZone: http://localhost:10086/eureka
~~~

## 消费服务

* 管理`restTemplate`

~~~java
@Bean
public static RestTemplate restTemplate(){
    return new RestTemplate();
}
~~~

* 注入`restTemplate`和`discoveryClient`

~~~java
@Autowired
private RestTemplate restTemplate;

@Autowired
private DiscoveryClient discoveryClient;
~~~

* 获取服务

~~~java
//获取服务地址
List<ServiceInstance> instances = discoveryClient.getInstances("user-service");
ServiceInstance serviceInstance = instances.get(0);
//请求服务
String url = "http://"+serviceInstance.getHost()+":"+serviceInstance.getPort()+"/user/query";
User[] user = restTemplate.getForObject(url,User[].class);
~~~

## Eureka高可用

### Eureka集群

* 为了保证eureka高可用，一般会搭建eureka集群，保证有2到3个eureka服务端服务端之间相互注册，服务端之间的数据同步，有一个客户端注册，所有的服务端都会同步收到注册数据。

* 并且eureka客户端只需要向服务端集群中的一个注册即可，注册后服务端会把服务端集群的所有地址信息响应给客户端

### 服务续约

在注册服务完成以后，服务提供者会维持一个心跳（定时向EurekaServer发起Rest请求，默认30秒一次）告诉EurekaServer服务正常，我们称为服务的续约行为

如果超过一段服务提供者没有发送心跳(默认90秒)，EurekaServer就会认为该服务宕机，会从服务列表中移除

配置：

~~~yaml
eureka:
  instance:
    lease-expiration-duration-in-seconds: 10 # 10秒即过期
    lease-renewal-interval-in-seconds: 5 # 5秒一次心跳
~~~

### 获取服务列表

当服务消费者启动是，会检测`eureka.client.fetch-registry=true`参数的值，如果为true，则会从Eureka Server服务的列表只读备份，然后缓存在本地。并且每30秒会重新获取并更新数据。我们可以通过下面的参数来修改：

~~~yaml
eureka:
  client:
    registry-fetch-interval-seconds: 5
~~~

### 失效剔除和自我保护

* 失效剔除

Eureka Server有一个定时任务隔60秒对所有失效的服务（超过90秒未响应）进行剔除。

可以通过`eureka.server.eviction-interval-timer-in-ms`参数对其进行修改，单位是毫秒，生成环境不要修改。

这个会对我们开发带来极大的不变，你对服务重启，隔了60秒Eureka才反应过来。开发阶段可以适当调整，比如10S

* 自我保护

Eureka会统计最近15分钟心跳失败的服务实例的比例是否超过了85%。在生产环境下，因为网络延迟等原因，心跳失败实例的比例很有可能超标，但是此时就把服务剔除列表并不妥当，因为服务可能没有宕机。Eureka就会把当前实例的注册信息保护起来，不予剔除。生产环境下这很有效，保证了大多数服务依然可用。

但是这给我们的开发带来了麻烦， 因此开发阶段我们都会关闭自我保护模式：

~~~yaml
eureka:
  server:
    enable-self-preservation: false # 关闭自我保护模式（缺省为打开）
    eviction-interval-timer-in-ms: 1000 # 扫描失效服务的间隔时间（缺省为60*1000ms）
~~~

# Ribbon

提供了一套微服务的负载均衡解决方案

## 开启Ribbon

* 依赖

~~~xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
</dependency>
~~~

* 注解

~~~java
@Bean
//在restTemplate上添加注解
@LoadBalanced
public static RestTemplate restTemplate(){
    return new RestTemplate();
}
~~~

## 使用Ribbon

使用`restTemplate`时，用服务名代替`host:port`:

~~~java
//请求服务
//user-service是生产服务在eureka中注册的名称
//默认的采用轮询的方式
String url = "http://user-service/user/query";
User[] user = restTemplate.getForObject(url,User[].class);
~~~

ribbon底层使用Interceptor拦截器实现服务名代替`host:port`

并实现均衡负载策略

## 均衡负载策略

通过spring核心配置文件修改均衡负载策略

ribbon的均衡负载策略类都继承了`AbstractLoadBalancerRule`抽象类并且在包`com.netflix.loadbalancer`下

可以通过指定均衡负载类来改变策略：

~~~yaml
user-service:
  ribbon:
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule
~~~

下面是可选的策略：

| 策略类名                  | 策略描述                                                     |
| ------------------------- | ------------------------------------------------------------ |
| BestAvailableRule         | 最小连接：选择一个最小的并发请求的server                     |
| AvailabilityFilteringRule | 过滤掉那些因为一直连接失败的server<br />并过滤掉那些高并发的的后端server<br />在剩下的可选server中随机 |
| WeightedResponseTime      | 根据响应时间分配一个weight权重<br />响应时间越长，weight越小，被选中的可能性越低。 |
| RetryRule                 | 对选定的负载均衡策略机上重试机制。                           |
| RoundRobinRule            | 轮询                                                         |
| RandomRule                | 随机                                                         |
| ZoneAvoidanceRule         | 复合判断server所在区域的性能和server的可用性选择server       |

# Hystrix

Hystix熔断器，在服务响应超时或错误是，进行熔断：消费降级

## 雪崩问题

微服务中,服务间调用错综复杂,一个请求可能调用多个服务才能使用,会形成非常复杂的链路

此时，当其中一个微服务异常时，所有使用到该服务的业务线程都会阻塞

而服务线程和并发数有限,请求一直阻塞,会导致服务器资源耗尽,就会导致其他的服务不能使用,形成雪崩效应.

解决方案：

* 线程隔离，服务降级
* 服务熔断

## 开启Hystrix

* 依赖

~~~xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
~~~

* 注解：在spirngboot启动类上添加注解：
    * `@EnableCircuitBreaker`

**注意:**springCloud提供了组合注解：`@SpringCloudApplication`它代替了注解：`@SpringBootApplication @EnableDiscoverClient @EnableCircuitBreaker`

## Hystrix使用

### 服务降级处理

在调用其他服务的方法上添加注解`@HystrixCommand(fallbackMethod)`

该注解会在声明的方法调用的eureka服务进行降级处理

如果发生调用eureka服务超时（默认1s）,或者异常那么就调用超时的方法

对于fallbackMethod方法：

* 返回值类型必须和主方法返回值类型一致
* 参数类型必须和主方法的参数类型保持一致

示例：

~~~java
@RestController
public class ConsumerController {

    @Autowired
    UserClient userClient;

    @Autowired
    DiscoveryClient discoveryClient;

    //fallbackMethod函数和当前方法的方法签名必须一致
    @GetMapping("/getUser")
    @HystrixCommand(defaultFallback = "queryAndError")
    public User getUser(Integer id){
        if(id % 2 == 0){
            throw new RuntimeException();
        }
        return userClient.getUser();
    }

    public User queryAndError(){
        return new User(403,"请少后续再试");
    }
}
~~~

### 统一降级处理

如果一个类中所有方法的错误降级方法都一致，那么可以在类上添加注解`DefaultProperties(defaultFallback)`

而方法上的`@HystrixCommand`注解可以省略`defaultFallback`参数了

## Hystrix配置

`@HystrixCommand`的`commandProperties`字段用来进行Hystrix配置该字段值为	`@HystrixProperty`注解

通过设置`@HystrixProperty`注解的`name`和`value`字段来配置Hystrix行为

hystrix的配置类：`com.netflix.hystrix.HystrixCommandProperties`

可以通过该类获取配置`@HystrixProperty`的`name`字段

### 超时时间

* 局部配置

~~~java
//配置局部超时时间为3000
@HystrixCommand(commandProperties ={@HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds",value = "3000")})
~~~

* 全局配置

~~~yaml
hystrix:
    command:
        default:
        #字段与@HystrixProperty的name字段一致
            execution:
                isolation:
                    thread:
                        timeoutInMilliseconds: 3000
~~~



### 熔断器状态

Hystrix熔断器有三种状态：

* closed关闭状态，所有请求正常访问
* open开启状态，当一定请求次数(默认20次)内请求失败超出了阈值(默认50%),那么开启熔断器,快速返回失败数据

* half open半开状态，熔断器完全开启后开始计时,一定时间后（默认5s）就开始处与半开状态

    此时熔断器会接受一部分请求，如果请求：

    * 仍然超时或者异常，熔断器将转变为开启状态
    * 正常响应，熔断器将转变为关闭状态

可以通过配置控制状态转换行为

* 局部配置

~~~java
@HystrixCommand(commandProperties =
                //关闭状态时，判断是否开启熔断器的接受次数
             @HystrixProperty(name = "circuitBreaker.requestVolumeThreshold",value = "10"),
                //失败阈值，超过该值熔断器将开启
             @HystrixProperty(name = "circuitBreaker.errorThresholdPercentage",value = "40"),
                //开到半开时间
             @HystrixProperty(name = "circuitBreaker.sleepWindowInMilliseconds",value = "10000"),
            })
~~~

* 全局配置

~~~yaml
hystrix:
    command:
        default:
            circuitBreaker:
                requestVolumeThreshold: 10
                errorThresholdPercentage: 40
                sleepWindowInMilliseconds: 10000
~~~



# Feign

伪装远程调用，让远程调用像本地调用一样

## 开启设置

* 依赖

~~~xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
~~~

* 注解：springboot启动类上添加注解：`@EnableFeignClients`

## 使用

### 创建Feign接口

~~~java
//标记为feign接口，注解参数值值为要调用的服务在eureka中的名称
@FeignClient("producer")
public interface UserClient {
	//GetMapping参数值为访问服务提供方的路径
    @GetMapping("getUser")
    User getUser();
}
~~~

### 调用Feign接口

~~~java
@RestController
public class ConsumerController {

    @Autowired
    UserClient userClient;

    @GetMapping("/getUser")
    public User getUser(Integer id){
        return userClient.getUser();
    }
}
~~~

### Feign集成Ribbon

Feign中本身已经集成了Ribbon依赖和自动配置

### Feign集成Hystix

Feign默认也有对Hystix的集成，但默认情况下是关闭的。需要通过下面的参数来开启

~~~yaml
feign:
  hystrix:
    enabled: true # 开启Feign的熔断功能
~~~

使用：

~~~java
//添加降级类fallback
@FeignClient(value = "producer" ,fallback = UserClientFallback.class)
public interface UserClient {

    @GetMapping("getUser")
    User getUser();
}

//定义fallback类 需要给该实现添加@Component注解,交给spring来管理
//需要实现主接口，并且重写的方法就为接口的fallback方法
@Component
public class UserClientFallback implements UserClient {
    @Override
    public User getUser() {
        return new User(403,"请稍后再试");
    }
}
~~~

建议不使用集成，而是单独使用



# Zuul

Zuul网关组件

网关是微服务架构中一个不可或缺的部分。通过服务网关统一向外系统提供REST API的过程中，除了具备服务路由、均衡负载功能之外，它还具备了权限控制等功能。

## 安装启动

* 依赖

~~~xml
<!--zuul 已经依赖了spring-boot-starter-web包了-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
</dependency>
~~~

* 注解：springboot启动类上添加注解：`@EnableZuulProxy`

## 配置映射

假设网关端口为10010

* 映射方式一：

~~~yaml
zuul:
	routes:
		#key是自定义的
		haha:
			path: /getUser/**
			url: http://localhost:8081
~~~

该配置将

`http://localhost:10010/getUser`映射为`http://localhost:8081/getUser`

以下配置需要依赖eureka，引入eureka服务端依赖

* 映射方式二：

~~~yaml
zuul:
	routes:
		haha:
			path: /service-consumer/**
			#consumer为服务名，服务在eureka中注册的名字
			serviceId: consumer 
~~~

该配置将

`http://localhost:10010/service-consumer/getUser`映射为

`http://localhost:8080/getUser`

* 映射方式三

映射方式二的简化操作

~~~yaml
zuul:
	routes:
		#consumer为服务名，服务在eureka中注册的名字
		consumer: /service-consumer/**
~~~

如果采用这种方式，zuul会默认将eureka中注册的其他服务按照这种方式映射

## 其他配置

* 禁止访问规则

~~~yaml
zuul:
    routes:
        consumer: /service-consumer/**
    ignored-services:
    #禁止访问的服务名称
        - producer
        - consumer

~~~

* 添加前缀

~~~yaml
#在发起请求时，路径就要以/api开头
zuul:
    prefix: /api
    
~~~

## 过滤器

要自定义Zuul的过滤器，需要继承ZuulFilter抽象类，并重写下面方法

~~~java
public abstract ZuulFilter implements IZuulFilter{

    abstract public String filterType(); //过滤器类型

    abstract public int filterOrder();   //过滤器优先级
    
    boolean shouldFilter();              //要不要过滤

    Object run() throws ZuulException;   //过滤器逻辑
}

~~~

* filterType返回字符串，代表过滤器的类型。包含以下4种：
    * pre：请求在被路由之前执行
    * routing：在路由请求时调用
    * post：在routing和error过滤器之后调用
    * error：处理请求时发生错误调用

* filterOrder通过返回的int值来定义过滤器的执行顺序，数字越小优先级越

* shouldFilter：返回一个Boolean值，判断该过滤器是否需要执行
    * true执行
    * false不执行

* run：过滤器的具体业务逻辑。

示例：

~~~java
@Component
public class AuthenticationFilter extends ZuulFilter {
    @Override
    public String filterType() {
        return FilterConstants.PRE_TYPE;
    }

    @Override
    public int filterOrder() {
        return FilterConstants.PRE_DECORATION_FILTER_ORDER -1;
    }

    @Override
    public boolean shouldFilter() {
        RequestContext currentContext = RequestContext.getCurrentContext();
        HttpServletRequest request = currentContext.getRequest();
        String requestURI = request.getRequestURI();
        return !"/api/consumer/login".equals(requestURI);
    }

    @Override
    public Object run() throws ZuulException {
        RequestContext currentContext = RequestContext.getCurrentContext();
        HttpServletRequest request = currentContext.getRequest();
        Object token = request.getParameter("token");
        if(token == null || token.equals("")) {
            //是否放行,false不放行,true放行
            currentContext.setSendZuulResponse(false);
            //响应状态码 403
            currentContext.setResponseStatusCode(403);
        }
        return null;
    }
}
~~~

## 均衡负载和熔断

Zuul中默认就已经集成了Ribbon负载均衡和Hystix熔断机制

但是所有的超时策略都是走的默认值。因此建议我们手动进行配置：

~~~yaml
hystrix:
  command:
    default:
      execution:
        isolation:
          thread:
            timeoutInMillisecond: 6000 # 熔断超时时长：6000ms
ribbon:
  ConnectTimeout: 250 # 连接超时时间(ms)
  ReadTimeout: 2000 # 通信超时时间(ms)
~~~

## Zuul高可用

所有用户的访问请求都要通过网关，如果只有一个Zuul网关，那么zuul宕机或出错，则所有的服务都不能被访问。

所以需要做zuul的高可用集群

zuul集群需要再被nginx反向代理并均衡负载

nginx再部署集群并配置虚拟IP

就实现了网关的高可用





