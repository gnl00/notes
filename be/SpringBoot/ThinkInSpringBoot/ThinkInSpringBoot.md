# Think In SpringBoot

## 源码分析

### SpringBoot 启动流程

先不加入 Web 内容，一个 SpringBoot 应用是如何启动的？

```java
@SpringBootApplication
public class SourceMain {
    public static void main(String[] args) {
        SpringApplication.run(SourceMain.class, args);
    }
}
```

让我们从主启动类开始探索，启动的调用流程如下：

1、SpringApplication#run(Class[], String[]) 静态方法，帮助运行 SpringApplication

2、SpringApplication#run(String...) 这是 SpringBoot 应用启动的关键方法

3、SpringApplication#refreshContext

4、SpringApplication#refresh

5、AbstractApplicationContext#refresh 刷新容器，Spring 应用也是调用这个方法刷新容器。



<br>

从 SpringApplication#run 方法开始分析：

```java
// SpringApplication#run
// Run the Spring application, creating and refreshing a new ApplicationContext
public ConfigurableApplicationContext run(String... args) {
  // 创建并初始化 BootstrapContext 启动上下文
  // 提供相应懒加载单例 bean，这些 bean 可能需要较高的代价去创建，或者在容器启动时会被需要
  DefaultBootstrapContext bootstrapContext = createBootstrapContext();
  ConfigurableApplicationContext context = null;
  
  // 为系统属性设置 java.awt.headless 值，其作用是判断当前是否运行在 headless 模式下
  
  // 获取运行监听器，监听运行上下文和主启动类
  // listeners 监听容器启动过程中的每一步
  SpringApplicationRunListeners listeners = getRunListeners(args);
  // set StartupStep = starting
  listeners.starting(bootstrapContext, this.mainApplicationClass);
  try {
    // 获取 main 方法的运行参数
    ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
    // 准备运行环境，创建并配置运行环境
    // StartupStep = environment-prepared
    ConfigurableEnvironment environment = prepareEnvironment(listeners, bootstrapContext, applicationArguments);
    // 配置需要忽略的 bean
    configureIgnoreBeanInfo(environment);
    // 获取打印的 banner，默认搜索 resources 下的 banner.txt
    Banner printedBanner = printBanner(environment);
    // 创建 ApplicationContext，会判断是否需要创建 WebApplicationContext
    // StartupStep = create -> create end
    context = createApplicationContext();
    // 设置 context 上下文的 StartupStep
    context.setApplicationStartup(this.applicationStartup);
    // 准备 context 上下文环境
    // StartupStep = context-prepared -> context-loaded
    prepareContext(bootstrapContext, context, environment, listeners, applicationArguments, printedBanner);
    // 刷新上下文环境，内部调用 AbstractApplicationContext#refresh
    // StartupStep = refresh -> post-process -> post-process end -> refresh end
    refreshContext(context); // 这是 Spring IoC 容器刷新的关键方法
    // ...
    // 下面的内容暂时放一放，后面会分析
    // ...
    afterRefresh(context, applicationArguments);
    // ...
    listeners.started(context, timeTakenToStartup);
    // ...
    callRunners(context, applicationArguments);
    // ...
  }
  // ...
  listeners.ready(context, timeTakenToReady);
  // ...
  return context;
}
```

从上面的方法可以看到，**相比与 Spring 应用**，SpringBoot 应用在启动的时候只是多了一些准备工作：创建启动上下文，准备运行环境，创建容器上下文（**判断是否是 Web 应用<sup>如何判断后面分析</sup>**）等工作，实际启动使用的都是 AbstractApplicationContext#refresh 方法。



<br>

接下来我们看关键的 AbstractApplicationContext#refresh 方法：

```java
// AbstractApplicationContext#refresh
// 刷新 context 并启动容器，如果容器启动失败，销毁之前创建的单例 Bean。调用此方法后，要么成功创建出所有需要的单例 Bean 启动成功，要么销毁所有单例 Bean 并关闭容器
public void refresh() throws BeansException, IllegalStateException {
  synchronized (this.startupShutdownMonitor) {
    // Create a new step and marks its beginning
    // 开始一个新的步骤 StartupStep
    // 开始的 StartupStep 就是在 SpringApplication#run 中
    // context.setApplicationStartup(this.applicationStartup)
    // 设置的 StartupStep
    // StartupStep = refresh
    StartupStep contextRefresh = this.applicationStartup.start("spring.context.refresh");

    // 准备刷新当前上下文
    // 当前上下文指的是容器上下文，即
    // SpringApplication#run 中
    // context = createApplicationContext() 这个上下文
    // 不是 bootstrapContext 
    prepareRefresh();

    // Tell the subclass to refresh the internal bean factory.
    // 让子类刷新内部的 Bean 工厂
    ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

    // Prepare the bean factory for use in this context.
    // 为当前 context 上下文准备 Bean 工厂
    // beanFactory 在 prepareBeanFactory(beanFactory) 方法中注册了
    // environment、systemProperties、systemEnvironment、applicationStartup
    // 这 4 个 Bean
    prepareBeanFactory(beanFactory);

    try {
      // Allows post-processing of the bean factory in context subclasses.
      // 对当前 context 的 Bean 工厂进行后置处理，空实现
      postProcessBeanFactory(beanFactory);

      // 上面 Startup.start("spring.context.refresh")
      // 此处 Startup.start("spring.context.beans.post-process")
      // 表示应用启动流程再进一步，从 refresh 到 post-process
      StartupStep beanPostProcess = this.applicationStartup.start("spring.context.beans.post-process");
      
      // Invoke factory processors registered as beans in the context.
      // 执行 Bean 工厂后置处理器 which registered as beans in the context
      // Instantiate and invoke all registered BeanFactoryPostProcessor beans
      invokeBeanFactoryPostProcessors(beanFactory);

      // Register bean processors that intercept bean creation.
      registerBeanPostProcessors(beanFactory);
      beanPostProcess.end();

      // Initialize message source for this context.
      initMessageSource();

      // Initialize event multicaster for this context.
      // 初始化当前 context 的 ApplicationEventMulticaster 应用事件多播器
      initApplicationEventMulticaster();

      // Initialize other special beans in specific context subclasses.
      // 在具体的 context 子类中初始化其他特殊的 Bean，空实现
      onRefresh();

      // Check for listener beans and register them.
      registerListeners();

      // Instantiate all remaining (non-lazy-init) singletons.
      // 完成 BeanFactory 初始化，主要负责初始化所有非懒加载的单例 Bean
      finishBeanFactoryInitialization(beanFactory);

      // Last step: publish corresponding event.
      // 做一些 cleanup 操作，并发布相关的事件
      finishRefresh();
    }

    catch (BeansException ex) {
      // Destroy already created singletons to avoid dangling resources.
      // 如果容器启动失败，则销毁所有以创建的单例 Bean，避免出现悬挂的资源
      destroyBeans();
      // Reset 'active' flag.
      cancelRefresh(ex);
      // Propagate exception to caller.
      throw ex;
    }

    finally {
      // Reset common introspection caches in Spring's core, since we
      // might not ever need metadata for singleton beans anymore...
      resetCommonCaches();
      contextRefresh.end();
    }
  }
}
```

可以看到，AbstractApplicationContext#refresh 方法使用 `synchronized` 关键字来修饰，**确保只启动一个 Spring 核心容器**。

AbstractApplicationContext#refresh 方法主要**和容器中 Bean 的创建有关**，如果 AbstractApplicationContext#refresh 执行流程无异常，则可以正确创建出所有需要的 Bean，并启动容器；如果 AbstractApplicationContext#refresh 方法执行过程中出现异常，则调用 AbstractApplicationContext#destroyBeans 方法摧毁所有已创建的单例 Bean。



<br>

说了这么多容器启动，**容器在何时启动？代码在哪里？**

兜兜转转又回到 SpringApplication#run，应用具体启动代码在后半段：

