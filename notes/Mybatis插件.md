# 1.插件概述

插件是用来改变或者扩展mybatis的原有的功能，mybaits的插件就是通过继承Interceptor拦截器实现的；在没有完全理解插件之前禁止使用插件对mybaits进行扩展，可能会导致严重的问题。


mybatis中能使用插件进行拦截的接口和方法如下：
* Executor（update、query 、 flushStatment 、 commit 、 rollback 、 getTransaction 、 close 、 isClose）
* StatementHandler（prepare 、 paramterize 、 batch 、 update 、 query）
* ParameterHandler（ getParameterObject 、 setParameters ）
* ResultSetHandler（ handleResultSets 、 handleCursorResultSets 、 handleOutputParameters ）


# 2.插件开发

定义一个阈值，当查询操作运行时间超过这个阈值记录日志供运维人员定位慢查询，插件实现步骤：
* step1.实现Interceptor接口方法
* step2.确定拦截的签名
* step3.在配置文件中配置插件
* step4.运行测试用例

## 2.1 Interceptor接口

```
public interface Interceptor {
  
  //执行拦截逻辑的方法
  Object intercept(Invocation invocation) throws Throwable;

  //target是被拦截的对象，它的作用就是给被拦截的对象生成一个代理对象
  Object plugin(Object target);

  //读取在plugin中设置的参数
  void setProperties(Properties properties);

}

```

## 2.2 实现拦截器

```
@Intercepts({
        // query(Statement statement, ResultHandler resultHandler)
        @Signature(type = StatementHandler.class, method = "query", args = {Statement.class, ResultHandler.class})
})
public class ThresholdInterceptor implements Interceptor {

    private long threshold;

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        long begin = System.currentTimeMillis();
        Object ret = invocation.proceed();
        long end = System.currentTimeMillis();
        long runtime = end - begin;
        if (runtime > threshold) {
            Object[] args = invocation.getArgs();
            Statement stmt = (Statement) args[0];
            MetaObject metaObject = SystemMetaObject.forObject(stmt);
            PreparedStatementLogger logger = (PreparedStatementLogger)metaObject.getValue("h");
            Statement real = logger.getPreparedStatement();
            System.out.println("sql语句： " + real.toString() + ", 执行时间： " + runtime + "毫秒， 超过阈值"  + threshold);
        }
        return ret;
    }

    @Override
    public Object plugin(Object target) {
        return Plugin.wrap(target, this);
    }

    @Override
    public void setProperties(Properties properties) {
        this.threshold = Long.valueOf(properties.getProperty("threshold"));
    }
}

```

这里为什么要使用MetaObject获取h，因为这里第一个入参Statement实际如下，在ConnectionLogger.invoke()里面创建了PreparedStatement，并对其进行了增强，获得了打印日志的能力，所以stmt最终是个代理对象。语句代理对象所持有的的InvocationHandler也即PreparedStatementLogger持有最终的PreparedStatement。

```
        PreparedStatement stmt = (PreparedStatement) method.invoke(connection, params);
        stmt = PreparedStatementLogger.newInstance(stmt, statementLog, queryStack);//创建代理对象
        return stmt;
```
PreparedStatementLogger.newInstance如下：

```
  public static PreparedStatement newInstance(PreparedStatement stmt, Log statementLog, int queryStack) {
    InvocationHandler handler = new PreparedStatementLogger(stmt, statementLog, queryStack);
    ClassLoader cl = PreparedStatement.class.getClassLoader();
    return (PreparedStatement) Proxy.newProxyInstance(cl, new Class[]{PreparedStatement.class, CallableStatement.class}, handler);
  }
```



## 2.3 在配置文件中进行配置

```
	<plugins>

		<plugin interceptor="software.mybatis.interceptors.ThresholdInterceptor">
			<property name="threshold" value="10"/>
		</plugin>

	</plugins>
```

## 2.4 测试

