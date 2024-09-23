# SpringAOP 概述

Spring AOP是用纯Java实现的。不需要特殊的编译过程。Spring AOP不需要控制类加载器的层次结构，因此适合在Servlet容器或应用服务器中使用。

Spring AOP目前只支持方法执行连接点（advice 在Spring Bean上执行方法）。

Spring AOP的AOP方法与其他大多数AOP框架不同。其目的不是提供最完整的AOP实现。相反，其目的是在AOP实现和Spring IoC之间提供一个紧密的整合，以帮助解决企业应用中的常见问题。

因此，例如，Spring框架的AOP功能通常是与Spring IoC容器一起使用的。切面是通过使用正常的Bean定义语法来配置的。这是与其他AOP实现的一个重要区别。

Spring AOP的设计目的不是与AspectJ竞争，以提供一个全面的AOP解决方案。Spring AOP等基于代理的框架和AspectJ等完整的框架都很有价值，它们是互补的，而不是竞争的。Spring将Spring AOP和IoC与AspectJ无缝集成，以便在基于Spring的应用架构中实现AOP的所有用途。这种整合并不影响Spring AOP API或AOP联盟的API。Spring AOP仍然是向后兼容的。

# 基于@AspectJ的AOP支持

`@AspectJ` 指的是将aspect作为带有注解的普通Java类来声明的一种风格。这种风格是从AspectJ项目引入的，但最终的实现机制还是SpringAOP的代理模式，是纯粹的Spring AOP，不依赖AspectJ编译器或织入器（weaver）

简单的POJO+@Aspect就是一个@AspectJ形式的Aspect

## 启用@AspectJ的支持

根据`@AspectJ`配置Spring AOP后，Spring会根据Bean是否被这些切面关注而自动代理。也就是说Spring会为需要注入横切逻辑的Bean生成一个代理来拦截方法调用。

`@AspectJ`支持可以通过XML或者Java风格的配置来启用。在此之前，需要保证项目中引入了`aspectjweaver.jar`库

### Java配置启用`@AspectJ`支持

在`@Configuration`类上添加 `@EnableAspectJAutoProxy` 注解以启用`@AspectJ`支持，如下所示：

~~~java
@Configuration
@EnableAspectJAutoProxy
public class AppConfig {
}
~~~

### XML配置启用`@AspectJ`支持

使用xsd形式的配置方式，使用使用 `aop:aspectj-autoproxy` 元素开启`@AspectJ`支持：

~~~xml
<!-- 开启aspectj自动代理-->
<aop:aspectj-autoproxy/>
~~~

当然在使用`<aop>`命名空间前需要在XML配置文件的顶部添加aop相关的schema

## Aspect

启用`@AspectJ`支持后，任何在容器中定义的bean，如果其类被`@Aspect`注解注释，就会被Spring自动检测到，并用于配置Spring AOP。例如：

~~~java
import org.aspectj.lang.annotation.Aspect;

@Aspect
@Component
public class NotVeryUsefulAspect {
}
~~~

需要注意，单独`@Aspect`注解并不会让当前类被Spring的组键扫描自动检测到，需要再单独添加一个单独的 `@Component` 注解

在Spring AOP中，切面本身不能成为其他切面的advice的目标。类上的 `@Aspect` 注解将其标记为一个切面，因此，会将其排除在自动代理之外。

## Pointcut

Pointcuts确定advice的连接点，从而使我们能够控制 advice 的运行时间。Spring AOP只支持Spring Bean的方法执行类型的Joinpoint，所以可以把pointcut看作是对Spring Bean上的方法执行的匹配。

一个Pointcut声明有如下两个部分：

* Pointcut Expression：切点表达式
* Pointcut Signature：由名称和任意参数组成的切点签名

在AOP的 `@AspectJ` 注解式中，一个pointcut签名是由一个常规的方法定义提供的，而pointcut表达式是通过使用 `@Pointcut` 注解来表示的（作为pointcut签名的方法必须是一个 `void` 返回类型）

下面例子可以清楚地区分切点签名和切点表达式。下面的例子定义了一个名为 `anyOldTransfer` 的切点，它匹配任何名为 `transfer` 的方法的执行

