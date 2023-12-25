---
description: Spring 框架原理探析
tag: 
  - Java
  - Spring
  - 后端
---



# Spring 原理探析

## IOC

### IOC 本质

IOC 容器实际上是一个 Map，SpringApplication 执行 ApplicationContext 刷新方法的时候对所有的非懒加载的单例 Bean 进行了初始化。

```
org.springframework.context.ConfigurableApplicationContext#refresh // 加载并刷新 Java 代码配置或者 XMl 配置

// 先初始化一些特殊上下文情况下的 Bean，比如 Servlet 或者 React 等 Web 上下文的特殊 Bean
// Initialize other special beans in specific context subclasses.
org.springframework.context.support.AbstractApplicationContext#onRefresh

// 接着初始化普通的非懒加载单例 Bean
// Instantiate all remaining (non-lazy-init) singletons.
org.springframework.context.support.AbstractApplicationContext#finishBeanFactoryInitialization
```

初始化非懒加载单例 Bean 的时候会冻结配置

```java
// org.springframework.context.support.AbstractApplicationContext#finishBeanFactoryInitialization
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
  // ...
  // Allow for caching all bean definition metadata, not expecting further changes.
  beanFactory.freezeConfiguration(); // 冻结配置（Java or XML），不允许再进行修改
  // Instantiate all remaining (non-lazy-init) singletons.
  beanFactory.preInstantiateSingletons();
}
```

初始化之前

```java
// DefaultListableBeanFactory 有一个 beanDefinitionNames 属性保存所有需要初始化的 Bean 定义
private volatile List<String> beanDefinitionNames = new ArrayList<>(256);

// org.springframework.beans.factory.support.DefaultListableBeanFactory#preInstantiateSingletons
public void preInstantiateSingletons() throws BeansException {
  // ...
  List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);
  // Trigger initialization of all non-lazy singleton beans... // 遍历并初始化所有非懒加载的单例 bean
  for (String beanName : beanNames) { // 遍历素有 Bean 定义
    RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
    if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
      if (isFactoryBean(beanName)) {
        Object bean = getBean(FACTORY_BEAN_PREFIX + beanName); // 对工厂Bean 的名称做特殊处理， 一般是加一个 & 前缀
        if (bean instanceof SmartFactoryBean<?> smartFactoryBean && smartFactoryBean.isEagerInit()) {
          getBean(beanName); // 进行 Bean 初始化
        }
      }
      else {
        getBean(beanName); // 进行 Bean 初始化
      }
    }
  }
  // Trigger post-initialization callback for all applicable beans...
}
```

…

```
org.springframework.beans.factory.support.AbstractBeanFactory#getBean(java.lang.String)
org.springframework.beans.factory.support.AbstractBeanFactory#doGetBean
```

下一个重点是 doGetBean

