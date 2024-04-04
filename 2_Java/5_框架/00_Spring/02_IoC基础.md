# IoC概述

在传统的Java SE的程序设计中，如果当前类依赖于某个类，我们会在类的构造函数中新建相应的依赖类对象，这样主动获取依赖的对象的设计会造成程序的高耦合

而控制反转(Inversion of Control)将由程序代码直接操控创建的对象调用权交给容器，通过容器来实现对象的创建与依赖管理，实现了优秀的插拔能力(高内聚，低耦合)和更高效清晰的对象管理。这样的容器就是IoC容器。

这样通过使用Ioc容器，获取依赖对象的方式发送了反转，由被依赖对象主动获取变成了由IoC容器将依赖对象交给被依赖对象，这样对象具有更好的可测试性、可重用性和可扩展性

依赖注入(Dependency Injection) 组件之间的依赖关系由容器来管理，应用程序依赖IoC容器，需要IoC容器来提供对象需要的外部资源(对象，文件，数据),

IoC和DI是对同一个概念的不同角度的描述

## IoC Service Provider

IoC Service Provider 是一个抽象出来的概念，它可以代指任何将IoC场景中的业务对象绑定到一起的实现方式。Spring的IoC容器是一个IoC Service Provider

IoC Service Provider 主要职责有两个：

* 业务对象的构建管理：在IoC场景中，业务对象无需创建对象，这部分工作交给了IoC Service Provider
* 业务对象间的依赖绑定：IoC Service Provider通过结合之前构建和管理的所有业务对象，以及各个业务对象间可以识别的依赖关系，将这些对象所依赖的对象注入绑定，从而保证每个业务对象在使用的时候，可以处于就绪状态

为了完成上述职责，IoC Service Provider 需要记录注册对象的信息(对象间的依赖关系等)：

* 直接编码方式：在容器启动之前，我们就可以通过程序编码的方式将被注入对象和依赖对象注册到容器中，并明确它们相互之间的依赖注入关系
* 配置文件方式：最常见的是通过XML文件来管理对象注册和对象间依赖关系  
* 元数据方式：直接在类中使用元数据(注解)信息来标注各个对象之间的依赖关系

## Spring的IoC容器

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

## 对象注册与依赖绑定方式

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
BeanFactory parentContainer = new XmlBeanFactory(new ClassPathResource("父容器配置文件路径"));
BeanFactory childContainer = new XmlBeanFactory(new ClassPathResource("子容器配置文件路径"),parentContainer);
~~~

# XML配置容器

XML格式的容器信息管理方式是Spring提供的最为强大、支持最为全面的方式

## XML配置文件结构

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

beans时XML配置文件中最顶层的元素,它的层级结构如下：

* `<beans>`:`[default-lazy-init|default-autowire|default-dependency-check|default-init-method|default-destroy-method]`
  * `<import>`:`[resource]` 可以引用外部其他xml配置文件 resource指定外部配置文件位置
  * `<alias>`:`[name|alias]` 可以为指定的bean起别名；属性name为要起别名beanId 属性alias是指定的别名
  * `<bean>`：描述对应容器的bean对象

beans标签最主要的子标签是bean标签，用以声明容器中的bean，其结构如下：

* `<bean>`:`[id|name|class|depends-on|autowire|parent|abstract|scope|lazy-init|factory-method|factory-bean|init-method]`
  * `<constructor-arg>&<property>`
    * `<value>`
    * `<ref>` 
    * `<idref>` 
    * `<bean>`
    * `<list>&<set>&<array>`
    * `<map>` 
    * `<props>`
    * `<null>`
  * `<lookup-method>`

## bean属性

通过设置bean标签的属性，控制容器管理bean 对象的行为，下面介绍bean常用的属性，其他重要属性会单独介绍：

* `id`：为bean对象在容器中的唯一标识，该id属性值必须在IoC容器中唯一，大部分情况下我们通过id找到容器中想要的对象可以不指定id属性，只指定全限定类名此时只能通过`getBean(Class<T> requiredType)`来获取Bean，此时如果找不到bean或者找到了多个bean都会抛异常

* `name`：用来指定bean的别名(alias)，name也需要保证在容器中的唯一性支持设置多个别名，之间用英文逗号分割，注意：

  * 如果不指定id，只指定name，那么name为Bean的标识符，并且需要在容器中唯一

  * 同时指定name和id，此时id为标识符，而name为Bean的别名，两者都可以找到目标Bean

* `class`：值为类的全限定名，指定bean对象的类型大部分情况下该属性是必须的

