# 注解式`Controller`的处理器

`org.springframework.web.method.HandlerMathod`是注解式`Controller`中处理器的抽象，对应一个`@Controller`类的`@RequestMapping`方法

`HandlerMethod`处理器方法一定有注解(方法体和方法参数上),并且框架需要根据这些注解声明来控制处理器行为，所以`HandlerMethod`很自然地继承了`org.springframework.core.annotation.AnnotatedMethod`，它是一个注解方法的抽象。

和方法紧密相关的一个概念就是方法的参数和返回值。`org.springframework.core.MethodParameter`就是对方法参数和返回值的抽象

## `MethodParameter`

`MethodParameter`是Spring对方法参数的抽象，它封装了方法参数的各种信息：如参数类型、参数名、参数位置、参数注解等

它有如下重要字段：

~~~java
private final Executable executable; //参数所在的方法(普通方法/构造器)
private final int parameterIndex; //参数在方法中的位置: -1表示返回值，0表示第一个入参
private int nestingLevel; //嵌套层数
Map<Integer, Integer> typeIndexesPerLevel; //嵌套索引映射
private volatile ParameterNameDiscoverer parameterNameDiscoverer; // 获取参数名parameterName需要用到该字段
private volatile Class<?> containingClass; 
//下面字段使用懒加载，在第一次调用对应的getter方法时初始化字段值。
private volatile Parameter parameter; //当前方法参数对应的 Parameter 实例。
private volatile Class<?> parameterType; //参数类型
private volatile Type genericParameterType; //参数类型
private volatile Annotation[] parameterAnnotations; //参数上的注解
volatile String parameterName; //参数名
private volatile MethodParameter nestedMethodParameter; // 嵌套方法参数
~~~

可以用对应的`getter`方法访问这些字段。

### 嵌套方法参数

可以通过`nested()`方法获得一个新的嵌套的`MethodParameter`，该方法参数与原方法参数指向同一个参数，但是嵌套层数更深。可以通过嵌套的`MethodParameter`访问方法参数的泛型类型。

例如对于下面类：

~~~java
@Controller
public class Demo {

    @RequestMapping("/te?t")
    @ResponseBody
    public String get(@RequestBody Map<String,User> user ){
        //....
    }
}
~~~

可以通过下面代码访问其方法参数和参数的泛型：

~~~java
Method get = Demo.class.getDeclaredMethod("get", Map.class);
MethodParameter methodParameter = new MethodParameter(get, 0);
MethodParameter nested1 = methodParameter.nested(0);
MethodParameter nested2 = methodParameter.nested(1);
System.out.println(methodParameter.getNestedParameterType()); //interface java.util.Map
System.out.println(nested1.getNestedParameterType()); //class java.lang.String
System.out.println(nested2.getNestedParameterType()); //class com.example.springboot.controller.User
~~~

## `AnnotatedMethod`

`AnnotatedMethod`是对注解方法的抽象，它封装了方法、方法参数、方法注解信息，可以通过它提供的如下方法访问：

~~~java
public final Method getMethod();//获取当前方法对应的Method实例
protected final Method getBridgedMethod(); // 获取当前方法的桥方法的Method实例(在继承重写方法时，如果重写的方法返回值与父类方法不同，那么编译器会自动生成一个桥方法)
public final MethodParameter[] getMethodParameters(); // 获取入参的方法参数列表
public MethodParameter getReturnType(); //获取返回值对应的方法参数
public boolean isVoid(); //返回值是否是 void
public <A extends Annotation> A getMethodAnnotation(Class<A> annotationType); //获取方法注解
public <A extends Annotation> boolean hasMethodAnnotation(Class<A> annotationType); //判断方法上是否存在指定注解
~~~

## `HandlerMethod`

创建一个`HandlerMethod`至少需要一个`@RequestMapping`方法对应的`Method`实例，和`@RequestMapping`方法所在类的Bean实例，即如下构造器：

~~~java
public HandlerMethod(Object bean, Method method);
~~~

`HandlerMethod`的如下字段和`bean`有关：

~~~java
private final Object bean;   //@RequestMapping方法所在类的实例
private final Class<?> beanType; //@RequestMapping方法所在类的类型
~~~

可以通过对应的`getter`方法访问它们。

### 解析`bean`

也可以通过提供一个`beanName`和`BeanFactory`来指定`@RequestMapping`所在类的实例：

~~~java
public HandlerMethod(String beanName, BeanFactory beanFactory, Method method);
~~~

此时`beanName`字符串会被绑定到`this.bean`上，`beanFactory`会绑定到`this.beanFactory`上

可以看到通过这个构造器创建的实例，其`this.bean`字段上绑定的不是真正的`bean`实例。而是一个`beanName`字符串。

所以它不能对外提供真正的`bean`实例，需要解析当前的`bean`。

此时可以通过`createWithResolvedBean()`方法解析`this.bean`字段，返回一个新的`HandlerMethod`，其逻辑如下：

* 如果当前实例的`this.bean`字符类型是字符串，那么调用`this.beanFactory.getBean(beanName)`获取`handler`实例，否则直接使用`this.bean`作为`handler`
* 创建一个新的`HandlerMethod`实例并返回，新实例的`this.resolvedFromHandlerMethod`字段为当前`HandlerMethod`实例

所以如果一个`HandlerMethod`是通过解析产生的，那么它的`this.resolvedFromHandlerMethod`字段就不为空，并且指向其源`HandlerMethod`

### 处理是否参数校验

`HandlerMethod`提供了两个方法，告诉客户端当前处理器方法是否需要进行参数校验：

~~~java
public boolean shouldValidateArguments();  //需要校验方法入参
public boolean shouldValidateReturnValue(); //需要检验方法返回值
~~~

