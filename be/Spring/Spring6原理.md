# spring-6

> 阅读顺序[Spring5原理](Spring5原理.md) => [Spring6原理](Spring6原理.md)

> 以单例创建为例

依旧是从 `SpringApplication.run(Main.class, args);` 开始。

## IOC

### IOC 容器创建

```shell
org.springframework.boot.SpringApplication#run(java.lang.Class<?>, java.lang.String...)
|- org.springframework.boot.SpringApplication#SpringApplication(org.springframework.core.io.ResourceLoader, java.lang.Class<?>...)
```

创建 SpringApplication 实例

```java
// org.springframework.boot.SpringApplication#SpringApplication(org.springframework.core.io.ResourceLoader, java.lang.Class<?>...)

public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) { // primarySources 表示主配置类，可以是当前模块的类，也可以是其他模块的类
    this.resourceLoader = resourceLoader;
    Assert.notNull(primarySources, "PrimarySources must not be null");
    this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
    this.properties.setWebApplicationType(WebApplicationType.deduceFromClasspath());
    this.bootstrapRegistryInitializers = new ArrayList<>(getSpringFactoriesInstances(BootstrapRegistryInitializer.class));
    // 设置 SpringContext 初始化器
    setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
    // 设置 SpringContext 监听器
    setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
    // 使用 StackWalker 从当前线程栈帧获取到当前主启动类，实际就是 Objects.equals(frame.getMethodName(), "main")) == 从方法名获取当前主启动类
    this.mainApplicationClass = deduceMainApplicationClass();
}
```

判断当前的 Web 应用类型

```shell
org.springframework.boot.SpringApplication#SpringApplication(org.springframework.core.io.ResourceLoader, java.lang.Class<?>...)
|- org.springframework.boot.WebApplicationType#deduceFromClasspath
```

```java
// org.springframework.boot.WebApplicationType

NONE, // The application should not run as a web application and should not start an embedded web server.
SERVLET, // The application should run as a servlet-based web application and should start an embedded servlet web server.
REACTIVE; // The application should run as a reactive web application and should start an embedded reactive web server.

private static final String[] SERVLET_INDICATOR_CLASSES = { "jakarta.servlet.Servlet", "org.springframework.web.context.ConfigurableWebApplicationContext" };

private static final String WEBMVC_INDICATOR_CLASS = "org.springframework.web.servlet.DispatcherServlet";
private static final String WEBFLUX_INDICATOR_CLASS = "org.springframework.web.reactive.DispatcherHandler";
private static final String JERSEY_INDICATOR_CLASS = "org.glassfish.jersey.servlet.ServletContainer";

static WebApplicationType deduceFromClasspath() {
    if (ClassUtils.isPresent(WEBFLUX_INDICATOR_CLASS, null) && !ClassUtils.isPresent(WEBMVC_INDICATOR_CLASS, null)
            && !ClassUtils.isPresent(JERSEY_INDICATOR_CLASS, null)) {
        return WebApplicationType.REACTIVE;
    }
    for (String className : SERVLET_INDICATOR_CLASSES) {
        if (!ClassUtils.isPresent(className, null)) {
            return WebApplicationType.NONE;
        }
    }
    return WebApplicationType.SERVLET;
}
```

如果 classpath 中存在：
- WEBMVC_INDICATOR_CLASS 判断为 WebMVC 应用
- WEBFLUX_INDICATOR_CLASS 判断为 WebFlux 应用

```shell
org.springframework.boot.SpringApplication#run(java.lang.Class<?>[], java.lang.String[])
| - org.springframework.boot.SpringApplication#run(java.lang.String...)
```

