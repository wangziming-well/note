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

容器启动阶段主要是将bean配置信息读取并加载成`BeanDefinition`。

Spring提供了一种叫做`BeanFactoryPostProcessor`的容器扩展机制 .该机制允许我们在容器实例化相应对象之前，对注册到容器的`BeanDefinition`所保存的信息做相应的修改。这就相当于在容器启动阶段最后加入一道工序，让我们对最终的`BeanDefinition`做一些额外的操作，比如修改其中bean定义的某些属性，为bean定义增加其他信息等  

如果要自定义实现`BeanFactoryPostProcessor`，通常我们需要实现`org.springframework.
beans.factory.config.BeanFactoryPostProcesso`r接口  

因为一个容器可能拥有多个`BeanFactoryPostProcessor`，这个时候可能需要实现类同时实现Spring的`org.springframework.core.Ordered`接口，以保证各个`BeanFactoryPostProcessor`可以按照预先设定的顺序执行（如果顺序紧要的话）  

接口定义如下：

~~~java
@FunctionalInterface
public interface BeanFactoryPostProcessor {
    void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;
}
~~~

但是，因为Spring已经提供了几个现成的BeanFactoryPostProcessor实现类，所以，大多时候，我们很少自己去实现某个`BeanFactoryPostProcessor`。常用的有：

* `PropertyPlaceholderConfigurer  `(已过时)
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

* 通过`ApplicationContext`使用`BeanFactoryPostProcessor  `

  `ApplicationContext`会自动识别配置文件中的`BeanFactoryPostProcessor`并应用它

  所以只需要将`BeanFactoryPostProcessor`添加到配置文件中即可生效

  ~~~xml
  <bean class="...PropertyPlaceholderConfigurer">
      <property name="location" value="demo.properties"/>
  </bean>
  ~~~

### `PropertyPlaceholderConfigurer  `

`PropertyPlaceholderConfigurer`允许我们在XML配置文件中使用占位符（PlaceHolder），并将这些占位符所代表的资源单独配置到简单的properties文件中来加载。 

实例：

如果`PropertyPlaceholderConfigurer`读取装配的properties文件如下：

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

使用了`PropertyPlaceholderConfigurer`后，它替换占位符的逻辑如下：

* `BeanFactory`在第一个阶段加载完成所有配置的配置信息是，`BeanDefinition`中保持的信息仍然是占位符形式的，如`${dog.age}`
* 当`PropertyPlaceholderConfigurer`调用postProcessBeanFactory进行后处理时，会将properties文件中的配置信息来替换相应`BeanDefinition`中占位符所表示的属性值

### `PropertyOverrideConfigurer  `

`PropertyOverrideConfigurer  `可以通过读取相应规则的properties文件将bean定义的property信息进行覆盖替换  

`PropertyOverrideConfigurer`读取的properties文件应该满足如下格式：

~~~properties
beanName.propertyName=value
~~~

这样，将将`PropertyOverrideConfigurer`加载到容器之后，bean对象xml文件中的值，将被properties文件中的值覆盖

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

当容器中配置的多个`PropertyOverrideConfigurer`对同一个bean定义的同一个property值进行处理的时候，最后一个将会生效。

### `CustomEditorConfigurer  `

`CustomEditorConfigurer`辅助性地将后期会用到的信息注册到容器，对`BeanDefinition`没有做任何变动  

spring容器通过一般通过XML配置文件来配置bean对象的类型,而XML只能记载字符串,但最终要转换为对象类型

完成这种由字符串到具体对象的转换，都需要这种转换规则相关的信息，而`CustomEditorConfigurer`就是帮助我们传达类似信息的

Spring内部通过JavaBean的`PropertyEditor`来帮助进行String类型到其他类型的转换工作  

`CustomEditorConfigurer`可以完成PropertyEditor到容器的注册

但现在主要通过`PropertyEditorRegistrar`来注册`PropertyEditor`

## Bean实例化阶段

第一阶段容器启动阶段结束后，容器目前拥有所有对象的`BeanDefinition`来保存实例化阶段将要用的必要信息

只有当请求方调用`BeanFactory`的`getBean()`来请求对象实例时，才会可能触发Bean实例化阶段的活动  

`getBean()`方法可以被客户端现实地调用，也可以在容器内部被隐式地调用，隐式调用有如下情况：

