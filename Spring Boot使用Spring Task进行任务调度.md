### 1、Spring中的定时任务

#### 1.1、使用xml形式

任务类MyScheduler:

```java
public class MyScheduler {

	public void print(){
		System.out.println("MyScheduler：" + LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss:SSS")));
	}
}
```

spring配置spring-task.xml:

```xml
<task:annotation-driven/>
	
<bean id="taskScheduler" class="org.com.cay.scheduler.MyScheduler"/>

<task:scheduled-tasks>
    <task:scheduled ref="taskScheduler" method="print" cron="0/5 * * * * ?"/>
</task:scheduled-tasks>
```

运行项目即可实现每隔5s会输出一次信息。



#### 1.2、使用注解形式

```java
@Component
public class OtherScheduler {

	@Scheduled(cron="0/10 * * * * ?")
	public void show(){
		System.out.println("OtherScheduler：" + LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss:SSS")));
	}
}
```

运行项目即可实现每隔10s会输出一次信息。



### 2、task命名空间

**task**命名空间使用 **TaskNamespaceHandler** 来处理:

```java
//{@code NamespaceHandler} for the 'task' namespace.
public class TaskNamespaceHandler extends NamespaceHandlerSupport {

	@Override
	public void init() {
		this.registerBeanDefinitionParser("annotation-driven", new AnnotationDrivenBeanDefinitionParser());
		this.registerBeanDefinitionParser("executor", new ExecutorBeanDefinitionParser());
		this.registerBeanDefinitionParser("scheduled-tasks", new ScheduledTasksBeanDefinitionParser());
		this.registerBeanDefinitionParser("scheduler", new SchedulerBeanDefinitionParser());
	}
}
```

而其中：

* **AnnotationDrivenBeanDefinitionParser**用于解析**<task:annotation-driven />**标签：

  ```xml
  <task:annotation-driven scheduler="" exception-handler="" executor="" mode="" proxy-target-class=""/>
  ```

  ​

* **SchedulerBeanDefinitionParser**用于解析**<task:scheduler />**标签:

  ```xml
  <task:scheduler id="调度器id" pool-size="线程池大小" />
  ```

  ​

* **ScheduledTasksBeanDefinitionParser**用于解析**<task:scheduled-tasks />**标签:

  ```xml
  <task:scheduled-tasks scheduler="指定使用调度器的id">
          <task:scheduled ref="任务bean" method="任务bean的方法" cron="" fixed-delay="" fixed-rate="" initial-delay="" trigger=""/>
  </task:scheduled-tasks>
  ```

  ​

* **ExecutorBeanDefinitionParser**用于解析**<task:executor />**标签:

  ```xml
  <task:executor id="" pool-size="" keep-alive="" queue-capacity="" rejection-policy="" />
  ```

  ​

### 3、源码分析

#### 3.1、SchedulerBeanDefinitionParser：

直接看类定义：

```java
public class SchedulerBeanDefinitionParser extends AbstractSingleBeanDefinitionParser {

	@Override
	protected String getBeanClassName(Element element) {
		return "org.springframework.scheduling.concurrent.ThreadPoolTaskScheduler";
	}

	@Override
	protected void doParse(Element element, BeanDefinitionBuilder builder) {
		String poolSize = element.getAttribute("pool-size");
		if (StringUtils.hasText(poolSize)) {
			builder.addPropertyValue("poolSize", poolSize);
		}
	}

}
```

可以想象，一旦配置了

```xml
<task:scheduler id="调度器id" pool-size="线程池大小" />
```

，就会向Spring容器中注册一个Bean，id为指定的id值，值为**org.springframework.scheduling.concurrent.ThreadPoolTaskScheduler**类的实例。

至于我为什么会先讲这个类，后面会解释。



#### 3.2、ExecutorBeanDefinitionParser：

直接看类定义：

```java
public class ExecutorBeanDefinitionParser extends AbstractSingleBeanDefinitionParser {

	@Override
	protected String getBeanClassName(Element element) {
		return "org.springframework.scheduling.config.TaskExecutorFactoryBean";
	}

	@Override
	protected void doParse(Element element, ParserContext parserContext, BeanDefinitionBuilder builder) {
		String keepAliveSeconds = element.getAttribute("keep-alive");
		if (StringUtils.hasText(keepAliveSeconds)) {
			builder.addPropertyValue("keepAliveSeconds", keepAliveSeconds);
		}
		String queueCapacity = element.getAttribute("queue-capacity");
		if (StringUtils.hasText(queueCapacity)) {
			builder.addPropertyValue("queueCapacity", queueCapacity);
		}
		configureRejectionPolicy(element, builder);
		String poolSize = element.getAttribute("pool-size");
		if (StringUtils.hasText(poolSize)) {
			builder.addPropertyValue("poolSize", poolSize);
		}
	}
    
    //configureRejectionPolicy函数...
}
```

同上节一样，一旦配置了

```xml
<task:executor id="" pool-size="" keep-alive="" queue-capacity="" rejection-policy="" />
```

，就会向Spring容器中注册一个Bean，值为**org.springframework.scheduling.config.TaskExecutorFactoryBean**类的实例对象。



#### 3.3、**AnnotationDrivenBeanDefinitionParser**：

该类用于解析使用注解形式的任务调度，看源码（部分）：

