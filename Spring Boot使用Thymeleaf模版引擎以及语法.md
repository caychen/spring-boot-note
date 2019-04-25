## 1、模版引擎


JSP，Velocity，Freemarker，Thymeleaf...

![](images\模版引擎.png)

Spring Boot推荐的模版引擎：**Thymeleaf**。



##  **2、引入Thymeleaf依赖**

```xml
<!-- 修改Spring Boot的默认版本 -->
<thymeleaf.version>3.0.9.RELEASE</thymeleaf.version>
<!-- 布局功能的支持程序
      thymeleaf3 对应layout2版本
      thymeleaf2 对应layout1版本
   -->
<thymeleaf-layout-dialect.version>2.2.2</thymeleaf-layout-dialect.version>

<!-- thymeleaf模版引擎 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
```


## 

## **3、Thymeleaf的使用&语法**

**ThymeleafProperties配置类：**

```java
@ConfigurationProperties(prefix = "spring.thymeleaf")
public class ThymeleafProperties {
    //默认编码
	private static final Charset DEFAULT_ENCODING = Charset.forName("UTF-8");
    //文档类型
	private static final MimeType DEFAULT_CONTENT_TYPE = MimeType.valueOf("text/html");
	//模版位置
	public static final String DEFAULT_PREFIX = "classpath:/templates/";
	//模版后缀
	public static final String DEFAULT_SUFFIX = ".html";

	private boolean checkTemplate = true;

	private boolean checkTemplateLocation = true;

	private String prefix = DEFAULT_PREFIX;

	private String suffix = DEFAULT_SUFFIX;

	private String mode = "HTML5";

	private Charset encoding = DEFAULT_ENCODING;

	private MimeType contentType = DEFAULT_CONTENT_TYPE;
    //缓存
	private boolean cache = true;

	private Integer templateResolverOrder;

	private String[] viewNames;

	private String[] excludedViewNames;

	private boolean enabled = true;
    
    //other code...
}
```




只要把模版html放置在classpath:/templates/目录下，thymeleaf就会自动渲染。





## 4、使用Thymeleaf：

 （1）、导入thymeleaf的名称空间：

```html
<html lang="en" xmlns:th="http://www.thymeleaf.org"></html>
```


​        （2）、thymeleaf语法：

```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
	<meta charset="UTF-8">
	<title>Title</title>
</head>
<body>
	<div th:text="${hello}">这里是原来的文本！</div>
</body>
</html>
```

​        （3）、语法规则：

​                i）、**th:text**：改变当前元素里面的文本内容

 **th:任意html属性**：用来替换原生html属性的值，例如id换成th:id， class换成th:class等等。

![](images\thymeleaf语法规则.png)


​                ii）、表达式：

```plain
Simple expressions:(表达式语法)
    Variable Expressions: ${...}：获取变量值，OGNL表达式
    	1）、获取对象的属性、调用方法
    	2）、使用内置的基本对象
		   #ctx : the context object.
            #vars: the context variables.
            #locale : the context locale.
            #request : (only in Web Contexts) the HttpServletRequest object.
            #response : (only in Web Contexts) the HttpServletResponse object.
            #session : (only in Web Contexts) the HttpSession object.
            #servletContext : (only in Web Contexts) the ServletContext object.
         3）、内置的工具对象
         	#execInfo : information about the template being processed.
            #messages : methods for obtaining externalized messages inside variables expressions, in the same way as they
            would be obtained using #{…} syntax.
            #uris : methods for escaping parts of URLs/URIs
            #conversions : methods for executing the configured conversion service (if any).
            #dates : methods for java.util.Date objects: formatting, component extraction, etc.
            #calendars : analogous to #dates , but for java.util.Calendar objects.
            #numbers : methods for formatting numeric objects.
            #strings : methods for String objects: contains, startsWith, prepending/appending, etc.
            #objects : methods for objects in general.
            #bools : methods for boolean evaluation.
            #arrays : methods for arrays.
            #lists : methods for lists.
            #sets : methods for sets.
            #maps : methods for maps.
            #aggregates : methods for creating aggregates on arrays or collections.
            #ids : methods for dealing with id attributes that might be repeated (for example, as a result of an iteration)
            
    Selection Variable Expressions: *{...}：选择表达式，跟${}功能类似
    	补充功能：配合th:object进行使用
    		<div th:object="${session.user}">
                <p>Name: <span th:text="*{firstName}">Sebastian</span>.</p>
                <p>Surname: <span th:text="*{lastName}">Pepper</span>.</p>
                <p>Nationality: <span th:text="*{nationality}">Saturn</span>.</p>
             </div>
             
    Message Expressions: #{...}：获取国际化内容
    Link URL Expressions: @{...}：定义URL链接
    	例子：@{/order/process(execId=${execId},execType='FAST')}
    	
    Fragment Expressions: ~{...}：片段引用表达式
    	
Literals:
    Text literals: 'one text' , 'Another one!' ,…
    Number literals: 0 , 34 , 3.0 , 12.3 ,…
    Boolean literals: true , false
    Null literal: null
    Literal tokens: one , sometext , main ,…
    
Text operations:
    String concatenation: +
    Literal substitutions: |The name is ${name}|
    Arithmetic operations:
    Binary operators: + , - , * , / , %
    Minus sign (unary operator): -
    
Boolean operations:
    Binary operators: and , or
    Boolean negation (unary operator): ! , not
    
Comparisons and equality:
    Comparators: > , < , >= , <= ( gt , lt , ge , le )
    Equality operators: == , != ( eq , ne )
    
Conditional operators:
    If-then: (if) ? (then)
    If-then-else: (if) ? (then) : (else)
    Default: (value) ?: (defaultvalue)
    
Special tokens:
	No-Operation: _
```


