# OAuth简介

OAuth是一种开放协议, 允许用户让第三方应用以安全且标准的方式获取该用户在某一网站，移动或者桌面应用上存储的秘密的资源（如用户个人信息，照片，视频，联系人列表），而无需将用户名和密码提供给第三方应用。

## OAuth认证流程

![OAuth认证流程](https://gitee.com/wangziming707/note-pic/raw/master/img/OAuth%E8%AE%A4%E8%AF%81%E6%B5%81%E7%A8%8B.png)

## OAuth角色

- 资源所有者(Resource Owner)： 能够许可受保护资源访问权限的实体。当资源所有者是个人时，它作为最终用户被提及。
- 用户代理(User Agent)： 指的的资源拥有者授权的一些渠道。一般指的是浏览器、APP
- 客户端(Client) 使用资源所有者的授权代表资源所有者发起对受保护资源的请求的应用程序。术语“客户端”并非特指任何特定的的实现特点（例如：应用程序是否在服务器、台式机或其他设备上执行）。
- 授权服务器(Authorization Server)： 在成功验证资源所有者且获得授权后颁发访问令牌给客户端的服务器。
    授权服务器和资源服务器之间的交互超出了本规范的范围。授权服务器可以和资源服务器是同一台服务器，也可以是分离的个体。一个授权服务器可以颁发被多个资源服务器接受的访问令牌。
- 资源服务器(Resource Server)： 托管受保护资源的服务器，能够接收和响应使用访问令牌对受保护资源的请求。

~~~nginx
http://localhost:8000/oauth/authorize?response_type=code&client_id=lisi&redirect_uri=https://www.baidu.com
~~~



# 授权服务器

使用springcloud框架构建授权模块

* 处于安全考虑，授权服务器必须实现springSecurity登录验证
* 资源服务器颁发令牌时将令牌存储到redis中，验证令牌时从redis中取出数据校验

## 依赖配置

* 依赖

~~~xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-oauth2</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

~~~

* springboot核心配置文件

~~~yaml
server:
    port: 8000
spring:
    redis:
        url: redis://localhost:6379
        database: 0
~~~

* 启动类

~~~java
//启动WebSecurity和AuthorizationServer
@SpringBootApplication
@EnableAuthorizationServer
@EnableWebSecurity
public class FirstOauthApplication {

    public static void main(String[] args) {
        SpringApplication.run(FirstOauthApplication.class, args);
    }
	//IOC PasswordEncoder 供密码加密
    @Bean
    public PasswordEncoder passwordEncoder(){
        return new BCryptPasswordEncoder();
    }
}
~~~

## 配置类

* 配置security

~~~java
@Configuration
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    @Autowired
    private PasswordEncoder passwordEncoder;

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        //设置登录页面
        http.formLogin();
        //认证所有请求
        http.authorizeRequests().anyRequest().authenticated();

    }
    
    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        //添加一个用户
        auth.inMemoryAuthentication()
                .withUser("admin")
                .password(passwordEncoder.encode("admin"))
                .roles("CEO")
                .authorities("sys:update","sys:delete","sys:queue");
    }
}
~~~

* 配置authorization

~~~java
@Configuration
public class AuthorizationConfig extends AuthorizationServerConfigurerAdapter {

    @Autowired
    private PasswordEncoder passwordEncoder;

    @Autowired
    private RedisConnectionFactory connectionFactory;

    @Bean
    public TokenStore tokenStore(){
        //使用redis存储令牌
        return new RedisTokenStore(connectionFactory);
    }

    @Override
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
        //在内存中定义一个第三方客户端
        clients.inMemory()
                .withClient("client")//用户名
                .secret(passwordEncoder.encode("123"))//密码
                .scopes("web-scopes")//令牌作用范围
                .authorizedGrantTypes("authorization_code")//授权类型
                .redirectUris("https://www.baidu.com")//授权后跳转页面
                .accessTokenValiditySeconds(7200);//令牌有效时间
    }

    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
        //配置令牌存储
        endpoints.tokenStore(tokenStore());
    }
}
~~~



## 授权类型

oauth提供了4中不同的授权方式

- 授权码模式（authorization code）
- 简化模式（implicit）
- 密码模式（resource owner password credentials）
- 客户端模式（client credentials）

通过配置类中client 的authorizedGrantTypes 改变授权服务器的授权类型，该方法支持的入参有：