```java
public class AnnotationDrivenBeanDefinitionParser implements BeanDefinitionParser {
    
    private static final String ASYNC_EXECUTION_ASPECT_CLASS_NAME =
			"org.springframework.scheduling.aspectj.AnnotationAsyncExecutionAspect";

	@Override
	public BeanDefinition parse(Element element, ParserContext parserContext) {
		Object source = parserContext.extractSource(element);

		// Register component for the surrounding <task:annotation-driven> element.
		CompositeComponentDefinition compDefinition = new CompositeComponentDefinition(element.getTagName(), source);
		parserContext.pushContainingComponent(compDefinition);

		// Nest the concrete post-processor bean in the surrounding component.
		BeanDefinitionRegistry registry = parserContext.getRegistry();

		String mode = element.getAttribute("mode");
		if ("aspectj".equals(mode)) {
			// mode="aspectj"
			registerAsyncExecutionAspect(element, parserContext);
		}
		else {
			// mode="proxy"
			if (registry.containsBeanDefinition(TaskManagementConfigUtils.ASYNC_ANNOTATION_PROCESSOR_BEAN_NAME)) {
				parserContext.getReaderContext().error(
						"Only one AsyncAnnotationBeanPostProcessor may exist within the context.", source);
			}
			else {
				BeanDefinitionBuilder builder = BeanDefinitionBuilder.genericBeanDefinition(
						"org.springframework.scheduling.annotation.AsyncAnnotationBeanPostProcessor");
				builder.getRawBeanDefinition().setSource(source);
				String executor = element.getAttribute("executor");
				if (StringUtils.hasText(executor)) {
					builder.addPropertyReference("executor", executor);
				}
				String exceptionHandler = element.getAttribute("exception-handler");
				if (StringUtils.hasText(exceptionHandler)) {
					builder.addPropertyReference("exceptionHandler", exceptionHandler);
				}
				if (Boolean.valueOf(element.getAttribute(AopNamespaceUtils.PROXY_TARGET_CLASS_ATTRIBUTE))) {
					builder.addPropertyValue("proxyTargetClass", true);
				}
				registerPostProcessor(parserContext, builder, TaskManagementConfigUtils.ASYNC_ANNOTATION_PROCESSOR_BEAN_NAME);
			}
		}

		if (registry.containsBeanDefinition(TaskManagementConfigUtils.SCHEDULED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
			parserContext.getReaderContext().error(
					"Only one ScheduledAnnotationBeanPostProcessor may exist within the context.", source);
		}
		else {
			BeanDefinitionBuilder builder = BeanDefinitionBuilder.genericBeanDefinition(
					"org.springframework.scheduling.annotation.ScheduledAnnotationBeanPostProcessor");
			builder.getRawBeanDefinition().setSource(source);
			String scheduler = element.getAttribute("scheduler");
			if (StringUtils.hasText(scheduler)) {
				builder.addPropertyReference("scheduler", scheduler);
			}
			registerPostProcessor(parserContext, builder, TaskManagementConfigUtils.SCHEDULED_ANNOTATION_PROCESSOR_BEAN_NAME);
		}

		// Finally register the composite component.
		parserContext.popAndRegisterContainingComponent();

		return null;
	}
    
    //other code...
    
}
```

从**AnnotationDrivenBeanDefinitionParserd**源码中可以看到，它注册了两个后置处理器，分别是：

==**org.springframework.scheduling.annotation.AsyncAnnotationBeanPostProcessor**==

==**org.springframework.scheduling.annotation.ScheduledAnnotationBeanPostProcessor**==

前者用于异步调用的处理器，而后者用于任务调度的处理器。

本文主要讲解任务调度，所以来看**ScheduledAnnotationBeanPostProcessor**处理器。

首先来看看**ScheduledAnnotationBeanPostProcessor**类定义：

```java
public class ScheduledAnnotationBeanPostProcessor
		implements MergedBeanDefinitionPostProcessor, DestructionAwareBeanPostProcessor,
		Ordered, EmbeddedValueResolverAware, BeanNameAware, BeanFactoryAware, ApplicationContextAware,
		SmartInitializingSingleton, ApplicationListener<ContextRefreshedEvent>, DisposableBean {}
```

从类定义来看，**ScheduledAnnotationBeanPostProcessor**实现了**ApplicationListener**接口，用于监听**ContextRefreshedEvent**事件。

先来看初始化后的后置处理方法**ScheduledAnnotationBeanPostProcessor#postProcessAfterInitialization**方法：

```java
@Override
public Object postProcessAfterInitialization(final Object bean, String beanName) {
    Class<?> targetClass = AopUtils.getTargetClass(bean);
    if (!this.nonAnnotatedClasses.contains(targetClass)) {
        //首先获取所有带有注解（@Scheduled和@Schedules）的方法
        Map<Method, Set<Scheduled>> annotatedMethods = MethodIntrospector.selectMethods(targetClass, 
new MethodIntrospector.MetadataLookup<Set<Scheduled>>() {
            @Override                                                                                     public Set<Scheduled> inspect(Method method) {
                Set<Scheduled> scheduledMethods = AnnotatedElementUtils.getMergedRepeatableAnnotations(method, Scheduled.class, Schedules.class);
                                                                                                            return (!scheduledMethods.isEmpty() ? scheduledMethods : null);
            }
		});
        //如果注解方法集合为空
        if (annotatedMethods.isEmpty()) {
            this.nonAnnotatedClasses.add(targetClass);
            if (logger.isTraceEnabled()) {
                logger.trace("No @Scheduled annotations found on bean class: " + bean.getClass());
            }
        }
        else {
            //带有注解的方法集合不为空时
            for (Map.Entry<Method, Set<Scheduled>> entry : annotatedMethods.entrySet()) {
                Method method = entry.getKey();
                for (Scheduled scheduled : entry.getValue()) {
                    //依次遍历处理注解方法，重点
                    processScheduled(scheduled, method, bean);
                }
            }
            if (logger.isDebugEnabled()) {
                logger.debug(annotatedMethods.size() + " @Scheduled methods processed on bean '" + beanName +
                             "': " + annotatedMethods);
            }
        }
    }
    return bean;
}
```