这对应`HandlerMethod`的如下字段：

~~~java
private final boolean validateArguments;
private final boolean validateReturnValue;
~~~

直接使用构造器创建的`HandlerMethod`的，这两个字段的值都默认为`false`,也就是说默认情况下，`HandlerMethod`不需要进行参数校验。

可以调用`createWithValidateFlags()`创建一个新的，允许参数校验的`HandlerMethod`实例，该主要逻辑如下：

* 创建一个新的`HandlerMethod`
* 新的`HandlerMethod`的`this.resolvedFromHandlerMethod`为当前`HandlerMethod`
* 新的`HandlerMethod`的`this.validateArguments`值为方法`MethodValidationInitializer.checkArguments()`返回值，该方法：
  * 如果当前项目中不存在`jakarta.validation.Validator`类，说明无法进行校验，返回`false`
  * 如果`HandlerMethod`所在的类上有`Validated`注释，那么方法验证由AOP代理进行，返回`false`
  * 遍历方法的参数，如果参数满足下面条件之一，说明需要进行参数验证，返回`true`,否则返回`false`
    * 参数上有`jakarta.validation.Constraint`注解
    * 参数上有`jakarta.validation.Valid`注解，并且当前参数类型是基于键值或者索引的容器(参数类型为`List`,`Map`或者数组)
    * 参数类型是容器类型，如果容器中元素的类型有注解`@Constraint`或者`@Valid`
* 新的`HandlerMethod`的`this.validateReturnValue`值为方法`MethodValidationInitializer.checkReturnValue()`返回值，该方法：
  * 如果当前项目中不存在`jakarta.validation.Validator`类，说明无法进行校验，返回`false`
  * 如果`HandlerMethod`所在的类上有`Validated`注释，那么方法验证由AOP代理进行，返回`false`
  * 如果返回值有注解`@Constraint`或者`@Valid`，返回`true`,否则返回`false`

### 处理`ResponseStatus`

如果当前处理器方法上有`@ResponseStatus`注释，那么：

* `this.responseStatus`字段会设置为`@ResponseStatus`的`code`值
* `this.responseStatusReason`会设置为`@ResponseStatus`的`reason`值
  * 如果`this.messageSource`不为空，那么`reason`会进行国际化处理

可以通过对应的`getter`方法方法这两个字段。

# 注解式`Controller`的`HandlerMapping`

`RequestMappingHandlerMapping`负责维护`HandlerMathod`和请求之间的映射关系。根据继承关系我们知道它有以下主要父类：

* `AbstractHandlerMethodMapping`
* `RequestMappingInfoHandlerMapping`

我们从父类开始依次分析：

## `AbstractHandlerMethodMapping<T>`

`AbstractHandlerMethodMapping`是定义请求和`HandlerMethod`之间的映射的抽象基类。

它的泛型`T`类型表示`HandlerMethod`的`mapping`映射信息，`T`包含请求与`HandlerMethod`的进行匹配时需要的条件信息。如映射到某个`HandlerMethod`需要的请求的路径/类型/请求头 等信息。每个`HandlerMethod`都需要绑定一个`T`用于请求映射。

`AbstractHandlerMethodMapping`定义下面内部类来辅助它提供处理器映射器的相关功能：

* `MappingRegistration`:一个`T`映射的注册项，主要包含一对`HandlerMethod`和对应的`T`映射实例。除此之外还有如下属性
  * `mappingName`：根据`HandlerMethodMappingNamingStrategy`生成的name
  * `directPaths`：映射T的直接路径集合
  * `corsConfig`:当前处理器是否启用了cors
* `Match`：对`T`映射和对应的`HandlerMathod`的浅包装，主要为了通过比较器在请求上下文中找到最佳匹配。
* `MatchComparator`:可以用客户端提供的比较器对`Match`进行比较

* `MappingRegistry`:一个`T`映射的注册表，维护`HandlerMethod`的所有映射关系。并提供给相应的查找方法。

### `MappingRegistry`

它维护以下`Map`:

~~~java
private final Map<T, MappingRegistration<T>> registry = new HashMap<>(); 
// T 到 MappingRegistration的映射
private final MultiValueMap<String, T> pathLookup = new LinkedMultiValueMap<>();
//路径字符串到List<T>的映射 因为一个path可能匹配多个处理器
private final Map<String, List<HandlerMethod>> nameLookup = new ConcurrentHashMap<>();
//name到List<HandlerMethod>的映射。因为name
private final Map<HandlerMethod, CorsConfiguration> corsLookup = new ConcurrentHashMap<>();
//HandlerMethod到CorsConfiguration的映射
~~~

其核心方法是`MappingRegistry.register()`，其方法签名如下：

~~~java
public void register(T mapping, Object handler, Method method);
~~~

主要逻辑为：（如果没有指定，方法默认为`AbstractHandlerMethodMapping`定义的方法）

* 根据提供的`handler`和`method`，生成一个`HandlerMethod`实例
* 检查是否已经注册，如果`this.registry`中已经存在了同样的`mapping`和`HandlerMethod`对，抛出一个`IllegalStateException`异常
* 调用`HandlerMethod.createWithValidateFlags()`启用其方法校验
* 调用`getDirectPaths()`方法(提供了一个默认的实现，但子类会重写该方法)通过`T`获取其直接路径(非模式)集合，遍历该集合，将`path`和`mapping`对放入`this.pathLookup`
* 调用`HandlerMethodMappingNamingStrategy.getName()`方法，根据`HandlerMethod`和`mapping`解析一个`name`,添加到`this.nameLookup`字段
* 调用**`initCorsConfiguration()`方法**(方法为空，由子类具体实现)生成一个`CorsConfiguration`实例，如果该实例不为`null`,将`HandlerMethod`和`CorsConfiguration`对添加到`this.corsLookup`
* 最后，创建一个`MappingRegistration`实例，将`mapping`实例和`MappingRegistration`实例对添加到`this.registry`

