# 0.为什么要使用Spring

Q.使用 Spring 框架能带来哪些好处？
* Dependency Injection(DI) 方法使得构造器和 JavaBean properties 文件中的依赖关系一目了然。
* 与 EJB 容器相比较，IoC 容器更加趋向于轻量级。这样一来 IoC 容器在有限的内存和 CPU 资源的情况下进行应用程序的开发和发布就变得十分有利。
* Spring 并没有闭门造车，Spring 利用了已有的技术比如 ORM 框架、logging 框架、J2EE、Quartz 和 JDK Timer，以及其他视图技术。
* Spring 框架是按照模块的形式来组织的。由包和类的编号就可以看出其所属的模块，开发者仅仅需要选用他们需要的模块即可。
* 要测试一项用 Spring 开发的应用程序十分简单，因为测试相关的环境代码都已经囊括在框架中了。更加简单的是，利用 JavaBean 形式的 POJO 类，可以很方便的利用依赖注入来写入测试数据。
* Spring 的 Web 框架亦是一个精心设计的 Web MVC 框架，为开发者们在 web 框架的选择上提供了一个除了主流框架比如 Struts、过度设计的、不流行 web 框架的以外的有力选项。
* Spring 提供了一个便捷的事务管理接口，适用于小型的本地事物处理（比如在单 DB 的环境下）和复杂的共同事物处理（比如利用 JTA 的复杂 DB 环境）。


Q.开发中主要使用 Spring 的什么技术 ? 
* IOC 容器管理各层的组件
* 使用 AOP 配置声明式事务
* 整合其他框架.

Q.简述 AOP 和 IOC 概念 
* AOP:
  - Aspect Oriented Program, 面向(方面)切面的编程;Filter(过滤器) 也是一种 AOP. AOP 是一种新的方法论, 是对传统 OOP(Object-Oriented Programming, 面向对象编程) 的补充. AOP 的主要编程对象是切面(aspect), 而切面模块化横切关注点.可以举例通过事务说明.
* IOC: Invert Of Control, 控制反转. 也成为 DI(依赖注入)其思想是反转 资源获取的方向. 传统的资源查找方式要求组件向容器发起请求查找资源.作为 回应, 容器适时的返回资源. 而应用了IOC 之后, 则是容器主动地将资源推送 给它所管理的组件,组件所要做的仅是选择一种合适的方式来接受资源. 这种行 为也被称为查找的被动形式

# 1.IOC容器

Q.什么是控制反转(IOC)？什么是依赖注入？
* 控制反转是应用于软件工程领域中的，在运行时被装配器对象来绑定耦合对象的一种编程技巧，对象之间耦合关系在编译时通常是未知的。在传统的编程方式中，业务逻辑的流程是由应用程序中的早已被设定好关联关系的对象来决定的。在使用控制反转的情况下，业务逻辑的流程是由对象关系图来决定的，该对象关系图由装配器负责实例化，这种实现方式还可以将对象之间的关联关系的定义抽象化。而绑定的过程是通过“依赖注入”实现的。
* 控制反转是一种以给予应用程序中目标组件更多控制为目的设计范式，并在我们的实际工作中起到了有效的作用。
* 依赖注入是在编译阶段尚未知所需的功能是来自哪个的类的情况下，将其他对象所依赖的功能对象实例化的模式。这就需要一种机制用来激活相应的组件以提供特定的功能，所以依赖注入是控制反转的基础。否则如果在组件不受框架控制的情况下，框架又怎么知道要创建哪个组件？
* 在Java 中依赖注入有以下三种实现方式：
  - 构造器注入
  - Setter 方法注入
  - 接口注入