* 如果请求的对象有依赖对象，且依赖依赖对象没有实例化；那么容器会先内部调用getBean()实例化依赖对象

  然后再实例化被请求的对象

* 对`BeanFactory`来说：对象实例化默认采用延迟初始化。只有当getBean被显示调用时，容器才会开始实例化对象；但是`ApplicationContext  `启动后就会实例化所有的bean，这样的实例化也是容器隐式进行的

调用`getBean()`方法时会先检查bean是否已经实例化，如果没有，会调用`createBean()`方法来进行具体的对象实例化  

* 可以在`AbstractBeanFactory`类的代码中查看到`getBean()`方法的完整实现逻辑
* 可以在其子类`AbstractAutowireCapableBeanFactory`的代码中查看`createBean()`方法的实现逻辑(doCreateBean)

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

默认情况下，容器内部采用的是`CglibSubclassingInstantiationStrategy`,通过其和相应的`BeanDefinition`来构造对象实例

### `BeanWrapper  `

上一步容器并不会直接返回构造完成的对象实例，而是以`BeanWrapper`对构造完成的对象实例进行包裹，返回相应的`BeanWrapper`实例  

`BeanWrapper`接口通常在Spring框架内部使用，它在spring中只有一个实现类`BeanWrapperImpl`。其作用是对某个bean进行“包裹”，然后对这个“包裹”的bean进行操作，比如设置或者获取bean的相应属性值  

`BeanWrapperImpl`的继承关系如下：

* `BeanWrapper`继承了`PropertyAccessor`接口，可以以统一的方式对对象属性进行访问，设置对象属性值  

* `BeanWrapper`定义同时又直接或者间接继承了`PropertyEditorRegistry`和`TypeConverter`接口

  所以`BeanWrapper`会使用容器注册的`PropertyEditor`来进行转换类型  

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

和`BeanFactoryPostProcessor`作用类似：

* 实现`BeanFactoryPostProcessor`的`BeanFactory`会在启动阶段的最后进行后处理

* 注册了`BeanPostProcessor`的`BeanFactory`(通过`ConfigurableBeanFactory.addBeanPostProcessor()`方法注册)会在实例化bean的合适时机进行后处理

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

通常使用`BeanPostProcessor  `的场景是处理标记接口实现类,或者为当前对象提供代理实现

比如`ApplicationContext`对应的那些`Aware`接口实际上就是通过`BeanPostProcessor`的方式进行处理的  

(见`ApplicationContextAwareProcessor.postProcessBeforeInitialization`)

Spring的AOP则使用`BeanPostProcessor`来为对象生成相应的代理对象  

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

Spring容器在实例化过程中完成"`BeanPostProcessor`的前置处理 "过程后,会检查对象是否实现了`InitializingBean`接口,如果是,则会调用其`afterPropertiesSet()`方法进一步调整对象实例的状态  

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

`ApplicationContext`是Spring提供的更高级的容器，相较于`BeanFactory`，它提供了更多的功能:

