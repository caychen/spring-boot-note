## 1、简介

使用Spring Boot：

（1）、创建Spring Boot应用，添加需要的模块；

（2）、Spring Boot对于支持自动配置的模块已经加载完毕，只需要在配置文件中指定少量配置信息即可；

（3）、编写业务逻辑代码。



## 2、Spring Boot对静态资源的映射规则：

#### 2.1 ResourceProperties

ResourceProperties是Spring Boot静态资源配置类

```java
@ConfigurationProperties(prefix = "spring.resources", ignoreUnknownFields = false)
public class ResourceProperties implements ResourceLoaderAware, InitializingBean {
    //可以设置和静态资源有关的参数，缓存时间等。。。
    
    
    private static final String[] SERVLET_RESOURCE_LOCATIONS = { "/" };

	private static final String[] CLASSPATH_RESOURCE_LOCATIONS = {
			"classpath:/META-INF/resources/", "classpath:/resources/",
			"classpath:/static/", "classpath:/public/" };
    
    //staticLocations即为上面连个变量的合集
    private String[] staticLocations = RESOURCE_LOCATIONS;
    
    static {
		RESOURCE_LOCATIONS = new String[CLASSPATH_RESOURCE_LOCATIONS.length
				+ SERVLET_RESOURCE_LOCATIONS.length];
		System.arraycopy(SERVLET_RESOURCE_LOCATIONS, 0, RESOURCE_LOCATIONS, 0,
				SERVLET_RESOURCE_LOCATIONS.length);
		System.arraycopy(CLASSPATH_RESOURCE_LOCATIONS, 0, RESOURCE_LOCATIONS,
				SERVLET_RESOURCE_LOCATIONS.length, CLASSPATH_RESOURCE_LOCATIONS.length);
	}
    
    //other code...
    
}
```



#### 2.2 WebMvcProperties

**WebMvcProperties**是WebMvc配置类 

```java
@ConfigurationProperties(prefix = "spring.mvc")
public class WebMvcProperties {
    //静态资源的路径pattern为/**
    private String staticPathPattern = "/**";
    
    //other code...
}
```



#### 2.3 WebMvcAutoConfiguration

MVC的自动配置都在**WebMvcAutoConfiguration**中

```java
@Configuration
@ConditionalOnWebApplication
@ConditionalOnClass({ Servlet.class, DispatcherServlet.class,
		WebMvcConfigurerAdapter.class })
@ConditionalOnMissingBean(WebMvcConfigurationSupport.class)
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE + 10)
@AutoConfigureAfter({ DispatcherServletAutoConfiguration.class,
		ValidationAutoConfiguration.class })
public class WebMvcAutoConfiguration {
    //other code...
}
```



先来看看静态资源文件的映射规则：

**WebMvcAutoConfiguration#addResourceHandlers**方法：

```java
@Override
public void addResourceHandlers(ResourceHandlerRegistry registry) {
    if (!this.resourceProperties.isAddMappings()) {
        logger.debug("Default resource handling disabled");
        return;
    }
    Integer cachePeriod = this.resourceProperties.getCachePeriod();
    //如果是/webjars/**请求，则将其映射到classpath:/META-INF/resources/webjars/目录下
    if (!registry.hasMappingForPattern("/webjars/**")) {
        customizeResourceHandlerRegistration(registry
                                             .addResourceHandler("/webjars/**")
                                             .addResourceLocations("classpath:/META-INF/resources/webjars/")
                                             .setCachePeriod(cachePeriod));
    }
    //如果是请求/**，将其映射到上述staticLocations指定的值
    String staticPathPattern = this.mvcProperties.getStaticPathPattern();
    if (!registry.hasMappingForPattern(staticPathPattern)) {
        customizeResourceHandlerRegistration(
            registry.addResourceHandler(staticPathPattern)
            .addResourceLocations(
                this.resourceProperties.getStaticLocations())
            .setCachePeriod(cachePeriod));
    }
}
```



**1）、所有/webjars/\**资源映射请求都去classpath:/META-INF/resources/webjars/找资源**

```java
Integer cachePeriod = this.resourceProperties.getCachePeriod();
if (!registry.hasMappingForPattern("/webjars/**")) {
    customizeResourceHandlerRegistration(registry
                                         .addResourceHandler("/webjars/**")
                                         .addResourceLocations("classpath:/META-INF/resources/webjars/")
                                         .setCachePeriod(cachePeriod));
}
```