Q.请解释下 Spring 框架中的 IoC？
* Spring 中的 org.springframework.beans 包和 org.springframework.context 包构成了 Spring 框架 IoC 容器的基础。
* BeanFactory 接口提供了一个先进的配置机制，使得任何类型的对象的配置成为可能。
* ApplicationContex 接口对 BeanFactory（是一个子接口）进行了扩展，在 BeanFactory的基础上添加了其他功能，比如与 Spring 的 AOP 更容易集成，也提供了处理 message resource的机制（用于国际化）、事件传播以及应用层的特别配置，比如针对 Web 应用的WebApplicationContext。
* org.springframework.beans.factory.BeanFactory 是 Spring IoC 容器的具体实现，用来包装和管理前面提到的各种 bean。BeanFactory 接口是 Spring IoC 容器的核心接口。
* IOC:把对象的创建、初始化、销毁交给 spring 来管理，而不是由开发者控制，实现控制反转。


Q.BeanFactory 和 ApplicationContext 有什么区别？
* BeanFactory 可以理解为含有 bean 集合的工厂类。BeanFactory 包含了种 bean 的定义，以便在接收到客户端请求时将对应的 bean 实例化。
* BeanFactory 还能在实例化对象的时生成协作类之间的关系。此举将 bean 自身与 bean 客户端的配置中解放出来。BeanFactory 还包含 了 bean 生命周期的控制，调用客户端的初始化方法（initialization methods）和销毁方法（destruction methods）。
* 从表面上看，application context 如同 bean factory 一样具有 bean 定义、bean 关联关系的设置，根据请求分发 bean 的功能。但 applicationcontext 在此基础上还提供了其他的功能。
  - 提供了支持国际化的文本消息
  - 统一的资源文件读取方式
  - 已在监听器中注册的 bean 的事件
* 以下是几种较常见的 ApplicationContext 实现方式：
  - ClassPathXmlApplicationContext：从 classpath 的 XML 配置文件中读取上下文，并生成上下文定义。
  - FileSystemXmlApplicationContext ：由文件系统中的 XML 配置文件读取上下文。
  - XmlWebApplicationContext：由 Web 应用的 XML 文件读取上下文。
  - AnnotationConfigApplicationContext(基于 Java 配置启动容器) 


Q.Spring 有几种配置方式？
* 基于 XML 的配置
* 基于注解的配置（@Configuration）
* 基于 Java 的配置（@Component-ComponentScan）


Q.如何用基于 XML 配置的方式配置 Spring？ 
* 在Spring 框架中，依赖和服务需要在专门的配置文件来实现，我常用的 XML 格式的配置文件。这些配置文件的格式通常用<beans>开头，然后一系列的 bean 定义和专门的应用配置选项组成。
* SpringXML 配置的主要目的时候是使所有的 Spring 组件都可以用 xml 文件的形式来进行配置。这意味着不会出现其他的 Spring 配置类型（比如声明的方式或基于 Java Class 的配置方式）
* Spring 的 XML 配置方式是使用被 Spring 命名空间的所支持的一系列的 XML 标签来实现的。
* Spring 有以下主要的命名空间：context、beans、jdbc、tx、aop、mvc 和 aso。

Q.如何用基于 Java 配置的方式配置Spring？
* Spring 对 Java 配置的支持是由@Configuration 注解和@Bean 注解来实现的。由@Bean 注解的方法将会实例化、配置和初始化一个 新对象，这个对象将由 Spring 的 IoC 容器来管理。
* @Bean 声明所起到的作用与&#60;bean/> 元素类似。被 @Configuration 所注解的类则表示这个类的主要目的是作为 bean 定义的资源。被@Configuration 声明的类可以通过在同一个类的 内部调用@bean 方法来设置嵌入 bean 的依赖关系。
* 要使用组件组建扫描，仅需用@ComponentScan进行注解即可，然后在指定包中查找被@Component 声明的类，找到后将这些类按照 Sring bean 定义进行注册。

