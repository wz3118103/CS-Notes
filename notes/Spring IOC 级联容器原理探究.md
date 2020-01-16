# 1.前言
Spring 框架核心组件之一是 IOC，IOC 主要负责管理 Bean 的创建和 Bean 之间的依赖注入；在一般的项目实践中我们只需要一个 IOC 容器来管理所有的 Bean 就可以了，但是这不是必然的，在 Spring MVC 框架中就是用了两级 IOC 容器来更好的管理业务 Bean 与Controller Bean；另外使用级联容器我们可以实现子 IOC 容器共享父容器的 Bean，并且可以达到各个子 IOC 容器的 Bean 相互隔离。

本文内容如下：

* Spring MVC 框架级联容器原理探究，父子容器如何创建的？各自作用是什么？
* Demo 演示如何使用级联 IOC，实现不同 IOC 管理的 Bean 相互隔离、子 IOC 共享父 IOC 管理的 Bean。


# 2.Spring MVC 框架中级联容器原理探究

SpringMVC 是目前使用比较多的框架，springmvc 里面经常会使用两级级联容器，并且每层容器都各有用途，本节就来探究下这两层级联容器如何创建，以及都有何作用。

## 2.1 如何配置两级容器

使用过 SpringMVC 的童鞋都知道，一般我们在 web.xml 里面会配置一个 ContextLoaderListener 和一个 DispatcherServlet，那么这两个东西到底是什么作用那？其实这就是配置了两个 spring IOC 容器，并且 DispatcherServlet 创建的 IOC 容器的父容器就是 ContextLoaderListene r创建的 IOC 容器。

一般我们会在 web.xml 里面进行配置如下：

配置 ContextLoaderListener

```
<listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>

<context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>WEB-INF/applicationContext.xml</param-value>
</context-param>
```

配置 DispatcherServlet

```
<servlet>
    <servlet-name>springmvc</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <load-on-startup>1</load-on-startup>
</servlet>
```

其中 ContextLoaderListener 会创建一个由应用程序上下文 XMLWebApplicationContext 来管理的 IOC 容器，IOC 里面的 bean 是通过 <context-param> 配置的 contextConfigLocation 参数对应的 WEB-INF/applicationContext.xml 来注入的。

DispatcherServlet 也会创建一个使用 XMLWebApplicationContext 管理的 IOC 容器，默认管理 WEB-INF/springmvc-servlet.xml 里面的 Controller Bean，当然用户也可以通过下面方式进行自定义配置文件路径：

```
<servlet>
  <servlet-name>springmvc</servlet-name>
  <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
  <load-on-startup>1</load-on-startup>
  <init-param>  
      <param-name>contextConfigLocation</param-name>  
      <param-value>classpath:springmvc-servlet.xml</param-value>  
  </init-param>
</servlet>
```

完成本节配置后就会形成两级级联的 IOC 容器了：

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/两级级联IOC容器.png" width="520px" > </div><br>


另外每个 IOC 容器都会被一个应用程序上下文来管理的。

## 2.2 ContextLoaderListener 创建 Root IOC 容器

本节我们来看看 ContextLoaderListener 是如何创建 Root IOC 容器的，首先看一下 ContextLoaderListener 的启动时序图：

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/ContextLoaderListener 的启动时序图.png" width="820px" > </div><br>


由于 ContextLoaderListener 实现了 ServletContextListener 接口，所以其有 contextInitialized 方法，该该法在 web 容器 tomcat 启动时候会被调用。

contextInitialized 内部调用的是 initWebApplicationContext 方法，下面我们看后面方法内容：

```
public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {
        ...
        try {

            //2.2.1
            if (this.context == null) {
                this.context = createWebApplicationContext(servletContext);
            }
            //2.2.2配置和刷新容器
            if (this.context instanceof ConfigurableWebApplicationContext) {
                ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) this.context;
                if (!cwac.isActive()) {
                    ...
                    configureAndRefreshWebApplicationContext(cwac, servletContext);
                }
            }
            //2.2.3记录root应用程序上下文
            servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);

            ...
            return this.context;
        }
        ...
}
```

其中代码 2.2.1 是具体创建 Spring Root IOC 容器对应的应用程序上下文的，其代码：

```
    protected WebApplicationContext createWebApplicationContext(ServletContext sc) {
        Class<?> contextClass = determineContextClass(sc);
        ...
        return (ConfigurableWebApplicationContext) BeanUtils.instantiateClass(contextClass);
    }
```

可知是实例化 contextClass 的一个实例，下面我们看 determineContextClass 里面获取的 Class 对象是什么：

