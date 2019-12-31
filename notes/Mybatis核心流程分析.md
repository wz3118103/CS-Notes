
<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/Mybatis核心流程三阶段.jpg" width="520px" > </div><br>

# 1.初始化——配置加载阶段

## 1.1 建造者模式

建造者模式（Builder Pattern）使用多个简单的对象一步一步构建成一个复杂的对象。这种类型的设计模式属于创建型模式，它提供了一种创建对象的最佳方式。


<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/builder.png" width="420px" > </div><br>

建造者模式使用场景：
* 需要生成的对象具有复杂的内部结构，实例化对象时要屏蔽掉对象内部的细节，让上层代码与复杂对象的实例化过程解耦，可以使用建造者模式；简而言之，如果“遇到多个构造器参数时要考虑用构建器”；
* 一个对象的实例化是依赖各个组件的产生以及装配顺序，关注的是一步一步地组装出目标对象，可以使用建造器模式。

与工程模式的区别：
* 对象复杂度
  - 建造者建造的对象更加复杂，是一个复合产品，它由各个部件复合而成，部件不同产品对象不同，生成的产品粒度细；
  - 在工厂方法模式里，我们关注的是一个产品整体，无须关心产品的各部分是如何创建出来的；
* 客户端参与程度
  - 建造者模式，导演对象参与了产品的创建，决定了产品的类型和内容，参与度高；适合实例化对象时属性变化频繁的场景；
  - 工厂模式，客户端对产品的创建过程参与度低，对象实例化时属性值相对比较固定；



## 1.2 Mybatis配置加载

### 1.2.1 DefaultSqlSessionFactory包含Configuration

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/DefaultSqlSessionFactory.png" width="420px" > </div><br>

```
public class MybatisQuickStart {
    private SqlSessionFactory sqlSessionFactory;

    @Before
    public void init() throws IOException {
        String resource = "software/mybatis/mybatis-quickstart-config.xml";
        InputStream inputStream = Resources.getResourceAsStream(resource);
        sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
        inputStream.close();
    }
```

SqlSessionFactoryBuilder.build()方法：

```
  public SqlSessionFactory build(InputStream inputStream) {
    return build(inputStream, null, null);
  }


  public SqlSessionFactory build(InputStream inputStream, String environment, Properties properties) {
    try {
      XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);
      return build(parser.parse());
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error building SqlSession.", e);
    } finally {
      ErrorContext.instance().reset();
      try {
        inputStream.close();
      } catch (IOException e) {
        // Intentionally ignore. Prefer previous error.
      }
    }
  }


  public SqlSessionFactory build(Configuration config) {
    return new DefaultSqlSessionFactory(config);
  }
```

build方法中，XMLConfigBuilder.parse()生成Configuration，然后将Configuration赋值给DefaultSqlSessionFactory。

### 1.2.2 XMLConfigBuilder解析mybatis-config.xml

XMLConfigBuilder.parse()方法：

```
  public Configuration parse() {
    if (parsed) {
      throw new BuilderException("Each XMLConfigBuilder can only be used once.");
    }
    parsed = true;
    parseConfiguration(parser.evalNode("/configuration"));
    return configuration;
  }


  private void parseConfiguration(XNode root) {
    try {
      //issue #117 read properties first
     //解析<properties>节点
      propertiesElement(root.evalNode("properties"));
      //解析<settings>节点
      Properties settings = settingsAsProperties(root.evalNode("settings"));
      loadCustomVfs(settings);
      //解析<typeAliases>节点
      typeAliasesElement(root.evalNode("typeAliases"));
      //解析<plugins>节点
      pluginElement(root.evalNode("plugins"));
      //解析<objectFactory>节点
      objectFactoryElement(root.evalNode("objectFactory"));
      //解析<objectWrapperFactory>节点
      objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
      //解析<reflectorFactory>节点
      reflectorFactoryElement(root.evalNode("reflectorFactory"));
      settingsElement(settings);//将settings填充到configuration
      // read it after objectFactory and objectWrapperFactory issue #631
      //解析<environments>节点
      environmentsElement(root.evalNode("environments"));
      //解析<databaseIdProvider>节点
      databaseIdProviderElement(root.evalNode("databaseIdProvider"));
      //解析<typeHandlers>节点
      typeHandlerElement(root.evalNode("typeHandlers"));
      //解析<mappers>节点
      mapperElement(root.evalNode("mappers"));
    } catch (Exception e) {
      throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
    }
  }

```

核心流程全在parseConfiguration()方法中。

environmentsElement()方法解析<environments>，每个environment都包含事务管理以及数据源。并且其Environment的构造方法是典型的建造者模式。

```
  private void environmentsElement(XNode context) throws Exception {
    if (context != null) {
      if (environment == null) {
        environment = context.getStringAttribute("default");
      }
      for (XNode child : context.getChildren()) {
        String id = child.getStringAttribute("id");
        if (isSpecifiedEnvironment(id)) {
          TransactionFactory txFactory = transactionManagerElement(child.evalNode("transactionManager"));
          DataSourceFactory dsFactory = dataSourceElement(child.evalNode("dataSource"));
          DataSource dataSource = dsFactory.getDataSource();
          Environment.Builder environmentBuilder = new Environment.Builder(id)
              .transactionFactory(txFactory)
              .dataSource(dataSource);
          configuration.setEnvironment(environmentBuilder.build());
        }
      }
    }
  }
```

```
	<environments default="development">
		<!-- 环境配置1，每个SqlSessionFactory对应一个环境 -->
		<environment id="development">
			<transactionManager type="JDBC" />
			<dataSource type="UNPOOLED">
				<property name="driver" value="${jdbc_driver}" />
				<property name="url" value="${jdbc_url}" />
				<property name="username" value="${jdbc_username}" />
				<property name="password" value="${jdbc_password}" />
			</dataSource>
		</environment>

	</environments>
```

### 1.2.3 核心mapperElement(root.evalNode("mappers"))调用XMLMapperBuilder

```
	<mappers>
		<!--直接映射到相应的mapper文件 -->
		<mapper resource="software/mybatis/sqlmapper/demo/TUserMapper.xml" />
		<mapper resource="software/mybatis/sqlmapper/demo/THealthReportFemaleMapper.xml" />
		<mapper resource="software/mybatis/sqlmapper/demo/THealthReportMaleMapper.xml" />
		<mapper class="software.mybatis.mapper.TJobHistoryAnnoMapper"/>
	</mappers>

```

```
  private void mapperElement(XNode parent) throws Exception {
    if (parent != null) {
      for (XNode child : parent.getChildren()) {//处理mapper子节点
        if ("package".equals(child.getName())) {//package子节点
          String mapperPackage = child.getStringAttribute("name");
          configuration.addMappers(mapperPackage);
        } else {//获取<mapper>节点的resource、url或mClass属性这三个属性互斥
          String resource = child.getStringAttribute("resource");
          String url = child.getStringAttribute("url");
          String mapperClass = child.getStringAttribute("class");
          if (resource != null && url == null && mapperClass == null) {//如果resource不为空
            ErrorContext.instance().resource(resource);
            InputStream inputStream = Resources.getResourceAsStream(resource);//加载mapper文件
            //实例化XMLMapperBuilder解析mapper映射文件
            XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource, configuration.getSqlFragments());
            mapperParser.parse();
          } else if (resource == null && url != null && mapperClass == null) {//如果url不为空
            ErrorContext.instance().resource(url);
            InputStream inputStream = Resources.getUrlAsStream(url);//加载mapper文件
            //实例化XMLMapperBuilder解析mapper映射文件
            XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, url, configuration.getSqlFragments());
            mapperParser.parse();
          } else if (resource == null && url == null && mapperClass != null) {//如果class不为空
            Class<?> mapperInterface = Resources.classForName(mapperClass);//加载class对象
            configuration.addMapper(mapperInterface);//向代理中心注册mapper
          } else {
            throw new BuilderException("A mapper element may only specify a url, resource or class, but not more than one.");
          }
        }
      }
    }
  }
```

XMLMapperBuilder.parse()：

```
  public void parse() {
	//判断是否已经加载该配置文件
    if (!configuration.isResourceLoaded(resource)) {
      configurationElement(parser.evalNode("/mapper"));//处理mapper节点
      configuration.addLoadedResource(resource);//将mapper文件添加到configuration.loadedResources中
      bindMapperForNamespace();//注册mapper接口
    }
    //处理解析失败的ResultMap节点
    parsePendingResultMaps();
    //处理解析失败的CacheRef节点
    parsePendingCacheRefs();
    //处理解析失败的Sql语句节点
    parsePendingStatements();
  }
```

**step1.configurationElement(parser.evalNode("/mapper"))**

```
<mapper namespace="software.mybatis.mapper.TUserMapper">
	
 	<!--<cache></cache>-->

	<resultMap id="BaseResultMap" type="TUser">

		<!-- <constructor> <idArg column="id" javaType="int"/> <arg column="user_name" 
			javaType="String"/> </constructor> -->
		<id column="id" property="id" />
		<result column="user_name" property="userName" />
		<result column="real_name" property="realName" />
		<result column="sex" property="sex" />
		<result column="mobile" property="mobile" />
		<result column="email" property="email" />
		<result column="note" property="note" />
	</resultMap>

	<resultMap id="userAndPosition1" extends="BaseResultMap" type="TUser">
		<association property="position" javaType="TPosition" columnPrefix="post_">
			<id column="id" property="id"/>
			<result column="name" property="postName"/>
			<result column="note" property="note"/>
		</association>
	</resultMap>

	<resultMap id="userAndPosition2" extends="BaseResultMap" type="TUser">
		<association property="position" fetchType="lazy"  column="position_id" select="software.mybatis.mapper.TPositionMapper.selectByPrimaryKey" />
	</resultMap>



	<select id="selectUserPosition1" resultMap="userAndPosition1">
		select
		    a.id, 
		    user_name,
			real_name,
			sex,
			mobile,
			email,
			a.note,
			b.id  post_id,
			b.post_name,
			b.note post_note
		from t_user a,
			t_position b
		where a.position_id = b.id

	</select>

	<select id="selectUserPosition2" resultMap="userAndPosition2">
		select
		a.id,
		a.user_name,
		a.real_name,
		a.sex,
		a.mobile,
		a.position_id
		from t_user a
	</select>
```