在注册完映射关系后，可以通过`MappingRegistry`提供的方法查找相关映射和处理器：

~~~java
Map<T, MappingRegistration<T>> getRegistrations(); //返回this.registry
List<T> getMappingsByDirectPath(String urlPath); //根据this.pathLookup查找
List<HandlerMethod> getHandlerMethodsByMappingName(String mappingName); //根据this.nameLookup查找
CorsConfiguration getCorsConfiguration(HandlerMethod handlerMethod); // 根据this.corsLooku查找
~~~

### 注册`HandlerMethod`

`AbstractHandlerMethodMapping`维护一个`MappingRegistry`字段：

~~~java
private final MappingRegistry mappingRegistry = new MappingRegistry();
~~~

在提供服务前，`AbstractHandlerMethodMapping`需要将可用的`HandlerMethod`注册到`this.mappingRegistry`中。

`AbstractHandlerMethodMapping`是通过`InitializingBean`接口提供的回调实现`HandlerMethod`的注册的。在`AbstractHandlerMethodMapping` Bean实例化完毕后，会调用`initHandlerMethods()`初始化并注册`HandlerMethod`.

`initHandlerMethods()`方法主要逻辑如下：

*  获取容器中所有Bean的`beanName`,判断**可配置字段`this.detectHandlerMethodsInAncestorContexts`**:
   * 如果该字段为`false`(默认值):仅获取当前容器中的`beanName`
   * 如果该字段为`true`:获取当前容器以及所有祖先容器中的`beanName`

*  接下来遍历获取到的所有`beanName`
*  获取`beanName`对应的`beanType`
*  调用`isHandler()`方法判断当前Bean是否是处理器
   * 该方法是抽象方法，在子类的实现中，当前`beanType`类型上有`@Controller`注解的会返回`true`
*  如果当前`Bean`是处理器，调用`detectHandlerMethods()`方法进行检测：
   * 根据`beanName`从容器中获取`handleType`处理器类型
   * 因为CGLIB代理，获得的`handleType`可能是CGLIB生成的子类，所以调用`ClassUtils.getUserClass()`方法获取其原始类`userType`;`userType`就是处理器类型
   * 遍历`userType`所有的用户定义方法`Method`
     * 调用**`getMappingForMethod()`**方法(抽象方法，具体由子类实现)获取`Method`对应的`T`映射。
     * 根据`Method`和处理器`bean`实例和`T`映射调用`this.mappingRegistry.register()`方法将其注册到`MappingRegistry`中

### `getHandlerInternal()`

`AbstractHandlerMethodMapping`实现了`AbstractHandlerMapping.getHadnlerInternel()`方法，可以根据`HttpServletRequest`请求，从`this.mappingRegistry`中找到匹配`HandlerMethod`。

`AbstractHandlerMethodMapping.getHadnlerInternel()`方法的主要逻辑为：(如果没有指定，下面提到的方法默认为`AbstractHandlerMethodMapping`声明)

* 调用`AbstractHandlerMapping.initLookupPath()`方法，获取请求在app上下文中的相对路径`lookupPath`
* 创建一个`Matche`列表`matches`，用以保存所有匹配的`HandlerMethod`
* 根据直接路径(非模式)查找匹配的映射：
  * 调用`this.mappingRegistry.getMappingsByDirectPath()`根据路径查找匹配的`T`映射列表
  * 调用**`getMatchingMapping()`方法**(抽象方法，需要子类实现)，根据`T`映射信息判断请求`HttpServletRequest`是否匹配
  * 如果匹配，创建一个`Match`实例放入`matches`中
* 如果上一步没有匹配的映射(当前`matches`为空)，那么直接遍历所有的注册项：
  * 调用`this.mappingRegistry.getRegistrations().keySet()`获取所有的`T`映射
  * 调用**`getMatchingMapping()`方法**(抽象方法，需要子类实现)，根据`T`映射信息判断请求`HttpServletRequest`是否匹配
  * 如果匹配，创建一个`Match`实例放入`matches`中
* 如果此时`matches`仍然为空，则说明没有匹配的的处理器，**调用`handleNoMatch()`方法**(抽象方法，需要子类实现)进行匹配失败的处理(通常会抛出一些`ServletException`异常)并返回`null`结束方法。
* 如果`matches`长度为1，说明只有一个匹配的`bestMatch`
* 如果`matches`长度不为1，需要找到一个最佳匹配`bestMatch`
  * 调用**`getMappingComparator()`方法**(抽象方法，需要子类实现)获取一个`Matcher`的比较器。
  * 根据上一步创建比较器对`matches`排序，找到最佳的匹配`bestMatch`
  * 如果当前请求是`cors`的预检请求，判断`matchs`中是否有处理器进行了cors配置，如果有，方法直接返回一个用于预检请求的模糊匹配处理器(一个空的处理器)
  * 如果当前的请求不是`cors`预检请求,那么检测匹配优先级：
    * 获取第二匹配的`Match`,`secondBestMatch`
    * 如果`bestMatch`和`secondBestMatch`在`Matcher`比较器下的优先级相同，说明有两个最佳匹配，抛出一个`IllegalStateException`异常
* 向requset域中添加属性:属性名为`org.springframework.web.servlet.HandlerMapping.bestMatchingHandler`,属性值为最佳匹配的`HandlerMethod`实例
* 调用**方法`handleMatch()`**进行找到匹配后的处理
  * 子类会重写该方法
