
# 1.IOC

## 1.1 什么是IoC容器？

所谓的IoC容器就是指的Spring中Bean工厂里面的Map存储结构（存储了Bean的实例）。

Spring框架中的工厂有哪些？
  * ApplicationContext接口
    - 实现了BeanFactory接口
    - 实现ApplicationContext接口的工厂，可以获取到容器中具体的Bean对象
  * BeanFactory工厂（是Spring框架早期的创建Bean对象的工厂接口）
    - 实现BeanFactory接口的工厂也可以获取到Bean对象


<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/ApplicationContext.png" width="920px" > </div><br>


其实通过源码分析，不管是BeanFactory还是ApplicationContext，其实最终的底层BeanFactory都是DefaultListableBeanFactory。


ApplicationContext和BeanFactory的区别？
  * 创建Bean对象的时机不同:
    - BeanFactory采取延迟加载，第一次getBean时才会初始化Bean。
    - ApplicationContext是加载完applicationContext.xml时，就创建具体的Bean对象的实例。（只对BeanDefition中描述为是单例的bean，才进行饿汉式加载）


<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/BeanFactory类层次.png" width="620px" > </div><br>

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/AppliCationContext类继承图.png" width="620px" > </div><br>

### 1.1.1 ClassPathXmlApplicationContext

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/ClassPathXmlApplicationContext.png" width="920px" > </div><br>

```
    @Test
    public void testLoadXml() {
        ApplicationContext ac = new ClassPathXmlApplicationContext("classpath:software/spring/xml/beans.xml");
        Person person = (Person) ac.getBean("person");
        System.out.println(person);
    }
```

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/ClassPathXmlApplicationContext实例.png" width="820px" > </div><br>

### 1.1.2 特别注意DefaultListableBeanFactory

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/beanFactory.jpg" width="620px" > </div><br>


<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/DefaultListableBeanFactory.png" width="1020px" > </div><br>


### 1.1.3 AnnotationConfigApplicationContext


<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/AnnotationConfigApplicationContext.png" width="1020px" > </div><br>

```
    @Test
    public void testLoadConfig() {
        ApplicationContext ac = new AnnotationConfigApplicationContext(MainConfig.class);
        Person person = (Person) ac.getBean("aPerson");
        System.out.println(person);

        String[] names = ac.getBeanNamesForType(Person.class);
        for (String name : names) {
            System.out.println(name);
        }
    }
```


<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/AnnotationConfigApplicationContext实例.png" width="820px" > </div><br>


<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/beanFatory实例.jpg" width="620px" > </div><br>


## 1.2 如何创建IOC容器

什么是web服务器（Servlet容器）？ 
* Tomcat、Jetty、Jboss等

什么是web容器？    
* ServletContext（Servlet上下文）、Servlet三大域对象（生命周期范围最大的一个）

什么是spring容器？
* ApplicationContext（Spring上下文，实现了BeanFactory）


### 1.2.1 创建方式

* ApplicationContext接口常用实现类
  - ClassPathXmlApplicationContext：
	它是从类的根路径下加载配置文件	推荐使用这种
  - FileSystemXmlApplicationContext：
	它是从磁盘路径上加载配置文件，配置文件可以在磁盘的任意位置。
  - AnnotationConfigApplicationContext:
	当我们使用注解配置容器对象时，需要使用此类来创建 spring 容器。它用来读取注解。
* Java应用中创建IoC容器
ApplicationContext context = new ClassPathXmlApplicationContext(xml路径);
* Web应用中创建IoC容器
web.xml中配置ContextLoaderListener接口，并配置ContextConfigLocation参数
  - web容器启动之后加载web.xml，此时加载ContextLoaderListener监听器（实现了ServletContextListener接口，该接口的描述请见下面《三类八种监听器》）
  - ContextLoaderListener监听器会在web容器启动的时候，触发contextInitialized()方法。
  - contextInitialized()方法会调用initWebApplicationContext()方法，该方法负责创建Spring容器（DefaultListableBeanFactory）。


### 1.2.2 Web三类八种监听器
* 监听域对象的生命周期：
  - ServletContextListener：
    * 创建：服务器启动
    * 销毁：服务器正常关闭
    * spring ContextLoaderListener(服务器启动时负责加载Spring配置文件)
  - HttpSessionListener
    * 创建：第一次访问request.getHttpSession();
    * 销毁：调用invalidate();非法关闭；过期
  - ServletRequestListener
    * 创建：每一次访问
    * 销毁：响应结束
* 监听域对象的属性：（添加、删除、替换）
  - ServletContextAttributeListener
  - HttpSessionAttributeListener
  - ServletRequestAttributeListener
* 监听HttpSession中JavaBean的改变：
  - HttpSessionBindingListener（HttpSession和JavaBean对象的绑定和解绑）
  - HttpSessionActivationListener（HttpSession的序列化，活化、钝化）


### 1.2.3 web容器初始化过程

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/web容器初始化.png" width="520px" > </div><br>


* step1.web服务器（tomcat）启动会加载web.xml（启动ContextLoaderListener监听器）

```
    <context-param>
        <param-name>contextClass</param-name>
        <param-value>
            org.springframework.web.context.support.AnnotationConfigWebApplicationContext
        </param-value>
    </context-param>
    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>com.imooc.o2o.config.WebConfig</param-value>
    </context-param>
    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>
```

* step2.web服务器启动后，会创建ServletContext（web上下文，也就是web容器），此时会触发ContextLoaderListener监听器的contextInitialized()方法。

```
	@Override
	public void contextInitialized(ServletContextEvent event) {
		initWebApplicationContext(event.getServletContext());
	}
```

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/ContextLoaderListener.png" width="820px" > </div><br>


<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/Tomcat启动时调用栈.jpg" width="620px" > </div><br>


* step3.contextInitialized()方法中会调用initWebApplicationContext()方法，该方法负责创建Spring容器和生产Bean对象。

ContextLoader.initWebApplicationContext()：

```
	public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {
		if (servletContext.getAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE) != null) {
			throw new IllegalStateException(
					"Cannot initialize context because there is already a root application context present - " +
					"check whether you have multiple ContextLoader* definitions in your web.xml!");
		}

		servletContext.log("Initializing Spring root WebApplicationContext");
		Log logger = LogFactory.getLog(ContextLoader.class);
		if (logger.isInfoEnabled()) {
			logger.info("Root WebApplicationContext: initialization started");
		}
		long startTime = System.currentTimeMillis();

		try {
			// Store context in local instance variable, to guarantee that
			// it is available on ServletContext shutdown.
			if (this.context == null) {
				this.context = createWebApplicationContext(servletContext);
			}
			if (this.context instanceof ConfigurableWebApplicationContext) {
				ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) this.context;
				if (!cwac.isActive()) {
					// The context has not yet been refreshed -> provide services such as
					// setting the parent context, setting the application context id, etc
					if (cwac.getParent() == null) {
						// The context instance was injected without an explicit parent ->
						// determine parent for root web application context, if any.
						ApplicationContext parent = loadParentContext(servletContext);
						cwac.setParent(parent);
					}
					configureAndRefreshWebApplicationContext(cwac, servletContext);
				}
			}
			servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);

			ClassLoader ccl = Thread.currentThread().getContextClassLoader();
			if (ccl == ContextLoader.class.getClassLoader()) {
				currentContext = this.context;
			}
			else if (ccl != null) {
				currentContextPerThread.put(ccl, this.context);
			}

			if (logger.isInfoEnabled()) {
				long elapsedTime = System.currentTimeMillis() - startTime;
				logger.info("Root WebApplicationContext initialized in " + elapsedTime + " ms");
			}

			return this.context;
		}
		catch (RuntimeException | Error ex) {
			logger.error("Context initialization failed", ex);
			servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, ex);
			throw ex;
		}
	}
```

* step4.initWebApplicationContext()方法负责创建WebApplicationContext，通过createWebApplicationContext()方法
  - WebApplicationContext是一个接口，此处创建的是它的默认实现类：XmlWebApplicationContext（web环境中的真正容器）


<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/创建WebApplicationContext.png" width="520px" > </div><br>

```
	protected WebApplicationContext createWebApplicationContext(ServletContext sc) {
		Class<?> contextClass = determineContextClass(sc);
		if (!ConfigurableWebApplicationContext.class.isAssignableFrom(contextClass)) {
			throw new ApplicationContextException("Custom context class [" + contextClass.getName() +
					"] is not of type [" + ConfigurableWebApplicationContext.class.getName() + "]");
		}
		return (ConfigurableWebApplicationContext) BeanUtils.instantiateClass(contextClass);
	}
```

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/init.jpg" width="520px" > </div><br>


* step5.加载spring配置文件，并创建beans。通过configureAndRefreshWebApplicationContext()方法

```
	protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac, ServletContext sc) {
		if (ObjectUtils.identityToString(wac).equals(wac.getId())) {
			// The application context id is still set to its original default value
			// -> assign a more useful id based on available information
			String idParam = sc.getInitParameter(CONTEXT_ID_PARAM);
			if (idParam != null) {
				wac.setId(idParam);
			}
			else {
				// Generate default id...
				wac.setId(ConfigurableWebApplicationContext.APPLICATION_CONTEXT_ID_PREFIX +
						ObjectUtils.getDisplayString(sc.getContextPath()));
			}
		}

		wac.setServletContext(sc);
		String configLocationParam = sc.getInitParameter(CONFIG_LOCATION_PARAM);
		if (configLocationParam != null) {
			wac.setConfigLocation(configLocationParam);
		}

		// The wac environment's #initPropertySources will be called in any case when the context
		// is refreshed; do it eagerly here to ensure servlet property sources are in place for
		// use in any post-processing or initialization that occurs below prior to #refresh
		ConfigurableEnvironment env = wac.getEnvironment();
		if (env instanceof ConfigurableWebEnvironment) {
			((ConfigurableWebEnvironment) env).initPropertySources(sc, null);
		}

		customizeContext(sc, wac);
		wac.refresh();
	}
```

