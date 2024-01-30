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

IOC 容器实际上是一个 Map

```java
// org.springframework.beans.factory.support.DefaultSingletonBeanRegistry
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);
```

…

### Bean 初始化与获取流程

> 以 SpringBoot 为例

SpringApplication 执行 ApplicationContext 刷新方法的时候对所有的非懒加载的单例 Bean 进行了初始化。

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
  // 首先检查是不是已经手动注册了单例 Bean；如果注册了直接从 IOC 中获取
  // Eagerly check singleton cache for manually registered singletons.
  Object sharedInstance = getSingleton(beanName);
  if (sharedInstance != null && args == null) {
    // if isSingletonCurrentlyInCreation(beanName) // 处理单例 Bean 创建循环引用
    // Returning eagerly cached instance of singleton bean +BeanName+ 
    // that is not fully initialized yet - a consequence of a circular reference
    // else Returning cached instance of singleton bean
    // 判断 isSingletonCurrentlyInCreation 即当前 Bean 是否处于创建状态
    // 如果处于创建状态且能够从 IOC 中获取到当前 Bean，说明存在循环引用
    // 否则直接从 IOC 缓存中获取 Bean 并返回
    // ...
    beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, null);
  }

  else {
    // 处理原型 Bean 创建循环引用
    // Fail if we're already creating this bean instance:
    // We're assumably within a circular reference.
    if (isPrototypeCurrentlyInCreation(beanName)) {
      throw new BeanCurrentlyInCreationException(beanName);
    }

    // 判断是否存在父类 BeanFactory
    // 一层层向父类 BeanFactory 查询是否存在 bean 定义，如果存在直接从父类 BeanFactory 获取 Bean
    // 有点类加载双亲委派机制的意思
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
      // 父类 BeanFactory 判断结束
    }

    // 走到这里说明 IOC 中不存在 Bean 缓存，开始一段新的逻辑
    if (!typeCheckOnly) {
      markBeanAsCreated(beanName); // 将 Bean 标记为已创建
    }

    StartupStep beanCreation = this.applicationStartup.start("spring.beans.instantiate")
      .tag("beanName", name); // 默认按照 BeanName 创建
    try {
      if (requiredType != null) {
        beanCreation.tag("beanType", requiredType::toString);
      }
      // 获取 bean 定义 BeanDefinition
      // MergedLocalBeanDefinition？
      // 是这样的：bean 定义可能手动注册过，会在容器中存在 bean 定义缓存，
      // 如果 bean 定义存在且该定义没有被标记为“过时的”=stale，
      // 则将当前要创建的 bean 定义和缓存中的 bean 定义合并，再执行 bean 创建流程
      RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
      // 检查要创建的 bean 是否是抽象类，抽象类无法创建，会抛出异常 BeanIsAbstractException
      checkMergedBeanDefinition(mbd, beanName, args);

      // 保证 bean 创建的时候其依赖均已经存在
      // 如果 bean 依赖存在，则循环创建所有的依赖 bean，
      // 即对所有的 depensOnBean 执行一遍 getBean 流程
      // Guarantee initialization of beans that the current bean depends on.
      String[] dependsOn = mbd.getDependsOn();
      if (dependsOn != null) {
        for (String dep : dependsOn) {
          // 存在依赖循环引用
          // if (isDependent(beanName, dep)) // Circular depends-on relationship between
          registerDependentBean(dep, beanName);
          try {
            getBean(dep);
          }
          // 依赖 bean 定于缺失 // depends on missing bean
        }
      }

      // 到这里说明 bean 依赖均已存在
      // 开始创建单例 bean
      // Create bean instance.
      if (mbd.isSingleton()) {
        // 1、真正执行 bean 创建方法 createBean，创建完成之后将 bean 放入单例 bean 容器 // createBean 后续展开
        // 2、然后再执行 getSingleton 从单例 bean 容器中获取 bean // getSingleton 后续展开
        sharedInstance = getSingleton(beanName, () -> {
          try {
            return createBean(beanName, mbd, args); // 真正执行 bean 创建方法
          }
          catch (BeansException ex) {
            // 如果创建 bean 的过程中出现异常，明确的将 bean 从容器中销毁
            // Explicitly remove instance from singleton cache: It might have been put there
            // eagerly by the creation process, to allow for circular reference resolution.
            // Also remove any beans that received a temporary reference to the bean.
            destroySingleton(beanName);
            throw ex;
          }
        });
        // 获取给定 bean 实例的对象，如果是 FactoryBean，可以是 bean 实例本身，也可以是其创建的对象。
        beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
        
        // 在这里可能会存在的疑惑
        // sharedInstance 是什么？从 IOC 中获取到的 beanInstance 不就是我们需要的 bean 吗？
        // 为什么已经获取到了 sharedInstance 还需要再执行 getObjectForBeanInstance 获取 beanInstance？
        // 别着急，本段代码块后面就有解释
      }

      // 原型 bean 创建逻辑
      // 逻辑和单例 bean 创建类似，多了 beforePrototypeCreation 和 afterPrototypeCreation
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

      // 创建其他声明周期的 bean，requets/session
      else {
        String scopeName = mbd.getScope();
        // if (!StringUtils.hasLength(scopeName)) // No scope name defined for bean
        Scope scope = this.scopes.get(scopeName);
        // if (scope == null) // No Scope registered for scope name
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
      beanCreation.end(); // bean 创建结束
      if (!isCacheBeanMetadata()) {
        clearMergedBeanDefinition(beanName);
      }
    }
  }
  return adaptBeanInstance(name, beanInstance, requiredType); // 将 bean 转换成 requiredType 对应的类型
}
```

…

> 首先，getSingleton 方法有什么用？
>
> *Return the (raw) singleton object registered under the given name.*
>
> 根据 beanName 返回一个 *Raw Singleton Object*。
>
> getObjectForBeanInstance 方法有什么用？
>
> *Get the object for the given bean instance, either the bean instance itself or its created object in case of a FactoryBean.*
>
> 从给定的 bean 实例获取到对应的对象，这个对象可能是它自己，也可能是可以创建 bean 实例的 FactoryBean。
>
> …
>
> 也就是说，在创建单例 bean 的时候：如果是普通 bean，经过 getObjectForBeanInstance 返回的还是普通 bean；如果是由 FactoryBean 创建的 bean，返回的则是对应的 FactoryBean。

…

下面看 createBean，主要做一些 bean 创建前的预处理工作：从类加载器获取对应的 bean Class 对象。

```java
// org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#createBean
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
  // Creating instance of bean + beanName
  RootBeanDefinition mbdToUse = mbd;
  // 从类加载器获取对应的 bean Class 对象
  // Make sure bean class is actually resolved at this point, and
  // clone the bean definition in case of a dynamically resolved Class
  // which cannot be stored in the shared merged bean definition.
  Class<?> resolvedClass = resolveBeanClass(mbd, beanName); 
  if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
    mbdToUse = new RootBeanDefinition(mbd);
    mbdToUse.setBeanClass(resolvedClass);
  }

  // Prepare method overrides.
  // ...

  // 对于一些特殊 bean 处理前后置处理器
  // Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
  // ...

  try {
    Object beanInstance = doCreateBean(beanName, mbdToUse, args);
    // Finished creating instance of bean
    return beanInstance;
  }
  // ...
}
```

…

接下来的重点是 doCreateBean，真正执行 bean 创建

```java
// org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#doCreateBean
protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
  throws BeanCreationException {

  // Instantiate the bean. // 使用 BeanWrapper 来包装 bean
  BeanWrapper instanceWrapper = null;
  if (mbd.isSingleton()) {
    instanceWrapper = this.factoryBeanInstanceCache.remove(beanName); // 先从缓存中查看 bean 是否存在
  }
  if (instanceWrapper == null) {
    // 根据 beanName，bean 定义，bean 构造方法参数 args 创建一个新的 bean 实例。会处理构造方法依赖注入。
    // 注意：此时 bean 还未进行属性赋值，仅创建了空对象，未进行初始化
    instanceWrapper = createBeanInstance(beanName, mbd, args); // 不存在则创建 bean 实例
  }
  Object bean = instanceWrapper.getWrappedInstance();
  Class<?> beanType = instanceWrapper.getWrappedClass(); // 为 bean 实例设置对应的类型
  if (beanType != NullBean.class) {
    mbd.resolvedTargetType = beanType;
  }

  // Allow post-processors to modify the merged bean definition.
  // 后置处理器锁
  // ...

  // 非懒加载单例 bean 设置：允许循环引用，并加入 SingletonFactory
  // Eagerly cache singletons to be able to resolve circular references
  // even when triggered by lifecycle interfaces like BeanFactoryAware.
  boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
                                    isSingletonCurrentlyInCreation(beanName));
  if (earlySingletonExposure) {
    // 为解决潜在的循环引用，急切将 bean 缓存
    // ...
    addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
  }

  // Initialize the bean instance. // 初始化 bean 实例
  Object exposedObject = bean;
  try {
    populateBean(beanName, mbd, instanceWrapper); // 填充 bean 属性
    // 初始化 bean 实例。上面 createBeanInstance 是创建实例，现在是初始化
    // 此时会执行 initMethod 初始化方法，以及后置处理器中的方法
    exposedObject = initializeBean(beanName, exposedObject, mbd);
  }
  // catch BeanCreationException 

  // 下面的逻辑是为了解决依赖注入，解决需要 earlySingletonExposure 的 bean
  // 可以跳过...直接看下一部分
  if (earlySingletonExposure) {
    Object earlySingletonReference = getSingleton(beanName, false);
    if (earlySingletonReference != null) {
      if (exposedObject == bean) {
        exposedObject = earlySingletonReference;
      }
      else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
        String[] dependentBeans = getDependentBeans(beanName);
        Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
        for (String dependentBean : dependentBeans) {
          if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
            actualDependentBeans.add(dependentBean);
          }
        }
        // if (!actualDependentBeans.isEmpty()) // 依赖异常 // throw BeanCurrentlyInCreationException
      }
    }
  }
  // Register bean as disposable. // 将 bean 注册为一次性的。只有单例 bean 才需要注册。
  try { registerDisposableBeanIfNecessary(beanName, bean, mbd); }
  // catch // BeanDefinitionValidationException 进行 bean 校验
  return exposedObject;
}
```

…

从 bean 的创建到初始化的流程都走完了，接下来 回到 getSingleton，在执行 doGetBean 的时候会调用到 getSingleton

```java
sharedInstance = getSingleton(beanName, () -> {
  // ...
  return createBean(beanName, mbd, args);
  // ...
});
```

接下来的重点 getSingleton 需要传入一个 beanName 和一个函数式接口 ObjectFactory。ObjectFactory 只有一个方法 getObject 即获取创建好的单例 bean。

```java
// org.springframework.beans.factory.support.DefaultSingletonBeanRegistry
// 用来存储单例 bean 的容器 singletonObjects，使用默认长度 256 的 ConcurrentHashMap 防止并发创建出现线程安全问题
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
  // Bean name must not be null
  synchronized (this.singletonObjects) { // 保证单例 bean 唯一
    Object singletonObject = this.singletonObjects.get(beanName); // 先从缓存中获取
    
    // 如果缓存中为 null 且当前 bean 处于被销毁的阶段，throw new BeanCreationNotAllowedException
    if (singletonObject == null) {
      
      // if (this.singletonsCurrentlyInDestruction) // throw BeanCreationNotAllowedException
      // Creating shared instance of singleton bean
      
      beforeSingletonCreation(beanName); // 检查单例 bean 是否处于创建状态
      boolean newSingleton = false; // 新的单例 bean 标志
      // 记录 bean 创建过程中被压制的异常 // ...
      
      try {
        singletonObject = singletonFactory.getObject(); // 获取单例 bean
        newSingleton = true;
      }
      catch (IllegalStateException ex) {
        // Has the singleton object implicitly appeared in the meantime ->
        // if yes, proceed with it since the exception indicates that state.
        singletonObject = this.singletonObjects.get(beanName);
        if (singletonObject == null) {
          throw ex;
        }
      }
      // catch BeanCreationException // 异常处理
      finally {
        if (recordSuppressedExceptions) {
          this.suppressedExceptions = null;
        }
        afterSingletonCreation(beanName); // 将 bean 从 singletonsCurrentlyInCreation 集合中移除
      }
      if (newSingleton) {
        addSingleton(beanName, singletonObject); // 先将 bean 加入容器缓存
      }
    }
    return singletonObject; // 再返回
  }
}
```

…

> **总结一下**
>
> getSingleton 的逻辑是：如果 Bean 在 singletonObjects 中已存在，则直接返回；否则创建 Bean 并放入 singletonObjects 缓存再返回。

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

### 依赖循环

* [三级缓存解决依赖循环](https://developer.aliyun.com/article/766880)
* [Spring 如何解决循环依赖](https://segmentfault.com/a/1190000039091691)
* [解决 Spring 循环依赖](https://www.skjava.com/article/1377399227)

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

