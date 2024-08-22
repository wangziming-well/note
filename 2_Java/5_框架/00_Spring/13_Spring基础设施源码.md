# 路径匹配

在继续分析前，需要先了解下面Spring 提供的进行路径匹配的Util工具类：

* `org.springframework.util.PathMatcher`
* `org.springframework.web.util.pattern.PathPattern`

这两个工具类都是用于路径匹配，主要功能基本一致，`PathPattern`是Spring5新增的路径匹配器。旨在取代`PathMatcher`，性能也更强。

## `PathMatcher`

`PathMatcher`只有一个实现类`AntPathMatcher`，主要用来判断给定的模式`pattern`和路径`path`是否匹配。

路径由路径分隔符分割成若干个路径段。

其中`pattern`模式的匹配规则如下：

* `?`匹配一个任意字符（不包括路径分隔符）
* `*`匹配零个或多个任意字符（不包括路径分隔符）
* `**`匹配路径中零个或者多个目录
* `{var:regex}`:匹配一个满足`regex`代表的正则表达式路径段，并将匹配到的值赋值给变量名`var`，可以省略正则表达式，此时匹配任意路径段

注意：模式和路径要么都是相对的，要么都是绝对的，否则无法进行匹配(匹配结果为false)

### `match()`

`match()`方法对路径进行完全匹配：

~~~java
public boolean match(String pattern, String path);
~~~

使用示例：

~~~java
AntPathMatcher matcher = new AntPathMatcher();

matcher.match("/test/a?c", "/test/abc"); //true
matcher.match("/test/a?c", "/test/abbc"); //false

matcher.match("/test/*.html", "/test/index.html"); //true
matcher.match("/test/*.html", "/test/index");

matcher.match("/test/*", "/test/index"); //true
matcher.match("/test/*", "/test/v1/index"); //false

matcher.match("/test/**/index.jsp", "/test/v1/v2/index.jsp");//true
matcher.match("/test/**/index.jsp", "/test/index.jsp");//true

matcher.match("test/**", "/test/v1/v2/index.jsp"); //false
matcher.match("test/**", "test/index.jsp"); //true
~~~

### `matchStart()`

进行不完全匹配，路径只需要完全匹配模式中前面部分的若干个路径段，例如：

~~~java
matcher.matchStart("/test/path/**", "/test"); //true
matcher.matchStart("/test/path/**", "/test/abc"); //false
~~~

### `extractPathWithinPattern()`

找出通配符`*`、`**`、`?`所在的路径段，并返回该路径段和其所有子路径，例如：

~~~java
matcher.extractPathWithinPattern("/test/**/*.html", "/test/p/index.jsp"); //  p/index.jsp
matcher.extractPathWithinPattern("/test/**/*.html", "/test/p/index.html"); // p/index.html
matcher.extractPathWithinPattern("/test/a?c/*", "/test/abc/v"); // abc/v
~~~

### `extractUriTemplateVariables()`

返回模板变量，即`{var:regex}`匹配的变量，例如：

~~~java
matcher.extractUriTemplateVariables("/test/{v1:\\w+}/{v2}.html", "/test/p/index.html");//  {v1=p, v2=index}
~~~

### `combine()`

`combine(String pattern1, String pattern2)`方法合并两个模式，主要逻辑如下：

* 如果两个模式其中一个为空(空字符串或者`null`)，直接返回另一个模式

* 如果`pattern1`和`pattern2`不相同，且`pattern1`不包含路径模板变量，且将`pattern2`作为路径时，`pattern1`完全匹配`pattern2`,直接返回`pattern2`

* 如果`pattern1`以`/*`结尾,去掉`pattern1`的后缀的`*`，然后拼接两个模式并返回

* 如果`pattern1`以`/**`结尾,直接拼接两个模式并返回

* 如果`pattern1`包含路径模板变量，或者`pattern1`没有匹配任意文件名(`pattern1`中不包含`*.`),或者路径分隔符为`.`,那么直接拼接两个模式并返回

