**针对注解型容器AnnotationConfigApplicationContext：**
* this()
  - 在父类GenericApplicationContext构造方法中：this.beanFactory = new DefaultListableBeanFactory()
  - this.reader = new AnnotatedBeanDefinitionReader(this)
    * AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry)
    * ——会注册注解处理器
    * ——internalConfigurationAnnotationProcessor对应ConfigurationClassPostProcessor（该处理器就是专门用来处理@Configuration注解）
    * ——internalAutowiredAnnotationProcessor对应AutowiredAnnotationBeanPostProcessor
    * ——internalCommonAnnotationProcessor对应CommonAnnotationBeanPostProcessor
  - this.scanner = new ClassPathBeanDefinitionScanner(this)
* register(componentClasses)，将componentClasses注册到Spring容器中
* refresh()


**针对配置型容器ClassPathXmlApplicationContext：**
* setConfigLocations(configLocations)
* refresh()


**IOC流程——Spring容器的refresh()【创建刷新】：**
* 1.prepareRefresh()刷新前的预处理;
  - 1）initPropertySources()初始化一些属性设置;子类自定义个性化的属性设置方法；
  - 2）getEnvironment().validateRequiredProperties();检验属性的合法等
  - 3）earlyApplicationEvents= new LinkedHashSet<ApplicationEvent>();保存容器中的一些早期的事件；
* 2.obtainFreshBeanFactory();获取BeanFactory；
  - 1）refreshBeanFactory();刷新【创建】BeanFactory；
	创建了一个this.beanFactory = new DefaultListableBeanFactory();
	设置id；
  - 2）getBeanFactory();返回刚才GenericApplicationContext创建的BeanFactory对象；
  - 3）将创建的BeanFactory【DefaultListableBeanFactory】返回；
  - 4）（ClassPathXmlApplicationContext）loadBeanDefinitions(beanFactory)，解析配置Bean的XML,并把Bean定义注册到BeanFatory。调用至BeanDefinitionParserDelegate.parseCustomElement()。
    * 对于 &#60;context:component-scan/&#62; 标签调用ComponentScanBeanDefinitionParser.parse()
      - 当我们在 XML 里面配置 &#60;context:component-scan/&#62; 标签后，Spring 框架会根据标签内指定的包路径下查找指定过滤条件的 Bean，并可以根据标签内配置的 BeanNameGenerator 生成 Bean 的名称，根据标签内配置的 scope-proxy 属性配置 Bean 被代理的方式，根据子标签 &#60;context:include-filter/&#62;,&#60;context:exclude-filter/&#62; 配置自定义过滤条件。
      - this.beanDefinitionMap.put(beanName, beanDefinition)
      - registerComponents(parserContext.getReaderContext(), beanDefinitions, element)，跟AnnotationConfigApplicationContext注入的注解处理器一样
    * 对于aop:config标签调用ConfigBeanDefinitionParser.parse()
* 3.prepareBeanFactory(beanFactory);BeanFactory的预准备工作（以上创建了beanFactory,现在对BeanFactory对象进行一些设置属性）；
  - 1）设置BeanFactory的类加载器、支持表达式解析器...
  - 2）添加部分BeanPostProcessor【ApplicationContextAwareProcessor】
  - 3）设置忽略的自动装配的接口EnvironmentAware、EmbeddedValueResolverAware、xxx；
  - 4）注册可以解析的自动装配；我们能直接在任何组件中自动注入：
	BeanFactory、ResourceLoader、ApplicationEventPublisher、ApplicationContext
  - 5）添加BeanPostProcessor【ApplicationListenerDetector】
  - 6）添加编译时的AspectJ；
  - 7）给BeanFactory中注册一些能用的组件；
	  environment【ConfigurableEnvironment】、
	  systemProperties【Map<String, Object>】、
	  systemEnvironment【Map<String, Object>】
* 4.postProcessBeanFactory(beanFactory);BeanFactory准备工作完成后进行的后置处理工作；
  - 子类通过重写这个方法来在BeanFactory创建并预准备完成以后做进一步的设置
