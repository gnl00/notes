---
description: SpringBoot 开发详解
tag: 
  - SpringBoot
  - 后端
---

# SpringBoot

> **和 Spring 的区别**
>
> * 配置，Starter 模块集成
>
> * 注解支持
>
> * 打包与部署

...

## 配置开发

…

### 配置文件

…

#### 命名风格

默认 `application.properties`，也可以根据环境来使用不同配置 `application-{env}.yml`，其中 env 可以是 test/dev/prod。比如  `application-test.yml` 表示测试环境配置。

…

> 配置文件默认以 application.properties 或者 application.yaml 为名称，此外还可以通过指定 `spring.config.name` 属性更改配置文件名。

…

使用了不同环境下的配置文件之后需要在默认配置中激活指定环境配置。

…

#### 指定生效的配置

1、配置文件中指定

```properties
# 激活指定环境配置
spring.profiles.active=dev
# 在默认配置文件中激活了指定环境的配置文件后，指定激活的配置文件和默认的配置文件中的配置内容【同时生效】
# 如果没有指定激活配置，默认配置文件和指定激活的配置文件中有【相同的配置内容】，则默认配置生效
# 如果有指定激活的配置文件，则指定【激活的配置内容生效】

# 指定激活的分组为mygroup组
spring.profiles.active=mygroup

spring.profiles.group.mygroup[0]=prod-1
spring.profiles.group.mygroup[1]=prod-2

spring.profiles.group.test[0]=test
```

2、启动命令行指定

```shell
# 运行时指定激活配置文件
java -jar boot-demo.jar --spring.profiles.active=test
```

在以上两种修改配置文件的操作中，以**命令行指定的配置优先**

…

---

<br>

### 配置注解

和 SpringBoot 配置相关的注解有：

* @Configuration
* @Import
* @Conditional
* @ImportResource
* @Profile
* …

…

#### @Configuration

> 标注在类上，使用 @Configuration 标注的类为配置类，配置类本身也会注册到容器中。配置类中可以使用 @Bean 标注在方法上给容器注册组件，默认注册的组件为单实例。

…

**proxyBeanMethods**

@Configuration 有一个属性：proxyBeanMethods 表示配置类中的 @Bean 方法是否应该被代理

* true 表示 Full 模式，保证 @Bean 方法无论被调用多少次返回的都是单例对象；

* false 表示 Lite 模式，保证 @Bean 方法无论被调用多少次返回的都是新对象。

…

`proxyBeanMethods=true` 的优点是可以保证在同一个ApplicationContext中，同一个bean只被创建一次，从而有效地利用了单例模式。缺点是：需要创建代理类，会对启动速度产生一定影响。

`proxyBeanMethods=false` 的优点是启动速度更快，因为不需要创建代理。但是这也意味着如果在同一配置类中有多个 @Bean 方法调用了同一个 @Bean 方法，那么会有多个实例被创建，不再享受 Spring 的单例保障。

…

**使用场景**

- 配置类创建的 Bean 之间**无依赖关系**，用 Lite 模式加速容器启动，减少容器对生成组件的判断；
- 配置类创建的 @Bean 方法需要用到之前保存在容器中的单实例组件，用 Lite 模式。

…

> proxyBeanMethods 看起来和 @EnableAspectJAutoProxy(proxyTargetClass = true) 配置有点类似，proxyTargetClass 为 true 表示强制使用 CGLib 代理。

…

---

…

上文介绍了指定不同环境下的配置文件，下面介绍指定不同环境下要注入 IOC 容器的 bean。

这就需要使用到 @Profile 注解。

…

####  @Profile

指定注入到容器中的某个 Bean 在什么环境生效

```java
@Profile("test") // 表明此组件只有在test环境才注入到容器中
@Component
public class TestComponent() {}

@Profile("dev")
@Component
public class DevComponent() {}

@Configuration
public class MyConfig() 
    @Profile("prod") // 标在方法上，表明这个 Bean 只在 prod 环境注入
    @Bean
    public Production production() { return null; }
}
```

…

---

#### @Import

> 标注在类上，自动给容器中注入某些类型的 bean，默认的 bean 名为全类名。

如果存在多个配置类，不需要给每个配置类都标上 @Configuration，只需要标注在一个类上，然后使用 @Import 注解导入其他配置类即可。

…

---

#### @Conditional

> 标注在类或方法上，根据条件给容器中注入 Bean。

```java
@Conditional
@ConditionalOnExpression
@ConditionalOnSingleCandidate // 容器中有某个 bean 且为单例
@ConditionalOnMissingBean // 容器中没有有某个 bean
@ConditionalOnBean // 容器中有某个 bean
@ConditionalOnJava // 当前 Java 版本
@ConditionalOnClass // 容器中包含有某个 Class
@ConditionalOnWebApplication  // web 应用
@ConditionalOnNotWebApplication // 不是 web 应用
@ConditionalOnMissingClass // 容器中不包含某个 Class
@ConditionalOnResource // 容器中包含有某个资源
```

…

---

#### @ImportResource

> 标注在类上，将 XML 配置文件中的内容加载到 Spring 容器中

```java
// 将 classpath 下的 spring-context.xml 加载到容器中
@ImportResource(locations = {"classpath:spring-context.xml"})
```

…

#### @SpringBootApplication

@SpringBootApplication 是一个组合注解，包含：

* @SpringBootConfiguration = @Configuration
* @EnableAutoConfiguration 自动扫描 classpath 中存在的依赖，比如 spring-boot-webmvc 依赖，并启用相应的配置
* @ComponentScan 根据条件扫描指定位置下的组件，并加载到容器中

…

```java
// SpringBootApplication.java
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
      @Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {}
```

…

@SpringBootConfiguration

```java
@Configuration
public @interface SpringBootConfiguration {}
```

…

@EnableAutoConfiguration，是一个组合注解，由下面两个注解组成

* @Import 导入一个或者多个组件
* @AutoConfigurationPackage 根据规则导入指定的包

主要作用是根据 classpath 中的依赖利用 @AutoConfigurationPackage 自动导入包，利用 @Import 导入配置，并开启自动配置。

```java
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {}
```

…

@ComponentScan

指定需要扫描哪些包，包含以下属性：

- basePackages 扫描时扫描哪些包
- basePackageClasses 扫描时扫描哪些类
- includeFilters 扫描时只扫描哪些组件
- excludeFilters 扫描的时排除哪些组件
- lazyInit 包扫描懒加载

```java
// 包扫描时包含哪些内容
@ComponentScan(includeFilters = {
    @ComponentScan.Filter(type = FilterType.ANNOTATION, classes = {Controller.class, Service.class})
})
```

…

---

<br>

### 配置绑定

…

> SpringBoot 提供 @ConfigurationProperties 注解将 Java Bean 与配置文件绑定，可以在对应的类上标注 @ConfigurationProperties 注解注册到容器中。或使用 @ConfigurationPropertiesScan 扫描导入配置类。

…

**配置绑定组合**

* @Component + @ConfigurationProperties
* @EnableConfigurationProperties 标注在 @Configuration 类上

…

#### @Component + @ConfigurationProperties

> 用于自定义类

**实现功能**

1、将 Bean 组件注入到 IOC 容器

2、开启注入的 Bean 组件的配置绑定功能

```java
// 首先需要先将 Bean 放到 SpringBoot 容器中，@ConfigurationProperties 才能生效
@Component
@ConfigurationProperties(prefix = "mycar")
public class Car {

    private String band;
    private Integer price;

    public String getBand() {
        return band;
    }

    public void setBand(String band) {
        this.band = band;
    }

    public Integer getPrice() {
        return price;
    }

    public void setPrice(Integer price) {
        this.price = price;
    }
}
```

```properties
# application.properties
mycar.band=BYD
mycar.price=100
```

…

---

#### @EnableConfigurationProperties

> 注入与绑定第三方类

**实现功能**

1、将 Bean 组件注入到 IOC 容器中

2、开启注入的 Bean 组件的配置绑定功能

```java
@Configuration
@EnableConfigurationProperties(Car.class)
public class MainConfig {}
```

```properties
# application.properties
mycar.band=WULING
mycar.price=1000
```

…

---

<br>

### 配置校验

可以使用 JSR303 的 @Validated 注解与 @ConfigurationProperties 配合，用来校验配置中的属性。

```java
@ConfigurationProperties("my.service")
@Validated
public class MyProperties {
    @NotNull
    private InetAddress remoteAddress;
    // getters/setters...
}
```

为了确保对组合对象的校验生效，需要在组合对象属性上标注 @Valid 注解。

```java
@ConfigurationProperties("my.service")
@Validated
public class MyProperties {
    @NotNull
    private InetAddress remoteAddress;
    @Valid
    private final Security security = new Security();
    // getters/setters...
    public static class Security {
        @NotEmpty
        private String username;
        // getters/setters...
    }
}
```

…

---

<br>

### 配置加载