```
protected Class<?> determineContextClass(ServletContext servletContext) {
        //我们没设置contextClass属性
        String contextClassName = servletContext.getInitParameter(CONTEXT_CLASS_PARAM);
        if (contextClassName != null) {
            try {
                return ClassUtils.forName(contextClassName, ClassUtils.getDefaultClassLoader());
            }
            catch (ClassNotFoundException ex) {
                throw new ApplicationContextException(
                        "Failed to load custom context class [" + contextClassName + "]", ex);
            }
        }
        else {
        //代码会走到这里
            contextClassName = defaultStrategies.getProperty(WebApplicationContext.class.getName());
            try {
                return ClassUtils.forName(contextClassName, ContextLoader.class.getClassLoader());
            }
            catch (ClassNotFoundException ex) {
                throw new ApplicationContextException(
                        "Failed to load default context class [" + contextClassName + "]", ex);
            }
        }
    }
```

可知在没有设置 contextClass 属性的时候会从配置文件 defaultStrategies 获取，而 defaultStrategies 的定义是：

```
    private static final String DEFAULT_STRATEGIES_PATH = "ContextLoader.properties";
    static {

        try {
            ClassPathResource resource = new ClassPathResource(DEFAULT_STRATEGIES_PATH, ContextLoader.class);
            defaultStrategies = PropertiesLoaderUtils.loadProperties(resource);
        }
        catch (IOException ex) {
            throw new IllegalStateException("Could not load 'ContextLoader.properties': " + ex.getMessage());
        }
    }
```

那么最后我们看看 ContextLoader.properties 的内容为：

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/ContextLoader.properties.png" width="820px" > </div><br>


所以代码 2.2.1 是创建了一个 XmlWebApplicationContext 应用程序上下文的实例。

代码 2.2.2 则是配置和刷新 XmlWebApplicationContext 实例管理的 IOC 容器

```
protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac, ServletContext sc) {
        ...
        wac.setServletContext(sc);

        //2.2.2.1
        String configLocationParam = sc.getInitParameter(CONFIG_LOCATION_PARAM);
        if (configLocationParam != null) {
            wac.setConfigLocation(configLocationParam);
        }

        ...
        //2.2.2.2
        customizeContext(sc, wac);
        wac.refresh();
    }
```

其中代码 2.2.2.1 则是从 ServletContext 里面获取我们在 web.xml 里面配置的 contextConfigLocation 变量的值，这里为 WEB-INF/ applicationContext.xml。

其中代码 2.2.2.2 则调用 XmlWebApplicationContext 的 refresh 方法刷新 IOC 容器，也就是解析 applicationContext.xml 里面的 bean 并放入到 IOC 容器 。

代码 2.2.3 则记录 Root 应用程序上下文到全局的 servlet Context，这个后面会使用

注：综上可知 ContextLoaderListener 的作用是创建一个XmlWebApplicationContext 的实例，然后解析applicationContext.xml 里面的 bean 放入到 XmlWebApplicationContext 实例管理的 IOC 容器里面。

## 2.3 DispatcherServlet 创建 Sub IOC 容器

本节我们来看看 DispatcherServlet 是如何使用 ContextLoader Listener 创建的 IOC 容器作为父容器来创建自己的 sub ioc 容器的，首先看下时序图：

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/DispatcherServlet创建子容器时序图.png" width="820px" > </div><br>


DispatcherServlet 类作用主要是分发请求到具体的 controller，其继承自 FrameworkServlet，后者有继承自 HttpServletBean，HttpServletBean 又继承自 GenericServlet，所以 DispatcherServlet 本身是一个 servlet，所以 tomcat 启动时候会调用其其 init 方法：

```
    public void init(ServletConfig config) throws ServletException {
    this.config = config;
    this.init();
    }

public final void init() throws ServletException {
        ...
        //(2.3.1)
        initServletBean();

}
```

initServletBean 的代码如下：

```
protected final void initServletBean() throws ServletException {
        ...
        try {//2.3.2
            this.webApplicationContext = initWebApplicationContext();
            //2.3.3
            initFrameworkServlet();
        }
        ...
}
```

我们重点看 initWebApplicationContext 代码：

```
protected WebApplicationContext initWebApplicationContext() {
        //2.3.2.1
        WebApplicationContext rootContext =
                WebApplicationContextUtils.getWebApplicationContext(getServletContext());
        WebApplicationContext wac = null;

        ...
        //2.3.2.2
        if (wac == null) {
            // No context instance is defined for this servlet -> create a local one
            wac = createWebApplicationContext(rootContext);
        }

        ...

        return wac;
    }
```

