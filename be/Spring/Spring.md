# Spring



## IOC

> 控制反转（Inversion of Control），即与用户自行编写代码创建对象相反，为了将耦合度降低，IOC将对象的创建和对象之间的调用过程交给Spring来进行管理

<br/>

**<u>1 - IOC 底层原理</u>**

> xml 解析、工厂模式、反射

1、解析 xml 配置文件，创建配置的对象

```java
<bean id="user" class="com.demo.bean.User"></bean>
```

2、创建工厂类，通过反射的 `newInstance()` 创建对象

<br/>

**<u>2 - IOC 接口</u>**

> BeanFactory & ApplicationContext

Spring 提供 IOC 容器两种实现方式（两个接口）：**BeanFactory** & **ApplicationContext**

**1、BeanFactory**，是 IOC 容器的基本实现，**是 Spring 内部使用的接口**。BeanFactory 在加载配置文件的时候不会创建对象（此处特指懒加载的单例对象），直到获取对象的时候才会创建对象

```java
// BeanFactory 完成这一步只会加载配置文件，不会创建对象
BeanFactory bf = new ClassPathXmlApplicationContext("/spring-bean-context.xml");
// 当获取容器中的对象时才会创建对象
User user = bf.getBean("user", User.class);
System.out.println(user.toString());
```



**2、ApplicationContext**，BeanFactory 的子接口，提供更多更强大的功能，一般开发人员使用该接口

```java
// 在加载配置文件的时候就会把配置文件中配置的对象创建
ApplicationContext ac = new ClassPathXmlApplicationContext("/spring-bean-context.xml");
// 获取容器中的对象
User user = ac.getBean("user", User.class);
System.out.println(user.toString());
```

<br/>

**<u>3 - IOC 管理 Bean（XML）</u>**

> Bean 的管理指的是**创建对象和依赖注入（DI）**

1、创建对象，Spring 创建对象的时候，默认执行无参的构造方法完成对象创建

```xml
<!-- 将对象的创建交给Spring来完成 -->
<!-- 配置User对象并赋值 -->
<bean id="user" class="com.demo.bean.User"></bean>
id：唯一标识，指创建对象的别名，可通过别名获取该对象
class：类全路径，通过全路径找到对应的类
```

<br/>

**2、依赖注入/属性值注入（DI）**

① `set()`方法注入

```xml
<bean id="user" class="com.demo.bean.User">
    
    <!-- 对User对象的属性设置值 -->
    <property name="name" value="张三" />
    <property name="age" value="20" />
    
    <!-- 对User对象的某个属性设置null值 -->
    <property name="score" >
    	<null/>
    </property>
    
    <!-- 对User对象的某个属性设置特殊值<<男>> -->
    <!-- 把带特殊符号的内容进行转义 -->
    <property name="address" value="&lt;&gt;<<男>>">
    </property>
    
    <!-- 对User对象的某个属性设置特殊值<<南宁>> -->
    <property name="address">
        <!-- 把带特殊符号的内容写到CDATA -->
    	<value><![CDATA[<<南宁>>]]></value>
    </property>
</bean>
```



② 有参构造方法注入

```xml
<bean id="user1" class="com.demo.bean.User">
    <!-- name：指定需要注入值的属性名 -->
    <constructor-arg name="name" value="王五"></constructor-arg>
    <constructor-arg name="age" value="15"></constructor-arg>
</bean>

<bean id="user2" class="com.demo.bean.User">
    <!-- index：index指构造方法中要注入值的属性的位置标识
             若不使用index则需要严格按照构造方法中属性的位置赋值 -->
    <constructor-arg index="0" value="李四"></constructor-arg>
    <constructor-arg index="1" value="21"></constructor-arg>
</bean>
```



③ **p名称空间**注入，使用p名称空间注入可简化基于 xml 配置方式

```xml
<!-- 1、在配置文件中<beans>标签内添加p名称空间支持 -->
xmlns:p="http://www.springframework.org/schema/p"
<!-- 在<bean>标签里面进行操作，进行属性注入 -->
<bean id="user3" class="com.demo.bean.User" p:name="User666" p:age="16" />
```



④ 属性注入——外部 Bean 注入

```xml
<bean id="userDao" class="com.demo.dao.impl.UserDaoImpl" />
<bean id="userServiceImpl" class="com.demo.service.impl.UserServiceImpl" >
    <property name="userDao" ref="userDao"></property>
</bean>
```

```java
UserService userService = ac.getBean("userServiceImpl", UserService.class);
userService.add();
```



⑤ 属性注入——内部 Bean 和级联赋值

⑥ 属性注入——外部 Bean 和级联赋值

⑦ 集合属性注入

<br/>

**3、FactoryBean**

> Spring 有两种类型的 Bean，一种普通 Bean，一种工厂 Bean（FactoryBean）

普通 Bean：在配置文件中配置的 Bean 的类型就是返回类型

工厂 Bean：在配置文件中定义 Bean 类型可以和返回类型不一样（多于 Spring 框架内部使用），用于创建/生产 Bean



**FactoryBean 的实现**

