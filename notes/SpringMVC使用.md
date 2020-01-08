
# 1.SpringMVC介绍

## 1.1三层架构介绍

开发架构一般都是基于两种形式，一种是 C/S 架构，也就是客户端/服务器；另一种是 B/S 架构，也就是浏览器服务器。在 JavaEE 开发中，几乎全都是基于 B/S 架构的开发。那么在 B/S 架构中，系统标准的三层架构包括：表现层、业务层、持久层。

三层架构中，每一层各司其职，接下来我们就说说每层都负责哪些方面： 
* 表现层：
  - 也就是我们常说的web 层。它负责接收客户端请求，向客户端响应结果，通常客户端使用http 协议请求web 层，web 需要接收 http 请求，完成 http 响应。
  - 表现层包括展示层和控制层：控制层负责接收请求，展示层负责结果的展示。
  - 表现层依赖业务层，接收到客户端请求一般会调用业务层进行业务处理，并将处理结果响应给客户端。
  - 表现层的设计一般都使用 MVC 模型。（MVC 是表现层的设计模型，和其他层没有关系）
* 业务层：
  - 也就是我们常说的 service 层。它负责业务逻辑处理，和我们开发项目的需求息息相关。web 层依赖业务层，但是业务层不依赖 web 层。
  - 业务层在业务处理时可能会依赖持久层，如果要对数据持久化需要保证事务一致性。（也就是我们说的， 事务应该放到业务层来控制）
* 持久层：
  - 也就是我们是常说的 dao 层。负责数据持久化，包括数据层即数据库和数据访问层，数据库是对数据进行持久化的载体，数据访问层是业务层和持久层交互的接口，业务层需要通过数据访问层将数据持久化到数据库中。通俗的讲，持久层就是和数据库交互，对数据库表进行曾删改查的。


## 1.2MVC设计模式介绍

MVC 全名是 Model View Controller，是模型(model)－视图(view)－控制器(controller)的缩写， 是一种用于设计创建 Web 应用程序表现层的模式。MVC 中每个部分各司其职：
* Model（模型）：
  - 模型包含业务模型和数据模型，数据模型用于封装数据，业务模型用于处理业务。
* View（视图）：
  - 通常指的就是我们的 jsp 或者 html。作用一般就是展示数据的。
  - 通常视图是依据模型数据创建的。
* Controller（控制器）：
  - 是应用程序中处理用户交互的部分。作用一般就是处理程序逻辑的。


## 1.3SpringMVC介绍

* Spring MVC是什么？
  - SpringMVC 是一种基于 Java 的实现 MVC 设计模型的请求驱动类型的轻量级 Web 框架，属于 SpringFrameWork 的后续产品，已经融合在 Spring Web Flow 里面。Spring 框架提供了构建 Web 应用程序的全功能 MVC 模块。使用 Spring 可插入的 MVC 架构，从而在使用 Spring 进行 WEB 开发时，可以选择使用 Spring 的 Spring MVC 框架或集成其他 MVC 开发框架，如 Struts1(现在一般不用)，Struts2 等。
  - SpringMVC 已经成为目前最主流的 MVC 框架之一，并且随着 Spring3.0 的发布，全面超越 Struts2，成为最优秀的 MVC 框架。
  - 它通过一套注解，让一个简单的 Java 类成为处理请求的控制器，而无须实现任何接口。同时它还支持RESTful 编程风格的请求。 
  - 总之：Spring MVC和Struts2一样，都是为了解决表现层问题的web框架，它们都是基于MVC设计模式的。而这些表现层框架的主要职责就是处理前端HTTP请求。

* SpringMVC如何处理请求？
  - 不过千万不要把MVC设计模式和工程的三层结构混淆，三层结构指的是表现层、业务层、数据持久层。而MVC只针对表现层进行设计。

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/SpringMVC处理.png" width="620px" > </div><br>

# 2.SpringMVC使用

## 2.1 MVC中的C：配置前端控制器

在web.xml中添加DispatcherServlet的配置。

```
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xmlns="http://java.sun.com/xml/ns/javaee"
	xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_2_5.xsd"
	version="2.5">

	<!-- 学习前置条件 -->
	<!-- 问题1：web.xml中servelet、filter、listener、context-param加载顺序 -->
	<!-- 问题2：load-on-startup标签的作用，影响了servlet对象创建的时机 -->
	<!-- 问题3：url-pattern标签的配置方式有四种：/dispatcherServlet、 /servlet/*  、*  、/ ,以上四种配置，加载顺序是如何的-->
	<!-- 问题4：url-pattern标签的配置为/*报错，原因是它拦截了JSP请求，但是又不能处理JSP请求。为什么配置/就不拦截JSP请求，而配置/*，就会拦截JSP请求-->
	<!-- 问题5：配置了springmvc去读取spring配置文件之后，就产生了spring父子容器的问题 -->
	
	<!-- 配置前端控制器 -->
	<servlet>
		<servlet-name>springmvc</servlet-name>
		<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
		<!-- 设置spring配置文件路径 -->
		<!-- 如果不设置初始化参数，那么DispatcherServlet会读取默认路径下的配置文件 -->
		<!-- 默认配置文件路径：/WEB-INF/springmvc-servlet.xml -->
		<init-param>
			<param-name>contextConfigLocation</param-name>
			<param-value>classpath:springmvc.xml</param-value>
		</init-param>
		<!-- 指定初始化时机，设置为2，表示Tomcat启动时，DispatcherServlet会跟随着初始化 -->
		<!-- 如果不指定初始化时机，DispatcherServlet就会在第一次被请求的时候，才会初始化，而且只会被初始化一次 -->
		<load-on-startup>2</load-on-startup>
	</servlet>
	<servlet-mapping>
		<servlet-name>springmvc</servlet-name>
		<!-- URL-PATTERN的设置 -->
		<!-- 不要配置为/*,否则报错 -->
		<!-- 通俗解释：/*，会拦截整个项目中的资源访问，包含JSP和静态资源的访问，对于静态资源的访问springMVC提供了默认的Handler处理器 -->
		<!-- 但是对于JSP来讲，springmvc没有提供默认的处理器，我们也没有手动编写对于的处理器，此时按照springmvc的处理流程分析得知，它短路了 -->
		<url-pattern>/</url-pattern>
	</servlet-mapping>
	<!-- <servlet> -->
	<!-- <servlet-name>sss</servlet-name> -->
	<!-- <servlet-class>sss</servlet-class> -->
	<!-- </servlet> -->
	<!-- <servlet-mapping> -->
	<!-- <servlet-name>sss</servlet-name> -->
	<!-- <url-pattern>/sss</url-pattern> -->
	<!-- </servlet-mapping> -->
</web-app>
```

创建springmvc.xml。


## 2.2 MVC中的M：处理器

### 2.2.1 编写处理器

处理器开发方式有多种：实现HttpRequestHandler接口、实现Controller接口、注解方式等。推荐注解方式。

注解方式必要的注解主要有以下两个：
* @Controller注解：在类上添加该注解，指定该类为一个请求处理器，不需要实现任何接口或者继承任何类。
* @RequestMapping注解：在方法上或者类上添加该注解，指定请求的url由该方法处理。如果url-pattern配置的是*的话，url中的“”可以加也可以不加。


处理器的返回值是ModelAndView对象，该对象的具体理解如下：
ModelAndView：方法返回值对象，该对象包含两个功能：
* 一个是将数据存储到Request域中
* 一个是设置响应视图，比如将视图设置为“/WEB-INF/jsp/item-list.jsp”


```
@Controller
public class ItemController {

	@RequestMapping("/queryItem")
	public ModelAndView queryItem() throws Exception {
		
		List<Item> itemList = new ArrayList<>();
		
		//商品列表
		Item item_1 = new Item();
		item_1.setName("联想笔记本");
		item_1.setPrice(6000f);
		item_1.setDetail("ThinkPad T430 联想笔记本电脑！");
		
		Item item_2 = new Item();
		item_2.setName("苹果手机");
		item_2.setPrice(5000f);
		item_2.setDetail("iphone6苹果手机！");
		
		itemList.add(item_1);
		itemList.add(item_2);
		//创建modelandView对象
		ModelAndView modelAndView = new ModelAndView();
		//添加model
		modelAndView.addObject("itemList", itemList);
		//添加视图
		modelAndView.setViewName("/WEB-INF/jsp/item-list.jsp");
		return modelAndView;
	}

}
```

