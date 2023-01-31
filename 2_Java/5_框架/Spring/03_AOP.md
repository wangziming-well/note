# AOP面向切面编程

在生产环境中，经常有业务交叉的情况；为了实现业务交叉时的解耦和，产生了面向切面的思想。

通过动态代理的方式，将增强类的功能插入到目标类中



## 动态代理

​	spring 底层使用动态代理来实现aop

支持两种动态代理方式:

* JDK动态代理：代理的类必须实现接口，通过实现接口来实现代理类
* CGLIB动态代理：代理的类不需要接口，通过生成类的子类来实现代理

## AOP术语

* 切面Aspect

  泛指交叉业务逻辑， 是通知和切点的结合

* 连接点JoinPoint

  指被切面织入的具体方法

* 切入点Pointcut

  指声明的一个或多个连接点的集合

* 目标对象Target

  指要被增强的

* 通知Advice

  通知描述了切面何时执行以及如何执行增强处理

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

