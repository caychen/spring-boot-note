在上一节中，我们简单的讲述了jpa的查询语法和使用教程，而这一节咱们来看看Spring Boot中对jpa是如何进行自动配置的。

​    

在Spring Boot自动配置的时候，一旦引入**spring-boot-starter-data-jpa**`，就会完成`**JpaRepositoriesAutoConfiguration**的自动配置。

**JpaRepositoriesAutoConfiguration**

```java
@Configuration
@ConditionalOnBean(DataSource.class)
@ConditionalOnClass(JpaRepository.class)
@ConditionalOnMissingBean({ JpaRepositoryFactoryBean.class,
		JpaRepositoryConfigExtension.class })
@ConditionalOnProperty(prefix = "spring.data.jpa.repositories", name = "enabled", havingValue = "true", matchIfMissing = true)
@Import(JpaRepositoriesAutoConfigureRegistrar.class)
@AutoConfigureAfter(HibernateJpaAutoConfiguration.class)
public class JpaRepositoriesAutoConfiguration {}
```


spring-data-jpa底层使用的是**Hibernate**作为实现，所以jpa的自动配置操作在Hibernate的自动配置之后。



**@AutoConfigureAfter**：表示在指定类完成后再进行自动配置，所以来看**HibernateJpaAutoConfiguration**源码。

咱们继续：

**HibernateJpaAutoConfiguration**

```java
@Configuration
@ConditionalOnClass({ LocalContainerEntityManagerFactoryBean.class, EntityManager.class })
@Conditional(HibernateEntityManagerCondition.class)
@AutoConfigureAfter({ DataSourceAutoConfiguration.class })
public class HibernateJpaAutoConfiguration extends JpaBaseConfiguration {

	//other...
    
	public HibernateJpaAutoConfiguration(DataSource dataSource,
			JpaProperties jpaProperties,
			ObjectProvider<JtaTransactionManager> jtaTransactionManager,
			ObjectProvider<TransactionManagerCustomizers> transactionManagerCustomizers) {
        //调用父类的构造方法
		super(dataSource, jpaProperties, jtaTransactionManager,
				transactionManagerCustomizers);
	}

	@Override
	protected AbstractJpaVendorAdapter createJpaVendorAdapter() {
		return new HibernateJpaVendorAdapter();
	}

	@Override
	protected Map<String, Object> getVendorProperties() {
		Map<String, Object> vendorProperties = new LinkedHashMap<String, Object>();
		vendorProperties.putAll(getProperties().getHibernateProperties(getDataSource()));
		return vendorProperties;
	}
    
    //other...
}
```




同样在**HibernateJpaAutoConfiguration**的源码中表示该自动配置需要在**DataSourceAutoConfiguration**完成后再进行，之前在分析**JdbcTemplateAutoConfiguration**的源码的时候已经分析过**DataSourceAutoConfiguration**，此处就不再讲述，一笔带过。

​    回想之前在使用Spring和JPA集成的时候，会配置一个**jpaVendorAdapter**属性，一般使用**HibernateJpaVendorAdapter**作为JPA持久化实现厂商类。如下是spring和jpa集成时的部分配置：

```xml
<property name="jpaVendorAdapter">
    <bean id="hibernateJpaVendorAdapter" class="org.springframework.orm.jpa.vendor.HibernateJpaVendorAdapter">
        <!-- 该bean的属性配置 -->
    </bean>
</property>
```


​    在**JpaBaseConfiguration**类中有个**createJpaVendorAdapter()**抽象方法，而在**HibernateJpaAutoConfiguration**类中进行了重载，创建使用**HibernateJpaVendorAdapter**作为JPA底层持久化实现厂商。



咱们来看父类的源码：



**父类JpaBaseConfiguration**