```java
protected <T> T doGetBean(
  String name, @Nullable Class<T> requiredType, @Nullable Object[] args, boolean typeCheckOnly)
  throws BeansException {

  String beanName = transformedBeanName(name);
  Object beanInstance;

  // Eagerly check singleton cache for manually registered singletons.
  Object sharedInstance = getSingleton(beanName);
  if (sharedInstance != null && args == null) {
    if (logger.isTraceEnabled()) {
      if (isSingletonCurrentlyInCreation(beanName)) {
        logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +
                     "' that is not fully initialized yet - a consequence of a circular reference");
      }
      else {
        logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
      }
    }
    beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, null);
  }

  else {
    // Fail if we're already creating this bean instance:
    // We're assumably within a circular reference.
    if (isPrototypeCurrentlyInCreation(beanName)) {
      throw new BeanCurrentlyInCreationException(beanName);
    }

    // Check if bean definition exists in this factory.
    BeanFactory parentBeanFactory = getParentBeanFactory();
    if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
      // Not found -> check parent.
      String nameToLookup = originalBeanName(name);
      if (parentBeanFactory instanceof AbstractBeanFactory abf) {
        return abf.doGetBean(nameToLookup, requiredType, args, typeCheckOnly);
      }
      else if (args != null) {
        // Delegation to parent with explicit args.
        return (T) parentBeanFactory.getBean(nameToLookup, args);
      }
      else if (requiredType != null) {
        // No args -> delegate to standard getBean method.
        return parentBeanFactory.getBean(nameToLookup, requiredType);
      }
      else {
        return (T) parentBeanFactory.getBean(nameToLookup);
      }
    }

    if (!typeCheckOnly) {
      markBeanAsCreated(beanName);
    }

    StartupStep beanCreation = this.applicationStartup.start("spring.beans.instantiate")
      .tag("beanName", name);
    try {
      if (requiredType != null) {
        beanCreation.tag("beanType", requiredType::toString);
      }
      RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
      checkMergedBeanDefinition(mbd, beanName, args);

      // Guarantee initialization of beans that the current bean depends on.
      String[] dependsOn = mbd.getDependsOn();
      if (dependsOn != null) {
        for (String dep : dependsOn) {
          if (isDependent(beanName, dep)) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                            "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
          }
          registerDependentBean(dep, beanName);
          try {
            getBean(dep);
          }
          catch (NoSuchBeanDefinitionException ex) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                            "'" + beanName + "' depends on missing bean '" + dep + "'", ex);
          }
        }
      }

      // Create bean instance.
      if (mbd.isSingleton()) {
        sharedInstance = getSingleton(beanName, () -> {
          try {
            return createBean(beanName, mbd, args);
          }
          catch (BeansException ex) {
            // Explicitly remove instance from singleton cache: It might have been put there
            // eagerly by the creation process, to allow for circular reference resolution.
            // Also remove any beans that received a temporary reference to the bean.
            destroySingleton(beanName);
            throw ex;
          }
        });
        beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
      }

      else if (mbd.isPrototype()) {
        // It's a prototype -> create a new instance.
        Object prototypeInstance = null;
        try {
          beforePrototypeCreation(beanName);
          prototypeInstance = createBean(beanName, mbd, args);
        }
        finally {
          afterPrototypeCreation(beanName);
        }
        beanInstance = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
      }

      else {
        String scopeName = mbd.getScope();
        if (!StringUtils.hasLength(scopeName)) {
          throw new IllegalStateException("No scope name defined for bean '" + beanName + "'");
        }
        Scope scope = this.scopes.get(scopeName);
        if (scope == null) {
          throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
        }
        try {
          Object scopedInstance = scope.get(beanName, () -> {
            beforePrototypeCreation(beanName);
            try {
              return createBean(beanName, mbd, args);
            }
            finally {
              afterPrototypeCreation(beanName);
            }
          });
          beanInstance = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
        }
        catch (IllegalStateException ex) {
          throw new ScopeNotActiveException(beanName, scopeName, ex);
        }
      }
    }
    catch (BeansException ex) {
      beanCreation.tag("exception", ex.getClass().toString());
      beanCreation.tag("message", String.valueOf(ex.getMessage()));
      cleanupAfterBeanCreationFailure(beanName);
      throw ex;
    }
    finally {
      beanCreation.end();
      if (!isCacheBeanMetadata()) {
        clearMergedBeanDefinition(beanName);
      }
    }
  }

  return adaptBeanInstance(name, beanInstance, requiredType);
}
```

…

```
org.springframework.beans.factory.support.AbstractBeanFactory#createBean
org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#doCreateBean
org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#createBeanInstance
// getSingleton 的逻辑是：如果 Bean 在 singletonObjects 中已存在则返回，否则将 Bean 放入 singletonObjects 再返回
org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#getSingleton(java.lang.String, org.springframework.beans.factory.ObjectFactory<?>)
```

…

---

### Bean 状态

> 这个记录来此某次面试官问：Spring 中的 Bean 是有状态的还是无状态的？
>
> 答曰：不清楚。
>
> 面试官答：无状态的。
>
> …
>
> 于是就研究了一番。

Spring 的 IOC 容器中存在多种不同生命周期的 bean：singleton、prototype、request、session。

有无状态之分：

