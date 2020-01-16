
# 1.SpringMVC原理分析

## 1.1 流程

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/SpringMVC框架.png" width="620px" > </div><br>


* step1、用户发送请求至前端控制器DispatcherServlet
* step2、DispatcherServlet收到请求调用HandlerMapping处理器映射器。
* step3、处理器映射器根据请求url找到具体的处理器，生成处理器对象及处理器拦截器(如果有则生成)一并返回给DispatcherServlet。
* step4、DispatcherServlet通过HandlerAdapter处理器适配器调用处理器
* step5、HandlerAdapter执行处理器(handler，也叫后端控制器)。
* step6、Controller执行完成返回ModelAndView
* step7、HandlerAdapter将handler执行结果ModelAndView返回给DispatcherServlet
* step8、DispatcherServlet将ModelAndView传给ViewReslover视图解析器
* step9、ViewReslover解析后返回具体View对象
* step10、DispatcherServlet对View进行渲染视图（即将模型数据填充至视图中）。
* step11、DispatcherServlet响应用户


## 1.2 组件说明

以下组件通常使用框架提供的实现：
* DispatcherServlet：前端控制器
用户请求到达前端控制器，它就相当于mvc模式中的C，dispatcherServlet是整个流程控制的中心，由它调用其它组件处理用户的请求，dispatcherServlet的存在降低了组件之间的耦合性。

* HandlerMapping：处理器映射器
HandlerMapping负责根据用户请求找到Handler即处理器，springmvc提供了不同的映射器实现不同的映射方式，例如：配置文件方式，实现接口方式，注解方式等。

* Handler：处理器
Handler 是继DispatcherServlet前端控制器的后端控制器，在DispatcherServlet的控制下Handler对具体的用户请求进行处理。
由于Handler涉及到具体的用户业务请求，所以一般情况需要程序员根据业务需求开发Handler。

* HandlAdapter：处理器适配器
通过HandlerAdapter对处理器进行执行，这是适配器模式的应用，通过扩展适配器可以对更多类型的处理器进行执行。

* View Resolver：视图解析器
View Resolver负责将处理结果生成View视图，View Resolver首先根据逻辑视图名解析成物理视图名即具体的页面地址，再生成View视图对象，最后对View进行渲染将处理结果通过页面展示给用户。 

* View：视图
springmvc框架提供了很多的View视图类型的支持，包括：jstlView、freemarkerView、pdfView等。我们最常用的视图就是jsp。
一般情况下需要通过页面标签或页面模版技术将模型数据通过页面展示给用户，需要由程序员根据业务需求开发具体的页面。


说明：在springmvc的各个组件中，处理器映射器、处理器适配器、视图解析器称为springmvc的三大组件。

需要用户开发的组件有：处理器、视图


## 1.3 默认配置文件

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/默认配置文件.jpg" width="520px" > </div><br>

spring-webmvc-xxx.jar包中有一个DispatcherServlet.properties文件，该配置中默认加载了一些springmvc默认的其他组件，其中就包括三大组件。

```
# Default implementation classes for DispatcherServlet's strategy interfaces.
# Used as fallback when no matching beans are found in the DispatcherServlet context.
# Not meant to be customized by application developers.

org.springframework.web.servlet.LocaleResolver=org.springframework.web.servlet.i18n.AcceptHeaderLocaleResolver

org.springframework.web.servlet.ThemeResolver=org.springframework.web.servlet.theme.FixedThemeResolver

org.springframework.web.servlet.HandlerMapping=org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping,\
	org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping,\
	org.springframework.web.servlet.function.support.RouterFunctionMapping

org.springframework.web.servlet.HandlerAdapter=org.springframework.web.servlet.mvc.HttpRequestHandlerAdapter,\
	org.springframework.web.servlet.mvc.SimpleControllerHandlerAdapter,\
	org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter,\
	org.springframework.web.servlet.function.support.HandlerFunctionAdapter


org.springframework.web.servlet.HandlerExceptionResolver=org.springframework.web.servlet.mvc.method.annotation.ExceptionHandlerExceptionResolver,\
	org.springframework.web.servlet.mvc.annotation.ResponseStatusExceptionResolver,\
	org.springframework.web.servlet.mvc.support.DefaultHandlerExceptionResolver

org.springframework.web.servlet.RequestToViewNameTranslator=org.springframework.web.servlet.view.DefaultRequestToViewNameTranslator

org.springframework.web.servlet.ViewResolver=org.springframework.web.servlet.view.InternalResourceViewResolver

org.springframework.web.servlet.FlashMapManager=org.springframework.web.servlet.support.SessionFlashMapManager
```

# 2.DispatcherServlet

## 2.1 Servlet生命周期