```java
// 1、创建类，让这个类作为工厂 Bean，实现工厂接口 FactoryBean
// 实现接口里的方法，在实现的方法中定义返回的 Bean 类型
public class MyFactoryBean implements FactoryBean<User> {
    /**
     * 定义返回 Bean 类型并且进行属性赋值操作
     */
    @Override
    public User getObject() throws Exception {
        return new User("userFromFB", 12);
    }
}
```

```xml
<!-- 配置自定义工厂 Bean -->
<bean id="myFactoryBean" class="com.demo.util.MyFactoryBean" ></bean>
```

```java
@Test
public void test4() {
User user = ac.getBean("myFactoryBean", User.class);
   System.out.println(user);
}
```

<br/>

**4、Bean 的作用域**

> `<bean>`标签中的`scope`属性。默认情况下，Spring 创建的 Bean 是单例的

1）Singleton：单例

2）Prototype：多例

3）Request：一次请求

4）Session：一次对话

<br/>

**5、Bean 的生命周期**

> 生命周期，即 Bean 从创建到销毁的过程

1）通过构造器创建 Bean 实例（无参构造函数）

2）调用`set()`方法

3）`postProcessBeforeInitialization()`，Bean 后置处理器中的Before方法

4）调用 Bean 的初始化方法（需要进行配置初始化方法）

```java
public void initMethod(){
    System.out.println("4 user initMethod...");
}
public void destroyMethod(){
    System.out.println("7 user destroyMethod...");
}
```

```xml
<!-- 在xml中对User配置初始化和销毁方法 -->
<bean id="user" class="com.demo.bean.User" init-method="initMethod" destroy-method="destroyMethod">
    <property name="name" value="张三" />
    <property name="age" value="20" />
</bean>
```

5）`postProcessAfterInitialization()`，Bean 后置处理器中的After方法

6）Bean 可获取

7）（若是单例 Bean）当容器关闭，调用 Bean 的销毁方法（需要进行配置销毁方法，并手动调用销毁方法）

<br/>

**6、Bean 自动装配**

> 自动装配即 Spring 根据指定装配规则（属性名或属性类型），Spring 自动将匹配的属性值进行注入的过程

通过配置`<bean>`标签的`autowire`属性来配置自动装配：

1）byName：根据属性名称注入，注入值的id和类的属性名称一致

2）byType：根据属性类型注入。xml 文件中若存在多个同类型的对象则 byType 会报错（可以同时指定名称和类型来避免）

<br/>

**7、引入外部属性文件**

```properties
jdbc.username=root
jdbc.password=root
jdbc.driverClass=com.mysql.jdbc.Driver
jdbc.url=jdbc:mysql:///userdb?useUnicode=true&characterEncoding=UTF-8&useSSL=false
```

```xml
<context:property-placeholder location="classpath:jdbc.properties" ignore-unresolvable="true" />

<!-- 通过引入外部属性文件配置 Druid 数据库连接池 -->
<bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource">
    <property name="username" value="${jdbc.username}" />
    <property name="password" value="${jdbc.password}"/>
    <property name="driverClassName" value="${jdbc.driverClass}"/>
    <property name="url" value="${jdbc.url}"/>
</bean>
```



<br/>

**<u>3 - IOC 管理 Bean（注解）</u>**

> 注解是代码特殊标记，可以使用在类上、方法上、属性上

1、配置注解扫描

```xml
<!-- use-default-filters="false" 不使用默认的扫描规则 -->
<context:component-scan base-package="com.demo" use-default-filters="false">
    <!-- context:include-filter 扫描哪些内容 -->
    <!-- type="annotation" 扫描注解类型 -->
    <!-- expression="org.springframework.stereotype.Controller" 扫描@Controller注解 -->
    <context:include-filter type="annotation" expression="org.springframework.stereotype.Controller"/>
    <!-- exclude-filter 扫描排除哪些内容 -->
    <!-- expression="org.springframework.stereotype.Service" 不扫描@Service注解 -->
    <context:exclude-filter type="annotation" expression="org.springframework.stereotype.Service"/>
</context:component-scan>
```



2、注解方式实现属性注入

1）`@Autowired`：<mark>根据属性类型进行注入</mark>（不需要添加 Setter 方法）

2）`@Qualifier`：<mark>根据属性名称进行注入</mark>，和 @Autowired 一起进行使用，指定同一类型的特定名称

3）`@Resource`：<mark>可根据类型也可根据名称注入</mark>，默认 byType

4）`@Value`：注入普通类型属性



3、完全注解开发

1）创建配置类，并创建对象

```java
@ComponentScan(basePackages = {"com.demo"})
@Configuration // 表示该类作为配置类
public class SpringConfig {
    @Bean
    public User user(){
        return new User("zhangsan", 20);
    }
}
```

2）从容器中获取对象

```java
public class TestConfig {
    private AnnotationConfigApplicationContext context = null;
    @Before
    @Test
    public void init(){
        context = new AnnotationConfigApplicationContext(SpringConfig.class);
    }

    @Test
    public void test1(){
        for (String name : context.getBeanDefinitionNames()) {
            System.out.println(name);
        }
        User user = (User) context.getBean("user");
        System.out.println(user);
    }
}
```