```java
refreshContext(context); // 刷新容器
// 容器刷新之后，提供一个 afterRefresh 钩子函数
afterRefresh(context, applicationArguments);
// ...
// 启动容器，在这里会将 StartupStep 设置为 started，从 refresh -> started
listeners.started(context, timeTakenToStartup);
// 将 context 上下文和应用启动参数传入 SpringApplication#callRunners 方法
// callRunners 方法会从 context 上下文中收集 ApplicationRunner.class 和 CommandLineRunner.class 类型的 Bean 
// 组成一个 runners 集合，然后再比较 runners 集合中各个 Runner 的运行优先级，
// 最后逐个执行 runners 集合中的每一个 Runner，优先级高的 Runner 先执行
callRunners(context, applicationArguments);
// ...
// 在这里会将 StartupStep 设置为 ready，从 started -> ready
listeners.ready(context, timeTakenToReady);
```

到这里，SpringBoot 应用算是启动了。



<br>

现在，我们回过头来挖掘之前我们忽略了的一些地方：**SpringBoot 启动过时还做了什么？**

将目光转移到 SpringApplication#run(Class[], String[])

```java
public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
	return new SpringApplication(primarySources).run(args);
}
```

之前我们忽略了 `new SpringApplication(primarySources)`，现在重新回过头来看，

```java
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
  // 设置启动类 Set
  this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
  // 从 Classpath 判断当前应用是否是 Web 应用
  this.webApplicationType = WebApplicationType.deduceFromClasspath();
	// ...
  // 设置主启动类
  this.mainApplicationClass = deduceMainApplicationClass();
}
```

现在，即将解决前面埋下的一个伏笔：**SpringBoot 应用如何判断当前应用是否是 Web 应用？**

<br>

### SpringBoot 中 Web 应用判断

主要看 WebApplicationType#deduceFromClasspath 这个方法：

```java
private static final String[] SERVLET_INDICATOR_CLASSES = { "javax.servlet.Servlet",
			"org.springframework.web.context.ConfigurableWebApplicationContext" };

private static final String WEBMVC_INDICATOR_CLASS = "org.springframework.web.servlet.DispatcherServlet";
private static final String WEBFLUX_INDICATOR_CLASS = "org.springframework.web.reactive.DispatcherHandler";
private static final String JERSEY_INDICATOR_CLASS = "org.glassfish.jersey.servlet.ServletContainer";

static WebApplicationType deduceFromClasspath() {
  // 类路径中只存在 org.springframework.web.reactive.DispatcherHandler
  // 表示当前应用是 Reactive 类型的 Web 应用
  if (ClassUtils.isPresent(WEBFLUX_INDICATOR_CLASS, null) && !ClassUtils.isPresent(WEBMVC_INDICATOR_CLASS, null)
      && !ClassUtils.isPresent(JERSEY_INDICATOR_CLASS, null)) {
    return WebApplicationType.REACTIVE;
  }
  // 类路径中不存在 javax.servlet.Servlet
  // 或 org.springframework.web.context.ConfigurableWebApplicationContext
  // 表示当前应用不是 Web 应用
  for (String className : SERVLET_INDICATOR_CLASSES) {
    if (!ClassUtils.isPresent(className, null)) {
      return WebApplicationType.NONE;
    }
  }
  // 否则当前是 Web 应用
  return WebApplicationType.SERVLET;
}
```

<br>

此外，SpringApplication#run 方法返回 ConfigurableApplicationContext，它是 ApplicationContext 接口的子接口，在 ApplicationContext 的基础上新增了一些方法：

* ConfigurableApplicationContext#getEnvironment 返回应用运行环境 ConfigurableEnvironment
* ConfigurableApplicationContext#getApplicationStartup 返回 ApplicationStartup 用来标记应用程序启动期间的步骤
* addApplicationListener 添加应用监听器

常用的 ConfigurableApplicationContext 实现类是 AbstractApplicationContext，它定义了一些和容器刷新相关的方法：

* AbstractApplicationContext#refresh
* AbstractApplicationContext#obtainFreshBeanFactory
* AbstractApplicationContext#prepareBeanFactory
* AbstractApplicationContext#postProcessBeanFactory
* AbstractApplicationContext#invokeBeanFactoryPostProcessors
* AbstractApplicationContext#registerBeanPostProcessors

这些方法在上面我们都有分析过，这里不做赘述。

> *注意*：在写代码的时候可能有些习惯让我们养成了向上转型的习惯，可能会将 ConfigurableApplicationContext 转成 ApplicationContext，**在这里不建议这么做**。因为 ConfigurableApplicationContext 的功能比 ApplicationContext 更丰富，使用 ConfigurableApplicationContext 是一个更好的选择。



<br>

**ConfigurableApplicationContext 在何时被创建？**

在 SpringApplication#run 方法中，调用 SpringApplication#createApplicationContext 方法实例化：

```java
ConfigurableApplicationContext context = null;
context = createApplicationContext();
```

继续挖掘，调用流程如下：

1、SpringApplication#createApplicationContext

2、DefaultApplicationContextFactory#create

3、DefaultApplicationContextFactory#getFromSpringFactories

```java
protected ConfigurableApplicationContext createApplicationContext() {
  return this.applicationContextFactory.create(this.webApplicationType);
}
```

```java
public ConfigurableApplicationContext create(WebApplicationType webApplicationType) {
  try {
    return getFromSpringFactories(webApplicationType, ApplicationContextFactory::create,
        AnnotationConfigApplicationContext::new);
  }
}
```

注意看这里的 DefaultApplicationContextFactory#getFromSpringFactories 方法，传入 `webApplicationType`、`ApplicationContextFactory::create`、`AnnotationConfigApplicationContext::new` 三个参数。

DefaultApplicationContextFactory#getFromSpringFactories 函数会根据 webApplicationType 参数判断：如果当前不是 Web 应用，则**默认使用** `AnnotationConfigApplicationContext::new` 创建 ConfigApplicationContext 实例；如果当前应用是 Web 应用，则会：

* 使用 AnnotationConfigServletWebServerApplicationContext.Factory#create
* 或者使用 AnnotationConfigReactiveWebServerApplicationContext.Factory#create

创建 ConfigApplicationContext。**具体使用哪一个类要看类路径中加载的类情况。**

```java
private <T> T getFromSpringFactories(WebApplicationType webApplicationType,
    BiFunction<ApplicationContextFactory, WebApplicationType, T> action, Supplier<T> defaultResult) {
  for (ApplicationContextFactory candidate : SpringFactoriesLoader.loadFactories(ApplicationContextFactory.class,
      getClass().getClassLoader())) {
    T result = action.apply(candidate, webApplicationType);
    if (result != null) { // 如果 ConfigApplicationContext 已创建直接返回 
      return result;
    }
  }
  // 否则使用默认的 AnnotationConfigApplicationContext::new 创建
  return (defaultResult != null) ? defaultResult.get() : null;
}
```



<br>

### 聊聊 BeanFactoryPostProcessor

对比 BeanPostProcessor 可以知道，BeanFactoryPostProcessor 应该是一个在获取了 BeanFactory 后执行的后置处理器。我们可以在 AbstractApplicationContext#refresh 中看到它的身影：

```java
invokeBeanFactoryPostProcessors(beanFactory);
```

执行所有的 BeanFactoryPostProcessor，该方法的描述是：实例化并调用所有已注册的BeanFactoryPostProcessor，如果给定了明确的顺序，则应遵守。且 BeanFactoryPostProcessor 必须在单例 Bean 实例化之前调用。

**为什么必须在单例 Bean 实例化之前调用？**

因为 BeanFactoryPostProcessor 的调用涉及到将 Bean 定义注册到 BeanFactory 中，需要先注册 Bean 定义才能对其进行实例化。

Bean 定义注册流程如下：

1、PostProcessorRegistrationDelegate#invokeBeanFactoryPostProcessors

2、PostProcessorRegistrationDelegate#invokeBeanDefinitionRegistryPostProcessors

3、ConfigurationClassPostProcessor#postProcessBeanDefinitionRegistry 重点<sup>*</sup>



