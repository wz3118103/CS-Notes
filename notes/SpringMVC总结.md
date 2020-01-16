
# 0.SpringMVC源码流程

* **初始化**
  - 1）DispatcherServlet构造时执行的静态代码块
    * ClassPathResource resource = new ClassPathResource(DEFAULT_STRATEGIES_PATH, DispatcherServlet.class)，DispatcherServlet.properties路径位于spring-webmvc-xxx.jar/org/springframework/web/servlet下面，与DispatcherServlet路径相同
      - org.springframework.web.servlet.LocaleResolver=org.springframework.web.servlet.i18n.AcceptHeaderLocaleResolver
      - org.springframework.web.servlet.ThemeResolver=org.springframework.web.servlet.theme.FixedThemeResolver
      - org.springframework.web.servlet.HandlerMapping=org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping,\
	org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping,\
	org.springframework.web.servlet.function.support.RouterFunctionMapping
      - org.springframework.web.servlet.HandlerAdapter=org.springframework.web.servlet.mvc.HttpRequestHandlerAdapter,\
	org.springframework.web.servlet.mvc.SimpleControllerHandlerAdapter,\
	org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter,\
	org.springframework.web.servlet.function.support.HandlerFunctionAdapter
      - org.springframework.web.servlet.HandlerExceptionResolver=org.springframework.web.servlet.mvc.method.annotation.ExceptionHandlerExceptionResolver,\
	org.springframework.web.servlet.mvc.annotation.ResponseStatusExceptionResolver,\
	org.springframework.web.servlet.mvc.support.DefaultHandlerExceptionResolver
      - org.springframework.web.servlet.RequestToViewNameTranslator=org.springframework.web.servlet.view.DefaultRequestToViewNameTranslator
      - org.springframework.web.servlet.ViewResolver=org.springframework.web.servlet.view.InternalResourceViewResolver
      - org.springframework.web.servlet.FlashMapManager=org.springframework.web.servlet.support.SessionFlashMapManager
    * defaultStrategies = PropertiesLoaderUtils.loadProperties(resource)
  - 2）GenericServlet.init()调用HttpServletBean.init()
    * PropertyValues pvs = new ServletConfigPropertyValues(getServletConfig(), this.requiredProperties)，获取属性值contextConfigLocation和contextClass
    * bw.setPropertyValues(pvs, true)，通过反射调用setContextConfiguration()和setContextClass()方法设置其值
    * initServletBean()调用FrameworkServlet.initServletBean()接下来所有流程都是属于初始化
    * ——this.webApplicationContext = initWebApplicationContext()
      - 根容器（父容器）rootContext =WebApplicationContextUtils.getWebApplicationContext(getServletContext())
      - 子容器 wac = createWebApplicationContext(rootContext)
      - ——contextClass = getContextClass()
      - ——wac = (ConfigurableWebApplicationContext) BeanUtils.instantiateClass(contextClass)
      - ——wac.setParent(parent)
      - ——configLocation = getContextConfigLocation()
      - ——wac.setConfigLocation(configLocation)
      - ——configureAndRefreshWebApplicationContext(wac)调用wac.refresh()
        * beanFactory = obtainFreshBeanFactory()会调用refreshBeanFactory()，对于ClassPathXmlApplicationContext 来说，调用AbstractRefreshableApplicationContext
        * ——createBeanFactory()创建一个DefaultListableBeanFactory
        * ——loadBeanDefinitions(beanFactory)，解析配置Bean的XML,并把Bean定义注册到BeanFatory，最后调用至BeanDefinitionParserDelegate.parseCustomElement()
          - String namespaceUri = getNamespaceURI(ele)，例如ele:[mvc:annotation-driven:null]，获取到的namespaceUri为http://www.springframework.org/schema/mvc
          - NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri)
          - ——aop -> AopNamespaceHandler
          - ——task -> TaskNamespaceHandler
          - ——mybatis-spring ->NamespaceHandler
          - ——tx -> TxNamespaceHandler
          - ——context -> ContextNamespaceHandler
          - ——mvc -> MvcNamespaceHandler
          - ——因此拿到的是MvcNamespaceHandler，对其进行反射创建和初始化init()
            * registerBeanDefinitionParser("annotation-driven", new AnnotationDrivenBeanDefinitionParser());
            * registerBeanDefinitionParser("default-servlet-handler", new DefaultServletHandlerBeanDefinitionParser());
            * registerBeanDefinitionParser("interceptors", new InterceptorsBeanDefinitionParser());
            * registerBeanDefinitionParser("resources", new ResourcesBeanDefinitionParser());
            * registerBeanDefinitionParser("view-controller", new ViewControllerBeanDefinitionParser());
            * registerBeanDefinitionParser("redirect-view-controller", new ViewControllerBeanDefinitionParser());
            * registerBeanDefinitionParser("status-controller", new ViewControllerBeanDefinitionParser());
            * registerBeanDefinitionParser("view-resolvers", new ViewResolversBeanDefinitionParser());
            * registerBeanDefinitionParser("tiles-configurer", new TilesConfigurerBeanDefinitionParser());
            * registerBeanDefinitionParser("freemarker-configurer", new FreeMarkerConfigurerBeanDefinitionParser());
            * registerBeanDefinitionParser("groovy-configurer", new GroovyMarkupConfigurerBeanDefinitionParser());
            * registerBeanDefinitionParser("script-template-configurer", new ScriptTemplateConfigurerBeanDefinitionParser());
            * registerBeanDefinitionParser("cors", new CorsBeanDefinitionParser());
          - handler.parse(ele, new ParserContext(this.readerContext, this, containingBd))
          - ——BeanDefinitionParser parser = findParserForElement(element, parserContext)
            * BeanDefinitionParser parser = this.parsers.get(localName)，localName：annotation-driven，根据localName找到的parser:AnnotationDrivenBeanDefinitionParser
          - ——parser.parse(element, parserContext)
            * 注册RequestMappingHandlerMapping
              - 间接实现了HandlerMapping接口
              - 实现了ApplicationContextAware
              - IntitalzingBean，方法afterPropertiesSet()调用initHandlerMethods()。
              - ——1.获取容器中所有bean 的name。根据detectHandlerMethodsInAncestorContexts bool变量的值判断是否获取父容器中的bean，默认为false。因此这里只获取Spring MVC容器中的bean，不去查找父容器
              - ——2.判断bean是否含有@Controller或者@RequestMappin注解
              - ——3.对含有注解的bean进行处理，获取handler函数信息。
              - ——3-1.获取bean的类信息
              - ——3-2.遍历函数获取有@RequestMapping注解的函数信息，对自己及所有实现接口类进行遍历
              - ——3-3.遍历映射函数列表，注册handler
            * 注册ConfigurableWebBindingInitializer
            * 注册RequestMappingHandlerAdapter
            * 注册CompositeUriComponentsContributorFactoryBean
            * 注册ConversionServiceExposingInterceptor
            * 注册MappedInterceptor
            * 注册ExceptionHandlerExceptionResolver
            * 注册ResponseStatusExceptionResolver
            * 注册DefaultHandlerExceptionResolver
        * finishBeanFactoryInitialization(beanFactory)，这里就是实例化上面注册的bean定义，以RequestMappingHandlerMapping为例，会调用((InitializingBean) bean).afterPropertiesSet()
          - RequestMappingHandlerMapping
          - 会将mapping和handlerMethod键值对放到this.mappingLookup中，mapping（也就是方法上的@RequestMapping获得的信息）: {GET /clearcache4shopcategory},handlerMethod代表（Controller和method）
          - 会将url和mapping键值对放到this.urlLookup中，其中url也就是/clearcache4shopcategory
          - this.registry.put(mapping, new MappingRegistration<>(mapping, handlerMethod, directUrls, name))
          - 可以通过url 找到对应 mapping信息，通过mapping可以找到对应的handlerMethod信息
        * finishRefresh()，publishEvent(new ContextRefreshedEvent(this))调用至FrameworkServlet.ContextRefreshListener.onApplicationEvent()，调用至FrameworkServlet.onApplicationEvent()，最终调用至DispatcherServlet.onRefresh()调用initStrategies(context)，会进行初始化initStrategies(context)
          - initMultipartResolver(context);
          - initLocaleResolver(context);
          - initThemeResolver(context);
          - initHandlerMappings(context);
          - initHandlerAdapters(context);
          - initHandlerExceptionResolvers(context);
          - initRequestToViewNameTranslator(context);
          - initViewResolvers(context);
          - initFlashMapManager(context);
          - 这里的核心作用是将IOC容器中实例化的bean分门别类地抽取出来，给DispatcherServlet来管理和使用
    * ——initFrameworkServlet()