再来看处理定时任务的方法**ScheduledAnnotationBeanPostProcessor#processScheduled**方法：

```java
protected void processScheduled(Scheduled scheduled, Method method, Object bean) {
    try {
        Assert.isTrue(method.getParameterTypes().length == 0,
                      "Only no-arg methods may be annotated with @Scheduled");

        Method invocableMethod = AopUtils.selectInvocableMethod(method, bean.getClass());
        //创建ScheduledMethodRunnable(父接口Runnable)对象，主要是用来反射定时方法
        Runnable runnable = new ScheduledMethodRunnable(bean, invocableMethod);
        boolean processedSchedule = false;
        String errorMessage =
            "Exactly one of the 'cron', 'fixedDelay(String)', or 'fixedRate(String)' attributes is required";

        // 初始化一个定时任务集合 
        Set<ScheduledTask> tasks = new LinkedHashSet<ScheduledTask>(4);

        // Determine initial delay
        //判断注解属性中是否带有initialDelay和initialDelayString
        long initialDelay = scheduled.initialDelay();
        String initialDelayString = scheduled.initialDelayString();
        if (StringUtils.hasText(initialDelayString)) {
            Assert.isTrue(initialDelay < 0, "Specify 'initialDelay' or 'initialDelayString', not both");
            if (this.embeddedValueResolver != null) {
                initialDelayString = this.embeddedValueResolver.resolveStringValue(initialDelayString);
            }
            try {
                initialDelay = Long.parseLong(initialDelayString);
            }
            catch (NumberFormatException ex) {
                throw new IllegalArgumentException(
                    "Invalid initialDelayString value \"" + initialDelayString + "\" - cannot parse into integer");
            }
        }

        // Check cron expression
        //判断是否是cron表达式
        String cron = scheduled.cron();
        if (StringUtils.hasText(cron)) {
            Assert.isTrue(initialDelay == -1, "'initialDelay' not supported for cron triggers");
            processedSchedule = true;
            //获取注解属性中的zone时区值
            String zone = scheduled.zone();
            if (this.embeddedValueResolver != null) {
                cron = this.embeddedValueResolver.resolveStringValue(cron);
                zone = this.embeddedValueResolver.resolveStringValue(zone);
            }
            TimeZone timeZone;
            if (StringUtils.hasText(zone)) {
                //解析时区
                timeZone = StringUtils.parseTimeZoneString(zone);
            }
            else {
                //创建默认时区
                timeZone = TimeZone.getDefault();
            }
            //将表达式解析成cron任务计划添加到任务集合中
            //调用ScheduledTaskRegistrar对象的scheduleCronTask(CronTask cronTask)方法
            tasks.add(this.registrar.scheduleCronTask(new CronTask(runnable, new CronTrigger(cron, timeZone))));
        }

        // At this point we don't need to differentiate between initial delay set or not anymore
        if (initialDelay < 0) {
            initialDelay = 0;
        }

        // Check fixed delay
        //判断是否有fixedDelay属性
        long fixedDelay = scheduled.fixedDelay();
        if (fixedDelay >= 0) {
            Assert.isTrue(!processedSchedule, errorMessage);
            processedSchedule = true;
            //根据属性值创建Interval任务然后添加到任务集合中
            //调用ScheduledTaskRegistrar对象的scheduleFixedDelayTask(IntervalTask task)方法
            tasks.add(this.registrar.scheduleFixedDelayTask(new IntervalTask(runnable, fixedDelay, initialDelay)));
        }
        //判断是否有fixedDelayString属性
        String fixedDelayString = scheduled.fixedDelayString();
        if (StringUtils.hasText(fixedDelayString)) {
            Assert.isTrue(!processedSchedule, errorMessage);
            processedSchedule = true;
            if (this.embeddedValueResolver != null) {
                fixedDelayString = this.embeddedValueResolver.resolveStringValue(fixedDelayString);
            }
            try {
                fixedDelay = Long.parseLong(fixedDelayString);
            }
            catch (NumberFormatException ex) {
                throw new IllegalArgumentException(
                    "Invalid fixedDelayString value \"" + fixedDelayString + "\" - cannot parse into integer");
            }
            //根据属性值创建Interval任务然后添加到任务集合中
            //调用ScheduledTaskRegistrar对象的scheduleFixedDelayTask(IntervalTask task)方法
            tasks.add(this.registrar.scheduleFixedDelayTask(new IntervalTask(runnable, fixedDelay, initialDelay)));
        }

        // Check fixed rate
        //判断是否有fixedRate属性
        long fixedRate = scheduled.fixedRate();
        if (fixedRate >= 0) {
            Assert.isTrue(!processedSchedule, errorMessage);
            processedSchedule = true;
            //根据属性值创建Interval任务然后添加到任务集合中
            //调用ScheduledTaskRegistrar对象的scheduleFixedRateTask(IntervalTask task)方法
            tasks.add(this.registrar.scheduleFixedRateTask(new IntervalTask(runnable, fixedRate, initialDelay)));
        }
        //判断是否有fixedRateString属性
        String fixedRateString = scheduled.fixedRateString();
        if (StringUtils.hasText(fixedRateString)) {
            Assert.isTrue(!processedSchedule, errorMessage);
            processedSchedule = true;
            if (this.embeddedValueResolver != null) {
                fixedRateString = this.embeddedValueResolver.resolveStringValue(fixedRateString);
            }
            try {
                fixedRate = Long.parseLong(fixedRateString);
            }
            catch (NumberFormatException ex) {
                throw new IllegalArgumentException(
                    "Invalid fixedRateString value \"" + fixedRateString + "\" - cannot parse into integer");
            }
            //根据属性值创建Interval任务然后添加到任务集合中
            //调用ScheduledTaskRegistrar对象的scheduleFixedRateTask(IntervalTask task)方法
            tasks.add(this.registrar.scheduleFixedRateTask(new IntervalTask(runnable, fixedRate, initialDelay)));
        }

        // Check whether we had any attribute set
        Assert.isTrue(processedSchedule, errorMessage);

        // Finally register the scheduled tasks
        synchronized (this.scheduledTasks) {
            Set<ScheduledTask> registeredTasks = this.scheduledTasks.get(bean);
            if (registeredTasks == null) {
                registeredTasks = new LinkedHashSet<ScheduledTask>(4);
                this.scheduledTasks.put(bean, registeredTasks);
            }
            registeredTasks.addAll(tasks);
        }
    }
    catch (IllegalArgumentException ex) {
        throw new IllegalStateException(
            "Encountered invalid @Scheduled method '" + method.getName() + "': " + ex.getMessage());
    }
}
```

