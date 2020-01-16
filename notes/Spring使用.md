
# 0.概要

* Spring其实指的是Spring Framework（spring 框架）。
* Spring框架的核心就是IoC（控制反转）和AOP（面向切面编程）。
  - IoC简单理解就是控制对象创建的角色由程序员反转为Spring IoC容器。IoC（核心中的核心）：Inverse of Control，控制反转。对象的创建权力由程序反转给Spring框架。
  - AOP简单理解就是针对目标对象进行动态代理，横向增强JavaBean的功能。AOP：Aspect Oriented Programming，面向切面编程。在不修改目标对象的源代码情况下，增强IoC容器中Bean的功能。
  - DI：Dependency Injection，依赖注入。在Spring框架负责创建Bean对象时，动态的将依赖对象注入到Bean组件中
  - Spring容器：指的就是IoC容器。


<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/spring架构.png" w****idth="620px" > </div><br>


* Spring IoC容器本质上就是创建类的实例的工厂，并且对类的实例进行管理。
* Spring IoC容器需要通过Bean工厂来实现，在Spring框架中，主要有两个工厂接口：BeanFactory接口和ApplicationContext接口（实现了BeanFactory接口）。
  - 其中BeanFactory接口是Spring早期创建Bean对象的工厂接口。
  - 而我们现在大多数是通过ApplicationContext接口进行Bean工厂的创建。
* Spring IoC 容器加载Bean信息的方式有XML配置方式和注解方式。
  - XML配置方式：bean标签
  - 注解方式：@Component、@Controller、@Service、@Repository，需要使用context:component-scan标签配合使用。
* Spring IoC容器的创建方式主要有两种场景：在Java Application中创建（jar包）和在Web Application（war包）中创建（重点）。
  - 在Java Application中创建Spring IoC容器主要是通过ApplicationContext接口的两个实现类来完成的：ClassPathXmlApplicationContext            和FileSystemXmlApplicationContext           。
  - 在Web Application中创建Spring IoC容器主要是通过ApplicationContext接口的子接口WebApplicationContext来实现的。
    * WebApplicationContext是通过ContextLoaderListener（实现ServletContextListener接口）创建之后，放入ServletContext域对象中的。

* Spring DI（依赖注入）是基于IoC使用的。简单理解就是Bean工厂在生成Bean对象的时候，如果Bean对象需要装配一个属性，那么就会通过DI将属性值注入给对象的属性。
  - 依赖注入的方式主要有构造方法注入（了解）和set方法注入（重点）。
  - set方法注入又分为手动装配方式注入和自动装配方式注入。
    * 手动装配方式（XML方式）：bean标签的子标签property，需要在类中指定set方法。
    * 自动装配方式（注解方式）：@Autowired注解、@Resource注解。
      - @Autowired：一部分功能是查找实例，从spring容器中根据类型（java类）获取对应的实例。另一部分功能就是赋值，将找到的实例，装配给另一个实例的属性值。（注意事项：一个java类型在同一个spring容器中，只能有一个实例）
      - @Resource：一部分功能是查找实例，从spring容器中根据Bean的名称（bean标签的名称）获取对应的实例。另一部分功能就是赋值，将找到的实例，装配给另一个实例的属性值。
* Spring AOP实现原理是什么？
  - 动态代理技术（反射）：基于JDK的动态代理和使用CGLib的动态代理。
  - 动态代理方式选择：根据是否实现接口来选择哪种代理方式。
* Spring 基于AspectJ的AOP开发



<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/Spring专题.png" width="720px" > </div><br>

# 1.组件添加到IOC容器


## 获取容器的方式

* XML-Java应用（ClassPathXmlApplicationContext）

```
    @Test
    public void testLoadXml() {
        ApplicationContext ac = new ClassPathXmlApplicationContext("classpath:software/spring/xml/beans.xml");
        Person person = (Person) ac.getBean("person");
        System.out.println(person);
    }
```

* XML-Web应用

```
<web-app>
    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath:spring/spring-*.xml</param-value>
    </context-param>
    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>

    ...
<web-app>
```

另一种省略了很多，只是在DispatcherServlet中配置了contextConfigLocation：

```
	<servlet>
		<servlet-name>spring-dispatcher</servlet-name>
		<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
		<init-param>
			<param-name>contextConfigLocation</param-name>
			<param-value>classpath:spring/spring-*.xml</param-value>
		</init-param>
	</servlet>
```

* 注解-Java应用（AnnotationConfigApplicationContext）

```
ApplicationContext context = new AnnotationConfigApplicationContext(SpringConfiguration.class);
UserService service = context.getBean(UserService.class);
service.saveUser();
```

* 注解-Web应用（AnnotationConfigWebApplicationContext）

```
<web-app>
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

    ...
<web-app>
```

## Spring 分模块开发

* 分模块开发的场景描述：
  - 表现层：spring配置文件，只想管理表现层的Bean
  - 业务层：spring配置文件，只想管理业务层的Bean，并且进行事务控制
  - 持久层：spring配置文件，只想管理持久层的Bean，并且还有需要管理数据源的Bean
  - 缓存
* 为了方便管理项目中不同层的Bean对象，一般都是将一个spring配置文件，分解为多个spring配置文件。
* 分解之后的spring配置文件如何一起被加载呢？？
  - 一种就是同时指定多个配置文件的地址一起加载
  - 另一种就是：定义一个import.xml文件，通过import标签将其他多个spring配置文件导入到该文件中，tomcat启动时只需要加载import.xml就可以

spring-web.xml

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

spring-service.xml

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:tx="http://www.springframework.org/schema/tx"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/context
       http://www.springframework.org/schema/context/spring-context.xsd
       http://www.springframework.org/schema/tx
       http://www.springframework.org/schema/tx/spring-tx.xsd">

    <!-- 扫描service包下所有使用注解的类型 -->
    <context:component-scan base-package="com.imooc.o2o.service" />

    <!-- 配置事务管理器 -->
    <bean id="transactionManager"
          class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <!-- 注入数据库连接池 -->
        <property name="dataSource" ref="dataSource" />
    </bean>

    <!-- 配置基于注解的声明式事务 -->
    <tx:annotation-driven transaction-manager="transactionManager" />