```
public class MybatisDemo {
	

	private SqlSessionFactory sqlSessionFactory;
	
	

	@Before
	public void init() throws IOException {
		
		String resource = "software/mybatis/mybatis-demo-config.xml";
		InputStream inputStream = Resources.getResourceAsStream(resource);
		// 1.读取mybatis配置文件创SqlSessionFactory
		sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
		inputStream.close();
	}

	@Test
	// 测试自动映射
	public void testAutoMapping() throws IOException {
		// 2.获取sqlSession
		SqlSession sqlSession = sqlSessionFactory.openSession();
		// 3.获取对应mapper
		TUserMapper mapper = sqlSession.getMapper(TUserMapper.class);
		// 4.执行查询语句并返回结果
		TUser user = mapper.selectByPrimaryKey(1);
		System.out.println(user);
	}
```

测试结果：

```
sql语句： com.mysql.cj.jdbc.ClientPreparedStatement: select
		 
		id, user_name, real_name, sex, mobile, email, note,
		position_id
	 
		from t_user
		where id = 1, 执行时间： 124毫秒， 超过阈值10
```

# 3.源码分析

## 3.1 插件初始化

在mybatis-config.xml中：

```
	<plugins>
		<plugin interceptor="software.mybatis.interceptors.ThresholdInterceptor">
			<property name="threshold" value="10"/>
		</plugin>

		 <plugin interceptor="com.github.pagehelper.PageInterceptor">
			<property name="pageSizeZero" value="true" />
		</plugin>
	</plugins>
```

在XMLConfigBuilder.parseConfigrutation()中：

```
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
```

pluginElement()：

```
  private void pluginElement(XNode parent) throws Exception {
    if (parent != null) {
      //遍历所有的插件配置
      for (XNode child : parent.getChildren()) {
    	//获取插件的类名
        String interceptor = child.getStringAttribute("interceptor");
        //获取插件的配置
        Properties properties = child.getChildrenAsProperties();
        //实例化插件对象
        Interceptor interceptorInstance = (Interceptor) resolveClass(interceptor).newInstance();
        //设置插件属性
        interceptorInstance.setProperties(properties);
        //将插件添加到configuration对象，底层使用list保存所有的插件并记录顺序
        configuration.addInterceptor(interceptorInstance);
      }
    }
  }
```

Configuration中：

```
  /*插件集合*/
  protected final InterceptorChain interceptorChain = new InterceptorChain();


  public void addInterceptor(Interceptor interceptor) {
    interceptorChain.addInterceptor(interceptor);
  }
```

InterceptorChain是个插件数组集合：

```
public class InterceptorChain {

  private final List<Interceptor> interceptors = new ArrayList<>();

  public Object pluginAll(Object target) {
    for (Interceptor interceptor : interceptors) {
      target = interceptor.plugin(target);
    }
    return target;
  }

  public void addInterceptor(Interceptor interceptor) {
    interceptors.add(interceptor);
  }
  
  public List<Interceptor> getInterceptors() {
    return Collections.unmodifiableList(interceptors);
  }

}

```


## 3.2 插件调用流程

### 3.2.1 插件在哪里调用

Configuration中创建四大对象时会调用interceptorChain.pluginAll()：

* newExecutor
* newStatementHandler
* newParameterHandler
* newResultSetHandler



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

```
  public StatementHandler newStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
	//创建RoutingStatementHandler对象，实际由statmentType来指定真实的StatementHandler来实现
	StatementHandler statementHandler = new RoutingStatementHandler(executor, mappedStatement, parameterObject, rowBounds, resultHandler, boundSql);
    statementHandler = (StatementHandler) interceptorChain.pluginAll(statementHandler);
    return statementHandler;
  }

```

```
  public ParameterHandler newParameterHandler(MappedStatement mappedStatement, Object parameterObject, BoundSql boundSql) {
    ParameterHandler parameterHandler = mappedStatement.getLang().createParameterHandler(mappedStatement, parameterObject, boundSql);
    parameterHandler = (ParameterHandler) interceptorChain.pluginAll(parameterHandler);
    return parameterHandler;
  }
```

