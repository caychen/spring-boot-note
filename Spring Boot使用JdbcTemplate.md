## 1、添加数据源

既然要使用JdbcTemplate，就需要添加jdbc的依赖。

```html
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
```



## 2、连接数据源，以mysql为例：

```html
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
</dependency>
```



## 3、在src/main/resources/application.properties中配置数据源信息

注意：其中driver-class可以不写，Spring Boot会自动从url中解析使用的数据源类

Spring Boot默认采用**tomcat-jdbc**连接池，如果需要**C3P0，DBCP，Druid**等作为连接池，需要加入相关依赖以及配置，这里不作说明，采用默认配置即可。

```plain
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
spring.datasource.url=jdbc:mysql://localhost:3306/test
spring.datasource.username=...
spring.datasource.password=...
```



底层默认使用tomcat-jdbc连接池，所以在默认情况下会创建一个基于Tomcat连接池的DataSource，并注入到Spring IOC容器中。

```java
abstract class DataSourceConfiguration {

	@SuppressWarnings("unchecked")
	protected <T> T createDataSource(DataSourceProperties properties,
			Class<? extends DataSource> type) {
		return (T) properties.initializeDataSourceBuilder().type(type).build();
	}

	/**
	 * Tomcat Pool DataSource configuration.
	 */
	@ConditionalOnClass(org.apache.tomcat.jdbc.pool.DataSource.class)
	@ConditionalOnProperty(name = "spring.datasource.type", havingValue = "org.apache.tomcat.jdbc.pool.DataSource", matchIfMissing = true)
	static class Tomcat extends DataSourceConfiguration {

        //创建一个基于Tomcat连接池的DataSource的Bean
        //由于底层使用Tomcat连接池，在不引入外部数据连接池的jar包之前，默认使用Tomcat的DataSource
		@Bean
		@ConfigurationProperties(prefix = "spring.datasource.tomcat")
		public org.apache.tomcat.jdbc.pool.DataSource dataSource(
				DataSourceProperties properties) {
			org.apache.tomcat.jdbc.pool.DataSource dataSource = createDataSource(
					properties, org.apache.tomcat.jdbc.pool.DataSource.class);
			DatabaseDriver databaseDriver = DatabaseDriver
					.fromJdbcUrl(properties.determineUrl());
			String validationQuery = databaseDriver.getValidationQuery();
			if (validationQuery != null) {
				dataSource.setTestOnBorrow(true);
				dataSource.setValidationQuery(validationQuery);
			}
			return dataSource;
		}

	}

	/**
	 * Hikari DataSource configuration.
	 */
	@ConditionalOnClass(HikariDataSource.class)
	@ConditionalOnProperty(name = "spring.datasource.type", havingValue = "com.zaxxer.hikari.HikariDataSource", matchIfMissing = true)
	static class Hikari extends DataSourceConfiguration {

        //创建基于hikari的DataSource的Bean
		@Bean
		@ConfigurationProperties(prefix = "spring.datasource.hikari")
		public HikariDataSource dataSource(DataSourceProperties properties) {
			return createDataSource(properties, HikariDataSource.class);
		}

	}

	/**
	 * DBCP DataSource configuration.
	 *
	 * @deprecated as of 1.5 in favor of DBCP2
	 */
	@ConditionalOnClass(org.apache.commons.dbcp.BasicDataSource.class)
	@ConditionalOnProperty(name = "spring.datasource.type", havingValue = "org.apache.commons.dbcp.BasicDataSource", matchIfMissing = true)
	@Deprecated
	static class Dbcp extends DataSourceConfiguration {

        //创建基于dbcp的DataSource的Bean
		@Bean
		@ConfigurationProperties(prefix = "spring.datasource.dbcp")
		public org.apache.commons.dbcp.BasicDataSource dataSource(
				DataSourceProperties properties) {
			org.apache.commons.dbcp.BasicDataSource dataSource = createDataSource(
					properties, org.apache.commons.dbcp.BasicDataSource.class);
			DatabaseDriver databaseDriver = DatabaseDriver
					.fromJdbcUrl(properties.determineUrl());
			String validationQuery = databaseDriver.getValidationQuery();
			if (validationQuery != null) {
				dataSource.setTestOnBorrow(true);
				dataSource.setValidationQuery(validationQuery);
			}
			return dataSource;
		}

	}

	/**
	 * DBCP DataSource configuration.
	 */
	@ConditionalOnClass(org.apache.commons.dbcp2.BasicDataSource.class)
	@ConditionalOnProperty(name = "spring.datasource.type", havingValue = "org.apache.commons.dbcp2.BasicDataSource", matchIfMissing = true)
	static class Dbcp2 extends DataSourceConfiguration {

        //创建基于dbcp2的DataSource的Bean
		@Bean
		@ConfigurationProperties(prefix = "spring.datasource.dbcp2")
		public org.apache.commons.dbcp2.BasicDataSource dataSource(
				DataSourceProperties properties) {
			return createDataSource(properties,
					org.apache.commons.dbcp2.BasicDataSource.class);
		}
	}
    
   	/**
	 * Generic DataSource configuration.
	 */
	@ConditionalOnMissingBean(DataSource.class)
	@ConditionalOnProperty(name = "spring.datasource.type")
	static class Generic {

        //可以在配置文件中通过spring.datasource.type指定外部的dataSource类型，并创建基于该type的DataSource的Bean
		@Bean
		public DataSource dataSource(DataSourceProperties properties) {
			return properties.initializeDataSourceBuilder().build();
		}

	}
}
```