```
public interface Servlet {
    void init(ServletConfig var1) throws ServletException;

    ServletConfig getServletConfig();

    void service(ServletRequest var1, ServletResponse var2) throws ServletException, IOException;

    String getServletInfo();

    void destroy();
}

```

* init：Servlet对象创建之后调用
* service：Servlet对象被HTTP请求访问时调用
* destroy：Servlet对象销毁之前调用


## 2.2 DispatcherServlet继承体系

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/DispatcherServlet.png" width="820px" > </div><br>


# 3.Servlet3.0

## 3.1 原生的servlet来处理 order的请求

jsp页面：

```
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
  <head>
    <title>Hello Servlet!</title>
  </head>
  <body>
  <a href="order">order</a>
  </body>
</html>
```


建立一个原生的servlet来处理 order的请求：

```
@WebServlet("/order")
public class JamesServlet  extends HttpServlet{
	@Override
	protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
		resp.getWriter().write("success...");
	}
}
```

## 3.2 共享库和运行时插件

参考Servlet3.1规范8.2.4节：

> ServletContainerInitializer 类通过 jar services API 查找。对于每一个应用，应用启动时，由容器创建一个ServletContainerInitializer 实例。 框架提供的ServletContainerInitializer 实现必须绑定在 jar 包的META-INF/services 目录中的一个叫做javax.servlet.ServletContainerInitializer 的文件，根据 jar services API，指定 ServletContainerInitializer 的实现。

> 除 ServletContainerInitializer 外，我们还有一个注解—HandlesTypes。在 ServletContainerInitializer 实现上的HandlesTypes 注解用于表示感兴趣的一些类，它们可能指定了 HandlesTypes 的 value 中的注解（类型、方法或自动级别的注解），或者是其类型的超类继承/实现了这些类之一。

> 在任何 Servlet Listener 的事件被触发之前，当应用正在启动时，ServletContainerInitializer 的 onStartup 方法将被调用。
ServletContainerInitializer’s 的 onStartup 得到一个类的 Set，其或者继承/实现 initializer 表示感兴趣的类，或者它是使用指定在@HandlesTypes 注解中的任意类注解的。


ServletContainerInitializer初始化web容器：

在web容器启动时为提供给第三方组件机会做一些初始化的工作，例如注册servlet或者filters等，servlet规范(JSR356)中通过ServletContainerInitializer实现此功能。


每个框架要使用ServletContainerInitializer就必须在对应的jar包的META-INF/services 目录创建一个名为javax.servlet.ServletContainerInitializer的文件，文件内容指定具体的ServletContainerInitializer实现类，那么，当web容器启动时就会运行这个初始化器做一些组件内的初始化工作。


### 3.2.1 创建META-INF/services目录

创建javax.servlet.ServletContainerInitializer文件：

```
com.enjoy.sevlet.JamesServletContainerInitializer
```

### 3.2.2 JamesServletContainerInitializer

```
//容器启动的时候会将@HandlesTypes指定的这个类型下面的子类（实现类，子接口等）传递过来；
//传入感兴趣的类型；
@HandlesTypes(value={JamesService.class})
public class JamesServletContainerInitializer implements ServletContainerInitializer{
	/**
	 * tomcat启动时加载应用的时候，会运行onStartup方法；
	 * 
	 * Set<Class<?>> arg0：感兴趣的类型的所有子类型(对实现了JamesService接口相关的)；
	 * ServletContext arg1:代表当前Web应用的ServletContext；一个Web应用一个ServletContext；
	 * 
	 * 1）、使用ServletContext注册Web组件（Servlet、Filter、Listener）
	 * 2）、使用编码的方式，在项目启动的时候给ServletContext里面添加组件；
	 * 		必须在项目启动的时候来添加；
	 * 		1）、ServletContainerInitializer得到的ServletContext；
	 * 		2）、ServletContextListener得到的ServletContext；
	 */
	@Override
	public void onStartup(Set<Class<?>> arg0, ServletContext arg1) throws ServletException {
		System.out.println("感兴趣的类型：");
		for (Class<?> claz : arg0) {
			System.out.println(claz);//当传进来后，可以根据自己需要利用反射来创建对象等操作
		}
		//注册servlet组件
		Dynamic servlet = arg1.addServlet("orderServlet", new OrderServlet());
		//配置servlet的映射信息（路径请求）
		servlet.addMapping("/orderTest");
		
		//注册监听器Listener
		arg1.addListener(OrderListener.class);
		
		//注册Filter
		javax.servlet.FilterRegistration.Dynamic filter = arg1.addFilter("orderFilter", OrderFilter.class);
		//添加Filter的映射信息，可以指定专门来拦截哪个servlet
		filter.addMappingForUrlPatterns(EnumSet.of(DispatcherType.REQUEST), true, "/*");
		
	}

}
```