<br>

### SpringBoot 应用的生命周期

曾经遇到过一个面试问题：**SpringBoot 应用的生命周期是什么？**

一下子没反应过来，竟不知道答 start、running、stop 还是答 Bean 的生命周期。当时还是太年轻，应该追问对方想知道 **SpringBoot 应用的生命周期**还是 **SpringBoot 容器中 Bean 的生命周期**。

下面来一一探讨。

<br>

这个问题很泛，如果你指的是 **SpringBoot 应用**，从简单来说，一个应用程序生命周期就是 start -> running -> stop。

如果指的是 **SpringBoot 中 context 容器上下文的生命周期**，从 ConfigurableApplicationContext 接口可以发现，它继承 **Lifecycle 接口**。Lifecycle 接口有 3 个方法，用来控制应用的启动和停止：

* start
* stop
* isRunning

所以从总体上来说，context 容器上下文的生命周期也就是这个三个 start、running、stop。再往细了分，**分成每一个启动步骤的话，即 StartupStep**，重点关注 SpringApplication#run 进行分析：

```java
public ConfigurableApplicationContext run(String... args) {
  DefaultBootstrapContext bootstrapContext = createBootstrapContext();
  ConfigurableApplicationContext context = null;
  // listeners 监听容器启动过程中的每一步，并为 StartupStep 赋值
  SpringApplicationRunListeners listeners = getRunListeners(args);
  // set StartupStep = application.starting
  listeners.starting(bootstrapContext, this.mainApplicationClass);
  try {
    // ...
    // StartupStep = application.environment-prepared
    ConfigurableEnvironment environment = prepareEnvironment(listeners, bootstrapContext, applicationArguments);
    // ...
    // StartupStep = context.annotated-bean-reader.create -> context.annotated-bean-reader.create end
    // annotated-bean-reader.create 步骤藏在 AnnotationConfigApplicationContext 构造方法中
    context = createApplicationContext();
    // 设置 StartupStep
    context.setApplicationStartup(this.applicationStartup);
    // StartupStep = application.context-prepared -> application.context-loaded
    prepareContext(bootstrapContext, context, environment, listeners, applicationArguments, printedBanner);
    // StartupStep = context.refresh -> context.beans.post-process -> context.beans.post-process end -> context.refresh end
    refreshContext(context);
    // StartupStep = application.started
    listeners.started(context, timeTakenToStartup);
  } catch () {
    // StartupStep = application.failed
    handleRunFailure(context, ex, null);
  }
  // ...
  try {
    // StartupStep = application.ready
    listeners.ready(context, timeTakenToReady);
  } catch () {
    // StartupStep = application.failed
    handleRunFailure(context, ex, null);
  }
  return context;
}
```

现在重新捋一下 StartupStep 涉及到的所有步骤：

1. application.starting
2. application.environment-prepared
3. context.annotated-bean-reader.create -> context.annotated-bean-reader.create end
4. application.context-prepared -> application.context-loaded
5. context.refresh -> context.beans.post-process -> context.beans.post-process end -> context.refresh end
6. application.started
7. application.ready
8. 如果 application.started 或 application.ready 步骤出现异常，则进入 application.failed

<br>

总结一下，可以简短的分为以下步骤：

1、应用启动

2、准备环境

3、context 上下文容器准备、容器加载、后置处理器执行

4、应用启动完成



<br>

观察 SpringBoot 源码我们还可以发现，有一个 ApplicationEvent 接口，在应用启动的时候也有它的身影。因此我们**还可以从 ApplicationEvent 入手分析 SpringBoot 应用的生命周期**。

SpringApplicationEvent 是 ApplicationEvent 的一个实现类，SpringApplicationEvent 的子类就代表着 SpringBoot 应用启动期间会发布的事件，我们也**可以使用发布出来的事件来描述 SpringBoot 应用的生命周期**。

将目光继续转移到 SpringApplication#run 方法，SpringApplicationEvent 由 `listeners` 监听器负责发布：

```java
// SpringApplication#run(String...)
SpringApplicationRunListeners listeners = getRunListeners(args);
```

按照事件发布的顺序由早到晚排序，包括以下事件：

org.springframework.boot.context.event.ApplicationStartingEvent

```java
listeners.starting(bootstrapContext, this.mainApplicationClass);
```

org.springframework.boot.context.event.ApplicationEnvironmentPreparedEvent

```java
ConfigurableEnvironment environment = prepareEnvironment(listeners, bootstrapContext, applicationArguments);
```

org.springframework.boot.context.event.ApplicationContextInitializedEvent

```java
prepareContext(bootstrapContext, context, environment, listeners, applicationArguments, printedBanner);
```

org.springframework.boot.context.event.ApplicationPreparedEvent

```java
// SpringApplication#run(String...)
prepareContext(bootstrapContext, context, environment, listeners, applicationArguments, printedBanner);

// SpringApplication#prepareContext
listeners.contextLoaded(context);
```

org.springframework.boot.context.event.ApplicationStartedEvent

```java
listeners.started(context, timeTakenToStartup);
```

org.springframework.boot.context.event.ApplicationReadyEvent

```java
listeners.ready(context, timeTakenToReady);
```

org.springframework.boot.context.event.ApplicationFailedEvent

```java
// SpringApplication#run(String...)
handleRunFailure(context, ex, listeners);

// SpringApplication#handleRunFailure
listeners.failed(context, exception);
```

以上就是 SpringBoot 应用启动期间所发布的事件了，总结一下：

1. ApplicationStartingEvent 在运行开始前发布
2. ApplicationEnvironmentPreparedEvent 在 context 创建前发布，为 context 提供运行环境
3. ApplicationContextInitializedEvent  在 context 初始化后发布
4. ApplicationPreparedEvent 在 context 加载后发布
5. ApplicationStartedEvent 在 context 刷新完成，但 SpringBoot 中的 runners 还未被调用时发布
6. AvailabilityChangeEvent 在 started 之后发布，紧随其后将应用状态 AvailabilityState 设置为 `LivenessState.CORRECT`
7. ApplicationReadyEvent 在 SpringBoot 中的 runners 被调用后发布
8. AvailabilityChangeEvent 在 ready 之后发布，紧随其后将应用状态 AvailabilityState 设置为 `ReadinessState.ACCEPTING_TRAFFIC` 表示应用可以处理请受
9. ApplicationFailedEvent 当应用启动过程中出现异常时发布

如果谈起 SpringBoot 应用的生命周期我们也可以从这方面来聊。



<br>可以看到，SpringBoot 应用启动期间**事件发布和启动步骤 StartupStep 都和 SpringApplicationRunListeners 有关**。 

SpringApplicationRunListeners 是 SpringApplicationRunListener 接口的合集，SpringApplicationRunListener 主要负责监听 SpringApplication#run 方法，SpringApplicationRunListener 接口内部定义了 SpringBoot 应用启动期间不同阶段需要执行的方法：

* starting
* environmentPrepared
* contextPrepared
* contextLoaded
* started
* ready
* failed

<br>

到这里，SpringBoot 应用的启动流程和生命周期暂告一段落。接下来探讨一下 Spring IoC 容器中 Bean。



<br>

### Bean 的注册流程

> 准确来说应该是注册了 BeanDefinition。

我们来聊一聊 **Spring 中 Bean 的创建流程，或者说 Bean 的生命周期**。首先要知道，在 Spring IoC 容器中，我们都是借助 BeanFactory 来创建 Bean 的。在启动的过程中需要先把 Bean 定义注册到 BeanFactory 中，后续需要创建/实例化的时候从 BeanFactory 中获取 Bean 信息，然后再进行实例化操作。



<br>

先**获取 BeanFactory**<sup>何时创建后面介绍</sup>：

1、AbstractApplicationContext#obtainFreshBeanFactory

2、GenericApplicationContext#getBeanFactory

再注册 Bean 定义。



<br>

自定义 Bean 注册流程

1、SpringApplication#prepareContext

2、SpringApplication#load