```java
@EnableConfigurationProperties(JpaProperties.class)
@Import(DataSourceInitializedPublisher.Registrar.class)
public abstract class JpaBaseConfiguration implements BeanFactoryAware {

	private final DataSource dataSource;

	private final JpaProperties properties;

	private final JtaTransactionManager jtaTransactionManager;

	private final TransactionManagerCustomizers transactionManagerCustomizers;

	private ConfigurableListableBeanFactory beanFactory;

	protected JpaBaseConfiguration(DataSource dataSource, JpaProperties properties,
			ObjectProvider<JtaTransactionManager> jtaTransactionManager,
			ObjectProvider<TransactionManagerCustomizers> transactionManagerCustomizers) {
		this.dataSource = dataSource;
		this.properties = properties;
		this.jtaTransactionManager = jtaTransactionManager.getIfAvailable();
		this.transactionManagerCustomizers = transactionManagerCustomizers
				.getIfAvailable();
	}

    //创建了TransactionManager的Bean
	@Bean
	@ConditionalOnMissingBean(PlatformTransactionManager.class)
	public PlatformTransactionManager transactionManager() {
		JpaTransactionManager transactionManager = new JpaTransactionManager();
		if (this.transactionManagerCustomizers != null) {
			this.transactionManagerCustomizers.customize(transactionManager);
		}
		return transactionManager;
	}

    //创建了jpaVendorAdapter适配器，并填充相应属性，最后返回
	@Bean
	@ConditionalOnMissingBean
	public JpaVendorAdapter jpaVendorAdapter() {
        //通过HibernateJpaAutoConfiguration子类创建HibernateJpaVendorAdapter适配器
		AbstractJpaVendorAdapter adapter = createJpaVendorAdapter();
		adapter.setShowSql(this.properties.isShowSql());
		adapter.setDatabase(this.properties.determineDatabase(this.dataSource));
		adapter.setDatabasePlatform(this.properties.getDatabasePlatform());
		adapter.setGenerateDdl(this.properties.isGenerateDdl());
		return adapter;
	}

    //通过jpaVendorAdapter与其他配置信息创建Builder构建器
	@Bean
	@ConditionalOnMissingBean
	public EntityManagerFactoryBuilder entityManagerFactoryBuilder(
			JpaVendorAdapter jpaVendorAdapter,
			ObjectProvider<PersistenceUnitManager> persistenceUnitManager) {
		EntityManagerFactoryBuilder builder = new EntityManagerFactoryBuilder(
				jpaVendorAdapter, this.properties.getProperties(),
				persistenceUnitManager.getIfAvailable());
		builder.setCallback(getVendorCallback());
		return builder;
	}

    //创建LocalContainerEntityManagerFactoryBean的Bean，用于JPA的容器管理EntityManagerFactory
	@Bean
	@Primary
	@ConditionalOnMissingBean({ LocalContainerEntityManagerFactoryBean.class,
			EntityManagerFactory.class })
	public LocalContainerEntityManagerFactoryBean entityManagerFactory(
			EntityManagerFactoryBuilder factoryBuilder) {
		Map<String, Object> vendorProperties = getVendorProperties();
		customizeVendorProperties(vendorProperties);
		return factoryBuilder.dataSource(this.dataSource).packages(getPackagesToScan())
				.properties(vendorProperties).jta(isJta()).build();
	}

    //other...
}
```


在父类JpaBaseConfiguration中创建了几个重要的Bean，这样创建Bean的过程类似之前spring-jpa集成时使用的xml配置文件：

```html
<bean id="dataSource" class="com.mchange.v2.c3p0.ComboPooledDataSource">
    <property name="driverClass" value="${jdbc.driverClass}"></property>
    <property name="jdbcUrl" value="${jdbc.jdbcUrl}"></property>
    <property name="user" value="${jdbc.user}"></property>
    <property name="password" value="${jdbc.password}"></property>
    <property name="initialPoolSize" value="${jdbc.initialPoolSize}"></property>
    <property name="maxPoolSize" value="${jdbc.maxPoolSize}"></property>
</bean>

<!-- 配置jpa的EntityManagerFactory -->
<bean id="entityManagerFactory" class="org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean">
    <property name="dataSource" ref="dataSource"></property>

    <!-- 配置jpa生产商的适配器 -->
    <property name="jpaVendorAdapter">
        <bean class="org.springframework.orm.jpa.vendor.HibernateJpaVendorAdapter"></bean>
    </property>

    <!-- 配置jpa的基本属性 -->
    <property name="jpaProperties">
        <props>
            <prop key="hibernate.show_sql">true</prop>
            <prop key="hibernate.format_sql">true</prop>
            <prop key="hibernate.hbm2ddl.auto">update</prop>
        </props>
    </property>
</bean>

<bean id="transactionManager" class="org.springframework.orm.jpa.JpaTransactionManager">
    <property name="entityManagerFactory" ref="entityManagerFactory"></property>
</bean>

<!-- 配置SpringData -->
<jpa:repositories base-package="xxxxx包名" entity-manager-factory-ref="entityManagerFactory" transaction-manager-ref="transactionManager"></jpa:repositories>
```


通过这几步java配置，基本上就完成了之前spring-jpa集成所需的所有配置信息，当然Spring Boot内部做了很多工作，这里就不再描述了。