* 走到这一步，说明`pattern1`不以`/*`、`/**`为后缀，不包含路径模板变量，会匹配任意文件名(包含`*.`),并且路径分隔符不为`.`

* 如果`pattern1`和`pattern2`都不匹配所有文件拓展名(`pattern`的中包含`.`，且后缀不为`.*`)，抛出一个`IllegalArgumentException`

  例如，对于`/test/*.jsp`和`/test/key.jsp`将抛出异常

* 最后，`pattern2`去掉文件后缀(去掉`.xxx`的后缀)，获得`pattern2` 的文件名`file2`(包含路径)

  * 如果`pattern1`匹配所有文件拓展名(以`*.*`作为后缀)，直接返回`pattern2`
  * 否则返回`file2`拼接上`pattern1`的文件拓展名


注意：两个模式的拼接会自动判断是否添加路径分隔符。

例如：

~~~java
matcher.combine("/test/**", "/test/*");  // /test/*
matcher.combine("/test/**", "test/*");  // /test/**/test/*
matcher.combine("/path1/*.*","/test/var.h");  // /test/var.h
matcher.combine("/path1/*.jsp","/test/var");  // /test/var.jsp
~~~

### `getPatternComparator()`

这个方法获取一个模式比较器，基于一个路径比较模式的通用性。通用性强的模式大于通用性弱的模式。

注意：模式的通用性是基于路径的，可能有两个模式A，B，对于路径P1来说A更通用，但对于路径P2来说B更通用。

在多个模式同时匹配上同一个路径的场景下，可以用模式比较器选出一个最佳/最优先(最不通用)的模式。比如一个URL请求能够匹配多个`Controller`时,可以选出一个最优先的`Controller`。

对于`pattern1`和`pattern2`和路径`path`；比较器按照下面顺序判断通用性，下面情况中`pattern1`更通用：

* `pattern1`为`null`或者等于`/**`  （都为`null`或者`/**`则通用性相同）
* `pattern2`与`path`相等（都与`path`相同则通用性相同）
* `pattern1`以`/**`为后缀且`pattern2`中没有`**`通配符
* `pattern1`和`pattern2`都以`/**`结尾，`pattern1`长度更短(模板变量长度算1)（长度相同相同则通用性相同）
* `pattern1`中有更多的通配符(`**`算两个通配符，`{}`和`*`算一个通配符)
* `pattern1`长度更短(模板变量长度算1)
* `pattern1`中单`*`通配符更多
* `pattern1`中模板变量更多
* 以上判断都通过则通用性相同

例如：

~~~java
AntPathMatcher matcher = new AntPathMatcher();
Comparator<String> comparator = matcher.getPatternComparator("/test/path/inde");
comparator.compare("/**", "/test/**"); //1
comparator.compare("/test/path/*.*", "/test/path/index.jsp"); // 1
comparator.compare("/test/path/**", "/test/**/inde"); //0
~~~

## 新路径匹配器

不同与`PathMatcher`把所有功能集中在一个类中,新的路径匹配围绕`PathPattern`拥有要给体系，在设计上更模块化，更加面向对象。涉及的几个类：

* `PathElement`：路径节点的抽象。一个路径可以拆分成多个路径节点对象。
* `PathContainer`：路径的结构化抽象。
* `PathPattern`：模式的抽象。路径模式匹配器的最核心API

* `PathPatternParser`：将一个String类型的模式解析为`PathPattern`实例，这是创建`PathPattern`实例的唯一方式

`PathPattern`提供的路径匹配功能基本和`AntPathMatcher`一样，不同之处在于：

* `**`通配符只能在模式最后
* 除了通配符：`?`、`*`、`**`、`{var:regex}`外，新提供一个`{*var} `用于捕获剩余所有路径段到变量`var`上

### `PathElement`

`PathElement`代表一个路径节点。例如路径段、路径分隔符。

`PathElement`中的`next`和`prev`指示当前节点的前后节点。所有的`PathElement`之间形成链状结构，构成一个完整的路径模板。

它不是一个公开的类，我们不需要仔细了解其实现细节，只需要了解一下它的不同实现即可。