* 5.invokeBeanFactoryPostProcessors(beanFactory);执行BeanFactoryPostProcessor的方法；
	BeanFactoryPostProcessor：BeanFactory的后置处理器。在BeanFactory标准初始化之后执行的；
	两个接口：BeanFactoryPostProcessor、BeanDefinitionRegistryPostProcessor
  - 先执行BeanDefinitionRegistryPostProcessor
    * BeanDefinitionRegistryPostProcessor的处理，BeanDefinitionRegistryPostProcessor 接口继承自 BeanFactoryPostProcessor，它新添加了一个接口，用来在BeanFactoryPostProcessor 实现类中 postProcessBeanFactory 方法执行前再注册一些 Bean 到 beanFactory 中。
    * AnnotationConfigApplicationContext，会在 refresh 阶段前注册一个ConfigurationClassPostProcessor，它实现了 BeanDefinitionRegistryPostProcessor、PriorityOrdered 两个接口
      - 因为实现了BeanDefinitionRegistryPostProcessor，所以在前面会执行执行 postProcessBeanDefinitionRegistry 方法。这个方法内部作用是使用ConfigurationClassParser 解析所有标注有 @Configuration 注解的类，并解析该类里面所有标注 @Bean 的方法和标注 @Import 的bean，并注入这些解析的 Bean 到 Spring上下文容器里面。
    * 1）获取所有的BeanDefinitionRegistryPostProcessor；
    * 2）看先执行实现了PriorityOrdered优先级接口的BeanDefinitionRegistryPostProcessor、
	    postProcessor.postProcessBeanDefinitionRegistry(registry)
    * 3）在执行实现了Ordered顺序接口的BeanDefinitionRegistryPostProcessor；
		postProcessor.postProcessBeanDefinitionRegistry(registry)
    * 4）最后执行没有实现任何优先级或者是顺序接口的BeanDefinitionRegistryPostProcessors；
		postProcessor.postProcessBeanDefinitionRegistry(registry)	
  - 再执行BeanFactoryPostProcessor的方法
    * （已废弃）PropertyPlaceholderConfigurer实现了PriorityOrdered接口，执行 postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) 方法对 Bean 定义的属性值中 ${...} 进行替换
    * 1）获取所有的BeanFactoryPostProcessor
    * 2）1看先执行实现了PriorityOrdered优先级接口的BeanFactoryPostProcessor、
		postProcessor.postProcessBeanFactory()
    * 3）在执行实现了Ordered顺序接口的BeanFactoryPostProcessor；
		postProcessor.postProcessBeanFactory()
    * 4）最后执行没有实现任何优先级或者是顺序接口的BeanFactoryPostProcessor；
		postProcessor.postProcessBeanFactory()
* 6.registerBeanPostProcessors(beanFactory);注册BeanPostProcessor（Bean的后置处理器）【 intercept bean creation】本阶段就是把用户注册的实现了BeanPostProcessor接口的 Bean 进行收集,然后放入到 BeanFactory 的 beanPostProcessors 属性里面,待后面使用。
	不同接口类型的BeanPostProcessor；在Bean创建前后的执行时机是不一样的
	BeanPostProcessor、
	DestructionAwareBeanPostProcessor、
	InstantiationAwareBeanPostProcessor、
	SmartInstantiationAwareBeanPostProcessor、
	MergedBeanDefinitionPostProcessor【internalPostProcessors】	
  - 1）获取所有的 BeanPostProcessor;后置处理器都默认可以通过PriorityOrdered、Ordered接口来执行优先级
  - 2）先注册PriorityOrdered优先级接口的BeanPostProcessor；
	把每一个BeanPostProcessor；添加到BeanFactory中
	beanFactory.addBeanPostProcessor(postProcessor);
  - 3）再注册Ordered接口的
  - 4）最后注册没有实现任何优先级接口的
  - 5）最终注册MergedBeanDefinitionPostProcessor；
  - 6）注册一个ApplicationListenerDetector；来在Bean创建完成后检查是否是ApplicationListener，如果是applicationContext.addApplicationListener((ApplicationListener<?>) bean);
* 7.initMessageSource();初始化MessageSource组件（做国际化功能；消息绑定，消息解析）；
  - 1）获取BeanFactory
  - 2）看容器中是否有id为messageSource的，类型是MessageSource的组件
	如果有赋值给messageSource，如果没有自己创建一个DelegatingMessageSource；
	MessageSource：取出国际化配置文件中的某个key的值；能按照区域信息获取；
  - 3）把创建好的MessageSource注册在容器中，以后获取国际化配置文件的值的时候，可以自动注入MessageSource；
	beanFactory.registerSingleton(MESSAGE_SOURCE_BEAN_NAME, this.messageSource);	
	MessageSource.getMessage(String code, Object[] args, String defaultMessage, Locale locale);以后可通过getMessage获取
* 8.initApplicationEventMulticaster();初始化事件派发器；
  - 1）获取BeanFactory
  - 2）从BeanFactory中获取applicationEventMulticaster的ApplicationEventMulticaster；
  - 3）如果上一步没有配置；创建一个SimpleApplicationEventMulticaster
  - 4）将创建的ApplicationEventMulticaster添加到BeanFactory中，以后其他组件直接自动注入
* 9.onRefresh();留给子容器（子类）
  - 1）子类重写这个方法，在容器刷新的时候可以自定义逻辑；