* `depends-on`：值为 beanName ，可以有多个用`,`分割，显示地声明类的依赖关系，让`depends-on`指定的bean对象先于当前bean实例化

  **bean对象的加载顺序:**

  * 没有依赖关系时，bean对象加载进容器的顺序为`<bean>`的声明顺序


  * 如果使用`<ref>`的元素明确指定对象的依赖关系，spring会自动根据依赖关系制定bean的加载顺序策略


  * 而在有的需求中,bean对象之间没有显式的依赖关系，但需要让容器在实例化对象A之前首先实例化对象B,此时就可以通过该属性显式地指定依赖关系


* `parent`：值为容器中的beanName 以指定继承关系,设置后将继承父bean 的所有property标签属性，如果子bean 有同名property ，将覆盖父bean

* `abstract`：值为`true`或`false`，指定bean是否为抽象的，如果是抽象的，将不会被加载进容器中

  * 当bean是抽象bean时，可以不用指定 bean 的 class 属性


  * 抽象bean配合parent可以将bean定义模板化


* `lazy-init`，默认情况下，ApplicationContext容器在初始化时，会实例化所有的bean，可以通过`lazy-init`控制该行为，当`lazy-init`为`true`时，该bean会只有在被用到时才会被实例化(类似与单例模式的懒汉式)

  **补充说明:**`<beans>`标签有属性`default-lazy-init`属性，控制所有bean的默认懒加载行为

* `init-method`指定bean的方法名，在对象初始化时调用指定方法，作用同`InitializingBean  `

* `destroy-method  `指定bean的方法名，在对象销毁时调用指定方法，作用同`DisposableBean  `

## bean生命周期

bean的scope属性用来声明bean对象的生命周期或者说限定场景

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

## 自动绑定

除了通过配置明确指定bean 之间的依赖关系，Spring还提供了根据bean定义的某些特点将相互依赖的某些bean直接自动绑定的功能.

通过bean标签的autowire属性，可以指定bean的自动绑定模式：

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

## 工厂方法

### 静态工厂方法

bean标签的`factory-method`属性的值为工厂类的获取实例的方法，针对的是静态工厂方法

通过工厂类获取实例的方法获取bean对象:

~~~xml
<!-- class为工厂类 -->
<!-- actory-method为获取实例的方法 -->
<bean id="bar" class="...StaticBarInterfaceFactory" factory-method="getInstance"/>
<!-- 该bean的类型为工厂类生成的类的类型 -->
~~~

如果获取实例方法有入参，可以通过`<constructor-arg>`标签为获取实例方法提供入参

此时使用`<constructor-arg>`传入的是工厂方法的参数，而不是静态工厂方法实现类的构造方法的参数  

~~~xml
<bean id="bar" class="...StaticBarInterfaceFactory" factory-method="getInstance">
    <constructor-arg>
        <ref bean="foobar"/>
    </constructor-arg>
</bean>
~~~

### 非静态方法

针对非静态工厂，我们需要实例化工厂再获取工厂生产的对象

此时可以用`factory-bean`代替静态工厂使用的`class`属性

~~~xml
<!-- 将工厂类注册到容器 -->
<bean id="barFactory" class="...NonStaticBarInterfaceFactory"/>
<!-- 指定factory-bean -->
<bean id="bar" factory-bean="barFactory" factory-method="getInstance"/>
~~~

### FactoryBean

FactoryBean是Spring容器提供的一种可以扩展容器对象实例化逻辑的接口，当

* 某些对象的实例化过程过于烦琐，通过XML配置过于复杂
* 某些第三方库不能直接注册到Spring容器  

可以考虑实现org.springframework.beans.factory.FactoryBean  接口，给出自己的对象实例化逻辑代码  

接口如下:

~~~java
public interface FactoryBean<T> {

	String OBJECT_TYPE_ATTRIBUTE = "factoryBeanObjectType";
	//返回生产的对象实例
	@Nullable
	T getObject() throws Exception;
	//返回的对象的类型
	@Nullable
	Class<?> getObjectType();
	//所“生产”的对象是否要以singleton形式存在于容器中
	default boolean isSingleton() {
		return true;
	}
}
~~~

这样实现了FactoryBean接口的bean定义，通过正常的id引用，容器返回的是FactoryBean所“生产”的对象类型，而非FactoryBean实现本身  

如果一定要取得FactoryBean本身的话，可以通过在bean定义的id之前加前缀`& `

实例：

实现了FactoryBean接口的类：

~~~java
public class NextDayDateFactoryBean implements FactoryBean {
    public Object getObject() throws Exception {
        return new DateTime().plusDays(1);
    }
    public Class getObjectType() {
        return DateTime.class;
    }
    public boolean isSingleton() {
        return false;
    }
}
~~~

xml定义：

~~~xml
<bean id="nextDayDate" class="...NextDayDateFactoryBean">
~~~

获取容器中对象：

