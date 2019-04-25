## 1、Spring Boot默认的错误处理机制

如果是浏览器，则返回一个默认的错误页面：

![](images\默认错误页面.png)

如果是其他测试工具，如Postman，则返回一个json数据：

![](images\postman错误信息.png)


原理：

 可以参照**ErrorMvcAutoConfiguration**，错误处理的自动配置类。该自动配置类给容器中添加了以下几个组件：




### **1）、ErrorPageCustomizer：错误页面定制器**

```java
@Bean
public ErrorPageCustomizer errorPageCustomizer() {
    return new ErrorPageCustomizer(this.serverProperties);
}
```


来看看ErrorPageCustomizer类定义：

```java
private static class ErrorPageCustomizer implements ErrorPageRegistrar, Ordered {

    private final ServerProperties properties;

    protected ErrorPageCustomizer(ServerProperties properties) {
        this.properties = properties;
    }

    @Override
    public void registerErrorPages(ErrorPageRegistry errorPageRegistry) {
        ErrorPage errorPage = new ErrorPage(this.properties.getServletPrefix()
                                            + this.properties.getError().getPath());
        errorPageRegistry.addErrorPages(errorPage);
    }
	//other code...
}
```


注册error页面，而页面的请求路径由getPath方法返回。

```java
@Value("${error.path:/error}")
private String path = "/error";

public String getPath() {
    return this.path;
}
```



所以当系统出现错误以后，会来到error请求进行处理。





### **2）、BasicErrorController：错误控制器**

创建一个Error控制器：

```java
@Bean
@ConditionalOnMissingBean(value = ErrorController.class, search = SearchStrategy.CURRENT)
public BasicErrorController basicErrorController(ErrorAttributes errorAttributes) {
    return new BasicErrorController(errorAttributes, this.serverProperties.getError(),
                                    this.errorViewResolvers);
}
```


看看Error控制器定义：



```java
@Controller
@RequestMapping("${server.error.path:${error.path:/error}}")
public class BasicErrorController extends AbstractErrorController {
    
    //浏览器首先返回text/html
    @RequestMapping(produces = "text/html")
	public ModelAndView errorHtml(HttpServletRequest request,
			HttpServletResponse response) {
		HttpStatus status = getStatus(request);
        //getErrorAttributes根据错误信息来封装一些model数据，用于页面显示
		Map<String, Object> model = Collections.unmodifiableMap(getErrorAttributes(
				request, isIncludeStackTrace(request, MediaType.TEXT_HTML)));
		response.setStatus(status.value());
        
        //返回错误页面，包含页面地址和页面内容
		ModelAndView modelAndView = resolveErrorView(request, response, status, model);
		return (modelAndView == null ? new ModelAndView("error", model) : modelAndView);
	}

    //其他测试工具默认返回json数据
	@RequestMapping
	@ResponseBody
	public ResponseEntity<Map<String, Object>> error(HttpServletRequest request) {
		Map<String, Object> body = getErrorAttributes(request,
				isIncludeStackTrace(request, MediaType.ALL));
		HttpStatus status = getStatus(request);
		return new ResponseEntity<Map<String, Object>>(body, status);
	}
    
    //other code...
}
```


从**@RequestMapping**属性上看得出，如果在配置文件中配置了server.error.path值，则使用指定的值作为错误请求；如果未配置，则查看是否配置了error.path；如果还是没有，则该控制器默认处理/error请求。

 该控制器处理错误请求，返回两种类型，分别是text/html和JSON数据，具体可以参看以下两图：

下图是浏览器访问时请求头中的accpet：

![](images\浏览器的请求头.png)

下图是测试工具访问时请求头中的accpet：

![](images\postman的请求头.png)


所以当使用浏览器访问时出现错误的时候，会进入**BasicErrorController**控制器中的errorHtml方法，而如果是测试工具访问时出现错误的时候，就进入error方法。



在响应页面的errorHtml方法中，会调用了父类的resolveErrorView方法：

```java
protected ModelAndView resolveErrorView(HttpServletRequest request,
			HttpServletResponse response, HttpStatus status, Map<String, Object> model) {
    //获取所有的视图解析器来处理这个错误信息，而这个errorViewResolvers对象其实就是DefaultErrorViewResolver对象的集合
    for (ErrorViewResolver resolver : this.errorViewResolvers) {
        ModelAndView modelAndView = resolver.resolveErrorView(request, status, model);
        if (modelAndView != null) {
            return modelAndView;
        }
    }
    return null;
}
```


