
# 1.default-servlet-handler作用

```
    <servlet>
        <servlet-name>spring-dispatcher</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>classpath:spring/spring-web.xml</param-value>
        </init-param>
        <init-param>
            <param-name>contextClass</param-name>
            <param-value>
                org.springframework.web.context.support.XmlWebApplicationContext
            </param-value>
        </init-param>
    </servlet>
    <servlet-mapping>
        <servlet-name>spring-dispatcher</servlet-name>
        <!--默认匹配所有需求-->
        <url-pattern>/</url-pattern>
    </servlet-mapping>
```

关于配置成"/"，在 Spring 的官方文档中这样描述：

```
The caveat to overriding the “/” Servlet mapping is that the RequestDispatcher for the default Servlet must be retrieved by name rather than by path.
```

Servlet 的 RequestDispatcher 必须通过名称而不是路径来检索。 换句话说就是 Spring MVC 将接收到的所有请求都看作是一个普通的请求，包括对于静态资源的请求。这样以来，所有对于静态资源的请求都会被看作是一个普通的后台控制器请求，导致请求找不到而报 404 异常错误。


对于这个问题 Spring MVC 在全局配置文件中提供了一个<mvc:default-servlet-handler/>标签。在 WEB 容器启动的时候会在上下文中定义一个 DefaultServletHttpRequestHandler，它会对DispatcherServlet的请求进行处理，如果该请求已经作了映射，那么会接着交给后台对应的处理程序，如果没有作映射，就交给 WEB 应用服务器默认的 Servlet 处理，从而找到对应的静态资源，只有再找不到资源时才会报错。


```
 <mvc:default-servlet-handler />
```


一般 WEB 应用服务器默认的 Servlet 都是 default。如果默认 Servlet 用不同名称自定义配置，或者在缺省 Servlet 名称未知的情况下使用了不同的 Servlet 容器，则必须显式提供默认 Servlet 的名称，如下：

```
<mvc:default-servlet-handler default-servlet-name="myCustomDefaultServlet"/>
```

# 2.default-servlet-handler元素解析

MvcNamespaceHandler.parse()，调用的是父类NamespaceHandlerSupport.parse()：

```
BeanDefinitionParser parser = findParserForElement(element, parserContext);
return (parser != null ? parser.parse(element, parserContext) : null);
```

parser为DefaultServletHandlerBeanDefinitionParser：

```
	public BeanDefinition parse(Element element, ParserContext parserContext) {
		Object source = parserContext.extractSource(element);

		String defaultServletName = element.getAttribute("default-servlet-name");
		RootBeanDefinition defaultServletHandlerDef = new RootBeanDefinition(DefaultServletHttpRequestHandler.class);
		defaultServletHandlerDef.setSource(source);
		defaultServletHandlerDef.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
		if (StringUtils.hasText(defaultServletName)) {
			defaultServletHandlerDef.getPropertyValues().add("defaultServletName", defaultServletName);
		}
		String defaultServletHandlerName = parserContext.getReaderContext().generateBeanName(defaultServletHandlerDef);
		parserContext.getRegistry().registerBeanDefinition(defaultServletHandlerName, defaultServletHandlerDef);
		parserContext.registerComponent(new BeanComponentDefinition(defaultServletHandlerDef, defaultServletHandlerName));

		Map<String, String> urlMap = new ManagedMap<>();
		urlMap.put("/**", defaultServletHandlerName);

		RootBeanDefinition handlerMappingDef = new RootBeanDefinition(SimpleUrlHandlerMapping.class);
		handlerMappingDef.setSource(source);
		handlerMappingDef.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
		handlerMappingDef.getPropertyValues().add("urlMap", urlMap);

		String handlerMappingBeanName = parserContext.getReaderContext().generateBeanName(handlerMappingDef);
		parserContext.getRegistry().registerBeanDefinition(handlerMappingBeanName, handlerMappingDef);
		parserContext.registerComponent(new BeanComponentDefinition(handlerMappingDef, handlerMappingBeanName));

		// Ensure BeanNameUrlHandlerMapping (SPR-8289) and default HandlerAdapters are not "turned off"
		MvcNamespaceUtils.registerDefaultComponents(parserContext, source);

		return null;
	}

```

parse()流程如下：

* step1.注册DefaultServletHttpRequestHandler
* step2.将"/\*\*"映射到上面配置的defaultServletHandlerName，并加入到urlMap中：urlMap.put("/\*\*", defaultServletHandlerName)
* step3.注册SimpleUrlHandlerMapping，并将urlMap添加到期BeanDefinition中去
* step4.又是注册默认的组件registerDefaultComponents()

# 3.处理过程


/o2o/shopadmin/shopmanagement返回/WEB-INF/html/shop/shopmanagement.html

