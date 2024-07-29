# AOP概述

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

缺点是不够灵活，如果横切关注点要改变织入到系统的位置，就需要重新修改Aspect定义文件，然后使用编译器重新编译Aspect并重新织入到系统中

### 动态AOP

第二代AOP， 大都通过Java语言提供的动态特性来实现Aspect织入到当前系统的过程AOP的各种概念实体全部都是普通的Java类

AOP的织入过程是在系统运行开始之后进行，而不是预先编译到系统类中，并且织入信息大都采用外部XML文件格式保存，可以在调整织入点以及织入逻辑单元的同时，不必变更系统其他模块，甚至在系统运行时，动态的更改织入逻辑

因为动态AOP采用对系统字节码进行操作的方式完成Aspect到系统的织入，所以难免会造成一定的运行时性能损失

## Java实现AOP

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

![AOP各个概念示意图](https://gitee.com/wangziming707/note-pic/raw/master/img/AOP%E5%90%84%E4%B8%AA%E6%A6%82%E5%BF%B5%E7%A4%BA%E6%84%8F%E5%9B%BE.png)

### Joinpoint

在系统运行之前，AOP的功能模块都需要织入到OOP的功能模块中。所以，要进行这样的织入过程，我们需要知道在系统的哪些执行点上进行织入操作

这些将要在其之上进行织入操作的系统执行点就是Joinpoint

只要允许，程序执行过程中的任何时点都可以称为横切逻辑的织入点，所有这些执行时点都是Joinpoint

下面是较为常用的Joinpoint类型：

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

当前的AOP产品所使用的Pointcut表达形式通常可以简单划分为以下几种：

* 直接指定Joinpoint所在方法名称：这种 Pointcut 表述方式比较简单，而且功能单一，通常只限于支持方法级别Joinpoint的AOP框架、只针对方法调用类型的Joinpoint、只针对方法执行类型的Joinpoint

  这种表述方式通常只限于Joinpoint较少且比较简单的情况

* 正则表达式：是比较普遍的Pointcut表达方式，可以充分利用正则表达式的强大功能，来归纳表述需要符合二某种条件的多组Joinpoint；Spring AOP 就是使用这种表达方式

* 使用特定的Pointcut表述语言：最为强大的表达Pointcut的方式，灵活性也很好，但具体实现复杂，AspectJ使用这种方式来指定Pointcut

#### Pointcut运算

Pointcut可以看作Joinpoint的集合，所以可以进行类似集合的运算

可以使用and 或者 or 等逻辑运算 进行并 交运算

### Advice

 Advice是单一横切关注点逻辑的载体，它代表将会织入到Joinpoint的横切逻辑

Advice之于Aspect，就像 Mehtod之于Class

按照Advice在Joinpoint位置执行时机的差异或者完成功能的不同，Advice可以分成多种具体的形式

* Before Advice ：在Joinpoint指定位置之前执行的Advice类型

* After Advice：在Joinpoint指定位置之后执行的Advice，该类型的Advice可以细分为以下三种：

  * After returning Advice :当前Joinpoint执行流程正常完成后才会执行
  * After throwing Advice :当前Joinpoint执行过程中抛出异常的情况下才会执行
  * After Adive:不管Joinpoint处执行流程是正常结束，还是抛出异常都会执行

* Around Advice ：对附加其上的Joinpoint进行包裹，可以在Joinpoint之前和之后都指定相应的逻辑

* Introduction : 与之前的几种Advice不同，它不是根据横切逻辑在Joinpoint处的执行时机来区分的，而是根据它可以完成的功能而区别与其他Advice类型

  Introduction可以为原有的对象添加新的特性或者行为

### Aspect

Aspect是对系统中的横切关注点逻辑进行模块化封装的AOP概念实体。

通常情况下，Aspect可以包含多个Pointcut以及相关Advice定义。

### Weaving&Weaver

AOP的横切逻辑组织完成后，需要通过织入器(Weaver)的织入(Weaving)过程，才能将AOP集成到现有业务系统上

织入器的职责就是完成横切关注点到系统的最终织入

AspectJ通过编译器ajc完成织入操作，那么ajc就是AspectJ的织入器

Spring AOP使用一组类来完成最终的织入操作，ProxyFactory类则是Spring AOP中最通用的织入器

### Target

符合Pointcut所指定的条件，将在织入过程中被织入横切逻辑的对象，称为目标对象(Target)