它有8个实现类，代表不同类型的路径节点：

* `SeparatorPathElement`:路径分隔符
* `WildcardPathElement`：仅有`*`通配符的路径段
* `SingleCharWildcardedPathElement`：带有`?`通配符的路径段
* `WildcardTheRestPathElement`：表示路径剩余部分的路径段，例如在`/foo/**`中的`/**`
* `CaptureVariablePathElement`表示捕获路径变量的路径段，例如`/foo/{id}`的`{id}`
* `CaptureTheRestPathElement`表示捕获剩余所有路径的路径段，例如 `/foo/{*foobar}`中的`/{*foobar}`
* `LiteralPathElement`表示字面路径段，例如`/foo/bar`中的`foo`和`bar`
* `RegexPathElement`表示正则表达式路径段，例如`/foo/*_*/*_{foobar}`中的`*_*`和`*_{foobar}`

### `PathContainer`

是URI路径的结构化表示，它有如下内部接口/类：

* `PathContainer.Element`:表示路径元素，它有如下子接口：
  * `PathContainer.Separator`:表示路径分隔符
  * `PathContainer.PathSegment`:表示路径段
* `Options`路径解析的可选项，可以设置路径分隔符和是否解码解析路径段

主要方法如下：

* `value()`返回路径对应的字符串
* `elements()`返回路径元素的有序列表

* `subPath()`截取返回子路径
* `static parsePath()`根据字符串解析返回一个`PathContainer`

### `PathPattern`

`PathPattern`代表一个模式，用于和代表路径的`PathContainer`进行匹配

`PathPattern`只有由`PathPatternParser`创建，例如：

~~~java
String patternStr = ....;
PathPattern parse = PathPatternParser.defaultInstance.parse(patternStr);
~~~

其主要方法如下：

#### `matches()`

和指定的路径进行完全匹配。例如：

~~~java
PathPattern pattern = PathPatternParser.defaultInstance.parse("/test/*/*.jsp");
PathContainer path = PathContainer.parsePath("/test/v1/index.jsp");
boolean matches = pattern.matches(path); // true
~~~

#### `matchAndExtract()`

匹配并捕获路径变量和路径参数。

如果不匹配，返回null，否则它返回一个`PathPattern.PathMatchInfo`, 模式匹配路径时捕获的路径变量和路径参数，它持有如下参数：

* `Map<String, String>  uriVariables`:由模式中`{var}`和`{*var}`捕获的路径变量，
* `Map<String, MultiValueMap<String, String>> matrixVariables`：被捕获的路径参数(如果没有被模板捕获，那么即使路径有参数，该字段也为空)，例如，针对路径`/test/v1;a=1;b=2,3/index.html`:
  * 使用模式`/test/{version}/index.html`进行匹配，该参数为：`{version={a=[1], b=[2, 3]}}`
  * 使用模式`/test/v1/{filename}`进行匹配，该参数为`{}`

使用示例：

~~~java
PathPattern pattern = PathPatternParser.defaultInstance.parse("/test/{version}/{filename}");
PathContainer path = PathContainer.parsePath("/test/v1;a=1;b=2,3/index.html");
PathPattern.PathMatchInfo pathMatchInfo = pattern.matchAndExtract(path);
System.out.println(pathMatchInfo); //PathMatchInfo[uriVariables={filename=index.html, version=v1}, matrixVariables={version={a=[1], b=[2, 3]}}]

~~~

#### `matchStartOfPath()`

如果不匹配，方法返回`null`,否则返回`PathPattern.PathRemainingMatchInfo`它持有如下参数：

* ` PathContainer pathMatched`:与模式匹配的路径前缀
* `PathContainer pathRemaining`：不匹配的剩余路径
* `PathMatchInfo pathMatchInfo`：匹配的捕获参数

示例：