* 10.registerListeners();给容器中将所有项目里面的ApplicationListener注册进来；
  - 1）从容器中拿到所有的ApplicationListener
  - 2）将每个监听器添加到事件派发器中；
	getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
  - 3）派发之前步骤产生的事件；
* 11.finishBeanFactoryInitialization(beanFactory);初始化所有剩下的单实例bean；
  beanFactory.preInstantiateSingletons();初始化后剩下的单实例bean
  - 1）获取容器中的所有Bean，依次进行初始化和创建对象
  - 2）获取Bean的定义信息；RootBeanDefinition
  - 3）Bean不是抽象的，是单实例的，是懒加载；
  - 3-1）判断是否是FactoryBean；是否是实现FactoryBean接口的Bean；
  - 3-2）不是工厂Bean。利用getBean(beanName);创建对象
    * 0、getBean(beanName)； ioc.getBean();
    * 1、doGetBean(name, null, null, false);
    * 2、getSingleton(beanName)先获取缓存中保存的单实例Bean《跟进去其实就是从MAP中拿》。如果能获取到说明这个Bean之前被创建过（所有创建过的单实例Bean都会被缓存起来）
	从private final Map<String, Object> singletonObjects = new ConcurrentHashMap<String, Object>(256);获取的
    * 3、缓存中获取不到，开始Bean的创建对象流程；
    * 4、标记当前bean已经被创建（防止多线程同时创建，使用synchronized）
    * 5、获取Bean的定义信息；
    * 6、getDependsOn()，bean.xml里创建person时，加depend-on="jeep,moon"是先把jeep和moon创建出来，获取当前Bean依赖的其他Bean;如果有按照getBean()把依赖的Bean先创建出来；】
    * 7、启动单实例Bean的创建流程；
      - 1）createBean(beanName, mbd, args);
      - 2）Object bean = resolveBeforeInstantiation(beanName, mbdToUse);让BeanPostProcessor先拦截返回代理对象；
		【InstantiationAwareBeanPostProcessor】：提前执行；
		先触发：postProcessBeforeInstantiation()；
		如果有返回值：触发postProcessAfterInitialization()；
      - 3）如果前面的InstantiationAwareBeanPostProcessor没有返回代理对象；调用4）
      - 4）Object beanInstance = doCreateBean(beanName, mbdToUse, args);创建Bean
        * step.1）【创建Bean实例】；createBeanInstance(beanName, mbd, args);
		利用工厂方法或者对象的构造器创建出Bean实例；
        * step.2）applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
		调用MergedBeanDefinitionPostProcessor的postProcessMergedBeanDefinition(mbd, beanType, beanName);
        * step.3）【Bean属性赋值】populateBean(beanName, mbd, instanceWrapper);
		赋值之前：
          - 1）拿到InstantiationAwareBeanPostProcessor后置处理器；
		  postProcessAfterInstantiation()；
          - 2）拿到InstantiationAwareBeanPostProcessor后置处理器；
		  postProcessPropertyValues()；
          - 3）应用Bean属性的值；为属性利用setter方法等进行赋值；
		  applyPropertyValues(beanName, mbd, bw, pvs);
        * step.4）、【Bean初始化】initializeBean(beanName, exposedObject, mbd);
          - 1）【执行Aware接口方法】invokeAwareMethods(beanName, bean);执行xxxAware接口的方法
			BeanNameAware\BeanClassLoaderAware\BeanFactoryAware
          - 2）【执行后置处理器初始化之前】applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
			BeanPostProcessor.postProcessBeforeInitialization（）;
          - 3）【执行初始化方法】invokeInitMethods(beanName, wrappedBean, mbd);
			3-1）是否是InitializingBean接口的实现；执行接口规定的初始化；
			3-2）是否自定义初始化方法；
          - 4）【执行后置处理器初始化之后】applyBeanPostProcessorsAfterInitialization
			BeanPostProcessor.postProcessAfterInitialization()；			
    * 8、将创建的Bean添加到缓存中singletonObjects；sharedInstance = getSingleton(beanName, lambda)
	addSingleton(beanName, singletonObject)，放到MAP中
	ioc容器就是这些Map；很多的Map里面保存了单实例Bean，环境信息。。。。；
  - 4）所有Bean都利用getBean创建完成以后；
   检查所有的Bean是否是SmartInitializingSingleton接口的；如果是；就执行afterSingletonsInstantiated()；
