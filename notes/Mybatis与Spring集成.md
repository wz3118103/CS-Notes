分析源码之前也需要源码下载并安装到本地仓库和开发工具中，方便给代码添加注释。

# 1.使用Mybatis-spring

## 1.1 使用MapperFactoryBean逐个注入UserMapper

```
	<!-- spring和MyBatis完美整合，不需要mybatis的配置映射文件 -->
	<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
		<property name="dataSource" ref="dataSource" />
		<property name="typeAliasesPackage" value="software.mybatis.entity" />
		<property name="mapperLocations" value="classpath:software/mybatis/sqlmapper/demo/*.xml" />
	</bean>

	 <bean id="tUserMapper" class="org.mybatis.spring.mapper.MapperFactoryBean">
		<property name="mapperInterface" value="software.mybatis.mapper.TUserMapper"></property>
		<property name="sqlSessionFactory" ref="sqlSessionFactory"></property>
	</bean>
```

注入tUserMapper：

```
@Service
public class UserServiceImpl implements UserService {

	Logger logger = LoggerFactory.getLogger(UserServiceImpl.class);
	
	@Resource(name="tUserMapper")
	private TUserMapper userMapper;

	@Override
	public TUser getUserById(Integer id) {
		logger.info("start");
		return userMapper.selectByPrimaryKey(id);
	}
}
```

测试代码：

```
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration("classpath:software/mybatis/applicationContext.xml")
public class MybatisSpringTest {

	@Resource
	private UserService us;
	
	@Test
	public void TestSpringMybatis(){
		System.out.println(us.getUserById(1).toString());
	}
	
}

```

## 1.2 使用MapperScannerConfigurer批量注入Mapper


```
	<!-- spring和MyBatis完美整合，不需要mybatis的配置映射文件 -->
	<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
		<property name="dataSource" ref="dataSource" />
		<property name="typeAliasesPackage" value="software.mybatis.entity" />
		<property name="mapperLocations" value="classpath:software/mybatis/sqlmapper/demo/*.xml" />
	</bean>

	
	<!-- DAO接口所在包名，Spring会自动查找其下的类 -->
	 <bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
		<property name="basePackage" value="software.mybatis.mapper" />
	</bean> 

```

```
@Service
public class UserServiceImpl implements UserService {

	Logger logger = LoggerFactory.getLogger(UserServiceImpl.class);
	
	@Resource
	private TUserMapper userMapper;

	@Override
	public TUser getUserById(Integer id) {
		logger.info("start");
		return userMapper.selectByPrimaryKey(id);
	}
}
```


# 2.SqlSessionFactoryBean源码分析

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/SqlSessionFactoryBean.png" width="820px" > </div><br>


## 2.1 SqlSessionFactoryBean

SqlSessionFactoryBean是一个FactoryBean，每次都通过getObject()返回bean，而不是返回自身。从getObject()可以看出，SqlSessionFactoryBean将SqlSessionFactory注入到容器里面：

```
  /**
   * 
   * 将SqlSessionFactory对象注入spring容器
   * {@inheritDoc}
   */
  @Override
  public SqlSessionFactory getObject() throws Exception {
    if (this.sqlSessionFactory == null) {
      afterPropertiesSet();
    }

    return this.sqlSessionFactory;
  }


  @Override
  public boolean isSingleton() {
    return true;
  }
```

## 2.2 InitializingBean

重写了afterPropertiesSet()方法。

```
  @Override
  //在spring容器中创建全局唯一的sqlSessionFactory
  public void afterPropertiesSet() throws Exception {
    notNull(dataSource, "Property 'dataSource' is required");
    notNull(sqlSessionFactoryBuilder, "Property 'sqlSessionFactoryBuilder' is required");
    state((configuration == null && configLocation == null) || !(configuration != null && configLocation != null),
              "Property 'configuration' and 'configLocation' can not specified with together");

    this.sqlSessionFactory = buildSqlSessionFactory();
  }
```

Mybatis中的配置除了settings不能移动到Spring中外，其他都可以移过来。配置在SqlSessionFactoryBean里面进行配置。基本流程跟Mybatis中相同：
* 配置内容全部注册到Configuration
* 使用XMLConfigBuilder、XMLMapperBuilder、XMLConfigBuilder进行解析