~~~java
PathPattern pattern = PathPatternParser.defaultInstance.parse("/test/{version}");
PathContainer path = PathContainer.parsePath("/test/v1;a=1;b=2,3/index.html");
PathPattern.PathRemainingMatchInfo result = pattern.matchStartOfPath(path);
System.out.println(result.getPathMatched()); // "/test/v1;a=1;b=2,3"
System.out.println(result.getPathRemaining()); // "/index.html"
System.out.println(result.getUriVariables()); // {version=v1}
System.out.println(result.getMatrixVariables()); // {version={a=[1], b=[2, 3]}}
~~~

#### `extractPathWithinPattern()`

找出通配符`*`、`**`、`?`所在的路径段，并返回该路径段和其所有子路径，例如：

~~~java
PathPattern pattern = PathPatternParser.defaultInstance.parse("/test/*/*.html");
PathContainer path = PathContainer.parsePath("/test/v1/index.html");
PathContainer result = pattern.extractPathWithinPattern(path); // v1/index.html
~~~

#### `combine()`

合并两个模式，合并的逻辑和`AntPathMatcher`一样，例如：

~~~java
PathPattern pattern = PathPatternParser.defaultInstance.parse("/test/*.html");
PathPattern toCombine = PathPatternParser.defaultInstance.parse("/path.*");
PathPattern combine = pattern.combine(toCombine); // /path.html
~~~

#### `compareTo()`

比较模式的通用性，通用性强的`PathPattern`更大，注意，和`AntPathMatcher`不同，`PathPattern`的通用性比较不需要基于某个路径。其通用性按照判断顺序考虑以下因素：

* `null`实例最通用
* 模式包含`/**`和`/{*ver}`这样的全匹配路径段(`CaptureTheRestPathElement`和`WildcardTheRestPathElement`)更通用
  * 如果两个路径都是全匹配路径段，模式长度(`normalizedLength`字段)越短越通用
* 模式中`*`通配符更多的更通用
  * 如果模式中`*`通配符一样多，`?`通配符更多的更通用
* 模式长度(`normalizedLength`字段)越短越通用
* 以上都相等时，直接比较模式代表的字符串字典顺序大小`String.compareTo()`

注意：模式长度(`normalizedLength`字段)计算方式如下：

* 匹配剩余路径段的模式段算长度1，例如`/**`和`/{*var}`
* 模板变量算长度算1,五变量名称无关，例如`{var}`和`{fooo}`长度都为1
* 其他字母长度都算1，例如`/`、`*`、`?`和普通字母

一个示例：

~~~java
String str1 = "/**";
String str2 = "/{*var}";
PathPattern pattern1 = PathPatternParser.defaultInstance.parse(str1);
PathPattern pattern2 = PathPatternParser.defaultInstance.parse(str2);
System.out.println(pattern1.compareTo(pattern2)); // -81
System.out.println(str1.compareTo(str2)); // -81
~~~

### `PathPatternParser`

用于解析字符串创建`PathPattern`,提供如下可配置字段以配置创建的`PathPattern`:

* `caseSensitive`：路径模式匹配是否区分大小写
* `pathOptions`解析URL路径的可选项

并且提供一个全局的静态实例：

~~~java
public static final PathPatternParser defaultInstance;
~~~

通过`parse()`方法创建实例：

~~~java
String patternStr = ....;
PathPattern parse = PathPatternParser.defaultInstance.parse(patternStr);
~~~

# 类型转换

接下来分析Spring的源码，看Spring是在什么地方、如何使用的类型转换。

## `TypeConverter`

`TypeConverter`是Spring 框架中的底层接口，`TypeConverter`统一整合了Spring框架中的各种类型转换API，如`Converter`、`PropertyEditor`、`GenericConverter`等。所以在框架内部可以方便地使用`TypeConverter`完成类型转换。其接口如下：

~~~java
public interface TypeConverter {

	<T> T convertIfNecessary(Object value, Class<T> requiredType) ;
	<T> T convertIfNecessary(Object value, Class<T> requiredType,MethodParameter methodParam);
	<T> T convertIfNecessary(Object value, Class<T> requiredType, Field field);
	<T> T convertIfNecessary(Object value, Class<T> requiredType,TypeDescriptor typeDescriptor) 
}
~~~