其中 2.3.2.1 作用是获取我们在 2.2 节讲解的记录到全局的 servlet Context 里面的 Root 应用程序上下文：

```
public static WebApplicationContext getWebApplicationContext(ServletContext sc) {
        return getWebApplicationContext(sc, WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE);
    }
其中 2.3.2.2 作用是创建 sub 应用程序上下文：
protected WebApplicationContext createWebApplicationContext(@Nullable ApplicationContext parent) {
        //2.3.2.2-1这里还是XmlWebApplicationContext
        Class<?> contextClass = getContextClass();
        ...
        ConfigurableWebApplicationContext wac =
                (ConfigurableWebApplicationContext) BeanUtils.instantiateClass(contextClass);

        wac.setEnvironment(getEnvironment());
        //2.3.2.2-2设置父容器
        wac.setParent(parent);
        //2.3.2.2-3获取xml配置文件
        String configLocation = getContextConfigLocation();
        if (configLocation != null) {
            wac.setConfigLocation(configLocation);
        }
        //2.3.2.2-4刷新应用程序上下文
        configureAndRefreshWebApplicationContext(wac);

        return wac;
    }
```

代码 2.3.2.2 - 1 创建 sub context XmlWebApplicationContext 的实例。
代码 2.3.2.2 - 2 则设置 sub context 的父亲为 root context。
代码 2.3.2.2 - 3 则获取 sub context 对应的 controller bean 所在的配置文件路径，如果在配置 DispatcherServlet 的时候配置了 init - param 参数 contextConfigLocation，则这里会使用用户配置的 xml 文件。

代码 2.3.2.2 - 4 则配置和刷新 XmlWebApplicationContext 实例的容器：

```
protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac) {
        ...

        wac.setServletContext(getServletContext());
        wac.setServletConfig(getServletConfig());
        //设置namespace
        wac.setNamespace(getNamespace());
        wac.addApplicationListener(new SourceFilteringListener(wac, new ContextRefreshListener()));
        ...
        //刷新容器
        wac.refresh();
    }
```

其中 getNamespace 获取的值为 springmvc-servlet，然后设置到 wac 中，这个是在当用户没有通过 contextConfigLocation 配置 xml 文件时候，会通过namespace拼接后默认查找 WEB-INF/springmvc-servlet.xml 文件，这个可以从 XmlWebApplicationContext 的 getDefaultConfigLocations 方法知道：

```
    protected String[] getDefaultConfigLocations() {
        if (getNamespace() != null) {
            return new String[] {DEFAULT_CONFIG_LOCATION_PREFIX + getNamespace() + DEFAULT_CONFIG_LOCATION_SUFFIX};
        }
        else {
            return new String[] {DEFAULT_CONFIG_LOCATION};
        }
    }

    public static final String DEFAULT_CONFIG_LOCATION = "/WEB-INF/applicationContext.xml";

    public static final String DEFAULT_CONFIG_LOCATION_PREFIX = "/WEB-INF/";

    public static final String DEFAULT_CONFIG_LOCATION_SUFFIX = ".xml";
```

设置完 namespace 后最后一步是调用 wac.refresh( )；// 刷新容器

注：DispatcherServlet 也会创建一个 XmlWebApplicationContext 应用程序上下文并管理一个 IOC 容器，并且该应用程序上下文使用 root context 作为父容器。

其配置文件默认是在 WEB-INF/springmvc-servlet.xml 位置，用户还可以通过配置 DispatcherServlet 的时候配置 contextConfigLocation 参数来自定义配置文件路径。

## 2.4 总结
综合知道一般我们在 ContextLoaderListener 创建的 root context 管理的 IOC 容器里面配置 bo 类用来具体操作业务，在 DispatcherServlet 创建的 sub context 管理的子容器里面配的 Controller 类，然后 Controller 里面具体调用 bo 类来实现业务。


# 3.利用父子容器实现 Bean 的 IOC 级别隔离

## 3.1 demo 结构