### 2.2.2 配置处理器

使用组件扫描器省去在spring容器配置每个controller类的繁琐。

使用<context:component-scan>自动扫描标记@controller的控制器类，在springmvc.xml配置如下：

```
<!-- 扫描controller注解,多个包中间使用半角逗号分隔 -->
<context:component-scan base-package="com.enjoy.springmvc.controller"/>
```

## 2.3 MVC中的V：创建视图

新建jsp文件，并将item-list.jsp复制到工程的/WEB-INF/jsp目录下


# 3.三大组件配置

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

## 3.1 注解映射器和适配器

### 3.1.1 通过bean标签配置

* RequestMappingHandlerMapping：注解式处理器映射器
  - 对类中标记@ResquestMapping的方法进行映射，根据ResquestMapping定义的url匹配ResquestMapping标记的方法，匹配成功返回HandlerMethod对象给前端控制器，HandlerMethod对象中封装url对应的方法Method。 

* RequestMappingHandlerAdapter：注解式处理器适配器
  - 对标记@ResquestMapping的方法进行适配。


```
<!--注解映射器 -->
<bean class="org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping"/>
<!--注解适配器 -->
<bean class="org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter"/>
```

### 3.1.2 通过mvc标签配置

```
<mvc:annotation-drivern />
```

mvc:annotation-drivern标签的作用，就是往spring容器中注册了以下的一些BeanDefinition：
* ContentNegotiationManagerFactoryBean
* RequestMappingHandlerMapping
* ConfigurableWebBindingInitializer
* RequestMappingHandlerAdapter
* CompositeUriComponentsContributorFactoryBean
* ConversionServiceExposingInterceptor
* MappedInterceptor
* ExceptionHandlerExceptionResolver
* ResponseStatusExceptionResolver
* DefaultHandlerExceptionResolver
* BeanNameUrlHandlerMapping
* HttpRequestHandlerAdapter
* SimpleControllerHandlerAdapter
* HandlerMappingIntrospector


## 3.2 视图解析器

在springmvc.xml文件配置如下：

```
	<bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
            <!-- 该视图解析器，默认的视图类就是JstlView，可以不写 -->
		<property name="viewClass"
			value="org.springframework.web.servlet.view.JstlView" />
		<property name="prefix" value="/WEB-INF/jsp/" />
		<property name="suffix" value=".jsp" />
	</bean>
```

* InternalResourceViewResolver：默认支持JSP视图解析
* viewClass：JstlView表示JSP模板页面需要使用JSTL标签库，所以classpath中必须包含jstl的相关jar 包。此属性可以不设置，默认为JstlView。
* prefix 和suffix：查找视图页面的前缀和后缀，最终视图的址为：前缀+逻辑视图名+后缀，逻辑视图名需要在controller中返回的ModelAndView指定，比如逻辑视图名为hello，则最终返回的jsp视图地址 “WEB-INF/jsp/hello.jsp”


# 4.SSM框架整合

将工程的三层结构中的JavaBean分别使用Spring容器（通过XML方式）进行管理。

1、整合持久层mapper，包括数据源、会话工程及mapper代理对象的整合；
2、整合业务层Service，包括事务及service的bean的配置；
3、整合表现层Controller，直接使用springmvc的配置。
4、Web.xml加载spring容器（包含多个XML文件）

Spring核心配置文件：
* applicationContext-dao.xml
* applicationContext-service.xml
* springmvc.xml

## 4.1 整合Dao层

applicationContext-dao.xml

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:context="http://www.springframework.org/schema/context" xmlns:p="http://www.springframework.org/schema/p"
	xmlns:aop="http://www.springframework.org/schema/aop" xmlns:tx="http://www.springframework.org/schema/tx"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
	http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd
	http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop.xsd http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx.xsd
	http://www.springframework.org/schema/util http://www.springframework.org/schema/util/spring-util.xsd">

	<!-- 加载db.properties -->
	<context:property-placeholder location="classpath:db.properties" />
	<!-- 配置数据源 -->
	<bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource"
		destroy-method="close">
		<property name="driverClassName" value="${jdbc.driver}" />
		<property name="url" value="${jdbc.url}" />
		<property name="username" value="${jdbc.username}" />
		<property name="password" value="${jdbc.password}" />
		<property name="maxActive" value="30" />
		<property name="maxIdle" value="5" />
	</bean>

	<!-- 配置SqlSessionFacotory -->
	<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
		<!-- 加载mybatis的配置文件（如果配置文件中没有配置项，可以忽略该文件） -->
		<property name="configLocation" value="classpath:mybatis/SqlMapConfig.xml" />
		<!-- 配置数据源 -->
		<property name="dataSource" ref="dataSource" />
	</bean>

	<!-- 配置mapper扫描器，SqlSessionConfig.xml中的mapper配置去掉 -->
	<bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
		<!-- 指定扫描的包 -->
		<property name="basePackage" value="com.kkb.ssm.mapper" />
	</bean>
</beans>
```

## 4.2 整合Service

applicationContext-service.xml

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:context="http://www.springframework.org/schema/context"                  xmlns:p="http://www.springframework.org/schema/p"
	xmlns:aop="http://www.springframework.org/schema/aop" 
      xmlns:tx="http://www.springframework.org/schema/tx"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
	http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd
	http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop.xsd http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx.xsd
	http://www.springframework.org/schema/util http://www.springframework.org/schema/util/spring-util.xsd">

	<!-- 扫描Service -->
	<context:component-scan base-package="com.kkb.ssm.service" />

	<!-- 配置事务 -->
	<!-- 事务管理器，对mybatis操作数据库进行事务控制，此处使用jdbc的事务控制 -->
	<bean id="transactionManager"
		class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
		<!-- 指定要进行事务管理的数据源 -->
		<property name="dataSource" ref="dataSource"></property>
	</bean>

	<!-- 通知 -->
	<tx:advice id="txAdvice" transaction-manager="transactionManager">
		<tx:attributes>
			<!-- 传播行为 -->
			<tx:method name="save*" propagation="REQUIRED" />
			<tx:method name="add*" propagation="REQUIRED" />
			<tx:method name="insert*" propagation="REQUIRED" />
			<tx:method name="delete*" propagation="REQUIRED" />
			<tx:method name="del*" propagation="REQUIRED" />
			<tx:method name="remove*" propagation="REQUIRED" />
			<tx:method name="update*" propagation="REQUIRED" />
			<tx:method name="modify*" propagation="REQUIRED" />
			<tx:method name="find*" read-only="true" />
			<tx:method name="query*" read-only="true" />
			<tx:method name="select*" read-only="true" />
			<tx:method name="get*" read-only="true" />
		</tx:attributes>
	</tx:advice>

	<!-- aop -->
	<aop:config>
		<aop:advisor advice-ref="txAdvice"
			pointcut="execution(* com.kkb.ssm.service.impl.*.*(..))" />
	</aop:config>

</beans>
```

## 4.3 整合Controller

Spring和springmvc之间无需整合，直接用springmvc的配置。


```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:p="http://www.springframework.org/schema/p"
	xmlns:context="http://www.springframework.org/schema/context"
	xmlns:mvc="http://www.springframework.org/schema/mvc"
	xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/mvc http://www.springframework.org/schema/mvc/spring-mvc.xsd
         http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd">

	<!-- 配置处理器映射器和处理器适配器 -->
	<mvc:annotation-driven />

	<!-- 配置视图解析器 -->
	<bean
		class="org.springframework.web.servlet.view.InternalResourceViewResolver">
		<property name="prefix" value="/WEB-INF/jsp/" />
		<property name="suffix" value=".jsp" />
	</bean>
	<!-- 使用注解的handler可以使用组件扫描器，加载handler -->
	<context:component-scan base-package="com.kkb.ssm.controller" />
</beans>
```