一般伴随着ServletContainerInitializer一起使用的还有HandlesTypes注解，通过HandlesTypes可以将感兴趣的一些类注入到ServletContainerInitializer的onStartup方法作为参数传入。

```
感兴趣的类型：
class com.enjoy.service.JamesServiceImpl
class com.enjoy.service.AbstractJamesService
interface com.enjoy.service.JamesServiceOther
```


使用ServletContext注册web组件（其实就是Servlet,Filter,Listener三大组件），对于我们自己写的JamesServlet，我们可以使用@WebServlet注解来加入JamesServlet组件，但若是我们要导入第三方阿里的连接池或filter，以前的web.xml方式就可通过配置加载就可以了，但现在我们使用ServletContext注入进来。

结果：

```
Connected to server

感兴趣的类型：
class com.enjoy.service.JamesServiceImpl
class com.enjoy.service.AbstractJamesService
interface com.enjoy.service.JamesServiceOther
UserListener...contextInitialized...

UserFilter...doFilter...
UserFilter...doFilter...

UserFilter...doFilter...

"D:\Program Files\apache-tomcat-8.5.47\bin\catalina.bat" stop


UserListener...contextDestroyed...

Disconnected from server
```


# 4.SpringMVC注解

## 4.1 不适用web.xml并引入相关依赖

```
	<dependencies>
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-webmvc</artifactId>
			<version>${spring.version}</version>
		</dependency>

		<dependency>
			<groupId>javax.servlet</groupId>
			<artifactId>servlet-api</artifactId>
			<version>3.0-alpha-1</version>
			<scope>provided</scope>
		</dependency>
	</dependencies>

	<build>
		<plugins>
			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-war-plugin</artifactId>
				<version>2.4</version>
				<configuration>
					<failOnMissingWebXml>false</failOnMissingWebXml>
				</configuration>
			</plugin>
		</plugins>
	</build>
```

## 4.2 查看spring-web.jar包

其META-INF/services/javax.servlet.ServletContainerInitializer中的内容是：

```
org.springframework.web.SpringServletContainerInitializer
```

## 4.3 SpringServletContainerInitializer

```
@HandlesTypes(WebApplicationInitializer.class)
public class SpringServletContainerInitializer implements ServletContainerInitializer {

	@Override
	public void onStartup(@Nullable Set<Class<?>> webAppInitializerClasses, ServletContext servletContext)
			throws ServletException {

		List<WebApplicationInitializer> initializers = new LinkedList<>();

		if (webAppInitializerClasses != null) {
			for (Class<?> waiClass : webAppInitializerClasses) {
				// Be defensive: Some servlet containers provide us with invalid classes,
				// no matter what @HandlesTypes says...
				if (!waiClass.isInterface() && !Modifier.isAbstract(waiClass.getModifiers()) &&
						WebApplicationInitializer.class.isAssignableFrom(waiClass)) {
					try {
						initializers.add((WebApplicationInitializer)
								ReflectionUtils.accessibleConstructor(waiClass).newInstance());
					}
					catch (Throwable ex) {
						throw new ServletException("Failed to instantiate WebApplicationInitializer class", ex);
					}
				}
			}
		}

		if (initializers.isEmpty()) {
			servletContext.log("No Spring WebApplicationInitializer types detected on classpath");
			return;
		}

		servletContext.log(initializers.size() + " Spring WebApplicationInitializers detected on classpath");
		AnnotationAwareOrderComparator.sort(initializers);
		for (WebApplicationInitializer initializer : initializers) {
			initializer.onStartup(servletContext);
		}
	}

}


```

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/Initializers.jpg" width="520px" > </div><br>


<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/JamesWebAppInitializer.png" width="620px" > </div><br>


查看JamesWebAppInitializer：

```
//web容器启动的时候创建对象；调用方法来初始化容器以前前端控制器
public class JamesWebAppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {

	//获取根容器的配置类；（Spring的配置文件）   父容器；
	@Override
	protected Class<?>[] getRootConfigClasses() {
		//指定配置类（配置文件）位置
		return new Class<?>[]{JamesRootConfig.class} ;
	}

	//获取web容器的配置类（SpringMVC配置文件）  子容器；
	@Override
	protected Class<?>[] getServletConfigClasses() {
		return new Class<?>[]{JamesAppConfig.class} ;
	}

	//获取DispatcherServlet的映射信息
	//  /：拦截所有请求（包括静态资源（xx.js,xx.png）），但是不包括*.jsp；
	//  /*：拦截所有请求；连*.jsp页面都拦截；jsp页面是tomcat的jsp引擎解析的；
	@Override
	protected String[] getServletMappings() {
		// TODO Auto-generated method stub
		return new String[]{"/"};
	}

}
```


