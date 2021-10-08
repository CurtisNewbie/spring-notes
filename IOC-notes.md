# IOC Notes 

关于 Spring 如何实现 IOC 容器, 三个重要的组件：

- ApplicationContext
- BeanFactory
- BeanPostProcessor

# 1. ApplicationContext 与 BeanFactory

首先了解 ApplicationContext 实现的**层级关系**（只包含较常用或关键的类）：

```

                         AbstractApplicationContext
                                     ^
                                     |
                  +------------------+--------------------+
                  |                                       |
       GenericApplicationContext       AbstractRefreshableApplicationContext
                  ^                                       ^
                  |                                       |
                  |                                       |
AnnotationConfigApplicationContext          AbstractXmlApplicationContext
                                                          ^
                                                          |
                                                          |
                                       +------------------+-----------------+
                                       |                                    |
                         FileSystemXmlApplicationContext     ClassPathXmlApplicationContext

```

`AbstractApplicationContext` 采用模板的设计模型，包含关键的 refresh 和初始化代码逻辑，所以关键流程主要看这个类。其中，`refresh()` 是一个非常关键的概念，因为部分 `ApplicationContext` 实现是支持重新加载 context 的，换句话来说就是，清除 `BeanFactory` 和已创建的 bean, 重新创建一个新的 `BeanFactory`，并且重新加载 `BeanDefinition`. 就算我们使用的 `ApplicationContext` 不需要支持刷新已创建的上下文，例如 `GenericApplicationContext`, 我们也会初始化上下文至少一次，初始化的操作，实际上和 `refresh` 方法没有很大区别。

要注意实现 `ApplicationContext` 的子类与实现 `BeanFactory` 的子类之间的关系。即使 `ApplicationContext` 本身继承了 `BeanFactory` 接口，可以简单的认为，`ApplicationContext` 的实现类**包含** `BeanFactory`. 所以 `ApplicationContext` 的实现将部分 `BeanFactory` 接口的请求 delegate 到内部的 `BeanFactory` 中。

包含 `BeanFactory` 的两个 `ApplicationContext` 的实现主要为：

- GenericApplicationContext

    - 只能调用一次 `refresh()` 方法
    - 并且在一开始就拥有创建好的 `DefaultListableBeanFactory`（在构造器中创建好）

- AbstractRefreshableApplicationContext

    - 支持多次 `refresh()` 刷新的实现，每次刷新都会重新创建内部的 `BeanFactory`
    - 子类只需要实现 `loadBeanDefinitions` 方法

这两个类中都包含了 `DefaultListableBeanFactory`，该实现类基本上实现了所有 `BeanFactory` 相关的接口，具体结构如下：

```

      SimpleAliasRegistry
               ^
               |
  DefaultSingletonBeanRegistry
               ^
               |
   FactoryBeanRegistrySupport
               ^
               |
      AbstractBeanFactory
               ^
               |
AbstractAutowireCapableBeanFactory
               ^
               |
   DefaultListableBeanFactory

```

这里也可以看出为什么使用的是 `DefaultListableBeanFactory`, 因为其包含了绝大部分需要的功能。具体每个类的功能如下：

- SimpleAliasRegistry

    - 提供 alias 存储的功能
    - 实现的方式基本为 `Map<String, String> aliasMap ....`

- DefaultSingletonBeanRegistry
    
    - 实现 `SingletonBeanRegistry`
    - Serves as base class for `BeanFactory` implementation, has no idea about `BeanDefinition`
    - 主要用于管理 singleton beans

- FactoryBeanRegistrySupport

    - Add support to use `FactoryBean` which is similar to `ObjectProvider` but with different exception handling.

- AbstractBeanFactory

    - Abstract base class for `BeanFactory`, contains two main methods to be implemented by subclasses:
        - `BeanDefinition getBeanDefinition(String beanName)`
        - `Object createBean(String beanName, RootBeanDefinition mbd, Object[] args)`

- AbstractAutowireCapableBeanFactory

    - Creating beans
    - Populating beans' properties
    - wiring
    - Initializing (e.g., init method) 
    - 包含以下方法由子类实现，用于解析需要被注入的 bean
        - `Object resolveDependency(DependencyDescriptor descriptor, String requestingBeanName, Set<String> autowiredBeanNames, TypeConverter typeConverter)`

- DefaultListableBeanFactory

    - Full-fledge bean factory based on bean definitions metadata, extensible through post processors.

# 2. 为什么 ApplicationContext 需要包含 BeanFactory

可以简单的认为 `BeanFactory` 就是管理 bean 的创建和存储。`ApplicationContext` 比 `BeanFactory` 管理的层级更高更广，当我们的应用启动-的时候, `ApplicationContext` 使用内部的 `BeanFactory` 去拿取一些我们需要的类，例如一些监听器或者 `BeanFactoryPostProcessor`。同时 `ApplicationContext` 也管理了什么时候 `BeanFactory` 进行 singleton bean 的实例化，属性赋值，还有初始化的操作。还句话来说，就是 `ApplicationContext` 管理 `BeanFactory`。

# 3. BeanPostProcessor 和 InstantiationAwareBeanPostProcessor

`BeanPostProcessor` 回调在 bean 初始化前后触发, 初始化本质上就是 `init` 方法还有 `InitializingBean` 的 `afterPropertiesSet` 方法.

```java
public interface BeanPostProcessor {

	default Object postProcessBeforeInitialization(Object bean, String beanName) 
            throws BeansException {
		return bean;
	}
	default Object postProcessAfterInitialization(Object bean, String beanName) 
            throws BeansException {
		return bean;
	}
}
```

`InstantiationAwareBeanPostProcessor` 回调在 bean 创建前后，还有应用 `PropertyValues` 之前触发。

```java
public interface InstantiationAwareBeanPostProcessor extends BeanPostProcessor {

	@Nullable default Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) 
            throws BeansException {
		return null;
	}

	default boolean postProcessAfterInstantiation(Object bean, String beanName) 
            throws BeansException {
		return true;
	}

	@Nullable default PropertyValues postProcessProperties(PropertyValues pvs, Object bean, 
            String beanName) throws BeansException {
		return null;
	}
    // ...
}
```


# 4. AbstractApplicationContext#refresh 方法

注意: 

`ClassPathXmlApplicationContext` 和 `AnnotationConfigApplicationContext` 都会在构造器中调用一次 `refresh()` 方法。该 `refresh()` 方法是定义在 `AbstractApplicationContext` 中的。

