# 基于注解的IOC

基于注解的Ioc配置时XML配置的替代方案，它依赖于字节码元数据来连接组件。

通过在相关类、方法和字段声明上使用注解，将配置从用XML描述bean装配转移到组件类本身。

这种基于注解的配置是通过`BeanPostProcessor`实现的，可以使用下面标签来隐式注册注解配置需要的`BeanPostProcessor`:

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xmlns:context="http://www.springframework.org/schema/context"
	xsi:schemaLocation="http://www.springframework.org/schema/beans
		https://www.springframework.org/schema/beans/spring-beans.xsd
		http://www.springframework.org/schema/context
		https://www.springframework.org/schema/context/spring-context.xsd">

	<context:annotation-config/>

</beans>
~~~

 `<context:annotation-config/>` 标签会隐式注册下面后处理器：

- `ConfigurationClassPostProcessor`
- `AutowiredAnnotationBeanPostProcessor`
- `CommonAnnotationBeanPostProcessor`
- `PersistenceAnnotationBeanPostProcessor`
- `EventListenerMethodProcessor`

## `@Autowired`

`@Autowired`注解用来进行自动依赖注入，它可以注释在类的构造器方法、set方法、普通方法、类字段上，让当前类获取指定类的实例对象，例如，通过`@Autowired`向`Person`中注入`Dog`实例：

~~~java
@Data
public class Person {

    private String name;
    @Autowired
    private Dog dog;

}
~~~



~~~java
@Data
public class Dog {
    private String name;
    private int age;
}
~~~

对应的XML配置：

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/context https://www.springframework.org/schema/context/spring-context.xsd">
    <context:annotation-config/>
    <bean name="person" class="com.wzm.spring.Person">
        <property name="name" value="wzm"/>
    </bean>
    <bean name="dog" class="com.wzm.spring.Dog">
        <property name="name" value="tom"/>
        <property name="age" value="12"/>
    </bean>

</beans>
~~~

注意XML文件中并没有指定Person的dog属性，但是容器在加载Bean是扫描到了`@Autowired`注解，会自动将对应的对象注入：

~~~java
ClassPathXmlApplicationContext container = new ClassPathXmlApplicationContext("spring.xml");
Person person = (Person)container.getBean("person");
System.out.println(person.getDog().getAge()); //12
~~~

或者直接在普通方法上注释：

~~~java
@Data
public class Person {

    private String name;
    private Dog dog;

    @Autowired
    public void demo(Dog dog){
        System.out.println(dog.getName());
    }

}
~~~

这样在加载bean时，会调用一次这个`demo()`方法，并将参数中的对象注入。

### 复合注入

可以在数组或者`Set`对象上注释`@Autowired`,这样Spring的Ioc容器会将容器中所有符合的类型都注入指定数组：

~~~java
@Data
public class Person {
    private String name;
    @Autowired
    private Dog[] dogs;
}
~~~

或者：

~~~java
@Data
public class Person {
    private String name;
    @Autowired
    private Set<Dog> dogs;
}
~~~

默认情况下，复合注入的顺序是按照容器中bean定义的注册顺序。如果需要自定义这个顺序，可以在目标bean上实现`org.springframework.core.Ordered`接口，或者使用`@Order`或标准的`@Priority`注解。

也可以在`Map`类型上注释`@Autowired`,此时`Map`的键必须是String，注入时，键为注入的bean的名称，值为bean对应的实例：

~~~java
@Data
public class Person {
    private String name;
    @Autowired
    private Map<String,Dog> dogs;
}
~~~

### 注入失败行为

默认情况下，当没有匹配的bean进行注入时，自动装配会失败，并抛出异常。即使是对数组、Set或MapSet，也至少需要一个匹配的元素。

可以使用在`@Autowired`中设置`required`属性为`false`,使容器忽略装配失败：

~~~java
@Data
public class Person {
    private String name;
    @Autowired(required = false)
    private Dog dog;
}
~~~

在一个bean类中，只能有一个构造器被required属性为true的`@Autowired`方法注释。

如果有多个构造器都被`@Autowired`注释，那么这些注解的`required`属性必须都为`false`,否则自动装配将不生效。在这种情况下，实例初始化时将选择能够满足的依赖项最多的构造器。如果所有构造器都无法满足，将调用无参/默认的构造器。

### 替代Aware接口注入

