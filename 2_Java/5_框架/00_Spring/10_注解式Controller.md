# 基于注解的Controller

Spring MVC提供了一个基于注解的编程模型，通过`@Controller`、`@RequestMapping`注解来表达请求映射、请求输入、异常处理等内容。注解的控制器具有灵活的方法签名，不需要继承基类，也不需要实现特定的接口。下面例子声明一个由注解定义的Controller：

~~~java
@Controller
public class HelloController {
    @RequestMapping("/hello")
    public String handle(Model model) {
        model.addAttribute("message", "Hello World!");
        return "index";
    }
}
~~~

示例中方法接受一个 `Model`，并返回一个 `Stri@ng` 的视图名称，但这不是固定了，可以指定灵活的方法入参和返回值，后面会详细介绍。

这样，一个`@Controller`方法注释的类中的每个`@RequestMapping`注释的方法都对应一个handler请求处理器，所以被`@RequestMapping`注释的方法也可以叫处理器方法或者处理方法(Handler Method)

## 注册`@Controller`

`@Controller`类作为组件需要注册到`DispatcherServlet`绑定的`WebApplicationContext`中。

因为`@Controller`有元注解`@Component`,所以它允许自动检测。

要实现对这种 `@Controller` Bean的自动检测，可以为Java配置添加组件扫描：

~~~java
@Configuration
@ComponentScan("org.example.web")
public class WebConfig {

    // ...
}
~~~

或者使用等效的XML配置：

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:p="http://www.springframework.org/schema/p"
    xmlns:context="http://www.springframework.org/schema/context"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        https://www.springframework.org/schema/context/spring-context.xsd">

    <context:component-scan base-package="org.example.web"/>
    <!-- ... -->
</beans>
~~~

# `@RequsetMapping`

 `@RequestMapping` 可以注释到`@Contorller`方法上，告知`DispatcherServlet`当前方法是一个Handler，并声明URL请求路径和Handler方法的映射关系。`@RequestMapping`提供各种属性，用以通过URL、HTTP方法、请求参数、header和媒体类型（meida type）与请求匹配映射。

也可以在`@Controller`类上注释 `@RequsetMapping`,来表达共享映射。

一个示例：

~~~java
@RestController
@RequestMapping("/persons")
class PersonController {

    @GetMapping("/{id}")
    public Person getPerson(@PathVariable Long id) {
        // ...
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public void add(@RequestBody Person person) {
        // ...
    }
}
~~~

## 方法匹配

`@RequestMapping`的`method`属性指定匹配映射的HTTP请求方法：

* 值为`RequestMethod`枚举:`GET, POST, HEAD, OPTIONS, PUT, PATCH, DELETE, TRACE.`
* 当在类级别设置`method`时，该类方法级别的`method`值将继承类级别上的`method`值，在方法级别上设置`method`值将覆盖继承的值

 `@RequestMapping` 有限定HTTP方法的变体：

- `@GetMapping`
- `@PostMapping`
- `@PutMapping`
- `@DeleteMapping`
- `@PatchMapping`

这些注解和 `@RequestMapping`唯一不同之处在于固定了其`method`属性

## URI路径匹配

 `@RequsetMapping`的`value/path`属性指定匹配映射的url路径:

即支持完全匹配，也支持ant风格的路径匹配(`*`、`**`、`?`通配符)。它支持捕获 pattern，例如 `{*spring}`。 `**` 在匹配多个路径段，但只允许在 pattern 的末端使用。

一些示例 pattern：

- `"/resources/ima?e.png"` - 匹配路径段中的一个字符
- `"/resources/*.png"` - 匹配一个路径段中的零个或多个字符
- `"/resources/**"` - 匹配多个路径段
- `"/projects/{project}/versions"` - 匹配一个路径段并将其作为一个变量捕获
- `"/projects/{project:[a-z]+}/versions"` - 匹配并捕获一个带有正则的变量

### 共享映射

类级别的`@RequsetMapping`是类中所有方法级别的`@RequestMapping`的主映射，方法级别的映射路径前会加上类级别的映射路径，例如:

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

在方法级别时，`path`支持使用相对路径，在刚才的路径中，`accept`和`/accept`效果一样

### URI变量

捕获的URI变量可以用 `@PathVariable` 访问。例如：

~~~java
@GetMapping("/owners/{ownerId}/pets/{petId}")
public Pet findPet(@PathVariable Long ownerId, @PathVariable Long petId) {
    // ...
}
~~~

可以在类和方法层面上声明URI变量，如下例所示：

~~~java
@Controller
@RequestMapping("/owners/{ownerId}")
public class OwnerController {