从代码可以看出，先设置了ConfigurableWebApplicationContext的配置路径configurationLocation，然后refresh()。


* step6.将spring容器context挂载到ServletContext 这个web容器上下文中。通过servletContext.setAttribute()方法


### 1.2.4 spring容器初始化过程

#### 1.2.4.1 主流程分析

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/spring容器初始化过程.png" width="720px" > </div><br>

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/spring容器处理bean过程.png" width="720px" > </div><br>

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/bean各个类处理.png" width="720px" > </div><br>


针对xml配置文件：

```
    @Test
    public void testLoadXml() {
        ApplicationContext ac = new ClassPathXmlApplicationContext("classpath:software/spring/xml/beans.xml");
        Person person = (Person) ac.getBean("person");
        System.out.println(person);
    }
```

```
	public ClassPathXmlApplicationContext(String configLocation) throws BeansException {
		this(new String[] {configLocation}, true, null);
	}
```

```
	public ClassPathXmlApplicationContext(
			String[] configLocations, boolean refresh, @Nullable ApplicationContext parent)
			throws BeansException {

		super(parent);
		setConfigLocations(configLocations);
		if (refresh) {
			refresh();
		}
	}
```


针对注解类型：

```
    @Test
    public void testLoadConfig() {
        ApplicationContext ac = new AnnotationConfigApplicationContext(MainConfig.class);
        Person person = (Person) ac.getBean("aPerson");
        System.out.println(person);

        String[] names = ac.getBeanNamesForType(Person.class);
        for (String name : names) {
            System.out.println(name);
        }
    }

```

```
	public AnnotationConfigApplicationContext(Class<?>... componentClasses) {
		this();
		register(componentClasses);
		refresh();
	}
```

重点关注AbstractApplicationContext.refresh()方法：

```
	public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			//1、 Prepare this context for refreshing.
			prepareRefresh();

               //创建DefaultListableBeanFactory（真正生产和管理bean的容器）
               //加载BeanDefition并注册到BeanDefitionRegistry
               //通过NamespaceHandler解析自定义标签的功能（比如:context标签、aop标签、tx标签）
			//2、 Tell the subclass to refresh the internal bean factory.
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			//3、 Prepare the bean factory for use in this context.
			prepareBeanFactory(beanFactory);

			try {
				//4、 Allows post-processing of the bean factory in context subclasses.
				postProcessBeanFactory(beanFactory);

                     //实例化并调用实现了BeanFactoryPostProcessor接口的Bean
                     //比如：PropertyPlaceHolderConfigurer（context:property-placeholer）
                     //就是此处被调用的，作用是替换掉BeanDefinition中的占位符（${}）中的内容
				//5、 Invoke factory processors registered as beans in the context.
				invokeBeanFactoryPostProcessors(beanFactory);

                     //创建并注册BeanPostProcessor到BeanFactory中（Bean的后置处理器）
                     //比如：AutowiredAnnotationBeanPostProcessor（实现@Autowired注解功能）
                     //      RequiredAnnotationBeanPostProcessor（实现@d注解功能）
                     //这些注册的BeanPostProcessor
				//6、 Register bean processors that intercept bean creation.
				registerBeanPostProcessors(beanFactory);

				//7、 Initialize message source for this context.
				initMessageSource();

				//8、 Initialize event multicaster for this context.
				initApplicationEventMulticaster();

				//9、 Initialize other special beans in specific context subclasses.
				onRefresh();

				//10、 Check for listener beans and register them.
				registerListeners();

                     //创建非懒加载方式的单例Bean实例（未设置属性）
                     //填充属性
                     //初始化实例（比如调用init-method方法）
                     //调用BeanPostProcessor（后置处理器）对实例bean进行后置处理
				//11、 Instantiate all remaining (non-lazy-init) singletons.
				finishBeanFactoryInitialization(beanFactory);

				//12、 Last step: publish corresponding event.
				finishRefresh();
			}

			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}

				// Destroy already created singletons to avoid dangling resources.
				destroyBeans();

				// Reset 'active' flag.
				cancelRefresh(ex);

				// Propagate exception to caller.
				throw ex;
			}

			finally {
				// Reset common introspection caches in Spring's core, since we
				// might not ever need metadata for singleton beans anymore...
				resetCommonCaches();
			}
		}
	}

```


* step1.ResourceLoader从存储介质中加载Spring配置信息，并使用Resource表示这个配置文件的资源；
* step2.BeanDefinitionReader读取Resource所指向的配置文件资源，然后解析配置文件。配置文件中每一个<bean>解析成一个BeanDefinition对象，并保存到BeanDefinitionRegistry中；
* step3.容器扫描BeanDefinitionRegistry中的BeanDefinition，使用Java的反射机制自动识别出Bean工厂后处理后器（实现BeanFactoryPostProcessor接口）的Bean，然后调用这些Bean工厂后处理器对BeanDefinitionRegistry中的BeanDefinition进行加工处理。主要完成以下两项工作：
  - 1）对使用到占位符的<bean>元素标签进行解析，得到最终的配置值，这意味对一些半成品式的BeanDefinition对象进行加工处理并得到成品的BeanDefinition对象；
  - 2）对BeanDefinitionRegistry中的BeanDefinition进行扫描，通过Java反射机制找出所有属性编辑器的Bean（实现java.beans.PropertyEditor接口的Bean），并自动将它们注册到Spring容器的属性编辑器注册表中（PropertyEditorRegistry）；
* step4.Spring容器从BeanDefinitionRegistry中取出加工后的BeanDefinition，并调用InstantiationStrategy着手进行Bean实例化的工作；
* step5.在实例化Bean时，Spring容器使用BeanWrapper对Bean进行封装，BeanWrapper提供了很多以Java反射机制操作Bean的方法，它将结合该Bean的BeanDefinition以及容器中属性编辑器，完成Bean属性的设置工作；
* step6.利用容器中注册的Bean后处理器（实现BeanPostProcessor接口的Bean）对已经完成属性设置工作的Bean进行后续加工，直接装配出一个准备就绪的Bean。


Spring容器确实堪称一部设计精密的机器，其内部拥有众多的组件和装置。Spring的高明之处在于，它使用众多接口描绘出了所有装置的蓝图，构建好Spring的骨架，继而通过继承体系层层推演，不断丰富，最终让Spring成为有血有肉的完整的框架。所以查看Spring框架的源码时，有两条清晰可见的脉络：
* 1）接口层描述了容器的重要组件及组件间的协作关系；
* 2）继承体系逐步实现组件的各项功能。


接口层清晰地勾勒出Spring框架的高层功能，框架脉络呼之欲出。有了接口层抽象的描述后，不但Spring自己可以提供具体的实现，任何第三方组织也可以提供不同实现， 可以说Spring完善的接口层使框架的扩展性得到了很好的保证。纵向继承体系的逐步扩展，分步骤地实现框架的功能，这种实现方案保证了框架功能不会堆积在某些类的身上，造成过重的代码逻辑负载，框架的复杂度被完美地分解开了。

Spring组件按其所承担的角色可以划分为两类：
* 1）物料组件：Resource、BeanDefinition、PropertyEditor以及最终的Bean等，它们是加工流程中被加工、被消费的组件，就像流水线上被加工的物料；
* 2）加工设备组件：ResourceLoader、BeanDefinitionReader、BeanFactoryPostProcessor、InstantiationStrategy以及BeanWrapper等组件像是流水线上不同环节的加工设备，对物料组件进行加工处理。


#### 1.2.4.2 创建BeanFactory流程分析

```
   //创建DefaultListableBeanFactory（真正生产和管理bean的容器）
   //加载BeanDefition并注册到BeanDefitionRegistry
   //通过NamespaceHandler解析自定义标签的功能（比如:context标签、aop标签、tx标签）
  //2、 Tell the subclass to refresh the internal bean factory.
  ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
```

调用AbstractApplicationContext.obtainFreshBeanFactory()：

```
	protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
		refreshBeanFactory();
		return getBeanFactory();
	}
```

调用AbstractRefreshableApplicationContext的refreshBeanFactory方法：

```
	@Override
	protected final void refreshBeanFactory() throws BeansException {
		if (hasBeanFactory()) {
			destroyBeans();
			closeBeanFactory();
		}
		try {
			DefaultListableBeanFactory beanFactory = createBeanFactory();
			beanFactory.setSerializationId(getId());
			customizeBeanFactory(beanFactory);
			loadBeanDefinitions(beanFactory);
			synchronized (this.beanFactoryMonitor) {
				this.beanFactory = beanFactory;
			}
		}
		catch (IOException ex) {
			throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
		}
	}


	protected DefaultListableBeanFactory createBeanFactory() {
		return new DefaultListableBeanFactory(getInternalParentBeanFactory());
	}
```

这里不仅创建了DefaultListableBeanFactory，并且读取了beans.xml配置文件，将bean的定义信息解析出来放到了DefaultListableBeanFactory中。

#### 1.2.4.3 加载解析BeanDefinition子流程（loadDefinitions方法）