## 4.4 web.xml

```
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xmlns="http://java.sun.com/xml/ns/javaee" xmlns:web="http://java.sun.com/xml/ns/javaee/web-app_2_5.xsd"
	xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_2_5.xsd"
	id="WebApp_ID" version="2.5">
	<display-name>ssm</display-name>

    <!-- 加载spring容器 -->
    <context-param>
    	<param-name>contextConfigLocation</param-name>
    	<param-value>
    		classpath:spring/applicationContext-*.xml
    	</param-value>
    </context-param>
    <listener>
    	<listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>

	<!-- 配置springmvc的前端控制器 -->
	<servlet>
		<servlet-name>springmvc</servlet-name>
		<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
		<init-param>
			<param-name>contextConfigLocation</param-name>
			<param-value>classpath:spring/springmvc.xml</param-value>
		</init-param>
		<load-on-startup>1</load-on-startup>
	</servlet>
	<servlet-mapping>
		<servlet-name>springmvc</servlet-name>
		<url-pattern>/</url-pattern>
	</servlet-mapping>

</web-app>
```


# 5.Controller方法返回值

## 5.1 不使用注解修饰

### 5.1.1 返回ModelAndView

controller方法中定义ModelAndView对象并返回，对象中可添加model数据、指定view。

### 5.1.2 返回void

在controller方法形参上可以定义request和response，使用request或response指定响应结果：

```
void service(HttpServletRequest request,HttpServletResponse response){}
```

* 1、使用request转发向页面，如下：
request.getRequestDispatcher("页面路径").forward(request, response);

* 2、也可以通过response页面重定向：
response.sendRedirect("url")

* 3、也可以通过response指定响应结果，例如响应json数据如下：
response.setCharacterEncoding("utf-8");
response.setContentType("application/json;charset=utf-8");
response.getWriter().write("json串");

### 5.1.3 返回字符串

* 逻辑视图名
return "item/item-list";

* redirect重定向
  - return "redirect:testRedirect";
  - redirect:
	相当于“response.sendRedirect()”
	浏览器URL发生改变
	Request域不能共享

* forward转发
  - return "forward:testForward";
  - forward：
	相当于“request.getRequestDispatcher().forward(request,response)”
	浏览器URL不发送改变
	Request域可以共享

## 5.2 使用注解修饰

### 5.2.1 返回带@ResponseBody注解的值

#### 5.2.1.1 @ResponseBody注解和@RequestBody注解介绍

* @ResponseBody的作用：
  - ResponseBody注解可以通过内置的9种HttpMessageConverter，匹配不同的Controller返回值类型，然后进行不同的消息转换处理。
  - 将转换之后的数据放到HttpServletResponse对象的响应体返回到页面，
  - 不同的HttpMessageConverter处理的数据，指定的ContentType值也不同。

* @RequestBody注解的作用和@ResponseBody注解正好相反，它是处理请求参数的Http消息转换的。

#### 5.2.1.2 常用的HttpMessageConverter

* MappingJacksonHttpMessageConverter处理POJO类型返回值
  - MappingJacksonHttpMessageConverter是专门处理POJO类型的。
  - 默认使用MappingJackson的JSON处理能力，将后台返回的Java对象（POJO类型），转为JSON格式输出到页面
  - 将响应体的Content-Type设置为application/json；charset=utf-8
* StringHttpMessageConverter处理String类型返回值
  - StringHttpMessageConverter是专门处理String类型的。
  - 调用response.getWriter()方法将String类型的字符串写回给调用者。 
  - 将响应体的Content-Type设置为text/plain；charset=utf-8


# 6.@RequestMapping

通过RequestMapping注解可以定义不同的处理器映射规则。

## 6.1 URL路径映射