* 12.finishRefresh();完成BeanFactory的初始化创建工作；IOC容器就创建完成；
  - 1）initLifecycleProcessor();初始化和生命周期有关的后置处理器；LifecycleProcessor
	默认从容器中找是否有lifecycleProcessor的组件【LifecycleProcessor】；如果没有new DefaultLifecycleProcessor();
	加入到容器；
	自己也可以尝试写一个LifecycleProcessor的实现类，可以在BeanFactory
	void onRefresh();
	void onClose();	
  - 2）getLifecycleProcessor().onRefresh();
	拿到前面定义的生命周期处理器（BeanFactory）；回调onRefresh()；
  - 3）publishEvent(new ContextRefreshedEvent(this));发布容器刷新完成事件；
  - 4）liveBeansView.registerApplicationContext(this);
	

**总结：**
* 1.Spring容器在启动的时候，先会保存所有注册进来的Bean的定义信息；
  - 1）xml注册bean；<bean>
  - 2）注解注册Bean；@Service、@Component、@Bean、xxx
* 2.Spring容器会合适的时机创建这些Bean
  - 1）用到这个bean的时候；利用getBean创建bean；创建好以后保存在容器中；
  - 2）统一创建剩下所有的bean的时候；finishBeanFactoryInitialization()；
* 3.后置处理器；BeanPostProcessor
  - 1）每一个bean创建完成，都会使用各种后置处理器进行处理；来增强bean的功能；
	AutowiredAnnotationBeanPostProcessor:处理自动注入
	AnnotationAwareAspectJAutoProxyCreator:来做AOP功能；
	增强的功能注解：AsyncAnnotationBeanPostProcessor
* 4.事件驱动模型
  ApplicationListener；事件监听；
  ApplicationEventMulticaster；事件派发：


**容器关闭close()方法，调用AbstractApplicationContext.doClose()：**
* publishEvent(new ContextClosedEvent(this))
  - 给所有注册了 ContextClosedEvent 事件的 ApplicationListener 发送通知。当应用程序上下文关闭时候，会给所有注册了 ContextClosedEvent 事件的 ApplicationListener 发送通知，所以用户可以实现 ApplicationListener 并关注 ContextClosedEvent 事件（如下代码），同时把该 Bean 注入到 Spring 容器，在应用程序上下文关闭时候做一些工作。
* destroyBeans()
  - 销毁应用程序上下文管理的 BeanFactory 里面的所有 Beans
  - 调用DefaultSingletonBeanRegistry.destroySingletons()
    * bean.destroy()
      - DisposableBeanAdapter.invokeCustomDestroyMethod()，调用至@Bean(initMethod = "init", destroyMethod = "destroy")
      - DisposableBeanAdapter.destroy()，调用((DisposableBean) this.bean).destroy()
* closeBeanFactory()
* onClose()
  - SpringBoot 的 Web 应用程序上下文 EmbeddedWebApplicationContext 实现的 onClose方法，在关闭应用程序上下文后关闭内嵌的 Web 容器




















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

# 2.1 @Component

@Component 对应<context:component-scan base-package="com.jiaduo.test" />

* step1.解析配置文件
  - 最后调用至ComponentScanBeanDefinitionParser.parse()
    * 当我们在 XML 里面配置 <context:component-scan/> 标签后，Spring 框架会根据标签内指定的包路径下查找指定过滤条件的 Bean，并可以根据标签内配置的 BeanNameGenerator 生成 Bean 的名称，根据标签内配置的 scope-proxy 属性配置 Bean 被代理的方式，根据子标签 <context:include-filter/>,<context:exclude-filter/> 配置自定义过滤条件。
    * 注册注解处理器
      - internalConfigurationAnnotationProcessor对应ConfigurationClassPostProcessor
      - internalAutowiredAnnotationProcessor对应AutowiredAnnotationBeanPostProcessor
      - internalCommonAnnotationProcessor对应CommonAnnotationBeanPostProcessor
* step2.ConfigurationClassPostProcessor会处理@Configuration注解
  AutowiredAnnotationBeanPostProcessor处理@Autowired注解
  CommonAnnotationBeanPostProcessor处理@Resource注解



## 2.2 注解 @Configuration、@ComponentScan、@Import、@PropertySource、@Bean工作原理

AnnotationConfigApplicationContext，会在 refresh 阶段前注册一个ConfigurationClassPostProcessor，它实现了 BeanDefinitionRegistryPostProcessor、PriorityOrdered 两个接口.

* 因为实现了BeanDefinitionRegistryPostProcessor，所以在前面会执行执行 postProcessBeanDefinitionRegistry 方法。这个方法内部作用是使用ConfigurationClassParser 解析所有标注有 @Configuration 注解的类，并解析该类里面所有标注 @Bean 的方法和标注 @Import 的bean，并注入这些解析的 Bean 到 Spring上下文容器里面。
* 因为实现了PriorityOrdered接口，所以该类有 getOrder 方法返回该类的优先级，这里实现为O rdered.LOWEST_PRECEDENCE，也就是优先级最低。


## 2.3 FactoryBean

