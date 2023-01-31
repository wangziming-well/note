# RabbitMQ简介

RabbitMQ是实现了高级消息队列协议（AMQP）的开源消息代理软件（亦称面向消息的中间件）实现了JMS接口

AMQP协议更多用在企业系统内，对数据一致性、稳定性和可靠性要求很高的场景，对性能和吞吐量的要求还在其次。



## JMS概述

JMS即Java消息服务（Java Message Service）应用程序接口。

是一个Java平台中关于面向消息中间件（MOM）的API。

用于在两个应用程序之间，或分布式系统中发送消息，进行异步通信。

JMS特点：

* 异步：JMS天生就是异步的，客户端获取消息的时候，不需要主动发送请求，消息会自动发送给可用的客户端。

* 可靠：JMS保证消息只会递送一次。

## RabbitMQ架构

![RabbitMQ架构](https://gitee.com/wangziming707/note-pic/raw/master/img/RabbitMQ%E6%9E%B6%E6%9E%84.png)

* Publisher&Consumer：消息的生产者和消费者
* Exchange:交换器，用来接收生产者发送的消息并将这些消息路由给服务器中的队列。消息交换机，它指定消息按什么规则，路由到哪个队列。

* Queue:消息的载体，每个消息都会被投到一个或多个队列，等待消费者连接到这个队列将其取走。它是消息的容器，也是消息的终点。

* Binding: 绑定，关联了exchange和queue

* Connection：网络连接，例如一个TCP连接。

* Channel : 消息通道，在客户端的每个连接里，可建立多个channel。

    多路复用连接中的一条独立双向数据流通道。信道是建立在真实的TCP连接内的虚拟连接，AMQP命令都是通过信道发出去的，不管是发布消息、订阅队列还是接收消息，这些动作都是通过信道完成。因为对于操作系统来说建立和销毁TCP都是非常昂贵的开销，所以引入了信道的概念以达到复用一条TCP连接的目的。

## RabbitMQ安装

* docker安装

~~~bash
# 创建RabbitMQ容器 使用management
docker run -d -p 5672:5672 -p 15672:15672 --name rabbitmq rabbitmq:management
~~~

* java客户端依赖

~~~java
<dependency>
    <groupId>com.rabbitmq</groupId>
    <artifactId>amqp-client</artifactId>
    <version>5.7.3</version>
</dependency>
~~~



# 基础实现

## 获取连接

通过连接工厂获取连接

~~~java
public static Connection getConnection() throws IOException, TimeoutException {
    ConnectionFactory connectionFactory = new ConnectionFactory();
    connectionFactory.setHost("192.168.134.128");
    connectionFactory.setUsername("guest");
    connectionFactory.setPassword("guest");
    connectionFactory.setPort(5672);
    return connectionFactory.newConnection();
}
~~~

## 生产消息

~~~java
//获取连接
Connection connection = getConnection();
//获取管道
Channel channel = connection.createChannel();
//声明消息队列
channel.queueDeclare("hello", true, false, false, null);
String message = "message";
//发布消息
channel.basicPublish("", "queue", MessageProperties.PERSISTENT_TEXT_PLAIN, message.getBytes());
//关闭连接
channel.close();
connection.close();
~~~

### queueDeclare

Channel 的queueDeclare方法签名如下：

~~~java
Queue.DeclareOk queueDeclare(String queue, boolean durable, boolean exclusive, boolean autoDelete,Map<String, Object> arguments);
~~~

* queue 声明的消息队列名称
* durable 声明的队列是否为持久队列(队列在服务器重启后仍存在)
* exclusive 声明的是否是一个独占队列(排他队列)
    * 只对首次声明它的连接（Connection）可见
    * 会在其连接断开的时候自动删除。
* autoDelete 声明的是否是一个自动删除队列(服务器将在不再使用时将其删除)
* arguments 队列的其他属性

### basicPublish

Channel 的basicPublish方法签名如下：

~~~java
void basicPublish(String exchange, String routingKey, BasicProperties props, byte[] body);
~~~

* exchange:要路由的交换机
* routingKey:路由关键字，指定要发布的目标队列

* BasicProperties 指定消息发布的类型
* body：消息本题

BasicProperties 类是多例模式，提供下面类型：

~~~properties
MINIMAL_BASIC: 空的基本属性，没有设置字段
MINIMAL_PERSISTENT_BASIC:空的基本属性，只有 deliveryMode 设置为 2（持久）
BASIC:"application/octet-stream" 非持久的
PERSISTENT_BASIC "application/octet-stream" 持久的
TEXT_PLAIN:"text/plain" 持久的
PERSISTENT_TEXT_PLAIN:"text/plain" 持久的
~~~

## 消费消息

~~~java
//获取连接
Connection connection = getConnection();
//获取管道
Channel channel = connection.createChannel();
//获取消息 该方法为阻塞方法
channel.basicConsume("queue", true, new DefaultConsumer(channel){
    @Override
    public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
        System.out.println(new String(body));
    }
});
~~~