## 整合 Mybatis

**步骤**

1、新建项目加入依赖

- Spring 依赖
- Mybatis 依赖
- MySQL 的驱动
- Spring 事务依赖
- Spring 和 Mybatis 集成依赖（由 Mybatis 官方提供，用来在 Spring 项目中创建 Mybatis 的 SqlSessionFactory 和 Dao 对象）

<br/>

2、创建实体类

3、创建 Dao 接口和 Mapper 文件

4、创建 Mybatis 主配置文件

5、创建 Service 接口和实现类

<br/>

6、创建 Spring 配置文件，声明 Mybatis 对象并交给 Spring 创建，交由 IOC 容器管理

1）创建数据源

2）SqlSessionFactory

3）Dao 对象

4）声明自定义的 Service

<br/>

7、创建测试类



## AOP

> AOP（Aspect Oriented Programming）面向切面编程，能够在不修改源代码的情况下添加新功能（通过代理方式），常用作异常处理和日志记录。两种实现：Spring AOP 和 AspectJ

<br/>

**<u>1 - 动态代理</u>**

1、JDK 动态代理，要求目标对象（被代理对象）必须实现接口

2、CGLib 动态代理，目标对象不需要实现接口，原理是**生成目标类的子类**。子类是增强过的，这个子类对象就是代理对象。所以使用 CGLib 生成动态代理，要求目标类必须能够被继承（不是 final 类）



**动态代理的作用**

1）在目标类源代码不改变的情况下，增加功能

2）减少代码的重复

3）专注业务逻辑代码

4）降低耦合，让业务功能和日志，事务非业务功能分离



<br/>

**<u>2 - 关键字</u>**

1、**Aspect**，切面，给目标类增加的功能就是切面，如日志，事务，统计信息，参数检查等都是切面

1）**切面方法特点**：一般都是非业务方法，独立使用

2）切面的**三个关键要素**：

① 切面的功能代码，切面干什么

② 切面的执行位置（使用 PointCut 表示切面执行的位置）

③ 切面的执行时间（使用 Advice 表示执行时间，在目标方法之前，还是在目标方法之后）

<br/>

3）如何面向切面编程

① 需要在分析项目功能时找出切面

② 合理安排切面的执行时间（在目标方法前，还是目标方法后）

③ 合理的安排切面执行的位置，在哪个类，哪个方法增加还是增强功能

<br/>

2、**JoinPoint**，连接点，连接业务方法和切面的位置，即某类中的业务方法

3、**PointCut**，切入点，指多个连接点方法的集合，多个方法

4、**目标对象**，给哪个类的方法添加功能，这个类就是目标对象

5、**Advice**，通知，通知表示切面功能执行的时间



<br/>

**<u>3 - AOP 的实现</u>**

1、引入依赖

```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-aspects</artifactId>
    <version>5.3.2</version>
    <scope>compile</scope>
</dependency>
```

2、开启注解支持`@EnableAspectJAutoProxy`

```java
/**
 * proxyTargetClass = true
 * 强制使用cglib动态代理
 */
@EnableAspectJAutoProxy(proxyTargetClass = true)
@ComponentScan(basePackages = {"com.demo"})
@Configuration
public class SpringConfig {
}
```

3、编写切面类，**被切面类也要加入 IOC 容器中**

