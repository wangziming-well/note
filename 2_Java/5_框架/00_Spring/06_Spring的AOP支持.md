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