在SpringServletContainerInitializer.onStart()最后调用JamesWebAppInitializer.onStartup()

```
		for (WebApplicationInitializer initializer : initializers) {
			initializer.onStartup(servletContext);
		}
```

调用栈依次为：

### 4.3.1 AbstractContextLoaderInitializer.onStartup()

创建根容器。

```
public abstract class AbstractContextLoaderInitializer implements WebApplicationInitializer {

	/** Logger available to subclasses. */
	protected final Log logger = LogFactory.getLog(getClass());


	@Override
	public void onStartup(ServletContext servletContext) throws ServletException {
		registerContextLoaderListener(servletContext);
	}

	/**
	 * Register a {@link ContextLoaderListener} against the given servlet context. The
	 * {@code ContextLoaderListener} is initialized with the application context returned
	 * from the {@link #createRootApplicationContext()} template method.
	 * @param servletContext the servlet context to register the listener against
	 */
	protected void registerContextLoaderListener(ServletContext servletContext) {
		WebApplicationContext rootAppContext = createRootApplicationContext();
		if (rootAppContext != null) {
			ContextLoaderListener listener = new ContextLoaderListener(rootAppContext);
			listener.setContextInitializers(getRootApplicationContextInitializers());
			servletContext.addListener(listener);
		}
		else {
			logger.debug("No ContextLoaderListener registered, as " +
					"createRootApplicationContext() did not return an application context");
		}
	}

```

### 4.3.2 AbstractDispatcherServletInitializer.onStartup()

DispatcherServlet初始化，创建子容器。

```
public abstract class AbstractDispatcherServletInitializer extends AbstractContextLoaderInitializer {

	/**
	 * The default servlet name. Can be customized by overriding {@link #getServletName}.
	 */
	public static final String DEFAULT_SERVLET_NAME = "dispatcher";


	@Override
	public void onStartup(ServletContext servletContext) throws ServletException {
		super.onStartup(servletContext);
		registerDispatcherServlet(servletContext);
	}

	/**
	 * Register a {@link DispatcherServlet} against the given servlet context.
	 * <p>This method will create a {@code DispatcherServlet} with the name returned by
	 * {@link #getServletName()}, initializing it with the application context returned
	 * from {@link #createServletApplicationContext()}, and mapping it to the patterns
	 * returned from {@link #getServletMappings()}.
	 * <p>Further customization can be achieved by overriding {@link
	 * #customizeRegistration(ServletRegistration.Dynamic)} or
	 * {@link #createDispatcherServlet(WebApplicationContext)}.
	 * @param servletContext the context to register the servlet against
	 */
	protected void registerDispatcherServlet(ServletContext servletContext) {
		String servletName = getServletName();
		Assert.hasLength(servletName, "getServletName() must not return null or empty");

		WebApplicationContext servletAppContext = createServletApplicationContext();
		Assert.notNull(servletAppContext, "createServletApplicationContext() must not return null");

		FrameworkServlet dispatcherServlet = createDispatcherServlet(servletAppContext);
		Assert.notNull(dispatcherServlet, "createDispatcherServlet(WebApplicationContext) must not return null");
		dispatcherServlet.setContextInitializers(getServletApplicationContextInitializers());

		ServletRegistration.Dynamic registration = servletContext.addServlet(servletName, dispatcherServlet);
		if (registration == null) {
			throw new IllegalStateException("Failed to register servlet with name '" + servletName + "'. " +
					"Check if there is another servlet registered under the same name.");
		}

		registration.setLoadOnStartup(1);
		registration.addMapping(getServletMappings());
		registration.setAsyncSupported(isAsyncSupported());

		Filter[] filters = getServletFilters();
		if (!ObjectUtils.isEmpty(filters)) {
			for (Filter filter : filters) {
				registerServletFilter(servletContext, filter);
			}
		}

		customizeRegistration(registration);
	}
```

### 4.3.3 AbstractAnnotationConfigDispatcherServletInitializer创建根容器和子容器