```java
/**
 * @Aspect 是AspectJ框架中的注解，表示当前类是切面类
 * @Component 需要将当前类放入容器中
 */
@Aspect
@Component
public class MyAspect {

    /**
     * @Pointcut 管理和定义切入点。如果项目中存在多个切入点表达式是重复的，可以复用的，则可以使用@Pointcut
     * value 切入点表达式
     * 特点：
     * 1. 当使用@Pointcut定义在一个方法上，此时这个方法的名称就是切入点表达式的别名
     *    其他通知中，value属性就可以使用这个方法名称代替切入点表达式
     * 2. 无需参数，无需编写代码的空方法
     *
     */
    @Pointcut(value = "execution(* com.demo.service.*..*(..))")
    public int myPointcut(){
        return 0;
    }

   /*
    * 切入点表达式写法
    *   固定格式：execution(访问权限符 返回值类型 方法全类名(参数表))
    *   通配符
    *       * ：
    *           1、匹配一个或多个字符
    *                   ("execution(public int com.demo.spring.caclculator.My*.*(int,int))")
    *           2、匹配任意一个参数
    *                   ("execution(public int com.demo.spring.caclculator.MyCalculatorImpl.*(int,*))")
    *           3、只能匹配一层路径
    *                   ("execution(public int com.demo.spring.*.MyCalculatorImpl.*(int,*))")
    *                   只能扫描spring下一层中的包
    *           4、权限修饰符不能用 * 来代替，可以省略不写则代表任意权限修饰符
    *       .. ：
    *           1、匹配任意多个参数
    *                   ("execution(public int com.demo.spring.caclculator.MyCalculatorImpl.*(..))")
    *
    *
    * */
  
    /**
     * @Before 前置通知
     * 定义方法，方法是实现切面功能的
     * 方法定义的要求：
     * 1. public 修饰
     * 2. 方法没有返回值
     * 3. 方法名称自定义
     * 4. 方法可以有参数，也可以无参数
     *      如果有参数，参数不是自定义的，而是几个定义好的参数类型可以使用
     *
     * JoinPoint：业务方法，要加入切面功能的业务方法
     * 作用：可以在通知方法中获取方法执行时的信息。例如方法名称，方法的实参
     *          如果切面功能中需要用到方法的信息，则加入JoinPoint
     *          JoinPoint参数的值是由框架赋予，因此必须是第一个位置的参数
     */
    @Before(value = "myPointcut()")
    public void myBefore(JoinPoint jp){
        System.out.println("切面前置通知 myBefore。。。");
        System.out.println("方法签名："+jp.getSignature());
        System.out.println("方法名："+jp.getSignature().getName());
        System.out.println("方法实参："+Arrays.toString(jp.getArgs()));

    }

    /** 
     * @AfterReturning 后置通知
     *      属性：1. value 切入点表达式
     *            2. returning 自定义的变量，表示方法的返回值
     *                  自定义的变量名(returning = "res")必须和通知方法的形参(Integer res)同时出现，并且名称一致
     *      特点：
     *          1. 在目标方法之后执行
     *          2. 能够获取到目标方法的返回值，可以根据这个返回值做不同的处理功能
     *          3. 可以修改这个返回值
     */
    @AfterReturning(value = "execution(* *..service.*.*(..))",returning = "res")
    public void myAfterReturning(JoinPoint jp, Integer res){
        System.out.println("切面后置通知 AfterReturning。。。");
        System.out.println("返回结果："+res);
        /**
         * 在后置通知修改值后主程序中获取到的值仍是旧值的原因：
         * 先执行目标方法，此时值已传给目标方法，再执行后置通知，则修改无法影响主程序中的值
         * 因为只改变了字面量，未修改取值的地址
         */
        if (123 == res){
            res = 456;
            System.out.println("值已修改");
        } else {
            System.out.println("未作改变");
        }
    }

    /**
     * @Around 环绕通知
     * 定义格式：
     *  1. public
     *  2. 必须有一个返回值，类型推荐使用Object
     *  3. 方法名称自定义
     *  4. 方法有参数，固定的参数 ProceedingJoinPoint
     *
     * 特点：
     *  1. 它是功能最强的通知
     *  2. 在目标方法前后都能增强功能
     *  3. 控制目标方法时候被调用执行
     *  4. 修改原来的目标方法执行结果，影响最后的结果
     *
     *  环绕通知，等同于JDk动态代理的 InvocationHandler 接口
     *  参数 ProceedingJoinPoint 等同于其中 invoke()方法的参数Method
     *      作用：执行目标方法
     *  返回值：即目标方法的执行结果，可被修改
     */
    @Around(value = "execution(* *..service.*.doAAA(..))")
    public Object myAround(ProceedingJoinPoint jp){
        Object res = null;
        System.out.println("切面环绕通知 myAround。。。");
        Object[] args = jp.getArgs();
        System.out.println("获取到的参数：" + args.toString());

        try {
            System.out.println("切面环绕通知 方法执行之前。。。做点什么");

            res = jp.proceed(args);

            res = "haaaaaa";

            System.out.println("切面环绕通知 方法执行之后。。。做点什么");

        } catch (Throwable throwable) {
            throwable.printStackTrace();
        } finally {
            return res;
        }
    }

    /**
     * @AfterThrowing 异常通知
     *
     * 执行：
     * try {
     *
     * } catch(Exception e){
     *      myAfterThrowing()
     * }
     */
    @AfterThrowing(value = "execution(* *..*.doException(..))", throwing = "e")
    public void myAfterThrowing(Exception e){
        System.out.println("切面异常通知 myAfterThrowing。。。" + e);
    }

    /**
     * @After 最终通知
     * 特点：
     *      1. 一般用于做清除或关闭工作
     *      2. 在目标方法之后执行，且一定会执行
     * 执行
     * try {
     *
     * } catch() {
     *
     * } finally{
     *     myAfter()
     * }
     */
    @After(value = "execution(* *..*.doException(..))")
    public void myAfter(){
        System.out.println("切面通知 myAfter。。。 ");
    }
}

/**
 *   普通通知方法的执行顺序
 *       前置--后置--正常返回
 *       前置--后置--异常返回
 *   环绕通知执行顺序
 *       前置--正常返回--后置
 *       前置--异常返回--后置
 *
 * 若同时存在环绕通知和普通通知
 *   环绕前置--普通前置--环绕正常/异常返回--环绕后置--普通后置--普通正常返回
 *
 * 若有多个切面通知同时作用，默认按照通知所在类的类名首字母排序执行，先执行后结束，后执行先结束。
 * 若想改变切面通知的执行顺序可以使用@Order(数值，数值越大优先级越高)

 *
 * Spring 对通知方法的要求不严格，可以是任意修饰符，任意返回值的。
 *  唯一要求是通知方法的参数不能乱写
  	因为通知方法是 Spring 利用反射调用的，每次方法调用得确定这个方法的参数表的值。
 */
```