    @GetMapping("/pets/{petId}")
    public Pet findPet(@PathVariable Long ownerId, @PathVariable Long petId) {
        // ...
    }
}
~~~

URI变量会自动转换为适当的类型，简单的类型（`int`、`long`、`Date` 等）是默认支持。也可以注册对任何其他数据类型的支持。后面在类型转换和`DataBinder`时会详细讲解。

可以明确地命名URI变量（例如，`@PathVariable("customId")`），但是如果名称相同，并且你的代码是用 `-parameters` 编译器标志编译的，你可以不考虑这个细节。

语法 `{varName:regex}` 用正则表达式声明一个URI变量，例如，给定 URL `"/spring-web-3.0.5.jar"`，以下方法可以提取名称、版本和文件扩展名：

~~~java
@GetMapping("/{name:[a-z-]+}-{version:\\d\\.\\d\\.\\d}{ext:\\.[a-z]+}")
public void handle(@PathVariable String name, @PathVariable String version, @PathVariable String ext) {
    // ...
}
~~~

## Content-Type匹配

`@RequsetMapping`的`consumes`属性指定匹配映射的HTTP请求`Content-Type`值

* 值可以为MediaType中指定的值
* 支持使用`!`表示不匹配指定的`Content-Type`值

例如：

~~~java
@PostMapping(path = "/pets", consumes = "application/json")
public void addPet(@RequestBody Pet pet) {
    // ...
}
~~~

以在类的层次上声明一个共享的 `consumes` 属性。此时方法级 `consumes` 属性覆盖而不是扩展类级声明。

## Accept匹配

`@RequsetMapping`的`produces`属性指定匹配映射HTTP请求的`Accept`值

* 值可以为MediaType中指定的值
* 支持使用`!`表示不匹配指定的`Accept`值

~~~java
@GetMapping(path = "/pets/{petId}", produces = "application/json")
@ResponseBody
public Pet getPet(@PathVariable String petId) {
    // ...
}
~~~

以在类的层次上声明一个共享的 `produces` 属性。此时方法级 `produces` 属性覆盖而不是扩展类级声明。

## header和参数匹配

`@RequsetMapping`的`params`属性指定匹配映射的HTTP请求参数:

* 值为形式为`myParam=myValue`表达式的字符串表示匹配指定的键值
* 表达式可以用`!=`表示不匹配指定的键值对
* 表达式可以是`myParam`或者`!myParam`表示匹配或者不匹配指定的键
* 当在类级别上设置`params`属性时，该类所有的方法级别的`params`将自动继承它，可以通过在方法上指定`params`值来覆盖继承的值

示例：

~~~java
@GetMapping(path = "/pets/{petId}", params = "myParam=myValue")
public void findPet(@PathVariable String petId) {
    // ...
}
~~~

`@RequsetMapping`的`headers`属性指定匹配映射的HTTP请求首部:

* 值为形式为`My-Header=myValue`表达式的字符串表示匹配指定请求首部的键值
* 和`params`一样，可以用`!=` 和`My-Header`表示不匹配指定的键值，或者匹配指定的键
* 支持通配符`*`

示例：

~~~java
@GetMapping(path = "/pets/{petId}", headers = "myHeader=myValue") 
public void findPet(@PathVariable String petId) {
    // ...
}
~~~

# Handler Method

`@RequestMapping` 处理器方法有一个灵活的签名，可以从一系列支持的 controller 方法参数和返回值中选择。

## 参数

可以作为Handler Method参数的类型和可以注释Handler Method方法参数的注解如下表:

| 方法参数                                                     | 说明                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `WebRequest`, `NativeWebRequest`                             | SpringMVC提供的对request参数、request、session属性的通用访问，是对ServletAPI的封装 |
| `ServletRequest`, <br />`ServletResponse`                    | 选择任意特定的 request 或 response 类型，如:`HttpServletRequest `, 或Spring提供的`MultipartRequest`, `MultipartHttpServletRequest`等 |
| `HttpSession`、`PushBuilder`、`Principal`、`HttpMethod`、`Locale`、`TimeZone`、`ZoneId`、`InputStream`、`Reader`、`OutputStream`、`Writer` | Servlet API暴露的对象和SpringMVC提供的对象                   |
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

## 返回值

@RequestMapping注释的方法返回值同样有以下两种：

* 可以使用特定类型的参数
* 可以使用注解注释方法以声明特殊的返回值

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

用该注解注释将前端的请求参数(请求参数或者form表单中的参数，即可以通过`ServletRequest.getParameter()`获取的请求参数)绑定到处理器方法的方法参数。

例如：

~~~java
@Controller
@RequestMapping("/pets")
public class EditPetForm {

