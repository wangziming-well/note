# IoC基本概念

在传统的Java SE的程序设计中，如果当前类依赖于某个类，我们会在类的构造函数中新建相应的依赖类对象，这样主动获取依赖的对象的设计会造成程序的高耦合

而控制反转(Inversion of Control)将由程序代码直接操控创建的对象调用权交给容器，通过容器来实现对象的创建与依赖管理，实现了优秀的插拔能力(高内聚，低耦合)和更高效清晰的对象管理。这样的容器就是IoC容器。

这样通过使用Ioc容器，获取依赖对象的方式发送了反转，由被依赖对象主动获取变成了由IoC容器将依赖对象交给被依赖对象，这样对象具有更好的可测试性、可重用性和可扩展性

依赖注入(Dependency Injection) 组件之间的依赖关系由容器来管理，应用程序依赖IoC容器，需要IoC容器来提供对象需要的外部资源(对象，文件，数据),

IoC和DI是对同一个概念的不同角度的描述

# IoC Service Provider

IoC Service Provider 是一个抽象出来的概念，它可以代指任何将IoC场景中的业务对象绑定到一起的实现方式。Spring的IoC容器是一个IoC Service Provider

IoC Service Provider 主要职责有两个：

* 业务对象的构建管理：在IoC场景中，业务对象无需创建对象，这部分工作交给了IoC Service Provider
* 业务对象间的依赖绑定：IoC Service Provider通过结合之前构建和管理的所有业务对象，以及各个业务对象间可以识别的依赖关系，将这些对象所依赖的对象注入绑定，从而保证每个业务对象在使用的时候，可以处于就绪状态  

为了完成上述职责，IoC Service Provider 需要记录注册对象的信息(对象间的依赖关系等)：

* 直接编码方式：在容器启动之前，我们就可以通过程序编码的方式将被注入对象和依赖对象注册到容器中，并明确它们相互之间的依赖注入关系
* 配置文件方式：最常见的是通过XML文件来管理对象注册和对象间依赖关系  
* 元数据方式：直接在类中使用元数据(注解？)信息来标注各个对象之间的依赖关系

# Spring的IoC容器

Spring的IoC容器是一个IoC Service Provider ，但是它除了提供基本的IoC支持外，还提供AOP框架支持、企业级服务集成等服务。

Spring提供了两种容器类型：

* `BeanFactory`:基础类型IoC容器，提供完整的IoC服务支持;默认采用延迟初始化策略;所以，相对来说，容器启动初期速度较快，所需要的资源有限。对于资源有限，并且功能要求不是很严格的场景， `BeanFactory`是比较合适的IoC容器选择。  
* `ApplicationContext`:`ApplicationContext`在`BeanFactory`的基础上构建，是相对比较高级的容器实现，除了拥有`BeanFactory`的所有支持， `ApplicationContext`还提供了其他高级特性，比如事件发布、国际化信息支持等  `ApplicationContext`所管理的对象，在该类型容器启动之后，默认全部初始化并绑定完成。所以，相对于`BeanFactory`来说， `ApplicationContext`要求更多的系统资源，同时，因为在启动时就完成所有初始化，容器启动时间较之`BeanFactory`也会长一些。在那些系统资源充足，并且要求更多功能的场景中，`ApplicationContext`类型的容器是比较合适的选择。  

他们的继承关系如下：