* **处理流程**
  - HttpServlet.service(ServletRequest req, ServletResponse res)
  - this.service(request, response);也即调用FrameworkServlet.service()
  - super.service(request, response)也即调用HttpServlet.service(HttpServletRequest req, HttpServletResponse resp)
  - 会根据方法的类型进行不同的调用：this.doGet(req, resp)或者this.doPost(req, resp)
  - 以doGet()为例，调用processRequest(request, response)，最终调用至DispatcherServlet.doService(request, response)，核心是doDispatch(request, response)，真正实现DispatcherServlet作为前端控制器的方法
    * step1.mappedHandler = getHandler(processedRequest)通过处理器映射器解析请求URL,获取对应的处理器对象,并添加了需要对该URL进行拦截的拦截器链表
      - 遍历this.handlerMappings，找到mapping
      - ——RequestMappingHandlerMapping处理@RequestMapping
      - ——BeanNameUrlHandlerMapping
      - ——SimpleUrlHandlerMapping处理（"/resources/**"  ->  ResourceHttpRequestHandler["/resources"]），处理css等文件
      - ——SimpleUrlHandlerMapping处理（"/**"  ->  DefaultServletHttpRequestHandler），处理html等文件
      - HandlerExecutionChain handler = mapping.getHandler(request)
      - ——通过url找到mapping再找到对应的HandlerMethod
      - ——根据url找到符合条件的interceptors
    * step2.HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler())根据查找到的处理器对象,找到可以适配它的处理器适配器对象
    * step3.mappedHandler.applyPreHandle(processedRequest, response)拦截器前置处理
      - this.interceptorIndex = i，记录最后一个成功的i，如果执行失败会倒序执行前面成功执行的interceptor的afterCompletion()
    * step4.mv = ha.handle(processedRequest, response, mappedHandler.getHandler())通过适配成功的处理器适配器对象,真正去执行处理器的方法,也就是Controller类的方法,返回的是shop/shoplist,model为空
      - args = getMethodArgumentValues(request, mavContainer, providedArgs)执行参数解析
      - ——在spring3.0之前，如果要自己实现一个从字符串到其它对象的转换，那么就需要实现PropertyEditor接口。PropertyEditor是遵循javaBean规范的属性处理器，其通过set方法设置属性值，通过get方法获取属性值，相应的转换逻辑就隐藏于其中。这个接口惟一的问题在于，他们不是线程安全的.并且只能实现从字符串到其它对象的转换。
      - ——在spring3.0，提供了一个更简单的类型转换器接口。在spring mvc中，可以使用内置的或者自己实现的接口来实现从请求参数到对象之间的转换。
      - ——Converter<S,T>接口：一个S类型的对象，将其转换为T类型的对象。其中S类型并不局限于字符串，可以是任何类型。这是与PropertyEditor的主要区别。
      - ——使用ConversionService及自定义转换器。ConversionServiceFactoryBean 默认内置了许多转换器，可以完成一般的数据类型转换，如：String类型参数转换为各种基本数据类型、包装类、Array、集合类等。但是，稍微复杂的转换就不行了，比如：日期格式的字符串要求转换为Date类型，以及其他开发者自定义的格式的数据的转换。这个时候，可以自定义数据转换器，并注册到Spring中即可。
      - doInvoke(args)反射调用方法
      - this.returnValueHandlers.handleReturnValue(returnValue, getReturnValueType(returnValue), mavContainer, webRequest)
      - ——找到返回值处理器：如果返回值使用@ResponseBody注解，返回的是RequestResponseBodyMethodProcessor
      - ——handler.handleReturnValue(returnValue, returnType, mavContainer, webRequest)调用返回值处理器进行处理，例如RequestResponseBodyMethodProcessor.handleReturnValue()
        * 从this.messageConverters挑选出一个能够进行转换的converter，genericConverter是MappingJackson2HttpMessageConverter，调用的是MappingJackson的API，该方法主要就是将POJO类使用MappingJackson的API转换为JSON字符串，这里是将HashMap转换为JSON字符串。
    * step5.mappedHandler.applyPostHandle(processedRequest, response, mv)拦截器后置处理，调用方向相反
    * step6.processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException)处理执行结果,也就是对进行ModelAndView处理：视图解析、视图渲染
      - 异常处理：mv = processHandlerException(request, response, handler, exception)
      - 正常情况：render(mv, request, response)
      - ——view = resolveViewName(viewName, mv.getModelInternal(), locale, request)
        * InternalResourceViewResolver定义了prefix和suffix，这里会加上去view.setUrl(getPrefix() + viewName + getSuffix());
      - ——view.render(mv.getModelInternal(), request, response)
  


