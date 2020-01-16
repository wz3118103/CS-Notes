
# 1.ResourcesBeanDefinitionParser.parse()方法

```
    <!-- 2.静态资源默认servlet配置
        (1)加入对静态资源的处理：js,gif,png
        (2)允许使用"/"做整体映射 -->
    <mvc:resources mapping="/resources/**" location="/resources/" />
```


MvcNamespaceHandler.init()：

```
registerBeanDefinitionParser("resources", new ResourcesBeanDefinitionParser());
```

在DefaultBeanDefinitionDocumentReader.parseBeanDefinitions()：

```
	protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
		if (delegate.isDefaultNamespace(root)) {
			NodeList nl = root.getChildNodes();
			for (int i = 0; i < nl.getLength(); i++) {
				Node node = nl.item(i);
				if (node instanceof Element) {
					Element ele = (Element) node;
					if (delegate.isDefaultNamespace(ele)) {
						parseDefaultElement(ele, delegate);
					}
					else {
						delegate.parseCustomElement(ele);
					}
				}
			}
		}
		else {
			delegate.parseCustomElement(root);
		}
	}
```

进而调用：

```
	public BeanDefinition parseCustomElement(Element ele) {
		return parseCustomElement(ele, null);
	}
```

调用：

```
	public BeanDefinition parseCustomElement(Element ele, @Nullable BeanDefinition containingBd) {
		String namespaceUri = getNamespaceURI(ele);
		if (namespaceUri == null) {
			return null;
		}
		NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
		if (handler == null) {
			error("Unable to locate Spring NamespaceHandler for XML schema namespace [" + namespaceUri + "]", ele);
			return null;
		}
		return handler.parse(ele, new ParserContext(this.readerContext, this, containingBd));
	}
```

调用：

```
	public BeanDefinition parse(Element element, ParserContext parserContext) {
		BeanDefinitionParser parser = findParserForElement(element, parserContext);
		return (parser != null ? parser.parse(element, parserContext) : null);
	}
```

根据localName：resources获取相应的BeanDefinitionParser，也即ResourcesBeanDefinitionParser。

```
	private BeanDefinitionParser findParserForElement(Element element, ParserContext parserContext) {
		String localName = parserContext.getDelegate().getLocalName(element);
		BeanDefinitionParser parser = this.parsers.get(localName);
		if (parser == null) {
			parserContext.getReaderContext().fatal(
					"Cannot locate BeanDefinitionParser for element [" + localName + "]", element);
		}
		return parser;
	}
```

重点来看其parser()方法：

```
	public BeanDefinition parse(Element element, ParserContext context) {
		Object source = context.extractSource(element);

		registerUrlProvider(context, source);

		RuntimeBeanReference pathMatcherRef = MvcNamespaceUtils.registerPathMatcher(null, context, source);
		RuntimeBeanReference pathHelperRef = MvcNamespaceUtils.registerUrlPathHelper(null, context, source);

		String resourceHandlerName = registerResourceHandler(context, element, pathHelperRef, source);
		if (resourceHandlerName == null) {
			return null;
		}

		Map<String, String> urlMap = new ManagedMap<>();
		String resourceRequestPath = element.getAttribute("mapping");
		if (!StringUtils.hasText(resourceRequestPath)) {
			context.getReaderContext().error("The 'mapping' attribute is required.", context.extractSource(element));
			return null;
		}
		urlMap.put(resourceRequestPath, resourceHandlerName);

		RootBeanDefinition handlerMappingDef = new RootBeanDefinition(SimpleUrlHandlerMapping.class);
		handlerMappingDef.setSource(source);
		handlerMappingDef.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
		handlerMappingDef.getPropertyValues().add("urlMap", urlMap);
		handlerMappingDef.getPropertyValues().add("pathMatcher", pathMatcherRef).add("urlPathHelper", pathHelperRef);

		String orderValue = element.getAttribute("order");
		// Use a default of near-lowest precedence, still allowing for even lower precedence in other mappings
		Object order = StringUtils.hasText(orderValue) ? orderValue : Ordered.LOWEST_PRECEDENCE - 1;
		handlerMappingDef.getPropertyValues().add("order", order);

		RuntimeBeanReference corsRef = MvcNamespaceUtils.registerCorsConfigurations(null, context, source);
		handlerMappingDef.getPropertyValues().add("corsConfigurations", corsRef);

		String beanName = context.getReaderContext().generateBeanName(handlerMappingDef);
		context.getRegistry().registerBeanDefinition(beanName, handlerMappingDef);
		context.registerComponent(new BeanComponentDefinition(handlerMappingDef, beanName));

		// Ensure BeanNameUrlHandlerMapping (SPR-8289) and default HandlerAdapters are not "turned off"
		// Register HttpRequestHandlerAdapter
		MvcNamespaceUtils.registerDefaultComponents(context, source);

		return null;
	}
```

registerResourceHandler()方法注册了ResourceHttpRequestHandler，这里会将location属性解析出来，设置到ResourceHttpRequestHandler的BeanDefinition中去。