    @GetMapping
    public String setupForm(@RequestParam("petId") int petId, Model model) {
        Pet pet = this.clinic.loadPet(petId);
        model.addAttribute("pet", pet);
        return "petForm";
    }
    // ...
}
~~~

如果目标方法参数类型不是 `String`，那么将自动进行类型转换。

使用该注解的方法参数是必须的，如果请求里没有这个参数，则会报错400 Bad Request。可以通过将`@RequestParam` 注解的 `required` 属性设置为 `false` 或者用 `java.util.Optional` 包装器声明该参数来指定方法参数是可选的。

可以注释在类型为Array或者List的参数上，可以为同一个参数名解析多个参数值:

~~~java
// url: /requestParam/demo1?list=1,2,3,4,5
@RequestMapping("/requestParam/demo1")
public void requestParam(@RequestParam("list") List<String> strings){
    System.out.println(strings.size());
}
~~~

可以注释在`MultipartFile`上，解析前端上传的文件：

~~~java
@PostMapping("/form")
public String handleFormUpload(@RequestParam("name") String name,
        @RequestParam("file") MultipartFile file) {

    if (!file.isEmpty()) {
        byte[] bytes = file.getBytes();
        // store the bytes somewhere
        return "redirect:uploadSuccess";
    }
    return "redirect:uploadFailure";
}
~~~



如果 `@RequestParam` 声明在`Map<String, String> `或者` MultiValueMap<String, String> `上,并且注解没有指定参数名，那么参数将接受所有的request域中的键值对:

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

使用`HttpMessageConverter`将  response对象的body 字符串反序列化为注释的参数类型:

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
public void requestPart(@RequestPart("user") User user,@RequestPart("file") MultipartFile file){
    System.out.println(user);
    System.out.println(file.getSize());
}
~~~

**注意:**`@RequestParam`也同样支持`multipart/form-data`的请求，它们之间的区别是:

`@RequestParam`使用`Converter`进行类型转换，只能转换简单类型

`@RequestPart`使用`HttpMessageConverter`进行类型转换，支持转换复杂的参数类型

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

该注解将请求头与处理器方法参数绑定。

考虑请求有如下header：

~~~http
Host                    localhost:8080
Accept                  text/html,application/xhtml+xml,application/xml;q=0.9
Accept-Language         fr,en-gb;q=0.7,en;q=0.3
Accept-Encoding         gzip,deflate
Accept-Charset          ISO-8859-1,utf-8;q=0.7,*;q=0.7
Keep-Alive              300
~~~

下面的例子获取 `Accept-Encoding` 和 `Keep-Alive` header 的值：

~~~java
@GetMapping("/demo")
public void handle(
        @RequestHeader("Accept-Encoding") String encoding, 
        @RequestHeader("Keep-Alive") long keepAlive) { 
    //...
}
~~~

如果目标方法参数类型不是 `String`，那么将自动进行类型转换。

当注释的参数类型为 `Map<String, String>`、`MultiValueMap<String, String>`、 `HttpHeaders`时，所有的header将会被绑定到参数Map中。如:

~~~java
@GetMapping("/demo")
public void handle(@RequestHeader HttpHeaders headers){
    for (Map.Entry<String, List<String>> entry :headers.entrySet()){
        System.out.println(entry.getKey()+"---"+entry.getValue());
    }
}
~~~

支持将逗号分隔的字符串转换为数组或字符串的集合或类型转换系统已知的其他类型。例如，用 `@RequestHeader("Accept")` 注解的方法参数可以是 `String` 类型，也可以是 `String[]` 或 `List<String>`。



## `@CookieValue`

用以访问HTTP请求的Cookie

~~~java
@GetMapping("/demo")
public void handle(@CookieValue("JSESSIONID") String cookie) { 
    //...
}
~~~

如果目标方法参数类型不是 `String`，那么将自动进行类型转换。

所以可以直接绑定`Cookie`类型的参数:

~~~java
@GetMapping("/demo")
public void handle(@CookieValue("JSESSIONID") Cookie cookie) { 
    //...
}
~~~

## `@ModelAttribute`

`@ModelAttribute`可以注释在方法参数和方法体上

### 注释在方法参数上

`@ModelAttribute`注释在方法入参时，可以用来访问model中的一个属性对象。

如果model中不存在该属性对象，会先让它实例化再访问。实例化对象时，这个对象的属性值会被Servlet请求参数的值填充，其中属性字段名称和参数的名称相匹配。这就是**数据绑定**。这样就不需要解析单个表单字段了。

~~~java
@PostMapping("/owners/{ownerId}/pets/{petId}/edit")
public String processSubmit(@ModelAttribute Pet pet) {
    // method logic...
}
~~~

`@ModelAttribute`的具体工作流程如：

* 获取model attribute 对象类型实例，这个实例通过以下几种方式获取：

    * 从Model中获取实例，model 属性名由`@ModelAttribute`的`value/name`值指定，如果不指定，则默认为参数名。如上例中，model属性名为pet。

    * 如果类级别的 `@SessionAttributes` 注解中有列 model attribute ，那么从HTTP session 中获取实例
    * 通过 `Converter` 获得实例， model attribute 名称与请求值的名称相匹配，如路径变量或请求参数
    * 通过默认构造函数进行实例化获取实例
    * 通过 “primary constructor” 进行实例化，参数与Servlet请求参数相匹配。参数名称是通过JavaBeans的 `@ConstructorProperties` 或通过字节码中的运行时参数名称确定的。


* 获取model attribute 实例后，则进行数据绑定:`WebDataBinder` 将Servlet请求参数名（查询参数和表单字段）与model attribute实例上的字段名相匹配。必要时会应用类型转换填充字段。
* 最后将参数绑定后的参数实例也添加到Model中暴露给视图

**使用`Converter<String,T>`来提供参数对象实例**：

当model attribute名称(`@ModelAttribute`的value值或者所在的参数名)与请求值(如路径变量或者请求参数)名称相匹配时,并且有一个从`String`到model attribute 类型的`Converter`时，就可以通过类型转换来提供model attribute实例。

例如model attribute 名是 `account`，与URI路径变量 `account` 相匹配，并且有一个注册的 `Converter<String, Account>` 可以从数据存储中加载 `Account`：

~~~java
@PutMapping("/accounts/{account}")
public String save(@ModelAttribute("account") Account account) {
    // ...
}
~~~

参数绑定可能会出现异常，会抛出一个 `BindException`，为了在处理器方法内部检查这个异常，可以在使用`@ModelAttribute`参数同时使用一个`BindingResult`类型的参数，在方法内部访问绑定结果:

~~~java
@PostMapping("/owners/{ownerId}/pets/{petId}/edit")
public String processSubmit(@ModelAttribute("pet") Pet pet, BindingResult result) {
    if (result.hasErrors()) {
        return "petForm";
    }
    // ...
}
~~~

在某些情况下，可能需要只访问一个model attribute而不进行数据绑定。此时可以将`Model`注入处理器方法直接访问它，或者可以指定`@ModelAttribute`注解的属性`binding=false`：

~~~java
@ModelAttribute
public AccountForm setUpForm() {
    return new AccountForm();
}

@ModelAttribute
public Account findAccount(@PathVariable String accountId) {
    return accountRepository.findOne(accountId);
}

@PostMapping("update")
public String update(@Valid AccountForm form, BindingResult result,
        @ModelAttribute(binding=false) Account account) {
    // ...
}
~~~

可以使用 `jakarta.validation.Valid` 注解或Spring的 `@Validated` 注解在数据绑定后进行验证：

~~~java
@PostMapping("/owners/{ownerId}/pets/{petId}/edit")
public String processSubmit(@Valid @ModelAttribute("pet") Pet pet, BindingResult result) {
    if (result.hasErrors()) {
        return "petForm";
    }
    // ...
}
~~~



**注意:**使用`@ModelAttribute`是可选的，默认请求下，任何不是简单类型( 由`BeanUtils#isSimpleProperty`定义)的参数，并且该参数没有被其他参数处理器处理过，那么该参数就会被视为被`@ModelAttribute`注解注释了。

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

`@SessionAttributes`用于在请求之间的HTTP Servlet Session中存储model attributes。这是一个类型级别的注解，声明指定的Model属性会被存储在HTTP Session中，以便在多个请求之间共享数据。

Model属性被添加到Session中后会一直存在，除非有处理器方法调用`SessionStatus.setComplete()`方法来清除存储。

例如：

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

示例中`/add`处理器会将请求的`User`实例放入`Model`中，因为`@SessionAttributes`声明，这个`User`实例也会同时被存储到Session域中。

当`/delete`请求发起后，session中的`User`实例才会被清除。

## ` @SessionAttribute`

从Servlet的session域中获取已经存在的属性(在之前的请求，或者`Filter`和`HandlerIntercepter`通过`HttpSession.setAttribute()`设置的)

~~~java
@GetMapping("/sessionAttribute/demo")
public void sessionAttribute(@SessionAttribute("user") User user){
    System.out.println(user);
}
~~~

## `@RequestAttribute`

从Servlet的request域中获取提前创建的已经存在的属性(例如在`Filter`和`HandlerIntercepter`中通过`ServletRequest.setAttribute()`)

~~~java
@GetMapping("/requestAttribute/demo")
public void requestAttribute(@RequestAttribute("user") User user){
    System.out.println(user);
}
~~~

## `@ResponseBody`

注解在方法体上，表示会通过`HttpMessageConverter`将返回对象序列化为指定指定格式后写入到response对象的body中，通常用来返回json或者XML。

使用此注解之后不会再走视图处理器，而是直接将数据写入到输入流中，它的效果等同于通过response对象输出指定格式的数据。

```java
@RequestMapping("/getUser")
@ResponseBody
public User user(){
    return new User("test","test");
}
```

上面代码等效于：

~~~java
@RequestMapping("/getUser")
@ResponseBody
public User user(User user, HttpServletResponse response){
    response.getWriter.write(JSONObject.fromObject(user).toString());
}
~~~

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
    public void redirect( @PathVariable String param,HttpServletRequest request){
        System.out.println("URL:"+ request.getRequestURL()+"?"+request.getQueryString());
    }
}
~~~