![ApplicationContext](https://gitee.com/wangziming707/note-pic/raw/master/img/ApplicationContext.png)

可以看出ApplicationContext接口间接继承自BeanFactory,所以说它是构建于BeanFactory之上的IoC容器

# BeanFactory

`BeanFactory`就是生产Bean的工厂。作为Spring提供的基本IoC容器，`BeanFactory`可以完成作为IoC Service Provider的所有职责，包括业务对象的注册和对象间依赖关系的绑定。

下面是`BeanFactory`的定义：

~~~java
public interface BeanFactory {

	String FACTORY_BEAN_PREFIX = "&";

	Object getBean(String name) throws BeansException;

	<T> T getBean(String name, Class<T> requiredType) throws BeansException;

	Object getBean(String name, Object... args) throws BeansException;

	<T> T getBean(Class<T> requiredType) throws BeansException;

	<T> T getBean(Class<T> requiredType, Object... args) throws BeansException;

	<T> ObjectProvider<T> getBeanProvider(Class<T> requiredType);

	<T> ObjectProvider<T> getBeanProvider(ResolvableType requiredType);

	boolean containsBean(String name);

	boolean isSingleton(String name) throws NoSuchBeanDefinitionException;

	boolean isPrototype(String name) throws NoSuchBeanDefinitionException;

	boolean isTypeMatch(String name, ResolvableType typeToMatch) throws NoSuchBeanDefinitionException;

	boolean isTypeMatch(String name, Class<?> typeToMatch) throws NoSuchBeanDefinitionException;

	@Nullable
	Class<?> getType(String name) throws NoSuchBeanDefinitionException;

	@Nullable
	Class<?> getType(String name, boolean allowFactoryBeanInit) throws NoSuchBeanDefinitionException;

	String[] getAliases(String name);
}
~~~

以上代码的方法基本都是查询相关：getBean 获取对象；containsBean 查询对象是否存在；getType 查询对象类型等

## BeanFactory的对象注册与依赖绑定方式

`BeanFactory`作为一个IoC Service Provider 为了能明确管理对象以及对象间的依赖关系，同样需要记录和管理这些信息。`BeanFactory`支持之前提到的三种方式管理

### 直接编码方式

可以使用 `DefaultListableBeanFactory `对象实现对象的注册和绑定

它的继承关系如下：

![ApplicationContext](https://gitee.com/wangziming707/note-pic/raw/master/img/DefaultListableBeanFactory.png)

可以看出它除了间接实现了`BeanFactory`接口，同样实现了`BeanDefinitionRegistry`接口

* `BeanFactory`接口只定义如何访问容器内管理的Bean的方法

* `BeanDefinitionRegistry`接口定义抽象了Bean的注册逻辑

通常情况下，具体的`BeanFactory`实现`BeanFactory`接口，还要实现`BeanDefinitionRegistry`接口来管理Bean的注册

`BeanDefinitionRegistry`定义了管理`BeanDefinition`实例的规范

在容器中的每个对象都有个`BeanDefinition`实例与之对应，该`BeanDefinition`的实例负责保存对象的所有必要信息，包括其对应的class类型、是否是抽象类、构造方法参数以及其他属性等

当客户端向`BeanFactory`请求相应对象时，`BeanFactory`会通过`BeanDefinition`保存的对应对象信息为客户端返回一个完备可用的对象实例

`BeanDefinition`的两个主要实现类是  `RootBeanDefinition`和 `ChildBeanDefinition`



**实例演示**

一个代码注册对象和注册绑定关系的实例：

~~~java
public class App {
    public static void main( String[] args ) {
        DefaultListableBeanFactory beanRegistry = new DefaultListableBeanFactory();
        BeanFactory container = bindViaCode(beanRegistry);
        Person person = (Person) container.getBean("person");
        System.out.println(person.getPets().getName());
    }

    public static BeanFactory bindViaCode(BeanDefinitionRegistry registry){
		//创建definition实例
        RootBeanDefinition personDefinition = new RootBeanDefinition(Person.class);
        RootBeanDefinition dogDefinition = new RootBeanDefinition(Dog.class);
        //通过构造方法注入指定依赖关系
        ConstructorArgumentValues argValues = new ConstructorArgumentValues();
        argValues.addIndexedArgumentValue(0,10);
        argValues.addIndexedArgumentValue(1,dogDefinition);
        personDefinition.setConstructorArgumentValues(argValues);
        //通过setter方法注入属性值
        MutablePropertyValues propertyValues = new MutablePropertyValues();
        propertyValues.addPropertyValue(new PropertyValue("name","www"));
        propertyValues.addPropertyValue(new PropertyValue("age",10));
        dogDefinition.setPropertyValues(propertyValues);
        //将bean定义注册到容器中
        registry.registerBeanDefinition("person",personDefinition);
        registry.registerBeanDefinition("dog",dogDefinition);
        return (BeanFactory) registry;
    }
}
~~~

对应pojo类：

~~~java
//pojo类
@Data
public class Dog {
    String name;
    String age;
}

@Data
public class Person {
    public Person(int age, Dog pets) {
        this.age = age;
        this.pets = pets;
    }
    int age;
    Dog pets;
}
~~~

### 配置文件方式

Spring 的IoC容器常用的配置文件格式是XML、其他的也支持properties文件（已过时）、groovy文件

采用外部配置文件时，Spring的IoC容器有一个统一的处理方式：

* 根据不同的外部配置文件格式，给出相应的`BeanDefinitionReader`实现类
* `BeanDefinitionReader`的相应实现类负责将相应的配置文件内容读取并映射到`BeanDefinition  `
* 然后将映射后的`BeanDefinition`注册到一个`BeanDefinitionRegistry  `
* `BeanDefinitionRegistry`即完成Bean的注册和加载

`BeanDefinitionReader  `完成解析文件格式、装配`BeanDefinition`之类的工作  

`BeanDefinitionRegistry `负责保存`BeanDefinition`

整个过程类似如下代码：

~~~java
BeanDefinitionRegistry beanRegistry = <某个BeanDefinitionRegistry实现类。通常为
DefaultListableBeanFactory>;
BeanDefinitionReader beanDefinitionReader = new BeanDefinitionReaderImpl(beanRegistry);
beanDefinitionReader.loadBeanDefinitions("配置文件路径");
~~~

当前最常用的配置文件为XML,以上过程可以由`XmlBeanDefinitionReader  `完成

#### XML配置格式的加载

配置文件 spring.xml:

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
    <bean id="person" class="org.example.pojo.Person">
        <property name="age" value="123"/>
        <property name="pets" ref="dog"/>
    </bean>

    <bean id="dog" class="org.example.pojo.Dog">
        <property name="age" value="10"/>
        <property name="name" value="aaa"/>
    </bean>
</beans>
~~~

pojo类：

~~~java
//pojo类
@Data
public class Dog {
    String name;
    String age;
}

@Data
public class Person {
    public Person(int age, Dog pets) {
        this.age = age;
        this.pets = pets;
    }
    int age;
    Dog pets;
}
~~~

加载XML文件的BeanFactroy使用实例：

~~~java
DefaultListableBeanFactory beanFactory = new DefaultListableBeanFactory();
XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(beanFactory);
reader.loadBeanDefinitions("spring.xml");
Person person =(Person) beanFactory.getBean("person");
System.out.println(person.getPets().getName());
~~~

### 注解方式

实体类：

~~~java
@Data
@Component
public class Person {
    int age =18;
    @Autowired
    Dog pets;
}

@Data
@Component
public class Dog {
    String name ="www";
    int age = 3;
}
~~~

配置文件spring.xml:

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/context
       https://www.springframework.org/schema/context/spring-context.xsd">
    <!--扫描指定包下的所有注解-->
    <context:component-scan base-package="org.example.pojo"/>
</beans>
~~~

使用容器：

~~~java
ClassPathXmlApplicationContext container = new ClassPathXmlApplicationContext("spring.xml");
Person person = (Person) container.getBean("person");
System.out.println(person.getPets().getName());
~~~

## BeanFactory分层

BeanFactory可以通过实现HierarchicalBeanFactory接口实现分层

容器A在初始化的时候，可以首先加载容器B中的所有对象定义，然后再加载自身的对象定义，这样，容器B就成为了容器A的父容器，容器A可以引用容器B中的所有对象定义：  

~~~java
BeanFactory parentContainer = new XmlBeanFactory(new ClassPathResource("父容器配置文件路
径"));
BeanFactory childContainer = new XmlBeanFactory(new ClassPathResource("子容器配置文件路
径"),parentContainer);
~~~







# XML加载spring容器

XML格式的容器信息管理方式是Spring提供的最为强大、支持最为全面的方式  

## XML头文件约束

所有使用XML文件进行配置信息加载的Spring IoC容器，包括BeanFactory和ApplicationContext的所有XML对应实现，都使用统一的XML格式。有相同的头文件约束：

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
  http://www.springframework.org/schema/beans/spring-beans-4.1.xsd">
    <!--beans  就是对象容器-->
</beans>
~~~

## `<beans>`

beans时XML配置文件中最顶层的元素,它的层级结构如下：

* `<beans>`:`[default-lazy-init|default-autowire|default-dependency-check|default-init-method|default-destroy-method]`
  * `<import>`:`[resource]` 可以引用外部其他xml配置文件 resource指定外部配置文件位置
  * `<alias>`:`[name|alias]` 可以为指定的bean起别名；属性name为要起别名beanId 属性alias是指定的别名
  * `<bean>`：描述对应容器的bean对象

## `<bean>`

### 标签结构

* `<bean>`:`[id|name|class|depends-on|autowire|parent|abstract|scope|lazy-init|factory-method|factory-bean]`
  * `<constructor-arg>&<property>`
    * `<value>`
    * `<ref>` 
    * `<idref>` 
    * `<bean>`
    * `<list>&<set>&<array>`
    * `<map>` 
    * `<props>`
    * `<null>`

### 属性

通过设置bean标签的属性，控制容器管理bean 对象的行为

#### `id`

id为bean对象在容器中的唯一标识，该id属性值必须在IoC容器中唯一

大部分情况下我们通过id找到容器中想要的对象

可以不指定id属性，只指定全限定类名

此时只能通过`getBean(Class<T> requiredType)`来获取Bean，此时如果找不到bean或者找到了多个bean都会抛异常

#### `name`

用来指定bean的别名(alias)，name也需要保证在容器中的唯一性

支持设置多个别名，之间用英文逗号分割

**补充说明:**

* 如果不指定id，只指定name，那么name为Bean的标识符，并且需要在容器中唯一

* 同时指定name和id，此时id为标识符，而name为Bean的别名，两者都可以找到目标Bean

#### `class`

值为类的全限定名，指定bean对象的类型

大部分情况下该属性是必须的

#### `depends-on`

值为 beanName ；可以有多个用`,`分割

显示地声明类的依赖关系，让`depends-on`指定的bean对象先于当前bean实例化

**bean对象的加载顺序:**

* 没有依赖关系时，bean对象加载进容器的顺序为`<bean>`的声明顺序

* 如果使用`<ref>`的元素明确指定对象的依赖关系，spring会自动根据依赖关系制定bean的加载顺序策略

* 而在有的需求中,bean对象之间没有显式的依赖关系，但需要让容器在实例化对象A之前首先实例化对象B,此时就可以通过该属性显式地指定依赖关系

#### `autowire`

除了通过配置明确指定bean 之间的依赖关系，Spring还提供了根据bean定义的某些特点将相
互依赖的某些bean直接自动绑定的功能

通过该属性，可以指定bean的自动绑定模式，

* `no`:默认值，不采用自动绑定
* `byName`根据类中声明的实例变量名称，与容器中bean的beanName进行匹配，相匹配的bean将自动绑定到该实例变量(通过setter方法)
* `byType`根据类中声明的实例变量类型，到容器中寻找相同类型的bean，并将匹配的bean自动绑定到该实例变量上(通过setter方法)
  * 如果没有找到，将不进行绑定
  * 如果找到一个，将自动绑定
  * 如果找到多个，将抛出异常`NoUniqueBeanDefinitionException`
* `constructor`针对构造方法参数类型进行自动绑定，在容器中寻找与构造方法参数类型匹配的bean类型
  * 如果匹配时有多个符合，将报错
  * 如果有多个构造方法，将按照构造方法参数个数，从最多到最少的顺序进行匹配

**注意：**

* 手工明确指定的绑定关系总会覆盖自动绑定模式的行为
* 自 动绑定对“原生类型、 String类型和Classes类型”以及“这些类型的数组”是无效的。  

**补充说明一：**自动绑定和手动绑定各有利弊，需要根据业务场景选择

自动绑定优点：

* 减少配置信息的工作量
* 某些情况下，即使为当前对象增加了新的依赖关系，但只要容器中存在相应的依赖对象，就不需要更改任何配置信息。

自动绑定的缺点：

* 自动绑定不如明确依赖关系一目了然
* 某些情况下，自动绑定无法满足系统需要，甚至导致系统行为异常或者不可预知。  
* 使用自动绑定，我们可能无法获得某些工具的良好支持，比如Spring IDE。

**补充说明二：**

`<beans>`标签有一个`default-autowire`可以指定所有`bean`的默认自动绑定模式

#### `parent`

值为容器中的beanName 以指定继承关系

设置后将继承父bean 的所有property标签属性

如果子bean 有同名property ，将覆盖父bean

#### `abstract`

值为`true`或`false`

指定bean是否为抽象的，如果是抽象的，将不会被加载进容器中

* 当bean是抽象bean时，可以不用指定 bean 的 class 属性

* 抽象bean配合parent可以将bean定义模板化

#### `scope`

scope用来声明bean对象的生命周期或者说限定场景

配置中的bean定义可以看作是一个模板，容器会根据这个模板来构造对象。  

而scope决定了这个模板构造多少对象实例，又该让这些构造完的对象实例存活多久  

spring提供了两种基本的scope类型:

* `singleton`是bean的默认scope类型，该类型的bean：
  * 在Spring的IoC容器中只存在一个实例，所有对该对象的引用将共享这个实例。
  * 如果该bean不是懒加载的，那么它的生命周期几乎和IoC容器一样

* `prototype`针对该类型的bean：
* 容器在接到该类型对象的请求的时候，会每次都重新生成一个新的对象实例给请求方
  
* 并且容器就不再拥有当前返回对象的引用  请求方需要自己负责当前返回对象的后继生命周期的管理工作，包括该对象的销毁

如果spring整合了web，那么针对web的域，spring还提供了额外的scope类型：

* `request`Spring 容 器 ， 即XmlWebApplicationContext会 为 每 个 HTTP 请 求 创建 一 个 全 新的RequestProcessor对象供当前请求使用，当请求结束后，该对象实例的生命周期即告结束。
* `session`Spring容器会为每个独立的session创建属于它们自己的全新的UserPreferences对象实例

#### `lazy-init`

默认情况下，ApplicationContext容器在初始化时，会实例化所有的bean

可以通过`lazy-init`控制该行为，当`lazy-init`为`true`时，该bean会只有在被用到时才会被实例化(类似与单例模式的懒汉式)

**补充说明:**`<beans>`标签有属性`default-lazy-init`属性，控制所有bean的默认懒加载行为

#### `factory-method`

该属性的值为工厂类的获取实例的方法，针对的是静态工厂方法

通过工厂类获取实例的方法获取bean对象

此时需要class为工厂类，factory-method为获取实例的方法时

该bean的类型为工厂类生成的类的类型

如果获取实例方法有入参，可以通过`<constructor-arg>`标签为获取实例方法提供入参

#### `factory-bean  `

针对非静态工厂，我们需要实例化工厂再获取工厂生产的对象

此时可以用`factory-bean`代替静态工厂使用的`class`属性

### 子标签

spring通过配置bean的子标签控制**依赖注入**行为

#### `<constructor-arg>  `

bean对象在初始化时，默认使用无参构造

可通过该标签使用有参构造进行实例化，注入属性成员值

* `name`构造函数形参名，指定该标签是为那个形参赋值
* `index` 构造函数形参列表索引，指定为第几个形参赋值
* `value`如果关联的参数是基本数据类型，可通过该属性指定形参值
* `type`指定关联的参数的类型
* `ref`如果关联的参数是引用数据类型，可通过该属性指定形参，属性值为beanName

#### `<property>  `

bean对象初始化时，也可以使用setter方法进行成员变量的初始化

* `name`要注入的成员属性名
* `value`注入的值，如果是基本数据类型
* `ref`注入的值，如果是引用数据类型

**注意：**使用使用`<property>`的setter方法注入时，要保证对象提供了默认的无参构造

#### `<constructor-arg>`和`<property>  `的子标签

spring提供功能丰富的子标签，以方便注入集合，列表，map等常用容器

子标签如下：

* `<value>`功能同value

  * `type`指定要注入的值

* `<ref>` 功能同ref属性

  * `bean `指定要注入的bean对象
  * `parent`指定父容器中定义的队形

* `<idref>` 类似于value，但多了一层检查，值必须属于容器中的beanName集合

  * `bean`要注入的String值，必须是被容器管理的beanName

* `<bean>`  内部bean，如普通bean功能一样，但只能被当前类调用

  * 因为就只有当前对象引用内部`<bean>`所指定的对象，所以，内部`<bean>`的id不是必须的  

* `<list>&<set>&<array>`  注入列表，集合，数组

  * `value-type`指定集合泛型
  * `merge`在bean继承关系中，对于集合属性，如果重写了配置属性，不设置merge属性，则也是重写了该集合。当merge=true，则将这个集合与父bean配置的集合合并。

* `<map>` 注入表 属性有`value-type key-type merge`

  * 该标签只有一个子标签`<entry>`来设置map的键值对

    具有属性：`key value value-type key-ref value-ref`

    特有子标签：`<key> <value>`

* `<props>`注入`java.util.Properties`对象

  * 具有属性`value-type merge`
  * 具有子标签`prop`定义配置项

* `<null>`注入null值

# Bean对象

bean对象是spring框架的核心对象之一

## Bean 生命周期

Spring Bean 的生命周期有四个阶段

* 实例化(Instantiation):当容器启动结束后，实例化所有的bean
* 属性赋值(Populate):依赖注入，Spring根据BeanDefinition中的信息 以及 通过BeanWrapper提供的设置属性的接口完成依赖注入
* 初始化(Initialization)
  * 检查Aware接口，如果bean对象实现了Aware接口，spring会调用接口的set方法，将指定的信息加载到bean中
  * 检查BeanPostProcessor  接口，如果bean对象实现了该接口，那么spring会调用接口的postProcessBeforeInitialization(Object obj, String s)  方法，实现一些自定义的处理
  * InitializingBean 与 init-method  ，如果Bean在Spring配置文件中配置了 init-method 属性，则会自动调用其配置的初始化方法  
  * 检查BeanPostProcessor  接口，如果bean对象实现了该接口，那么spring会调用接口的，postProcessAfterInitialization(Object obj, String s)方法  
* 销毁(Destruction)
  * 检查 DisposableBean  接口，会调用其实现的destroy()方法；  
  * destroy-method  ：如果这个Bean的Spring配置中配置了destroy-method属性，会自动调用其配置的销毁方法。  

## Bean线程安全

spring并未对bean的多线程做封装处理

对于一个单例bean，是否线程安全看它是否是无状态的(不保存数据)，如果是无状态的，那么显然线程是安全的，否则，就有线程安全问题

解决方案：

* 对有状态的bean 可以将它的 scope 设置为 Prototype 保证每个线程获取的对象和对象的数据都是独立的



## 循环依赖

在spring中，两个bean对象互相依赖，那么在创建阶段，这两个对象会因为互相依赖而无法创建成功，两个对象都会卡在属性赋值阶段

解决方案： spring 引入了三级缓存(三个Map)：

- singletonObjects,  一级缓存
- earlySingletonObjects,  二级缓存
- singletonFactories 三级缓存

在bean对象还未创建成功时，就将它的引用地址暴露出去，以供其他对象获取

实例：

1. 对象A要创建到Spring容器中，从一级缓存singletonObject获取A，不存在，开始实例化A，最终在三级缓存singletonObjectFactory添加(A，A的函数式接口创建方法)，这时候A有了自己的内存地址
2. 设置属性B，B也从一级缓存singletonObject获取B，不存在，开始实例化B，最终在三级缓存singletonObjectFactory添加(B，B的函数式接口创建方法)，这时候B有了自己的内存地址
3. B中开始给属性A赋值，此时会找到三级缓存中的A，并将A放入二级缓存中。删除三级缓存
4. B初始化完成，从三级缓存singletonObjectFactory直接put到一级缓存singletonObject，并删除二级和三级缓存的自己
5. A成功得到B，A完成初始化动作，从二级缓存中移入一级缓存，并删除二级和三级缓存的自己
6. 最终A和B都进入一级缓存中待用户使用



# 基于注解的IOC

## 注解扫描

`<context:component-scan>`标签可以配置注解扫描，spring会扫描该标签指定的包下的所有类，并将被注解标记为bean 的对象交给spring容器管理

该标签有如下属性：

* `base-package`指定扫描的包

配置注解扫描：

~~~xml
<!--扫描指定包下的所有注解-->
<context:component-scan base-package="com.bjpn"/>
~~~

## 注解

实体类注解：

* `@controller`表示处理器类
* `@Service`当前类为业务层
* `@Respository`当前类为持久层
* `@Component`普通类

以上注解都有name属性，以指定bean的beanName；不写默认是类名的驼峰式

依赖注入注解：

* `@Autowired`在属性上，表示该属性由容器根据set方法进行依赖注入
* `@Qualifier`辅助注解，指定要注入的类（在属性为接口，且项目中该接口不止有一个实现类时）

除了spring提供的`@Autowired`注解，javax包还提供了`@Resource`同样可以实现依赖注入

但不同的是：@Resource默认按照ByName自动注入  

 