```java
// AbstractApplicationContext.refresh()
@Override
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        // 1)
        prepareRefresh();
        // 2)
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
        // 3)
        prepareBeanFactory(beanFactory);

        try {
            // 4)
            postProcessBeanFactory(beanFactory);
            // 5)
            invokeBeanFactoryPostProcessors(beanFactory);
            // 6)
            registerBeanPostProcessors(beanFactory);
            // 7)
            initMessageSource();
            // 8)
            initApplicationEventMulticaster();
            // 9)
            onRefresh();
            // 10)
            registerListeners();
            // 11)
            finishBeanFactoryInitialization(beanFactory);
            // 12)
            finishRefresh();
        }
        // ...
    }
}
```

## 4.1 准备初始化

初始化状态，设置时间等, 本质上没有做什么特别重要的事情

实际代码：

```java
// AbstractApplicationContext.prepareRefresh()
protected void prepareRefresh() {
    this.startupDate = System.currentTimeMillis();
    this.closed.set(false);
    this.active.set(true);
    initPropertySources();
    getEnvironment().validateRequiredProperties();
    if (this.earlyApplicationListeners == null) {
        this.earlyApplicationListeners = new LinkedHashSet<>(this.applicationListeners);
    } else {
        this.applicationListeners.clear();
        this.applicationListeners.addAll(this.earlyApplicationListeners);
    }
    this.earlyApplicationEvents = new LinkedHashSet<>();
}
```

## 4.2 刷新和获取新的 BeanFactory 

该方法主要是从子类拿新的 `ConfigurableListableBeanFactory`. 该方法内部包含两个核心方法: 

- `protected abstract void refreshBeanFactory() throws BeansException, IllegalStateException;`
- `public abstract ConfigurableListableBeanFactory getBeanFactory() throws IllegalStateException;`

`AbstractApplicationContext.refreshBeanFactory()` 方法是用于通知子类刷新内部的 `BeanFactory`, 刷新本质上就是清除旧的 `BeanFactory` 和内部保存的 bean, 重新创建一个。而 `AbstractApplicationContext.getBeanFactory()` 就是从子类获取 `BeanFactory`。

实际代码：

```java
// AbstractApplicationContext.obtainFreshBeanFactory()
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
    refreshBeanFactory();
    return getBeanFactory();
}
```

两个 `AbstractApplicationContext` 的子类实现了 `refreshBeanFactory()` 方法，它们分别是：

- `AbstractRefreshableApplicationContext`
- `GenericApplicationContext`

### 4.2.1 AbstractRefreshableApplicationContext

对于 `AbstractRefreshableApplicationContext` 来说，该实现的核心就是，清除 `BeanFactory` 内的 bean，关闭 `BeanFactory`, 创建一个新的 `BeanFactory` 并且通过 `loadBeanDefinitions(BeanFactory)` 方法扫描加载 `BeanDefinition`，这个方法才是真正的-核心。注意 `loadBeanDefinitions` 方法是 `AbstractRefreshableApplicationContext` 的 abstract 方法，基于这个方法，子类可-自定义加载 `BeanDefinition` 的方式，例如，读取 xml 或 扫描 annotation。

```java
// AbstractRefreshableApplicationContext.refreshBeanFactory()
protected final void refreshBeanFactory() throws BeansException {
    if (hasBeanFactory()) {
        destroyBeans();
        closeBeanFactory();
    }
    try {
        DefaultListableBeanFactory beanFactory = createBeanFactory();
        beanFactory.setSerializationId(getId());
        customizeBeanFactory(beanFactory);
        loadBeanDefinitions(beanFactory);
        this.beanFactory = beanFactory;
    } catch (IOException ex) {
        throw new ApplicationContextException("I/O error parsing bean definition source for " + 
            getDisplayName(), ex);
    }
}
```

### 4.2.2 GenericApplicationContext

该实现本质上什么都没做，像基于 annotation 的 `AnnotationConfigApplicationContext` (继承了 `GenericApplicationContext`) 不需要支持 refresh, 因为标签都是写在代码里的。一般需要 refresh 的都是基于文件的，例如 xml，即使不需要支持 refresh, 像 `AnnotationConfigApplicationContext` 也需要在创建的时候扫描路径，读取和加载 `BeanDefinition`，本质都一样。

```java
// GenericApplicationContext.refreshBeanFactory()
protected final void refreshBeanFactory() throws IllegalStateException {
    if (!this.refreshed.compareAndSet(false, true)) {
        throw new IllegalStateException(
                "GenericApplicationContext does not support multiple refresh attempts:" + 
                " just call 'refresh' once");
    }
    this.beanFactory.setSerializationId(getId());
}
```

例如，在 `AnnotationConfigApplicationContext` 其中一个构造器中，有显式的 `scan` 方法和 `refresh` 方法调用，但是再次的调用 `refresh` 方法会报错，具体如上面展示的代码，因为 `GenericApplicationContext` 只支持 `refresh` 一次。

```java
public AnnotationConfigApplicationContext(String... basePackages) {
    this();
    scan(basePackages);
    refresh();
}
```

可以看出 `AnnotationConfigApplicationContext` 类中，对基于 annotation 的 `BeanDefinition` 的扫描和加载是在 `scan` 方法中。`AnnotationConfigApplicationContext` 类中包含了 `ClassPathBeanDefinitionScanner`, 这个 scanner 专门负责对 annotation 像 `@Component` 的扫描。具体看 `scan` 方法：

```java
// AnnotationConfigApplicationContext.scan(...)
public void scan(String... basePackages) {
    // 1)
    this.scanner.scan(basePackages);
}

// ClassPathBeanDefinitionScanner.scan(...)
public int scan(String... basePackages) {
    int beanCountAtScanStart = this.registry.getBeanDefinitionCount();
    // 2)
    doScan(basePackages);

    // Register annotation config processors, if necessary.
    if (this.includeAnnotationConfig) {
        AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
    }
    return (this.registry.getBeanDefinitionCount() - beanCountAtScanStart);
}

// ClassPathBeanDefinitionScanner.doScan(...)
protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
    Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();
    for (String basePackage : basePackages) {
        // 3) 
        Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
        for (BeanDefinition candidate : candidates) {
            ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
            candidate.setScope(scopeMetadata.getScopeName());
            String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
            if (candidate instanceof AbstractBeanDefinition) {
                postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
            }
            if (candidate instanceof AnnotatedBeanDefinition) {
                AnnotationConfigUtils.processCommonDefinitionAnnotations(
                    (AnnotatedBeanDefinition) candidate
                );
            }
            if (checkCandidate(beanName, candidate)) {
                BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
                definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(
                                        scopeMetadata, 
                                        definitionHolder, 
                                        this.registry);
                beanDefinitions.add(definitionHolder);
                registerBeanDefinition(definitionHolder, this.registry);
            }
        }
    }
    return beanDefinitions;
}
```

