# SpringBoot事务管理@Transactional注解原理

## 1、依赖包

### 1.1、 SpringBoot中的依赖包

众所周知，在SpringBoot中凡是需要跟数据库打交道的，基本上都要显式或者隐式添加jdbc的依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
```

### 1.2、 Spring中的依赖包

在讲SpringBoot中的事务管理之前，先来讲下Spring的事务管理。在Spring项目中，加入的是spring-jdbc依赖：

```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-jdbc</artifactId>
</dependency>
```



#### 1.2.1、配置版事务

在使用配置文件的方式中，通常会在Spring的配置文件中配置事务管理器，并注入数据源：

```xml
<!-- 注册数据源 -->
<bean id="dataSource" class="...">
	<property name="" value=""/>
</bean>

<!-- 注册事务管理器 -->
<bean id="txManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="dataSource" />
</bean>

<!-- 开启事务注解 -->
<tx:annotation-driven transaction-manager="txManager" />
```

接下来可以直接在业务层Service的方法上或者类上添加**@Transactional**。



#### 1.2.2、注解版事务

首先需要注册两个Bean，分别对应上面Spring配置文件中的两个Bean：

```java
@Configuration
public class TxConfig {

    @Bean
    public DataSource dataSource() {
        ComboPooledDataSource dataSource = new ComboPooledDataSource();
        dataSource.setUser("...");
        dataSource.setPassword("...");
        dataSource.setDriverClass("...");
        dataSource.setJdbcUrl("...");
        return dataSource;
    }
  
    @Bean
    public PlatformTransactionManager platformTransactionManager() {
		return new DataSourceTransactionManager(dataSource());//放入数据源
    }
}
```

==**PlatformTransactionManager**这个Bean非常重要，要使用事务管理，就必须要在IOC容器中注册一个事务管理器，而在使用**@Transactional**注解的时候，默认会从IOC容器中查找对应的事务管理器。==

一般人都会认为到这已经结束了，可以正常使用**@Transactional**注解了，正常则commit，异常则rollback。各位可以简单尝试在Service层的方法中手动制造异常，看看事务是否会回滚？此处留给大家自行尝试...



其实到这，事务在遇到异常后，还是没有正常回滚，为什么呢？缺少一个注解**@EnableTransactionManagement**，该注解的功能是**==开启基于注解的事务管理功能==**，需要在配置类上添加该注解即可，这样**@Transactional**的事务提交和回滚就会生效。

```java
@EnableTransactionManagement//重要
@Configuration
public class TxConfig {

    @Bean
    public DataSource dataSource() {
        ComboPooledDataSource dataSource = new ComboPooledDataSource();
        dataSource.setUser("...");
        dataSource.setPassword("...");
        dataSource.setDriverClass("...");
        dataSource.setJdbcUrl("...");
        return dataSource;
    }
  
   	//重要
    @Bean
    public PlatformTransactionManager platformTransactionManager() {
		return new DataSourceTransactionManager(dataSource());//放入数据源
    }
}
```



## 2、@Transactional原理

这里咱们再回到SpringBoot的事务管理。在大多数SpringBoot项目中，简单地只要在配置类或者主类上添加**@EnableTransactionManagement**，并在业务层Service上添加**@Transactional**即可实现事务的提交和回滚。

因为在依赖jdbc或者jpa之后，会自动配置**TransactionManager**。

* 依赖jdbc会自动配置**DataSourceTransactionManager**：

```java
@Configuration
@ConditionalOnClass({ JdbcTemplate.class, PlatformTransactionManager.class })
@AutoConfigureOrder(Ordered.LOWEST_PRECEDENCE)
@EnableConfigurationProperties(DataSourceProperties.class)
public class DataSourceTransactionManagerAutoConfiguration {

	@Configuration
	@ConditionalOnSingleCandidate(DataSource.class)
	static class DataSourceTransactionManagerConfiguration {

		private final DataSource dataSource;

		private final TransactionManagerCustomizers transactionManagerCustomizers;

		DataSourceTransactionManagerConfiguration(DataSource dataSource,
				ObjectProvider<TransactionManagerCustomizers> transactionManagerCustomizers) {
			this.dataSource = dataSource;
			this.transactionManagerCustomizers = transactionManagerCustomizers
					.getIfAvailable();
		}