![ApplicationContext](https://gitee.com/wangziming707/note-pic/raw/master/img/ApplicationContext.png)

包括`BeanFactoryPostProcessor`、 `BeanPostProcessor`以及其他特殊类型bean的自动识别、容器启动后bean实例的自动初始化、国际化的信息支持、容器内事件发布等  

Spring 为`ApplicationContext`提供了几个常用的实现：

* `FileSystemXmlApplicationContext  `:从文件系统加载bean定义以及相关资源的`ApplicationContext`实现  
* `ClassPathXmlApplicationContext  `:从Classpath加载bean定义以及相关资源的`ApplicationContext`实现  
* `XmlWebApplicationContext  `Web应用程序的`ApplicationContext`实现  

## 统一资源加载策略

Spring提供了一套基于`org.springframework.core.io.Resource`和`org.springframework.core.io.ResourceLoader`接口的资源抽象和加载策略。  

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

通过`ResourceLoader.getResource(String location)`方法我们就可以根据指定的资源位置，定位到具体的资源实例。

#### `DefaultResourceLoader`

`DefaultResourceLoader `是`ResourceLoader`的一个默认的实现类，它默认的资源查找处理逻辑如下：

* 首先检查资源路径是否以`classpath:`前缀打头，如果是，则尝试构造`ClassPathResource`类型资源并返回  
* 资源路径不是以`classpath:`前缀打头：
  * 通过URL来定位资源，如果没抛出`MalformedURLException`，有则会构造`UrlResource`类型的资源并返回  
  * 如果还是无法根据资源路径定位指定的资源，则委派`getResourceByPath(String) `方法来定位资源  ：默认实现逻辑是，构造`ClassPathResource`类型的资源并返回

如果最终没有找到符合条件的相应资源， `getResourceByPath(String)`方法就会构造一个实际上并不存在的资源并返回  

#### `FileSystemResourceLoader  `

为了避免`DefaultResourceLoader`在最后`getResourceByPath(String)`方法上的不恰当处理,可以使用`FileSystemResourceLoader  `,它继承自`DefaultResourceLoader`但覆写了`getResourceByPath(String) ` 使之从文件系统加载资源并以`FileSystemResource`类型返回  

#### `ResourcePatternResolver`

`ResourcePatternResolver `接口是`ResourceLoader  `的拓展,它的定义如下:

~~~java
public interface ResourcePatternResolver extends ResourceLoader {

	String CLASSPATH_ALL_URL_PREFIX = "classpath*:";

	Resource[] getResources(String locationPattern) throws IOException;

}
~~~

它的`getResources`方法可以根据指定的资源路径匹配模式 返回多个Resource实例  

##### `PathMatchingResourcePatternResolver  `

`PathMatchingResourcePatternResolver`是`ResourcePatternResolver`最常用的实现。

在构造`PathMatchingResourcePatternResolver`实例的时候，可以指定一个`ResourceLoader`，如果不指定的话，则 `PathMatchingResourcePatternResolver`内 部 会 默 认 构 造 一 个 `DefaultResourceLoader`实例  

`PathMatchingResourcePatternResolver`内部会将匹配后确定的资源路径，
委派给它的`ResourceLoader`来查找和定位资源  

### ApplicationContext与ResourceLoader

从之前的`ApplicationContext`继承关系图中可以看到，它实现了`ResourcePatternResolver  `,所以也间接实现了`ResourceLoader`

所以，任何的`ApplicationContext`实现都可以看作是一个`ResourceLoader`甚至`ResourcePatternResolver  `

通常，所有的`ApplicationContext  `实现类都直接或间接地继承`AbstractApplicationContext`  

`AbstractApplicationContext  `直接继承`DefaultResourceLoader`,所以它的`getResource  `就是直接使用的`DefaultResourceLoader  `

而`getResources `使用的是`PathMatchingResourcePatternResolver  `的实例完成

这样 `ApplicationContext`就支持了Spring的统一资源加载

作为`ResourceLoader`的`ApplicationContext`显然可以直接调用getResource方法来进行资源定位获取，但除此之外，作为容器和`ResourceLoader`身份的`ApplicationContext`显然后更多的应用

#### ResourceLoader类型的注入

之前的`Aware`章节已经提到过 `ResourceLoaderAware  `会将`ApplicationContext`‘容器自身作为`ResourceLoader`注入到bean对象中

这样就可以在bean对象中直接使用`ResourceLoader`

享受Spring提供的统一资源加载服务

#### Resource类型的注入

如果使用`BeanFactory`，那么对于那些Spring容器提供的默认的`PropertyEditors`无法识别的对象类型，我们可以提供自定义的`PropertyEditor`实现并注册到容器中，以供容器做类型转换的时候使用。

默认情况下，BeanFactory容器不会为Resource类型提供相应的PropertyEditor  

但ApplicationContext默认就可以正确识别Resource类型并转换后注入相关对象

它在启动时，会通过ResourceEditorRegistrar  来注册ResourceEditor  到容器中

ResourceEditor  就是Spring提供的针对Resource类型的PropertyEditor  

所以如果使用ApplicationContext，那么可以直接将XML文件中的RUL字符串转换为Resource对象

#### 特定ApplicationContext的Resource加载行为

对于URL所表示的资源路径来说，通常开头会有一个协议前缀，比如`file:`、 `http:`、`ftp:  `等

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

Spring的ApplicationContext容器内的事件发布机制，主要用于单一容器内的简单消息通知和处理，并不适合分布式、多进程、多容器之间的事件通知。  

### `ApplicationEvent  `

Spring容器内自定义的事件类型，继承自`EventObject`

默认情况下，Spring提供了三个实现：

* `ContextClosedEvent  `:` ApplicationContext`容器在即将关闭的时候发布的事件类型  
* `ContextRefreshedEvent  `: `ApplicationContext`容器在初始化或者刷新的时候发布的事件类型  

* `RequestHandledEvent`： Web请求处理后发布的事件，其有一子类`ServletRequestHandledEvent`提供特定于Java EE的Servlet相关事件  

### `ApplicationListener  `

ApplicationContext容器内使用的自定义事件监听器接口定义，继承自`java.util.EventListener`。 

它的定义如下：

~~~java
@FunctionalInterface
public interface ApplicationListener<E extends ApplicationEvent> extends EventListener {
	void onApplicationEvent(E event);
}
~~~

`ApplicationContext`容器在启动时，会自动识别并加载`EventListener`类型bean定义，一旦容器内有事件发布，将通知这些注册到容器的`EventListener`。  

### `ApplicationEventPublisher`

`ApplicationContext`继承了 `ApplicationEventPublisher`，它的定义如下：

```java
@FunctionalInterface
public interface ApplicationEventPublisher {

   default void publishEvent(ApplicationEvent event) {
      publishEvent((Object) event);
   }

   void publishEvent(Object event);

}
```

接口提供了`publishEvent`方法，所以`ApplicationContext`还能作为事件发布者的角色

`ApplicationContext`容器的具体实现类 把实现事件的发布和监听器的注册任务委托给了`ApplicationEventMulticaster  `接口的实现类

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

* 抽象实现类`AbstractApplicationEventMulticaster`它实现了事件监听器的管理功能；出于灵活性和扩展性考虑，事件的发布功能则委托给了其子类  
  
* 实现类`SimpleApplicationEventMulticaster`: `AbstractApplicationEventMulticaster`的一个子类实现，添加了事件发布功能的实现，其默认使用了SyncTaskExecutor进行事件的发布，事件是同步顺序发布的

### 发布自定义事件

需要自定义`ApplicationEvent `事件

需要自定义的监听器 `ApplicationListener`并注册到 `ApplicationContext`容器中

需要自定义的事件发布者`ApplicationEventPublisher`并注册到`ApplicationContext`容器中

将`ApplicationEventPublisher`注入到bean中

bean就拥有了发布事件的能力

## 加载多配置文件

在创建容器时，可以一次性加载多个配置文件到ApplicationContext容器中：

~~~java
//使用字符串数组
String[] locations = new String[]{ "conf/dao-tier.springxml", 
"conf/view-tier.springxml", "conf/business-tier.springxml"};
ApplicationContext container = new ClassPathXmlApplicationContext(locations);
//使用通配符
ApplicationContext container = new FileSystemXmlApplicationContext("conf/**/*.springxml");
~~~

# 循环依赖

Spring ioc容器中，在bean对象的创建阶段，如果两个或者多个单例bean相互依赖，在不加任何处理的情况下，这这些bean会因为互相依赖而无法创建成功。

在不做任何处理的情况下，Spring 容器只需要为单例bean对象准备一个缓存即可，这样获取一个bean对象的过程如下：

* 查询单例bean缓存，有则返回，没有则进行下一步
* 创建单例bean
  * 实例化bean
  * 依赖注入，获取当前bean依赖的bean，并注入；获取依赖bean的过程一样，从第一步开始。
  * 将创建好的bean放入缓存并返回

如果遇到循环依赖，那么上述流程就会出现问题，例如：

~~~xml
<bean id="a" class="A">
    <property name="b" ref="b"/>
</bean>

<bean id="b" class="B">
    <property name="a" ref="a"/>
</bean>
~~~

一开始A和B都没有实例化，都不在缓存中。

我们从A的实例化开始，首先因为缓存中没有A，所有需要创建A对象，先实例化A，然后需要依赖注入B，所以需要先获取B对象，因为缓存中没有B，需要创建B对象，先实例化B，然后又需要依赖注入A，此时因为缓存中没有A，那么开始重复创建A的实例。

这样的过程会持续循环下去，导致不断重复创建A和B的实例，最终导致OOM。

## 两个缓存

要想解决循环依赖，那么就需要让已经实例化但没有完成创建的bean能够被引用。这就需要再引入一个缓存。有了两个缓存后，通过容器获取bean的流程变为：

* 查询缓存一，有则返回，没有则进行下一步
* 创建单例bean
  * 实例化bean对象
  * 对象存入缓存二中
  * 依赖注入，获取当前bean依赖的bean，并注入；获取依赖bean的过程一样，从第一步开始。
  * 将创建好的bean放入缓存一并返回

在这个方案中，互相依赖的A，B对象的实例化过程如下：在获取对象A时，先实例化A，在将A存入缓存二，此时因为依赖注入需要实例化B，实例化B后放入缓存二，因为依赖注入需要获取A，此时A已经在缓存二中可以被获取了，所以可以直接返回A并依赖注入。这样流程可以继续并完成。

## 动态代理中的两个缓存

使用两个缓存可以解决大部分场景下的循环依赖，但是在有AOP代理的情况下，事情又发生了变化。

我们知道在AOP中作为目标的类会被AOP使用代理生成代理类以织入横切逻辑。所以在之前的流程中需要加入创建代理对象的过程：

* 查询缓存一，有则返回，没有则进行下一步
* 创建单例bean
  * 实例化bean对象
  * 对象存入缓存二中
  * 依赖注入，获取当前bean依赖的bean，并注入；获取依赖bean的过程一样，从第一步开始。
  * **创建bean对象的代理对象**
  * 将创建好的bean代理对象放入缓存一并返回

这里我们将创建bean对象的代理对象放到了依赖注入之后，所以只有缓存一中的对象是代理对象，而缓存二中的对象是原对象。这会导致在循环依赖中bean获取的依赖对象不是代理对象而是原对象。而我们的预期是容器对外暴露的bean都是处理过后的代理对象，所以这显然不行，一种方法是将代理对象的生成放到存入缓存二之前：

* 查询缓存一，有则返回，没有则进行下一步
* 创建单例bean
  * 实例化bean对象
  * **创建bean对象的代理对象**
  * 对象存入缓存二中
  * 依赖注入，获取当前bean依赖的bean，并注入；获取依赖bean的过程一样，从第一步开始。
  * 将创建好的bean代理对象放入缓存一并返回

这样暴露在缓存一和缓存二中的对象就都是处理过后的代理对象了。

但是对于正常的AOP代理bean，其代理对象的创建时机最好还是放到bean创建完成后，放入缓存一之前。所以Spring在这个方案的基础上进行了改进

## Spring解决方案

所以Spirng在上面方案的基础上进行了改进，将生成AOP代理对象的时机延后到了bean生成完毕，放入缓存一之前，具体流程如下：

* 从缓存中查询bean：
  * 查询缓存一中是否有bean实例，如果有，直接返回bean实例，如果没有，进入下一步
  * 判断bean是否正在构建(通过查询辅助单例缓存)，如果否，直接返回null，如果是，进入下一步
  * 查询缓存二中是否有bean实例，如果有，直接返回bean实例，如有没有，进入下一步
  * 查询缓存三中是否有bean对应的`ObjectFactory`,如果没有，直接返回null，如果有，进行下一步
  * 获取上一步的`ObjectFactory`，并通过它生成单例bean，将单例bean 放入缓存二并返回
* 如果上一步返回的bean为null，创建单例bean：
  * 将beanName放入辅助单例缓存
  * 实例化对象
  * 生成对应的`ObjectFactory`放入缓存三
  * 依赖注入，获取当前bean依赖的bean，并注入；获取依赖bean的过程一样，从第一步开始。
  * 如果需要生成AOP代理，那么获取AOP代理对象：
    * 判断AOP代理是否已经存在(通过查询缓存二)，如果存在，直接返回代理对象，否则进入下一步
    * 获取刚刚放入缓存三的`ObjectFactory`，通过它生成并返回bean代理对象
  * 将bean存入缓存一

首先我们在介绍下流程中涉及到的缓存：

* 缓存一：完全创建好的单例bean
* 缓存二：已实例化但未完成创建的单例bean
* 缓存三：单例bean对应的`ObjectFactory`
* 辅助单例缓存：正在构建的单例bean

这个方案能够在循环依赖场景下延迟AOP代理对象的生成时机：新增一个缓存三，并在缓存三中放入bean对应的`ObjectFactory`工厂对象。这样在实例化对象后就将对应的`ObjectFactory`放入缓存三,只有在循环依赖的情况下，会提前获取`ObjectFactory`对象并生成代理对象。否则正常情况下，代理对象的生成时机仍然放在bean构建完毕，放入缓存一前。