</beans>
```

spring-dao.xml

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/context
       http://www.springframework.org/schema/context/spring-context.xsd">
    <!-- 配置整合mybatis过程 -->
    <!-- 1.配置数据库相关参数properties的属性：${url} -->
    <!--<context:property-placeholder location="classpath:jdbc.properties"/>-->
    <bean class="com.imooc.o2o.util.EncryptPropertyPlaceholderConfigurer">
        <property name="locations">
            <list>
                <!--需要解密的文件都可以放到这里-->
                <value>classpath:jdbc.properties</value>
                <value>classpath:redis.properties</value>
                <value>classpath:application.properties</value>
            </list>
        </property>
        <property name="fileEncoding" value="UTF-8"/>
    </bean>

    <!-- 2.数据库连接池 -->
    <bean id="abstractDataSource" abstract="true"
          class="com.mchange.v2.c3p0.ComboPooledDataSource" destroy-method="close">
        <!-- c3p0连接池的私有属性 -->
        <property name="maxPoolSize" value="30"/>
        <property name="minPoolSize" value="10"/>
        <property name="initialPoolSize" value="10"/>

        <!-- 关闭连接后不自动commit -->
        <property name="autoCommitOnClose" value="false"/>
        <!-- 获取连接超时时间 -->
        <property name="checkoutTimeout" value="10000"/>
        <!-- 当获取连接失败重试次数 -->
        <property name="acquireRetryAttempts" value="2"/>
    </bean>

    <bean id="master" parent="abstractDataSource">
        <!-- 配置连接池属性 -->
        <property name="driverClass" value="${jdbc.driver}" />
        <property name="jdbcUrl" value="${jdbc.master.url}" />
        <property name="user" value="${jdbc.username}" />
        <property name="password" value="${jdbc.password}" />
    </bean>

    <bean id="slave" parent="abstractDataSource">
        <!-- 配置连接池属性 -->
        <property name="driverClass" value="${jdbc.driver}" />
        <property name="jdbcUrl" value="${jdbc.slave.url}" />
        <property name="user" value="${jdbc.username}" />
        <property name="password" value="${jdbc.password}" />
    </bean>

    <!--配置动态数据源，targetDataSources就是路由数据源所对应的名称-->
    <bean id="dynamicDataSource" class="com.imooc.o2o.dao.split.DynamicDataSource">
        <property name="targetDataSources">
            <map>
                <entry value-ref="master" key="master"></entry>
                <entry value-ref="slave" key="slave"></entry>
            </map>
        </property>
    </bean>

    <!--将dynamicDataSource放入懒加载中（程序执行时才决定使用哪个dataSource）-->
    <bean id="dataSource"
          class="org.springframework.jdbc.datasource.LazyConnectionDataSourceProxy">
        <property name="targetDataSource">
            <ref bean="dynamicDataSource"/>
        </property>
    </bean>

    <!-- 3.配置SqlSessionFactory对象 -->
    <bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
        <!-- 注入数据库连接池 -->
        <property name="dataSource" ref="dataSource"/>
        <!-- 配置MyBaties全局配置文件:mybatis-config.xml -->
        <property name="configLocation" value="classpath:mybatis-config.xml"/>
        <!-- 扫描entity包 使用别名 -->
        <property name="typeAliasesPackage" value="com.imooc.entity"/>
        <!-- 扫描sql配置文件:mapper需要的xml文件 -->
        <property name="mapperLocations" value="classpath:mapper/*.xml"/>
    </bean>

    <!-- 4.配置扫描Dao接口包，动态实现Dao接口，注入到spring容器中 -->
    <bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
        <!-- 注入sqlSessionFactory -->
        <property name="sqlSessionFactoryBeanName" value="sqlSessionFactory"/>
        <!-- 给出需要扫描Dao接口包 -->
        <property name="basePackage" value="com.imooc.o2o.dao"/>
    </bean>
</beans>
```

最后是spring-redis.xml

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
    http://www.springframework.org/schema/beans/spring-beans.xsd
    http://www.springframework.org/schema/context
    http://www.springframework.org/schema/context/spring-context.xsd">
    <!-- Redis连接池的设置 -->
    <bean id="jedisPoolConfig" class="redis.clients.jedis.JedisPoolConfig">
        <!-- 控制一个pool可分配多少个jedis实例 -->
        <property name="maxTotal" value="${redis.pool.maxActive}" />
        <!-- 连接池中最多可空闲maxIdle个连接 ，这里取值为20，表示即使没有数据库连接时依然可以保持20空闲的连接，而不被清除，随时处于待命状态。 -->
        <property name="maxIdle" value="${redis.pool.maxIdle}" />
        <!-- 最大等待时间:当没有可用连接时,连接池等待连接被归还的最大时间(以毫秒计数),超过时间则抛出异常 -->
        <property name="maxWaitMillis" value="${redis.pool.maxWait}" />
        <!-- 在获取连接的时候检查有效性 -->
        <property name="testOnBorrow" value="${redis.pool.testOnBorrow}" />
    </bean>

    <!-- 创建Redis连接池，并做相关配置 -->
    <bean id="jedisWritePool" class="com.imooc.o2o.cache.JedisPoolWriper"
          depends-on="jedisPoolConfig">
        <constructor-arg index="0" ref="jedisPoolConfig" />
        <constructor-arg index="1" value="${redis.hostname}" />
        <constructor-arg index="2" value="${redis.port}" type="int" />
        <constructor-arg index="3" value="${redis.timeout}" type="int" />
        <constructor-arg index="4" value="${redis.password}"/>
    </bean>

    <!-- 创建Redis工具类，封装好Redis的连接以进行相关的操作 -->
    <bean id="jedisUtil" class="com.imooc.o2o.cache.JedisUtil" scope="singleton">
        <property name="jedisPool">
            <ref bean="jedisWritePool" />
        </property>
    </bean>
    <!-- Redis的key操作 -->
    <bean id="jedisKeys" class="com.imooc.o2o.cache.JedisUtil$Keys"
          scope="singleton">
        <constructor-arg ref="jedisUtil"></constructor-arg>
    </bean>
    <!-- Redis的Strings操作 -->
    <bean id="jedisStrings" class="com.imooc.o2o.cache.JedisUtil$Strings"
          scope="singleton">
        <constructor-arg ref="jedisUtil"></constructor-arg>
    </bean>
</beans>
```

在web.xml中配置所有spring-*.xml配置文件

```
	<servlet>
		<servlet-name>spring-dispatcher</servlet-name>
		<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
		<init-param>
			<param-name>contextConfigLocation</param-name>
			<param-value>classpath:spring/spring-*.xml</param-value>
		</init-param>
	</servlet>