```
	private String registerResourceHandler(ParserContext context, Element element,
			RuntimeBeanReference pathHelperRef, @Nullable Object source) {

		String locationAttr = element.getAttribute("location");
		if (!StringUtils.hasText(locationAttr)) {
			context.getReaderContext().error("The 'location' attribute is required.", context.extractSource(element));
			return null;
		}

		RootBeanDefinition resourceHandlerDef = new RootBeanDefinition(ResourceHttpRequestHandler.class);
		resourceHandlerDef.setSource(source);
		resourceHandlerDef.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);

		MutablePropertyValues values = resourceHandlerDef.getPropertyValues();
		values.add("urlPathHelper", pathHelperRef);
		values.add("locationValues", StringUtils.commaDelimitedListToStringArray(locationAttr));

		String cacheSeconds = element.getAttribute("cache-period");
		if (StringUtils.hasText(cacheSeconds)) {
			values.add("cacheSeconds", cacheSeconds);
		}

		Element cacheControlElement = DomUtils.getChildElementByTagName(element, "cache-control");
		if (cacheControlElement != null) {
			CacheControl cacheControl = parseCacheControl(cacheControlElement);
			values.add("cacheControl", cacheControl);
		}

		Element resourceChainElement = DomUtils.getChildElementByTagName(element, "resource-chain");
		if (resourceChainElement != null) {
			parseResourceChain(resourceHandlerDef, context, resourceChainElement, source);
		}

		Object manager = MvcNamespaceUtils.getContentNegotiationManager(context);
		if (manager != null) {
			values.add("contentNegotiationManager", manager);
		}

		String beanName = context.getReaderContext().generateBeanName(resourceHandlerDef);
		context.getRegistry().registerBeanDefinition(beanName, resourceHandlerDef);
		context.registerComponent(new BeanComponentDefinition(resourceHandlerDef, beanName));
		return beanName;
	}
```

这里总结一下parse()方法流程

* step1.注册了ResourceHttpRequestHandler，这里会将location属性解析出来，设置到ResourceHttpRequestHandler的BeanDefinition中去
* step2.将mapping解析出来，然后将mapping与resourceHandler的映射放到urlMap中去，urlMap.put(resourceRequestPath, resourceHandlerName)
* step3.注册SimpleUrlHandlerMapping，并将urlMap添加到期BeanDefinition中去
* step4.MvcNamespaceUtils.registerDefaultComponents(context, source)
  - registerBeanNameUrlHandlerMapping(parserContext, source);注册BeanNameUrlHandlerMapping
  - registerHttpRequestHandlerAdapter(parserContext, source);注册HttpRequestHandlerAdapter
  - registerSimpleControllerHandlerAdapter(parserContext, source);注册SimpleControllerHandlerAdapter
  - registerHandlerMappingIntrospector(parserContext, source);注册HandlerMappingIntrospector


这里的核心就是SimpleUrlHandlerMapping，其将mapping与包含location属性的ResourceHttpRequestHandler进行映射，因此匹配mapping属性值的静态资源访问路径将交由ResourceHttpRequestHandler实例来处理。


# 2.访问静态资源时的处理过程


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


/WEB-INF/html/shop/shopmanagement.html包含resources/css/shop/shopmanagement.css

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


# 3.ResourceHttpRequestHandler.handleRequest()

```
public void handleRequest(HttpServletRequest request, HttpServletResponse response)
        throws ServletException, IOException {

    // For very general mappings (e.g. "/") we need to check 404 first
    // 使用SpringMVC默认的方式(例如PathResourceResolver)获取Resource
    // 这里是一个扩展点, 开发者可以依据自身的需求加入自定义的ResourceResolver; 下面的部分我将给出一个例子
    Resource resource = getResource(request);
    // 没有找到就直接返回了
    if (resource == null) {
        logger.trace("No matching resource found - returning 404");
        response.sendError(HttpServletResponse.SC_NOT_FOUND);
        return;
    }

    // 如果这次的HTTP方法为 OPTIONS, 也是直接返回
    if (HttpMethod.OPTIONS.matches(request.getMethod())) {
        response.setHeader("Allow", getAllowHeader());
        return;
    }

    // Supported methods and required session
    checkRequest(request);

    // Header phase
    // 没有修改就直接返回了
    if (new ServletWebRequest(request, response).checkNotModified(resource.lastModified())) {
        logger.trace("Resource not modified - returning 304");
        return;
    }

    // Apply cache settings, if any
    prepareResponse(response);

    // Check the media type for the resource
    MediaType mediaType = getMediaType(request, resource);

    // Content phase
    if (METHOD_HEAD.equals(request.getMethod())) {
        setHeaders(response, resource, mediaType);
        logger.trace("HEAD request - skipping content");
        return;
    }

    ServletServerHttpResponse outputMessage = new ServletServerHttpResponse(response);
    // 如果这次不是分Range, 开始准备Response
    if (request.getHeader(HttpHeaders.RANGE) == null) {
        // 开始设置响应头
        // 注意这里有个细节是, 这里会设置响应头 "Content-Length"
        setHeaders(response, resource, mediaType);
        // 这里的 resourceHttpMessageConverter 实际为 ResourceHttpMessageConverter实例
        // 注意这里就会将读取到的Resource推送给响应流
        this.resourceHttpMessageConverter.write(resource, mediaType, outputMessage);
    }
    else {
        // 分Range, 略
    }
}
————————————————
版权声明：本文为CSDN博主「夫礼者」的原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/lqzkcx3/article/details/78601545
```