~~~java
//获取下一天的DateTime
Object nextDayDate = container.getBean("nextDayDate");
//获取工厂类本类的实例
Object factoryBean = container.getBean("&nextDayDate");
~~~





## 依赖注入

spring通过配置bean的子标签控制**依赖注入**行为

### `<constructor-arg>  `

bean对象在初始化时，默认使用无参构造

可通过该标签使用有参构造进行实例化，注入属性成员值

* `name`构造函数形参名，指定该标签是为那个形参赋值
* `index` 构造函数形参列表索引，指定为第几个形参赋值
* `value`如果关联的参数是基本数据类型，可通过该属性指定形参值
* `type`指定关联的参数的类型
* `ref`如果关联的参数是引用数据类型，可通过该属性指定形参，属性值为beanName

### `<property>  `

bean对象初始化时，也可以使用setter方法进行成员变量的初始化

* `name`要注入的成员属性名
* `value`注入的值，如果是基本数据类型
* `ref`注入的值，如果是引用数据类型

**注意：**使用使用`<property>`的setter方法注入时，要保证对象提供了默认的无参构造

### `<constructor-arg>`和`<property>  `的子标签

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

### `<replaced-method>`

该标签可以实现方法替换

可以用新的方法实现覆盖掉原来某个方法的实现逻辑  

它有如下属性：

* name:要替换的方法名
* replacer:指定替换者类，该类必须实现MethodReplacer接口，会将方法替换成reimplement方法

使用该标签需要用到`MethodReplacer`接口：

~~~java
public interface MethodReplacer {
	Object reimplement(Object obj, Method method, Object[] args) throws Throwable;
}
~~~

它有子标签`<arg-type>`

如果要替换的方法存在参数，或者对象存在多个重载的方法，可以通过`<arg-type>`明确指定将要替换的方法参数类型  

实例：

~~~java
public class Demo {

    public void print(){
        System.out.println("我是原函数");
    }
}

public class DemoReplacer implements MethodReplacer {
    
    public Object reimplement(Object obj, Method method, Object[] args) throws Throwable {
        System.out.println("我是替换后函数");
        return null;
    }
}
~~~

配置文件：

~~~xml
<bean id="demoReplacer" class="org.example.pojo.DemoReplacer"/>
<bean id="demo" class="org.example.pojo.Demo">
    <replaced-method name="print" replacer="demoReplacer"/>
</bean>
~~~



## 方法注入

在大多数场景中，容器中的大部分bean都是单例的，当一个单例bean依赖于另一个单例bean、或者一个非单例bean依赖于另一个非单例bean时，可以直接将一个bean定义为另一个bean的属性来处理这种依赖关系。

但是，这种方法在依赖关系的两个bean生命周期不同的时候就会出现问题，例如一个单例bean A需要使用非单例bean B，可能每次在A上调用方法上都需要B，但是为了线程安全或者其他考虑，每次调用时都需要请求一个新的B实例。因为容器只创建一次单例Bean A，所以只有一次设置属性的机会，此时容器无法每次为A提供B的新实例。

要解决这个问题，一种方法是使用`Aware`接口直接注入容器，在需要beanB时，直接调用容器的`getBean()`方法。这种方法并不理想，因为业务代码已经和Spring框架耦合了。所以Spring提供了方法注入的高级特性，用以处理这种情况。

### BeanFactoryAware接口

`BeanFactory`的`getBean`方法每次调用都会取得新的对象实例。

所以想让对象拥有同样的功能，只需要让对象拥有一个`BeanFactory`的引用即可

Spring框架提供了一个BeanFactoryAware接口，容器在实例化实现了该接口的bean定义的过程中，会自动将容器本身注入该bean。 这样， 该bean就持有了它所处的BeanFactory的引用,它的定义如下：

~~~java
public interface BeanFactoryAware extends Aware {

	void setBeanFactory(BeanFactory beanFactory) throws BeansException;

}
~~~

这样我们可以通过实现该接口：

~~~java
@Data
public class Person implements BeanFactoryAware {
    private BeanFactory beanFactory;
    public Dog getDog() {
        return (Dog)beanFactory.getBean("dog");
    }
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
        this.beanFactory = beanFactory;
    }
}
~~~

配置为：

~~~xml
<bean id="person"  class="org.example.pojo.Person" autowire="constructor" scope="prototype">
</bean>
<bean id="dog" class="org.example.pojo.Dog" scope="prototype">
    <property name="name" value="www"/>
    <property name="age" value="3"/>
</bean>
~~~

这样每次调用getDog方法，获取的都是不同的Dog实例

### `<lookup-method>`

bean的子标签`<lookup-method>`可以实现方法注入。

Spring框架通过使用CGLIB库生成字节码来动态生成一个覆盖`<lookup-method>`指定的方法的子类，让这个子类的返回值为容器中指定的bean，每次调用时都会重新从容器中请求。