@RequestMapping(value="/item")或@RequestMapping("/item"）

value的值是数组，可以将多个url映射到同一个方法

@RequestMapping(value={"/item",”/queryItem”})

## 6.2 窄化请求映射

在class上添加@RequestMapping(url)指定通用请求前缀， 限制此类下的所有方法的访问请求url必须以请求前缀开头，通过此方法对url进行模块化分类管理。

## 6.3 请求方法限定

* 限定GET方法
  - @RequestMapping(method = RequestMethod.GET)
  - 如果通过Post访问则报错：
HTTP Status 405 - Request method 'POST' not supported
  - 例如：
@RequestMapping(value="/editItem",method=RequestMethod.GET)

* 限定POST方法
  - @RequestMapping(method = RequestMethod.POST)
  - 如果通过Post访问则报错：
HTTP Status 405 - Request method 'GET' not supported

* GET和POST都可以
  - @RequestMapping(method={RequestMethod.GET,RequestMethod.POST})


# 7.请求参数绑定

## 7.1 什么是请求参数绑定

* 请求参数格式
	默认是key/value格式，比如：  http://XXXXX?id=1&type=301
* 请求参数值的数据类型
	都是字符串类型的各种值
* 请求参数值要绑定的目标类型
	Controller类中的方法参数，比如简单类型、POJO类型、集合类型等。
* SpringMVC内置的参数解析组件
	默认内置了24种参数解析组件（ArgumentResolver）
* 什么是参数绑定？
	就是将请求参数串中的value值获取到之后，再进行类型转换，然后将转换后的值赋值给Controller类中方法的形参，这个过程就是参数绑定。

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/请求参数绑定.jpg" width="620px" > </div><br>


## 7.2 默认支持的参数类型（Servlet API支持）

Controller方法形参中可以随时添加如下类型的参数，处理适配器会自动识别并进行赋值。

* HttpServletRequest
通过request对象获取请求信息
* HttpServletResponse
通过response处理响应信息
* HttpSession
通过session对象得到session中存放的对象
* InputStream、OutputStream
* Reader、Writer
* Model/ModelMap
ModelMap继承自LinkedHashMap，Model是一个接口，它们的底层实现都是同一个类（BindingAwareModelMap），作用就是向页面传递数据，相当于Request的作用，如下：

```
//调用service查询商品信息
Item item = service.queryItemById(id);
model.addAttribute("item", item);
```

## 7.3 绑定简单数据类型

### 7.3.1 简单类型参数绑定方式

* 简单类型指的就是8种基本类型数据以及它们的包装类，还有String类型。
* 在springmvc中，对于java简单类型的参数，推荐的参数绑定方式有两种：
1、直接绑定
2、注解绑定


### 7.3.2 直接绑定

* 使用要求
  - http请求参数的key和controller方法的形参名称一致
* 请求URL
  - http://localhost:8080/xxx/findItem?id=1
* Controller方法
  - Controller的形参为Integer id,它和请求参数的key一致，所以直接绑定成功
 
```
    @RequestMapping(value = "/findItem")
	public String findItem(Integer id) {
         System.out.println("接收到的请求参数是："+ id);
		return "success";
	}
```

### 7.3.3 注解绑定

* 使用要求
  - 请求参数的key和controller方法的形参名称不一致时，需要使用@RequestParam注解才能将请求参数绑定成功。
* 请求URL
  - http://localhost:8080/xxx/findItem?itemid=1
* Controller方法
  - Controller的形参为Integer id,它和请求的参数不一致，要使用@RequestParam注解才能绑定成功

```
     @RequestMapping(value = "/findItem")
	// 通过@RequestParam注解绑定简单类型
	public String findItem(@RequestParam("itemid") Integer id) {
		  System.out.println("接收到的请求参数是："+ id);
		  return "success";
	}
```

* RequestParam注解介绍
  - value：参数名字，即入参的请求参数名字，如value=“itemid”表示请求的参数中的名字为itemid的参数的值将传入；
  - required：是否必须，默认是true，表示请求中一定要有相应的参数，否则将报；
TTP Status 400 - Required Integer parameter 'XXXX' is not present
  - defaultValue：默认值，表示如果请求中没有同名参数时的默认值


## 7.4 绑定POJO类型

* 使用要求
  - 控制器方法的参数类型是 POJO 类型。
  - 要求表单中参数名称和 POJO 类的属性名称保持一致。
* 请求URL
  - http://localhost:8080/xxx/updateItem?id=1&name=iphone&price=1000
* Controller方法
 

```
public class Items {

  private Integer id;
  private String name;
  private Float price;
  private String pic;
  private Date createTime;
}
```

```
	@RequestMapping("/updateItem")
	public String updateItem(Integer id,Item item) {
		
         System.out.println("接收到的请求参数是："+ id);
         System.out.println("接收到的请求参数是："+ item);
		return "success";
	}
```

## 7.5 绑定包装POJO

包装POJO类，依然是一个POJO，只是说为了方便沟通，将POJO中包含另一个POJO的这种类，称之为包装POJO。

* 包装对象

```
public class ItemQueryVO {
	//商品信息
	private Item item;
}
```

* 页面定义（item-list.jsp）

```
查询条件：
		<table width="100%" border=1>
			<tr>
				<td>商品名称：<input type="text" name="item.name" /></td>
				<td><input type="submit" value="查询" /></td>
			</tr>
		</table>
```

* Controller方法


## 7.6 使用简单类型数组接收参数

* 使用要求
  - 通过HTTP请求批量传递简单类型数据的情况，Controller方法中可以用String[]或者pojo的String[]属性接收（两种方式任选其一），但是不能使用集合接收
* 请求URL
  - http://localhost:8080/xxx/deleteItem?id=1&id=2&id=3
* Controller方法

```
    @RequestMapping("/deleteItem")
	public String deleteitem(Integer[] itemId){
		
		return "success";
	}
```

## 7.7 使用POJO类型集合或者数组接收参数

* 使用要求
  - 批量传递的请求参数，最终要使用List<POJO>来接收，那么这个List<POJO>必须放在另一个POJO类中。
* 接收商品列表的POJO

```
public class ItemQueryVO {

	// 商品信息
	private Item item;
	// 其他信息

	// 商品信息集合
	private List<Item> itemList;
}
```

* 请求URL
  - http://localhost:8080/xxx/batchUpdateItem?itemList[0].id=1&itemList[0].name=iphone&itemList[0].price=1000&itemList[1].id=2& itemList[1].name=iphone x& itemList[1].price=2000
* Controller方法
```
	@RequestMapping("/batchUpdateItem")
	public String batchUpdateItem(ItemQueryVO vo) {
		return "success";
	}
```

## 7.8 自定义参数绑定

* 请求URL
  - http://localhost:8080/xxx/saveItem?date=2018-08-12
* Controller方法
  - 但是如果将date参数的类型由String改为Date，则报错
```
    @RequestMapping("/saveItem")
	public String saveItem(String date){
		System.out.println("接收到的请求参数是："+ date);
		return "success";
	}
```

* 自定义Converter

```
public class DateConverter implements Converter<String, Date> {

	@Override
	public Date convert(String source) {
		SimpleDateFormat simpleDateFormat = new SimpleDateFormat("yyyy-MM-dd");
		try {
			return simpleDateFormat.parse(source);
		} catch (ParseException e) {
			e.printStackTrace();
		}
		return null;
	}
}
```

* 配置Converter
  - 在springmvc.xml中，进行以下配置

```
	<!-- 加载注解驱动 -->
	<mvc:annotation-driven conversion-service="conversionService"/>
	<!-- 转换器配置 -->
	<bean id="conversionService"
		class="org.springframework.format.support.FormattingConversionServiceFactoryBean">
		<property name="converters">
			<set>
				<bean class="com.kkb.ssm.controller.converter.DateConverter"/>
			</set>
		</property>
	</bean>
```


# 8.@ControllerAdvice注解

## 8.1 @ControllerAdvice

* 该注解顾名思义是一个增强器，是对注解了@Controller注解的类进行增强。
* 该注解使用@Component注解，这样的话当我们使用<context:component-scan>扫描时也能扫描到。
* 该注解内部使用@ExceptionHandler、@InitBinder、@ModelAttribute注解的方法会应用到所有的Controller类中 @RequestMapping注解的方法。


```
@ControllerAdvice
public class MyControllerAdvice {

	//应用到所有@RequestMapping注解方法，在其执行之前把返回值放入ModelMap中
	@ModelAttribute
	public Map<String, Object> ma(){
		Map<String, Object> map = new HashMap<>();
		map.put("name", "tom");
		return map;
	}
	
	//应用到所有【带参数】的@RequestMapping注解方法，在其执行之前初始化数据绑定器
	@InitBinder
	public void initBinder(WebDataBinder dataBinder) {
	}
	
	//应用到所有@RequestMapping注解的方法，在其抛出指定异常时执行
	@ExceptionHandler(Exception.class)
	@ResponseBody
	public String handleException(Exception e) {
		return e.getMessage();
	}
}
```

## 8.2 @ModelAttribute

* 该注解特点：主要作用于ModelMap这个模型对象的，用于在视图中显示数据。
* 该注解注意事项：和@ResponseBody注解的使用是互斥的。
* 该注解有两个用途：
  - 作用于方法（有没有@RequestMapping注解的方法都可以）
    * 非@RequestMapping注解方法
      - 执行时机：在本类内所有@RequestMapping注解方法之前执行。
      - 作用：
        * 如果方法有返回值，则直接将该返回值放入ModelMap中，key可以指定。
        * 如果方法没有返回值，则可以利用它的执行时机这一特点，做一些预处理。
    * 有@RequestMapping注解方法
      - 作用：
        * 返回值会放入ModelMap中
        * 逻辑视图名称根据请求URL生成，比如URL：item/itemList，那么这个URL就是逻辑视图名称
  - 作用于方法参数（有@RequestMapping注解的方法）
    * STEP1：获取ModelMap中指定的数据（由@ModelAttribute注解的方法产生）
    * STEP2：将使用该注解的参数对象放到ModelMap中。
    * 如果STEP1和STEP2中的对象在ModelMap中key相同，则合并这两部分对象的数据。


## 8.3 @InitBinder

* 由 @InitBinder 标识的方法，可以通过PropertyEditor解决数据类型转换问题，比如String-->Date类型。
* 不过PropertyEditor只能完成String-->任意类型的转换，这一点不如Converter灵活，Converter可以完成任意类型-->任意类型的转换。
* 它可以对 WebDataBinder 对象进行初始化，WebDataBinder 是 DataBinder 的子类,用于完成由表单字段到 JavaBean 属性的绑定。
* 注意事项：
  - @InitBinder方法不能有返回值,它必须声明为void。
  - @InitBinder方法的参数通常是 WebDataBinder。


## 8.4 @ExceptionHandler

* 这个注解表示Controller中任何一个@RequestMapping方法发生异常，则会被注解了@ExceptionHandler的方法拦截到。
* 拦截到对应的异常类则执行对应的方法，如果都没有匹配到异常类，则采用近亲匹配的方式。


# 9.异常处理器

springmvc在处理请求过程中出现异常信息交由异常处理器进行处理，自定义异常处理器可以实现一个系统的异常处理逻辑。

## 9.1 异常理解

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/异常.png" width="620px" > </div><br>

* 异常包含编译时异常和运行时异常，其中编译时异常也叫预期异常。
* 运行时异常
  - 运行时异常只有在项目运行的情况下才会发现，编译的时候不需要关心。
  - 运行时异常，比如：空指针异常、数组越界异常，对于这样的异常，只能通过程序员丰富的经验来解决和测试人员不断的严格测试来解决。
  - 运行时异常：都是RuntimeException类及其子类异常，如NullPointerException(空指针异常)、IndexOutOfBoundsException(下标越界异常)等，这些异常是不检查异常，程序中可以选择捕获处理，也可以不处理。这些异常一般是由程序逻辑错误引起的，程序应该从逻辑角度尽可能避免这类异常的发生。
  - 运行时异常的特点是Java编译器不会检查它，也就是说，当程序中可能出现这类异常，即使没有用try-catch语句捕获它，也没有用throws子句声明抛出它，也会编译通过。
* 编译时异常
  - 编译时异常，比如：数据库异常、文件读取异常、自定义异常等。对于这样的异常，必须使用try catch代码块或者throws关键字来处理异常。
  - 非运行时异常 （编译异常）：是RuntimeException以外的异常，类型上都属于Exception类及其子类。从程序语法角度讲是必须进行处理的异常，如果不处理，程序就不能编译通过。如IOException、SQLException等以及用户自定义的Exception异常，一般情况下不自定义检查异常。


## 9.2 异常处理思路

系统中异常包括两类：预期异常（编译时异常）和运行时异常RuntimeException，前者通过捕获异常从而获取异常信息，后者主要通过规范代码开发、测试通过手段减少运行时异常的发生。

系统的dao、service、controller出现都通过throws Exception向上抛出，最后由springmvc前端控制器交由异常处理器进行异常处理，如下图，全局范围只有一个异常处理器：


<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/异常处理体系.png" width="620px" > </div><br>


## 9.3 自定义异常类

为了区别不同的异常通常根据异常类型自定义异常类，这里我们创建一个自定义系统异常，如果controller、service、dao抛出此类异常说明是系统预期处理的异常信息。


```
public class BusinessException extends Exception {
	
	private static final long serialVersionUID = 1L;

	//异常信息
	private String message;
	
	public BusinessException(String message) {
		super(message);
		this.message = message;
	}

	public String getMessage() {
		return message;
	}

	public void setMessage(String message) {
		this.message = message;
	}
}
```

## 9.4 自定义异常处理器

```
public class BusinessExceptionResolver implements HandlerExceptionResolver {

	@Override
	public ModelAndView resolveException(HttpServletRequest request,
			HttpServletResponse response, Object handler, Exception ex) {
		
		//自定义预期异常
		BusinessException businessException = null; 
		//如果抛出的是系统自定义的异常
		if(ex instanceof BusinessException){
			businessException = (BusinessException) ex;
		}else{
			businessException = new BusinessException("未知错误");
		}
		
		ModelAndView modelAndView = new ModelAndView();
		//把错误信息传递到页面
		modelAndView.addObject("message", businessException.getMessage());
		//指向错误页面
		modelAndView.setViewName("error");
		return modelAndView;
	}
}
```

## 9.5 错误页面

```
<%@ page language="java" contentType="text/html; charset=UTF-8"
    pageEncoding="UTF-8"%>
<!DOCTYPE html PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN" "http://www.w3.org/TR/html4/loose.dtd">
<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=UTF-8">
<title>错误信息</title>
</head>
<body>
${message }
</body>
</html>
```

## 9.6 异常处理器配置


在springmvc.xml中加入以下代码：

```
<!-- 自定义异常处理器（全局） -->
<bean class="com.kkb.ssm.resolver.BusinessExceptionResolver"/>
```

## 9.7 抛出异常示例

```
@RequestMapping(value = "/showItemEdit")
public String showItemEdit(Integer id,Model model) throws Exception{

	// 查询要显示的商品内容
	Item item = itemService.queryItemById(id);

	if(item == null) throw new BusinessException("查询不到商品无法修改");

	model.addAttribute("item", item);
	// 由于配置了ViewResolver，所以此处只写逻辑视图名称即可
	return "item/item-edit";
}
```


# 10.上传图片

SpringMVC文件上传的实现，是由commons-fileupload这个jar包实现的。


```
<dependency>
	<groupId>commons-fileupload</groupId>
	<artifactId>commons-fileupload</artifactId>
	<version>xxx</version>
</dependency>
```

文件上传需要指定enctype=”multipart/form-data”。


在springmvc.xml中配置multipart类型解析器：

```
<!-- multipart类型解析器，文件上传 -->
<bean id="multipartResolver" class="org.springframework.web.multipart.commons.CommonsMultipartResolver">
	<!-- 上传文件的最大尺寸 5M-->
	<property name="maxUploadSize" value="5242880"/>
</bean>
```

注意：在图片虚拟目录 中，一定将图片目录分级创建（提高i/o性能），一般我们采用按日期(年、月、日)进行分级创建。


在tomcat安装目录中的conf/server.xml文件中，添加虚拟目录：
```
<Context docBase="E:\03-teach\07-upload\temp" path="/pic" reloadable="false"/>

```


# 11.JSON数据交互

* 为什么使用JSON进行数据交互
  - JSON数据格式比较简单、解析比较方便，在接口调用及HTML页面Ajax调用时较常用。
* JSON交互方式
  - 请求是KV，响应是JSON（推荐使用）
  - 请求是JSON，响应是JSON
* 依赖包

```
<dependency>
	<groupId>com.fasterxml.jackson.core</groupId>
	<artifactId>jackson-databind</artifactId>
	<version>xxx</version>
</dependency>
```

## 11.1 输入key/value、输出JSON

* JSP页面

```
function responseKV(){
	$.ajax({
		type:"post",
		url:'${pageContext.request.contextPath }/responseKV',
		//输入是key/value时，默认就指定好了contentType了，不需要再指定了
		//contentType:'application/json;charset=utf-8',
		//data为key/value形式
		data:'name=json测试&price=999',
		success:function(data){
			alert(data);
		}
	});
}
```

* Controller类

```
// 输入是key/value，输出是json
// @ResponseBody 将返回值转成json串响应给前台
@RequestMapping("/responseKV")
@ResponseBody
public Item responseKV(Item item) {
return item;
}
```

* 页面控制台输出

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/页面.jpg" width="520px" > </div><br>


## 11.2 输入JSON、输出JSON

* JSP页面

```
function requestJson(){
	
	$.ajax({
		type:"post",
		url:'${pageContext.request.contextPath }/requestJson',
		//输入是json是 ，需要指定contentType为application/json
		contentType:'application/json;charset=utf-8',
		data:'{"name":"json测试","price":999}',
		success:function(data){
			alert(data.name);
		}
	});
}
```

* Controller类

```
@Controller
public class JsonController {

	// 输入是json，输出是json
	// @RequestBody 将请求的json串转成java对象
	// @ResponseBody 将返回值转成json串响应给前台
	@RequestMapping("/requestJson")
    @ResponseBody
	public Item requestJson(@RequestBody Item item) {

		return item;
	}
}
```

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/JSON请求.jpg" width="520px" > </div><br>

# 12.Mock测试（模拟测试）

* 什么是mock测试
  - 在测试过程中，对于某些不容易构造或者不容易获取的对象，用一个虚拟的对象来创建以便测试的测试方法，就是mock测试。
    * Servlet、Request、Response等Servlet API相关对象本来是由Servlet容器（tomcat）创建的。
  - 这个虚拟的对象就是mock对象。
  - mock对象就是真实对象在调试期间的代替品。
* 为什么使用mock测试
  - 避免开发模块之间的耦合
  - 轻量、简单、灵活

## 12.1 MockMVC介绍

基于RESTful风格的SpringMVC的测试，我们可以测试完整的Spring MVC流程，即从URL请求到控制器处理，再到视图渲染都可以测试。

* MockMVCBuilder
  - MockMvcBuilder是用来构造MockMvc的构造器
  - 其主要有两个实现：StandaloneMockMvcBuilder和DefaultMockMvcBuilder，分别对应之前的两种测试方式。
  - 对于我们来说直接使用静态工厂MockMvcBuilders创建即可
* MockMVCBuilders
  - 负责创建MockMVCBuilder对象
  - 有两种创建方式
    * standaloneSetup(Object... controllers): 
      通过参数指定一组控制器，这样就不需要从上下文获取了。
    * webAppContextSetup(WebApplicationContext wac)
	  指定WebApplicationContext，将会从该上下文获取相应的控制器并得到相应的MockMvc
* MockMvc
  - 对于服务器端的Spring MVC测试支持主入口点。
  - 通过MockMVCBuilder构造
  - MockMVCBuilder由MockMVCBuilders建造者的静态方法去建造。
  - 核心方法：perform(RequestBuilder rb)--- 执行一个RequestBuilder请求，会自动执行SpringMVC的流程并映射到相应的控制器执行处理，该方法的返回值是一个ResultActions；
* ResultActions
  - andExpect：添加ResultMatcher验证规则，验证控制器执行完成后结果是否正确；
  - andDo：添加ResultHandler结果处理器，比如调试时打印结果到控制台；
  - andReturn：最后返回相应的MvcResult；然后进行自定义验证/进行下一步的异步处理；
* MockMvcRequestBuilders
  - 用来构建请求的
  - 其主要有两个子类MockHttpServletRequestBuilder和MockMultipartHttpServletRequestBuilder（如文件上传使用），即用来Mock客户端请求需要的所有数据。
* MockMvcResultMatchers
  - 用来匹配执行完请求后的结果验证
  - 如果匹配失败将抛出相应的异常
  - 包含了很多验证API方法
* MockMvcResultHandlers
  - 结果处理器，表示要对结果做点什么事情
  - 比如此处使用MockMvcResultHandlers.print()输出整个响应结果信息。
* MvcResult
  - 单元测试执行结果，可以针对执行结果进行自定义验证逻辑。


## 12.2 MockMVC使用

依赖：

```
        <!-- spring 单元测试组件包 -->
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-test</artifactId>
			<version>5.0.7.RELEASE</version>
		</dependency>

		<!-- 单元测试Junit -->
		<dependency>
			<groupId>junit</groupId>
			<artifactId>junit</artifactId>
			<version>4.12</version>
		</dependency>

		<!-- Mock测试使用的json-path依赖 -->
		<dependency>
			<groupId>com.jayway.jsonpath</groupId>
			<artifactId>json-path</artifactId>
			<version>2.2.0</version>
		</dependency>
```

测试类：

* @WebAppConfiguration
用于声明一个ApplicationContext集成测试加载WebApplicationContext。

```
//@WebAppConfiguration：可以在单元测试的时候，不用启动Servlet容器，就可以获取一个Web应用上下文

@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = "classpath:spring/*.xml")
@WebAppConfiguration
public class TestMockMVC {

	@Autowired
	private WebApplicationContext wac;

	private MockMvc mockMvc;

	@Before
	public void setup() {
		// 初始化一个MockMVC对象的方式有两种：单独设置、web应用上下文设置
		// 建议使用Web应用上下文设置
		mockMvc = MockMvcBuilders.webAppContextSetup(wac).build();
	}

	@Test
	public void test() throws Exception {
		// 通过perform去发送一个HTTP请求
		// andExpect：通过该方法，判断请求执行是否成功
		// andDo :对请求之后的结果进行输出
		MvcResult result = mockMvc.perform(MockMvcRequestBuilders.get("/item/showEdit").param("id", "1"))
				.andExpect(MockMvcResultMatchers.view().name("item/item-edit"))
				.andExpect(MockMvcResultMatchers.status().isOk())
				.andDo(MockMvcResultHandlers.print())
				.andReturn();
		
		System.out.println("================================");
		System.out.println(result.getHandler());
	}
	
	@Test
	public void test2() throws Exception {
		// 通过perform去发送一个HTTP请求
		// andExpect：通过该方法，判断请求执行是否成功
		// andDo :对请求之后的结果进行输出
		MvcResult result = mockMvc.perform(get("/item/findItem").param("id", "1").accept(MediaType.APPLICATION_JSON))
				.andExpect(status().isOk())
				.andExpect(content().contentTypeCompatibleWith(MediaType.APPLICATION_JSON))
				.andExpect(jsonPath("$.id").value(1))
				.andExpect(jsonPath("$.name").value("台式机123"))
				.andDo(print())
				.andReturn();
		
		System.out.println("================================");
		System.out.println(result.getHandler());
	}
}
```


# 13.RESTful支持

## 13.1 什么是RESTful？

* 什么是REST？
  - REST（英文：Representational State Transfer，简称 REST，意思是：（资源）表述性状态转化）描述了一个架构样式的网络系统， 比如 web 应用程序。
  - 它是一种软件架构风格、设计风格，而不是标准，只是提供了一组设计原则和约束条件。它主要用于客户端和服务器交互类的软件。基于这个风格设计的软件可以更简洁，更有层次，更易于实现缓存等机制。
  - 它本身并没有什么实用性，其核心价值在于如何设计出符合REST风格的网络接口。
* 什么是RESTful？
  - REST指的是一组架构约束条件和原则。满足这些约束条件和原则的应用程序或设计就是 RESTful。
* RESTful特性
  - 资源（Resources）：网络上的一个实体，或者说是网络上的一个具体信息。它可以是一段文本、一张图片、一首歌曲、一种服务，总之就是一个具体的存在。可以用一个 URI（统一资源定位符）指向它，每种资源对应一个特定的 URI 。要获取这个资源，访问它的 URI 就可以，因此 URI 即为每一个资源的独一无二的识别符。
  - 表现层（Representation）：把资源具体呈现出来的形式，叫做它的表现层 （Representation）。比如，文本可以用 txt 格式表现，也可以用 HTML 格式、XML 格式、JSON 格式表现，甚至可以采用二进制格式。
  - 状态转化（State Transfer）：每发出一个请求，就代表了客户端和服务器的一次交互过程。HTTP协议，是一个无状态协议，即所有的状态都保存在服务器端。因此，如果客户端想要操作服务器， 必须通过某种手段，让服务器端发生“状态转化”（State Transfer）。而这种转化是建立在表现层之上的，所以就是 “ 表现层状态转化” 。具体说， 就是 HTTP 协议里面，四个表示操作方式的动词：GET 、POST 、PUT 、DELETE 。它们分别对应四种基本操作：GET 用来获取资源，POST 用来新建资源，PUT 用来更新资源，DELETE 用来删除资源。
* 如何设计RESTful应用程序的API
  - 路径设计：数据库设计完毕之后，基本上就可以确定有哪些资源要进行操作，相对应的路径也可以设计出来
  - 动词设计：也就是针对资源的具体操作类型，由HTTP动词表示，常用的HTTP动词如下：POST、DELETE、PUT、GET
* RESTful 的示例：
  - /account/1	HTTP GET ：	得到 id = 1 的 account
  - /account/1	HTTP DELETE： 删除 id = 1 的 account
  - /account/1	HTTP PUT：	更新 id = 1 的 account


## 13.2 SpringMVC对RESTful的支持

### 13.2.1 RESTful的URL路径变量

* URL-PATTERN ：设置为/，方便拦截RESTful 请求。
* @PathVariable：可以解析出来URL中的模板变量（{id}）


```
URL:	http://localhost:8080/ssm/item/1/zhangsan

Controller:
	@RequestMapping(“{id}/{name}”)
	@ResponseBody
	public Item queryItemById(@PathVariable Integer id, @PathVariable String name)
```


### 13.2.2 RESTful的CRUD

* @RequestMapping：通过设置method属性的CRUD，可以将同一个URL映射到不同的HandlerMethod方法上
* @GetMapping、@PostMapping、@PutMapping、@DeleteMapping注解同@RequestMapping注解的method属性设置。


### 13.2.3 RESTful的资源表述

* RESTful服务中一个重要的特性就是一种资源可以有多种表现形式，在SpringMVC中可以使用ContentNegotiatingManager这个内容协商管理器来实现这种方式。
* 内容协商的方式有三种：
  - 扩展名,比如.json表示我要JSON格式数据、.xml表示我要XML格式数据
  - 请求参数：默认是”format”
  - 请求头设置Accept参数，比如设置Accept为application/json表示要JSON格式数据
* 不过现在RESTful响应的数据一般都是JSON格式，所以一般也不使用内容协商管理器，直接使用@ResponseBody注解将数据按照JSON格式返回


## 13.3 静态资源访问<mvc:resources>

如果在DispatcherServlet中设置url-pattern为 /则必须对静态资源进行访问处理。

在springmvc.xml文件中，使用mvc:resources标签，具体如下：

```
<!-- 当DispatcherServlet配置为/来拦截请求的时候，需要配置静态资源的访问映射 -->
<mvc:resources location="/js/" mapping="/js/**"/>
<mvc:resources location="/css/" mapping="/css/**"/>
```

Springmvc会把mapping映射到ResourceHttpRequestHandler，这样静态资源在经过DispatcherServlet转发时就可以找到对应的Handler了。


# 14.拦截器

SpringMVC的拦截器主要是针对特定处理器进行拦截的。

## 14.1 SpringMVC拦截器介绍

* SpringMVC拦截器（Interceptor）实现对每一个请求处理前后进行相关的业务处理，类似与servlet中的Filter。
* SpringMVC 中的Interceptor 拦截请求是通过HandlerInterceptor来实现的。
* 在SpringMVC中定义一个Interceptor非常简单，主要有4种方式：
  - 1）实现Spring的HandlerInterceptor接口；
  - 2）继承实现了HandlerInterceptor接口的类，比如Spring 已经提供的实现了HandlerInterceptor 接口的抽象类HandlerInterceptorAdapter；
  - 3）实现Spring的WebRequestInterceptor接口；
  - 4）继承实现了WebRequestInterceptor的类；


## 14.2 定义拦截器

实现HandlerIntercepter接口：

```
public class MyHandlerIntercepter1 implements HandlerInterceptor{

	//Handler执行前调用
	//应用场景：登录认证、身份授权
	//返回值为true则是放行，为false是不放行
	@Override
	public boolean preHandle(HttpServletRequest request,
			HttpServletResponse response, Object handler) throws Exception {
		
		return false;
	}

	//进入Handler开始执行，并且在返回ModelAndView之前调用
	//应用场景：对ModelAndView对象操作，可以把公共模型数据传到前台，可以统一指定视图
	@Override
	public void postHandle(HttpServletRequest request,
			HttpServletResponse response, Object handler,
			ModelAndView modelAndView) throws Exception {
		
	}
	//执行完Handler之后调用
	//应用场景：统一异常处理、统一日志处理
	@Override
	public void afterCompletion(HttpServletRequest request,
			HttpServletResponse response, Object handler, Exception ex)
			throws Exception {
		
	}

}

```

## 14.3 配置拦截器

SpringMVC拦截器是绑定在HandlerMapping中的，即；如果某个HandlerMapping中配置拦截，则该HandlerMapping映射成功的Handler会使用该拦截器。


### 14.3.1 针对单个HandlerMapping配置

只有通过该处理器映射器查找到的处理器，才能使用该拦截器。

如果现在有两个处理器映射器：其中一个设置了处理器拦截器，另外一个没有设置，如果通过第二个映射器查找到的处理器，是无法使用拦截器的。

```
<bean class="org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping">
	<property name="interceptors">
		<list>
			<ref bean="interceptor" />
		</list>
	</property>
</bean>
<bean id="interceptor" class="com.kkb.ssm.interceptor.MyHandlerInterceptor" />

```

### 14.3.2 全局拦截器配置（推荐）

SpringMVC的全局拦截器配置，其实是把配置的拦截器注入到每个已初始化的HandlerMapping中了。

```
<!-- 配置全局mapping的拦截器 -->
<mvc:interceptors>
     <!-- 公共拦截器可以拦截所有请求，而且可以有多个 -->
     <bean class="com.kkb.ssm.interceptor.MyHandlerInterceptor1" />
    <bean class="com.kkb.ssm.interceptor.MyHandlerInterceptor2" />
	<!-- 如果有多个拦截器，则按照顺序进行配置 -->
	<mvc:interceptor>
		<!-- /**表示所有URL和子URL路径 -->
		<mvc:mapping path="/test/**" />
         <!-- 特定请求的拦截器只能有一个 -->
		<bean class="com.kkb.ssm.interceptor.MyHandlerInterceptor3" />
	</mvc:interceptor>
</mvc:interceptors>

```

## 14.4 多拦截器拦截规则

如果有多个拦截器，那么配置到springmvc.xml中最上面的拦截器，拦截优先级最高。
	
## 14.5 拦截器应用（实现登录认证）
### 14.5.1 需求

拦截器对访问的请求URL进行拦截校验
1、如果请求的URL是公开地址（无需登录就可以访问的URL,具体指的就是保护login字段的请求URL），采取放行。
2、如果用户session存在，则放行。
3、如果用户session中不存在，则跳转到登录页面。

### 14.5.2 登录jsp页面

```
<%@ page language="java" contentType="text/html; charset=UTF-8"
    pageEncoding="UTF-8"%>
<!DOCTYPE html PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN" "http://www.w3.org/TR/html4/loose.dtd">
<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=UTF-8">
<title>登录页面</title>
</head>
<body>
	<form action="${pageContext.request.contextPath }/login" method="post">
		<table align="center" border="1" cellspacing="0" >
			<tr>
				<td>用户名：<input type="text" name="username"/></td>
			</tr>
			<tr>
				<td>密    码：<input type="text" name="password"/></td>
			</tr>
			<tr>
				<td><input type="submit" value="登录"/></td>
			</tr>
		</table>
	</form>
</body>
</html>
```

### 14.5.3 Controller类

```
@Controller
public class LoginController {

	//显示登录页面
	@RequestMapping("/loginPage")
	public String loginPage(){
		return "login";
	}
	
	// 登录
	@RequestMapping("/login")
	public String login(HttpSession session, String username, String password) {
		// Service进行用户身份验证

		// 把用户信息保存到session中
		session.setAttribute("username", username);

		// 重定向到商品列表页面
		return "redirect:/item/queryItem";
	}

	// 退出
	@RequestMapping("/logout")
	public String logout(HttpSession session) {
		
		//清空session
		session.invalidate();
		// 重定向到登录页面
		return "redirect:/loginPage";
	}
}
```

### 14.5.4 HandlerInterceptor类

```
public class LoginInterceptor implements HandlerInterceptor {

	@Override
	public boolean preHandle(HttpServletRequest request,
			HttpServletResponse response, Object handler) throws Exception {
		
		//获取请求的URI
		String requestURI = request.getRequestURI();
				
		System.out.println(requestURI);		
//		1、	如果请求的URL是公开地址（无需登录就可以访问的URL），采取放行。
		if(requestURI.indexOf("login")>-1) return true;
//		2、	如果用户session存在，则放行。
		String username = (String) request.getSession().getAttribute("username");
		if(username !=null && !username.equals("")) return true;
//		3、	如果用户session中不存在，则跳转到登录页面。
		response.sendRedirect("/ssm/loginPage");
		return false;
	}

	@Override
	public void postHandle(HttpServletRequest request,
			HttpServletResponse response, Object handler,
			ModelAndView modelAndView) throws Exception {
		// TODO Auto-generated method stub

	}

	@Override
	public void afterCompletion(HttpServletRequest request,
			HttpServletResponse response, Object handler, Exception ex)
			throws Exception {
		// TODO Auto-generated method stub

	}

}
```


### 14.5.5 HandlerInterceptor配置

```
<!-- 配置全局mapping的拦截器 -->
<mvc:interceptors>
	<!-- 如果有多个拦截器，则按照顺序进行配置 -->
	<mvc:interceptor>
		<!-- /**表示所有URL和子URL路径 -->
		<mvc:mapping path="/**" />
		<bean class="com.kkb.ssm.interceptor.LoginInterceptor" />
	</mvc:interceptor>
	<!-- 如果有多个拦截器，则按照顺序进行配置 -->
	<mvc:interceptor>
		<!-- /**表示所有URL和子URL路径 -->
		<mvc:mapping path="/**" />
		<bean class=" com.kkb.ssm.interceptor.MyHandlerInterceptor" />
	</mvc:interceptor>
</mvc:interceptors>
```

# 15.SpringMVC父子容器


<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/picsSpringMVC父子容器.png" width="720px" > </div><br>


问题1：在子容器声明一个Bean，然后父容器中使用@Autowired注解注入，验证注入是否成功。


问题2：@Autowired注解的开启方式有哪些？如果父容器中不开启@Autowired注解，是如何操作？


# 16.跨域处理


## 16.1 什么是跨域？

* 由于浏览器对于Javascript的同源策略的限制，导致A网站不能通过JS（主要就是Ajax请求）去访问B网站的数据，于是跨域问题就出现了。


* 跨域指的是域名、端口、协议的组合不同就是跨域。
	http://www.kkb.com/
	https://www.kkb.com
	http://www.kkb.cn
	http://www.kkb.com:8080/
	
## 16.2 为什么要有同源策略？

* 举例说明：比如一个黑客程序，他利用IFrame把真正的银行登录页面嵌到他的页面上，当你使用真实的用户名，密码登录时，他的页面就可以通过Javascript读取到你的表单中input中的内容，这样用户名，密码就轻松到手了。


<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/黑客盗取1.png" width="520px" > </div><br>

## 16.3 如何解决跨域？

* 解决跨域的方式有多种，比如基于JavaScript的解决方式、基于Jquery的JSONP方式、以及基于CORS的方式。

* JSONP和CORS的区别之一：JSONP只能解决get方式提交、CORS不仅支持GET方式，同时也支持POST提交方式。


## 16.4 什么是CORS？

* CORS是一个W3C标准，全称是"跨域资源共享"（Cross-origin resource sharing）。

* 它允许浏览器向跨源服务器，发出XMLHttpRequest请求，从而克服了AJAX只能同源使用的限制。

* CORS需要浏览器和服务器同时支持。目前，所有浏览器都支持该功能，IE浏览器不能低于IE10。

* CORS原理：只需要向响应头header中注入Access-Control-Allow-Origin，这样浏览器检测到header中的Access-Control-Allow-Origin，则就可以跨域操作了。


## 16.5 CORS请求分类
### 16.5.1 CORS请求分类标准
浏览器将CORS请求分成两类：简单请求（simple request）和非简单请求（not-so-simple request）。
* 只要同时满足以下两大条件，就属于简单请求。

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/简单请求.png" width="520px" > </div><br>

* 凡是不同时满足上面两个条件，就属于非简单请求。

浏览器对这两种请求的处理，是不一样的。


### 16.5.2 简单请求

对于简单请求，浏览器直接发出CORS请求。具体来说，就是在头信息之中，增加一个Origin字段。

* 请求信息：

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/请求信息.png" width="520px" > </div><br>

* 响应信息：

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/响应信息.png" width="520px" > </div><br>

* 字段说明
  - （1）Access-Control-Allow-Origin
该字段是必须的。它的值要么是请求时Origin字段的值，要么是一个*，表示接受任意域名的请求。
  - （2）Access-Control-Allow-Credentials
该字段可选。它的值是一个布尔值，表示是否允许发送Cookie。默认情况下，Cookie不包括在CORS请求之中。设为true，即表示服务器明确许可，Cookie可以包含在请求中，一起发给服务器。这个值也只能设为true，如果服务器不要浏览器发送Cookie，删除该字段即可。

### 16.5.3 非简单请求

* 非简单请求是那种对服务器有特殊要求的请求，比如请求方法是PUT或DELETE，或者Content-Type字段的类型是application/json。

* 非简单请求的CORS请求，会在正式通信之前，增加一次HTTP查询请求，称为"预检"请求（preflight）。

* 浏览器先询问服务器，当前网页所在的域名是否在服务器的许可名单之中，以及可以使用哪些HTTP动词和头信息字段。只有得到肯定答复，浏览器才会发出正式的XMLHttpRequest请求，否则就报错。

* 请求信息：
  - HTTP请求的方法是PUT，并且发送一个自定义头信息X-Custom-Header。
  - 浏览器发现，这是一个非简单请求，就自动发出一个"预检"请求，要求服务器确认可以这样请求。下面是这个"预检"请求的HTTP头信息。
  
<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/非简单请求.png" width="520px" > </div><br>

* 请求信息：
  - "预检"请求用的请求方法是OPTIONS，表示这个请求是用来询问的。头信息里面，关键字段是Origin，表示请求来自哪个源。
  - 除了Origin字段，"预检"请求的头信息包括两个特殊字段。
    * （1）Access-Control-Request-Method
该字段是必须的，用来列出浏览器的CORS请求会用到哪些HTTP方法，上例是PUT。
    * （2）Access-Control-Request-Headers
该字段是一个逗号分隔的字符串，指定浏览器CORS请求会额外发送的头信息字段，上例是X-Custom-Header。
  - 一旦服务器通过了"预检"请求，以后每次浏览器正常的CORS请求，就都跟简单请求一样，会有一个Origin头信息字段。服务器的回应，也都会有一个Access-Control-Allow-Origin头信息字段。


## 16.6 CORS实现

使用springmvc的拦截器实现

### 16.6.1 跨域不提交Cookie

```
public class AllowOriginInterceptor implements HandlerInterceptor {
 
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object arg2) throws Exception {
       // 有跨域行为时参考网址 http://namezhou.iteye.com/blog/2384434
       if (request.getHeader("Origin") != null) {
           response.setContentType("text/html;charset=UTF-8");
           // 允许哪一个URL
          response.setHeader("Access-Control-Allow-Origin", "*");
           // 允许那种请求方法
          response.setHeader("Access-Control-Allow-Methods", "POST, GET, OPTIONS, DELETE");
          response.setHeader("XDomainRequestAllowed", "1");
           System.out.println("正在跨域");
       }
       return true;
    }
 }
```

配置拦截器。

### 16.6.2 跨域提交Cookie

#### 16.6.2.1 注意事项

* Access-Control-Allow-Credentials 为 true的时候，Access-Control-Allow-Origin一定不能设置为”*”，否则报错
* 如果有多个拦截器，一定要把处理跨域请求的拦截器放到首位。

#### 16.6.2.2 JS代码

Jquery Ajax：


```
$.ajax({
url: '自己要请求的url',
method:'请求方式',  //GET POST PUT DELETE
xhrFields:{withCredentials:true},
success:function(data){
   //自定义请求成功做什么
},
error:function(){
//自定义请求失败做什么
}
})

```

angularJS：

* step1.全局 在模块配置中添加


```
app.config(['$httpProvider',function($httpProvider) { 
  $httpProvider.defaults.withCredentials = true; 
} 
]);
```

* step2单个请求


```
$http.get(url, {withCredentials: true});
$http.post(url,data, {withCredentials: true});
$httpProvider.defaults.withCredentials = true;
```

#### 16.6.2.3 Java代码

```
public class AllowOriginInterceptor implements HandlerInterceptor {
 
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object arg2) throws Exception {
       // 有跨域行为时参考网址 http://namezhou.iteye.com/blog/2384434
       if (request.getHeader("Origin") != null) {
           response.setContentType("text/html;charset=UTF-8");
           // 允许哪一个URL 访问 request.getHeader("Origin") 根据请求来的url动态允许
          response.setHeader("Access-Control-Allow-Origin", request.getHeader("Origin"));
           // 允许那种请求方法
          response.setHeader("Access-Control-Allow-Methods", "POST, GET, OPTIONS, DELETE,HEAD");
           response.setHeader("Access-Control-Max-Age", "0");
           // 允许请求头里的参数列表
           response.setHeader("Access-Control-Allow-Headers",
                  "Origin, No-Cache, X-Requested-With, If-Modified-Since, Pragma, Last-Modified, Cache-Control, Expires, Content-Type, X-E4M-With,userId,token");
           // 允许对方带cookie访问
     response.setHeader("Access-Control-Allow-Credentials", "true");
          response.setHeader("XDomainRequestAllowed", "1");
           System.out.println("正在跨域");
       }
       return true;
    }
}
```

