AbstractRefreshableApplicationContext的refreshBeanFactory方法中的loadBeanDefinitions(beanFactory)方法。

##### 1.2.4.3.1 xml配置 

针对xml配置文件，调用AbstractXmlApplicationContext.loadBeanDefinitions(beanFactory)：

```
	@Override
	protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
		// Create a new XmlBeanDefinitionReader for the given BeanFactory.
		XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

		// Configure the bean definition reader with this context's
		// resource loading environment.
		beanDefinitionReader.setEnvironment(this.getEnvironment());
		beanDefinitionReader.setResourceLoader(this);
		beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

		// Allow a subclass to provide custom initialization of the reader,
		// then proceed with actually loading the bean definitions.
		initBeanDefinitionReader(beanDefinitionReader);
		loadBeanDefinitions(beanDefinitionReader);
	}
```

AbstractXmlApplicationContext -> AbstractBeanDefinitionReader  -> XmlBeanDefinitionReader），一直调用到XmlBeanDefinitionReader 类的doLoadBeanDefinitions方法。

```
	protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
			throws BeanDefinitionStoreException {

		try {
            // 将XML文档中的信息保存到Document对象中
			Document doc = doLoadDocument(inputSource, resource);
            // 解析Document获取BeanDefinition信息，并进行注册
			int count = registerBeanDefinitions(doc, resource);
			if (logger.isDebugEnabled()) {
				logger.debug("Loaded " + count + " bean definitions from " + resource);
			}
			return count;
		}
     }


	public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
		BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
		int countBefore = getRegistry().getBeanDefinitionCount();
		documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
		return getRegistry().getBeanDefinitionCount() - countBefore;
	}

```

先来看看 XmlBeanDefinitionReader.createReaderContext()方法

```
	public XmlReaderContext createReaderContext(Resource resource) {
		return new XmlReaderContext(resource, this.problemReporter, this.eventListener,
				this.sourceExtractor, this, getNamespaceHandlerResolver());
	}


	public NamespaceHandlerResolver getNamespaceHandlerResolver() {
		if (this.namespaceHandlerResolver == null) {
			this.namespaceHandlerResolver = createDefaultNamespaceHandlerResolver();
		}
		return this.namespaceHandlerResolver;
	}


	protected NamespaceHandlerResolver createDefaultNamespaceHandlerResolver() {
		ClassLoader cl = (getResourceLoader() != null ? getResourceLoader().getClassLoader() : getBeanClassLoader());
		return new DefaultNamespaceHandlerResolver(cl);
	}
```

至此，13个NamespaceHandlerResolver初始化成功。

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/namespaceHandler.jpg" width="920px" > </div><br>


再进入DefaultBeanDefinitionDocumentReader类的registerBeanDefinitions方法

```
	@Override
	public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
		this.readerContext = readerContext;
		doRegisterBeanDefinitions(doc.getDocumentElement());
	}
```

继续进入到该类的doRegisterBeanDefinitions方法看看，这是真正干活的方法。

```
	protected void doRegisterBeanDefinitions(Element root) {
		// Any nested <beans> elements will cause recursion in this method. In
		// order to propagate and preserve <beans> default-* attributes correctly,
		// keep track of the current (parent) delegate, which may be null. Create
		// the new (child) delegate with a reference to the parent for fallback purposes,
		// then ultimately reset this.delegate back to its original (parent) reference.
		// this behavior emulates a stack of delegates without actually necessitating one.
		BeanDefinitionParserDelegate parent = this.delegate;
		this.delegate = createDelegate(getReaderContext(), root, parent);

		if (this.delegate.isDefaultNamespace(root)) {
			String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
			if (StringUtils.hasText(profileSpec)) {
				String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
						profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
				// We cannot use Profiles.of(...) since profile expressions are not supported
				// in XML config. See SPR-12458 for details.
				if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
					if (logger.isDebugEnabled()) {
						logger.debug("Skipped XML bean definition file due to specified profiles [" + profileSpec +
								"] not matching: " + getReaderContext().getResource());
					}
					return;
				}
			}
		}

		preProcessXml(root);
		parseBeanDefinitions(root, this.delegate);
		postProcessXml(root);

		this.delegate = parent;
	}

```

进入该类的parseBeanDefinitions()方法：

```
	protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
		if (delegate.isDefaultNamespace(root)) {
			NodeList nl = root.getChildNodes();
			for (int i = 0; i < nl.getLength(); i++) {
				Node node = nl.item(i);
				if (node instanceof Element) {
					Element ele = (Element) node;
					if (delegate.isDefaultNamespace(ele)) {
                        // 解析默认元素，例如import、bean
						parseDefaultElement(ele, delegate);
					}
					else {
                        // 解析自定义元素，比如aop、mvc、tx
						delegate.parseCustomElement(ele);
					}
				}
			}
		}
		else {
            // 解析自定义元素，比如aop、mvc、tx
			delegate.parseCustomElement(root);
		}
	}
```

解析默认元素：

```
	private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
		if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
			importBeanDefinitionResource(ele);
		}
		else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
			processAliasRegistration(ele);
		}
		else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
			processBeanDefinition(ele, delegate);
		}
		else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
			// recurse
			doRegisterBeanDefinitions(ele);
		}
	}
```

解析自定义元素（AOP标签、tx标签）：

BeanDefinitionParserDelegate类的parseCustomElement方法：

```
	@Nullable
	public BeanDefinition parseCustomElement(Element ele) {
		return parseCustomElement(ele, null);
	}


	@Nullable
	public BeanDefinition parseCustomElement(Element ele, @Nullable BeanDefinition containingBd) {
        // 根据指定元素，查找beans根标签中xmlns:xxx属性至
        // 比如根据aop:xxx，找到xmlns:aop的值
		String namespaceUri = getNamespaceURI(ele);
		if (namespaceUri == null) {
			return null;
		}
        // 根据命名空间URL解析出对应的NamespaceHandler
		NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
		if (handler == null) {
			error("Unable to locate Spring NamespaceHandler for XML schema namespace [" + namespaceUri + "]", ele);
			return null;
		}
        // 调用NamespaceHandler执行解析
		return handler.parse(ele, new ParserContext(this.readerContext, this, containingBd));
	}
```

进入DefaultNamespaceHandlerResolver类的resolve方法看看：

```
	@Override
	@Nullable
	public NamespaceHandler resolve(String namespaceUri) {
		Map<String, Object> handlerMappings = getHandlerMappings();
        // 查找NamespaceHandler
		Object handlerOrClassName = handlerMappings.get(namespaceUri);
		if (handlerOrClassName == null) {
			return null;
		}
		else if (handlerOrClassName instanceof NamespaceHandler) {
			return (NamespaceHandler) handlerOrClassName;
		}
		else {
			String className = (String) handlerOrClassName;
			try {
				Class<?> handlerClass = ClassUtils.forName(className, this.classLoader);
				if (!NamespaceHandler.class.isAssignableFrom(handlerClass)) {
					throw new FatalBeanException("Class [" + className + "] for namespace [" + namespaceUri +
							"] does not implement the [" + NamespaceHandler.class.getName() + "] interface");
				}
				NamespaceHandler namespaceHandler = (NamespaceHandler) BeanUtils.instantiateClass(handlerClass);
                // 调用对应的NamespaceHandler的子类方法，初始化自定义标签对应的功能处理组件
				namespaceHandler.init();
				handlerMappings.put(namespaceUri, namespaceHandler);
				return namespaceHandler;
			}
			catch (ClassNotFoundException ex) {
				throw new FatalBeanException("Could not find NamespaceHandler class [" + className +
						"] for namespace [" + namespaceUri + "]", ex);
			}
			catch (LinkageError err) {
				throw new FatalBeanException("Unresolvable class definition for NamespaceHandler class [" +
						className + "] for namespace [" + namespaceUri + "]", err);
			}
		}
	}

```

namespaceHandler.init();这个方法是很重要的。它实现了自定义标签到处理类的注册工作，不过NamespaceHandler是一个接口，具体的init方法需要不同的实现类进行实现，我们通过AopNamespaceHandler了解一下init的作用，其中aop:config标签是由ConfigBeanDefinitionParser类进行处理：

```
public class AopNamespaceHandler extends NamespaceHandlerSupport {

	@Override
	public void init() {
		// In 2.0 XSD as well as in 2.1 XSD.
		registerBeanDefinitionParser("config", new ConfigBeanDefinitionParser());
		registerBeanDefinitionParser("aspectj-autoproxy", new AspectJAutoProxyBeanDefinitionParser());
		registerBeanDefinitionDecorator("scoped-proxy", new ScopedProxyBeanDefinitionDecorator());

		// Only in 2.0 XSD: moved to context namespace as of 2.1
		registerBeanDefinitionParser("spring-configured", new SpringConfiguredBeanDefinitionParser());
	}
```

NamespaceHandlerSupport中：

```
public abstract class NamespaceHandlerSupport implements NamespaceHandler {

	private final Map<String, BeanDefinitionParser> parsers = new HashMap<>();

	protected final void registerBeanDefinitionParser(String elementName, BeanDefinitionParser parser) {
		this.parsers.put(elementName, parser);
	}
```

```
class ConfigBeanDefinitionParser implements BeanDefinitionParser {

	@Override
	@Nullable
	public BeanDefinition parse(Element element, ParserContext parserContext) {
```