* 有状态是指在 bean 的生命周期内维护了某些状态变量，并且其他地方获取该 bean 时，仍能够访问到之前保存的状态变量数据；
* 无状态是指在 bean 的生命周期中不存在可来回传递/操作的，可共享的变量，每次操作都是独立的，每次获取都是一个新的 Bean。

…

由此可见：Spring 中的 Bean 既存在有状态的，也存在无状态的。

…

单例 bean 能被所有能获取到 IOC 容器的地方访问到，bean 本身是共享的。如果单例 bean 无任何成员变量，不保存任何状态，则说明是无状态的，线程安全的，比如 dao 层的类；如果单例 bean 是有状态的，在并发环境下就需要注意线程安全问题。

…

---

<br>

## AOP

**代理实现**

* JDK 动态代理
* CGLib 代理

…

---

<br>

## 事务管理

### 事务接口及抽象类

```java
// Spring 事务的顶层父类，用来管理 Spring 事务
public interface TransactionManager {}

// Spring 事务框架中最基础/重要的接口
public interface PlatformTransactionManager extends TransactionManager {
    /**
     * 此方法根据参数 TransactionDefinition 返回一个 TransactionStatus 对象
     * TransactionStatus 对象就可以看成是一个事务
     * 返回的 TransactionStatus 可能是一个新事务或者已存在的事务（如果当前调用栈中存在事务）
     * 参数 TransactionDefinition 描述传播行为、隔离级别、超时等
     * 此方法会根据参数对事务传播行为的定义，返回一个当前处于活跃状态的事务（如果存在），或创建一个新的事务
     * 参数对事务隔离级别或者超时时间的设置，会忽略已存在的事务，只作用于新建的事务
     * 并非所有事务定义设置都会受到每个事务管理器的支持，在遇到不受支持的设置时事务管理器会抛出异常
     */
    TransactionStatus getTransaction(TransactionDefinition definition) throws TransactionException;
    void commit(TransactionStatus status) throws TransactionException;
    void rollback(TransactionStatus status) throws TransactionException;
}

// 事务定义，用来定制事务的传播行为、隔离级别和超时等操作
public interface TransactionDefinition {
  
    // Spring 事务的隔离级别与 JDBC 定义的隔离级别对应
    int ISOLATION_DEFAULT = -1;
  
  	/**
  	 * 脏读：读取到另一个事务已修改但未提交的数据
  	 * 不可重复度：当事务 A 首先读取数据，事务 B 也读取同一个数据，并将数据修改，而后事务 A 再次读取就会得到和第一次读取不一样的结果。主要在修改场景发生
  	 * 幻读：一个事务读取所有满足 WHERE 条件的行，第二个事务插入一条满足 WHERE 条件的记录，第一个事务使用相同条件重新读取，在第二次读取中读取出额外的 "幻影 "记录。主要在插入/删除场景下发生
  	 */
  
  	// 读未提交
  	// 可读取到另一个事务修改但未提交的数据
  	// 存在脏读/不可重复度/幻读
  	int ISOLATION_READ_UNCOMMITTED = 1;  // same as java.sql.Connection.TRANSACTION_READ_UNCOMMITTED;
  
  	// 读已提交
  	// 解决脏读，存在不可重复度/幻读
  	int ISOLATION_READ_COMMITTED = 2;  // same as java.sql.Connection.TRANSACTION_READ_COMMITTED;
  
  	// 可重复度
  	// 解决脏读/不可重复度，存在幻读
		int ISOLATION_REPEATABLE_READ = 4;  // same as java.sql.Connection.TRANSACTION_REPEATABLE_READ;
  
  	// 可序列化/串行化，事务串行化执行，一次只允许一个事务操作
  	// 解决脏读/不可重复度/幻读
		int ISOLATION_SERIALIZABLE = 8;  // same as java.sql.Connection.TRANSACTION_SERIALIZABLE;


  	// 以下为 Spring 事务管理支持的传播行为，一共 7 种
  	/**
  	 * 如果当前存在事务，则加入；如果事务不存在，则新建
  	 * 是 Spring 事务的默认传播行为，通常定义事务同步范围
  	 */
    int PROPAGATION_REQUIRED = 0;
  	
  	// 如果当前存在事务，则加入；如果事务不存在，则以无事务的方式运行
  	int PROPAGATION_SUPPORTS = 1;
  
  	// 如果当前存在事务，则加入；如果不存在则抛出异常
  	int PROPAGATION_MANDATORY = 2;
  
  	// 如果存在事务，则暂停当前事务，创建新事务
		int PROPAGATION_REQUIRES_NEW = 3;
  	
  	// 总是以无事务的方式运行
		int PROPAGATION_NOT_SUPPORTED = 4;
  	
  	// 如果当前存在事务则抛出异常
  	int PROPAGATION_NEVER = 5;
  	
  	// 如果当前存在事务，则在嵌套事务中执行，否则表现为 PROPAGATION_REQUIRED
		int PROPAGATION_NESTED = 6;
  
  	// 是否将事务优化为只读事务，只读标志适用于任何事务隔离级别
    default boolean isReadOnly() {
      return false;
    }
}

/**
 * PlatformTransactionManager 的抽象实现类，它预先实现了定义的传播行为，并负责处理事务的同步。
 * 如果需要自定义事务管理框架，继承 AbstractPlatformTransactionManager 即可。子类只需要关心事务的开始，暂停，恢复和提交。
 */
public abstract class AbstractPlatformTransactionManager implements PlatformTransactionManager, Serializable {}
```



