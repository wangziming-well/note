# 路径匹配

Spring 提供的进行路径匹配的Util工具类：

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

## TypeConverterDelegate核心逻辑

我们已经知道`TypeConveter`的类型转换是委托给了`TypeConverterDelegate.convertIfNecessary()`来完成的，接下来我们来了解这个方法的大致逻辑：

* 如果持有的`PropertyEditorRegistry`注册了`conversionService`,并且有传参`typeDescriptor`，那么会尝试使用`ConversionService.canConvert()`进行类型转换，如果能够转换，直接返回转换结果，否则进入下一步
* 通过持有的`PropertyEditorRegistry`查找匹配的`PropertyEditor`,如果找到则尝试通过`PropertyEditor`进行类型转换，然后进入下一步
* 如果源类型是可迭代类型(如`Map`、`Collection`等)并且`typeDescriptor`有描述迭代元素的目标类型，那么将遍历这些子元素，对每个子元素递归调用`convertIfNecessary()`

# 属性访问

`PropertyAccessor`接口，这个接口提供了一种访问JavaBean属性的通用机制，其主要定义如下：

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

其中`propertyName`可以访问嵌套路径和有序集合：使用`.`来访问嵌套对象属性，使用`[n]`来访问集合属性。例如对下面对象`People`：

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

其继承体系如下：