* authorization_code：授权码授权

* implicit 静默授权

* password 密码授权

* client_credentials 客户端授权

* refresh_token 刷新token

每种授权方式请求token方式不同，请求url的格式也不同

### 授权码模式

第三方应用先申请一个授权码，然后再用该码获取令牌。



#### 服务端设置

~~~java
clients.inMemory()
    .withClient("client-code")
    .secret(passwordEncoder.encode("123"))
    .scopes("web-scopes")
    .authorizedGrantTypes("authorization_code")//授权模式
    .redirectUris("https://www.baidu.com")
    .accessTokenValiditySeconds(7200);

~~~



#### 请求获取code

* 请求方式：GET
* 请求URI: /oauth/authorize
* 请求认证：无
* 请求参数：
    * response_type  ： code  响应一个code码
    * client_id:第三方客户端账户ID(clients定义的用户名)
    * redirect_uri :请求成功后的重定向地址，要是公网的地址，必须https

* 示例：

~~~
请求:
http://localhost:8000/oauth/authorize?response_type=code&client_id=client-code&redirect_uri=https://www.baidu.com 
响应:
https://www.baidu.com/?code=WU9jlb
~~~

#### 请求获取token

* 请求方式：POST
* 请求URI: /oauth/token
* 请求认证：Basic Auth
    * Username: client用户名
    * Password:client密码
* Body：
    * code  ：授权码(授权类型为authorization_code时)
    * grant_type:授权类型，(authorization_code,)
    * redirect_uri :请求成功后的重定向地址，要是公网的地址，必须https

* 示例：

~~~json
请求:{
    URL : 'http://localhost:8000/oauth/token',
    authorization : {
    	Username :  "client-code" ,
    	Password :  123 
	}，
    body : {
    	code : "ZJEmHQ" ,
    	grant_type : "authorization_code"，
    	redirect_uri : 'https://www.baidu.com' 
	}
}
响应:{
    "access_token": "69ac0495-22ee-4a94-b6ba-f3cf90317439",
    "token_type": "bearer",
    "expires_in": 7199,
    "scope": "web-scopes"
}
~~~



### 简化模式

不通过第三方应用程序的服务器，直接在浏览器中向认证服务器申请令牌，跳过了"授权码"这个步骤

#### 服务端设置

~~~java
clients.inMemory()
	.withClient("client-implicit")
    .secret(passwordEncoder.encode("123"))
    .scopes("web-scopes")
    .authorizedGrantTypes("implicit") //授权模式为静默
    .redirectUris("https://www.baidu.com")
    .accessTokenValiditySeconds(7200);
~~~

#### 请求token

与授权码模式的授权请求url一样

* 请求方式：GET
* 请求URI: /oauth/authorize
* 请求认证：无
* 请求参数：
    * response_type  ： token 直接响应code
    * client_id:第三方客户端账户ID(clients定义的用户名)
    * redirect_uri :请求成功后的重定向地址，要是公网的地址，必须https

* 示例

~~~nginx
请求:
http://localhost:8000/oauth/authorize?response_type=token&client_id=client-implicit&redirect_uri=https://www.baidu.com
响应：
https://www.baidu.com/#access_token=f5726f06-4528-4257-8d5c-96b3d70f80f1&token_type=bearer&expires_in=7199&scope=web-scopes
~~~

### 密码模式

用户向客户端提供自己的用户名和密码，这通常用在用户对客户端高度信任的情况

#### 服务端设置

* 创建password类型的账户

~~~java
clients.inMemory().withClient("client-password")
    .secret(passwordEncoder.encode("123"))
    .scopes("web-scopes")
    .authorizedGrantTypes("password")   //密码授权
    .redirectUris("https://www.baidu.com")
    .accessTokenValiditySeconds(7200);  
~~~

* 将`springSecurity`的`AuthenticationManager`认证管理器交给容器管理

~~~java
//重写 WebSecurityConfigurerAdapter的authenticationManagerBean方法并加上@Bean注解
@Bean
@Override
public AuthenticationManager authenticationManagerBean() throws Exception {
    return super.authenticationManagerBean();
}
~~~

* 注入到`AuthorizationConfig `并声明给`endpoints`

~~~java
@Autowired
private AuthenticationManager authenticationManager;

@Override
public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
    //配置令牌存储
    endpoints.tokenStore(tokenStore())
        .authenticationManager(authenticationManager);//配置认证管理器
}
~~~