```
  public ResultSetHandler newResultSetHandler(Executor executor, MappedStatement mappedStatement, RowBounds rowBounds, ParameterHandler parameterHandler,
      ResultHandler resultHandler, BoundSql boundSql) {
    ResultSetHandler resultSetHandler = new DefaultResultSetHandler(executor, mappedStatement, parameterHandler, resultHandler, boundSql, rowBounds);
    resultSetHandler = (ResultSetHandler) interceptorChain.pluginAll(resultSetHandler);
    return resultSetHandler;
  }
```

### 3.2.2 责任链模式

责任链模式：就是把一件工作分别经过链上的各个节点，让这些节点依次处理这个工作；和装饰器模式不同，每个节点都知道后继者是谁；适合为完成同一个请求需要多个处理类的场景。

责任链模式优点：
* 降低耦合度。它将请求的发送者和接收者解耦。
* 简化了对象。使得对象不需要知道链的结构。
* 增强给对象指派职责的灵活性。通过改变链内的成员或者调动它们的次序，允许动态地新增或者删除责任。
* 增加新的请求处理类很方便。

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/责任链模式.png" width="420px" > </div><br>

### 3.2.3 插件中责任链模式的体现

拦截同一种对象的同一个方法，例如都拦截 Executor 的 query 方法，这时配置拦截器的顺序就会对这里有影响了。

```
<plugins>
    <plugin interceptor="com.github.pagehelper.ExecutorQueryInterceptor1"/>
    <plugin interceptor="com.github.pagehelper.ExecutorQueryInterceptor2"/>
    <plugin interceptor="com.github.pagehelper.ExecutorQueryInterceptor3"/>
</plugins>
```

在org.apache.ibatis.session.Configuration 中有如下方法：

```
public void addInterceptor(Interceptor interceptor) {
    interceptorChain.addInterceptor(interceptor);
}
```

MyBatis 会按照拦截器配置的顺序依次添加到 interceptorChain 中，其内部就是 List<Interceptor> interceptors。


插件调用在Configuration.newExecutor中：

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
    if (cacheEnabled) {
        executor = new CachingExecutor(executor);
    }
    executor = (Executor) interceptorChain.pluginAll(executor);
    return executor;
}

```

```
public Object pluginAll(Object target) {
    for (Interceptor interceptor : interceptors) {
        target = interceptor.plugin(target);
    }
    return target;
}
```

前面我们配置拦截器的顺序是1，2，3。在这里也会按照 1，2，3 的顺序被层层代理，代理后的结构如下：

```
Interceptor3:{
    Interceptor2: {
        Interceptor1: {
            target: Executor
        }
    }
}
```

从这个结构应该就很容易能看出来，将来执行的时候肯定是按照 3>2>1>Executor>1>2>3 的顺序去执行的。 

因此这里是责任链模式的变形，通过链表来指定处理的顺序。


## 3.3 分析Plugin

```
public class Plugin implements InvocationHandler {
  //封装的真正提供服务的对象
  private final Object target;
  //自定义的拦截器
  private final Interceptor interceptor;
  //解析@Intercepts注解得到的signature信息
  private final Map<Class<?>, Set<Method>> signatureMap;

  private Plugin(Object target, Interceptor interceptor, Map<Class<?>, Set<Method>> signatureMap) {
    this.target = target;
    this.interceptor = interceptor;
    this.signatureMap = signatureMap;
  }

