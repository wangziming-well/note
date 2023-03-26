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

#### `factory-bean  `

针对非静态工厂，我们需要实例化工厂再获取工厂生产的对象

此时可以用`factory-bean`代替静态工厂使用的`class`属性

~~~xml
<!-- 将工厂类注册到容器 -->
<bean id="barFactory" class="...NonStaticBarInterfaceFactory"/>
<!-- 指定factory-bean -->
<bean id="bar" factory-bean="barFactory" factory-method="getInstance"/>
~~~

##### FactoryBean

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

#### `init-method`

指定bean的方法名，在对象初始化时调用指定方法

作用同`InitializingBean  `

#### `destroy-method  `

指定bean的方法名，在对象销毁时调用指定方法

作用同`DisposableBean  `

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

#### `<lookup-method>`

实现方法注入：通过相应方法为主体对象注入依赖对象  

* `name:`指定注入的方法名
* `bean:`指定方法返回的对象

`<lookup-method>`指定的方法签名必须符合如下格式：

~~~java
<public|protected> [abstract] <return-type> theMethodName(no-arguments);
~~~

也就是说，该方法必须能被子类实现或者覆写，因为容器会未我们要进行方法注入的对象使用Cglib动态代理生成一个子类实现，替代当前对象。

当指定方法被调用的时候，如果指定的bean是prototype型的，那么每次调用name指定方法时，容器可以返回都是不同的实例

以上是使用方法注入的方式达到“每次调用都让容器返回新的对象实例”的目的，还可以使用下面方式达到相同的目的：

##### BeanFactoryAware接口

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

实际上，方法注入动态生成的子类，完成的是与以上类似的逻辑，只不过实现细节上不同而已。  

##### ObjectFactoryCreatingFactoryBean

ObjectFactoryCreatingFactoryBean是spring提供的一个FactoryBean实现，它的方法返回一个ObjectFactory实例

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

#### `<replaced-method>`

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

# Spring IoC原理

Spring的IoC容器所起的作用如图所示：