```



## 给容器中注册组件的方式
* 1）@Bean: [导入第三方的类或包的组件],比如Person为第三方的类, 需要在我们的IOC容器中使用
* 2）包扫描+组件的标注注解(@ComponentScan:  @Controller, @Service  @Reponsitory  @ Componet),一般是针对 我们自己写的类,使用这个
* 3）@Import:[快速给容器导入一个组件] 注意:@Bean有点简单
  -  3-1）@Import(要导入到容器中的组件):容器会自动注册这个组件,bean 的 id为全类名
  -  3-2）ImportSelector:是一个接口,返回需要导入到容器的组件的全类名数组
  -  3-3）ImportBeanDefinitionRegistrar:可以手动添加组件到IOC容器, 所有Bean的注册可以使用BeanDifinitionRegistry
* 4）使用Spring提供的FactoryBean(工厂bean)进行注册

## 基于beans.xml配置文件

* bean标签作用：
  - 用于配置对象让 spring 来创建的。
  - 默认情况下它调用的是类中的无参构造函数。如果没有无参构造函数则不能创建成功。
* bean标签属性：
  - id：给对象在容器中提供一个唯一标识。用于获取对象。
  - class：指定类的全限定类名。用于反射创建对象。默认情况下调用无参构造函数。
  - scope：指定对象的作用范围。
    * singleton :默认值，单例的（在整个容器中只有一个对象）.
    * prototype :多例的.
    * request	:WEB 项目中,Spring 创建一个 Bean 的对象,将对象存入到 request 域中.
    * session	:WEB 项目中,Spring 创建一个 Bean 的对象,将对象存入到 session 域中.
    * global session	:WEB 项目中,应用在 Portlet 环境.如果没有 Portlet 环境那么globalSession 相当于 session.
  - init-method：指定类中的初始化方法名称。
  - destroy-method：指定类中销毁方法名称。比如DataSource的配置中一般需要指定destroy-method=“close”。
* bean的作用范围：
  - 单例对象：scope="singleton"
    * 一个应用只有一个对象的实例。它的作用范围就是整个引用。
    * 生命周期：
      - 对象出生：当应用加载，创建容器时，对象就被创建了。
      - 对象活着：只要容器在，对象一直活着。
      - 对象死亡：当应用卸载，销毁容器时，对象就被销毁了。
  - 多例对象：scope="prototype"
    * 每次访问对象时，都会重新创建对象实例。
    * 生命周期：
      - 对象出生：当使用对象时，创建新的对象实例。
      - 对象活着：只要对象在使用中，就一直活着。
      - 对象死亡：当对象长时间不用时，被 java 的垃圾回收器回收了。


```
    <bean id="person" class="software.spring.xml.Person">
        <property name="name" value="wang"></property>
        <property name="age" value="20"></property>
    </bean>
```

解析配置文件：

```
    @Test
    public void testLoadXml() {
        ApplicationContext ac = new ClassPathXmlApplicationContext("classpath:software/spring/xml/beans.xml");
        Person person = (Person) ac.getBean("person");
        System.out.println(person);
    }
```



## 基于注解@Configuration

```
@Configuration
public class MainConfig {
    @Bean("aPerson")
    public Person person() {
        return new Person("wang", 20);
    }
}
```

解析配置类：

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

## ComponetScan扫描规则

在xml配置文件中配置component-scan，然后就可以扫描注解@Component，或者它的衍生注解@Controller、@Service、@Repository标注的类：

```
<context:component-scan base-package="com.imooc.o2o.service" />

```

使用注解@ComponentScan：

* 相当于context:component-scan标签
* 组件扫描器，扫描@Component、@Controller、@Service、@Repository注解的类
* 该注解是编写在类上面的，一般配合@Configuration注解一起使用
* basePackages：用于指定要扫描的包
* value：和basePackages作用一样
* excludeFilters = Filter[] 指定扫描的时候按照什么规则排除那些组件
* includeFilters = Filter[] 指定扫描的时候只需要包含哪些组件
  - useDefaultFilters = false 默认是true,扫描所有组件，要改成false
* 类型
  - FilterType.ANNOTATION：按照注解
  - FilterType.ASSIGNABLE_TYPE：按照给定的类型；比如按BookService类型
  - FilterType.ASPECTJ：使用ASPECTJ表达式
  - FilterType.REGEX：使用正则指定
  - FilterType.CUSTOM：使用自定义规则，自已写类，实现TypeFilter接口

```
@Configuration
@ComponentScan(value = "software.spring.componentscan", includeFilters = {
        @ComponentScan.Filter(type= FilterType.CUSTOM, classes = {ClassTypeFilter.class})
}, useDefaultFilters = false)
public class MainConfig2 {
    @Bean
    public Person person() {
        return new Person("wang", 20);
    }
}
```


## @Import

当系统中有多个配置类时怎么办呢？
* spring配置文件中的import标签
* 如果是注解时，使用@Import来组合多个配置类


@Import
* 用来组合多个配置类
* 相当于spring配置文件中的import标签
* 在引入其他配置类时，可以不用再写@Configuration 注解。当然，写上也没问
题。
* 属性value：用来指定其他配置类的字节码文件

除了可以引入其他配置文件，还可以引入bean、ImportSelector和ImportBeanDefinitionRegistrar
* ImportSelector可以批量导入组件的全类名数组
* 通过ImportBeanDefinitionRegistrar自定义注册,向容器中注册bean

```
@Configuration
@Import(value = {Dog.class, Cat.class, WangImportSelector.class, WangImportBeanDefinitionRegistrar.class})
public class MainConfig {
    @Bean("wang")
    public Person person() {
        return new Person("wang", 20);
    }

    @Bean("monkey1")
    public  WangFactoryBean wangFactoryBean() {
        return new WangFactoryBean();
    }

    @Bean("monkey2")
    public  WangFactoryBean wangFactoryBean2() {
        return new WangFactoryBean();
    }
}
```

```
public class WangImportSelector implements ImportSelector {
    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        return new String[]{"software.spring.createbean.bean.Fish",
                "software.spring.createbean.bean.Tiger"};
    }
}