3、BeanDefinitionLoader#load(java.lang.Class<?>)

4、AnnotatedBeanDefinitionReader#register

5、AnnotatedBeanDefinitionReader#registerBean(java.lang.Class<?>)

6、AnnotatedBeanDefinitionReader#doRegisterBean

7、BeanDefinitionReaderUtils#registerBeanDefinition

以上步骤将**主启动类、配置类以及其他在类上标注了注解的类（@Service、@Repository 等）**，注册到 DefaultListableBeanFactory 中。

<br>

接下来注册剩下的 Bean，流程如下：

AbstractApplicationContext#refresh

AbstractApplicationContext#invokeBeanFactoryPostProcessors 

ConfigurationClassPostProcessor#processConfigBeanDefinitions

注意看 ConfigurationClassPostProcessor#processConfigBeanDefinitions 中的 for 循环，

```java
// ConfigurationClassPostProcessor#processConfigBeanDefinitions
for (String beanName : candidateNames) {
  BeanDefinition beanDef = registry.getBeanDefinition(beanName);
  if (beanDef.getAttribute(ConfigurationClassUtils.CONFIGURATION_CLASS_ATTRIBUTE) != null) {
    // ...
  }
  else if (ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory)) { // 如果是配置类，加入 configCandidates
    configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));
  }
}
```

遍历已经注册的 beanNames，检查遍历到的 bean 是否是配置类，如果是配置类就添加到 configClasses 集合中。继续往下，

```java
// ConfigurationClassPostProcessor#processConfigBeanDefinitions
this.reader.loadBeanDefinitions(configClasses);
```

遍历配置类集合，

```java
// ConfigurationClassBeanDefinitionReader#loadBeanDefinitions
public void loadBeanDefinitions(Set<ConfigurationClass> configurationModel) {
  TrackedConditionEvaluator trackedConditionEvaluator = new TrackedConditionEvaluator();
  for (ConfigurationClass configClass : configurationModel) { // 遍历配置类集合
    loadBeanDefinitionsForConfigurationClass(configClass, trackedConditionEvaluator);
  }
}
```

再从配置类中加载 @Bean 方法

```java
// ConfigurationClassBeanDefinitionReader#loadBeanDefinitionsForConfigurationClass
private void loadBeanDefinitionsForConfigurationClass(
    ConfigurationClass configClass, TrackedConditionEvaluator trackedConditionEvaluator) {
  // ...
  if (configClass.isImported()) {
    // 注册 @Import 注解指定的类
    registerBeanDefinitionForImportedConfigurationClass(configClass);
  }
  for (BeanMethod beanMethod : configClass.getBeanMethods()) { // 如果有 @Bean 方法，将其注册到容器中
    loadBeanDefinitionsForBeanMethod(beanMethod);
  }

  loadBeanDefinitionsFromImportedResources(configClass.getImportedResources());
 loadBeanDefinitionsFromRegistrars(configClass.getImportBeanDefinitionRegistrars());
}
```



<br>

扫描注册的逻辑是这样的：首先扫描类上的注解，比如：@SpringApplication、@Component、@Configuration 等注解，将带有注解的类注册到容器中。再遍历所有配置类，将配置类中 @Import 注解指定的类和 @Bean 方法注册到容器中。



<br>

为什么说前面的步骤**只是注册了 Bean 定义而不是创建 Bean 呢？**看一下上面的过程有没有执行类的构造函数就知道了。以下面的代码为例：

```java
@SpringBootApplication
public class SourceMain {
    public static void main(String[] args) {
        ConfigurableApplicationContext ac = SpringApplication.run(SourceMain.class, args);
    }
    @Bean(initMethod = "init", destroyMethod = "destroy")
    public User user() {
        return new User();
    }
}
public class User {
    public User() {
        System.out.println("non-args constructor");
    }
}
```

1、观察 ConfigurationClassBeanDefinitionReader#loadBeanDefinitionsForConfigurationClass 方法

2、发现 `loadBeanDefinitionsForBeanMethod(beanMethod)` 这一句代码执行完了

3、构造函数中的 "non-args constructor" 并没有输出



<br>

Spring 框架内置的 Bean 也是需要注册到 BeanFactory 中的，只是步骤和用户自定义的 Bean 注册步骤略有不同。

注册 internalXXXX 类型的 Bean：

1、SpringApplication#run

2、SpringApplication#createApplicationContext

3、DefaultApplicationContextFactory#create

4、AnnotationConfigApplicationContext#AnnotationConfigApplicationContext()

5、AnnotatedBeanDefinitionReader#AnnotatedBeanDefinitionReader()

6、AnnotationConfigUtils#registerAnnotationConfigProcessors

<br>

注册和 context 上下文相关的 Bean：

AbstractApplicationContext#prepareBeanFactory



<br>

说回 BeanFactory。上面只看到了 BeanFactory 的获取，**BeanFactory 在何时被创建？**

在创建 context 上下文的时候。以默认的 AnnotationConfigApplicationContext 为例，它是 GenericApplicationContext 的子类，而 BeanFactory 被定义在 GenericApplicationContext 中，

```java
// GenericApplicationContext
private final DefaultListableBeanFactory beanFactory;
```

在 GenericApplicationContext 构造函数被执行的时候会创建 BeanFactory：

```java
public GenericApplicationContext() {
  	this.beanFactory = new DefaultListableBeanFactory();
}
```

在实例化 AnnotationConfigApplicationContext 的时候也会执行到 GenericApplicationContext 的构造函数，流程如下：

1、SpringApplication#run(String...)

2、SpringApplication#createApplicationContext

3、DefaultApplicationContextFactory#create

4、AnnotationConfigApplicationContext#AnnotationConfigApplicationContext()

5、GenericApplicationContext#GenericApplicationContext()

在创建 context 上下文的时候 BeanFactory 同时会被创建。



<br>

注册完了 **Bean 在何时被创建？创建流程是什么样子的？**

<br>

### Bean 的创建流程

Spring 中自定义的 Bean 分为懒加载和非懒加载两种加载模式，由 @Lazy 注解来指定。此外还分为单例和原型两种类型。

对于非懒加载的单例 bean，IoC 容器刷新的时候会将其创建好，缓存到一个 ConcurrentHashMap 中，在后续使用的时候直接从这个缓存中拿；对于懒加载的单例 Bean 和原型 Bean，Spring IoC 容器会在我们需要的时候才执行创建操作。

<br>

首先看**非懒加载的单例 Bean**，创建流程如下：

1、AbstractApplicationContext#refresh

2、AbstractApplicationContext#finishBeanFactoryInitialization

3、DefaultListableBeanFactory#preInstantiateSingletons

重点关注 DefaultListableBeanFactory#preInstantiateSingletons 方法：