* ste1.mappedHandler = getHandler(processedRequest)
  - this.handlerMappings
    * RequestMappingHandlerMapping普通的后台控制器请求
    * BeanNameUrlHandlerMapping
    * SimpleUrlHandlerMapping： /resources/** -> ResourceHttpRequestHandler#0
    * SimpleUrlHandlerMapping ：/** -> DefaultServletHttpRequestHandler#0
  - /o2o/shopadmin/shopmanagement 对应的是从RequestMappingHandlerMapping获取的mappedHandler
    * handler：com.imooc.o2o.web.shopadmin.ShopAdminController#shopManagement()
    * 三个interceptorList：ConversionServiceExposingInterceptor，ResourceUrlProviderExposingInterceptor，ShopLoginInterceptor
* ste2.HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler())
  - RequestMapppingHandlerAdapter
* ste3.mappedHandler.applyPreHandle(processedRequest, response)
  - 调用三个interceptor进行调用，interceptor.preHandle(request, response, this.handler)
* ste4.mv = ha.handle(processedRequest, response, mappedHandler.getHandler())
  - returnValue = invokeForRequest(webRequest, mavContainer, providedArgs) 调用ShopAdminController#shopManagement()返回shop/shopmanagement
  - this.returnValueHandlers.handleReturnValue()
* ste5.mappedHandler.applyPostHandle(processedRequest, response, mv)
  - 反方向调用三个interceptor进行调用，interceptor.postHandle(request, response, this.handler, mv)
* ste6.processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException)
  - render(mv, request, response)
    * resolveViewName()解析视图为/WEB-INF/html/shop/shopmanagement.html
    * view.render()



发起/o2o/WEB-INF/html/shop/shopmanagement.html


* ste1.mappedHandler = getHandler(processedRequest)
  - this.handlerMappings
    * RequestMappingHandlerMapping普通的后台控制器请求
    * BeanNameUrlHandlerMapping
    * SimpleUrlHandlerMapping： /resources/** -> ResourceHttpRequestHandler#0
    * SimpleUrlHandlerMapping ：/** -> DefaultServletHttpRequestHandler#0
  - /o2o/WEB-INF/html/shop/shopmanagement.html 对应的是从SimpleUrlHandlerMapping（默认，最后一个）获取的mappedHandler
    * handler：DefaultServletHttpRequestHandler
    * 三个interceptorList：ConversionServiceExposingInterceptor，ResourceUrlProviderExposingInterceptor，ShopLoginInterceptor
* ste2.HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler())
  - HttpRequestHandlerAdapter
* ste3.mappedHandler.applyPreHandle(processedRequest, response)
  - 调用三个interceptor进行调用，interceptor.preHandle(request, response, this.handler)
* ste4.mv = ha.handle(processedRequest, response, mappedHandler.getHandler())
  - ((HttpRequestHandler) handler).handleRequest(request, response)
    * rd.forward(request, response)
* ste5.mappedHandler.applyPostHandle(processedRequest, response, mv)
  - 反方向调用三个interceptor进行调用，interceptor.postHandle(request, response, this.handler, mv)
* ste6.processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException)
  - render(mv, request, response)
    * resolveViewName()解析视图为/WEB-INF/html/shop/shopmanagement.html
    * view.render()


/WEB-INF/html/shop/shopmanagement.html包含/resources/css/shop/shopmanagement.css

* ste1.mappedHandler = getHandler(processedRequest)
  - this.handlerMappings
    * RequestMappingHandlerMapping普通的后台控制器请求
    * BeanNameUrlHandlerMapping
    * SimpleUrlHandlerMapping： /resources/** -> ResourceHttpRequestHandler#0
    * SimpleUrlHandlerMapping ：/** -> DefaultServletHttpRequestHandler#0
  - /o2o/WEB-INF/html/shop/shopmanagement.html 对应的是从SimpleUrlHandlerMapping获取的mappedHandler
    * handler：ResourceHttpRequestHandler
    * 三个interceptorList：ConversionServiceExposingInterceptor，ResourceUrlProviderExposingInterceptor，ShopLoginInterceptor
* ste2.HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler())
  - HttpRequestHandlerAdapter
* ste3.mappedHandler.applyPreHandle(processedRequest, response)
  - 调用三个interceptor进行调用，interceptor.preHandle(request, response, this.handler)
* ste4.mv = ha.handle(processedRequest, response, mappedHandler.getHandler())
  - ((HttpRequestHandler) handler).handleRequest(request, response)
    * resource = getResource(request)，[/resources/css/shop/shopmanagement.css]
* ste5.mappedHandler.applyPostHandle(processedRequest, response, mv)
  - 反方向调用三个interceptor进行调用，interceptor.postHandle(request, response, this.handler, mv)
* ste6.processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException)
  - render(mv, request, response)
    * resolveViewName()解析视图为/WEB-INF/html/shop/shopmanagement.html
    * view.render()