```

```
public class WangImportBeanDefinitionRegistrar implements ImportBeanDefinitionRegistrar {
    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        boolean containsDog = registry.containsBeanDefinition("software.spring.createbean.bean.Dog");
        boolean containsCat = registry.containsBeanDefinition("software.spring.createbean.bean.Cat");
        if (containsCat && containsDog) {
            RootBeanDefinition beanDefinition = new RootBeanDefinition(Pig.class);
            registry.registerBeanDefinition("pig", beanDefinition);
        }
    }
}
```


## @PropertySource

* 加载properties配置文件
* 相当于context:property-placeholder标签
  - value属性：用于指定properties文件路径，如果在类路径下，需要写上classpath


```
@Configuration
@PropertySource(“classpath:jdbc.properties”)
public class JdbcConfig {
  @Value("${jdbc.driver}")
  private String driver;

  @Value("${jdbc.url}")
  private String url;

  @Value("${jdbc.username}")
  private String username;

  @Value("${jdbc.password}")
  private String password;

  /**
  *创建一个数据源，并存入 spring 容器中
  *@return
  */
  @Bean(name="dataSource")
  public DataSource createDataSource() {
    try {
      ComboPooledDataSource ds = new ComboPooledDataSource(); 
      ds.setDriverClass(driver);
      ds.setJdbcUrl(url); 
      ds.setUser(username); 
      ds.setPassword(password);
      return ds;
    } catch (Exception e) {
      throw new RuntimeException(e);
    }
  }
}
```



## 实例化bean的三种方式

* 使用默认无参构造函数
  - 在默认情况下：它会根据默认无参构造函数来创建类对象。
  - 如果 bean中没有默认无参构造函数，将会创建失败。

```
<bean id="userService" class="com.kkb.spring.service.UserServiceImpl"/>
```

* 静态工厂

```
public class StaticFactory {
  public static UserService createUserService(){
    return new UserServiceImpl();
  }
}
```

```
<!-- 此种方式是:
使用 StaticFactory 类中的静态方法 createUserService 创建对象，并存入 spring 容器
	id 属性：指定 bean 的 id，用于从容器中获取
	class 属性：指定静态工厂的全限定类名
	factory-method 属性：指定生产对象的静态方法
-->
<bean id="userService" class="com.kkb.spring.factory.StaticFactory" factory-method="createUserService"></bean>
```

* 实例工厂

```
public class InstanceFactory {
  public UserService createUserService(){
    return new UserServiceImpl();
  }
}
```

```
<!-- 此种方式是：
	先把工厂的创建交给 spring 来管理。
	然后在使用工厂的 bean 来调用里面的方法
		factory-bean 属性：用于指定实例工厂 bean 的 id。
		factory-method 属性：用于指定实例工厂中创建对象的方法。
-->
<bean id="instancFactory" class="com.kkb.factory.InstanceFactory"></bean>

<bean id="userService" factory-bean="instancFactory" factory-method="createUserService"></bean>
```


## 实例化bean的注解方式@Component

* @Component注解
  - 把资源让 spring 来管理。相当于在 xml 中配置一个 bean
  - value：指定 bean 的 id。如果不指定 value 属性，默认 bean 的 id 是当前类的类名。首字母小写
* @Controller、@Service、@Repository注解
  - 他们三个注解都是针对@Component的衍生注解
  - 他们的作用及属性都是一模一样的。他们只不过是提供了更加明确的语义化。
    * @Controller：一般用于表现层的注解。
	* @Service：一般用于业务层的注解。
    * @Repository：一般用于持久层的注解。
* 细节：如果注解中有且只有一个属性要赋值时，且名称是 value，value 在赋值是可以不写。


## 实例化bean的注解方式@Bean

* @Bean标注在方法上(返回某个实例的方法)，等价于spring配置文件中的<bean>
* 主要用来配置非自定义的bean，比如DruidDataSource、SqlSessionFactory

## scope

* prototype: 多实例：IOC容器启动并不会去调用方法创建对象放在容器中，而是每次获取的时候才会调用方法创建对象
* singleton: 单实例（默认）：IOC容器启动会调用方法创建对象放到IOC容器中以后每交获取就是直接从容器（理解成从map.get对象）中拿  
* request:  主要针对WEB应用，同一次请求创建一个实例
* session:  同一个session创建一个实例（后面两个用得不多，了解即可）

## Lazy

针对单实例bean，默认情况单实例会在容器启动时创建对象。懒加载会在第一次使用bean时才进行创建和初始化。

## @Conditional

使用@Conditional，容器可以根据条件选择性添加bean。



## FactoryBean

通过工厂方式获取bean。其注册的bean不是Factory，而是FactoryBean.getObject()创建的bean。

```
public class WangFactoryBean implements FactoryBean<Monkey> {
    @Override
    public Monkey getObject() throws Exception {
        return new Monkey();
    }

    @Override
    public Class<?> getObjectType() {
        return Monkey.class;
    }

    @Override
    public boolean isSingleton() {
        return true;
    }
}
```

# 2.bean的生命周期

* bean的生命周期:指bean创建、初始化到销毁的过程
* bean的生命周期是由容器进行管理的
* 可以自定义bean初始化和销毁方法: 容器在bean进行到当前生命周期的时候, 来调用自定义的初始化和销毁方法

## 指定初始化和销毁方法

```
    @Bean(initMethod = "init", destroyMethod = "destroy")
    public Bike bike() {
        return new Bike();
    }
```

这个适用于单实例。

多实例: 容器只负责初始化,但不会管理bean, 容器关闭不会调用销毁方法

## 让Bean实现 InitializingBean 和 DisposableBean接口

* InitializingBean(定义初始化逻辑,可点进去看此类)
  - afterPropertiesSet()方法:当beanFactory创建好对象,且把bean所有属性设置好之后,会调这个方法,相当于初始化方法
* DisposableBean(定义销毁逻辑,可点进去看此类)
  - destory()方法,当bean销毁时,会把单实例bean进行销毁


## JSR250规则

* @PostConstruct: 在Bean创建完成,且属于赋值完成后进行初始化,属于JDK规范的注解
* @PreDestroy: 在bean将被移除之前进行通知, 在容器销毁之前进行清理工作


## BeanPostProcessor也可以起到类似作用

BeanPostProcessor类在在bean初始化前后进行一些处理工作
* postProcessBeforeInitialization():在初始化之前进行后置处理工作(在init-method之前),
任何初始化方法调用之前(比如在InitializingBean的afterPropertiesSet初始化之前,或自定义init-method调用之前使用)        
* postProcessAfterInitialization():在初始化之后进行后置处理工作


<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/BeanPostProcessor调用栈.jpg" width="620px" > </div><br>

在AbstractAutowireCapableBeanFactory.initializeBean()：

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
			wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
		}

		return wrappedBean;
	}
```