​               iii）、thymeleaf公共页面元素抽取：

​                      1）、抽取公共片段

```html
<div th:fragment="copy">
    &copy; 版权所有
</div>
```

​                      2）、引入公共片段

```html
<div th:insert="~{footer :: copy}"></div>
```


**~{templatename :: #selectorId}**  表示：模版名::#选择器id



```html
代码片段
<div id="copy-section">
	&copy; 2011 The Good Thymes Virtual Grocery
</div>

插入片段
<div th:insert="~{footer :: #copy-section}"></div>
```


**~{templatename :: fragmentname}** 表示：模版名::片段名

```html
代码片段
<div th:fragment="copy">
	&copy; 2011 The Good Thymes Virtual Grocery
</div>

插入片段
<div th:insert="~{footer :: copy}"></div>
```



​                      3）、默认效果：insert的功能片段会被插入到div标签中。



 **如果使用th:insert等属性进行引入，可以不用写~{}，可以直接写templatename::#selectorId/fragmentname。**

**但是如果是行内写法，必须加上~{}， 如[[~{}]]。**



三种引入公共片段的th属性：

 **th:insert**：将公共片段整个插入到声明引入的元素中

 **th:replace**：将声明引入的元素替换为公共片段

 **th:include**：将被引入的片段的内容包含到引入的元素中

```html
代码片段
<footer th:fragment="copy">
    &copy; 版权所有
</footer>

引入方式
<div th:insert="~{footer::copy}"></div>
<div th:replace="~{footer::copy}"></div>
<div th:include="~{footer::copy}"></div>

实际效果：
th:insert效果
<div>
    <footer>
    	&copy; 版权所有
	</footer>
</div>

th:replace效果
<footer>
    &copy; 版权所有
</footer>

th:include效果
<div>
    &copy; 版权所有
</div>
```


​                        （4）、在引入代码片段的时候，可以使用传递参数的方式，这样在代码片段中就可以使用传递过来的参数。

```html
<div th:fragment="frag">
...
</div>

<div th:replace="::frag (onevar=${value1},twovar=${value2})">
```


或

```html
<div th:fragment="frag (onevar,twovar)">
<p th:text="${onevar} + ' - ' + ${twovar}">...</p>
</div>

<div th:replace="::frag (${value1},${value2})">...</div>
使用命名参数时顺序不重要
<div th:replace="::frag (onevar=${value1},twovar=${value2})">...</div>
<div th:replace="::frag (twovar=${value2},onevar=${value1})">...</div>
```


比如，在templates/commons/bar.html中定义了如下代码片段

```html
<nav id="sidebar">
	<a class="nav-link" th:class="${activeUrl == 'main' ? 'nav-link active' : 'nav-link'}" ...>	
    	...
    </a>
</nav>
```


在引入该代码片段的时候，可以使用传递参数的方式。

```html
<div th:replace="commons/bar::#sidebar(activeUrl='main')">...</div>
或
<div th:replace="commons/bar::#sidebar(activeUrl='emps')">...</div>
```



至于其他Thymeleaf语法可以参考[Thymeleaf官网](https://mp.weixin.qq.com/cgi-bin/www.thymeleaf.org)：http://www.thymeleaf.org。