> [配置文件加载原理分析](https://www.jianshu.com/p/b60b20fe41ae)

SpringBoot 支持多种配置方式：

* @Configuration 配置类；

* 配置文件 `*.properties` 和 `*.yaml`。

  如果在同一个地方同时存在 properties 和 YML 文件，会以 properties 配置为准。

  > 因为 Spring Boot 在读取配置文件时，先加载的是 properties 配置文件，然后才是 yaml 配置文件。如果两个文件中存在相同的配置项，在读取时会先读取 properties 配置文件中的值，忽略 yaml 配置文件中的值。

* @PropertySource，配合 @Configuration 配置类使用。标注在配置上，引入外部配置文件。

  > 注意：此方法添加的配置会在容器被刷新后才加载，因此对于 `logging` 和 `spring.main.*`  配置来说加载时间太迟。此时可继承 EnvironmentPostProcessor 接口，在自定义环境后置处理器中处理外部配置。

  …

  > YAML 配置文件不能使用 @PropertySource 来加载。可以使用
  >
  > 1. YamlPropertiesFactoryBean
  > 2. 或 YamlMapFactoryBean
  >
  > 来加载 YAML 文件，前者将 YAML 文件加载为 properties 文件，后者加载为 Map。
  >
  > …
  >
  > 还可以使用 YamlPropertySourceLoader 将 YAML 文件加载为 Spring 的 PropertySource 类。

* 系统环境变量

* Java 系统配置

* JNDI 配置 from `java:comp/env`

* ServletContext 初始化参数

* ServletConfig 初始化参数

* 来自 SPRING_APPLICATION_JSON 内嵌 JSON 的参数

* 命令行参数

…

**配置文件加载顺序**

* 应用程序内的配置文件，如 `application.properties`
* 应用程序内不同环境的配置文件，如`application-dev.properties`
* 应用程序外部的配置文件
* 应用程序外部不同环境的配置文件
* 命令行参数

…

> 后加载的配置文件会覆盖前面配置文件中的内容

…

**配置文件扫描**

SpringBoot 启动时，会从以下地方扫描配置文件：

1、classpath

* classpath 根目录
* classpath 中的 config 目录

2、当前目录下

* 当前根目录
* 当前目录下的 config 目录
* 当前 config 子目录中的直接子目录

3、还可以通过 `spring.config.location` 指定配置文件存放路径

…

---

<br>

### 自动配置

拿 spring-boot-starter-web 依赖来说，在导入该依赖后 SpringBoot 在会自动应用某些默认配置。

在启动过程中，SpringBoot 会检查 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 文件，大概内容如下：

```
org.springframework.boot.autoconfigure.web.servlet.DispatcherServletAutoConfiguration
org.springframework.boot.autoconfigure.web.servlet.ServletWebServerFactoryAutoConfiguration
org.springframework.boot.autoconfigure.web.servlet.error.ErrorMvcAutoConfiguration
org.springframework.boot.autoconfigure.web.servlet.HttpEncodingAutoConfiguration
org.springframework.boot.autoconfigure.web.servlet.MultipartAutoConfiguration
org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration
```

如果 classpath 中存在对应的类，SpringBoot 会自动将其加载。

…

**自动配置流程梳理**

1、SpringBoot 先加载所有的自动配置类，`xxxAutoConfiguration`

2、每个自动配置类按照默认条件的值生效，默认都会绑定配置文件 `xxxProperties` 指定的值

3、生效的配置类就会给容器中装配对应的组件

4、只要容器中有对应的组件，对应的功能也会自动开启

5、自定义配置

1）在自定义的配置类利用 @Bean 标注方法，替换底层的组件方法

2）看要自定义的组件绑定的配置文件是什么值，到 `application.propeties` 中修改。

配置加载流程：`xxxAutoConfiguration` –> 组件 –> `xxxProperties` 中拿到对应的值 –> `application.properties`

…

---

<br>

## Web 开发

> 利用 Spring Boot 可以使用内置的 Tomcat/Jetty/Undertow/Netty 快速开发一个 HTTP 服务。大多数的 Spring Web 应用都是直接使用 `spring-boot-starter-web` 模块来进行快速开发的。

> 与纯 Spring Web MVC 相比，Spring Boot 的 Web 模块初始化顺序是完全不同的。Spring Web MVC 使用外部的 Servlet 容器，并在 Servlet 容器的生命周期使用勾子函数来完成一系列配置。
>
> Spring Boot 使用 Spring 配置启动内置的 Servlet 容器，并声明一系列的 Filter 和 Servlet，注册到内置 Servlet 容器中。

…

> 如果想要记录所有请求的细节，可以在配置文件中打开 `spring.mvc.log-request-details` 或 `spring.codec.log-request-details`。

…

### Web 环境检测

**上下文创建**

* 如果依赖列表中存在 Spring MVC，SpringBoot 会使用 AnnotationConfigServletWebServerApplicationContext 来创建容器；
* 如果存在 Spring WebFlux 相关依赖会使用 AnnotationConfigReactiveWebServerApplicationContext 来创建容器；
* 否则默认使用 AnnotationConfigApplicationContext

…

---

### 自定义配置

如果想修改 SpringMVC 的配置，可以创建一个 WebMvcConfigurer 接口的自定义的配置类，并标上 @Configuration 注解即可。 

…

---

### 静态资源访问

> 只要将静态资源（图片，CSS，JS）放在指定的目录下，即可通过相应路径访问。

SpringBoot 默认静态资源目录：

- `/static`
- `/public` 
- `/resources`
- `/META-INF/resources`

…

访问：`当前项目根路径/静态资源名` => `localhost:8080/静态资源名`

…

还可以通过配置来对静态资源目录进行自定义

```yaml
spring:
  web:
    resources:
      static-locations: [classpath:/haha/] # 在原有的静态资源文件路径基础上，自定义另外的静态资源存放目录
```

…

配置自定义静态资源访问前缀

```yaml
spring:
  web:
    resources:
      static-locations: [classpath:/haha/]
  mvc:
    # 在配置文件中修改静态资源访问前缀
    static-path-pattern: /resources/** # 此时静态资源访问路径为: localhost:8080/resouces/静态资源名
```

…

静态资源请求流程

1）请求进来之后先去找 Controller 中的方法，看有没有方法能处理当前请求

2）不能处理的请求交给静态资源处理器进行处理

3）静态资也未找到的资源返回 404

…

> ⚠️ 自定义静态资源路径和访问前缀都会导致欢迎页和 favicon 失效

…

---

### Restful 风格

**1、Web 页面表单开启 Restful Url支持**

1）修改配置文件

```yaml
spring:
  mvc:
    hiddenmethod:
      filter:
        enabled: true # 开启 html 表单的 Restful 风格
```

2）HTML 页面中表单设置隐藏的属性值

```html
<form action="/user" method="post">
    <input type="hidden" name="_method" value="put" >
    <input type="text" >put-ipt
    <input type="submit">
</form>
<form action="/user" method="post">
    <input type="hidden" name="_method" value="delete" >
    <input type="text" >delete-ipt
    <input type="submit">
</form>
```

…

**2、将 `_method` 改成自定义**

```java
// WebMvcAutoConfiguration.java
@Bean
@ConditionalOnMissingBean(HiddenHttpMethodFilter.class) // 若是组件中没有 HiddenHttpMethodFilter，则启用 Spring 默认的配置
@ConditionalOnProperty(prefix = "spring.mvc.hiddenmethod.filter", name = "enabled", matchIfMissing = false)
public OrderedHiddenHttpMethodFilter hiddenHttpMethodFilter() {
   return new OrderedHiddenHttpMethodFilter();
}

// HiddenHttpMethodFilter.java
public class HiddenHttpMethodFilter extends OncePerRequestFilter {

	private static final List<String> ALLOWED_METHODS =
			Collections.unmodifiableList(Arrays.asList(HttpMethod.PUT.name(),
					HttpMethod.DELETE.name(), HttpMethod.PATCH.name()));

	/** Default method parameter: {@code _method}. */
    // Spring 默认的配置即为需要传过来名为"_method"的参数
	public static final String DEFAULT_METHOD_PARAM = "_method";
}

// 1、新建自定义配置类
// WebConfig.java

// proxyBeanMethods = false 不需要依赖其他组件，proxyBeanMethods设置为false，跳过扫描，加快启动
@Configuration(proxyBeanMethods = false)
public class WebConfig {

    @Bean
    public HiddenHttpMethodFilter hiddenHttpMethodFilter(){
        HiddenHttpMethodFilter hiddenHttpMethodFilter = new HiddenHttpMethodFilter();
        // 2、将参数_method修改成自定义"_gnl"
        // 3、在html页面中的表单设置<input type="hidden" name="_gnl" value="put" >
        hiddenHttpMethodFilter.setMethodParam("_gnl");
        return hiddenHttpMethodFilter;
    }

}
```

…

---

### Web 开发注解

* @PathVariable
* @RequestHeader
* @ModelAttribute

* @RequestParam

* @MatrixVariable

* @CookieValue

* @RequestBody

* @ResponseBody

```java
// http://localhost:8080/pv/zhangsan/aaa/18?id=10&scores=80&scores=90
@ResponseBody
@GetMapping("/pv/{name}/aaa/{age}")
public Map<String, Object> test(@PathVariable(value ="name") String name,
                                            @PathVariable(value = "age") Integer age,
                                            // 可以用@PathVariable Map<String,String> 来接收
                                            // 路径中所有@PathVariable 的参数，Map的key和value都必须是String类型
                                            @PathVariable Map<String, String> pvMap,
                                            // @RequestHeader(value = "User-Agent") 获取请求头中的某个参数
                                            @RequestHeader(value = "User-Agent") String userAgent,
                                            // @RequestHeader Map<String, String> 获取请求头中所有的参数信息
                                            // 需用Map<String, String> 来接收
                                            @RequestHeader Map<String, String> header,
                                            @RequestParam(value = "id") String id,
                                            @RequestParam(value = "scores") List<Integer> scores,
                                            @RequestParam Map<String, String> paramsMap,
                                            @CookieValue(value = "Webstorm-85c34080") String cookieStr,
                                            @CookieValue(value = "Webstorm-85c34080") Cookie cookie){

    Map<String, Object> map = new HashMap<>();
    map.put("name", name);
    map.put("age", age);
    map.put("pvMap", pvMap);
    map.put("userAgent", userAgent);
    map.put("header", header);
    map.put("id", id);
    map.put("scores", scores);
    map.put("paramsMap", paramsMap);
    map.put("cookie", cookieStr);
    System.out.println(cookie.getName() + "==>" + cookie.getComment());
    return map;
}

// @RequestBody 测试
@ResponseBody
@PostMapping("/save")
public Map<String, Object> requestBodyMethod(@RequestBody String content){
    HashMap<String, Object> map = new HashMap<>();
    map.put("content", content);
    return map;
}

/**
 * @RequestAttribute 测试
 * http://localhost:8080/toSuccess
 */
@GetMapping("/toSuccess")
public String toSuccess(HttpServletRequest request) {
    request.setAttribute("msg", "toSuccess中放入的msg");
    request.setAttribute("code", 200);
    return "forward:/success";
}
@ResponseBody
@GetMapping("/success")
public Map<String, Object> requestAttributeMethod(@RequestAttribute("msg") String msg,
                                                  @RequestAttribute("code") Integer code,
                                                  HttpServletRequest request) {
    Map<String, Object> map = new HashMap<>();
    String reqMsg = (String) request.getAttribute("msg");
    map.put("reqMsg", reqMsg);
    map.put("msg", msg);
    map.put("code", code);
    return map;
}


/**
 * @MatrixVariable 矩阵变量
 * SpringBoot 默认是禁用了矩阵变量的功能，需要手动开启
 * 请求路径 ==> /stuff/1;age=20/2;age=30;brand=xiaomi,huawei,oppo,vivo
 
 * http://localhost:8080/stuff/1;age=20/2;age=30;brand=xiaomi,huawei,oppo,vivo
 */
@ResponseBody
@GetMapping("/stuff/{boss}/{emp}")
public Map<String, Object> matrixVariable(@PathVariable("boss") String boss,
                                          @PathVariable("emp") String emp,
                                          @MatrixVariable(value = "age", pathVar = "boss") String bossAge,
                                          @MatrixVariable(value = "age", pathVar = "emp") String empAge,
                                          @MatrixVariable(value = "brand") List<String> brands){
    HashMap<String, Object> map = new HashMap<>();
    map.put("boss", boss);
    map.put("bossAge", bossAge);
    map.put("empAge", empAge);
    map.put("brands", brands);
    return map;
}

// 手动启用矩阵变量功能
@Bean
public WebMvcConfigurer webMvcConfigurer(){
    return new WebMvcConfigurer() {
        @Override
        public void configurePathMatch(PathMatchConfigurer configurer) {
            UrlPathHelper urlPathHelper = new UrlPathHelper();
            // removeSemicolonContent 在 SpringBoot 底层默认为 true，即默认移除请求路径中分号后面的内容，
            // 则 @MatrixVariable 无法获取到路径分号后的内容，设置为 false 表示不移除分号后面的内容
            // @MatrixVariable 注解的功能即可生效
            urlPathHelper.setRemoveSemicolonContent(false);
            configurer.setUrlPathHelper(urlPathHelper);
        }
    };
}
```

…

---

### 自定义格式转换

重写 Web 配置类中的 addFormatters 方法，并实现 Converter 接口，完成自定义格式转换。

```java
@Configuration(proxyBeanMethods = false)
public class WebConfig {
    @Bean
    public WebMvcConfigurer webMvcConfigurer(){
        return new WebMvcConfigurer() {

            // 注入自定义转换器
            @Override
            public void addFormatters(FormatterRegistry registry) {
                // 将获取到的 String 类型转换为自定义类型
                registry.addConverter(new Converter<String, Pet>() {
                    @Override
                    public Pet convert(String source) {
                        if (StringUtils.hasLength(source)){
                            String[] split = source.split(",");
                            Pet pet = new Pet();
                            pet.setName(split[0]);
                            pet.setAge(Integer.valueOf(split[1]));
                            return pet;
                        }
                        return null;
                    }
                });
            }
        };
    }
}
```

…

---

### 过滤器

1、手动注册

```java
@Slf4j
@Order(value = Ordered.HIGHEST_PRECEDENCE + 1)
@WebFilter(urlPatterns = {"/test/*"})
public class TestFilter implements Filter {
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {}

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        log.info("[TestFilter] doFilter() ==> request {}", request);
        chain.doFilter(request, response); // 注意：处理完后要放行
    }

    @Override
    public void destroy() {}
}
```

```java
@Configuration
public class WebAppConfig implements WebMvcConfigurer {
    @Bean
    public FilterRegistrationBean filterRegistrationBean() {
        FilterRegistrationBean frb = new FilterRegistrationBean();
        frb.setFilter(new TestFilter()); // 注册 Filter
        return frb;
    }
}
```

…

2、@WebFilter + @Component 标注在 Filter 类上

3、当使用嵌入式容器时，可以在主启动类标注 @ServletComponentScan 来扫描标注有 @WebServlet/@WebFilter/@WebListener 的类。

> @ServletComponentScan 对外部容器无效。

…

---

### 拦截器

Filter 和 Interceptor 区别
 *  Filter 是 Servlet 原生组件，可以脱离 Spring 框架运作
 *  Interceptor 是 Spring 定义的接口，只能在 Spring 框架内运行，可以使用 Spring 框架的功能

…

自定义实现

```java
/**
 * 登录拦截器
 *  1. 配置好要拦截和放行的请求
 *  2. 将拦截器注入到容器中
 */

// 1、实现接口
@Slf4j
public class LoginInterceptor implements HandlerInterceptor {

    // 执行目标方法前
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {

        String requestURI = request.getRequestURI();
        log.info("preHandle拦截的请求路径是{}",requestURI);

        // 登录检查逻辑
        
        System.out.println("登陆检查");

        // 返回true放行
        return true;
    }

    // 执行目标方法后
    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
        log.info("postHandle执行{}",modelAndView);
    }

    // 页面渲染后
    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        log.info("afterCompletion执行异常{}",ex);
    }
}

// 2、注入到IOC容器中
@Bean
public WebMvcConfigurer webMvcConfigurer(){
    return new WebMvcConfigurer() {
        @Override
        public void addInterceptors(InterceptorRegistry registry) {
            registry.addInterceptor(new LoginInterceptor())
                // 拦截所有请求，包括静态资源
                .addPathPatterns("/**")
                // 放行请求
                .excludePathPatterns("/", "/login", "/css/**", "/js/**", "/img/**");
        }
    }
}
```

…

---

### 文件上传

…

```yaml
spring:
  servlet:
    multipart:
      max-file-size: 10MB # 单个上传文件大小
      max-request-size: 100MB # 单次请求总上传文件大小
```

```html
<form action="/upload" method="post" enctype="multipart/form-data">
    <input type="text" name="name">
    <input type="file" name="headImg">
    <!-- 多文件上传 -->
    <input type="file" name="photos" multiple>
    <button>upload</button>
</form>
```

```java
@ResponseBody
@PostMapping("/upload")
public Map<String, Object> fileUpload(@RequestParam("name") String name,
                         @RequestPart("headImg") MultipartFile headImg,
                         @RequestPart("photos") MultipartFile[] photos) throws IOException {
  
  	private String filePath = "E:\\DevData\\Data\\upload\\";

    Map<String, Object> map = new HashMap<>();
    map.put("name", name);
    map.put("headImgName", headImg.getName());
    map.put("headImgSize", headImg.getSize());
    map.put("photosLength",photos.length);

    if (!headImg.isEmpty()){
        String filename = headImg.getOriginalFilename();
        headImg.transferTo(new File(filePath + filename));
    }

    for (MultipartFile photo : photos) {
        if (!photo.isEmpty()){
            String filename = photo.getOriginalFilename();
            photo.transferTo(new File(filePath + filename));
        }
    }
    return map;
}
```

…

**原理**

> 稍后补充…

```java
// MultipartAutoConfiguration.java
```

…

---

### 异常处理

全局异常处理

```java
@Slf4j
@ControllerAdvice
public class GlobalExceptionHandler {

    /**
     * @ExceptionHandler 表明用当前方法来处理全局异常
     * value = { ArithmeticException.class, ClassNotFoundException.class }
     * 表示当前方法处理 ArithmeticException 和 ClassNotFoundException
     *
     * 若有多个方法能同时处理某类异常，以精确优先
     * 存在本类和全局异常处理时，本类内的异常处理方法优先处理（就近原则）
     */
    @ExceptionHandler(value = { ArithmeticException.class, ClassNotFoundException.class })
    public String exceptionHandler(Exception e){
        return "出错啦！"+e.getMessage();
    }
}
```

…

**原理**

```java
// ErrorMvcAutoConfiguration.java
```

…

---

### Servlet 容器自定义配置

* ServletRegistrationBean 注册自定义 Servlet
* FilterRegistrationBean 注册自定义 Filter
* ServletListenerRegistrationBean 注册自定义 Listener

…

---

###  WebServer 自定义配置

除了使用 `server.*` 配置，还可以使用代码修改 server 配置：

```java
@Component
public class MyWebServerFactoryCustomizer implements WebServerFactoryCustomizer<ConfigurableWebServerFactory> {
    @Override
    public void customize(ConfigurableWebServerFactory factory) {
        factory.setPort(8888);
    }
}
```

…

---

## SpringBoot 自定义功能

…

### 启动事件监听

`SpringApplication` 在启动过程中会发布多种事件。以下事件是和 SpringApplication 绑定在一起的，发布顺序如下：

* ApplicationStartingEvent 在项目运行时发送
* ApplicationEnvironmentPreparedEvent 在运行环境已准备好，但是未创建上下文时发布
* ApplicationContextInitializedEvent 当 ApplicationContext 准备好且 ApplicationContextInitializers 被调用，未定义任何 Bean 之前
* ApplicationPreparedEvent 在容器刷新之前，Bean 定义被加载之后
* ApplicationStartedEvent 上下文被刷新之后，应用程序和命令行被调用之前
* 
* ApplicationReadyEvent 应用程序和命令行运行之后
* ApplicationFailedEvent 应用启动遇到异常时

> 上面的事件可以从 `org.springframework.boot.context.event` 包中找到

…

下面是和容器状态相关的事件

* org.springframework.boot.availability.AvailabilityChangeEvent 主要用于监听 Spring 容器状态变化的事件

…

以下事件不是和 SpringApplication 绑定在一起的，会在 ApplicationPreparedEvent 之后和 ApplicationStartedEvent 之前发布：

* WebServerInitializedEvent 在 WebServer 准备就绪后
* ContextRefreshedEvent 当 ApplicationContext 刷新后

…

**自定义事件监听器**

```java
@Slf4j
@Component // 将监听器注册到容器中
public class MyApplicationReadyEventListener implements ApplicationListener<ApplicationReadyEvent> {
    {
        log.info("Initial MyApplicationReadyEventListener");
    }
    @Override
    public void onApplicationEvent(ApplicationReadyEvent event) {
        log.info("onApplicationEvent MyApplicationReadyEventListener");
    }
}
```

…

> **事件发布时机**
>
> 有些事件的发布甚至早于 ApplicationContext 创建时，比如 ApplicationPreparedEvent，不能使用 @Bean 来注册对应的事件监听器，只能使用
>
> * SpringApplication#addListeners
> * 或 SpringApplicationBuilder#listeners
>
> 方法来添加事件监听器。
>
> …
>
> 如果想要在程序启动时自动注册监听器，可以在 `META-INF/spring.factories` 文件中添加事件监听器
>
> ```
> org.springframework.context.ApplicationListener=\
> com.example.MyListener,\
> com.example.MyListener2
> ```

…

```java
@Slf4j
public class MyListener implements ApplicationListener<ApplicationPreparedEvent> {
    @Override
    public void onApplicationEvent(ApplicationPreparedEvent event) {
        log.info("onApplicationEvent {}", event);
    }
}
```

手动添加监听器1

```java
@SpringBootApplication
public class CoreFeatureMain {
    public static void main(String[] args) {
        ApplicationListener myListener = new MyListener();
        SpringApplicationBuilder builder = new SpringApplicationBuilder();

        builder.listeners(myListener);

        builder.sources(CoreFeatureMain.class);
        ApplicationContext ac = builder.run(args);
        System.out.println("Start date: " + ac.getStartupDate());
    }
}
```

手动添加监听器2

```java
ApplicationListener myListener = new MyListener();
ConfigurableApplicationContext ac = SpringApplication.run(CoreFeatureMain.class, args);
ac.addApplicationListener(myListener);
```

或者在 `META-INF/spring.factories` 中添加

```
org.springframework.context.ApplicationListener=\
  com.demo.listener.MyListener
```

…

---

### 自定义 Starter

starter 是一组依赖合集，包含自动配置模块内相关的依赖。

…

**内容**

一个自定义 starter 应该包含以下内容：

* autoconfigure 模块，包含自动配置代码和自动配置类文件：*.import 和 spring.factories。
* starter 模块，实际上是一个空的 jar，它的唯一目的是为自动配置模块需要的依赖。

如果自动配置模块比较简单，可以将两个模块合并成一个模块。

…

**命名**

* SpringBoot 官方维护的 starter 都叫 `spring-boot-starter-xxx`
* 为了区分，自定义的 starter 一般命名为 `xxx-spring-boot-starter`

…

**创建**

1、starter 项目添加依赖

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-autoconfigure</artifactId>
        <version>${spring-boot.version}</version>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-autoconfigure-processor</artifactId>
        <version>${spring-boot.version}</version>
        <optional>true</optional>
    </dependency>
</dependencies>
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <configuration>
                <excludes>
                    <exclude>
                        <groupId>org.springframework.boot</groupId>
                        <artifactId>spring-boot-autoconfigure-processor</artifactId>
                    </exclude>
                </excludes>
            </configuration>
        </plugin>

        <!-- 配置当前项目 JDK 版本 -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>3.8.1</version>
            <configuration>
                <source>17</source>
                <target>17</target>
                <encoding>UTF-8</encoding>
            </configuration>
        </plugin>
    </plugins>
</build>
```

2、编写配置代码

```java
@ConfigurationProperties("xxx")
public class XXXProperties {
    private String info;
    private String version;
    // getter and setter
}

public class XXXConfig {
    private XXXProperties props;
    public XXXConfig(XXXProperties props) {
        this.props = props;
    }
    public XXXProperties getProps() {
        return props;
    }
    // getter and setter
}

@AutoConfiguration
@EnableConfigurationProperties(value = {XXXProperties.class})
public class XXXAutoConfiguration {
    @Bean
    @ConditionalOnMissingBean(XXXConfig.class)
    public XXXConfig xxxConfig(XXXProperties props) {
        XXXConfig config = new XXXConfig(props);
        return config;
    }
}
```

3、添加 `META-INF/spring.factories`

```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  com.demo.config.XXXAutoConfiguration
```

4、执行 `mvn clean install`

5、在项目中引用，即可使用

…

---

### 获取 pom 配置

> [expand-properties](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#howto.properties-and-configuration.expand-properties)

如果项目继承 `spring-boot-starter-parent` 可以在配置文件中直接使用 `@...@` 获取到 `pom.xml` 配置的信息

```properties
spring.application.name=@project.artifactId@
app.encoding=@project.build.sourceEncoding@
app.java.version=@java.version@
```

自定义分隔符

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-resources-plugin</artifactId>
    <version>@latest</version>
    <configuration>
        <delimiters>
            <delimiter>@</delimiter>
        </delimiters>
        <useDefaultDelimiters>false</useDefaultDelimiters>
    </configuration>
</plugin>
```

如果没有继承自 `spring-boot-starter-parent`，可以在 `pom.xml` 中添加以下内容

```xml
<build>
  <plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-resources-plugin</artifactId>
    <version>@latest</version>
  </plugin>
  <resources>
      <resource>
          <directory>src/main/resources</directory>
          <filtering>true</filtering>
      </resource>
  </resources>
</build>
```
...

---

### 启动异常

在启动 SpringBoot 应用时，可能会因为端口占用问题出现 `Port 8080 was already in use` 等启动异常。这是借助 FailureAnalyzer 接口实现的功能，启动异常分析。

如果在启动的时候需要进行自定义操作，抛出异常时，可以继承 FailureAnalyzer 的子类 `AbstractFailureAnalyzer<X>`，其中 X 表示要处理的异常。自定义启动异常处理类需要注册到 *resources/META-INF/spring.factories* 文件中，SpringBoot 在启动时会自动扫描该文件，并应用该类。

```java
// 要处理的异常
@Slf4j
public class InvalidIFOException extends RuntimeException {
    String info;
    public String getInfo() {
        return info;
    }
    public InvalidIFOException(String message) {
        super(message);
    }
    public static String buildMessage(String info) {
        return "Invalid info property: "+ info +", it must be start with: IFO";
    }
}

// 自定义启动异常分析器
@Slf4j
public class InvalidIFOFailureAnalyzer extends AbstractFailureAnalyzer<InvalidIFOException> {
    @Override
    protected FailureAnalysis analyze(Throwable rootFailure, InvalidIFOException cause) {
        return new FailureAnalysis(getDescription(cause), getAction(), cause);
    }
    public String getDescription(InvalidIFOException cause) {
        return InvalidIFOException.buildMessage(cause.getMessage());
    }
    public String getAction() {
        return "Change your configuration to start with IFO";
    }
}
```

配置 `spring.factories`

```
org.springframework.boot.diagnostics.FailureAnalyzer=\
  com.demo.exception.InvalidIFOFailureAnalyzer
```

最终效果如下

```shell
***************************
APPLICATION FAILED TO START
***************************

Description:

Invalid info property: aaa, it must be start with: IFO

Action:

Change your configuration to start with IFO
```

> 如果在处理异常时需要用到 BeanFactory 或者 Environment，可以让异常处理类继承 BeanFactoryAware 或 EnvironmentAware

…

---

### 获取启动方法参数

如果想要访问 `SpringApplication#run` 方法中的 args 参数，可以在自定义 Bean 中传入 ApplicationArguments，以获取启动参数。

```java
@Slf4j
@Component
public class MyArgument {
    public MyArgument(ApplicationArguments args) {
        for (String arg : args.getSourceArgs()) {
            log.info("Argument: {}", arg);
        }
    }
}
```

…

---

<br>

## 数据开发

…

### 数据源配置

SpringBoot 默认使用 `HikariDataSource`，如果引入 `spring-boot-starter-jdbc` 或 `spring-boot-starter-data-jpa` 依赖，SpringBoot 会自动配置 HikariDataSource。

**修改数据源**

* 可以通过 `spring.datasource.type` 指定数据源

* 通过编码的方式配置数据源

  ```java
  @Configuration
  public class DataMainConfig {
      @Bean
      public DataSource dataSource() {
          HikariDataSource ds = DataSourceBuilder.create()
                  .username("root")
                  .url("jdbc:mysql://localhost:3306/db_test")
                  .type(HikariDataSource.class).build();
          return null;
      }
  }
  ```

  …

* 引入相关配置，比如` druid-spring-boot-starter`，SpringBoot 会使用 DruidDataSource

…

---

### 多数据源配置

> 什么时候需要使用多数据源？
>
> * 分库操作（读写库分离、不同业务库分离）
> * 多租户系统，一个系统中有多个租户，每个租户有自己的系统

配置类

```java
@Configuration(proxyBeanMethods = false)
public class MyDataSourcesConfiguration {

    @Bean
    @Primary // 保证后续的数据操作中至少能得到一个数据源，也可以使用 @Qualifier 来更精确指定
    @ConfigurationProperties("app.datasource.first")
    public DataSourceProperties firstDataSourceProperties() {
        return new DataSourceProperties();
    }

    @Bean
    @Primary
    @ConfigurationProperties("app.datasource.first.configuration")
    public HikariDataSource firstDataSource(DataSourceProperties firstDataSourceProperties) {
        return firstDataSourceProperties.initializeDataSourceBuilder().type(HikariDataSource.class).build();
    }

    @Bean
    @ConfigurationProperties("app.datasource.second")
    public BasicDataSource secondDataSource() {
        return DataSourceBuilder.create().type(BasicDataSource.class).build();
    }
}
```

配置文件

```properties
app.datasource.first.url=jdbc:mysql://localhost/first
app.datasource.first.username=dbuser
app.datasource.first.password=dbpass
app.datasource.first.configuration.maximum-pool-size=30

app.datasource.second.url=jdbc:mysql://localhost/second
app.datasource.second.username=dbuser
app.datasource.second.password=dbpass
app.datasource.second.max-total=30
```

…

如果存在大于 2 个数据源

```java
@Bean
@Qualifier("dataSource1")
@ConfigurationProperties(prefix = "spring.datasource.ds1")
public DataSource dataSource1() {
    return DataSourceBuilder.create().build();
}
@Bean
@Qualifier("dataSource2")
@ConfigurationProperties(prefix = "spring.datasource.ds2")
public DataSource dataSource2() {
    return DataSourceBuilder.create().build();
}
@Bean
@Qualifier("dataSource3")
@ConfigurationProperties(prefix = "spring.datasource.ds3")
public DataSource dataSource3() {
    return DataSourceBuilder.create().build();
}
// ...
```

…

---

### MySQL

1、引入依赖

* spring-boot-starter-data-jdbc
* mysql-connector-java

2、修改配置文件

```yaml
spring:
  datasource:
    url: jdbc:mysql:///springboot?useSSL=false
    username: root
    password: root
    driver-class-name: com.mysql.jdbc.Driver
```

…

**单独引入数据源**

1、引入依赖

* druid

2、修改配置文件

```yaml
spring:
  datasource:
    url: jdbc:mysql:///springboot?useSSL=false
    username: root
    password: root
    driver-class-name: com.mysql.jdbc.Driver
    # 与在配置类中配置效果等同
    # filters: stat,wall
```

3、修改配置类

```java
@Configuration
public class MainConfig {

    /**
     * 将数据源注入到IOC容器
     * 默认配置是看容器中没有配置DataSource才会注入默认的数据源 @ConditionalOnMissingBean({ DataSource.class })
     */
    @ConfigurationProperties(prefix = "spring.datasource")
    @Bean
    public DataSource dataSource() throws SQLException {
        DruidDataSource dataSource = new DruidDataSource();
        //dataSource.setFilters("stat");开启sql监控功能，开启防火墙
        dataSource.setFilters("stat,wall");
        return dataSource;
    }
    
    /**
     * 配置druid监控页
     */
    @Bean
    public ServletRegistrationBean statViewServlet() {
        StatViewServlet statViewServlet = new StatViewServlet();
        ServletRegistrationBean<StatViewServlet> registrationBean =
                new ServletRegistrationBean<>(statViewServlet, "/druid/*");
        return registrationBean;
    }
    
    /**
     * druid配置web应用监控
     * 配置WebStatFilter用于采集web-jdbc关联监控的数据
     */
    @Bean
    public FilterRegistrationBean webStatFilter() {
        WebStatFilter webStatFilter = new WebStatFilter();
        FilterRegistrationBean<WebStatFilter> registrationBean = new FilterRegistrationBean<>(webStatFilter);
        registrationBean.setUrlPatterns(Arrays.asList("/*"));
        registrationBean.addInitParameter("exclusions","*.js,*.gif,*.jpg,*.png,*.css,*.ico,/druid/*");
        return registrationBean;
    }
}
```

…

**starter 方式引入数据源**

1、引入依赖

* druid-spring-boot-starter

2、修改配置文件

```yaml
# 扩展配置项
spring:
  datasource:
    url: jdbc:mysql:///springboot?useSSL=false
    username: root
    password: root
    driver-class-name: com.mysql.jdbc.Driver
    # 与在配置类中配置效果等同
    # filters: stat,wall
    druid:
      # 监控Spring容器中的bean
      aop-patterns: com.demo.*
      filters: stat,wall,slf4j # 开启监控功能和防火墙功能，并用slf4j记录下来
      filter:
        stat: # 对filter启用功能的详细配置
          enabled: true
          # 慢查询sql时间
          slow-sql-millis: 1000
          # 打印慢查询sql
          log-slow-sql: true
        wall:
          enabled: true
          config:
            # drop-table-allow: false
            # 允许发送查询的sql
            # selelct-allow: true
            # 不允许发送更新的sql
            # update-allow: false
      stat-view-servlet: # 启用监控页功能
        enabled: true
        # 设置druid登录页的账号密码
        login-username: admin
        login-password: admin
        # 是否启用重置按钮
        reset-enable: false
      web-stat-filter: # 监控web应用
        enabled: true
        url-pattern: /*
        exclusions: "*.js,*.gif,*.jpg,*.png,*.css,*.ico,/druid/*"
```

3、自动配置类 DruidDataSourceAutoConfigure

…

---

### MyBatis

1、引入依赖

* mybatis-spring-boot-starter

2、修改配置

```yaml
mybatis:
  # config-location和configuration不能同时存在，不能同时使用配置文件（mybatis-config.xml）和在application配置文件中设置配置项
  # config-location: classpath:mybatis/mybatis-config.xml
  mapper-locations: classpath:/mapper/*.xml
  type-aliases-package: com.demo.boot.bean
  # 指定mybatis全局配置文件中的相关配置项
  configuration: 
    map-underscore-to-camel-case: true
```

…

自动配置类 MybatisAutoConfiguration

```java
@Configuration
@ConditionalOnClass({ SqlSessionFactory.class, SqlSessionFactoryBean.class })
@ConditionalOnSingleCandidate(DataSource.class)
@EnableConfigurationProperties(MybatisProperties.class) // 绑定MyBatis配置项类
@AutoConfigureAfter({ DataSourceAutoConfiguration.class, MybatisLanguageDriverAutoConfiguration.class })
public class MybatisAutoConfiguration implements InitializingBean {
  
  // 自动配置SqlSessionFactory
  @Bean
  @ConditionalOnMissingBean
  public SqlSessionFactory sqlSessionFactory(DataSource dataSource /*使用容器中的数据源*/) throws Exception {
      //...
  }
	
  // 自动配置SqlSessionTemplate，其中组合了SqlSession
  @Bean
  @ConditionalOnMissingBean
  public SqlSessionTemplate sqlSessionTemplate(SqlSessionFactory sqlSessionFactory) {
      //...
  }
    
    // 只要在操作MyBatis的接口标注了@Mapper注解，就会被自动扫描
    
}

// 自动扫描指定的包，表明该包下的类全为 Mapper
@MapperScan("com.demo.boot.mapper")
```



**<u>3 - 开启缓存（选择性开启）</u>**

1、**一级缓存**

> Mabits 默认开启一级缓存

Mybatis 的一级缓存是指 SQLSession，一级缓存的作用域是 SQlSession。在同一个 SqlSession 中，执行相同的 SQL 查询时，第一次会去查询数据库，并写在缓存中，第二次会直接从缓存中取。每次查询会先去缓存中找，如果找不到，再去数据库查询，然后把结果写到缓存中。当执行 SQL 时两次查询中间发生了增删改的操作，则 SQLSession 的缓存会被清空。Mybatis 的内部缓存使用一个 HashMap，key 为`hashcode+statementId+sql`语句。value 为查询出来的结果集映射成的 Java 对象。SqlSession 执行 insert、update、delete 等操作 commit 后会清空该 SQLSession 缓存。

2、**二级缓存**

二级缓存是 Mapper 级别的，Mybatis 默认是没有开启二级缓存的。第一次调用 Mapper 下的 SQL 去查询用户的信息，查询到的信息会存放在该 Mapper 对应的二级缓存区域。第二次调用 namespace 下的 Mapper 映射文件中，相同的 SQL去查询用户信息，会去对应的二级缓存内取结果。

…

---

### MyBatis Plus

1、依赖

* mybatis-plus-boot-starter

2、修改配置文件

```yaml
mybatis-plus:
  configuration:
    map-underscore-to-camel-case: true
  mapper-locations: #mapper-locations 默认位置为 classpath:mapper/*.xml
  type-aliases-package: com.demo.boot.bean
```

3、配置类

```java
@Configuration
public class MybatisPlusConfig {

    //分页配置
    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {

        MybatisPlusInterceptor mybatisPlusInterceptor = new MybatisPlusInterceptor();

        PaginationInnerInterceptor paginationInnerInterceptor = new PaginationInnerInterceptor();

        // 设置请求的页面大于最大页后操作， true调回到首页，false 继续请求  默认false
        // paginationInterceptor.setOverflow(false);
        // 设置最大单页限制数量，默认 500 条，-1 不受限制
        // paginationInterceptor.setLimit(500);
        // 开启 count 的 join 优化,只针对部分 left join

        paginationInnerInterceptor.isOptimizeJoin();
        paginationInnerInterceptor.setOverflow(true);
        mybatisPlusInterceptor.addInnerInterceptor(paginationInnerInterceptor);

        return mybatisPlusInterceptor;
    }

}
```

4、Mapper 类

```java
// 标注@Mapper或在配置类@MapperScan(basePackages = "com.demo.boot.mapper") 自动扫描指定包下的类
@Mapper
public interface AccountMapper extends BaseMapper<Account> {}

// Service接口
public interface AccountService extends IService<Account> {}

// Service实现类
@Service
public class AccountServiceImpl extends ServiceImpl<AccountMapper, Account> implements AccountService {}
```

5、测试类

```java
@Test
public void testPage() {
    // 设置分页信息
    Page<Account> page = new Page<>();
    page.setPages(1L);
    page.setSize(5l);
    // 设置过滤条件
    QueryWrapper<Account> queryWrapper = new QueryWrapper<>();
    // queryWrapper.ge("age", 30); 筛选岁数大于30的数据
    Page<Account> accountPage = accountService.page(page, queryWrapper);
    accountPage.getRecords().forEach(System.out::println);
}
```

…

**自动配置类**

```java
@Configuration
@ConditionalOnClass({SqlSessionFactory.class, SqlSessionFactoryBean.class})
@ConditionalOnSingleCandidate(DataSource.class)
/**
 * 配置项绑定
 * mapperLocations默认值 classpath*:/mapper/**/*.xml
 * 任意包的类路径下的所有mapper文件夹下任意路径下的所有xml都是sql映射文件。建议以后sql映射文件，放在 mapper下
 */
@EnableConfigurationProperties(MybatisPlusProperties.class)
@AutoConfigureAfter({DataSourceAutoConfiguration.class, MybatisPlusLanguageDriverAutoConfiguration.class})
public class MybatisPlusAutoConfiguration implements InitializingBean {
    
    //SqlSessionFactory 自动配置好。底层是容器中默认的数据源
    
    //SqlSessionTemplate 自动配置好
}

//@Mapper 标注的接口也会被自动扫描
//建议直接 @MapperScan("com.demo.boot.mapper") 批量扫描
//只需要自己编写的Mapper继承 BaseMapper 就可以拥有Mybatis-Plus的功能
```

…

---

### Spring Data Jpa

1、引入依赖

* spring-boot-starter-data-jpa
* mysql-connector-java

2、修改配置文件

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/boot-jpa-demo?useSSL=false
    username: root
    password: root
    driver-class-name: com.mysql.jdbc.Driver
  jpa:
    database-platform: org.hibernate.dialect.MySQL5InnoDBDialect
    hibernate:
      ddl-auto: update # 每次运行程序，没有表格会新建表格，表内有数据不会清空，只会更新(推荐)
    show-sql: true # 配置在日志中打印出执行的 SQL 语句信息
    # ddl-auto:create 每次运行该程序，没有表格会新建表格，表内有数据会清空
    # ddl-auto:create-drop 每次程序结束的时候会清空表
    # ddl-auto:update 每次运行程序，没有表格会新建表格，表内有数据不会清空，只会更新(推荐)
    # ddl-auto:validate 运行程序会校验数据与数据库的字段类型是否相同，不同会报错
```

3、编写实体类

```java
@Data
@Table(name = "tb_user")
@Entity
@Getter
@Setter
@AllArgsConstructor
@NoArgsConstructor
public class User implements Serializable {
    private static final long serialVersionUID = 1L;
    @Id
    @GeneratedValue(strategy = GenerationType.AUTO, generator = "") // 主键生成策略
    private Long id;
    @Column(name="name")
    private String name;
    private Integer age;
}
```

4、编写DAO接口

```java
// JpaRepository<实体类类型，主键类型> 用来完成基本CRUD操作
// JpaSpecificationExecutor<实体类类型> 用于复杂查询（分页等查询操作）
@Repository
public interface UserDao extends JpaRepository<User, Long>, JpaSpecificationExecutor<User> {

    List<User> findByAgeAfter(Integer age);

    // 分页查询 1，使用原生 sql 语句
    @Query(value = "select * from tb_user limit :page, :size ", nativeQuery = true)
    List<User> getByPage(@Param("page") int page,@Param("size") int size);

    // 分页查询 2，方法已经在 PagingAndSortingRepository 中定义，直接使用即可
    // Page<User> findAll(Pageable pageable);

    // 使用 jpql 方式查询
    // 当使用 @Query 注解时，约定方法会失效。如 findByName 方法实际查询条件是 @Query 中的语句，而非方法名中的ByName
    // from 后面紧接着的是使用 @Table 标注的对应【实体类名】，而【非数据库表名】
    // ?1 方法中的表示第一个参数
    @Query(value = "from User where name = ?1")
    User jpqlFindByName(String name);

    // 使用 jpql 方式更新
    // 使用 jpql 更新时需要加上 @Modifying 注解
    // 使用 jpql 进行更新或者删除操作时，需要加上 @Transactional 注解
    // 更新、删除操作方法的返回值只能是void或int/Integer
    @Query(value = "update User set age = ?1 where name = ?2")
    @Modifying
    @Transactional
    int jpqlUpdate(Integer age, String name);

}

```

5、启动类

```java
@SpringBootApplication
public class MainApplication {
    public static void main(String[] args) {
        SpringApplication.run(MainApplication.class, args);
    }
}
```

6、测试类

```java
@SpringBootTest
class MainApplicationTests {

    @Autowired
    UserDao userDao;

    @Test
    void testFindAll() {
        System.out.println(userDao.findAll());
    }

    @Test
    public void testFindByCondition() {
        System.out.println(userDao.findByAgeAfter(25));
    }

    // 单条件排序
    @Test
    public void testFindAllAndSort() {
        System.out.println(userDao.findAll(Sort.by(Sort.Direction.ASC, "name")));
    }

    // 多条件排序
    @Test
    public void testFindAllAndMultiSort() {
        Sort.Order idOrder = Sort.Order.asc("id");
        Sort.Order nameOrder = Sort.Order.desc("name");
        System.out.println(userDao.findAll(Sort.by(idOrder, nameOrder)));
    }

    // 根据值长度排序
    @Test
    public void testSortByLength() {}

    @Test
    public void testJpqlQuery() {
        System.out.println(userDao.jpqlFindByName("lgn"));
    }

    @Test
    public void testJpqlUpdate() {
        System.out.println(userDao.jpqlUpdate(18, "lgn"));
    }

    @Test
    public void testFindByPage() {
        System.out.println(userDao.getByPage(0, 5));
    }

    @Test
    public void testFindByPage2() {
        // 分页
        PageRequest pageRequest = PageRequest.of(0, 5);
        // 排序并分页
        PageRequest pageRequest2 = PageRequest.of(0, 5, Sort.by("name").ascending());
        Page<User> userPage = userDao.findAll(pageRequest);
        System.out.println(userPage.getContent());
        System.out.println(userPage.getTotalPages());
        System.out.println(userPage.getTotalElements());
    }

    @Test
    public void testSave() {
        User u = new User(null, "alen", 33);
        System.out.println(userDao.save(u));
    }

    @Test
    public void testUpdate() {
        User u = new User(1l, "lgn", 21);
        System.out.println(userDao.save(u));
    }
}

```

…

**Specifications 动态查询和多表查询**

> [spring data jpa Specifications动态查询、多表查询](https://juejin.cn/post/6999796039310016520)

…

---

### Redis

…

1、引入依赖

* spring-boot-starter-data-redis
* Redis 的 Java 客户端 jedis

2、配置文件

```yaml
spring:
  redis:
  	# 只配置url或者单独配host和port
  	# url: redis://user:password@example.com:6379
  	host: 
  	port:
  	# 指定客户端jedis or lettuce（默认）
  	client-type: jedis
    jedis:
      pool:
        max-active: 10
        min-idle: 5
```

…

**自动配置类**

```java
// RedisAutoConfiguration.java

@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(RedisOperations.class)
@EnableConfigurationProperties(RedisProperties.class) // 配置绑定RedisProperties
/**
 * Lettuce，Jedis 用来操作 redis 的 Java 客户端
 * LettuceConnectionConfiguration，JedisConnectionConfiguration 自动准备两个连接工厂，需要哪个启用哪个
 */
@Import({ LettuceConnectionConfiguration.class, JedisConnectionConfiguration.class })
public class RedisAutoConfiguration {}
```

…

---

<br>

## 远程接口调用

…

### 原生 Http 请求

```java
@RequestMapping("/doPostGetJson")
public String doPostGetJson() throws ParseException {
   // 此处将要发送的数据转换为json格式字符串
   String jsonText = "{id: 1}";
   JSONObject json = (JSONObject) JSONObject.parse(jsonText);
   JSONObject sr = doPost(json);
   System.out.println("返回参数: " + sr);
   return sr.toString();
}

public static JSONObject doPost(JSONObject data) {
   HttpClient client = HttpClients.createDefault();
   // 要调用的接口方法
   String url = "http://localhost:8090/getJson";
   HttpPost post = new HttpPost(url);
   JSONObject jsonObject = null;
   try {
      StringEntity s = new StringEntity(data.toString());
      s.setContentEncoding("UTF-8");
      s.setContentType("application/json");
      post.setEntity(s);
      post.addHeader("content-type", "text/xml");
      HttpResponse res = client.execute(post);
      String response1 = EntityUtils.toString(res.getEntity());
      System.out.println(response1);
      if (res.getStatusLine().getStatusCode() == HttpStatus.SC_OK) {
         String result = EntityUtils.toString(res.getEntity());// 返回json格式: 
         jsonObject = JSONObject.parseObject(result);
      }
   } catch (Exception e) {
      throw new RuntimeException(e);
   }
   return jsonObject;
}
```

…

---

### RestTemplate

**Get 请求**

1）`getForEntity(Stringurl,Class responseType,Object…urlVariables)`

```java
// http://USER-SERVICE/user?name={name)
RestTemplate restTemplate=new RestTemplate();
Map<String,String> params=new HashMap<>();
params.put("name","dada");  // 参数 name-value
ResponseEntity<String> responseEntity=restTemplate.getForEntity("http://USERSERVICE/user?name={name}",String.class,params);
  
```

…

2）`getForEntity(URI url,Class responseType)`

> 该方法使用 URI 对象来替代之前的 url 和 urlVariables 参数来指定访问地址和参数绑定。URI 是JDK `java.net` 包下的一个类，表示一个统一资源标识符（Uniform Resource Identifier）引用。

```java
RestTemplate restTemplate=new RestTemplate();
UriComponents uriComponents=UriComponentsBuilder.fromUriString("http://USER-SERVICE/user?name={name}")
    .build()
    .expand("dodo")
    .encode();
URI uri=uriComponents.toUri();
ResponseEntity<String> responseEntity=restTemplate.getForEntity(uri,String.class).getBody();
```

…

3）`getForObject()`

> `getForObject()` 方法可以理解为对 `getForEntity()` 的进一步封装，它通过 HttpMessageConverterExtractor 对 HTTP 的请求响应体 body 内容进行对象转换，实现请求直接返回包装好的对象内容。

```java
getForObject(String url,Class responseType,Object...urlVariables)
getForObject(String url,Class responseType,Map urlVariables)
getForObject(URI url,Class responseType)
```

…

**Post 请求**

> Post 请求提供有三种方法，`postForEntity()`、`postForObject()`和`postForLocation()`

```java
RestTemplate restTemplate=new RestTemplate();
User user=newUser("didi",30);
ResponseEntity<String> responseEntity=restTemplate.postForEntity("http://USER-SERVICE/user",user,String.class); // 提交的body内容为user对象，请求的返回的body类型为String
String body=responseEntity.getBody();

// postForEntity() 存在如下三种方法的重载

postForEntity(String url,Object request,Class responseType,Object... uriVariables)
postForEntity(String url,Object request,Class responseType,Map uriVariables)
postForEntity(URI url,Object request，Class responseType)
```

…

---

### Feign 远程调用

1、引入依赖

* spring-cloud-starter-feign

2、启动类添加 `@EnableFeignClients `，被调用方主启动类添加 `@EnableDiscoveryClient`

3、编写 Feign 接口

```java
// 最终请求路径为 http://10.2.1.148:3333/decision/person
@FeignClient(url = "http://10.2.1.148:3333/",name="engine")
public interface DecisionEngineService {
　　@RequestMapping(value="/decision/person",method= RequestMethod.POST)
　　public JSONObject getEngineMesasge(@RequestParam("uid") String uid,@RequestParam("productCode") String productCode);
}
```

4、在代码中调用

```java
@Autowired
private DecisionEngineService decisionEngineService ;
// ...
decisionEngineService.getEngineMesasge("uid" ,  "productCode");
```

…

---

<br>

## API 限流

> 高并发的三板斧：缓存，降级和限流。

### LimitRater

> 下文采用 guava 的 LimitRater 实现，是本地限流。若要实现分布式限流可以使用 Redis+Lua+AOP。

…

1、引入依赖

```xml
<dependency>
  <groupId>io.github.forezp</groupId>
  <artifactId>distributed-limit-core</artifactId>
  <version>1.0.4</version>
</dependency>
```

2、配置文件

```yaml
# 本地限流，只需要配置limit.type=local
# limit.type: local
# 采用Redis限流，配置limit.type=redis，以及redis的配置
limit.type: redis

spring:
  redis:
    host: localhost
    port: 6379
#    password: ee
    database: 1
    pool:
      max-active: 8
      max-wait: -1
      max-idle: 500
      min-idle: 0
    timeout: 0
```

3、在 Controller 对应的方法上加 `@Limit` 注解

```java
@RestController
public class LimitController {
    @GetMapping("/test")
    // identifier 用作身份识别
    // key 为限流的key
    // limtNum 为限制的次数
    // seconds 为多少秒
    // limtNum + seconds 即表示在seconds秒时间内最大的请求次数
    // identifier 和 key 支持 Spel 表达式
    // 如果仅API维度，identifier 为空即可；如果仅用户维度，key为空即可。
    @Limit(identifier = "forezp", key = "test", limtNum = 10, seconds = 1)
    public String test() {
        return "11";
    }
}
```

4、如果以请求用户维度去限流，可以动态的设置的 identifier 的值

```java
@Component
public class IndentifierInterceptor extends HandlerInterceptorAdapter {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        // 获取用户的信息，比如解析Token获取用户名，
        // 这么做主要是在基于@Limit注解在Controller的时候，能都动态设置identifier信息
        // 从而以用户维度进行限流
        String identifier = "forezp";
        IdentifierThreadLocal.set( identifier );
        return true;
    }

}
```

5、拦截器实现 

```java
@Component
public class WebInterceptor extends HandlerInterceptorAdapter {