它提供了四个重载的`convertIfNecessary()`方法，在实现中前三个重载通常会将`MethodParameter`和`Field`转换为`TypeDescriptor`然后调用第四个方法。

`TypeConverter`的实现通常会继承`TypeConverterSupport`,其主要声明如下：

~~~java
public abstract class TypeConverterSupport extends PropertyEditorRegistrySupport implements TypeConverter {

	
	TypeConverterDelegate typeConverterDelegate;
    //.....三个重载的convertIfNecessary()方法最终会调用下面方法
    
	private <T> T convertIfNecessary(String propertyName, Object value,Class<T> requiredType, TypeDescriptor typeDescriptor) throws TypeMismatchException {
        //....
        return this.typeConverterDelegate.convertIfNecessary(
                propertyName, null, value, requiredType, typeDescriptor);
        //....
	}
}
~~~

可以看到`TypeConverterSupport`提供的类型转换服务最终会委托给内部的`TypeConverterDelegate`代理来完成。

类型转换最终肯定还是要由`ConversionService`和`PropertyEditor`等服务完成。所以`TypeConverterDelegate`作为类型转换的代理就需要一个`PropertyEditorRegistrySupport`，来访问它注册持有的`ConversionService`和`PropertyEditor`来完成最终的类型转换。

而`TypeConverterSupport`同时还继承了`PropertyEditorRegistrySupport`,所以在`TypeConverter`实现中，`TypeConverterDelegate`持有的`PropertyEditorRegistrySupport`通常就是`TypeConverter`本身，例如`SimpleTypeConverter`：

~~~java
public class SimpleTypeConverter extends TypeConverterSupport {

	public SimpleTypeConverter() {
		this.typeConverterDelegate = new TypeConverterDelegate(this);
		registerDefaultEditors();
	}
}
~~~

### TypeConverterDelegate核心逻辑

我们已经知道`TypeConveter`的类型转换是委托给了`TypeConverterDelegate.convertIfNecessary()`来完成的，接下来我们来了解这个方法的大致逻辑：

* 如果持有的`PropertyEditorRegistry`注册了`conversionService`,并且有传参`typeDescriptor`，那么会尝试使用`ConversionService.canConvert()`进行类型转换，如果能够转换，直接返回转换结果，否则进入下一步
* 通过持有的`PropertyEditorRegistry`查找匹配的`PropertyEditor`,如果找到则尝试通过`PropertyEditor`进行类型转换，然后进入下一步
* 如果源类型是可迭代类型(如Map、Collection等)并且`typeDescriptor`有描述迭代元素的目标类型，那么将遍历这些子元素，对每个子元素递归调用`convertIfNecessary()`

## 容器持有的类型转换器

`AbstractBeanFactory`持有如下与类型转换相关的字段：

~~~java
private ConversionService conversionService;
private final Set<PropertyEditorRegistrar> propertyEditorRegistrars = new LinkedHashSet<>(4);
private final Map<Class<?>, Class<? extends PropertyEditor>> customEditors = new HashMap<>(4);
private TypeConverter typeConverter;
~~~

### 转换器来源

我们来分析这些类型转换器的非直接手动配置来源：

* `conversionService`：`ApplicationContext`初始化时会检测容器中ID为`conversionService`的ConversionService Bean,将其注册到BeanFactory

* `propertyEditorRegistrars`
  * 通过`CustomEditorConfigurer`这个`BeanFactoryPostProcessor`在容器初始化时向`BeanFactory`注册自定义的`PropertyEditorRegistrar`
  * `ApplicationContext`在初始化`refresh()`时会注册一个`ResourceEditorRegistrar()`
* `customEditors`:通过`CustomEditorConfigurer`这个`BeanFactoryPostProcessor`在容器初始化时向`BeanFactory`注册`PropertyEditor`
* `typeConverter`:`BeanFactory`通过`getTypeConverter()`方法获取`TypeConverter`，如果持有的`typeConverter`为空，那么这个方法会:
  * 实例化一个`SimpleTypeConverter`
  * 将`BeanFactory`持有的`conversionService`注册到`SimpleTypeConverter`
  * 调用将`BeanFactory`持有的`propertyEditorRegistrars`中的`PropertyEditorRegistrar`的`registerCustomEditors()`方法。将`PropertyEditor`注册到`SimpleTypeConverter`
  * 将`BeanFactory`持有的`customEditors`中的注册`PropertyEditor`到`SimpleTypeConverter`