```
public abstract class AbstractAnnotationConfigDispatcherServletInitializer
		extends AbstractDispatcherServletInitializer {

	/**
	 * {@inheritDoc}
	 * <p>This implementation creates an {@link AnnotationConfigWebApplicationContext},
	 * providing it the annotated classes returned by {@link #getRootConfigClasses()}.
	 * Returns {@code null} if {@link #getRootConfigClasses()} returns {@code null}.
	 */
	@Override
	@Nullable
	protected WebApplicationContext createRootApplicationContext() {
		Class<?>[] configClasses = getRootConfigClasses();
		if (!ObjectUtils.isEmpty(configClasses)) {
			AnnotationConfigWebApplicationContext context = new AnnotationConfigWebApplicationContext();
			context.register(configClasses);
			return context;
		}
		else {
			return null;
		}
	}

	/**
	 * {@inheritDoc}
	 * <p>This implementation creates an {@link AnnotationConfigWebApplicationContext},
	 * providing it the annotated classes returned by {@link #getServletConfigClasses()}.
	 */
	@Override
	protected WebApplicationContext createServletApplicationContext() {
		AnnotationConfigWebApplicationContext context = new AnnotationConfigWebApplicationContext();
		Class<?>[] configClasses = getServletConfigClasses();
		if (!ObjectUtils.isEmpty(configClasses)) {
			context.register(configClasses);
		}
		return context;
	}
```

root根容器与servlet容器的区别在哪呢？