至此，我们了解到了xml中的aop标签都是由哪些类进行处理的了。不过init方法只是注册了标签和处理类的对应关系，那么什么时候调用处理类进行解析的呢？我们再回到BeanDefinitionParserDelegate类的parseCustomElement方法看看。


```
	@Nullable
	public BeanDefinition parseCustomElement(Element ele) {
		return parseCustomElement(ele, null);
	}


	@Nullable
	public BeanDefinition parseCustomElement(Element ele, @Nullable BeanDefinition containingBd) {
        // 根据指定元素，查找beans根标签中xmlns:xxx属性至
        // 比如根据aop:xxx，找到xmlns:aop的值
		String namespaceUri = getNamespaceURI(ele);
		if (namespaceUri == null) {
			return null;
		}
        // 根据命名空间URL解析出对应的NamespaceHandler
		NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
		if (handler == null) {
			error("Unable to locate Spring NamespaceHandler for XML schema namespace [" + namespaceUri + "]", ele);
			return null;
		}
        // 调用NamespaceHandler执行解析
		return handler.parse(ele, new ParserContext(this.readerContext, this, containingBd));
	}

```

最后一行执行了parse方法，那么parse方法，在哪呢？我们需要到NamespaceHandlerSupport类中去看看，它是实现NamespaceHandler接口的，并且AopNamespaceHandler是继承了NamespaceHandlerSupport类，那么该方法也会继承到AopNamespaceHandler类中。

```
	@Override
	@Nullable
	public BeanDefinition parse(Element element, ParserContext parserContext) {
		BeanDefinitionParser parser = findParserForElement(element, parserContext);
        // 调用对应的BeanDefinitionParser进行parse解析，例如ConfigBeanDefinitionParser
		return (parser != null ? parser.parse(element, parserContext) : null);
	}

	/**
	 * Locates the {@link BeanDefinitionParser} from the register implementations using
	 * the local name of the supplied {@link Element}.
	 */
	@Nullable
	private BeanDefinitionParser findParserForElement(Element element, ParserContext parserContext) {
		String localName = parserContext.getDelegate().getLocalName(element);
        // this.parsers由init方法初始化
        // 根据标签名称，找到对应的处理器
		BeanDefinitionParser parser = this.parsers.get(localName);
		if (parser == null) {
			parserContext.getReaderContext().fatal(
					"Cannot locate BeanDefinitionParser for element [" + localName + "]", element);
		}
		return parser;
	}
```

后续具体想了解哪个自定义标签的处理逻辑，可以自行去查找xxxNamespaceHandler类进行分析。

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/解析xml里面定义的bean信息.png" width="920px" > </div><br>


##### 1.2.4.3.2 注解解析bean定义信息

```
	public AnnotationConfigApplicationContext(Class<?>... componentClasses) {
		this();
		register(componentClasses);
		refresh();
	}
```

* step1.在this()中已经创建了DefaultListableBeanFactory，并且已经加载了5个默认的beanDefinition。
* step2.register()中将componentClasses也放到了beanDefinition中

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/注解创建beanFactory.jpg" width="620px" > </div><br>


#### 1.2.4.4 创建Bean流程分析

```
  //创建非懒加载方式的单例Bean实例（未设置属性）
  //填充属性
  //初始化实例（比如调用init-method方法）
  //调用BeanPostProcessor（后置处理器）对实例bean进行后置处理
  //11、 Instantiate all remaining (non-lazy-init) singletons.
  finishBeanFactoryInitialization(beanFactory);
```

AbstractApplicationContext.finishBeanFactoryInitialization()方法：

```
	protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
		// Initialize conversion service for this context.
		if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
				beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
			beanFactory.setConversionService(
					beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
		}

		// Register a default embedded value resolver if no bean post-processor
		// (such as a PropertyPlaceholderConfigurer bean) registered any before:
		// at this point, primarily for resolution in annotation attribute values.
		if (!beanFactory.hasEmbeddedValueResolver()) {
			beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
		}

		// Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
		String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
		for (String weaverAwareName : weaverAwareNames) {
			getBean(weaverAwareName);
		}

		// Stop using the temporary ClassLoader for type matching.
		beanFactory.setTempClassLoader(null);

		// Allow for caching all bean definition metadata, not expecting further changes.
		beanFactory.freezeConfiguration();

		// Instantiate all remaining (non-lazy-init) singletons.
		beanFactory.preInstantiateSingletons();
	}
```

继续进入DefaultListableBeanFactory类的preInstantiateSingletons方法：

```
	@Override
	public void preInstantiateSingletons() throws BeansException {
		if (logger.isTraceEnabled()) {
			logger.trace("Pre-instantiating singletons in " + this);
		}

		// Iterate over a copy to allow for init methods which in turn register new bean definitions.
		// While this may not be part of the regular factory bootstrap, it does otherwise work fine.
		List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

		// Trigger initialization of all non-lazy singleton beans...
		for (String beanName : beanNames) {
			RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
			if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
				if (isFactoryBean(beanName)) {
					Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
					if (bean instanceof FactoryBean) {
						final FactoryBean<?> factory = (FactoryBean<?>) bean;
						boolean isEagerInit;
						if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
							isEagerInit = AccessController.doPrivileged((PrivilegedAction<Boolean>)
											((SmartFactoryBean<?>) factory)::isEagerInit,
									getAccessControlContext());
						}
						else {
							isEagerInit = (factory instanceof SmartFactoryBean &&
									((SmartFactoryBean<?>) factory).isEagerInit());
						}
						if (isEagerInit) {
							getBean(beanName);
						}
					}
				}
				else {
					getBean(beanName);
				}
			}
		}

		// Trigger post-initialization callback for all applicable beans...
		for (String beanName : beanNames) {
			Object singletonInstance = getSingleton(beanName);
			if (singletonInstance instanceof SmartInitializingSingleton) {
				final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
				if (System.getSecurityManager() != null) {
					AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
						smartSingleton.afterSingletonsInstantiated();
						return null;
					}, getAccessControlContext());
				}
				else {
					smartSingleton.afterSingletonsInstantiated();
				}
			}
		}
	}
```

进入AbstractBeanFactory.getBean()方法：

```
	@Override
	public Object getBean(String name) throws BeansException {
		return doGetBean(name, null, null, false);
	}
```

doGetBean：

```
				// Create bean instance.
				if (mbd.isSingleton()) {
					sharedInstance = getSingleton(beanName, () -> {
						try {
							return createBean(beanName, mbd, args);
						}
						catch (BeansException ex) {
							// Explicitly remove instance from singleton cache: It might have been put there
							// eagerly by the creation process, to allow for circular reference resolution.
							// Also remove any beans that received a temporary reference to the bean.
							destroySingleton(beanName);
							throw ex;
						}
					});
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}

				else if (mbd.isPrototype()) {
					// It's a prototype -> create a new instance.
					Object prototypeInstance = null;
					try {
						beforePrototypeCreation(beanName);
						prototypeInstance = createBean(beanName, mbd, args);
					}
					finally {
						afterPrototypeCreation(beanName);
					}
					bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
				}

				else {

```

进入AbstractAutowireCapableBeanFactory类的方法createBean：

```
		try {
			// Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
			Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
			if (bean != null) {
				return bean;
			}
		}
		catch (Throwable ex) {
			throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
					"BeanPostProcessor before instantiation of bean failed", ex);
		}

		try {
			Object beanInstance = doCreateBean(beanName, mbdToUse, args);
			if (logger.isTraceEnabled()) {
				logger.trace("Finished creating instance of bean '" + beanName + "'");
			}
			return beanInstance;
		}
```

doCreateBean()方法：

```
	protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
			throws BeanCreationException {

		// Instantiate the bean.
		BeanWrapper instanceWrapper = null;
		if (mbd.isSingleton()) {
			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
		}
		if (instanceWrapper == null) {
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}
		final Object bean = instanceWrapper.getWrappedInstance();
		Class<?> beanType = instanceWrapper.getWrappedClass();
		if (beanType != NullBean.class) {
			mbd.resolvedTargetType = beanType;
		}

        ...

		// Initialize the bean instance.
		Object exposedObject = bean;
		try {
			populateBean(beanName, mbd, instanceWrapper);
			exposedObject = initializeBean(beanName, exposedObject, mbd);
		}
		catch (Throwable ex) {
			if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
				throw (BeanCreationException) ex;
			}
			else {
				throw new BeanCreationException(
						mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
			}
		}


```

先去看看initializeBean方法是如何调用BeanPostProcessor的，因为这个牵扯到我们对于AOP动态代理的理解。

```
	protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
		if (System.getSecurityManager() != null) {
			AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
				invokeAwareMethods(beanName, bean);
				return null;
			}, getAccessControlContext());
		}
		else {
			invokeAwareMethods(beanName, bean);
		}

		Object wrappedBean = bean;
		if (mbd == null || !mbd.isSynthetic()) {
			wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
		}

		try {
			invokeInitMethods(beanName, wrappedBean, mbd);
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					(mbd != null ? mbd.getResourceDescription() : null),
					beanName, "Invocation of init method failed", ex);
		}
		if (mbd == null || !mbd.isSynthetic()) {
            // AOP动态代理就是在此处发生的
			wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
		}

		return wrappedBean;
	}
```

# 2.组件添加到IOC容器

# 3.bean生命周期



# 4.赋值及依赖注入

# 5.Aware获取底层组件

# 6.AOP

## xml配置文件方式

AopNamespaceHandlelr