    private Map<String, LimitEntity> limitEntityMap = Maps.newConcurrentMap();

    @Autowired
    LimitExcutor limitExcutor;

    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        //限流2个维度:  用户和api维度
        //比如用户名
        String identifier = "forezp";
        //api维度
        String key = request.getRequestURI();
        String composeKey = KeyUtil.compositeKey( identifier, key );
        LimitEntity limitEntity = limitEntityMap.get( composeKey );
        if (limitEntity == null) {
            limitEntity = new LimitEntity();
            limitEntity.setIdentifier( identifier );
            limitEntity.setKey( key );
            //这可以在数据库中配置或者缓存中读取，在这里我写死
            limitEntity.setSeconds( 1 );
            limitEntity.setLimtNum( 10 );
            limitEntityMap.putIfAbsent( composeKey, limitEntity );
        }
        if (!limitExcutor.tryAccess( limitEntity )) {
            throw new LimitException( "you fail access, cause api limit rate ,try it later" );
        }

        return true;
    }
}
```

6、注册拦截器

```java
@Configuration
public class WebConfig extends WebMvcConfigurerAdapter {

    @Autowired
    IndentifierInterceptor indentifierInterceptor;

    @Autowired
    WebInterceptor webInterceptor;

    /**
     * 注册 拦截器
     */
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor( indentifierInterceptor );
        registry.addInterceptor( webInterceptor );
    }

}
```

…

---

<br>

## Actuator 监控

…

0、依赖

* spring-boot-starter-actuator

1、配置类

```yaml
# 扩展配置
management:
  endpoints:
    # 默认为true，开启所有的监控endpoint
    enabled-by-default: false
    web:
      exposure:
        # 以web方式监控所有endpoint
        include: "*"
  endpoint: # 对某个端点进行配置
    health:
      # 按需开启端点
      enabled: true
      show-details: always # 显示所有健康信息
    info:
      enabled: true
    metrics:
      enabled: true
