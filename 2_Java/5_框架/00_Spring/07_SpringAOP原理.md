

# Spring AOP原理

Spring AOP是Spring核心框架的重要组成部分，与Spring IOC容器和Spring框架对其他JavaEE服务的集成 构成的Spring框架的核心

Spring AOP采用 Java作为AOP的实现语言(AOL)，在Java语言的基础之上，Spring AOP对AOP的概念进行了适当的抽象和实现

Spring AOP属于动态AOP采用动态代理机制和字节码生成技术实现。而动态代理是通过代理模式实现的(Proxy Pattern)

## 代理模式

在代理机制中：代理处于请求者和被请求者之间，可以隔绝两者直接的直接交互，代理者全权代理被请求者，拥有它的全部职能

实现了代理机制的设计模式，就叫代理模式，在代理模式中，通常涉及4种角色：

![代理模式](https://gitee.com/wangziming707/note-pic/raw/master/img/%E4%BB%A3%E7%90%86%E6%A8%A1%E5%BC%8F.png)

* ISubject: 该接口是对被访问者或者访问者资源的抽象
* RealSubject:被访问者或者被访问资源的具体实现类
* ProxySubject:被访问者或者被访问资源的代理实现类，该类持有ISubject接口的一个具体实例
* Client:代表访问者的抽象角色，Client将会访问ISubject类型的对象或者资源，在代理场景中，Client无法直接访问RealSubject获取资源，而是通过ProxySubject

在代理模式中，ProxySubject常常在RealSubject提供的资源基础上添加一些逻辑功能

在AOP中，Target就是 RealSubject，在为这个Target创建代理对象时，可以将横切逻辑添加到这个代理对象中。

我们可以发现，这样静态代理的方式，即使Joinpoint相同，如果对应的Target类型不同，那么就需要针对的所有的Target类型，创建对应的代理对象，即使这些代理对象所要添加的横切逻辑是一样的。这样会导致资源的浪费。

## 动态代理

动态代理能够解决静态代理实现AOP出现的问题

JDK提供了一种动态代理的规范：`java.lang.reflect.Proxy`类和`java.lang.reflect.InvocationHandler`接口

### 接口介绍

可以通过`Proxy`类的`newProxyInstance()`方法获取代理对象实例，它的方法签名如下：

~~~java
public static Object newProxyInstance(ClassLoader loader,Class<?>[] interfaces,InvocationHandler h)
~~~

需要传入被代理对象的`ClassLoader`,和被代理对象的接口类定义和的`InvocationHandler`

`InvocationHandler`定义如下：

~~~java
public interface InvocationHandler {

    public Object invoke(Object proxy, Method method, Object[] args)
        throws Throwable;
}
~~~

`newProxyInstance()`将生成一个代理对象，当代理对象发生方法调用时，这个代理对象会请求`InvocationHandler`，让它来进行真正的方法调用处理。实际的方法调用处理是在`InvocationHandler.invoke()`中进行的。

我们来解释一下`invoke()`方法的入参，假设`newProxyInstance()`方法返回一个对象`Object pro`，这个对象发生了方法调用`pro.doSomething(a)`,那么它会导致`InvocationHandler.invoke()`方法被调用，  `InvocationHandler.invoke()`的入参和返回值解释如下：

* `Object proxy`：是发起请求的代理对象，这里就是`pro`
* `Method method`：导致当前`invoke`方法被调用的代理对象的方法，这里是`doSomething()`方法
* `Object[] args`：导致当前`invoke`方法被调用的代理对象的方法入参，这里是`a`
* `return`：会成为代理对象的方法的返回值

所以，`newProxyInstance()`生成的代理对象的每个方法的方法体都类似下面代码：

~~~java
InvocationHandler h;  //newProxyInstance()方法传入的InvocationHandler

public Object doSomething(Object[] args){

    return h.invoke(this,getCurrentMethod() ,args );
}
~~~

### 示例

下面是使用动态代理的一个示例：

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

定义ITarget的`InvocationHandler`

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

如果我们在`MyInvocationHandler`中传入多个被代理对象，那么就实现了使用一个通用的代理对象来代理多个被代理对象。就可以实现横切逻辑的复用。

这样通过动态代理的方式，就实现了AOP

`InvocationHandler`是实现横切逻辑的地方，是AOP中的Advice

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

MethodMatcher 重载定义了两个matches方法，他们的区别是 是否会对方法参数进行校验

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