## 4、使用JdbcTemplate操作数据库

SpringBoot中的 **JdbcTemplate** 是自动配置的，可以直接使用 **@Autowired** 或者 **@Resource** 来注入到需要的类中。

### **JdbcTemplateAutoConfiguration**

```java
@Configuration
@ConditionalOnClass({ DataSource.class, JdbcTemplate.class })
@ConditionalOnSingleCandidate(DataSource.class)
//在DataSourceAutoConfiguration自动配置类完成后再进行自动配置该类JdbcTemplateAutoConfiguration
@AutoConfigureAfter(DataSourceAutoConfiguration.class)
public class JdbcTemplateAutoConfiguration {

	private final DataSource dataSource;

	public JdbcTemplateAutoConfiguration(DataSource dataSource) {
		this.dataSource = dataSource;
	}

    //创建一个JdbcTemplate的Bean
	@Bean
	@Primary
	@ConditionalOnMissingBean(JdbcOperations.class)
	public JdbcTemplate jdbcTemplate() {
		return new JdbcTemplate(this.dataSource);
	}

	@Bean
	@Primary
	@ConditionalOnMissingBean(NamedParameterJdbcOperations.class)
	public NamedParameterJdbcTemplate namedParameterJdbcTemplate() {
		return new NamedParameterJdbcTemplate(this.dataSource);
	}

}
```


在**JdbcTemplateAutoConfiguration**类中会创建一个**JdbcTemplate**的Bean，所以在使用的时候可以直接注入。



### DataSourceAutoConfiguration

```java
@Configuration
@ConditionalOnClass({ DataSource.class, EmbeddedDatabaseType.class })
//启动DataSourceProperties配置类
@EnableConfigurationProperties(DataSourceProperties.class)
@Import({ Registrar.class, DataSourcePoolMetadataProvidersConfiguration.class })
public class DataSourceAutoConfiguration {

	private static final Log logger = LogFactory
			.getLog(DataSourceAutoConfiguration.class);

	@Bean
	@ConditionalOnMissingBean
	public DataSourceInitializer dataSourceInitializer(DataSourceProperties properties,
			ApplicationContext applicationContext) {
        //根据配置文件属性创建DataSource的初始化器
		return new DataSourceInitializer(properties, applicationContext);
	}
    
    //...
}
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)



DataSourceInitializer

```java
class DataSourceInitializer implements ApplicationListener<DataSourceInitializedEvent> {
    private final DataSourceProperties properties;
    private final ApplicationContext applicationContext;
    private DataSource dataSource;
    private boolean initialized = false;
    DataSourceInitializer(DataSourceProperties properties,
            ApplicationContext applicationContext) {
        this.properties = properties;
        this.applicationContext = applicationContext;
    }
    @PostConstruct
    public void init() {
        //当DataSourceProperties初始化完成后
        if (!this.properties.isInitialize()) {
            logger.debug("Initialization disabled (not running DDL scripts)");
            return;
        }
        if (this.applicationContext.getBeanNamesForType(DataSource.class, false,
                false).length > 0) {
            //获取DataSource的bean
            this.dataSource = this.applicationContext.getBean(DataSource.class);
        }
        if (this.dataSource == null) {
            logger.debug("No DataSource found so not initializing");
            return;
        }
        //执行schema脚本
        runSchemaScripts();
    }


