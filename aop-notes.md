# AOP Notes

Spring AOP 实现基于两个部分： 

- Proxy 如何创建
    - CGLIB
    - JDK Proxy
- 什么时候创建并返回 Proxy
    - BeanPostProcessor

核心的类和结构：

```
           BeanPostProcessor
                  ^
                  |
   InstantiationAwareBeanPostProcessor
                  ^
                  |
SmartInstantiationAwareBeanPostProcessor
                  ^
                  |
        AbstractAutoProxyCreator     // postProcessAfterInitialization
                  ^
                  |
     AbstractAdvisorAutoProxyCreator // getAdvicesAndAdvisorsForBean
                  ^
                  |
  AspectJAwareAdvisorAutoProxyCreator
                  ^
                  |
 AnnotationAwareAspectJAutoProxyCreator // findCandidateAdvisors
```

## AbstractAutoProxyCreator 和 BeanPostProcessor

首先，`AbstractAutoProxyCreator` 是核心的类, 它通过实现 `BeanPostProcessor` 的 `postProcessAfterInitialization` 方法，来为传入的 bean 加一层 proxy。而 `AbstractAdvisorAutoProxyCreator` 作为它的子类，提供了 `AbstractAutoProxyCreator` 的 `getAdvicesAndAdvisorsForBean` 方法, 该方法用于收集该 bean 将会使用的 advice 和 interceptor。

首先看 `AbstractAutoProxyCreator` 对 `BeanPostProcessor.postProcessAfterInitialization` 方法的实现，本质上就是当当前 bean 能被 proxied 时，使用 `wrapIfNecessary` 方法。

```java
public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
    if (bean != null) {
        Object cacheKey = getCacheKey(bean.getClass(), beanName);
        if (this.earlyProxyReferences.remove(cacheKey) != bean) {
            return wrapIfNecessary(bean, beanName, cacheKey);
        }
    }
    return bean;
}
```

而 `wrapIfNecessary` 方法也非常直接，主要依赖于子类实现的 `getAdvicesAndAdvisorsForBean` 方法来进行 advice 和 interceptor 的收集，然后通过 `createProxy` 方法进行 proxy 的创建。

核心方法：

- `abstract Object[] getAdvicesAndAdvisorsForBean(Class<?> beanClass, String beanName, TargetSource customTargetSource) throws BeansException`
- `Object createProxy(Class<?> beanClass, String beanName, Object[] specificInterceptors, TargetSource targetSource)`

```java
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
    if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
        return bean;
    }
    if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
        return bean;
    }
    if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
        this.advisedBeans.put(cacheKey, Boolean.FALSE);
        return bean;
    }

    // Create proxy if we have advice.
    Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
    if (specificInterceptors != DO_NOT_PROXY) {
        this.advisedBeans.put(cacheKey, Boolean.TRUE);
        Object proxy = createProxy(
                bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
        this.proxyTypes.put(cacheKey, proxy.getClass());
        return proxy;
    }

    this.advisedBeans.put(cacheKey, Boolean.FALSE);
    return bean;
}
```

## 收集 Advices 和 Advisors

先看 `AbstractAdvisorAutoProxyCreator` 对 `getAdvicesAndAdvisorsForBean` 方法的实现，该实现说到底就是从 `BeanFactory` 里拿 `Advisor` 类型的 bean。

```java
protected Object[] getAdvicesAndAdvisorsForBean(
        Class<?> beanClass, String beanName, @Nullable TargetSource targetSource) {

    List<Advisor> advisors = findEligibleAdvisors(beanClass, beanName);
    if (advisors.isEmpty()) {
        return DO_NOT_PROXY;
    }
    return advisors.toArray();
}
```

继续往里走，进入 `findEligibleAdvisors` 方法, 这里关键的方法是 `findCandidateAdvisors` 方法。该方法查出所有的 advisor, 然后我们再判断哪些可用于该类。

```java
protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName) {
    List<Advisor> candidateAdvisors = findCandidateAdvisors();
    List<Advisor> eligibleAdvisors = findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);
    extendAdvisors(eligibleAdvisors);
    if (!eligibleAdvisors.isEmpty()) {
        eligibleAdvisors = sortAdvisors(eligibleAdvisors);
    }
    return eligibleAdvisors;
}
```

进入 `findCandidateAdvisors` 方法，我们可以看到这个方法是基于 `BeanFactoryAdvisorRetrievalHelper` 的，而其内部也是通过 `BeanFactory` 来查找 advisor 而已。