上述源码中讲到了**ScheduledMethodRunnable**类，该类实现了Runnable接口：

```java
public class ScheduledMethodRunnable implements Runnable {

	private final Object target;
	private final Method method;

    //constructor and getter/setter...

	@Override
	public void run() {
		try {
            //定时任务的反射机制
			ReflectionUtils.makeAccessible(this.method);
			this.method.invoke(this.target);
		}
		catch (InvocationTargetException ex) {
			ReflectionUtils.rethrowRuntimeException(ex.getTargetException());
		}
		catch (IllegalAccessException ex) {
			throw new UndeclaredThrowableException(ex);
		}
	}
}
```

在**processScheduled**方法中，对于每种不同属性的情况，会创建不同类型的Task。此处我们以**CronTask**为例，其余会在后续中提到，接下来咱们来看**ScheduledTaskRegistrar#scheduleCronTask**方法：

```java
public ScheduledTask scheduleCronTask(CronTask task) {
   ScheduledTask scheduledTask = this.unresolvedTasks.remove(task);
   boolean newTask = false;
   if (scheduledTask == null) {
      scheduledTask = new ScheduledTask();
      newTask = true;
   }
    //重点
   if (this.taskScheduler != null) {
      scheduledTask.future = this.taskScheduler.schedule(task.getRunnable(), task.getTrigger());
   }
   else {
      addCronTask(task);
      this.unresolvedTasks.put(task, scheduledTask);
   }
   return (newTask ? scheduledTask : null);
}
```

因为**AnnotationDrivenBeanDefinitionParser**是基于注解形式，无法在注解上指定使用的TaskScheduler任务调度器，所以在上述源码中有taskScheduler变量，在Spring启动过程中，父接口**ApplicationListener**在Spring容器未完全启动之前，即在**ContextRefreshedEvent**事件未发布之前，该taskScheduler变量仍为null值。



当**ContextRefreshedEvent**事件发布后，监听到**ContextRefreshedEvent**事件，则会触发方法：

```java
@Override
public void onApplicationEvent(ContextRefreshedEvent event) {
    if (event.getApplicationContext() == this.applicationContext) {
        //完成注册后
        finishRegistration();
    }
}
```

看**ScheduledAnnotationBeanPostProcessor#finishRegistration**方法:

```java
private void finishRegistration() {
    if (this.scheduler != null) {
        this.registrar.setScheduler(this.scheduler);
    }

    if (this.beanFactory instanceof ListableBeanFactory) {
        Map<String, SchedulingConfigurer> beans =
            ((ListableBeanFactory) this.beanFactory).getBeansOfType(SchedulingConfigurer.class);
        List<SchedulingConfigurer> configurers = new ArrayList<SchedulingConfigurer>(beans.values());
        AnnotationAwareOrderComparator.sort(configurers);
        for (SchedulingConfigurer configurer : configurers) {
            configurer.configureTasks(this.registrar);
        }
    }

    //如果有任务，但是scheduler任务调度器为null时
    if (this.registrar.hasTasks() && this.registrar.getScheduler() == null) {
        Assert.state(this.beanFactory != null, "BeanFactory must be set to find scheduler by type");
        try {
            // 开始查找TaskScheduler类型的Bean
            this.registrar.setTaskScheduler(resolveSchedulerBean(TaskScheduler.class, false));
        }
        catch (NoUniqueBeanDefinitionException ex) {
            logger.debug("Could not find unique TaskScheduler bean", ex);
            try {
                //根据名字和类来查找
                this.registrar.setTaskScheduler(resolveSchedulerBean(TaskScheduler.class, true));
            }
            catch (NoSuchBeanDefinitionException ex2) {
                //logger
            }
        }
        catch (NoSuchBeanDefinitionException ex) {
            logger.debug("Could not find default TaskScheduler bean", ex);
            // Search for ScheduledExecutorService bean next...
            try {
                // 如果没有TaskScheduler类型的Bean，则查找ScheduledExecutorService类型的Bean
                this.registrar.setScheduler(resolveSchedulerBean(ScheduledExecutorService.class, false));
            }
            catch (NoUniqueBeanDefinitionException ex2) {
                logger.debug("Could not find unique ScheduledExecutorService bean", ex2);
                try {
                    //根据名字和类来查找
                    this.registrar.setScheduler(resolveSchedulerBean(ScheduledExecutorService.class, true));
                }
                catch (NoSuchBeanDefinitionException ex3) {
                    //logger
                }
            }
            catch (NoSuchBeanDefinitionException ex2) {
                //logger
            }
        }
    }

    this.registrar.afterPropertiesSet();
}
```