```
  private void configurationElement(XNode context) {
    try {
    	//获取mapper节点的namespace属性
      String namespace = context.getStringAttribute("namespace");
      if (namespace == null || namespace.equals("")) {
        throw new BuilderException("Mapper's namespace cannot be empty");
      }
      //设置builderAssistant的namespace属性
      builderAssistant.setCurrentNamespace(namespace);
      //解析cache-ref节点
      cacheRefElement(context.evalNode("cache-ref"));
      //重点分析 ：解析cache节点----------------1-------------------
      cacheElement(context.evalNode("cache"));
      //解析parameterMap节点（已废弃）
      parameterMapElement(context.evalNodes("/mapper/parameterMap"));
      //重点分析 ：解析resultMap节点（基于数据结果去理解）----------------2-------------------
      resultMapElements(context.evalNodes("/mapper/resultMap"));
      //解析sql节点
      sqlElement(context.evalNodes("/mapper/sql"));
      //重点分析 ：解析select、insert、update、delete节点 ----------------3-------------------
      buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
    } catch (Exception e) {
      throw new BuilderException("Error parsing Mapper XML. The XML location is '" + resource + "'. Cause: " + e, e);
    }
  }
```

**step1-1.cacheElement(context.evalNode("cache"))**

```
  private void cacheElement(XNode context) throws Exception {
    if (context != null) {
      //获取cache节点的type属性，默认为PERPETUAL
      String type = context.getStringAttribute("type", "PERPETUAL");
      //找到type对应的cache接口的实现
      Class<? extends Cache> typeClass = typeAliasRegistry.resolveAlias(type);
      //读取eviction属性，既缓存的淘汰策略，默认LRU
      String eviction = context.getStringAttribute("eviction", "LRU");
      //根据eviction属性，找到装饰器
      Class<? extends Cache> evictionClass = typeAliasRegistry.resolveAlias(eviction);
      //读取flushInterval属性，既缓存的刷新周期
      Long flushInterval = context.getLongAttribute("flushInterval");
      //读取size属性，既缓存的容量大小
      Integer size = context.getIntAttribute("size");
     //读取readOnly属性，既缓存的是否只读
      boolean readWrite = !context.getBooleanAttribute("readOnly", false);
      //读取blocking属性，既缓存的是否阻塞
      boolean blocking = context.getBooleanAttribute("blocking", false);
      Properties props = context.getChildrenAsProperties();
      //通过builderAssistant创建缓存对象，并添加至configuration
      builderAssistant.useNewCache(typeClass, evictionClass, flushInterval, size, readWrite, blocking, props);
    }
  }

```

这里通过建造者CacheBuilder构造二级缓存，并放到configuration中去。

```
 //通过builderAssistant创建缓存对象，并添加至configuration
  public Cache useNewCache(Class<? extends Cache> typeClass,
      Class<? extends Cache> evictionClass,
      Long flushInterval,
      Integer size,
      boolean readWrite,
      boolean blocking,
      Properties props) {
	//经典的建造起模式，创建一个cache对象
    Cache cache = new CacheBuilder(currentNamespace)
        .implementation(valueOrDefault(typeClass, PerpetualCache.class))
        .addDecorator(valueOrDefault(evictionClass, LruCache.class))
        .clearInterval(flushInterval)
        .size(size)
        .readWrite(readWrite)
        .blocking(blocking)
        .properties(props)
        .build();
    //将缓存添加至configuration，注意二级缓存以命名空间为单位进行划分
    configuration.addCache(cache);
    currentCache = cache;
    return cache;
  }

```


**step1-2.resultMapElements(context.evalNodes("/mapper/resultMap"))**

```
  //解析resultMap节点,实际就是解析sql查询的字段与pojo属性之间的转化规则
  private void resultMapElements(List<XNode> list) throws Exception {
	//遍历所有的resultmap节点
    for (XNode resultMapNode : list) {
      try {
    	 //解析具体某一个resultMap节点
        resultMapElement(resultMapNode);
      } catch (IncompleteElementException e) {
        // ignore, it will be retried
      }
    }
  }

  private ResultMap resultMapElement(XNode resultMapNode) throws Exception {
    return resultMapElement(resultMapNode, Collections.<ResultMapping> emptyList());
  }


  private ResultMap resultMapElement(XNode resultMapNode, List<ResultMapping> additionalResultMappings) throws Exception {
    ErrorContext.instance().activity("processing " + resultMapNode.getValueBasedIdentifier());
    //获取resultmap节点的id属性
    String id = resultMapNode.getStringAttribute("id",
        resultMapNode.getValueBasedIdentifier());
    //获取resultmap节点的type属性
    String type = resultMapNode.getStringAttribute("type",
        resultMapNode.getStringAttribute("ofType",
            resultMapNode.getStringAttribute("resultType",
                resultMapNode.getStringAttribute("javaType"))));
    //获取resultmap节点的extends属性，描述继承关系
    String extend = resultMapNode.getStringAttribute("extends");
    //获取resultmap节点的autoMapping属性，是否开启自动映射
    Boolean autoMapping = resultMapNode.getBooleanAttribute("autoMapping");
    //从别名注册中心获取entity的class对象
    Class<?> typeClass = resolveClass(type);
    Discriminator discriminator = null;
    //记录子节点中的映射结果集合
    List<ResultMapping> resultMappings = new ArrayList<>();
    resultMappings.addAll(additionalResultMappings);
    //从xml文件中获取当前resultmap中的所有子节点，并开始遍历
    List<XNode> resultChildren = resultMapNode.getChildren();
    for (XNode resultChild : resultChildren) {
      if ("constructor".equals(resultChild.getName())) {//处理<constructor>节点
        processConstructorElement(resultChild, typeClass, resultMappings);
      } else if ("discriminator".equals(resultChild.getName())) {//处理<discriminator>节点
        discriminator = processDiscriminatorElement(resultChild, typeClass, resultMappings);
      } else {//处理<id> <result> <association> <collection>节点
        List<ResultFlag> flags = new ArrayList<>();
        if ("id".equals(resultChild.getName())) {
          flags.add(ResultFlag.ID);//如果是id节点，向flags中添加元素
        }
        //创建ResultMapping对象并加入resultMappings集合中
        resultMappings.add(buildResultMappingFromContext(resultChild, typeClass, flags));
      }
    }
    //实例化resultMap解析器
    ResultMapResolver resultMapResolver = new ResultMapResolver(builderAssistant, id, typeClass, extend, discriminator, resultMappings, autoMapping);
    try {
      //通过resultMap解析器实例化resultMap并将其注册到configuration对象
      return resultMapResolver.resolve();
    } catch (IncompleteElementException  e) {
      configuration.addIncompleteResultMap(resultMapResolver);
      throw e;
    }
  }
```
buildResultMapping采用建造者模式构造出ResultMapping：

```
  public ResultMapping buildResultMapping(
      Class<?> resultType,
      String property,
      String column,
      Class<?> javaType,
      JdbcType jdbcType,
      String nestedSelect,
      String nestedResultMap,
      String notNullColumn,
      String columnPrefix,
      Class<? extends TypeHandler<?>> typeHandler,
      List<ResultFlag> flags,
      String resultSet,
      String foreignColumn,
      boolean lazy) {
    Class<?> javaTypeClass = resolveResultJavaType(resultType, property, javaType);
    TypeHandler<?> typeHandlerInstance = resolveTypeHandler(javaTypeClass, typeHandler);
    List<ResultMapping> composites = parseCompositeColumnName(column);
    return new ResultMapping.Builder(configuration, property, column, javaTypeClass)
        .jdbcType(jdbcType)
        .nestedQueryId(applyCurrentNamespace(nestedSelect, true))
        .nestedResultMapId(applyCurrentNamespace(nestedResultMap, true))
        .resultSet(resultSet)
        .typeHandler(typeHandlerInstance)
        .flags(flags == null ? new ArrayList<ResultFlag>() : flags)
        .composites(composites)
        .notNullColumns(parseMultipleColumnNames(notNullColumn))
        .columnPrefix(columnPrefix)
        .foreignColumn(foreignColumn)
        .lazy(lazy)
        .build();
  }
```

其中ResultMapping类如下：

```
public class ResultMapping {

  private Configuration configuration;//引用的configuration对象
  private String property;//对应节点的property属性
  private String column;//对应节点的column属性
  private Class<?> javaType;//对应节点的javaType属性
  private JdbcType jdbcType;//对应节点的jdbcType属性
  private TypeHandler<?> typeHandler;//对应节点的typeHandler属性
  private String nestedResultMapId;////对应节点的resultMap属性,嵌套结果时使用
  private String nestedQueryId;////对应节点的select属性,嵌套查询时使用
  private Set<String> notNullColumns;//对应节点的notNullColumn属性
  private String columnPrefix;//对应节点的columnPrefix属性
  private List<ResultFlag> flags;//标志,id 或者 constructor
  private List<ResultMapping> composites;
  private String resultSet;//对应节点的resultSet属性
  private String foreignColumn;//对应节点的foreignColumn属性
  private boolean lazy;//对应节点的fetchType属性,是否延迟加载

  ResultMapping() {
  }

  public static class Builder {
    private ResultMapping resultMapping = new ResultMapping();
```

再通过ResultMapResolver.resolve()：

