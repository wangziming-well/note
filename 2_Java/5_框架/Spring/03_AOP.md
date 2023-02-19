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

* Introduction : 与之前的几种Advice不同，它不是根据横切逻辑在Joinpoint出的执行时机来区分的，而是根据它可以完成的功能而区别与其他Advice类型

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



# Spring AOP原理

Spring AOP是Spring核心框架的重要组成部分，与Spring IOC容器和Spring框架对其他JavaEE服务的集成 构成的Spring框架的核心

Spring AOP采用 Java作为AOP的实现语言(AOL)，在Java语言的基础之上，Spring AOP对AOP的概念进行了适当的抽象和实现

Spring AOP属于动态AOP采用动态代理机制和字节码生成技术实现。而动态代理是通过代理模式实现的(Proxy Pattern)

## 代理模式

在代理机制中：代理处于请求者和被请求者之间，可以隔绝两者直接的直接交互，代理者全权代理被请求者，拥有它的全部职能

实现了代理机制的设计模式，就叫代理模式，在代理模式中，通常涉及4种角色：

![代理模式](https://gitee.com/wangziming707/note-pic/raw/master/img/%E4%BB%A3%E7%90%86%E6%A8%A1%E5%BC%8F.png)

* ISubject: 该几口是对被访问者或者访问者资源的抽象
* RealSubject:被访问者或者被访问资源的具体实现类
* ProxySubject:被访问者或者被访问资源的代理实现类，该类持有ISubject接口的一个具体实例
* Client:代表访问者的抽象角色，Client将会访问ISubject类型的对象或者资源，再代理场景中，Client无法直接访问RealSubject获取资源，而是通过ProxySubject

在代理模式中，ProxySubject常常在RealSubject提供的资源基础上添加一些逻辑功能

在AOP中，Target就是 RealSubject，在为这个Target创建代理对象时，可以将横切逻辑添加到这个代理对象中。

但是，这样这样静态代理的方式，即使Joinpoint相同，如果对应的Target类型不同，那么就需要针对的所有的Target类型，创建对应的代理对象，但是实际上，这些代理对象所要添加的横切逻辑时一样的，

## 动态代理

动态代理能够解决静态代理实现AOP出现的问题

JDK提供了一种动态代理的规范：`java.lang.reflect.Proxy`类和`java.lang.reflect.InvocationHandler`接口

可以通过Proxy类的newProxyInstance方法获取代理对象实例，它的方法签名如下：

~~~java
public static Object newProxyInstance(ClassLoader loader,Class<?>[] interfaces,InvocationHandler h)
~~~

需要传入RealSubject的ClassLoader,和RealSubject的接口类定义和RealSubject的InvocationHandler

InvocationHandler定义如下：

~~~java
public interface InvocationHandler {

    public Object invoke(Object proxy, Method method, Object[] args)
        throws Throwable;
}
~~~

在invoke方法中调用被代理类的实际资源方法，并进行一些额外的处理

下面是实例：

定义被代理对象和接口：

~~~java
//subject接口
public interface ITarget {
    void request();
}
//subject实现类
public class Target implements ITarget {
    public void request(){
        System.out.println("request方法执行");
    }
}
~~~

定义ITarget的InvocationHandler

~~~java
public class MyInvocationHandler implements InvocationHandler {

    private final ITarget target;

    public MyInvocationHandler(ITarget target) {
        this.target = target;
    }

    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        if (method.getName().equals("request")){
            System.out.println("在方法调用前执行本句子");
            Object invoke = method.invoke(target, args);
            System.out.println("在方法调用后执行本句子");
            return invoke;
        }
        return null;
    }
}
~~~

生成代理对象：

~~~java
Target target = new Target();
ITarget iTarget =(ITarget) Proxy.newProxyInstance(
    target.getClass().getClassLoader(),
    new Class[]{ITarget.class},
    new MyInvocationHandler(target)
);
iTarget.request();
~~~

这样通过动态代理的方式，就实现了AOP

InvocationHandler是实现横切逻辑的地方，是AOP中的Advice

该种方式实现的AOP要求目标类型必须实现相应的Interface

Spring AOP 默认使用动态代理的方式生成代理对象实现AOP

如果目标对象没有Interface Spring AOP会尝试使用CGLIB(Code Generation Library)的动态字节码生成类库，为目标对象生成动态的代理对象实例

## 动态字节码生成

使用动态字节码生成技术扩展对象行为的原理是：我们可以对目标对象进行继承，生成相应的子类，子类通过覆写来扩展父类的行为。

只要将横切逻辑放入到子类中，然后让系统使用扩展后的子类，就能达到织入的效果了。

借助CGLIB动态字节码生成库，可以在系统允许期间动态得为目标对象生成相应的扩展子类

具体的可以通过CGLIB提供的Enhancer 类和 MethodInterceptor接口实现代理子类的生成

MethodInterceptor接口定义：

~~~java
//用以增强目标类
public interface MethodInterceptor
extends Callback
{

    public Object intercept(Object obj, java.lang.reflect.Method method, Object[] args,MethodProxy proxy) throws Throwable;

}
~~~

参数说明：

* obj:cglib生成的代理对象
* method:被代理对象的方法
* atgs:传入方法的参数
* proxy:代理的的方法

Enhancer 提供了下面方法以增强类：

~~~java
//设置将要被继承生成子类的被增强类
void setSuperclass(Class superclass);
//设置回调函数，用以增强子类的方法
void setCallback(final Callback callback);
//生成子类
Object create();
~~~

**实例：**

* 目标类(被增强类)：

~~~java
public class Target {
    public void print(){
        System.out.println("执行print函数");
    }
}
~~~

* 定义回调:

~~~java
public class PrintCallable implements MethodInterceptor {
    public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
        if (method.getName().equals("print")){
            System.out.println("方法执行前逻辑");
            Object o = proxy.invokeSuper(obj, args);
            System.out.println("方法执行后逻辑");
            return o;
        }
        return null;
    }
}
~~~

* 实现增强

~~~java
Enhancer enhancer = new Enhancer();
enhancer.setSuperclass(Target.class);
enhancer.setCallback(new PrintCallable());
Target target =(Target) enhancer.create();
target.print();
~~~



# SpringAOP概念实体

## Joinpoint

通过之前描述的SpringAOP的实现方式，我们可以看出SpringAOP只支持方法执行类型的Joinpoint，但这样已经能够应付大部分的需求了，如果想要更强大的AOP支持，可以使用AspectJ之类的AOP产品

## Pointcut

Spring中定义了接口`Pointcut`作为其AOP框架中所有PointCut的最顶级抽象，其定义如下：

~~~java
public interface Pointcut {

	ClassFilter getClassFilter();

	MethodMatcher getMethodMatcher();

	Pointcut TRUE = TruePointcut.INSTANCE;

}
~~~

定义了两个方法用来捕获系统中的Pointcut

并提供了一个`TruePointcut`实例，如果`Pointcut`类型为`TruePointcut`，默认会对系统中的所有对象和对象上的所有方法进行匹配。

### ClassFilter

ClassFilter接口的作用是对Joinpoint所处的对象进行Class级别的匹配，其定义如下：