# 3.赋值以及依赖注入DI

* 什么是依赖？
  - 依赖指的就是Bean实例中的属性
  - 属性分为：简单类型（8种基本类型和String类型）的属性、POJO类型的属性、集合数组类型的属性。
* 什么是依赖注入？
  - 依赖注入：Dependency Injection。它是 spring 框架核心 ioc 的具体实现。
* 为什么要进行依赖注入？
  - 我们的程序在编写时，通过控制反转，把对象的创建交给了 spring，但是代码中不可能出现没有依赖的情况。
  - ioc 解耦只是降低他们的依赖关系，但不会消除。例如：我们的业务层仍会调用持久层的方法。那这种业务层和持久层的依赖关系，在使用 spring 之后，就让 spring 来维护了。
  - 简单的说，就是坐等框架把持久层对象传入业务层，而不用我们自己去获取。
* 依赖注入的方式（基于XML）
  - 构造函数注入
    * 顾名思义，就是使用类中的构造函数，给成员变量赋值。
    * 注意，赋值的操作不是我们自己做的，而是通过配置的方式，让 spring 框架来为我们注入。
    * 涉及的标签：constructor-arg
      - index:指定参数在构造函数参数列表的索引位置
      - name:指定参数在构造函数中的名称
      - value:它能赋的值是基本数据类型和 String 类型
      - ref:它能赋的值是其他 bean 类型，也就是说，必须得是在配置文件中配置过的 bean
  - set方法注入
    * 手动装配方式（XML方式）
      - bean标签的子标签property，需要在类中指定set方法
    * 自动装配方式（注解方式）
      - @Autowired：一部分功能是查找实例，从spring容器中根据类型（java类）获取对应的实例。另一部分功能就是赋值，将找到的实例，装配给另一个实例的属性值。（注意事项：一个java类型在同一个spring容器中，只能有一个实例）
      - Resource：一部分功能是查找实例，从spring容器中根据Bean的名称（bean标签的名称）获取对应的实例。另一部分功能就是赋值，将找到的实例，装配给另一个实例的属性值。
  - 使用p名称空间注入数据（本质上还是调用set方法）
    * xmlns:p="http://www.springframework.org/schema/p
    * p:属性名 = ""  
    * p:属性名-ref = ""  
* 依赖注入不同类型的属性（基于XML）
  - 简单类型（value）
    * &#60;property name="id" value="1"&#62;&#60;/property&#62;
  - 引用类型（ref）
    * &#60;property name="userDao" ref="userDao"&#62;&#60;/property&#62;
  - 集合类型（数组）
    * 如果是数组或者List集合，注入配置文件的方式是一样的
      - &#60;property name="arrs"&#62;&#60;list&#62;&#60;/list&#62;&#60;/property&#62;
      - 如果集合内是简单类型，使用value子标签，如果是POJO类型，则使用bean标签
    * 如果是Set集合
      - &#60;property name="sets"&#62;&#60;set&#62;&#60;/set&#62;&#60;/property&#62;
      - 如果集合内是简单类型，使用value子标签，如果是POJO类型，则使用bean标签
    * 如果是Map集合
      - &#60;property name="map"&#62;&#60;map&#62;&#60;/map&#62;&#60;/property&#62;
      - &#60;entry key="老王2" value="38"/&#62;
    * 如果是Properties集合
      - &#60;property name="pro"&#62;&#60;props&#62;&#60;/props&#62;&#60;/property&#62;
      - &#60;prop key="uname"&#62;root&#60;/prop&#62;



## @Value

使用@Value对属性进行赋值：
* 可以是基本的字符串
* SpringEl表达式
* 配置文件里面的值


## @Autowired @Qualifier @Primary

不管@Autowired是放到属性、参数、set方法还是构造方法, 都是从容器里取到的bean。

@Autowired依赖注入步骤：
* step1.在使用@Autowired时，首先在容器中查询对应类型的bean（默认按类型进行查找）
* step2.如果查询结果刚好为一个，就将该bean装配给@Autowired指定的数据
* step3.如果查询的结果不止一个，那么@Autowired会根据名称来查找。
* step4.如果查询的结果为空，那么会抛出异常。解决方法时，使用required=false

想使用名称装配可以结合@Qualifier注解进行使用。

@Qualifier
* 在自动按照类型注入的基础之上，再按照 Bean 的 id 注入。
* 它在给字段注入时不能独立使用，必须和@Autowire 一起使用；但是给方法参数注入时，可以独立使用。


@Primary和@Qualifier
* @Primary 优先方案，被注解的实现，优先被注入
* @Qualifier   先声明后使用，相当于多个实现起多个不同的名字，注入时候告诉我你要注入哪个。和@Primary一起使用时，优先选择@Qualifier指定的
* @Primary：和@Qualifier 一样，使用场景经常是：在spring 中使用注解，常使用@Autowired， 默认是根据类型Type来自动注入的。但有些特殊情况，对同一个接口，可能会有几种不同的实现类，而默认只会采取其中一种的情况下 @Primary 的作用就出来了。

## @Resource(JSR250)和@Inject(JSR330) 

@Resource：
* @Resource注解默认根据参数名字寻找bean注入
* 如果没有指定name属性，当注解写在字段上时，默认取字段名进行按照名称查找，当找不到与名称匹配的bean时才按照类型进行装配。
* 但是需要注意的是，如果name属性一旦指定，就只会按照名称进行装配
* @Resource注解不支持spring的@Primary注解优先注入

@Inject：
* 需要依赖javax.inject
* @Inject注解默认是根据参数名去寻找bean注入
* @Inject注解支持spring的@Primary注解优先注入
* @Inject注解可以增加@Named注解指定注入的bean

三种依赖注入对比：
* @Autowired是spring专有注解，@Resource是java中JSR250中的规范，@Inject是java中JSR330中的规范
* @Autowired支持参数required=false，@Resource，@Inject都不支持
* @Autowired，和@Inject支持@Primary注解优先注入，@Resource不支持
* @Autowired通过@Qualifier指定注入特定bean,@Resource可以通过参数name指定注入bean,@Inject需要@Named注解指定注入bean


# 4.Aware获取spring底层组件

自定义组件想要使用Spring容器底层的组件(ApplicationContext, BeanFactory, ......)

思路: 自定义组件实现xxxAware, 在创建对象的时候, 会调用接口规定的方法注入到相关组件:Aware