# 1.springMVC之url和bean映射原理和源码解析-映射基本过程

（1）springMVC配置映射，需要在xml配置文件中配置<mvc:annotation-driven >  </mvc:annotation-driven>

（2）配置后，该配置将会交由org.springframework.web.servlet.config.MvcNamespaceHandler处理，该类会转交给org.springframework.web.servlet.config.AnnotationDrivenBeanDefinitionParser做解析。

（3）AnnotationDrivenBeanDefinitionParser做解析后，会向IOC容器中注册多个对象。其中有这个对象org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping

（4）RequestMappingHandlerMapping对象在IOC实例化bean阶段，会调用该对象afterPropertiesSet()方法做url和bean的映射。

（5）该映射是从IOC中找到标注有@Controller，@RequestMapping注解的bean，然后反射所有的方法Mehod对象和注解中配置的url值进行映射。

（6）在org.springframework.web.servlet.DispatcherServlet初始化调用init()方法，在IOC解析，aop编制好后，会调用initStrategies(ApplicationContext context) 方法里调用initHandlerMappings(context)中。将IOC中所有实现HandlerMapping接口的bean注入到dispatcherServlet属性List<HandlerMapping> handlerMappings中

 
# 2.springMVC拦截器的注入即应用的原理解析

（1）MvcNamespaceHandler在解析<mvc:annotation-driven ></mvc:annotation-driven>时向IOC容器中注册RequestMappingHandlerMapping

