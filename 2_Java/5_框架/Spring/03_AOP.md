# AOP

AOP面向对象编程(Aspect-Oriented Programming)是对 OOP面向对象编程(Object-Oriented Programming)的强有力的补充

## AOP理论

软件开发的目的，最终是为了解决各种需求，包括业务需求和系统需求

* 使用OOP面向对象的方法，可以很好的解决业务需求：

  面向对象编程能很好的对业务需求进行抽象、封装和模块化，因为业务需求和具体实现之间的关系基本上是一对一的。所以使用面向对象编程可以很好的开发和维护

* 但是对于系统需求，OOP就不能很好的应对了：

  系统需求和实现基本上是一对多的：像是日记记录、安全检查等系统需求，系统中的每个业务对象都需要加入日志记录、安全检查；

  如果使用面向对象的方式实现并集成到系统中，那么需求的实现代码就会遍及每个业务对象；随着业务对象的增加，系统需求的实现会在系统中各处散落，极大的增加了开发和维护的难度

面对OOP开发的困境，AOP应运而生：

AOP能将日记记录、安全检查、事务管理等系统需求（这样的需求在AOP中就叫做横切关注点 cross-cutting concern） 以Aspect的方式进行封装模块化

然后把以Class形式模块化的业务需求和以Aspect形式模块化的系统需求通过织入的方式

拼接在一起，整个系统就算完成了