例如对于上面的`@Controller`，访问`/redirect`会在`/target`输出以下内容：

~~~http
URL:http://127.0.0.1:8080/spring/target/test?number=123
~~~

可以看到在源处理器中作为model的`number`属性重定向后变成了查询参数暴露到了URL中

对于一些敏感数据，最好不要暴露到URL上。此时可以在源处理器上使用`Model`的子接口:`RedirectAttributes`，通过`RedirectAttributes.addFlashAttribute()`添加的model属性不会在重定向中暴露到URL上：

~~~java
@Controller
public class RedirectController {
    @GetMapping("/redirect")
    public String demo(RedirectAttributes redirectAttributes){
        redirectAttributes.addAttribute("param","test");
        redirectAttributes.addFlashAttribute("number",123);
        return "redirect:/target/{param}";
    }

    @GetMapping("/target/{param}")
    public void redirect( @PathVariable String param,HttpServletRequest request,@ModelAttribute("number") String number){
        System.out.println("URL:"+ request.getRequestURL()+"?"+request.getQueryString());
        System.out.println("param="+param+";number=" + number);
    }
}
~~~

此时访问的输出为：

~~~http
URL:http://127.0.0.1:8080/spring/target/test;jsessionid=10C5153B017B24FA4E3E6685D423A171?null
param=test;number=123
~~~

`addFlashAttribute()`添加的属性需要通过`@ModelAttribute`获取

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



# Controller的公共处理

## `@InitBinder`

SpringMVC可以使用`WebDataBinder`完成数据绑定和类型转换，其在Web请求中的下面场合发挥作用：

- 将请求参数（即表单或查询数据）绑定到一个 model 对象。
- 将基于字符串的请求值（如请求参数、路径变量、header、cookies等）转换为 controller 方法参数的目标类型。
- 在渲染HTML表单时，将 model 对象值格式化为 `String` 值。

可以用`@InitBinder`访问 `WebDataBinder`，在controller接受请求前，完成一些初始化操作。

`@InitBinder`通常注释在一个`void`返回值、有一个`WebDataBinder`参数的方法，例如：

~~~java
@Controller
public class FormController {

    @InitBinder
    public void initBinder(WebDataBinder binder) {
        SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd");
        dateFormat.setLenient(false);
        binder.registerCustomEditor(Date.class, new CustomDateEditor(dateFormat, false));
    }

    // ...
}


~~~

##  `@ExceptionHandler`

可以在`@Controller`类中定义一个`@ExceptionHandler`方法用来处理来自controller方法的异常，例如：

~~~java
@Controller
public class SimpleController {

    // ...
    @ExceptionHandler
    public ResponseEntity<String> handle(IOException ex) {
        // ...
    }
}
~~~

我们称`@ExceptionHandler`注释的方法为异常处理方法。示例中的异常处理方法将拦截`IOException`类型的异常并处理。

注解声明可以缩小要匹配的异常类型，例如：

~~~java
@ExceptionHandler({FileSystemException.class, RemoteException.class})
public ResponseEntity<String> handle(IOException ex) {
    // ...
}
~~~

或者：