从上面源码中，会从BeanFactory中试图获取**TaskScheduler**或**ScheduledExecutorService**的Bean。

```java
public static final String DEFAULT_TASK_SCHEDULER_BEAN_NAME = "taskScheduler";

private <T> T resolveSchedulerBean(Class<T> schedulerType, boolean byName) {
   if (byName) {
       //根据name和类来查找
      T scheduler = this.beanFactory.getBean(DEFAULT_TASK_SCHEDULER_BEAN_NAME, schedulerType);
      if (this.beanFactory instanceof ConfigurableBeanFactory) {
         ((ConfigurableBeanFactory) this.beanFactory).registerDependentBean(
               DEFAULT_TASK_SCHEDULER_BEAN_NAME, this.beanName);
      }
      return scheduler;
   }
    //根据类来查找
   else if (this.beanFactory instanceof AutowireCapableBeanFactory) {
      NamedBeanHolder<T> holder = ((AutowireCapableBeanFactory) this.beanFactory).resolveNamedBean(schedulerType);
      if (this.beanFactory instanceof ConfigurableBeanFactory) {
         ((ConfigurableBeanFactory) this.beanFactory).registerDependentBean(
               holder.getBeanName(), this.beanName);
      }
      return holder.getBeanInstance();
   }
   else {
      return this.beanFactory.getBean(schedulerType);
   }
}
```

这里就要解释下在==第3.1、SchedulerBeanDefinitionParser小节==留下的疑惑：

* ==如果配置了<task:scheduler id="调度器id" pool-size="线程池大小" />，就会注册一个Bean，该Bean的值为**org.springframework.scheduling.concurrent.ThreadPoolTaskScheduler**类的实例，而这个类又实现了**TaskScheduler**接口。所以如果配置了，就能从BeanFactory中获取该Bean的值，然后使用**ScheduledTaskRegistrar**的**setTaskScheduler**方法设置TaskScheduler对象的值，此时taskScheduler就不再为null了，而是**ThreadPoolTaskScheduler**类型的任务调度器。==
* ==如果未配置<task:scheduler id="调度器id" pool-size="线程池大小" />，则无法从BeanFactory中获取对应class或者name的Bean，此时taskScheduler仍为null。==



接着看**ScheduledTaskRegistrar#afterPropertiesSet和scheduleTasks**方法：

```java
@Override
public void afterPropertiesSet() {
    scheduleTasks();
}

protected void scheduleTasks() {
    //判断taskScheduler是否为null，如果配置了<task:scheduler id="" pool-size=""/>标签，则taskScheduler就不为null了，直接跳过创建调度器的代码
    if (this.taskScheduler == null) {
        //则创建一个单线程的线程池，和一个ConcurrentTaskScheduler类型的任务调度器
        this.localExecutor = Executors.newSingleThreadScheduledExecutor();
        this.taskScheduler = new ConcurrentTaskScheduler(this.localExecutor);
    }
    //将任务添加到任务调度器中进行执行
    if (this.triggerTasks != null) {
        for (TriggerTask task : this.triggerTasks) {
            addScheduledTask(scheduleTriggerTask(task));
        }
    }
    if (this.cronTasks != null) {
        for (CronTask task : this.cronTasks) {
            addScheduledTask(scheduleCronTask(task));
        }
    }
    if (this.fixedRateTasks != null) {
        for (IntervalTask task : this.fixedRateTasks) {
            addScheduledTask(scheduleFixedRateTask(task));
        }
    }
    if (this.fixedDelayTasks != null) {
        for (IntervalTask task : this.fixedDelayTasks) {
            addScheduledTask(scheduleFixedDelayTask(task));
        }
    }
}

private void addScheduledTask(ScheduledTask task) {
    if (task != null) {
        this.scheduledTasks.add(task);
    }
}
```

又回归到了**ScheduledTaskRegistrar#scheduleCronTask**方法，在进入**scheduleTasks**方法之后，会先判断taskScheduler是否为null值，如果是null，则创建一个**ConcurrentTaskScheduler**类型的TaskScheduler；如果不为null(即**ThreadPoolTaskScheduler**类似)，则跳过创建过程。所以在**scheduleCronTask**方法中的taskScheduler已经不为null了，所以可以执行调度器scheduler的schedule方法：

```java
public ScheduledTask scheduleCronTask(CronTask task) {
    ScheduledTask scheduledTask = this.unresolvedTasks.remove(task);
    boolean newTask = false;
    if (scheduledTask == null) {
        scheduledTask = new ScheduledTask();
        newTask = true;
    }
    //此时taskScheduler不为null
    if (this.taskScheduler != null) {
        //执行TaskScheduler的schedule方法
        scheduledTask.future = this.taskScheduler.schedule(task.getRunnable(), task.getTrigger());
    }
    else {
        addCronTask(task);
        this.unresolvedTasks.put(task, scheduledTask);
    }
    return (newTask ? scheduledTask : null);
}
```