<br>

**事务传播行为**

| 传播行为                  | 说明                                                         |
| ------------------------- | ------------------------------------------------------------ |
| PROPAGATION_REQUIRED      | 如果当前存在事务，则加入；如果事务不存在，则新建             |
| PROPAGATION_SUPPORTS      | 如果当前存在事务，则加入；如果事务不存在，则以无事务的方式运行 |
| PROPAGATION_MANDATORY     | 如果当前存在事务，则加入；如果不存在则抛出异常               |
| PROPAGATION_REQUIRES_NEW  | 如果存在事务，则暂停当前事务，创建新事务                     |
| PROPAGATION_NOT_SUPPORTED | 总是以无事务的方式运行                                       |
| PROPAGATION_NEVER         | 如果当前存在事务则抛出异常                                   |
| PROPAGATION_NESTED        | 如果当前存在事务，则在事务中创建事务，并在嵌套事务中执行，否则表现为 PROPAGATION_REQUIRED |



### 声明式事务

#### 开启声明式事务

Spring 的声明式事务支持需手动开启，注解驱动使用 `@EnableTransactionManagement` 标注在 Spring 配置类上；XML 开发则在配置文件加上 `<tx:annotation-driven/>` 。

使用 @EnableTransactionManagement 相对来说更加灵活，因为它**不仅可以根据名称还能根据类型**将 TransactionManager 加载到 IOC 容器中。 

…

@EnableTransactionManagement 和 `<tx:annotation-driven/>` 只会扫描和它们自己相同的应用上下文内的 `@Transactional` 注解，也就是说，如果在  DispatcherServlet 的 WebApplicationContext 中标注 @EnableTransactionManagement，它只会扫描和 Controller 同级别下的 @Transactional。

…

#### @Transactional

`@Transactional` 注解可以标注在类或者 `public` 方法上，如果标注在 `protected/private` 方法或者包内可见的方法上*不会报错，但是在这些地方 Spring 事务不会生效*。

如果在 Spring 配置类上标注 `@EnableTransactionManagement`，可以通过注入自定义的 `TransactionAttributeSource` 来让事务可以在类中的非 `public` 方法中生效。

```java
/**
 * enable support for protected and package-private @Transactional methods in
 * class-based proxies.
 */
@Bean
TransactionAttributeSource transactionAttributeSource() {
    return new AnnotationTransactionAttributeSource(false);
}
```

…

**类和方法的优先级**

@Transactional 注解可以同时标注在类和方法上，但是**标注在方法上的优先级会比标注在类上的优先级高**。