    private void runSchemaScripts() {
        //查看全局配置文件中是否有spring.datasource.schema
		List<Resource> scripts = getScripts("spring.datasource.schema",
				this.properties.getSchema(), "schema");
		if (!scripts.isEmpty()) {
			String username = this.properties.getSchemaUsername();
			String password = this.properties.getSchemaPassword();
			runScripts(scripts, username, password);
			try {
				this.applicationContext
						.publishEvent(new DataSourceInitializedEvent(this.dataSource));
				// The listener might not be registered yet, so don't rely on it.
				if (!this.initialized) {
                    //执行数据插入脚本
					runDataScripts();
					this.initialized = true;
				}
			}
			catch (IllegalStateException ex) {
				logger.warn("Could not send event to complete DataSource initialization ("
						+ ex.getMessage() + ")");
			}
		}
	}


	@Override
	public void onApplicationEvent(DataSourceInitializedEvent event) {
		if (!this.properties.isInitialize()) {
			logger.debug("Initialization disabled (not running data scripts)");
			return;
		}
		// NOTE the event can happen more than once and
		// the event datasource is not used here
        //如果初始化完成后，执行数据插入
		if (!this.initialized) {
			runDataScripts();
			this.initialized = true;
		}
	}


	private void runDataScripts() {
		List<Resource> scripts = getScripts("spring.datasource.data",
				this.properties.getData(), "data");
		String username = this.properties.getDataUsername();
		String password = this.properties.getDataPassword();
		runScripts(scripts, username, password);
	}


	private List<Resource> getScripts(String propertyName, List<String> resources,
			String fallback) {
		if (resources != null) {
            //如果配置了，则从指定的资源文件中加载脚本
			return getResources(propertyName, resources, true);
		}
        //如果未配置，则使用classpath*:schema-all.sql或者classpath*：schema.sql作为脚本名，其中platform默认为all
		String platform = this.properties.getPlatform();
		List<String> fallbackResources = new ArrayList<String>();
		fallbackResources.add("classpath*:" + fallback + "-" + platform + ".sql");
		fallbackResources.add("classpath*:" + fallback + ".sql");
		return getResources(propertyName, fallbackResources, false);
	}


	private List<Resource> getResources(String propertyName, List<String> locations,
			boolean validate) {
		List<Resource> resources = new ArrayList<Resource>();
		for (String location : locations) {
			for (Resource resource : doGetResources(location)) {
				if (resource.exists()) {
					resources.add(resource);
				}
				else if (validate) {
					throw new ResourceNotFoundException(propertyName, resource);
				}
			}
		}
		return resources;
	}


	private Resource[] doGetResources(String location) {
		try {
			SortedResourcesFactoryBean factory = new SortedResourcesFactoryBean(
					this.applicationContext, Collections.singletonList(location));
			factory.afterPropertiesSet();
			return factory.getObject();
		}
		catch (Exception ex) {
			throw new IllegalStateException("Unable to load resources from " + location,
					ex);
		}
	}