```java
// DefaultListableBeanFactory#preInstantiateSingletons
// Ensure that all non-lazy-init singletons are instantiated, also considering FactoryBeans
// 确保对所有非懒加载的单例 Bean 和 FactoryBean 进行实例化
public void preInstantiateSingletons() throws BeansException {
  // 遍历所有 Bean 定义，初始化所有非懒加载的单例 bean 和 FactoryBean
  // Trigger initialization of all non-lazy singleton beans...
  List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);
  for (String beanName : beanNames) {
    RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
    // 如果 bean 是单例且非懒加载
    if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
      if (isFactoryBean(beanName)) { // 如果是 FactoryBean 执行以下操作
        // FactoryBean 的 beanName 都有一个特定的前缀 “&”
        Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
        if (bean instanceof FactoryBean) {
          FactoryBean<?> factory = (FactoryBean<?>) bean;
          boolean isEagerInit;
          // 这个 if 检查当前 FactoryBean 是否需要立即初始化，isEagerInit
          // 其实就是看当前 FactoryBean 是否是懒加载
          if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
            isEagerInit = AccessController.doPrivileged(
                (PrivilegedAction<Boolean>) ((SmartFactoryBean<?>) factory)::isEagerInit,
                getAccessControlContext());
          }
          else {
            isEagerInit = (factory instanceof SmartFactoryBean &&
                ((SmartFactoryBean<?>) factory).isEagerInit());
          }
          if (isEagerInit) { // 立即初始化 FactoryBean
            getBean(beanName);
          }
        }
      }
      else { // 非 FactoryBean 的懒加载单例 Bean 直接初始化
        getBean(beanName); // 后续在详细介绍 getBean 方法
      }
    }
  }
  
  // 下面是 Bean 初始化后置处理
  // Trigger post-initialization callback for all applicable beans...
  for (String beanName : beanNames) { // 遍历所有 beanNames
    // 获取到已经创建好的单例 Bean
    Object singletonInstance = getSingleton(beanName);
    if (singletonInstance instanceof SmartInitializingSingleton) {
      // 开启一个新的 StartupStep = beans.smart-initialize
      StartupStep smartInitialize = this.getApplicationStartup().start("spring.beans.smart-initialize")
          .tag("beanName", beanName);
      // 将获取到的单例 Bean 转换为 SmartInitializingSingleton
      SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
      // 下面 if 方法就是执行单例 Bean 创建后的后置处理操作
      if (System.getSecurityManager() != null) {
        AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
          smartSingleton.afterSingletonsInstantiated();
          return null;
        }, getAccessControlContext());
      }
      else {
        smartSingleton.afterSingletonsInstantiated();
      }
      smartInitialize.end(); // beans.smart-initialize end
    }
  }
}
```



<br>

接下来看 AbstractBeanFactory#getBean 方法，具体的 Bean 创建流程如下 ：

1、AbstractBeanFactory#getBean

2、AbstractBeanFactory#doGetBean

3.1、DefaultSingletonBeanRegistry#getSingleton

3.2、AbstractAutowireCapableBeanFactory#createBean

4、AbstractAutowireCapableBeanFactory#doCreateBean

5、AbstractAutowireCapableBeanFactory#createBeanInstance 决定创建 Bean 的构造函数

6、AbstractAutowireCapableBeanFactory#instantiateBean

7、SimpleInstantiationStrategy#instantiate

8、BeanUtils#instantiateClass 利用 Java 反射机制，用构造函数创建 Bean



<br>

**创建 Bean 之后何时将其放入 IoC 容器的缓存中呢？**

在 DefaultSingletonBeanRegistry#getSingleton 方法中，让我们来看一下：

```java
// DefaultSingletonBeanRegistry

// Cache of singleton objects: bean name to bean instance 缓存单例 Bean
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
  synchronized (this.singletonObjects) { // 避免并发创建操作
    Object singletonObject = this.singletonObjects.get(beanName);
    if (singletonObject == null) {
      // if (this.singletonsCurrentlyInDestruction)
      
      beforeSingletonCreation(beanName);
      boolean newSingleton = false;
      boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
      if (recordSuppressedExceptions) {
        this.suppressedExceptions = new LinkedHashSet<>();
      }
      try {
        // 获取刚创建的单例 Bean
        singletonObject = singletonFactory.getObject();
        newSingleton = true;
      }
      // catch (ex) {}
      finally {
        if (recordSuppressedExceptions) {
          this.suppressedExceptions = null;
        }
        afterSingletonCreation(beanName);
      }
      if (newSingleton) { // 如果是新的单例 Bean，将其添加到缓存中
        addSingleton(beanName, singletonObject);
      }
    }
    return singletonObject;
  }
}
```



<br>

### Bean 的获取流程

到这里，Bean 的注册和创建流程已经理清了。那如果想要**使用一个已经创建了的非懒加载单例 Bean 该如何操作呢？**

可以**使用 AbstractApplicationContext#getBean 方法手动获取 Bean**，

```java
ConfigurableApplicationContext ac = SpringApplication.run(SourceMain.class, args);
User user = ac.getBean("user", User.class);
```

如果**指定了 beanName 和 beanType** 流程如下：

1、AbstractApplicationContext#getBean

2、AbstractBeanFactory#getBean
3、AbstractBeanFactory#doGetBean

4、DefaultSingletonBeanRegistry#getSingleton

5、DefaultSingletonBeanRegistry#getSingleton(String, boolean)，注意看这个方法和上面的 DefaultSingletonBeanRegistry#getSingleton(String, ObjectFactory<?>) 方法入参不同。

```java
// DefaultSingletonBeanRegistry#getSingleton

protected Object getSingleton(String beanName, boolean allowEarlyReference) {
  // 首先检查单例缓存中是否存在需要获取的 Bean，
  // 存在则直接返回，不存在则锁住 singletonObjects 然后从 singletonFactory 中获取，
  // 如果 singletonFactory 中不存在，则返回 null
  // Quick check for existing instance without full singleton lock
  Object singletonObject = this.singletonObjects.get(beanName);
  if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
    singletonObject = this.earlySingletonObjects.get(beanName);
    if (singletonObject == null && allowEarlyReference) {
      synchronized (this.singletonObjects) {
        // Consistent creation of early reference within full singleton lock
        singletonObject = this.singletonObjects.get(beanName);
        if (singletonObject == null) {
          singletonObject = this.earlySingletonObjects.get(beanName);
          if (singletonObject == null) {
            ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
            if (singletonFactory != null) {
              singletonObject = singletonFactory.getObject();
              this.earlySingletonObjects.put(beanName, singletonObject);
              this.singletonFactories.remove(beanName);
            }
          }
        }
      }
    }
  }
  return singletonObject;
}
```



<br>

如果没有指定 beanType，

```java
ConfigurableApplicationContext ac = SpringApplication.run(SourceMain.class, args);
User user = ac.getBean(User.class);
```

流程和上面的有所不同：

1、DefaultListableBeanFactory#getBean(Class<T>)

2、DefaultListableBeanFactory#getBean(Class<T>, .Object...)

3、DefaultListableBeanFactory#resolveBean

4、DefaultListableBeanFactory#resolveNamedBean(ResolvableType, Object[], boolean)<sup>*</sup> 关键在这里，如果存在多个同类型的 Bean 会在这个方法中判断

5、DefaultListableBeanFactory#resolveNamedBean(String, ResolvableType, Object[])

6、AbstractBeanFactory#getBean(String, Class<T>, Object...)

7、AbstractBeanFactory#doGetBean

8、DefaultSingletonBeanRegistry#getSingleton(String)

9、DefaultSingletonBeanRegistry#getSingleton(String, boolean)

分析一下 DefaultListableBeanFactory#resolveNamedBean(ResolvableType, Object[], boolean) 这个方法：

```java
private <T> NamedBeanHolder<T> resolveNamedBean(
    ResolvableType requiredType, @Nullable Object[] args, boolean nonUniqueAsNull) throws BeansException {

  // Return the names of beans matching the given type (including subclasses)
  // 获取和 requiredType 及其子类对应的所有 candidateName
  String[] candidateNames = getBeanNamesForType(requiredType);

  // 过滤 candidateNames
  if (candidateNames.length > 1) {
    List<String> autowireCandidates = new ArrayList<>(candidateNames.length);
    for (String beanName : candidateNames) {
      // !containsBeanDefinition(beanName) 此处卡了一下，思考为什么不是 containsBeanDefinition(beanName) 呢？
      // 因为 beanName 可能是别名，而注册到 BeanFactory 的定义的名称不是别名，
      // 所以此处就是为了处理存在别名的情况
      // getBeanDefinition(beanName).isAutowireCandidate()) 存在 bean 定义且可注入
      if (!containsBeanDefinition(beanName) || getBeanDefinition(beanName).isAutowireCandidate()) {
        autowireCandidates.add(beanName);
      }
    }
    if (!autowireCandidates.isEmpty()) {
      candidateNames = StringUtils.toStringArray(autowireCandidates);
    }
  }

  // 如果 requiredType 只有一个 Bean 存在，直接返回
  if (candidateNames.length == 1) {
    return resolveNamedBean(candidateNames[0], requiredType, args);
  }
  else if (candidateNames.length > 1) { // 如果存在多个同类型的 Bean
    // 使用一个 Map 保存候选 Bean
    Map<String, Object> candidates = CollectionUtils.newLinkedHashMap(candidateNames.length);
    for (String beanName : candidateNames) {
      // 如果单例缓存中存在
      if (containsSingleton(beanName) && args == null) {
        // 则直接获取，然后加入 candidates
        Object beanInstance = getBean(beanName);
        candidates.put(beanName, (beanInstance instanceof NullBean ? null : beanInstance));
      }
      else {
        candidates.put(beanName, getType(beanName)); // 加入 candidates
      }
    }
    // 判断 candidates 中是否存在 @Primary 标注的类
    String candidateName = determinePrimaryCandidate(candidates, requiredType.toClass());
    if (candidateName == null) { // 如果不存在 @Primary 标注的类，取最高优先级的 Bean
      candidateName = determineHighestPriorityCandidate(candidates, requiredType.toClass());
    }
    if (candidateName != null) { // 如果存在 @Primary，返回标注了的 Bean
      Object beanInstance = candidates.get(candidateName);
      if (beanInstance == null) {
        return null;
      }
      if (beanInstance instanceof Class) {
        return resolveNamedBean(candidateName, requiredType, args);
      }
      return new NamedBeanHolder<>(candidateName, (T) beanInstance);
    }
    if (!nonUniqueAsNull) {
      throw new NoUniqueBeanDefinitionException(requiredType, candidates.keySet());
    }
  }

  return null;
}
```



