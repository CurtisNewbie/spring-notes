# Spring Transaction Management Notes

# 1. 事务属性

Spring 在对事务定义了 6 个属性

1. name 
    - 事务的名称
2. isolation_level 
    - 隔离的等级
3. timeout
    - 超时的时间
4. is_read_only 
    - 是否只读 
5. propagation_behavior
    - 传播行为 
6. rollbackFor* / noRollbackFor*
    - 回滚机制`

相应的可以在 `@Transactional` 中定义或看 `TransactionDefinition` 和 `TransactionAttribute` 类.

## 1.1 TransactionDefinition

```java
public interface TransactionDefinition {

	default int getPropagationBehavior() {
		return PROPAGATION_REQUIRED;
	}

	default int getIsolationLevel() {
		return ISOLATION_DEFAULT;
	}

	default int getTimeout() {
		return TIMEOUT_DEFAULT;
	}

	default boolean isReadOnly() {
		return false;
	}

	@Nullable
	default String getName() {
		return null;
	}
}
```

## 1.2 TransactionAttribute

注意, `TransactionAttribute` 继承 `TransactionDefinition`

```java
public interface TransactionAttribute extends TransactionDefinition {

	@Nullable
	String getQualifier();

	Collection<String> getLabels();

	boolean rollbackOn(Throwable ex);

}
```

## 1.3 @Transactional

基于 annotation 的编写方式:

```java
public @interface Transactional {

	@AliasFor("transactionManager")
	String value() default "";

	@AliasFor("value")
	String transactionManager() default "";

	String[] label() default {};

	Propagation propagation() default Propagation.REQUIRED;

	Isolation isolation() default Isolation.DEFAULT;

	int timeout() default TransactionDefinition.TIMEOUT_DEFAULT;

	String timeoutString() default "";

	boolean readOnly() default false;

	Class<? extends Throwable>[] rollbackFor() default {};

	String[] rollbackForClassName() default {};

	Class<? extends Throwable>[] noRollbackFor() default {};

	String[] noRollbackForClassName() default {};
}
```

默认的隔离等级是基于数据库的默认等级，而事务传播的默认行为是 `REQUIRED`.

# 2. 事务传播行为

Spring 制定 7 种事务传播行为，对应的常量存储在 `TransactionDefinition` 中:

```java
public interface TransactionDefinition {

	int PROPAGATION_REQUIRED = 0;
	int PROPAGATION_SUPPORTS = 1;
	int PROPAGATION_MANDATORY = 2;
	int PROPAGATION_REQUIRES_NEW = 3;
	int PROPAGATION_NOT_SUPPORTED = 4;
	int PROPAGATION_NEVER = 5;
	int PROPAGATION_NESTED = 6;

}
```

或者我们也可以看 `enum Propagation`, 不过使用的值是一致的:

```java
public enum Propagation {

	REQUIRED(TransactionDefinition.PROPAGATION_REQUIRED),
	SUPPORTS(TransactionDefinition.PROPAGATION_SUPPORTS),
	MANDATORY(TransactionDefinition.PROPAGATION_MANDATORY),
	REQUIRES_NEW(TransactionDefinition.PROPAGATION_REQUIRES_NEW),
	NOT_SUPPORTED(TransactionDefinition.PROPAGATION_NOT_SUPPORTED),
	NEVER(TransactionDefinition.PROPAGATION_NEVER),
	NESTED(TransactionDefinition.PROPAGATION_NESTED);
}
```

1. `PROPAGATION_REQUIRED`
    - 如果事务不存在, 则创建一个新的事务 (默认等级)
2. `PROPAGATION_SUPPORTS`
    - 如果事务存在, 支持当前事务, 如果不存在事务, 以非事务方式执行
3. `PROPAGATION_MANDATORY`
    - 如果事务存在, 支持当前事务, 如果不存在事务, 抛异常  
4. `PROPAGATION_REQUIRES_NEW`
    - 如果事务存在, 挂起 (suspend) 当前事务并且创建新的事务 
    - 当异常发生, 只回滚新创建的事务, 而不是挂起的事务
5. `PROPAGATION_NOT_SUPPORTED`
    - 如果事务存在, 挂起 (suspend) 当前事务, 并且以非事务的方式执行
6. `PROPAGATION_NEVER` 
    - 如果事务存在, 抛异常, 以非事务的方式执行
7. `PROPAGATION_NESTED`
    - 如果事务存在, 则在嵌套事务中执行, 否则创建新事务 
    - 基于保存点, 实际上是同一个物理事务, 外部异常则全部回滚, 内部异常则回滚到上一个保存点

# 3. 基于代码编程的事务管理

对于代码编程的事务管理主要使用的是:

- `PlatformTransactionManager`
- `TransactionTemplate`

在下面代码中也可以看出, 我们使用了 `TransactionTemplate` 自带的基于 Callback 的方式编写事务代码. 但 `TransactionTemplate` 本身是用于简化事务管理的, 核心仍然是 `PlatformTransactionManager`.

```java
private final TransactionTemplate transactionTemplate;

public UserServiceImpl(PlatformTransactionManager transactionManager) {
    transactionTemplate = new TransactionTemplate(transactionManager);
}