~~~java
@Pointcut("execution(* transfer(..))") // the pointcut expression
private void anyOldTransfer() {} // the pointcut signature
~~~

Pointcut Expression的载体为`@Pointcut`，该注解是方法级别的，所以表达式也只能在某个方法上声明；Pointcut Expression所附着的方法称为该Pointcut Expression的Pointcut Signature(切入点签名)

而Pointcut Expression由两部分组成：

* Pointcut Designator(Pointcut标识符)。表明该Pointcut将以怎样的行为来匹配表达式
* 表达式匹配模式，在Pointcut指示符之内可以指定具体的匹配模式

例如对于下边pointcut表达式,execution为Pointcut标识符，`public * *(..)`为表达式匹配模式

~~~java
execution(public * *(..))
~~~

### Pointcut表达式

AspectJ的Pointcut表达式可用的Pointcut标识符很丰富，基本包含所有的Joinpoint类型的表述，但是SpringAOP只支持方法级别的Joinpoint，所以可以使用的标识符只有少数的几个：

#### execution

execution匹配拥有指定方法签名的Joinpoint

使用execution标识符的表达式匹配模式的规定格式如下：

`execution(modifiers-pattern? ret-type-pattern declariong-type-pattern? name-pattern(param-pattern) throws-pattern?)`

其中：

* `?`表示可选匹配
* `modifiers-pattern`匹配访问权限类型 可省略
* `ret-type-pattern`匹配返回值类型
* `declaring-type-pattern`匹配包名类名 可省略
* `name-pattern(param-pattern)`匹配方法名(参数)
* `throws-pattern`匹配异常 可省略

以上匹配可以使用如下符号：

* `*`表示相邻的多个任意字符

* `..`
  * 在方法参数中，表示任意多个参数
  * 在包名后，表示当前包及其子包路径

* `+`
  * 在类名后，表示当前类及其子类
  * 在接口后，表示当前接口及其实现类

#### within

within标识符只接受类型声明，它将会匹配指定类下所有的Joinpoint

因为SpringAOP只支持方法级别的Joinpoint，所以使用within指定某个类后，它将匹配指定类所声明的所有方法执行

#### this&target

在AspectJ中：

* this代指调用方法一方所在的对象(caller)
* target代指被调用方法所在的对象(calle)

示例：`this(A) && target(B)`表示A调用B上的方法时才会匹配

SpringAOP中的this和target标识符语义有别于AspectJ中的两个标识符的原始语义：

* this代指目标对象的代理对象，在Spring中也就是容器中bean 的实际类型
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

#### args

该标志符用来匹配拥有指定参数类型Joinpoint

与execution标志符可以直接匹配方法参数类型签名不同，args的会在运行时动态检查参数类型

比如对于表达式`args(com.demo.spring.User)`，即使方法签名是`public boolean login(Object user)`，只要在运行时传入的是User实例，表达式仍然可以捕捉到该Joinpoint

#### 注解类标识符

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

#### bean

Spring AOP还支持一个额外的名为 `bean` 的pointcut标识符，这个标识符匹配具有指定的beanName的bean，其形式如下：

~~~java
bean(idOrNameOfBean)
~~~

`idOrNameOfBean` 标记可以是任何Spring Bean的名称。可以使用 `*` 字符的有限通配符支持

### 组合切点表达式

可以通过使用 `&&`、`||` 和 `!` 来组合 pointcut 表达式。也可以通过名称来引用pointcut表达式。下面例子显示了他们的运用：

~~~java
package com.xyz;

@Aspect
public class Pointcuts {

    @Pointcut("execution(public * *(..))")
    public void publicMethod() {}

    @Pointcut("within(com.xyz.trading..*)")
    public void inTrading() {}

    @Pointcut("publicMethod() && inTrading()")
    public void tradingOperation() {}
}
~~~

如果不在同一个Aspect上，可以通过通过引用 `@Aspect` 类的全称名称和 `@Pointcut` 方法的名称，来引用pointcut，例如下面在另一个类中引用了`Pointcuts`类的切点