~~~java
@ExceptionHandler({FileSystemException.class, RemoteException.class})
public ResponseEntity<String> handle(Exception ex) {
    // ...
}
~~~

### 支持的参数

`@ExceptionHandler` 方法支持以下参数：

| Method 参数                                                  | 说明                                                         |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| 异常类型                                                     | 用于访问抛出的异常。                                         |
| `HandlerMethod`                                              | 用于访问引发异常的 controller 方法。                         |
| `WebRequest`, `NativeWebRequest`                             | 对 request parameter 以及 request 和 session 属性的通用访问，无需直接使用Servlet API。 |
| `jakarta.servlet.ServletRequest`, `jakarta.servlet.ServletResponse` | 选择任何特定的请求或响应类型（例如，`ServletRequest` 或 `HttpServletRequest` 或 Spring 的 `MultipartRequest` 或 `MultipartHttpServletRequest`）。 |
| `jakarta.servlet.http.HttpSession`                           | 执行一个 session 的存在。因此，这样的参数永远不会是 `null` 的。 请注意，session 访问不是线程安全的。如果允许多个请求同时访问一个session，请考虑将 `RequestMappingHandlerAdapter` 实例的 `synchronizeOnSession` 标志设置为 `true`。 |
| `java.security.Principal`                                    | 目前认证的用户—如果知道的话，可能是一个特定的 `Principal` 实现类。 |
| `HttpMethod`                                                 | 请求的HTTP方法。                                             |
| `java.util.Locale`                                           | 当前的请求 locale，由最具体的 `LocaleResolver` 决定—实际上就是配置的 `LocaleResolver` 或 `LocaleContextResolver`。 |
| `java.util.TimeZone`, `java.time.ZoneId`                     | 与当前请求相关的时区，由 `LocaleContextResolver` 决定。      |
| `java.io.OutputStream`, `java.io.Writer`                     | 用于访问原始 response body，如Servlet API所暴露的。          |
| `java.util.Map`, `org.springframework.ui.Model`, `org.springframework.ui.ModelMap` | 用于访问 model 的错误响应。总是空的。                        |
| `RedirectAttributes`                                         | 指定在重定向情况下使用的属性--（即附加到查询字符串中），以及临时存储到重定向后的请求中的Flash attributes。 |
| `@SessionAttribute`                                          | 用于访问任何 session attribute，与由于类级 `@SessionAttributes` 声明而存储在 session 中的 model attributes 不同。 |
| `@RequestAttribute`                                          | 用于访问 request attributes。                                |

### 支持的返回值

`@ExceptionHandler` 方法支持以下返回值：

| 返回值                                          | 说明                                                         |
| :---------------------------------------------- | :----------------------------------------------------------- |
| `@ResponseBody`                                 | 返回值通过 `HttpMessageConverter` 实例进行转换并写入响应中。 |
| `HttpEntity<B>`, `ResponseEntity<B>`            | 返回值指定通过 `HttpMessageConverter` 实例转换完整的响应（包括HTTP header 和 body）并写入响应中。 |
| `ErrorResponse`                                 | 要渲染一个RFC 7807错误响应，并在 body 中提供详细信息         |
| `ProblemDetail`                                 | 要渲染一个RFC 7807错误响应，并在 body 中提供详细信息         |
| `String`                                        | 用 `ViewResolver` 实现解析的视图名称，并与隐式 model 一起使用—通过命令对象和 `@ModelAttribute` 方法确定。handler method 也可以通过声明一个 `Model` 参数（如前所述）以编程方式丰富 model。 |
| `View`                                          | 一个与隐含 model 一起用于渲染的 `View` 实例—通过命令对象和 `@ModelAttribute` 方法确定。handler method 也可以通过声明一个 `Model` 参数（前面已经描述过），以编程方式丰富 model。 |
| `java.util.Map`, `org.springframework.ui.Model` | 要添加到隐式 model 中的属性，视图名称通过 `RequestToViewNameTranslator` 隐式确定。 |
| `@ModelAttribute`                               | 一个将被添加到 model 中的属性，其视图名称通过 `RequestToViewNameTranslator` 隐式确定。注意，`@ModelAttribute` 是可选的。见本表末尾的 "任何其他返回值"。 |
| `ModelAndView` object                           | 要使用的 view 和 model attributes，以及可选的响应状态。      |
| `void`                                          | 如果一个方法有一个 `ServletResponse` 或者 `OutputStream` 参数，或者 `@ResponseStatus` 注解，那么这个方法的返回类型为 `void` （或者返回值为 `null`），被认为是完全处理了响应。如果 controller 进行了积极的 `ETag` 或 `lastModified` 时间戳检查，也是如此。如果以上都不是 true， `void` 的返回类型也可以表示REST控制器的 “no response body” 或HTML controller 的默认视图名称选择。 |
| 任何其他返回值                                  | 如果一个返回值没有与上述任何一个匹配，并且不是一个简单的类型（由 `BeanUtils#isSimpleProperty`确定），默认情况下，它被当作一个要添加到模型中的 model attribute。如果它是一个简单的类型，它仍然是未解析的。 |

## ` @ControllerAdvice `

`@ExceptionHandler`, `@InitBinder`, 和 `@ModelAttribute` 方法只在他们所在的`@Controller`类或子类中生效。

如果需要他们在所有指定的controller中生效，那么可以将他们声明在`@ControllerAdvice`中。从名字就可以看出，这是一个针对controller的AOP切面。

`@ControllerAdvice`有元注解 `@Component`，所以 可以通过 组件扫描 注册为Spring Bean。

`@RestControllerAdvice` 有元注解 `@ControllerAdvice` 和 `@ResponseBody` ，意味着`@RestControllerAdvice`的  `@ExceptionHandler` 方法的返回值将通过响应体 message 转换，而不是通过视图。