webjars：以jar包的方式引入的静态资源，<http://www.webjars.org/>

以jQuery为例：引入jQuery的jar包

```html
<!-- 引入webjars-jQuery -->
<dependency>
    <groupId>org.webjars</groupId>
    <artifactId>jquery</artifactId>
    <version>3.3.1-1</version>
</dependency>
```

![](images\以webjars模式引入的jQuery.png)

请求jQuery正确的路径为:<http://localhost:8080/webjars/jquery/3.3.1-1/jquery.js>

**总结：所有以/webjars/\**的资源映射请求会去classpath:/META-INF/resources/webjars/找资源。**



**2）、/\** 访问当前项目的任何资源（静态资源的文件夹）**

```java
String staticPathPattern = this.mvcProperties.getStaticPathPattern();
if (!registry.hasMappingForPattern(staticPathPattern)) {
    customizeResourceHandlerRegistration(
        registry.addResourceHandler(staticPathPattern)
        .addResourceLocations(
            this.resourceProperties.getStaticLocations())
        .setCachePeriod(cachePeriod));
}
```


`this.mvcProperties.getStaticPathPattern()`返回/**

`this.resourceProperties.getStaticLocations()`返回

```plain
"/",
"classpath:/META-INF/resources/", 
"classpath:/resources/",
"classpath:/static/", 
"classpath:/public/"
```



**总结：任何访问/\**的请求，就会去静态资源文件夹下查找对应的资源名。**



**3）、欢迎页（首页）的资源映射**

```java
@Bean
public WelcomePageHandlerMapping welcomePageHandlerMapping(
    ResourceProperties resourceProperties) {
    return new WelcomePageHandlerMapping(resourceProperties.getWelcomePage(),
                                         this.mvcProperties.getStaticPathPattern());
}
```



**ResourceProperties#getWelcomePage**方法：

```java
public Resource getWelcomePage() {
    for (String location : getStaticWelcomePageLocations()) {
        Resource resource = this.resourceLoader.getResource(location);
        try {
            if (resource.exists()) {
                resource.getURL();
                return resource;
            }
        }
        catch (Exception ex) {
            // Ignore
        }
    }
    return null;
}
```



**ResourceProperties#getStaticWelcomePageLocations**方法：

```java
//获取所有静态资源文件夹下的欢迎页
private String[] getStaticWelcomePageLocations() {
    String[] result = new String[this.staticLocations.length];
    for (int i = 0; i < result.length; i++) {
        String location = this.staticLocations[i];
        if (!location.endsWith("/")) {
            location = location + "/";
        }
        result[i] = location + "index.html";
    }
    return result;
}
```


总结：从源码来看，静态资源文件夹下的所有index.html页面会被/**映射。



**4）、所有的\**/favicon.ico 都会在静态资源文件夹下查找：**

```java
@Configuration
@ConditionalOnProperty(value = "spring.mvc.favicon.enabled", matchIfMissing = true)
public static class FaviconConfiguration {

    private final ResourceProperties resourceProperties;

    public FaviconConfiguration(ResourceProperties resourceProperties) {
        this.resourceProperties = resourceProperties;
    }

    @Bean
    public SimpleUrlHandlerMapping faviconHandlerMapping() {
        SimpleUrlHandlerMapping mapping = new SimpleUrlHandlerMapping();
        mapping.setOrder(Ordered.HIGHEST_PRECEDENCE + 1);
        mapping.setUrlMap(Collections.singletonMap("**/favicon.ico",
                                                   faviconRequestHandler()));
        return mapping;
    }

    @Bean
    public ResourceHttpRequestHandler faviconRequestHandler() {
        ResourceHttpRequestHandler requestHandler = new ResourceHttpRequestHandler();
        requestHandler
            .setLocations(this.resourceProperties.getFaviconLocations());
        return requestHandler;
    }
}
```



#### **ResourceProperties#getFaviconLocations**方法：

```java
//ResourceProperties#getFaviconLocations()
List<Resource> getFaviconLocations() {
    List<Resource> locations = new ArrayList<Resource>(
        this.staticLocations.length + 1);
    if (this.resourceLoader != null) {
        for (String location : this.staticLocations) {
            locations.add(this.resourceLoader.getResource(location));
        }
    }
    locations.add(new ClassPathResource("/"));
    return Collections.unmodifiableList(locations);
}
```



### 总结：任何路径下请求\**/favicon.ico，都会去静态资源文件下查找。** 