```

2、访问`http://localhost:8080/actuator` 查看所有暴露的 endpoint

…

---

### Endpoint 信息

1、健康检查端点 `health ` ，用于检查应用健康状况。`http://localhost:8080/actuator/health`

2、查看应用运行指标 `metrics ` ，展示详细的，层级的空间指标信息。`http://localhost:8080/actuator/metrics`

3、查看应用某项指标详情信息`http://localhost:8080/actuator/metrics/http.server.requests`

…

---

### 自定义 Endpoint

1、编写配置文件

```yaml
info:
  appName: boot-demo-web
  appVersion: 1.0.0
  projectName: @project.artifactId@
  projectVersion: @project.version@
```

2、自定义 Info 类

```java
@Component
public class AppInfoContributor implements InfoContributor {
    @Override
    public void contribute(Info.Builder builder) {
        builder.withDetail("hello", "world")
                .withDetail("msg", "自定义Info信息")
                .withDetails(Collections.singletonMap("hhh", "some msg..."));
    }
}
```

3、自定义 `health` 信息

```java
@Component //将组件注入容器
public class MyCusHealthIndicator extends AbstractHealthIndicator {

    // 检查健康信息
    @Override
    protected void doHealthCheck(Health.Builder builder) throws Exception {

        Map<String, Object> map = new HashMap<>();
        if (true){
            // 写法1
            builder.up();
            map.put("healthMsg", "你的磁盘文件贼健康~");
            map.put("ms", 100);
        }else {
            // 写法2
            builder.status(Status.OUT_OF_SERVICE);
            map.put("unHealthMsg", "你的磁盘文件有点不健康哦~");
            map.put("ms", 1000);

        }

        // 检查完成返回见结果信息
        builder.withDetail("code", 200)
                .withDetails(map);
    }
}
```