以SqlSessionFactoryBean为例：
* 实现了InitializingBean
  - 调用((InitializingBean) bean).afterPropertiesSet()调用SqlSessionFactoryBean.afterPropertiesSet()
  - this.sqlSessionFactory = buildSqlSessionFactory()
* 实现了FactoryBean
  - getObjectForBeanInstance(sharedInstance, name, beanName, mbd)
    * 首先判断beanName是否是以&开头，如果是FactoryBean，直接返回FactoryBean自身
    * 不是FactoryBean类型的bean，直接返回
    * getObjectFromFactoryBean()
      - 最后调用SqlSessionFactoryBean.getObject()返回之前创建的this.sqlSessionFactory

# 3.bean生命周期

# 4.赋值及依赖注入

@Autowired、@Value是AutowiredAnnotationBeanPostProcessor进行处理的。

* AutowiredAnnotationBeanPostProcessor实现了BeanFactoryAware接口,在registerBeanPostProcessors(beanFactory)阶段BeanPostProcessor实例化时,调用setBeanFactory()方法
* finishBeanFactoryInitialization(beanFactory)创建bean
  - applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName),MergedBeanDefinitionPostProcessor接口方法调用
    * 这里是先通过 buildAutowiringMetadata 收集当前 Bean 中的注解信息,其中会先查找当前类里面的注解信息,对应在变量上标注 @Autowired 的变量会创建一个 AutowiredFieldElement 实例用来记录注解信息,对应在 set 方法上标注 @Autowired 的方法会创建一个 AutowiredMethodElement 对象来保存注解信息。然后会递归解析当前类的直接父类里面的注解,并把最远父类到当前类里面的注解信息依次存放到InjectionMetadata对象（内部使用集合保存所有方法和属性上的注解元素对象）,然后缓存起来以便后面使用,这里的缓存实际是个并发 map
  - populateBean(beanName, mbd, instanceWrapper)调用InstantiationAwareBeanPostProcessor接口方法
    * postProcessProperties()方法属性注入
      - findAutowiringMetadata(beanName, bean.getClass(), pvs)从缓存直接返回
      - metadata.inject(bean, beanName, pvs)依赖注入
        * ReflectionUtils.makeAccessible(field);
        * field.set(bean, value);
* 总结：
  - @Autowired 的使用简化了我们的开发，其原理是使用 AutowiredAnnotationBeanPostProcessor 类来实现，该类实现了 Spring 框架的一些扩展接口，通过实现 BeanFactoryAware 接口使其内部持有了 BeanFactory（可轻松的获取需要依赖的的 Bean）；通过实现 MergedBeanDefinitionPostProcessor 扩展接口，在 BeanFactory 里面的每个 Bean 实例化前获取到每个 Bean 里面的 @Autowired 信息并缓存下来；通过实现 InstantiationAwareBeanPostProcessor扩展接口在 BeanFactory 里面的每个 Bean 实例后从缓存取出对应的注解信息，获取依赖对象，并通过反射设置到 Bean 属性里面。
  - 实现了 BeanFactoryAware 接口，让 AutowiredAnnotationBeanPostProcessor 获取到当前 Spring 应用程序上下文管理的 BeanFactory。例如：这个在后面依赖注入时，需要使用beanFatory解析依赖的bean。
  - 实现了 MergedBeanDefinitionPostProcessor 接口，postProcessMergedBeanDefinition()通过 findAutowiringMetadata 方法找到当前 Bean 中标注 @Autowired 注解的属性变量和方法，然后递归解析当前类父类注解信息，最后将这些信息放到缓存Map中
  - 继承了 InstantiationAwareBeanPostProcessorAdapter，也即实现了接口InstantiationAwareBeanPostProcessor。调用postProcessProperties()方法，首先获取当前 Bean 里面的依赖元数据信息，由于已经收集到了缓存，所以这里是直接从缓存获取的；逐个调用 InjectionMetadata 内部集合里面存放的属性和方法注解对象的 inject 方法，通过反射设置依赖的属性值和反射调用 set 方法设置属性值。
    * 解析value的时候，如果此时bean还没有注入beanFactory，需要调用beanFactory.getBean(beanName)将其注入。这也是为什么要传入BeanFactory的原因。

# 5.Aware获取底层组件

# 6.AOP



## 6.1 xml配置文件方式