```
public class AopNamespaceHandler extends NamespaceHandlerSupport {

	/**
	 * Register the {@link BeanDefinitionParser BeanDefinitionParsers} for the
	 * '{@code config}', '{@code spring-configured}', '{@code aspectj-autoproxy}'
	 * and '{@code scoped-proxy}' tags.
	 */
	@Override
	public void init() {
		// In 2.0 XSD as well as in 2.1 XSD.
		registerBeanDefinitionParser("config", new ConfigBeanDefinitionParser());
		registerBeanDefinitionParser("aspectj-autoproxy", new AspectJAutoProxyBeanDefinitionParser());
		registerBeanDefinitionDecorator("scoped-proxy", new ScopedProxyBeanDefinitionDecorator());

		// Only in 2.0 XSD: moved to context namespace as of 2.1
		registerBeanDefinitionParser("spring-configured", new SpringConfiguredBeanDefinitionParser());
	}

}
```

## 注解方式

# 7.事务

## 事务管理之基于AspectJ的XML方式

```
public class TxNamespaceHandler extends NamespaceHandlerSupport {

	static final String TRANSACTION_MANAGER_ATTRIBUTE = "transaction-manager";

	static final String DEFAULT_TRANSACTION_MANAGER_BEAN_NAME = "transactionManager";


	static String getTransactionManagerName(Element element) {
		return (element.hasAttribute(TRANSACTION_MANAGER_ATTRIBUTE) ?
				element.getAttribute(TRANSACTION_MANAGER_ATTRIBUTE) : DEFAULT_TRANSACTION_MANAGER_BEAN_NAME);
	}


	@Override
	public void init() {
		registerBeanDefinitionParser("advice", new TxAdviceBeanDefinitionParser());
		registerBeanDefinitionParser("annotation-driven", new AnnotationDrivenBeanDefinitionParser());
		registerBeanDefinitionParser("jta-transaction-manager", new JtaTransactionManagerBeanDefinitionParser());
	}

}
```

分析TxAdviceBeanDefinitionParser：

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/TxAdviceBeanDefinitionParser.png" width="620px" > </div><br>

TxAdviceBeanDefinitionParser的继承体系，TxAdviceBeanDefinitionParser-> AbstractSingleBeanDefinitionParser -> AbstractBeanDefinitionParser，因为根据上面loadBeanDefinitions流程源码分析，我们知道自定义元素的解析工作是从一个namespaceHandler.parser方法开始的，该方法在AbstractBeanDefinitionParser类中：

AbstractBeanDefinitionParser.parse()：

```
	@Override
	@Nullable
	public final BeanDefinition parse(Element element, ParserContext parserContext) {
        // 解析文档，获取BeanDefinition
		AbstractBeanDefinition definition = parseInternal(element, parserContext);
		if (definition != null && !parserContext.isNested()) {
			try {
				String id = resolveId(element, definition, parserContext);
				if (!StringUtils.hasText(id)) {
					parserContext.getReaderContext().error(
							"Id is required for element '" + parserContext.getDelegate().getLocalName(element)
									+ "' when used as a top-level tag", element);
				}
				String[] aliases = null;
				if (shouldParseNameAsAliases()) {
					String name = element.getAttribute(NAME_ATTRIBUTE);
					if (StringUtils.hasLength(name)) {
						aliases = StringUtils.trimArrayElements(StringUtils.commaDelimitedListToStringArray(name));
					}
				}
				BeanDefinitionHolder holder = new BeanDefinitionHolder(definition, id, aliases);
                // 完成BeanDefinition的注册
				registerBeanDefinition(holder, parserContext.getRegistry());
				if (shouldFireEvents()) {
					BeanComponentDefinition componentDefinition = new BeanComponentDefinition(holder);
					postProcessComponentDefinition(componentDefinition);
					parserContext.registerComponent(componentDefinition);
				}
			}
			catch (BeanDefinitionStoreException ex) {
				String msg = ex.getMessage();
				parserContext.getReaderContext().error((msg != null ? msg : ex.toString()), element);
				return null;
			}
		}
		return definition;
	}
```

重点关心如何获取BeanDefinition对象的，所以接下来，我们进入parseInternal方法，该方法在AbstractSingleBeanDefinitionParser中：

```
	@Override
	protected final AbstractBeanDefinition parseInternal(Element element, ParserContext parserContext) {
		BeanDefinitionBuilder builder = BeanDefinitionBuilder.genericBeanDefinition();
		String parentName = getParentName(element);
		if (parentName != null) {
			builder.getRawBeanDefinition().setParentName(parentName);
		}
        // 找到真正处理tx:advisor标签的类
		Class<?> beanClass = getBeanClass(element);
		if (beanClass != null) {
			builder.getRawBeanDefinition().setBeanClass(beanClass);
		}
		else {
			String beanClassName = getBeanClassName(element);
			if (beanClassName != null) {
				builder.getRawBeanDefinition().setBeanClassName(beanClassName);
			}
		}
		builder.getRawBeanDefinition().setSource(parserContext.extractSource(element));
		BeanDefinition containingBd = parserContext.getContainingBeanDefinition();
		if (containingBd != null) {
			// Inner bean definition must receive same scope as containing bean.
			builder.setScope(containingBd.getScope());
		}
		if (parserContext.isDefaultLazyInit()) {
			// Default-lazy-init applies to custom bean definitions as well.
			builder.setLazyInit(true);
		}
        // 主要是解析tx:advisor中的子标签信息，并将解析到的信息封装到BeanDefition中
		doParse(element, parserContext, builder);
		return builder.getBeanDefinition();
	}

```

来到TxAdviceBeanDefinitionParser类，因为getBeanClass方法和doParser方法都在该类里面：

```
	@Override
	protected Class<?> getBeanClass(Element element) {
        // 这个类就是tx:advisor标签对应的处理类
		return TransactionInterceptor.class;
	}

	@Override
	protected void doParse(Element element, ParserContext parserContext, BeanDefinitionBuilder builder) {
        // 找到事务管理器，将它作为属性引用关联到TransactionInterceptor类中
		builder.addPropertyReference("transactionManager", TxNamespaceHandler.getTransactionManagerName(element));

		List<Element> txAttributes = DomUtils.getChildElementsByTagName(element, ATTRIBUTES_ELEMENT);
		if (txAttributes.size() > 1) {
			parserContext.getReaderContext().error(
					"Element <attributes> is allowed at most once inside element <advice>", element);
		}
		else if (txAttributes.size() == 1) {
			// Using attributes source.
			Element attributeSourceElement = txAttributes.get(0);
			RootBeanDefinition attributeSourceDefinition = parseAttributeSource(attributeSourceElement, parserContext);
            // 解析tx:attributes标签，并将解析到值关联到TransactionInterceptor类中
			builder.addPropertyValue("transactionAttributeSource", attributeSourceDefinition);
		}
		else {
			// Assume annotations source.
			builder.addPropertyValue("transactionAttributeSource",
					new RootBeanDefinition("org.springframework.transaction.annotation.AnnotationTransactionAttributeSource"));
		}
	}
```

重点了解一下TransactionInterceptor这个类，它是我们分析的最终目标：

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/TransactionInterceptor.png" width="820px" > </div><br>

```
public class TransactionInterceptor extends TransactionAspectSupport implements MethodInterceptor, Serializable {

	@Override
	@Nullable
	public Object invoke(MethodInvocation invocation) throws Throwable {
		// Work out the target class: may be {@code null}.
		// The TransactionAttributeSource should be passed the target class
		// as well as the method, which may be from an interface.
		Class<?> targetClass = (invocation.getThis() != null ? AopUtils.getTargetClass(invocation.getThis()) : null);

		// Adapt to TransactionAspectSupport's invokeWithinTransaction...
        // 真正实现AOP事务功能的方法
		return invokeWithinTransaction(invocation.getMethod(), targetClass, invocation::proceed);
	}

```

invokeWithInTransaction方法在TransactionInterceptor类的父类TransactionAspectSupport中

```
	@Nullable
	protected Object invokeWithinTransaction(Method method, @Nullable Class<?> targetClass,
			final InvocationCallback invocation) throws Throwable {

		// If the transaction attribute is null, the method is non-transactional.
		TransactionAttributeSource tas = getTransactionAttributeSource();
		final TransactionAttribute txAttr = (tas != null ? tas.getTransactionAttribute(method, targetClass) : null);
		final TransactionManager tm = determineTransactionManager(txAttr);

		PlatformTransactionManager ptm = asPlatformTransactionManager(tm);
		final String joinpointIdentification = methodIdentification(method, targetClass, txAttr);

		if (txAttr == null || !(ptm instanceof CallbackPreferringPlatformTransactionManager)) {
			// Standard transaction demarcation with getTransaction and commit/rollback calls.
            // 封装事务状态信息，开启事务
			TransactionInfo txInfo = createTransactionIfNecessary(ptm, txAttr, joinpointIdentification);

			Object retVal;
			try {
				// This is an around advice: Invoke the next interceptor in the chain.
				// This will normally result in a target object being invoked.
				retVal = invocation.proceedWithInvocation();
			}
			catch (Throwable ex) {
				// target invocation exception
                // 回滚事务
				completeTransactionAfterThrowing(txInfo, ex);
				throw ex;
			}
			finally {
                // 清除事务
				cleanupTransactionInfo(txInfo);
			}

			if (vavrPresent && VavrDelegate.isVavrTry(retVal)) {
				// Set rollback-only in case of Vavr failure matching our rollback rules...
				TransactionStatus status = txInfo.getTransactionStatus();
				if (status != null && txAttr != null) {
					retVal = VavrDelegate.evaluateTryFailure(retVal, txAttr, status);
				}
			}

            // 提交事务
			commitTransactionAfterReturning(txInfo);
			return retVal;
		}
	}

```