<br/>

**<u>4 - 切面织入时期</u>**

1、**编译器**织入（静态织入），即编译出来的 `class` 文件字节码就已经被织入

2、**类装载期**织入

3、**动态代理**织入，在**运行期间为目标类增强生成子类**的方式（Spring AOP 采用）

动态织入又分为动静两种：

1）静是织入过程只在第一次调用 target 时执行

2）动指每次调用 target 都执行



<br/>

**<u>5 - Spring AOP 和 AspectJ 的区别</u>**

1、能力不同

1）Spring AOP 提供一个简单 AOP 实现。**Spring AOP 只能应用于由 Spring 容器管理的 Bean**

2）AspectJ 目的是提供完整的 AOP 解决方案。它相对于 Spring AOP 来说更强大，也更复杂。**AspectJ 可以应用在所有的 POJO 对象**。

<br/>

2、织入时期不同

1）Spring AOP 采用的是动态织入（运行时织入）

2）AspectJ 采用的是静态织入（加载时织入）即编译出来的 `class` 文件字节码就已经被织入

<br/>

3、应用场景

* **框架**

  如果没有使用 Spring 框架，不能使用 Spring AOP。因为它无法管理任何超出 Spring 容器范围的 Bean。如果使用 Spring 框架，可以使用 Spring AOP

* **灵活性**

  Spring AOP 不是一个完整的 AOP 解决方案。如果希望得到更强大的 JoinPoint 支持，可以选择 AspectJ

* **性能**

  在应用程序有成千上万个切面时，动态织入的速度会比较慢，所以最好选择 AspectJ 静态织入。速度比较快

* **兼容性**

  这两个框架都是完全兼容的。在使用 Spring 框架时，仍然可以使用 AspectJ



**总结**

| SpringAOP                                                    | AspectJ                                                      |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| 在纯 Java 中实现                                             | 使用 Java 编程语言的扩展实现                                 |
| 不需要单独的编译过程                                         | 除非设置 LTW，否则需要 AspectJ 编译器 (ajc)                  |
| 只能使用运行时织入                                           | 运行时织入不可用。支持编译时、编译后和加载时织入             |
| 功能不强-仅支持方法级编织                                    | 更强大 - 可以编织字段、方法、构造函数、静态初始值设定项、最终类/方法等 |
| 只能在由 Spring 容器管理的 Bean 上实现                       | 可以在所有域对象上实现                                       |
| 仅支持方法执行切入点                                         | 支持所有切入点                                               |
| 代理是由目标对象创建的，并且切面应用在这些代理上             | 在执行应用程序之前 (在运行时) 前，各方面直接在代码中进行织入 |
| 编译时织入比运行时织入快得多。Spring AOP 是基于代理的框架, 因此在应用程序启动时会创建代理。另外每个方面还有一些方法调用，这会对性能产生负面影响 | AspectJ 在应用程序执行之前将这些方面编织到主代码中，因此没有额外的运行时开销，与 Spring AOP 不同。基于这些原因，基准表明 AspectJ 的速度几乎比 Spring AOP 快8到35倍 |
| 易于学习和应用                                               | 相对于 Spring AOP 来说更复杂                                 |





<br/>

## Spring 事务

> Spring 事务的本质其实就是数据库对事务的支持，没有数据库的事务支持，Spring 是无法提供事务功能的



<br/>

**<u>1 - 声明式事务</u>**

把事务相关的资源和内容都提供给 Spring 进行管理，Spring 就能处理事务提交、回滚，几乎不用增加额外的代码



<br/>

**<u>2 - 事务概括</u>**

**1、什么是事务**

> 事务是指一组 SQL 语句的集合，集合中的 SQL 语句可能是 insert、update、select、delete。我们希望这些多个 SQL 语句都能成功执行或者都失败，这些 SQL 语句的执行是一致的，作为一个整体执行



**四个特性（ACID）**

1、**原子性**

事务要么全部提交成功，要么全部失败回滚，不能只执行其中一部分操作

2、**一致性**

1）操作前后总量不变，事务的执行不能破坏数据库的完整性和一致性，一个事务在执行之前和执行之后，数据库必须处于一致性状态

2）如果数据库系统在运行过程中发生故障，有些事务尚未完成就被迫中断，这些未完成的事务对数据库所做的修改已有一部分写入物理数据库，这时数据库就处于一种不正确的状态，也就是不一致的状态

<br/>

3、**隔离性**

事务的隔离性是指在并发环境中，并发的事务是相互隔离的，一个事务的执行不能被其他事务干扰。不同的事务并发操作相同的数据时，每个事务都有各自完成的数据空间，即一个事务内部的操作及使用的数据对其他事务是隔离的，并发执行的各个事务之间不能相互干扰

4、**持久性**，事务提交之后，表发生的变化是永久的。一旦事务提交，那么它对数据库中对应数据的状态的变更就会永久保存到数据库中，即使发生系统崩溃或者服务器宕机等故障，只要数据库能够重新启动，那么一定能将其恢复到事务成功的状态