可以看出, `scan` 方法的核心就是 `ClassPathBeanDefinitionScanner.scan(...)` 方法, 而这个方法内部的核心在于对 `BeanDefinition` 的查询, 也就是 `Set<BeanDefinition> candidates = findCandidateComponents(basePackage)`, 注意 `ClassPathBeanDefinitionScanner` 继承了类 `ClassPathScanningCandidateComponentProvider`, 并且方法 `findCandidateComponents` 由该父类实现, 具体如下：

```java
// ClassPathScanningCandidateComponentProvider.findCandidateComponents(...)
public Set<BeanDefinition> findCandidateComponents(String basePackage) {
    if (this.componentsIndex != null && indexSupportsIncludeFilters()) {
        return addCandidateComponentsFromIndex(this.componentsIndex, basePackage);
    } else {
        // 4)
        return scanCandidateComponents(basePackage);
    }
}

// ClassPathScanningCandidateComponentProvider.scanCandidateComponents(...)
private Set<BeanDefinition> scanCandidateComponents(String basePackage) {
    Set<BeanDefinition> candidates = new LinkedHashSet<>();
    try {
        String packageSearchPath = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX +
                resolveBasePackage(basePackage) + '/' + this.resourcePattern;
        Resource[] resources = getResourcePatternResolver().getResources(packageSearchPath);
        for (Resource resource : resources) {
            if (resource.isReadable()) {
                try {
                    MetadataReader metadataReader = getMetadataReaderFactory()
                        .getMetadataReader(resource);

                    // 5) 
                    if (isCandidateComponent(metadataReader)) {
                        ScannedGenericBeanDefinition sbd = 
                            new ScannedGenericBeanDefinition(metadataReader);
                        sbd.setSource(resource);
                        if (isCandidateComponent(sbd)) {
                            candidates.add(sbd);
                        }
                    }
                } catch (Throwable ex) {
                    throw new BeanDefinitionStoreException(
                            "Failed to read candidate component class: " + resource, ex);
                }
            }
        }
    }
    catch (IOException ex) {
        throw new BeanDefinitionStoreException("I/O failure during classpath scanning", ex);
    }
    return candidates;
}
```

这个方法的重点也就是通过 `isCandidateComponent` 方法判断一个 component 是不是 candidate，具体代码如下。使用的是 `includeFilters` 和 `excludeFilters`。

```java
// ClassPathScanningCandidateComponentProvider.isCandidateComponent(...)
protected boolean isCandidateComponent(MetadataReader metadataReader) throws IOException {
    // 6)
    for (TypeFilter tf : this.excludeFilters) {
        if (tf.match(metadataReader, getMetadataReaderFactory())) {
            return false;
        }
    }
    for (TypeFilter tf : this.includeFilters) {
        if (tf.match(metadataReader, getMetadataReaderFactory())) {
            return isConditionMatch(metadataReader);
        }
    }
    return false;
}
```

对于这个 `includeFilters` 的加载也非常的简单直接，我们完全可以增加自己写的 annotation 用于标记需要注入的 `BeanDefinition`。加载默认 annotation 的具体代码如下：

```java
// ClassPathScanningCandidateComponentProvider.registerDefaultFilters(...)
protected void registerDefaultFilters() {
    this.includeFilters.add(new AnnotationTypeFilter(Component.class));
    ClassLoader cl = ClassPathScanningCandidateComponentProvider.class.getClassLoader();
    try {
        this.includeFilters.add(new AnnotationTypeFilter(
                ((Class<? extends Annotation>) ClassUtils.forName("javax.annotation.ManagedBean", cl)), 
                    false));
    } catch (ClassNotFoundException ex) { }
    try {
        this.includeFilters.add(new AnnotationTypeFilter(
                ((Class<? extends Annotation>) ClassUtils.forName("javax.inject.Named", cl)), 
                    false));
    } catch (ClassNotFoundException ex) { }
}
```

### 4.2.3 BeanDefinition

在 Spring 中，`BeanDefinition` 本质上就是一个 bean 的代表。在我们创建一个新的 `BeanFactory` 的时候，我们在这一步骤也尝试加载所有 `BeanDefinition` 即使我们没有真正的创建实例。以下是部分 `BeanDefinition` 的 API 方法：

```java
public interface BeanDefinition extends AttributeAccessor, BeanMetadataElement {

	void setParentName(@Nullable String parentName);
	String getParentName();

	void setBeanClassName(@Nullable String beanClassName);
	String getBeanClassName();

	void setScope(@Nullable String scope);
	String getScope();

	void setLazyInit(boolean lazyInit);
	boolean isLazyInit();

	void setDependsOn(@Nullable String... dependsOn);
	String[] getDependsOn();

	void setAutowireCandidate(boolean autowireCandidate);
	boolean isAutowireCandidate();

	void setPrimary(boolean primary);
	boolean isPrimary();

	void setFactoryBeanName(@Nullable String factoryBeanName);
	String getFactoryBeanName();

	void setFactoryMethodName(@Nullable String factoryMethodName);
	String getFactoryMethodName();

	ConstructorArgumentValues getConstructorArgumentValues();

	MutablePropertyValues getPropertyValues();

	void setInitMethodName(@Nullable String initMethodName);
	String getInitMethodName();

	void setDestroyMethodName(@Nullable String destroyMethodName);
	String getDestroyMethodName();

	ResolvableType getResolvableType();

	boolean isSingleton();

	boolean isPrototype();

	boolean isAbstract();

    // ....
}
```

上面已经有提到基于 annotation 的 `BeanDefinition` 如何加载。本质上来说像 `AbstractXmlApplicationContext` 对 `BeanDefinition` 的加载也非常直接，就是读取 xml 文件，解析里面的元素，然后组装到 `BeanDefinition` 里面而已。除了 `BeanDefinition` 的加载, 这些 `BeanDefinition` 也需要注册到 `BeanFactory` 中，注册的方式也都是一致的，使用的是 

```java
BeanDefinitionRegistry.registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
``` 

最关键的是，`DefaultListableBeanFactory` 也是一个 `BeanDefinitionRegistry`, 所以我们也是直接注册到我们的 `BeanFactory` 中。总的来说，`obtainFreshBeanFactory()` 方法执行后，新的 `BeanFactory` 已经创建了，而且 `BeanDefinition` 也都已经加载并且注册-完毕了。

## 4.3 注册 Bean 和 BeanPostProcessor 到 BeanFactory 中

该方法属于 `AbstractApplicationContext`，用于配置 `BeanPostProcessor`, 如: `ApplicationContextAwareProcessor`，和注册部分 bean 到 beanFactory 中, 让开发人员能够进行注入，例如, `this` 作为 `ApplicationContext`；`this` 作为 `ApplicationEventPublisher` 等。