Q.怎样用注解的方式配置 Spring？
* Spring 在 2.5 版本以后开始支持用注解的方式来配置依赖注入。可以用注解的方式来替代 XML 方式的 bean 描述，可以将 bean 描述转移到组件类的内部，只需要在相关类上、方法上或者字段声明上使用注解即可。注解注入将会被容器在 XML注入之前被处理，所以后者会覆盖掉前者对于同一个属性的处理结果。
* 注解装配在 Spring 中是默认关闭的。所以需要在 Spring 文件中配置一下才能使用基于注解的装配模式。
* 在 &#60;context:annotation-config/>标签配置完成以后，就可以用注解的方式在 Spring 中向属性、方法和构造方法中自动装配变量。
* 下面是几种比较重要的注解类型：
  - @Autowired：该注解应用于有值设值方法、非设值方法、构造方法和变量。
  - @Qualifier：该注解和@Autowired 注解搭配使用，用于消除特定 bean 自动装配的歧义。
  - JSR-250 Annotations：Spring 支持基于 JSR-250 注解的以下注解，@Resource、@PostConstruct 和 @PreDestroy。


Q.请解释 Spring Bean 的生命周期？
* Spring Bean 的生命周期简单易懂。在一个 bean 实例被初始化时，需要执行一系列的初始化操作以达到可用的状态。同样的，当一个 bean 不在被调用时需要进行相关的析构操作，并从 bean 容器中移除。
* Spring bean factory 负责管理在 spring 容器中被创建的 bean 的生命周期。Bean 的生命周期由两组回调（call back）方法组成。
  - 初始化之后调用的回调方法。
  - 销毁之前调用的回调方法。
* Spring 框架提供了以下四种方式来管理 bean 的生命周期事件：
  - InitializingBean 和 DisposableBean 回调接口
  - 针对特殊行为的其他 Aware 接口
  - Bean 配置文件中的init()方法和 destroy()方法
  - @PostConstruct 和@PreDestroy 注解方式


Q.Spring Bean 的作用域之间有什么区别？
* Spring 容器中的 bean 可以分为 5 个范围。所有范围的名称都是自说明的，但是为了避免混淆，还是让我们来解释一下：
  - singleton：这种 bean 范围是默认的，这种范围确保不管接受到多少个请求，每个容器中只有一个bean 的实例，单例的模式由 bean factory 自身来维护。
  - prototype：原形范围与单例范围相反，为每一个 bean 请求提供一个实例。
  - request：在请求 bean 范围内会每一个来自客户端的网络请求创建一个实例，在请求完成以后，bean 会失效并被垃圾回收器回收。
  - Session：与请求范围类似，确保每个 session 中有一个 bean 的实例，在 session 过期后，bean会随之失效。
  - global- session：global-session 和 Portlet 应用相关。当你的应用部署在 Portlet 容器中工作时，它包含很多 portlet。如果 你想要声明让所有的 portlet 共用全局的存储变量的话，那么这全局变量需要存储在 global-session 中。全局作用域与 Servlet 中的 session 作用域效果相同。

Q.什么是 Spring inner beans？ 
* 在 Spring 框架中，无论何时 bean 被使用时，当仅被调用了一个属性。一个明智的做法是将这个bean 声明为内部 bean。内部 bean 可以用 setter 注入“属性”和构造方法注入“构造参数”的方式来实现。

Q.Spring 框架中的单例 Beans 是线程安全的么？
* Spring 框架并没有对单例 bean 进行任何多线程的封装处理。关于单例 bean 的线程安全和并发问题需要开发者自行去搞定。但实际上，大部分的 Spring bean 并没有可变的状态(比如 Serview 类 和 DAO 类)，所以在某种程度上说 Spring 的单例 bean 是线程安全的。如果你的 bean 有多种状态的话（比如 View Model 对象），就需要自行保证线程安全。
* 最浅显的解决办法就是将多态 bean 的作用域由“singleton”变更为“prototype”。