<br/>

**隔离级别**

1、**读未提交（READ UNCOMMITTED）**，隔离级别最低，存在脏读、不可重复读、幻读

1）**脏读**，即当一个事务正在访问数据，对数据进行了修改，而这种修改还没有提交到数据库中，此时另外一个事务也访问这个数据，并且读取到了数据

2）**不可重复读**，是指在一个事务内，多次读同一数据，得到的是不一样的结果。如事务A在读取某数据，同时事务B在修改该数据，事务A无法看到这个数据项在事务A操作过程中的中间值，只能看到修改的结果（不允许脏读）。在事务A还没有结束时，事务C也访问同一数据，并且进行修改，之后事务A再获取到的数据和第一次获取到的数据就是不一样的。**在第一个事务中的两次读数据之间，由于第二个事务的修改，第一个事务两次读到的的数据可能是不一样的，称为不可重复读**。

<mark>**不可重复读的重点是修改**</mark>。同样的条件，读取过的数据，再次读取出来发现值不一样。

3）**幻读**，指同样的事务操作，在前后两个时间段内执行对同一个数据的读取，可能出现不一样的结果。如，第一个事务对一个表中的数据进行了修改，这种修改涉及到表中的全部数据行。同时第二个事务也修改这个表中的数据，这种修改是向表中插入一行新数据。那么以后就会发生操作第一个事务的用户发现表中还有没有修改的数据行，就好象发生了幻觉一样。

<mark>**幻读的重点在于新增或者删除**</mark>。同样的条件，第 1 次和第 2 次读出来的记录数不一样

<br/>

2、**读已提交（READ COMMITTED）**，读已提交只允许获取已经提交的数据，解决脏读，存在不可重复读、幻读

3、**可重复读（REPEATABLE READ）**，保证在事务处理过程中，多次读取同一个数据时，其结果和事务开始时刻读取到的值是一致的。解决脏读和不可重复读，存在幻读

4、**串行化（SERIALIZABLE）**，最严格的事务隔离级别，它要求所有事务被串行化，即事务只能一个接着一个进行处理，不能并发执行。**脏读、不可重复读、幻读都不会出现**



| 隔离级别 | 脏读 | 不可重复读 | 幻读 |
| -------- | ---- | ---------- | ---- |
| 读未提交 | 有   | 有         | 有   |
| 读已提交 | 无   | 有         | 有   |
| 可重复读 | 无   | 无         | 有   |
| 串行化   | 无   | 无         | 无   |





<br/>

**2、什么时候想到使用事务**

当操作涉及到多个表或者是多个 SQL 语句时，需要保证这些语句都能成功才能完成该功能，或者都失败，保证操作是符合要求的。通常在项目中，控制事务的代码会放在 Service 类的业务方法上，因为业务方法会调用多个 Dao 方法，执行多条 SQL 语句

<br/>

3、通常使用 JDBC 或 Mybatis 访问数据库时如何处理事务

1）JDBC 访问数据库，处理事务

```java
Connection conn; conn.commit();
conn.rollback();
```

2）Mybatis 访问数据库，处理事务

```java
sqlSession.commit();
sqlSession.rollback();
```

<br/>

4、在`3`中事务的处理方式有什么不足？

1）不同的数据库访问技术，处理事务的对象，方法不同，需要了解不同数据库访问技术使用事务的原理

2）需要掌握多种数据库中事务的处理逻辑，什么时候提交事务

3）需要掌握处理事务的多种方法

4）多种数据库的访问技术有不同的事务处理的机制，对象，方法

<br/>5、怎么解决`3`中的不足

Spring 提供一种处理事务的统一模型，能够使用统一的步骤/方式完成多种不同数据库的事务管理

<br/>



<u>**3 - 使用 Spring 来管理事务需要怎么做/做什么**</u>

> Spring 处理事务的模型，使用的步骤都是固定的，把事务使用的信息提供给 Spring 即可

1、创建 Spring 事务管理器对象，完成 commit 和 rollback

事务管理器接口`PlatformTransactionManager`，定义了事务重要方法 `commit()` 和`rollback()`。Spring 把每一种数据库访问计数对应的事务管理器都创建好，Mybatis 访问数据库使用`DataSourceTransactionManager`。

告诉 Spring 使用的数据库类型，即声明数据库访问技术对应的事务管理器实现类，在 Spring 的配置文件中使用`<bean>`标签声明即可

<br/>

2）业务方法需要什么样的事务（说明需要的事务类型）

① 事务的隔离级别，Spring 事务默认采用数据库默认的事务隔离级别，MySQL 默认为 REPEATABLE_READ，Oracle 默认为 READ_COMMITTED

② 事务的超时时间，表示一个方法最长的执行时间，如果方法执行超过了时间，事务就会回滚。单位是秒，整数值，默认是`-1`

③ 事务的传播行为（7个传播行为），控制业务方法是不是有事务的，是什么样的事务。表示业务方法调用时，事务在方法之间是如何使用的

<br/>

3）**Spring 提交事务和回滚事务的时机**