```java
protected List<Advisor> findCandidateAdvisors() {
    Assert.state(this.advisorRetrievalHelper != null, "No BeanFactoryAdvisorRetrievalHelper available");
    return this.advisorRetrievalHelper.findAdvisorBeans();
}

public List<Advisor> findAdvisorBeans() {
    // Determine list of advisor bean names, if not cached already.
    String[] advisorNames = this.cachedAdvisorBeanNames;
    if (advisorNames == null) {
        // Do not initialize FactoryBeans here: We need to leave all regular beans
        // uninitialized to let the auto-proxy creator apply to them!
        advisorNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
                this.beanFactory, Advisor.class, true, false);
        this.cachedAdvisorBeanNames = advisorNames;
    }
    if (advisorNames.length == 0) {
        return new ArrayList<>();
    }

    List<Advisor> advisors = new ArrayList<>();
    for (String name : advisorNames) {
        if (isEligibleBean(name)) {
            if (this.beanFactory.isCurrentlyInCreation(name)) {
                if (logger.isTraceEnabled()) {
                    logger.trace("Skipping currently created advisor '" + name + "'");
                }
            }
            else {
                try {
                    advisors.add(this.beanFactory.getBean(name, Advisor.class));
                }
                catch (BeanCreationException ex) {
                    Throwable rootCause = ex.getMostSpecificCause();
                    if (rootCause instanceof BeanCurrentlyInCreationException) {
                        BeanCreationException bce = (BeanCreationException) rootCause;
                        String bceBeanName = bce.getBeanName();
                        if (bceBeanName != null 
                                && this.beanFactory.isCurrentlyInCreation(bceBeanName)) {
                            // ...
                            continue;
                        }
                    }
                    throw ex;
                }
            }
        }
    }
    return advisors;
}
```

那么除了这种类型的 advisor, 基于标签的是如何扫描且加入的呢。`AnnotationAwareAspectJAutoProxyCreator` 作为子类 overrides 了 `findCandidateAdvisors` 方法，加入了通过对 Class 使用反射查看是否存在 `@Aspect` 的方式发现的 advisor，在此子类，我们仍然会调用 `AbstractAdvisorAutoProxyCreator`, 所以以上对 `Advisor` 从 `BeanFactory` 中的查询仍有效。

```java
protected List<Advisor> findCandidateAdvisors() {
    List<Advisor> advisors = super.findCandidateAdvisors();
    if (this.aspectJAdvisorsBuilder != null) {
        advisors.addAll(this.aspectJAdvisorsBuilder.buildAspectJAdvisors());
    }
    return advisors;
}
```

这里可以看出，对 annotation 的扫描是基于 `BeanFactoryAspectJAdvisorsBuilder` 的， 该方法核心在于，拿取所有 beanName, 拿取他们的 Class, 用反射查看是否带有 `@Aspect` 标签。 

```java
public List<Advisor> buildAspectJAdvisors() {
    List<String> aspectNames = this.aspectBeanNames;

    if (aspectNames == null) {
        synchronized (this) {
            aspectNames = this.aspectBeanNames;
            if (aspectNames == null) {
                List<Advisor> advisors = new ArrayList<>();
                aspectNames = new ArrayList<>();
                String[] beanNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
                        this.beanFactory, Object.class, true, false);
                for (String beanName : beanNames) {
                    if (!isEligibleBean(beanName)) {
                        continue;
                    }
                    // We must be careful not to instantiate beans eagerly as in this case they
                    // would be cached by the Spring container but would not have been weaved.
                    Class<?> beanType = this.beanFactory.getType(beanName, false);
                    if (beanType == null) {
                        continue;
                    }

                    if (this.advisorFactory.isAspect(beanType)) {
                        aspectNames.add(beanName);
                        AspectMetadata amd = new AspectMetadata(beanType, beanName);
                        if (amd.getAjType().getPerClause().getKind() == PerClauseKind.SINGLETON) {
                            MetadataAwareAspectInstanceFactory factory =
                                    new BeanFactoryAspectInstanceFactory(this.beanFactory, beanName);
                            List<Advisor> classAdvisors = this.advisorFactory.getAdvisors(factory);
                            if (this.beanFactory.isSingleton(beanName)) {
                                this.advisorsCache.put(beanName, classAdvisors);
                            }
                            else {
                                this.aspectFactoryCache.put(beanName, factory);
                            }
                            advisors.addAll(classAdvisors);
                        }
                        else {
                            // Per target or per this.
                            if (this.beanFactory.isSingleton(beanName)) {
                                throw new IllegalArgumentException("Bean with name '" + 
                                        beanName +
                                        "' is a singleton, but aspect instantiation model is" + 
                                        " not singleton");
                            }
                            MetadataAwareAspectInstanceFactory factory =
                                    new PrototypeAspectInstanceFactory(this.beanFactory, beanName);
                            this.aspectFactoryCache.put(beanName, factory);
                            advisors.addAll(this.advisorFactory.getAdvisors(factory));
                        }
                    }
                }
                this.aspectBeanNames = aspectNames;
                return advisors;
            }
        }
    }

    if (aspectNames.isEmpty()) {
        return Collections.emptyList();
    }
    List<Advisor> advisors = new ArrayList<>();
    for (String aspectName : aspectNames) {
        List<Advisor> cachedAdvisors = this.advisorsCache.get(aspectName);
        if (cachedAdvisors != null) {
            advisors.addAll(cachedAdvisors);
        }
        else {
            MetadataAwareAspectInstanceFactory factory = this.aspectFactoryCache.get(aspectName);
            advisors.addAll(this.advisorFactory.getAdvisors(factory));
        }
    }
    return advisors;
}
```