所以从上述源码中看得出，在响应页面的时候，会在父类的**resolveErrorView**方法中获取所有的**ErrorViewResolver**对象（**DefaultErrorViewResolver**对象），一起来解析这个错误信息。



### **3）、DefaultErrorViewResolver：默认错误视图处理器**

```java
@Bean
@ConditionalOnBean(DispatcherServlet.class)
@ConditionalOnMissingBean
public DefaultErrorViewResolver conventionErrorViewResolver() {
    return new DefaultErrorViewResolver(this.applicationContext,
                                        this.resourceProperties);
}
```


来看DefaultErrorViewResolver类定义：

```java
public class DefaultErrorViewResolver implements ErrorViewResolver, Ordered {
    private static final Map<Series, String> SERIES_VIEWS;

	static {
		Map<Series, String> views = new HashMap<Series, String>();
		views.put(Series.CLIENT_ERROR, "4xx");
		views.put(Series.SERVER_ERROR, "5xx");
		SERIES_VIEWS = Collections.unmodifiableMap(views);
	}
    
    @Override
	public ModelAndView resolveErrorView(HttpServletRequest request, HttpStatus status,
			Map<String, Object> model) {
        //先以错误状态码作为错误页面名
		ModelAndView modelAndView = resolve(String.valueOf(status), model);
		if (modelAndView == null && SERIES_VIEWS.containsKey(status.series())) {
            //如果无法处理，则使用4xx或者5xx作为错误页面名
			modelAndView = resolve(SERIES_VIEWS.get(status.series()), model);
		}
		return modelAndView;
	}

	private ModelAndView resolve(String viewName, Map<String, Object> model) {
        //错误页面：error/400，或者error/404，或者error/500...
		String errorViewName = "error/" + viewName;
        //模版引擎可以解析到这个页面地址就用模版引擎来解析
		TemplateAvailabilityProvider provider = this.templateAvailabilityProviders
				.getProvider(errorViewName, this.applicationContext);
		if (provider != null) {
            //模版引擎能够解析到页面的情况下返回到errorViewName指定的视图
			return new ModelAndView(errorViewName, model);
		}
        //模版引擎不能够解析到页面的情况下，就在静态资源文件夹下查找errorViewName对应的页面
		return resolveResource(errorViewName, model);
	}

	private ModelAndView resolveResource(String viewName, Map<String, Object> model) {
		for (String location : this.resourceProperties.getStaticLocations()) {
			try {
				Resource resource = this.applicationContext.getResource(location);
                //从静态资源文件中查找errorViewName对应的页面
				resource = resource.createRelative(viewName + ".html");
				if (resource.exists()) {
                    //如果存在，则直接返回
					return new ModelAndView(new HtmlResourceView(resource), model);
				}
			}
			catch (Exception ex) {
			}
		}
		return null;
	}
}
```


步骤：

 第1步：假设访问出现了404报错，则状态码status=404，首先根据状态码status生成一个视图error/status；

 第2步：然后使用模版引擎去解析这个视图error/status，就是去查找classpath类路径下的templates模版文件夹下的error文件夹下是否有status.html这个页面；

 第3步：如果模版引擎能够解析到这个视图，则将该视图和model数据封装成ModelAndView返回并结束；否则进入第4步；

 第4步：假设解析不到error/status视图，则依次从静态资源文件中查找error/status.html，如果存在，则进行封装返回并结束；否则进入第5步；

 第5步：在模版引擎解析不到error/status视图，静态文件夹下都没有error/status.html的情况下，使用error/4xx作为视图名，即此时status=4xx，重新返回第1步进行查找；

 第6步：如果最后还是未找到，则使用Spring Boot默认错误页面。



### **4）、DefaultErrorAttributes：**

```java
@Bean
@ConditionalOnMissingBean(value = ErrorAttributes.class, search = SearchStrategy.CURRENT)
public DefaultErrorAttributes errorAttributes() {
    return new DefaultErrorAttributes();
}
```


看看DefaultErrorAttributes类定义：

```java
@Order(Ordered.HIGHEST_PRECEDENCE)
public class DefaultErrorAttributes
		implements ErrorAttributes, HandlerExceptionResolver, Ordered {
    
    @Override
    public Map<String, Object> getErrorAttributes(RequestAttributes requestAttributes,
                                                  boolean includeStackTrace) {
        //用来生成页面model数据
        Map<String, Object> errorAttributes = new LinkedHashMap<String, Object>();
        errorAttributes.put("timestamp", new Date());
        addStatus(errorAttributes, requestAttributes);
        addErrorDetails(errorAttributes, requestAttributes, includeStackTrace);
        addPath(errorAttributes, requestAttributes);
        return errorAttributes;
    }
    
    //other code...
}
```