参见[官方文档](https://docs.spring.io/spring/docs/5.2.2.RELEASE/spring-framework-reference/web.html#spring-web)：

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/mvc-context-hierarchy.png" width="620px" > </div><br>

很明显，servlet的容器用来处理@Controller，视图解析，和web相关组件而root根容器主要针对服务层，和数据源DAO层及事务控制相关处理。


## 4.4 JamesWebAppInitializer配置父子容器

```
//web容器启动的时候创建对象；调用方法来初始化容器以前前端控制器
public class JamesWebAppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {

	//获取根容器的配置类；（Spring的配置文件）   父容器；
	@Override
	protected Class<?>[] getRootConfigClasses() {
		//指定配置类（配置文件）位置
		return new Class<?>[]{JamesRootConfig.class} ;
	}

	//获取web容器的配置类（SpringMVC配置文件）  子容器；
	@Override
	protected Class<?>[] getServletConfigClasses() {
		return new Class<?>[]{JamesAppConfig.class} ;
	}

	//获取DispatcherServlet的映射信息
	//  /：拦截所有请求（包括静态资源（xx.js,xx.png）），但是不包括*.jsp；
	//  /*：拦截所有请求；连*.jsp页面都拦截；jsp页面是tomcat的jsp引擎解析的；
	@Override
	protected String[] getServletMappings() {
		// TODO Auto-generated method stub
		return new String[]{"/"};
	}

}
```

### 4.4.1 根容器处理service和dao层

```
//Spring的容器不扫描controller;父容器
@ComponentScan(value="com.enjoy",excludeFilters={
		@Filter(type=FilterType.ANNOTATION,classes={Controller.class})
})
public class JamesRootConfig {

}
```

```
@Service
public class OrderService   {
	public String goBuy(String orderId){
		return "orderId===="+orderId;
	}
}
```

### 4.4.2 servlet容器处理controller、视图解析及web相关

```
//SpringMVC只扫描Controller；子容器
//useDefaultFilters=false 禁用默认的过滤规则；
@ComponentScan(value="com.enjoy",includeFilters={
		@Filter(type=FilterType.ANNOTATION,classes={Controller.class})
},useDefaultFilters=false)
@EnableWebMvc
public class JamesAppConfig implements WebMvcConfigurer  {
	// WebMvcConfigurationSupport
	
	//定制视图解析器
	@Override
	public void configureViewResolvers(ViewResolverRegistry registry) {
		//比如我们想用JSP解析器,默认所有的页面都从/WEB-INF/AAA.jsp
		registry.jsp("/WEB-INF/pages/",".jsp");
	}
	
	//静态资源访问,静态资源交给tomcat来处理
	 @Override
	public void configureDefaultServletHandling(DefaultServletHandlerConfigurer configurer) {
		 configurer.enable();
	}
	
	 //拦截器
	 @Override
	public void addInterceptors(InterceptorRegistry registry) {
		registry.addInterceptor(new JamesInterceptor()).addPathPatterns("/**");
	}
}

```


```
@Controller
public class OrderController   {
	@Autowired
	OrderService orderService;
	
	@ResponseBody
	@RequestMapping("/buy")
	public String buy(){
		return orderService.goBuy("12345678");
	}
	
	//相当于会找 /WEB-INF/pages/ok.jsp
	@RequestMapping("/ok")
	public String ok(){
		return "ok";
	}
}

```

```
public class JamesInterceptor implements HandlerInterceptor{
	//在目标方法运行之间执行
	@Override
	public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
			throws Exception {
		System.out.println("----preHandle-------------");
		return true;
	}
	//在目标方法执行之后执行
	@Override
	public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler,
			ModelAndView modelAndView) throws Exception {
		System.out.println("----postHandle-------------");
	}

	//页面响应之后执行
	@Override
	public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex)
			throws Exception {
		System.out.println("----afterCompletion-------------");
	}

}

```

## 4.5 配置SpringMVC

### 4.5.1 传统xml配置方式

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:context="http://www.springframework.org/schema/context"
       xmlns:mvc="http://www.springframework.org/schema/mvc"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
    http://www.springframework.org/schema/beans/spring-beans.xsd
    http://www.springframework.org/schema/context
    http://www.springframework.org/schema/context/spring-context.xsd
    http://www.springframework.org/schema/mvc
    http://www.springframework.org/schema/mvc/spring-mvc-3.2.xsd">
    <!-- 配置SpringMVC -->
    <!-- 1.开启SpringMVC注解模式 -->
    <mvc:annotation-driven />

    <!-- 2.静态资源默认servlet配置
        (1)加入对静态资源的处理：js,gif,png
        (2)允许使用"/"做整体映射 -->
    <mvc:resources mapping="/resources/**" location="/resources/" />
    <mvc:default-servlet-handler />

    <!-- 3.定义视图解析器 -->
    <bean id="viewResolver"
          class="org.springframework.web.servlet.view.InternalResourceViewResolver">
        <property name="prefix" value="/WEB-INF/html/"></property>
        <property name="suffix" value=".html"></property>
    </bean>

    <!-- 文件上传解析器 -->
    <bean id="multipartResolver"
          class="org.springframework.web.multipart.commons.CommonsMultipartResolver">
        <property name="defaultEncoding" value="utf8"></property>
        <!--1024 * 1024 * 20 = 20M-->
        <property name="maxUploadSize" value="20971520"></property>
        <property name="maxInMemorySize" value="20971520"></property>
    </bean>


    <!-- 4.扫描web相关的bean -->
    <context:component-scan base-package="com.imooc.o2o.web" />

    <!-- 5.权限拦截器 -->
    <mvc:interceptors>
        <!--校验是否已登录了店家管理系统的拦截器-->
        <mvc:interceptor>
            <mvc:mapping path="/shopadmin/**"/>
            <mvc:exclude-mapping path="/shopadmin/addshopauthmap"/>
            <bean id="ShopInterceptor" class="com.imooc.o2o.interceptor.shopadmin.ShopLoginInterceptor"/>
        </mvc:interceptor>

        <!--校验是否对该商店有操作权限的拦截器-->
        <mvc:interceptor>
            <mvc:mapping path="/shopadmin/**"/>
            <mvc:exclude-mapping path="/shopadmin/shoplist"/>
            <mvc:exclude-mapping path="/shopadmin/getshoplist"/>
            <mvc:exclude-mapping path="/shopadmin/getshopinitinfo"/>
            <mvc:exclude-mapping path="/shopadmin/registershop"/>
            <mvc:exclude-mapping path="/shopadmin/shopoperation"/>
            <mvc:exclude-mapping path="/shopadmin/shopmanagement"/>
            <mvc:exclude-mapping path="/shopadmin/getshopmanagementinfo"/>
            <mvc:exclude-mapping path="/shopadmin/addshopauthmap"/>
            <bean id="ShopPermissionInterceptor" class="com.imooc.o2o.interceptor.shopadmin.ShopPermissionInterceptor"/>
        </mvc:interceptor>
    </mvc:interceptors>
</beans>
```

### 4.5.2 使用注解配置

```
//SpringMVC只扫描Controller；子容器
//useDefaultFilters=false 禁用默认的过滤规则；
@ComponentScan(value="com.enjoy",includeFilters={
		@Filter(type=FilterType.ANNOTATION,classes={Controller.class})
},useDefaultFilters=false)
@EnableWebMvc
public class JamesAppConfig implements WebMvcConfigurer  {
	// WebMvcConfigurationSupport
	
	//定制视图解析器
	@Override
	public void configureViewResolvers(ViewResolverRegistry registry) {
		//比如我们想用JSP解析器,默认所有的页面都从/WEB-INF/AAA.jsp
		registry.jsp("/WEB-INF/pages/",".jsp");
	}
	
	//静态资源访问,静态资源交给tomcat来处理
	 @Override
	public void configureDefaultServletHandling(DefaultServletHandlerConfigurer configurer) {
		 configurer.enable();
	}
	
	 //拦截器
	 @Override
	public void addInterceptors(InterceptorRegistry registry) {
		registry.addInterceptor(new JamesInterceptor()).addPathPatterns("/**");
	}
}

```


# 5.原生Servlet3.0异步请求

```
@WebServlet("/order")
public class JamesServlet  extends HttpServlet{
	//重写doget方法
	@Override
	protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
		System.out.println(Thread.currentThread()+"start.....");

		try {
			buyCards();
		} catch (InterruptedException e) {
			e.printStackTrace();
		}

		resp.getWriter().write("order sucesful....");
		System.out.println(Thread.currentThread()+" end..........");
	}

	public void buyCards() throws InterruptedException{
		System.out.println(Thread.currentThread()+".............");
		Thread.sleep(5000);//模拟业务操作
	}
}