[demo下载地址](https://github.com/zhailuxu/SpringIOC)

本节使用 SpringBoot 搭建一个模拟父子容器 bean 的 demo，demo 目录结构如下：

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/demo目录结构.png" width="620px" > </div><br>


其中标注 2 的为三个普通的 POJO 的 Bean，标注 3 的为三个 xml 配置文件，context.xml 分别注入三个 POJO 的 bean: root - context.xml 内容：

```
    <bean class="com.learn.java.root.bean.RootBean">
        <property name="name" value="rootBean"></property>
    </bean>
</beans>
```

sub-context-one.xml 内容：

```
<bean class="com.learn.java.subone.bean.SubOneBean">
        <property name="name" value="subOneBean"></property>
    </bean>
```

sub-context-two.xml 内容：

```
<bean class="com.learn.java.subtwo.bean.SubTwoBean">
        <property name="name" value="subTwoBean"></property>
</bean>
```

其中标注 1 的为一个配置 bean，用来创建两个 sub context：

```
@Configuration
public class BeanConfig implements ApplicationContextAware, DisposableBean {

    public ClassPathXmlApplicationContext getSubContextOne() {
        return subContextOne;
    }

    public void setSubContextOne(ClassPathXmlApplicationContext subContextOne) {
        this.subContextOne = subContextOne;
    }

    public ClassPathXmlApplicationContext getSubContextTwo() {
        return subContextTwo;
    }

    public void setSubContextTwo(ClassPathXmlApplicationContext subContextTwo) {
        this.subContextTwo = subContextTwo;
    }

    private ClassPathXmlApplicationContext subContextOne = null;
    private ClassPathXmlApplicationContext subContextTwo = null;

    @Override
    public void destroy() throws Exception {
        if (null != subContextOne) {
            subContextOne.destroy();
        }
        if (null != subContextTwo) {
            subContextTwo.destroy();
        }

    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        subContextOne = new ClassPathXmlApplicationContext(new String[] { "sub-context-one.xml" }, applicationContext);

        subContextTwo = new ClassPathXmlApplicationContext(new String[] { "sub-context-two.xml" }, applicationContext);

    }

}
```

标注4的为启动类，也是测试类：

```
@RestController
@SpringBootApplication
@ComponentScan(basePackages = { "com.learn.java.config" })
@ImportResource("root-context.xml")
public class App {

    private static ConfigurableApplicationContext rootConext;
    @Autowired
    private BeanConfig beanConfig;

    @RequestMapping("/home")
    String home() {
        return "Hello World!";
    }

    /**
     * 子容器访问根容器里面的bean
     * 
     * @return
     */
    @RequestMapping("/testSubAccessRootBean")
    String testSubAccessRootBean() {
        ClassPathXmlApplicationContext subConext = beanConfig.getSubContextOne();
        RootBean rootBean = subConext.getBean(RootBean.class);
        if (null != rootBean) {
            return rootBean.getName();
        } else {
            return "can not found bean";

        }

    }

    /**
     * 根容器访问子容器里面Bean
     * 
     * @return
     */
    @RequestMapping("/testRootAccessSubBean")
    String testRootAccessSubBean() {
        try {
            SubOneBean sub = rootConext.getBean(SubOneBean.class);
            if (null != sub) {
                return sub.getName();
            } else {
                return "can not found bean";

            }
        }catch(Exception e) {

            return e.getLocalizedMessage();
        }


    }

    /**
     * 子容器访问兄弟子容器Bean
     * @return
     */
    @RequestMapping("/testSubAccessSubBean")
    String testSubAccessSubBean() {
        try {
            ClassPathXmlApplicationContext subConext = beanConfig.getSubContextOne();
            SubTwoBean subTwoBean = subConext.getBean(SubTwoBean.class);
            if (null != subTwoBean) {
                return subTwoBean.getName();
            } else {
                return "can not found bean";

            }
        }catch(Exception e) {
            return e.getLocalizedMessage();
        }

    }

    public static void main(String[] args) {
        rootConext = SpringApplication.run(App.class, args);
    }
}

```

## 4.2 demo 概要讲解

在 App.java 也就是 Boot 启动类里面我们使用 rootConext 保存了 Root Context，在调用 SpringApplication.run 启动 Boot 应用后会创建一个 Root Context，demo 中的 RootBean、BeanConfig 就是被注入到 Root Context 里面的。

BeanConfig 里面启动了两个 sub context ClassPathXmlApplication Context 实例，分别加载 sub-context-one.xml和sub-context-two.xml 里面配置的 bean，并且创建 ClassPathXmlApplicationContext 的时候使用 Root Context 作为了父上下文。

最后容器结构就构成了如下图：

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/容器结构图.png" width="620px" > </div><br>


其中两个 sub context 里面的 bean 不能相互引用，也就是这两个 sub 容器之间是隔离的。但是两个 sub 容器都可以访问 root context 管理的容器。

# 参考
 - [spring 文档](https://docs.spring.io/spring/docs/5.2.3.RELEASE/spring-framework-reference/core.html#spring-core)
 - [SpringMVC 文档](https://docs.spring.io/spring/docs/5.2.3.RELEASE/spring-framework-reference/web.html#spring-web)