```
  protected SqlSessionFactory buildSqlSessionFactory() throws IOException {

    Configuration configuration;

    XMLConfigBuilder xmlConfigBuilder = null;
    if (this.configuration != null) {//如果configuration不为空，则使用该对象，并对其进行配置
      configuration = this.configuration;
      if (configuration.getVariables() == null) {
        configuration.setVariables(this.configurationProperties);
      } else if (this.configurationProperties != null) {
        configuration.getVariables().putAll(this.configurationProperties);
      }
    } else if (this.configLocation != null) {//创建xmlConfigBuilder，读取mybatis的核心配置文件
      xmlConfigBuilder = new XMLConfigBuilder(this.configLocation.getInputStream(), null, this.configurationProperties);
      configuration = xmlConfigBuilder.getConfiguration();
    } else {//如果configuration为空，实例化一个configuration对象
      if (LOGGER.isDebugEnabled()) {
        LOGGER.debug("Property 'configuration' or 'configLocation' not specified, using default MyBatis Configuration");
      }
      configuration = new Configuration();
      if (this.configurationProperties != null) {
        configuration.setVariables(this.configurationProperties);
      }
    }
    //设置objectFactory
    if (this.objectFactory != null) {
      configuration.setObjectFactory(this.objectFactory);
    }
    //设置objectWrapperFactory
    if (this.objectWrapperFactory != null) {
      configuration.setObjectWrapperFactory(this.objectWrapperFactory);
    }
  //设置vfs
    if (this.vfs != null) {
      configuration.setVfsImpl(this.vfs);
    }
  //扫描指定的包typeAliasesPackage，注册别名
    if (hasLength(this.typeAliasesPackage)) {
      String[] typeAliasPackageArray = tokenizeToStringArray(this.typeAliasesPackage,
          ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS);
      for (String packageToScan : typeAliasPackageArray) {
        configuration.getTypeAliasRegistry().registerAliases(packageToScan,
                typeAliasesSuperType == null ? Object.class : typeAliasesSuperType);
        if (LOGGER.isDebugEnabled()) {
          LOGGER.debug("Scanned package: '" + packageToScan + "' for aliases");
        }
      }
    }
  //为typeAliases指定的类注册别名
    if (!isEmpty(this.typeAliases)) {
      for (Class<?> typeAlias : this.typeAliases) {
        configuration.getTypeAliasRegistry().registerAlias(typeAlias);
        if (LOGGER.isDebugEnabled()) {
          LOGGER.debug("Registered type alias: '" + typeAlias + "'");
        }
      }
    }
    //注册插件
    if (!isEmpty(this.plugins)) {
      for (Interceptor plugin : this.plugins) {
        configuration.addInterceptor(plugin);
        if (LOGGER.isDebugEnabled()) {
          LOGGER.debug("Registered plugin: '" + plugin + "'");
        }
      }
    }
    //扫描指定的包typeHandlersPackage，注册类型解析器
    if (hasLength(this.typeHandlersPackage)) {
      String[] typeHandlersPackageArray = tokenizeToStringArray(this.typeHandlersPackage,
          ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS);
      for (String packageToScan : typeHandlersPackageArray) {
        configuration.getTypeHandlerRegistry().register(packageToScan);
        if (LOGGER.isDebugEnabled()) {
          LOGGER.debug("Scanned package: '" + packageToScan + "' for type handlers");
        }
      }
    }
  //为typeHandlers指定的类注册类型解析器
    if (!isEmpty(this.typeHandlers)) {
      for (TypeHandler<?> typeHandler : this.typeHandlers) {
        configuration.getTypeHandlerRegistry().register(typeHandler);
        if (LOGGER.isDebugEnabled()) {
          LOGGER.debug("Registered type handler: '" + typeHandler + "'");
        }
      }
    }
    //配置databaseIdProvider
    if (this.databaseIdProvider != null) {//fix #64 set databaseId before parse mapper xmls
      try {
        configuration.setDatabaseId(this.databaseIdProvider.getDatabaseId(this.dataSource));
      } catch (SQLException e) {
        throw new NestedIOException("Failed getting a databaseId", e);
      }
    }
  //配置缓存
    if (this.cache != null) {
      configuration.addCache(this.cache);
    }
     //使用xmlConfigBuilder读取mybatis的核心配置文件
    if (xmlConfigBuilder != null) {
      try {
        xmlConfigBuilder.parse();

        if (LOGGER.isDebugEnabled()) {
          LOGGER.debug("Parsed configuration file: '" + this.configLocation + "'");
        }
      } catch (Exception ex) {
        throw new NestedIOException("Failed to parse config resource: " + this.configLocation, ex);
      } finally {
        ErrorContext.instance().reset();
      }
    }
    //默认使用SpringManagedTransactionFactory作为事务管理器
    if (this.transactionFactory == null) {
      this.transactionFactory = new SpringManagedTransactionFactory();
    }
   //设置Environment
    configuration.setEnvironment(new Environment(this.environment, this.transactionFactory, this.dataSource));

  //根据mapperLocations的配置，处理映射配置文件以及相应的mapper接口
    if (!isEmpty(this.mapperLocations)) {
      for (Resource mapperLocation : this.mapperLocations) {
        if (mapperLocation == null) {
          continue;
        }

        try {
          XMLMapperBuilder xmlMapperBuilder = new XMLMapperBuilder(mapperLocation.getInputStream(),
              configuration, mapperLocation.toString(), configuration.getSqlFragments());
          xmlMapperBuilder.parse();
        } catch (Exception e) {
          throw new NestedIOException("Failed to parse mapping resource: '" + mapperLocation + "'", e);
        } finally {
          ErrorContext.instance().reset();
        }

        if (LOGGER.isDebugEnabled()) {
          LOGGER.debug("Parsed mapper file: '" + mapperLocation + "'");
        }
      }
    } else {
      if (LOGGER.isDebugEnabled()) {
        LOGGER.debug("Property 'mapperLocations' was not specified or no matching resources found");
      }
    }
    //最终使用sqlSessionFactoryBuilder创建sqlSessionFactory
    return this.sqlSessionFactoryBuilder.build(configuration);
  }

```