① 当业务方法执行成功，没有异常抛出，Spring 在方法执行后提交事务，事务管理器执行 `commit()`

② 当业务方法抛出运行时异常，Spring 执行回滚，调用事务管理器`rollback()`

③ 当业务方法抛出非运行时异常，主要是受查异常（在代码中必须处理的异常，如 IOException、SQLException），提交事务

<br/>



<u>**4 - Spring 事务管理总结**</u>

1、管理事务的是事务管理器接口和它的实现类

2、Spring 的事务是一个统一模型

① 指定要使用的事务管理器实现类

② 指定方法需要的隔离级别，传播行为，超时等

<br/>

**<u>5 - Spring 实现事务管理</u>**

> 适合中小型项目

1、引入事务依赖

```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-tx</artifactId>
    <version>5.3.2</version>
</dependency>
```

2、配置文件中创建事务管理器，并开启事务注解

```xml
<!--创建事务管理器-->
<bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <!--注入数据源-->
    <property name="dataSource" ref="dataSource" />
</bean>

<!--开启事务注解，即可扫描到@Transactional注解-->
<tx:annotation-driven transaction-manager="transactionManager"></tx:annotation-driven>
```

3、在 Service 层代码中使用`@Transactional`注解

1）解标在类上，说明该类中的所有方法都启用事务管理

2）标在方法上，说明该方法启用事务管理

3）`@Transactional`属性：

* propagation，传播行为，默认为 REQUIRE

  REQUIRE：如果有事务在运行，当前的方法就在这个事务内运行，否则，就启动一个新的事务，并在自己的事务内运行

  REQUIRE_NEW：当前方法比会启动新事务，并在它自己的事务内运行，如果有事务在运行，则应该将正在运行的事务挂起，等待新事务执行完毕，再恢复挂起的事务

  SUPPORTS：如果有事务在运行，当前的方法就在这个事务内运行。否则不在事务内运行

* isolation，隔离级别（解决脏读，幻/虚读，不可重复读），默认为 REPEATABLE_READ 
* timeout，超时时间，默认值`-1`，单位秒
* readOnly，是否只读，默认为 false
* rollBackFor，出现哪些异常进行事务回滚。Spring 框架会首先检查方法抛出的异常是不是在 rollbackFor 的属性值中，如果异常在 rollbackFor 列表中，不管什么类型的异常，一定回滚；如果抛出的异常不在 rollbackFor 列表中，Spring 会判断该异常是否是 RunTimeException，如果是，一定回滚
* noRollBackFor，出现哪些异常不进行事务回滚



<br/>

**<u>6 - AspectJ 框架 AOP 方式实现事务管理</u>**

> 适合大型项目，有很多的类和业务方法，需要配置大量事务。使用 AspectJ 框架，在 Spring 配置文件中声明类和方法需要的事务。这种方式业务方法和事务配置完全分离

1、引入`spring-aspects`依赖

2、声明事务管理器对象

```xml
<bean id="XXX" class="DataSourceTransactionManager"></bean>
```

3、声明方法需要的事务类型

```xml
<!--1. 创建事务管理器-->
<bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <!--注入数据源-->
    <property name="dataSource" ref="dataSource" />
</bean>

<!--2. 开启事务注解，即可扫描到@Transactional注解-->
<tx:annotation-driven transaction-manager="transactionManager"></tx:annotation-driven>

<!--3. 声明业务方法的业务属性（隔离级别，传播行为，超时时间）
    id：自定义事务名称
    transaction-manager：事务管理器对象的id
    -->
<tx:advice id="myAdvice" transaction-manager="transactionManager">
    <!--tx:attributes：配置事务属性-->
    <tx:attributes>
        <!--tx:method：要配置事务的方法名，method可以有多个，分别给不同的方法设置事务属性
            name：方法名称
                    1. 完整的方法名称，不带包名和类名
                    2. 方法可以使用通配符*，表示任意字符
            propagation：传播行为
            isolation：隔离级别
            rollback-for：指定的异常类名，全限定类名。发生异常一定回滚
        -->
        <tx:method name="selectAllUser" propagation="REQUIRED" isolation="DEFAULT"
                   rollback-for="java.lang.RuntimeException, com.demo.util.MyException"/>

        <!--使用通配符指定更多方法-->
        <!--添加方法-->
        <tx:method name="add*" propagation="REQUIRES_NEW" />
        <!--修改方法-->
        <tx:method name="modify*" />
        <!--删除方法-->
        <tx:method name="remove*" />
        <!--查询方法-->
        <tx:method name="query*" propagation="SUPPORTS" read-only="true" />
    </tx:attributes>
</tx:advice>
```

4、配置 AOP，指定哪些类需要创建代理

```xml
<!--配置aop-->
<aop:config>
    <!--
        expression="execution(* *..service..*.*(..))"：任意目录下service包中任意类的任意方法
    -->
    <aop:pointcut id="servicePointcut" expression="execution(* *..service..*.*(..))"/>
    <!--
        配置增强器：关联advice和pointcut
        advice-ref：通知id
        pointcut-ref：切入点表达式id
    -->
    <aop:advisor advice-ref="myAdvice" pointcut-ref="servicePointcut" />
</aop:config>
```