**TaskScheduler**是个接口，主要有**ThreadPoolTaskScheduler**和**ConcurrentTaskScheduler**两个实现类。

看**ThreadPoolTaskScheduler#schedule**方法：

```java
@Override
public ScheduledFuture<?> schedule(Runnable task, Trigger trigger) {
    ScheduledExecutorService executor = getScheduledExecutor();
    try {
        ErrorHandler errorHandler =
            (this.errorHandler != null ? this.errorHandler : TaskUtils.getDefaultErrorHandler(true));
        return new ReschedulingRunnable(task, trigger, executor, errorHandler).schedule();
    }
    catch (RejectedExecutionException ex) {
        throw new TaskRejectedException("Executor [" + executor + "] did not accept task: " + task, ex);
    }
}
```

再看**ConcurrentTaskScheduler#schedule**方法：

```java
@Override
public ScheduledFuture<?> schedule(Runnable task, Trigger trigger) {
    try {
        if (this.enterpriseConcurrentScheduler) {
            return new EnterpriseConcurrentTriggerScheduler().schedule(decorateTask(task, true), trigger);
        }
        else {
            ErrorHandler errorHandler =
                (this.errorHandler != null ? this.errorHandler : TaskUtils.getDefaultErrorHandler(true));
            //调用ReschedulingRunnable对象的schedule方法，触发run方法
            return new ReschedulingRunnable(task, trigger, this.scheduledExecutor, errorHandler).schedule();
        }
    }
    catch (RejectedExecutionException ex) {
        throw new TaskRejectedException("Executor [" + this.scheduledExecutor + "] did not accept task: " + task, ex);
    }
}
```

其实两者类似，最后都是调用了**ReschedulingRunnable**类的schedule方法，而schedule方法会触发Runnable接口实现类的run方法的执行，从而导致自定义的任务方法触发执行。

来看看**ReschedulingRunnable**类的构造函数和执行方法：

```java
class ReschedulingRunnable extends DelegatingErrorHandlingRunnable implements ScheduledFuture<Object> {
    //构造函数
    public ReschedulingRunnable(Runnable delegate, Trigger trigger, ScheduledExecutorService executor, ErrorHandler errorHandler) {
        super(delegate, errorHandler);
        this.trigger = trigger;
        this.executor = executor;
    }

    public ScheduledFuture<?> schedule() {
        synchronized (this.triggerContextMonitor) {
            //计算下一次执行时间
            this.scheduledExecutionTime = this.trigger.nextExecutionTime(this.triggerContext);
            if (this.scheduledExecutionTime == null) {
                return null;
            }
            long initialDelay = this.scheduledExecutionTime.getTime() - System.currentTimeMillis();
            this.currentFuture = this.executor.schedule(this, initialDelay, TimeUnit.MILLISECONDS);
            return this;
        }
    }

    @Override
    public void run() {
        Date actualExecutionTime = new Date();
        //调用父类的run方法
        super.run();
        Date completionTime = new Date();
        synchronized (this.triggerContextMonitor) {
            this.triggerContext.update(this.scheduledExecutionTime, actualExecutionTime, completionTime);
            if (!this.currentFuture.isCancelled()) {
                // 继续无限调度
                schedule();
            }
        }
    }
}
```

super.run();会调用父类**DelegatingErrorHandlingRunnable**的run方法，父类run方法中最重要的一句：**this.delegate.run();**即调用Runnable的run方法，而此时Runnable对象实际上是上述**ScheduledMethodRunnable**对象，其中封装了自定义的任务方法。最后归根结底，就是通过一层层封装来执行自定义的任务方法。



#### 3.4、ScheduledTasksBeanDefinitionParser

该类用于解析使用xml配置形式的任务调度，看源码（部分）：