Aware接口可以让bean类获取容器支持的资源，`@Autowired`类可以替代它让bean类获取下面实例资源：`BeanFactory`, `ApplicationContext`, `Environment`, `ResourceLoader`, `ApplicationEventPublisher`, 和`MessageSource`. 

例如：

~~~java
public class MovieRecommender {

	@Autowired
	private ApplicationContext context;

	public MovieRecommender() {
	}

	// ...
}
~~~



## `@Qualifier`

`@Autowired`是通过类型进行自动装配的，如果只需要一个实例，但是容器中多多个匹配的类型，自动装配将失败。所以可以使用`@Qualifier`配合`@Autowired`，`@Qualifier`的值用来指定要注入的qualifier限定符值，那么将使用beanName，例如：

~~~java
@Data
public class Person {
    private String name;
    @Autowired
    @Qualifier("dog1")
    private Dog dog;
}
~~~

也可以在构造器的参数前注释`@Qualifier`：

~~~java
@Data
public class Person {
    @Autowired
    public void Person(@Qualifier("dog1") Dog dog){
        this.dog = dog;
    }
    ......
}
~~~

与其对应的XML定义为：

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/context https://www.springframework.org/schema/context/spring-context.xsd">
    <context:annotation-config/>
    <bean name="person" class="com.wzm.spring.Person">
        <property name="name" value="wzm"/>
    </bean>


    <bean class="com.wzm.spring.Dog">
        <qualifier value="dog1"/>
        <property name="name" value="d1"/>
        <property name="age" value="12"/>
    </bean>
    <bean class="com.wzm.spring.Dog">
        <qualifier value="dog2"/>
        <property name="name" value="d2"/>
        <property name="age" value="12"/>
    </bean>
</beans>
~~~

如果没有定义qualifier的值，那么beanName将作为bean的默认qualifier值

需要注意，即使配合 `@Qualifier`注解，`@Autowired`仍然是按照类型注入的，只不过缩小了类型注入的范围。所以如果用`@Qualifier`指定一个不是匹配类型的bean，那么匹配就会失败。

需要注意`@Qualifier`匹配的是指定的`qualifier`值的bean，而`qualifier`值在默认情况下是beanName。beanName不能重复，但是`qualifier`值是可以重复的，但同一类型的bean的qualifier值最好不要重复，负责可能会造成自动装配失败。除非是对数组、集合的自动装配

### 自定义`@Qualifier`注解

可以用`@Qualifier`注释在自定义注解上，让自定义注解同样成为限定注解，这样自定义注解就关联`<qualifier/>`标签的type值，自定义注解的value就关联`<qualifier/>`标签的value值。

例如如下自定义注解：

~~~java
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Qualifier
public @interface Main {
    String value();
}
~~~

这样`@Main`就具有`@Qualifier`的功能：

~~~java
@Data
public class Person {
    private String name;
    @Autowired
    @Main("dog1")
    private Dog dog;
}
~~~

对应的XML配置为：

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/context https://www.springframework.org/schema/context/spring-context.xsd">
    <context:annotation-config/>
    <bean name="person" class="com.wzm.spring.Person">
        <property name="name" value="wzm"/>
    </bean>


    <bean class="com.wzm.spring.Dog">
        <qualifier type="com.wzm.spring.Main" value="dog1"/>
        <property name="name" value="d1"/>
        <property name="age" value="12"/>
    </bean>
    <bean class="com.wzm.spring.Dog">
        <qualifier type="com.wzm.spring.Main"  value="dog2"/>
        <property name="name" value="d2"/>
        <property name="age" value="12"/>
    </bean>
</beans>
~~~

也可以不使用带`value`的自定义注解，如：

~~~java
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Qualifier
public @interface Main {
}
~~~

对应的使用和XML配置如下：

~~~java
@Data
public class Person {
    private String name;
    @Autowired
    @Main
    private Dog dog;
}
~~~

配置：

~~~xml
<bean class="com.wzm.spring.Dog">
    <qualifier type="com.wzm.spring.Main" />
    <property name="name" value="d1"/>
    <property name="age" value="12"/>
</bean>
<bean class="com.wzm.spring.Dog">
    <property name="name" value="d2"/>
    <property name="age" value="12"/>
</bean>
~~~

## `@Resource`

Spring也支持使用 JSR-250 `@Resource `(`jakarta.annotation.Resource`)注解来进行注入，`@Resource`注解可以应用在域和set方法上。