4、自定义 `metrics` 监控指标

```java
// 将 getByPage 方法的调用次数作为自定义指标注册到 metrics
@Service
public class AccountServiceImpl extends ServiceImpl<AccountMapper, Account> implements AccountService {

    @Autowired
    AccountMapper accountMapper;

    Counter counter;

    public AccountServiceImpl(MeterRegistry meterRegistry) {
        // 将指标注册到 actuator/metrics
        counter = meterRegistry.counter("accountServiceImpl.getByPage.count");
    }

    @Override
    public List<Account> getByPage(Long pageNum, Long pageSize) {
        Page<Account> page = new Page<>();
        page.setPages(pageNum);
        page.setSize(pageSize);
        Page<Account> accountPage = accountMapper.selectPage(page, null);

        // counter计数+1
        counter.increment();

        return accountPage.getRecords();
    }
}
```

5、自定义 Endpoint

```java
@Component
@Endpoint(id = "my-custom") // 自定义端点名
public class MyCustomEndpoint { // 自定义端点默认暴露，不用配置暴露属性

    /**
     * @ReadOperation 表明是当前方法端点的读操作
     * getSomeInfo() 表明当前方法是一个属性，不能出现参数，不能传参
     * http://localhost:8080/actuator/my-custom
     */
    @ReadOperation
    public Map getSomeInfo() {
        return Collections.singletonMap("customInfo", "this is the msg from my custom info");
    }

    /**
     * @WriteOperation 表明当前方法是端点的写操作
     * 方法同样不能传参
     */
    @WriteOperation
    public void setSomeInfo() {
        System.out.println("setting some infos now...");
    }

}
```