```
public class ResultMapResolver {
  private final MapperBuilderAssistant assistant;
  private final String id;
  private final Class<?> type;
  private final String extend;
  private final Discriminator discriminator;
  private final List<ResultMapping> resultMappings;
  private final Boolean autoMapping;

  public ResultMapResolver(MapperBuilderAssistant assistant, String id, Class<?> type, String extend, Discriminator discriminator, List<ResultMapping> resultMappings, Boolean autoMapping) {
    this.assistant = assistant;
    this.id = id;
    this.type = type;
    this.extend = extend;
    this.discriminator = discriminator;
    this.resultMappings = resultMappings;
    this.autoMapping = autoMapping;
  }

  public ResultMap resolve() {
    return assistant.addResultMap(this.id, this.type, this.extend, this.discriminator, this.resultMappings, this.autoMapping);
  }
```

MapperBuilderAssistant.addResultMap()方法：

```
  //实例化resultMap并将其注册到configuration对象
  public ResultMap addResultMap(
      String id,
      Class<?> type,
      String extend,
      Discriminator discriminator,
      List<ResultMapping> resultMappings,
      Boolean autoMapping) {
	 //完善id，id的完整格式是"namespace.id"
    id = applyCurrentNamespace(id, false);
    //获得父类resultMap的完整id
    extend = applyCurrentNamespace(extend, true);

    //针对extend属性的处理
    if (extend != null) {
      if (!configuration.hasResultMap(extend)) {
        throw new IncompleteElementException("Could not find a parent resultmap with id '" + extend + "'");
      }
      ResultMap resultMap = configuration.getResultMap(extend);
      List<ResultMapping> extendedResultMappings = new ArrayList<>(resultMap.getResultMappings());
      extendedResultMappings.removeAll(resultMappings);
      // Remove parent constructor if this resultMap declares a constructor.
      boolean declaresConstructor = false;
      for (ResultMapping resultMapping : resultMappings) {
        if (resultMapping.getFlags().contains(ResultFlag.CONSTRUCTOR)) {
          declaresConstructor = true;
          break;
        }
      }
      if (declaresConstructor) {
        Iterator<ResultMapping> extendedResultMappingsIter = extendedResultMappings.iterator();
        while (extendedResultMappingsIter.hasNext()) {
          if (extendedResultMappingsIter.next().getFlags().contains(ResultFlag.CONSTRUCTOR)) {
            extendedResultMappingsIter.remove();
          }
        }
      }
      //添加需要被继承下来的resultMapping对象结合
      resultMappings.addAll(extendedResultMappings);
    }
    //通过建造者模式实例化resultMap,并注册到configuration.resultMaps中
    ResultMap resultMap = new ResultMap.Builder(configuration, id, type, resultMappings, autoMapping)
        .discriminator(discriminator)
        .build();
    configuration.addResultMap(resultMap);
    return resultMap;
  }

```

最后构造出ResultMap，并将其放入到configuration的resultMaps中去。

```
public class ResultMap {
  private Configuration configuration;//configuration对象

  private String id;//resultMap的id属性
  private Class<?> type;//resultMap的type属性
  private List<ResultMapping> resultMappings;//除discriminator节点之外的映射关系
  private List<ResultMapping> idResultMappings;//记录ID或者<constructor>中idArg的映射关系
  private List<ResultMapping> constructorResultMappings;////记录<constructor>标志的映射关系
  private List<ResultMapping> propertyResultMappings;//记录非<constructor>标志的映射关系
  private Set<String> mappedColumns;//记录所有有映射关系的columns字段
  private Set<String> mappedProperties;//记录所有有映射关系的property字段
  private Discriminator discriminator;//鉴别器，对应discriminator节点
  private boolean hasNestedResultMaps;//是否有嵌套结果映射
  private boolean hasNestedQueries;////是否有嵌套查询
  private Boolean autoMapping;//是否开启了自动映射
```

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/ResultMap.jpg" width="620px" > </div><br>


**step1-3.buildStatementFromContext(context.evalNodes("select|insert|update|delete"))——XMLStatementBuilder**

```
  //解析select、insert、update、delete节点
  private void buildStatementFromContext(List<XNode> list) {
    if (configuration.getDatabaseId() != null) {
      buildStatementFromContext(list, configuration.getDatabaseId());
    }
    buildStatementFromContext(list, null);
  }
```

```
  //处理所有的sql语句节点并注册至configuration对象
  private void buildStatementFromContext(List<XNode> list, String requiredDatabaseId) {
    for (XNode context : list) {
      //创建XMLStatementBuilder 专门用于解析sql语句节点
      final XMLStatementBuilder statementParser = new XMLStatementBuilder(configuration, builderAssistant, context, requiredDatabaseId);
      try {
    	//解析sql语句节点
        statementParser.parseStatementNode();
      } catch (IncompleteElementException e) {
        configuration.addIncompleteStatement(statementParser);
      }
    }
  }
```

XMLStatementBuilder.parseStatementNode()方法如下：

```
  public void parseStatementNode() {
	//获取sql节点的id
    String id = context.getStringAttribute("id");
    String databaseId = context.getStringAttribute("databaseId");

    if (!databaseIdMatchesCurrent(id, databaseId, this.requiredDatabaseId)) {
      return;
    }
    /*获取sql节点的各种属性*/
    Integer fetchSize = context.getIntAttribute("fetchSize");
    Integer timeout = context.getIntAttribute("timeout");
    String parameterMap = context.getStringAttribute("parameterMap");
    String parameterType = context.getStringAttribute("parameterType");
    Class<?> parameterTypeClass = resolveClass(parameterType);
    String resultMap = context.getStringAttribute("resultMap");
    String resultType = context.getStringAttribute("resultType");
    String lang = context.getStringAttribute("lang");
    LanguageDriver langDriver = getLanguageDriver(lang);

    Class<?> resultTypeClass = resolveClass(resultType);
    String resultSetType = context.getStringAttribute("resultSetType");
    StatementType statementType = StatementType.valueOf(context.getStringAttribute("statementType", StatementType.PREPARED.toString()));
    ResultSetType resultSetTypeEnum = resolveResultSetType(resultSetType);

    
    
    //根据sql节点的名称获取SqlCommandType（INSERT, UPDATE, DELETE, SELECT）
    String nodeName = context.getNode().getNodeName();
    SqlCommandType sqlCommandType = SqlCommandType.valueOf(nodeName.toUpperCase(Locale.ENGLISH));
    boolean isSelect = sqlCommandType == SqlCommandType.SELECT;
    boolean flushCache = context.getBooleanAttribute("flushCache", !isSelect);
    boolean useCache = context.getBooleanAttribute("useCache", isSelect);
    boolean resultOrdered = context.getBooleanAttribute("resultOrdered", false);

    // Include Fragments before parsing
    //在解析sql语句之前先解析<include>节点
    XMLIncludeTransformer includeParser = new XMLIncludeTransformer(configuration, builderAssistant);
    includeParser.applyIncludes(context.getNode());

    // Parse selectKey after includes and remove them.
    //在解析sql语句之前，处理<selectKey>子节点，并在xml节点中删除
    processSelectKeyNodes(id, parameterTypeClass, langDriver);
    
    // Parse the SQL (pre: <selectKey> and <include> were parsed and removed)
    //解析sql语句是解析mapper.xml的核心，实例化sqlSource，使用sqlSource封装sql语句
    SqlSource sqlSource = langDriver.createSqlSource(configuration, context, parameterTypeClass);
    String resultSets = context.getStringAttribute("resultSets");//获取resultSets属性
    String keyProperty = context.getStringAttribute("keyProperty");//获取主键信息keyProperty
    String keyColumn = context.getStringAttribute("keyColumn");///获取主键信息keyColumn
    
    //根据<selectKey>获取对应的SelectKeyGenerator的id
    KeyGenerator keyGenerator;
    String keyStatementId = id + SelectKeyGenerator.SELECT_KEY_SUFFIX;
    keyStatementId = builderAssistant.applyCurrentNamespace(keyStatementId, true);
    
    
    //获取keyGenerator对象，如果是insert类型的sql语句，会使用KeyGenerator接口获取数据库生产的id；
    if (configuration.hasKeyGenerator(keyStatementId)) {
      keyGenerator = configuration.getKeyGenerator(keyStatementId);
    } else {
      keyGenerator = context.getBooleanAttribute("useGeneratedKeys",
          configuration.isUseGeneratedKeys() && SqlCommandType.INSERT.equals(sqlCommandType))
          ? Jdbc3KeyGenerator.INSTANCE : NoKeyGenerator.INSTANCE;
    }

    //通过builderAssistant实例化MappedStatement，并注册至configuration对象
    builderAssistant.addMappedStatement(id, sqlSource, statementType, sqlCommandType,
        fetchSize, timeout, parameterMap, parameterTypeClass, resultMap, resultTypeClass,
        resultSetTypeEnum, flushCache, useCache, resultOrdered, 
        keyGenerator, keyProperty, keyColumn, databaseId, langDriver, resultSets);
  }
```

最后通过MapperBuilderAssistant.addMappedStatement构造MappedStatement，并注册到configuration对象中去。

```
public final class MappedStatement {

  private String resource;//节点的完整的id属性，包括命名空间
  private Configuration configuration;
  private String id;//节点的id属性
  private Integer fetchSize;//节点的fetchSize属性,查询数据的条数
  private Integer timeout;//节点的timeout属性，超时时间
  private StatementType statementType;//节点的statementType属性,默认值：StatementType.PREPARED;疑问？
  private ResultSetType resultSetType;//节点的resultSetType属性,jdbc知识
  private SqlSource sqlSource;//节点中sql语句信息
  private Cache cache;//对应的二级缓存
  private ParameterMap parameterMap;//已废弃
  private List<ResultMap> resultMaps;//节点的resultMaps属性
  private boolean flushCacheRequired;//节点的flushCache属性是否刷新缓存
  private boolean useCache;//节点的useCache属性是否使用二级缓存
  private boolean resultOrdered;
  private SqlCommandType sqlCommandType;//sql语句的类型，包括：INSERT, UPDATE, DELETE, SELECT
  private KeyGenerator keyGenerator;//节点keyGenerator属性
  private String[] keyProperties;
  private String[] keyColumns;
  private boolean hasNestedResultMaps;//是否有嵌套resultMap
  private String databaseId;
  private Log statementLog;
  private LanguageDriver lang;
  private String[] resultSets;//多结果集使用
```

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/mappedStatement.jpg" width="620px" > </div><br>

