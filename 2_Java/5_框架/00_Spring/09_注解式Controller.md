# 基于注解的Controller

在之前对SpringMVC组件的讨论中，Handler处理器可以是任何形式的，只要为自定义的Handler配置对应的HandlerMapping和HandlerAdapter，就能够将其集成到SpringMVC的工作流程中。

SpringMVC提供的基于注解的Controller也是这样，提供了相应的基于注解的HandlerMapping和HandlerAdapter实现，就能为用户提供相应的注解开发支持,在之前的讨论中，我们已经提到过这样的实现:

* RequestMappingHandlerMapping
* RequestMappingHandlerAdapter

基于以上组件，SpringMVC提供了注解`@Controller`、`@RequestMapping`，通过注释来声明基于注解的Controller,告知SpringMVC框架该类是注释的Handler:

~~~java
@Controller
public class AnnotatedController {
    @RequestMapping( "/accept")
    public String accept(){
        System.out.println("收到请求");
        return "index";
    }
}
~~~

并配置以下内容:

~~~xml
<mvc:annotation-driven/>
<context:component-scan base-package="com.wzm.spring"/>
~~~

启动mvc的注解驱动，并将刚刚注释的类交给ioc容器管理即可

这样，一个`@Controller`方法注释的类中的每个`@RequestMapping`注释的方法都对应一个handler，叫做处理器方法(Handler Method)

# 基本Handler Method

我们可以使用`@Controller`和`@RequestMapping`注解注释以声明一个基于注解的Controller

因为一个`@RequestMapping`注释的方法在SpringMVC就对应一个Controller，所以被`@RequestMapping`注释的方法也可以叫处理器方法或者处理方法(Handler Method)

## `@Controller`

想要让一个普通的pojo类成为SpringMVC框架下的handler处理器，必须使用`@Controller`注解注释该类

`@Controller`的定义如下:

~~~java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface Controller {

	@AliasFor(annotation = Component.class)
	String value() default "";

}

~~~

可以看到它被`@Component`注解注释，所有被注释了`@Controller`注解的类，同样可以被扫描管理到Spring的ioc容器中，而不需要额外使用其他注解

除此之外，`RequestMappingHandlerMapping`在收到Web请求时，匹配的就是所有容器中被`@Controller`注释的对象

## `@RequsetMapping`

只拥有一个`@Controller`注解，不能让`RequestMappingHandlerMapping`知道应该将Web请求映射到哪个Controller上，需要`@RequsetMapping`提供必要的映射信息。

`@RequsetMapping`可以应用到方法级别和类级别上，定义如下:

~~~java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Mapping
public @interface RequestMapping {
	String name() default "";
	@AliasFor("path")
	String[] value() default {};
	@AliasFor("value")
	String[] path() default {};
	RequestMethod[] method() default {};
	String[] params() default {};
	String[] headers() default {};
	String[] consumes() default {};
	String[] produces() default {};
}
~~~

* `value/path`：指定匹配映射的url路径:

  * 即支持完全匹配，也支持ant风格的路径匹配(`*`、`**`、`?`通配符)

  * 类级别的`@RequsetMapping`是类中所有方法级别的`@RequestMapping`的主映射，方法级别的映射路径前会加上类级别的映射路径，例如:

    ~~~java
    @Controller
    @RequestMapping("/annotation")
    public class AnnotatedController {
        @RequestMapping( "/accept")
        public String accept(){
            System.out.println("收到请求");
            return "index";
        }
    }
    ~~~

    accept()方法对应的映射为`/annotation/accept`

  * 在方法级别时，`path`支持使用相对路径，在刚才的路径中，`accept`和`/accept`效果一样

* `method`:指定匹配映射的HTTP请求方法：

  * 值为`RequestMethod`枚举:`GET, POST, HEAD, OPTIONS, PUT, PATCH, DELETE, TRACE.`
  * 当在类级别设置`method`时，该类方法级别的`method`值将继承类级别上的`method`值，在方法级别上设置`method`值将覆盖继承的值

* `params`：指定匹配映射的HTTP请求参数:

  * 值为形式为`myParam=myValue`表达式的字符串表示匹配指定的键值
  * 表达式可以用`!=`表示不匹配指定的键值对
  * 表达式可以是`myParam`或者`!myParam`表示匹配或者不匹配指定的键
  * 当在类级别上设置`params`属性时，该类所有的方法级别的`params`将自动继承它，可以通过在方法上指定`params`值来覆盖继承的值

* `headers`:指定匹配映射的HTTP请求首部:

  * 值为形式为`My-Header=myValue`表达式的字符串表示匹配指定请求首部的键值
  * 和`params`一样，可以用`!=` 和`My-Header`表示不匹配指定的键值，或者匹配指定的键
  * 支持通配符`*`

* `consumes`:指定匹配映射的HTTP请求`Content-Type`值

  * 值可以为MediaType中指定的值
  * 支持使用`!`表示不匹配指定的`Content-Type`值

* `produces`：指定匹配映射HTTP请求的`Accept`值

  * 值可以为MediaType中指定的值
  * 支持使用`!`表示不匹配指定的`Accept`值

另外SpringMVC提供了子注解:

- `@GetMapping`
- `@PostMapping`
- `@PutMapping`
- `@DeleteMapping`
- `@PatchMapping`

他们与`@RequsetMapping`不同之处只在`method`值已经预先固定

# Handler Method参数

@RequestMapping注释的方法参数可以有很多选择，

* 可以使用特定类型的参数，RequestMappingHandlerAdapter会对特定的参数类型进行赋值或者其他操作，
* 可以使用注解注释参数，指示通知RequestMappingHandlerAdapter对该参数进行特定操作