![PropertyAccessor](https://gitee.com/wangziming707/note-pic/raw/master/img/PropertyAccessor.png)

* `PropertyAccessor`的实现类都会继承`AbstractNestablePropertyAccessor`，它集成了`TypeConverterSupport`如果设置类型和属性类型不一致，就会使用`TypeConverterSupport`提供的类型转换服务尝试进行类型转换
* 属性访问器体系只有两个实现类：
  * `DirecFieldAccessor`：同通反射直接访问属性
  * `BeanWrapperImpl`:通过`setter/getter`方法访问属性

## `AbstractNestablePropertyAccessor`

`AbstractNestablePropertyAccessor`是提供属性访问的基础设置类，他为其子类，例如`BeanWrapperImpl`提供属性访问的共用逻辑。

他的核心方法是`setPropertyValue(PropertyTokenHolder,PropertyValue)`和`getPropertyValue(PropertyTokenHolder)`。

并且它有抽象内部类`PropertyHandler`,提供属性处理器的抽象。通过`getLocalPropertyHandler()`获取属性对应的处理器。

### `PropertyHandler`

`PropertyHandler`用于处理对应的属性，子类需要实现该内部类以提供不同的属性处理策略，其主要方法为：

~~~java
public Class<?> getPropertyType(); //获取属性类型
public boolean isReadable(); //属性是否可写
public boolean isWritable(); //属性是否可读
public abstract Object getValue() //获取属性值
public abstract void setValue(Object value) //设置属性值
~~~

### `setPropertyValue()`

调用`setPropertyValue()`时，最终会调用下面方法进行类型转换和属性设置：

* 如果要设置的是可迭代对象的键值，则调用方法`AbstractNestablePropertyAccessor.processKeyedProperty()`
* 如果要设置不是可迭代对象的键值，则调用方法`AbstractNestablePropertyAccessor.processLocalProperty()`

我们以方法`AbstractNestablePropertyAccessor.processLocalProperty()`为例，梳理设定属性的逻辑：

* 根据`propertyName`调用`getLocalPropertyHandler()`方法获得要设置的属性对应的`PropertyHandler`
* 如果属性不可写(通过`PropertyHandler.isWritable()`)，判断它是否可选
  * 若属性是可选的，则记录一条debug日志并返回
  * 否则抛出一个`NotWritablePropertyException`并返回
* 如果需要进行类型转换(通过`PropertyHandler.conversionNecessary`判断)则调用`AbstractNestablePropertyAccessor.convertIfNecessary()`进行类型转换
* 调用`PropertyHandler.setValue()`将要设置的值设置到属性上

### `getPropertyValue()`

`getPropertyValue()`方法的主要逻辑如下：

* 根据`propertyName`获得要设置的属性对应的`PropertyHandler`
* 如果属性不可读，抛出一个`NotReadablePropertyException`异常并返回
* 调用`PropertyHandler.getValue()`获取属性值
* 如果访问的是属性的键值，会根据属性值的类型(例如`Map`、`List`、`Array`等)获取键值

## `BeanWrapper`

`BeanWrapper`是Spring Ioc框架中的重要组件，它是一个属性访问器，并且包装了Bean对象，提供对bean属性的访问和设置。它只有一个实现`BeanWrapperImpl`

`BeanWrapperImpl`在其`PropertyHandler`的实现`BeanPropertyHandler`使用`setter/getter`方法访问属性

## `DirectFieldAccessor`

`BeanWrapperImpl`在其`PropertyHandler`的实现`FieldPropertyHandler`使用反射直接访问属性

# 对象验证

Spring提供完整的对象验证服务，`org.springframework.validation.Validator`是Spring 校验器的顶级接口。`org.springframework.validation.Errors`代表校验失败的错误信息。

对象验证不通过的原因有两类，分别对应下面两个类：

* `ObjectError`：代表拒绝对象的全局错误原因。
* `FieldError`：代表拒绝对象的字段原因，即特定字段值校验不通过原因。

## `Errors`

`Errors`封装校验的错误信息，因为Spring支持嵌套对象校验，所以`Errors`也支持嵌套路径，它提供下面接口：

嵌套路径支持：

~~~java
String getObjectName(); //获取根对象的名称
default void setNestedPath(String nestedPath); //设置嵌套路径
default String getNestedPath(); //获取当前嵌套路径
default void pushNestedPath(String subPath); //在当前路径的基础上添加子路径
default void popNestedPath(); //回到上一个路径
~~~

错误信息注册：

~~~java
default void reject(String errorCode);
default void reject(String errorCode, String defaultMessage);
void reject(String errorCode, Object[] errorArgs, @Nullable String defaultMessage);
//注册一个全局错误ObjectError
default void rejectValue(String field, String errorCode);
default void rejectValue(String field, String errorCode, String defaultMessage);
void rejectValue(String field, String errorCode,Object[] errorArgs,String defaultMessage);
//注册一个字段错误FieldError
default void addAllErrors(Errors errors);
//合并errors到当前错误中
~~~

访问错误信息：

~~~java
default <T extends Throwable> void failOnError(Function<String, T> messageToException);
//根据messageToException抛出异常
default boolean hasErrors();
default int getErrorCount();
default List<ObjectError> getAllErrors();
//访问所有错误
default boolean hasGlobalErrors();
default int getGlobalErrorCount();
List<ObjectError> getGlobalErrors();
default ObjectError getGlobalError();
//访问全局错误
default boolean hasFieldErrors();
default int getFieldErrorCount();
List<FieldError> getFieldErrors();
default FieldError getFieldError();
default boolean hasFieldErrors(String field);
default int getFieldErrorCount(String field);
default List<FieldError> getFieldErrors(String field);
default FieldError getFieldError(String field);
//访问字段错误
~~~

访问字段：

~~~java
Object getFieldValue(String field);
default Class<?> getFieldType(String field);
~~~

`AbstractErrors`是`Errors`的抽象基类，为子类实现提供嵌套路径的实现与 全局错误和字段错误的管理

## `Validator`

`Validator`是对象验证的顶级接口，它的主要方法如下：

~~~java
boolean supports(Class<?> clazz); //判断当前验证器是否支持指定的类型
void validate(Object target, Errors errors); //对target进行验证，验证结果注册到给定的errors
default Errors validateObject(Object target); //对target进行验证，返回Errors验证结果
~~~









# 数据绑定

`DataBinder`是Spring框架中提供对象数据绑定和校验的基础设置类，常用于网络请求中的对象绑定，如`WebDatBinder`.提供对象属性的设置,允许类型转换，也提供校验。通过`PropertyValues`提供的属性信息对目标类进行绑定，绑定结果抽象为`BindingResult`

## `BindingResult`

`BindingResult`接口表示为`DataBinder`的绑定结果，可以通过` DataBinder.getBindingResult() `方法获取。它拓展了`Errors`接口，所以可以提供错误信息的注册。它的定义如下：

~~~java
public interface BindingResult extends Errors {
	Object getTarget(); //返回包装的目标对象
	Map<String, Object> getModel(); //返回一个状态模型Map，暴露当前的BindingResult实例和target实例
	Object getRawFieldValue(String field); //返回指定字段的字段值
	PropertyEditor findEditor(String field, @Nullable Class<?> valueType); //根据指定域和类型获取属性编辑器
	PropertyEditorRegistry getPropertyEditorRegistry(); //返回底层的属性编辑器注册表
	String[] resolveMessageCodes(String errorCode); // 处理错误码
	String[] resolveMessageCodes(String errorCode, String field); // 处理错误码
	void addError(ObjectError error); //添加错误到错误列表，这个方法通常由 BindingErrorProcessor使用
	default void recordFieldValue(String field, Class<?> type, @Nullable Object value);
    //记录指定字段难道值，在无法构建目标对象实例时使用
	default void recordSuppressedField(String field);
    //将指定的不允许字段标记为 suppressed隐藏。DataBinder会为检测到的每个字段值调用此函数，以定位不允许的字段。
	default String[] getSuppressedFields();
    //返回在绑定过程中被隐藏的字段列表。
}
~~~

其继承体系中主要的类有：

![BindingResult](https://gitee.com/wangziming707/note-pic/raw/master/img/BindingResult.png)

### `AbstractBindingResult`

`AbstractBindingResult`是`BindingResult`的抽象基类，提供了对 `ObjectErrors `和 `FieldErrors `的通用管理

它持有`ObjectError`列表字段：

~~~java
private final List<ObjectError> errors = new ArrayList<>();
~~~

可以通过对应的`add/addAll`和`get/getAll`方法设置和访问错误信息列表。

### `AbstractPropertyBindingResult`

`AbstractPropertyBindingResult`是`AbstractBindingResult`的子类，在此基础上提供了通过`PropertyAccessor`进行字段访问的预实现

其`getPropertyAccessor()`获取底层的对`target`对象的属性访问器。

### `BeanPropertyBindingResult`

`BeanPropertyBindingResult`使用`BeanWrapper`作为目标对象的属性访问器，其`getPropertyAccessor()`方法实际返回一个`BenBeanWrapper`

### `DirectFieldBindingResult`

`DirectFieldBindingResult`使用`DirectFieldAccessor`作为目标对象的属性访问器，其`getPropertyAccessor()`方法实际返回一个`DirectFieldAccessor`

## `DataBinder`

`DataBinder`允许通过构造函数和 setter 方法将属性值绑定到目标对象，还支持验证和绑定结果分析。可以通过指定允许的字段模式、必填字段、自定义编辑器等来自定义绑定过程。

`DataBinder`的`bind()`方法实现对象字段的绑定，`validate()`对对象进行校验。`getBindingResult()`方法获取一个代表绑定结果的`BindingResult`实例。

`DataBinder`持有以下重要属性：

~~~java
private Object target; //进行数据绑定的目标对象实例
ResolvableType targetType; //目标对象的类型，当target为null时，可以用setTargetType方法设置该值，以用类型构造器创建一个target实例
private final String objectName; //目标类型的名称，如果不指定，默认为target
private AbstractPropertyBindingResult bindingResult; //表示绑定结果
private ExtendedTypeConverter typeConverter; //内部的类型转换器，为DataBinder对外提供类型转换功能
private String[] allowedFields; //允许绑定的属性名，支持*通配符
private String[] disallowedFields; //不允许绑定的属性名，支持*通配符
private String[] requiredFields; //必填的属性名，支持*通配符
private NameResolver nameResolver; //使用构造器创建target实例时，使用NameResolver根据构造器的参数类型解析一个name，后续会通过该name解析一个value值来作为构造器的入参
private ConversionService conversionService; //用于转换属性值类型的ConversionService
private MessageCodesResolver messageCodesResolver; //用于处理错误信息
private BindingErrorProcessor bindingErrorProcessor = new DefaultBindingErrorProcessor(); //绑定错误的处理策略，默认为DefaultBindingErrorProcessor
private final List<Validator> validators = new ArrayList<>(); //在绑定过程中进行校验的Validator，可以通过对应的setter方法和add方法设置
private Predicate<Validator> excludedValidators; //用于过滤validators的谓词，可以通过setter方法设置
~~~

### `getBindingResult()`

`this.bindingResult`是一个`AbstractPropertyBindingResult`,创建时会与`target`目标对象绑定，可以通过`this.bindingResult`获取目标对象的属性访问器`PropertyAccessor`,以访问目标对象属性

`this.bindingResult`是懒加载的,在`DataBinder`内部，通过`getBindingResult()`或者`getInternalBindingResult()`来获取`this.bindingResult`，第一次调用该方法时会创建一个`AbstractPropertyBindingResult`实例，并赋值该字段：

* 如果`this.directFieldAccess`为`true`，创建一个`DirectFieldBindingResult`
* 否则，创建一个`BeanPropertyBindingResult`

### `getPropertyAccessor()`

`getPropertyAccessor()`方法用于获取目标对象对应的属性访问器，获取方式是通过`this.bindingResult`获取，定义如下：

~~~java
protected ConfigurablePropertyAccessor getPropertyAccessor() {
    return getInternalBindingResult().getPropertyAccessor();
}
~~~

### `bind()`

先大致了解如下类/接口：

* `PropertyValue`:代表属性值，封装了属性名和值
* `PropertyValues`:`PropertyValue`的容器，通常使用它的实现`MutablePropertyValues`

`bind()`方法进行数据绑定。其主要逻辑为：

* 将传入的`PropertyValues`转换为`MutablePropertyValues`
* 校验允许字段，遍历`MutablePropertyValues`中的`PropertyValue`
  * 判断属性名是否允许绑定，满足下面两个条件，属性名才允许绑定：
    * `this.allowedFields`字段为空，或者属性名与`this.allowedFields`中的模式匹配
    * `this.disallowedFields`字段为空，或者属性名与`this.disallowedFields`中的模式不匹配
  * 如果当前属性名不允许绑定：
    * 将当前`PropertyValue`从`MutablePropertyValues`中移除
    * 调用`getBindingResult().recordSuppressedField()`记录移除的字段名

* 校验必填字段，如果`this.requiredFields`不为空，遍历`this.requiredFields`：遍历项为`field`
  * 判断当前`field`字段是否为空，
    * 获取`MutablePropertyValues`中属性名为`field`的`PropertyValue`，如果`PropertyValue`为`null`则当前字段为空
    * 如果`PropertyValue`的`value`值为`null`，则当前字段为空
    * 如果当前字段类型为`String`,若为空字符串，则当前字段为空
    * 若当前字段类型为`String[]`,若数组长度为0，或者第一个数组值为`null`或者空字符串，则当前字段为空
  * 如果当前`field`字段为空：调用`this.bindingErrorProcessor.processMissingFieldError()`.将错误信息注册到`this.bindingResult`中
* 应用属性值到目标对象中
  * 调用`getPropertyAccessor().setPropertyValues()`将`MutablePropertyValues`代表的属性值设置到对象中
  * 上一步如果有抛出`PropertyBatchUpdateException`异常，则调用`this.bindingErrorProcessor.processPropertyAccessException()`注册该异常到`this.bindingResult`中

### `validate()`

`validate()`对绑定好的对象进行校验，其主要逻辑为：