```java
// (readOnly = true) 表示该事务从事的操作是读操作
@Transactional(readOnly = true)
public class DefaultFooService implements FooService {

    public Foo getFoo(String fooName) {}

    // 标注在方法上的优先级大于类上
    @Transactional(readOnly = false, propagation = Propagation.REQUIRES_NEW)
    public void updateFoo(Foo foo) {}
}
```

…

**属性设置**

* *propagation* default `Propagation.REQUIRED`
* *isolation* default `Isolation.DEFAULT`
* *timeout* default `TransactionDefinition.TIMEOUT_DEFAULT = -1`
* *readOnly* 是否是只读事务 default `false`
* *rollbackFor*  Any `RuntimeException` or `Error` triggers rollback, and any checked `Exception` does not.
* *noRollbackFor*

…

### 编程式事务

> 因为开发中用的大多都是声明式事务，编程式事务做了解即可

…

#### TransactionTemplate

类似 JdbcTemplate，由 Spring 提供的操作事务的模版方法类。

```java
public class SimpleService implements Service {
    // single TransactionTemplate shared amongst all methods in this instance
    private final TransactionTemplate transactionTemplate;

    // use constructor-injection to supply the PlatformTransactionManager
    public SimpleService(PlatformTransactionManager transactionManager) {
        this.transactionTemplate = new TransactionTemplate(transactionManager);
    }

  	// with result
    public Object someServiceMethod() {
        return transactionTemplate.execute(new TransactionCallback() {
            // the code in this method runs in a transactional context
            public Object doInTransaction(TransactionStatus status) {
                updateOperation1();
                return resultOfUpdateOperation2();
            }
        });
    }

  	// without result
    public Object methodWithoutResult() {
        transactionTemplate.execute(new TransactionCallbackWithoutResult() {
            protected void doInTransactionWithoutResult(TransactionStatus status) {
                updateOperation1();
                updateOperation2();
            }
        });
      }
  
    	// with rollback
    public Object methodWithoutRollback() {
        transactionTemplate.execute(new TransactionCallbackWithoutResult() {
            protected void doInTransactionWithoutResult(TransactionStatus status) {
                try {
                    updateOperation1();
                    updateOperation2();
                } catch (SomeBusinessException ex) {
                		status.setRollbackOnly();
                }
            }
        });
      }
}
```

…

#### TransactionManager

```java
DefaultTransactionDefinition def = new DefaultTransactionDefinition();
// 定义事务属性，如事务名，传播行为，隔离级别等
def.setName("SomeTxName");
def.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);

TransactionManager txManager = new JdbcTransactionManager();
TransactionStatus status = txManager.getTransaction(def);
try {
    // put your business logic here
} catch (MyException ex) {
    txManager.rollback(status);
    throw ex;
}
txManager.commit(status);
```

…

**声明式事务和编程式事务如何选择**

* 如果只是在代码中进行小规模的事务操作，选择编程式，比如 `TransactionTemplate`；
* 如果存在大量事务操作，优先选择声明式事务，操作简单，并且还把事务逻辑和业务逻辑分离开，利于维护；
* 如果使用的是 Spring 框架，推荐使用声明式事务。

…

---

### 回滚规则

Spring 事务只会在遇到运行时异常和未受检查异常时会滚，也就是说只有在遇到 `RuntimeException` 及其之类或者 `Error` 及其之类的时候才会回滚。事务遇到受检查异常时，不会回滚，而是将其捕获并抛出。

但仍然可以通过指定回滚规则，精确配置哪些异常类型会将事务标记为回滚，包括已检查的异常。

…

---

### 事务失效

1、注解 `@Transactional` 修饰的类非 Spring 容器对象；

2、用 `@Transactional` 修饰方法，且该方法被类内部方法调用；

3、注解 `@Transactional` 修饰的方法非 public 修饰；

4、代码中出现的异常被 catch 代码块捕获，而不是被 Spring 事务框架捕获;

5、Spring 事务 `rollback` 策略默认是 `RuntimeException` 及其子类和 `Error` 及其之类，其他情况如果未提前定义则事务失效；

6、数据库不支持事务。



<br>

## 参考

[spring#transaction](https://docs.spring.io/spring-framework/docs/current/reference/html/data-access.html#transaction)