**step2.configuration.addLoadedResource(resource)**

将mapper文件添加到configuration.loadedResources中


**step3.bindMapperForNamespace()注册mapper接口**

```
//注册mapper接口
  private void bindMapperForNamespace() {
	//获取命名空间
    String namespace = builderAssistant.getCurrentNamespace();
    if (namespace != null) {
      Class<?> boundType = null;
      try {
    	//通过命名空间获取mapper接口的class对象
        boundType = Resources.classForName(namespace);
      } catch (ClassNotFoundException e) {
        //ignore, bound type is not required
      }
      if (boundType != null) {
        if (!configuration.hasMapper(boundType)) {//是否已经注册过该mapper接口？
          // Spring may not know the real resource name so we set a flag
          // to prevent loading again this resource from the mapper interface
          // look at MapperAnnotationBuilder#loadXmlResource
          //将命名空间添加至configuration.loadedResource集合中
          configuration.addLoadedResource("namespace:" + namespace);
          //将mapper接口添加到mapper注册中心
          configuration.addMapper(boundType);
        }
      }
    }
  }

```

```
  public <T> void addMapper(Class<T> type) {
    mapperRegistry.addMapper(type);
  }
```
MapperRegistry.addMapper()方法：

```
//将mapper接口的工厂类添加到mapper注册中心
  public <T> void addMapper(Class<T> type) {
    if (type.isInterface()) {
        if (hasMapper(type)) {
          throw new BindingException("Type " + type + " is already known to the MapperRegistry.");
        }
      boolean loadCompleted = false;
      try {
    	//实例化Mapper接口的代理工程类，并将信息添加至knownMappers
        knownMappers.put(type, new MapperProxyFactory<T>(type));
        // It's important that the type is added before the parser is run
        // otherwise the binding may automatically be attempted by the
        // mapper parser. If the type is already known, it won't try.
        //解析接口上的注解信息，并添加至configuration对象
        MapperAnnotationBuilder parser = new MapperAnnotationBuilder(config, type);
        parser.parse();
        loadCompleted = true;
      } finally {
        if (!loadCompleted) {
          knownMappers.remove(type);
        }
      }
    }
  }

```

其中MapperRegistry如下：

```
public class MapperRegistry {

  private final Configuration config;//config对象，mybatis全局唯一的
  //记录了mapper接口与对应MapperProxyFactory之间的关系
  private final Map<Class<?>, MapperProxyFactory<?>> knownMappers = new HashMap<>();

```

### 1.2.4 三种builder总结

* XMLConfigBuilder： 主要负责解析mybatis-config.xml；
* XMLMapperBuilder： 主要负责解析映射配置文件；
* XMLStatementBuilder： 主要负责解析映射配置文件中的SQL节点。

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/XMLConfigBuilder.png" width="520px" > </div><br>

三种Builder继承自BaseBuilder，因此都由Configuration的引用。所以都会将各自解析的部分注册到Configuration中去。

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/Mybatis配置加载建造者模式.png" width="520px" > </div><br>

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/所有Builder都将信息汇总到Configuration.jpg" width="620px" > </div><br>

### 1.2.5 Configuration详解

Configuration包含mybatis-config.xml中所有配置信息。

```
public class Configuration {

  protected Environment environment;

  /* 是否启用行内嵌套语句**/
  protected boolean safeRowBoundsEnabled;
  protected boolean safeResultHandlerEnabled = true;
  /* 是否启用数据组A_column自动映射到Java类中的驼峰命名的属性**/
  protected boolean mapUnderscoreToCamelCase;
  
  /*当对象使用延迟加载时 属性的加载取决于能被引用到的那些延迟属性,否则,按需加载(需要的是时候才去加载)**/
  protected boolean aggressiveLazyLoading;
  
  /*是否允许单条sql 返回多个数据集  (取决于驱动的兼容性) default:true **/
  protected boolean multipleResultSetsEnabled = true;
  
  /*-允许JDBC 生成主键。需要驱动器支持。如果设为了true，这个设置将强制使用被生成的主键，有一些驱动器不兼容不过仍然可以执行。  default:false**/
  protected boolean useGeneratedKeys;
  
  /* 使用列标签代替列名。不同的驱动在这方面会有不同的表现， 具体可参考相关驱动文档或通过测试这两种不同的模式来观察所用驱动的结果。**/
  protected boolean useColumnLabel = true;
  
  /*配置全局性的cache开关，默认为true**/
  protected boolean cacheEnabled = true;
  protected boolean callSettersOnNulls;
  protected boolean useActualParamName = true;
  protected boolean returnInstanceForEmptyRow;

  /* 日志打印所有的前缀 **/
  protected String logPrefix;
  
  /* 指定 MyBatis 所用日志的具体实现，未指定时将自动查找**/
  protected Class <? extends Log> logImpl;
  protected Class <? extends VFS> vfsImpl;
  /* 设置本地缓存范围，session：就会有数据的共享，statement：语句范围，这样不会有数据的共享**/
  protected LocalCacheScope localCacheScope = LocalCacheScope.SESSION;
  /* 设置但JDBC类型为空时,某些驱动程序 要指定值**/
  protected JdbcType jdbcTypeForNull = JdbcType.OTHER;
  
  /* 设置触发延迟加载的方法**/
  protected Set<String> lazyLoadTriggerMethods = new HashSet<>(Arrays.asList("equals", "clone", "hashCode", "toString"));
  
  /* 设置驱动等待数据响应超时数**/
  protected Integer defaultStatementTimeout;
  
  /* 设置驱动返回结果数的大小**/
  protected Integer defaultFetchSize;
  
  /* 执行类型，有simple、reuse及batch**/
  protected ExecutorType defaultExecutorType = ExecutorType.SIMPLE;
  
  /*指定 MyBatis 应如何自动映射列到字段或属性*/
  protected AutoMappingBehavior autoMappingBehavior = AutoMappingBehavior.PARTIAL;
  protected AutoMappingUnknownColumnBehavior autoMappingUnknownColumnBehavior = AutoMappingUnknownColumnBehavior.NONE;

  protected Properties variables = new Properties();
  
  protected ReflectorFactory reflectorFactory = new DefaultReflectorFactory();
  
  /*MyBatis每次创建结果对象的新实例时，它都会使用对象工厂（ObjectFactory）去构建POJO*/
  protected ObjectFactory objectFactory = new DefaultObjectFactory();
  protected ObjectWrapperFactory objectWrapperFactory = new DefaultObjectWrapperFactory();

  /*延迟加载的全局开关*/
  protected boolean lazyLoadingEnabled = false;
  
  /*指定 Mybatis 创建具有延迟加载能力的对象所用到的代理工具*/
  protected ProxyFactory proxyFactory = new JavassistProxyFactory(); // #224 Using internal Javassist instead of OGNL

  protected String databaseId;
  /**
   * Configuration factory class.
   * Used to create Configuration for loading deserialized unread properties.
   *
   * @see <a href='https://code.google.com/p/mybatis/issues/detail?id=300'>Issue 300 (google code)</a>
   */
  protected Class<?> configurationFactory;
  
  /*插件集合*/
  protected final InterceptorChain interceptorChain = new InterceptorChain();
  
  /*TypeHandler注册中心*/
  protected final TypeHandlerRegistry typeHandlerRegistry = new TypeHandlerRegistry();
  
  /*TypeAlias注册中心*/
  protected final TypeAliasRegistry typeAliasRegistry = new TypeAliasRegistry();
  protected final LanguageDriverRegistry languageRegistry = new LanguageDriverRegistry();
  //-------------------------------------------------------------

  /*mapper接口的动态代理注册中心*/
  protected final MapperRegistry mapperRegistry = new MapperRegistry(this);

  /*mapper文件中增删改查操作的注册中心*/
  protected final Map<String, MappedStatement> mappedStatements = new StrictMap<>("Mapped Statements collection");
  
  /*mapper文件中配置cache节点的 二级缓存*/
  protected final Map<String, Cache> caches = new StrictMap<>("Caches collection");
  
  /*mapper文件中配置的所有resultMap对象  key为命名空间+ID*/
  protected final Map<String, ResultMap> resultMaps = new StrictMap<>("Result Maps collection");
  protected final Map<String, ParameterMap> parameterMaps = new StrictMap<>("Parameter Maps collection");
  
  /*mapper文件中配置KeyGenerator的insert和update节点，key为命名空间+ID*/
  protected final Map<String, KeyGenerator> keyGenerators = new StrictMap<>("Key Generators collection");

  /*加载到的所有*mapper.xml文件*/
  protected final Set<String> loadedResources = new HashSet<>();
  
  /*mapper文件中配置的sql元素，key为命名空间+ID*/
  protected final Map<String, XNode> sqlFragments = new StrictMap<>("XML fragments parsed from previous mappers");

  protected final Collection<XMLStatementBuilder> incompleteStatements = new LinkedList<>();
  protected final Collection<CacheRefResolver> incompleteCacheRefs = new LinkedList<>();
  protected final Collection<ResultMapResolver> incompleteResultMaps = new LinkedList<>();
  protected final Collection<MethodResolver> incompleteMethods = new LinkedList<>();

  /*
   * A map holds cache-ref relationship. The key is the namespace that
   * references a cache bound to another namespace and the value is the
   * namespace which the actual cache is bound to.
   */
  protected final Map<String, String> cacheRefMap = new HashMap<>();
```