```java
public class ScheduledTasksBeanDefinitionParser extends AbstractSingleBeanDefinitionParser {
    
    @Override
	protected String getBeanClassName(Element element) {
		return "org.springframework.scheduling.config.ContextLifecycleScheduledTaskRegistrar";
	}
    
    //other code...
    @Override
	protected void doParse(Element element, ParserContext parserContext, BeanDefinitionBuilder builder) {
		builder.setLazyInit(false); // lazy scheduled tasks are a contradiction in terms -> force to false
        //创建四种集合，分别用来管理四种不同参数或者属性的Task任务
		ManagedList<RuntimeBeanReference> cronTaskList = new ManagedList<RuntimeBeanReference>();
		ManagedList<RuntimeBeanReference> fixedDelayTaskList = new ManagedList<RuntimeBeanReference>();
		ManagedList<RuntimeBeanReference> fixedRateTaskList = new ManagedList<RuntimeBeanReference>();
		ManagedList<RuntimeBeanReference> triggerTaskList = new ManagedList<RuntimeBeanReference>();
        //解析xml节点
		NodeList childNodes = element.getChildNodes();
		for (int i = 0; i < childNodes.getLength(); i++) {
			Node child = childNodes.item(i);
            //判断是否是<task:scheduled />节点
			if (!isScheduledElement(child, parserContext)) {
				continue;
			}
			Element taskElement = (Element) child;
            //获取节点属性
			String ref = taskElement.getAttribute("ref");
			String method = taskElement.getAttribute("method");

			// Check that 'ref' and 'method' are specified
			if (!StringUtils.hasText(ref) || !StringUtils.hasText(method)) {
				parserContext.getReaderContext().error("Both 'ref' and 'method' are required", taskElement);
				// Continue with the possible next task element
				continue;
			}

            //获取节点属性值
			String cronAttribute = taskElement.getAttribute("cron");
			String fixedDelayAttribute = taskElement.getAttribute("fixed-delay");
			String fixedRateAttribute = taskElement.getAttribute("fixed-rate");
			String triggerAttribute = taskElement.getAttribute("trigger");
			String initialDelayAttribute = taskElement.getAttribute("initial-delay");

			boolean hasCronAttribute = StringUtils.hasText(cronAttribute);
			boolean hasFixedDelayAttribute = StringUtils.hasText(fixedDelayAttribute);
			boolean hasFixedRateAttribute = StringUtils.hasText(fixedRateAttribute);
			boolean hasTriggerAttribute = StringUtils.hasText(triggerAttribute);
			boolean hasInitialDelayAttribute = StringUtils.hasText(initialDelayAttribute);

			if (!(hasCronAttribute || hasFixedDelayAttribute || hasFixedRateAttribute || hasTriggerAttribute)) {
				parserContext.getReaderContext().error(
						"one of the 'cron', 'fixed-delay', 'fixed-rate', or 'trigger' attributes is required", taskElement);
				continue; // with the possible next task element
			}

			if (hasInitialDelayAttribute && (hasCronAttribute || hasTriggerAttribute)) {
				parserContext.getReaderContext().error(
						"the 'initial-delay' attribute may not be used with cron and trigger tasks", taskElement);
				continue; // with the possible next task element
			}

            //创建Runnable对象，并返回该对象的name，见下文#1
			String runnableName =
					runnableReference(ref, method, taskElement, parserContext).getBeanName();

            //以下四种类型的Task，将不同属性和Runnable对象实例绑定对应的Task类上，见下文#2以CronTask为例
			if (hasFixedDelayAttribute) {
				fixedDelayTaskList.add(intervalTaskReference(runnableName,
						initialDelayAttribute, fixedDelayAttribute, taskElement, parserContext));
			}
			if (hasFixedRateAttribute) {
				fixedRateTaskList.add(intervalTaskReference(runnableName,
						initialDelayAttribute, fixedRateAttribute, taskElement, parserContext));
			}
			if (hasCronAttribute) {
				cronTaskList.add(cronTaskReference(runnableName, cronAttribute,
						taskElement, parserContext));
			}
			if (hasTriggerAttribute) {
				String triggerName = new RuntimeBeanReference(triggerAttribute).getBeanName();
				triggerTaskList.add(triggerTaskReference(runnableName, triggerName,
						taskElement, parserContext));
			}
		}
		String schedulerRef = element.getAttribute("scheduler");
		if (StringUtils.hasText(schedulerRef)) {
            //如果有节点上scheduler属性，则进行绑定
			builder.addPropertyReference("taskScheduler", schedulerRef);
		}
		builder.addPropertyValue("cronTasksList", cronTaskList);
		builder.addPropertyValue("fixedDelayTasksList", fixedDelayTaskList);
		builder.addPropertyValue("fixedRateTasksList", fixedRateTaskList);
		builder.addPropertyValue("triggerTasksList", triggerTaskList);
	}
    
    //other code...
}
```

使用**ScheduledTasksBeanDefinitionParser**解析器，会创建一个类型为**org.springframework.scheduling.config.ContextLifecycleScheduledTaskRegistrar**的Bean。



**doParse**源码的#1处

**ScheduledTasksBeanDefinitionParser#runnableReference**方法：

```java
private RuntimeBeanReference runnableReference(String ref, String method, Element taskElement, ParserContext parserContext) {
    BeanDefinitionBuilder builder = BeanDefinitionBuilder.genericBeanDefinition(
        "org.springframework.scheduling.support.ScheduledMethodRunnable");
    builder.addConstructorArgReference(ref);
    builder.addConstructorArgValue(method);
    return beanReference(taskElement, parserContext, builder);
}
```

再看看**ScheduledTasksBeanDefinitionParser#beanReference**方法定义：

```java
private RuntimeBeanReference beanReference(Element taskElement,
			ParserContext parserContext, BeanDefinitionBuilder builder) {
    // Extract the source of the current task
    builder.getRawBeanDefinition().setSource(parserContext.extractSource(taskElement));
    String generatedName = parserContext.getReaderContext().generateBeanName(builder.getRawBeanDefinition());
    parserContext.registerBeanComponent(new BeanComponentDefinition(builder.getBeanDefinition(), generatedName));
    return new RuntimeBeanReference(generatedName);
}
```

上述源码中可以看出，它声明了一个**ScheduledMethodRunnable**对象的实例，并把任务类和任务的方法绑定到该对象上，并进行注册。



**doParse**源码的#2处，以**CronTask**为例：

**ScheduledTasksBeanDefinitionParser#cronTaskReference**方法：

```java
private RuntimeBeanReference cronTaskReference(String runnableBeanName,
			String cronExpression, Element taskElement, ParserContext parserContext) {
    BeanDefinitionBuilder builder = BeanDefinitionBuilder.genericBeanDefinition(
        "org.springframework.scheduling.config.CronTask");
    builder.addConstructorArgReference(runnableBeanName);
    builder.addConstructorArgValue(cronExpression);
    return beanReference(taskElement, parserContext, builder);
}
```

同**runnableReference**方法一样，在**cronTaskReference**方法中，同样声明了一个**CronTask**对象实例，然后将Runnable对象的name以及cron表达式绑定到**CronTask**对象实例上，并进行注册。

最后将所有的任务加入到不同的任务集合中，并把任务集合绑定到类型为**org.springframework.scheduling.config.ContextLifecycleScheduledTaskRegistrar**的Bean上。