把Spring底层的组件可以注入到自定义的bean中,ApplicationContextAware是利用ApplicationContextAwareProcessor来处理的, 其它XXXAware也类似, 都有相关的Processor来处理, 其实就是后置处理器来处理。


XXXAware---->功能使用了XXXProcessor来处理的, 这就是后置处理器的作用。


# 5.AOP

## 纵向抽取

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/纵向抽取.png" width="620px" > </div><br>


## 横向抽取

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/横向抽取.png" width="620px" > </div><br>

## AOP作用和优势

AOP:面向切面编程，底层就是动态代理。

AOP作用：
* AOP采取横向抽取机制，补充了传统纵向继承体系（OOP）无法解决的重复性代码优化（性能监视、事务管理、安全检查、缓存）
* 将业务逻辑和系统处理的代码（关闭连接、事务管理、操作日志记录）解耦。

AOP优势：
* 重复性代码被抽取出来之后，维护更加方便


## AOP术语

* Joinpoint(连接点)   -- 所谓连接点是指那些被拦截到的点。在spring中,这些点指的是方法,因为spring只支持方法类型的连接点
* Pointcut(切入点)        -- 所谓切入点是指我们要对哪些Joinpoint进行拦截的定义
* Advice(通知/增强)    -- 所谓通知是指拦截到Joinpoint之后所要做的事情就是通知.通知分为前置通知,后置通知,异常通知,最终通知,环绕通知(切面要完成的功能)
* Introduction(引介) -- 引介是一种特殊的通知在不修改类代码的前提下, Introduction可以在运行期为类动态地添加一些方法或Field
* Target(目标对象)     -- 代理的目标对象
* Weaving(织入)      -- 是指把增强应用到目标对象来创建新的代理对象的过程
* Proxy（代理）       -- 一个类被AOP织入增强后，就产生一个结果代理类
* Aspect(切面)        -- 是切入点和通知的结合
* Advisor（通知器、顾问）		--和Aspect很相似


<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/AOP术语说明.png" width="720px" > </div><br>


## AOP实现之AspectJ

* AspectJ是一个Java实现的AOP框架，它能够对java代码进行AOP编译（一般在编译期进行），让java代码具有AspectJ的AOP功能（当然需要特殊的编译器）
* 可以这样说AspectJ是目前实现AOP框架中最成熟，功能最丰富的语言。更幸运的是，AspectJ与java程序完全兼容，几乎是无缝关联，因此对于有java编程基础的工程师，上手和使用都非常容易。
* 了解AspectJ应用到java代码的过程（这个过程称为织入），对于织入这个概念，可以简单理解为aspect(切面)应用到目标函数(类)的过程。
* 对于织入这个过程，一般分为动态织入和静态织入，动态织入的方式是在运行时动态将要增强的代码织入到目标类中，这样往往是通过动态代理技术完成的，如Java JDK的动态代理(Proxy，底层通过反射实现)或者CGLIB的动态代理(底层通过继承实现)，Spring AOP采用的就是基于运行时增强的代理技术
* ApectJ采用的就是静态织入的方式。ApectJ主要采用的是编译期织入，在这个期间使用AspectJ的acj编译器(类似javac)把aspect类编译成class字节码后，在java目标类编译时织入，即先编译aspect类再编译目标类。


<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/AspectJ.png" width="620px" > </div><br>

## AOP实现之Spring AOP

* Spring AOP是通过动态代理技术实现的
* 而动态代理是基于反射设计的。
* 动态代理技术的实现方式有两种：基于接口的JDK动态代理和基于继承的CGLib动态代理。

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/SpringAOP.png" width="620px" > </div><br>


## AOP三步骤

* step1.将业务逻辑组件和切面类都加入到容器中, 告诉spring哪个是切面类(@Aspect)
* step2.在切面类上的每个通知方法上标注通知注解, 告诉Spring何时运行(写好切入点表达式,参照官方文档)
* step3.开启基于注解的AOP模式  @EableXXXX

```
@Aspect
public class LogAspects {
	@Pointcut("execution(public int software.spring.aop.Calculator.*(..))")
	public void pointCut(){}
	
	//@before代表在目标方法执行前切入, 并指定在哪个方法前切入
	@Before("pointCut()")
	public void logStart(JoinPoint joinPoint){
		System.out.println(joinPoint.getSignature().getName()+"除法运行....参数列表是:{"+Arrays.asList(joinPoint.getArgs())+"}");
	}

	@After("pointCut()")
	public void logEnd(JoinPoint joinPoint){
		System.out.println(joinPoint.getSignature().getName()+"除法结束......");
		
	}

	@AfterReturning(value="pointCut()",returning="result")
	public void logReturn(Object result){
		System.out.println("除法正常返回......运行结果是:{"+result+"}");
	}

	@AfterThrowing(value="pointCut()",throwing="exception")
	public void logException(Exception exception){
		System.out.println("运行异常......异常信息是:{"+exception+"}");
	}
	
	@Around("pointCut()")
	public Object Around(ProceedingJoinPoint proceedingJoinPoint) throws Throwable{
		System.out.println("@Around:执行目标方法之前...");
		Object obj = proceedingJoinPoint.proceed();//相当于开始调div地
		System.out.println("@Around:执行目标方法之后...");
		return obj;
	}
}
```

切入点表达式：
execution([修饰符] 返回值类型 包名.类名.方法名(参数))
* execution：必须要
* 修饰符：可省略
* 返回值类型：必须要，但是可以使用*通配符
* 包名	：
  - 多级包之间使用.分割
  - 包名可以使用*代替，多级包名可以使用多个*代替
  - 如果想省略中间的包名可以使用 ..
* 类名
  - 可以使用*代替
  - 也可以写成*DaoImpl
* 方法名： 
  - 也可以使用*好代替
  - 也可以写成add*
* 参数：
  - 参数使用*代替
  - 如果有多个参数，可以使用 ..代替



配置类：

```
@Configuration
@EnableAspectJAutoProxy
public class MainConfig {
    @Bean
    public Calculator calculator() {
        return new Calculator();
    }

    @Bean
    public LogAspects logAspects() {
        return new LogAspects();
    }
}

```

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/AOP通知.png" width="520px" > </div><br>


通知方法, 分以下五种:
* 前置通知：
  - 执行时机：目标对象方法之前执行通知
  - 配置文件：&#60;aop:before method="before" pointcut-ref="myPointcut"/&#62;
  - 注解：@Before
  - 应用场景：方法开始时可以进行校验