#### 请求token

* 请求方式：POST
* 请求URI: /oauth/token
* 请求认证：Basic Auth
    * Username: client用户名
    * Password:client密码
* Body：
    * grant_type:授权类型，(password)
    * username:资源拥有者的账户
    * password:资源拥有者的密码

* 示例：

~~~json
请求 : {
    URL : 'http://localhost:8000/oauth/token',
    authorization : {
    	Username :  "client-passoword" ,
    	Password :  123 
	}，
    body : {
    	grant_type : "password" ,
    	username : "admin",
    	password : "admin"
	}
}
响应 : {
    "access_token": "69ac0495-22ee-4a94-b6ba-f3cf90317439",
    "token_type": "bearer",
    "expires_in": 7199,
    "scope": "web-scopes"
}
~~~



### 客户端模式

授权整个第三方应用访问权限，不需要通过用户

#### 服务端设置

~~~java
clients.inMemory()
    .withClient("client-credentials")
    .secret(passwordEncoder.encode("123"))
    .scopes("web-scopes")
    .authorizedGrantTypes("client_credentials")   //客户端授权
    .redirectUris("https://www.baidu.com")
    .accessTokenValiditySeconds(7200);
~~~

#### 请求token

* 请求方式：POST
* 请求URI: /oauth/token
* 请求认证：Basic Auth
    * Username: client用户名
    * Password:client密码
* Body：
    * grant_type:授权类型，(client_credentials)

* 示例：

~~~json
请求 : {
    URL : 'http://localhost:8000/oauth/token',
    authorization : {
    	Username :  "client-credentials" ,
    	Password :  123 
	}，
    body : {
    	grant_type : "client_credentials"
	}
}
响应 : {
    "access_token": "69ac0495-22ee-4a94-b6ba-f3cf90317439",
    "token_type": "bearer",
    "expires_in": 7199,
    "scope": "web-scopes"
}
~~~



# 资源服务器

资源服务器的资源需要经过token验证在允许访问

课通过多种方式进行校验



## 依赖配置

* 依赖

~~~xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-oauth2</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
~~~

* springboot启动类注解：

~~~java
//开启资源服务器，和security权限验证
@EnableResourceServer
@EnableGlobalMethodSecurity(prePostEnabled = true)
~~~

* ResourceConfig鉴权

~~~java
//对暴露接口进行鉴权
@Configuration
public class ResourceConfig extends ResourceServerConfigurerAdapter {
    @Override
    public void configure(HttpSecurity http) throws Exception {
        //放行free接口
        http.authorizeRequests().antMatchers("/free").permitAll();
        //其他接口需要鉴权
        http.authorizeRequests().anyRequest().authenticated();
    }

}
~~~

## 校验Token



### 普通token

对于普通的token需要资源服务器访问授权服务器获取token进行校验

#### 授权服务器

对于普通的token，授权服务器需要暴露一个用户信息的接口，以供资源服务器校验

并且授权服务器也需要注册为资源服务器以让用户接口可以直接通过token访问

* 暴露接口

~~~java
//在restController中
@GetMapping("/info")
public Object info(Principal principal){
    return principal;
}
~~~

* 注册为资源服务

~~~java
@SpringBootApplication
@EnableWebSecurity
@EnableAuthorizationServer
@EnableResourceServer
public class JdbcOauthApplication {

    public static void main(String[] args) {
        SpringApplication.run(JdbcOauthApplication.class);
    }

}
~~~

#### 资源服务器

资源服务器需要设置访问授权服务器的链接接口

在核心配置文件中配置：

~~~yaml
security:
    oauth2:
        resource:
        	#授权服务器暴露的用户信息接口
            user-info-uri: http://localhost:8081/info
~~~



### JWT

使用JWT作为令牌，资源服务器可以不需要访问授权服务器就可以对token进行校验



#### 授权服务器

AuthorizationConfig中TokenStore使用JWT样式：

~~~java
@Configuration
public class AuthorizationConfig extends AuthorizationServerConfigurerAdapter {


    @Autowired
    private AuthenticationManager authenticationManager;

    @Bean
    public TokenStore tokenStore(){
        //使用JWT生成令牌
        return new JwtTokenStore(jwtAccessTokenConverter());
    }

