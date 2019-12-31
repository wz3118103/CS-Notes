
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

XMLConfigBuilder： 主要负责解析mybatis-config.xml；
XMLMapperBuilder： 主要负责解析映射配置文件；
XMLStatementBuilder： 主要负责解析映射配置文件中的SQL节点。

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

# 4.核心组件Executor