关键类：
* Configuration ： Mybatis启动初始化的核心就是将所有xml配置文件信息加载到Configuration对象中， Configuration是单例的，生命周期是应用级的；
* MapperRegistry：mapper接口动态代理工厂类的注册中心。在MyBatis中，通过mapperProxy实现InvocationHandler接口，MapperProxyFactory用于生成动态代理的实例对象；
* ResultMap：用于解析mapper.xml文件中的resultMap节点，使用ResultMapping来封装id，result等子元素；
* MappedStatement：用于存储mapper.xml文件中的select、insert、update和delete节点，同时还包含了这些节点的很多重要属性；
* SqlSource：mapper.xml文件中的sql语句会被解析成SqlSource对象，经过解析SqlSource包含的语句最终仅仅包含？占位符，可以直接提交给数据库执行

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/配置文件与Configuration映射关系.jpg" width="620px" > </div><br>

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/Mybatis初始化.png" width="820px" > </div><br>


# 2.代理阶段——binding模块分析

## 2.1 为什么使用mapper接口就能操作数据库？

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/动态代理增强.jpg" width="620px" > </div><br>

* 找到session中对应的方法执行 ——> MapperMethod.SqlCommand. type +MapperMethod.MethodSignature.returnType
* 找到命名空间和方法名 ——> MapperMethod.SqlCommand. name
* 传递参数 ——> MapperMethod.MethodSignature.paramNameResolver

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/selectOne.jpg" width="520px" > </div><br>

```
    @Test
    public void quickStart() {
        SqlSession sqlSession = sqlSessionFactory.openSession();
        TUserMapperQuickStart mapper = sqlSession.getMapper(TUserMapperQuickStart.class);
        TUserQuickStart user = mapper.selectByPrimaryKey(1);
        System.out.println(user);
    }
```

DefaultSqlSession.getMapper()：

```
  @Override
  public <T> T getMapper(Class<T> type) {
    return configuration.<T>getMapper(type, this);
  }
```

Configuration.getMapper()：

```
  public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    return mapperRegistry.getMapper(type, sqlSession);
  }
```

MapperRegistry.getMapper()：

```
  public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
    if (mapperProxyFactory == null) {
      throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
    }
    try {
      return mapperProxyFactory.newInstance(sqlSession);
    } catch (Exception e) {
      throw new BindingException("Error getting mapper instance. Cause: " + e, e);
    }
  }
```

可知mapper最后返回的是实现了TUserMapperQuickStart接口的代理类对象。

* step1.该代理类调用mapper.selectByPrimaryKey会转调至MapperProxy.invoke方法
* step2.invoke方法会转调至MapperMethod.execute(sqlSession, args);
* step3.MapperMethod.execute会转调至SqlSession.selectOne()方法
* step4.selectOne(String statement, Object parameter)最后统一调用selectList(statement, parameter)



## 2.2 MapperRegistry

衔接第一部分中的mapper阶段的step3.bindMapperForNamespace()注册mapper接口。

MapperRegistry.addMapper()方法如下：

```
//将mapper接口的工厂类添加到mapper注册中心
  public <T> void addMapper(Class<T> type) {
    if (type.isInterface()) {
        if (hasMapper(type)) {
          throw new BindingException("Type " + type + " is already known to the MapperRegistry.");
        }
      boolean loadCompleted = false;
      try {
    	//实例化Mapper接口的代理工程类，并将信息添加至knownMappers
        knownMappers.put(type, new MapperProxyFactory<T>(type));
        // It's important that the type is added before the parser is run
        // otherwise the binding may automatically be attempted by the
        // mapper parser. If the type is already known, it won't try.
        //解析接口上的注解信息，并添加至configuration对象
        MapperAnnotationBuilder parser = new MapperAnnotationBuilder(config, type);
        parser.parse();
        loadCompleted = true;
      } finally {
        if (!loadCompleted) {
          knownMappers.remove(type);
        }
      }
    }
  }
```

## 2.3 MapperProxyFactory

```
public class MapperProxyFactory<T> {

  //mapper接口的class对象
  private final Class<T> mapperInterface;
//key是mapper接口中的某个方法的method对象，value是对应的MapperMethod，MapperMethod对象不记录任何状态信息，所以它可以在多个代理对象之间共享
  private final Map<Method, MapperMethod> methodCache = new ConcurrentHashMap<>();

  public MapperProxyFactory(Class<T> mapperInterface) {
    this.mapperInterface = mapperInterface;
  }

  public Class<T> getMapperInterface() {
    return mapperInterface;
  }

  public Map<Method, MapperMethod> getMethodCache() {
    return methodCache;
  }

  @SuppressWarnings("unchecked")
  protected T newInstance(MapperProxy<T> mapperProxy) {
	//创建实现了mapper接口的动态代理对象
    return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
  }

  public T newInstance(SqlSession sqlSession) {
	 //每次调用都会创建新的MapperProxy对象
    final MapperProxy<T> mapperProxy = new MapperProxy<>(sqlSession, mapperInterface, methodCache);
    return newInstance(mapperProxy);
  }

```

其newInstance方法会创建mapperInterface的代理对象，该代理对象的InvocationHandler是MapperProxy。这里就是动态代理的核心。

## 2.4 MapperProxy

```
public class MapperProxy<T> implements InvocationHandler, Serializable {

  private static final long serialVersionUID = -6424540398559729838L;
  private final SqlSession sqlSession;//记录关联的sqlsession对象
  private final Class<T> mapperInterface;//mapper接口对应的class对象；
//key是mapper接口中的某个方法的method对象，value是对应的MapperMethod，MapperMethod对象不记录任何状态信息，所以它可以在多个代理对象之间共享
  private final Map<Method, MapperMethod> methodCache;

  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      if (Object.class.equals(method.getDeclaringClass())) {//如果是Object本身的方法不增强
        return method.invoke(this, args);
      } else if (isDefaultMethod(method)) {
        return invokeDefaultMethod(proxy, method, args);
      }
    } catch (Throwable t) {
      throw ExceptionUtil.unwrapThrowable(t);
    }
    //从缓存中获取mapperMethod对象，如果缓存中没有，则创建一个，并添加到缓存中
    final MapperMethod mapperMethod = cachedMapperMethod(method);
    //调用execute方法执行sql
    return mapperMethod.execute(sqlSession, args);
  }

```

```
  private MapperMethod cachedMapperMethod(Method method) {
    return methodCache.computeIfAbsent(method, k -> new MapperMethod(mapperInterface, method, sqlSession.getConfiguration()));
  }
```

## 2.5 最终的执行落到了MapperMethod

MapperMethod：封装了Mapper接口中对应方法的信息，以及对应的sql语句的信息；它是mapper接口与映射配置文件中sql语句的桥梁； MapperMethod对象不记录任何状态信息，所以它可以在多个代理对象之间共享；
* SqlCommand ： 从configuration中获取方法的命名空间.方法名以及SQL语句的类型；
* MethodSignature：封装mapper接口方法的相关信息（入参，返回类型）；
* ParamNameResolver： 解析mapper接口方法中的入参.


```
public class MapperMethod {
  //从configuration中获取方法的命名空间.方法名以及SQL语句的类型
  private final SqlCommand command;
  //封装mapper接口方法的相关信息（入参，返回类型）；
  private final MethodSignature method;

  public MapperMethod(Class<?> mapperInterface, Method method, Configuration config) {
    this.command = new SqlCommand(config, mapperInterface, method);
    this.method = new MethodSignature(config, mapperInterface, method);
  }
```
来瞧一下MapperMethod.SqlCommand：

```
  public static class SqlCommand {
	//sql的名称，命名空间+方法名称
    private final String name;
    //获取sql语句的类型
    private final SqlCommandType type;

    public SqlCommand(Configuration configuration, Class<?> mapperInterface, Method method) {
      final String methodName = method.getName();//获取方法名称
      final Class<?> declaringClass = method.getDeclaringClass();
      //从configuration中获取mappedStatement
      MappedStatement ms = resolveMappedStatement(mapperInterface, methodName, declaringClass,
          configuration);
      if (ms == null) {
        if(method.getAnnotation(Flush.class) != null){
          name = null;
          type = SqlCommandType.FLUSH;
        } else {
          throw new BindingException("Invalid bound statement (not found): "
              + mapperInterface.getName() + "." + methodName);
        }
      } else {//如果mappedStatement不为空
        name = ms.getId();//获取sql的名称，命名空间+方法名称
        type = ms.getSqlCommandType();//获取sql语句的类型
        if (type == SqlCommandType.UNKNOWN) {
          throw new BindingException("Unknown execution method for: " + name);
        }
      }
    }

```

```
public enum SqlCommandType {
  UNKNOWN, INSERT, UPDATE, DELETE, SELECT, FLUSH;
}

```

再来瞧一下MapperMethod.MethodSignature：

```
  public static class MethodSignature {

    private final boolean returnsMany;//返回参数是否为集合类型或数组
    private final boolean returnsMap;//返回参数是否为map
    private final boolean returnsVoid;//返回值为空
    private final boolean returnsCursor;//返回值是否为游标类型
    private final boolean returnsOptional;//返回值是否为Optional
    private final Class<?> returnType;//返回值类型
    private final String mapKey;
    private final Integer resultHandlerIndex;
    private final Integer rowBoundsIndex;
    private final ParamNameResolver paramNameResolver;//该方法的参数解析器

    public MethodSignature(Configuration configuration, Class<?> mapperInterface, Method method) {
      //通过类型解析器获取方法的返回值类型
      Type resolvedReturnType = TypeParameterResolver.resolveReturnType(method, mapperInterface);
      if (resolvedReturnType instanceof Class<?>) {
        this.returnType = (Class<?>) resolvedReturnType;
      } else if (resolvedReturnType instanceof ParameterizedType) {
        this.returnType = (Class<?>) ((ParameterizedType) resolvedReturnType).getRawType();
      } else {
        this.returnType = method.getReturnType();
      }
      //初始化返回值等字段
      this.returnsVoid = void.class.equals(this.returnType);
      this.returnsMany = configuration.getObjectFactory().isCollection(this.returnType) || this.returnType.isArray();
      this.returnsCursor = Cursor.class.equals(this.returnType);
      this.returnsOptional = Jdk.optionalExists && Optional.class.equals(this.returnType);
      this.mapKey = getMapKey(method);
      this.returnsMap = this.mapKey != null;
      this.rowBoundsIndex = getUniqueParamIndex(method, RowBounds.class);
      this.resultHandlerIndex = getUniqueParamIndex(method, ResultHandler.class);
      this.paramNameResolver = new ParamNameResolver(configuration, method);
    }
```
最终调用MapperMethod.execute()方法：