### 初始化`BeanWrapper`

除了`typeConverter`外这些转换器主要会被注册到`BeanWrapper`中，帮助Spring容器在创建Bean实例后设置实例的字段，这个动作在`AbstractBeanFactory.initBeanWrapper()`方法中，其主要逻辑为：

* 将`BeanFactory`持有的 `conversionService`设置给`BeanWrapper`
* 调用`AbstractBeanFactory.registerCustomEditors()`
  * 调用`propertyEditorRegistrars`中中的`PropertyEditorRegistrar`的`registerCustomEditors()`方法。将`PropertyEditor`注册到`BeanWrapper`
  * `customEditors`将`BeanFactory`持有的`customEditors`中的注册`PropertyEditor`到`BeanWrapper`

而关于`TypeConverter`在`BeanFactory`的`getBean()`的最后，`BeanFactory`会尝试用`TypeConverter`将Bean转换为`requiredType`指定的类型

除此之外，`TypeConverter`会在依赖注入、属性赋值等各个方面发挥类型转换的作用。

## `BeanWrapper`

`BeanWrapper`是Spring Ioc框架中的重要组件，包装了Bean对象，并提供对bean属性的访问和设置。它只有一个实现`BeanWrapperImpl`，其继承体系为:

![BeanWrapperImpl](https://gitee.com/wangziming707/note-pic/raw/master/img/BeanWrapperImpl.png)

可以看到`BeanWrapper`本身实现了`TypeConverter`和`PropertyEditorRegistry`，可以进行类型转换，与其他的`TypeConverter`一样，其类型转换任务是委托给`TypeConverterDelegate`完成的。

除此之外，`BeanWrapper`还实现了`PropertyAccessor`接口，这个接口提供了一种访问JavaBean属性的通用机制，其主要定义如下：

~~~java
public interface PropertyAccessor {

	boolean isReadableProperty(String propertyName);
	boolean isWritableProperty(String propertyName);
	Class<?> getPropertyType(String propertyName) throws BeansException;
	TypeDescriptor getPropertyTypeDescriptor(String propertyName) throws BeansException;
	Object getPropertyValue(String propertyName) throws BeansException;
	void setPropertyValue(String propertyName, @Nullable Object value) throws BeansException;
	void setPropertyValue(PropertyValue pv) throws BeansException;
	void setPropertyValues(Map<?, ?> map) throws BeansException;
	void setPropertyValues(PropertyValues pvs) throws BeansException;
	void setPropertyValues(PropertyValues pvs, boolean ignoreUnknown);
	void setPropertyValues(PropertyValues pvs, boolean ignoreUnknown, boolean ignoreInvalid);
}
~~~

可以看到，它提供了通过`propertyName`访问对象属性和通过`PropertyValue/Map`设置对象属性的一系列方法。

其中`propertyName`可以访问嵌套路径和有序集合：使用`.`来访问嵌套对象属性，使用`[n]`来访问集合属性。

例如对下面对象`People`：

~~~java
@Data
public class People {
    public String name;
    public String age;
    public People father;
    public People mather;
    public List<String> tels;

}
~~~

可以用下面示例访问它的属性：

~~~java
People people = new People();
BeanWrapperImpl beanWrapper = new BeanWrapperImpl(people);
beanWrapper.setPropertyValue("name","张三");
beanWrapper.setPropertyValue("age",14);
beanWrapper.setPropertyValue("father",new People());
beanWrapper.setPropertyValue("father.name","里斯");
beanWrapper.setPropertyValue("tels",new ArrayList<String>());
beanWrapper.setPropertyValue("tels[0]","100086");
beanWrapper.setPropertyValue("tels[1]","11000");

System.out.println(people);
~~~

注意示例中设置的类型就是属性的类型，如果设置类型和属性类型不一致，`BeanWrapper`会使用`TypeConverterDelegate`尝试进行类型转换

## `AbstractNestablePropertyAccessor`

`AbstractNestablePropertyAccessor`是提供属性访问的基础设置类，他为其子类，例如`BeanWrapperImpl`提供属性访问的共用逻辑。

他的核心方法是`setPropertyValue(PropertyTokenHolder,PropertyValue)`和`getPropertyValue(PropertyTokenHolder)`

### `setPropertyValue()`

调用`setPropertyValue()`时，最终会调用下面方法进行类型转换和属性设置：

* 如果要设置的是可迭代对象的键值，则调用方法`AbstractNestablePropertyAccessor.processKeyedProperty()`
* 如果要设置不是可迭代对象的键值，则调用方法`AbstractNestablePropertyAccessor.processLocalProperty()`

我们以方法`AbstractNestablePropertyAccessor.processLocalProperty()`为例，梳理设定属性的逻辑：

* 根据`propertyName`获得要设置的属性对应的`PropertyHandler`，通过`PropertyHandler`我们可以知道对应的属性是否可读可写，是否需要进行类型转换，也可以通过它设置属性值。
* 如果属性不可写，判断它是否可选
  * 若属性是可选的，则记录一条debug日志并返回
  * 否则抛出一个`NotWritablePropertyException`并返回
* 如果需要进行类型转换(通过`PropertyHandler.conversionNecessary`判断)则调用`AbstractNestablePropertyAccessor.convertIfNecessary()`进行类型转换
  * 最终是调用`typeConverterDelegate.convertIfNecessary()`完成类型转换
* 调用`PropertyHandler.setValue()`将要设置的值设置到属性上(主要通过调用属性的`setter`方法)

### `getPropertyValue()`

`getPropertyValue()`方法的主要逻辑如下：

* 根据`propertyName`获得要设置的属性对应的`PropertyHandler`
* 如果属性不可读，抛出一个`NotReadablePropertyException`异常并返回
* 调用`PropertyHandler.getValue()`获取属性值
* 如果访问的是属性的键值，会根据属性值的类型(例如`Map`、`List`、`Array`等)获取键值

## `DataBinder`

`DataBinder`是Spring框架中提供对象数据绑定和校验的基础设置类，常用于网络请求中的对象绑定，如`WebDatBinder`

使用它进行数据绑定和校验的基本代码如下：

~~~java
Foo target = new Foo();
DataBinder binder = new DataBinder(target);
binder.setValidator(new FooValidator());
// bind to the target object
binder.bind(propertyValues);
// validate the target object
binder.validate();
// get BindingResult that includes any validation errors
BindingResult results = binder.getBindingResult();
~~~

以上演示是通过`setter`方法绑定属性，它也提供通过构造器注入属性发方式。

`DataBinder`持有以下重要属性：

~~~java
private Object target; //进行数据绑定的目标对象
private AbstractPropertyBindingResult bindingResult; //绑定结果对象
private ConversionService conversionService;
private ExtendedTypeConverter typeConverter; 
private final List<Validator> validators = new ArrayList<>();
~~~

其核心方法是`dobind(MutablePropertyValues)`,方法主要逻辑如下：

* 调用`checkAllowedFields()`过滤不允许绑定的字段
* 调用`checkRequiredFields()`检查必填字段，如果有必填字段为空，会向`BindingResult`注册一个错误
* 调用`applyPropertyValues()`进行数据绑定：
  * 调用`getPropertyAccessor()`获取一个用于访问对象属性的`ConfigurablePropertyAccessor`
    * 该方法会先调用`getInternalBindingResult()`初始化(如果没有)一个`AbstractPropertyBindingResult`赋值到`this.bindingResult`
    * 然后调用`bindingResult.getPropertyAccessor()`方法获取target的属性访问器
  * 调用`ConfigurablePropertyAccessor.setPropertyValues()`方法绑定属性值

# 数据绑定