<br>

到这里，Bean 的获取就暂时分析完了，接下来总结一下前面的几个点，来看看 SpringBoot 中 Bean 的生命周期。

<br>

### Bean 的生命周期

自定义非懒加载单例 Bean 的生命周期如下：

1. 构造方法
2. BeanNameAware#setBeanName（此外还有一系列 XXXAware 接口）
3. BeanPostProcessor#postProcessBeforeInitialization
4. @PostConstruct 标注的方法
5. InitializingBean#afterPropertiesSet
6. initMethod（@Bean 的 initMethod 属性指定的方法）
7. BeanPostProcessor#postProcessAfterInitialization
8. getter/setter
9. 使用 Bean
10. @PreDestroy 标注的方法
11. destroy 方法（@Bean 的 destroy 属性指定的方法或重写 DisposableBean#destroy）

<br>

> 如果想要在 Bean 的构造方法执行之后对 Bean 进行一些操作，SpringBoot 官方推荐使用 @PostConstruct 而非 InitializingBean。因为使用 @PostConstruct 可以**减少业务代码和 Spring 代码之间的耦合。**
>
> 发散官方的思维，如果想要在 Bean 被销毁的时候做点别的操作，那当然最好是在 @Bean 的 destroy 属性指定销毁方法，执行对应的业务逻辑，这样可以减少业务代码和 Spring 框架代码的耦合。

<br>

生命周期观察代码如下：

```java
public class User implements ApplicationContextAware, BeanNameAware, InitializingBean, DisposableBean {
    private int id;
    private String name;

    @PostConstruct
    public void postConstruct() {
        System.out.println("PostConstruct");
    }

    @PreDestroy
    public void preDestroy() {
        System.out.println("preDestroy");
    }

    public void init() {
        System.out.println("init");
    }

    public void destroy() {
        System.out.println("destroy");
    }

    public User() {
        System.out.println("non-args constructor");
    }
    public int getId() {
        System.out.println("getting id");
        return id;
    }
    public void setId(int id) {
        System.out.println("setting id");
        this.id = id;
    }
    public String getName() {
        System.out.println("getting name");
        return name;
    }
    public void setName(String name) {
        System.out.println("setting name");
        this.name = name;
    }

    @Override
    public void setBeanName(String name) {
        System.out.println("BeanNameAware#setBeanName");
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("InitializingBean#afterPropertiesSet");
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        System.out.println("ApplicationContextAware#setApplicationContext");
    }
}
```

```java
// BeanPostProcessor 实现类会应用于所有的 Bean，可以通过 beanName 来判定具体需要处理的 Bean
@Component
public class CusBeanPostProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
      System.out.println("BeanPostProcessor#postProcessBeforeInitialization");
        return bean;
    }
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("BeanPostProcessor#postProcessAfterInitialization");
        return bean;
    }
}
```

```java
@SpringBootApplication
public class SourceMain {
    public static void main(String[] args) {
        ConfigurableApplicationContext ac = SpringApplication.run(SourceMain.class, args);
        User user = ac.getBean("user", User.class);
        user.setId(1);
        int userId = user.getId();
        System.out.println(user);
    }
    @Bean(initMethod = "init", destroyMethod = "destroy")
    public User user() {
        return new User();
    }
}
```



<br>

### Bean 的作用域