在启动时，`RequestMappingHandlerMapping` 和 `ExceptionHandlerExceptionResolver` 检测 controller advice bean，并在运行时应用它们。来自 `@ControllerAdvice` 的全局 `@ExceptionHandler` 方法，会在来自 `@Controller` 的局部方法之后被应用。相反地，全局的 `@ModelAttribute` 和 `@InitBinder` 方法会在本地方法之前应用。

`@ControllerAdvice` 注解有一些属性，可以让你缩小它们适用的 controller 和 handler 的范围,例如：

~~~java
// Target all Controllers annotated with @RestController
@ControllerAdvice(annotations = RestController.class)
public class ExampleAdvice1 {}

// Target all Controllers within specific packages
@ControllerAdvice("org.example.controllers")
public class ExampleAdvice2 {}

// Target all Controllers assignable to specific classes
@ControllerAdvice(assignableTypes = {ControllerInterface.class, AbstractController.class})
public class ExampleAdvice3 {}
~~~

# 异步请求

SpringMVC支持支持各种形式的异步请求。

## 单一异步返回

### `DeferredResult`

如果Handler Method的返回值是`DeferredResult`，那么在请求在没有超时或者`DeferredResult`对象还没有调用`setResult()`进行响应前，请求不会获得响应，但是Servlet容器线程可以提前结束。

这样`DeferredResult`允许另起线程来进行结果处理(即这种操作提升了服务短时间的吞吐能力)，并`setResult()`，如此以来这个请求不会占用服务连接池太久，如果超时或设置`setResult()`，会立即响应。

一个使用`DeferredResult`，在新线程中处理请求，进行异步响应的示例：

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

最终的结果在`MyThread.run()`方法中处理：

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

生产中通常请求线程池，而不是直接创建线程。

### `Callable`

作用和`DeferredResult`类似，但是`Callable`指定的异步线程的任务是通过SpringMVC配置(AsyncTaskExecutor配置)运行的，不需要手动创建线程。

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

### SpringMVC处理异步响应

`Servlet`容器提供异步响应机制：

- `ServletRequest` 可以通过调用 `request.startAsync()` 进入异步模式。这样Servlet（以及任何 filter）可以退出方法，而不会触发请求的响应；这样可以在其他线程中处理请求。
- 对 `request.startAsync()` 的调用返回 `AsyncContext`，你可以用它来进一步控制异步处理。例如，它提供了 `dispatch` 方法，它类似于 Servlet API中的forward，只是它允许应用程序在 Servlet 容器线程上恢复请求处理。
- `ServletRequest` 提供了对当前 `DispatcherType` 的访问，你可以用它来区分处理初始请求、异步调度、转发和其他调度器类型。

`DeferredResult` 的处理工作如下：

- 控制器返回 `DeferredResult`，并将其保存在一些可以访问的内存队列或列表中。
- Spring MVC 调用 `request.startAsync()`。
- 同时，`DispatcherServlet` 和所有配置的 filter 退出请求处理线程，但响应仍然开放。
- 应用程序从某个线程设置 `DeferredResult`，Spring MVC将请求调度回Servlet容器。
- `DispatcherServlet` 被再次调用，并以异步产生的返回值恢复处理。

`Callable` 处理的工作方式如下：

- 控制器返回一个 `Callable`。
- Spring MVC 调用 `request.startAsync()`，并将 `Callable` 提交给 `TaskExecutor`，在一个单独的线程中进行处理。
- 同时，`DispatcherServlet` 和所有的 filter 退出 Servlet 容器线程，但响应仍然开放。
- 最终，`Callable` 产生了一个结果，Spring MVC将请求调度回Servlet容器以完成处理。
- `DispatcherServlet` 被再次调用，并以来自 `Callable` 的异步产生的返回值继续进行处理。

###  `AsyncHandlerInterceptor`

`AsyncHandlerInterceptor`扩展了`HandlerInterceptor`拦截器，相对于普通的拦截器，它提供了一个异步请求中的回调时机：在请求线程已经结束，异步线程开始时，



## 异步响应流

 `DeferredResult` 和 `Callable` 可以实现一个单一的异步返回值。如果想产生多个异步值，并让这些值写入响应中，可以使用异步响应流。

### `ResponseBodyEmitter`

使用`ResponseBodyEmitter`可以异步返回多个值。产生一个对象流，其中每个对象都被 `HttpMessageConverter `序列化并写入响应中。

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

### `SseEmitter`

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

### `StreamingResponseBody`

如果不需要消息转化，可以通过`StreamingResponseBody`来直接流向响应的 `OutputStream` 

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

## 异步配置

异步请求处理功能必须在Servlet容器级别启用。MVC配置也为异步请求暴露了几个选项。

### 配置Servlet容器

Filter 和 Servlet 的声明有一个 `asyncSupported` 标志，需要设置为 `true` 以启用异步请求处理。此外，`Filter` 映射（mappings）的`jakarta.servlet.DispatchType`应该设置 `ASYNC` 

#### web.xml配置

对于`web.xml`配置，需要将`<servlet>`和`<filter>`的子标签`<async-supported>`设置为`true`，并且将`<filter-mapping>`的子标签`<dispatcher>`设置为`true`，例如：

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
<filter-mapping>
    <filter-name>encodingFilter</filter-name>
    <servlet-name>dispatcherServlet</servlet-name>
    <dispatcher>ASYNC</dispatcher>
</filter-mapping>
~~~

#### Java配置

我们使用`WebApplicationInitializer`接口来完成java配置，只要使用它的一个实现`AbstractAnnotationConfigDispatcherServletInitializer`就会自动完成servlet容器的异步配置：