![153308_0Xp8_220508.png](https://gitee.com/wangziming707/note-pic/raw/master/img/AOP%E7%A4%BA%E6%84%8F.png)



## AOP实现

目前AOP Aspect的实现还是基于OOP之上的，AOP的各个概念实体，最终都需要以某中方式集成到OOP中 ，将AOP组件集成到OOP组件的过程，在AOP中称为织入(Weave)过程

将业务需求和系统需求通过OOP和AOP以模块化的形式开发完成，通过织入的方式就能完成系统的集成和使用

具体织入的方式分为静态和动态两种：

### 静态AOP

第一代AOP

相应的横切关注点以Aspect实现后，会通过特定的编译器，直接将实现后的Aspect编译并织入到系统的静态类中。

静态AOP的Aspect直接以字节码的形式编译到类中，不会对系统的运行造成任何性能损失

缺点是不够灵活，如果横切关注点要改变织入到系统的位置，就需要重新求改Aspect定义未见，然后使用编译器重新编译Aspect并重新织入到系统中

### 动态AOP

第二代AOP， 大都通过Java语言提供的动态特性来实现Aspect织入到当前系统的过程AOP的各种概念实体全部都是普通的Java类

AOP的织入过程是在系统运行开始之后进行，而不是预先编译到系统类中，并且织入信息大都采用外部XML文件格式保存，可以在调整织入点以及织入逻辑单元的同时，不必变更系统其他模块，甚至在系统运行时，动态的更改织入逻辑

因为动态AOP采用对系统字节码进行操作的方式完成Aspect到系统的织入，所以难免会造成一定的运行时性能损失

## Java中的AOP

在Java平台上可以使用过多种方式实现AOP

### 动态代理

JDK 拥有动态代理(Dynamic Proxy)机制，可以在运行期间，为相应的接口(Interface)动态生成对应的代理对象。

所以，我们可以将横切关注点逻辑封装到动态代理的 InvocationHandler中，然后在系统运行期间，根据横切关注点需要织入的模块位置，将横切逻辑织入到相应的代理类中。

这样，以动态代理为载体的横切逻辑，现在就可以与系统的其他实现模块一起工作了

用动态代理的方式实现AOP，需要织入横切关注点逻辑的模块类都得实现相应的接口，因为动态代理机制只针对接口有效

Spring AOP 默认情况下采用这种机制实现AOP功能

### 动态字节码增强

Java虚拟机启动时会通过ClassLoader将class文件加载到程序中，class文件是 java源码通过Javac编译器编译生成的

但只要符合Java class 规范，我们也可以使用CGLB等Java 工具库，在程序运行时，动态构建字节码的class文件，并加载使用

这就是动态字节码增强技术

所有我们可以使用这个技术，为模块类生成相应的子类，并将横切逻辑加到这些子类中

让应用程序在执行期间使用的时这些动态生成的子类，从而达到将横切逻辑织入系统的目的

使用动态字节码增强技术，即使模块类没有实现相应的接口，我们仍然可以对其进行拓展

但是，如果需要拓展的类以及类中的实例方法等声明为final的话，就无法对其进行子类化的拓展了

Spring AOP 在无法采用动态代理机制进行AOP功能扩展时，会使用CGLIB库的动态字节码增强支持来实现AOP

### 自定义类加载器

Java 程序的class 都是通过相应的类加载器（ClassLoader）加载到Java虚拟机后才运行的。默认的类加载器会读取class字节码文件，然后解析并加载这些class文件到虚拟机运行。

那么我们可以通过自定义类加载器的方式完成横切逻辑到系统的织入，自定义类加载器通过读取外部文件规定的织入规则和必要信息，在今安在class文件期间，将横切逻辑添加到模块类的现有逻辑中，将改动后的class交给Java虚拟机运行

## AOP组成概念

### Joinpoint

在系统运行之前，AOP的功能模块都需要织入到OOP的功能模块中。所以，要进行这样的织入过程，我们需要知道在系统的哪些执行点上进行织入操作

这些将要在其之上进行织入操作的系统执行点就是Joinpoint

只要允许，程序执行过程中的任何时点都可以称为横切逻辑的织入点，所有这些执行时点都是Joinpoint

下面时较为常用的Joinpoint类型：

* 方法调用(Method Call) ：方法被调用时所处的程序执行点，是在调用对象上的执行点
* 方法执行(Method Call execution)：方法内部执行开始时点，是在被调用到的方法逻辑执行的时点
* 构造方法调用(Constructor Call):对象的构造方法被调用进行初始化的时点
* 构造方法执行(Constructor Call Execution):对象的构造方法内部执行的开始时点
* 字段设置(Field Set):对象的属性通过setter方法被设置或者直接被设置的时点
* 字段获取(Field Get):对象的属性通过getter方法被获取或者直接被获取的时点
* 异常处理执行(Exception Handler Execution):异常抛出后，对应的异常处理逻辑执行的时点
* 类初始化(Class initialization):类中静态类型或者静态代码块初始化的时点

### Pointcut

Pointcut概念表示的是Joinpoint的表述方式。可以看作一个或多个Joinpoint的集合

在将横切逻辑织入当前系统的过程中，需要参照Pointcut规定的Joinpoint信息，才能知道该往系统的哪些Joinpoint上织入横切逻辑

#### Pointcut表述方式







#### Pointcut运算





# Spring AOP



























## AsperctJ实现AOP

Spring整合了Asperct框架以实现AOP

### maven依赖

~~~xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-aspects</artifactId>
    <version>5.2.5.RELEASE</version>
</dependency>
~~~

### 通知类型

* 前置通知 `@Before`
* 后置通知`@After`
* 环绕通知`@Around`
* 异常通知`@AfterThrowing`
* 最终通知`@AfterReturning`

### 切入表达式

AspectJ 用切入表达式的形式指定切入点

表达式形式为:

`execution(modifiers-pattern? ret-type-pattern declariong-type-pattern? name-pattern(param-pattern) throws-pattern?)`

* `?`表示可选匹配
* `modifiers-pattern`匹配访问权限类型
* `ret-type-pattern`匹配返回值类型
* `declaring-type-pattern`匹配包名类名
* `name-pattern(param-pattern)`匹配方法名(参数)
* `throws-pattern`匹配异常

以上匹配可以由符号表示含义：

* `*`

  0至多个任意字符

* `..`

  * 在方法参数中，表示任意多个参数
  * 在包名后，表示当前包及其子包路径

* `+`

  * 在类名后，表示当前类及其子类
  * 在接口后，表示当前接口及其实现类

示例：

~~~java
举例：
execution(public * *(..)) 
指定切入点为：任意公共方法。
execution(* set*(..)) 
指定切入点为：任何一个以“set”开始的方法。
execution(* com.xyz.service.*.*(..)) 
指定切入点为：定义在 service 包里的任意类的任意方法。
execution(* com.xyz.service..*.*(..))
指定切入点为：定义在 service 包或者子包里的任意类的任意方法。“..”出现在类名中时，后
面必须跟“*”，
表示包、子包下的所有类。
execution(* *..service.*.*(..))
指定所有包下的 serivce 子包下所有类（接口）中所有方法为切入点
execution(public void com.bjpn.service.UserService.addUser(..))
~~~



### 非注解方式实现AOP

* 目标类：

~~~java
@Component
public class ATM {
    public void addMoney(){
        System.out.println("存储");
    }
}
~~~

* 增强类：

~~~java
@Component
public class Log {
    public void operationLog(){
        System.out.println("记录操作");
    }
    public void balanceLog(){
        System.out.println("记录余额");
    }
}
~~~

* 配置：

要在存款操作前记录操作，在存款操作后记录余额

~~~xml
<!--配置AOP-->
<aop:config>
    <!--配置切面-->
    <!--表示这个切面的增强类是log-->
    <aop:aspect ref="log">
        <!--在切面上配置切点-->
        <aop:pointcut id="addMoney" expression="execution(* com.bjpn.bean.*.addMoney(..))"/>
        <!--设置通知，在指定切点以指定通知方式调用指定方法-->
        <aop:before method="operationLog" pointcut-ref="addMoney"/>
        <aop:after method="balanceLog" pointcut-ref="addMoney"/>
    </aop:aspect>
</aop:config>
~~~

### 注解方式实现AOP

* 配置

~~~xml
<!--开启AOP注解-->
<aop:aspectj-autoproxy/>
~~~



* 目标类

~~~java
@Component("atm")
public class ATM {
    public void addMoney(){
        System.out.println("存储");
    }
}
~~~

* 增强类

~~~java
@Component
@Aspect
public class Log {
	//设置切点，需要再定义一个方法，当做切点的名称
    @Pointcut("execution(* com.bjpn.bean.*.addMoney(..))")
    public void addMoneyP(){}

    @After("addMoneyP()")
    public void operationLog(){
        System.out.println("记录操作");
    }
    @Before("addMoneyP()")
    public void balanceLog(){
        System.out.println("记录余额");
    }
}
~~~