~~~java
@FunctionalInterface
public interface ClassFilter {

	boolean matches(Class<?> clazz);

	ClassFilter TRUE = TrueClassFilter.INSTANCE;

}
~~~

matches 传入一个Class，当方法返回true时，表示该Class匹配Pointcut所规定的类型

### MethodMatcher

MethodMatcher对Joinpoint所处的方法进行匹配，定义如下：

~~~java
public interface MethodMatcher {

	boolean matches(Method method, Class<?> targetClass);

	boolean isRuntime();

	boolean matches(Method method, Class<?> targetClass, Object... args);

	MethodMatcher TRUE = TrueMethodMatcher.INSTANCE;

}
~~~

MethodMatcher 重载定义了两个matches方法，他们的去呗就是是否会对方法参数进行校验

根据isRuntime的返回值情况，MethodMatcher有两大不同的实现抽象类：

#### StaticMethodMatcher

StaticMethodMatcher定义如下：

~~~java
public abstract class StaticMethodMatcher implements MethodMatcher {
	@Override
	public final boolean isRuntime() {
		return false;
	}

	@Override
	public final boolean matches(Method method, Class<?> targetClass, Object... args) {
		// should never be invoked because isRuntime() returns false
		throw new UnsupportedOperationException("Illegal MethodMatcher usage");
	}
}
~~~

它覆写的`isRuntime()`方法，固定返回`false`

当`isRuntime`返回`false`时，MethodMatcher不会对方法参数进行校验，所以该类实现了方法`matches(Method method, Class<?> targetClass, Object... args)`如果调用了验证参数的matches方法，就直接抛出`UnsupportedOperationException`

#### DynamicMethodMatcher

DynamicMethodMatcher定义如下：

~~~java
public abstract class DynamicMethodMatcher implements MethodMatcher {
	@Override
	public final boolean isRuntime() {
		return true;
	}

	@Override
	public boolean matches(Method method, Class<?> targetClass) {
		return true;
	}
}
~~~

它覆写的`isRuntime()`方法，固定返回`true`

当`isRuntime`返回`true`时，进行方法匹配时：

* 首先调用`matches(Method method, Class<?> targetClass)`进行不校验参数的匹配，如果返回false则不匹配
* 如果上一步返回true，则在调用`boolean matches(Method method, Class<?> targetClass, Object... args)`方法进行校验参数的匹配，如果返回true，才算匹配

所以该方法覆写了`matches(Method method, Class<?> targetClass) `方法，让其默认返回true，这样当调用DynamicMethodMatcher进行匹配时，会直接调用进行校验参数的 matches方法

### 常用Pointcut

spring提供了许多Pointcut实现，以供满足不同的匹配需求，以下给出常用的四线

* `NameMatchMethodPointcut`通过方法名进行匹配

  除了可以指定方法名，还可以使用`*`通配符进行简单的模糊匹配

* `JdkRegexpMethodPointcut`通过正则表达式匹配

  匹配的是方法的方法签名，而不是单单方法名

* `AnnotationMatchingPointcut`通过注解匹配

  拥有指定注解的方法和类，如果是类，匹配类下所有方法

* `ComposablePointcut`可以提供pointcut逻辑运算，通过union和intersection进行交并运算，也可以通过，Pointcuts工具类
* `ControlFlowPointcut`可以匹配被指定类调用的指定方法

## Advice

String提供了Advice接口作为AOP Advice的顶级抽象，Advice实现了将被织入到Pointcut规定的Joinpoint处的横切逻辑。在Spring中，Advice按照自身实例(instance)能否在目标对象类的所有实例中共享这一标准，可以划分为：per-class类型的Advice和per-instance类型的Advice

### per-class

per-class类型的Advice是指：该类型的Advice的实例可以在目标对象类的所有实例之间共享

这种类型的Advice通常只是提供方法拦截的功能，不会为目标对象保持任何状态或者添加新的特性

per-class类型的Advice类型接口的继承关系如下：