主要看这里 `this.advisorFactory.isAspect(beanType)`, 进入该方法就能看到对 annotation 的判断：

```java
public boolean isAspect(Class<?> clazz) {
		return (hasAspectAnnotation(clazz) && !compiledByAjc(clazz));
}

private boolean hasAspectAnnotation(Class<?> clazz) {
    return (AnnotationUtils.findAnnotation(clazz, Aspect.class) != null);
}
```

到这里，对 Advice 和 Advisors 的搜集已经完成了。

## 创建 Proxy

接下来看对 Proxy 的创建，也就是 `AbstractAutoProxyCreator.createProxy` 方法。

```java
protected Object createProxy(Class<?> beanClass, @Nullable String beanName,
        @Nullable Object[] specificInterceptors, TargetSource targetSource) {

    if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
        AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, 
            beanName, beanClass);
    }

    ProxyFactory proxyFactory = new ProxyFactory();
    proxyFactory.copyFrom(this);

    if (proxyFactory.isProxyTargetClass()) {
        if (Proxy.isProxyClass(beanClass)) {
            for (Class<?> ifc : beanClass.getInterfaces()) {
                proxyFactory.addInterface(ifc);
            }
        }
    }
    else {
        if (shouldProxyTargetClass(beanClass, beanName)) {
            proxyFactory.setProxyTargetClass(true);
        }
        else {
            evaluateProxyInterfaces(beanClass, proxyFactory);
        }
    }

    Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
    proxyFactory.addAdvisors(advisors);
    proxyFactory.setTargetSource(targetSource);
    customizeProxyFactory(proxyFactory);

    proxyFactory.setFrozen(this.freezeProxy);
    if (advisorsPreFiltered()) {
        proxyFactory.setPreFiltered(true);
    }

    // Use original ClassLoader if bean class not locally loaded in overriding class loader
    ClassLoader classLoader = getProxyClassLoader();
    if (classLoader instanceof SmartClassLoader 
            && classLoader != beanClass.getClassLoader()) {
        classLoader = ((SmartClassLoader) classLoader).getOriginalClassLoader();
    }
    return proxyFactory.getProxy(classLoader);
}
```

实际上最关键的方法也就是最后一行 `proxyFactory.getProxy(classLoader)`，在该方法中，我们关注 `createAopProxy` 方法。

```java
public Object getProxy(@Nullable ClassLoader classLoader) {
    return createAopProxy().getProxy(classLoader);
}

protected final synchronized AopProxy createAopProxy() {
    if (!this.active) {
        activate();
    }
    return getAopProxyFactory().createAopProxy(this);
}
```

这个 `createAopProxy` 方法由 `DefaultAopProxyFactory` 实现，实现代码如下。本质上就是选择使用 JDK Proxy 或者 CGLIB Proxy。然后这个 proxy 内部包含了 interceptor chain，调用的时候就会根据顺序触发该方法对应的 interceptors.

```java
public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
    if (!NativeDetector.inNativeImage() &&
            (config.isOptimize() || config.isProxyTargetClass() 
                || hasNoUserSuppliedProxyInterfaces(config))) {
        Class<?> targetClass = config.getTargetClass();
        if (targetClass == null) {
            throw new AopConfigException("TargetSource cannot determine target class: " +
                    "Either an interface or a target is required for proxy creation.");
        }
        if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
            return new JdkDynamicAopProxy(config);
        }
        return new ObjenesisCglibAopProxy(config);
    }
    else {
        return new JdkDynamicAopProxy(config);
    }
}
```