进入createTransactionIfNecessary方法看看，事务是如何开启的

```
	protected TransactionInfo createTransactionIfNecessary(@Nullable PlatformTransactionManager tm,
			@Nullable TransactionAttribute txAttr, final String joinpointIdentification) {

		// If no name specified, apply method identification as transaction name.
		if (txAttr != null && txAttr.getName() == null) {
			txAttr = new DelegatingTransactionAttribute(txAttr) {
				@Override
				public String getName() {
					return joinpointIdentification;
				}
			};
		}

		TransactionStatus status = null;
		if (txAttr != null) {
			if (tm != null) {
				status = tm.getTransaction(txAttr);
			}
			else {
				if (logger.isDebugEnabled()) {
					logger.debug("Skipping transactional joinpoint [" + joinpointIdentification +
							"] because no transaction manager has been configured");
				}
			}
		}
		return prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);
	}
```

进入AbstractPlatformTransactionManager中的getTransaction方法继续了解事务是如何开启的：

```
	@Override
	public final TransactionStatus getTransaction(@Nullable TransactionDefinition definition)
			throws TransactionException {

        // 事务传播特性处理逻辑
		if (isExistingTransaction(transaction)) {
			// Existing transaction found -> check propagation behavior to find out how to behave.
			return handleExistingTransaction(def, transaction, debugEnabled);
		}

		// Check definition settings for new transaction.
		if (def.getTimeout() < TransactionDefinition.TIMEOUT_DEFAULT) {
			throw new InvalidTimeoutException("Invalid transaction timeout", def.getTimeout());
		}

		// No existing transaction found -> check propagation behavior to find out how to proceed.
		if (def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_MANDATORY) {
			throw new IllegalTransactionStateException(
					"No existing transaction found for transaction marked with propagation 'mandatory'");
		}
		else if (def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRED ||
				def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW ||
				def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
			SuspendedResourcesHolder suspendedResources = suspend(null);
			if (debugEnabled) {
				logger.debug("Creating new transaction with name [" + def.getName() + "]: " + def);
			}
			try {
				boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
				DefaultTransactionStatus status = newTransactionStatus(
						def, transaction, true, newSynchronization, debugEnabled, suspendedResources);
                // 开启事务
				doBegin(transaction, def);
				prepareSynchronization(status, def);
				return status;
			}
			catch (RuntimeException | Error ex) {
				resume(null, suspendedResources);
				throw ex;
			}
		}
		else {
			
```

接下来，该进入doBegin方法了，不过该方法在具体的平台事务管理器的子类中，我们此处使用DataSourceTransactionManager子类进行源码跟踪：

```
	@Override
	protected void doBegin(Object transaction, TransactionDefinition definition) {
		DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;
		Connection con = null;

		try {
			if (!txObject.hasConnectionHolder() ||
					txObject.getConnectionHolder().isSynchronizedWithTransaction()) {
				Connection newCon = obtainDataSource().getConnection();
				if (logger.isDebugEnabled()) {
					logger.debug("Acquired Connection [" + newCon + "] for JDBC transaction");
				}
				txObject.setConnectionHolder(new ConnectionHolder(newCon), true);
			}

			txObject.getConnectionHolder().setSynchronizedWithTransaction(true);
			con = txObject.getConnectionHolder().getConnection();

			Integer previousIsolationLevel = DataSourceUtils.prepareConnectionForTransaction(con, definition);
			txObject.setPreviousIsolationLevel(previousIsolationLevel);
			txObject.setReadOnly(definition.isReadOnly());

			// Switch to manual commit if necessary. This is very expensive in some JDBC drivers,
			// so we don't want to do it unnecessarily (for example if we've explicitly
			// configured the connection pool to set it already).
			if (con.getAutoCommit()) {
				txObject.setMustRestoreAutoCommit(true);
				if (logger.isDebugEnabled()) {
					logger.debug("Switching JDBC Connection [" + con + "] to manual commit");
				}
                // 开启事务
				con.setAutoCommit(false);
			}
```

DataSourceTransactionManager的事务管理是通过底层的JDBC代码实现的，但是不同的平台事务管理器，它们底层的事务处理也是不同的。



## 声明式事务

* service类上或者方法上加注解：
  - 类上加@Transactional：表示该类中所有的方法都被事务管理
  - 方法上加@Transactional：表示只有该方法被事务管理
* 开启事务注解：

```
    <!-- 配置事务管理器 -->
    <bean id="transactionManager"
          class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <!-- 注入数据库连接池 -->
        <property name="dataSource" ref="dataSource" />
    </bean>

    <!-- 配置基于注解的声明式事务 -->
    <tx:annotation-driven transaction-manager="transactionManager" />
```
* @EnableTransactionManagement可以代替xml配置文件

# 8.BeanFactory的两个重要后置处理器


# BeanPostProcessor
ApplicationContextAwareProcessor->ApplicationContextAware

ApplicationContextAwareProcessor我们发现, 这个后置处理器其实就是判断我们的bean有没有实现ApplicationContextAware 接口,并处理相应的逻辑,其实所有的后置处理器原理均如此.


BeanValidationPostProcess分析:数据校验
当对象创建完,给bean赋值后,在WEB用得特别多;把页面提交的值进行校验


InitDestroyAnnotationBeanPostProcessor
此处理器用来处理@PostConstruct, @PreDestroy, 怎么知道这两注解是前后开始调用的呢, 就是 InitDestroyAnnotationBeanPostProcessor这个处理的

 Spring底层对BeanPostProcessor的使用, 包括bean的赋值, 注入其它组件, 生命周期注解功能,@Async, 等等


# @Autowired源码解析


# Aware注入spring底层组件原理
自定义组件想要使用Spring容器底层的组件(ApplicationContext, BeanFactory, ......)
    思路: 自定义组件实现xxxAware, 在创建对象的时候, 会调用接口规定的方法注入到相关组件:Aware

# AOP源码解析

@EnableAspectJAutoProxy；核心从这个入手,AOP整个功能要启作用,就是靠这个,加入它才有AOP

//导入了此类,点进去看
@Import(AspectJAutoProxyRegistrar.class)
public @interface EnableAspectJAutoProxy {
    //proxyTargetClass属性，默认false，采用JDK动态代理织入增强(实现接口的方式)；如果设为true，则采用CGLIB动态代理织入增强
 	boolean proxyTargetClass() default false;
    //通过aop框架暴露该代理对象，aopContext能够访问
 	boolean exposeProxy() default false;
}


它引入AspectJAutoProxyRegistrar, 并实现了ImportBeanDefinitionRegistrar接口

ImportBeanDefinitionRegistrar接口作用: 能给容器中自定义注册组件

看注册bean的如何处理?
AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry);
注册一个这个组件, 如果有需要的话....

想注册一个AnnotationAwareAspectJAutoProxyCreator的组件, 如果registry已经有了的话,就执行以下操作;
但是我们的注册中还没有, 第一次, 所以来创建一个cls, 用registry把bean的定义做好, bean的名叫做internalAutoProxyCreator

其实就是利用@EnableAspectJAutoProxy中的AspectJAutoProxyRegistrar给我们容器中注册一个AnnotationAwareAspectJAutoProxyCreator组件;
翻译过来其实就叫做 ”注解装配模式的ASPECT切面自动代理创建器”组件

判断if(registry.containsBeanDefinition(ATUO_PROXY_CREATOR_BEAN_NAME))
{
    如果容器中bean已经有了 internalAutoProxyCreator, 执行内部内容
}
else
创建AnnotationAwareAspectJAutoProxyCreator信息; 把此bean注册在registry中.
做完后, 相当于
其实就是 ATUO_PROXY_CREATOR_BEAN_NAME值为internalAutoProxyCreator,给容器中注册internalAutoProxyCreator组件, 该组件类型为AnnotationAwareAspectJAutoProxyCreator.class



一, 分析创建和注册AnnotationAwareAspectJAutoProxyCreator的流程:
1）、register()传入配置类，准备创建ioc容器
2）、注册配置类，调用refresh（）刷新创建容器；
3）、registerBeanPostProcessors(beanFactory);注册bean的后置处理器来方便拦截bean的创建(主要是分析创建AnnotationAwareAspectJAutoProxyCreator)；
 	1）、 先获取ioc容器已经定义了的需要创建对象的所有BeanPostProcessor
 	2）、给容器中加别的BeanPostProcessor
 	3）、优先注册实现了PriorityOrdered接口的BeanPostProcessor；
 	4）、再给容器中注册实现了Ordered接口的BeanPostProcessor；
 	5）、注册没实现优先级接口的BeanPostProcessor；
 	6）、注册BeanPostProcessor，实际上就是创建BeanPostProcessor对象，保存在容器中；
 		创建internalAutoProxyCreator的BeanPostProcessor【其实就是AnnotationAwareAspectJAutoProxyCreator】
 		1）、创建Bean的实例
 		2）、populateBean；给bean的各种属性赋值
 		3）、initializeBean：初始化bean；
 				1）、invokeAwareMethods()：处理Aware接口的方法回调
 				2）、applyBeanPostProcessorsBeforeInitialization()：应用后置处理器的postProcessBeforeInitialization（）
 				3）、invokeInitMethods()；执行自定义的初始化方法
 				4）、applyBeanPostProcessorsAfterInitialization()；执行后置处理器的postProcessAfterInitialization（）；
 		4）、BeanPostProcessor(AnnotationAwareAspectJAutoProxyCreator)创建成功；--》aspectJAdvisorsBuilder
 	7）、把BeanPostProcessor注册到BeanFactory中；
 		beanFactory.addBeanPostProcessor(postProcessor);

