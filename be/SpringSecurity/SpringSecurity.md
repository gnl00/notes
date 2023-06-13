# Spring Security



**引言**

安全方面的两个主要区域是【认证】和【授权】

Web应用的安全性包括用户认证（Authentication）和用户授权（Authorization）两部分

- 用户认证，指的是验证某个用户是否为系统中的合法主体，也就是说【用户能否访问该系统】。用户认证一般主要由用户提供用户名和密码。系统通过校验用户名和密码来完成认证过程，即系统判断用户是否能登录
- 用户授权，指的是【验证某个用户是否有权限执行某个操作】。在一个系统中，不同用户所具有的权限是不同的。系统会为不同的用户分配不同的角色，而每个角色则对应一系列的权限，即系统判断用户是否有权限去做某些操作



**配置**

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```



**基本原理**

> Spring Security本质是一个过滤器链，即由许多个过滤器组成的一条过滤器链



**应用启动加载过滤器链**

```java
org.springframework.security.web.context.request.async.WebAsyncManagerIntegrationFilter,
org.springframework.security.web.context.SecurityContextPersistenceFilter,
org.springframework.security.web.header.HeaderWriterFilter,
org.springframework.security.web.csrf.CsrfFilter,
org.springframework.security.web.authentication.logout.LogoutFilter,
// UsernamePasswordAuthenticationFilter 对/login的POST请求做拦截，校验表单中的用户名和密码
org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter,
org.springframework.security.web.authentication.ui.DefaultLoginPageGeneratingFilter, org.springframework.security.web.authentication.ui.DefaultLogoutPageGeneratingFilter, 
org.springframework.security.web.authentication.www.BasicAuthenticationFilter, org.springframework.security.web.savedrequest.RequestCacheAwareFilter, org.springframework.security.web.servletapi.SecurityContextHolderAwareRequestFilter, 
org.springframework.security.web.authentication.AnonymousAuthenticationFilter, org.springframework.security.web.session.SessionManagementFilter, 
// ExceptionTranslationFilter 异常过滤器，用来处理认证授权过程中抛出的异常
org.springframework.security.web.access.ExceptionTranslationFilter, 
// FilterSecurityInterceptor 方法级权限过滤器，
org.springframework.security.web.access.intercept.FilterSecurityInterceptor
```



**重要接口**

- `UserDetailsService` 查询数据库用户民和密码过程

- `PasswordEncoder` 用户密码加密



## Web权限方案



### 认证

三种认证方式

**配置登录用户名和密码**

1. 配置文件

   ```properties
   spring.security.user.name=admin
   spring.security.user.password=admin
   ```

2. 配置类

   ```java
   // SecurityConfig.java
   
   @Configuration
   public class SecurityConfig extends WebSecurityConfigurerAdapter {
   
       @Override
       protected void configure(AuthenticationManagerBuilder auth) throws Exception {
   
           // 对密码进行加密
           String encodePassword = passwordEncoder().encode("admin");
   
           auth.inMemoryAuthentication()
                   .withUser("admin")
                   .password(encodePassword)
                   .roles("admin");
       }
   
       @Bean
       public PasswordEncoder passwordEncoder() {
           return new BCryptPasswordEncoder();
       }
   }
   ```

3. 自定义实现类

   ```java
   // UserDetailsServiceImpl.java
   
   @Service("userDetailsService")
   public class UserDetailsServiceImpl implements UserDetailsService {
   
       @Resource
       BCryptPasswordEncoder passwordEncoder;
   
       @Override
       public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
   
           List<GrantedAuthority> authorities = AuthorityUtils.createAuthorityList("admin", "root");
   
           User user = new User("admin", passwordEncoder.encode("123"), authorities);
   
           return user;
       }
   
   }
   ```

   ```java
   // 自定义实现类整合数据库进行用户信息认证
   
   @Service("userDetailsService")
   public class UserDetailsServiceImpl implements UserDetailsService {
   
       @Resource
       BCryptPasswordEncoder passwordEncoder;
   
       @Resource
       private UserMapper userMapper;
   
       @Override
       public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
   
           // 连接数据库操作
           QueryWrapper<User> wrapper = new QueryWrapper();
           wrapper.eq("name", username);
           User user = userMapper.selectOne(wrapper);
   
           if (user == null){
               // 认证失败，抛异常
               throw new UsernameNotFoundException("用户不存在...");
           } else {
               // 认证成功
               List<GrantedAuthority> authorities = AuthorityUtils.createAuthorityList(user.getName(), "root");
   
               return new org.springframework.security.core.userdetails.User(user.getName(), passwordEncoder.encode(user.getPassword()), authorities);
   
           }
       }
   
   }
   ```
   
   ```java
   // SecurityConfig.java
   
   @Configuration
   public class SecurityConfig extends WebSecurityConfigurerAdapter {
   
       @Resource
       UserDetailsService userDetailsService;
   
       @Override
       protected void configure(AuthenticationManagerBuilder auth) throws Exception {
           auth.userDetailsService(userDetailsService)
                   .passwordEncoder(passwordEncoder());
       }
   
       @Bean
       public PasswordEncoder passwordEncoder() {
           return new BCryptPasswordEncoder();
       }
   }
   ```



**自定义登录页面**

> 自定义登录界面，不需要认证也能访问

1. 修改配置类

```java
// 
// SecurityConfig.java