Q.请举例说明如何在 Spring 中注入一个 Java Collection？
* Spring 提供了以下四种集合类的配置元素：
  - &#60;list&#62; : 该标签用来装配可重复的 list 值。
  - &#60;set> : 该标签用来装配没有重复的 set 值。
  - &#60;map>: 该标签可用来注入键和值可以为任何类型的键值对。
  - &#60;props> : 该标签支持注入键和值都是字符串类型的键值对。


Q.如何向 Spring Bean 中注入一个 Java.util.Properties？
* 第一种方法是使用如下面代码所示的&#60;props&#62; 标签：
* 也可用”util:”命名空间来从 properties 文件中创建出一个 properties bean，然后利用 setter 方法注入 bean 的引用。


Q.请解释 Spring Bean 的自动装配？
* 在 Spring 框架中，在配置文件中设定 bean 的依赖关系是一个很好的机制，Spring 容器还可以自动装配合作关系 bean 之间的关联关系。这意味着 Spring 可以通过向 Bean Factory 中注入的方式自动搞定 bean 之间的依赖关系。自动装配可以设置在每个 bean 上，也可以设定在特定的 bean上。
* 除了 bean 配置文件中提供的自动装配模式，还可以使用@Autowired 注解来自动装配指定的 bean。在使用@Autowired 注解之前需要在按照如下的配置方式在 Spring 配置文件进行配置才可以使用。
  - &#60;context:annotation-config /&#62; 
  - 也可以通过在配置文件中配置 AutowiredAnnotationBeanPostProcessor 达到相同的效果。

Q.请解释自动装配模式的区别？
* no：这是 Spring 框架的默认设置，在该设置下自动装配是关闭的，开发者需要自行在 bean 定义中用标签明确的设置依赖关系。
* byName：该选项可以根据 bean 名称设置依赖关系。当向一个 bean 中自动装配一个属性时，容器将根据 bean 的名称自动在在配置文件中查询一个匹配的 bean。如果找到的话，就装配这个属性，如果没找到的话就报错。
* byType：该选项可以根据 bean 类型设置依赖关系。当向一个 bean 中自动装配一个属性时，容器将根据 bean 的类型自动在在配置文件中查询一个匹配的 bean。如果找到的话，就装配这个属性，如果没找到的话就报错。
* constructor：构造器的自动装配和 byType 模式类似，但是仅仅适用于与有构造器相同参数的bean，如果在容器中没有找到与构造器参数类型一致的 bean，那么将会抛出异常。
* autodetect：该模式自动探测使用构造器自动装配或者 byType 自动装配。首先，首先会尝试找合适的带参数的构造器，如果找到的话就是用构造器自动装配，如果在 bean 内部没有找到相应的构造器或者是无参构造器，容器就会自动选择 byTpe 的自动装配方式。


Q.请举例解释@Autowired 注解？
* @Autowired 注解对自动装配何时何处被实现提供了更多细粒度的控制。@Autowired 注解被用于在 bean 的设值方法上自动装配 bean的属性，一个参数或者带有任意名称或带有多个参数的方法。
* 比如，可以在设值方法上使用@Autowired 注解来替代配置文件中的 <property>元素。当 Spring 容器在setter 方法上找到@Autowired 注解时，会尝试用 byType自动装配。
* 当然我们也可以在构造方法上使用@Autowired 注解。带有@Autowired 注解的构造方法意味着在创建一个 bean 时将会被自动装配，即便在配置文件中使用<constructor-arg> 元素。

Q.请举例说明@Qualifier 注解？