注意:以上是创建和注册AnnotationAwareAspectJAutoProxyCreator的过程
  
  			AnnotationAwareAspectJAutoProxyCreator => InstantiationAwareBeanPostProcessor





二, 如何创建增强的Caculator增强bean的流程:

  1,refresh--->finishBeanFactoryInitialization(beanFactory);完成BeanFactory初始化工作；创建剩下的单实例bean
  		1）、遍历获取容器中所有的Bean，依次创建对象getBean(beanName);
  				getBean->doGetBean()->getSingleton()->
  		2）、创建bean
  				【AnnotationAwareAspectJAutoProxyCreator在所有bean创建之前会有一个拦截，InstantiationAwareBeanPostProcessor，会调用postProcessBeforeInstantiation()】
  				2.1）、先从缓存中获取当前bean，如果能获取到，说明bean是之前被创建过的，直接使用，否则再创建；
  					只要创建好的Bean都会被缓存起来
  				2.2）、createBean（）;创建bean；
  					AnnotationAwareAspectJAutoProxyCreator 会在任何bean创建之前先尝试返回bean的实例
  					【BeanPostProcessor是在Bean对象创建完成初始化前后调用的】
  					【InstantiationAwareBeanPostProcessor是在创建Bean实例之前先尝试用后置处理器返回对象的】
  					2.2.1）、resolveBeforeInstantiation(beanName, mbdToUse);解析BeforeInstantiation,如果能返回代理对象就使用，如果不能就继续,后置处理器先尝试返回对象；
  			bean = applyBeanPostProcessorsBeforeInstantiation（）：
  			拿到所有后置处理器，如果是InstantiationAwareBeanPostProcessor;
  			就执行postProcessBeforeInstantiation
  			if (bean != null) {
					bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
			}
  
  					2.2.2）、doCreateBean(beanName, mbdToUse, args);真正的去创建一个bean实例；和单实例bean创建流程一样；
  		

			  			
  三,【AnnotationAwareAspectJAutoProxyCreator】作用:
 	InstantiationAwareBeanPostProcessor
  ：
  1）、每一个bean创建之前，调用postProcessBeforeInstantiation()；
  		关心MathCalculator和LogAspect的创建
  		1）、判断当前bean是否在advisedBeans中（保存了所有需要增强bean）
  		2）、判断当前bean是否是基础类型的Advice、Pointcut、Advisor、AopInfrastructureBean，
  			或者是否是切面（@Aspect）
  		3）、是否需要跳过
  			1）、获取候选的增强器（切面里面的通知方法）【List<Advisor> candidateAdvisors】
  				每一个封装的通知方法的增强器是 InstantiationModelAwarePointcutAdvisor；
  				判断每一个增强器是否是 AspectJPointcutAdvisor 类型的；返回true
  			2）、永远返回false
  
  2）、创建对象
  postProcessAfterInitialization；
  		return wrapIfNecessary(bean, beanName, cacheKey);//包装如果需要的情况下
  		1）、获取当前bean的所有增强器（通知方法）  Object[]  specificInterceptors
  			1、找到候选的所有的增强器（找哪些通知方法是需要切入当前bean方法的）
  			2、获取到能在bean使用的增强器。
  			3、给增强器排序
  		2）、保存当前bean在advisedBeans中；
  		3）、如果当前bean需要增强，创建当前bean的代理对象；
  			1）、获取所有增强器（通知方法）
  			2）、保存到proxyFactory
  			3）、创建代理对象：Spring自动决定
  				JdkDynamicAopProxy(config);jdk动态代理；
  				ObjenesisCglibAopProxy(config);cglib的动态代理；
  		4）、给容器中返回当前组件使用cglib增强了的代理对象；
  		5）、以后容器中获取到的就是这个组件的代理对象，执行目标方法的时候，代理对象就会执行通知方法的流程；


# 声明式事务

 @EnableTransactionalManagement开启基于注解的事务管理功能

# BeanFactory的两个重要后置处理器

BeanFactoryPostProcessor

作用如下：  		
* 在BeanFactory标准初始化之后调用，来定制和修改BeanFactory的内容；
* 所有的bean定义已经保存加载到beanFactory，但是bean的实例还未创建


BeanDefinitionRegistryPostProcessor

postProcessBeanDefinitionRegistry()在所有bean定义信息将要被加载，bean实例还未创建的。



扩展原理：
 * BeanPostProcessor：bean后置处理器，bean创建对象初始化前后进行拦截工作的
 * 
 * 1、BeanFactoryPostProcessor：beanFactory的后置处理器；
 * 		在BeanFactory标准初始化之后调用，来定制和修改BeanFactory的内容；
 * 		所有的bean定义已经保存加载到beanFactory，但是bean的实例还未创建
 * 
 * 
 * BeanFactoryPostProcessor原理:
 * 1)、ioc容器创建对象
 * 2)、invokeBeanFactoryPostProcessors(beanFactory);
 * 		如何找到所有的BeanFactoryPostProcessor并执行他们的方法；
 * 			1）、直接在BeanFactory中找到所有类型是BeanFactoryPostProcessor的组件，并执行他们的方法
 * 			2）、在初始化创建其他组件前面执行
 * 



 * 2、BeanDefinitionRegistryPostProcessor extends BeanFactoryPostProcessor
 * 		postProcessBeanDefinitionRegistry();
 * 		在所有bean定义信息将要被加载，bean实例还未创建的；
 * 
 * 		优先于BeanFactoryPostProcessor执行；
 * 		利用BeanDefinitionRegistryPostProcessor给容器中再额外添加一些组件；
 * 
 * 	原理：
 * 		1）、ioc创建对象
 * 		2）、refresh()-》invokeBeanFactoryPostProcessors(beanFactory);
 * 		3）、从容器中获取到所有的BeanDefinitionRegistryPostProcessor组件。
 * 			1、依次触发所有的postProcessBeanDefinitionRegistry()方法
 * 			2、再来触发postProcessBeanFactory()方法BeanFactoryPostProcessor； 为什么先执行postProcessBeanDefinitionRegistry()方法？两方法打断点，debug栈
 * 
 * 		4）、再来从容器中找到BeanFactoryPostProcessor组件；然后依次触发postProcessBeanFactory()方法
 **/ 	


# IOC容器处理流程


＝＝＝＝＝IOC流程＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝
Spring容器的refresh()【创建刷新】;
1、prepareRefresh()刷新前的预处理;
	1）、initPropertySources()初始化一些属性设置;子类自定义个性化的属性设置方法；
	2）、getEnvironment().validateRequiredProperties();检验属性的合法等
	3）、earlyApplicationEvents= new LinkedHashSet<ApplicationEvent>();保存容器中的一些早期的事件；
2、obtainFreshBeanFactory();获取BeanFactory；
	1）、refreshBeanFactory();刷新【创建】BeanFactory；
			110行：创建了一个this.beanFactory = new DefaultListableBeanFactory();
			设置id；
	2）、getBeanFactory();返回刚才GenericApplicationContext创建的BeanFactory对象；
	3）、将创建的BeanFactory【DefaultListableBeanFactory】返回；



3、prepareBeanFactory(beanFactory);BeanFactory的预准备工作（以上创建了beanFactory,现在对BeanFactory对象进行一些设置属性）；
	1）、设置BeanFactory的类加载器、支持表达式解析器...
	2）、添加部分BeanPostProcessor【ApplicationContextAwareProcessor】
	3）、设置忽略的自动装配的接口EnvironmentAware、EmbeddedValueResolverAware、xxx；
	4）、注册可以解析的自动装配；我们能直接在任何组件中自动注入：
			BeanFactory、ResourceLoader、ApplicationEventPublisher、ApplicationContext
	5）、添加BeanPostProcessor【ApplicationListenerDetector】
	6）、添加编译时的AspectJ；
	7）、给BeanFactory中注册一些能用的组件；
		environment【ConfigurableEnvironment】、
		systemProperties【Map<String, Object>】、
		systemEnvironment【Map<String, Object>】
4、postProcessBeanFactory(beanFactory);BeanFactory准备工作完成后进行的后置处理工作；
	1）、子类通过重写这个方法来在BeanFactory创建并预准备完成以后做进一步的设置
======================以上是BeanFactory的创建及预准备工作==================================


5、invokeBeanFactoryPostProcessors(beanFactory);执行BeanFactoryPostProcessor的方法；
	BeanFactoryPostProcessor：BeanFactory的后置处理器。在BeanFactory标准初始化之后执行的；
	两个接口：BeanFactoryPostProcessor、BeanDefinitionRegistryPostProcessor
	1）、执行BeanFactoryPostProcessor的方法；
		先执行BeanDefinitionRegistryPostProcessor
		1）、83行：获取所有的BeanDefinitionRegistryPostProcessor；
		2）、86行：看先执行实现了PriorityOrdered优先级接口的BeanDefinitionRegistryPostProcessor、
			postProcessor.postProcessBeanDefinitionRegistry(registry)
		3）、99行：在执行实现了Ordered顺序接口的BeanDefinitionRegistryPostProcessor；
			postProcessor.postProcessBeanDefinitionRegistry(registry)
		4）、109行：最后执行没有实现任何优先级或者是顺序接口的BeanDefinitionRegistryPostProcessors；
			postProcessor.postProcessBeanDefinitionRegistry(registry)
			
		
		再执行BeanFactoryPostProcessor的方法
		1）、139行：获取所有的BeanFactoryPostProcessor
		2）、147行：看先执行实现了PriorityOrdered优先级接口的BeanFactoryPostProcessor、
			postProcessor.postProcessBeanFactory()
		3）、167行：在执行实现了Ordered顺序接口的BeanFactoryPostProcessor；
			postProcessor.postProcessBeanFactory()
		4）、175行：最后执行没有实现任何优先级或者是顺序接口的BeanFactoryPostProcessor；
			postProcessor.postProcessBeanFactory()