（2）MvcNamespaceHandler在解析<mvc:interceptors> </mvc:interceptors>时向IOC容器中注册一个个org.springframework.web.servlet.handler.MappedInterceptor拦截器对象

（3）RequestMappingHandlerMapping类是实现ApplicationContextAware接口的类，会在实例化的时候执行setApplicationContext(ApplicationContext context)方法，该方法内部初始化拦截器.将IOC容器中所有的MappedInterceptor对象赋值给RequestMappingHandlerMapping类的继承下来的属性private final List<MappedInterceptor> mappedInterceptors = new ArrayList<MappedInterceptor>();中

（4）未来在客户端发送请求，匹配到controller的时候会在RequestMappingHandlerMapping内部找到这个controller路径符合的拦截器，执行。并缓存起来。

 
# 3.springMVC之请求数据注入Controller方法原理解析

（1）在加载IOC容器的时候，解析xml配置文件，解析到<mvc:annotation-driven >  </mvc:annotation-driven>配置后，会向IOC容器中注册一个对象org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter

（2）同时会解析<mvc:message-converters register-defaults="true">配置，看是否有自定义的注入转换器。如果有，并且register-defaults=false,则就使用配置的注入转换器。如果register-defaults=true，则默认还会添加多个spring框架默认的注入转换器。将注入转换器加载到IOC容器中。举例如下（不全）

* org.springframework.http.converter.ByteArrayHttpMessageConverter
* org.springframework.http.converter.StringHttpMessageConverter
* org.springframework.http.converter.ResourceHttpMessageConverter
* org.springframework.http.converter.xml.SourceHttpMessageConverter
* org.springframework.http.converter.support.AllEncompassingFormHttpMessageConverter

（3）将这些注入转换器形成一个集合。赋值给RequestMappingHandlerAdapter的属性messageConverters。

（4）在IOC容器实例化RequestMappingHandlerAdapter的方法的时候，会调用afterPropertiesSet()方法做初始化操作。会将注入器包装成 RequestResponseBodyMethodProcessor对象放入List<HandlerMethodArgumentResolver>，赋值给this.argumentResolvers属性。