`@Resource`有一个`name`属性，默认情况下，Spring将其解释为beanName进行注入的匹配。所以和`@Autowired`按照类型匹配不同，它是按照名称进行匹配的。例如：

~~~java
@Data
public class Person {
    private String name;

    @Resource(name = "dog1")
    private Dog dog;
}
~~~

如果没有指定`name`属性，那么`@Resource`将使用默认名称，在属性上时，默认名称就是属性名，在setter 方法上时，默认名称为set方法的参数名。

此外，如果`@Resource`没有匹配到指定的name，那么会退一步，按照类型再进行匹配，这样可以像`@Autowired`一样，注入一些容器提供的资源项，如`ApplicationContext`,如：

~~~java
public class MovieRecommender {

	@Resource
	private CustomerPreferenceDao customerPreferenceDao;
	//如果按名称没有找到beanName为customerPreferenceDao的bean，那么将按照类型进行匹配
	@Resource
	private ApplicationContext context; 

	public MovieRecommender() {
	}

	// ...
}
~~~

## `@Value`

`@Value`可以用来注入外部配置

使用如下配置类：

~~~java
@Configuration
@PropertySource("classpath:application.properties")
@ComponentScan("com.wzm.spring")
public class AppConfig {
}
~~~

并使用`AnnotationConfigApplicationContext`:

~~~java
AnnotationConfigApplicationContext container = new AnnotationConfigApplicationContext(AppConfig.class);
~~~

其中`application.properties`内容为：

~~~properties
dog.name = kiki
dog.age = 10
~~~

此时可以使用`@Value`进行注入：

~~~java
@Data
@Component
public class Dog {
    public Dog(@Value("${dog.name}") String name, 
               @Value("${dog.age}") int age){
        this.age = age;
        this.name = name;
    }
    ....
}
~~~

## `@PostConstruct` 和 `@PreDestroy`

`CommonAnnotationBeanPostProcessor`不仅识别`@Resource`注解，还识别JSR-250生命周期注解：`jakarta.annotation.PostConstruct`和`jakarta.annotation.PreDestroy`。这些注解提供了一种替代初始化回调和销毁回调中描述的生命周期回调机制的方法。

只要`CommonAnnotationBeanPostProcessor`在`ApplicationContext`中注册，携带这些注解的方法就会在与相应的Spring生命周期接口方法或显式声明的回调方法相同点处被调用。例如：

~~~java
@Component
public class CachingMovieLister {

	@PostConstruct
	public void populateMovieCache() {
        System.out.println("初始化");
	}

	@PreDestroy
	public void clearMovieCache() {
        System.out.println("销毁");
	}
}
~~~

## `@Lookup`

使用`@Lookup`注解可以实现Spring的方法注入，对应XML配置中的`<lookup-method/>`标签，例如：

~~~java
public abstract class CommandManager {

	public Object process(Object commandState) {
		Command command = createCommand();
		command.setState(commandState);
		return command.execute();
	}

	@Lookup("myCommand")
	protected abstract Command createCommand();
}
~~~

## 注解扫描和组件管理

之前的示例都是使用XML或者注解配置来指定Spring容器内的每个`BeanDefinition` 的配置元数据的。如果生产中需要注册到容器中类都需要手动配置，显然很繁琐且重复。所以Spring提供了一种通过扫描classpath隐式检测bean组件的选项。会将扫描到的被类似`@Component`注解注释的类自动注册到容器中。

任意一个类被`@Component`注解注释，则表示这个类将作为Spring容器的备选组件，在被扫描后将注册到容器中去。

除了`@Component`外，Spring还提供了`@controller`、`@Service`、`@Respository`注解，是`@Component`的特化注解类型，表示被注释的类是处理器类、业务层和持久层。目前这三个注解的行为和`@Component`是一样的，但是之后的Spring版本这几个注解可能会有特化的语义。

Spring能够自动检测如下被这四个组件注解注释的类，例如：

~~~java
@Service
public class SimpleMovieLister {

	private MovieFinder movieFinder;

	public SimpleMovieLister(MovieFinder movieFinder) {
		this.movieFinder = movieFinder;
	}
}
~~~

和

~~~java
@Repository
public class JpaMovieFinder implements MovieFinder {
	// implementation elided for clarity
}
~~~