@Override
public UserEntity loadUserByUsername(@NotEmpty String s) throws UsernameNotFoundException {
    return transactionTemplate.execute(status -> {
        Objects.requireNonNull(s);
        UserEntity userEntity = userMapper.findByUsername(s);
        if (userEntity == null)
            try {
                throw new UsernameNotFoundException("...");
            } catch (UsernameNotFoundException e) {
                status.setRollbackOnly();
            }
        return userEntity;
    });
}
```

## 3.1 PlatformTransactionManager 

首先 `PlatformTransactionManager` 继承 `TransactionManager`, 但 `TransactionManager` 只是一个 marker interface.

```java
public interface TransactionManager {

}

public interface PlatformTransactionManager extends TransactionManager {

	TransactionStatus getTransaction(@Nullable TransactionDefinition definition)
			throws TransactionException;

	void commit(TransactionStatus status) throws TransactionException;

	void rollback(TransactionStatus status) throws TransactionException;
}
```

我们可以看到 `PlatformTransactionManager` 提供了三个接口:

- `TransactionStatus getTransaction(TransactionDefinition)`
    - 返回当前事务或创建新的事务, 取决于事务的属性 (`TransactionDefinition` 中的 `getPropagationBehavior()`), 
- `void commit(TransactionStatus status)`
    - 提交事务
- `void rollback(TransactionStatus status)` 
    - 回滚事务

依赖这三个接口，我们应该可以进行事务管理, 但 `TransactionTemplate` 进一步简化了以 programmatic 编写事务管理的代码.

## 3.2 TransactionTemplate

我们可以看到 `TransactionTemplate` 内部包含了 `PlatformTransactionManager`, 同时我们留意, `TransactionTemplate` 是一个 `InitializingBean`, 在该 bean 初始化的时候会检查 `PlatformTransactionManager` 是否存在.

```java
public class TransactionTemplate extends DefaultTransactionDefinition
		implements TransactionOperations, InitializingBean {

	@Nullable
	private PlatformTransactionManager transactionManager;

	@Override
	public void afterPropertiesSet() {
		if (this.transactionManager == null) {
			throw new IllegalArgumentException("...");
		}
	}

	@Override
	@Nullable
	public <T> T execute(TransactionCallback<T> action) throws TransactionException {
		Assert.state(this.transactionManager != null, "No PlatformTransactionManager set");

		if (this.transactionManager instanceof CallbackPreferringPlatformTransactionManager) {
			return ((CallbackPreferringPlatformTransactionManager) this.transactionManager)
                .execute(this, action);
		} else {
			TransactionStatus status = this.transactionManager.getTransaction(this);
			T result;
			try {
				result = action.doInTransaction(status);
			} catch (RuntimeException | Error ex) {
				rollbackOnException(status, ex);
				throw ex;
			} catch (Throwable ex) {
				rollbackOnException(status, ex);
				throw new UndeclaredThrowableException("...");
			}
			this.transactionManager.commit(status);
			return result;
		}
	}

	private void rollbackOnException(TransactionStatus status, Throwable ex) 
            throws TransactionException {
		try {
			this.transactionManager.rollback(status);
		} catch (TransactionSystemException ex2) {
			ex2.initApplicationException(ex);
			throw ex2;
		} catch (RuntimeException | Error ex2) {
			throw ex2;
		}
	}

}
```

接下来我们重点看 `execute(TransactionCallback)` 方法, 本质上来说 `TransactionCallBack` 就是一个简单的 callback, 这个方法运行完这个 callback 以后就会在返回结果之前使用 `TransactionManager.commit` 方法进行提交, 特别的地方主要在于对异常的处理, 当 `TransactionTemplate` 捕获到异常的时候就会使用内部的 `TransactionManager.rollback` 方法进行回滚.

# 4. 基于 Annotation 的事务管理

Spring 中基于 Annotation 的事务管理是通过 AOP 实现的, 本质上就是对 bean 生成动态 proxy, 内部包含每个方法对应的 interceptor, 在方法调用之后会先过一遍 interceptor. 

对于 Spring-Boot AutoConfiguration, 主要看类 `TransactionAutoConfiguration`。重点看 `@EnableTransactionManagement`.

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(TransactionManagementConfigurationSelector.class)
public @interface EnableTransactionManagement {

	boolean proxyTargetClass() default false;
	AdviceMode mode() default AdviceMode.PROXY;
	int order() default Ordered.LOWEST_PRECEDENCE;
}
```

留意 `@Import(TransactionManagementConfigurationSelector.class)`, 这个标签引入了这个类, 而这个类进一步引入了 `PROXY` / `ASPECTJ` 两种模式对应的配置类. 

```java
public class TransactionManagementConfigurationSelector extends 
                AdviceModeImportSelector<EnableTransactionManagement> {

	@Override
	protected String[] selectImports(AdviceMode adviceMode) {
		switch (adviceMode) {
			case PROXY:
				return new String[] {AutoProxyRegistrar.class.getName(),
						ProxyTransactionManagementConfiguration.class.getName()};
			case ASPECTJ:
				return new String[] {determineTransactionAspectClass()};
			default:
				return null;
		}
	}

	private String determineTransactionAspectClass() {
		return (ClassUtils.isPresent("javax.transaction.Transactional", getClass().getClassLoader()) ?
				TransactionManagementConfigUtils.JTA_TRANSACTION_ASPECT_CONFIGURATION_CLASS_NAME :
				TransactionManagementConfigUtils.TRANSACTION_ASPECT_CONFIGURATION_CLASS_NAME);
	}
}
```