### basicConsume

Channel 的basicConsume方法签名如下：

~~~java
String basicConsume(String queue, boolean autoAck, Consumer callback) 
~~~

* queue：获取消息的队列名

* autoAck：

    * true:表示自动确认，只要消息从队列中获取，无论消费者获取到消息后是否成功消费，都会认为消息已经成功消费 	 
    * false:表示手动确认，消费者获取消息后，服务器会将该消息标记为不可用状态，等待消费者的反馈，如果消费者一直没有反馈，那么该消息将一直处于不可用状态，并且服务器会认为该消费者已经挂掉，不会再给其发送消息，直到该消费者反馈。

* callback：消费消息的回调接口，消息在该接口的方法中被消费

    可以传入DefaultConsumer并重写其handleDelivery方法

    ~~~java
    void handleDelivery(String consumerTag,Envelope envelope,AMQP.BasicProperties properties,byte[] body);
    ~~~

    * consumerTag 与消费者相关的消费者标签
    * envelope 消息的打包数据
    * properties 消息的内容头数据
    * body 消息体（不透明的、特定于客户端的字节数组）

# 工作模式

RabbitMQ可以通过在声明exchange交换机时设置交换机类型来实现不同的工作模式（或者直接不声明exchange，即通过默认交换机）

针对消费者与消息队列：

* Simple
* Work

针对交换机的路由模式：

* Publish/Subscribe : Fanout
* Routing : Direct
* Topic: Topic

两类模式可以组合使用



## Simple