* 1.配置解析：obtainFreshBeanFactory()调用refreshBeanFactory()，对于ClassPathXmlApplicationContext 来说，调用AbstractRefreshableApplicationContext
  - createBeanFactory()，创建一个DefaultListableBeanFactory
  - loadBeanDefinitions(beanFactory)，调用至调用至BeanDefinitionParserDelegate.parseCustomElement()，对于aop:config标签调用ConfigBeanDefinitionParser.parse()
    * configureAutoProxyCreator(parserContext, element)，AopNamespaceUtils.registerAspectJAutoProxyCreatorIfNecessary(parserContext, element)
      - AopConfigUtils.registerAspectJAutoProxyCreatorIfNecessary，注册名称为internalAutoProxyCreator的AspectJAwareAdvisorAutoProxyCreator到 Spring IOC 容器，该类的作用是自动创建代理类
      - useClassProxyingIfNecessary()，解析 aop:config 标签里面的 proxy-target-class 和 expose-proxy 属性值
        * AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry)，针对 proxy-target-class 属性（默认 false），如果设置为 true，则会强制使用 CGLIB 生成代理类
        * AopConfigUtils.forceAutoProxyCreatorToExposeProxy(registry)，对于 expose-proxy 属性（默认 false）如果设置为 true，则对一个类里面的嵌套方法调用的方法也进行代理
    * parsePointcut(elt, parserContext)，对 aop:config 标签里面的 aop:pointcut 元素进行解析，每个 pointcut 元素会创建一个 AspectJExpressionPointcut 的 bean 定义，并注册到Spring IOC
    * parseAdvisor(elt, parserContext)，对 aop:config 标签里面的 aop:advisor 元素进行解析，每个 advisor 元素会创建一个 DefaultBeanFactoryPointcutAdvisor 的 bean 定义，并注册到 Spring IOC
    * parseAspect(elt, parserContext)，对 aop:config 标签里面的 aop:aspect 元素进行解析，会解析 aop:aspect 元素内的子元素，每个子元素会对应创建一个 AspectJPointcutAdvisor 的 bean 定义，并注册到 Spring IOC
* 2.registerBeanPostProcessors(beanFactory)
  - 上面注册的AspectJAwareAdvisorAutoProxyCreator类实现了BeanFactoryAware接口,BeanPostProcessor实例化,并调用setBeanFactory()方法
  - BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class)，调用了实现BeanFactoryAware接口的setBeanFactory()方法
* 3.finishBeanFactoryInitialization(beanFactory)，preInstantiateSingletons()方法中循环getBean(beanName)
  - 在initializeBean(beanName, exposedObject, mbd)中，调用后置处理器调用的是AspectJAwareAdvisorAutoProxyCreator 类实现的BeanPostProcessor接口的方法postProcessAfterInitialization()，看是否需要进行代理wrapIfNecessary(bean, beanName, cacheKey)：
    * getAdvicesAndAdvisorsForBean()，调用至AbstractAdvisorAutoProxyCreator,在 Spring 容器中查找可以对当前 bean 进行增强的通知 bean，找到spring容器中所有Advisor类型的Bean，哪些增强advice可以应用到当前 bean 上，这个是通过切点表达式来匹配的。
    * createProxy()，最终createAopProxy().getProxy(classLoader)，决定使用JDK动态代理还是Cglib代理
      - cglib动态代理增强方法在callback里面，而关于AOP的callback是DynamicAdvisedInterceptor
      - jdk动态代理：JdkDynamicAopProxy 类实现了 InvocationHandler 接口


## 6.2 注解方式

* @EnableAspectJAutoProxy引入了@Import(AspectJAutoProxyRegistrar.class)，AspectJAutoProxyRegistrar实现了 ImportBeanDefinitionRegistrar
  - 执行时机invokeBeanFactoryPostProcessors(beanFactory)调用ConfigurationClassPostProcessor.postProcessBeanDefinitionRegistry(registry)
    * loadBeanDefinitionsFromRegistrars(configClass.getImportBeanDefinitionRegistrars())，调用至AspectJAutoProxyRegistrar.registerBeanDefinitions()
      - 调用AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry)，注册名字为internalAutoProxyCreator的AnnotationAwareAspectJAutoProxyCreator
* AnnotationAwareAspectJAutoProxyCreator，与InfrastructureAdvisorAutoProxyCreator继承关系类似，都继承了AbstractAdvisorAutoProxyCreator，也即最终继承自BeanFactoryAware、InstantiationAwareBeanPostProcessor和BeanPostProcessor
  - this.aspectJAdvisorFactory = new ReflectiveAspectJAdvisorFactory(beanFactory)
  - this.aspectJAdvisorsBuilder = new BeanFactoryAspectJAdvisorsBuilderAdapter(beanFactory, this.aspectJAdvisorFactory)
  - 同理会在拦截对象bean初始化后调用postProcessAfterInitialization()进行代理对象的创建
    * specificInterceptors = getAdvicesAndAdvisorsForBean()
    * proxy = createProxy()