		@Bean
		@ConditionalOnMissingBean(PlatformTransactionManager.class)
		public DataSourceTransactionManager transactionManager(
				DataSourceProperties properties) {
			DataSourceTransactionManager transactionManager = new DataSourceTransactionManager(
					this.dataSource);
			if (this.transactionManagerCustomizers != null) {
				this.transactionManagerCustomizers.customize(transactionManager);
			}
			return transactionManager;
		}

	}

}
```

* 依赖jpa会自动配置**JpaTransactionManager**:

```java
@EnableConfigurationProperties(JpaProperties.class)
@Import(DataSourceInitializedPublisher.Registrar.class)
public abstract class JpaBaseConfiguration implements BeanFactoryAware {
    
    //other code...
    
	@Bean
	@ConditionalOnMissingBean(PlatformTransactionManager.class)
	public PlatformTransactionManager transactionManager() {
		JpaTransactionManager transactionManager = new JpaTransactionManager();
		if (this.transactionManagerCustomizers != null) {
			this.transactionManagerCustomizers.customize(transactionManager);
		}
		return transactionManager;
	}
}
```



### 2.1、**@EnableTransactionManagement**

具体原理还要从注解**@EnableTransactionManagement**说起：

```java
@Import(TransactionManagementConfigurationSelector.class)
public @interface EnableTransactionManagement {
	boolean proxyTargetClass() default false;
    
    AdviceMode mode() default AdviceMode.PROXY;
    
    int order() default Ordered.LOWEST_PRECEDENCE;
}
```

从**@EnableTransactionManagement**签名上可以看到，它导入了**TransactionManagementConfigurationSelector**类，其作用就是利用该类想容器中导入组件。而**TransactionManagementConfigurationSelector**主要向容器中导入了两个组件，分别是**AutoProxyRegistrar**和**ProxyTransactionManagementConfiguration**：

```java
public class TransactionManagementConfigurationSelector extends AdviceModeImportSelector<EnableTransactionManagement> {

	/**
	 * {@inheritDoc}
	 * @return {@link ProxyTransactionManagementConfiguration} or
	 * {@code AspectJTransactionManagementConfiguration} for {@code PROXY} and
	 * {@code ASPECTJ} values of {@link EnableTransactionManagement#mode()}, respectively
	 */
	@Override
	protected String[] selectImports(AdviceMode adviceMode) {
		switch (adviceMode) {
			case PROXY:
				return new String[] {AutoProxyRegistrar.class.getName(), ProxyTransactionManagementConfiguration.class.getName()};
			case ASPECTJ:
				return new String[] {TransactionManagementConfigUtils.TRANSACTION_ASPECT_CONFIGURATION_CLASS_NAME};
			default:
				return null;
		}
	}
}
```



### 2.2、AutoProxyRegistrar

咱们先来看看**AutoProxyRegistrar**做了哪些事？

```java
public class AutoProxyRegistrar implements ImportBeanDefinitionRegistrar {

	@Override
	public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
		boolean candidateFound = false;
		Set<String> annoTypes = importingClassMetadata.getAnnotationTypes();
		for (String annoType : annoTypes) {
			AnnotationAttributes candidate = AnnotationConfigUtils.attributesFor(importingClassMetadata, annoType);
			if (candidate == null) {
				continue;
			}
			Object mode = candidate.get("mode");
			Object proxyTargetClass = candidate.get("proxyTargetClass");
			if (mode != null && proxyTargetClass != null && AdviceMode.class == mode.getClass() &&
					Boolean.class == proxyTargetClass.getClass()) {
				candidateFound = true;
				if (mode == AdviceMode.PROXY) {
                    //注册自动代理创建器
					AopConfigUtils.registerAutoProxyCreatorIfNecessary(registry);
					if ((Boolean) proxyTargetClass) {
						AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
						return;
					}
				}
			}
		}
		//other code...
	}

}
```

由于默认情况下mode为**AdviceMode.PROXY**，所以会通过AopConfigUtils.registerAutoProxyCreatorIfNecessary方法向容器中注册自动代理创建器：

```java
//AopConfigUtils工具类