![RabbitMQ工作模式-简单模式](https://gitee.com/wangziming707/note-pic/raw/master/img/RabbitMQ%E5%B7%A5%E4%BD%9C%E6%A8%A1%E5%BC%8F-%E7%AE%80%E5%8D%95%E6%A8%A1%E5%BC%8F.png)

simple模式是一种简单的收发模式,一个队列只有一个消费者在监听，该队列所有的消息都由这一个消费者消费。

该模式下不需要指定交换机，RabbitMQ会通过默认的`default AMQP交换机`将我们的消息投递到指定的队列。

它是一种Direct类型的交换机，队列与它绑定时的binding key其实就是队列的名称。

![RabbitMQ默认交换机](https://gitee.com/wangziming707/note-pic/raw/master/img/RabbitMQ%E9%BB%98%E8%AE%A4%E4%BA%A4%E6%8D%A2%E6%9C%BA.png)

### 生产者

~~~java
Connection connection = getConnection();
Channel channel = connection.createChannel();

channel.queueDeclare("queue", true, false, false, null);
String message = "message";
// 不需要指定exchange，入参为空字符串即可，rountingkey为要投递的队列名
channel.basicPublish("", "queue", MessageProperties.PERSISTENT_TEXT_PLAIN, message.getBytes());

channel.close();
connection.close();
~~~

### 消费者

只有一个消费者消费该消息队列

~~~JAVA
Connection connection = getConnection();
Channel channel = connection.createChannel();
channel.basicConsume("queue", true, new DefaultConsumer(channel){
    @Override
    public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
        System.out.println(new String(body));
    }
});
~~~

## Work

![RabbitMQ工作模式-Work](https://gitee.com/wangziming707/note-pic/raw/master/img/RabbitMQ%E5%B7%A5%E4%BD%9C%E6%A8%A1%E5%BC%8F-Work.png)

work模式采用的也是默认的default AMQP交换机

多个消费者可以监听同一个队列，但是一条消息只能有一个消费者获得消息

该模式下消费者和生产者的代码和Simple一样，只是消费者有多个

work消息队列有2种消息分发模式

* 轮询分发：一个消费者消费一条，按均分配，work模式下默认是采用轮询分发方式。

* 公平分发：根据消费者的消费能力进行公平分发，处理得快的分得多，处理的慢的分得少

### 公平分发

公平分发模式下我们需要修改RabbitMQ的配置

* 将消息确认模式改为手动确认

* 将预处理模式更改为每次读取1条消息，在消费者未返回确认之前，不再进行下一条消息的消费

~~~java
Connection connection = getConnection();
Channel channel = connection.createChannel();
//保证一次只分发一个消息
channel.basicQos(1);
//关闭autoAck
channel.basicConsume("queue", false, new DefaultConsumer(channel){
    @Override
    public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
        System.out.println( name+":" +new String(body));
        //手动回执
        this.getChannel().basicAck(envelope.getDeliveryTag(),false);
    }
});
~~~

## Publish/Subscribe

![RabbitMQ工作模式-Fanout](https://gitee.com/wangziming707/note-pic/raw/master/img/RabbitMQ%E5%B7%A5%E4%BD%9C%E6%A8%A1%E5%BC%8F-Fanout.png)

发布订阅模式，此模式下交换机类型为`fanout`

交换器绑定多个消息队列，投递到该路由的消息将被路由到该交换器绑定的所有消息队列

### 生产者

~~~java
Connection connection = getConnection();
Channel channel = connection.createChannel();
//指定交换机类型为fanout
channel.exchangeDeclare("fanout_exchange", "fanout");

channel.queueDeclare("fanout_queue1", true, false, false, null);
channel.queueDeclare("fanout_queue2", true, false, false, null);
//该模式下不需要指定routingKey
channel.queueBind("fanout_queue1", "fanout_exchange", "");
channel.queueBind("fanout_queue2", "fanout_exchange", "");
//投递消息
String message = "message";
channel.basicPublish("fanout_exchange", "",MessageProperties.PERSISTENT_TEXT_PLAIN, message.getBytes());

channel.close();
connection.close();
~~~

## Routing

![RabbitMQ工作模式-Routing](https://gitee.com/wangziming707/note-pic/raw/master/img/RabbitMQ%E5%B7%A5%E4%BD%9C%E6%A8%A1%E5%BC%8F-Routing.png)

路由模式，对应的交换机类型是Direct

生产者投递消息时需要指定routing key，交换机会根据routing key 将消息投递到指定的队列

### 生产者

~~~java
//获取连接和管道
Connection connection = getConnection();
Channel channel = connection.createChannel();
//声明交换机，指定交换机类型为direct
channel.exchangeDeclare("direct_exchange", "direct");
//声明队列
channel.queueDeclare("direct_queue1", true, false, false, null);
channel.queueDeclare("direct_queue2", true, false, false, null);
//绑定交换机和队列，需要指定routingKey
channel.queueBind("direct_queue1", "direct_exchange", "key1");
channel.queueBind("direct_queue2", "direct_exchange", "key2");
//投递消息，向交换机direct_exchange下绑定的routingKey为key1的队列投递消息
String message = "message";
channel.basicPublish("direct_exchange", "key1",
                     MessageProperties.PERSISTENT_TEXT_PLAIN, message.getBytes());
//关闭资源
channel.close();
connection.close();
~~~

## Topic

![RabbitMQ工作模式-Topic](https://gitee.com/wangziming707/note-pic/raw/master/img/RabbitMQ%E5%B7%A5%E4%BD%9C%E6%A8%A1%E5%BC%8F-Topic.png)

主题模式，对应的交换机类型是Topic，

该模式支持模糊匹配，生产者投递消息时，Topic类型的交换机会根据routing key匹配所有符合队列与交换机绑定时指定的binding key规则的队列，并将消息投递到那些队列中。

该模式下bindingKey通过`.`对关键字进行分级

bindingKey通配符：

* `*`匹配一级关键字
* `#`匹配多级关键字

例如：对bindingKey`*.rabbit.*`和`a.#` ，

`a.rabbit.b`都能匹配

`a.rabbit`匹配`a.#`

`xxx.rabbit.yyy`匹配`*.rabbit.*`

### 生产者

~~~java

Connection connection = getConnection();
Channel channel = connection.createChannel();
//声明交换机，指定交换机类型为topic
channel.exchangeDeclare("topic_exchange", "topic");

channel.queueDeclare("topic_queue1", true, false, false, null);
channel.queueDeclare("topic_queue2", true, false, false, null);
//绑定交换机和队列，指定带通配符的分级routingKey
channel.queueBind("topic_queue1", "topic_exchange", "*.rabbit.*");
channel.queueBind("topic_queue2", "topic_exchange", "lazy.#");
String message = "message";
channel.basicPublish("topic_exchange", "lazy.rabbit",
                     MessageProperties.PERSISTENT_TEXT_PLAIN, message.getBytes());
channel.close();
connection.close();
~~~

# 保证消息的可靠性

消息队列作为应用间的通信，受网络环境等因素影响，在收发消息时，不可避免得会出现消息丢失的情况，如果没有其他措施进行确认，接受消息方和投递消息方无法得知该消息是否丢失(无法针对丢失消息进行补偿)，也就无法保证消息的可靠性

RabbitMQ针对此提供了几种措施

## 事务

RabbitMQ实现了AMQP协议的事务规范，保证消息的可靠性

事务的实现主要是对信道(Channel)的设置，主要方法如下：

* `channel.txSelect()  `声明启动事务模式
* `channel.txCommit() `提交事务
* `channel.txRollback()`回滚事务

该模式下，生产者发布消息前后会和消息队列进行一次通信确认

### 正常事务

~~~java
channel.txSelect();
channel.basicPublish("exchange", "key",
MessageProperties.PERSISTENT_TEXT_PLAIN, "message".getBytes());
channel.txCommit();
~~~

此时发布消息流程为：

![RabbitMQ正常事务流程](https://gitee.com/wangziming707/note-pic/raw/master/img/RabbitMQ%E6%AD%A3%E5%B8%B8%E4%BA%8B%E5%8A%A1%E6%B5%81%E7%A8%8B.png)

### 回滚事务

~~~java
try {
    channel.txSelect();
    channel.basicPublish("exchange", "key", MessageProperties.PERSISTENT_TEXT_PLAIN, message.getBytes());
    int result = 1 / 0;
    channel.txCommit();
} catch (Exception e) {
    e.printStackTrace();
    channel.txRollback();
}
~~~

![RabbitMQ回滚事务流程](https://gitee.com/wangziming707/note-pic/raw/master/img/RabbitMQ%E5%9B%9E%E6%BB%9A%E4%BA%8B%E5%8A%A1%E6%B5%81%E7%A8%8B.png)

### 事务总结

通过上面的分析，事务确实能保证消息的可靠性，但每次发布消息都伴随着生产者和消息队列的数次通讯， 因此使用事务机制会大幅降低RabbitMQ的性能

所以在实际生产时很少使用事务

## Confirm模式

针对消息生产者，保证发布消息的可靠性

设置Channel进行发送方确认的。最终确保生产者消息全部发送成功。confirm确认模式要比事务快。

![RabbitMQ消息确认](https://gitee.com/wangziming707/note-pic/raw/master/img/RabbitMQ%E6%B6%88%E6%81%AF%E7%A1%AE%E8%AE%A4.jpg)

在confirm模式下，生产者发布消息后会等待RabbitMQ回执发送结果，根据结果判断消息是否发送成功

该模式下的确认消息的操作是异步的，所以生产者发送消息后不需要等回执结果，可以直接继续发送下一条消息

开启管道channel的confirm模式：

~~~java
channel.confirmSelect();
~~~

在该模式下有三种确认消息的方式：

### 单条确认

~~~java
//开启confirm模式
channel.confirmSelect();
channel.basicPublish("exchange","key",MessageProperties.PERSISTENT_TEXT_PLAIN, "message".getBytes());
//返回消息确认结果，超过5s没有返回，自动返回false
boolean b = channel.waitForConfirms(5000);
System.out.println(b);
~~~

### 批量确认

~~~java
//开启confirm模式
channel.confirmSelect();
//此处发送批量消息
channel.basicPublish("exchange","key",MessageProperties.PERSISTENT_TEXT_PLAIN, "message".getBytes());
//批量确认消息发送结果，如果消息确认失败或者超时，将抛出异常
channel.waitForConfirmsOrDie();
~~~

批量确认，如果确认失败，那么无法知道是哪条消息发送失败，该批次的消息必须都进行补偿，此时又会出现重复消费的问题

### 异步确认

通过监听器异步确认

~~~java
//开启confirm模式
channel.confirmSelect();
//定义消息确认监听器
channel.addConfirmListener(new ConfirmListener() {
    //消息确认回调
    @Override
    public void handleAck(long deliveryTag, boolean multiple) throws IOException {
        System.out.println("确认消息,编号:" + deliveryTag + "   " + multiple);
    }
	//消息未确认回调
    @Override
    public void handleNack(long deliveryTag, boolean multiple) throws IOException {
        System.out.println("消息没被确认,编号:" + deliveryTag + "   " + multiple);
    }
});
//发送消息
channel.basicPublish("exchange","key",MessageProperties.PERSISTENT_TEXT_PLAIN, "message".getBytes());
~~~





##  ACK机制

针对消息消费者，保证消费消息的可靠性

了保证消息从队列可靠的达到消费者，RabbitMQ 提供了消息确认机制（Message Acknowledgement）。

当消息队列发送消息给消费者时，不会立即将消息删除，而是等待消费者的确认，消费者确认收到（并消费）消息后才会删除该消息

消费者在订阅队列时，可以指定 autoAck 参数

* 当 autoAck为true 时，消息在发送到消费者后就会被确认，然后从内存（或者磁盘）中删除，而不管消费者是否真正地消费到了这些消息

* 当 autoAck为false 时，RabbitMQ 会等待消费者显式地回复确认信号后才从内存（或者磁盘）中移除消息（实际上是先打上删除标记，之后在删除）。



### 自动ACK

~~~java
//autoACK为true 
channel.basicConsume("queue", true, new DefaultConsumer(channel){
    @Override
    public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
        //此时消费者并为消费该消息，但RabbitMQ已经确认了该消息
        //所以该消息实际上也丢失了
        int i = 10/0;
        System.out.println(new String(body));
    }
});
~~~

### 手动ACK

~~~java
//autoACK为false
channel.basicConsume("queue", false, new DefaultConsumer(channel){
    @Override
    public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
        int i = 10/0;
        System.out.println(new String(body));
        //手动确认
        this.getChannel().basicAck(envelope.getDeliveryTag(),true);
    }
});
~~~



# Springboot整合RabbitMQ

## 依赖配置

* maven依赖

~~~xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
~~~

* springboot核心配置文件

~~~yaml
spring:
    rabbitmq:
        host: 192.168.134.128
        port: 5672
        username: guest
        password: guest

~~~

## 声明RabbitMQ组件

springboot可以通过配置类通知RabbitMQ声明exchange交换机和queue队列，并绑定

只需要将对应的类交给springboot管理，springboot会自动声明

这些类在`org.springframework.amqp.core`核心包下,它们的继承关系如下

![Declarable](https://gitee.com/wangziming707/note-pic/raw/master/img/Declarable.png)

### Declarable

该接口为声明组件的顶级接口

继承该接口的类，如果被springboot容器管理，那么在初始化上下文阶段将被AmqpAdmin(RabbitAdmin)自动声明

AmqpAdmin是用来操作RabbitMQ组件，用来声明，删除，获取 queue, binding，exchange的

#### Declarable自动声明

RabbitAdmin在加载到Bean容器之后，会声明一个ConnectionListener

该监听器将在建立连接后将Spring容器中实现Declarable接口的实例(Exchange,Queue,Binding)声明给RabbitMQ

### AbstractExchange

是所有交换机的抽象父类，其完整构造方法的方法签名如下：

~~~java
public AbstractExchange(String name, boolean durable, boolean autoDelete, Map<String, Object> arguments) 
~~~

* `name`要声明的交换机名称
* `durable`设置要声明的交换机是否为持久的(交换机是否在服务器重启后仍存在)
* `autoDelete`当所有绑定队列都不在使用时，是否自动删除交换器
* `arguments`用于声明该交换机的其他参数

### Binding

在springboot集成rabbitmq中，Binding实例不是直接new出来的，而是通过

BindingBuilder的静态方法构建出来的，该Builder按照链式编程的思想构建，让我们可以基于流式API创建Binding：

~~~java
BindingBuilder.bind(queue) //绑定的队列
                .to(exchange)//绑定的交换机
                .with("key")//绑定的routingKey
                .noargs();//表示构建结束，返回Binding对象
~~~

### Queue

其完整构造方法签名如下：

~~~java
public Queue(String name, boolean durable, boolean exclusive, boolean autoDelete,@Nullable Map<String, Object> arguments) 
~~~

* `name`要声明的队列名称
* `durable`设置要声明的队列是否为持久的(交换机是否在服务器重启后仍存在)
* `exclusive`声明的是否是一个独占队列
* `autoDelete`当所有消费客户端连接断开后，是否自动删除队列
* `arguments`用于声明该队列的其他参数

## 收发消息

SpringBoot集成RabbitMQ后可以通过RabbitTemplate和注解轻松实现消息的投递和消费

### 消息

springboot封装RabbitMQ后，对传递的消息进行了封装，实际传递的是：`org.springframework.amqp.core.Message` 对象

它主要由两部分组成

~~~java
//消息属性
private final MessageProperties messageProperties;
//消息主体
private final byte[] body;
~~~

传递和接受消息时会由`MessageConverter`进行byte[]与对象之间的转化

### 投递消息

使用RabbitTemplate的convertAndSend方法，方法签名如下

~~~java
void convertAndSend(String exchange, String routingKey, Object message)
~~~

* `exchange`交换机名
* `routingKey`路由键名
* `message`要发送的消息

### 消费消息

可通过`@RabbitListener`消费注解

#### @RabbitListener声明在方法上

此时可通过`@Payload `和 `@Headers `注解可以获取消息对象Message中的` body `和 `messageProperties`

它们都会被 MessageConvert 转换器解析转换后(使用 fromMessage 方法进行转换)，将结果绑定在对应注解的方法中。

MessageConvert 会根据Message的MessageProperties的content_type解析返回数据，用@Payload时必须用content_type允许的类型接收，否则会报错

* `application/octet-stream`：二进制字节数组存储，使用 byte[]

* `application/x-java-serialized-object`：java 对象序列化格式存储，使用 Object、相应类型（反序列化时类型应该同包同名，否者会抛出找不到类异常）

* `text/plain`：文本数据类型存储，使用 String

* `application/json`：JSON 格式，使用 Object、相应类型

示例：

~~~java
//@RabbitListener所在类必须交给spring管理

//声明该类是队列queue的监听器
@RabbitListener(queues = "queue")
//@Payload声明的参数类型必须与发送时的类型一致
//@Headers声明的参数类型必须是MessageHeaders
public void getMessage(@Payload User user,
                       @Headers MessageHeaders headers){
    System.out.println(user);
    System.out.println(headers);
}
~~~

#### @RabbitListener声明在类上

声明在类上时，需要和注解`@RabbitHandler`搭配使用

`@RabbitListener `标注在类上面表示当有收到消息的时候，就交给 `@RabbitHandler `注解的方法进行分发处理，具体使用哪个方法处理，根据 `MessageConverter` 转换后的参数类型

此时`@Payload `和 `@Headers `注解仍然可用

示例：

~~~java
//根据消息的类型，投递到不同的处理器消费
@Component
@RabbitListener(queues = "queue")
public class JMSListener {
    @RabbitHandler
    public void getUser(User user){
        System.out.println(user);
    }

    @RabbitHandler
    public void getStr(String message){
        System.out.println(message);
    }
}
~~~

#### @RabbitListener 常用参数

* `@RabbitListener`的常用参数声明如下：

~~~java
//监听器监听的队列，可以监听多个队列
String[] queues() default {};

//值格式为"n-m"
//指定消费者的线程数量,一个线程会打开一个Channel最少为n个，最多为m个
String concurrency() default "";

//值为@Queue注解数组
//该值将被RabbitAdmin声明到RabbitMQ上，并绑定该监视器
//该队列将被绑定到RabbitMQ的default exchange上
Queue[] queuesToDeclare() default {};

//值为@QueueBinding注解数组
//该绑定，和绑定对应的交换机和数组将被RabbitAdmin声明到RabbitMQ上
//声明的队列将被绑定到该监视器上
QueueBinding[] bindings() default {};
//设定监听器的消息确认模式
//值有: none,manual,auto
String ackMode() default "";
~~~

* `@QueueBinding`的常用参数声明如下：

~~~java
//要绑定的队列
Queue value();

//要绑定的交换机
Exchange exchange();

//routingKey的值
String[] key() default {};
~~~

* `@Queue`的常用参数声明如下

~~~java
//队列名
@AliasFor("name") String value()default "";
//是否是持久队列，默认是
String durable() default "";
//是否是独占队列，默认不是
String exclusive() default "";
//是否自动删除.默认不是
String autoDelete() default "";
~~~

* `@Exchange`的常用参数声明如下：

~~~java
//交换机名 
@AliasFor("value")String name() default "";
//交换机类型，默认为direct
String type() default ExchangeTypes.DIRECT;
//是否为持久交换机，默认为true
String durable() default TRUE;
//是否自动删除，默认为false
String autoDelete() default FALSE;
~~~



## 死信队列

死信队列：没有被及时消费的消息存放的队列

消息没有被及时消费有以下几点原因：

* 消息被拒绝（basic.reject/ basic.nack）并且不再重新投递 requeue=false

* TTL(time-to-live) 消息超时未消费

* 达到最大队列长度

配置如下：

~~~java
@Configuration
public class JMSConfig {

    @Bean
    public Queue queue(){
        //设置该队列信息的超时时间为10s，超过10秒没被消费的消息将被投递到死信队列dl_queue中(通过dl_exchange)
        HashMap<String, Object> map = new HashMap<>();
        map.put("x-message-ttl",10000);
        map.put("x-dead-letter-exchange","dl_exchange");
        map.put("x-dead-letter-routing-key","dl_key");
        return new Queue("queue",true,false,false,map);
    }

    @Bean
    public Exchange exchange(){
        return new DirectExchange("exchange");
    }

    @Bean
    public Binding binding(Queue queue,Exchange exchange){
        return BindingBuilder.bind(queue) .to(exchange).with("key").noargs();
    }

    @Bean
    public Queue dl_queue(){
        return new Queue("dl_queue",true,false,false,null);
    }

    @Bean
    public Exchange dl_exchange(){
        return new DirectExchange("dl_exchange");
    }

    @Bean
    public Binding dl_binding(){
        return BindingBuilder.bind(dl_queue()).to(dl_exchange()).with("dl_key").noargs();
    }
}
~~~

## 自定义RabbitTemplate

springboot提供的rabbitTemplate没有开启confirm模式，想要生产者开启confirm模式，需要自定义RabbitTemplate

实例：

* springboot核心配置文件：

~~~yaml
spring:
    rabbitmq:
        host: 192.168.134.128
        port: 5672
        username: guest
        password: guest
~~~

需要将上面配置读取到java对象中

* 读取配置：

~~~java
@ConfigurationProperties(prefix = "spring.rabbitmq")
@Component
@Data
@NoArgsConstructor
@AllArgsConstructor
public class RabbitMQSource {
    private String host;
    private int port;
    private String username;
    private String password;
}
~~~

* 自定义连接工厂

~~~java
@Bean
public ConnectionFactory connectionFactory(RabbitMQSource source){
    com.rabbitmq.client.ConnectionFactory connectionFactory = new com.rabbitmq.client.ConnectionFactory();
    connectionFactory.setHost(source.getHost());
    connectionFactory.setPort(source.getPort());
    connectionFactory.setUsername(source.getUsername());
    connectionFactory.setPassword(source.getPassword());
    CachingConnectionFactory cachingConnectionFactory = new CachingConnectionFactory(connectionFactory);
    //默认的连接工厂是不支持rabbitTemplate的ConfirmCallback和ReturnsCallback功能的，需要手动打开
    cachingConnectionFactory.setPublisherConfirmType(CachingConnectionFactory.ConfirmType.SIMPLE);
    cachingConnectionFactory.setPublisherReturns(true);
    return cachingConnectionFactory;
}
~~~

* 自定义RabbitTemplate

~~~java
@Bean
public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory){
    RabbitTemplate rabbitTemplate = new RabbitTemplate(connectionFactory);
    //setMandatory交换器无法根据自身类型和路由键找到一个符合条件的队列时的处理方式
	//true：RabbitMQ会调用Basic.Return命令将消息返回给生产者
	//false：RabbitMQ会把消息直接丢弃
    rabbitTemplate.setMandatory(true);
	//投递成功将调用该函数
    rabbitTemplate.setConfirmCallback((correlationData, ack, cause) ->
                                      System.out.println("消息到达路由"));
    //投递失败的消息将传入该回调函数
    rabbitTemplate.setReturnsCallback(returned ->
                                      System.out.println("消息投递失败:"+new String(returned.getMessage().getBody())));
    return rabbitTemplate;
}
~~~