* 后置通知：
  - 执行时机：目标对象方法之后执行通知，有异常则不执行了
  - 配置文件：&#60;aop:after-returning method="afterReturning" pointcut-ref="myPointcut"/&#62;
  - 注解：@AfterReturning
  - 应用场景：可以修改方法的返回值
* 最终通知：
  - 执行时机：目标对象方法之后执行通知，有没有异常都会执行
  - 配置文件：&#60;aop:after method="after" pointcut-ref="myPointcut"/&#62;
  - 注解：@After
  - 应用场景：例如像释放资源
* 环绕通知：
  - 执行时机：目标对象方法之前和之后都会执行。
  - 配置文件：&#60;aop:around method="around" pointcut-ref="myPointcut"/&#62;
  - 注解： @Around
  - 应用场景：事务、统计代码执行时机
* 异常抛出通知：
  - 执行时机：在抛出异常后通知
  - 配置文件：&#60;aop:after-throwing method=" afterThrowing " pointcut- ref="myPointcut"/&#62;
  - 注解： @AfterThrowing
  - 应用场景：包装异常


开启AOP自动代理
* 配置文件：&#60;aop:aspectj-autoproxy/&#62;
* 注解：@EnableAspectJAutoProxy


## 参数传递

假如一个业务组件是有参数的，并且参数要传递给通知，那么怎么做呢？

业务方法：

```
public class TestBiz {
    public void init(String name,int age){
        System.out.println("test init name " + name + ",age " + age);
    }
}
```

定义一个接收参数的环绕通知

```
public class TestAspect {

    public Object aroundinit(ProceedingJoinPoint joinPoint,String name,int age){
        Object obj = null;
        try {
            System.out.println("Aspect before around");
            obj = joinPoint.proceed();
            System.out.println("Aspect around" + name+","+age);
            System.out.println("Aspect after around");
        } catch (Throwable throwable) {
            throwable.printStackTrace();
        }
        return obj;
    }
}
```

在xml中配置

```
    <bean id="testAspect" class="com.mmdet.learn.ssm.testaop.TestAspect"/>
    <bean id="testBiz" class="com.mmdet.learn.ssm.testaop.TestBiz"/>

    <aop:config>
        <aop:aspect id="testAspectAop" ref="testAspect">
                    <aop:around method="aroundinit" pointcut="execution(* com.mmdet.learn.ssm.testaop.TestBiz.init(String,int)) and args(name, age)"/>
        </aop:aspect>
    </aop:config>
```

关键在于切入点的定义上

```
"execution(* com.mmdet.learn.ssm.testaop.TestBiz.init(String,int)) and args(name, age)"
```

init中定义参数的类型，args中定义的是参数名，用and连接，要和业务组件定的参数名一致，通知接收的时候，也必须保持一致。

# 6.事务

## 事务简介

* 事务：指的是逻辑上一组操作，组成这个事务的各个执行单元，要么一起成功,要么一起失败！
* 事务的特性（ACID）
  - 原子性（Atomicity）
    * 原子性是指事务包含的所有操作要么全部成功，要么全部失败回滚。
  - 一致性（Consistency）
    * 一致性是指事务必须使数据库从一个一致性状态变换到另一个一致性状态，也就是说一个事务执行之前和执行之后都必须处于一致性状态。
    * 拿转账来说，假设用户A和用户B两者的钱加起来一共是5000，那么不管A和B之间如何转账，转几次账，事务结束后两个用户的钱相加起来应该还得是5000，这就是事务的一致性。
  - 隔离性（Isolation）
    * 隔离性是当多个用户并发访问数据库时，比如操作同一张表时，数据库为每一个用户开启的事务，不能被其他事务的操作所干扰，多个并发事务之间要相互隔离。
  - 持久性（Durability）
    * 持久性是指一个事务一旦被提交了，那么对数据库中的数据的改变就是永久性的，即便是在数据库系统遇到故障的情况下也不会丢失提交事务的操作。
* 事务并发问题（隔离性导致）
  - 脏读：一个事务读取到另一个事务未提交的数据。
  - 不可重复读：一个事务因读取到另一个事务已提交的数据。导致对同一条记录读取两次以上的结果不一致。update操作
  - 幻读：一个事务因读取到另一个事务已提交的数据。导致对同一张表读取两次以上的结果不一致。insert、delete操作
* 事务隔离级别
  - 四种隔离级别
    * Read uncommitted (读未提交)：最低级别，任何情况都无法保证。
    * Read committed (读已提交)：可避免脏读的发生。
    * Repeatable read (可重复读)：可避免脏读、不可重复读的发生。
    * Serializable (串行化)：可避免脏读、不可重复读、幻读的发生。
  - 默认隔离级别
    * 大多数数据库的默认隔离级别是Read committed，比如Oracle、DB2等。
    * MySQL数据库的默认隔离级别是Repeatable read。
  - 注意事项
    * 隔离级别越高，越能保证数据的完整性和一致性，但是对并发性能的影响也越大。
    * 对于多数应用程序，可以优先考虑把数据库系统的隔离级别设为Read Committed。它能够避免脏读取，而且具有较好的并发性能。尽管它会导致不可重复读、幻读这些并发问题，在可能出现这类问题的个别场合，可以由应用程序采用悲观锁或乐观锁来控制。


## Spring框架的事务管理相关的类和API

Spring并不直接管理事务，而是提供了多种事务管理器，他们将事务管理的职责委托给Hibernate或者JTA等持久化机制所提供的相关平台框架的事务来实现。 Spring事务管理器的接口是PlatformTransactionManager，通过这个接口，Spring为各个平台如JDBC、Hibernate等都提供了对应的事务管理器，但是具体的实现就是各个平台自己的事情了。


<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/Spring事务管理接口.png" width="620px" > </div><br>


* PlatformTransactionManager接口     -- 平台事务管理器.(真正管理事务的类)。该接口有具体的实现类，根据不同的持久层框架，需要选择不同的实现类！ 
* TransactionDefinition接口          -- 事务定义信息.(事务的隔离级别,传播行为,超时,只读)
* TransactionStatus接口              -- 事务的状态（是否新事务、是否已提交、是否有保存点、是否回滚）
* 总结：上述对象之间的关系：平台事务管理器PlatformTransactionManager真正管理事务对象，根据事务定义的信息TransactionDefinition 进行事务管理，在管理事务中产生一些状态，将状态记录到TransactionStatus中
* PlatformTransactionManager接口中实现类和常用的方法
  - 接口的实现类
    * 如果使用的Spring的JDBC模板或者MyBatis（IBatis）框架，需要选择DataSourceTransactionManager实现类
    * 如果使用的是Hibernate的框架，需要选择HibernateTransactionManager实现类
  - 该接口的常用方法
    * void commit(TransactionStatus status) 
    * TransactionStatus getTransaction(TransactionDefinition definition) 
    * void rollback(TransactionStatus status) 