例如, 对应 `PROXY`, 该类引入 `ProxyTransactionManagementConfiguration` , 然后我们在被引入的配置类中看到创建 AOP `MethodInterceptor` (`TransactionInterceptor`) 和 advisor (`BeanFactoryTransactionAttributeSourceAdvisor`) 的代码:  

```java
@Configuration(proxyBeanMethods = false)
@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
public class ProxyTransactionManagementConfiguration 
                        extends AbstractTransactionManagementConfiguration {

	@Bean(name = TransactionManagementConfigUtils.TRANSACTION_ADVISOR_BEAN_NAME)
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public BeanFactoryTransactionAttributeSourceAdvisor transactionAdvisor(
			TransactionAttributeSource transactionAttributeSource, 
            TransactionInterceptor transactionInterceptor) {

		BeanFactoryTransactionAttributeSourceAdvisor advisor = 
            new BeanFactoryTransactionAttributeSourceAdvisor();
		advisor.setTransactionAttributeSource(transactionAttributeSource);
		advisor.setAdvice(transactionInterceptor);
		if (this.enableTx != null) {
			advisor.setOrder(this.enableTx.<Integer>getNumber("order"));
		}
		return advisor;
	}

	@Bean
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public TransactionAttributeSource transactionAttributeSource() {
		return new AnnotationTransactionAttributeSource();
	}

	@Bean
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public TransactionInterceptor transactionInterceptor(TransactionAttributeSource    
            transactionAttributeSource) {
		TransactionInterceptor interceptor = new TransactionInterceptor();
		interceptor.setTransactionAttributeSource(transactionAttributeSource);
		if (this.txManager != null) {
			interceptor.setTransactionManager(this.txManager);
		}
		return interceptor;
	}
}
```

那么这个时候我们知道, 对应带有 `@Transactional` 的方法由 `TransactionInterceptor` 通过 AOP 生成的动态代理进行管理, 只要看 `TransactionInterceptor` 在方法调用前后做了什么就知道事务如何管理了.

## 4.1 TransactionInterceptor

`TransactionInterceptor` 实现了 `MethodInterceptor`, 具体接口如下:

```java
public interface MethodInterceptor extends Interceptor {

	Object invoke( MethodInvocation invocation) throws Throwable;
}
```

然后我们看 `TransactionInterceptor` 对这个接口的实现:

```java
@Override
@Nullable
public Object invoke(MethodInvocation invocation) throws Throwable {
    Class<?> targetClass = (invocation.getThis() != null ? 
        AopUtils.getTargetClass(invocation.getThis()) : null);

    return invokeWithinTransaction(invocation.getMethod(), targetClass, 
        new CoroutinesInvocationCallback() {
            @Override
            @Nullable
            public Object proceedWithInvocation() throws Throwable {
                return invocation.proceed();
            }
            @Override
            public Object getTarget() {
                return invocation.getThis();
            }
            @Override
            public Object[] getArguments() {
                return invocation.getArguments();
            }
    });
}
```

可以看到我们实际上还是调用的 `invokeWithinTransaction` 方法，而这个方法源自父类 `TransactionAspectSupport`, 需要关注的是, 这里的代码仍然是在使用 `PlatformTransactionManager`, 具体如下:

```java
// TransactionAspectSupport
protected Object invokeWithinTransaction(Method method, @Nullable Class<?> targetClass,
        final InvocationCallback invocation) throws Throwable {

    // 1)
    TransactionAttributeSource tas = getTransactionAttributeSource();
    final TransactionAttribute txAttr = 
            (tas != null ? tas.getTransactionAttribute(method, targetClass) : null);
    final TransactionManager tm = determineTransactionManager(txAttr);

    // reactive transaction management code, ignored ......

    PlatformTransactionManager ptm = asPlatformTransactionManager(tm);
    final String joinpointIdentification = methodIdentification(method, targetClass, txAttr);

    // 2)
    if (txAttr == null || !(ptm instanceof CallbackPreferringPlatformTransactionManager)) {
        TransactionInfo txInfo = createTransactionIfNecessary(ptm, txAttr, joinpointIdentification);
        Object retVal;
        try {
            // 3)
            retVal = invocation.proceedWithInvocation();
        } catch (Throwable ex) {
            // 4)
            completeTransactionAfterThrowing(txInfo, ex);
            throw ex;
        } finally {
            // 5)
            cleanupTransactionInfo(txInfo);
        }
        // vavr library code, ignored ......

        // 6)
        commitTransactionAfterReturning(txInfo);
        return retVal;
    }

    // 7)
    else {
        Object result;
        final ThrowableHolder throwableHolder = new ThrowableHolder();

        try {
            result = ((CallbackPreferringPlatformTransactionManager) ptm).execute(txAttr, status -> {
                TransactionInfo txInfo = 
                    prepareTransactionInfo(ptm, txAttr, joinpointIdentification, status);
                try {
                    Object retVal = invocation.proceedWithInvocation();
                    // vavr library code, ignored ......
                    return retVal;
                } catch (Throwable ex) {
                    if (txAttr.rollbackOn(ex)) {
                        if (ex instanceof RuntimeException) {
                            throw (RuntimeException) ex;
                        } else {
                            throw new ThrowableHolderException(ex);
                        }
                    } else {
                        throwableHolder.throwable = ex;
                        return null;
                    }
                } finally {
                    cleanupTransactionInfo(txInfo);
                }
            });
        } catch (ThrowableHolderException ex) {
            throw ex.getCause();
        } catch (TransactionSystemException ex2) {
            if (throwableHolder.throwable != null) {
                ex2.initApplicationException(throwableHolder.throwable);
            }
            throw ex2;
        } catch (Throwable ex2) {
            throw ex2;
        }

        if (throwableHolder.throwable != null) {
            throw throwableHolder.throwable;
        }
        return result;
    }
}
```