```

从头到尾都是一个线程在操作：


```
Thread[http-nio-8080-exec-4,5,main]start.....
Thread[http-nio-8080-exec-4,5,main].............

Thread[http-nio-8080-exec-4,5,main] end..........
```


## 5.1 获取异步对象

参见Servlet3.1规范9.6节：

> 实现了 AsyncContext 接口的对象可从 ServletRequest 的一个 startAsync 方法中获得，一旦有了 AsyncContext对象，你就能够使用它的 complete()方法来完成请求处理，或使用下面描述的转发方法。

> 可以使用 AsyncContext 中dispatch()方法来转发请求


```
@WebServlet(value="/asyncOrder", asyncSupported = true)
public class OrderAsyncServlet extends HttpServlet{
	//支持异步处理asyncSupported = true
	//重写doget方法
	@Override
	protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
		//151个……
		System.out.println("主线程开始……"+Thread.currentThread()+"start....."+System.currentTimeMillis());
		AsyncContext startAsync = req.startAsync();

		startAsync.start(() -> {
				try {
					System.out.println("副线程开始……"+Thread.currentThread()+"start....."+System.currentTimeMillis());

					buyCards();
					startAsync.complete();
//					AsyncContext asyncContext = req.getAsyncContext();
//					ServletResponse response = asyncContext.getResponse();
					resp.getWriter().write("order sucesful....");
					System.out.println("副线程结束……"+Thread.currentThread()+"end....."+System.currentTimeMillis());

				} catch (InterruptedException e) {
					// TODO Auto-generated catch block
					e.printStackTrace();
				} catch (IOException e) {
					// TODO Auto-generated catch block
					e.printStackTrace();
				}
		});

		System.out.println("主线程结束……"+Thread.currentThread()+"end....."+System.currentTimeMillis());
		//主线程的资源断开……
	}

	public void buyCards() throws InterruptedException{
		System.out.println(Thread.currentThread()+".............");
		Thread.sleep(5000);//模拟业务操作
	}
}
```


之前一直出现异常：

```
java.lang.IllegalStateException: It is illegal to call this method if the current request is not in asynchronous mode (i.e. isAsyncStarted() returns false)
```

改进，注释掉两行，然后直接使用resp即可：

```
//	AsyncContext asyncContext = req.getAsyncContext();
//	ServletResponse response = asyncContext.getResponse();
	resp.getWriter().write("order sucesful....");
```

结果：

```
主线程开始……Thread[http-nio-8080-exec-7,5,main]start.....1578291454072
副线程开始……Thread[http-nio-8080-exec-3,5,main]start.....1578291454087
Thread[http-nio-8080-exec-3,5,main].............
主线程结束……Thread[http-nio-8080-exec-7,5,main]end.....1578291454087

副线程结束……Thread[http-nio-8080-exec-3,5,main]end.....1578291459088
```

## 5.2 异步请求原理

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/异步请求原理.png" width="620px" > </div><br>


# 6.SpringMVC异步处理

参见[官方文档](https://docs.spring.io/spring/docs/5.2.2.RELEASE/spring-framework-reference/web.html#mvc-ann-async)：

```
Spring MVC has an extensive integration with Servlet 3.0 asynchronous request processing:

DeferredResult and Callable return values in controller methods and provide basic support for a single asynchronous return value.

Controllers can stream multiple values, including SSE and raw data.

Controllers can use reactive clients and return reactive types for response handling.
```

```
@GetMapping("/quotes")
@ResponseBody
public DeferredResult<String> quotes() {
    DeferredResult<String> deferredResult = new DeferredResult<String>();
    // Save the deferredResult somewhere..
    return deferredResult;
}

// From some other thread...
deferredResult.setResult(result);
```

```
public Callable<String> processUpload(final MultipartFile file) {

    return new Callable<String>() {
        public String call() throws Exception {
            // ...
            return "someView";
        }
    };
}
```

## 6.1 Callable

```
@Controller
public class AsyncOrderController {
	