* 返回`bestMatch`中的`HandlerMethod`实例

## `RequestCondition<T>`

`RequestCondition`是请求条件的抽象接口。用于判断`HttpServletRequest`请求是否符合给定的条件。

其中泛型`T`表示条件的类型。(通常就是实现类本身)

接口定义如下:

~~~java
public interface RequestCondition<T> {

	T combine(T other); //合并两个条件
	@Nullable
	T getMatchingCondition(HttpServletRequest request); //判断请求是否符合条件
	int compareTo(T other, HttpServletRequest request); //就同一请求比较两个条件，在一个请求符合多个条件时使用该方法选出一个最佳的条件
}

~~~

它有如下实现：

| 类名                             | 描述                                                         | 多个条件关系 | 对应的`@RequestMapping`字段 |
| -------------------------------- | ------------------------------------------------------------ | ------------ | --------------------------- |
| `AbstractRequestCondition`:      | 抽象基类                                                     |              |                             |
|                                  |                                                              |              |                             |
| `CompositeRequestCondition`      | 复合请求条件，内部持有多种请求条件(多个`RequestConditionHolder`) | “且”(`&&`)   |                             |
| `RequestMappingInfo`             | 请求映射信息，内部持有下面8类请求条件的实例。`RequestMappingInfo`与请求匹配当前仅当持有的8个请求条件全部匹配 |              |                             |
| `RequestConditionHolder`         | 请求条件的持有者，内部持有一个`RequestCondition`，允许在请求条件的类型未知请求下处理请求条件。 |              |                             |
| `ConsumesRequestCondition`       | 对请求头的`Content-Type`字段的请求条件                       | ”或“(`||`)   | `consumes`或`headers`       |
| `ProducesRequestCondition`       | 对请求期望响应体的`MediaType`的请求条件。默认请求下通过请求头的`Accept`字段决定。 | ”或“(`||`)   | `produces`或`headers`       |
| `HeadersRequestCondition`        | 对请求头的请求条件                                           | ”且“(`&&`)   | `headers`                   |
| `ParamsRequestCondition`         | 对请求参数的请求条件                                         | ”且“(`&&`)   | `params`                    |
| `PathPatternsRequestCondition`   | 对请求路径与一组模式是否匹配的请求条件,使用` PathPattern`进行路径模式匹配 | ”或“(`||`)   | `path/value`                |
| `PatternsRequestCondition`       | 对请求路径与一组模式是否匹配的请求条件,使用`AntPathMatcher`进行路径模式匹配 | ”或“(`||`)   | `path/value`                |
| `RequestMethodsRequestCondition` | 对请求方法的请求条件，使用`RequestMethod`枚举的方法          | ”或“(`||`)   | `method`                    |

下面详解了解其中几个重要的实现

### `AbstractRequestCondition`

`AbstractRequestCondition`是`RequestCondition`的抽象基类。几乎所有的`RequestCondition`实现都会继承这个抽象基类。

`AbstractRequestCondition`在`RequestCondition`接口的基础上拓展了提供了`Content`的概念，`Content`表示请求条件的离散项的集合。例如请求条件对HTTP请求方法有限制，那么`Content`可能就为`[POST,GET]`.

它提供下面方法以访问`content`:

~~~java
public boolean isEmpty(); //返回content是否为空
protected abstract Collection<?> getContent();
protected abstract String getToStringInfix(); //在打印离散项集合时离散项之间的间隔符，例如对于HTTP请求方法，使用||作为间隔符
~~~

同时`AbstractRequestCondition`还根据`content`重写了`Object`的`equals()`、`hashCode()`、`toString()`方法

### `PathPatternsRequestCondition`

`PathPatternsRequestCondition`它内部维护一个`patterns`字段：

~~~java
private final SortedSet<PathPattern> patterns;
~~~

因为是`SortedSet`，所以`PathPattern`容器中已经根据其`Comparable`接口定义的优先级排好序了

#### `getMatchingCondition()`

在`getMatchingCondition()`方法完成对请求的条件校验，主要逻辑如下：

* 根据从请求域中获取属性`ServletRequestPathUtils.PATH_ATTRIBUTE`，是一个`RequestPath`(收到请求时，`DispatcherServlet`会在其`doService()`方法中设置该属性)

* 如果该`RequestPath`为`null`，抛出一个`IllegalArgumentException`异常。

* 调用`RequestPath.pathWithinApplication()`获取上下文路径后的请求路径`PathContainer`

* 遍历`this.patterns`:调用`PathPattern.matches()`方法判断是否与上一步给定的路径匹配，如果匹配将其添加到新的`TreeSet<PathPattern>`集合`matches`

* 如果`matches`为`null`表示请求条件不通过，返回`null`

* 如果`matches`不为`null`,表示请求条件通过，根据`matches`创建一个新的`PathPatternsRequestCondition`实例返回。

  可见返回的新实例中持有的都是匹配的模式。

#### `combine()`

在`combine()`方法完成请求条件的合并：

* 设两个`PathPatternsRequestCondition`为`this`和`other`
* 如果两个请求条件内容都为空，返回一个匹配根路径的请求条件(对应模式为`/`)
* 如果一个请求条件为空，返回另一个请求条件
* 如果两个条件都不为空：
  * 对`this.patterns`中的每个`PathPattern`，调用`PathPattern.combine()`合并`other.patterns`中的每一个`PathPattern`,(即假设`this.patterns`有3个`PathPattern`，`other.patterns`有4个`PathPattern`，那么去重前会产生12个`PathPattern`)
  * 将每个合并结果添加到`SortedSet<PathPattern>`集合(因为是`SortedSet`所以会自动去重)中，根据这个集合创建一个新的`PathPatternsRequestCondition`并返回