```java
// org.springframework.boot.SpringApplication#run(java.lang.String...)

// Run the Spring application, creating and refreshing a new ApplicationContext
public ConfigurableApplicationContext run(String... args) {
    Startup startup = Startup.create();
    if (this.properties.isRegisterShutdownHook()) {
        SpringApplication.shutdownHook.enableShutdownHookAddition();
    }
    // 一个简单的引导上下文，可在应用启动和应用环境启动后置处理的期间使用，为应用的初始化过程提供支持。
    // 支持延迟加载一些可能创建成本较高或需要在 ApplicationContext 初始化之前共享的对象
    // A simple bootstrap context that is available during startup and Environment post-processing up to the point that the ApplicationContext is prepared.
    // Provides lazy access to singletons that may be expensive to create, or need to be shared before the ApplicationContext is available.
    DefaultBootstrapContext bootstrapContext = createBootstrapContext();
    // 此时 ApplicationContext 还未初始化，BootstrapContext 主要在 ApplicationContext 准备完成之前提供临时的上下文支持。
    ConfigurableApplicationContext context = null;
    configureHeadlessProperty();
    // 注册 ApplicationContext 启动监听器（应用启动监听器）
    SpringApplicationRunListeners listeners = getRunListeners(args);
    listeners.starting(bootstrapContext, this.mainApplicationClass);
    try {
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
        // Create and configure the environment
        /*
         创建并配置对应的环境（WebMVC/WebFlux 环境）
         会根据
         - META-INF/spring.factories（位置在 spring-boot 或者自定义配置）
         - META-INF/org.springframework.boot.autoconfigure.AutoConfiguration.imports （位置在 spring-boot-autoconfigure）
         配置文件加载对应的 AutoConfiguration 配置类
         */
        ConfigurableEnvironment environment = prepareEnvironment(listeners, bootstrapContext, applicationArguments);
        Banner printedBanner = printBanner(environment); // 打印 Banner
        context = createApplicationContext(); // 创建 IOC 容器
        context.setApplicationStartup(this.applicationStartup);
        /*
            进行 IOC 容器的初始化
            - 配置环境 ApplicationEnvironment/ReactiveEnvironment/ServletEnvironment
            - 后置处理 postProcessApplicationContext
            - 关闭 bootstrapContext
            - 加载一些关键 Bean
         */
        prepareContext(bootstrapContext, context, environment, listeners, applicationArguments, printedBanner);
        // 刷新容器 ==> 可跳转到后面 [Bean 创建流程]
        refreshContext(context);
        afterRefresh(context, applicationArguments);
        startup.started();
        if (this.properties.isLogStartupInfo()) {
            new StartupInfoLogger(this.mainApplicationClass, environment).logStarted(getApplicationLog(), startup);
        }
        listeners.started(context, startup.timeTakenToStarted());
        callRunners(context, applicationArguments);
    }
    catch (Throwable ex) {
        throw handleRunFailure(context, ex, listeners);
    }
    try {
        if (context.isRunning()) {
            listeners.ready(context, startup.ready());
        }
    }
    catch (Throwable ex) {
        throw handleRunFailure(context, ex, null);
    }
    return context;
}
```

<br>

### 配置类扫描

从 `@ComponentScan` 开始

```shell
org.springframework.context.support.AbstractApplicationContext#refresh
|- org.springframework.context.support.AbstractApplicationContext#invokeBeanFactoryPostProcessors # 在容器上下文中注册 Bean。Invoke factory processors registered as beans in the context
|- ...
|- org.springframework.context.annotation.ConfigurationClassParser#processConfigurationClass
|- org.springframework.context.annotation.ConfigurationClassParser#doProcessConfigurationClass
|- org.springframework.context.annotation.ComponentScanAnnotationParser#parse
|- org.springframework.context.annotation.ClassPathBeanDefinitionScanner#doScan
```

从配置的 basePackages 开始扫描

```java
// org.springframework.context.annotation.ClassPathBeanDefinitionScanner#doScan
protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
    Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();
    for (String basePackage : basePackages) {
        Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
        for (BeanDefinition candidate : candidates) {
            ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
            candidate.setScope(scopeMetadata.getScopeName());
            String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
            if (candidate instanceof AbstractBeanDefinition abstractBeanDefinition) {
                postProcessBeanDefinition(abstractBeanDefinition, beanName);
            }
            if (candidate instanceof AnnotatedBeanDefinition annotatedBeanDefinition) {
                AnnotationConfigUtils.processCommonDefinitionAnnotations(annotatedBeanDefinition);
            }
            if (checkCandidate(beanName, candidate)) {
                BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
                definitionHolder =
                        AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
                beanDefinitions.add(definitionHolder);
                registerBeanDefinition(definitionHolder, this.registry); // 将扫描到的 BeanDefinition 信息注册到容器中
            }
        }
    }
    return beanDefinitions;
}
```

往下 findCandidateComponents

```shell
org.springframework.context.annotation.ClassPathBeanDefinitionScanner#findCandidateComponents
|- org.springframework.context.annotation.ClassPathScanningCandidateComponentProvider#scanCandidateComponents
```

接下来看 `scanCandidateComponents` 方法