~~~java
@Aspect
public class A {

    @Pointcut("com.xyz.Pointcuts.publicMethod() && com.xyz.Pointcuts.inTrading()")
    public void tradingOperation() {} (3)
}
~~~

### 示例

~~~java
execution(public * *(..));
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
举例
within(com.demo.spring.*);
匹配com.demo.spring包下所有类下的所有方法
within(com.demo.spring..*);
匹配com.demo.spring包以及子包下所有类的所有方法
this(com.xyz.service.AccountService);
匹配实现了 AccountService 接口的代理的所有方法
target(com.xyz.service.AccountService);
匹配实现了 AccountService 接口的目标对象的所有方法
args(java.io.Serializable);
匹配在运行时传递的参数是 Serializable 的方法
@target(org.springframework.transaction.annotation.Transactional);
匹配目标对象有 @Transactional 注解的类的所有方法
~~~

## Advice

Advice 承载横切逻辑代码，与一个切点表达式相关联，在切点匹配的方法执行之前、之后或周围（around）运行。

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

### `@AfterReturning`

当一个匹配的方法执行正常返回时，After returning advice 运行：

~~~java
@Aspect
public class AfterReturningExample {

    @AfterReturning("execution(* com.xyz.dao.*.*(..))")
    public void doAccessCheck() {
        // ...
    }
}
~~~



`@AfterReturning`有一个独有属性`returning`,它将方法返回值绑定到Advice定义的所在方法入参，`returning` 属性中使用的名称必须与advice方法中的参数名称相对应。当一个方法执行返回时，返回值会作为相应的参数值传递给advice方法。

~~~java
@AfterReturning(pointcut = "within(com.wzm.spring.aspectj.entity.AfterReturningTarget)",returning = "result")
public void advice(Object result){
    ...
}
~~~

`returning` 子句也限制了匹配，只匹配那些返回指定类型的值的方法执行（在这种情况下是 `Object`，它匹配任何返回值），如果需要限制返回值类型，可以参考如下例子：

~~~java
@AfterReturning(pointcut = "within(com.wzm.spring.aspectj.entity.AfterReturningTarget)",returning = "result")
public void advice(String result){
    ...
}
~~~

### `@AfterThrowing`

当一个匹配的方法执行通过抛出异常退出时，After throwing advice 运行。你可以通过使用 `@AfterThrowing` 注解来声明它，如下例所示。

~~~java
@Aspect
public class AfterThrowingExample {

    @AfterThrowing("execution(* com.xyz.dao.*.*(..))")
    public void doRecoveryActions() {
        // ...
    }
}
~~~

`@AfterThrowing`有一个独有属性`throwing`，并将相应的异常绑定的具体的方法参数上，例如:

~~~java
@AfterThrowing(pointcut = "execution(* *..AfterThrowingTarget.*(..))",throwing = "e")
public void advice(DataAccessException e){
    System.out.println(e.getMessage());
}
~~~

在 `throwing` 属性中使用的名称必须与advice方法中的参数名称相对应。当一个方法的执行通过抛出一个异常退出时，该异常将作为相应的参数值传递给advice方法。

`throwing` 子句也限制了匹配，只能匹配那些抛出指定类型的异常的方法执行（本例中是 `DataAccessException`）。

### `@After`

不管Joinpoint处方法是抛出异常，还是正常返回，都能触发After Advice 的执行。After advice必须准备好处理正常和异常的返回条件。它通常被用于释放资源和类似的目的。

~~~java
@Aspect
public class AfterFinallyExample {

    @After("execution(* com.xyz.dao.*.*(..))")
    public void doReleaseLock() {
        // ...
    }
}
~~~

### `@AroundAdvice`

被`@AroundAdvice`注释的advice方法有以下要求：

* 方法返回类型最好是`Object`
* 方法的第一个参数必须是 `ProceedingJoinPoint` 类型

在 advice 方法的body中，一般必须在 `ProceedingJoinPoint` 上调用 `proceed()`，以使关注点底层方法运行。