@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        // 自定义登录界面
        http.formLogin()
                // 登陆界面设置
                .loginPage("/login.html")
                // 登录访问路径
                .loginProcessingUrl("/user/login")
                // 登陆成功之后跳转到的路径
                .defaultSuccessUrl("/test/loginSuccess")
                .permitAll()

                .and()
                .authorizeRequests()
                // 设置哪些路径不需要认证，可以直接访问
                .antMatchers("/", "/test/hello", "/user/login")
                .permitAll()

                .anyRequest()
                .authenticated()

                // 关闭csrf防护
                .and().csrf().disable();
    }
}
```

2. 创建相关Controller和页面

   ```html
   <form action="/user/login" method="post">
       <!-- 必须 name="username" -->
       name <input type="text" name="username" /><br />
       password <input type="password" name="password" />
       <input type="submit"/>
   </form>
   ```



### 授权

**基于角色或权限进行访问控制**

- `hasAuthority()` 表示当前主体具有【**某种指定的权限**】，有则返回true，否则返回false

  1. 配置路径访问权限

     ```java
     @Override
     protected void configure(HttpSecurity http) throws Exception {
         // 自定义登录界面
         http.formLogin()
                 // 表示当前登录用户必须具有admin权限才可访问此路径
                 .antMatchers("/test/index").hasAuthority("admin");
     }
     ```
2. 在 `UserDetailService` 设置返回用户的权限
  
   ```java
     @Override
     public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
     
         // 连接数据库操作
         if (user == null){
             // 认证失败
         } else {
             // 认证成功，设置用户权限
             List<GrantedAuthority> authorities = AuthorityUtils.commaSeparatedStringToAuthorityList("admin");
     
             return new org.springframework.security.core.userdetails.User(user.getName(), passwordEncoder.encode(user.getPassword()), authorities);
     
         }
     }
   ```

-  `hasAnyAuthority()`  表示当前登录主体具有【**任何提供的权限**】

  ```java
  .antMatchers("/test/index").hasAnyAuthority("admin, root")
  ```

- `hasRole()`  如果用户具有给定角色就返回true，允许访问，否则403

  ```java
  // 当前登录用户具有为给定角色
  .antMatchers("/test/index").hasRole("manager")
  ```

  ```java
  List<GrantedAuthority> authorities = AuthorityUtils.commaSeparatedStringToAuthorityList("admin", "ROLE_manager");
  ```

-  `hasAnyRole()` 当前用户具有任意角色即可访问

  ```java
  // 当前登录用户具有为给定角色
  .antMatchers("/test/index").hasAnyRole("manager", "employee")
  ```







### 自定义错误页面

**配置项**

```java
@Override
protected void configure(HttpSecurity http) throws Exception {
    // 自定义403页面
    http.exceptionHandling().accessDeniedPage("/4xx.html");
}
```



### 注解

> 在控制器方法上添加注解



**`@EnableGlobalMethodSecurity(securedEnabled = true)`**

- 标注在启动类上或配置类上
- 开启SpringSecurity的注解支持功能

**`@Secured`** 

- 标注在方法上

- 用户具有某个角色才可以访问此方法

  ```java
  @Secured({"ROLE_ADMIN"})
  @GetMapping("/update")
  public String update(){
      return "test, update...";
  }
  ```

**`@PreAuthorize`** 在方法访问之前进行验证，可以将用户的 `role` 和 `permissions` 参数传入到方法中

- 开启Pre和Post 注解支持

  ```java
  @EnableGlobalMethodSecurity(securedEnabled = true, prePostEnabled = true)
  ```

- 标注在方法上

  ```java
  @GetMapping("/update")
  // @Secured({"ROLE_ADMIN"})
  // 在方法执行之前检查是否有 ROLE_ADMIN 角色
  @PreAuthorize("hasAnyRole('ROLE_ADMIN')")
  public String update(){
      return "test, update...";
  }
  ```

**`@PostAuthorize`** 在方法执行之后再进行权限验证，适合验证带有返回值的权限

**`@PreFilter`** 权限验证之前对【传入数据】进行过滤

**`@PostFilter`** 权限验证之后对【返回数据】进行过滤

- 标注在方法上

  ```java
  @GetMapping("/update")
  // @Secured({"ROLE_ADMIN"})
  @PreAuthorize("hasAnyRole('ROLE_ADMIN')")
  @PostFilter("filterObject.name=='aaa'")
  public List<User> update(){
  
      List<User> userList = new ArrayList<>();
  
      userList.add(new User(10, "aaa", "qqq"));
      userList.add(new User(20, "bbb", "zzz"));
      // [User(id=10, name=aaa, password=qqq), User(id=20, name=bbb, password=zzz)]
      System.out.println(userList);
      return userList;
  }
  ```



### 用户注销

**配置项**

1. 配置文件中

   ```java
   // 用户注销操作
   http.logout().logoutUrl("/user/logout").logoutSuccessUrl("/test/logoutSuccess").permitAll();
   ```

2.  页面设置

   ```html
   <a href="/user/logout">logout</a>
   ```



### RememberMe

> 1. 登录成功之后生成cookie加密串
> 2. 将cookie加密串和对应的用户信息字符串保存到数据库
> 3. 下次登录的时候将浏览器携带的cookie加密串和数据库中的加密串对比



**步骤**

1. 创建数据库表

   ```sql
   create table persistent_logins (
       username varchar(64) not null,
       series varchar(64) primary key,
       token varchar(64) not null,
       last_used timestamp not null
    )
   ```

2.  修改配置类

   ```java
   // 1. 注入数据源
   @Resource
   DruidDataSource dataSource;
   
   // 2. 配置 PersistentTokenRepository 对象
   @Bean
   PersistentTokenRepository tokenRepository() {
   
       JdbcTokenRepositoryImpl jdbcTokenRepository = new JdbcTokenRepositoryImpl();
       jdbcTokenRepository.setDataSource(dataSource);
       // 启动的时候创建相应的数据库表
       jdbcTokenRepository.setCreateTableOnStartup(true);
       return jdbcTokenRepository;
   }
   
   // 3. 配置 HttpSecurity
   @Override
   protected void configure(HttpSecurity http) throws Exception {
       // 自定义登录界面
       http.formLogin()
           // ...
           .and().rememberMe().tokenRepository(tokenRepository())
           // 设置token的有效时长，单位为秒
           .tokenValiditySeconds(60)
           .userDetailsService(userDetailsService)
   
           // 关闭csrf防护
           .and().csrf().disable();
   }
   ```

3.  在页面添加复选框

   ```html
   <input type="checkbox" name="remember-me" title="记住我"/>
   ```



### CSRF

> Cross-Site-Request-Forgery，跨站请求伪造





### OAuth2



1. **授权码模式 Authorization Code**
2. **密码模式**
3. **客户端模式**
4. **刷新令牌**



## 跨域问题

### 方法①【推荐】

`CorsFilter.java`

```java
/**
 * CorsFilter 跨域请求拦截器
 *
 * @author gnl
 */