~~~java
public class MyWebAppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {

    @Override
    protected Class<?>[] getRootConfigClasses() {
        return null;
    }

    @Override
    protected Class<?>[] getServletConfigClasses() {
        return new Class<?>[] { MyWebConfig.class };
    }

    @Override
    protected String[] getServletMappings() {
        return new String[] { "/" };
    }
}
~~~

### 配置SpringMVC

MVC配置暴露了以下与异步请求处理有关的选项：

- Java 配置：在 `WebMvcConfigurer` 上使用 `configureAsyncSupport `回调。
- XML 命名空间: 使用 `<mvc:annotation-driven>` 下的 `<async-support>` 元素。

你可以配置以下内容：

- 异步请求的默认超时值，如果没有设置，则取决于底层Servlet容器。
- `AsyncTaskExecutor`，用于响应式（Reactive）类型流时的阻塞写入，以及执行从 controller 方法返回的 `Callable` 实例。如果使用响应式（Reactive）类型的流媒体或有返回 `Callable` 的 controller 方法，那么强烈建议配置这个属性，因为在默认情况下，它是一个 `SimpleAsyncTaskExecutor`。
- `DeferredResultProcessingInterceptor` 的实现和 `CallableProcessingInterceptor` 的实现。

# CORS

## CORS原理

出于最基本的安全考虑，浏览器的通常会采取同源策略。

文档的源就是文档URL的协议、主机和端口。必须协议、主机和端口都相同的文档URL，文档才算是同源。

同源策略指的是脚本只能向所在页面文档的源发起请求。

如果想要跨源发起请求，就必须使用跨源资源共享(Cross-Origin Resource Sharing,CORS)。

CORS扩展了HTTP协议，增加了新的`Origin`请求头和`Access-Control-Allow-Origin`响应头。

前端如果想要发起跨源请求，必须在请求头中添加`Origin`字段，声明本次请求来自哪个源。而跨源的目标网站会添加`Access-Control-Allow-Origin`响应头来声明允许跨源请求的范围。

浏览器将CORS请求分成两类：简单请求（simple request）和非简单请求（not-so-simple request）。只要同时满足以下两大条件，就属于简单请求。

* 请求方法是HEAD，GET，POST三种方法之一

* HTTP的头信息不超出以下几种字段：

  - Accept

  - Accept-Language

  - Content-Language

  - Last-Event-ID

  - Content-Type：只限于三个值application/x-www-form-urlencoded、multipart/form-data、text/plain

凡是不同时满足上面两个条件，就属于非简单请求。浏览器对这两种请求的处理，是不一样的。

### 简单请求

对于简单请求，浏览器直接发出CORS请求。在请求头中声明`Origin`字段表示本次请求的源，例如：

~~~http
GET /cors HTTP/1.1
Origin: http://api.bob.com
Host: api.alice.com
Accept-Language: en-US
Connection: keep-alive
User-Agent: Mozilla/5.0...
~~~

不论本次请求的源是否在许可的范围内。目标服务器都会返回一个正常的HTTP响应。浏览器会获取响应的`Access-Control-Allow-Origin`响应头判断本次请求是否在许可范围内，如果不再，浏览器会抛出一个跨源请求异常。

如果Origin指定的域名在许可范围内，服务器返回的响应，会多出几个头信息字段。例如：

~~~http
Access-Control-Allow-Origin: http://api.bob.com
Access-Control-Allow-Credentials: true
Access-Control-Expose-Headers: FooBar
Content-Type: text/html; charset=utf-8
~~~

面的头信息之中，有三个与CORS请求相关的字段，都以Access-Control-开头。

* `Access-Control-Allow-Origin`：该字段是必须的。它的值要么是请求时Origin字段的值，要么是一个*，表示接受任意域名的请求。

* `Access-Control-Allow-Credentials`：该字段可选。它的值是一个布尔值，表示是否允许发送Cookie。默认情况下，Cookie不包括在CORS请求之中。设为true，即表示服务器明确许可，Cookie可以包含在请求中，一起发给服务器。这个值也只能设为true，如果服务器不要浏览器发送Cookie，删除该字段即可。

* `Access-Control-Expose-Headers`：该字段可选。CORS请求时，XMLHttpRequest对象的getResponseHeader()方法只能拿到6个基本字段：Cache-Control、Content-Language、Content-Type、Expires、Last-Modified、Pragma。如果想拿到其他字段，就必须在Access-Control-Expose-Headers里面指定。上面的例子指定，getResponseHeader('FooBar')可以返回FooBar字段的值。

### 非简单请求

非简单请求的CORS请求，会在正式通信之前，增加一次HTTP查询请求，称为"预检"请求

浏览器会预先询问一次服务器，只有当前网页所在的域名是否在服务器的许可名单之中，以及可以使用哪些HTTP动词和头信息字段。只有得到肯定答复，浏览器才会发出正式的XMLHttpRequest请求，否则就报错。

浏览器发起的预检请求的示例如下：

~~~http
OPTIONS /cors HTTP/1.1
Origin: http://api.bob.com
Access-Control-Request-Method: PUT
Access-Control-Request-Headers: X-Custom-Header
Host: api.alice.com
Accept-Language: en-US
Connection: keep-alive
User-Agent: Mozilla/5.0...
~~~

"预检"请求用的请求方法是OPTIONS，表示这个请求是用来询问的。头信息里面，关键字段是Origin，表示请求来自哪个源。

服务器收到预检请求后，前如果确认跨源请求，就可以做出回应,例如：