在Error控制器处理错误的时候会调用**DefaultErrorAttributes**的**getErrorAttributes**方法来生成model数据，用于页面显示或者json数据的返回。

* model数据：

* timestamp：时间戳

* status：状态码

* error：错误提示

* exception：异常对象

* message：错误消息

* errors：jsr303数据校验错误内容

* path：错误请求路径



### **5）、defaultErrorView：Spring Boot默认视图View**

```java
private final SpelView defaultErrorView = new SpelView(
				"<html><body><h1>Whitelabel Error Page</h1>"
						+ "<p>This application has no explicit mapping for /error, so you are seeing this as a fallback.</p>"
						+ "<div id='created'>${timestamp}</div>"
						+ "<div>There was an unexpected error (type=${error}, status=${status}).</div>"
						+ "<div>${message}</div></body></html>");

@Bean(name = "error")
@ConditionalOnMissingBean(name = "error")
public View defaultErrorView() {
    return this.defaultErrorView;
}
```


​    在BasicErrorController返回前，如果在模版文件夹下或者静态资源文件夹下都无法找到对应的错误页面，则默认使用error作为视图名，而在**ErrorMvcAutoConfiguration**类中创建了名为error的视图Bean，用来作为Spring Boot默认的错误页面。



## **2、定制错误页面**

2.1、定制错误页面

* 1）、在有模版引擎的情况下，将错误页面命名为状态码.html，并放在模版文件夹下的error文件夹下，发生此状态码的错误就会来到对应的页面。可以使用4xx和5xx作为错误页面的文件名来匹配这种类型的所有错误。精确错误页面优先，当没有精确错误的页面，才去找4xx或者5xx错误页面。

* 2）、如果没有模版引擎的情况下，就会去静态资源文件夹下查找错误页面。

* ）、上两者都没有错误页面，则默认使用Spring Boot的错误提示页面，即使用error作为视图名。



2.2 定制错误Json数据

* 1）、自定义异常处理类并返回json数据

```java
@ControllerAdvice
public class MyExceptionHandler {

	@ResponseBody
	@ExceptionHandler(value= NotExistException.class)
	public Map<String, Object> handler(Exception e){
		Map<String, Object> map = new HashMap<>();
		map.put("message", e.getMessage());

		return map;
	}
}
```

缺点：浏览器和测试工具都返回json数据，没有自适应的功能。

* 2）、转发到/error请求进行自适应响应处理

```java
@ControllerAdvice
public class MyExceptionHandler {

	@ExceptionHandler(NotExistException.class)
	public String handler(Exception e, HttpServletRequest request){
		Map<String, Object> map = new HashMap<>();
//		Integer statusCode = (Integer) request.getAttribute("javax.servlet.error.status_code");
		//需要传入自己的错误状态码 4xx或者5xx（重要，否则是状态码为200，会找不到对应的错误页面）
		request.setAttribute("javax.servlet.error.status_code", 400);

		map.put("email", "412425870@qq.com");
		map.put("error", "发生错误啦！");
		request.setAttribute("map", map);

		//转发到/error请求
		return "forward:/error";
	}
}
```


缺点：虽然能够自适应，但是无法将自定义的错误信息传给页面或者json数据。





* 3）、将自定义的数据传递给页面或者json数据（重点）


​    自定义**ErrorAttributes**类，来覆盖**ErrorMvcAutoConfiguration**自动配置类中创建的默认**DefaultErrorAttributes**的Bean。

```java
@Component
public class MyErrorAttributes extends DefaultErrorAttributes {

	@Override
	public Map<String, Object> getErrorAttributes(RequestAttributes requestAttributes, boolean includeStackTrace) {
		Map<String, Object> map = super.getErrorAttributes(requestAttributes, includeStackTrace);

		//在原来的错误信息上添加自定义的错误信息
		map.put("author", "caychen");

        //获取异常处理类中设置的错误信息
		Map<String, Object> other = (Map<String, Object>) requestAttributes.getAttribute("map", RequestAttributes.SCOPE_REQUEST);
		map.put("other", other);
		return map;
	}
}
```

这样就可以在模版页面上获取对应的错误信息了。

```html
<h1>status: [[${status}]]</h1>
<h2>timestamp: [[${timestamp}]]</h2>
<h2>error: [[${error}]]</h2>
<h2>exception: [[${exception}]]</h2>
<h2>author: [[${author}]]</h2>
<h2 th:if="${other.error}" th:text="${other.error}"></h2>
<h2 th:if="${other.email}" th:text="${other.email}"></h2>
```