#### `compareTo()`

在`compareTo()`方法中比较两个请求条件的通用性/大小(通用性越强的请求条件越大)

* `PathPatternsRequestCondition`中的`PathPattern`本身就按从小到大排序存储。

* 设当前引用为`this`，另一个引用为`other`

* 直接比较其字典顺序(第一位与第一位比较，如果第一位就区分出了大小，直接返回，否则比较下一位。如果可比较的所有位大小都相等，则更长的更小)

### `ProducesRequestCondition`

持有如下关键字段：

~~~java
private final List<ProduceMediaTypeExpression> expressions; 
private final ContentNegotiationManager contentNegotiationManager;
~~~

其中`ProduceMediaTypeExpression`用于匹配一个`MediaType`，可以匹配一个指定的`MediaType`或者匹配除了指定`MediaType`外的所有`MediaType`(有持有的`negated`字段决定)

`expressions`由`@RequestMapping`注解的`produces`和`headers`值提供。

`ContentNegotiationManager`内容协商处理器解析请求期望的响应类型。如果创建`ProducesRequestCondition`时未指定，将使用默认的`ContentNegotiationManager`，这个默认的`ContentNegotiationManager`只持有一个`HeaderContentNegotiationStrategy`协商策略。

其`getMatchingCondition()`核心方法的主要逻辑如下：

* 如果`HttpServletRequest`请求是一个cors预检请求，直接返回一个空的`ProducesRequestCondition`实例
* 如果当前`ProducesRequestCondition`实例本身就是空的(`this.expressions`为空)，直接返回本身
* 委托持有的`ContentNegotiationManager`实例，调用其`resolveMediaTypes()`方法进行内容协商，根据请求解析出请求期望的响应类型，保存到`List<MediaType> acceptedMediaTypes`列表中
* 遍历`this.expressions`,调用`ProduceMediaTypeExpression.matche()`，判断当前表达式是否匹配给定的`acceptedMediaTypes`。如果匹配，将表达式放入新的`List<ProduceMediaTypeExpression> result`列表中
* 如果`result`不空，则匹配通过，根据`result`创建一个新的`ProducesRequestCondition`实例并返回。
* 如果`result`为空，判断`acceptedMediaTypes`中是否有`MediaType.ALL`，有说明请求期望任意的响应类型，说明匹配通过，返回一个空的`ProducesRequestCondition`实例
* 否则，说明所有的表达式都不匹配请求期望的`MeidaType`，匹配失败，返回`null`

### `RequestMappingInfo`

`RequestMappingInfo`表示请求映射信息，其中保存了各种类型的请求条件，用于匹配请求。只有当`RequestMappingInfo`持有的所有请求条件都与请求匹配时，才表示`RequestMappingInfo`与请求匹配。

它持有如下请求条件：

~~~java
private final PathPatternsRequestCondition pathPatternsCondition;
private final PatternsRequestCondition patternsCondition;
private final RequestMethodsRequestCondition methodsCondition;
private final ParamsRequestCondition paramsCondition;
private final HeadersRequestCondition headersCondition;
private final ConsumesRequestCondition consumesCondition;
private final ProducesRequestCondition producesCondition;
private final RequestConditionHolder customConditionHolder;
~~~

其中`pathPatternsCondition`和`patternsCondition`必须一个为`null`另一个不为`null`

这取决于当前`RequestMappingInfo`是用`AntPatnMatcher`还是用`PathPattern`进行路径匹配。

因为`RequestMappingInfo`的创建需要多个请求条件。所以使用了Builder设计模式，对外提供了方便创建`RequsetMappingInfo`实例的`RequestMappingInfo.Builder` API：

~~~java
public static Builder paths(String... paths); //根据提供的路径/模式返回一个builder
public Builder mutate(); //根据当前的实例返回一个Builder，允许在当前实例的基础上生成一个新的实例
~~~

`Builder`接口提供如下方法，用于设置`RequestMappingInfo`的各种字段。

~~~java
public interface Builder {
    Builder paths(String... paths);
    Builder methods(RequestMethod... methods);
    Builder params(String... params);
    Builder headers(String... headers);
    Builder consumes(String... consumes);
    Builder produces(String... produces);
    Builder mappingName(String name);
    Builder customCondition(RequestCondition<?> condition);
    Builder options(BuilderConfiguration options);
    RequestMappingInfo build();
}
~~~

其中`BuilderConfiguration`为使用`Builder`创建实例需要的配置选项容器，其中可以配置下面选项：

~~~java
private PathPatternParser patternParser; //用于创建PathPatternsRequestCondition请求条件实例
private PathMatcher pathMatcher; //用于创建PatternsRequestCondition请求条件实例
private ContentNegotiationManager contentNegotiationManager;//用于创建ProducesRequestCondition请求条件实例
~~~

一个使用`Builder`创建实例的示例：

~~~java
RequestMappingInfo.BuilderConfiguration options = new RequestMappingInfo.BuilderConfiguration();
ContentNegotiationManager contentNegotiationManager = new ContentNegotiationManager(new FixedContentNegotiationStrategy(MediaType.APPLICATION_JSON));
options.setContentNegotiationManager(contentNegotiationManager);
RequestMappingInfo requestMappingInfo = RequestMappingInfo.paths("/test/**")
        .methods(RequestMethod.GET, RequestMethod.POST)
        .produces(MediaType.APPLICATION_JSON_VALUE)
        .options(options)
        .build();
~~~

## `RequestMappingInfoHandlerMapping`

`AbstractHandlerMethodMapping`的泛型`T`类型期望一个`HandlerMethod`的`mapping`映射信息