public static final String AUTO_PROXY_CREATOR_BEAN_NAME =
			"org.springframework.aop.config.internalAutoProxyCreator";

public static BeanDefinition registerAutoProxyCreatorIfNecessary(BeanDefinitionRegistry registry) {
    return registerAutoProxyCreatorIfNecessary(registry, null);
}

public static BeanDefinition registerAutoProxyCreatorIfNecessary(BeanDefinitionRegistry registry, Object source) {
    return registerOrEscalateApcAsRequired(InfrastructureAdvisorAutoProxyCreator.class, registry, source);
}

private static BeanDefinition registerOrEscalateApcAsRequired(Class<?> cls, BeanDefinitionRegistry registry, Object source) {
    Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
    if (registry.containsBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME)) {
        BeanDefinition apcDefinition = registry.getBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME);
        if (!cls.getName().equals(apcDefinition.getBeanClassName())) {
            int currentPriority = findPriorityForClass(apcDefinition.getBeanClassName());
            int requiredPriority = findPriorityForClass(cls);
            if (currentPriority < requiredPriority) {
                apcDefinition.setBeanClassName(cls.getName());
            }
        }
        return null;
    }
    RootBeanDefinition beanDefinition = new RootBeanDefinition(cls);
    beanDefinition.setSource(source);
    beanDefinition.getPropertyValues().add("order", Ordered.HIGHEST_PRECEDENCE);
    beanDefinition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
    registry.registerBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME, beanDefinition);
    return beanDefinition;
}
```

可以看到会向容器中注册了一个名为**internalAutoProxyCreator**，类型为**InfrastructureAdvisorAutoProxyCreator**的组件Bean，而这个类其实同**AnnotationAwareAspectJAutoProxyCreator**类似，也是**AbstractAdvisorAutoProxyCreator**的实现类，而它的顶层接口类其实就是**BeanPostProcessor**，所以它的逻辑跟**AnnotationAwareAspectJAutoProxyCreator**大体上一样：**==利用后置处理器机制在对象创建之后，并包装成代理对象，在代理对象执行目标方法的时候利用拦截器链进行拦截==**，具体过程可以参考AOP原理。

====================================================================================================================================================================================================================================================================================================================================================================================



### 2.3、ProxyTransactionManagementConfiguration

另一个组件为**ProxyTransactionManagementConfiguration**：

```java
@Configuration
public class ProxyTransactionManagementConfiguration extends AbstractTransactionManagementConfiguration {

	@Bean(name = TransactionManagementConfigUtils.TRANSACTION_ADVISOR_BEAN_NAME)
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public BeanFactoryTransactionAttributeSourceAdvisor transactionAdvisor() {
		BeanFactoryTransactionAttributeSourceAdvisor advisor = new BeanFactoryTransactionAttributeSourceAdvisor();
		advisor.setTransactionAttributeSource(transactionAttributeSource());
		advisor.setAdvice(transactionInterceptor());
		advisor.setOrder(this.enableTx.<Integer>getNumber("order"));
		return advisor;
	}

	@Bean
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public TransactionAttributeSource transactionAttributeSource() {
		return new AnnotationTransactionAttributeSource();
	}

	@Bean
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public TransactionInterceptor transactionInterceptor() {
		TransactionInterceptor interceptor = new TransactionInterceptor();
		interceptor.setTransactionAttributeSource(transactionAttributeSource());
		if (this.txManager != null) {
			interceptor.setTransactionManager(this.txManager);
		}
		return interceptor;
	}

}
```

**ProxyTransactionManagementConfiguration**给容器中注入了多个组件Bean，其中：

- **transactionAdvisor**这个Bean用于创建一个事务增强器。
- **transactionAttributeSource**这个Bean主要保存事务属性，其中包括用于解析不同包下的**@Transactional**注解的解析器：

```java
public AnnotationTransactionAttributeSource() {
    this(true);
}