  //静态方法，用于帮助Interceptor生成动态代理
  public static Object wrap(Object target, Interceptor interceptor) {
	//解析Interceptor上@Intercepts注解得到的signature信息
    Map<Class<?>, Set<Method>> signatureMap = getSignatureMap(interceptor);
    Class<?> type = target.getClass();//获取目标对象的类型
    Class<?>[] interfaces = getAllInterfaces(type, signatureMap);//获取目标对象实现的接口（拦截器可以拦截4大对象实现的接口）
    if (interfaces.length > 0) {
      //使用jdk的方式创建动态代理
      return Proxy.newProxyInstance(
          type.getClassLoader(),
          interfaces,
          new Plugin(target, interceptor, signatureMap));
    }
    return target;
  }

  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      //获取当前接口可以被拦截的方法
      Set<Method> methods = signatureMap.get(method.getDeclaringClass());
      if (methods != null && methods.contains(method)) {//如果当前方法需要被拦截，则调用interceptor.intercept方法进行拦截处理
        return interceptor.intercept(new Invocation(target, method, args));
      }
      //如果当前方法不需要被拦截，则调用对象自身的方法
      return method.invoke(target, args);
    } catch (Exception e) {
      throw ExceptionUtil.unwrapThrowable(e);
    }
  }

  private static Map<Class<?>, Set<Method>> getSignatureMap(Interceptor interceptor) {
	 //获取@Intercepts注解实例
    Intercepts interceptsAnnotation = interceptor.getClass().getAnnotation(Intercepts.class);
    // issue #251
    if (interceptsAnnotation == null) {
      throw new PluginException("No @Intercepts annotation was found in interceptor " + interceptor.getClass().getName());      
    }
    //获取其中@Signature实例，并将里面的method信息添加至signatureMap中
    Signature[] sigs = interceptsAnnotation.value();
    Map<Class<?>, Set<Method>> signatureMap = new HashMap<>();
    for (Signature sig : sigs) {
      Set<Method> methods = signatureMap.computeIfAbsent(sig.type(), k -> new HashSet<>());
      try {
        Method method = sig.type().getMethod(sig.method(), sig.args());
        methods.add(method);
      } catch (NoSuchMethodException e) {
        throw new PluginException("Could not find method on " + sig.type() + " named " + sig.method() + ". Cause: " + e, e);
      }
    }
    return signatureMap;
  }

  private static Class<?>[] getAllInterfaces(Class<?> type, Map<Class<?>, Set<Method>> signatureMap) {
    Set<Class<?>> interfaces = new HashSet<>();
    while (type != null) {
      for (Class<?> c : type.getInterfaces()) {
        if (signatureMap.containsKey(c)) {
          interfaces.add(c);
        }
      }
      type = type.getSuperclass();
    }
    return interfaces.toArray(new Class<?>[interfaces.size()]);
  }

}

```

生成代理的调用路径：

SimpleExecutor.doQuery()：

```
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

Configuration.newStatementHandler()：

```
  public StatementHandler newStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
	//创建RoutingStatementHandler对象，实际由statmentType来指定真实的StatementHandler来实现
	StatementHandler statementHandler = new RoutingStatementHandler(executor, mappedStatement, parameterObject, rowBounds, resultHandler, boundSql);
    statementHandler = (StatementHandler) interceptorChain.pluginAll(statementHandler);
    return statementHandler;
  }
```

InterceptorChain.pluginAll()：

```
public class InterceptorChain {

  private final List<Interceptor> interceptors = new ArrayList<>();

  public Object pluginAll(Object target) {
    for (Interceptor interceptor : interceptors) {
      target = interceptor.plugin(target);
    }
    return target;
  }
```

调用至自己实现的插件：

```
    @Override
    public Object plugin(Object target) {
        return Plugin.wrap(target, this);
    }
```

最后调用至Plugin.wrap()。

```
  //静态方法，用于帮助Interceptor生成动态代理
  public static Object wrap(Object target, Interceptor interceptor) {
	//解析Interceptor上@Intercepts注解得到的signature信息
    Map<Class<?>, Set<Method>> signatureMap = getSignatureMap(interceptor);
    Class<?> type = target.getClass();//获取目标对象的类型
    Class<?>[] interfaces = getAllInterfaces(type, signatureMap);//获取目标对象实现的接口（拦截器可以拦截4大对象实现的接口）
    if (interfaces.length > 0) {
      //使用jdk的方式创建动态代理
      return Proxy.newProxyInstance(
          type.getClassLoader(),
          interfaces,
          new Plugin(target, interceptor, signatureMap));
    }
    return target;
  }
```