@Component
public class CorsFilter extends OncePerRequestFilter {

    // static final String ORIGIN = "Origin";
    static final String OPTIONS = "OPTIONS";

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {

        // 获取请求来源
        // String origin = request.getHeader(ORIGIN);
        // System.out.println(origin);

        // * or origin as u prefer
        response.setHeader("Access-Control-Allow-Origin", "*");
        response.setHeader("Access-Control-Allow-Credentials", "true");
        response.setHeader("Access-Control-Allow-Methods", "PUT, POST, GET, OPTIONS, DELETE");
        response.setHeader("Access-Control-Max-Age", "3600");
        // Authorization header中自定义的token位置
        response.setHeader("Access-Control-Allow-Headers", "content-type, Authorization");

        if (OPTIONS.equals(request.getMethod())) {
            response.setStatus(HttpServletResponse.SC_OK);
        } else {
            filterChain.doFilter(request, response);
        }
    }

}
```

`SecurityConfig.java`

```java
/**
 * SecurityConfig SpringSecurity配置类
 *
 * @author gnl
 */

@Configuration
@EnableGlobalMethodSecurity(securedEnabled = true, prePostEnabled = true)
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {

        http.csrf().disable()
// *********************************************************************************
                .addFilterBefore(new JwtTokenAuthenticationFilter(userDetailsService, handlerExceptionResolver), UsernamePasswordAuthenticationFilter.class)
                .headers()
                .cacheControl();
// *********************************************************************************

    }
}
```



### 方法②【有时候有点傻逼】

`CorsConfig.java`

```java
/**
 * CorsConfig 跨域配置类
 *
 * @author gnl
 */

@Slf4j
@Configuration
public class CorsConfig implements WebMvcConfigurer {

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new HandlerInterceptor() {
            @Override
            public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {

                // 允许所有地址
                response.setHeader("Access-Control-Allow-Origin", "*");
                response.setHeader("Access-Control-Allow-Methods", "PUT, POST, GET, OPTIONS, DELETE");
                response.setHeader("Access-Control-Max-Age", "3600");
                // Authorization header中自定义的token位置
                response.setHeader("Access-Control-Allow-Headers", "content-type, Authorization");

                return true;
            }
        });
    }
}
```

### 方法③【也有点傻逼】

`CorsConfig.java`

```java
@Configuration
public class CrosConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**")
            // allowedOriginPatterns 注意不是 allowedOrigin
                .allowedOriginPatterns("*")
                .allowedMethods("GET","HEAD","POST","PUT","DELETE","OPTIONS")
                .allowCredentials(true)
                .maxAge(3600)
                .allowedHeaders("*");
    }
}
```