	private void runScripts(List<Resource> resources, String username, String password) {
		if (resources.isEmpty()) {
			return;
		}
		ResourceDatabasePopulator populator = new ResourceDatabasePopulator();
		populator.setContinueOnError(this.properties.isContinueOnError());
		populator.setSeparator(this.properties.getSeparator());
		if (this.properties.getSqlScriptEncoding() != null) {
			populator.setSqlScriptEncoding(this.properties.getSqlScriptEncoding().name());
		}
		for (Resource resource : resources) {
			populator.addScript(resource);
		}
		DataSource dataSource = this.dataSource;
		if (StringUtils.hasText(username) && StringUtils.hasText(password)) {
			dataSource = DataSourceBuilder.create(this.properties.getClassLoader())
					.driverClassName(this.properties.determineDriverClassName())
					.url(this.properties.determineUrl()).username(username)
					.password(password).build();
		}
		DatabasePopulatorUtils.execute(populator, dataSource);
	}
}		List<Resource> scripts = getScripts("spring.datasource.schema",
				this.properties.getSchema(), "schema");
		if (!scripts.isEmpty()) {
			String username = this.properties.getSchemaUsername();
			String password = this.properties.getSchemaPassword();
			runScripts(scripts, username, password);
			try {
				this.applicationContext
						.publishEvent(new DataSourceInitializedEvent(this.dataSource));
				// The listener might not be registered yet, so don't rely on it.
				if (!this.initialized) {
                    //执行数据插入脚本
					runDataScripts();
					this.initialized = true;
				}
			}
			catch (IllegalStateException ex) {
				logger.warn("Could not send event to complete DataSource initialization ("
						+ ex.getMessage() + ")");
			}
		}
	}


	@Override
	public void onApplicationEvent(DataSourceInitializedEvent event) {
		if (!this.properties.isInitialize()) {
			logger.debug("Initialization disabled (not running data scripts)");
			return;
		}
		// NOTE the event can happen more than once and
		// the event datasource is not used here
        //如果初始化完成后，执行数据插入
		if (!this.initialized) {
			runDataScripts();
			this.initialized = true;
		}
	}


	private void runDataScripts() {
		List<Resource> scripts = getScripts("spring.datasource.data",
				this.properties.getData(), "data");
		String username = this.properties.getDataUsername();
		String password = this.properties.getDataPassword();
		runScripts(scripts, username, password);
	}


	private List<Resource> getScripts(String propertyName, List<String> resources,
			String fallback) {
		if (resources != null) {
            //如果配置了，则从指定的资源文件中加载脚本
			return getResources(propertyName, resources, true);
		}
        //如果未配置，则使用classpath*:schema-all.sql或者classpath*：schema.sql作为脚本名，其中platform默认为all
		String platform = this.properties.getPlatform();
		List<String> fallbackResources = new ArrayList<String>();
		fallbackResources.add("classpath*:" + fallback + "-" + platform + ".sql");
		fallbackResources.add("classpath*:" + fallback + ".sql");
		return getResources(propertyName, fallbackResources, false);
	}


	private List<Resource> getResources(String propertyName, List<String> locations,
			boolean validate) {
		List<Resource> resources = new ArrayList<Resource>();
		for (String location : locations) {
			for (Resource resource : doGetResources(location)) {
				if (resource.exists()) {
					resources.add(resource);
				}
				else if (validate) {
					throw new ResourceNotFoundException(propertyName, resource);
				}
			}
		}
		return resources;
	}


	private Resource[] doGetResources(String location) {
		try {
			SortedResourcesFactoryBean factory = new SortedResourcesFactoryBean(
					this.applicationContext, Collections.singletonList(location));
			factory.afterPropertiesSet();
			return factory.getObject();
		}
		catch (Exception ex) {
			throw new IllegalStateException("Unable to load resources from " + location,
					ex);
		}
	}


	private void runScripts(List<Resource> resources, String username, String password) {
		if (resources.isEmpty()) {
			return;
		}
		ResourceDatabasePopulator populator = new ResourceDatabasePopulator();
		populator.setContinueOnError(this.properties.isContinueOnError());
		populator.setSeparator(this.properties.getSeparator());
		if (this.properties.getSqlScriptEncoding() != null) {
			populator.setSqlScriptEncoding(this.properties.getSqlScriptEncoding().name());
		}
		for (Resource resource : resources) {
			populator.addScript(resource);
		}
		DataSource dataSource = this.dataSource;
		if (StringUtils.hasText(username) && StringUtils.hasText(password)) {
			dataSource = DataSourceBuilder.create(this.properties.getClassLoader())
					.driverClassName(this.properties.determineDriverClassName())
					.url(this.properties.determineUrl()).username(username)
					.password(password).build();
		}
		DatabasePopulatorUtils.execute(populator, dataSource);
	}
}
```




DataSourceConfiguration

​    底层默认使用**tomcat-jdbc**连接池，所以在默认情况下会创建一个基于Tomcat连接池的DataSource，并注入到Spring IOC容器中。

```java
abstract class DataSourceConfiguration {
    @SuppressWarnings("unchecked")
    protected <T> T createDataSource(DataSourceProperties properties, Class<? extends DataSource> type) {
        return (T) properties.initializeDataSourceBuilder().type(type).build();
    }
    /**
     * Tomcat Pool DataSource configuration.
     */
    @ConditionalOnClass(org.apache.tomcat.jdbc.pool.DataSource.class)
    @ConditionalOnProperty(name = "spring.datasource.type", havingValue = "org.apache.tomcat.jdbc.pool.DataSource", matchIfMissing = true)
    static class Tomcat extends DataSourceConfiguration {
        //创建一个基于Tomcat连接池的DataSource的Bean
        //由于底层使用Tomcat连接池，在不引入外部数据连接池的jar包之前，默认使用Tomcat的DataSource
        @Bean
        @ConfigurationProperties(prefix = "spring.datasource.tomcat")
        public org.apache.tomcat.jdbc.pool.DataSource dataSource(DataSourceProperties properties) {
            org.apache.tomcat.jdbc.pool.DataSource dataSource = createDataSource(properties, org.apache.tomcat.jdbc.pool.DataSource.class);
            DatabaseDriver databaseDriver = DatabaseDriver.fromJdbcUrl(properties.determineUrl());
            String validationQuery = databaseDriver.getValidationQuery();
            if (validationQuery != null) {
                dataSource.setTestOnBorrow(true);
                dataSource.setValidationQuery(validationQuery);
            }
            return dataSource;
        }
    }
    /**
     * Hikari DataSource configuration.
     */
    @ConditionalOnClass(HikariDataSource.class)
    @ConditionalOnProperty(name = "spring.datasource.type", havingValue = "com.zaxxer.hikari.HikariDataSource", matchIfMissing = true)
    static class Hikari extends DataSourceConfiguration {
        //创建基于hikari的DataSource的Bean
        @Bean
        @ConfigurationProperties(prefix = "spring.datasource.hikari")
        public HikariDataSource dataSource(DataSourceProperties properties) {
            return createDataSource(properties, HikariDataSource.class);
        }
    }
    /**
     * DBCP DataSource configuration.
     *
     * @deprecated as of 1.5 in favor of DBCP2
     */
    @ConditionalOnClass(org.apache.commons.dbcp.BasicDataSource.class)
    @ConditionalOnProperty(name = "spring.datasource.type", havingValue = "org.apache.commons.dbcp.BasicDataSource", matchIfMissing = true)
    @Deprecated
    static class Dbcp extends DataSourceConfiguration {
        //创建基于dbcp的DataSource的Bean
        @Bean
        @ConfigurationProperties(prefix = "spring.datasource.dbcp")
        public org.apache.commons.dbcp.BasicDataSource dataSource(DataSourceProperties properties) {
            org.apache.commons.dbcp.BasicDataSource dataSource = createDataSource(properties, org.apache.commons.dbcp.BasicDataSource.class);
            DatabaseDriver databaseDriver = DatabaseDriver.fromJdbcUrl(properties.determineUrl());
            String validationQuery = databaseDriver.getValidationQuery();
            if (validationQuery != null) {
                dataSource.setTestOnBorrow(true);
                dataSource.setValidationQuery(validationQuery);
            }
            return dataSource;
        }
    }
    /**
     * DBCP DataSource configuration.
     */
    @ConditionalOnClass(org.apache.commons.dbcp2.BasicDataSource.class)
    @ConditionalOnProperty(name = "spring.datasource.type", havingValue = "org.apache.commons.dbcp2.BasicDataSource", matchIfMissing = true)
    static class Dbcp2 extends DataSourceConfiguration {
        //创建基于dbcp2的DataSource的Bean
        @Bean
        @ConfigurationProperties(prefix = "spring.datasource.dbcp2")
        public org.apache.commons.dbcp2.BasicDataSource dataSource(DataSourceProperties properties) {
            return createDataSource(properties,org.apache.commons.dbcp2.BasicDataSource.class);
        }
    }
    /**
     * Generic DataSource configuration.
     */
    @ConditionalOnMissingBean(DataSource.class)
    @ConditionalOnProperty(name = "spring.datasource.type")
    static class Generic {
        //可以在配置文件中通过spring.datasource.type指定外部的dataSource类型，并创建基于该type的DataSource的Bean
        @Bean
        public DataSource dataSource(DataSourceProperties properties) {
            return properties.initializeDataSourceBuilder().build();
        }
    }
}
```


SpringBoot默认支持：

- **org.apache.tomcat.jdbc.pool.DataSource**
- **com.zaxxer.hikari.HikariDataSource**
- **org.apache.commons.dbcp.BasicDataSource**
- **org.apache.commons.dbcp2.BasicDataSource**

上面讲到**DataSourceInitializer**是**ApplicationListener**类型的监听器

它的作用主要有：

​    i）**、runSchemaScripts()**;运行建表语句

​    ii）、**runDataScripts()**;运行插入数据的sql语句

默认只需要将文件命名为：

​            schema-*.sql 用于建表操作

​            data-*.sql 用于数据操作

​        默认规则：**schema.sql**或者**schema-all.sql**



自定义规则：**spring.datasource.schema=[classpath:department.sql]**

最后使用**JdbcTemplate**操作数据库，就类似在Spring框架中使用**JdbcTemplate**一样，这里就不再累赘了。

附上代码：

User实体类：

```java
public class User implements Serializable {