最后调用jdk动态代理创建拦截接口的代理对象，这里就是StatementHandler。

所以最后所有的StatementHandler的所有操作都会通过InvocationHandler也即Plugin.invoke()来进行功能增强。Plugin.invoke()会判断方法是否需要被拦截，如果需要拦截会走interceptor.intercept()，也就是我们自己实现的intercept()方法。

```
  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      //获取当前接口可以被拦截的方法
      Set<Method> methods = signatureMap.get(method.getDeclaringClass());
      if (methods != null && methods.contains(method)) {//如果当前方法需要被拦截，则调用interceptor.intercept方法进行拦截处理
        return interceptor.intercept(new Invocation(target, method, args));
      }
      //如果当前方法不需要被拦截，则调用对象自身的方法
      return method.invoke(target, args);
    } catch (Exception e) {
      throw ExceptionUtil.unwrapThrowable(e);
    }
  }

```

### 3.3.1 Invocation只是简单做了个封装

```
public class Invocation {

  private final Object target;
  private final Method method;
  private final Object[] args;

  public Invocation(Object target, Method method, Object[] args) {
    this.target = target;
    this.method = method;
    this.args = args;
  }

  public Object getTarget() {
    return target;
  }

  public Method getMethod() {
    return method;
  }

  public Object[] getArgs() {
    return args;
  }

  public Object proceed() throws InvocationTargetException, IllegalAccessException {
    return method.invoke(target, args);
  }

}
```

# 4.分页插件PageHelper

分页插件的使用；
* 中文文档：https://github.com/pagehelper/Mybatis-PageHelper/blob/master/README_zh.md
* 使用手册：https://github.com/pagehelper/Mybatis-PageHelper/blob/master/wikis/zh/HowToUse.md

分页插件的注意事项；
* 注意事项：https://github.com/pagehelper/Mybatis-PageHelper/blob/master/wikis/zh/Important.md


## 4.1 实现原理

* step1.首先生成count查询，查总数
* step2.使用Limit ?, ?进行分页

```
2020-01-01 10:18:13.350 [main] DEBUG s.m.mapper.TUserMapper.selectByEmailAndSex1_COUNT - ==>  Preparing: SELECT count(0) FROM t_user a WHERE a.email LIKE CONCAT('%', ?, '%') AND a.sex = ? 
2020-01-01 10:18:13.421 [main] DEBUG s.m.mapper.TUserMapper.selectByEmailAndSex1_COUNT - ==> Parameters: qq.com(String), 1(Byte)
2020-01-01 10:18:13.552 [main] DEBUG s.m.mapper.TUserMapper.selectByEmailAndSex1_COUNT - <==      Total: 1
sql语句： com.mysql.cj.jdbc.ClientPreparedStatement: SELECT count(0) FROM t_user a WHERE a.email LIKE CONCAT('%', 'qq.com', '%') AND a.sex = 1, 执行时间： 132毫秒， 超过阈值10


2020-01-01 10:18:13.562 [main] DEBUG s.mybatis.mapper.TUserMapper.selectByEmailAndSex1 - ==>  Preparing: select id, user_name, real_name, sex, mobile, email, note, position_id from t_user a where a.email like CONCAT('%', ?, '%') and a.sex = ? LIMIT ?, ? 
2020-01-01 10:18:13.563 [main] DEBUG s.mybatis.mapper.TUserMapper.selectByEmailAndSex1 - ==> Parameters: qq.com(String), 1(Byte), 2(Integer), 2(Integer)
2020-01-01 10:18:13.634 [main] DEBUG s.mybatis.mapper.TUserMapper.selectByEmailAndSex1 - <==      Total: 2
```