public AnnotationTransactionAttributeSource(boolean publicMethodsOnly) {
    this.publicMethodsOnly = publicMethodsOnly;
    this.annotationParsers = new LinkedHashSet<TransactionAnnotationParser>(2);
    this.annotationParsers.add(new SpringTransactionAnnotationParser());
    if (jta12Present) {
        this.annotationParsers.add(new JtaTransactionAnnotationParser());
    }
    if (ejb3Present) {
        this.annotationParsers.add(new Ejb3TransactionAnnotationParser());
    }
}
```

其中包括Spring中的@Transactional注解，同时也包括了Jta和Ejb3中的@Transactional注解解析器。



- **transactionInterceptor**这个Bean主要用于创建事务拦截器，其中设置上面所说到的事务属性Bean，同时也保存了事务管理器。**TransactionInterceptor**其实也是**MethodInterceptor**的子类。所以在代理对象执行目标方法的时候，方法拦截器就会进行拦截并工作。具体如何拦截工作，请继续往下看**invoke**方法：

```java
@Override
public Object invoke(final MethodInvocation invocation) throws Throwable {
    // Work out the target class: may be {@code null}.
    // The TransactionAttributeSource should be passed the target class
    // as well as the method, which may be from an interface.
    Class<?> targetClass = (invocation.getThis() != null ? AopUtils.getTargetClass(invocation.getThis()) : null);

    // Adapt to TransactionAspectSupport's invokeWithinTransaction...
    return invokeWithinTransaction(invocation.getMethod(), targetClass, new InvocationCallback() {
        @Override
        public Object proceedWithInvocation() throws Throwable {
            return invocation.proceed();
        }
    });
}
```

重点就在**invokeWithinTransaction**方法:

```java
protected Object invokeWithinTransaction(Method method, Class<?> targetClass, final InvocationCallback invocation)
			throws Throwable {

    // If the transaction attribute is null, the method is non-transactional.
    //获取事务注解属性
    final TransactionAttribute txAttr = getTransactionAttributeSource().getTransactionAttribute(method, targetClass);
    //获取事务管理器
    final PlatformTransactionManager tm = determineTransactionManager(txAttr);
    final String joinpointIdentification = methodIdentification(method, targetClass, txAttr);

    if (txAttr == null || !(tm instanceof CallbackPreferringPlatformTransactionManager)) {
        // Standard transaction demarcation with getTransaction and commit/rollback calls.
        //
        TransactionInfo txInfo = createTransactionIfNecessary(tm, txAttr, joinpointIdentification);
        Object retVal = null;
        try {
            // This is an around advice: Invoke the next interceptor in the chain.
            // This will normally result in a target object being invoked.
            retVal = invocation.proceedWithInvocation();
        }
        catch (Throwable ex) {
            // target invocation exception
            completeTransactionAfterThrowing(txInfo, ex);
            throw ex;
        }
        finally {
            cleanupTransactionInfo(txInfo);
        }
        commitTransactionAfterReturning(txInfo);
        return retVal;
    }

    else {
        // It's a CallbackPreferringPlatformTransactionManager: pass a TransactionCallback in.
        try {
            Object result = ((CallbackPreferringPlatformTransactionManager) tm).execute(txAttr, new TransactionCallback<Object>() {
 	  @Override                                                                            public Object doInTransaction(TransactionStatus status) {                               TransactionInfo txInfo = prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);
          try {
	              return invocation.proceedWithInvocation();
          } catch (Throwable ex) {
              if (txAttr.rollbackOn(ex)) {
              // A RuntimeException: will lead to a rollback.
              	if (ex instanceof RuntimeException) {
                   throw (RuntimeException) ex;
                } else {
               		throw new ThrowableHolderException(ex);
                }
              }else {
                  // A normal return value: will lead to a commit.
                  return new ThrowableHolder(ex);
              }
          } finally {
              cleanupTransactionInfo(txInfo); 
          }                                                                                }                                                                                });
            // Check result: It might indicate a Throwable to rethrow.
            if (result instanceof ThrowableHolder) {
                throw ((ThrowableHolder) result).getThrowable();
            }
            else {
                return result;
            }
        }
        catch (ThrowableHolderException ex) {
            throw ex.getCause();
        }
    }
}
```

其实大家看这段源码就能看到：**1、获取事务相关属性；2、获取事务管理器；3、如果需要事务支持，则创建一个事务；4、执行目标方法；5、如果执行正常，则进行事务提交；6、如果执行异常，则进行事务回滚。**