（5）在网络请求过来的时候，会根据反射得到要执行的controller方法的参数列表。根据参数列表，找到合适的HttpMessageConverter去从HttpserveltRequest中读取请求体，然后进行参数组装。将所有的参数列表组装成数据。然后反射执行Controller方法。

（6）执行完Controller的方法后，会调用属性returnValueHandlers（也会把HttpMessageConverter包装RequestResponseBodyMethodProcessor对象加入List<HandlerMethodReturnValueHandler> handlers中赋值给该属性）。进行相应的回调操作。

 

# 4.DispatcherServlet执行网络请求，转发致Controller的大致过程

 （1）几个关键的对象

* org.springframework.web.servlet.HandlerMapping接口实现类org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping
  - 该对象是做url和controller的映射作用。同时还存放了拦截器的配置。
  - 在解析xml配置 <mvc:annotation-driven >时候spring会编码方式注入IOC容器，在IOC容器实例化的时候会调用afterPropertiesSet()方法初始化映射关系。调用setApplicationContext()方法的时候初始化拦截器配置。
* org.springframework.web.servlet.HandlerAdapter接口实现类org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter
  - 该对象包换了http请求注入转换器，http响应注入转换器。

 
（2）DispatcherServlet执行过程

* http请求进入，先通过HttpServeleRequest对象从DispatcherServlet属性List<HandlerMapping> handlerMappings中获取拦截器链子，请求执行的Controller包装类HandlerExecutionChain mappedHandler。
* 然后根据HandlerExecutionChain mappedHandler 从List<HandlerAdapter> handlerAdapters中获取一个RequestMappingHandlerAdapter
* 执行拦截器请求前的方法preHandle方法
* RequestMappingHandlerAdapter调用handle方法执行请求。（注入，反射执行controller方法）
* 执行拦截器请求前的方法postHandle方法



# 5.<mvc:annotation-driven >  </mvc:annotation-driven>配置，由AnnotationDrivenBeanDefinitionParser解析会在IOC中注册的bean大致如下

1、会自动注册RequestMappingHandlerMapping、RequestMappingHandlerAdapter、ExceptionHandlerExceptionResolver三个bean支持使用了像@RquestMapping、ExceptionHandler等等的注解的controller 方法去处理请求。
2、支持使用了ConversionService的实例对表单参数进行类型转换。
3、支持使用@NumberFormat、@NumberFormat注解对数据类型进行格式化。
4、支持使用@Valid对javaBean进行JSR-303验证。
5、支持使用@RequestBody、@ResponseBody。


AnnotationDrivenBeanDefinitionParser解析会在IOC中注册的bean如下：

* 1）RequestMappingHandlerMapping
  - 属性：ContentNegotiationManagerFactoryBean
* 2）ConfigurableWebBindingInitializer
  - 属性：FormattingConversionServiceFactoryBean
  - 属性：OptionalValidatorFactoryBean
* 3）RequestMappingHandlerAdapter
  - 属性：contentNegotiationManager：ContentNegotiationManagerFactoryBean
  - 属性：webBindingInitializer：  ConfigurableWebBindingInitializer
  - 属性：messageConverters 
    * 3-1）&#60;mvc:message-converters register-defaults="true"&#62;
    ByteArrayHttpMessageConverter(默认加载)
    StringHttpMessageConverter（默认加载）
    FastJsonHttpMessageConverter（配置）
    ResourceHttpMessageConverter（默认加载）
    SourceHttpMessageConverter（默认加载）
    AllEncompassingFormHttpMessageConverter（默认加载）
    AtomFeedHttpMessageConverter（不加载）
    RssChannelHttpMessageConverter（不加载）
    MappingJackson2XmlHttpMessageConverter（不加载）
    Jaxb2RootElementHttpMessageConverter（默认加载）
    GsonHttpMessageConverter（不加载）
* 4）CompositeUriComponentsContributorFactoryBean
  - 属性：handlerAdapter：RequestMappingHandlerAdapter
  - 属性：conversionService：FormattingConversionServiceFactoryBean
* 5）ConversionServiceExposingInterceptor
* 6）ExceptionHandlerExceptionResolver
* 7）MappedInterceptor
* 8）ResponseStatusExceptionResolver
* 9）DefaultHandlerExceptionResolver
* 10）BeanNameUrlHandlerMapping
* 11）HttpRequestHandlerAdapter
* 12）SimpleControllerHandlerAdapter