要自动检测这些类并注册相应的bean，需要在`@Configuration`类中添加`@ComponentScan`注解，其中basePackages属性是要扫描的包：

~~~java
@Configuration
@ComponentScan(basePackages = "org.example")
public class AppConfig  {
	// ...
}
~~~

或者也可以直接用`value`属性，如：`@ComponentScan("org.example")`

对应的XML配置如下：

配置注解扫描：

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xmlns:context="http://www.springframework.org/schema/context"
	xsi:schemaLocation="http://www.springframework.org/schema/beans
		https://www.springframework.org/schema/beans/spring-beans.xsd
		http://www.springframework.org/schema/context
		https://www.springframework.org/schema/context/spring-context.xsd">

	<context:component-scan base-package="org.example"/>

</beans>
~~~

注意：使用`<context:component-scan>`会隐式启用`<context:annotation-config>`的功能。所以在使用`<context:component-scan>`时，无需包含`<context:annotation-config>`元素

并且使用`<context:component-scan>`会隐式注册以下组件：

* `AutowiredAnnotationBeanPostProcessor`

* `CommonAnnotationBeanPostProcessor`

### 扫描过滤器

默认情况下，被 `@Component`, `@Repository`, `@Service`, `@Controller`, `@Configuration`或者自定义注解(被`@Component`注释的)注释的类才会被标记为候选组件。

但是，我们可以应用自定义过滤器来修改和扩展这种行为。具体通过`@ComponentScan`注解的`includeFilters`或`excludeFilters`属性添加（或者作为`<context:component-scan>`元素在XML配置中的`<context:include-filter />` 或 `<context:exclude-filter /> `子元素）

filter元素需要类型和表达式属性。以下表格描述了过滤选项：

| 过滤类型          | 表达式示例                   | 描述                                                         |
| :---------------- | :--------------------------- | :----------------------------------------------------------- |
| annotation (默认) | `org.example.SomeAnnotation` | 被指定注解注释的类                                           |
| assignable        | `org.example.SomeClass`      | 特定的类/接口或者其子类或者实现                              |
| aspectj           | `org.example..*Service+`     | AspectJ类型表达式指定的类                                    |
| regex             | `org\.example\.Default.*`    | 正则表达式                                                   |
| custom            | `org.example.MyTypeFilter`   | 对`org.springframework.core.type.TypeFilter`接口的自定义实现 |

例如:

~~~java
@Configuration
@ComponentScan(basePackages = "org.example",
		includeFilters = @Filter(type = FilterType.REGEX, pattern = ".*Stub.*Repository"),
		excludeFilters = @Filter(Repository.class))
public class AppConfig {
	// ...
}
~~~

或者XML配置：

~~~xml
<beans>
	<context:component-scan base-package="org.example">
		<context:include-filter type="regex"
				expression=".*Stub.*Repository"/>
		<context:exclude-filter type="annotation"
				expression="org.springframework.stereotype.Repository"/>
	</context:component-scan>
</beans>
~~~

也可以通过在`@ComponentScan`注解上设置`useDefaultFilters=false`或者设置`<component-scan/>`元素的属性`use-default-filters="false"` 来禁用默认过滤器。这禁止了对使用`@Component`、`@Repository`、`@Service`、`@Controller`、`@RestController`或`@Configuration`注解或元注解的类的自动检测。

### 在组件中定义bean

可以在被`@Component`注释的组件中使用`@Bean`来向容器注册bean，就像在`@Configuration`中的`@Bean`行为一样：

~~~java
@Component
public class FactoryMethodComponent {

	@Bean
	@Qualifier("public")
	public TestBean publicInstance() {
		return new TestBean("publicInstance");
	}

	public void doWork() {
		//组件方法......
	}
}
~~~

### 扫描组件的名称

当组件被类路径包扫描器扫描并加载到容器中时，其beanName将由`BeanNameGenerator`解析生成。

默认将使用`AnnotationBeanNameGenerator`:如果对应的组件注解有`value`属性，那么会使用value属性作为beanName：

~~~java
@Component("d")
public class Dog {
	......
}
~~~

否则将使用未大写的非限定类名，例如类名为`MovieFinderImpl `,则默认的beanName为`movieFinderImpl`

如果需要使用自定义的`BeanNameGenerator`，可以设置`@ComponentScan`的`nameGenerator `属性：

~~~java
@Configuration
@ComponentScan(basePackages = "org.example", nameGenerator = MyNameGenerator.class)
public class AppConfig {
	// ...
}
~~~