`RequestMappingInfoHandlerMapping`实现了`AbstractHandlerMethodMapping`并固定类型参数`T`为`RequestMappingInfo`。

`RequestMappingInfoHandlerMapping`在`AbstractHandlerMethodMapping`的基础上提供`RequestMappingInfo`相关的逻辑。

创建`RequestMappingInfoHandlerMapping`时，会创建一个`RequestMappingInfoHandlerMethodMappingNamingStrategy`设置到`AbstractHandlerMethodMapping.namingStrategy`字段，`AbstractHandlerMethodMapping`根据该字段策略生成`MappingRegistry`的名称。

`AbstractHandlerMethodMapping`中有一些需要实现/重写的方法，在`RequestMappingInfoHandlerMapping`中得到了重写/实现。包括下面方法，其中映射`T`在此时就是`RequestMappingInfo`

* `getDirectPaths()`通过`T`映射获取直接路径集合，这里直接调用了`RequestMappingInfo.getDirectPaths()`方法

* `getMatchingMapping()`:判断给定的`T`映射是否匹配请求，不匹配返回`null`,这里直接调用`RequestMappingInfo.getMatchingCondition()`方法

* `getMappingComparator()`：获取`T`映射的比较器，直接使用的`RequestMappingInfo.compareTo()`生成一个比较器

* `handleMatch()`：匹配成功后，对请求做一些处理，向request域中添加一些属性：

    | 属性名                                                 | 属性值说明                                               |
    | ------------------------------------------------------ | -------------------------------------------------------- |
    | `HandlerMapping.PATH_WITHIN_HANDLER_MAPPING_ATTRIBUTE` | 请求在app上下文中的相对路径`lookupPath`                  |
    | `HandlerMapping.MATRIX_VARIABLES_ATTRIBUTE`            | 使用`PathPattern`/`AntPathMatcher`匹配路径捕获的矩阵变量 |
    | `HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE`      | 使用`PathPattern`/`AntPathMatcher`匹配路径捕获的路径变量 |
    | `HandlerMapping.BEST_MATCHING_PATTERN_ATTRIBUTE`       | 与路径匹配的最佳模式的字符串                             |

* `handleNoMatch()`：按照次序判断不匹配的原因，抛出异常，例如，因为请求方法不匹配，则抛出一个`HttpRequestMethodNotSupportedException`异常

还有一个重要的方法`getMappingForMethod()`没有实现，它最终会在`RequestMappingHandlerMapping`中实现。

`initCorsConfiguration()`

## `RequestMappingHandlerMapping`

`RequestMappingHandlerMapping`是注解式`Controller`的`HandlerMapping`的最终实现。

`RequestMappingHandlerMapping`继承`RequestMappingInfoHandlerMapping`，在此基础上:

* 解析`@Controller`类上的` @RequestMapping`和`@HttpExchange`(SpringWebFlux 的声明式Http客户端)注解信息；将其转换为`RequsetMappingInfo`
* 解析`@CrossOrigin`注解信息，将其转换为`CorsConfiguration`

以上逻辑通过实现`AbstractHandlerMethodMapping` 的    `getMappingForMethod()`和`initCorsConfiguration()`方法完成

### `getMappingForMethod()`

`getMappingForMethod()`方法根据处理器方法对应的`Method`实例和处理器方法所在的`@Controller`类信息生成一个`RequestMappingInfo`,其主要逻辑如下：

* 通过方法上的注解解析`RequestMappingInfo`：

  * 获取方法上的`@RequestMapping`注解信息，如果有多个`@RequestMapping`注解，按顺序取第一个

  * 获取方法上的`@HttpExchange`注解信息，如果有多个`@HttpExchange`注解，抛出一个`IllegalStateException`异常

  * 如果方法上既有`@RequestMapping`也有`@HttpExchange`注解，抛出一个`IllegalStateException`异常

  * 如果方法上既没有`@RequestMapping`也没有`@HttpExchange`注解，方法直接返回`null`,即当前类不是处理器方法

  * 使用`RequestMappingInfo.Builder`，通过`@RequestMapping`或者`@HttpExchange`提供的信息创建一个`RequestMappingInfo`实例
    * 创建过程中的`RequestMappingInfo.BuilderConfiguration`由`this.config`字段提供，该字段在Bean实例化完成后被配置

* 通过类上的注解解析一个`RequestMappingInfo`，过程和第一步一样
* 调用`RequestMappingInfo.combine()`将类上的请求映射信息与方法上的合并，最终的合并行为由`RequsetCondition`决定，例如：
  * 对于路径的合并，最终是调用`AntPathMatcher`或者`PathPattern`的`combine()`方法进行两个路径模式的合并
  * 对于请求方法的合并，最终为两个请求方法集合的合并去重。

### `initCorsConfiguration()`

该方法将类和方法上的`@CrossOrigin`注解信息解析为`CorsConfiguration`:

* 获取方法和类上的`@CrossOrigin`信息
* 如果方法和类上都没有`@CrossOrigin`注解，方法直接返回`null`
* 创建以一个`CorsConfiguration`实例
* 将`@CrossOrigin`上提供的cors信息配置到`CorsConfiguration`实例中，方法上的信息会覆盖类上的信息

# 注解式`Controller`的`HandlerAdapter`

`RequestMappingHandlerAdapter`负责为`DispatcherServlet`适配`HandlerMethod`处理器， 根据继承关系，它有如下主要父类：

* `WebContentGenerator`：为其子类提供Web响应内容的生成逻辑。可以根据配置来设置HTTP响应头来控制响应。
* `AbstractHandlerMethodAdapter`：是支持`HandlerMethod`的`HandlerAdapter`的抽象基类，提供`handleInternal()`抽象方法，实现`HandlerMethod`的调用处理。