```java
// org.springframework.context.annotation.ClassPathScanningCandidateComponentProvider#scanCandidateComponents
private Set<BeanDefinition> scanCandidateComponents(String basePackage) {
    Set<BeanDefinition> candidates = new LinkedHashSet<>();
    try {
        // 假如 basePackage = com.demo
        // packageSearchPath = classpath*:com/demo/**/*.class，拿到与包名同级的 .class 文件
        String packageSearchPath = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX + resolveBasePackage(basePackage) + '/' + this.resourcePattern;
        // 使用 Spring 封装好的 ResourceLoader 来读取 classpath 的字节码文件
        Resource[] resources = getResourcePatternResolver().getResources(packageSearchPath);
        for (Resource resource : resources) {
            String filename = resource.getFilename();
            if (filename != null && filename.contains(ClassUtils.CGLIB_CLASS_SEPARATOR)) { continue; /* Ignore CGLIB-generated classes in the classpath */ }
            try {
                MetadataReader metadataReader = getMetadataReaderFactory().getMetadataReader(resource);
                if (isCandidateComponent(metadataReader)) { // 读取字节码文件，生成 BeanDefinition
                    ScannedGenericBeanDefinition sbd = new ScannedGenericBeanDefinition(metadataReader);
                    sbd.setSource(resource);
                    if (isCandidateComponent(sbd)) {
                        candidates.add(sbd);
                    }
                }
            }
            catch (FileNotFoundException ex) {}
            catch (ClassFormatException ex) {} // throw BeanDefinitionStoreException
            catch (Throwable ex) {} // throw BeanDefinitionStoreException
        }
    }
    catch (IOException ex) {} // throw BeanDefinitionStoreException
    return candidates;
}
```

至此就完成了 Bean 配置的扫描和 BeanDefinition 的注册。在接下来的流程中就能使用 BeanDefinition 来创建 Bean 了。

<br>

### Bean 创建流程

直接定位

```shell
org.springframework.context.support.AbstractApplicationContext#refresh
|- org.springframework.context.support.AbstractApplicationContext#finishBeanFactoryInitialization
|- org.springframework.beans.factory.support.DefaultListableBeanFactory#preInstantiateSingletons
```

到这里发现源码和 Spring5 有点不一样：

```java
// org.springframework.beans.factory.support.DefaultListableBeanFactory#preInstantiateSingletons
public void preInstantiateSingletons() throws BeansException {
    // Trigger initialization of all non-lazy singleton beans...
    List<CompletableFuture<?>> futures = new ArrayList<>();
    this.preInstantiationThread.set(PreInstantiation.MAIN);
    this.mainThreadPrefix = getThreadNamePrefix();
    try {
        for (String beanName : beanNames) {
            RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
            if (!mbd.isAbstract() && mbd.isSingleton()) {
                CompletableFuture<?> future = preInstantiateSingleton(beanName, mbd);
                if (future != null) {
                    futures.add(future);
                }
            }
        }
    } finally {
        this.mainThreadPrefix = null;
        this.preInstantiationThread.remove();
    }

    if (!futures.isEmpty()) {
        try {
            CompletableFuture.allOf(futures.toArray(new CompletableFuture<?>[0])).join();
        } catch (CompletionException ex) {}
    }
}
```

Spring6 使用 CompletableFuture 进行多线程的 Bean 创建，并且把 `DefaultListableBeanFactory#preInstantiateSingletons` 方法拆成了两个（其他的暂且不说，至少方便了看源码 :P）

```shell
org.springframework.beans.factory.support.DefaultListableBeanFactory#preInstantiateSingletons
|- org.springframework.beans.factory.support.DefaultListableBeanFactory#preInstantiateSingleton
```

```java
// org.springframework.beans.factory.support.DefaultListableBeanFactory#preInstantiateSingleton

@Nullable
private CompletableFuture<?> preInstantiateSingleton(String beanName, RootBeanDefinition mbd) {
    if (mbd.isBackgroundInit()) { // 判断是否有其他线程在进行当前 Bean 创建流程
        // 没有其他线程在创建当前 Bean
        Executor executor = getBootstrapExecutor();
        if (executor != null) {
            String[] dependsOn = mbd.getDependsOn(); // 先创建依赖的 Bean
            if (dependsOn != null) {
                for (String dep : dependsOn) {
                    getBean(dep);
                }
            }
            CompletableFuture<?> future = CompletableFuture.runAsync(
                    () -> instantiateSingletonInBackgroundThread(beanName), executor); // 在后台线程进行当前 Bean 的创建
            // 避免 循环依赖，addSingletonFactory 将 当前 Bean 的 工厂类 缓存到 singletonFactories（三级缓存）
            addSingletonFactory(beanName, () -> {
                try {
                    future.join();
                } catch (CompletionException ex) { ReflectionUtils.rethrowRuntimeException(ex.getCause()); }
                return future;  // not to be exposed, just to lead to ClassCastException in case of mismatch
            });
            return (!mbd.isLazyInit() ? future : null);
        }
    }

    if (!mbd.isLazyInit()) {
        try {
            instantiateSingleton(beanName);
        }
        catch (BeanCurrentlyInCreationException ex) {}
    }
    return null;
}
```