或者XML配置：

~~~xml
<beans>
	<context:component-scan base-package="org.example"
		name-generator="org.example.MyNameGenerator" />
</beans>
~~~

### 设置组件Scope

可以使用`@Scope`注解设置组件的scope，默认情况下，为单例：

~~~java
@Scope("prototype")
@Repository
public class MovieFinderImpl implements MovieFinder {
	// ...
}
~~~

### `@Qualifier `

在使用`@Autowired`进行自动装配时，可以使用`@Qualifier`匹配bean 的qualifier元数据值来缩小匹配范围。bean的qualifier值可以使用XML中的`<qualifier/>`标签进行设置。

在注解中，也可以在备选组件的类上注释`@Qualifier`来设置当前bean的qualifier值：

~~~java
@Component
@Qualifier("Action")
public class ActionMovieCatalog implements MovieCatalog {
	// ...
}
~~~

# 基于Java的容器配置

现在介绍的注解已经能够大大简化XML配置了，但是目前为止XML配置仍然是必须的。

Spring提供了一些注解，让我们只用Java代码就能配置Spring容器，完全替代XML配置。这就是Spring的Java配置。

在Spring的Java配置中，核心的组件是`@Configuration`注解的类和`@Bean`注解的方法

`@Bean`注解用来指示当前方法是用来实例化、配置并初始化一个新对象，这个新对象会被Spring的IoC容器管理的。`@Bean`注解与XML配置中的`<bean/>`标签起着相同的作用。可以在任何`@Component`类中使用`@Bean`，但最好搭配`@Configuration`类使用

用`@Configuration`注解一个类，表示它的主要目的是作为bean定义的来源。此外，`@Configuration`类允许通过在同一类中调用其他@Bean方法来定义bean之间的依赖关系，一个简单示例如下：

~~~java
@Configuration
public class AppConfig {

	@Bean
	public MyServiceImpl myService() {
		return new MyServiceImpl();
	}
}
~~~

上述的Java代码等价于下面的XML配置：

~~~xml
<beans>
	<bean id="myService" class="com.acme.services.MyServiceImpl"/>
</beans>
~~~

## `AnnotationConfigApplicationContext`

`AnnotationConfigApplicationContext`类可以接收`@Configuration`类作为输入，此外还可以接受普通的`@Component`类

当`@Configuration`类被作为输入提供时，`@Configuration`类本身会被注册为一个bean，而且该类中所有声明的`@Bean`方法也都将被注册为bean.

当`@Component`类和JSR-330类作为`AnnotationConfigApplicationContext`的输入时，它们会被注册为bean，并假设在必要的地方在这些类中使用了依赖注入元数据，如`@Autowired`

### 注册组件

就像实例化`ClassPathXmlApplicationContext`时输入Spring XML文件，也可以在实例化`AnnotationConfigApplicationContext`时使用`@Configuration`类作为输入。这样就完全不需要XML文件来使用Spring容器了：

~~~java
public static void main(String[] args) {
	ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
	MyService myService = ctx.getBean(MyService.class);
	myService.doStuff();
}
~~~

或者其他普通`@Component`类也可以作为其入参：

~~~java
public static void main(String[] args) {
	ApplicationContext ctx = new AnnotationConfigApplicationContext(MyServiceImpl.class, Dependency1.class, Dependency2.class);
	MyService myService = ctx.getBean(MyService.class);
	myService.doStuff();
}
~~~

其中`MyServiceImpl`通过`@Autowired`依赖于`Dependency1`和`Dependency2`

除了使用构造器输入`@Configuration`类和`@Component`类，还可以使用其`register()`方法输入：

~~~java
public static void main(String[] args) {
	AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();
	ctx.register(AppConfig.class, OtherConfig.class);
	ctx.register(AdditionalConfig.class);
	ctx.refresh();
	MyService myService = ctx.getBean(MyService.class);
	myService.doStuff();
}
~~~

### 组件扫描

要开启组件扫描，需要在`@Configuration`类上注释`@ComponentScan`注解，如下所示：

~~~java
@Configuration
@ComponentScan(basePackages = "com.acme")
public class AppConfig  {
	// ...
}
~~~

这类似于下面XML配置：

~~~xml
<beans>
	<context:component-scan base-package="com.acme"/>
</beans>
~~~