	@ResponseBody
	@RequestMapping("/order01")
	public Callable<String> order01(){
		System.out.println("主线程开始..."+Thread.currentThread()+"==>"+System.currentTimeMillis());
		
		Callable<String> callable = new Callable<String>() {
			@Override
			public String call() throws Exception {
				System.out.println("副线程开始..."+Thread.currentThread()+"==>"+System.currentTimeMillis());
				Thread.sleep(2000);
				System.out.println("副线程开始..."+Thread.currentThread()+"==>"+System.currentTimeMillis());
				return "order buy successful........";
			}
		};
		
		System.out.println("主线程结束..."+Thread.currentThread()+"==>"+System.currentTimeMillis());
		return callable;
	}

}

```

```
----preHandle-------------
主线程开始...Thread[http-nio-8080-exec-6,5,main]==>1578292702182
主线程结束...Thread[http-nio-8080-exec-6,5,main]==>1578292702191

副线程开始...Thread[MvcAsync1,5,main]==>1578292702228
副线程结束...Thread[MvcAsync1,5,main]==>1578292704229
----preHandle-------------
----postHandle-------------
----afterCompletion-------------
```

Callable处理流程：

```
Callable processing works as follows:

The controller returns a Callable.

Spring MVC calls request.startAsync() and submits the Callable to a TaskExecutor for processing in a separate thread.

Meanwhile, the DispatcherServlet and all filters exit the Servlet container thread, but the response remains open.

Eventually the Callable produces a result, and Spring MVC dispatches the request back to the Servlet container to complete processing.

The DispatcherServlet is invoked again, and processing resumes with the asynchronously produced return value from the Callable.
```

为什么有两次preHandle？

* 上面的Callable已经给出了解释
* 请求进入时拦截了一次，将Callable返回结果时，将请求重新派发给容器时又拦截了一次，所以进了两次拦截


```
----preHandle-------------/springmvc/order01

主线程开始...Thread[http-nio-8080-exec-4,5,main]==>1578293255604
主线程结束...Thread[http-nio-8080-exec-4,5,main]==>1578293255613
===========DispatcherServlet及Filter退出线程=======

===========等待Callable执行=============
副线程开始...Thread[MvcAsync1,5,main]==>1578293255650
副线程结束...Thread[MvcAsync1,5,main]==>1578293257651
===========Callable执行完成=============

===========再次收到之前重发过来的请求=============
----preHandle-------------/springmvc/order01
----postHandle-------------
----afterCompletion-------------
```



## 6.2 DeferredResult

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/订单需求.png" width="620px" > </div><br>

需求描述：以创建订单为例，tomcat启动线程1来完成一个请求，但实际上是订单服务才能创建订单，那么tomcat线程应该把请求转发给订单服务，使用消息中间件来处理，订单服务把处理结果也放到消息中间件，由tomcat的线程N拿到结果后，响应给客户端。

写一个队列为模拟消息中间件：

```
public class JamesDeferredQueue {
	
	private static Queue<DeferredResult<Object>> queue = new ConcurrentLinkedQueue<DeferredResult<Object>>();
	
	public static void save(DeferredResult<Object> deferredResult){
		queue.add(deferredResult);
	}
	
	public static DeferredResult<Object> get( ){
		return queue.poll();
	}

}

```

操作请求

```
@Controller
public class AsyncOrderController {
	
	//其实相当于我们说的tomcat的线程1，来处理用户请求，并将请求的操作放到Queue队列里
	@ResponseBody
	@RequestMapping("/createOrder")
	public DeferredResult<Object> createOrder(){
		DeferredResult<Object> deferredResult = new DeferredResult<>((long)10000, "create fail...");

		JamesDeferredQueue.save(deferredResult);
		
		return deferredResult;
	}
	
	////其实相当于我们说的tomcat的线程N，来处理用户请求，并将请求的操作放到Queue队列里
	@ResponseBody
	@RequestMapping("/create")
	public String create(){
		//创建订单（按真实操作应该是从订单服务取，这里直接返回）
		String order = UUID.randomUUID().toString();//模拟从订单服务获取的订单信息（免调接口）
		DeferredResult<Object> deferredResult = JamesDeferredQueue.get();
		deferredResult.setResult(order);
		return "create success, orderId == "+order;
	}
```



先运行createOrder，再运行create。

测试结果：

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/测试结果.jpg" width="620px" > </div><br>

通过create (tomcat线程N处理的订单结果)，异步返回给createOrder(tomcat线程1),两结果一致，异步返回了结果。


DeferredResult运行原理：

```
DeferredResult processing works as follows:

The controller returns a DeferredResult and saves it in some in-memory queue or list where it can be accessed.

Spring MVC calls request.startAsync().

Meanwhile, the DispatcherServlet and all configured filters exit the request processing thread, but the response remains open.

The application sets the DeferredResult from some thread, and Spring MVC dispatches the request back to the Servlet container.

The DispatcherServlet is invoked again, and processing resumes with the asynchronously produced return value.
```


