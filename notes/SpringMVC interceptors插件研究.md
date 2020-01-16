
# 1.InterceptorsBeanDefinitionParser.parse()

```
    <!-- 5.权限拦截器 -->
   <mvc:interceptors>
        &lt;!&ndash;校验是否已登录了店家管理系统的拦截器&ndash;&gt;
        <mvc:interceptor>
            <mvc:mapping path="/shopadmin/**"/>
            <mvc:exclude-mapping path="/shopadmin/addshopauthmap"/>
            <bean id="ShopInterceptor" class="com.imooc.o2o.interceptor.shopadmin.ShopLoginInterceptor"/>
        </mvc:interceptor>

        &lt;!&ndash;校验是否对该商店有操作权限的拦截器&ndash;&gt;
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
```



```
class InterceptorsBeanDefinitionParser implements BeanDefinitionParser {

	public BeanDefinition parse(Element element, ParserContext context) {
		context.pushContainingComponent(
				new CompositeComponentDefinition(element.getTagName(), context.extractSource(element)));

		RuntimeBeanReference pathMatcherRef = null;
		if (element.hasAttribute("path-matcher")) {
			pathMatcherRef = new RuntimeBeanReference(element.getAttribute("path-matcher"));
		}

		List<Element> interceptors = DomUtils.getChildElementsByTagName(element, "bean", "ref", "interceptor");
		for (Element interceptor : interceptors) {
			RootBeanDefinition mappedInterceptorDef = new RootBeanDefinition(MappedInterceptor.class);
			mappedInterceptorDef.setSource(context.extractSource(interceptor));
			mappedInterceptorDef.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);

			ManagedList<String> includePatterns = null;
			ManagedList<String> excludePatterns = null;
			Object interceptorBean;
			if ("interceptor".equals(interceptor.getLocalName())) {
				includePatterns = getIncludePatterns(interceptor, "mapping");
				excludePatterns = getIncludePatterns(interceptor, "exclude-mapping");
				Element beanElem = DomUtils.getChildElementsByTagName(interceptor, "bean", "ref").get(0);
				interceptorBean = context.getDelegate().parsePropertySubElement(beanElem, null);
			}
			else {
				interceptorBean = context.getDelegate().parsePropertySubElement(interceptor, null);
			}
			mappedInterceptorDef.getConstructorArgumentValues().addIndexedArgumentValue(0, includePatterns);
			mappedInterceptorDef.getConstructorArgumentValues().addIndexedArgumentValue(1, excludePatterns);
			mappedInterceptorDef.getConstructorArgumentValues().addIndexedArgumentValue(2, interceptorBean);

			if (pathMatcherRef != null) {
				mappedInterceptorDef.getPropertyValues().add("pathMatcher", pathMatcherRef);
			}

			String beanName = context.getReaderContext().registerWithGeneratedName(mappedInterceptorDef);
			context.registerComponent(new BeanComponentDefinition(mappedInterceptorDef, beanName));
		}

		context.popAndRegisterContainingComponent();
		return null;
	}
```

parser()：

* step1.遍历所有interceptor
* step2.针对每一个interceptor都注册一个MappedInterceptor，每一个都有三个参数
  - includePatterns
  - excludePatterns
  - interceptorBea


# 2.处理过程


/o2o/shopadmin/shopmanagement返回/WEB-INF/html/shop/shopmanagement.html

* ste1.mappedHandler = getHandler(processedRequest)
  - this.handlerMappings
    * RequestMappingHandlerMapping普通的后台控制器请求
    * BeanNameUrlHandlerMapping
    * SimpleUrlHandlerMapping： /resources/** -> ResourceHttpRequestHandler#0
    * SimpleUrlHandlerMapping ：/** -> DefaultServletHttpRequestHandler#0
  - /o2o/shopadmin/shopmanagement 对应的是从RequestMappingHandlerMapping获取的mappedHandler
    * handler：com.imooc.o2o.web.shopadmin.ShopAdminController#shopManagement()
    * 根据路径匹配情况，有三个interceptorList：ConversionServiceExposingInterceptor，ResourceUrlProviderExposingInterceptor，ShopLoginInterceptor
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