# 6.适配器

springmvc通过HandlerMapping获取到可以处理的handler，这些handler的类型各不相同，对请求的预处理，参数获取都不相同，最简单的做法是根据不同的handler类型，做一个分支处理，不同的handler编写不同的代码。

这样的问题是很明显的，分支判断复杂，代码庞大，不符合单一职责原则。如果要增加一种handler类型，需要修改代码增加分支处理，违反了开闭原则。DispatcherServelt与多个handler发生了交互，违反迪米特法则。

而使用适配器模式，就可以很好的解决这个问题：

不直接对handler进行处理，而是将handler交给适配器HandlerAdapter去处理，这样DispatcherServlet交互的类就只剩下一个接口，HandlerAdapter，符合迪米特法则，尽可能少的与其他类发生交互；

将handler交给HandlerAdapter处理后，不同类型的handler被对应类型的HandlerAdapter处理，每个HandlerAdapter都只完成单一的handler处理，符合单一职责原则；

如果需要新增一个类型的handler，只需要新增对应类型的HandlerAdapter就可以处理，无需修改原有代码，符合开闭原则。

这样，不同的handler的不同处理方式，就在HandlerAdapter中得到了适配，对于DispatcherServlet来讲，只需要统一的调用HandlerAdapter的handle()方法就可以了，无需关注不同handler的处理细节。


HandlerMapping抽象了请求URL到请求处理器之间的映射，而HandlerAdapter用于封装对请求处理器的真正调用。打个比方讲，当某个客户请求到达时，DispatcherServlet会询问HandlerMapping哪个请求处理器能处理,然后找到支持该请求处理器的HandlerAdapter,然后将该请求处理器的调用交给该HandlerAdapter完成。这个例子简单地描述了DispatcherServlet处理请求工作的主流程，可以看出，DispatcherServlet所做的事情很简单，可以认为仅仅是一个主流程，而无需关心底层细节。而这里面很重要的两个底层细节: 请求到请求处理器的映射关系,请求处理器如何处理请求的具体逻辑，都不是DispatcherServlet关注点，相反，HandlerMapping和HandlerAdapter这两个队员承担了处理这些底层细节的角色，而且它们之间的分工很明确，可以协调配合却又不重叠。


HandlerMapping接口：

```
public interface HandlerMapping {
    // 找到跟指定请求对应的请求处理器，封装为一个HandlerExecutionChain返回
	HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception;
}
```

HandlerAdapter 接口

```
public interface HandlerAdapter {
    // 当前请求处理器适配器是否支持指定的请求处理器handler
    boolean supports(Object handler);
    // 针对给定的请求/响应对象request/response调用给定的请求处理器handler,
    // 这里的handler一定符合条件 : this.supports(handler)==true
    ModelAndView handle(HttpServletRequest request,HttpServletResponse response,Object handler) throws Exception;
    // 跟 HttpServlet 定义的getLastModified语义相同
    long getLastModified(HttpServletRequest request, Object handler);
}
```

另外Spring MVC框架自身提供了一些HandlerMapping和HandlerAdapter的实现类。这些实现类基本上能满足开发人员所有的各种请求/请求处理器的映射需求，一般无需开发人员另外提供自定义实现。

框架自身提供的HandlerMapping实现类：

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/HandlerMapping.jpg" width="620px" > </div><br>

框架自身提供的HandleAdapter实现类

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/HandleAdapater.jpg" width="620px" > </div><br>


应用配置时,开发人员需要配置不同类型和目的的HandlerMapping/HandlerAdapter组件。可能通过属性文件，可能通过注解，也可能是通过代码直接配置。

当然，如果是基于Spring boot的Spring MVC应用，框架提供了缺省的配置，如无额外需求，开发人员无需另外配置。

应用启动时，各种配置会被汇集起来,构建HandlerMapping和HandlerAdapter的组件bean并注册到容器，这些组件bean会基于HandlerMapping和HandlerAdapter的实现类。并最终会被DispatcherServlet引用。

应用运行时，正如上面DispatcherServlet主流程所描述的，HandlerMapping和HandlerAdapter组件bean在相应的环节被应用起来协调完成整个请求的处理。