以供Handler Method内部使

可以作为Handler Method参数的类型和可以注释Handler Method方法参数的注解如下表:

| 方法参数                                                     | 说明                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `WebRequest`, `NativeWebRequest`                             | SpringMVC提供的对request参数、request、session属性的通用访问，是对ServletAPI的封装 |
| `ServletRequest`, <br />`ServletResponse`                    | ServletAPI，也可以使用具体的实现如:`HttpServletRequest `, `MultipartRequest`, `MultipartHttpServletRequest`等 |
| `HttpSession`、`PushBuilder`、`Principal`、`HttpMethod`、`Locale`、`TimeZone`、`ZoneId`、`InputStream`、`Reader`、`OutputStream`、`Writer` | RequestMappingHandlerAdapter会从requestAPI获取对应的对象     |
| `@PathVariable`                                              | 用以访问URI的模板变量                                        |
| `@RequestParam`                                              | 用该注解注释以绑定Servlet的request域中的参数(请求参数或者form表单中的参数) |
| `@RequestHeader`                                             | 用以访问请求的请求首部信息                                   |
| `@CookieValue`                                               | 用以访问请求的Cookie信息                                     |
| `@RequestBody`                                               | 用以访问请求的请求体信息使用 `HttpMessageConverter` 实现     |
| `HttpEntity<B>`                                              | 用以访问请求头和请求体                                       |
| `@RequestPart`                                               | 用以访问格式为 `multipart/form-data`的请求的请求体的Part, 通过`HttpMessageConverter`来实现转化。 |
| `Map`, `Model`, `ModelMap`                                   | 用以访问视图信息                                             |
| `RedirectAttributes`                                         | 重定向时传递参数，让参数不会出现在URL中                      |
| `@ModelAttribute`                                            | 用以访问Model中存在的属性，如果不存在，则实例化并进行数据绑定 |
| `Errors`, `BindingResult`                                    | 访问`@ModelAtrribute`的数据绑定和校验的错误信息；或者`@RequestBody`和`@RequestPart`参数的检验的错误信息，在使用`@Valide`校验参数后必须声明`Errors`、`BindingResult` |
| `SessionStatus` + class-level `@SessionAttributes`           | 在`@Controller`类上标记了`@SessionAttributes`后，可以在方法级别的参数上声明`SessionStatus`用以清空`@SessionAttributes`声明的attributes |
| `@SessionAttribute`                                          | 绑定session属性到方法参数上                                  |
| `@RequestAttribute`                                          | 绑定request属性到方法参数上                                  |
| 其他参数                                                     | 如果一个方法参数于表中以上参数都不匹配，并且它是一个简单类型(由BeanUtils#isSimpleProperty定义)，那么这个参数将被视为一个`@RequestParam`参数，否则他将被是为一个`@ModelAttribute`参数 |

# Handler Method返回值

@RequestMapping注释的方法返回值同样有以下两种：

* 可以使用特定类型的参数
* 可以使用注解注释参数，指示通知RequestMappingHandlerAdapter对该参数进行特定操作

以帮助`RequestMappingHandlerAdapter`完成最终的ModelAndView

| 方法返回值                                                   | 说明                                                         |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| `@ResponseBody`                                              | 方法返回值将通过`HttpMessageConverter`转化并写入响应体。     |
| `HttpEntity<B>`, `ResponseEntity<B>`                         | 指定完整的响应(包括响应头和响应状态)的返回值。通过`HttpMessageConverter`转化 |
| `HttpHeaders`                                                | 返回一个只有响应头没有响应体的响应                           |
| `String`                                                     | 返回一个字符串，将被视为逻辑视图名，供`ViewResolver`解析生成视图，并渲染隐式的模型(由`@ModelAttribute`提供，或者方法参数声明的Model) |
| `View`                                                       | 直接返回一个 `View`实例，会被隐式的模型渲染(由`@ModelAttribute`提供，或者方法参数声明的Model) |
| `Map`, `Model`                                               | 返回一个将被添加到隐式Model的Map或者Model。此时viewName由 `RequestToViewNameTranslator`确定。 |
| `@ModelAttribute`                                            | 表示返回值将被添加到model中，因为没有指定view或者view的逻辑视图名，将使用`RequestToViewNameTranslator` 来隐式返回视图名 |
| `ModelAndView`                                               | 返回要使用的视图和模型属性以及(可选的)响应状态。             |
| `void`                                                       | 一个方法如果返回值是`void`类型或者返回了`null`值，那么它需要在方法内部对response进行了处理，比如通过方法参数:`ServletResponse`、`OutputStream`或者注释`@ResponseStatus`.如果控制器已经做了一个正的`ETag `或`lastModified` 时间戳检查，同样是可以的。如果以上都不成立，`void`返回类型也可以表示REST控制器的“无响应体”或HTML控制器的默认视图名称选择。 |
| `DeferredResult<V>`                                          | 从其他线程异步返回结果                                       |
| `Callable<V>`                                                | 在SpringMVC的管理线程上异步生成返回值                        |
| `ListenableFuture<V>`, `CompletionStage<V>`, `CompletableFuture<V>` | `DeferredResult`的替代对象                                   |
| `ResponseBodyEmitter`, `SseEmitter`                          | 异步响应多个值。                                             |
| `StreamingResponseBody`                                      | 可以通过`StreamingResponseBody`来异步响应输出流              |
| Other return values                                          | 如果返回值用以上方式都无法解析，那么它将被是为一个Model属性。如果返回值是一个简单类型(BeanUtils#isSimpleProperty)，那么将无法解析。 |

#  Handler Method可用注解

## `@RequestParam`

用该注解注释以绑定Servlet的request域中的参数(请求参数或者form表单中的参数)

如果取的是post请求的请求体参数，要求请求体的格式为`application/x-www-form-urlencoded`或者`multipart/form-data`

~~~java
@RequestMapping("/requestParam/demo?param=test")
public void requestParam(@RequestParam("param") String param){
    // ...
}
~~~

* 可以注释在类型为Array或者List的参数上，以解析同一参数名称的多个参数值。

  ~~~java
  // url: /requestParam/demo1?list=1,2,3,4,5
  @RequestMapping("/requestParam/demo1")
  public void requestParam(@RequestParam("list") List<String> strings){
      System.out.println(strings.size());
  }
  ~~~

* 如果注释的参数类型不是String，而是其他简单类型(由BeanUtils#isSimpleProperty定义)，SpringMVC将自动尝试使用类型转换

  * 简单类型包括：基本数据类型及其包装类、Enum、CharSequence、Number、Date、Temporal、URI、URL、Locale、Class

  ~~~java
  @RequestMapping("/demo")
  public void handle(@RequestParam("date") @DateTimeFormat(pattern="yyyy-MM-dd") Date date){
      System.out.println(date);
  }
  ~~~

* 如果启用了`MultipartResolver`，它会将格式为`multipart/form-data`的请求体解析为普通的请求参数

  这样`@RequestParam`就可以注释`MultipartFile`类型的参数以接收上传的文件:

  ~~~java
  @PostMapping("/requestParam/demo2")
  public void requestParam(@RequestParam("file")MultipartFile file,
                           @RequestParam("username") String username){
      System.out.println(file.getSize());
      System.out.println(username);
  }
  ~~~

  也可以将参数声明为`List<MultipartFile>`以接受同一参数名的多个文件值

  ~~~java
  @PostMapping("/requestParam/demo3")
  public void requestParams(@RequestParam("file") List<MultipartFile> files){
      for (MultipartFile file :files){
          System.out.println(file.getSize());
      }
  }
  ~~~

* 如果该注解指定的参数为` Map<String, String> `或者` MultiValueMap<String, String> `,并且注解没有指定参数名，那么参数将接受所有的request域中的键值对:

  ~~~java
  @PostMapping("/requestParam/demo4")
  public void requestParams(@RequestParam Map<String,String> maps){
      for (Map.Entry<String,String> entry:maps.entrySet()){
          System.out.println(entry.getKey()+"---"+entry.getValue());
      }
  }
  ~~~

**注意：**使用`@RequestParam`注解是可选的，对于任何简单类型的参数，如果没有任何相关的注解，那么默认将采用`@RequestParam`声明的绑定方式，从ServletRequest域中尝试绑定对应参数名的键

## `@RequestBody`

使用`HttpMessageConverter`将请求参数序列化为注释的参数类型:

将参数类型进行实例化，并用set方法进行参数绑定

~~~java
@PostMapping( "/requestBody/demo0")
public void requestBody(@RequestBody User user){
    System.out.println(user);
}
~~~

其中User类定义如下:

~~~java
@Data
public class User {
    private String username;
    private String password;
}
~~~

**注意:**作为`@RequestBody`注释的方法类型对象，必须有无参构造，而且对象内想要被映射的参数必须实现对应的set方法，否则无法映射

## `@RequestPart`

适用于请求体格式为`multipart/form-data`的复杂请求，可以同时解析对象和二进制文件:

~~~java
@PostMapping("/requestPart/demo0")
public void requestPart(@RequestPart("user") User user,
                        @RequestPart("file") MultipartFile file){
    System.out.println(user);
    System.out.println(file.getSize());
}
~~~

**注意:**`@RequestParam`也同样支持`multipart/form-data`的请求，它们之间的区别是:

`@RequestParam`使用Converter进行类型转换，只能转换简单类型

@RequestPart使用HttpMessageConverters 进行类型转换，支持转换复杂的参数类型

## `@PathVariable`

用以访问URI的模板变量

~~~java
@GetMapping("/owners/{ownerId}/pets/{petId}")
public Pet findPet(@PathVariable Long ownerId, @PathVariable Long petId) {
    // ...
}
~~~

如果注释的参数类型是` Map<String, String> `，则将填充所有的路径变量到该Map中

~~~java
@GetMapping("/demo/{version}/{id}")
public void handle(@PathVariable Map<String,String> maps){
    // ...
}
~~~

## `@RequestHeader`

用以访问HTTP请求的请求首部

~~~java
@GetMapping("/demo")
public void handle(
        @RequestHeader("Accept-Encoding") String encoding, 
        @RequestHeader("Keep-Alive") long keepAlive) { 
    //...
}
~~~

除了直接指定请求首部的值，还可以绑定如下类型参数以访问所有的请求首部参数:

 `Map<String, String>`、`MultiValueMap<String, String>`、 `HttpHeaders`,如:

~~~java
@GetMapping("/demo")
public void handle(@RequestHeader HttpHeaders headers){
    for (Map.Entry<String, List<String>> entry :headers.entrySet()){
        System.out.println(entry.getKey()+"---"+entry.getValue());
    }
}
~~~

## `@CookieValue`

用以访问HTTP请求的Cookie

~~~java
@GetMapping("/demo")
public void handle(@CookieValue("JSESSIONID") String cookie) { 
    //...
}
~~~

也可以直接绑定`Cookie`类型的参数:

~~~java
@GetMapping("/demo")
public void handle(@CookieValue("JSESSIONID") Cookie cookie) { 
    //...
}
~~~

## `@ModelAttribute`

`@ModelAttribute`可以注释在方法参数和方法体上

### 注释在方法参数上

`@ModelAttribute`注释在方法入参时:

可以解析`form-data`或者`x-www-form-urlencoded`格式的请求体中参数

可以从model中访问或者创建一个对象，然后通过`WebDataBinder`将该对象和请求参数进行绑定

* 先从Model中获取指定的参数类型，由`@ModelAttribute`注解的value/name值指定绑定的model属性名，如果不指定，默认由参数类型获取:
  * 如User类默认绑定的属性名是user

  * 如`List<User>`类默认绑定的属性名是userList

* 如果获取不到，则实例化参数类型

* 从ServletRequest域中获取对应的字段，绑定到参数实例中。这被称为参数绑定。

* 最后将参数绑定后的参数实例也添加到Model中暴露给视图

~~~java
@PostMapping("/modelAttribute/demo")
public void modelAttribute(@ModelAttribute User user, Model model){
    System.out.println(user);
    User u =(User) model.getAttribute("user");
    System.out.println(u);
}
~~~

在实例化参数类型后，会应用参数绑定，将ServletRequest参数名匹配到目标参数类型的字段名

参数绑定可能会出现异常，为了在方法内部处理异常，可以同时定义`BindingResult`类型的参数，在方法内部访问绑定结果:

~~~java
@PostMapping("/modelAttribute/demo1")
public void modelAttribute(@ModelAttribute User user, BindingResult bindingResult){
    System.out.println(user);
    System.out.println(bindingResult.hasErrors());
}
~~~

如果只想要访问Model中的属性，而不想进行参数绑定，可以指定`@ModelAttribute`注解的属性`binding=false`。

~~~java
@ModelAttribute
public User setModel(){
    return new User("test","test");
}

@PostMapping("/modelAttribute/demo2")
public void modelAttribute(@ModelAttribute(binding = false) User user){
    System.out.println(user);
}
~~~

**注意:**使用`@ModelAttribute`是可选的，默认请求下，任何不是简单类型( 由BeanUtils#isSimpleProperty定义)的参数，并且该参数没有被其他参数处理器处理过，那么该参数就会被视为被`@ModelAttribute`注解注释了。

### 注释在方法体上

单独注释`@Controller`类中的方法，为该类中其他所有`@RequestMapping`方法初始化model

一个`@Controller`控制器中可以有任意数量的`@ModelAttribute`方法。接收到Web请求后，所有的这些`@ModelAttribute`方法将先于`@RequestMapping`方法被调用，为其初始化model

`@ModelAttribute`方法可以有多种结构:

* 有返回值，会将返回值添加为model属性:

  ~~~java
  @ModelAttribute
  public User setModel(){
      return new User("test","test");
  }
  ~~~

* 无返回值，在方法体内添加任意model属性:

  ~~~java
  @ModelAttribute
  public void setModel(Model model) {
      model.addAttribute(new User("test","test"));
      ......
  }
  ~~~

当然，`@ModelAttribute`方法和`@RequestMapping`方法一样，可以定义特定类型的方法参数或者为方法参数注释特定注解，以访问请求的信息

除了单独注释`@Controller`类中的方法，`@ModelAttribute`还可以于`@RequestMapping`注解组合使用，以注释方法的返回值是`model`中的属性。通常情况下这种使用方式是不必要的，因为这是controller的默认行为，除非你的返回值是String，而不想让该String被解释为是view 的逻辑名:

~~~java
@PostMapping("/modelAttribute/demo3")
@ModelAttribute("key")
public String modelAttribute(){
    return "value";
}
~~~

## `@SessionAttributes`

`@SessionAttributes`用于在请求之间的HTTP Servlet Session中存储model属性：

~~~java
@Controller
@SessionAttributes("user")
@RequestMapping("/user")
public class SessionAttributesController {
    @PostMapping("/add")
    public void addUser(User user){
        System.out.println(user);
        System.out.println("已添加");
    }
    @GetMapping("/get")
    @ResponseBody
    public User getUser(Model model){
        return (User) model.getAttribute("user");
    }
    @GetMapping("/delete")
    public void deleteUser( SessionStatus status){
        status.setComplete();
    }
}
~~~

## ` @SessionAttribute`

从Servlet的session域中获取已经存在的属性(之前的请求创建的，或者Filter和HandlerIntercepter创建的)

~~~java
@GetMapping("/sessionAttribute/demo")
public void sessionAttribute(@SessionAttribute("user") User user){
    System.out.println(user);
}
~~~

## `@RequestAttribute`

从Servlet的request域中获取提前创建的已经存在的属性(Filter和HandlerIntercepter创建的)

~~~java
@GetMapping("/requestAttribute/demo")
public void requestAttribute(@RequestAttribute("user") User user){
    System.out.println(user);
}
~~~

## `@ResponseBody`

```java
@RequestMapping("/getUser")
@ResponseBody
public User user(){
    return new User("test","test");
}
```

该注解也可和`@Controller`组合使用，表示该`@Controller`类中所有方法都隐士注释了`@ResponseBody`

spring4.0提供了`@RestController`表示`@Controller`和`@ResponseBody`的组合注解

**注意:**作为`@ResponseBody`注解方法返回值的对象，一般需要实现`Serializable`接口，才能实现序列化成json字符串。

# Handler Method可用实体类

SpringMVC允许和提供了一些对象，以声明在Handler Method的参数和返回值上

## `WebRequest`

`WebRequest`、`NativeWebRequest`:SpringMVC提供的对request参数、request、session属性的通用访问，是对ServletAPI的封装

示例：

~~~java
@RequestMapping("accept")
public void accept(WebRequest request){
	......
}
~~~

## `ServletRequest`&`ServletResponse`

`ServletRequest`、`ServletResponse`:ServletAPI，也可以使用具体的实现如:`HttpServletRequest `, `MultipartRequest`, `MultipartHttpServletRequest`等

示例：

~~~java
@RequestMapping("accept")
public void accept(HttpServletRequest request, HttpServletResponse response){
    // ...
}
~~~

## ServletRequestAPI对象

可以直接指定下面类型的对象，RequestMappingHandlerAdapter会通过Servlet RequestAPI从当前请求获取对应的对象:

`HttpSession`、`PushBuilder`、`Principal`、`HttpMethod`、`Locale`、`TimeZone`等

~~~java
@RequestMapping("accept")
public void accept(HttpSession session,HttpMethod method){
	// ...
}
~~~

**注意:**对于`HttpSession`，该对象不是线程安全的，如果有多个Controller使用同一个`HttpSession`对象的情况，需要将`RequestMappingHandlerAdapter`实例的`synchronizeOnSession`设置为true

## `Model`

SpringMVC在调用处理器方法之前，会创建一个隐式的Model对象，作为模型数据的存储容器

如果Handler Method的入参有声明`Map`、`Model`或者其子类(如`ModelMap`),则SpringMVC会将事先创造的Model的引用传递给入参，或者将其中的属性填充到map里供方法访问model属性。

可以在处理方法内访问model属性，也可以像model中添加model

~~~java
@RequestMapping("/accept")
public String accept(Model model){
	// ...
}
~~~

## `RedirectAttributes`

默认情况下，所有model属性都被认为是作为重定向URL中的URI模板变量公开的。在其余属性中，那些基本类型或基本类型的集合或数组将自动作为查询参数追加。

~~~java
@Controller
public class RedirectController {
    @GetMapping("/redirect")
    public String demo(Model model){
        model.addAttribute("param","test");
        model.addAttribute("number",123);
        return "redirect:/target/{param}";
    }

    @GetMapping("/target/{param}")
    public void redirect(@PathVariable("param") String param, int number, HttpServletRequest request){
        System.out.println(request.getRequestURI());
        Map<String, String[]> parameterMap = request.getParameterMap();
        for (Map.Entry<String, String[]> entry: parameterMap.entrySet()){
            System.out.println(entry.getKey()+"---"+ Arrays.toString(entry.getValue()));
        }
        System.out.println(param);
        System.out.println(number);
    }
}
~~~

但是有时候我们不希望有些参数出现在URI中，就可以使用Model的子接口:`RedirectAttributes`,同样可以传递参数，且不会出现在URI中:

~~~java
@Controller
public class RedirectController {
    @GetMapping("/redirect")
    public String demo(RedirectAttributes redirectAttributes) {
        redirectAttributes.addAttribute("param","test");
        return "redirect:/target";
    }
    @GetMapping("/target")
    public void redirect(String param) {
        System.out.println(param);
    }
}
~~~

## `HttpEntity`

与`@RequestBody`作用类似，解析请求域中键值对，将其序列化为指定的实体类:

~~~java
@PostMapping("demo7")
public void handle(HttpEntity<User> httpEntity){
    System.out.println(httpEntity.getBody());
}
~~~

同时也可以作为方法的返回值，被写入响应体

~~~java
@RequestMapping("/getUserByEntity")
public HttpEntity<User> getUserByHttpEntity(){
    return new HttpEntity<>(new User("test","test"));
}
~~~

## `ResponseEntity`

作为方法的返回值，和`@ResponseBody`类似，但会返回响应状态和响应头:

~~~java
@RequestMapping("/getUserByResponseEntity")
public ResponseEntity<User> getUserByResponseEntity(){
    return ResponseEntity.ok(new User("test","test"));
}
~~~

# 处理Handler Method出入参

我们知道，Web请求和响应的正文都是不同媒体格式的字符串类型，而我们定义的HandlerMethod出入参一般是Java对象，`RequestMappingHandlerAdapter`需要一些组件的支持，以实现Java对象和Web请求响应体的转换

## HttpMessageConverter

在前面介绍MehtodHandler方法的返回值和入参时，已经多次提到了`HttpMessageConverter`,它能帮助`RequestMappingHandlerAdapter`：

* 将读取到的Web请求体中的请求体信息转换为对象传递给Handler Method的参数
* 将Handler Method返回的对象转换为响应体信息以写入Web响应体

`@RequestBody`、`@ResponseBody`、`@RequestPart`、`HttpEntity`、`ResponseEntity`背后的实现都需要`HttpMessageConverter`的支持

以实现类型转换和数据绑定

其定义如下:

~~~java
public interface HttpMessageConverter<T> {
	//指示给定的类型是否是可读的，同时指示支持的请求体Content-Type
	boolean canRead(Class<?> clazz, MediaType mediaType);
	//指示可写的对象类型，同时指示支持的响应体Content-Type
	boolean canWrite(Class<?> clazz, MediaType mediaType);
	//返回当前转换器支持的媒体类型
	List<MediaType> getSupportedMediaTypes();
	//将请求信息流转换为T类型对象
	T read(Class<? extends T> clazz, HttpInputMessage inputMessage);
	//请T类型对象实例转换为输出信息流
	void write(T t, MediaType contentType, HttpOutputMessage outputMessage);
}
~~~

Spring为`HttpMessageConverter`提供了众多的实现类，以支持不同场景的类型转换:

| `HttpMessageConverter`实现                | `<T>` Type                         | Supported Read<br />Content-Type    | Supported Write <br />Content-Type                           |
| ----------------------------------------- | ---------------------------------- | ----------------------------------- | ------------------------------------------------------------ |
| `StringHttpMessageConverter`              | `String`                           | `*/*`                               | `text/plain`                                                 |
| `FormHttpMessageConverter`                | `MultiValueMap`<br />`<String, ?>` | `application/x-www-form-urlencoded` | `application/x-www-form-urlencoded`<br />`multipart/form-data` |
| `AllEncompassingFormHttpMessageConverter` | `MultiValueMap`<br />`<String, ?>` | `application/x-www-form-urlencoded` | `application/x-www-form-urlencoded`<br />`multipart/form-data` |
| `ResourceHttpMessageConverter`            | `Resource`                         | `*/*`                               | 由写入的`Resource`决定                                       |
| `BufferedImageHttpMessageConverter`       | `BufferedImage`                    | 由注册的` image readers`决定        | `BuferedImage`对应的类型                                     |
| `ByteArrayHttpMessageConverter`           | `byte[]`                           | `*/*`                               | `application/octet-stream`                                   |
| `SourceHttpMessageConverter`              | `T extends Source`                 | `text/xml`、`application/xml`       | `text/xml`、`application/xml`                                |
| `MappingJackson2HttpMessageConverter`     | `Object`                           | `application/json`                  | `application/json`                                           |
| `RssChannelHttpMessageConverter`          | `Channel`                          | `application/rss+xml`               | `application/rss+xml`                                        |
| ······                                    |                                    |                                     |                                                              |

`RequestMappingHandlerAdapter`默认已经装配了下面`HttpMessageConverter`:

* `StringHttpMessageConverter`
* `ByteArrayHttpMessageConverter`
* `SourceHttpMessageConverter`
* `AllEncompassingFormHttpMessageConverter`

如果想要使用其他的`HttpMessageConverter`，需要在SpringWeb容器中显示定义`RequestMappingHandlerAdapter`并指定`HttpMessageConverter`

## DataBinder

 `@RequestParam`, `@RequestHeader`, `@PathVariable`, `@CookieValue`、`@ModelAttribute`等注解指示的方法参数的传递，需要`DataBinder`来实现数据绑定，绑定过程包括数据的转换、格式化、校验等

其中`@ModelAttribute`绑定的是参数对象中的字段

![DataBinder工作流程](https://gitee.com/wangziming707/note-pic/raw/master/img/DataBinder%E5%B7%A5%E4%BD%9C%E6%B5%81%E7%A8%8B.jpeg)

SpringMVC将ServletRequest对象以及处理方法的入参对象实例传递给`DataBinder`

* `DataBinder`首先调用装配在SpringWeb上下文的ConverionService组件进行数据转换、数据格式化的工作，将ServletRequest中的消息填充到入参对象中
* 然后调用`Validator`组件对已经绑定了请求消息数据的入参对象进行数据合法性校验，最终生成数据绑定结果BindingResult对象

### 数据转换

数据类型转换工作由DataBinder委托ConversionService完成，其定义如下:

~~~Java
public interface ConversionService {
	boolean canConvert(Class<?> sourceType, Class<?> targetType);
	//判断是否能将source类型转换为target类型
	boolean canConvert(TypeDescriptor sourceType, TypeDescriptor targetType);
	//通过TypeDescriptor，判断是否能将source类型转换为target类型
    //TypeDescriptor不但描述了需要转换类的信息，还描述了宿主类的上下文信息，如注解等
	<T> T convert(Object source, Class<T> targetType);
	//进行类型转换
	Object convert(Object source, TypeDescriptor sourceType, TypeDescriptor targetType);
    //通过TypeDescriptor进行类型转换
}
~~~

可以通过`ConversionServiceFactoryBean`注册一个`DefaultConversionService`在SpringIoC容器中

`DefaultConversionService`提前注册了一些常用的`Converter`以支持常见的类型转换，在使用`ConversionServiceFactoryBean`时，可以通过`converters`添加一些其他的或者自定义的`Converter`：

~~~xml
<bean id="myConversionService" class="org.springframework.context.support.ConversionServiceFactoryBean">
    <property name="converters">
        <set>
            <bean class="com.wzm.spring.converter.MyConverter"/>
        </set>
    </property>
</bean>
~~~

ConversionServiceFactoryBean支持注册以下三种`converter`:

* `Converter<S, T>`
* `GenericConverter`
* `ConverterFactory`

使用SpringMVC提供的命名空间`mvc`启动注解驱动时:

~~~xml
<mvc:annotation-driven/>
~~~

如果没有指定，将注册一个默认的`RequestMappingHandlerAdapter`。和默认的`ConversionService`，即`FormattingConversionServiceFactoryBean`

可以通过指定属性`conversion-service`覆盖默认的`ConversionService`

~~~xml
<mvc:annotation-driven conversion-service="myConversionService" >
~~~

#### 使用自定义Converter

通过以上信息，我们可以自定义一个Converter并注册到SpringMVC的框架流程中:

~~~java
public class MyConverter implements Converter<String, User> {
    //String format: "username;password"
    @Override
    public User convert(String source) {
        String[] split = source.split(";");
        return new User(split[0],split[1]);
    }
}
~~~

这样就可以使用对应的String格式来传递User:

~~~java
@RequestMapping("/converter")
@ResponseBody
public String converter(User user){
    System.out.println(user);
    return "ok";
}
//GET: /converter?user=wangziming;123321
//print: User(username=wangziming, password=123321)
~~~

### 数据格式化

在对日期、时间、数字、货币等具有一定格式的数据进行类型转化时，需要使用特定的格式转换器进行转换，根据不同的本地化环境和给定的格式信息进行转换。

在刚刚讲数据转换时，可能已经注意到了，SpringMVC注册的默认的`ConversionService`是`FormattingConversionServiceFactoryBean`,它生产`FormattingConversionService`通过它的名字就可以知道，它同样负责数据的格式化工作，实际上该类实现了`FormatterRegistry`以注册格式化组件，`FormatterRegistry`定义如下:

~~~java
public interface FormatterRegistry extends ConverterRegistry {
	void addPrinter(Printer<?> printer);
	void addParser(Parser<?> parser);
	void addFormatter(Formatter<?> formatter);
	void addFormatterForFieldType(Class<?> fieldType, Formatter<?> formatter);
	void addFormatterForFieldType(Class<?> fieldType, Printer<?> printer, Parser<?> parser);
	void addFormatterForFieldAnnotation(AnnotationFormatterFactory<? extends Annotation> annotationFormatterFactory);

}
~~~

该接口规范了添加`Printer`、`Parser`、`Formatter`、`AnnotationFormatterFactory`的方法，以让实现该接口的类拥有数据格式化的能力:

* `Printer`将对象和根据locale转换为字符串
* `Parser`将字符串根据locale转化为对象
* `Formatter`只继承`Printer`和`Parser`
* `AnnotationFormatterFactory`：生产格式化器来格式化用特定注解标注的字段的值

使用注解注释参数以指示格式化格式相对更方便一些，注解方式的格式化工作由`AnnotationFormatterFactory`完成，Spring提供了一些实现类:

* `DateTimeFormatAnnotationFormatterFactory`—对应注解`@DateTimeFormat`
* `NumberFormatAnnotationFormatterFactory`—对应注解`@NumberFormat `

等

启用SpringMVC的注解驱动后，默认支持这两个格式化注解:

~~~java
@RequestMapping("/formatter")
@ResponseBody
public String formatter(@DateTimeFormat(pattern = "yyyy-MM-dd") Date date,
                        @NumberFormat(pattern = "#,###.##") long number){
    System.out.println(date);
    System.out.println(number);
    return "OK";
}
//GET: /formatter?date=2021-01-01&number=2,222.01
//Print Fri Jan 01 00:00:00 CST 2021 和 2222
~~~

### 数据校验

#### JSR-303

应用程序在执行业务逻辑前，必须通过数据校验以保证接收到的输入数据是正确合法的，比如工资必须是正的、生日必须是过去的时间等等。为了避免数据校验的代码分散在应用程序的不同层，最好将验证逻辑与相应的域模型进行绑定，将代码验证的逻辑集中起来管理。

JSR-303是Java为Bean数据合法性校验提供的标准框架。`hibernate-validator`包是对该框架的实现，可以引入它来进行数据校验:

~~~xml
 <!-- 实现了jdk的Validator、ConstraintValidator、Constraint接口，提供校验实现 -->
 <dependency>
     <groupId>org.hibernate.validator</groupId>
     <artifactId>hibernate-validator</artifactId>
     <version>6.1.5.Final</version>
 </dependency>

~~~

JSR-303定义了如下注解规范:

| 注解                        | 代码内容                                                 |
| --------------------------- | -------------------------------------------------------- |
| @Null                       | 被注释的元素必须为 null                                  |
| @NotNull                    | 被注释的元素必须不为 null                                |
| @AssertTrue                 | 被注释的元素必须为 true                                  |
| @AssertFalse                | 被注释的元素必须为 false                                 |
| @Min(value)                 | 被注释的元素必须是一个数字，其值必须大于等于指定的最小值 |
| @Max(value)                 | 被注释的元素必须是一个数字，其值必须小于等于指定的最大值 |
| @DecimalMin(value)          | 被注释的元素必须是一个数字，其值必须大于等于指定的最小值 |
| @DecimalMax(value)          | 被注释的元素必须是一个数字，其值必须小于等于指定的最大值 |
| @Size(max, min)             | 被注释的元素的大小必须在指定的范围内                     |
| @Digits (integer, fraction) | 被注释的元素必须是一个数字，其值必须在可接受的范围内     |
| @Past                       | 被注释的元素必须是一个过去的日期                         |
| @Future                     | 被注释的元素必须是一个将来的日期                         |
| @Pattern(value)             | 被注释的元素必须符合指定的正则表达式                     |

hibernate-validator提供的额外注解:

| @代码     | 代码内容                               |
| --------- | -------------------------------------- |
| @Email    | 被注释的元素必须是电子邮箱地址         |
| @Length   | 被注释的字符串的大小必须在指定的范围内 |
| @NotEmpty | 被注释的字符串的必须非空               |
| @Range    | 被注释的元素必须在合适的范围内         |

#### Spring Validator

spring有自己的数据验证框架，同时支持JSR-303框架

spring提供的验证框架的顶级接口是`org.springframework.validation.Validator`

spring提供的`LocalValidatorFactoryBean`同时实现了spring的`Validator`和 JSR-303框架的`Validator`

`<mvc:annotation-driven/>`默认会装配一个`LocalValidatorFactoryBean`,只要在需要检验数据的参数注释`@Valid`注解，即可通知`DataBinder`在完成数据转化后使用`LocalValidator`进行数据校验，数据校验的结果可以通过声明` BindingResult`和`Errors`来访问

定义进行数据校验的对象:

~~~java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class User implements Serializable {
    @NonNull
    private String username;
    @NonNull
    private String password;
    @Past
    @DateTimeFormat(pattern = "yyyy-MM-dd")
    private Date birthday;
}
~~~

进行数据校验:

~~~java
@RequestMapping("/validator")
@ResponseBody
public String validator(@Valid User user, BindingResult result){
    System.out.println(user);
    System.out.println(result.hasErrors());
    List<ObjectError> allErrors = result.getAllErrors();
    for (ObjectError error : allErrors) {
        System.out.println(error);
    }
    return "OK";
}
//GET:    /validator?password=123123&birthday=2029-02-02
//Print:  username不能为null ，birthday 需要是一个过去的时间
~~~

# 异步请求

servlet和SpringMVC支持对请求进行异步响应。接收到Web请求后，可以在保持Web连接的情况下结束Servlet容器线程，在其他线程进行响应处理，要开启异步支持，首先需要设置Servlet和所有的Filter支持异步`<async-supported>`:

~~~xml
<servlet>
    <servlet-name>dispatcherServlet</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath:dispatcher-servlet-annotation.xml</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
    <async-supported>true</async-supported>
</servlet>

<filter>
    <filter-name>encodingFilter</filter-name>
    <filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
    <async-supported>true</async-supported>
</filter>
~~~

并且可以设置mvc以控制异步行为:

~~~xml
<mvc:annotation-driven enable-matrix-variables="true" >
    <mvc:async-support default-timeout="100000"/>
</mvc:annotation-driven>
~~~

然后可以声明下面的Handler Method返回值以实现异步请求

## `DeferredResult`

当一个请求到达API接口，如果该API接口的return返回值是DeferredResult，在没有超时或者DeferredResult对象设置setResult时，接口不会返回，但是Servlet容器线程会结束，`DeferredResult`另起线程来进行结果处理(即这种操作提升了服务短时间的吞吐能力)，并`setResult()`，如此以来这个请求不会占用服务连接池太久，如果超时或设置`setResult()`，接口会立即返回。

~~~java
@Controller
@RequestMapping("/async")
public class AsynchronousController {
    @GetMapping("/demo1")
    @ResponseBody
    public DeferredResult<String> demo1(){
        DeferredResult<String> deferredResult = new DeferredResult<>();
        MyThread myThread = new MyThread(deferredResult);
        myThread.start();
        return deferredResult;
    }
}
~~~

最终的结果在MyThread的run方法中处理：

~~~java
class MyThread extends Thread{
    private final DeferredResult<String> deferredResult;
    public MyThread(DeferredResult<String> deferredResult){
        this.deferredResult = deferredResult;
    }
    @SneakyThrows
    @Override
    public void run() {
        Thread.sleep(3000);
        deferredResult.setResult("123");
    }
}
~~~

## `Callable`

作用和DeferredResult类型，不过让SpringMVC线程来异步执行callable任务:

~~~java
@GetMapping("/demo")
@ResponseBody
public Callable<String> demo2(){
    return () ->{
        Thread.sleep(3000);
        return "132";
    };
}
~~~

如果不设置，默认使用`SimpleAsyncTaskExecutor`来执行异步任务

## `ResponseBodyEmitter`

使用`DeferredResult`和`Callable`只能异步返回一个值。

使用`ResponseBodyEmitter`可以异步返回多个值。

```java
@GetMapping("/demo3")
public ResponseBodyEmitter demo3(){
    ResponseBodyEmitter emitter = new ResponseBodyEmitter();
    EmitterThread thread = new EmitterThread(emitter);
    thread.start();
    return emitter;
}
```

其中EmitterThread为:

~~~java
class EmitterThread extends Thread{
    private final ResponseBodyEmitter emitter;
    public EmitterThread(ResponseBodyEmitter emitter){
        this.emitter = emitter;
    }
    @SneakyThrows
    @Override
    public void run() {
        emitter.send("123");
        Thread.sleep(1000);
        emitter.send("456");
        Thread.sleep(1000);
        emitter.send("789");
        emitter.complete();
    }
}
~~~

也可以将`ResponseBodyEmitter`作为`ResponseEntity`的body，以设置响应头和状态码。

可以用fetchAPI接收响应流：

~~~javascript
function demo() {
    let url = "....../async/demo3"
    fetch(url).then((response) => push(response.body.getReader()));
}


function push(reader) {
    reader.read().then(({done, value}) => {
        if (done) {
            return;
        }
        console.log(Uint8ArrayToString(value));
        $("#content").append(Uint8ArrayToString(value))
        push(reader);
    });
}

function Uint8ArrayToString(fileData){
    let dataString = "";
    for (let i = 0; i < fileData.length; i++) {
        dataString += String.fromCharCode(fileData[i]);
    }
    return dataString
}
~~~

## `SseEmitter`

`SseEmitter `(`ResponseBodyEmitter`的子类)提供了对服务器发送事件的支持，其中从服务器发送的事件按照W3C SSE(Server-Sent Events)规范格式化。

~~~java
@GetMapping("/demo4")
public SseEmitter demo4(){
    SseEmitter emitter = new SseEmitter();
    EmitterThread thread = new EmitterThread(emitter);
    thread.start();
    return emitter;
}
~~~

其中`EmitterThread`和上一个示例一样

需要用sse规定的EventSourceAPI来处理响应:

~~~javascript
function demo() {
    let url = "....../async/demo4"
    const evtSource = new EventSource(url);
    evtSource.onmessage = function(event) {
        $("#content").append(event.data)
    }
    evtSource.onerror = function (event) {
        console.log("error")
        evtSource.close();
    }
}
~~~

## `StreamingResponseBody`

可以通过`StreamingResponseBody`来异步响应输出流

~~~java
@GetMapping("/demo5")
public StreamingResponseBody demo5(){
    return outputStream -> {
        for (int i = 0; i < 10; i++) {
            outputStream.write(Integer.toString(i).getBytes(StandardCharsets.UTF_8));
            try {
                Thread.sleep(500);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        }
    };
}
~~~