除此之外，也可以使用其`scan()`方法实现相同的组件扫描功能:

~~~java
public static void main(String[] args) {
	AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();
	ctx.scan("com.acme");
	MyService myService = ctx.getBean(MyService.class);
}
~~~

## `@Bean`

`@Bean`是一个方法级别的注解，与XML配置中的`<bean/>`标签对应，注解提供`<bean/>`标签支持的一部分属性：

- `init-method`
- `destroy-method`
- `autowire`
- `name`

可以在带有`@Configuration`注解或`@Component`注解的类中使用`@Bean`注解

### 声明Bean

要在`ApplicationContext`中注册一个bean，可以使用`@Bean`注释一个方法，这个bean 的类型由方法的返回值指定。默认情况下，bean的名称与方法名相同，例如：

~~~java
@Configuration
public class AppConfig {

	@Bean
	public TransferServiceImpl transferService() {
		return new TransferServiceImpl();
	}
}
~~~

上面代码等同于下面XML配置：

~~~xml
<beans>
	<bean id="transferService" class="com.acme.TransferServiceImpl"/>
</beans>
~~~

### Bean依赖

一个`@Bean`注解的方法可以有任意多个参数，这些参数就是当前bean的依赖项，例如：

~~~java
@Configuration
public class AppConfig {

	@Bean
	public TransferService transferService(AccountRepository accountRepository) {
		return new TransferServiceImpl(accountRepository);
	}
}
~~~

在实例化bean对象，调用`@Bean`方法时，Spring会自动匹配容器中匹配的bean依赖项到方法入参，

### 声明周期回调方法

由`@Bean`方法注册的bean支持Spring容器支持的所有生命周期回调方法，如 `@PostConstruct` 和`@PreDestroy` 、 `InitializingBean`,和`DisposableBean`接口以及`*Aware`接口

`@Bean`也提供`initMethod `和`destroyMethod `属性，类似于XML`<bean/>`的`init-method`和`destroy-method`属性:

~~~java
public class BeanOne {

	public void init() {
		// initialization logic
	}
}

public class BeanTwo {

	public void cleanup() {
		// destruction logic
	}
}

@Configuration
public class AppConfig {

	@Bean(initMethod = "init")
	public BeanOne beanOne() {
		return new BeanOne();
	}

	@Bean(destroyMethod = "cleanup")
	public BeanTwo beanTwo() {
		return new BeanTwo();
	}
}
~~~

### 指定Scope

可以在`@Bean`方法上直接注释`@Scope`以设置对应bean的scope：

~~~java
@Configuration
public class MyConfiguration {

	@Bean
	@Scope("prototype")
	public Encryptor encryptor() {
		// ...
	}
}
~~~

### 指定beanName

默认情况下，bean的名称与方法名相同，这种行为可以使用`@Bean`的`name`属性进行覆盖：

~~~java
@Configuration
public class AppConfig {

	@Bean("myThing")
	public Thing thing() {
		return new Thing();
	}
}
~~~

可以通过让单个name属性接收一个字符串数组，来给bean起多个别名，例如：

~~~java
@Configuration
public class AppConfig {

	@Bean({"dataSource", "subsystemA-dataSource", "subsystemB-dataSource"})
	public DataSource dataSource() {
		// instantiate, configure and return DataSource bean...
	}
}
~~~

## `@Configuration`

`@Configuration`是一个类级别的注解，指示一个对象是bean定义的来源。这个对象通过`@Bean`方法声明bean。

### Bean依赖关系

当bean之间存在依赖关系时，可以用`@Bean`方法调用另一个`@Bean`方法来表达这种依赖关系

~~~java
@Configuration
public class AppConfig {

	@Bean
	public BeanOne beanOne() {
		return new BeanOne(beanTwo());
	}

	@Bean
	public BeanTwo beanTwo() {
		return new BeanTwo();
	}
}
~~~

这里`beanOne`通过构造函数注入接收到了对`beanTwo`的引用

这种声明bean之间依赖关系的方法只在`@Bean`方法被声明在`@Configuration`类中时才有效。不能使用普通的`@Component`类来声明bean之间的依赖关系。

### 实现方法注入

Spring提供方法注入机制，使用XML的`<lookup-method/>`和注解的`@Lookup`注解都可以实现方法注入。

在使用`@Configuration`的Java代码配置时，可以直接创建子类，重写抽象方法的方式来查找新的prototype对象，实现方法注入，如：