`@RequestMapping`方法(即`HandlerMethod`)允许多种特定类型或注解的入参和返回值。`RequestMappingHandlerAdapter`需要

* 为`HandlerMethod`方法提供它需要的的参数值。
* 处理`HandlerMethod`方法的返回值，根据不同的返回值生成`ModelAndView`或者设置`response`。

`RequestMappingHandlerAdapter`将这样的工作委托给了专门的API来完成：

* `HandlerMethodArgumentResolver`:解析`HandlerMethod`方法的入参并提供参数值
* `HandlerMethodReturnValueHandler`：解析`HandlerMethod`的返回类型，并根据返回值生成`ModelAndView`或者设置到Web响应中

另外`RequestMappingHandlerAdapter`需要实际调用`@RequestMapping`方法，以执行其中处理请求的相关逻辑。它使用`HandlerMethod`的子类`InvocableHandlerMethod`来完成这样的调用

## `HandlerMethodArgumentResolver`

`HandlerMethodArgumentResolver`会解析参数类型和参数上的注解，并据此从相关的数据源(请求域、请求参数、Model等)获取相关数据，将其转换成参数需要的类型，以供`@RequestMapping`方法调用使用，其定义如下：

~~~java
public interface HandlerMethodArgumentResolver {


	boolean supportsParameter(MethodParameter parameter); //返回此解析器是否支持给定的方法参数

	@Nullable
	Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
			NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception;
    //解析方法参数，并根据给定请求生成对应的参数值。提供对请求模型的访问

}
~~~

其中：

* `ModelAndViewContainer`：为解析器提供对模型的访问
* `WebDataBinderFactory`:用于创建`WebDataBinder`，用于对参数值的数据绑定和类型转换

针对`@RequestMapping`方法允许的不同参数类型和参数注解，`HandlerMethodArgumentResolver`提供了对应的实现，具体整理如下：

| 类名                                                         | 支持的方法参数/注解                                          | 参数值来源                                                   |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `ErrorsMethodArgumentResolver`                               | `Errors`                                                     | `WebDataBinder`绑定其他参数时产生的`Errors`                  |
| `ExpressionValueMethodArgumentResolver`                      | `@Value`                                                     | 从`Environment`环境变量中解析`@Value`指定的表达式            |
| `MatrixVariableMethodArgumentResolver`<br />`MatrixVariableMapMethodArgumentResolver` | `@MatrixVariable`                                            | 路径中的匹配到的矩阵变量                                     |
| `PathVariableMethodArgumentResolver`<br />`PathVariableMapMethodArgumentResolver` | `@PathVariable`                                              | 路径中的匹配到的路径变量                                     |
| `PrincipalMethodArgumentResolver`                            | `Principal`                                                  | `HttpServletRequest`的`getUserPrincipal()`方法返回           |
| `RedirectAttributesMethodArgumentResolver`                   | `RedirectAttributes`                                         | 新建的一个`RedirectAttributesModelMap`实例该实例会作为一个`ModelMap`传递给重定向处理器 |
| `RequestAttributeMethodArgumentResolver`                     | `@RequestAttribute`                                          | `HttpServletRequest`的`getAttribute()`方法                   |
| `RequestHeaderMethodArgumentResolver`<br />`RequestHeaderMapMethodArgumentResolver` | `@RequestHeader`                                             | `HttpServletRequest`的`getHeaders()`方法                     |
| `RequestParamMethodArgumentResolver`<br />`RequestParamMapMethodArgumentResolver` | `@RequestParam`                                              | `MultipartHttpServletRequest`的`getFiles()/getFile()`方法或者`HttpServletRequest`的`getParameterValues()`方法 |
| `RequestPartMethodArgumentResolver`                          | `@RequestPart`                                               | `MultipartHttpServletRequest`的`getFiles()/getFile()`方法或者`MultipartHttpServletRequest`方法的`getPart().getInputStream()`方法或者使用`HttpMessageConverter`将请求体转换为参数类型 |
| `ServletCookieValueMethodArgumentResolver`                   | `@CookieValue`                                               | `HttpServletRequest`的`getCookies()`方法                     |
| `ServletRequestMethodArgumentResolver`                       | `WebRequest`、`ServletRequest`、<br />`MultipartRequest`、`HttpSession`<br />`PushBuilder`、`Principal`<br />`InputStream`、`Reader`<br />`HttpMethod`、`Locale`<br />`TimeZone`、`ZoneId` | 调用`HttpServletRequest`的各种方法或者将请求包装成特定类型   |
| `ServletResponseMethodArgumentResolver`                      | `ServletResponse`、<br />`OutputStream`、`Writer`            | 将响应包装成`ServletResponse`类型，或者调用`ServletResponse`的`getOutputStream()`或者`getWriter()`方法 |
| `SessionAttributeMethodArgumentResolver`                     | `@SessionAttribute`                                          | 调用`HttpServletRequest`的`getSession().getAttribute()`方法  |
| `SessionStatusMethodArgumentResolver`                        | `SessionStatus`                                              | 调用`ModelAndViewContainer`的`getSessionStatus()`方法        |
| `UriComponentsBuilderMethodArgumentResolver`                 | `UriComponentsBuilder`<br />`ServletUriComponentsBuilder`    | 通过`request`新建一个`ServletUriComponentsBuilder`实例       |
| `HttpEntityMethodProcessor`                                  | `HttpEntity`、`RequestEntity`                                | 使用`HttpMessageConverter`将请求体转换为`HttpEntity`泛型指定的类型 |
| `MapMethodProcessor`                                         | `Map`                                                        | 调用`ModelAndViewContainer`的`getModel()`方法                |
| `ModelMethodProcessor`                                       | `Model`                                                      | 调用`ModelAndViewContainer`的`getModel()`方法                |
| `RequestResponseBodyMethodProcessor`                         | `@RequestBody`                                               | 使用`HttpMessageConverter`将请求体转换为参数类型             |
| `ServletModelAttributeMethodProcessor`                       | `ModelAttribute`                                             | 调用`ModelAndViewContainer`的`getModel().get()`获取属性，  如果`Model`中没有该属性，从`request`的请求参数或者路径变量中获取值，并且添加到`Model`属性中 |