…

---

<br>

## 单元测试

**整合 JUnit5**

```java
//JUnit5Test.java

@DisplayName("JUnit5-功能测试")
public class JUnit5Test {

    @BeforeAll
    public static void testBeforeAll() {
        System.out.println("BeforeAll 在【所有】方法开始之前运行，方法需要使用static关键字修饰");
    }

    @BeforeEach
    public void testBeforeEach() {
        System.out.println("BeforeEach 【每个】测试方法开始之前运行");
    }

    @DisplayName("DisplayName测试")
    @Timeout(value = 600, unit = TimeUnit.MILLISECONDS)
    @Test
    public void testDisplayName() {
        System.out.println("DisplayName 测试");
    }

    @Disabled //进行单元测试时禁用此方法
    @Test
    public void test() {
        System.out.println("test junit5...");
    }

    @RepeatedTest(value = 5)
    public void testRepeat() {
        System.out.println("repeat 5 times");
    }

    @DisplayName("测试简单断言")
    @Test
    public void testSimpleAssertions() {
        int res = sum(2, 3);
        /**
         * 断言测试 前面的断言失败，后面的断言不会继续执行
         * assertEquals 判断两个值是否相等
         * 参数[1]为期望值
         * 参数[2]为实际返回值
         * 参数[3]为自定义错误提示信息
         * 若为true则运行成功，若为false则会报错
         * @author gnl
         * @date 2021/2/2 11:30
         */
        //Assertions.assertEquals(6, res);
        //Assertions.assertEquals(6, res, "求和运算逻辑测试失败");

        // 判断两个对象引用是否相等
        String originStr = "abc";
        String a = new String("abc");
        String str = "abc";
        assertSame(originStr, str, "两个对象引用不相等");
        assertSame(originStr, a, "两个对象引用不相等");

    }

    int sum(int a, int b) {
        return a+b;
    }

    @Test
    public void testArrayAssertions() {
        assertArrayEquals(new int[] {1,2}, new int [] {2, 2});
    }

    @DisplayName("测试组合断言")
    @Test
    public void testCombinedAssertions() {
        // 所有断言全部需要成功
        Assertions.assertAll("testAll",
                () -> assertTrue(true && true, "结果不为true"),
                () -> assertEquals(1 ,2, "两个值不相等"));
    }

    @DisplayName("测试异常断言")
    @Test
    public void testExceptedAssertions() {
        assertThrows(ArithmeticException.class, () -> { int i = 10 / 1; } ,"ex提示信息");
    }

    @DisplayName("测试前置条件")
    @Test
    public void testAssumption() {
        //假设失败执行整体单元测试时会和Disable标注的方法一样跳过
        Assumptions.assumeTrue(false, "结果不是true抛出的信息");
    }

    @DisplayName("参数化测试")
    @ParameterizedTest
    @ValueSource(ints = {1,2,3,4,5,6})
    public void testParameterized(int i) {
        System.out.println(i);
    }

    @DisplayName("参数化测试，参数值从方法中获得")
    @ParameterizedTest
    @MethodSource(value = {"stringProvider"})
    public void testParameterize2(String str) {
        System.out.println(str);
    }

    //作为参数化测试的MethodSource，需要是static修饰的方法
    static Stream<String> stringProvider() {
        return Stream.of("apple", "banana", "lemon");
    }

    @AfterEach
    public void testAfterEach() {
        System.out.println("AfterEach 【每个】测试方法之后运行");
    }

    @AfterAll
    public static void testAfterAll() {
        System.out.println("AfterAll 在【所有】方法完成之后运行，方法需要使用static关键字修饰");
    }

}
```