![perClassAdvice](https://gitee.com/wangziming707/note-pic/raw/master/img/perClassAdvice.png)



#### MethodBeforeAdvice

Before Advice所实现的横切逻辑将在相应的Joinpoint之前执行

Spring提供的`BeforeAdvice`是标志接口，没有定义任何方法，要在Spring中实现Before Advice,可以实现`MethodBeforeAdvice`接口，其定义如下：

~~~java
public interface MethodBeforeAdvice extends BeforeAdvice {

	void before(Method method, Object[] args, @Nullable Object target) throws Throwable;

}
~~~

在before方法中提供横切逻辑

#### `ThrowsAdvice`

String 中定义的`ThrowsAdvice `对应的 AOP概念中的AfterThrowingAdvice

该接口没有定义任何方法，是标志接口，但在实现该接口时，方法定义要遵循如下规则：

~~~java
public void afterThrowing([Method method, Object[] args,Object target,] Exception ex)
~~~

前三个参数可以省略，可以根据拦截的`Throwable`的不同类型，重载定义多个`afterThrowing`方法，

框架将会使用Java 反射机制调用这些方法

#### `AfterReturningAdvice`

`AfterReturningAdvice`接口定义如下：

~~~java
public interface AfterReturningAdvice extends AfterAdvice {

	void afterReturning(@Nullable Object returnValue, Method method, Object[] args, @Nullable Object target) throws Throwable;

}
~~~

通过`AfterReturningAdvice`我们可以访问当前Joinpoint的方法返回值、方法、方法参数以及所在的目标对象。

#### MethodInterceptor

Spring并没有直接定义对应的Around Advice接口，而是使用了AOP Alliance的标准接口`MethodInterceptor`:

~~~java
public interface MethodInterceptor extends Interceptor {
	Object invoke(MethodInvocation invocation) throws Throwable;
}
~~~

通过 `MethodInterceptor.invoke`方法的`MethodInvocation`参数，我们可以控制相应Joinpoint的拦截行为：

通过调用`MethodInvocation`的`proceed()`方法，可以让程序执行继续沿着调用链传播。如果在invoke方法中没有调用`proceed()`方法，程序将在当前`MethodInterceptor`处停止

我们可以在`proceed()`方法(也就是Joinpoint处的逻辑)执行前后插入相应的逻辑，甚至捕获`proceed()`方法可能抛出的异常，这就是它能履行Around Advice职责的原因

### per-instance

per-instance类型的Advice不会在目标类的所有实例之间共享，会为不同的实例对象保存它们各自的状态和相关逻辑

在Spring AOP中，Introduction是唯一的per-instance型Advice

Introduction 可以在不改动目标类定义的情况下，为目标类添加新的属性和行为

在Spring中，为目标对象添加新的属性和行为必须声明相应的接口以及相应的实现。这样，再通过特定的拦截器将新的接口定义以及实现类中的逻辑附加到目标对象之上。

#### IntroductionInterceptor

这个特定的拦截器就是`IntroductionInterceptor`,其定义如下：

~~~java
public interface IntroductionInterceptor extends MethodInterceptor, DynamicIntroductionAdvice {
}
~~~

它的继承关系如下：

![IntroductionInterceptor](https://gitee.com/wangziming707/note-pic/raw/master/img/IntroductionInterceptor.png)

它继承了`MethodInterceptor`和`DynamicIntroductionAdvice`

`DynamicIntroductionAdvice`的定义如下：

~~~java
public interface DynamicIntroductionAdvice extends Advice {
	boolean implementsInterface(Class<?> intf);
}
~~~

通过`DynamicIntroductionAdvice`，可以界定当前的`IntroductionInterceptor`为那些接口提供相应的拦截功能。

通过`MethodInterceptor`,`IntroductionInterceptor`可以处理新添加的接口上的方法调用

`IntroductionInterceptor`有两个具体的实现类：

* `DelegatingIntroductionInterceptor`
* `DelegatePerTargetObjectIntroductionInterceptor`

#### DelegatingIntroductionInterceptor

`DelegatingIntroductionInterceptor`不会自己实现将要添加到目标对象上的新的逻辑行为，而是委派(delegate)给其他实现类

具体实例如下：

~~~java
Tester tester = new Tester();
        DelegatingIntroductionInterceptor advice = new DelegatingIntroductionInterceptor(tester);
~~~

将`tester`交给`DelegatingIntroductionInterceptor`,这样在织入时，会将tester的方法织入到目标对象中

#### DelegatePerTargetObjectIntroductionInterceptor

`DelegatePerTargetObjectIntroductionInterceptor`会在内部持有一个目标对象与相应Introduction逻辑实现类之间的映射关系。

当每个目标对象上的新定义的接口方法被调用的时候`DelegatePerTargetObjectIntroductionInterceptor`会拦截这些调用，然后找到映射关系中目标对象实例对应的Introduction实现类实例。

所以不需要给`DelegatePerTargetObjectIntroductionInterceptor`提供delegate接口实例，只需要delegete接口类型和对应的实现类的类型就可以了

具体使用实例如下：

~~~java
DelegatePerTargetObjectIntroductionInterceptor advice = new DelegatePerTargetObjectIntroductionInterceptor(Tester.class, ITester.class);
~~~

## Aspect

AOP理论中的Aspect定义中可以有多个Pointcut和多个Advice

而spring提供的Aspect：Advisor通常只持有一个Pointcut和一个Advice

所以Advisor是一种特殊的Aspect

Advisor可以划分为两类：

![Advisor](https://gitee.com/wangziming707/note-pic/raw/master/img/Advisor.png)

### PointcutAdvisor

PointcutAdvisor体系的继承关系如下：

![PointcutAdvisor](https://gitee.com/wangziming707/note-pic/raw/master/img/PointcutAdvisor.png)

* `DefaultPointcutAdvisor`是最通用的`PointcutAdvisor`,除了不能指定Introduction类型的Advice外，剩下任何类型的Pointcut和Advice都可以通过它来使用,

    其内部维护了一个Pointcut类型属性和一个Advice类型属性，可以通过构造函数为其赋值，也可以通过setter方法赋值

* `NameMatchMethodPointcutAdvisor`,它限定了内部可以使用的Pointcut类型为`NameMatchMethodPointcut`,且不可更改，Advice仍然除了Introduction外其他类型都可以

* `RegexpMethodPointcutAdvisor`,它限定了内部可以使用的Pointcut类型为`AbstractRegexpMethodPointcut`,且不可更改，Advice仍然除了Introduction外其他类型都可以

除此之外还有一个比较少用的`DefaultBeanFactoryPointcutAdvisor`,它实现了`BeanFactoryAware`接口，所以它必须被注册在`BeanFactory`容器中，它会自动通过容器中的Advice注册的beanName，来关联对应的Advice

### IntroductionAdvisor

`IntroductionAdvisor`仅限于Introduction的使用场景

`IntroductionAdvisor`体系比较简单，只有一个`DefaultIntroductionAdvisor`

![DefaultIntroductionAdvisor](https://gitee.com/wangziming707/note-pic/raw/master/img/DefaultIntroductionAdvisor.png)

`DefaultIntroductionAdvisor`只可以指定`Introduction`型的Advice(`IntroductionInterceptor`)以及被拦截的接口类型

### Ordered

在之前的继承关系图中可以看出，所以的Advisor实现类都直接或者间接的实现了Ordered接口

Ordered接口用以处理同一Joinpoint处有多个Advisor的情况

Ordered用以指定此时的执行优先级

如果没有指定优先级，默认按照它们的声明顺序来应用它们

## Weaver

SpringAOP提供的weaver都继承了`ProxyCreatorSupport`：

![ProxyCreatorSupport](https://gitee.com/wangziming707/note-pic/raw/master/img/ProxyCreatorSupport.png)

### `ProxyCreatorSupport`

Spring AOP 提供的织入器都会继承`ProxyCreatorSupport`，获取它提供的AOP织入的公用逻辑支持

* `ProxyCreatorSupport`继承了`AdvisedSupport`，它提供了生成代理的必要信息
* `ProxyCreatorSupport`内部持有了`AopProxyFactory`，通过它获取`AopProxy`；`AopProxy`用于创建对象

#### `AopProxy`

Spring AOP框架内使用AopProxy对使用不同的代理实现机制进行了适度的抽象，

AopProxy的定义如下:

~~~java
public interface AopProxy {
	Object getProxy();

	Object getProxy(@Nullable ClassLoader classLoader);
}
~~~

它的体系继承关系如下：

![AopProxy](https://gitee.com/wangziming707/note-pic/raw/master/img/AopProxy.png)

##### `AopProxyFactory`

一般通过AopProxyFactory获取AopProxy实例，AopProxyFactory定义如下：

~~~java
public interface AopProxyFactory {
	AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException;
}
~~~

它只有一个实现类DefaultAopProxyFactory，其生成逻辑如下：

~~~java
if (config.isPotimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces){
    //创建Cglib2AopProsxy实例并返回
} else{
    //创建JdkDynamicAopProxy实例并返回
}
~~~

#### `AdvisedSupport`

`ProxyCreatorSupport`继承自`AdvisedSupport`，并且`AopProxyFactory`根据传入的`AdvisedSupport `来判断生成 `AopProxy`实例

`AdvisedSupport`提供了生成代理对象的必要信息

`AdvisedSupport`实现了两个接口：

* `ProxyConfig`:记载生成代理对象的控制信息
* `Advised`:供生成代理对象的具体信息

##### `ProxyConfig`

`ProxyConfig`记载生成代理对象的控制信息，它定义了5个Boolean型的属性。分别控制在生成代理对象的时候，应该采取哪些措施：

* proxyTargetClass：若为true，ProxyFactory会使用CGLIB对目标对象进行代理，默认值为false

* optimize：主要用于告知代理对象是否需要采取进一步的优化措施，如代理对象生成之后，即使为其添加或移除了相应的Advice，代理对象也可以忽略这种变动。

  当该属性为true时，ProxyFactory会使用CGLIB进行代理对象的生成。默认值为false

* opaque:用于控制生成的代理对象是否可以强制转型为Advised，默认值为false，表示可以强制转型为Advised，我们可以通过Advised查询代理对象的一些状态

* exposeProxy:若为true，可以让SpringAOP在生成代理对象时，将当前代理对象绑定到ThreadLocal。如果目标对象需要访问当前代理对象，可以通过AopContext.currentProxy()取得，处于性能考虑，默认值false

* frozen:若为true，当针对代理对象生成的各项信息配置完成后，则不允许修改

  默认值为false

##### `Advised`

`Advised`提供生成代理对象的具体信息，默认情况下，SpringAOP提供的代理对象都可以强制转化为Advised类型，其定义如下：

~~~java
public interface Advised extends TargetClassAware {
	boolean isFrozen();

	boolean isProxyTargetClass();

	Class<?>[] getProxiedInterfaces();

	boolean isInterfaceProxied(Class<?> intf);

	void setTargetSource(TargetSource targetSource);

	TargetSource getTargetSource();

	void setExposeProxy(boolean exposeProxy);

	boolean isExposeProxy();

	void setPreFiltered(boolean preFiltered);

	boolean isPreFiltered();

	Advisor[] getAdvisors();

	void addAdvisor(Advisor advisor) throws AopConfigException;

	void addAdvisor(int pos, Advisor advisor) throws AopConfigException;

	boolean removeAdvisor(Advisor advisor);

	void removeAdvisor(int index) throws AopConfigException;

	int indexOf(Advisor advisor);

	boolean replaceAdvisor(Advisor a, Advisor b) throws AopConfigException;

	void addAdvice(Advice advice) throws AopConfigException;

	void addAdvice(int pos, Advice advice) throws AopConfigException;

	boolean removeAdvice(Advice advice);

	int indexOf(Advice advice);

	String toProxyConfigString();

}
~~~

我们可以使用Advised接口相应代理对象所持有的Advisor，进行添加Advisor、移除Advisor等相关动作。

### `ProxyFactory`

ProxyFactory是SpringAOP提供的最基本的weaver

使用ProxyFactory需要指定下面两个最基本的东西：

* 要被织入的目标对象：可以通过ProxyFactory的构造方法或者setter方法传入
* 要织入到目标对象的Aspect，也就是Advisor，除了指定Advisor，可以直接指定Advice
    * 对于Introduction之外的Advice类型，ProxyFactory内部会为这些Advice构造相应的Advisor，不过构造出的Advisor中的Pointcut为Pointcut.TRUE
    * 对于Introduction类型的Advice：
        * 若是IntroductionInfo的子类实现，因为本身包含了必要的描述信息，内部会为其构造一个DefaultIntroductionAdvisor
        * 若不是IntroductionInfo的子类实现而是DynamicIntroductionAdvice的实现，将抛出AopConfigException异常

#### 基于接口的代理

默认情况下，如果目标类实现了相应接口，那么ProxyFactory就会对目标类进行基于接口的代理，但是我们也可以使用ProxyFactory提供的方法：

~~~java
void setInterfaces(Class<?>... interfaces) 
~~~

来通知ProxyFactory强制使用基于接口的代理(在既可以使用接口代理，也可以使用类的代理的情况下)

#### 基于类的代理

默认情况下，如果目标类没有实现任何接口，那么ProxyFactory会对目标类对目标类进行基于类的代理，但是我们也可以使用ProxyFactory提供的方法：

~~~java
void setProxyTargetClass(boolean proxyTargetClass); //传入true，使用基于类的代理
~~~

来通知ProxyFactory强制使用基于类的代理(在既可以使用接口代理，也可以使用类的代理的情况下)

#### Introduction的织入

* 进行Introduction的织入时，不需要指定Pointcut
* Spring的Introduction支持只能通过接口定义为当前对象添加新的行为

使用ProxyFactory进行Introduction 的织入实例：

~~~java
//创建advice
Tester tester = new Tester();
DelegatingIntroductionInterceptor advice = new DelegatingIntroductionInterceptor(tester);
//创建weaver
Developer developer = new Developer();
ProxyFactory weaver = new ProxyFactory(developer);
//设置weaver
weaver.setInterfaces(IDeveloper.class,ITester.class);
weaver.addAdvice(advice);
//获取并使用代理
Object proxy = weaver.getProxy();
((IDeveloper)proxy).developSoftware();
((ITester)proxy).testSoftware();
~~~

基于类的代理：

~~~java
//创建advice
Tester tester = new Tester();
DelegatingIntroductionInterceptor advice = new DelegatingIntroductionInterceptor(tester);
//创建weaver
Developer developer = new Developer();
ProxyFactory weaver = new ProxyFactory(developer);
//设置weaver
weaver.setProxyTargetClass(true);
weaver.setInterfaces(ITester.class);
weaver.addAdvice(advice);
//获取并使用代理
Object proxy = weaver.getProxy();
((Developer)proxy).developSoftware();
((ITester)proxy).testSoftware();
~~~

### `ProxyFactoryBean`

从继承关系上看`ProxyFactoryBean`继承了`ProxyCreatorSupport`并实现了`FactoryBean`

所以说它是一个生成代理的 工厂Bean

我们知道`FactoryBean`在容器里被调用时，将直接返回`getObject()`方法的返回值。

在`ProxyFactoryBean`中，`getObject()`方法返回的就是对应的代理对象

#### 配置属性

`ProxyFactoryBean`继承了`ProxyCreatorSupport`，所以继承了它所有的配置属性，除此之外它还有自己独有的属性：

* `proxyInterfaces`：指定代理对象的接口类型，如果要为代理对象添加新的接口逻辑，需要在此处指定，和setInterfaces等效

* `interceptorNames`：可以指定多个将要织入到目标对象的Advice、拦截器以及Advisor
  * 可以在某个元素之后添加通配符`*`进行匹配
* `singleton`:设置代理对象是否是每次返回同一个

#### 实例

**per-class：**

~~~xml
<!-- 定义advisor -->
<bean id="advisor" class="org.springframework.aop.support.DefaultPointcutAdvisor">
    <property name="pointcut">
        <bean class="org.springframework.aop.TruePointcut"/>
    </property>
    <property name="advice">
        <bean class="com.wzm.spring.aop.container.MyMethodInterceptor"/>
    </property>
</bean>
<!-- 定义ProxyFactoryBean -->
<bean id="demo" class="org.springframework.aop.framework.ProxyFactoryBean">
    <property name="target">
        <bean class="com.wzm.spring.pojo.impl.Developer"/>
    </property>
    <property name="interceptorNames">
        <list>
            <value>advisor</value>
        </list>
    </property>
</bean>
~~~

**per-instance：**

* 使用`DelegatingIntroductionInterceptor`

~~~xml
<bean id="target" class="com.wzm.spring.pojo.impl.Target" scope="prototype"/>
<bean id="interceptor" class="org.springframework.aop.support.DelegatingIntroductionInterceptor">
    <constructor-arg>
        <bean class="com.wzm.spring.pojo.impl.Counter"/>
    </constructor-arg>
</bean>
<bean id="introductionTarget" class="org.springframework.aop.framework.ProxyFactoryBean" scope="prototype">
    <property name="targetName" value="target"/>
    <property name="proxyInterfaces">
        <list>
            <value>com.wzm.spring.pojo.ITarget</value>
            <value>com.wzm.spring.pojo.ICounter</value>
        </list>
    </property>
    <property name="interceptorNames">
        <list>
            <value>interceptor</value>
        </list>
    </property>
</bean>
~~~

获取代理：

~~~java
ClassPathXmlApplicationContext container = new ClassPathXmlApplicationContext("spring.xml");
ICounter proxy1 =(ICounter) container.getBean("introductionTarget");
ICounter proxy2 =(ICounter) container.getBean("introductionTarget");
System.out.println(proxy1.getCounter());
System.out.println(proxy1.getCounter());
System.out.println(proxy2.getCounter());
~~~

之前已经介绍过`DelegatingIntroductionInterceptor`织入的类时公用的，

proxy1和proxy2公用一个counter实例，所以上述代码会输出1，2，3

* `DelegatePerTargetObjectIntroductionInterceptor`使用：

~~~xml
<bean id="target" class="com.wzm.spring.pojo.impl.Target" scope="prototype"/>
<bean id="interceptor" class="org.springframework.aop.support.DelegatePerTargetObjectIntroductionInterceptor">
    <constructor-arg index="0" value="com.wzm.spring.pojo.impl.Counter"/>
    <constructor-arg index="1" value="com.wzm.spring.pojo.ICounter"/>
</bean>
<bean id="introductionTarget" class="org.springframework.aop.framework.ProxyFactoryBean" scope="prototype">
    <property name="targetName" value="target"/>
    <property name="proxyInterfaces">
        <list>
            <value>com.wzm.spring.pojo.ITarget</value>
            <value>com.wzm.spring.pojo.ICounter</value>
        </list>
    </property>
    <property name="interceptorNames">
        <list>
            <value>interceptor</value>
        </list>
    </property>
</bean>
~~~

获取代理：

~~~java
ClassPathXmlApplicationContext container = new ClassPathXmlApplicationContext("spring.xml");
ICounter proxy1 =(ICounter) container.getBean("introductionTarget");
ICounter proxy2 =(ICounter) container.getBean("introductionTarget");
System.out.println(proxy1.getCounter());
System.out.println(proxy1.getCounter());
System.out.println(proxy2.getCounter());
~~~

因为`DelegatePerTargetObjectIntroductionInterceptor`的特性，即使是公用一个Interceptor，它也会为每个代理对象保存单独的状态，所以，上面代理输出结果，1，2，1

### AutoProxy

SpringAOP给出了自动代理(AutoProxy)机制，可以帮助我们解决使用`ProxyFactoryBean`配置工作量比较大的问题,要使用自动代理，需要使用ApplicationContext，自动代理体系继承关系如下：

![AutoProxyCreator](https://gitee.com/wangziming707/note-pic/raw/master/img/AutoProxyCreator.png)

自动代理的逻辑主要集成在  `AbstractAutoProxyCreator`中，它继承了BeanPostProcessor，主要在`BeanPostProcessor`的`postProcessBeforeInitialization()`方法中完成拦截并织入横切逻辑的

常用的AutoProxyCreator如下：

* `BeanNameAutoProxyCreator`
* `DefaultAdvisorAutoProxyCreator`

如果要实现自定义的AutoProxyCreator，比如实现基于注解的自动代理；可以在`AbstractAutoProxyCreator`或者`AbstractAdvisorAutoProxyCreator`的基础上进行扩展；重写`BeanPostProcessor`的拦截方法

#### `BeanNameAutoProxyCreator`

`BeanNameAutoProxyCreator`内部有两个属性：

* `beanNames`:指定容器内要被代理的目标对象的beanName,可以使用通配符`*`以简化配置
* `interceptorNames`: 指定要应用到目标对象上的Advisor、advice或者拦截器

**实例:**

~~~xml
<bean id="target" class="com.wzm.spring.pojo.impl.Target" scope="prototype"/>
<bean id="interceptor" class="com.wzm.spring.aop.container.MyMethodInterceptor"/>
<bean class="org.springframework.aop.framework.autoproxy.BeanNameAutoProxyCreator">
    <property name="beanNames">
        <list>
            <idref bean="target"/>
        </list>
    </property>
    <property name="interceptorNames">
        <list>
            <idref bean="interceptor"/>
        </list>
    </property>
</bean>
~~~

#### `DefaultAdvisorAutoProxyCreator`

只需要在ApplicationContext中注册该bean，它就会自动进行扫描容器中的advisor，并进行自动织入

~~~xml
<!-- 定义DefaultAdvisorAutoProxyCreator -->
<bean class="org.springframework.aop.framework.autoproxy.DefaultAdvisorAutoProxyCreator"/>
<!-- 定义advisor -->
<bean class="org.springframework.aop.support.DefaultPointcutAdvisor">
    <property name="pointcut">
        <bean class="org.springframework.aop.support.JdkRegexpMethodPointcut">
            <property name="pattern" value=".*execute.*"/>
        </bean>
    </property>
    <property name="advice">
        <bean class="com.wzm.spring.aop.container.MyMethodInterceptor"/>
    </property>
</bean>
<!-- target -->
<bean id="target" class="com.wzm.spring.pojo.impl.Target" scope="prototype"/>
~~~

因为DefaultAdvisorAutoProxyCreator处理的范围比较大，所以需要精细设置Advisor的pointcut定义

## TargetSource

在使用ProxyFactory、ProxyFactoryBean时，通过它的setTarget()方法指定具体的目标对象，ProxyFactoryBean还可以通过setTargetName()指定目标对象在Ioc容器中的beanName

实际上，这些设置目标对象的方法底层都是通过TargetSource实现类对目标对象进行封装，实现了设置目标对象的统一处理

所以我们也可以直接通过setTargetSource方法来完成目标对象的设置

TargetSource可以看作目标对象的容器:

* 提供一个目标对象池，每次从TargetSource取得的目标对象都是从目标对象池中取得

* TargetSource实现类可以持有多个目标对象实例，每次方法调用时，会按照某种规则，返回相应的目标对象实例

当然它也可以持有一个目标对象实例。

定义如下：

~~~java
public interface TargetSource extends TargetClassAware {
	//	返回目标对象类型
	@Override
	@Nullable
	Class<?> getTargetClass();
	// 是否返回同一个目标对象实例,如果为false，具体调用时会调用releaseTarget方法释放对象
	boolean isStatic();
	// 返回目标对象实例
	@Nullable
	Object getTarget() throws Exception;
	//释放当前调用的目标对象 
	void releaseTarget(Object target) throws Exception;

}
~~~

它的常用实现：

* `SingletonTargetSource`

  内部只持有一个目标对象，每次方法调用都会返回这同一个目标对象

  可以直接在创建SingletonTargetSource实例时，在构造方法上传入目标对象实例

* `PrototypeTargetSource`

  内部只持有一个目标对象，每次方法调用时会返回一个新的目标对象实例供调用

  * 目标对象的bean定义的scope必须时prototype的
  * 只能通过targetBeanName指定目标对象的bean定义名称而不是直接引用

* `HotSwappableTargetSource`

  内部只持有一个目标对象，但可以通过 `swap()`方法替换持有的目标对象

* `CommonsPool2TargetSource`

  内部只有一个目标对象池，相同的类不同的实例，使用方法PrototypeTargetSource相同

* `ThreadLocalTargetSource`

  为不同的线程提供不同的目标对象，是基于ThreadLocal的简单封装

# @AspectJ形式的AOP

Spring2.0发布后，Spring AOP 支持新的使用方式

* 支持AspectJ5发布的@AspectJ形式的AOP实现方式，可以直接使用POJO来定义Aspect以及相关的Advice，并使用一套标准的注解来标注这些POJO，SpringAOP会根据注解信息查找相关的Aspect定义，并将其声明的横切逻辑织入当前系统
* 简化了XML配置方式。使用新的基于XSD的配置方式，可以使用aop独有的命名空间

@AspectJ代表一种定义Aspect的风格，这种方式是从AspectJ引入的，但最终的实现机制还是SpringAOP的代理模式

简单的POJO+@Aspect就是一个@AspectJ形式的Aspect

## 使用方式

可以通过POJO和注解标注的方式定义Aspect：

~~~java
@Aspect
@Component
public class LogAspect {
    @Pointcut("execution(* *..pojo.*.*(..))")
    public void pointcutName(){}

    @Around("pointcutName()")
    public Object beforeExecute(ProceedingJoinPoint joinPoint) throws Throwable {
        System.out.println("方法执行前");
        Object proceed = joinPoint.proceed();
        System.out.println("方法执行前");
        return proceed;
    }
}
~~~

* `@Aspect`标记当前类为aspect
* `@Pointcut`将pointcut绑定到当前方法上
* `@Around @After @Before` 标记当前方法为横切逻辑

定义好Aspect后，我们可以使用如下方式织入：

### 编程方式织入

我们在介绍Weaver是，在继承体系中有一个`AspectJProxyFactory`没有介绍，它就是适用于@AspectJ方式的织入器的，具体织入过程如下：

~~~java
AspectJProxyFactory weaver = new AspectJProxyFactory();
weaver.setTarget(new Target());
weaver.addAspect(LogAspect.class);
ITarget proxy = weaver.getProxy();
proxy.execute();
~~~

织入方式和ProxyFactory差不多

### 自动代理织入

在介绍AutoProxy自动代理是，继承体系中有一个`AspectJAwareAdvisorAutoProxyCreator`没有介绍，它就是适用于@AspectJ方式的自动代理类，只需要将其注册到容器中：

~~~xml
<bean class="org.springframework.aop.aspectj.annotation.AnnotationAwareAspectJAutoProxyCreator"/>
<bean id="target" class="com.wzm.spring.pojo.impl.Target"/>
<bean class="com.wzm.spring.aspectj.LogAspect"/>
~~~

或者使用xsd形式的配置方式：

~~~xml
<!-- 扫描aspect和target类-->
<context:component-scan base-package="com.wzm.spring.pojo,com.wzm.spring.aspectj"/>
<!-- 开启aspectj自动代理-->
<aop:aspectj-autoproxy/>
~~~

## @AspectJ形式的Pointcut

在`@Aspect`所标注的Aspect定义类内使用`@Pointcut`注解，指定AspectJ形式的Pointcut表达式后，将注解标注到某个方法上即可

@AspectJ形式的Pointcut声明有如下两个部分：

* Pointcut Expression
* Pointcut Signature

### Pointcut Expression

Pointcut Expression：切入点表达式的载体为`@Pointcut`，该注解是方法级别的，所以表达式也只能在某个方法上声明；Pointcut Expression所附着的方法称为该Pointcut Expression的Pointcut Signature(切入点签名)

Pointcut Expression由两部分组成：

* Pointcut Desinator(Pointcut标识符)。表明该Pointcut将以怎样的行为来匹配表达式
* 表达式匹配模式，在Pointcut标识符之内可以指定具体的匹配模式

Pointcut Expression 支持`&& || !`逻辑运算

#### Pointcut Desinator

AspectJ的Pointcut表达式可用的Pointcut标识符很丰富，基本包含所有的Joinpoint类型的表述，但是SpringAOP只支持方法级别的Joinpoint，所以可以使用的标识符只有少数的几个：

##### execution

execution:匹配拥有指定方法签名的Joinpoint，使用execution标识符的Pointcut表达式的规定格式如下：

`execution(modifiers-pattern? ret-type-pattern declariong-type-pattern? name-pattern(param-pattern) throws-pattern?)`

* `?`表示可选匹配
* `modifiers-pattern`匹配访问权限类型 可省略
* `ret-type-pattern`匹配返回值类型
* `declaring-type-pattern`匹配包名类名 可省略
* `name-pattern(param-pattern)`匹配方法名(参数)
* `throws-pattern`匹配异常 可省略

以上匹配可以由符号表示含义：

* `*`表示相邻的多个任意字符

* `..`
  * 在方法参数中，表示任意多个参数
  * 在包名后，表示当前包及其子包路径

* `+`
  * 在类名后，表示当前类及其子类
  * 在接口后，表示当前接口及其实现类

示例：

~~~java
举例：
execution(public * *(..)) ;
指定切入点为：任意公共方法。
execution(* set*(..)) ;
指定切入点为：任何一个以“set”开始的方法。
execution(* com.xyz.service.*.*(..)) ;
指定切入点为：定义在 service 包里的任意类的任意方法。
execution(* com.xyz.service..*.*(..));
指定切入点为：定义在 service 包或者子包里的任意类的任意方法。“..”出现在类名中时，后
面必须跟“*”，
表示包、子包下的所有类。
execution(* *..service.*.*(..));
指定所有包下的 serivce 子包下所有类（接口）中所有方法为切入点
execution(public void com.bjpn.service.UserService.addUser(String ));
~~~

##### within

within标识符只接受类型声明，它将会匹配指定类型下所有的Joinpoint

因为SpringAOP只支持方法级别的Joinpoint，所以使用within指定某个类后，它将匹配指定类所声明的所有方法执行

示例：

~~~java
举例
within(com.demo.spring.*);
匹配com.demo.spring包下所有类型内部的方法级别的Joinpoint
within(com.demo.spring..*);
匹配com.demo.spring包以及子包下所有类型内部的方法级别的Joinpoint
~~~

##### this&target

在AspectJ中：

* this代指调用方法一方所在的对象(caller)
* target代指被调用方法所在的对象(calle)

示例：`this(A) && target(B)`表示A调用B上的方法时才会匹配

SpringAOP中的this和target标识符语义有别于AspectJ中的两个标识符的原始语义：

* this代指目标对象的代理对象
* target代指目标对象

实际上，从代理模式来看，代理对象通常和目标对象的类型是相同的，因为

目标对象和它的代理对象实现同一个接口。

假设我们有如下对象定义：

~~~java
public interface ITarget{
    ...
}
public class Target implements ITarget{
    ...
}
~~~

* `this(ITarget)`和`target(ITarget)`
  * 基于接口的代理：目标对象Target实现了ITarget，代理对象同样实现了ITarget；以上表达式效果相同
  * 基于类的代理：目标对象Target实现了ITarget，代理对象继承了Target间接实现了ITarget；以上表达式效果相同

* `this(Target)`和`target(Target)`
  * 基于接口的代理：目标对象本身就是Target，代理对象只是ITarget的实现类；以上表达式效果不同
  * 基于类的代理：目标对象本身就是Target，代理对象继承了Target；以上表达式效果相同

通常this和target标志符都在Pointcut表达式中于其他标志符结合使用，进一步加强匹配的限定规则

##### args

该标志符用来匹配拥有指定参数类型、指定参数数量的方法级Joinpoint

与execution标志符可以直接匹配方法参数类型签名不同，args的会在运行时动态检查参数类型

比如对于表达式`args(com.demo.spring.User)`，即使方法签名是`public boolean login(Object user)`，只要在运行时传入的是User实例，表达式仍然可以捕捉到该Joinpoint

##### 注解类标识符

注解类标识符，将匹配被指定注解标识的类/方法，并根据不同注解标识符进行不同规则的匹配

* `@within`只接受注解类型，并对被指定注解类型所标注的类生效，匹配被指定注解类型所标注的类的所有方法
* `@Target`只接受注解类型，如果目标对象拥有`@Target`标志符所指定的注解类型，那么对象内部所有的Joinpoint将被匹配，对于SpringAOP 中，就是目标对象中的所有方法级别的Joinpoint将被匹配
* `@args`只接受注解类型且只能指定一个，如果该次传入的参数类型被`@args`所指定的注解标注，那么当前Joinpoint将被匹配

* `@annotation`匹配被`@annotation`指定的注解标注的方法

**演示：**

首先自定义一个注解：

~~~java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD,ElementType.TYPE})
public @interface Test {

}
~~~

然后在某aspect中定义pointcut:

~~~java
@Pointcut("@within(com.wzm.spring.annotation.Test)")
public void demo(){}
~~~

该pointcut可以匹配下面类的所有方法：

~~~java
@Test
@Component
public class Foo {
    public void method1(){
        System.out.println("method1");
    }

    public void method2(){
        System.out.println("method2");
    }
}
~~~

### Pointcut Signature

Pointcut Signature：在这里是一个方法定义，它是Poincut Expression的载体，Pointcut Signature 所在的方法定义要求返回类型必须是void

方法的范围修饰符在aspect中语义与java相同：

* public 型的Pointcut Signature可以在其他Aspect定义中引用
* private型的Pointcut Signature只能在当前Aspect定义中引用

可以将Pointcut Signature作为相应Pointcut Expression的标识符，在Pointcut Expression的定义中取代重复的Pointcut表达式，示例如下：

~~~java
@Pointcut("execution(void mehtod1())")
public void method1Execution(){}

@Pointcut("method1Execution()")
private void stillMethodExecution(){}
~~~

### 原理

@AspectJ形式声明的Pointcut表达式，在SpringAOP内部会被解析为`AspectJExpressionPointcut`实例，它的继承体系如下:

![AspectJExpressionPointcut](https://gitee.com/wangziming707/note-pic/raw/master/img/AspectJExpressionPointcut.png)

在`AspectJProxyFactory`或者`AnnotationAwareAspectJAutoProxyCreator`通过反射或者Aspect中@Pointcut定义的AspectJ形式的Pointcut定义后，会构造一个对应的`AspectJExpressionPointcut`实例,其内部持有通过反射获得的Pointcut表达式

`AspectJExpressionPointcut`是`Pointcut`的实现，也属于SpringAOP的`Pointcut`定义之一，

仍然通过ClassFilter 和 MethodMatcher来进行Joinpoint的匹配，只是具体匹配委托给了AspectJ类库

## @AspectJ形式的Advice

@AspectJ形式的Advice就是在@Aspect标注的Aspect定义类中的被Advice注解标注的普通方法

可以用于标注的对应Advice定义方法的注解有：

* `@Before`：用于标注Before Advice定义所在的方法
* `@AfterReturning`：用于标注After Returning Advice定义所在的方法
* `@AfterThrowing`：用于标注After Throwing Advice定义所在的方法
* `@After`：用于标注After Advice定义所在的方法
* `@Around`：用于标注Around Advice定义所在的方法，即拦截器类型的Advice
* `@DeclareParents`用于标注Introduction类型的Advice，该注解对应标注对象的域而不是方法

### `@Before`

它的成员变量value必须指定，可以直接指定Poincut Expression ，也可以指定单独声明的Pointcut Signature

示例：

~~~java
@Pointcut("@within(com.wzm.spring.annotation.Test)")
public void demo(){}

@Before("demo()")
public Object beforeExecute() {
    ...
}
~~~

或者：

~~~java
@Before("@within(com.wzm.spring.annotation.Test)")
public Object beforeExecute() {
    ...
}
~~~

#### 访问方法参数

我们可能会需要在Advice定义中访问Joinpoint处的方法参数，可以通过如下两种方式：

* `Joinpoint`:可以将BeforeAdvice的方法的第一个参数声明为`JoinPoint`类型，通过它可以获取Joinpoint处方法的参数值。或者方法的其他信息：

  ~~~java
  @Before("pointcut()")
  public void beforeAdvice(JoinPoint joinPoint){
      Object[] args = joinPoint.getArgs();
  	...
  }
  ~~~

* `args`绑定：`args`标志符除了可以指定方法参数类型，还可以指定参数名称

  当指定的是参数名称时，它会将这个参数名称绑定到对象的Advice方法

  ~~~java
  @Before("pointcut() && args(count)")
  public void beforeAdvice(int count){
      ...
  }
  ~~~

  * args指定的参数名称必须和Advice定义所在方法的参数名称相同
  * Advice定义所在方法的参数类型也会参与匹配，上述例子不会匹配`String count`

上述两种访问方法参数的方式可以同时使用，但JoinPoint必须在第一个参数位置

**拓展使用：**

* 除了 Around Advice 和 Introduction 外，其他的Advice类型都可以在Advice定义所在方法的第一个参数位置声明JoinPoint类型的参数

* 除了 execution标志符不会直接指定对象类型之外，其他的标志符都可以直接指定对象类型；

  这些标志符和`args`一样，他们也可以指定参数名称，作用和`args`指定参数并绑定到Advice上是一样的

### `@AfterThrowing`

`@AfterThrowing`有一个独有属性`throwing`，它限定了Advice定义方法的参数名----必须和该属性值相同；并将相应的异常绑定的具体的方法参数上

示例:

~~~java
@AfterThrowing(pointcut = "execution(* *..AfterThrowingTarget.*(..))",throwing = "e")
public void advice(RuntimeException e){
    System.out.println(e.getMessage());
}
~~~

使用JoinPoint也可以完成同样的事情

### `@AfterReturning`

`@AfterReturning`有一个独有属性`returning`,它将方法返回值绑定到Advice定义的所在方法：

~~~java
@AfterReturning(pointcut = "within(com.wzm.spring.aspectj.entity.AfterReturningTarget)",returning = "result")
public void advice(String result){
    ...
}
~~~

### `@After`

不管Joinpoint处方法是抛出异常，还是正常返回，都能触发After Advice 的执行，该类型的Advice适合做一些网络连接、数据库资源的释放

### `@AroundAdvice`

之前提过，`@Before @AfterReturning @AfterThrowing @After`所标注的方法的第一个参数可以为`JoinPoint`类型

但对于`@AroundAdvice`来说，它的第一个参数必须是`ProceedingJoinPoint`类型

通常情况下，我们需要通过`ProceedingJoinPoint`的`proceed()`方法继续调用链的执行

示例：

~~~java
@Around("execution(* *..AroundTarget.*(..))")
public Object advice(ProceedingJoinPoint joinPoint) throws Throwable {
    ...
    Object proceed = joinPoint.proceed();
    ...
    return proceed;
}
~~~

调用`ProceedingJoinPoint`的`proceed()`时，可以传入`Object[]`数组来对传入的参数进行处理：

~~~java
@Around("execution(* *..AroundTarget.*(..)) && args(message)")
public Object advice(ProceedingJoinPoint joinPoint,String message) throws Throwable {
    message = message +"...该条消息已被处理";
    return joinPoint.proceed(new Object[]{message});
}
~~~

除了使用`args`标志符，也可以使用ProceedingJoinPoint来获取方法参数

### `@DeclareParents`

@DeclareParents指定Introduction类型的Advice

该注解只能标注Aspect类中的实例变量

Introduction类型的Advice可以将新添加的行为逻辑以新的接口定义添加到目标对象上

实例变量的类型对应的就是新增加的接口类型

示例：

~~~java
@Aspect
@Component
public class IntroductionAspect {

    @DeclareParents(
            value = "com.wzm.spring.aspectj.entity.IntroductionTarget",
            defaultImpl = Counter.class)
    ICounter counter;

}
~~~

### Advice的执行顺序

对于多个Advice，如果它们匹配同一个Joinpoint，他们的执行顺序有一定的规则：

* 在同一个Aspect定义中，Advice的执行顺序由声明顺序决定,先声明的拥有高优先级
  * 对于Before Advice 高优先级的先运行
  * 对于After Advice 高优先级的后运行
* 在不同的Aspect定义中，根据Ordered接口的规范决定优先级`getOrder()`返回值小的优先级高

# 基于Schema的AOP

要使用基于Schema的AOP，IoC容器的配置文件应该使用基于Schema的XML，同时在文件头中增加针对AOP的命名空间声明：

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/context
       https://www.springframework.org/schema/context/spring-context.xsd
       http://www.springframework.org/schema/aop
       http://www.springframework.org/schema/aop/spring-aop-4.3.xsd">
</beans>
~~~

在spring-aop规定的命名空间下，可以使用标签`<aop:config>`来进行aop配置，它只有一个属性`proxy-target-class`用以控制是使用基于类的代理还是基于接口的代理

它内部有三个子元素，结构如下：

* `<aop:config>` : 
    * `<aop:pointcut>`
    * `<aop:advisor>`
    * `<aop:aspect>`

它们在config标签中的定义顺序必须如上排序

## `<aop:pointcut>`

`<aop:config>`内部可以声明一个或者多个`<aop:pointcut>`，可以被`<aop:advisor>`或者`<aop:aspect>`引用

它有如下属性：

* `id`：指定当前Pointcut定义的标志id
* `expression`指定当前Pointcut的 Pointcut Expression

**注意：**

 Pointcut Expression可以用 `&& || !`来进行逻辑运算，但在xml文件中禁止使用`&&`字符，所以我们可以用`and or not`来替代

## `<aop:advisor>`

用以定义Advisor，也就是SpringAOP中的Aspect，它有如下属性：

* id：指定当前Advisor定义的标志id
* pointcut-ref：指定当前Advisor对应的Pointcut的对象引用，或者`<aop:advisor>`的id
* pointcut:指定AspectJ形式的Pointcut表达式
* advice-ref:指定当前Advisor对应的Advice对象引用
* order:指定当前Advisor的顺序

示例：

~~~xml
<bean id="advice" class="com.wzm.spring.schema.MyMethodInterceptor"/>
<bean id="pointcut" class="org.springframework.aop.support.JdkRegexpMethodPointcut">
    <property name="pattern" value=".*execute.*"/>
</bean>
<bean id="target2" class="com.wzm.spring.pojo.impl.Target"/>

<aop:config>
    <aop:pointcut id="pointcutId" expression="execution(* *..*(..))"/>
    <!--pointcut-ref指定对象引用-->
    <aop:advisor advice-ref="advice" pointcut-ref="pointcut"/>
    <!--pointcut-ref指定<aop:pointcut>id-->
    <aop:advisor advice-ref="advice" pointcut-ref="pointcutId"/>
    <!--pointcut指定pointcut expression-->
    <aop:advisor advice-ref="advice" pointcut="execution(* *..*(..))"/>
</aop:config>
~~~

## `<aop:aspect>`

通过`<aop:aspect>`，可以基于POJO对象定义Aspect    

它有以下属性：

* `id`Aspect定义在配置文件中的id
* `ref`指向Aspect定义对应的容器内的bean定义
* `order`Aspect定义对应的顺序号

它内部可以有如下标签：

### Pointcut

* `<aop:pointcut>`和 `<aop:config>`内的`<aop:pointcut>`一样，不过只能在`<aop:aspect>`内部被引用

### Advice

* `<aop:beofre>`
* `<aop:after-returning>`
    * `returning`指定返回值的参数名,该值需要与方法声明中的参数名称相同
* `<aop:after-throwing>`
    * `throwing`:指定异常的参数名称，该值要与方法声明中的参数名称相同
* `<aop:after>`
* `<aop:around>`
* `<aop:declare-parents>`
    * `type-matching`:指定将要对那些目标对象进行Introduction逻辑织入
    * `implement-interface`：指定新增加的Introduction行为的接口定义类型
    * `default-impl`:指定增加的Introduction行为的接口定义的默认实现类

**注意：**

除了`<aop:declare-parents>`外，其他Advice标签都具有以下属性：

* `method `：指定Advice对应的方法
* `arg-names`:指定Advice对应的方法的参数名 多个参数名用逗号隔开
* `pointcut-ref`:指定 引用的`<aop:pointcut>`的id
* `pointcut`:指定Pointcut表达式

以绑定pointcut和附着方法

示例：

~~~xml
<bean id="aspectDemo" class="com.wzm.spring.schema.AspectDemo"/>
<aop:config>
    <aop:aspect id="aspectDemo" ref="aspectDemo">
        <aop:pointcut id="p" expression="execution(* *..Target.*(..))"/>
        <aop:before method="doBefore" pointcut-ref="p" arg-names="joinpoint"/>
        <aop:after-returning method="doAfterReturning" pointcut-ref="p" returning="result"/>
        <aop:after-throwing method="doAfterThrowing" pointcut-ref="p" throwing="e"/>
        <aop:after method="doAfter" pointcut-ref="p"/>
        <aop:around method="doAround" pointcut-ref="p"/>
        <aop:declare-parents types-matching="com.wzm.spring.pojo.impl.Target"
                             implement-interface="com.wzm.spring.pojo.ICounter"
                             default-impl="com.wzm.spring.pojo.impl.Counter"/>
    </aop:aspect>
</aop:config>
~~~

# 由代理机制衍生的问题

==todo==