注意事项：

* `ServletModelAttributeMethodProcessor`有字段`annotationNotRequired`,如果该字段为`true`，它们还可以支持参数类型是`BeanUtils.isSimpleProperty()`指定的简单类型，无需注解。
* `RequestPartMethodArgumentResolver`除了支持`@RequestPart`外，还支持一种情况：参数有`@RequestParam`注解并且参数类型是Multipart参数(类型为`MultipartFile`、`Part`以及包含它们的集合/数组)
* `RequestParamMethodArgumentResolver`除了支持`@RequestParam`外，还支持一种情况：参数类型是Multipart参数(类型为`MultipartFile`、`Part`以及包含它们的集合/数组)
* 处理器映射时，`RequestMappingInfoHandlerMapping`的`handleMatch()`方法会向`request`域中存储`MatrixVariableMethodArgumentResolver`和`PathVariableMethodArgumentResolver`等需要的矩阵变量/路径变量

### `HandlerMethodArgumentResolverComposite`

`HandlerMethodArgumentResolverComposite`是`HandlerMethodArgumentResolver`的组合。它内部持有一组`HandlerMethodArgumentResolver`。

`HandlerMethodArgumentResolverComposite`会将方法参数的处理委托给它拥有的适合的`HandlerMethodArgumentResolver`。如果持有的所有的`HandlerMethodArgumentResolver`都不支持该参数类型，将抛出一个异常

## `HandlerMethodReturnValueHandler`

`HandlerMethodArgumentResolver`用于处理`HanderMethod`方法的返回值，根据返回值类型有不同的处理策略，其定义如下：

~~~java
public interface HandlerMethodReturnValueHandler {

	boolean supportsReturnType(MethodParameter returnType);

	void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
			ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception;

}
~~~

`RequestMappingHandlerAdapter`用到的`HandlerMethodReturnValueHandler`如下：

| 类名                                           | 支持的类型/注解                                              | 返回值的处理                                                 |
| ---------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `AsyncTaskMethodReturnValueHandler`            | `WebAsyncTask`                                               | 调用`WebAsyncManager`的`startCallableProcessing()`方法       |
| `CallableMethodReturnValueHandler`             | `Callable`                                                   | 调用`WebAsyncManager`的`startCallableProcessing()`方法       |
| `DeferredResultMethodReturnValueHandler`       | `DeferredResult`<br />`ListenableFuture`<br />`CompletionStage` | 调用`WebAsyncManager`的`startDeferredResultProcessing()`方法 |
| `HttpHeadersReturnValueHandler`                | `HttpHeaders`                                                | 调用`HttpServletResponse`的`addHeader()`方法                 |
| `ModelAndViewMethodReturnValueHandler`         | `ModelAndView`                                               | 调用`ModelAndViewContainer`的各种`set`方法，将`ModelAndView`中的值设置到其中 |
| `ModelAndViewResolverMethodReturnValueHandler` | 任意类型                                                     | 调用`ModelAndViewResolver`的`resolveModelAndView()`方法将返回值转换为`ModelAndView`，然后将其放入到`ModelAndViewContainer`中 |
| `ResponseBodyEmitterReturnValueHandler`        | `ResponseBodyEmitter`                                        | 创建一个`DeferredResult`实例，然后调用`WebAsyncManager`的`startDeferredResultProcessing()`方法以开启异步响应。每次`ResponseBodyEmitter`调用`send()`方法，实际就会通过`HttpMessageConverter`将`send()`的参数转换并写入请求体 |
| `StreamingResponseBodyReturnValueHandler`      | `StreamingResponseBody`                                      | 将`StreamingResponseBody`包装成一个`StreamingResponseBodyTask`,这也是一个`Callable`接口，然后调用`WebAsyncManager`的`startCallableProcessing()`方法开启异步响应 |
| `HttpEntityMethodProcessor`                    | `HttpEntity`、`RequestEntity`                                | 将`HttpEntity`或者`RequestEntity`的`body`通过`HttpMessageConverter`转换并写入响应体中 |
| `MapMethodProcessor`                           | `Map`                                                        | 将`Map`中的键值对作为属性其放入`ModelAndViewContainer`的`Model`中 |
| `ModelMethodProcessor`                         | `Model`                                                      | 将返回`Model`中的属性全部写入`ModelAndViewContainer`的`Model`中 |
| `RequestResponseBodyMethodProcessor`           | `@RequestBody`                                               | 通过`HttpMessageConverter`将返回值转换并写入响应体中         |
| `ServletModelAttributeMethodProcessor`         | `@ModelAttribute`                                            | 将返回值作为一个属性值写入`ModelAndViewContainer`的`Model`中 |

### `HandlerMethodReturnValueHandlerComposite`

`HandlerMethodReturnValueHandlerComposite`是`HandlerMethodReturnValueHandler`的组合，它持有一组`HandlerMethodReturnValueHandler`实例，将方法返回值的处理委托给这些实例。如果所有的实例都不支持该方法返回值，将抛出异常

## `RequestMappingHandlerAdapter`







# TODO

 `ModelAndViewContainer`

`ModelAndViewResolver`
