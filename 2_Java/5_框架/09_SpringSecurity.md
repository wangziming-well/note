# SpringSecurity简介

是一种基于 Spring AOP 和 Servlet 过滤器的安全框架

它提供全面的安全性解决方案，同时在 Web 请求级和方法调用级处理身份认证和访问控制

它的底层实现为一条过滤器链

## 身份认证

在前后端分离中，前后端交互基于安全考虑，需要进行身份认证，常用的认证方式有：

* Session-Cookie验证
* Token验证
* OAuth开放授权

SpringSecurity的默认的身份认证是基于Token的

### Session-Cookie验证

认证流程：

![session认证流程](https://gitee.com/wangziming707/note-pic/raw/master/img/session%E8%AE%A4%E8%AF%81%E6%B5%81%E7%A8%8B.png)



存在的问题：

* 当客户访问量增加，服务端需要存储大量的session会话，服务端压力大
* 服务端为集群时，需要解决session跨服务器问题
    * 可以采用使用缓存服务器来保证共享
    * 第三方缓存来保存session

* 由于依赖cookie，所以存在CSRF安全问题

### Token 认证

认证流程：

![token认证流程](https://gitee.com/wangziming707/note-pic/raw/master/img/token%E8%AE%A4%E8%AF%81%E6%B5%81%E7%A8%8B.png)

token(令牌，访问资源的凭证) 认证是一种机制，具体的实现可以是一个随机的字符串，也可以是标准的jwt。

token需要查库验证token 是否有效，而JWT不用查库或者少查库，直接在服务端进行校验。

优点：

* 轻量级：JWT是非常轻量级的，传输的方式多样化，可以通过URL/POST参数/HTTP头部等方式传输。（一般放在 x-access-token里）
* 无状态/跨域认证：令牌包含所有用于标识用户的信息，这消除了对会话状态的需要。 如果我们使用负载平衡器，我们可以将用户传递给任何服务器，而不是绑定到我们登录的同一台服务器上。
* 可重用性/扩展性：我们可以有许多独立的服务器在多个平台和域上运行，并重复使用相同的令牌来验证用户。 构建与另一个应用程序共享权限的应用程序很容易。
* 安全性：无需担心跨站请求伪造（CSRF）攻击。

## RBAC

基于角色的访问控制（RBAC）是实施面向企业安全策略的一种有效的访问控制方式。

其基本思想是，对系统操作的各种权限不是直接授予具体的用户，而是在用户集合与权限集合之间建立一个角色集合。每一种角色对应一组相应的权限。一旦用户被分配了适当的角色后，该用户就拥有此角色的所有操作权限。



# SpringSecurity架构

## 过滤器链

SpringSecurity通过过滤器链实现身份认证

但这些过滤器并不直接处理用户的认证，也不直接处理用户的授权，

而是把它们交给了认证管理器（`AuthenticationManager`）和决策管理器（`AccessDecisionManager`）进行处理

![SpringSecurity过滤器链](https://gitee.com/wangziming707/note-pic/raw/master/img/SpringSecurity%E8%BF%87%E6%BB%A4%E5%99%A8%E9%93%BE.png)

**过滤器链中主要的几个过滤器及其作用:**

* `SecurityContextPersistenceFilter `

    主要是在认证 Filter链执行之前 维护`SecurityContextHolder `方便我们后续通过`SecurityContextHolder.getContext()`获取当前会话用户信息

* `UsernamePasswordAuthenticationFilter`

    进行用户认证操作，默认匹配URL为/login且必须为POST请求。

## 认证流程



![SpringSecurity认证时序图](https://gitee.com/wangziming707/note-pic/raw/master/img/SpringSecurity%E8%AE%A4%E8%AF%81%E6%97%B6%E5%BA%8F%E5%9B%BE.jpg)

* 用户的登录请求会被`UsernamePasswordAuthenticationFilter`拦截，过滤器会先将用户提交的username，password封装成`Authentication`接口对象，实现类通常为`UsernamePasswordAuthenticationToken`
* 过滤器将封装好的`Authentication`提交给`AuthenticationManager`进行认证

* 认证成功后， `AuthenticationManager `身份管理器返回一个被填充了信息的（包括上面提到的权限信息，身份信息，细节信息，但密码通常会被移除） Authentication 实例。





# WebSecurityConfigurerAdapter

可以通过自定义配置类实现WebSecurityConfigurerAdapter并重写它的configure方法来自定spring security的安全策略

它有下面3个重载的方法：

* `configure(HttpSecurity http)`
* `configure(WebSecurity web)`
* `configure(AuthenticationManagerBuilder auth)`

通过重写这些方法，并设置入参：`http web auth`进行自定义配置

这三个对象实现了链式编程的思想，他们大部分配置方法的返回值就是自己

## AuthenticationManagerBuilder

它的继承关系如下

~~~mermaid
classDiagram
direction BT
class AbstractConfiguredSecurityBuilder~O, B~
class AbstractSecurityBuilder~O~
class AuthenticationManagerBuilder
class ProviderManagerBuilder~B~ {
<<Interface>>

}
class SecurityBuilder~O~ {
<<Interface>>

}

AbstractConfiguredSecurityBuilder~O, B~  -->  AbstractSecurityBuilder~O~ 
AbstractConfiguredSecurityBuilder~O, B~  ..>  SecurityBuilder~O~ 
AbstractSecurityBuilder~O~  ..>  SecurityBuilder~O~ 
AuthenticationManagerBuilder  -->  AbstractConfiguredSecurityBuilder~O, B~ 
AuthenticationManagerBuilder  ..>  ProviderManagerBuilder~B~ 
ProviderManagerBuilder~B~  -->  SecurityBuilder~O~ 
~~~

`SecurityBuilder`是一个构建器，它构建的对象是`AuthenticationManager`认证管理器

所以`AuthenticationManagerBuilder`是配置认证管理的

### ` inMemoryAuthentication()`

在内存中认证，它返回一个`InMemoryUserDetailsManagerConfigurer`对象，允许我们在内存中配置用来认证的用户信息(`UserDetails`)

`InMemoryUserDetailsManagerConfigurer`继承关系如下：

* `UserDetailsManagerConfigurer`
    * `InMemoryUserDetailsManagerConfigurer`

`InMemoryUserDetailsManagerConfigurer`内部只有一个构造方法，它的所有方法由其父类`UserDetailsManagerConfigurer`实现

`UserDetailsManagerConfigurer`：

* 内部维护两个字段：

~~~java
//UserDetailsBuilder为UserDetailsManagerConfigurer内部类，用来构建UserDetails
//userBuilders内的UserDetailsBuilder会被执行生成UserDetails，然后添加到users中
private final List<UserDetailsBuilder> userBuilders = new ArrayList<>();

//users内的UserDetails最终会被加载到内存中，供身份验证
private final List<UserDetails> users = new ArrayList<>();
~~~

* 提供以下方法：

**注意：**C为继承该类的子类的类型，在这里就是InMemoryUserDetailsManagerConfigurer

~~~java
//通过UserDetails添加用户信息到内存
public final C withUser(UserDetails userDetails) {
    this.users.add(userDetails);
    return (C) this;
}
//通过User.UserBuilder添加用户信息到内存
public final C withUser(User.UserBuilder userBuilder) {
    this.users.add(userBuilder.build());
    return (C) this;
}

//通过username用户名添加用户信息到内存，
//返回一个userBuilder，后续需要继续操作userBuilder添加其他用户信息
public final UserDetailsBuilder withUser(String username) {
    UserDetailsBuilder userBuilder = new UserDetailsBuilder((C) this);
    userBuilder.username(username);
    this.userBuilders.add(userBuilder);
    return userBuilder;
}
~~~

* 提供一个内部类`UserDetailsBuilder`

~~~java
public final class UserDetailsBuilder {
	//本类几乎所有方法几乎都是封装的UserBuilder方法
    private UserBuilder user;
	//其外部类对象这里是InMemoryUserDetailsManagerConfigurer
    private final C builder;
	//只有一个构造函数，没有无参构造，也就是说创建该类必须传入持有它的类的实例
    private UserDetailsBuilder(C builder) {
        this.builder = builder;
    }

	//返回持有它的类的实例(InMemoryUserDetailsManagerConfigurer)
    //链式编程设计
    public C and() {
        return this.builder;
    }

	//初始化user并传入username,也就是说该方法必须是第一个被调用的
    private UserDetailsBuilder username(String username) {
        this.user = User.withUsername(username);
        return this;
    }

	//下面所有方法用来设置UserDetails的其他信息
    public UserDetailsBuilder password(String password)
        
    public UserDetailsBuilder roles(String... roles) 

    public UserDetailsBuilder authorities(GrantedAuthority... authorities) 

    public UserDetailsBuilder authorities(List<? extends GrantedAuthority> authorities) 

    public UserDetailsBuilder authorities(String... authorities) 

    public UserDetailsBuilder accountExpired(boolean accountExpired) 

    public UserDetailsBuilder accountLocked(boolean accountLocked) 

    public UserDetailsBuilder credentialsExpired(boolean credentialsExpired) 

    public UserDetailsBuilder disabled(boolean disabled) 

}
~~~



### `userDetailsService()`

方法签名如下

~~~java
public <T extends UserDetailsService> DaoAuthenticationConfigurer<AuthenticationManagerBuilder, T> userDetailsService(
			T userDetailsService)
~~~

基于传入的自定义`UserDetailsService`添加身份验证

返回一个`DaoAuthenticationConfigurer`来允许自定义身份验证



## HttpSecurity

配置请求访问策略

### 绑定权限

配置请求路径的权限认证

~~~java
// 配置放行的路径
http.authorizeHttpRequests()
    .antMatchers("/login/toLogin","/login/doLogin")
    .permitAll();
//绑定权限 
http.authorizeHttpRequests()
                .antMatchers("/query")
                .hasAnyAuthority("sys:query")
//配置除放行路径外，全部路径进行权限认证
http.authorizeHttpRequests()
    .anyRequest()
    .authenticated();
~~~

**注意：**`anyRequest()`必须定义在`antMatchers()`后

### 登录登出

配置登录页面和登录请求

~~~java
http.formLogin()
    //配置登录页面
    .loginPage("/login/toLogin")
    //配置登录请求路径
    .loginProcessingUrl("/login/doLogin")
    //配置用户名参数
    .usernameParameter("username")
    //配置用户密码参数
    .passwordParameter("password")
    //配置登录成功跳转页面
    .successForwardUrl("/index")
   	//配置登录失败跳转页面
    .failureForwardUrl("/login/toLogin");
~~~

配置登出设置

~~~java
http.logout()
    .logoutUrl("/logout")
    .logoutSuccessUrl("/login/toLogin");
~~~

### 添加过滤器

向SringSecurity过滤器链插入自定义过滤器

~~~java
http.addFilterBefore(myFilter, UsernamePasswordAuthenticationFilter.class);
~~~

### 设置禁止跨站域请求伪造

~~~java
http.csrf().disable();
~~~

### 记住我

~~~java
http.rememberMe()
    //前端登录表单控制开启记住我的字段
    .rememberMeParameter("remember-me")
    .tokenRepository(jdbcTokenRepository());
~~~

~~~java
@Bean
public PersistentTokenRepository jdbcTokenRepository(){
    JdbcTokenRepositoryImpl jdbcTokenRepository = new JdbcTokenRepositoryImpl();
    //dataSource直接注入即可
    jdbcTokenRepository.setDataSource(dataSource);
    jdbcTokenRepository.setCreateTableOnStartup(true);
    return jdbcTokenRepository;
}
~~~

### 自定义处理器

用于前后端分离，传输json数据

~~~java
//拒绝访问处理器
http.exceptionHandling()
    .accessDeniedHandler(accessDeniedHandler);
//登录成功失败处理器
http.formLogin()
    .successHandler(authenticationSuccessHandler)
    .failureHandler(authenticationFailureHandler);
~~~





## WebSecurity

是用来创建核心过滤器FilterChainProxy实例

### 设置资源过滤

~~~java
 web.ignoring().mvcMatchers("/resources/**");
~~~

# 相关类和接口

## UserDetails

为SpringSecurity访问用户核心信息提供标准接口

如果想要让自定义的JavaBean接入SpringSecurity，必须实现该接口

~~~java
public interface UserDetails extends Serializable {
	
    //获取用户权限
	Collection<? extends GrantedAuthority> getAuthorities();
	//获取用户密码
	String getPassword();
	//获取用户名
	String getUsername();
	//判断账户是否未过期
	boolean isAccountNonExpired();
	//判断账户是否未锁定
	boolean isAccountNonLocked();
	//判断账户凭证未过期
	boolean isCredentialsNonExpired();
	//判断账户是否可用
	boolean isEnabled();

}
~~~

## UserDetailsService

想要让自定义的获取用户信息服务类接入SpringSecurity

必须实现该接口

~~~java
public interface UserDetailsService {

	UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;

}

~~~

## Authentication

SpringSecurity定义的身份验证信息接口

~~~java
public interface Authentication extends Principal, Serializable {
	//获取该身份的权限
	Collection<? extends GrantedAuthority> getAuthorities();

	//获取凭证，通常是密码
	Object getCredentials();

	//获取其他信息，通常是IP地址等
	Object getDetails();

	//被认证的主体的身份。
	Object getPrincipal();

	//已通过身份验证
	boolean isAuthenticated();

	//设置权限
	void setAuthenticated(boolean isAuthenticated) throws IllegalArgumentException;

}
~~~



# SpringSecurity注解

在Controller上添加注解：

* `@PreAuthorize ` 在方法调用前进行权限检查

* `@PostAuthorize `在方法调用后进行权限检查

可以替代WebSecurityConfig中的

`http.authorizeHttpRequests().antMatchers() .hasAnyAuthority()`

配置，给url配置权限

上面的两个注解如果要使用的话

必须在SpringbootApplication启动类或者WebSecurityConfig类上添加下面注解
`@EnableGlobalMethodSecurity(prePostEnabled = true)`
如果只使用PreAuthorize 就只用开启prePostEnabled = true





# JWT

Json web token (JWT), 是为了在网络应用环境间传递声明而执行的一种基于JSON的开放标准

JWT的声明一般被用做身份认证令牌，也可以增加一些额外的其它业务逻辑所必须的声明信息，该token也可直接被用于认证，也可被加密。

与普通的令牌相比，JWT令牌签发时不需要将其保存起来

当用户带着令牌请求时，只需要根据密钥算法解码即可，能够解析成功则验证通过

## JWT结构

JWT的官网：https://jwt.io/ 可以对JWT解析和加密

他的格式如下：

### header

JWT头的字段时固定的

* alg:JWT采用的加密算法
* typ:数据类型JWT

~~~java
{
    "alg": "HS256",
    "typ": "JWT"
}
~~~

### payload

载荷，可以自定义字段属性在这部分，官方也提供了几个标准字段

* iss (issuer)：签发人
* exp (expiration time)：过期时间
* sub (subject)：主题
* aud (audience)：受众
* nbf (Not Before)：生效时间
* iat (Issued At)：签发时间
* jti (JWT ID)：编号

~~~json
payload:{
    "sub": "1234567890",
    "name": "John Doe",
    "iat": 1516239022
}
~~~

### verify signature

对前面两部分的签名，保证数据的安全性，防止数据篡改

~~~json
HMACSHA256(
 	base64UrlEncode(header) + "." +
  	base64UrlEncode(payload),
	your-256-bit-secret
)
~~~

指定一个密钥（secret）。然后使用 Header 里面指定的签名算法（默认是 HMAC SHA256），按照上面的公式产生签名。

## JWT特征

* 默认是不加密的，不要将敏感数据写入载荷中
* 可用于认证和交换信息，降低数据库压力
* 是无状态的，一旦 JWT 签发了，在到期之前就会始终有效

* 为了减少盗用，JWT 不应该使用 HTTP 协议明码传输，要使用 HTTPS 协议传输。

## Java使用JWT

* 依赖

~~~xml
<dependency>
    <groupId>com.auth0</groupId>
    <artifactId>java-jwt</artifactId>
    <version>3.11.0</version>
</dependency>
~~~

* 编码

~~~java
public static String createJwt(){
    HashMap<String,Object> header = new HashMap<>();
    header.put("alg","HS256");
    header.put("typ","JWT");

    Date create = new Date();
    Calendar instance = Calendar.getInstance();
    instance.add(Calendar.SECOND,7200);
    Date expireTime = instance.getTime();
    return JWT.create()
        .withHeader(header)
        .withClaim("username","zhangsan")
        .withClaim("phone","15975612338")
        .withIssuedAt(create)
        .withExpiresAt(expireTime)
        .withSubject("subject")
        .sign(Algorithm.HMAC256("jwt-powernode"));
}
~~~

* 解码

~~~java
public static void decode(String jwt){
    JWTVerifier build = JWT.require(Algorithm.HMAC256("jwt-powernode")).build();
    DecodedJWT verify = build.verify(jwt);
    String username = verify.getClaim("username").asString();
    System.out.println(username);
}
~~~

## SpringSecurity集成JWT

## JWT工具类

~~~java
public class JWTUtils {
    public static String generateJWT(Authentication auth){
        HashMap<String, Object> header = new HashMap<>();
        header.put("alg","HS256");
        header.put("typ","JWT");
        //获取令牌颁发和过期时间
        Date currentTime = new Date();
        Calendar instance = Calendar.getInstance();
        instance.add(Calendar.SECOND,7200);
        Date expireTime  = instance.getTime();
        //获取登录用户权限
        Collection<? extends GrantedAuthority> authorities = auth.getAuthorities();
        List<String> authList = authorities.stream().map(Object::toString).collect(Collectors.toList());
        //获取用户名
        String userName = auth.getName();

        return JWT.create().withHeader(header)
                .withIssuedAt(currentTime)
                .withExpiresAt(expireTime)
                .withClaim("auth", authList)
                .withClaim("username", userName)
                .withSubject("subject")
                .sign(Algorithm.HMAC256("jwt-powernode"));
    }

    public static Authentication decode(String jwt){
        JWTVerifier jwtVerifier = JWT.require(Algorithm.HMAC256("jwt-powernode")).build();
        DecodedJWT verify = null;
        try {
            verify = jwtVerifier.verify(jwt);
        } catch (JWTVerificationException e) {
            return null;
        }
        Date expiresTime = verify.getExpiresAt();
        int i = expiresTime.compareTo(new Date());
        if(i < 0)
            return null;
        String username = verify.getClaim("username").asString();
        List<String> authsStr = verify.getClaim("auth").asList(String.class);
        if(authsStr == null)
            return null;
        List<SimpleGrantedAuthority> auths = authsStr.stream().map(SimpleGrantedAuthority::new).collect(Collectors.toList());
        return new UsernamePasswordAuthenticationToken(username,null,auths);
    }
}
~~~

## 登录成功处理器

在用户登录成功后，为该用户生成并返回jwt令牌，存储在cookies中

~~~java
@Component
public class MyAuthenticationSuccessHandler implements AuthenticationSuccessHandler {

    @Override
    public void onAuthenticationSuccess(HttpServletRequest request, HttpServletResponse response, Authentication authentication) throws IOException, ServletException {
        String jwt = JWTUtils.generateJWT(authentication);
        Cookie cookie = new Cookie("token",jwt);
        cookie.setMaxAge(7200);
        response.addCookie(cookie);
        Map<String, Object> map = new HashMap<>();
        map.put("code",200);
        map.put("msg","OK");
        response.getWriter().write(JSONObject.toJSONString(map));
    }
}
~~~

设置成功处理器

~~~java
http.formLogin().successHandler(successHandler);
~~~

## token拦截器

当用户访问核心业务时，拦截并校验token

~~~java
@Component
public class CheckTokenFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        String URI = request.getRequestURI();
        //放行登录页面和登录请求
        if(URI.equals("/login")||URI.equals("/doLogin")){
            filterChain.doFilter(request,response);
            return;
        }

        //获取token
        Cookie[] cookies = request.getCookies();
        String jwt = null;
        for(Cookie cookie:cookies){
            if(cookie.getName().equals("token")){
                jwt = cookie.getValue();
                break;
            }
        }
        if( jwt != null && !jwt.equals("") ){
            Authentication auth = JWTUtils.decode(jwt);
            if(auth != null){
                String username = auth.getPrincipal().toString();
                System.out.println(auth);
                System.out.println(username);
                filterChain.doFilter(request,response);
                return;
            }
        }

        //其他情况不放行
        response.setContentType("text/html;charset=utf8");
        response.getWriter().write("403,没有权限");
    }
}
~~~

设置拦截器

~~~java
http.addFilterBefore(checkTokenFilter, UsernamePasswordAuthenticationFilter.class);
~~~