实际代码：

```java
// AbstractApplicationContext.prepareBeanFactory(...) 
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    beanFactory.setBeanClassLoader(getClassLoader());
    if (!shouldIgnoreSpel) {
        beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(
            beanFactory.getBeanClassLoader())
        );
    }
    beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

    // Configure the bean factory with context callbacks.
    beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
    beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
    beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
    beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
    beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationStartupAware.class);

    // BeanFactory interface not registered as resolvable type in a plain factory.
    // MessageSource registered (and found for autowiring) as a bean.
    beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
    beanFactory.registerResolvableDependency(ResourceLoader.class, this);
    beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
    beanFactory.registerResolvableDependency(ApplicationContext.class, this);

    // Register early post-processor for detecting inner beans as ApplicationListeners.
    beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

    // Detect a LoadTimeWeaver and prepare for weaving, if found.
    if (!NativeDetector.inNativeImage() && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
        beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
        beanFactory.setTempClassLoader(
            new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader())
        );
    }
    // Register default environment beans.
    if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
        beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
    }
    if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
        beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, 
            getEnvironment().getSystemProperties()
        );
    }
    if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
        beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, 
            getEnvironment().getSystemEnvironment()
        );
    }
    if (!beanFactory.containsLocalBean(APPLICATION_STARTUP_BEAN_NAME)) {
        beanFactory.registerSingleton(APPLICATION_STARTUP_BEAN_NAME, 
            getApplicationStartup()
        );
    }
}
```

## 4.4 给子类提供机会，对 BeanFactory 进行 Post-Processing

默认情况下，`AbstractApplicationContext` 在该方法中什么都没有做，子类可以 override 这个方法, 然后对 `BeanFactory` 做一些配置，修改，或者注册 `BeanPostProcessor`.

实际代码：

```java
// AbstractApplicationContext.postProcessBeanFactory(...) 
protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
}
```

在 `GenericApplicationContext` 中，该方法被覆写，具体也是注册一些 bean 和  `BeanPostProcessor`.

```java
// GenericApplicationContext.postProcessBeanFactory(...) 
@Override
protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    if (this.servletContext != null) {
        beanFactory.addBeanPostProcessor(new ServletContextAwareProcessor(this.servletContext));
        beanFactory.ignoreDependencyInterface(ServletContextAware.class);
    }
    WebApplicationContextUtils.registerWebApplicationScopes(beanFactory, this.servletContext);
    WebApplicationContextUtils.registerEnvironmentBeans(beanFactory, this.servletContext);
}
```

## 4.5 调用 beanFactory 内部注册的 BeanFactoryPostProcessor

这个方法内核心的代码基本就是拿 `BeanFactory` 内部的 `BeanFactoryPostProcessor`, 然后对该 `BeanFactory` 进行处理。具体如下：

```java
// AbstractApplicationContext.invokeBeanFactoryPostProcessors(...) 
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
    // 1)
    PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, 
            getBeanFactoryPostProcessors());

    // ...
}

// 2)
for (BeanFactoryPostProcessor postProcessor : postProcessors) {
    // ...
    postProcessor.postProcessBeanFactory(beanFactory);
    postProcessBeanFactory.end();
}
```

## 4.6 注册 BeanPostProcessors 用于创建新 Bean 的时候使用

该方法主要负责将 `BeanPostProcessor` 类型的 bean 创建，并且按顺序注册到 `BeanFactory` 中。具体实现方法如下: 

1. 从 `BeanFactory` 里拿 `BeanPostProcessor` 的 beanName，数据格式为 String[]
2. 逐个 beanName 拿 `BeanPostProcessor` 实例，如果实例不存在就创建实例，并且将他们注册到 `BeanFactory` 中

```java
// AbstractApplicationContext.registerBeanPostProcessors(...) 
protected void registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory) {
    // 1)
    PostProcessorRegistrationDelegate.registerBeanPostProcessors(beanFactory, this);
}

// 2)
// ...
String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);
// ...
for (String ppName : postProcessorNames) {
    if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
        BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
        priorityOrderedPostProcessors.add(pp);
        if (pp instanceof MergedBeanDefinitionPostProcessor) {
            internalPostProcessors.add(pp);
        }
    } else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
        orderedPostProcessorNames.add(ppName);
    } else {
        nonOrderedPostProcessorNames.add(ppName);
    }
}
// ...
for (BeanPostProcessor postProcessor : postProcessors) {
    beanFactory.addBeanPostProcessor(postProcessor);
}
```

## 4.7 初始化国际化信息

```java
// AbstractApplicationContext.initMessageSource(...) 
initMessageSource();
```

## 4.8 初始化 ApplicationEventMulticaster

本质上就是在我们需要发送特定事件之前，先把 `ApplicationEventMulticaster` 创建，具体代码如下：

```java
// AbstractApplicationContext.initApplicationEventMulticaster(...) 
protected void initApplicationEventMulticaster() {
    ConfigurableListableBeanFactory beanFactory = getBeanFactory();
    if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
        this.applicationEventMulticaster =
                beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, 
                    ApplicationEventMulticaster.class);
        // ...
    }
    else {
        this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
        beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, 
                this.applicationEventMulticaster);

        // ...
    }
}
```

## 4.9 调用回调，通知子类上下文刷新

```java
// AbstractApplicationContext.onRefresh(...) 
onRefresh();
```

该方法由 `AbstractApplicationContext` 实现，默认什么都不做。

## 4.10 注册监听器

该方法本质上就是找到所有的 `ApplicationListener`, 并把这些监听器注册到 `ApplicationEventMulticaster` 中，部分的 `ApplicationListener` 是通过 `BeanFactory` 拿取的，具体代码如下：

```java
// AbstractApplicationContext.registerListeners()
protected void registerListeners() {
    for (ApplicationListener<?> listener : getApplicationListeners()) {
        getApplicationEventMulticaster().addApplicationListener(listener);
    }
    // Do not initialize FactoryBeans here: We need to leave all regular beans
    // uninitialized to let post-processors apply to them!
    String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
    for (String listenerBeanName : listenerBeanNames) {
        getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
    }
    // Publish early application events now that we finally have a multicaster...
    Set<ApplicationEvent> earlyEventsToProcess = this.earlyApplicationEvents;
    this.earlyApplicationEvents = null;
    if (!CollectionUtils.isEmpty(earlyEventsToProcess)) {
        for (ApplicationEvent earlyEvent : earlyEventsToProcess) {
            getApplicationEventMulticaster().multicastEvent(earlyEvent);
        }
    }
}
```

## 4.11 创建所有非 lazy-init 的 Singleton Beans