…

---

<br>

## 第三方整合

### 整合 Log4j2

> [log4j2-extensions](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#features.logging.log4j2-extensions)

…

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <!-- 排除掉logging，不使用logback，改用log4j2 -->
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-logging</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<!-- log4j2 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-log4j2</artifactId>
</dependency>
<!-- 需要添加 log4j-spring-boot 依赖，才能正常获取到 ${spring:spring.application.name} -->
<dependency>
    <groupId>org.apache.logging.log4j</groupId>
    <artifactId>log4j-spring-boot</artifactId>
</dependency>
```

```yaml
# application.yaml，配置文件中配置扫描 log4j2.xml 配置文件
logging:
  config: classpath:log4j2-spring.xml
```

…

配置文件 `log4j2-spring.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!--
    Because the standard log4j2.xml configuration file is loaded too early, you cannot use extensions in it.
    You need to either use log4j2-spring.xml or define a logging.config property.
-->

<!-- 日志级别从小到大 ALL < TRACE < DEBUG < INFO < WARN < ERROR < FATAL < OFF -->

<Configuration status="WARN"> <!-- 默认输出 warn 及以上 -->
    <properties>
        <Property name="log_path" value="./logs" />
        <Property name="applicationName" value="${spring:spring.application.name}" />
        <Property name="file_suffix" value="log" />
        <Property name="full_name" value="${log_path}/${applicationName}.${file_suffix}" />
    </properties>

    <!-- 定义日志 appender -->
    <appenders>
        <!-- 控制台 -->
        <console name="Console" target="SYSTEM_OUT">
            <!-- 日志输出格式 -->
            <PatternLayout pattern="%d{yyyy-MM-dd HH:mm:ss.SSS} [%t] %-5level %logger{36} --> %msg%n"/>
        </console>

        <!-- all 及以上 -->
        <RollingFile name="AllAppender"
                     fileName="${full_name}"
                     filePattern="${log_path}/$${date:yyyy-MM}/all-%d{yyyy-MM-dd}-%i.log.gz">
            <!-- 日志格式 -->
            <PatternLayout>
                <pattern>%d %p %C{} [%t] %m%n</pattern>
            </PatternLayout>
            <Policies>
                <!-- 日志文件切分参数 -->
                <!-- 日志基础文件大小，超过就触发日志文件滚动更新 -->
                <SizeBasedTriggeringPolicy size="100 MB"/>
                <!-- 日志文件滚动更新间隔时间 -->
                <TimeBasedTriggeringPolicy interval="1" />
            </Policies>
            <!-- 日志文件个数上限，默认为 7，超过大小后会被压缩：依据 filePattern 中的 %i -->
            <DefaultRolloverStrategy max="100"/>
        </RollingFile>
    </appenders>

    <loggers> <!-- 定义 logger 并引入，appender 才会生效 -->
        <root level="debug"> <!-- 指定项目的根日志，如果子 appenderRef 没有单独指定 logger，会使用 root 作为默认的日志输出 -->
            <appenderRef ref="AllAppender" /> <!-- 默认输出 debug 及以上 -->
            <appenderRef ref="Console" level="info" /> <!-- 控制台输出 info 日志 -->
        </root>
    </loggers>
</Configuration>
```

…

---

<br>

### 整合 JWT

> JWT（JSON Web Token），通过 JSON 形式作为 Web 应用中的令牌，用于在各方之间安全地将信息作为 JSON 对象传输。在传输的过程中还可以完成数据加密、签名等操作

…

#### 传统 Session-Cookie 认证

**1、认证流程**

1）由于 http 协议本身是一种无状态的协议，这意味着如果用户向服务端提供了用户名和密码来进行用户认证之后，下一次请求时，用户还需要再一次进行用户认证才行；

2）因 http 协议无状态，服务端并不能知道是哪个用户发出的请求，所以为了让服务端能识别是哪个用户发出的请求，我们只能在服务器存储一份用户的登录信息；

3）这份登录信息会在<mark>第一次响应</mark>时传递给浏览器，并在浏览器端保存为 cookie，以便下次发送请求时携带着发送到服务器。这样服务器就能识别请求来自哪个用户了

…

**2、存在问题**

1）每个特定域名下生成的 cookie 有数量限制（对不同的浏览器来说不同），cookie存储的长度不超过 4kb

2）基于 cookie 来进行用户识别，如果 cookie 被截获，用户就容易受到跨站请求伪造的攻击

3）Cookie 跨站是不能共享的，这样的话如果你要实现多应用（多系统）的单点登录（SSO），使用 Cookie 来做需要的话就很困难了

4）Cookie 是存储在客户端的，而 Session 是存储在服务器的。每个用户认证之后都要在服务器做一个记录，将用户信息保存为 session，通常来说 session 都是保存在内存中，随着认证用户的增多，服务端的开销会明显增大

5）用户认证之后，服务端做认证记录，如果认证记录被保存在内存中的话，意味着<mark>用户下次请求还必须要在这台服务器</mark>上，这样才能拿到授权的资源。这样在分布式应用上，响应的限制了负载均衡的能力，也限制了应用的扩展能力

6）多个服务之间需要解决 session 共享问题，不适用于微服务

…

#### JWT 认证

…

![image-20210209121037489](./assets/image-20210209121037489.png)

**1、认证流程**

1）前端通过 Web 表单将自己的用户名和密码发送到后端的接口。这个过程一般是一个 http 的 POST 请求，建议的方式是通过 SSL 加密的传输（ https 协议），从而避免敏感信息被嗅探

2）后端核对用户密码成功后，将用户的 id 和其他信息作为 JWT 的 Payload（负载），将其与头部分别进行 Base64 编码拼接后进行签名操作，形成一个 JWT（Token）。形成的 JWT 就是一个由 `header.payload.signature` 三部分形成的字符串

3）后端将 JWT 字符串作为登录成功的返回结果返回给前端。前端可以将返回的结果保存到 `localStorage` 或 `sessionStorage` 上，退出登录时前端删除保存的 JWT（后端同时将 JWT 设置进黑名单）即可

4）前端在每次请求时将 JWT 放入 Http 请求的 header 中的 Authoriztion 位（解决 XSS 和 XSRF）

5）后端检查 `Authoriztion` 是否存在 JWT，若存在验证 JWT 的有效性。例如，检查签名是否正确；检查 Token 是否过期；检查 Token 的接收方是否是自己（可选）

6）验证通过后后端使用 JWT 中包含的用户信息进行其他逻辑操作，进行与数据库的操作，返回相应的结果。并将 JWT 的过期时间刷新

…

**2、优势**

1、简洁（Compact），可以通过 URL，POST 参数或者在 http 的 header 发送，因为数据量小，传输速度也很快

2、自包含（Self-contained），负载中包含了用户所需要的所有信息（用户ID），避免了多次查询数据库

3、Token 是以 JSON 加密的形式保存在客户端的，所以 JWT 是跨语言的，原则上任何 Web 形式都支持

4、不需要在服务端保存会话信息，特别适用于分布式微服务

5、在移动端原生请求是没有 cookie 之说的，而 sessionid 依赖于 cookie，sessionid 就不能用 cookie 来传了，如果用 token 的话，由于它是随着 header 的 authoriize 传过来的，也就不存在此问题，换句话说token 天生支持移动平台，可扩展性好

…

**3、存在问题**

1）token太长。token 是 header，payload 编码后加上签名得来，所以一般要比 sessionId 长很多，很有可能超出 cookie 的大小限制（cookie 一般有大小限制的，如 4kb），如果你在 token 中存储的信息越长，那么 token 本身也会越长，这样的话由于你每次请求都会带上 token，对请求来是个不小的负担

2）token 不太安全。token 存储在 localStorage 中，这类的本地存储是可以被 JavaScript 直接读取的，**从存储角度来看**也不安全。另外，token 一旦生成无法让其失效，必须等到它过期才行，如果服务端检测到了一个安全威胁，也无法使相关的 token 失效，**所以 token 更适合一次性的命令认证，设置一个比较短的有效期**

…

---

#### JWT 作用

1、授权，最常用，一旦用户登录，后续每个请求都要携带 JWT 令牌才得以允许访问其他路径、服务和资源。单点登录是当今广泛使用 JWT 的一项功能，因为它开销很小，并且可以在不同的域名中轻松使用

2、信息交换，JWT 是在各方之间安全地传输信息的好方法，因为可以对 JWT 进行签名（使用公钥/密钥），所以可以确保信息交换的准确性，并且还可以验证传输的内容是否遭到篡改

…

---

#### JWT 结构

> `Header.Payload.Signature`

1、`Header` 标头

```json
{
    "typ": "JWT",
    "alg": "SHA256"
}
```

标头通常由两部分组成：

1）令牌的类型（即 JWT ）

2）使用的签名算法，如 SHA256，RSA

最后再将 `Header` 使用 `Base64` 编码，组成 JWT 的第一部分

2、`Payload` 负载

```json
{
    "sub": 123456789,
    "name": "somebody",
    "admin": true
}
```

`payload` 包含声明，即有关实体（通常是用户），和其他数据声明 ，存放用户信息中的**不敏感信息**，如用户Id，用户名，角色信息

最后再将 `payload` 使用 `Base64` 编码，组成 JWT 的第二部分

3、`Signature` 签名\

```
HMACSHA256(base64UrlEncode(header).base64UrlEncode(payload),secret);
```

`signature` 需要使用编码后的 `header` 和 `payload` 以及**自定义提供的密钥**组成一串字符串，然后 使用 `header` 中指定的签名算法进行签名。签名的作用是保证 JWT 没有被篡改过。

最后一步签名的过程，实际上是对头部以及负载内容进行签名，防止内容被篡改。如果有人对头部以及负载内容解码之后进行修改，再进行编码，最后加上之前的签名组合形成新的 JWT 的话，此时服务器就会判断出签名不一致。如果要对新的头部和负载进行签名在不知道密钥的情况下得出来的签名也是不一致的

…

---

####  JWT 注意细节

1）token 一旦由 server 生成，它就是有效的，直到过期，无法让 token 失效。除非 为 token 设立一个黑名单，在校验 token 前先过一遍此黑名单。如果在黑名单里则此  token 失效，但是会增加维护黑名单的成本，一般的做法是当客户端登出要让 token 失效时，直接在本地移除 token ，下次登录重新生成 token 

2） token 一般是放在 header 的 Authorization 自定义头里，不是放在 Cookie 里的，这主要是为了解决跨域不能共享 Cookie 的问题

3）可自定义注解标注在需要 token 认证的方法上，token 拦截器只需要拦截有该注解的方法即可

…

---

#### SpringBoot 整合

```xml
<dependency>
    <groupId>com.auth0</groupId>
    <artifactId>java-jwt</artifactId>
    <version>3.13.0</version>
</dependency>
```

```java
public class JWTUtils { // JWTUtils jwt工具类

    private final static String TOKEN_SIGNATURE = "QWs!!@#%&%TR123~*";
    private final static Integer TOKEN_EXPIRES = 5;

    /**
     * 生成token
     */
    public static String generateToken(Map<String, String> payload) {

        Calendar calendar = Calendar.getInstance();
        calendar.add(Calendar.MINUTE, TOKEN_EXPIRES);

        // 创建jwt builder
        JWTCreator.Builder builder = JWT.create();
        // 设置payload
        payload.forEach((k, v) -> {
            builder.withClaim(k, v);
        });

        // 设置过期时间
        builder.withExpiresAt(calendar.getTime());

        // 设置加密算法和密钥并签名
        String token = builder.sign(Algorithm.HMAC256(TOKEN_SIGNATURE));

        return token;
    }
    
    /** 
     * 验证token合法性
     */
    public static void verifyToken(String token) {
        // 若抛出异常则说明token验证失败
        JWT.require(Algorithm.HMAC256(TOKEN_SIGNATURE)).build().verify(token);
    }

    /**
     * 解析token
     */
    public static DecodedJWT parseToken(String token) {
        DecodedJWT decodedJWT = JWT.require(Algorithm.HMAC256(TOKEN_SIGNATURE)).build().verify(token);

        return decodedJWT;
    }

    /**
     * 刷新token
     */
    public static void refreshExpires(String token) {
        // 获取原来token里的内容，再重新生成token
    }

}
```

…

---

<br>

### 整合 Swagger3

…

#### 实现

…

```xml
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-boot-starter</artifactId>
    <version>3.0.0</version>
</dependency>
```

```properties
springfox.documentation.swagger-ui.enabled=true
```

```java
@EnableOpenApi
@Configuration
public class SwaggerConfig {

    // dev组
    @Bean
    public Docket createRestApiGroupDev() {

        ApiInfo apiInfo = new ApiInfoBuilder()
                // api标题
                .title("SpringBoot整合Swagger3")
                // api描述
                .description("测试一下swagger api的描述")
                .contact(new Contact("gnl", "http://gnlee.gitee.io", "1193898858@qq.com"))
                .version("1.0.0")
                .build();

        //文档类型 SWAGGER_2
        return new Docket(DocumentationType.OAS_30)
                .apiInfo(apiInfo)
                .select()
                // 指定扫描包含@ApiOperation 注解的方法
                .apis(RequestHandlerSelectors.withMethodAnnotation(ApiOperation.class))
                // 路径过滤器
                .paths(PathSelectors.any())
                .build()
                .groupName("swagger3 开发组");
    }

    // test组
    @Bean
    public Docket createRestApiGroupTest() {

        ApiInfo apiInfo = new ApiInfoBuilder()
                // api标题
                .title("SpringBoot整合Swagger3")
                // api描述
                .description("测试一下swagger api的描述")
                .contact(new Contact("gnl", "http://gnlee.gitee.io", "1193898858@qq.com"))
                .version("1.0.0")
                .build();

        //文档类型 SWAGGER_2
        return new Docket(DocumentationType.OAS_30)
                .apiInfo(apiInfo)
                .select()
                // 指定扫描包含@ApiOperation 注解的方法
                .apis(RequestHandlerSelectors.withMethodAnnotation(ApiOperation.class))
                // 路径过滤器
                .paths(PathSelectors.any())
                .build()
                .groupName("swagger3 测试组");
    }

}
```

```java
//User.java

@ApiModel("用户Bean")
@Data
@ToString
@RequiredArgsConstructor
public class User implements Serializable {

    @ApiModelProperty("用户姓名")
    private String name;

    @ApiModelProperty("用户年龄")
    private Integer age;

}
```

```java
//UserController.java

@Api(tags = {"用户管理Api"})
@RestController
public class UserController {

    @ApiOperation("用户的hello请求")
    @GetMapping("/hello")
    public String hello(){
        return "hello, swagger";
    }

    @ApiOperation("用户需要参数的请求")
    @GetMapping("/rp")
    public User requireParams(@ApiParam("用户名") @RequestParam("name") String name,
                              @ApiParam("用户年龄") @RequestParam("age") Integer age){
        User user = new User(name, age);
        return user;
    }


    @PostMapping("/user/{name}/{password}")
    @ApiOperation(value = "保存用户api", notes = "保存用户详情")
    @ApiImplicitParams({
            @ApiImplicitParam(name = "name", value = "用户名", dataType = "String", defaultValue = "gnl", paramType = "path"),
            @ApiImplicitParam(name = "password", value = "用户密码", dataType = "String", defaultValue = "123456", paramType = "path")
    })
    @ApiResponses({
            @ApiResponse(code = 401, message = "401 msg"),
            @ApiResponse(code = 403, message = "403 msg"),
            @ApiResponse(code = 404, message = "404 msg")
    })
    public Map<String, Object> saveUser(@PathVariable("name") String name,@PathVariable("password") String password) {
        Map<String, Object> map = new HashMap<>();
        map.put("code", 200);
        map.put("name", name);
        map.put("password", password);
        return map;
    }

}
```

…

#### 注解说明

1、`@Api` 用在请求的类上，表示对类的说明

- tags，说明该类的作用，可以在UI界面上看到的注解
- value，该参数没什么意义，在UI界面上也看到，所以不需要配置

…

2、`@ApiOperation` 用在请求的方法上，说明方法的用途、作用

- value，说明方法的用途、作用
- notes，方法的备注说明

…

3、`@ApiImplicitParams` 用在请求的方法上，表示一组参数说明

`@ApiImplicitParam` 用在 `@ApiImplicitParams` 注解中，指定一个请求参数的各个方面

- name，参数名
- value，参数的汉字说明、解释
- required，参数是否必须传
- paramType，参数放在哪个地方，header/query/path/body/form

- dataType，参数类型，默认 String，其它有`dataType="Integer"`等
- defaultValue，参数的默认值

…

4、`@ApiResponses` 用在请求的方法上，表示一组响应

`@ApiResponse` 用在 `@ApiResponses` 中，一般用于表达一个错误的响应信息

- code，状态码，例如200
- message，信息，例如"请求参数有误"
- response，抛出异常的类

…

5、`@ApiModel` 用于响应类上，表示一个返回响应数据的信息（这种一般用在 POST 创建的时候，使用 `@RequestBody` 这样的场景，请求参数无法使用 `@ApiImplicitParam` 注解进行描述的时候）   

`@ApiModelProperty` 用在属性上，描述响应类的属性

…

> ⚠ 正式版本发布的时候要关闭 Swagger

…

---

<br>

## 创建 Docker 镜像

> [SpringBoot 官方文档](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#container-images)

**依赖**

1、首先在依赖中引入 `spring-boot-maven-plugin` 用于将 SpringBoot 项目打包成一个 jar 包。

```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
</plugin>
```

> *注意*
>
> 如果项目直接依赖 `spring-boot-starter-parent`，可以直接使用该插件。如果使用的是 `spring-boot-dependencies` 或者其他依赖，需要稍作修改：
>
> ```xml
> <plugin>
>     <groupId>org.springframework.boot</groupId>
>     <artifactId>spring-boot-maven-plugin</artifactId>
>     <executions>
>         <execution>
>             <goals>
>               	<!-- 需要指定执行 repackage，在 MANIFEST.MF 生成对应的入口 -->
>                 <goal>repackage</goal>
>             </goals>
>         </execution>
>     </executions>
> </plugin>
> ```

2、其次需要在 spring-boot-maven-plugin 中添加配置

```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <layers>
            <!-- 生成 layers.idx 文件，用于制作 docker 镜像 -->
            <enabled>true</enabled>
        </layers>
    </configuration>
</plugin>
```

该配置主要用于生成 docker 镜像需要的 `layers.idx` 文件，文件内容大概如下：

```
- "dependencies":
  - BOOT-INF/lib/library1.jar
  - BOOT-INF/lib/library2.jar
- "spring-boot-loader":
  - org/springframework/boot/loader/JarLauncher.class
  - org/springframework/boot/loader/jar/JarEntry.class
- "snapshot-dependencies":
  - BOOT-INF/lib/library3-SNAPSHOT.jar
- "application":
  - META-INF/MANIFEST.MF
  - BOOT-INF/classes/a/b/C.class
```

3、最后执行打包操作 `mvn clean package`，`layers.idx` 文件会自动生存并加入 jar 包中。

…

---

**Dockfile**

创建 Dockfile 文件，名字就叫做 Dockfile。并将内容补充：

```dockerfile
FROM eclipse-temurin:17-jre as builder
WORKDIR application
# jar 相对 Dockerfile 所在的位置
ARG JAR_FILE=target/*.jar
COPY ${JAR_FILE} application.jar
RUN java -Djarmode=layertools -jar application.jar extract

FROM eclipse-temurin:17-jre
WORKDIR application
COPY --from=builder application/dependencies/ ./
COPY --from=builder application/spring-boot-loader/ ./
COPY --from=builder application/snapshot-dependencies/ ./
COPY --from=builder application/application/ ./
ENTRYPOINT ["java", "org.springframework.boot.loader.JarLauncher"]
```

最后在 Dockfile 所在的目录打包成 docker 镜像：

```shell
docker build -t imageName:version .

docke images # 查看生成的镜像
```

…

---

<br>

## 参考

**数据操作**

[spring data jpa（概述、快速入门、内部原理剖析、查询使用方式）](https://juejin.cn/post/6999782550910533645)

[使用Spring Data JPA进行分页和排序](https://zhuanlan.zhihu.com/p/57545677)

[SpringBoot - 集成Spring Data JPA](https://juejin.cn/post/6844904030641061895)

[数据库分库分表策略的具体实现方案](https://blog.csdn.net/qq_34964197/article/details/87283408)

[java分库分表技术](https://blog.csdn.net/qq_39291929/article/details/80795221)

**JWT**

[SpringBoot集成JWT实现token验证](https://www.jianshu.com/p/e88d3f8151db)

[Spring Boot认证：整合Jwt ](https://www.cnblogs.com/youngdeng/p/12868473.html)

[你管这破玩意儿叫 Token?](https://mp.weixin.qq.com/s/StIo-eC-7UeBr-fb6lIZkQ)

Log4J

[SpringBoot 使用 log4j2](https://blog.csdn.net/u010663021/article/details/108388817)

[Springboot整合log4j2日志全解](https://www.cnblogs.com/keeya/p/10101547.html)