![SpringIoC作用](https://gitee.com/wangziming707/note-pic/raw/master/img/SpringIoC%E4%BD%9C%E7%94%A8.png)

它会以某种方式加载Configuration Metadata(通常也就是XML格式的配置信息)，然后根据这些信息绑定整个系统的对象，最终组装成一个可用的基于轻量级容器的应用系统。

Spring的IoC容器实现上述功能的过程，可大致划分为两个阶段：

* 容器启动阶段
  * 最开始，容器会通过某种途径加载Configuration Metadata，除了代码方式，大部分情况下，容器需要依赖某些工具类(`BeanDefinitionReader`)对加载的Configuration Metadata进行解析和分析并将分析后的信息编组为对应的`BeanDefinition`
  * 然后将`BeanDefinition`注册到对应的`BeanDefinitionRegistry  `
* Bean实例化阶段
  * 当容器收到某个bean的请求时，或者因依赖关系容器需要隐式地调用getBean方法时
  * 容器会首先检查所请求的对象之前是否已经初始化
    * 如果有，直接返回对象
    * 如果没有  根据注册的BeanDefinition所提供的信息实例化被请求对象，并为其注入依赖。如果该对象实现了某些回调接口，也会根据回调接口的要求来装配它。当该对象装配完毕之后，容器会立即将其返回请求方使用。  

Spring的IoC容器在实现的时候，充分运用了这两个实现阶段的不同特点，在每个阶段都加入了相应的容器扩展点，以便我们可以根据具体场景的需要加入自定义的扩展逻辑  

## 容器启动阶段

容器启动阶段主要是将bean配置信息读取并加载成BeanDefinition。

Spring提供了一种叫做`BeanFactoryPostProcessor`的容器扩展机制 .该机制允许我们在容器实例化相应对象之前，对注册到容器的BeanDefinition所保存的信息做相应的修改。这就相当于在容器启动阶段最后加入一道工序，让我们对最终的BeanDefinition做一些额外的操作，比如修改其中bean定义的某些属性，为bean定义增加其他信息等  

如果要自定义实现BeanFactoryPostProcessor，通常我们需要实现org.springframework.
beans.factory.config.BeanFactoryPostProcessor接口  

因为一个容器可能拥有多个BeanFactoryPostProcessor，这个时候可能需要实现类同时实现Spring的org.springframework.core.Ordered接口，以保证各个BeanFactoryPostProcessor可以按照预先设定的顺序执行（如果顺序紧要的话）  

接口定义如下：

~~~java
@FunctionalInterface
public interface BeanFactoryPostProcessor {
    void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;
}
~~~

但是，因为Spring已经提供了几个现成的BeanFactoryPostProcessor实现类，所以，大多时候，我们很少自己去实现某个BeanFactoryPostProcessor。常用的有：

* `PropertyPlaceholderConfigurer  `(该方法现已过时)
* `PropertyOverrideConfigurer `
* `CustomEditorConfigurer  `

### 使用`BeanFactoryPostProcessor`

两 种 方 式 来 应 用 `BeanFactoryPostProcessor `， 分 别 针 对 基 本 的 IoC 容 器BeanFactory和较为先进的容器`ApplicationContext`。  

* 手动装配`BeanFactory`使用的`BeanFactoryPostProcessor `：

    ~~~java
    //声明BeanFactory并加载xml配置文件
    DefaultListableBeanFactory beanFactory = new DefaultListableBeanFactory();
    XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(beanFactory);
    reader.loadBeanDefinitions("spring.xml");
    //声明要使用的BeanFactoryPostProcessor
    PropertyPlaceholderConfigurer propertyPostProcessor = new PropertyPlaceholderConfigurer();
    propertyPostProcessor.setLocation(new ClassPathResource("demo.properties"));
    //执行后处理操作
    propertyPostProcessor.postProcessBeanFactory(beanFactory);
    ~~~

* 通过ApplicationContext使用BeanFactoryPostProcessor  

    ApplicationContext会自动识别配置文件中的BeanFactoryPostProcessor并应用它

    所以只需要将BeanFactoryPostProcessor添加到配置文件中即可生效

    ~~~xml
    <bean class="...PropertyPlaceholderConfigurer">
        <property name="location" value="demo.properties"/>
    </bean>
    ~~~

### `PropertyPlaceholderConfigurer  `

PropertyPlaceholderConfigurer允许我们在XML配置文件中使用占位符（PlaceHolder），并将这些占位符所代表的资源单独配置到简单的properties文件中来加载。 

实例：

如果PropertyPlaceholderConfigurer读取装配的properties文件如下：

~~~properties
dog.name = haha
dog.age = 3
~~~

那么可以直接在xml配置文件中通过占位符使用这些属性和值 

~~~xml
<bean id="dog" class="...Dog">
    <property name="name" >
        <value>${dog.name}</value>
    </property>
    <property name="age">
        <value>${dog.age}</value>
    </property>
</bean>
~~~

通过这样的方式就将业务相关的配置信息与spring的ioc配置分别管理

使用了PropertyPlaceholderConfigurer后，它替换占位符的逻辑如下：

* BeanFactory在第一个阶段加载完成所有配置的配置信息是，BeanDefinition中保持的信息仍然是占位符形式的，如${dog.age}
* 当PropertyPlaceholderConfigurer调用postProcessBeanFactory进行后处理时，会将properties文件中的配置信息来替换相应BeanDefinition中占位符所表示的属性值

### `PropertyOverrideConfigurer  `

PropertyOverrideConfigurer  可以通过读取相应规则的properties文件将bean定义的property信息进行覆盖替换  

PropertyOverrideConfigurer读取的properties文件应该满足如下格式：

~~~properties
beanName.propertyName=value
~~~

这样，将将PropertyOverrideConfigurer加载到容器之后，bean对象xml文件中的值，将被properties文件中的值覆盖

实例：

properties配置：

~~~properties
dog.name = haha
dog.age = 10
~~~

springIoC配置：

~~~xml
<bean class="...PropertyOverrideConfigurer">
    <property name="location" value="demo.properties"/>
</bean>

<bean id="dog" class="...Dog" scope="prototype">
    <property name="name" value="www" />
    <property name="age" value="3"/>
</bean>
~~~

这样进入dog bean进入第二阶段前，`name `和`dog `将由原来的`www`和`3 `替换为 `haha `和`10`

**注意：**

当容器中配置的多个PropertyOverrideConfigurer对同一个bean定义的同一个property值进行处理的时候，最后一个将会生效。

### `CustomEditorConfigurer  `

`CustomEditorConfigurer  `辅助性地将后期会用到的信息注册到容器，对`BeanDefinition`没有做任何变动  

spring容器通过一般通过XML配置文件来配置bean对象的类型,而XML只能记载字符串,但最终要转换为对象类型

完成这种由字符串到具体对象的转换，都需要这种转换规则相关的信息，而CustomEditorConfigurer就是帮助我们传达类似信息的

Spring内部通过JavaBean的PropertyEditor来帮助进行String类型到其他类型的转换工作  

CustomEditorConfigurer可以完成PropertyEditor到容器的注册

但现在主要通过PropertyEditorRegistrar  来注册PropertyEditor

## Bean实例化阶段

第一阶段容器启动阶段结束后，容器目前拥有所有对象的BeanDefinition来保存实例化阶段将要用的必要信息

只有当请求方调用BeanFactory的getBean()来请求对象实例时，才会可能触发Bean实例化阶段的活动  

getBean()方法可以被客户端现实地调用，也可以在容器内部被隐式地调用，隐式调用有如下情况：

* 如果请求的对象有依赖对象，且依赖依赖对象没有实例化；那么容器会先内部调用getBean()实例化依赖对象

    然后再实例化被请求的对象

* 对BeanFactory来说：对象实例化默认采用延迟初始化。只有当getBean被显示调用时，容器才会开始实例化对象；但是ApplicationContext  启动后就会实例化所有的bean，这样的实例化也是容器隐式进行的

调用getBean()方法时会先检查bean是否已经实例化，如果没有，会调用createBean()方法来进行具体的对象实例化  

* 可以在AbstractBeanFactory类的代码中查看到getBean()方法的完整实现逻辑
* 可以在其子类AbstractAutowireCapableBeanFactory的代码中查看createBean()方法的实现逻辑(doCreateBean)

实例化过程如图：

![Bean的实例化过程](https://gitee.com/wangziming707/note-pic/raw/master/img/Bean%E7%9A%84%E5%AE%9E%E4%BE%8B%E5%8C%96%E8%BF%87%E7%A8%8B.png)

### 实例化bean对象

容器在实例化bean时，采用“策略模式(Strategy Pattern)”

可以通过反射或者CGLIB动态字节码生成来初始化bean实例或者动态生成其子类

实例化策略的定义接口及其实现的继承关系如下：

![CglibSubclassingInstantiationStrategy](https://gitee.com/wangziming707/note-pic/raw/master/img/CglibSubclassingInstantiationStrategy.png)

* `InstantiationStrategy  `是实例化策略的抽象接口  

* 其直接子类`SimpleInstantiationStrategy`实现了简单的对象实例化功能，可以通过反射来实例化对象

* `CglibSubclassingInstantiationStrategy`继承了`SimpleInstantiationStrategy`的以反射方式实例化对象的功能，并且通过CGLIB的动态字节码生成功能，该策略实现类可以动态生成某个类的子类，进而满足了方法注入所需的对象实例化需求  

默认情况下，容器内部采用的是CglibSubclassingInstantiationStrategy,通过其和相应的BeanDefinition来构造对象实例

### `BeanWrapper  `

上一步容器并不会直接返回构造完成的对象实例，而是以BeanWrapper对构造完成的对象实例进行包裹，返回相应的BeanWrapper实例  

BeanWrapper接口通常在Spring框架内部使用，它在spring中只有一个实现类BeanWrapperImpl。其作用是对某个bean进行“包裹”，然后对这个“包裹”的bean进行操作，比如设置或者获取bean的相应属性值  

BeanWrapperImpl的继承关系如下：

![BeanWrapperImpl](https://gitee.com/wangziming707/note-pic/raw/master/img/BeanWrapperImpl.png)

* BeanWrapper继承了PropertyAccessor接口，可以以统一的方式对对象属性进行访问，设置对象属性值  

* BeanWrapper定义同时又直接或者间接继承了PropertyEditorRegistry和TypeConverter接口

    所以BeanWrapper会使用容器注册的PropertyEditor来进行转换类型  

### `Aware`

当对象实例化完成并且相关属性以及依赖设置完成后，Spring容器会检查当前bean是否实现了Aware接口

(`AbstractAutowireCapableBeanFactory.doCreateBean`方法中调用的`initializeBean`的第一步:调用`invokeAwareMethods`)

如果是，则将这些Aware接口定义中规定的依赖注入给当前对象实例 

Aware接口及其子接口：

![Aware](https://gitee.com/wangziming707/note-pic/raw/master/img/Aware.png)

* `BeanNameAware  `:如果Spring容器检测到当前对象实例实现了该接口，会将该对象实例的bean定义对应的beanName设置到当前对象实例  
* `BeanClassLoaderAware  `:如果容器检测到当前对象实例实现了该接口,会将对应加载当前bean的`Classloader`注入当前对象实例    默认会使用加载`org.springframework.util.ClassUtils`类的`Classloader`。  
* `BeanFactoryAware  `如果Spring容器检测到当前对象实例实现了该接口，会将该对象实例的bean定义的所属beanFactory引用设置到当前对象实例  

`BeanFactory`会检查以上三个`Aware`接口，除此之外`ApplicationContext  `还会额外检查下面接口：

(见`ApplicationContextAwareProcessor.invokeAwareInterfaces`)

* `ResourceLoaderAware  `:将当前ApplicationContext自身设置到对象实例  （因为`ApplicationContext  `实现了`ResourceLoader  `接口）
* `ApplicationEventPublisherAware  `将自身注入当前对象  （因为`ApplicationContext  `实现了`ApplicationEventPublisher  `接口  ）
* `MessageSourceAware  `将自身注入当前对象  （因为`ApplicationContext  `实现了`MessageSource`接口  ）
* `ApplicationContextAware  `将自身注入当前对象  

### `BeanPostProcessor  `

和`BeanFactoryPostProcessor`作用类似

实现`BeanFactoryPostProcessor`的`BeanFactory`会在启动阶段的最后进行后处理

实现`BeanFactoryPostProcessor`的`Bean`会在实例化后的合适时机进行后处理

它的定义如下:

~~~java
public interface BeanPostProcessor {
	@Nullable
	default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}

	@Nullable
	default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}
}
~~~

接口定义了两个方法:

* `postProcessBeforeInitialization `会在检查`Aware`步骤后随即被检查调用（见流程图）(`AbstractAutowireCapableBeanFactory.doCreateBean`方法中调用的`initializeBean`的第二步：调用`applyBeanPostProcessorsBeforeInitialization`)

* `postProcessAfterInitialization`会在检查`init-method`步骤后随即被检查调用(见流程图)

    (`AbstractAutowireCapableBeanFactory.doCreateBean`方法中调用的`initializeBean`的第最后一步：调用`applyBeanPostProcessorsAfterInitialization`)

**应用场景:**

通常使用`BeanPostProcessor  `的场景时处理标记接口实现类,或者为当前对象提供代理实现

比如ApplicationContext对应的那些Aware接口实际上就是通过BeanPostProcessor的方式进行处理的  

(见`ApplicationContextAwareProcessor.postProcessBeforeInitialization`)

Spring的AOP则使用BeanPostProcessor来为对象生成相应的代理对象  

**除此之外:**

实际上,有一种特殊的`BeanPostProcessor  `:`InstantiationAwareBeanPostProcessor  `

如果bean对象实现了`InstantiationAwareBeanPostProcessor  `接口,那么在实例化bean对象步骤之前,会使用

`InstantiationAwareBeanPostProcessor  `来构造对象实例,构造成功后直接返回对象实例;而不会按照常规流程继续执行

### `InitializingBean`

InitializingBean是容器内部广泛使用的一个对象生命周期标识接口，其定义如下  :

~~~java
public interface InitializingBean {
	void afterPropertiesSet() throws Exception;
}
~~~

Spring容器在实例化过程中完成"BeanPostProcessor的前置处理 "过程后,会检查对象是否实现了InitializingBean接口,如果是,则会调用其afterPropertiesSet()方法进一步调整对象实例的状态  

(`AbstractAutowireCapableBeanFactory.doCreateBean`方法中调用的`initializeBean`的第三步：调用`invokeInitMethods`)

如果直接让业务对象实现这个接口,显得Spring容器比较有侵入性.作为代替,可以使用`<bean>`的`init-method`属性

`InitializingBean`接口和`init-method`属性可同时使用

### `DisposableBean  `

如果singleton类型的bean对象实现了`DisposableBean`接口,那么容器将为该实例注册一个用于对象销毁的回调(Callback),在对象实例销毁之前,执行销毁逻辑

~~~java
public interface DisposableBean {
	void destroy() throws Exception;
}
~~~

如果直接让业务对象实现这个接口,显得Spring容器比较有侵入性.作为代替,可以使用`<bean>`的`destroy-method`属性

`DisposableBean`接口和`destroy-method`属性可同时使用

可以通过BeanFactory销毁容器管理的所有singleton对象:

* `BeanFactory`:`ConfigurableBeanFactory  `提供的`destroySingletons()  `方法
* `ApplicationContext  `:`AbstractApplicationContext  `提供的`registerShutdownHook()`方法



# ApplicationContext

ApplicationContext是Spring提供的更高级的容器，相较于BeanFactory，它提供了更多的功能:

![ApplicationContext](https://gitee.com/wangziming707/note-pic/raw/master/img/ApplicationContext.png)

包括BeanFactoryPostProcessor、 BeanPostProcessor以及其他特殊类型bean的自动识别、容器启动后bean实例的自动初始化、国际化的信息支持、容器内事件发布等  

Spring 为ApplicationContext提供了几个常用的实现：

* `FileSystemXmlApplicationContext  `:从文件系统加载bean定义以及相关资源的ApplicationContext实现  
* `ClassPathXmlApplicationContext  `:从Classpath加载bean定义以及相关资源的ApplicationContext实现  
* `XmlWebApplicationContext  `Web应用程序的ApplicationContext实现  

## 统一资源加载策略

Spring提供了一套基于org.springframework.core.io.Resource和org.springframework.core.io.ResourceLoader接口的资源抽象和加载策略。  

### `Resource  `

Spring框架内部使用`Resource`接口作为所有资源的抽象和访问接口：

它的定义如下：

~~~java
public interface Resource extends InputStreamSource {

	boolean exists();

	default boolean isReadable() {
		return exists();
	}

	default boolean isOpen() {
		return false;
	}

	default boolean isFile() {
		return false;
	}

	URL getURL() throws IOException;

	URI getURI() throws IOException;

	File getFile() throws IOException;

	default ReadableByteChannel readableChannel() throws IOException {
		return Channels.newChannel(getInputStream());
	}

	long contentLength() throws IOException;

	long lastModified() throws IOException;

	Resource createRelative(String relativePath) throws IOException;

	@Nullable
	String getFilename();

	String getDescription();

}
~~~

Spring根据资源的不同类型提供了一些实现类：

* `ByteArrayResource `将字节（ byte）数组提供的数据作为一种资源进行封装  
* `ClassPathResource `该实现从Java应用程序的ClassPath中加载具体资源并进行封装  可以使用指定的类加载器（ ClassLoader）或者给定的类进行资源加载。   
* `FileSystemResource `对java.io.File类型的封装，  
* `UrlResource `通过java.net.URL进行的具体资源查找定位的实现类  
* `InputStreamResource  `将给定的InputStream视为一种资源的Resource实现类，较为少用  

### `ResourceLoader`

`Resource`定义了资源，`ResourceLoader`定义了如何查找和定位这些资源

`ResourceLoader`接口是资源查找定位策略的统一抽象  

它的定义如下：

~~~java
public interface ResourceLoader {

	String CLASSPATH_URL_PREFIX = ResourceUtils.CLASSPATH_URL_PREFIX;

	Resource getResource(String location);

	@Nullable
	ClassLoader getClassLoader();

}
~~~

通过`Resource.getResource(String location)`方法我们就可以根据指定的资源位置，定位到具体的资源实例。

#### `DefaultResourceLoader`

DefaultResourceLoader 是ResourceLoader的一个默认的实现类，它默认的资源查找处理逻辑如下：

* 首先检查资源路径是否以`classpath:`前缀打头，如果是，则尝试构造ClassPathResource类型资源并返回  
* 资源路径不是以`classpath:`前缀打头：
  * 通过URL来定位资源，如果没抛出MalformedURLException，有则会构造UrlResource类型的资源并返回  
  * 如果还是无法根据资源路径定位指定的资源，则委派getResourceByPath(String) 方法来定位资源  ：默认实现逻辑是，构造ClassPathResource类型的资源并返回

如果最终没有找到符合条件的相应资源， getResourceByPath(String)方法就会构造一个实际上并不存在的资源并返回  

#### `FileSystemResourceLoader  `

为了避免`DefaultResourceLoader`在最后`getResourceByPath(String)`方法上的不恰当处理,可以使用`FileSystemResourceLoader  `,它继承自`DefaultResourceLoader`但覆写了`getResourceByPath(String) ` 使之从文件系统加载资源并以FileSystemResource类型返回  

#### `ResourcePatternResolver`

`ResourcePatternResolver `接口是`ResourceLoader  `的拓展,它的定义如下:

~~~java

public interface ResourcePatternResolver extends ResourceLoader {

	String CLASSPATH_ALL_URL_PREFIX = "classpath*:";

	Resource[] getResources(String locationPattern) throws IOException;

}
~~~

它的getResources方法可以根据指定的资源路径匹配模式 返回多个Resource实例  

##### `PathMatchingResourcePatternResolver  `

`PathMatchingResourcePatternResolver`是`ResourcePatternResolver`最常用的实现。

在构造`PathMatchingResourcePatternResolver`实例的时候，可以指定一个`ResourceLoader`，如果不指定的话，则 `PathMatchingResourcePatternResolver`内 部 会 默 认 构 造 一 个 `DefaultResourceLoader`实例  

`PathMatchingResourcePatternResolver`内部会将匹配后确定的资源路径，
委派给它的`ResourceLoader`来查找和定位资源  

### ApplicationContext与ResourceLoader

从之前的ApplicationContext继承关系图中可以看到，它继承了`ResourcePatternResolver  `,所以也间接继承了`ResourceLoader`

所以，任何的`ApplicationContext`实现都可以看作是一个`ResourceLoader`甚至`ResourcePatternResolver  `

通常，所有的`ApplicationContext  `实现类都直接或间接地继承`AbstractApplicationContext`  

`AbstractApplicationContext  `直接继承`DefaultResourceLoader`,所以它的`getResource  `就是直接使用的`DefaultResourceLoader  `

而`getResources `使用的是`PathMatchingResourcePatternResolver  `的实例完成

这样 ApplicationContext就支持了Spring的统一资源加载

作为ResourceLoader的ApplicationContext显然可以直接调用getResource方法来进行资源定位获取，但除此之外，作为容器和ResourceLoader身份的ApplicationContext显然后更多的应用

#### ResourceLoader类型的注入

之前的Aware章节已经提到过 `ResourceLoaderAware  `会将`ApplicationContext`‘容器自身作为`ResourceLoader`注入到bean对象中

这样就可以在bean对象中直接使用ResourceLoader

享受Spring提供的统一资源加载服务

#### Resource类型的注入

如果使用BeanFactory，那么对于那些Spring容器提供的默认的PropertyEditors无法识别的对象类型，我们可以提供自定义的PropertyEditor实现并注册到容器中，以供容器做类型转换的时候使用。

默认情况下，BeanFactory容器不会为Resource类型提供相应的PropertyEditor  

但ApplicationContext默认就可以正确识别Resource类型并转换后注入相关对象

它在启动时，会通过ResourceEditorRegistrar  来注册ResourceEditor  到容器中

ResourceEditor  就是Spring提供的针对Resource类型的PropertyEditor  

所以如果使用ApplicationContext，那么可以直接将XML文件中的RUL字符串转换为Resource对象

#### 特定ApplicationContext的Resource加载行为

对于URL所表示的资源路径来说，通常开头会有一个协议前缀，比如`file:`、 `http:`、``ftp:  `等

除此之外，Spring还拓展了协议的前缀的集合：提供了`classpath :`、`classpath*:`前缀

 `classpath*`:与`classpath:`的唯一区别就在于，如果能够在classpath中找到多个指定的资源，则返回多个

对于特定的`ApplicationContext`实现，有不同的默认资源加载行为：

* `ClassPathXmlApplicationContext  `: 在实例化提供xml配置文件时和Resource类型的注入时，提供的URL 如果没有指定前缀，将默认使用`classpath:` 或者`classpath*: `前缀，从classpath下定位资源

* `FileSystemXmlApplicationContext  `: 在实例化提供xml配置文件时和Resource类型的注入时，提供的URL 如果没有指定前缀，将默认使用`file:`前缀，从文件系统定位资源

## 国际化信息支持

### MessageSource

Java SE提供了`Locale  `、`ResourceBundle  `接口以提供国际化支持

Spring对Java SE 的国际化基础上，进一步抽象了国际化信息的访问接口，也就是org.springframework.context.MessageSource  接口，定义如下：

~~~java
public interface MessageSource {
	//根据传入的资源条目的键(code) 、信息参数(args)、以及Locale来查找信息;如果没有找到，则返回指定的 defaultMessage
	@Nullable
	String getMessage(String code, @Nullable Object[] args, @Nullable String defaultMessage, Locale locale);
	//根据传入的资源条目的键(code) 、信息参数(args)、以及Locale来查找信息
	String getMessage(String code, @Nullable Object[] args, Locale locale) throws NoSuchMessageException;
	//根据传入的MessageSourceResolvable、以及Locale来查找信息 MessageSourceResolvable 是对code、args、defaultMessage的封装
	String getMessage(MessageSourceResolvable resolvable, Locale locale) throws NoSuchMessageException;

}
~~~

Spring提供了三种MessageSource的实现：

* `StaticMessageSource`：MessageSource接口的简单实现，可以通过编程的方式添加信息条目，多用于测试，不应该用于正式的生产环境  
* `ResourceBundleMessageSource`:基于标准的`java.util.`而实现的`MessageSource`，对其父类`AbstractMessageSource`的行为进行了扩展，提供对多个`ResourceBundle`的缓存以提高查询速度。同时，对于参数化的信息和非参数化信息的处理进行了优化，并对用于参数化信息格式化的`MessageFormat`实例也进行了缓存。它是最常用的、用于正式生产环境下的`MessageSource`实现    
* `ReloadableResourceBundleMessageSource  `:同样基于标准的`java.util.ResourceBundle`而构建的`MessageSource`实现类 ，但通过其`cacheSeconds`属性可以指定时间段，以定期刷新并检查底层的properties资源文件是否有变更。对于properties资源文件的加载方式也与`ResourceBundleMessageSource`有所不同，可以通过`ResourceLoader`来加载信息资源文件。 使用`ReloadableResourceBundleMessageSource`时，
  应该避免将信息资源文件放到classpath中，因为这无助于`ReloadableResourceBundleMessageSource`定期加载文件变更。  

### ApplicationContext和MessageSource

通过ApplicationContext的继承关系可知，ApplicationContext也实现了MessageSource

在默认情况下， ApplicationContext将委派容器中一个名称为messageSource的MessageSource接口实现来完成MessageSource应该完成的职责。如果找不到这样一个名字的MessageSource实现， ApplicationContext内部会默认实例化一个不含任何内容的StaticMessageSource实例，以保证相应的方法调用。所以通常情况下，如果要提供容器内的国际化信息支持，我们需要进行如下的配置：

~~~xml
<bean id="messageSource" class="...MessageSource实现类">
    <property name="basenames">
        <list>
            <value>basename</value>
            ...
        </list>
    </property>
</bean>
~~~

这样配置好后，可以直接使用依赖注入的方式让其他的bean支持国际化

也可以让bean实现MessageSourceAware  将ApplicationContext对象作为MessageSource注入到bean对象中

## 容器内事件发布

Spring SE 提供了一套基于` java.util.EventObject`类和`java.util.EventListener`接口的事件发布规范

ApplicationContext容器内部基于此规范实现了一套事件发布规范

Spring的ApplicationContext容器内的事件发布机制，主要用于单一容器内的简单消息通知和处
理，并不适合分布式、多进程、多容器之间的事件通知。  

### `ApplicationEvent  `

Spring容器内自定义的事件类型，继承自`EventObject`

默认情况下，Spring提供了三个实现：

* `ContextClosedEvent  `:` ApplicationContext`容器在即将关闭的时候发布的事件类型  
* `ContextRefreshedEvent  `: `ApplicationContext`容器在初始化或者刷新的时候发布的事件类型  

* `RequestHandledEvent`： Web请求处理后发布的事件，其有一子类`ServletRequestHandledEvent`提供特定于Java EE的Servlet相关事件  

### `ApplicationListener  `

ApplicationContext容器内使用的自定义事件监听器接口定义，继承自java.util.EventListener。 

它的定义如下：

~~~java
@FunctionalInterface
public interface ApplicationListener<E extends ApplicationEvent> extends EventListener {
	void onApplicationEvent(E event);
}
~~~

ApplicationContext容器在启动时，会自动识别并加载EventListener类型bean定义，一旦容器内有事件发布，将通知这些注册到容器的EventListener。  

### `ApplicationEventPublisher`

ApplicationContext继承了 ApplicationEventPublisher，它的定义如下：

```java
@FunctionalInterface
public interface ApplicationEventPublisher {

   default void publishEvent(ApplicationEvent event) {
      publishEvent((Object) event);
   }

   void publishEvent(Object event);

}
```

接口提供了`publishEvent`方法，所以ApplicationContext还能作为事件发布者的角色

ApplicationContext容器的具体实现类 把实现事件的发布和监听器的注册任务委托给了ApplicationEventMulticaster  接口的实现类

### `ApplicationEventMulticaster  `

该接口定义了具体事件监听器的注册管理以及事件发布的方法 :

~~~java
public interface ApplicationEventMulticaster {

	void addApplicationListener(ApplicationListener<?> listener);

	void addApplicationListenerBean(String listenerBeanName);

	void removeApplicationListener(ApplicationListener<?> listener);

	void removeApplicationListenerBean(String listenerBeanName);

	void removeAllListeners();

	void multicastEvent(ApplicationEvent event);

	void multicastEvent(ApplicationEvent event, @Nullable ResolvableType eventType);

}
~~~

它有以下实现：

* 抽象实现类`AbstractApplicationEventMulticaster`它实现了事件监听器的管理
  功能；出于灵活性和扩展性考虑，事件的发布功能则委托给了其子类  

* 实现类`SimpleApplicationEventMulticaster`: `AbstractApplicationEventMulticaster`的一个子类实现，添加了事件发布功能的实现，其默认使用了SyncTaskExecutor进行事件的发布，事件是同步顺序发布的

### 应用

需要自定义ApplicationEvent 事件

需要自定义的监听器 ApplicationListener并注册到 ApplicationContext容器中

需要自定义的事件发布者ApplicationEventPublisher并注册到ApplicationContext容器中

将ApplicationEventPublisher注入到bean中

bean就拥有了发布事件的能力

## 加载多配置文件

在创建容器时，可以一次性加载多个配置文件到ApplicationContext容器中：

~~~java
//使用字符串数组
String[] locations = new String[]{ "conf/dao-tier.springxml", 
"conf/view-tier.springxml", "conf/business-tier.springxml"};
ApplicationContext container = new ClassPathXmlApplicationContext(locations);
//使用通配符
ApplicationContext container = new FileSystemXmlApplicationContext("conf/**/*.springxml")
~~~

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

* `@Autowired`在属性上，容器按照类型为该属性自动依赖注入（通过setter方法）
* `@Qualifier`：`@Autowired`的辅助注解，指定beanName，以byName的方式自动注入

除了spring提供的`@Autowired`注解，javax包还提供了`@Resource`同样可以实现依赖注入

但不同的是：@Resource默认按照ByName自动注入  

# 其他

## Bean线程安全

spring并未对bean的多线程做封装处理

对于一个单例bean，是否线程安全看它是否是无状态的(不保存数据)，如果是无状态的，那么显然线程是安全的，否则，就有线程安全问题

解决方案：

* 对有状态的bean 可以将它的 scope 设置为 Prototype 保证每个线程获取的对象和对象的数据都是独立的

## 循环依赖

==todo==

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
