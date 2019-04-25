## 1、Spring应用程序处理国际化资源的步骤：

 1）、编写国际化配置文件；

 2）、使用**ResourceBundleMessageSource**管理国际化资源文件；

 3）、在页面使用fmt:message取出国际化内容。



## 2、Spring Boot处理国际化资源步骤：

1）、编写国际化配置文件，抽取页面中需要进行显示的国际化信息。

![](images\国际化预定义属性.png)



2）、Spring Boot自动配置了管理国际化资源文件的组件：

```java
@Configuration
@ConditionalOnMissingBean(value = MessageSource.class, search = SearchStrategy.CURRENT)
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE)
@Conditional(ResourceBundleCondition.class)
@EnableConfigurationProperties
@ConfigurationProperties(prefix = "spring.messages")
public class MessageSourceAutoConfiguration {
    
    /**
	 * Comma-separated list of basenames (essentially a fully-qualified classpath
	 * location), each following the ResourceBundle convention with relaxed support for
	 * slash based locations. If it doesn't contain a package qualifier (such as
	 * "org.mypackage"), it will be resolved from the classpath root.
	 */
	private String basename = "messages";
    
    //other code...
    
    @Bean
	public MessageSource messageSource() {
		ResourceBundleMessageSource messageSource = new ResourceBundleMessageSource();
		if (StringUtils.hasText(this.basename)) {
            //设置国际化资源文件的基础名
			messageSource.setBasenames(StringUtils.commaDelimitedListToStringArray(
					StringUtils.trimAllWhitespace(this.basename)));
		}
		if (this.encoding != null) {
			messageSource.setDefaultEncoding(this.encoding.name());
		}
		messageSource.setFallbackToSystemLocale(this.fallbackToSystemLocale);
		messageSource.setCacheSeconds(this.cacheSeconds);
		messageSource.setAlwaysUseMessageFormat(this.alwaysUseMessageFormat);
		return messageSource;
	}
}
```


默认情况下，国际化资源文件的基础名为**messages**，且存放在classpath根路径下，即messages.properties、messages_zh_CN.properties、messages_en_US.properties等等，这样就无需在配置文件中设置spring.messages.basename=...了，但是如果基础名不为messages或者不在classpath根路径下，则需要手动添加spring.messages.basename=文件名.自定义的基础名，如果有多个就用逗号分隔。例如：

```plain
spring.messages.basename=i18n.login
```



3）、去页面获取国际化的值：

```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
	<head>
		<meta http-equiv="Content-Type" content="text/html; charset=UTF-8">
		<meta name="viewport" content="width=device-width, initial-scale=1, shrink-to-fit=no">
		<meta name="description" content="">
		<meta name="author" content="">
		<title>Signin Template for Bootstrap</title>
		<!-- Bootstrap core CSS -->
		<link href="asserts/css/bootstrap.min.css" th:href="@{/webjars/bootstrap/4.1.0/css/bootstrap.css}" rel="stylesheet">
		<!-- Custom styles for this template -->
		<link href="asserts/css/signin.css" th:href="@{/asserts/css/signin.css}" rel="stylesheet">
	</head>

	<body class="text-center">
		<form class="form-signin" action="dashboard.html">
			<img class="mb-4" th:src="@{/asserts/img/bootstrap-solid.svg}" src="asserts/img/bootstrap-solid.svg" alt="" width="72" height="72">
			<h1 class="h3 mb-3 font-weight-normal" th:text="#{login.tip}">Please sign in</h1>
			<label class="sr-only" th:text="#{login.username}">Username</label>
			<input type="text" class="form-control" placeholder="Username" th:placeholder="#{login.username}" required="" autofocus="">
			<label class="sr-only" th:text="#{login.password}">Password</label>
			<input type="password" class="form-control" placeholder="Password" th:placeholder="#{login.password}" required="">
			<div class="checkbox mb-3">
				<label>
                                      <input type="checkbox" value="remember-me"> [[#{login.rememberme}]]
                                </label>
			</div>
			<button class="btn btn-lg btn-primary btn-block" type="submit" th:text="#{login.signin}">Sign in</button>
			<p class="mt-5 mb-3 text-muted">© 2017-2018</p>
			<a class="btn btn-sm">中文</a>
			<a class="btn btn-sm">English</a>
		</form>
	</body>
</html>
```