Q.构造方法注入和设值注入有什么区别？
* 在设值注入方法支持大部分的依赖注入，如果我们仅需要注入 int、string 和 long 型的变量，我们不要用设值的方法注入。对于基本类型，如果我们没有注入的话，可以为基本类型设置默认值。在构造方法 注入不支持大部分的依赖注入，因为在调用构造方法中必须传入正确的构造参数，否则的话为报错。
* 设值注入不会重写构造方法的值。如果我们对同一个变量同时使用了构造方法注入又使用了设置方法注入的话，那么构造方法将不能覆盖由设值方法注入的值。很明显，因为构造方法尽在对象被创建时调用。
* 在使用设值注入时有可能还不能保证某种依赖是否已经被注入，也就是说这时对象的依赖关系有可能是不完整的。而在另一种情况下，构造器注入则不允许生成依赖关系不完整的对象。
* 在设值注入时如果对象 A 和对象 B 互相依赖，在创建对象 A 时 Spring 会抛出ObjectCurrentlyInCreationException 异常，因为在 B 对象被创建之前 A 对象是不能被创建的，反之亦然。所以 Spring 用设值注入的方法解决了循环依赖的问题，因对象的设值方法是在对象被创建之前被调用的。

Q.Spring 框架中有哪些不同类型的事件？
* Spring 的 ApplicationContext 提供了支持事件和代码中监听器的功能。我们可以创建 bean 用来监听在 ApplicationContext 中发布的事件。ApplicationEvent类和在 ApplicationContext 接口中处理的事件，如果一个 bean 实现了ApplicationListener 接口，当一个 ApplicationEvent 被发布以后，bean 会自动被通
知。
* Spring 提供了以下 5 中标准的事件：
  - 上下文更新事件（ContextRefreshedEvent）：该事件会在 ApplicationContext 被初始化或者更新时发布。也可以在调用 ConfigurableApplicationContext 接口中的 refresh()方法时被触发。
  - 上下文开始事件（ContextStartedEvent）：当容器调用ConfigurableApplicationContext的Start()方法开始/重新开始容器时触发该事件。
  - 上下文停止事件（ContextStoppedEvent）：当容器调用ConfigurableApplicationContext的Stop()方法停止容器时触发该事件。
  - 上下文关闭事件（ContextClosedEvent）：当 ApplicationContext 被关闭时触发该事件。容器被关闭时，其管理的所有单例 Bean 都被销毁。
  - 请求处理事件（RequestHandledEvent）：在 Web 应用中，当一个 http 请求（request）结束触发该事件。
* 除了上面介绍的事件以外，还可以通过扩展 ApplicationEvent 类来开发自定义的事件。
  为了监听这个事件，还需要创建一个监听器。
  之后通过 applicationContext 接口的 publishEvent()方法来发布自定义事件。

Q.FileSystemResource 和 ClassPathResource 有何区别？
* 在 FileSystemResource 中需要给出 spring-config.xml 文件在你项目中的相对路径或者绝对路径。在 ClassPathResource 中 spring 会在 ClassPath 中自动搜寻配置文件，所以要把ClassPathResource 文件放在 ClassPath 下。
* 如果将 spring-config.xml 保存在了 src 文件夹下的话，只需给出配置文件的名称即可，因为src 文件夹是默认。
* 简而言之，ClassPathResource 在环境变量中读取配置文件，FileSystemResource 在配置文件中读取配置文件。


Q.Spring 框架中都用到了哪些设计模式？
* 代理模式—在 AOP 和 remoting 中被用的比较多。
* 单例模式—在 spring 配置文件中定义的 bean 默认为单例模式。
* 模板方法—用来解决代码重复的问题。比如. RestTemplate, JmsTemplate, JpaTemplate。 
* 端控制器—Spring 提供了 DispatcherServlet 来对请求进行分发。
* 视图帮助(View Helper )—Spring 提供了一系列的 JSP 标签，高效宏来辅助将分散的代码整合在视图里。
* 依赖注入—贯穿于 BeanFactory / ApplicationContext 接口的核心理念。
* 工厂模式—BeanFactory 用来创建对象的实例

Q.Spring 的单例实现原理
* Spring 对 Bean 实例的创建是采用单例注册表的方式进行实现的，而这个注册表的缓存是ConcurrentHashMap 对象。

Q.在 Spring 中如何配置 Bean ?
* 通过全类名（反射）
* 通过工厂方法（静态工厂方法 & 实 例工厂方法）
* FactoryBean