6、registerBeanPostProcessors(beanFactory);注册BeanPostProcessor（Bean的后置处理器）【 intercept bean creation】
		不同接口类型的BeanPostProcessor；在Bean创建前后的执行时机是不一样的
		BeanPostProcessor、
		DestructionAwareBeanPostProcessor、
		InstantiationAwareBeanPostProcessor、
		SmartInstantiationAwareBeanPostProcessor、
		MergedBeanDefinitionPostProcessor【internalPostProcessors】、
		
		1）、189行：获取所有的 BeanPostProcessor;后置处理器都默认可以通过PriorityOrdered、Ordered接口来执行优先级
		2）、204行：先注册PriorityOrdered优先级接口的BeanPostProcessor；
			把每一个BeanPostProcessor；添加到BeanFactory中
			beanFactory.addBeanPostProcessor(postProcessor);
		3）、224行：再注册Ordered接口的
		4）、236行：最后注册没有实现任何优先级接口的
		5）、最终注册MergedBeanDefinitionPostProcessor；
		6）、注册一个ApplicationListenerDetector；来在Bean创建完成后检查是否是ApplicationListener，如果是
			applicationContext.addApplicationListener((ApplicationListener<?>) bean);


7、initMessageSource();初始化MessageSource组件（做国际化功能；消息绑定，消息解析）；
		1）、718行：获取BeanFactory
		2）、719行：看容器中是否有id为messageSource的，类型是MessageSource的组件
			如果有赋值给messageSource，如果没有自己创建一个DelegatingMessageSource；
				MessageSource：取出国际化配置文件中的某个key的值；能按照区域信息获取；
		3）、739行：把创建好的MessageSource注册在容器中，以后获取国际化配置文件的值的时候，可以自动注入MessageSource；
			beanFactory.registerSingleton(MESSAGE_SOURCE_BEAN_NAME, this.messageSource);	
			MessageSource.getMessage(String code, Object[] args, String defaultMessage, Locale locale);以后可通过getMessage获取


8、initApplicationEventMulticaster();初始化事件派发器；
		1）、753行：获取BeanFactory
		2）、754行：从BeanFactory中获取applicationEventMulticaster的ApplicationEventMulticaster；
		3）、762行：如果上一步没有配置；创建一个SimpleApplicationEventMulticaster
		4）、763行：将创建的ApplicationEventMulticaster添加到BeanFactory中，以后其他组件直接自动注入


9、onRefresh();留给子容器（子类）
		1、子类重写这个方法，在容器刷新的时候可以自定义逻辑；


10、registerListeners();给容器中将所有项目里面的ApplicationListener注册进来；
		1、822行：从容器中拿到所有的ApplicationListener
		2、824行：将每个监听器添加到事件派发器中；
			getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
		3、832行：派发之前步骤产生的事件；


11、finishBeanFactoryInitialization(beanFactory);初始化所有剩下的单实例bean；
	1、867行：beanFactory.preInstantiateSingletons();初始化后剩下的单实例bean，跟进
		1）、734行：获取容器中的所有Bean，依次进行初始化和创建对象
		2）、738行：获取Bean的定义信息；RootBeanDefinition
		3）、739行：Bean不是抽象的，是单实例的，是懒加载；
			1）、740行：判断是否是FactoryBean；是否是实现FactoryBean接口的Bean；
			2）、760行：不是工厂Bean。利用getBean(beanName);创建对象
				0、199行：getBean(beanName)； ioc.getBean();
				1、doGetBean(name, null, null, false);
				2、246行： getSingleton(beanName)先获取缓存中保存的单实例Bean《跟进去其实就是从MAP中拿》。如果能获取到说明这个Bean之前被创建过（所有创建过的单实例Bean都会被缓存起来）
					从private final Map<String, Object> singletonObjects = new ConcurrentHashMap<String, Object>(256);获取的
				3、缓存中获取不到，开始Bean的创建对象流程；
				4、287行：标记当前bean已经被创建（防止多线程同时创建，使用synchronized）
				5、291行:获取Bean的定义信息；
				6、295行：getDependsOn()，bean.xml里创建person时，加depend-on="jeep,moon"是先把jeep和moon创建出来
				         【获取当前Bean依赖的其他Bean;如果有按照getBean()把依赖的Bean先创建出来；】
				7、启动单实例Bean的创建流程；
					1）、462行：createBean(beanName, mbd, args);
					2）、490行：Object bean = resolveBeforeInstantiation(beanName, mbdToUse);让BeanPostProcessor先拦截返回代理对象；
						【InstantiationAwareBeanPostProcessor】：提前执行；
						先触发：postProcessBeforeInstantiation()；
						如果有返回值：触发postProcessAfterInitialization()；
					3）、如果前面的InstantiationAwareBeanPostProcessor没有返回代理对象；调用4）
					4）、501行：Object beanInstance = doCreateBean(beanName, mbdToUse, args);创建Bean
						 1）、541行：【创建Bean实例】；createBeanInstance(beanName, mbd, args);
						 	利用工厂方法或者对象的构造器创建出Bean实例；
						 2）、applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
						 	调用MergedBeanDefinitionPostProcessor的postProcessMergedBeanDefinition(mbd, beanType, beanName);
						 3）、578行：【Bean属性赋值】populateBean(beanName, mbd, instanceWrapper);
						 	赋值之前：
						 	1）、拿到InstantiationAwareBeanPostProcessor后置处理器；
						 		1305行：postProcessAfterInstantiation()；
						 	2）、拿到InstantiationAwareBeanPostProcessor后置处理器；
						 		1348行：postProcessPropertyValues()；
						 	=====赋值之前：===
						 	3）、应用Bean属性的值；为属性利用setter方法等进行赋值；
						 		applyPropertyValues(beanName, mbd, bw, pvs);
						 4）、【Bean初始化】initializeBean(beanName, exposedObject, mbd);
						 	1）、1693行：【执行Aware接口方法】invokeAwareMethods(beanName, bean);执行xxxAware接口的方法
						 		BeanNameAware\BeanClassLoaderAware\BeanFactoryAware
						 	2）、1698行：【执行后置处理器初始化之前】applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
						 		BeanPostProcessor.postProcessBeforeInitialization（）;
						 	3）、1702行：【执行初始化方法】invokeInitMethods(beanName, wrappedBean, mbd);
						 		1）、是否是InitializingBean接口的实现；执行接口规定的初始化；
						 		2）、是否自定义初始化方法；
						 	4）、1710行：【执行后置处理器初始化之后】applyBeanPostProcessorsAfterInitialization
						 		BeanPostProcessor.postProcessAfterInitialization()；
						
					5）、将创建的Bean添加到缓存中singletonObjects；sharedInstance = getSingleton(beanName, ()跟进去
					     254行：addSingleton（），放到MAP中
				ioc容器就是这些Map；很多的Map里面保存了单实例Bean，环境信息。。。。；
		所有Bean都利用getBean创建完成以后；
			检查所有的Bean是否是SmartInitializingSingleton接口的；如果是；就执行afterSingletonsInstantiated()；


12、finishRefresh();完成BeanFactory的初始化创建工作；IOC容器就创建完成；
		1）、882行：initLifecycleProcessor();初始化和生命周期有关的后置处理器；LifecycleProcessor
			默认从容器中找是否有lifecycleProcessor的组件【LifecycleProcessor】；如果没有new DefaultLifecycleProcessor();
			加入到容器；
			
			自己也可以尝试写一个LifecycleProcessor的实现类，可以在BeanFactory
				void onRefresh();
				void onClose();	
		2）、	885行：getLifecycleProcessor().onRefresh();
			拿到前面定义的生命周期处理器（BeanFactory）；回调onRefresh()；
		3）、888行：publishEvent(new ContextRefreshedEvent(this));发布容器刷新完成事件；
		4）、891行：liveBeansView.registerApplicationContext(this);
	
	======总结===========
	1）、Spring容器在启动的时候，先会保存所有注册进来的Bean的定义信息；
		1）、xml注册bean；<bean>
		2）、注解注册Bean；@Service、@Component、@Bean、xxx
	2）、Spring容器会合适的时机创建这些Bean
		1）、用到这个bean的时候；利用getBean创建bean；创建好以后保存在容器中；
		2）、统一创建剩下所有的bean的时候；finishBeanFactoryInitialization()；
	3）、后置处理器；BeanPostProcessor
		1）、每一个bean创建完成，都会使用各种后置处理器进行处理；来增强bean的功能；
			AutowiredAnnotationBeanPostProcessor:处理自动注入
			AnnotationAwareAspectJAutoProxyCreator:来做AOP功能；
			xxx....
			增强的功能注解：
			AsyncAnnotationBeanPostProcessor
			....
	4）、事件驱动模型；
		ApplicationListener；事件监听；
		ApplicationEventMulticaster；事件派发：