```
  public Object execute(SqlSession sqlSession, Object[] args) {
    Object result;
    //根据sql语句类型以及接口返回的参数选择调用不同的
    switch (command.getType()) {
      case INSERT: {
    	Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.insert(command.getName(), param));
        break;
      }
      case UPDATE: {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.update(command.getName(), param));
        break;
      }
      case DELETE: {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.delete(command.getName(), param));
        break;
      }
      case SELECT:
        if (method.returnsVoid() && method.hasResultHandler()) {//返回值为void
          executeWithResultHandler(sqlSession, args);
          result = null;
        } else if (method.returnsMany()) {//返回值为集合或者数组
          result = executeForMany(sqlSession, args);
        } else if (method.returnsMap()) {//返回值为map
          result = executeForMap(sqlSession, args);
        } else if (method.returnsCursor()) {//返回值为游标
          result = executeForCursor(sqlSession, args);
        } else {//处理返回为单一对象的情况
          //通过参数解析器解析解析参数
          Object param = method.convertArgsToSqlCommandParam(args);
          result = sqlSession.selectOne(command.getName(), param);
          if (method.returnsOptional() &&
              (result == null || !method.getReturnType().equals(result.getClass()))) {
            result = OptionalUtil.ofNullable(result);
          }
        }
        break;
      case FLUSH:
        result = sqlSession.flushStatements();
        break;
      default:
        throw new BindingException("Unknown execution method for: " + command.getName());
    }
    if (result == null && method.getReturnType().isPrimitive() && !method.returnsVoid()) {
      throw new BindingException("Mapper method '" + command.getName() 
          + " attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
    }
    return result;
  }

```

最终调用至SqlSession的方法。


## 2.6 总结

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/binding模块.png" width="420px" > </div><br>

* MapperRegistry ： mapper接口和对应的代理对象工厂的注册中心；
* MapperProxyFactory：用于生成mapper接口动态代理的实例对象；
* MapperProxy：实现了InvocationHandler接口，它是增强mapper接口的实现；
* MapperMethod：封装了Mapper接口中对应方法的信息，以及对应的sql语句的信息；它是mapper接口与映射配置文件中sql语句的桥梁


# 3.数据读写阶段——Mybatis接口层

## 3.1 策略模式

策略模式（Strategy Pattern）策略模式定义了一系列的算法，并将每一个算法封装起来，而且使他们可以相互替换，让算法独立于使用它的客户而独立变化。

策略模式的使用场景：

* 针对同一类型问题的多种处理方式，仅仅是具体行为有差别时； 
* 出现同一抽象类有多个子类，而又需要使用 if-else 或者 switch-case 来选择具体子类时。

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/策略模式.png" width="420px" > </div><br>

策略模式的使用：
* 通过配置文件来加载数据源的不同实现（UNPOOLED、POOLED、JNDI）
* Spring中通过@注解来更换不同的实现

## 3.2 SqlSession

SqlSession是MyBaits对外提供的最关键的核心接口，通过它可以执行数据库读写命令、获取映射器、管理事务等。

```
public class DefaultSqlSession implements SqlSession {

  private final Configuration configuration;//configuration对象，全局唯一
  private final Executor executor;//底层依赖的excutor对象

  private final boolean autoCommit;//是否自动提交事务
  private boolean dirty;//当前缓存是否有脏数据
  private List<Cursor<?>> cursorList;
```

样例代码：

```
    @Test
    public void quickStart() {
        SqlSession sqlSession = sqlSessionFactory.openSession();
        TUserMapperQuickStart mapper = sqlSession.getMapper(TUserMapperQuickStart.class);
        TUserQuickStart user = mapper.selectByPrimaryKey(1);
        System.out.println(user);
    }
```

DefaultSqlSessionFactory.openSession()方法如下：

```
  @Override
  public SqlSession openSession() {
    return openSessionFromDataSource(configuration.getDefaultExecutorType(), null, false);
  }


  //从数据源获取数据库连接
  private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
    Transaction tx = null;
    try {
    	//获取mybatis配置文件中的environment对象
      final Environment environment = configuration.getEnvironment();
      //从environment获取transactionFactory对象
      final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
      //创建事务对象
      tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
      //根据配置创建executor
      final Executor executor = configuration.newExecutor(tx, execType);
      //创建DefaultSqlSession
      return new DefaultSqlSession(configuration, executor, autoCommit);
    } catch (Exception e) {
      closeTransaction(tx); // may have fetched a connection so lets call close()
      throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }

```

Configuration.newExecutor()如下：

```
  public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
    executorType = executorType == null ? defaultExecutorType : executorType;
    executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
    Executor executor;
    if (ExecutorType.BATCH == executorType) {
      executor = new BatchExecutor(this, transaction);
    } else if (ExecutorType.REUSE == executorType) {
      executor = new ReuseExecutor(this, transaction);
    } else {
      executor = new SimpleExecutor(this, transaction);
    }
    //如果有<cache>节点，通过装饰器，添加二级缓存的能力
    if (cacheEnabled) {
      executor = new CachingExecutor(executor);
    }
    //通过interceptorChain遍历所有的插件为executor增强，添加插件的功能
    executor = (Executor) interceptorChain.pluginAll(executor);
    return executor;
  }
```


<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/SqlSession.png" width="620px" > </div><br>

SqlSession查询接口嵌套关系如下：

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/SqlSession查询接口嵌套关系.jpg" width="720px" > </div><br>

DefaultSqlSession：

```
  public <T> T selectOne(String statement, Object parameter) {
    // Popular vote was to return null on 0 results and throw exception on too many.
    List<T> list = this.<T>selectList(statement, parameter);
    if (list.size() == 1) {
      return list.get(0);
    } else if (list.size() > 1) {
      throw new TooManyResultsException("Expected one result (or null) to be returned by selectOne(), but found: " + list.size());
    } else {
      return null;
    }
  }


  @Override
  public <E> List<E> selectList(String statement, Object parameter) {
    return this.selectList(statement, parameter, RowBounds.DEFAULT);
  }


  @Override
  public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
    try {
      //从configuration中获取要执行的sql语句的配置信息
      MappedStatement ms = configuration.getMappedStatement(statement);
      //通过executor执行语句，并返回指定的结果集
      return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
```

最终都是通过Executor来执行。

# 4.核心组件Executor

## 4.1 模板模式

一个抽象类公开定义了执行它的方法的方式/模板。它的子类可以按需要重写方法实现，但调用将以抽象类中定义的方式进行。定义一个操作中的算法的骨架，而将一些步骤延迟到子类中。模板方法使得子类可以不改变一个算法的结构即可重定义该算法的某些特定实现。

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/模板模式.png" width="520px" > </div><br>

使用场景：遇到由一系列步骤构成的过程需要执行，这个过程从高层次上看是相同的，但是有些步骤的实现可能不同，这个时候就需要考虑用模板模式了。比如：Executor查询操作流程。

## 4.2 Executor
Executor是MyBaits核心接口之一，定义了数据库操作最基本的方法，SqlSession的功能都是基于它来实现的。

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/Executor.png" width="520px" > </div><br>

BaseExecutor：抽象类，实现了executor接口的大部分方法，主要提供了缓存管理和事务管理的能力，其他子类需要实现的抽象方法为：doUpdate,doQuery等方法。

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/BaseExecutor.query.png" width="620px" > </div><br>


调用链如下：

```
  public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
    try {
      //从configuration中获取要执行的sql语句的配置信息
      MappedStatement ms = configuration.getMappedStatement(statement);
      //通过executor执行语句，并返回指定的结果集
      return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
```

CachingExecutor.query()方法——二级缓存：

```
  @Override
  public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
	//获取sql语句信息，包括占位符，参数等信息
    BoundSql boundSql = ms.getBoundSql(parameterObject);
  //拼装缓存的key值
    CacheKey key = createCacheKey(ms, parameterObject, rowBounds, boundSql);
    return query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }

  @Override
  public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
      throws SQLException {
	//从MappedStatement中获取二级缓存
    Cache cache = ms.getCache();
    if (cache != null) {
      flushCacheIfRequired(ms);
      if (ms.isUseCache() && resultHandler == null) {
        ensureNoOutParams(ms, boundSql);
        @SuppressWarnings("unchecked")
        List<E> list = (List<E>) tcm.getObject(cache, key);//从二级缓存中获取数据
        if (list == null) {
          //二级缓存为空，才会调用BaseExecutor.query
          list = delegate.<E> query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
          tcm.putObject(cache, key, list); // issue #578 and #116
        }
        return list;
      }
    }
    return delegate.<E> query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }

```

BaseExecutor.query()方法——一级缓存：