这段代码是使用 `PlatformTransactionManager` 的核心:

1. 首先我们尝试获取当前线程该方法的 `TransactionAttribute txAttr` 事务属性, 如果 `txAttr == null`, 当前线程并没有事务.
2. 如果当前线程没有事务或者不是基于回调类型的 `PlatformTransactionManager`, 我们进入该分支, 我们依据 `TransactionAttribute txAttr` 决定我们是否需要创建一个新事务, 如果有已存在的事务, 我们直接返回现有事务 (具体看后面)
3. 调用方法 (也可能是下一个 `MethodInterceptor`), 获取结果
4. 如果捕获到异常, 我们判断是否需要回滚, 需要的话我们就回滚 (回忆 `@Transactional(noRollbackFor = ...)`), 不需要我们就提交并且-抛出异常
5. 总是为当前线程恢复到旧的 `TransactionInfo` 状态
6. 如果执行成功, 没有异常, 这里提交事务
7. 基于 CallbackPreferringPlatformTransactionManager 的编写方式, 跟上面本质上没有太大区别


所以这里我们也可以总结为四个核心的方法:

- `TransactionInfo txInfo = createTransactionIfNecessary(ptm, txAttr, joinpointIdentification)`
    - 获取或创建事务
- `completeTransactionAfterThrowing(txInfo, ex)`
    - 提交或回滚事务
- `commitTransactionAfterReturning(txInfo)`
    - 提交事务
- `cleanupTransactionInfo(txInfo)`


## 4.2 TransactionAspectSupport 的 createTransactionIfNecessary 方法

回忆, `TransactionAspectSupport` 是 `TransactionInterceptor` 的父类.

```java
// TransactionAspectSupport
protected TransactionInfo createTransactionIfNecessary(@Nullable PlatformTransactionManager tm,
        @Nullable TransactionAttribute txAttr, final String joinpointIdentification) {

    // 1)
    if (txAttr != null && txAttr.getName() == null) {
        txAttr = new DelegatingTransactionAttribute(txAttr) {
            @Override
            public String getName() {
                return joinpointIdentification;
            }
        };
    }

    // 2)
    TransactionStatus status = null;
    if (txAttr != null) {
        if (tm != null) {
            status = tm.getTransaction(txAttr);
        }
    }

    // 3)
    return prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);
}

// TransactionAspectSupport
protected TransactionInfo prepareTransactionInfo(@Nullable PlatformTransactionManager tm,
        @Nullable TransactionAttribute txAttr, String joinpointIdentification,
        @Nullable TransactionStatus status) {

    TransactionInfo txInfo = new TransactionInfo(tm, txAttr, joinpointIdentification);
    if (txAttr != null) {
        // 4)
        txInfo.newTransactionStatus(status);
    }

    // 5)
    txInfo.bindToThread();
    return txInfo;
}

// TransactionAspectSupport
private void bindToThread() {
    this.oldTransactionInfo = transactionInfoHolder.get();
    transactionInfoHolder.set(this);
}
```

1. 如果 ``TransactionAttribute != null`, 但是没有名字, 我们设置事务的名字 
2. 如果 ``TransactionAttribute != null` 我们通过 `PlatformTransactionManager` 获取 `TransactionStatus`, `TransactionStatus` 对象代表一个事务的状态, 例如, `isNewTransaction`, `isRollbackOnly`, `isCompleted` 等 
3. 组装 `TransactionInfo` 对象, 实际上该对象就是把所有信息包装到一齐而已
4. 在我们新创建的 `TransactionInfo` 对象中设置我们目前的状态 `TransactionStatus` 
5. 记录旧的事务状态, 更新新的事务状态 (就是目前的状态), 这里的代码使用 `ThreadLocal<TransactionInfo>`, 所以事务能够在不同方-法中传播, 这也是为什么这个方法命名为 `bindToThread`.
    - `TransactionInfo` 是 `TransactionAspectSupport` 的 static inner class, `ThreadLocal<TransactionInfo> transactionInfoHolder` 是 static variable. 

## 4.3 TransactionAspectSupport 的 completeTransactionAfterThrowing(TransactionInfo, Throwable) 方法

这个方法非常直接, 我们通过 `Throwable` 和 `TransactionAttribute.rollbackOn` 方法判断我们是否应该对该异常回滚, 因为异常已经发生, 我们在这一步必须结束这个事务, 不管是提交还是回滚. 而提交和回滚也是通过 `PlatformTransactionManager` 进行的.
```java
// TransactionAspectSupport 
protected void completeTransactionAfterThrowing(@Nullable TransactionInfo txInfo, Throwable ex) {
    if (txInfo != null && txInfo.getTransactionStatus() != null) {
        if (txInfo.transactionAttribute != null && txInfo.transactionAttribute.rollbackOn(ex)) {
            try {
                txInfo.getTransactionManager().rollback(txInfo.getTransactionStatus());
            } catch (TransactionSystemException ex2) {
                ex2.initApplicationException(ex);
                throw ex2;
            } catch (RuntimeException | Error ex2) {
                throw ex2;
            }
        } else {
            try {
                txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
            } catch (TransactionSystemException ex2) {
                ex2.initApplicationException(ex);
                throw ex2;
            } catch (RuntimeException | Error ex2) {
                throw ex2;
            }
        }
    }
}
```

