## 1、SpringMVC自动配置官方文档

Spring Boot官方文档：[Spring Boot中Springmvc配置文档](https://docs.spring.io/spring-boot/docs/1.5.12.RELEASE/reference/htmlsingle/#boot-features-developing-web-applications)



## **2、Spring MVC auto-configuration**

**Spring Boot 提供了大多数SpringMVC应用常用的自动配置项。**

以下是Spring Boot对SpringMVC的默认配置（来自官网，自行翻译）:

- 自动配置了 `ContentNegotiatingViewResolver` 和 `BeanNameViewResolver` 的Beans.
    - **给容器自定义添加一个视图解析器，该ContentNegotiatingViewResolver 的bean会自动组合进来。**
    - 自动配置了**ViewResolver：ContentNegotiatingViewResolver**是组合了所有的视图解析器

```java
public class WebMvcAutoConfiguration {
    //other code...
    
    @Bean
    @ConditionalOnBean(ViewResolver.class)
    @ConditionalOnMissingBean(name = "viewResolver", value = ContentNegotiatingViewResolver.class)
    public ContentNegotiatingViewResolver viewResolver(BeanFactory beanFactory) {
        ContentNegotiatingViewResolver resolver = new ContentNegotiatingViewResolver();
        resolver.setContentNegotiationManager(
            beanFactory.getBean(ContentNegotiationManager.class));
        // ContentNegotiatingViewResolver uses all the other view resolvers to locate
        // a view so it should have a high precedence
        resolver.setOrder(Ordered.HIGHEST_PRECEDENCE);
        return resolver;
    }
}
```


其中**ContentNegotiatintViewResolver**类实现**ViewResoler**接口:

```java
public class ContentNegotiatingViewResolver extends WebApplicationObjectSupport
		implements ViewResolver, Ordered, InitializingBean {
	//other code...
	
	@Override
	public View resolveViewName(String viewName, Locale locale) throws Exception {
		RequestAttributes attrs = RequestContextHolder.getRequestAttributes();
		Assert.state(attrs instanceof ServletRequestAttributes, "No current ServletRequestAttributes");
		//获取所有的MediaType，例如application/json,text/html等等。。。
		List<MediaType> requestedMediaTypes = getMediaTypes(((ServletRequestAttributes) attrs).getRequest());
		if (requestedMediaTypes != null) {
			//获取所有的视图
			List<View> candidateViews = getCandidateViews(viewName, locale, requestedMediaTypes);
			//根据请求获取最适合的视图
			View bestView = getBestView(candidateViews, requestedMediaTypes, attrs);
			if (bestView != null) {
				return bestView;
			}
		}
		//other code...
	}
	
	private List<View> getCandidateViews(String viewName, Locale locale, List<MediaType> requestedMediaTypes)
			throws Exception {

		List<View> candidateViews = new ArrayList<View>();
		for (ViewResolver viewResolver : this.viewResolvers) {
			View view = viewResolver.resolveViewName(viewName, locale);
			if (view != null) {
				candidateViews.add(view);
			}
			for (MediaType requestedMediaType : requestedMediaTypes) {
				List<String> extensions = this.contentNegotiationManager.resolveFileExtensions(requestedMediaType);
				for (String extension : extensions) {
					String viewNameWithExtension = viewName + '.' + extension;
					view = viewResolver.resolveViewName(viewNameWithExtension, locale);
					if (view != null) {
						candidateViews.add(view);
					}
				}
			}
		}
		//other code...
		
		return candidateViews;
	}
}
```


- 支持静态资源文件夹和webjars
- 静态首页的访问。
- 自定义favicon图标
- 自动注册了 `Converter`, `GenericConverter`, `Formatter` Beans.
    - Converter：转换器，用于类型转换
    - Formatter：格式化器

```java
@Bean
//如果配置了spring.mvc.date-format，则自动注册Formatter<Date>的Bean
@ConditionalOnProperty(prefix = "spring.mvc", name = "date-format")
public Formatter<Date> dateFormatter() {
    return new DateFormatter(this.mvcProperties.getDateFormat());
}
```

​        

​	**自动添加的格式化转换器，只需添加到容器中即可。**

- 支持`HttpMessageConverters` 消息转换器。
    - **HttpMessageConverter**：SpringMVC用来转换Http请求和响应的
    - `**HttpMessageConverters**` 是从容器中获取的所有的**HttpMessageConverter**，**如果需要给容器中添加HttpMessageConverter，只需要将自定义的组件注册在容器中即可。**
- 自动配置 `MessageCodesResolver` 用于错误代码的生成规则。
    - PREFIX_ERROR_CODE： error_code + "." + object name + "." + field
    - POSTFIX_ERROR_CODE：object name + "." + field + "." + error_code

- 自动使用 `ConfigurableWebBindingInitializer`Bean。

    - 可以自定义配置一个`ConfigurableWebBindingInitializer`来替换默认的，需要添加到容器中。
    - 初始化**WebDataBinder**：用于将请求数据绑定到数据模型等。

    

     **包org.springframework.boot.autoconfigure.web是web的所有自动配置场景。**

​    来自官网：   

​    If you want to keep Spring Boot MVC features, and you just want to add additional [MVC configuration](https://docs.spring.io/spring/docs/4.3.16.RELEASE/spring-framework-reference/htmlsingle#mvc) (interceptors, formatters, view controllers etc.) you can add your own `@Configuration` class of type `WebMvcConfigurerAdapter`, but **without** `@EnableWebMvc`. If you wish to provide custom instances of `RequestMappingHandlerMapping`, `RequestMappingHandlerAdapter` or `ExceptionHandlerExceptionResolver` you can declare a `WebMvcRegistrationsAdapter` instance providing such components.

​    If you want to take complete control of Spring MVC, you can add your own `@Configuration` annotated with `@EnableWebMvc`.



## 3、扩展SpringMVC

**编写一个配置类，是WebMvcConfigurerAdapter类的子类，但是不能标注@EnableWebMvc注解。**

这样既保留了所有的自动配置，也能使用自定义的扩展配置。

```java
//使用WebMvcConfigurerAdapter可以来扩展SpringMVC的功能
@Configuration
public class MyMvcConfig extends WebMvcConfigurerAdapter{

	@Override
	public void addViewControllers(ViewControllerRegistry registry) {
//		super.addViewControllers(registry);
        //浏览器发送/cay请求，会直接跳到success页面。
		registry.addViewController("/cay").setViewName("success");
	}
}
```


原理：

* 1）、WebMvcAutoConfiguration是SpringMVC的自动配置类，在内部维护了一个内部类WebMvcAutoConfigurationAdapter，该类又继承自WebMvcConfigurerAdapter，看定义：

```java
@Configuration
@Import(EnableWebMvcConfiguration.class)
@EnableConfigurationProperties({ WebMvcProperties.class, ResourceProperties.class })
public static class WebMvcAutoConfigurationAdapter extends WebMvcConfigurerAdapter {}
```


​    而**WebMvcConfigurerAdapter**又实现了**WebMvcConfigurer**接口：

```java
public abstract class WebMvcConfigurerAdapter implements WebMvcConfigurer {}
```

​    所以自定义的配置类（此例为MyMvcConfig）是个**WebMvcConfigurer**接口的实现类。



* 2）、在做**WebMvcAutoConfigurationAdapter**自动配置时会导入**@Import(EnableWebMvcConfiguration.class)**

```java
@Configuration
public static class EnableWebMvcConfiguration extends DelegatingWebMvcConfiguration {}
```



父类：

```java
@Configuration
public class DelegatingWebMvcConfiguration extends WebMvcConfigurationSupport {

	private final WebMvcConfigurerComposite configurers = new WebMvcConfigurerComposite();

	//从容器中获取所有的WebMvcConfigurer
	@Autowired(required = false)
	public void setConfigurers(List<WebMvcConfigurer> configurers) {
		if (!CollectionUtils.isEmpty(configurers)) {
			this.configurers.addWebMvcConfigurers(configurers);
		}
	}
}
```



* 3）、容器中所有的WebMvcConfigurer都会一起起作用。

```java
class WebMvcConfigurerComposite implements WebMvcConfigurer {

	private final List<WebMvcConfigurer> delegates = new ArrayList<WebMvcConfigurer>();

	public void addWebMvcConfigurers(List<WebMvcConfigurer> configurers) {
		if (!CollectionUtils.isEmpty(configurers)) {
			this.delegates.addAll(configurers);
		}
	}
    
    @Override
	public void addFormatters(FormatterRegistry registry) {
		for (WebMvcConfigurer delegate : this.delegates) {
			delegate.addFormatters(registry);
		}
	}

	@Override
	public void addInterceptors(InterceptorRegistry registry) {
		for (WebMvcConfigurer delegate : this.delegates) {
			delegate.addInterceptors(registry);
		}
	}
    
    @Override
	public void addViewControllers(ViewControllerRegistry registry) {
		for (WebMvcConfigurer delegate : this.delegates) {
			delegate.addViewControllers(registry);
		}
	}
    
    //other code...
}
```


从源码中可以看到，从容器中获取的所有**WebMvcConfigurer**对象都会被调用对应的配置方法。



* 4）、最后可以结合第2点和第3点看出，自定义的配置类也会被调用。



 **总结：SpringMVC的自动配置和自定义的扩展配置都会起作用。**



## 4、全面接管SpringMVC

​    在配置类上使用**@EnableWebMvc**注解，这样Spring Boot对SpringMVC的自动配置就失效了，所有都需要自定义配置。

原理：为什么使用了**@EnableWebMvc**后自动配置就失效了?

* 1）、EnableWebMvc注解的定义

```java
@Import(DelegatingWebMvcConfiguration.class)
public @interface EnableWebMvc {}
```

​         

* 2）、导入了DelegatingWebMvcConfiguration组件配置

```java
@Configuration
public class DelegatingWebMvcConfiguration extends WebMvcConfigurationSupport {}
```

​        

* 3）、查看WebMvcAutoConfiguration自动配置类的签名

```java
@Configuration
@ConditionalOnWebApplication
@ConditionalOnClass({ Servlet.class, DispatcherServlet.class,
		WebMvcConfigurerAdapter.class })
//如果容器中没有该组件的时候，这个自动配置类就自动生效。
@ConditionalOnMissingBean(WebMvcConfigurationSupport.class)
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE + 10)
@AutoConfigureAfter({ DispatcherServletAutoConfiguration.class,
		ValidationAutoConfiguration.class })
public class WebMvcAutoConfiguration {}
```

​          

* 4）、@EnableWebMvc将WebMvcConfigurationSupport组件导入进来了，导致Spring Boot对SpringMVC的自动配置失效，即WebMvcAutoonfiguration配置类未注册成功。



## 5、修改SpringBoot的默认配置

**模式：**

* 1）、Spring Boot在自动配置很多组件的时候，会先检查容器中是否有用户自定义配置的Bean或者组件。如果有，就使用用户自定义的；如果没有，Spring Boot才自动配置；如果有些组件可以有多个，用户可以自定义添加组件，并加入到容器中，这样Spring Boot会自动将用户配置的和默认的组合起来。

* 2）、在Spring Boot中会有很多的Configurer帮助用户进行扩展配置。