该方法内部调用了一个关键方法 `BeanFactory.preInstantiateSingletons()`, 该方法用于创建所有非 lazy-init 和 abstract 的 Singleton beans。注意的是，我们使用的 `BeanFactory` 是 `DefaultListableBeanFactory`， 该方法也是由 `DefaultListableBeanFactory` 实现。

```java
// AbstractApplicationContext.finishBeanFactoryInitialization(...)
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
    // Initialize conversion service for this context.
    if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
            beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
        beanFactory.setConversionService(
                beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
    }
    if (!beanFactory.hasEmbeddedValueResolver()) {
        beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
    }
    // Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
    String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
    for (String weaverAwareName : weaverAwareNames) {
        getBean(weaverAwareName);
    }
    // Stop using the temporary ClassLoader for type matching.
    beanFactory.setTempClassLoader(null);
    // Allow for caching all bean definition metadata, not expecting further changes.
    beanFactory.freezeConfiguration();

    // 0)
    // Instantiate all remaining (non-lazy-init) singletons.
    beanFactory.preInstantiateSingletons();
}

// DefaultListableBeanFactory.preInstantiateSingletons()
public void preInstantiateSingletons() throws BeansException {
    // 1)
    List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);
    // 2)
    for (String beanName : beanNames) {
        RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
        // 3)
        if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
            // 4)
            if (isFactoryBean(beanName)) {
                Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
                if (bean instanceof FactoryBean) {
                    FactoryBean<?> factory = (FactoryBean<?>) bean;
                    boolean isEagerInit;
                    if (System.getSecurityManager() != null 
                                    && factory instanceof SmartFactoryBean) {
                        isEagerInit = AccessController.doPrivileged(
                                (PrivilegedAction<Boolean>) ((SmartFactoryBean<?>) factory)::isEagerInit,
                                getAccessControlContext());
                    } else {
                        isEagerInit = (factory instanceof SmartFactoryBean &&
                                ((SmartFactoryBean<?>) factory).isEagerInit());
                    }
                    if (isEagerInit) {
                        getBean(beanName);
                    }
                }
            } else {
                // 5)
                getBean(beanName);
            }
        }
    }
    // ...
}
```

`DefaultListableBeanFactory.preInstantiateSingletons()` 方法逻辑：

1. 拿出所有注册的 `beanDefinitionNames` 列表
2. 迭代 `beanDefinitionsNames` 中的 `beanName`
3. 判断，如果 bean 是非 abstract 的，是 Singleton 而且不是 lazy-init 的继续, 否则跳过
4. 如果是 `FactoryBean`, 使用 `"&" + beanName` 组合的名字去取 bean, 然后判断是否是 eager init 的 bean, 如果是，调用方法 `BeanFactory.getBean(String name)`
5. 如果是普通 bean, 直接调用 `BeanFactory.getBean(String name)`

这部分代码的重点仍然是 Singleton bean 的创建，只不过多了 `FactoryBean` 的特例，本质是一样的。重点关注 `getBean(String beanName)` 方法。

### 4.11.1 BeanFactory.getBean(String name) 方法

接下来重点看如何创建 bean 的。`BeanFactory.getBean(String name)` 方法由 `AbstractBeanFactory` 实现，其中内部调用了 `AbstractBeanFactory.doGetBean(...)` 方法。具体代码如下：

```java
protected <T> T doGetBean(
        String name, @Nullable Class<T> requiredType, @Nullable Object[] args, boolean typeCheckOnly)
        throws BeansException {

    // 1) 
    String beanName = transformedBeanName(name);
    Object beanInstance;

    // 2) 
    Object sharedInstance = getSingleton(beanName);
    if (sharedInstance != null && args == null) {
        // 3)
        beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, null);
    } else {
        if (isPrototypeCurrentlyInCreation(beanName)) {
            throw new BeanCurrentlyInCreationException(beanName);
        }

        // 4) 
        BeanFactory parentBeanFactory = getParentBeanFactory();
        if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
            // Not found -> check parent.
            String nameToLookup = originalBeanName(name);
            if (parentBeanFactory instanceof AbstractBeanFactory) {
                return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
                        nameToLookup, requiredType, args, typeCheckOnly);
            } else if (args != null) {
                return (T) parentBeanFactory.getBean(nameToLookup, args);
            } else if (requiredType != null) {
                return parentBeanFactory.getBean(nameToLookup, requiredType);
            } else {
                return (T) parentBeanFactory.getBean(nameToLookup);
            }
        }

        if (!typeCheckOnly) {
            markBeanAsCreated(beanName);
        }

        try {
            if (requiredType != null) {
                beanCreation.tag("beanType", requiredType::toString);
            }
            RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
            checkMergedBeanDefinition(mbd, beanName, args);

            // 5)
            String[] dependsOn = mbd.getDependsOn();
            if (dependsOn != null) {
                for (String dep : dependsOn) {
                    if (isDependent(beanName, dep)) {
                        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                "Circular depends-on relationship between '" + beanName + 
                                "' and '" + dep + "'");
                    }
                    registerDependentBean(dep, beanName);
                    try {
                        getBean(dep);
                    } catch (NoSuchBeanDefinitionException ex) {
                        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                "'" + beanName + "' depends on missing bean '" + dep + "'", ex);
                    }
                }
            }

            // 6) 
            // Create bean instance.
            if (mbd.isSingleton()) {
                sharedInstance = getSingleton(beanName, () -> {
                    try {
                        // 7) 
                        return createBean(beanName, mbd, args);
                    } catch (BeansException ex) {
                        destroySingleton(beanName);
                        throw ex;
                    }
                });
                beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
            } else if (mbd.isPrototype()) {
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
            } else {
                String scopeName = mbd.getScope();
                if (!StringUtils.hasLength(scopeName)) {
                    throw new IllegalStateException("No scope name defined for bean ´" + 
                        beanName + "'");
                }

                // 8)
                Scope scope = this.scopes.get(scopeName);
                if (scope == null) {
                    throw new IllegalStateException("No Scope registered for scope name '" + 
                        scopeName + "'");
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
                } catch (IllegalStateException ex) {
                    throw new ScopeNotActiveException(beanName, scopeName, ex);
                }
            }
        } catch (BeansException ex) {
            cleanupAfterBeanCreationFailure(beanName);
            throw ex;
        } finally {
            beanCreation.end();
        }
    }
    // 9)
    return adaptBeanInstance(name, beanInstance, requiredType);
}
```

核心的代码逻辑有8个步骤：

1. 获取真正的 beanName

```java
protected String transformedBeanName(String name) {
    return canonicalName(BeanFactoryUtils.transformedBeanName(name));
}

public String canonicalName(String name) {
    String canonicalName = name;
    String resolvedName;
    do {
        resolvedName = this.aliasMap.get(canonicalName);
        if (resolvedName != null) {
            canonicalName = resolvedName;
        }
    }
    while (resolvedName != null);
    return canonicalName;
}
```