如果在没有参数的情况下调用 `proceed()` ，会导致调用者的原始参数在底层方法被调用时被提供给它。除此之外有一个重载的 `proceed()` 方法，它接受一个参数数组（`Object[]`）。当底层方法被调用时，数组中的值将被用作该方法的参数。

around advice 返回的值是方法的调用者看到的返回值。

请注意， `proceed` 可以被调用一次，多次，或者根本就不在 around advice 的 body 中调用。所有这些都是合法的。

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

### Advice参数

可以在advice签名中声明你需要的参数

#### 访问当前的 `JoinPoint`

任何 advice method 都可以声明一个 `org.aspectj.lang.JoinPoint` 类型的参数作为其第一个参数。请注意，around advice 方法需要声明一个 `ProceedingJoinPoint` 类型的第一个参数，它是 `JoinPoint` 的一个子类。

`JoinPoint` 接口提供了许多有用的方法。例如：

- `getArgs()`: 返回方法的参数。
- `getThis()`: 返回代理对象。
- `getTarget()`: 返回目标对象。
- `getSignature()`: 返回正在被 advice 的方法的描述。
- `toString()`: 打印对所 advice 的方法的有用描述。

一个示例：

~~~java
@Before("pointcut()")
public void beforeAdvice(JoinPoint joinPoint){
    Object[] args = joinPoint.getArgs();
	...
}
~~~

#### 向 Advice 传递参数

`args`标志符除了可以指定方法参数类型，还可以指定参数名称，此时advice方法必须有相同参数名称的入参。advice 方法被调用时，相应的参数值会被传递。例如：

~~~java
@Before("execution(* com.xyz.dao.*.*(..)) && args(account,..)")
public void validateAccount(Account account) {
    // ...
}
~~~

pointcut 表达式的 `args(account,..)` 部分有两个作用：

* 进一步限定连接点的范围：连接点方法至少需要一个参数，并且传递给该参数的参数是一个 `Account` 的实例。
* 使实际的 `Account` 对象通过 `account` 参数传递给 advice 方法

在通过名称来引用pointcut表达式时，如果引用的表达式中有类似`args(account,..)`这样的参数声明，那么引用时需要在引用的方法名称后加上参数：

~~~java
@Pointcut("execution(* com.xyz.dao.*.*(..)) && args(account,..)")
private void accountDataAccessOperation(Account account) {}

@Before("accountDataAccessOperation(account)")
public void validateAccount(Account account) {
    // ...
}
~~~

代理对象（`this`）、目标对象（`target`）和注解（`@within`、`@target`、`@annotation` 和 `@args`）都可以用类似的方式(用参数名代替类型名)绑定。

一个`@annotation`的示例，首先定义一个注解：

~~~java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Auditable {
    AuditCode value();
}
~~~

下面显示了与 `@Auditable` 方法的执行相匹配的 advice：

~~~java
@Before("com.xyz.Pointcuts.publicMethod() && @annotation(auditable)") (1)
public void audit(Auditable auditable) {
    AuditCode code = auditable.value();
    // ...
}
~~~

`JoinPoint`参数和`args`绑定参数两种访问方法参数的方式可以同时使用，但JoinPoint必须在第一个参数位置



### Advice的执行顺序

Advice可以通过在在切面类中实现 `org.springframework.core.Ordered` 接口或用 `@Order` 注解来指定他们的优先级。给定两个切面，从 `Ordered.getOrder()` 返回较低值的切面（或注解值）具有较高的优先权。

两个相同类型的Advice在同一个连接点运行时，会根据优先级来确定先后顺序。

* 在进入连接点时首先运行优先级高的advice，即对于Before Advice，优先级高的先运行。
* 在退出连接点时最后运行优先级高的advice，即对于After Advice， 高优先级的后运行。

## Introduction

Introduction（在AspectJ中被称为类型间声明）使一个切面能够声明advice对象实现一个给定的接口，并代表这些对象提供该接口的实现。可以让目标对象有一个新的父接口，从而获得新的行为，所以这种方式也叫增强。