* 整体执行流程
  - AnnotationConfigApplicationContext acc = new AnnotationConfigApplicationContext(MainConfig.class)
    * invokeBeanFactoryPostProcessors(beanFactory)，ConfigurationClassPostProcessor解析配置类,注册配置类中定义的所有BeanDefinition
    * registerBeanPostProcessors(beanFactory)，实例化CommonAnnotationBeanPostProcessor、AutowiredAnnotationBeanPostProcessor和AnnotationAwareAspectJAutoProxyCreator
    * finishBeanFactoryInitialization(beanFactory)
      - applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName)，MergedBeanDefinitionPostProcessor类型的Processor进行处理
      - populateBean(beanName, mbd, instanceWrapper)
        * CommonAnnotationBeanPostProcessor，处理WebServiceRef、EJB、Resource注解，并进行诸如
        * AutowiredAnnotationBeanPostProcessor处理@Autowired注解，并进行注入
      - initializeBean(beanName, exposedObject, mbd)
        * AnnotationAwareAspectJAutoProxyCreator注解，必要时创建动态代理wrapIfNecessary(bean, beanName, cacheKey)
          - specificInterceptors = getAdvicesAndAdvisorsForBean()，返回的顺序是logException、logReturn、logEnd、Around、logStart
          - createProxy()
  - Calculator calculator = acc.getBean(Calculator.class)
  - calculator.div(1, 1)
    * DynamicAdvisedInterceptor.intercept()
    * ——chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass)
    * ——retVal = new CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy).proceed()，最终调用ReflectiveMethodInvocation.proceed()
      - interceptorOrInterceptionAdvice = this.interceptorsAndDynamicMethodMatchers.get(0)进而调用：((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this)
      - ——mi.proceed()，因为上面讲this传入了invoke，这里又是调用this.proceed()方法，又会进入到ReflectiveMethodInvocation.proceed()
        * interceptorOrInterceptionAdvice = this.interceptorsAndDynamicMethodMatchers.get(1)进而调用：((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this)，进入到logException（AspectJAfterThrowingAdvice）的invoke()方法
        * ——mi.proceed()
          - 进入logReturn(AfterReturningAdviceInterceptor).invoke()方法
          - ——retVal = mi.proceed()
            * 进入到logEnd(AspectJAfterAdvice)的invoke方法
            * ——mi.proceed()
              - 进入到around（AspectJAroundAdvice）的invoke方法，调用invokeAdviceMethod(pjp, jpm, null, null)，最终调用到自定义的Around方法
              - ——真正方法调用前的操作
              - ——proceedingJoinPoint.proceed()
                * 最终又调用到了ReflectiveMethodInvocation.proceed()，调用logStart（MethodBeforeAdviceInterceptor）的invoke()
                * ——this.advice.before()
                * ——mi.proceed()
                  - **this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1，直接调用，this.methodProxy.invoke(this.target, this.arguments)最终会调用实际的目标方法invokeJoinpoint()**
              - ——真正方法后的操作
            * ——finally会调用invokeAdviceMethod()
          - ——this.advice.afterReturning()
          - ——return retVal
        * ——invokeAdviceMethod(getJoinPointMatch(), null, ex)



# 7.事务

## 7.1 事务管理之基于AspectJ的XML方式

* obtainFreshBeanFactory()
  - loadBeanDefinitions(beanFactory)
    * 对于tx:advice标签调用TxAdviceBeanDefinitionParser.parse()
      - tx:advice 作用是创建一个 TransactionInterceptor 拦截器，内部维护事务配置信息。
      - AbstractSingleBeanDefinitionParser.parseInternal()
        * TxAdviceBeanDefinitionParser.getBeanClass()，返回TransactionInterceptor
        * doParse(element, parserContext, builder)，通过循环解析 tx:attributes 标签里面的所有 tx:method 标签，每个 tx:method 对应一个 RuleBasedTransactionAttribute 对象，其中 tx:method 标签中除了可以配置事务传播性，还可以配置事务隔离级别，超时时间，是否只读，和回滚策略。
        * builder.getBeanDefinition()，BeanDefinitionBuilder 是一个建造者模式，用来构造一个 bean 定义，这个 bean 定义最终会生成一个 TransactionInterceptor 的实例
      - registerBeanDefinition(holder, parserContext.getRegistry())，注册到Spring容器
*  tx:advice 作用是创建一个 TransactionInterceptor 拦截器，内部维护事务配置信息。这个拦截器在AOP是作为advice增强的。Spring 事务管理通过配置一个 AOP 切面来实现，其中定义了一个切点用来决定对哪些方法进行方法拦截，定义了一个 TransactionInterceptor 通知，来对拦截到的方法进行事务增强。
  - 也即cglib代理对象callback，enhancer.setCallBack(MethodInterceptor)，最后通过调用callback也即MethodInterceptor.intercept()方法来实现增强
  - Aop代理调研DynamicAdviseInterceptor.intercept()方法
    * retVal = new CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy).proceed()
    * 最后调用至TransactionInterceptor.invoke()方法
* TransactionInterceptor.invoke(）调用至invokeWithinTransaction(invocation.getMethod(), targetClass, invocation::proceed) 
  - createTransactionIfNecessary(ptm, txAttr, joinpointIdentification)，标准事务,内部有getTransaction（开启事务） 和commit（提交）/rollback（回滚）事务被调用
    * tm.getTransaction(txAttr)
      - doBegin(transaction, def)
        * obtainDataSource().getConnection()
        * con.setAutoCommit(false)
    * prepareTransactionInfo(tm, txAttr, joinpointIdentification, status)
  - invocation.proceedWithInvocation()，这是一个环绕通知,调用proceedWithInvocation激活拦截器链里面的下一个拦击器
  - 发生异常completeTransactionAfterThrowing()，调用txInfo.getTransactionManager().rollback(txInfo.getTransactionStatus())
  - 事务执行成功commitTransactionAfterReturning(txInfo)，调用txInfo.getTransactionManager().commit(txInfo.getTransactionStatus())



<tx:annotation-driven/>,声明使用注解式事务,可替代配置文件中的tx:advice和aop:config
* <tx:annotation-driven/> 注解的功能也是类似的会创建一个小型切面，不同在于切点是添加了 @Transactional 注解的方法。
* 该标签是 AnnotationDrivenBeanDefinitionParser 进行解析的，其parse()方法如下：
  - aspectj模式
  - proxy模式
    * AopAutoProxyConfigurer.configureAutoProxyCreator(element, parserContext)
      - registerAutoProxyCreatorIfNecessary调用AopConfigUtils.registerAutoProxyCreatorIfNecessary
        * 注册名字为internalAutoProxyCreator的InfrastructureAdvisorAutoProxyCreator。InfrastructureAdvisorAutoProxyCreator 负责收集标注了 @Transactional 注解的方法
        * 其实现了BeanPostProcessor，在bean初始化后会会调用postProcessAfterInitialization()方法，具体实现在其父类AbstractAutoProxyCreator中
          - wrapIfNecessary(bean, beanName, cacheKey)
            * specificInterceptors = getAdvicesAndAdvisorsForBean()
            * ——candidateAdvisors = findCandidateAdvisors()，调用InfrastructureAdvisorAutoProxyCreator.isEligibleAdvisorBean(beanName)，beanName为internalTransactionAdvisor
            * ——findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName)，最后调用到SpringTransactionAnnotationParser. parseTransactionAnnotation ()；解析 @Transactional 上的属性值，比如事务隔离性了，事务传播性了等，然后保存到 TransactionAttribute 返回。
      - 注册AnnotationTransactionAttributeSource
      - 注册TransactionInterceptor(设置transactionManagerBeanName;设置transactionAttributeSource)
      - 注册BeanFactoryTransactionAttributeSourceAdvisor,小型切面,对标注了 @Transactional 注解的方法进行事务功能增强
        * 设置transactionAttributeSource
        * 设置增强adviceBeanName为上面注册的TransactionInterceptor
        * 设置order







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



## 7.2 声明式事务（基于注解AOP的事务管理）

* @EnableTransactionManagement 会@Import(TransactionManagementConfigurationSelector.class)
* TransactionManagementConfigurationSelector
  - PROXY模式，注入如下两个组件
    * AutoProxyRegistrar，调用位置：invokeBeanFactoryPostProcessors()
      - ConfigurationClassPostProcessor.postProcessBeanDefinitionRegistry()，调用至loadBeanDefinitionsFromRegistrars(configClass.getImportBeanDefinitionRegistrars())，最终调用至AutoProxyRegistrar.registerBeanDefinitions()
      - AopConfigUtils.registerAutoProxyCreatorIfNecessary()，也即注册名字为internalAutoProxyCreator的InfrastructureAdvisorAutoProxyCreator；InfrastructureAdvisorAutoProxyCreator利用后置处理器机制在对象创建以后，包装对象，返回一个代理对象（增强器），代理对象执行方法利用拦截器链进行调用
    * ProxyTransactionManagementConfiguration，是个@Configuration(proxyBeanMethods = false)；注册三个bean用来进行拦截增强，在初始化bean之后调用BeanPostProcessor——InfrastructureAdvisorAutoProxyCreator.postProcessAfterInitialization()创建代理wrapIfNecessary()，然后寻找增强类时创建的
      - 注册一个名字为internalTransactionAdvisor的BeanFactoryTransactionAttributeSourceAdvisor
      - 注册AnnotationTransactionAttributeSource
      - 注册TransactionInterceptor




使用方式：

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