看看**ContextLifecycleScheduledTaskRegistrar**的类定义:

```java
public class ContextLifecycleScheduledTaskRegistrar extends ScheduledTaskRegistrar implements SmartInitializingSingleton {

	@Override
	public void afterPropertiesSet() {
		// no-op
	}

	@Override
	public void afterSingletonsInstantiated() {
		scheduleTasks();
	}

}
```

在处理**ContextLifecycleScheduledTaskRegistrar**的Bean的时候，会触发父类的**scheduleTasks**方法。

此时又回归到了**ScheduledTaskRegistrar**的**scheduleTasks**方法了。

**ScheduledTaskRegistrar#scheduleTasks**方法：

```java
protected void scheduleTasks() {
    if (this.taskScheduler == null) {
        this.localExecutor = Executors.newSingleThreadScheduledExecutor();
        this.taskScheduler = new ConcurrentTaskScheduler(this.localExecutor);
    }
    if (this.triggerTasks != null) {
        for (TriggerTask task : this.triggerTasks) {
            addScheduledTask(scheduleTriggerTask(task));
        }
    }
    if (this.cronTasks != null) {
        for (CronTask task : this.cronTasks) {
            addScheduledTask(scheduleCronTask(task));
        }
    }
    if (this.fixedRateTasks != null) {
        for (IntervalTask task : this.fixedRateTasks) {
            addScheduledTask(scheduleFixedRateTask(task));
        }
    }
    if (this.fixedDelayTasks != null) {
        for (IntervalTask task : this.fixedDelayTasks) {
            addScheduledTask(scheduleFixedDelayTask(task));
        }
    }
}
```

在**ScheduledTasksBeanDefinitionParser**的**doParse**方法中有几行代码：

```java
String schedulerRef = element.getAttribute("scheduler");
if (StringUtils.hasText(schedulerRef)) {
    //如果有节点上scheduler属性，则进行绑定
    builder.addPropertyReference("taskScheduler", schedulerRef);
}
```

正如我注释所言：

* ==如果节点上指定了scheduler属性，则会把scheduler的引用Bean绑定到**ContextLifecycleScheduledTaskRegistrar(ScheduledTaskRegistrar子类)**类实例上，则父类**ScheduledTaskRegistrar**的taskScheduler的值就会赋值为指定的scheduler任务调度器的值，此时的taskScheduler通常为**ThreadPoolTaskScheduler**类型。==
* ==如果节点上未指定scheduler属性，即使配置了**<task:scheduler id="" pool-size="" />**，也不会把当前任务绑定到之前配置的scheduler上去，因为此时taskScheduler为null，它会自己创建一个**ConcurrentTaskScheduler**类型的TaskScheduler，而不是实际配置的**ThreadPoolTaskScheduler**类型的TaskScheduler。==




最后执行的步骤跟使用注解的方式就一样了，这里就不再累赘了。请参看第3.3小节。



### 4、其他

在**ScheduledTaskRegistrar**类中定义了根据三种不同类型的Task执行的四个方法：

Task类型有：

- **CronTask**
- **TriggerTask**
- **IntervalTask**

方法：

- **ScheduledTask scheduleCronTask(CronTask task);**
  - ScheduledFuture<?> schedule(Runnable task, Trigger trigger);
  - ScheduledFuture<?> schedule(Runnable task, Date startTime);
- **ScheduledTask scheduleTriggerTask(TriggerTask task);**
  - ScheduledFuture<?> schedule(Runnable task, Trigger trigger);
  - ScheduledFuture<?> schedule(Runnable task, Date startTime);
- **ScheduledTask scheduleFixedRateTask(IntervalTask task);**
  - ScheduledFuture<?> scheduleAtFixedRate(Runnable task, long period);
  - ScheduledFuture<?> scheduleAtFixedRate(Runnable task, Date startTime, long period);
- **ScheduledTask scheduleFixedDelayTask(IntervalTask task);**
  - ScheduledFuture<?> scheduleWithFixedDelay(Runnable task, long delay);
  - ScheduledFuture<?> scheduleWithFixedDelay(Runnable task, Date startTime, long delay);

其中重载方法取决于有没有设置initial-delay属性。

对于**schedule**的两个重载方法，

* 如果设置了initial-delay属性，则执行**ScheduledExecutorService**对象的**schedule**方法；
* 如果未设置则执行**ReschedulingRunnable**对象的**schedule**方法

对于另外四个方法，都是执行**ScheduledExecutorService**各自的scheduleXxx同名方法。



### 5、Spring Boot中使用Schedule

只要在主入口类上加入**@EnableScheduling** 即可启动任务调度了。

```java
@Import(SchedulingConfiguration.class)
public @interface EnableScheduling {}
```

其中导入了**SchedulingConfiguration**类。

```java
@Configuration
@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
public class SchedulingConfiguration {

	@Bean(name = TaskManagementConfigUtils.SCHEDULED_ANNOTATION_PROCESSOR_BEAN_NAME)
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public ScheduledAnnotationBeanPostProcessor scheduledAnnotationProcessor() {
		return new ScheduledAnnotationBeanPostProcessor();
	}
}
```

在**SchedulingConfiguration**类中，创建了一个**ScheduledAnnotationBeanPostProcessor**的Bean。看过上文的小伙伴肯定就熟悉这个类了，在讲解使用注解的时候，在AnnotationDrivenBeanDefinitionParser解析<task:annotation-driven />的类中会创建一个**ScheduledAnnotationBeanPostProcessor**的Bean，后续的过程就同使用注解的方式一样了。