`@DeclareParents`指定Introduction类型的Advice,该注解只能标注Aspect类中的实例变量，这样实例变量的类型会增强到匹配到的目标类上。也就是目标类的代理类实际上会多出一个实例变量类型的父接口。而这个父接口具体的实现由`@DeclareParents`的`defaultImpl`属性指定。

例如，给定一个名为 `UsageTracked` 的接口和一个名为 `DefaultUsageTracked` 的接口的实现，下面这个切面声明所有服务接口的实现者也实现 `UsageTracked` 接口

~~~java
@Aspect
public class UsageTracking {

    @DeclareParents(value="com.xyz.service.*+", defaultImpl=DefaultUsageTracked.class)
    public static UsageTracked mixin;

    @Before("execution(* com.xyz..service.*.*(..)) && this(usageTracked)")
    public void recordUsage(UsageTracked usageTracked) {
        usageTracked.incrementUseCount();
    }

}
~~~

这样目标类可以直接当作`UsageTracked`来使用，例如：

~~~java
UsageTracked usageTracked = context.getBean("myService", UsageTracked.class);
~~~

# 基于Schema的AOP

Spring也提供了对使用 `aop` 命名空间标签定义切面的支持。它支持与使用 `@AspectJ` 风格时完全相同的 pointcut 表达式和advice种类。

要使用本节描述的 `aop` 命名空间标签，你需要导入 `spring-aop` schema，即对配置文件头部进行如下声明：

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

在spring-aop规定的命名空间下，所有的aop配置都必须在标签`<aop:config>`中进行，它只有一个属性`proxy-target-class`用以控制是使用基于类的代理还是基于接口的代理

它内部有三个子元素，结构如下：

* `<aop:config>` : 
  * `<aop:pointcut>`
  * `<aop:advisor>`
  * `<aop:aspect>`

它们在config标签中的定义顺序必须如上排序

## Aspect

在使用schema支持时，一个切面仍然是一个普通的Java对象，在Spring 容器中需要作为bean被引用。这个bean的方法将承载横切逻辑代码。作为advice被引用。

可以通过使用 `<aop:aspect>` 元素来声明一个切面，并通过使用 `ref` 属性来引用支持 Bean，如下例所示。

~~~xml
<aop:config>
    <aop:aspect id="myAspect" ref="aBean">
        ...
    </aop:aspect>
</aop:config>

<bean id="aBean" class="...">
    ...
</bean>
~~~

本例中aBean就是作为aspect的java对象。

## Pointcut

可以在一个 `<aop:config>` 元素中声明一个命名的 pointcut，让 pointcut 的定义在几个切面和advice之间共享：

~~~xml
<aop:config>
    <aop:pointcut id="businessService" expression="execution(* com.xyz.service.*.*(..))" />
</aop:config>
~~~

也可以在一个切面中声明pointcut:

~~~xml
<aop:config>
    <aop:aspect id="myAspect" ref="aBean">
        <aop:pointcut id="businessService" expression="execution(* com.xyz.service.*.*(..))"/>
        ...
    </aop:aspect>
</aop:config>
~~~

与 `@AspectJ` 切面的方式相同，通过使用基于schema的定义风格来声明的 pointcuts 可以绑定传递参数，例如下面这个 pointcut 收集了 `this` 对 象作为连接点上下文并将其传递给advice

~~~xml
<aop:config>
    <aop:aspect id="myAspect" ref="aBean">
        <aop:pointcut id="businessService" expression="execution(* com.xyz.service.*.*(..)) &amp;&amp; this(service)"/>
        <aop:before pointcut-ref="businessService" method="monitor"/>
        ...
    </aop:aspect>
</aop:config>
~~~

其advice定义如下：

~~~java
public void monitor(Object service) {
    // ...
}
~~~

在组合point的子表达式时，可以使用 `and`、`or` 和 `not` 关键字来代替 `&&`、`||` 和 `!`，分别。例如，前面的pointcut可以更好地写成如下：

~~~xml
<aop:config>
    <aop:aspect id="myAspect" ref="aBean">
        <aop:pointcut id="businessService"
            expression="execution(* com.xyz.service.*.*(..)) and this(service)"/>
        <aop:before pointcut-ref="businessService" method="monitor"/>
        ...
    </aop:aspect>