~~~http
HTTP/1.1 200 OK
Date: Mon, 01 Dec 2008 01:15:39 GMT
Server: Apache/2.0.61 (Unix)
Access-Control-Allow-Origin: http://api.bob.com
Access-Control-Allow-Methods: GET, POST, PUT
Access-Control-Allow-Headers: X-Custom-Header
Access-Control-Max-Age: 1728000
Content-Type: text/html; charset=utf-8
Content-Encoding: gzip
Content-Length: 0
Keep-Alive: timeout=2, max=100
Connection: Keep-Alive
Content-Type: text/plain
~~~

对预检请求的回应中与CORS相关的字段有：

* `Access-Control-Allow-Methods`该字段必需，它的值是逗号分隔的一个字符串，表明服务器支持的所有跨域请求的方法。注意，返回的是所有支持的方法，而不单是浏览器请求的那个方法。这是为了避免多次"预检"请求。

* `Access-Control-Allow-Headers`如果浏览器请求包括Access-Control-Request-Headers字段，则Access-Control-Allow-Headers字段是必需的。它也是一个逗号分隔的字符串，表明服务器支持的所有头信息字段，不限于浏览器在"预检"中请求的字段。

* `Access-Control-Allow-Credentials`该字段与简单请求时的含义相同。

* `Access-Control-Max-Age`该 字段可选，用来指定本次预检请求的有效期，单位为秒。上面结果中，有效期是20天（1728000秒），即允许缓存该条回应1728000秒（即20天），在此期间，不用发出另一条预检请求。

一旦服务器通过了"预检"请求，以后每次浏览器正常的CORS请求，就都跟简单请求一样，会有一个Origin头信息字段。服务器的回应，也都会有一个Access-Control-Allow-Origin头信息字段。

## SpringMVC处理CORS请求

Spring MVC `HandlerMapping` 实现提供了对CORS的内置支持。在成功地将一个请求映射到一个 handler 后， `HandlerMapping` 实现会检查给定请求和 handler 的CORS配置并采取进一步的行动。预检请求被直接处理，而简单和实际的CORS请求被拦截、验证，并设置必要的CORS响应头。

每个 `HandlerMapping` 都可以配置单独使用基于 URL pattern 的 `CorsConfiguration` 映射。在大多数情况下，应用程序使用MVC Java配置或XML命名空间来声明这种映射，这会让一个单一的全局映射被传递给所有 `HandlerMapping` 实例

也可以使用`@CrossOrigin`注解进行handler级别的CORS配置。

## `@CrossOrigin`

`@CrossOrigin` 注解允许注解的 controller 方法进行跨域请求，例如：

~~~java
@RestController
@RequestMapping("/account")
public class AccountController {

    @CrossOrigin
    @GetMapping("/{id}")
    public Account retrieve(@PathVariable Long id) {
        // ...
    }

    @DeleteMapping("/{id}")
    public void remove(@PathVariable Long id) {
        // ...
    }
}
~~~

默认情况下，`@CrossOrigin` ：

- 允许所有 origin。
- 允许所有 headers。
- 允许controller 方法所映射的所有HTTP方法。
- maxAge默认为30分钟

`allowCredentials` 默认不启用，因为这会暴露敏感的用户特定信息（如 cookies 和CSRF令牌）

也可以将`@CrossOrigin`注释在`@Controller`类上，这样`@Controller`类的所有方法都会继承这个`@CrossOrigin`,而方法上的`@CrossOrigin`会覆盖类上的配置。

~~~java
@CrossOrigin(maxAge = 3600)
@RestController
@RequestMapping("/account")
public class AccountController {

    @CrossOrigin("https://domain2.com")
    @GetMapping("/{id}")
    public Account retrieve(@PathVariable Long id) {
        // ...
    }

    @DeleteMapping("/{id}")
    public void remove(@PathVariable Long id) {
        // ...
    }
}
~~~

## 全局配置

使用MVC的Java配置或MVC的XML命名空间来配置全局的 CORS

默认情况下，全局配置会：

- 允许所有 origins。
- 允许所有 headers。
- 允许`GET`， `HEAD` 和 `POST` 方法。
- `maxAge` 被设置为30分钟

###  Java 配置

使用 `CorsRegistry` 回调启用 CORS：

~~~java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {

        registry.addMapping("/api/**")
            .allowedOrigins("https://domain2.com")
            .allowedMethods("PUT", "DELETE")
            .allowedHeaders("header1", "header2", "header3")
            .exposedHeaders("header1", "header2")
            .allowCredentials(true).maxAge(3600);

        // Add more mappings...
    }
}
~~~

### XML配置

可以使用 `<mvc:cors>` 元素启用 CORS：

~~~xml
<mvc:cors>
    <mvc:mapping path="/api/**"
        allowed-origins="https://domain1.com, https://domain2.com"
        allowed-methods="GET, PUT"
        allowed-headers="header1, header2, header3"
        exposed-headers="header1, header2" allow-credentials="true"
        max-age="123" />

    <mvc:mapping path="/resources/**"
        allowed-origins="https://domain1.com" />

</mvc:cors>
~~~



# 处理Handler Method参数

## 数据格式化

Spring的`org.springframework.format.annotation `包提供两个格式化注解：

*  `@DateTimeFormat` 用来格式化`Date`、`Calendar`、`Long`(时间戳)和`java.time`包下类型
*  `@NumberFormat` 用来格式化`Number`字段

处理器方法入参可以直接使用这两个格式化注解:

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

## 数据校验

MVC配置会默认配置一个`LocalValidatorFactoryBean`,只要在需要检验数据的参数注释`@Valid`注解，即可通知`DataBinder`在完成数据转化后使用`LocalValidator`进行数据校验，数据校验的结果可以通过声明` BindingResult`和`Errors`来访问

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