继续往下

```shell
org.springframework.beans.factory.support.DefaultListableBeanFactory#preInstantiateSingleton
|- org.springframework.beans.factory.support.DefaultListableBeanFactory#instantiateSingletonInBackgroundThread
|- org.springframework.beans.factory.support.DefaultListableBeanFactory#instantiateSingleton
|- org.springframework.beans.factory.support.AbstractBeanFactory#getBean(java.lang.String)
|- org.springframework.beans.factory.support.AbstractBeanFactory#doGetBean
    |- org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#getSingleton(java.lang.String) # Return the (raw) singleton object 尝试获取单例，分别从 一二三级缓存 中尝试获取
    |- org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#getSingleton(java.lang.String, org.springframework.beans.factory.ObjectFactory<?>) # 执行 Bean 创建逻辑
    |- org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#createBean(java.lang.String, org.springframework.beans.factory.support.RootBeanDefinition, java.lang.Object[])
    |- org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#doCreateBean # Actually create the specified bean
```

```java
// org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#doCreateBean

protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) throws BeanCreationException {
    // Instantiate the bean.
    BeanWrapper instanceWrapper = null;
    if (mbd.isSingleton()) {
        instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
    }
    if (instanceWrapper == null) {
        // 创建对应的 Bean 实例
        instanceWrapper = createBeanInstance(beanName, mbd, args);
    }
    Object bean = instanceWrapper.getWrappedInstance();
    Class<?> beanType = instanceWrapper.getWrappedClass();
    if (beanType != NullBean.class) {
        mbd.resolvedTargetType = beanType;
    }

    // Allow post-processors to modify the merged bean definition.
    synchronized (mbd.postProcessingLock) {
        if (!mbd.postProcessed) {
            try {
                applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
            }
            catch (Throwable ex) {}
            mbd.markAsPostProcessed();
        }
    }

    // Eagerly cache singletons to be able to resolve circular references
    // even when triggered by lifecycle interfaces like BeanFactoryAware.
    boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences && isSingletonCurrentlyInCreation(beanName));
    if (earlySingletonExposure) {
        addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean)); // 将 当前 Bean 的 工厂类 缓存到 singletonFactories（三级缓存）
    }

    // Initialize the bean instance.
    Object exposedObject = bean;
    populateBean(beanName, mbd, instanceWrapper); // 从 BeanDefinition 获取相应的 Bean 属性，填充 Bean 实例
    exposedObject = initializeBean(beanName, exposedObject, mbd); // 进行 Bean 初始化操作，invokeAwareMethods/applyBeanPostProcessorsBeforeInitialization/applyBeanPostProcessorsAfterInitialization

    if (earlySingletonExposure) {
        Object earlySingletonReference = getSingleton(beanName, false); // 将 Bean 的 工厂类 缓存到 earlySingletonObjects（二级缓存）
        if (earlySingletonReference != null) {
            if (exposedObject == bean) {
                exposedObject = earlySingletonReference;
            }
            else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
                String[] dependentBeans = getDependentBeans(beanName);
                Set<String> actualDependentBeans = CollectionUtils.newLinkedHashSet(dependentBeans.length);
                for (String dependentBean : dependentBeans) {
                    if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
                        actualDependentBeans.add(dependentBean);
                    }
                }
                if (!actualDependentBeans.isEmpty()) {}
            }
        }
    }
    // Register bean as disposable. 
    // 将 bean 注册为 一次性的。只有单例 bean 才需要注册
    registerDisposableBeanIfNecessary(beanName, bean, mbd);
    return exposedObject;
}
```

创建 Bean 并放入 一级缓存

```shell
org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#getSingleton(java.lang.String, org.springframework.beans.factory.ObjectFactory<?>)
|- org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#addSingleton(java.lang.String, java.lang.Object) # 将 Bean 放入 一级缓存 singletonObjects
```

```java
// org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#addSingleton(java.lang.String, java.lang.Object)

protected void addSingleton(String beanName, Object singletonObject) {
    Object oldObject = this.singletonObjects.putIfAbsent(beanName, singletonObject); // 将 Bean 放入 一级缓存 singletonObjects
    if (oldObject != null) {
        throw new IllegalStateException("Could not register object [" + singletonObject +
                "] under bean name '" + beanName + "': there is already object [" + oldObject + "] bound");
    }
    this.singletonFactories.remove(beanName);
    this.earlySingletonObjects.remove(beanName);
    this.registeredSingletons.add(beanName);

    Consumer<Object> callback = this.singletonCallbacks.get(beanName);
    if (callback != null) {
        callback.accept(singletonObject);
    }
}
```