[Spring 官方文档](https://docs.spring.io/spring-framework/reference/core/beans/factory-scopes.html)有讲到，Bean 的作用域：

* [singleton](https://docs.spring.io/spring-framework/reference/core/beans/factory-scopes.html#beans-factory-scopes-singleton)
* [prototype](https://docs.spring.io/spring-framework/reference/core/beans/factory-scopes.html#beans-factory-scopes-prototype)
* [request](https://docs.spring.io/spring-framework/reference/core/beans/factory-scopes.html#beans-factory-scopes-request)
* [session](https://docs.spring.io/spring-framework/reference/core/beans/factory-scopes.html#beans-factory-scopes-session)
* [application](https://docs.spring.io/spring-framework/reference/core/beans/factory-scopes.html#beans-factory-scopes-application)
* [websocket](https://docs.spring.io/spring-framework/reference/web/websocket/stomp/scope.html)



<br>

> *注意*：注意区分 Bean 的创建流程和 Bean 的作用域



<br>

### 自动配置原理分析

先从 @Configuration 入手，先讲一下它的 proxyBeanMethods 属性，顾名思义：代理 Bean 方法。

```java
boolean proxyBeanMethods() default true;
```

源码中是这么说的：

> Specify whether @Bean methods should get proxied in order to enforce bean lifecycle behavior.

翻译过来就是：指定 @Bean 方法是否应该被代理，以便执行 Bean 生命周期的各种行为。其实就是说，你是否需要对 @Bean 方法进行代理。先来看一下它的表现：

```java
@Configuration(proxyBeanMethods = true)
public class CusConfig {
    @Bean
    public ServiceA serviceA() {
        return new ServiceA(serviceC()); // 直接调用 serviceC() 方法
    }
    @Bean
    public ServiceB serviceB() {
        return new ServiceB(serviceC()); // 直接调用 serviceC() 方法
    }
    @Bean
    public ServiceC serviceC() {
        ServiceC serviceC = new ServiceC();
        System.out.println(serviceC);
        return serviceC;
    }
}
```

默认值是 true，表示要求 SpringBoot 对 CusConfig 类中的 3 个 @Bean 方法都进行代理。可以看到上面 serviceA() 和 serviceB() 方法都调用到了 serviceC() 方法，按照正常的逻辑来说理应对 ServiceC 进行多次实例化。但是这里设置 proxyBeanMethods=true，SpringBoot 就会对 serviceC() 进行代理，最终不管调用多少次 serviceC() 方法返回的都是 SpringBoot 生成的代理对象，都是同一个对象。

这也被称为 Full 模式，对配置类中每个直接调用 @Bean 方法的地方进行代理。有点抽象，从输出结果观察一下：

```java
ServiceA a = ac.getBean(ServiceA.class);
ServiceB b = ac.getBean(ServiceB.class);
ServiceC c = ac.getBean(ServiceC.class);
```

直接调用 serviceC() 方法的 true 模式，输出内容如下，只创建了一个单例 ServiceC：

```
com.boot.bean.ServiceC@622ef26a
com.boot.bean.ServiceC@622ef26a
com.boot.bean.ServiceC@622ef26a
```

在 SpringBoot 2 之前，使用的都是 Full 模式，随着配置类的增加，SpringBoot 需要创建的代理类也随着增加，生成太多代理类会拖慢 SpringBoot 的启动。为了加快启动速度，SpringBoot 引入了一个 Lite 模式，就是将 proxyBeanMethods 设置为 false。

继续看刚才的代码，将 proxyBeanMethods 设置为 false：

```java
@Configuration(proxyBeanMethods = false)
public class CusConfig {
    @Bean
    public ServiceA serviceA() {
        return new ServiceA(serviceC()); // 直接调用 serviceC() 方法
    }
    @Bean
    public ServiceB serviceB() {
        return new ServiceB(serviceC()); // 直接调用 serviceC() 方法
    }
    @Bean
    public ServiceC serviceC() {
        ServiceC serviceC = new ServiceC();
        System.out.println(serviceC);
        return serviceC;
    }
}
```

直接调用 serviceC() 方法的 false 模式，输出内容如下，创建了 3 个不同的 ServiceC 对象：

```
com.boot.bean.ServiceC@e72dba7
com.boot.bean.ServiceC@33c2bd
com.boot.bean.ServiceC@1dfd5f51
```

**如何在 proxyBeanMethods=false 的情况下使用同一个单实例呢？**只要将直接方法调用更换成方法参数注入即可：

```java
@Configuration(proxyBeanMethods = false)
public class CusConfig {
    @Bean
    public ServiceA serviceA(ServiceC serviceC) {
        return new ServiceA(serviceC);
    }
    @Bean
    public ServiceB serviceB(ServiceC serviceC) {
        return new ServiceB(serviceC);
    }
    @Bean
    public ServiceC serviceC() {
        return new ServiceC();
    }
}
```

这样才能保证创建出来的 ServiceC 在容器中是一个单实例。

<br>

接下来，看看 SpringBoot 如何导入类进行配置。和下面这几个注解有关：

* @Import，容器导入一个或多个配置类

* @ImportResource，导入一个或多个配置文件（properties/yaml）

* @Conditional，根据条件导入组件

  ```java
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

  

<br>

回到标题，继续来分析 SpringBoot 的自动配置。看到 @SpringBootApplication，它是由 3 个注解组成的复合注解：

* @SpringBootConfiguration，SpringBoot 应用只能有一个 @SpringBootConfiguration 注解，可以看成是 @Configuration 的替代。
* @EnableAutoConfiguration，为 SpringBoot 应用开启自动配置功能。
* @ComponentScan，负责 SpringBoot 应用组件扫描。

<br>

重点在 @EnableAutoConfiguration，它也是一个复合注解：

```java
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration
```

* @AutoConfigurationPackage，自动导入包中的类
* @Import(AutoConfigurationImportSelector.class)，AutoConfigurationImportSelector 用来筛选真正需要导入的类。

<br>

AutoConfigurationImportSelector 是一个自动配置导入选择器，继承自很多接口：

* DeferredImportSelector 接口，主要用来定义延迟配置自动导入规则。
* BeanClassLoaderAware、ResourceLoaderAware、BeanFactoryAware、EnvironmentAware 在启动的各个流程进行操作。
* Ordered 定义类实例化的顺序，Ordered#getOrder 方法返回的数值越小，类实例化的优先级就越高。

<br>

**SpringBoot 从哪里决定自动导入的类呢？**主要有两个文件，都定义在 spring-boot-autoconfigure 模块中：

* AutoConfiguration.imports
* spring.factories

<br>

先讲 *.imports 文件。SpringBoot 在启动的过程中会检查 spring-boot-autoconfigure 模块中的 

* `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 文件，

该文件中包含了一系列需要 SpringBoot 自动导入的类。比如我们导入了一个 spring-boot-starter-web 模块，SpringBoot 在启动的时候会扫描 Classpath 路径，如果发现存在 AutoConfiguration.imports 中定义的类，比如：

```
org.springframework.boot.autoconfigure.web.servlet.DispatcherServletAutoConfiguration
org.springframework.boot.autoconfigure.web.servlet.ServletWebServerFactoryAutoConfiguration
org.springframework.boot.autoconfigure.web.servlet.error.ErrorMvcAutoConfiguration
org.springframework.boot.autoconfigure.web.servlet.HttpEncodingAutoConfiguration
org.springframework.boot.autoconfigure.web.servlet.MultipartAutoConfiguration
org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration
```

SpringBoot 就会使用 ImportCandidates 类将它们导入。

```java
public final class ImportCandidates implements Iterable<String> {
	private static final String LOCATION = "META-INF/spring/%s.imports";
}
```

<br>

此外还有一个 `META-INF/spring.factories` 文件，

```
# Auto Configuration Import Listeners
org.springframework.boot.autoconfigure.AutoConfigurationImportListener=\
org.springframework.boot.autoconfigure.condition.ConditionEvaluationReportAutoConfigurationImportListener

# Auto Configuration Import Filters
org.springframework.boot.autoconfigure.AutoConfigurationImportFilter=\
org.springframework.boot.autoconfigure.condition.OnBeanCondition,\
org.springframework.boot.autoconfigure.condition.OnClassCondition,\
org.springframework.boot.autoconfigure.condition.OnWebApplicationCondition
```

SpringBoot 使用 SpringFactoriesLoader 类来加载 spring.factories 文件。

```java
public final class SpringFactoriesLoader {
	// The location to look for factories. Can be present in multiple JAR files.
	public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";
}
```

> 从 SpringBoot 2.7 开始官方逐步使用 AutoConfiguration.imports 来代替 spring.factories 文件。在 SpringBoot 3.0 更是直接将 spring.factories 标注为弃用。

SpringFactoriesLoader 加载 spring.factories 可以看成是 Java SPI 的变种。只不过 SpringBoot 使用的是 SpringFactoriesLoader 来加载，Java 使用的是 ServiceLoader 类来加载。

<br>

自动配置导入的配置文件 *.import 和 spring.factories 文件一般都在 spring-boot-xxx-autoconfigure 包中。这就涉及到 starter 了，**什么是 spring-boot-starter 呢？**

<br>

一个 starter 应该包括两部分内容：

* starter，没什么大用，实际上是一个空的 jar 包，提供 autoconfigure 模块需要的依赖。
* autoconfigure，包含自动配置类，*.import 和 spring.factories 文件都在里面。

如果配置比较简单，可以将两个模块合并成一个。

…

### AutoConfigurationImportSelector#selectImports

> [方法解析](https://juejin.cn/post/7006909137925701639)

<br>

## Web 源码分析

### Tomcat 启动流程

SpringBoot 的 Web 模块内置 Tomcat，使用起来很方便，只需要引入 spring-boot-starter-web 依赖即可使用。相比于 Spring 不再需要复杂的配置，开箱即用。那么相比于外部手动配置的 Tomcat，**SpringBoot 内部的 Tomcat 是如何启动的呢？**

在上面 [SpringBoot 的启动流程中](#SpringBoot 中 Web 应用判断)，我们已经分析过，SpringBoot 是根据类路径中有无 Web 应用相关的类来判断的：

1、SpringApplication#SpringApplication(ResourceLoader, Class<?>...)

```java
this.webApplicationType = WebApplicationType.deduceFromClasspath();
```

2、WebApplicationType#deduceFromClasspath

```java
private static final String[] SERVLET_INDICATOR_CLASSES = { "javax.servlet.Servlet",
    "org.springframework.web.context.ConfigurableWebApplicationContext" };

private static final String WEBMVC_INDICATOR_CLASS = "org.springframework.web.servlet.DispatcherServlet";

private static final String WEBFLUX_INDICATOR_CLASS = "org.springframework.web.reactive.DispatcherHandler";

private static final String JERSEY_INDICATOR_CLASS = "org.glassfish.jersey.servlet.ServletContainer";

static WebApplicationType deduceFromClasspath() {
  // WebFlux
  if (ClassUtils.isPresent(WEBFLUX_INDICATOR_CLASS, null) && !ClassUtils.isPresent(WEBMVC_INDICATOR_CLASS, null)
      && !ClassUtils.isPresent(JERSEY_INDICATOR_CLASS, null)) {
    return WebApplicationType.REACTIVE;
  }
  // 非 Web 应用
  for (String className : SERVLET_INDICATOR_CLASSES) {
    if (!ClassUtils.isPresent(className, null)) {
      return WebApplicationType.NONE;
    }
  }
  return WebApplicationType.SERVLET; // 否则是 WebMVC
}
```

3、SpringApplication#createApplicationContext

```java
protected ConfigurableApplicationContext createApplicationContext() {
  // 假设引入了 spring-boot-starter-web
  // this.webApplicationType = WebApplicationType.SERVLET
  return this.applicationContextFactory.create(this.webApplicationType);
}
```

4、DefaultApplicationContextFactory#create

```java
public ConfigurableApplicationContext create(WebApplicationType webApplicationType) {
  try {
    // getFromSpringFactories 会根据 webApplicationType 创建对应的 context 上下文
    // AnnotationConfigApplicationContext
    // or AnnotationConfigServletWebServerApplicationContext
    // or AnnotationConfigReactiveWebServerApplicationContext
    return getFromSpringFactories(webApplicationType, ApplicationContextFactory::create,
        AnnotationConfigApplicationContext::new);
  }
}
```

假设我们引入了 spring-boot-starter-web 就会拿到 AnnotationConfigServletWebServerApplicationContext 实例，

同时它还是 ServletWebServerApplicationContext 的子类（接下来要分析），接下来我们需要关注的是，**Tomcat 如何启动？**

<br>

还记得 AbstractApplicationContext#refresh 方法中的 AbstractApplicationContext#onRefresh 吗？在分析 SpringBoot 启动流程的时候我们见过它，并且在纯 SpringBoot（非 Web 应用）的时候它就是一个空实现。但是在 Web 应用中它却是我们关注的重点。

如果引入了 Web MVC 相关的依赖，AbstractApplicationContext#onRefresh 的实现是 ServletWebServerApplicationContext#onRefresh。

```java
// ServletWebServerApplicationContext
protected void onRefresh() {
  super.onRefresh();
  try {
    createWebServer(); // 重点在这里
  }
}

private void createWebServer() {
  // 在实例化 AnnotationConfigServletWebServerApplicationContext 的时候
  // 也会实例化 ServletWebServerApplicationContext
  // 此时 this.webServer = null
  WebServer webServer = this.webServer;
  ServletContext servletContext = getServletContext();
  // 第一次启动 webServer = null，servletContext = null
  if (webServer == null && servletContext == null) {
    StartupStep createWebServer = this.getApplicationStartup().start("spring.boot.webserver.create");
    // getWebServerFactory 会得到一个名为 tomcatServletWebServerFactory 的 Bean
    // 如何得到呢？在之前分析 SpringBoot 启动流程的时候已经分析过了
    // 关键在 AbstractApplicationContext#refresh 方法中的
    // AbstractApplicationContext#invokeBeanFactoryPostProcessors 方法
    // 注册所需要的 Bean 定义
    // 包括 tomcatServletWebServerFactory 这个 Bean
    ServletWebServerFactory factory = getWebServerFactory();
    createWebServer.tag("factory", factory.getClass().toString());
    // getWebServer 是关键的一步
    // 调用 TomcatServletWebServerFactory#getWebServer 创建内嵌 Tomcat
    this.webServer = factory.getWebServer(getSelfInitializer()); // *
    createWebServer.end();
    getBeanFactory().registerSingleton("webServerGracefulShutdown",
        new WebServerGracefulShutdownLifecycle(this.webServer));
    getBeanFactory().registerSingleton("webServerStartStop",
        new WebServerStartStopLifecycle(this, this.webServer));
  }
  else if (servletContext != null) {
    try {
      getSelfInitializer().onStartup(servletContext);
    } //catch (ServletException ex) {}
  }
  initPropertySources();
}
```

我们的关注点在标 `*` 的那一行代码，是创建内嵌 Tomcat 服务器的关键，调用的是 TomcatServletWebServerFactory#getWebServer 方法对 Tomcat 进行实例化，并做一些初始化操作：

```java
public WebServer getWebServer(ServletContextInitializer... initializers) {
  if (this.disableMBeanRegistry) {
    Registry.disableRegistry();
  }
  Tomcat tomcat = new Tomcat(); // 实例化 tomcat
  // 设置 tomcat 根路径
  File baseDir = (this.baseDirectory != null) ? this.baseDirectory : createTempDir("tomcat");
  tomcat.setBaseDir(baseDir.getAbsolutePath());
  // 设置 tomcat 生命周期监听器
  for (LifecycleListener listener : this.serverLifecycleListeners) {
    tomcat.getServer().addLifecycleListener(listener);
  }
  // 对 tomcat 进行初始化
  Connector connector = new Connector(this.protocol); // 默认 Http11NioProtocol
  connector.setThrowOnFailure(true);
  tomcat.getService().addConnector(connector);
  // 自定义 Connector
  // 其中会设置默认端口 8080
  customizeConnector(connector);
  tomcat.setConnector(connector);
  tomcat.getHost().setAutoDeploy(false);
  // Engine 是 Tomcat 最顶层的容器
  // 其中包含多个 Service
  // Service 是一个或者多个 Connector 的集合
  configureEngine(tomcat.getEngine());
  for (Connector additionalConnector : this.additionalTomcatConnectors) {
    tomcat.getService().addConnector(additionalConnector);
  }
  prepareContext(tomcat.getHost(), initializers);
  return getTomcatWebServer(tomcat);
}
```

再上面的代码中可以看到 `customizeConnector(connector);` 这一句代码自定义 Connector，设置了默认的启动端口 8080，其实是使用 TomcatConnectorCustomizer 接口来实现自定义操作的。当然你也可以自定义一个 TomcatConnectorCustomizer 的实现类，对 Tomcat 进行一些自定义操作。你自定义的操作会覆盖默认的操作。

往后看，在 TomcatServletWebServerFactory#getTomcatWebServer 中：

```java
protected TomcatWebServer getTomcatWebServer(Tomcat tomcat) {
  return new TomcatWebServer(tomcat, getPort() >= 0, getShutdown());
}
```

使用一个 TomcatWebServer 类包装并启动 Tomcat：

```java
// 包装 Tomcat
public TomcatWebServer(Tomcat tomcat, boolean autoStart, Shutdown shutdown) {
  this.tomcat = tomcat;
  this.autoStart = autoStart;
  this.gracefulShutdown = (shutdown == Shutdown.GRACEFUL) ? new GracefulShutdown(tomcat) : null;
  initialize();
}

// 初始化并启动 Tomcat
private void initialize() throws WebServerException {
  synchronized (this.monitor) { // 保证只启动一个 Tomcat
    try {
      addInstanceIdToEngineName();

      Context context = findContext();
      context.addLifecycleListener((event) -> {
        if (context.equals(event.getSource()) && Lifecycle.START_EVENT.equals(event.getType())) {
          // Remove service connectors so that protocol binding doesn't
          // happen when the service is started.
          removeServiceConnectors();
        }
      });
      // Start the server to trigger initialization listeners
      this.tomcat.start(); // 启动 Tomcat
			// ...
      try {
        ContextBindings.bindClassLoader(context, context.getNamingToken(), getClass().getClassLoader());
      }
      // 所有的 Tomcat 线程都是守护线程，这里创建一个非阻塞的非守护线程用来关闭 Tomcat
      // Unlike Jetty, all Tomcat threads are daemon threads. We create a
      // blocking non-daemon to stop immediate shutdown
      startDaemonAwaitThread();
    }
  }
}
```

到这里 Tomcat 的启动就结束了，总的来说就是在创建上下文 context 的时候创建并启动 Tomcat。



<br>

## 参考

https://docs.spring.io/spring-framework/reference/core.html

https://blog.csdn.net/weixin_42189048/article/details/108673635

https://mp.weixin.qq.com/s/yLf8Ap3efxO4Fv4UpUIgIg