## 4.4 TransactionAspectSupport 的 commitTransactionAfterReturning(TransactionInfo) 方法

这一步并没有特别的地方, 主要是使用 `PlatformTransactionManager.commit` 方法提交事务

```java
protected void commitTransactionAfterReturning(@Nullable TransactionInfo txInfo) {
    if (txInfo != null && txInfo.getTransactionStatus() != null) {
        txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
    }
}
```

## 4.5 TransactionAspectSupport 的 cleanupTransactionInfo(TransactionInfo) 方法

这个方法用于恢复上一段事务的状态, 主要使用 `TransactionAspectSupport` 内部的 `static ThreadLocal<TransactionInfo> transactionInfoHolder`.

```java
protected void cleanupTransactionInfo(@Nullable TransactionInfo txInfo) {
    if (txInfo != null) {
        txInfo.restoreThreadLocalStatus();
    }
}

// TransactionAspectSupport
private void restoreThreadLocalStatus() {
    transactionInfoHolder.set(this.oldTransactionInfo);
}
```

# 5. PlatformTransactionManager

由上面代码可以看出, 事务管理核心在于 `PlatformTransactionManager` 的[三个方法](#31-platformtransactionmanager)

## 5.1 PlatformTransactionManager 的 getTransaction(TransactionDefinition) 方法

```java
TransactionStatus getTransaction(@Nullable TransactionDefinition definition) throws TransactionException
```

首先看核心的类的层级关系:

```
    PlatformTransactionManager
                ^
                |
AbstractPlatformTransactionManager
                ^
                |
  DataSourceTransactionManager
                ^
                |
     JdbcTransactionManager
```

我们可以看到, 最核心的类是 `AbstractPlatformTransactionManager`, 同样也是用的 template design pattern.

```java
// AbstractPlatformTransactionManager
public final TransactionStatus getTransaction(@Nullable TransactionDefinition definition)
        throws TransactionException {

    // 1)
    TransactionDefinition def = (definition != null ? definition : TransactionDefinition.withDefaults());

    // 2)
    Object transaction = doGetTransaction();

    // 3)
    if (isExistingTransaction(transaction)) {
        // 4)
        return handleExistingTransaction(def, transaction, debugEnabled);
    }

    if (def.getTimeout() < TransactionDefinition.TIMEOUT_DEFAULT) {
        throw new InvalidTimeoutException("..."));
    }

    // 5)
    if (def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_MANDATORY) {
        throw new IllegalTransactionStateException("...");
    } else if (def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRED ||
            def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW ||
            def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
        
        // 6)
        SuspendedResourcesHolder suspendedResources = suspend(null);
        try {
            return startTransaction(def, transaction, debugEnabled, suspendedResources);
        } catch (RuntimeException | Error ex) {
            resume(null, suspendedResources);
            throw ex;
        }
    } else {
        // 7)
        boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
        return prepareTransactionStatus(def, null, true, newSynchronization, debugEnabled, null);
    }
}
```

1. 首先判断传入的 `TransactionDefinition` 是否为 `null`, 如果是, 我们创建一个使用默认值.
2. 我们从子类获取 `transaction` 对象 (这是旧事务, 或者说现有事务), 对于 `DataSourceTransactionManager`, 该对象是 `DataSourceTransactionObject`
    - 这里要关注的是 `doGetTransaction` 是 abstract 方法, 而我们这里拿到的 `transaction` 对象是使用 `ThreadLocal` 与当前线程绑定的 (如果上一个方法已经创建了事务, 那么这里就可以检测到这个事务了)
    - `TransactionSynchronizationManager` 内部包含多个 static `ThreadLocal` 变量用于同一线程不同方法之间事务传播的管理.
    - 如果当前线程并没有已创建的事务, `ConnectionHolder conHolder` 应该是 `null`, 这代表新事务

```java
// DataSourceTransactionManager
protected Object doGetTransaction() {
    DataSourceTransactionObject txObject = new DataSourceTransactionObject();
    txObject.setSavepointAllowed(isNestedTransactionAllowed());
    ConnectionHolder conHolder =
            (ConnectionHolder) TransactionSynchronizationManager.getResource(obtainDataSource());
    txObject.setConnectionHolder(conHolder, false);
    return txObject;
}

// TransactionSynchronizationManager
public static Object getResource(Object key) {
    Object actualKey = TransactionSynchronizationUtils.unwrapResourceIfNecessary(key);
    Object value = doGetResource(actualKey);
    return value;
}

// TransactionSynchronizationManager
private static Object doGetResource(Object actualKey) {
    Map<Object, Object> map = resources.get();
    if (map == null) {
        return null;
    }
    Object value = map.get(actualKey);
    if (value instanceof ResourceHolder && ((ResourceHolder) value).isVoid()) {
        map.remove(actualKey);
        // Remove entire ThreadLocal if empty...
        if (map.isEmpty()) {
            resources.remove();
        }
        value = null;
    }
    return value;
}
```

3. 然后我们检查是否已经有事务存在, 如果存在我们要检查我们的 propagation_behavior 来决定我们如果进行下一步. 该方法默认返回 `false`, 由子类 override (如 `DataSourceTransactionManager`). 

```java
// AbstractPlatformTransactionManager
protected boolean isExistingTransaction(Object transaction) throws TransactionException {
    return false;
}

// DataSourceTransactionManager
protected boolean isExistingTransaction(Object transaction) {
    DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;
    return (txObject.hasConnectionHolder() && txObject.getConnectionHolder().isTransactionActive());
}
```

4. 接下来的方法是对已存在事务处理的问题, 主要判断 propagation behavior 传播行为.
    - 传播行为:
        1. `PROPAGATION_NEVER` 抛异常
        2. `PROPAGATION_NOT_SUPPORTED`, 挂起当前线程 
        3. `PROPAGATION_REQUIRES_NEW`, 挂起当前线程, 创建新线程
        4. `PROPAGATION_NESTED`, 嵌套事务, 使用 JDBC SavePoints 功能
        5. `PROPAGATION_SUPPORTS`, `PROPAGATION_REQUIRED` 或 `PROPAGATION_MANDATORY` 加入现有线程
    - 这里留意 `resume` 方法对事务的挂起, 这里说的挂起就是将 `TransactionSynchronizationManager` 对该线程关于事务绑定的 `ThreadLocal` 数据都保存到 `DefaultTransactionStatus.suspendedResources` 中并且清除, 等新事务完成, 再把这些信息恢复到 `TransactionSynchronizationManager` 中 (具体看 `cleanupAfterCompletion` ).

```java
// AbstractPlatformTransactionManager
private TransactionStatus handleExistingTransaction(
        TransactionDefinition definition, Object transaction, boolean debugEnabled)
        throws TransactionException {
    // 1)
    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NEVER) {
        throw new IllegalTransactionStateException("...");
    }
    // 2)
    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NOT_SUPPORTED) {
        Object suspendedResources = suspend(transaction);
        boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
        return prepareTransactionStatus(
                definition, null, false, newSynchronization, debugEnabled, suspendedResources);
    }
    // 3)
    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW) {
        SuspendedResourcesHolder suspendedResources = suspend(transaction);
        try {
            return startTransaction(definition, transaction, debugEnabled, suspendedResources);
        } catch (RuntimeException | Error beginEx) {
            resumeAfterBeginException(transaction, suspendedResources, beginEx);
            throw beginEx;
        }
    }
    // 4)
    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
        if (!isNestedTransactionAllowed()) {
            throw new NestedTransactionNotSupportedException("...");
        }
        if (useSavepointForNestedTransaction()) {
            DefaultTransactionStatus status =
                    prepareTransactionStatus(definition, transaction, false, false, debugEnabled, null);
            status.createAndHoldSavepoint();
            return status;
        } else {
            return startTransaction(definition, transaction, debugEnabled, null);
        }
    }

    // 5)
    if (isValidateExistingTransaction()) {
        if (definition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT) {
            Integer currentIsolationLevel = TransactionSynchronizationManager.
                    getCurrentTransactionIsolationLevel();
            if (currentIsolationLevel == null 
                    || currentIsolationLevel != definition.getIsolationLevel()) {
                Constants isoConstants = DefaultTransactionDefinition.constants;
                throw new IllegalTransactionStateException("...");
            }
        }
        if (!definition.isReadOnly()) {
            if (TransactionSynchronizationManager.isCurrentTransactionReadOnly()) {
                throw new IllegalTransactionStateException("...");
            }
        }
    }
    boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
    return prepareTransactionStatus(definition, transaction, false, newSynchronization, 
        debugEnabled, null);
}
```

5. 如果没有已经存在的事务, 并且传播行为是 `PROPAGATION_MANDATORY`, 我们抛出异常
6. 如果没有已经存在的事务, 并且传播行为是 `PROPAGATION_REQUIRED` || `PROPAGATION_REQUIRES_NEW` || `PROPAGATION_NESTED`, 我们创建新事务并且开始事务. 
    - 所谓的开始 (`doBegin` 方法), 就是分配 `ConnectionHolder` (内部包含 `Connection` 对象) 并且使用 `Connection` 对象设置一些初始值
    - 例如, 设置 `Connection#setAutoCommit(false)`
    - 例如, 对于 `read-only` 事务, 我们使用 `Connection#createStatement()` 方法获得 `Statement` 然后执行 `"SET TRANSACTION READ ONLY"` (对于 `DataSourceTransactionManager` 归根到底就是用 `JDBC`)
    - 同时, 记录事务信息到 `TransactionSynchronizationManager` 中

```java
// AbstractPlatformTransactionManager
private TransactionStatus startTransaction(TransactionDefinition definition, Object transaction,
        boolean debugEnabled, @Nullable SuspendedResourcesHolder suspendedResources) {

    boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
    DefaultTransactionStatus status = newTransactionStatus(
            definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
    doBegin(transaction, definition);
    prepareSynchronization(status, definition);
    return status;
}

// DataSourceTransactionManager
protected void doBegin(Object transaction, TransactionDefinition definition) {
    DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;
    Connection con = null;

    try {
        if (!txObject.hasConnectionHolder() ||
                txObject.getConnectionHolder().isSynchronizedWithTransaction()) {
            Connection newCon = obtainDataSource().getConnection();
            txObject.setConnectionHolder(new ConnectionHolder(newCon), true);
        }

        txObject.getConnectionHolder().setSynchronizedWithTransaction(true);
        con = txObject.getConnectionHolder().getConnection();

        Integer previousIsolationLevel = 
                DataSourceUtils.prepareConnectionForTransaction(con, definition);
        txObject.setPreviousIsolationLevel(previousIsolationLevel);
        txObject.setReadOnly(definition.isReadOnly());

        if (con.getAutoCommit()) {
            txObject.setMustRestoreAutoCommit(true);
            con.setAutoCommit(false);
        }

        prepareTransactionalConnection(con, definition);
        txObject.getConnectionHolder().setTransactionActive(true);

        int timeout = determineTimeout(definition);
        if (timeout != TransactionDefinition.TIMEOUT_DEFAULT) {
            txObject.getConnectionHolder().setTimeoutInSeconds(timeout);
        }

        // Bind the connection holder to the thread.
        if (txObject.isNewConnectionHolder()) {
            TransactionSynchronizationManager.bindResource(obtainDataSource(), 
                txObject.getConnectionHolder());
        }
    } catch (Throwable ex) {
        if (txObject.isNewConnectionHolder()) {
            DataSourceUtils.releaseConnection(con, obtainDataSource());
            txObject.setConnectionHolder(null, false);
        }
        throw new CannotCreateTransactionException("...");
    }
}

```

7. 对于 `PROPAGATION_SUPPORTS` || `PROPAGATION_NOT_SUPPORTED` || `PROPAGATION_NEVER` 创建空事务, 也就是没事务

## 5.2 PlatformTransactionManager 的 commit(TransactionStatus) 方法

这个方法会判断该事务是否应该 commit, 对于应该 rollback 的情况, 这个方法也会调用 `processRollback` 方法进行回滚

```java
// AbstractPlatformTransactionManager 
@Override
public final void commit(TransactionStatus status) throws TransactionException {
    // 1)
    if (status.isCompleted()) {
        throw new IllegalTransactionStateException("...");
    }
    // 2)
    DefaultTransactionStatus defStatus = (DefaultTransactionStatus) status;
    if (defStatus.isLocalRollbackOnly()) {
        processRollback(defStatus, false);
        return;
    }
    if (!shouldCommitOnGlobalRollbackOnly() && defStatus.isGlobalRollbackOnly()) {
        processRollback(defStatus, true);
        return;
    }
    // 3)
    processCommit(defStatus);
}
```
1. 事务已完成不能再次提交
2. 如果被标记为 `rollbackOnly`, 调用 `processRollback` 方法回滚
    - 首先我们这里可以看到很多关于事务提交的回调方法, 如 `triggerBeforeCompletion` 和 `triggerAfterCompletion` 方法, 实际调用的都是保存在 `TransactionSynchronizationManager` 中 `ThreadLocal` 的生命周期对象, 具体的类是 `TransactionSynchronization`.
    - `doRollback` 方法才是真正回滚的方法, 该方法由子类实现, 对于 `DataSourceTransactionManager` 来说就是单纯的 `Connection#rollback()` 方法的调用
    
```java
// AbstractPlatformTransactionManager
private void processRollback(DefaultTransactionStatus status, boolean unexpected) {
    try {
        boolean unexpectedRollback = unexpected;

        try {
            triggerBeforeCompletion(status);

            if (status.hasSavepoint()) {
                status.rollbackToHeldSavepoint();
            } else if (status.isNewTransaction()) {
                doRollback(status);
            } else {
                // Participating in larger transaction
                if (status.hasTransaction()) {
                    if (status.isLocalRollbackOnly() || isGlobalRollbackOnParticipationFailure()) {
                        doSetRollbackOnly(status);
                    }
                }
                // Unexpected rollback only matters here if we're asked to fail early
                if (!isFailEarlyOnGlobalRollbackOnly()) {
                    unexpectedRollback = false;
                }
            }
        } catch (RuntimeException | Error ex) {
            triggerAfterCompletion(status, TransactionSynchronization.STATUS_UNKNOWN);
            throw ex;
        }

        triggerAfterCompletion(status, TransactionSynchronization.STATUS_ROLLED_BACK);

        // Raise UnexpectedRollbackException if we had a global rollback-only marker
        if (unexpectedRollback) {
            throw new UnexpectedRollbackException("...");
        }
    } finally {
        cleanupAfterCompletion(status);
    }
}

// DataSourceTransactionManager
protected void doRollback(DefaultTransactionStatus status) {
    DataSourceTransactionObject txObject = (DataSourceTransactionObject) status.getTransaction();
    Connection con = txObject.getConnectionHolder().getConnection();
    try {
        con.rollback();
    } catch (SQLException ex) {
        throw translateException("JDBC rollback", ex);
    }
}
```

3. 调用 `processCommit`  提交事务
    - 同样的这里也设计对 `TransactionSynchronization` 对象生命周期方法的调用
    - 核心仍然在子类的 `doCommit` 方法中, 对于 `DataSourceTransactionManager` 来说也就是 `Connection#commit()` 方法的调用

```java
// AbstractPlatformTransactionManager
private void processCommit(DefaultTransactionStatus status) throws TransactionException {
    try {
        boolean beforeCompletionInvoked = false;

        try {
            boolean unexpectedRollback = false;
            prepareForCommit(status);
            triggerBeforeCommit(status);
            triggerBeforeCompletion(status);
            beforeCompletionInvoked = true;

            if (status.hasSavepoint()) {
                unexpectedRollback = status.isGlobalRollbackOnly();
                status.releaseHeldSavepoint();
            } else if (status.isNewTransaction()) {
                unexpectedRollback = status.isGlobalRollbackOnly();
                doCommit(status);
            } else if (isFailEarlyOnGlobalRollbackOnly()) {
                unexpectedRollback = status.isGlobalRollbackOnly();
            }

            if (unexpectedRollback) {
                throw new UnexpectedRollbackException("...");
            }
        } catch (UnexpectedRollbackException ex) {
            triggerAfterCompletion(status, TransactionSynchronization.STATUS_ROLLED_BACK);
            throw ex;
        } catch (TransactionException ex) {
            if (isRollbackOnCommitFailure()) {
                doRollbackOnCommitException(status, ex);
            } else {
                triggerAfterCompletion(status, TransactionSynchronization.STATUS_UNKNOWN);
            }
            throw ex;
        } catch (RuntimeException | Error ex) {
            if (!beforeCompletionInvoked) {
                triggerBeforeCompletion(status);
            }
            doRollbackOnCommitException(status, ex);
            throw ex;
        }

        try {
            triggerAfterCommit(status);
        } finally {
            triggerAfterCompletion(status, TransactionSynchronization.STATUS_COMMITTED);
        }
    } finally {
        cleanupAfterCompletion(status);
    }
}

// DataSourceTransactionManager
protected void doCommit(DefaultTransactionStatus status) {
    DataSourceTransactionObject txObject = (DataSourceTransactionObject) status.getTransaction();
    Connection con = txObject.getConnectionHolder().getConnection();
    try {
        con.commit();
    } catch (SQLException ex) {
        throw translateException("JDBC commit", ex);
    }
}
```

## 5.3 PlatformTransactionManager 的 rollback(TransactionStatus) 方法

这个方法的回滚, 实质上也是调用 `processRollback` 方法

```java
// AbstractPlatformTransactionManager 
@Override
public final void rollback(TransactionStatus status) throws TransactionException {
    if (status.isCompleted()) {
        throw new IllegalTransactionStateException("...");
    }

    DefaultTransactionStatus defStatus = (DefaultTransactionStatus) status;
    processRollback(defStatus, false);
}
```

# 6. 关键类

## 6.1 TransactionAspectSupport.TransactionInfo 

```java
protected static final class TransactionInfo {

    @Nullable
    private final PlatformTransactionManager transactionManager;

    @Nullable
    private final TransactionAttribute transactionAttribute;

    private final String joinpointIdentification;

    @Nullable
    private TransactionStatus transactionStatus;

    @Nullable
    private TransactionInfo oldTransactionInfo;
}
```
## 6.2 DefaultTransactionStatus 和 TransactionStatus 

```java
public class DefaultTransactionStatus extends AbstractTransactionStatus {

	@Nullable
	private final Object transaction;

	private final boolean newTransaction;

	private final boolean newSynchronization;

	private final boolean readOnly;

	private final boolean debug;

	@Nullable
	private final Object suspendedResources;

}

public abstract class AbstractTransactionStatus implements TransactionStatus {

	private boolean rollbackOnly = false;

	private boolean completed = false;

	@Nullable
	private Object savepoint;

}
```

## 6.3 TransactionSynchronizationManager 

```java
public abstract class TransactionSynchronizationManager {

    // DataSource to ConnectionHolder / SqlSessionFactory to SqlSessionHolder
	private static final ThreadLocal<Map<Object, Object>> resources =
			new NamedThreadLocal<>("Transactional resources");

	private static final ThreadLocal<Set<TransactionSynchronization>> synchronizations =
			new NamedThreadLocal<>("Transaction synchronizations");

	private static final ThreadLocal<String> currentTransactionName =
			new NamedThreadLocal<>("Current transaction name");

	private static final ThreadLocal<Boolean> currentTransactionReadOnly =
			new NamedThreadLocal<>("Current transaction read-only status");

	private static final ThreadLocal<Integer> currentTransactionIsolationLevel =
			new NamedThreadLocal<>("Current transaction isolation level");

	private static final ThreadLocal<Boolean> actualTransactionActive =
			new NamedThreadLocal<>("Actual transaction active");

}
```

## 6.4 TransactionSynchronization

```java
public interface TransactionSynchronization extends Ordered, Flushable {

	@Override
	default int getOrder() {
		return Ordered.LOWEST_PRECEDENCE;
	}

	default void suspend() {
	}

	default void resume() {
	}

	@Override
	default void flush() {
	}

	default void beforeCommit(boolean readOnly) {
	}

	default void beforeCompletion() {
	}

	default void afterCommit() {
	}

	default void afterCompletion(int status) {
	}
}
```

## 6.4 DataSourceTransactionObject (所谓的 Object transaction)

```java
private static class DataSourceTransactionObject extends JdbcTransactionObjectSupport {

    private boolean newConnectionHolder;

    private boolean mustRestoreAutoCommit;

    // inherited from JdbcTransactionObjectSupport 
	@Nullable
	private ConnectionHolder connectionHolder;
}
```