	private Integer uid;
	private String uname;
	private int uage;

	//getter和setter
}
```




User控制器：

```java
@RestController
@RequestMapping("/user")
public class UserController {

	@Autowired
	private IUserService userService;

	@PostMapping("/")
	public String insertUser(User user){
		int count = userService.insertUser(user);
		if(count != 1){
			return "error";
		}else{
			return "success";
		}
	}

	@GetMapping("/{uid}")
	public User getUserById(@PathVariable Integer uid){
		User user = userService.getUserById(uid);

		return user;
	}

	@GetMapping("/")
	public List<User> getAllUsers(){
		return userService.getAllUsers();
	}

	@PutMapping("/{uid}")
	public String updateUserById(@PathVariable Integer uid, User user){
		return userService.updateUserById(uid, user) == 1 ? "success" : "error";
	}

	@DeleteMapping("/{uid}")
	public String deleteUserById(@PathVariable Integer uid){
		return userService.deleteUserById(uid) == 1 ? "success" : "error";
	}
}
```



User业务层接口：

```java
public interface IUserService {

	int insertUser(User user);

	User getUserById(Integer id);

	List<User> getAllUsers();

	int updateUserById(Integer uid, User user);

	int deleteUserById(Integer uid);
}
```



User业务层接口实现类：

```java
@Service
public class UserServiceImpl implements IUserService {