效果：根据浏览器语言设置的语言信息进行切换国际化语言。



4）、如何通过链接切换国际化语言？

原理：

 国际化区域信息对象Locale，有个区域信息对象处理器**LocaleResolver**，在**WebMvcAutoConfiguration**自动配置类中配置了一个**LocaleResolver**的Bean。

```java
@Bean
@ConditionalOnMissingBean
@ConditionalOnProperty(prefix = "spring.mvc", name = "locale")
public LocaleResolver localeResolver() {
    if (this.mvcProperties
        .getLocaleResolver() == WebMvcProperties.LocaleResolver.FIXED) {
        return new FixedLocaleResolver(this.mvcProperties.getLocale());
    }
    AcceptHeaderLocaleResolver localeResolver = new AcceptHeaderLocaleResolver();
    localeResolver.setDefaultLocale(this.mvcProperties.getLocale());
    return localeResolver;
}
```


- 如果在配置文件中配置了**spring.mvc.locale-resolver=fixed**，且指定了**spring.mvc.locale**的值，则页面使用的是指定且固定的国际语言（**FixedLocaleResolver**）；
- **spring.mvc.locale-resolver**默认为**accept-header**，即通过浏览器请求头中的Accept-Language 作为判断依据（**AcceptHeaderLocaleResolver**）：

```java
public class AcceptHeaderLocaleResolver implements LocaleResolver {
    
    @Override
	public Locale resolveLocale(HttpServletRequest request) {
		Locale defaultLocale = getDefaultLocale();
		if (defaultLocale != null && request.getHeader("Accept-Language") == null) {
			return defaultLocale;
		}
		Locale requestLocale = request.getLocale();
		List<Locale> supportedLocales = getSupportedLocales();
		if (supportedLocales.isEmpty() || supportedLocales.contains(requestLocale)) {
			return requestLocale;
		}
		Locale supportedLocale = findSupportedLocale(request, supportedLocales);
		if (supportedLocale != null) {
			return supportedLocale;
		}
		return (defaultLocale != null ? defaultLocale : requestLocale);
	}
    
    //other code..
}
```



使用链接切换国际化语言的步骤：**只需覆盖默认的LocaleResolver对象**

 i）、编写一个自定义的LocaleResolver类：

```java
//使用链接的方法发送国际语言代码
public class MyLocaleResolver  implements LocaleResolver{

	private final Logger logger = LoggerFactory.getLogger(MyLocaleResolver.class);

	@Override
	public Locale resolveLocale(HttpServletRequest request) {
		String lan = request.getParameter("lan");
		Locale locale = Locale.getDefault();
		if(StringUtils.hasText(lan)){
			try{
				String[] split = lan.split("_");
				locale = new Locale(split[0], split[1]);
			}catch (Exception e){
				e.printStackTrace();
				logger.error("错误信息:", e);
			}
		}
		return locale;
	}

	@Override
	public void setLocale(HttpServletRequest request, HttpServletResponse response, Locale locale) {

	}
}
```


先从请求参数中获取lan数据，然后通过下划线分割成语言和国家，然后通过语言和国家来封装成一个Locale对象并返回，如果未指定lan参数，则默认使用default的Locale对象。



​        ii）、修改页面切换语言的链接，添加国际化语言代码：

```html
<a class="btn btn-sm" th:href="@{/(lan='zh_CN')}">中文</a>
<a class="btn btn-sm" th:href="@{/(lan='en_US')}">English</a>
```




​        iii）、将自定义的LocaleResolver覆盖默认的LocaleResolver：

```java
@Configuration
public class MyMvcConfig extends WebMvcConfigurerAdapter{

	//使用自定义的LocaleResolver来替换掉默认的LocaleResolver
	@Bean
	public LocaleResolver localeResolver(){
		return new MyLocaleResolver();
	}
    //other code...
}
```


​    这样配置了自定义的LocaleResolver对象就会把WebMvcAutoConfiguration自动配置类中预定义的LocaleResolver的Bean屏蔽掉，Spring Boot就会使用自定义的LocaleResolver对象。

**注意：一旦使用链接来切换国际化语言，则会导致通过浏览器语言切换国际化语言的功能失效了。**