    @Bean
    public JwtAccessTokenConverter jwtAccessTokenConverter(){
        JwtAccessTokenConverter jwtAccessTokenConverter = new JwtAccessTokenConverter();
        //设置一个jwt签名,头部和载荷不需要我们设置，默认已经给设置好了
        jwtAccessTokenConverter.setSigningKey("auth-jwt");
        return jwtAccessTokenConverter;
    }

    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
        //配置令牌存储
        endpoints.tokenStore(tokenStore())
                .authenticationManager(authenticationManager)//配置认证管理器
                .accessTokenConverter(jwtAccessTokenConverter());
    }
}
~~~

#### 资源服务器

删除配置文件中的：user-info-uri

ResourceConfig：

~~~java
@Configuration
public class ResourceConfig extends ResourceServerConfigurerAdapter {
    @Override
    public void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests().antMatchers("/free").permitAll();
        http.authorizeRequests().anyRequest().authenticated();
    }

    @Bean
    public TokenStore tokenStore() {
        return new JwtTokenStore(jwtAccessTokenConverter());
    }


    @Bean
    public JwtAccessTokenConverter jwtAccessTokenConverter() {
        JwtAccessTokenConverter jwtAccessTokenConverter = new JwtAccessTokenConverter();
        jwtAccessTokenConverter.setSigningKey("auth-jwt");
        return jwtAccessTokenConverter;
    }


    @Override
    public void configure(ResourceServerSecurityConfigurer resources) throws Exception {
        resources.tokenStore(tokenStore());
    }
}

~~~

## 访问资源服务器

* 方式一：请求头中加入认证参数：
    * Key :` Authorization`
    * Value:  `bearer 令牌`

* 方式二：请求体中加入参数：
    * Key: `access_token`
    * Value: `令牌`

# JWT使用非对称加密

使用非对称加密的安全性更强，只要私钥不泄露，第三方就无法自己生成token

## 生成私钥公钥

使用OpenSSL软件，输入以下指令

~~~bash
# 生成私钥 私钥文件位置在软件文件夹的bin目录下 文件名为 cxs-jwt.jks
keytool -genkeypair -alias cxs-jwt -validity 3650 -keyalg RSA -dname "CN=jwt,OU=jtw,O=jwt,L=zurich,S=zurich,C=CH" -keypass cxs123 -keystore cxs-jwt.jks -storepass cxs123

# 根据私钥生成公钥
keytool -list -rfc --keystore cxs-jwt.jks | openssl x509 -inform pem -pubkey
# 输入密钥库口令
cxs123
# 控制台会打印出公钥，将其复制并保存到PublicKey.txt文件
# 从BEGIN PUBLIC KEY 到 END PUBLIC KEY 的所有部分
~~~

这样就获得了两个文件：

* PublicKey.txt
* cxs-jwt.jks

## 改造授权服务端

将cxs-jwt.jks文件复制到项目的resources文件夹下

改造AuthorizationConfig类的jwtAccessTokenConverter()方法：

~~~java
@Bean
public JwtAccessTokenConverter jwtAccessTokenConverter(){
    //使用非对称加密，只要私钥不泄密，别人无法生成token，公钥只能验证token
    JwtAccessTokenConverter jwtAccessTokenConverter = new JwtAccessTokenConverter();
    //将文件加载进来
    ClassPathResource resource = new ClassPathResource("cxs-jwt.jks");
    //得到keyStore
    KeyStoreKeyFactory keyFactory = new KeyStoreKeyFactory(resource, "cxs123".toCharArray());
    //设置私钥
    KeyPair keyPair = keyFactory.getKeyPair("cxs-jwt");
    jwtAccessTokenConverter.setKeyPair(keyPair);
    return jwtAccessTokenConverter;
}
~~~



## 改造资源服务器

将PublicKey.txt文件复制到项目resources文件夹下

改造ResourceConfig类中的 jwtAccessTokenConverter()方法：

~~~java
@Bean
public JwtAccessTokenConverter jwtAccessTokenConverter() {
    JwtAccessTokenConverter jwtAccessTokenConverter = new JwtAccessTokenConverter();

    ClassPathResource resource = new ClassPathResource("PublicKey.txt");
    //使用了hutool需要导包
    String publicKey = FileUtil.readString(resource.getFile(), Charset.defaultCharset());
    //设置加密公钥
    jwtAccessTokenConverter.setVerifierKey(publicKey);

    return jwtAccessTokenConverter;
}
~~~



