	@Autowired
	private IUserDao userDao;

	@Override
	public int insertUser(User user) {
		return userDao.insertUser(user);
	}

	@Override
	public User getUserById(Integer id) {
		return userDao.getUserById(id);
	}

	@Override
	public List<User> getAllUsers() {
		return userDao.getAllUsers();
	}

	@Override
	public int updateUserById(Integer uid, User user) {
		return userDao.updateUserById(uid, user);
	}

	@Override
	public int deleteUserById(Integer uid) {
		return userDao.deleteUserById(uid);
	}
}
```



User持久层接口：

```java
public interface IUserDao {

	int insertUser(User user);

	User getUserById(Integer id);

	List<User> getAllUsers();

	int updateUserById(Integer uid, User user);

	int deleteUserById(Integer uid);
}
```



User持久层接口实现类：

```java
@Repository
public class UserDaoImpl implements IUserDao {

	@Autowired
	private JdbcTemplate jdbcTemplate;

	@Override
	public int insertUser(User user) {
		return jdbcTemplate.update("insert into user values(null, ?, ?)", user.getUname(), user.getUage());
	}

	@Override
	public User getUserById(Integer id) {
		Object[] params = {id};
		return jdbcTemplate.queryForObject("select * from user where uid=?", params, new UserRowMapper());
	}

	@Override
	public List<User> getAllUsers() {
		return jdbcTemplate.query("select * from user", new UserRowMapper());
	}

	@Override
	public int updateUserById(Integer uid, User user) {
		StringBuilder sb = new StringBuilder();
		List<Object> params = new ArrayList<>();
		boolean isUpdate = false;

		if (user != null) {
			sb.append("update user set ");

			if (user.getUname() != null && !"".equals(StringUtils.trimWhitespace(user.getUname()))) {
				isUpdate = true;
				sb.append(" uname = ?,");
				params.add(StringUtils.trimWhitespace(user.getUname()));
			}
			if (user.getUage() != 0) {
				isUpdate = true;
				sb.append(" uage = ?,");
				params.add(user.getUage());
			}
		}

		if (isUpdate) {
			sb.deleteCharAt(sb.length() - 1);

			sb.append(" where uid = ? ");
			params.add(user.getUid());
			return jdbcTemplate.update(sb.toString(), params.toArray());
		} else {
			return 0;
		}

	}

	@Override
	public int deleteUserById(Integer uid) {
		return jdbcTemplate.update("delete from user where uid = ?", uid);
	}
}
```



User数据映射类：


```java
public class UserRowMapper implements RowMapper<User> {
	@Override
	public User mapRow(ResultSet resultSet, int i) throws SQLException {
		User user = new User();
		user.setUid(resultSet.getInt("uid"));
		user.setUname(resultSet.getString("uname"));
		user.setUage(resultSet.getInt("uage"));
		return user;
	}
}
```


本节使用**Restful**风格进行编写，大概知道**Restful**格式即可，此处不多讲述。