~~~java
@Bean
@Scope("prototype")
public AsyncCommand asyncCommand() {
	AsyncCommand command = new AsyncCommand();
	return command;
}

@Bean
public CommandManager commandManager() {

	return new CommandManager() {
		protected Command createCommand() {
			return asyncCommand();
		}
	}
}
~~~

其中`CommandManager`为：

~~~java
public abstract class CommandManager {
	public Object process(Object commandState) {

		Command command = createCommand();

		command.setState(commandState);
		return command.execute();
	}

	protected abstract Command createCommand();
}
~~~

## `@Import`

在XML配置中，可以使用`<import/>`标签引入外部其他xml配置文件，对应的，Java配置可以使用`@Import`注解引入其他`@Configuration`类，例如：

~~~java
@Configuration
public class ConfigA {

	@Bean
	public A a() {
		return new A();
	}
}

@Configuration
@Import(ConfigA.class)
public class ConfigB {

	@Bean
	public B b() {
		return new B();
	}
}
~~~

Spring4.2后，`@Import`也可以导入普通组件。

## 结合XML和Java配置

Spring的`@Configuration`类支持不旨在取代Spring XML。Spring的XML命名空间仍然是配置容器的好方法。

在以XML配置为中心实例化容器时，可以使用`ClassPathXmlApplicationContext`，如果需要使用Java配置，不要忘了`@Configuration`类也是组件类，直接启用`<context:annotation-config/>`并在XML中配置`@Configuration`类即可。

要以Java为中心实例化容器时可以使用`AnnotationConfigApplicationContext`，如果需要导入XML配置，可以使用`@ImportResource`

### 以XML配置为中心

要在在以XML配置为中心实例化的容器中使用配置类，可以直接在xml文件中配置`@Configuration`类，并启用`<context:annotation-config/>`组件以识别出`@Configuraiton`和`@Bean`注解即可：

~~~java
@Configuration
public class AppConfig {

	@Autowired
	private DataSource dataSource;

	@Bean
	public AccountRepository accountRepository() {
		return new JdbcAccountRepository(dataSource);
	}

	@Bean
	public TransferService transferService() {
		return new TransferService(accountRepository());
	}
}
~~~

对应的XML配置文件`system-test-config.xml` 为：

~~~xml
<beans>
	<!-- enable processing of annotations such as @Autowired and @Configuration -->
	<context:annotation-config/>
	<context:property-placeholder location="classpath:/com/acme/jdbc.properties"/>

	<bean class="com.acme.AppConfig"/>

	<bean class="org.springframework.jdbc.datasource.DriverManagerDataSource">
		<property name="url" value="${jdbc.url}"/>
		<property name="username" value="${jdbc.username}"/>
		<property name="password" value="${jdbc.password}"/>
	</bean>
</beans>
~~~

容器的启动代码为：

~~~java
public static void main(String[] args) {
	ApplicationContext ctx = new ClassPathXmlApplicationContext("classpath:/com/acme/system-test-config.xml");
	TransferService transferService = ctx.getBean(TransferService.class);
	// ...
}
~~~

因为`@Configuration`本身就带有`@Component`的元注解，所以用`@Configuration`注解的类本身扣手组件扫描的候选对象。所以也可以直接启用`<context:component-scan/>`来扫描`@Configuration`类

### 以Java配置为中心

以Java配置为中心的容器也可以通过`@ImportResource`注解引入XML配置。例如：

~~~java
@Configuration
@ImportResource("classpath:/com/acme/properties-config.xml")
public class AppConfig {

	@Value("${jdbc.url}")
	private String url;

	@Value("${jdbc.username}")
	private String username;

	@Value("${jdbc.password}")
	private String password;

	@Bean
	public DataSource dataSource() {
		return new DriverManagerDataSource(url, username, password);
	}
}
~~~

其中XML文件properties-config.xml配置为：

~~~xml
<beans>
	<context:property-placeholder location="classpath:/com/acme/jdbc.properties"/>
</beans>
~~~

~~~properties
jdbc.properties
jdbc.url=jdbc:hsqldb:hsql://localhost/xdb
jdbc.username=sa
jdbc.password=
~~~

容器启动代码为：

~~~java
public static void main(String[] args) {
	ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
	TransferService transferService = ctx.getBean(TransferService.class);
	// ...
}
~~~