```
  @Override
  public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
    if (closed) {//检查当前executor是否关闭
      throw new ExecutorException("Executor was closed.");
    }
    if (queryStack == 0 && ms.isFlushCacheRequired()) {//非嵌套查询，并且FlushCache配置为true，则需要清空一级缓存
      clearLocalCache();
    }
    List<E> list;
    try {
      queryStack++;//查询层次加一
      list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;//查询以及缓存
      if (list != null) {
    	 //针对调用存储过程的结果处理
        handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
      } else {
    	 //缓存未命中，从数据库加载数据
        list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
      }
    } finally {
      queryStack--;
    }
    
    
    if (queryStack == 0) {
      for (DeferredLoad deferredLoad : deferredLoads) {//延迟加载处理
        deferredLoad.load();
      }
      // issue #601
      deferredLoads.clear();
      if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {//如果当前sql的一级缓存配置为STATEMENT，查询完既清空一集缓存
        // issue #482
        clearLocalCache();
      }
    }
    return list;
  }





  //真正访问数据库获取结果的方法
  private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    List<E> list;
    localCache.putObject(key, EXECUTION_PLACEHOLDER);//在缓存中添加占位符
    try {
      //调用抽象方法doQuery，方法查询数据库并返回结果，可选的实现包括：simple、reuse、batch
      list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
    } finally {
      localCache.removeObject(key);//在缓存中删除占位符
    }
    localCache.putObject(key, list);//将真正的结果对象添加到一级缓存
    if (ms.getStatementType() == StatementType.CALLABLE) {//如果是调用存储过程
      localOutputParameterCache.putObject(key, parameter);//缓存输出类型结果参数
    }
    return list;
  }
```

这里的doQuery()就是子类需要实现的，比如这里是SimpleExecutor.doQuery()：

```
  @Override
  //查询的实现
  public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
    Statement stmt = null;
    try {
      Configuration configuration = ms.getConfiguration();//获取configuration对象
      //创建StatementHandler对象，
      StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
      //StatementHandler对象创建stmt,并使用parameterHandler对占位符进行处理
      stmt = prepareStatement(handler, ms.getStatementLog());
      //通过statementHandler对象调用ResultSetHandler将结果集转化为指定对象返回
      return handler.<E>query(stmt, resultHandler);
    } finally {
      closeStatement(stmt);
    }
  }
```

## 4.3 Executor的三个实现类解读

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/SimpleExecutor.png" width="820px" > </div><br>

* SimpleExecutor：默认配置，使用PrepareStatement对象访问数据库，每次访问都要创建新的PrepareStatement对象；
* ReuseExecutor：使用预编译PrepareStatement对象访问数据库，访问时，会重用缓存中的statement对象；
* BatchExecutor：实现批量执行多条SQL语句的能力

通过对SimpleExecutor doQuery()方法的解读发现，Executor是个指挥官，它在调度三个小弟工作：

* StatementHandler：它的作用是使用数据库的Statement或PrepareStatement执行操作，启承上启下作用；
* ParameterHandler：对预编译的SQL语句进行参数设置，SQL语句中的的占位符“？”都对应BoundSql.parameterMappings集合中的一个元素，在该对象中记录了对应的参数名称以及该参数的相关属性
* ResultSetHandler：对数据库返回的结果集（ResultSet）进行封装，返回用户指定的实体类型


<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/Executor内部运作过程.png" width="620px" > </div><br>


```
  @Override
  //查询的实现
  public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
    Statement stmt = null;
    try {
      Configuration configuration = ms.getConfiguration();//获取configuration对象
      //创建StatementHandler对象，
      StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
      //StatementHandler对象创建stmt,并使用parameterHandler对占位符进行处理
      stmt = prepareStatement(handler, ms.getStatementLog());
      //通过statementHandler对象调用ResultSetHandler将结果集转化为指定对象返回
      return handler.<E>query(stmt, resultHandler);
    } finally {
      closeStatement(stmt);
    }
  }



  //创建Statement
  private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
    Statement stmt;
    //获取connection对象的动态代理，添加日志能力；
    Connection connection = getConnection(statementLog);
    //通过不同的StatementHandler，利用connection创建（prepare）Statement
    stmt = handler.prepare(connection, transaction.getTimeout());
    //使用parameterHandler处理占位符
    handler.parameterize(stmt);
    return stmt;
  }
```

这里返回的PreparedStatement为ClientPreparedStatement：

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/ClientPreparedStatement.png" width="520px" > </div><br>

调用至BaseStatementHandler.prepare()方法：

```
  @Override
  //使用模板模式，定义了获取Statement的步骤，其子类实现实例化Statement的具体的方式；
  public Statement prepare(Connection connection, Integer transactionTimeout) throws SQLException {
    ErrorContext.instance().sql(boundSql.getSql());
    Statement statement = null;
    try {
      //通过不同的子类实例化不同的Statement，分为三类：simple(statment)、prepare(prepareStatement)、callable(CallableStatementHandler)
      statement = instantiateStatement(connection);
      //设置超时时间
      setStatementTimeout(statement, transactionTimeout);
      //设置数据集大小
      setFetchSize(statement);
      return statement;
    } catch (SQLException e) {
      closeStatement(statement);
      throw e;
    } catch (Exception e) {
      closeStatement(statement);
      throw new ExecutorException("Error preparing statement.  Cause: " + e, e);
    }
  }
```

调用至PreparedStatementHandler.instantiateStatement()方法：

```
  @Override
  //使用底层的prepareStatement对象来完成对数据库的操作
  protected Statement instantiateStatement(Connection connection) throws SQLException {
    String sql = boundSql.getSql();
    //根据mappedStatement.getKeyGenerator字段，创建prepareStatement
    if (mappedStatement.getKeyGenerator() instanceof Jdbc3KeyGenerator) {//对于insert语句
      String[] keyColumnNames = mappedStatement.getKeyColumns();
      if (keyColumnNames == null) {
    	//返回数据库生成的主键
        return connection.prepareStatement(sql, PreparedStatement.RETURN_GENERATED_KEYS);
      } else {
    	//返回数据库生成的主键填充至keyColumnNames中指定的列
        return connection.prepareStatement(sql, keyColumnNames);
      }
    } else if (mappedStatement.getResultSetType() != null) {
     //设置结果集是否可以滚动以及其游标是否可以上下移动，设置结果集是否可更新
      return connection.prepareStatement(sql, mappedStatement.getResultSetType().getValue(), ResultSet.CONCUR_READ_ONLY);
    } else {
      //创建普通的prepareStatement对象
      return connection.prepareStatement(sql);
    }
  }
```

创建好语句，设置好参数（占位符）后，执行语句查询PreparedStatementHandler.query：

```
  @Override
  public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
    PreparedStatement ps = (PreparedStatement) statement;
    ps.execute();
    return resultSetHandler.<E> handleResultSets(ps);
  }
```

这里的操作流程说明：

* step1.ps.execute()会执行ClientPreparedStatement.execute()，会将执行结果放在StatementImpl.results中
* step2.handleResultSets(ps)的getFirstResultSet(stmt)和getNextResultSet(stmt)会从StatementImpl.results获取结果。

```
public class StatementImpl implements JdbcStatement {

    /** The current results */
    protected ResultSetInternalMethods results = null;
```


执行完以后，调用DefaultResultSetHandler.handleResultSets()处理结果。

```
  @Override
  public List<Object> handleResultSets(Statement stmt) throws SQLException {
    ErrorContext.instance().activity("handling results").object(mappedStatement.getId());
    //用于保存结果集对象
    final List<Object> multipleResults = new ArrayList<>();

    int resultSetCount = 0;
    //statment可能返回多个结果集对象，这里先取出第一个结果集
    ResultSetWrapper rsw = getFirstResultSet(stmt);
    //获取结果集对应resultMap，本质就是获取字段与java属性的映射规则
    List<ResultMap> resultMaps = mappedStatement.getResultMaps();
    int resultMapCount = resultMaps.size();
    validateResultMapsCount(rsw, resultMapCount);//结果集和resultMap不能为空，为空抛出异常
    while (rsw != null && resultMapCount > resultSetCount) {
     //获取当前结果集对应的resultMap
      ResultMap resultMap = resultMaps.get(resultSetCount);
      //根据映射规则（resultMap）对结果集进行转化，转换成目标对象以后放入multipleResults中
      handleResultSet(rsw, resultMap, multipleResults, null);
      rsw = getNextResultSet(stmt);//获取下一个结果集
      cleanUpAfterHandlingResultSet();//清空nestedResultObjects对象
      resultSetCount++;
    }
    //获取多结果集。多结果集一般出现在存储过程的执行，存储过程返回多个resultset，
    //mappedStatement.resultSets属性列出多个结果集的名称，用逗号分割；
    //多结果集的处理不是重点，暂时不分析
    String[] resultSets = mappedStatement.getResultSets();
    if (resultSets != null) {
      while (rsw != null && resultSetCount < resultSets.length) {
        ResultMapping parentMapping = nextResultMaps.get(resultSets[resultSetCount]);
        if (parentMapping != null) {
          String nestedResultMapId = parentMapping.getNestedResultMapId();
          ResultMap resultMap = configuration.getResultMap(nestedResultMapId);
          handleResultSet(rsw, resultMap, null, parentMapping);
        }
        rsw = getNextResultSet(stmt);
        cleanUpAfterHandlingResultSet();
        resultSetCount++;
      }
    }

    return collapseSingleResultList(multipleResults);
  }
```


### 4.3.1 上面整个查询和转换流程等价于jdbc操作流程

```
            String sql;
            sql = "SELECT * FROM t_user where user_name= ? ";
            stmt = conn.prepareStatement(sql);
            stmt.setString(1, "lison");
            System.out.println(stmt.toString());//打印sql
            ResultSet rs = stmt.executeQuery();


            // STEP 5: 从resultSet中获取数据并转化成bean
            while (rs.next()) {
                System.out.println("------------------------------");
                // Retrieve by column name
                TUser user = new TUser();
//				user.setId(rs.getInt("id"));
//				user.setUserName(rs.getString("user_name"));
                user.setRealName(rs.getString("real_name"));
                user.setSex(rs.getByte("sex"));
                user.setMobile(rs.getString("mobile"));
                user.setEmail(rs.getString("email"));
                user.setNote(rs.getString("note"));

                System.out.println(user.toString());

                users.add(user);
            }
```