# 3.MapperFactoryBean源码分析

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/MapperFactoryBean.png" width="620px" > </div><br>

## 3.1 FactoryBean

```
  /**
   * 通过在容器中的mapperRegistry，返回当前mapper接口的动态代理
   * 
   * {@inheritDoc}
   */
  @Override
  public T getObject() throws Exception {
    return getSqlSession().getMapper(this.mapperInterface);
  }

  @Override
  public boolean isSingleton() {
    return true;
  }
```

这里取出来的是SqlSessionTemplate.getMapper()：

```
  @Override
  public <T> T getMapper(Class<T> type) {
    return getConfiguration().getMapper(type, this);
  }

  @Override
  public Configuration getConfiguration() {
    return this.sqlSessionFactory.getConfiguration();
  }

```

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/SqlSessionTemplate.png" width="520px" > </div><br>

后面从Configuration中获取Mapper，就跟之前Mybatis的流程一致。

## 3.2 private TUserMapper userMapper为什么没有问题

因为这里使用的是MapperFactoryBean，每次都是从getObject()中获取实例，而getObject()获取的都是新创建的代理对象，因此不会有作用域和生命周期的问题。


# 4.MapperScannerConfigurer源码分析

针对每一个Mapper都配置MapperFactoryBean太麻烦，可以使用MapperScannerConfigurer进行批量自动处理。

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/MapperScannerConfigurer.png" width="820px" > </div><br>

## 4.1 BeanDefinitionRegistryPostProcessor

BeanDefinitionRegistryPostProcessor对类进行增强，将Mapper接口增强为MapperFactoryBean那样。然后将其注入到容器中。

```
  @Override
  public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
    if (this.processPropertyPlaceHolders) {//占位符处理
      processPropertyPlaceHolders();
    }
    //实例化ClassPathMapperScanner，并对scanner相关属性进行配置
    ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);
    scanner.setAddToConfig(this.addToConfig);
    scanner.setAnnotationClass(this.annotationClass);
    scanner.setMarkerInterface(this.markerInterface);
    scanner.setSqlSessionFactory(this.sqlSessionFactory);
    scanner.setSqlSessionTemplate(this.sqlSessionTemplate);
    scanner.setSqlSessionFactoryBeanName(this.sqlSessionFactoryBeanName);
    scanner.setSqlSessionTemplateBeanName(this.sqlSessionTemplateBeanName);
    scanner.setResourceLoader(this.applicationContext);
    scanner.setBeanNameGenerator(this.nameGenerator);
    scanner.registerFilters();//根据上述配置，生成过滤器，只扫描合条件的class
    //扫描指定的包以及其子包
    scanner.scan(StringUtils.tokenizeToStringArray(this.basePackage, ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS));
  }

```