<br/>

**<u>7 - 事务失效场景</u>**

1、注解`@Transactional`配置的方法非 public 权限修饰；

2、注解`@Transactional`所在类非 Spring 容器管理的 Bean；

3、注解`@Transactional`所在类中，注解修饰的方法被类内部方法调用；

4、业务代码抛出异常类型非 RuntimeException，事务失效；

5、业务代码中存在异常时，使用`try-catch`语句块捕获，而 catch 语句块没有抛出异常;（最难被排查到问题且容易忽略）；

6、注解`@Transactional`中 Propagation 属性值设置错误即 `Propagation.NOT_SUPPORTED`（一般不会设置此种传播机制）；

7、MySQL 关系型数据库，且存储引擎是 MyISAM 而非 InnoDB，则事务会不起作用（基本开发中不会遇到）





<br/>

## Spring 5 新特性

<br/>

**<u>1 - 支持`@Nullable`注解</u>**

1、标注在方法上，表示该方法返回可以为空

2、标注在属性值上，表示属性值可以为空

3、标注在参数前，表示参数可以为空



<br/>

**<u>2 - SpringWebFlux</u>**

> SpringWebFlux 是Spring 5 新添加的模块，用于 Web 开发，功能和 SpringMVC 类似，WebFlux 使用的是当前比较流行的响应式编程框架

WebFlux 是一种异步非阻塞的框架，异步非阻塞的框架在 Servlet3.1 之后才支持，核心是基于 Reactor 的相关 API 实现的

**异步非阻塞**

1）异步和同步，针对调用者，调用者发送请求，如果等着对方回应之后才去做其他事情就是同步；如果发送请求之后不用等着对方回应就去做其他事情就是异步

2）阻塞和非阻塞，针对被调用者，被调用者收到请求之后**，做完请求任务之后**才给出反馈就是阻塞；**收到请求之后马上给出反馈**，然后再去做其他任务就是非阻塞

<br/>

**特点**

1）非阻塞式，在有限资源下，提高系统的吞吐量和伸缩性，以Reactor为基础实现响应式编程

2）函数式编程

<br/>

**和 SpringMVC 比较**

1）两个框架都可以使用注解方式实现，都运行在 Tomcat 等容器中

2）SpringMVC 采用命令式编程，SpringWebFlux 采用异步响应式编程



> 实际生产中还是 SpringMVC 优势大，且 WebFlux 对关系型数据库不是很友好，暂且先做了解…

<br/>

**<u>3 - 支持 JUnit5</u>**

1、引入依赖

```xml
<dependency>
  <groupId>org.springframework</groupId>
  <artifactId>spring-test</artifactId>
  <version>5.3.2</version>
</dependency>

<dependency>
    <scope>compile</scope>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter-api</artifactId>
    <version>5.6.3</version>
</dependency>
```

2、创建测试类

```java
// 1、单独两个注解
@ExtendWith(SpringExtension.class)
@ContextConfiguration("/spring/spring-context.xml")
```

```java
// 2、使用复合注解
@SpringJUnitConfig(locations = {"/spring/spring-context.xml"})
```





<br/>

**<u>4 - 日志整合</u>**

> Spring 5 整合 Log4j 2

1、引入依赖

```xml
<!--日志依赖-->
<dependency>
    <groupId>org.apache.logging.log4j</groupId>
    <artifactId>log4j-core</artifactId>
    <version>2.14.0</version>
    <scope>compile</scope>
</dependency>
<dependency>
    <scope>compile</scope>
    <groupId>org.apache.logging.log4j</groupId>
    <artifactId>log4j-api</artifactId>
    <version>2.14.0</version>
</dependency>
<dependency>
    <scope>compile</scope>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-api</artifactId>
    <version>1.7.30</version>
</dependency>
<dependency>
    <groupId>org.apache.logging.log4j</groupId>
    <artifactId>log4j-slf4j-impl</artifactId>
    <version>2.14.0</version>
</dependency>
```

2、resouces 目录下添加配置文件`log4j2.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!--日志级别以及优先级排序：    OFF > FATAL > ERROR > WARN > INFO > DEBUG > TRACE > ALL -->
<Configuration status="WARN">
    <!--先定义所有的appender-->
    <appenders>
        <!--输出日志信息到控制台-->
        <console name="Console" target="SYSTEM_OUT">
            <!--控制日志输出的格式-->
            <PatternLayout pattern="%d{yyyy-MM-dd HH:mm:ss.SSS} [%t] %-5level %logger{36} - %msg%n"/>
        </console>
    </appenders>
    <!--定义logger，只有定义了logger并引入appender，appender才会生效-->
    <loggers>
        <!--root：用于指定项目的根日志，如果没有单独指定logger，则会使用root作为默认的日志输出-->
        <root level="info">
            <appenderRef ref="Console"/>
        </root>
    </loggers>
</Configuration> 
```











**参考**

[9种设计模式在Spring中的运用，一定要非常熟练！](https://mp.weixin.qq.com/s/ThK3QTGxIQla6AjKosZNVw)