### 4.3.2 StatementHandler分析

StatementHandler完成Mybatis最核心的工作，也是Executor实现的基础；功能包括：创建statement对象，为sql语句绑定参数，执行增删改查等SQL语句、将结果映射集进行转化。

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/StatementHandler.png" width="520px" > </div><br>


* BaseStatementHandler：所有子类的抽象父类，定义了初始化statement的操作顺序，由子类实现具体的实例化不同的statement（模板模式）；
* RoutingStatementHandler：Excutor组件真正实例化的子类，使用静态
* 代理模式，根据上下文决定创建哪个具体实体类；
* SimpleStatmentHandler ：使用statement对象访问数据库，无须参数化；
* PreparedStatmentHandler ：使用预编译PrepareStatement对象访问数据库；
* CallableStatmentHandler ：调用存储过程


```
public class RoutingStatementHandler implements StatementHandler {

  private final StatementHandler delegate;//底层封装的真正的StatementHandler对象

  public RoutingStatementHandler(Executor executor, MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
    //RoutingStatementHandler最主要的功能就是根据mappedStatment的配置，生成一个对应的StatementHandler对象并赋值给delegate
    switch (ms.getStatementType()) {
      case STATEMENT:
        delegate = new SimpleStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
        break;
      case PREPARED:
        delegate = new PreparedStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
        break;
      case CALLABLE:
        delegate = new CallableStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
        break;
      default:
        throw new ExecutorException("Unknown statement type: " + ms.getStatementType());
    }

  }
```

### 4.3.3 ResultSetHandler分析

ResultSetHandler将从数据库查询得到的结果按照映射配置文件的映射规则，映射成相应的结果集对象。

核心操作在DefaultResultSetHandler.handleResultSets()的handleResultSet中：

```
  private void handleResultSet(ResultSetWrapper rsw, ResultMap resultMap, List<Object> multipleResults, ResultMapping parentMapping) throws SQLException {
    try {
      if (parentMapping != null) {//处理多结果集的嵌套映射
        handleRowValues(rsw, resultMap, null, RowBounds.DEFAULT, parentMapping);
      } else {
        if (resultHandler == null) {//如果resultHandler为空，实例化一个人默认的resultHandler
          DefaultResultHandler defaultResultHandler = new DefaultResultHandler(objectFactory);
          //对ResultSet进行映射，映射结果暂存在resultHandler中
          handleRowValues(rsw, resultMap, defaultResultHandler, rowBounds, null);
          //将暂存在resultHandler中的映射结果，填充到multipleResults
          multipleResults.add(defaultResultHandler.getResultList());
        } else {
          //使用指定的rusultHandler进行转换
          handleRowValues(rsw, resultMap, resultHandler, rowBounds, null);
        }
      }
    } finally {
      // issue #228 (close resultsets)
      //调用resultset.close()关闭结果集
      closeResultSet(rsw.getResultSet());
    }
  }

```

```
  public void handleRowValues(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping) throws SQLException {
    if (resultMap.hasNestedResultMaps()) {//处理有嵌套resultmap的情况
      ensureNoRowBounds();
      checkResultHandler();
      handleRowValuesForNestedResultMap(rsw, resultMap, resultHandler, rowBounds, parentMapping);
    } else {//处理没有嵌套resultmap的情况
      handleRowValuesForSimpleResultMap(rsw, resultMap, resultHandler, rowBounds, parentMapping);
    }
  }
```


```
  //简单映射处理
  private void handleRowValuesForSimpleResultMap(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping)
      throws SQLException {
	//创建结果上下文，所谓的上下文就是专门在循环中缓存结果对象的
    DefaultResultContext<Object> resultContext = new DefaultResultContext<>();
    //1.根据分页信息，定位到指定的记录
    skipRows(rsw.getResultSet(), rowBounds);
    //2.shouldProcessMoreRows判断是否需要映射后续的结果，实际还是翻页处理，避免超过limit
    while (shouldProcessMoreRows(resultContext, rowBounds) && rsw.getResultSet().next()) {
      //3.进一步完善resultMap信息，主要是处理鉴别器的信息
      ResultMap discriminatedResultMap = resolveDiscriminatedResultMap(rsw.getResultSet(), resultMap, null);
      //4.读取resultSet中的一行记录并进行映射，转化并返回目标对象
      Object rowValue = getRowValue(rsw, discriminatedResultMap);
      //5.保存映射结果对象
      storeObject(resultHandler, resultContext, rowValue, parentMapping, rsw.getResultSet());
    }
  }
```

```
  //4.读取resultSet中的一行记录并进行映射，转化并返回目标对象
  private Object getRowValue(ResultSetWrapper rsw, ResultMap resultMap) throws SQLException {
    final ResultLoaderMap lazyLoader = new ResultLoaderMap();
    //4.1 根据resultMap的type属性，实例化目标对象
    Object rowValue = createResultObject(rsw, resultMap, lazyLoader, null);
    if (rowValue != null && !hasTypeHandlerForResultObject(rsw, resultMap.getType())) {
      //4.2 对目标对象进行封装得到metaObjcect,为后续的赋值操作做好准备
      final MetaObject metaObject = configuration.newMetaObject(rowValue);
      boolean foundValues = this.useConstructorMappings;//取得是否使用构造函数初始化属性值
      if (shouldApplyAutomaticMappings(resultMap, false)) {//是否使用自动映射
    	 //4.3一般情况下 autoMappingBehavior默认值为PARTIAL，对未明确指定映射规则的字段进行自动映射
        foundValues = applyAutomaticMappings(rsw, resultMap, metaObject, null) || foundValues;
      }
       //4.4 映射resultMap中明确指定需要映射的列
      foundValues = applyPropertyMappings(rsw, resultMap, metaObject, lazyLoader, null) || foundValues;
      foundValues = lazyLoader.size() > 0 || foundValues;
      //4.5 如果没有一个映射成功的属性，则根据<returnInstanceForEmptyRow>的配置返回null或者结果对象
      rowValue = foundValues || configuration.isReturnInstanceForEmptyRow() ? rowValue : null;
    }
    return rowValue;
  }
```

```
  private boolean applyPropertyMappings(ResultSetWrapper rsw, ResultMap resultMap, MetaObject metaObject, ResultLoaderMap lazyLoader, String columnPrefix)
      throws SQLException {
	//从resultMap中获取明确需要转换的列名集合
    final List<String> mappedColumnNames = rsw.getMappedColumnNames(resultMap, columnPrefix);
    boolean foundValues = false;
    //获取ResultMapping集合
    final List<ResultMapping> propertyMappings = resultMap.getPropertyResultMappings();
    for (ResultMapping propertyMapping : propertyMappings) {
      String column = prependPrefix(propertyMapping.getColumn(), columnPrefix);//获得列名，注意前缀的处理
      if (propertyMapping.getNestedResultMapId() != null) {
        // the user added a column attribute to a nested result map, ignore it
    	//如果属性通过另外一个resultMap映射，则忽略
        column = null;
      }
      if (propertyMapping.isCompositeResult()//如果是嵌套查询，column={prop1=col1,prop2=col2}
          || (column != null && mappedColumnNames.contains(column.toUpperCase(Locale.ENGLISH)))//基本类型映射
          || propertyMapping.getResultSet() != null) {//嵌套查询的结果
    	//获得属性值
        Object value = getPropertyMappingValue(rsw.getResultSet(), metaObject, propertyMapping, lazyLoader, columnPrefix);
        // issue #541 make property optional
        //获得属性名称
        final String property = propertyMapping.getProperty();
        if (property == null) {//属性名为空跳出循环
          continue;
        } else if (value == DEFERED) {//属性名为DEFERED，延迟加载的处理
          foundValues = true;
          continue;
        }
        if (value != null) {
          foundValues = true;
        }
        if (value != null || (configuration.isCallSettersOnNulls() && !metaObject.getSetterType(property).isPrimitive())) {
          // gcode issue #377, call setter on nulls (value is not 'found')
          //通过metaObject为目标对象设置属性值
          metaObject.setValue(property, value);
        }
      }
    }
    return foundValues;
  }
```

#### 4.3.3.1 上面流程跟测试代码流程一致（使用反射基础组件完成）

```
	@Test
	public void reflectionTest(){
		
		//反射工具类初始化
		ObjectFactory objectFactory = new DefaultObjectFactory();
		TUser user = objectFactory.create(TUser.class);
		ObjectWrapperFactory objectWrapperFactory = new DefaultObjectWrapperFactory();
		ReflectorFactory reflectorFactory = new DefaultReflectorFactory();
		MetaObject metaObject = MetaObject.forObject(user, objectFactory, objectWrapperFactory, reflectorFactory);
		

		//模拟数据库行数据转化成对象
		//1.模拟从数据库读取数据
		Map<String, Object> dbResult = new HashMap<>();
		dbResult.put("id", 1);
		dbResult.put("user_name", "lison");
		dbResult.put("real_name", "李晓宇");
		TPosition tp = new TPosition();
		tp.setId(1);
		dbResult.put("position_id", tp);
		//2.模拟映射关系
		Map<String, String> mapper = new HashMap<String, String>();
		mapper.put("id", "id");
		mapper.put("userName", "user_name");
		mapper.put("realName", "real_name");
		mapper.put("position", "position_id");
		
		//3.使用反射工具类将行数据转换成pojo
		BeanWrapper objectWrapper = (BeanWrapper) metaObject.getObjectWrapper();
		
		Set<Entry<String, String>> entrySet = mapper.entrySet();
		for (Entry<String, String> colInfo : entrySet) {
			String propName = colInfo.getKey();
			Object propValue = dbResult.get(colInfo.getValue());
			PropertyTokenizer proTokenizer = new PropertyTokenizer(propName);
			objectWrapper.set(proTokenizer, propValue);
		}
		System.out.println(metaObject.getOriginalObject());
		
	}
```