该标签有两个属性：

* `name:`指定注入的方法名
* `bean:`指定方法返回的对象

因为需要实现动态代理，所以`<lookup-method>`指定的方法签名必须符合如下格式：

~~~java
<public|protected> [abstract] <return-type> theMethodName(no-arguments);
~~~

这样如果lookup method指定的bean类scope是prototype 时，每次调用这个指定方法都从容器中重新申请一次，这样每次调用使用的就会是独立的bean实例了。

例如:

~~~java
package fiona.apple;

// no more Spring imports!

public abstract class CommandManager {

	public Object process(Object commandState) {
		// grab a new instance of the appropriate Command interface
		Command command = createCommand();
		// set the state on the (hopefully brand new) Command instance
		command.setState(commandState);
		return command.execute();
	}

	// okay... but where is the implementation of this method?
	protected abstract Command createCommand();
}
~~~

对应的配置为：

~~~xml
<!-- a stateful bean deployed as a prototype (non-singleton) -->
<bean id="myCommand" class="fiona.apple.AsyncCommand" scope="prototype">
	<!-- inject dependencies here as required -->
</bean>

<!-- commandProcessor uses statefulCommandHelper -->
<bean id="commandManager" class="fiona.apple.CommandManager">
	<lookup-method name="createCommand" bean="myCommand"/>
</bean>
~~~







### ObjectFactoryCreatingFactoryBean

`ObjectFactoryCreatingFactoryBean`可以实现类似的效果，并且同样与业务代码解耦：

`ObjectFactoryCreatingFactoryBean`是spring提供的一个`FactoryBean`实现，它的方法返回一个`ObjectFactory`实例

通过这个实例可以为我们返回容器管理的相关对象  

它的定义如下：

~~~java
public class ObjectFactoryCreatingFactoryBean extends AbstractFactoryBean<ObjectFactory<Object>> {

	@Nullable
	private String targetBeanName;

	public void setTargetBeanName(String targetBeanName) {
		this.targetBeanName = targetBeanName;
	}

	@Override
	public void afterPropertiesSet() throws Exception {
		Assert.hasText(this.targetBeanName, "Property 'targetBeanName' is required");
		super.afterPropertiesSet();
	}

	@Override
	public Class<?> getObjectType() {
		return ObjectFactory.class;
	}

	@Override
	protected ObjectFactory<Object> createInstance() {
		BeanFactory beanFactory = getBeanFactory();
		Assert.state(beanFactory != null, "No BeanFactory available");
		Assert.state(this.targetBeanName != null, "No target bean name specified");
		return new TargetBeanObjectFactory(beanFactory, this.targetBeanName);
	}

	@SuppressWarnings("serial")
	private static class TargetBeanObjectFactory implements ObjectFactory<Object>, Serializable {

		private final BeanFactory beanFactory;

		private final String targetBeanName;

		public TargetBeanObjectFactory(BeanFactory beanFactory, String targetBeanName) {
			this.beanFactory = beanFactory;
			this.targetBeanName = targetBeanName;
		}

		@Override
		public Object getObject() throws BeansException {
			return this.beanFactory.getBean(this.targetBeanName);
		}
	}
}
~~~

它的继承关系如下：

![ObjectFactoryCreatingFactoryBean](https://gitee.com/wangziming707/note-pic/raw/master/img/ObjectFactoryCreatingFactoryBean.png)

通过以上可以看出:

* `ObjectFactoryCreatingFactoryBean`实现了`BeanFactoryAware`接口最终获取对象实例依然是通过`BeanFactoryAware`接口返回的容器对象的引用来完成的

* 它还实现了`FactoryBean`所以通过id将直接获取它生产的对象，这里就是`TargetBeanObjectFactory`的实例

它返回的ObjectFactory实例只是特定于与Spring容器进行交互的一个实现而已。使用它的好处就是，隔离了客户端对象对BeanFactory的直接引用。  

实例：

* spring配置：

~~~xml
<bean id="dog" class="org.example.pojo.Dog" scope="prototype">
    <property name="name" value="www"/>
    <property name="age" value="3"/>
</bean>
<bean id="dogFactory" class="...ObjectFactoryCreatingFactoryBean">
    <property name="targetBeanName">
        <idref bean="dog"/>
    </property>
</bean>
<bean id="person"  class="...Person">
    <property name="dogFactory" ref="dogFactory"/>
</bean>
~~~

* Java代码：

~~~java
@Data
public class Dog {
    String name;
    int age;
}

@Data
public class Person {
    private ObjectFactory<Dog> dogFactory;
    public Dog getDog(){
        return dogFactory.getObject();
    }
}
~~~