2. 查看 singleton 是否已被创建, 如果已被创建，则尝试直接返回创建好的 bean (该 bean 可能是个 FactoryBean)

```java
public Object getSingleton(String beanName) {
    return getSingleton(beanName, true);
}

protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    Object singletonObject = this.singletonObjects.get(beanName);
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        singletonObject = this.earlySingletonObjects.get(beanName);
        if (singletonObject == null && allowEarlyReference) {
            synchronized (this.singletonObjects) {
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

3. 检查 bean 是否为 FactoryBean, 如果是，先看缓存是否已经有使用该 `FactoryBean` 创建过的 bean, 没有的话，先 cast 成 `FactoryBean`，然后使用 `FactoryBean.getObject()` 方法拿 bean，同时该 bean 也会保存到缓存中。

```java
protected Object getObjectForBeanInstance(
        Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd) {

    if (BeanFactoryUtils.isFactoryDereference(name)) {
        if (beanInstance instanceof NullBean) {
            return beanInstance;
        }
        if (!(beanInstance instanceof FactoryBean)) {
            throw new BeanIsNotAFactoryException(beanName, beanInstance.getClass());
        }
        if (mbd != null) {
            mbd.isFactoryBean = true;
        }
        return beanInstance;
    }

    if (!(beanInstance instanceof FactoryBean)) {
        return beanInstance;
    }

    Object object = null;
    if (mbd != null) {
        mbd.isFactoryBean = true;
    } else {
        object = getCachedObjectForFactoryBean(beanName);
    }
    if (object == null) {
        FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
        // Caches object obtained from FactoryBean if it is a singleton.
        if (mbd == null && containsBeanDefinition(beanName)) {
            mbd = getMergedLocalBeanDefinition(beanName);
        }
        boolean synthetic = (mbd != null && mbd.isSynthetic());
        object = getObjectFromFactoryBean(factory, beanName, !synthetic);
    }
    return object;
}
```

4. 检查我们是否有这个 BeanDefinition, 如果没有，我们查看父级的 BeanFactory，看父级是否拥有, 如果父级没有这个 bean 但有这个 `BeanDefinition`，那么这个 bean 也是由父级创建，逻辑一样。

```java
BeanFactory parentBeanFactory = getParentBeanFactory();
if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
    String nameToLookup = originalBeanName(name);
    if (parentBeanFactory instanceof AbstractBeanFactory) {
        return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
                nameToLookup, requiredType, args, typeCheckOnly);
    } else if (args != null) {
        return (T) parentBeanFactory.getBean(nameToLookup, args);
    } else if (requiredType != null) {
        return parentBeanFactory.getBean(nameToLookup, requiredType);
    } else {
        return (T) parentBeanFactory.getBean(nameToLookup);
    }
}
```

5. 优先按 dependsOn 规定顺序创建 bean, 这里 `registerDependentBean(...)` 方法也就是记录依赖信息到 `Map` 中。

```java
RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
checkMergedBeanDefinition(mbd, beanName, args);

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
        } catch (NoSuchBeanDefinitionException ex) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                    "'" + beanName + "' depends on missing bean '" + dep + "'", ex);
        }
    }
}

public void registerDependentBean(String beanName, String dependentBeanName) {
    String canonicalName = canonicalName(beanName);
    synchronized (this.dependentBeanMap) {
        Set<String> dependentBeans =
                this.dependentBeanMap.computeIfAbsent(canonicalName, 
                    k -> new LinkedHashSet<>(8));
        if (!dependentBeans.add(dependentBeanName)) {
            return;
        }
    }
    synchronized (this.dependenciesForBeanMap) {
        Set<String> dependenciesForBean =
                this.dependenciesForBeanMap.computeIfAbsent(dependentBeanName, 
                    k -> new LinkedHashSet<>(8));
        dependenciesForBean.add(canonicalName);
    }
}
```

6. `dependsOn` bean 都已经创建，根据类型创建当前 bean 的实例

```java
// Create bean instance.
if (mbd.isSingleton()) {
    sharedInstance = getSingleton(beanName, () -> {
        try {
            return createBean(beanName, mbd, args);
        } catch (BeansException ex) {
            destroySingleton(beanName);
            throw ex;
        }
    });
    beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
}
```

7. 创建 bean 实例 (属性未被完全设置), 关键在于方法 `AbstractBeanFactory.createBean(String beanName, RootBeanDefinition mbd, Object[] args)`. 该方法由 `AbstractAutowireCapableBeanFactory` 实现，下面会讲到。

8. 如果该 bean 既不是 singleton 也不是 prototype, 从 `Map<String, Scope> scope` 里根据 scope 的名字拿对应的 `Scope` 对象，`Scope` 本质上就是一个 `Map`，实现的方式可以是使用 `ThreadLocal`. 注意, 这里的 `Scope.get(String name, ObjectFactory<?> objectFactory)` 方法中的 `ObjectFactory` 是只有当该 bean 不存在时才使用的。

```java
Scope scope = this.scopes.get(scopeName);
if (scope == null) {
    throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
}
try {
    Object scopedInstance = scope.get(beanName, () -> {
        beforePrototypeCreation(beanName);
        try {
            return createBean(beanName, mbd, args);
        } finally {
            afterPrototypeCreation(beanName);
        }
    });
    beanInstance = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
} catch (IllegalStateException ex) {
    throw new ScopeNotActiveException(beanName, scopeName, ex);
}
```

9. 检查创建对象的类型是否正确，看是否需要进行转换

```java
<T> T adaptBeanInstance(String name, Object bean, @Nullable Class<?> requiredType) {
    // Check if required type matches the type of the actual bean instance.
    if (requiredType != null && !requiredType.isInstance(bean)) {
        try {
            Object convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
            if (convertedBean == null) {
                throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
            }
            return (T) convertedBean;
        } catch (TypeMismatchException ex) {
            throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
        }
    }
    return (T) bean;
}
```

### 4.11.2 AbstractAutowireCapableBeanFactory.createBean(...) 方法  

该方法是 `AbstractBeanFactory` 的一个 abstract 方法，并且由 `AbstractAutowireCapableBeanFactory` 实现。

```java
protected abstract Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException;
```

具体实现代码如下：

```java
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
        throws BeanCreationException {

    RootBeanDefinition mbdToUse = mbd;

    // 1)
    Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
    if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
        // 2)
        mbdToUse = new RootBeanDefinition(mbd);
        mbdToUse.setBeanClass(resolvedClass);
    }

    // Prepare method overrides.
    try {
        mbdToUse.prepareMethodOverrides();
    } catch (BeanDefinitionValidationException ex) {
        throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
                beanName, "Validation of method overrides failed", ex);
    }
    try {
        // 3)
        // Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
        Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
        if (bean != null) {
            return bean;
        }
    } catch (Throwable ex) {
        throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
                "BeanPostProcessor before instantiation of bean failed", ex);
    }
    try {
        // 4)
        Object beanInstance = doCreateBean(beanName, mbdToUse, args);
        return beanInstance;
    } catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
        throw ex;
    } catch (Throwable ex) {
        throw new BeanCreationException(
                mbdToUse.getResourceDescription(), 
                beanName, "Unexpected exception during bean creation", ex);
    }
}
```

该方法包含4个步骤：

1. 通过 beanName 找到 Class<?>
2. 使用 copy constructor 复制传入 `BeanDefinition` 
3. 在创建 bean 之前，调用所有 `InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation(Class<?> beanClass, String beanName)` 方法. 该方法给予 `InstantiationAwareBeanPostProcessor` 一个机会去返回一个需要被创建的 bean，如果返回的对象不为 null, 则这个对象就作为应该被创建的实例，这也代表我们后面不会尝试创建这个 bean 了。对于这种情况，我们直接过 `BeanPostProcessor.postProcessAfterInitialization(Object bean, String beanName)` 方法，该回调用于 bean 创建以后，而且已经被初始化 (如, init 方法已被调用)。该方法也提供了一个创建 wrapper 的机会。

```java
protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
    Object bean = null;
    if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
        if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
            Class<?> targetType = determineTargetType(beanName, mbd);
            if (targetType != null) {
                bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
                if (bean != null) {
                    bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
                }
            }
        }
        mbd.beforeInstantiationResolved = (bean != null);
    }
    return bean;
}