* TransactionDefinition
  - 事务隔离级别的常量
    * static int ISOLATION_DEFAULT              -- 采用数据库的默认隔离级别
    * static int ISOLATION_READ_UNCOMMITTED 
    * static int ISOLATION_READ_COMMITTED 
    * static int ISOLATION_REPEATABLE_READ 
    * static int ISOLATION_SERIALIZABLE 
  - 事务的传播行为常量（不用设置，使用默认值）
    * 先解释什么是事务的传播行为：解决的是业务层之间的方法调用！！
    * PROPAGATION_REQUIRED（默认值） -- A中有事务,使用A中的事务.如果没有，B就会开启一个新的事务,将A包含进来.(保证A,B在同一个事务中)，默认值！！
    * PROPAGATION_SUPPORTS          -- A中有事务,使用A中的事务.如果A中没有事务.那么B也不使用事务.
    * PROPAGATION_MANDATORY         -- A中有事务,使用A中的事务.如果A没有事务.抛出异常.
    * PROPAGATION_REQUIRES_NEW      -- A中有事务,将A中的事务挂起.B创建一个新的事务.(保证A,B没有在一个事务中)
    * PROPAGATION_NOT_SUPPORTED     -- A中有事务,将A中的事务挂起.
    * PROPAGATION_NEVER             -- A中有事务,抛出异常.
    * PROPAGATION_NESTED            -- 嵌套事务.当A执行之后,就会在这个位置设置一个保存点.如果B没有问题.执行通过.如果B出现异常,运行客户根据需求回滚(选择回滚到保存点或者是最初始状态)


## 编程式事务管理

* 说明：Spring为了简化事务管理的代码:提供了模板类 TransactionTemplate，所以手动编程的方式来管理事务，只需要使用该模板类即可！！

* 手动编程方式的具体步骤如下：
  - 步骤一:配置一个事务管理器，Spring使用PlatformTransactionManager接口来管理事务，所以咱们需要使用到他的实现类！！

```
        <!-- 配置事务管理器 -->
        <bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
            <property name="dataSource" ref="dataSource"/>
        </bean>
 ```

  - 步骤二:配置事务管理的模板

```
        <!-- 配置事务管理的模板 -->
        <bean id="transactionTemplate" class="org.springframework.transaction.support.TransactionTemplate">
            <property name="transactionManager" ref="transactionManager"/>
        </bean>
```

  - 步骤三:在需要进行事务管理的类中,注入事务管理的模板

```
        <bean id="accountService" class="com.itheima.demo1.AccountServiceImpl">
            <property name="accountDao" ref="accountDao"/>
            <property name="transactionTemplate" ref="transactionTemplate"/>
        </bean>
```

  - 步骤四:在业务层使用模板管理事务:
 
```
       // 注入事务模板对象
        private TransactionTemplate transactionTemplate;
        public void setTransactionTemplate(TransactionTemplate transactionTemplate) {
            this.transactionTemplate = transactionTemplate;
        }

        public void pay(final String out, final String in, final double money) {
            transactionTemplate.execute(new TransactionCallbackWithoutResult() {

                protected void doInTransactionWithoutResult(TransactionStatus status) {
                    // 扣钱
                    accountDao.outMoney(out, money);
                    int a = 10/0;
                    // 加钱
                    accountDao.inMoney(in, money);
                }
            });
        }
```

## 声明式事务管理

声明式事务管理又分成两种方式
* 基于AspectJ的XML方式
* 基于AspectJ的注解方式

### 事务管理之基于AspectJ的XML方式

* 步骤一:配置一个事务管理器，Spring使用PlatformTransactionManager接口来管理事务，所以咱们需要使用到他的实现类！！

```
        <!-- 配置事务管理器 -->
        <bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
            <property name="dataSource" ref="dataSource"/>
        </bean>
```

* 步骤二：配置事务增强

```
<tx:advice id="txAdvice" transaction-manager="transactionManager">
  <tx:attributes>
    <tx:method name="transfer*" />
    <tx:method name="submitOrder" />
  </tx:attributes>
</tx:advice>
```

* 步骤三：配置切面

```
<aop:config>
  <aop:advisor advice-ref="txAdivce" pointcut="execution(* cn..service.*.*(..))" />
</aop:config>
```

### 事务管理之基于AspectJ的注解方式

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



# 7.BeanFactory的两个重要后置处理器

## BeanFactoryPostProcessor

作用如下：  		
* 在BeanFactory标准初始化之后调用，来定制和修改BeanFactory的内容；
* 所有的bean定义已经保存加载到beanFactory，但是bean的实例还未创建


## BeanDefinitionRegistryPostProcessor

postProcessBeanDefinitionRegistry()在所有bean定义信息将要被加载，bean实例还未创建的。


# 8.Spring整合Junit

* step1.添加spring-test包依赖

```
    <!-- 8)Spring test 对JUNIT等测试框架的简单封装 -->
    <dependency>
      <groupId>org.springframework</groupId>
      <artifactId>spring-test</artifactId>
      <version>${spring.version}</version>
      <scope>test</scope>
    </dependency>
```

* step2.通过@RunWith注解，指定spring的运行器。Spring的运行器是SpringJunit4ClassRunner。
* step3.通过@ContextConfiguration注解，指定spring运行器需要的配置文件路径
* step4.通过@Autowired注解给测试类中的变量注入数据


BaseTest类：
```
/**
 * 配置spring和junit整合，junit启动时加载spring IOC容器
 */
@RunWith(SpringJUnit4ClassRunner.class)
// 告诉junit spring配置文件的位置
@ContextConfiguration({"classpath:spring/spring-dao.xml",
        "classpath:spring/spring-service.xml",
        "classpath:spring/spring-redis.xml"})
public class BaseTest {
}
```

真正的测试类：
```
public class AreaDaoTest extends BaseTest {
    @Autowired
    private AreaDao areaDao;

    @Test
    public void testQueryArea() {
        List<Area> areaList = areaDao.queryArea();
        assertEquals(2, areaList.size());
    }
}
```