</aop:config>
~~~

## Advice

基于schema的AOP支持使用了与 @AspectJ 风格相同的五种advice，而且它们的语义完全相同，可以使用下面标签声明advice，注意这些标签只能在`<aop:aspect>`中声明：

* `<aop:beofre>`
* `<aop:after-returning>`
  * `returning`指定返回值的参数名,该值需要与方法声明中的参数名称相同
* `<aop:after-throwing>`
  * `throwing`:指定异常的参数名称，该值要与方法声明中的参数名称相同
* `<aop:after>`
* `<aop:around>`

以上Advice标签都具有以下属性：

* `method `：标识一个提供 advice body 的方法。这个方法必须为当前advice所在的aspect所引用的bean定义(即`<aop:aspect>`标签的ref属性所指定的bean)
* `arg-names`:指定Advice对应的方法的参数名 多个参数名用逗号隔开
* `pointcut-ref`:指定 引用的`<aop:pointcut>`的id
* `pointcut`:指定Pointcut表达式

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
    </aop:aspect>
</aop:config>
~~~

`AspectDemo`的声明如下：

~~~java
public class AspectDemo {

    public void doBefore(JoinPoint joinpoint){System.out.println("doBefore:"+joinpoint.getTarget());}

    public void doAfterReturning(Object result){System.out.println("doAfterReturning:"+result);}

    public void doAfterThrowing(RuntimeException e){System.out.println("doAfterThrowing:"+e.getMessage());}

    public void doAfter(JoinPoint joinPoint){System.out.println("doAfter:"+joinPoint.getTarget());}

    public void doAround(JoinPoint joinPoint){System.out.println("doAround:"+joinPoint.getTarget());}

}
~~~

## Introduction

可以通过在 `aop:aspect` 里面使用 `aop:declaration-parents` 元素来做一个introduction,`<aop:declare-parents>`属性如下：

* `type-matching`:指定将要对那些目标对象进行Introduction逻辑织入
* `implement-interface`：指定新增加的Introduction行为的接口定义类型
* `default-impl`:指定增加的Introduction行为的接口定义的默认实现类

一个示例：

~~~xml
<aop:aspect id="usageTrackerAspect" ref="usageTracking">
    <aop:declare-parents
        types-matching="com.xyz.service.*+"
        implement-interface="com.xyz.service.tracking.UsageTracked"
        default-impl="com.xyz.service.tracking.DefaultUsageTracked"/>
    <aop:before
        pointcut="execution(* com.xyz..service.*.*(..))
            and this(usageTracked)"
            method="recordUsage"/>
</aop:aspect>
~~~

支持 `usageTracking` bean的类将包含以下方法。

~~~java
public void recordUsage(UsageTracked usageTracked) {
    usageTracked.incrementUseCount();
}
~~~

## Advisor

"advisor" 的概念来自于Spring中定义的AOP支持，在AspectJ中并没有直接的对应。advisor就像一个小的独立的切面，它有一个单一的advice。advice本身由一个bean表示，并且必须实现 Spring 的 Advice Type中描述的advice接口之一。advisor可以利用AspectJ的pointcut表达式。

`<aop:advisor>`用以定义Advisor，它有如下属性：

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

其中`MyMethodInterceptor`定义如下:

~~~java
public class MyMethodInterceptor implements MethodInterceptor {
    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        System.out.println("执行前");
        Object proceed = invocation.proceed();
        System.out.println("执行后");
        return proceed;
    }
}
~~~

`org.aopalliance.intercept.MethodInterceptor`间接实现了`org.aopalliance.aop.Advice`接口

实际上Advisor常与 transactional advice 一起使用：

~~~xml
<aop:config>
    <aop:pointcut id="businessService" expression="execution(* com.xyz.service.*.*(..))"/>
    <aop:advisor pointcut-ref="businessService" advice-ref="tx-advice" />
</aop:config>

<tx:advice id="tx-advice">
    <tx:attributes>
        <tx:method name="*" propagation="REQUIRED"/>
    </tx:attributes>
</tx:advice>
~~~