protected Object applyBeanPostProcessorsBeforeInstantiation(Class<?> beanClass, String beanName) {
    for (InstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().instantiationAware) {
        Object result = bp.postProcessBeforeInstantiation(beanClass, beanName);
        if (result != null) {
            return result;
        }
    }
    return null;
}

public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
        throws BeansException {

    Object result = existingBean;
    for (BeanPostProcessor processor : getBeanPostProcessors()) {
        Object current = processor.postProcessAfterInitialization(result, beanName);
        if (current == null) {
            return result;
        }
        result = current;
    }
    return result;
}
```

4. 如果没有返回 bean 实例，那么 bean 的创建由 `AbstractAutowireCapableBeanFactory.doCreateBean(...)` 方法创建, 这个方法在下面会解释。

### 4.11.3 AbstractAutowireCapableBeanFactory.doCreateBean(...) 方法

该方法由 `AbstractAutowireCapableBeanFactory` 定义和实现。

```java
protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
        throws BeanCreationException {

    // Instantiate the bean.
    BeanWrapper instanceWrapper = null;
    if (mbd.isSingleton()) {
        instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
    }
    if (instanceWrapper == null) {
        // 1)
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
            } catch (Throwable ex) {
                throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                        "Post-processing of merged bean definition failed", ex);
            }
            mbd.postProcessed = true;
        }
    }
    // Eagerly cache singletons to be able to resolve circular references
    // even when triggered by lifecycle interfaces like BeanFactoryAware.
    boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
            isSingletonCurrentlyInCreation(beanName));
    if (earlySingletonExposure) {
        addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
    }

    // Initialize the bean instance.
    Object exposedObject = bean;
    try {
        // 2)
        populateBean(beanName, mbd, instanceWrapper);

        // 3)
        exposedObject = initializeBean(beanName, exposedObject, mbd);
    }
    catch (Throwable ex) {
        if (ex instanceof BeanCreationException 
                && beanName.equals(((BeanCreationException) ex).getBeanName())) {
            throw (BeanCreationException) ex;
        } else {
            throw new BeanCreationException(
                mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
        }
    }

    if (earlySingletonExposure) {
        Object earlySingletonReference = getSingleton(beanName, false);
        if (earlySingletonReference != null) {
            if (exposedObject == bean) {
                exposedObject = earlySingletonReference;
            } else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
                String[] dependentBeans = getDependentBeans(beanName);
                Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
                for (String dependentBean : dependentBeans) {
                    if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
                        actualDependentBeans.add(dependentBean);
                    }
                }
                if (!actualDependentBeans.isEmpty()) {
                    throw new BeanCurrentlyInCreationException(beanName, ....
                    // ...
                }
            }
        }
    }
    // Register bean as disposable.
    try {
        registerDisposableBeanIfNecessary(beanName, bean, mbd);
    } catch (BeanDefinitionValidationException ex) {
        throw new BeanCreationException(
                mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
    }
    return exposedObject;
}
```

该方法包含最关键的四个方法：

1. `createBeanInstance(...)` 创建 bean 实例, 这里就是单纯的对象实例化，归根到底就是两种场景
    - 有参注入构造的方式
    - 无参直接反射构造的方式
2. `populateBean(...)` 属性设值，这里是依赖注入的时机，所有 non-lazy singleton bean 都已经实例化, 下面会讲到
3. `initialize(...)` 初始化方法，如，`init` 方法调用，相关回调触发等，下面会讲到

### 4.11.4 AbstractAutowireCapableBeanFactory.populateBean(...) 方法

该方法主要用于提供属性设值，依赖注入的机会, non-lazy 的 singleton bean 都已经实例化，如果这里可以放心的注入。

```java
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
    if (bw == null) {
        if (mbd.hasPropertyValues()) {
            throw new BeanCreationException(
                    mbd.getResourceDescription(), beanName, 
                    "Cannot apply property values to null instance");
        } else {
            // Skip property population phase for null instance.
            return;
        }
    }

    // 1)
    // Give any InstantiationAwareBeanPostProcessors the opportunity to modify the
    // state of the bean before properties are set. This can be used, for example,
    // to support styles of field injection.
    if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
        for (InstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().instantiationAware) {
            if (!bp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
                return;
            }
        }
    }
    PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);

    // 2) 
    int resolvedAutowireMode = mbd.getResolvedAutowireMode();
    if (resolvedAutowireMode == AUTOWIRE_BY_NAME || resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
        MutablePropertyValues newPvs = new MutablePropertyValues(pvs);
        // Add property values based on autowire by name if applicable.
        if (resolvedAutowireMode == AUTOWIRE_BY_NAME) {
            autowireByName(beanName, mbd, bw, newPvs);
        }
        // Add property values based on autowire by type if applicable.
        if (resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
            autowireByType(beanName, mbd, bw, newPvs);
        }
        pvs = newPvs;
    }

    boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
    boolean needsDepCheck = (mbd.getDependencyCheck() != AbstractBeanDefinition.DEPENDENCY_CHECK_NONE);

    PropertyDescriptor[] filteredPds = null;
    if (hasInstAwareBpps) {
        if (pvs == null) {
            pvs = mbd.getPropertyValues();
        }

        // 3)
        for (InstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().instantiationAware) {
            PropertyValues pvsToUse = bp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
            if (pvsToUse == null) {
                if (filteredPds == null) {
                    filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
                }
                pvsToUse = bp.postProcessPropertyValues(pvs, filteredPds, 
                    bw.getWrappedInstance(), beanName);
                if (pvsToUse == null) { return; }
            }
            pvs = pvsToUse;
        }
    }
    if (needsDepCheck) {
        if (filteredPds == null) {
            filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
        }
        checkDependencies(beanName, mbd, filteredPds, pvs);
    }

    // 4)
    if (pvs != null) {
        applyPropertyValues(beanName, mbd, bw, pvs);
    }
}
```

1. 首先调用一遍 `boolean InstantiationAwareBeanPostProcessor.postProcessAfterInstantiation(Object bean, String beanName)` 方法，该方法返回的 boolean 值决定该对象是否需要属性设值，如果返回 false 则直接结束该方法，默认返回 true。
2. 开始使用 `autowireByName(...)` 和 `autowireByType(...)` 进行属性注入 (不包含 `@Autowired` 类似的依赖注入), 本质上使用的也是 `AutowireCapableBeanFactory.resolveDependency` 方法，该方法也由 `DefaultListableBeanFactory` 实现。
3. 然后调用一遍 `PropertyValues InstantiationAwareBeanPostProcessor.postProcessProperties(PropertyValues pvs, Object bean, String beanName)` 方法。这个方法给予回调一个机会去更改将要被 applied 的 `PropertyValues`, 同时这个方法也被用于进行依赖注入，例如 `AutowiredAnnotationBeanPostProcessor`, 关键是，这里使用的依赖注入并不是通过 `PropertyValues` 而是直接进行反射注入 (看下面代码)。

```java
public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) {
    InjectionMetadata metadata = findAutowiringMetadata(beanName, bean.getClass(), pvs);
    try {
        metadata.inject(bean, beanName, pvs);
    } catch (BeanCreationException ex) {
        throw ex;
    } catch (Throwable ex) {
        throw new BeanCreationException(beanName, "Injection of autowired dependencies failed", ex);
    }
    return pvs;
}
```

4. 最后，应用 `PropertyValues` 到 bean 中。

### 4.11.5 AbstractAutowireCapableBeanFactory.initializeBean(...) 方法

该方法主要用于初始化 bean, 此时 bean 已经实例化，并且属性都已经设置完毕。这个方法里则会调用 aware 相关的回调，`init` 方法还有 `BeanPostProcessor` 相关的回调。 

```java
protected Object initializeBean(String beanName, Object bean, @Nullable RootBeanDefinition mbd) {
    if (System.getSecurityManager() != null) {
        AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
            invokeAwareMethods(beanName, bean);
            return null;
        }, getAccessControlContext());
    } else {
        // 1)
        invokeAwareMethods(beanName, bean);
    }

    Object wrappedBean = bean;
    if (mbd == null || !mbd.isSynthetic()) {
        // 2)
        wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
    }

    try {
        // 3)
        invokeInitMethods(beanName, wrappedBean, mbd);
    } catch (Throwable ex) {
        throw new BeanCreationException(
                (mbd != null ? mbd.getResourceDescription() : null),
                beanName, "Invocation of init method failed", ex);
    }
    if (mbd == null || !mbd.isSynthetic()) {
        // 4)
        wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
    }
    return wrappedBean;
}
```

该方法包含四个步骤：

1. 调用 aware 相关回调

```java
private void invokeAwareMethods(String beanName, Object bean) {
    if (bean instanceof Aware) {
        if (bean instanceof BeanNameAware) {
            ((BeanNameAware) bean).setBeanName(beanName);
        }
        if (bean instanceof BeanClassLoaderAware) {
            ClassLoader bcl = getBeanClassLoader();
            if (bcl != null) {
                ((BeanClassLoaderAware) bean).setBeanClassLoader(bcl);
            }
        }
        if (bean instanceof BeanFactoryAware) {
            ((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
        }
    }
} 
```

2. 调用 `BeanPostProcessor.postProcessBeforeInitialization(Object bean, String beanName)` 方法，该方法返回的 bean 会作为我们接下来使用的 bean, 可以认为这个方法给予一个返回 bean wrapper 的机会，默认是返回传入的 bean. 我们也可以实现该方法, 调用相关的 aware 方法。

```java
public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)
        throws BeansException {

    Object result = existingBean;
    for (BeanPostProcessor processor : getBeanPostProcessors()) {
        Object current = processor.postProcessBeforeInitialization(result, beanName);
        if (current == null) {
            return result;
        }
        result = current;
    }
    return result;
}
```

3. 调用 init 方法，或，如果是 `InitializingBean` 就调用 `InitializingBean.afterPropertiesSet()` 方法，初始化本质-上就是这两个方法的调用

```java
protected void invokeInitMethods(String beanName, Object bean, @Nullable RootBeanDefinition mbd)
        throws Throwable {

    boolean isInitializingBean = (bean instanceof InitializingBean);
    if (isInitializingBean 
            && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
        if (System.getSecurityManager() != null) {
            try {
                AccessController.doPrivileged((PrivilegedExceptionAction<Object>) () -> {
                    ((InitializingBean) bean).afterPropertiesSet();
                    return null;
                }, getAccessControlContext());
            }
            catch (PrivilegedActionException pae) {
                throw pae.getException();
            }
        } else {
            ((InitializingBean) bean).afterPropertiesSet();
        }
    }
    if (mbd != null && bean.getClass() != NullBean.class) {
        String initMethodName = mbd.getInitMethodName();
        if (StringUtils.hasLength(initMethodName) &&
                !(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&
                !mbd.isExternallyManagedInitMethod(initMethodName)) {
            invokeCustomInitMethod(beanName, bean, mbd);
        }
    }
}
```

4. 调用 `BeanPostProcessor.postProcessAfterInitialization(Object bean, String beanName)`, 同样的，返回的 bean 我们会作为接下来使用的 bean. 

```java
public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
        throws BeansException {

    Object result = existingBean;
    for (BeanPostProcessor processor : getBeanPostProcessors()) {
        Object current = processor.postProcessAfterInitialization(result, beanName);
        if (current == null) {
            return result;
        }
        result = current;
    }
    return result;
}
```