```
	public int scan(String... basePackages) {
		int beanCountAtScanStart = this.registry.getBeanDefinitionCount();

		doScan(basePackages);

		// Register annotation config processors, if necessary.
		if (this.includeAnnotationConfig) {
			AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
		}

		return (this.registry.getBeanDefinitionCount() - beanCountAtScanStart);
	}


```

ClassPathMapperScanner：

```

  @Override
  public Set<BeanDefinitionHolder> doScan(String... basePackages) {
	//通过父类的扫描，获取所有复合条件的BeanDefinitionHolder对象
    Set<BeanDefinitionHolder> beanDefinitions = super.doScan(basePackages);

    if (beanDefinitions.isEmpty()) {
      logger.warn("No MyBatis mapper was found in '" + Arrays.toString(basePackages) + "' package. Please check your configuration.");
    } else {
      //处理扫描得到的BeanDefinitionHolder集合，将集合中的每一个mapper接口转换成MapperFactoryBean后，注册至spring容器
      processBeanDefinitions(beanDefinitions);
    }

    return beanDefinitions;
  }




  //处理扫描得到的BeanDefinitionHolder集合，将集合中的每一个mapper接口转换成MapperFactoryBean后，注册至spring容器
  private void processBeanDefinitions(Set<BeanDefinitionHolder> beanDefinitions) {
    GenericBeanDefinition definition;
    //遍历集合
    for (BeanDefinitionHolder holder : beanDefinitions) {
      definition = (GenericBeanDefinition) holder.getBeanDefinition();

      if (logger.isDebugEnabled()) {
        logger.debug("Creating MapperFactoryBean with name '" + holder.getBeanName() 
          + "' and '" + definition.getBeanClassName() + "' mapperInterface");
      }

      // the mapper interface is the original class of the bean
      // but, the actual class of the bean is MapperFactoryBean
      //将添加扫描到的接口类型作为构造函数的入参
      definition.getConstructorArgumentValues().addGenericArgumentValue(definition.getBeanClassName()); // issue #59
      //讲bean的类型转换成mapperFactoryBean
      definition.setBeanClass(this.mapperFactoryBean.getClass());
      //增加addToConfig属性
      definition.getPropertyValues().add("addToConfig", this.addToConfig);

      boolean explicitFactoryUsed = false;
      //增加sqlSessionFactory属性
      if (StringUtils.hasText(this.sqlSessionFactoryBeanName)) {
        definition.getPropertyValues().add("sqlSessionFactory", new RuntimeBeanReference(this.sqlSessionFactoryBeanName));
        explicitFactoryUsed = true;
      } else if (this.sqlSessionFactory != null) {
        definition.getPropertyValues().add("sqlSessionFactory", this.sqlSessionFactory);
        explicitFactoryUsed = true;
      }
      //增加sqlSessionTemplate属性
      if (StringUtils.hasText(this.sqlSessionTemplateBeanName)) {
        if (explicitFactoryUsed) {
          logger.warn("Cannot use both: sqlSessionTemplate and sqlSessionFactory together. sqlSessionFactory is ignored.");
        }
        definition.getPropertyValues().add("sqlSessionTemplate", new RuntimeBeanReference(this.sqlSessionTemplateBeanName));
        explicitFactoryUsed = true;
      } else if (this.sqlSessionTemplate != null) {
        if (explicitFactoryUsed) {
          logger.warn("Cannot use both: sqlSessionTemplate and sqlSessionFactory together. sqlSessionFactory is ignored.");
        }
        definition.getPropertyValues().add("sqlSessionTemplate", this.sqlSessionTemplate);
        explicitFactoryUsed = true;
      }

      //修改自动注入的方式 bytype
      if (!explicitFactoryUsed) {
        if (logger.isDebugEnabled()) {
          logger.debug("Enabling autowire by type for MapperFactoryBean with name '" + holder.getBeanName() + "'.");
        }
        definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);
      }
    }
  }
```

通过将Mapper的definition转换为mapperFactoryBean类型，并添加mapperFactoryBean所需的各种属性。最后将Mapper的注入方式改为byType。