Q.IOC 容器对 Bean 的生命周期
* 通过构造器或工厂方法创建 Bean 实例
* 为 Bean 的属性设置值和对其他 Bean 的引用
* 将 Bean 实 例 传 递 给 Bean 后 置 处 理 器 的 postProcessBeforeInitialization 方 法
* 调用 Bean 的初始化方法(init-method)
* 将 Bean 实 例 传 递 给 Bean 后 置 处 理 器 的 postProcessAfterInitialization 方法
* Bean 可以使用了
* 当容器关闭时, 调用 Bean 的销毁方法(destroy-method)

# 2.AOP

Q.Spring AOP 实现原理
* Spring AOP 中的动态代理主要有两种方式，JDK 动态代理和 CGLIB 动态代理。JDK 动态代理通过反射来接收被代理的类，并且要求被代理的类必须实现一个接口。JDK 动态代理的核心是 InvocationHandler 接口和 Proxy 类。
* 如果目标类没有实现接口，那么 Spring AOP 会选择使用 CGLIB 来动态代理目标类。CGLIB（Code Generation Library），是一个代码生成的类库，可以在运行时动态的生成某个类的子类，注意，CGLIB 是通过继承的方式做的动态代理，因此如果某个类被标记为 final，那么它是无法使用 CGLIB 做动态代理的。动态代理（cglib 与 JDK） 
* JDK 动态代理类和委托类需要都实现同一个接口。也就是说只有实现了某个接口的类可以使用 Java 动态代理机制。但是，事实上使用中并不是遇到的所有类都会给你实现一个接口。因此，对于没有实现接口的类，就不能使用该机制。而 CGLIB 则可以实现对类的动态代理。




# 3.事务
Q.Spring 事务实现方式
* 1、编码方式
  所谓编程式事务指的是通过编码方式实现事务，即类似于 JDBC 编程实现事务管理。
* 2、声明式事务管理方式
  声明式事务管理又有两种实现方式：基于 xml 配置文件的方式；另一个实在业务方法上进行@Transaction 注解，将事务规则应用到业务逻辑中

Q.Spring 事务底层原理
* a、划分处理单元——IOC
  由于 spring 解决的问题是对单个数据库进行局部事务处理的，具体的实现首相用 spring中的 IOC 划分了事务处理单元。并且将对事务的各种配置放到了 ioc 容器中（设置事务管理器，设置事务的传播特性及隔离机制）。
* b、AOP 拦截需要进行事务处理的类
  Spring 事务处理模块是通过 AOP 功能来实现声明式事务处理的，具体操作（比如事务实行的配置和读取，事务对象的抽象），用 TransactionProxyFactoryBean 接口来使用 AOP功能，生成 proxy 代理对象，通过 TransactionInterceptor 完成对代理方法的拦截，将事务处理的功能编织到拦截的方法中。读取 ioc 容器事务配置属性，转化为 spring 事务处理需要的内部数据结构（TransactionAttributeSourceAdvisor），转化为TransactionAttribute 表示的数据对象。
* c、对事物处理实现（事务的生成、提交、回滚、挂起）
  spring 委托给具体的事务处理器实现。实现了一个抽象和适配。适配的具体事务处理器：DataSource 数据源支持、hibernate 数据源事务处理支持、JDO 数据源事务处理支持，JPA、JTA 数据源事务处理支持。这些支持都是通过设计PlatformTransactionManager、AbstractPlatforTransaction 一系列事务处理的支持。 为常用数据源支持提供了一系列的 TransactionManager。 
* d、结合
  PlatformTransactionManager 实现了 TransactionInterception 接口，让其与TransactionProxyFactoryBean 结合起来，形成一个 Spring 声明式事务处理的设计体系。



# 4.与其他模块的整合

## 4.1 与Mybatis整合

## 4.2 与SpringMVC整合

## 4.3 与Tomcat整合




