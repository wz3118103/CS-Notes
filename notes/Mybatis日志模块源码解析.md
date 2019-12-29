日志模块需要解决的三个问题：
* MyBatis没有提供日志的实现类，需要接入第三方的日志组件，但第三方日志组件都有各自的Log级别，且各不相同，二MyBatis统一提供trace、debug、warn、error四个级别；
* 自动扫描日志实现，并且第三方日志插件加载优先级如下：slf4J → commonsLoging → Log4J2 → Log4J → JdkLog;
* 日志的使用要优雅的嵌入到主体功能中。


# 1.使用适配器模式统一日志接口
## 1.1 适配器模式
适配器模式（Adapter Pattern）是作为两个不兼容的接口之间的桥梁，将一个类的接口转换成客户希望的另外一个接口。适配器模式使得原本由于接口不兼容而不能一起工作的那些类可以一起工作。


<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/适配器模式.jpg" width="520px" > </div><br>


适用场景：当调用双方都不太容易修改的时候，为了复用现有组件可以使用适配器模式；在系统中接入第三方组件的时候经常被使用到。


注意：如果系统中存在过多的适配器，会增加系统的复杂性，设计人员应考虑对系统进行重构。

## 1.2 日志模块类图
<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/日志模块类图.jpg" width="520px" > </div><br>


<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/Log适配器类图.png" width="520px" > </div><br>


<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/Slf4j.png" width="520px" > </div><br>

# 2.自动按优先级扫描并加载第三方日志插件

这里使用静态代码块实现自动扫描、按优先级加载。

```
public final class LogFactory {

  /**
   * Marker to be used by logging implementations that support markers
   */
  public static final String MARKER = "MYBATIS";

  //被选定的第三方日志组件适配器的构造方法
  private static Constructor<? extends Log> logConstructor;

  //自动扫描日志实现，并且第三方日志插件加载优先级如下：slf4J → commonsLoging → Log4J2 → Log4J → JdkLog
  static {
    tryImplementation(LogFactory::useSlf4jLogging);
    tryImplementation(LogFactory::useCommonsLogging);
    tryImplementation(LogFactory::useLog4J2Logging);
    tryImplementation(LogFactory::useLog4JLogging);
    tryImplementation(LogFactory::useJdkLogging);
    tryImplementation(LogFactory::useNoLogging);
  }
```

其中tryImplementation执行前会判断logConstructor是否为空，只有为空时才会加载。因此，这里只要按照优先级安排tryImplementation语句的顺序即可实现日志的优先级加载。
```
  private static void tryImplementation(Runnable runnable) {
    // //当构造方法不为空才执行方法
    if (logConstructor == null) {
      try {
        runnable.run();
      } catch (Throwable t) {
        // ignore
      }
    }
  }
```

## 2.1 以slf4j为例分析是怎样加载的
```
  public static synchronized void useSlf4jLogging() {
    setImplementation(org.apache.ibatis.logging.slf4j.Slf4jImpl.class);
  }
```

```
  //通过指定的log类来初始化构造方法
  private static void setImplementation(Class<? extends Log> implClass) {
    try {
      Constructor<? extends Log> candidate = implClass.getConstructor(String.class);
      Log log = candidate.newInstance(LogFactory.class.getName());
      if (log.isDebugEnabled()) {
        log.debug("Logging initialized using '" + implClass + "' adapter.");
      }
      logConstructor = candidate;
    } catch (Throwable t) {
      throw new LogException("Error setting Log implementation.  Cause: " + t, t);
    }
  }
```
这里的加载核心是Log log = candidate.newInstance(LogFactory.class.getName());

该方法会调用Slf4jImpl的构造方法。

看看Slf4jImpl的构造方法：
```
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class Slf4jImpl implements Log {

  private Log log;

  public Slf4jImpl(String clazz) {
    Logger logger = LoggerFactory.getLogger(clazz);

    if (logger instanceof LocationAwareLogger) {
      try {
        // check for slf4j >= 1.6 method signature
        logger.getClass().getMethod("log", Marker.class, String.class, int.class, String.class, Object[].class, Throwable.class);
        log = new Slf4jLocationAwareLoggerImpl((LocationAwareLogger) logger);
        return;
      } catch (SecurityException e) {
        // fail-back to Slf4jLoggerImpl
      } catch (NoSuchMethodException e) {
        // fail-back to Slf4jLoggerImpl
      }
    }

    // Logger is not LocationAwareLogger or slf4j version < 1.6
    log = new Slf4jLoggerImpl(logger);
  }
```

使用LoggerFactory.getLogger(clazz)会找到logback实现类，然后加载成功日志实现。


Slf4jLocationAwareLoggerImpl就是在封装了一层，实际使用logger进行日志记录。
```
class Slf4jLocationAwareLoggerImpl implements Log {
  
  private static final Marker MARKER = MarkerFactory.getMarker(LogFactory.MARKER);

  private static final String FQCN = Slf4jImpl.class.getName();

  private final LocationAwareLogger logger;

  Slf4jLocationAwareLoggerImpl(LocationAwareLogger logger) {
    this.logger = logger;
  }

  @Override
  public boolean isDebugEnabled() {
    return logger.isDebugEnabled();
  }
```

最后在log.debug打印出日志：
```
  //通过指定的log类来初始化构造方法
  private static void setImplementation(Class<? extends Log> implClass) {
    try {
      Constructor<? extends Log> candidate = implClass.getConstructor(String.class);
      Log log = candidate.newInstance(LogFactory.class.getName());
      if (log.isDebugEnabled()) {
        log.debug("Logging initialized using '" + implClass + "' adapter.");
      }
```
```
[main] DEBUG org.apache.ibatis.logging.LogFactory - Logging initialized using 'class org.apache.ibatis.logging.slf4j.Slf4jImpl' adapter.
```

# 3.使用动态代理优雅嵌入日志打印功能
## 3.1代理模式
定义：给目标对象提供一个代理对象，并由代理对象控制对目标对象的引用；

目的：
* 作为中介解耦客户端和真实对象，保护真实对象安全；
* 防止直接访问目标对象给系统带来的不必要复杂性；
* 对业务进行增强，增强点多样化如：前入、后入、异常


<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/代理模式.jpg" width="520px" > </div><br>


<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/动态代理.jpg" width="520px" > </div><br>

## 3.2 日志模块JDBC包类图

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/JdbcLogger.png" width="620px" > </div><br>

*  ConnectionLogger：负责打印连接信息和SQL语句，并创建PreparedStatementLogger；
*  PreparedStatementLogger：负责打印参数信息，并创建ResultSetLogger
*  ResultSetLogge：r负责打印数据结果信息。


## 3.3 ConnectionLogger
<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/ConnectionLogger.jpg" width="620px" > </div><br>

调用栈如下：
```
  private ConnectionLogger(Connection conn, Log statementLog, int queryStack) {
    super(statementLog, queryStack);
    this.connection = conn;
  }
```
```
  public static Connection newInstance(Connection conn, Log statementLog, int queryStack) {
    InvocationHandler handler = new ConnectionLogger(conn, statementLog, queryStack);
    ClassLoader cl = Connection.class.getClassLoader();
    return (Connection) Proxy.newProxyInstance(cl, new Class[]{Connection.class}, handler);
  }
```
```
  protected Connection getConnection(Log statementLog) throws SQLException {
    Connection connection = transaction.getConnection();
    if (statementLog.isDebugEnabled()) {
      return ConnectionLogger.newInstance(connection, statementLog, queryStack);
    } else {
      return connection;
    }
  }
```
```
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

SimpleExecutor.doQuery方法：
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

### 3.3.1 statementLog
```
import org.apache.ibatis.logging.Log;
import org.apache.ibatis.logging.LogFactory;

public final class MappedStatement {

  private Log statementLog;

  public static class Builder {
    private MappedStatement mappedStatement = new MappedStatement();

    public Builder(Configuration configuration, String id, SqlSource sqlSource, SqlCommandType sqlCommandType) {

      String logId = id;
      if (configuration.getLogPrefix() != null) {
        logId = configuration.getLogPrefix() + id;
      }
      mappedStatement.statementLog = LogFactory.getLog(logId);

    }

```

getLog方法调用LogFactory加载时生成的logConstructor来创建Log：

```
  public static Log getLog(String logger) {
    try {
      return logConstructor.newInstance(logger);
    } catch (Throwable t) {
      throw new LogException("Error creating logger for logger " + logger + ".  Cause: " + t, t);
    }
  }
```

注意logConstructor是org.apache.ibatis.logging.LogFactory的静态变量：

```
public final class LogFactory {

  /**
   * Marker to be used by logging implementations that support markers
   */
  public static final String MARKER = "MYBATIS";

  //被选定的第三方日志组件适配器的构造方法
  private static Constructor<? extends Log> logConstructor;
```

### 3.3.2 怎样使用代理对象打印日志？

```
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

RoutingStatementHandler.prepare：
```
  @Override
  public Statement prepare(Connection connection, Integer transactionTimeout) throws SQLException {
    return delegate.prepare(connection, transactionTimeout);
  }
```

BaseStatementHandler.prepare：
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

PreparedStatementHandler.instantiateStatement：
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

最后的connection.prepareStatement(sql)调用至代理对象ConnectionLogger.invoke()方法：
* step1.首先打印日志
* step2.然后调用真正的connection.prepareStatement(sql)——method.invoke(connection, params)

```
  @Override
  //对连接的增强
  public Object invoke(Object proxy, Method method, Object[] params)
      throws Throwable {
    try {
    	//如果是从Obeject继承的方法直接忽略
      if (Object.class.equals(method.getDeclaringClass())) {
        return method.invoke(this, params);
      }
      //如果是调用prepareStatement、prepareCall、createStatement的方法，打印要执行的sql语句
      //并返回prepareStatement的代理对象，让prepareStatement也具备日志能力，打印参数
      if ("prepareStatement".equals(method.getName())) {
        if (isDebugEnabled()) {
          debug(" Preparing: " + removeBreakingWhitespace((String) params[0]), true);//打印sql语句
        }        
        PreparedStatement stmt = (PreparedStatement) method.invoke(connection, params);
        stmt = PreparedStatementLogger.newInstance(stmt, statementLog, queryStack);//创建代理对象
        return stmt;
      } else if ("prepareCall".equals(method.getName())) {
        if (isDebugEnabled()) {
          debug(" Preparing: " + removeBreakingWhitespace((String) params[0]), true);//打印sql语句
        }        
        PreparedStatement stmt = (PreparedStatement) method.invoke(connection, params);//创建代理对象
        stmt = PreparedStatementLogger.newInstance(stmt, statementLog, queryStack);
        return stmt;
      } else if ("createStatement".equals(method.getName())) {
        Statement stmt = (Statement) method.invoke(connection, params);
        stmt = StatementLogger.newInstance(stmt, statementLog, queryStack);//创建代理对象
        return stmt;
      } else {
        return method.invoke(connection, params);
      }
    } catch (Throwable t) {
      throw ExceptionUtil.unwrapThrowable(t);
    }
  }
```

### 3.3.3 BaseJdbcLogger中的debug方法调用statementLog.debug()

```
//所有日志增强的抽象基类
public abstract class BaseJdbcLogger {

  //保存preparestatment中常用的set方法（占位符赋值）
  protected static final Set<String> SET_METHODS = new HashSet<>();
  //保存preparestatment中常用的执行sql语句的方法
  protected static final Set<String> EXECUTE_METHODS = new HashSet<>();

  //保存preparestatment中set方法的键值对
  private final Map<Object, Object> columnMap = new HashMap<>();

  //保存preparestatment中set方法的key值
  private final List<Object> columnNames = new ArrayList<>();
  //保存preparestatment中set方法的value值
  private final List<Object> columnValues = new ArrayList<>();

  protected Log statementLog;
  protected int queryStack;

  /*
   * Default constructor
   */
  public BaseJdbcLogger(Log log, int queryStack) {
    this.statementLog = log;
    if (queryStack == 0) {
      this.queryStack = 1;
    } else {
      this.queryStack = queryStack;
    }
  }

  static {
    SET_METHODS.add("setString");
    SET_METHODS.add("setNString");
    SET_METHODS.add("setInt");
    SET_METHODS.add("setByte");
    SET_METHODS.add("setShort");
    SET_METHODS.add("setLong");
    SET_METHODS.add("setDouble");
    SET_METHODS.add("setFloat");
    SET_METHODS.add("setTimestamp");
    SET_METHODS.add("setDate");
    SET_METHODS.add("setTime");
    SET_METHODS.add("setArray");
    SET_METHODS.add("setBigDecimal");
    SET_METHODS.add("setAsciiStream");
    SET_METHODS.add("setBinaryStream");
    SET_METHODS.add("setBlob");
    SET_METHODS.add("setBoolean");
    SET_METHODS.add("setBytes");
    SET_METHODS.add("setCharacterStream");
    SET_METHODS.add("setNCharacterStream");
    SET_METHODS.add("setClob");
    SET_METHODS.add("setNClob");
    SET_METHODS.add("setObject");
    SET_METHODS.add("setNull");

    EXECUTE_METHODS.add("execute");
    EXECUTE_METHODS.add("executeUpdate");
    EXECUTE_METHODS.add("executeQuery");
    EXECUTE_METHODS.add("addBatch");
  }

  protected void setColumn(Object key, Object value) {
    columnMap.put(key, value);
    columnNames.add(key);
    columnValues.add(value);
  }

  protected Object getColumn(Object key) {
    return columnMap.get(key);
  }

  protected String getParameterValueString() {
    List<Object> typeList = new ArrayList<>(columnValues.size());
    for (Object value : columnValues) {
      if (value == null) {
        typeList.add("null");
      } else {
        typeList.add(objectValueString(value) + "(" + value.getClass().getSimpleName() + ")");
      }
    }
    final String parameters = typeList.toString();
    return parameters.substring(1, parameters.length() - 1);
  }

  protected String objectValueString(Object value) {
    if (value instanceof Array) {
      try {
        return ArrayUtil.toString(((Array) value).getArray());
      } catch (SQLException e) {
        return value.toString();
      }
    }
    return value.toString();
  }

  protected String getColumnString() {
    return columnNames.toString();
  }

  protected void clearColumnInfo() {
    columnMap.clear();
    columnNames.clear();
    columnValues.clear();
  }

  protected String removeBreakingWhitespace(String original) {
    StringTokenizer whitespaceStripper = new StringTokenizer(original);
    StringBuilder builder = new StringBuilder();
    while (whitespaceStripper.hasMoreTokens()) {
      builder.append(whitespaceStripper.nextToken());
      builder.append(" ");
    }
    return builder.toString();
  }

  protected boolean isDebugEnabled() {
    return statementLog.isDebugEnabled();
  }

  protected boolean isTraceEnabled() {
    return statementLog.isTraceEnabled();
  }

  protected void debug(String text, boolean input) {
    if (statementLog.isDebugEnabled()) {
      statementLog.debug(prefix(input) + text);
    }
  }

  protected void trace(String text, boolean input) {
    if (statementLog.isTraceEnabled()) {
      statementLog.trace(prefix(input) + text);
    }
  }

  private String prefix(boolean isInput) {
    char[] buffer = new char[queryStack * 2 + 2];
    Arrays.fill(buffer, '=');
    buffer[queryStack * 2 + 1] = ' ';
    if (isInput) {
      buffer[queryStack * 2] = '>';
    } else {
      buffer[0] = '<';
    }
    return new String(buffer);
  }

}

```

### 3.3.4 代理增强的梳理



<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/动态代理.jpg" width="520px" > </div><br>

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/ConnectionLogger代理增强实质.png" width="620px" > </div><br>

```
  public static Connection newInstance(Connection conn, Log statementLog, int queryStack) {
    InvocationHandler handler = new ConnectionLogger(conn, statementLog, queryStack);
    ClassLoader cl = Connection.class.getClassLoader();
    return (Connection) Proxy.newProxyInstance(cl, new Class[]{Connection.class}, handler);
  }
```

JdbcLogger是怎样使用动态代理模式将日志功能侵入jdbc操作？它是怎样将日志功能作为增强加入的？

* 通过Proxy对象来生成实现了Connection接口的代理对象
这里ConnectionLogger要代理的是Connection接口，通过InvocationHandler也即ConnectionLogger来持有Connection实例DruidPooledConnection。
* 通过InvocationHandler——ConnectionLogger来实现具体的日志增强
  - 通过继承BaseJdbcLogger抽象类，增强的日志打印功能全在BaseJdbcLogger抽象类中
  - 该BaseJdbcLogger需要Log statementLog，从ConnectionLogger的构造方法中传入
* 通过如下的字节码分析，可知代理功能的实质如下
  - 1）Proxy.newProxyInstance通过修改Connection字节码，加载实现Connection接口和继承Proxy类的代理类，并通过代理类的构造方法cons.newInstance(new Object[]{h})新建代理类对象
  - 2）代理对象重写了了Connection接口的方法，其中的prepareStatement会调用InvocationHandler也即ConnectionLogger.invoke()方法
  - 3）InvocationHandler也即ConnectionLogger的invoke方法，可对被代理对象DruidPooledConnection进行前后增强。

字节码分析生成的代理对象：

```
public final class $Proxy0
  extends Proxy
  implements Connection
{
  private static Method m1;
  private static Method m2;
  private static Method m3;
  private static Method m0;
  private static Method m4;
  
  public $Proxy0(InvocationHandler paramInvocationHandler)
  {
    super(paramInvocationHandler);
  }

  PreparedStatement prepareStatement(String sql)
  {
    try
    {
      this.h.invoke(this, m4, null);
      return;
    }
    catch (Error|RuntimeException localError)
    {
      throw localError;
    }
    catch (Throwable localThrowable)
    {
      throw new UndeclaredThrowableException(localThrowable);
    }
  }
```

其中Proxy类：

```
public class Proxy implements java.io.Serializable {

    /**
     * the invocation handler for this proxy instance.
     * @serial
     */
    protected InvocationHandler h;

    @CallerSensitive
    public static Object newProxyInstance(ClassLoader loader,
                                          Class<?>[] interfaces,
                                          InvocationHandler h)
        throws IllegalArgumentException
    {
        Objects.requireNonNull(h);

        final Class<?>[] intfs = interfaces.clone();
        final SecurityManager sm = System.getSecurityManager();
        if (sm != null) {
            checkProxyAccess(Reflection.getCallerClass(), loader, intfs);
        }

        /*
         * Look up or generate the designated proxy class.
         */
        Class<?> cl = getProxyClass0(loader, intfs);

        /*
         * Invoke its constructor with the designated invocation handler.
         */
        try {
            if (sm != null) {
                checkNewProxyPermission(Reflection.getCallerClass(), cl);
            }

            final Constructor<?> cons = cl.getConstructor(constructorParams);
            final InvocationHandler ih = h;
            if (!Modifier.isPublic(cl.getModifiers())) {
                AccessController.doPrivileged(new PrivilegedAction<Void>() {
                    public Void run() {
                        cons.setAccessible(true);
                        return null;
                    }
                });
            }
            return cons.newInstance(new Object[]{h});
        } catch (IllegalAccessException|InstantiationException e) {
            throw new InternalError(e.toString(), e);
        } catch (InvocationTargetException e) {
            Throwable t = e.getCause();
            if (t instanceof RuntimeException) {
                throw (RuntimeException) t;
            } else {
                throw new InternalError(t.toString(), t);
            }
        } catch (NoSuchMethodException e) {
            throw new InternalError(e.toString(), e);
        }
    }
```

## 3.4 PreparedStatementLogger

调用栈如下：

```
  private PreparedStatementLogger(PreparedStatement stmt, Log statementLog, int queryStack) {
    super(statementLog, queryStack);
    this.statement = stmt;
  }
```
```
  public static PreparedStatement newInstance(PreparedStatement stmt, Log statementLog, int queryStack) {
    InvocationHandler handler = new PreparedStatementLogger(stmt, statementLog, queryStack);
    ClassLoader cl = PreparedStatement.class.getClassLoader();
    return (PreparedStatement) Proxy.newProxyInstance(cl, new Class[]{PreparedStatement.class, CallableStatement.class}, handler);
  }
```

实在ConnectionLogger中调用的：
```
public final class ConnectionLogger extends BaseJdbcLogger implements InvocationHandler {

 //真正的连接对象
  private final Connection connection;

  private ConnectionLogger(Connection conn, Log statementLog, int queryStack) {
    super(statementLog, queryStack);
    this.connection = conn;
  }

  @Override
  //对连接的增强
  public Object invoke(Object proxy, Method method, Object[] params)
      throws Throwable {
    try {
    	//如果是从Obeject继承的方法直接忽略
      if (Object.class.equals(method.getDeclaringClass())) {
        return method.invoke(this, params);
      }
      //如果是调用prepareStatement、prepareCall、createStatement的方法，打印要执行的sql语句
      //并返回prepareStatement的代理对象，让prepareStatement也具备日志能力，打印参数
      if ("prepareStatement".equals(method.getName())) {
        if (isDebugEnabled()) {
          debug(" Preparing: " + removeBreakingWhitespace((String) params[0]), true);//打印sql语句
        }        
        PreparedStatement stmt = (PreparedStatement) method.invoke(connection, params);
        stmt = PreparedStatementLogger.newInstance(stmt, statementLog, queryStack);//创建代理对象
        return stmt;
      } else if ("prepareCall".equals(method.getName())) {
        if (isDebugEnabled()) {
          debug(" Preparing: " + removeBreakingWhitespace((String) params[0]), true);//打印sql语句
        }        
        PreparedStatement stmt = (PreparedStatement) method.invoke(connection, params);//创建代理对象
        stmt = PreparedStatementLogger.newInstance(stmt, statementLog, queryStack);
        return stmt;
      } else if ("createStatement".equals(method.getName())) {
        Statement stmt = (Statement) method.invoke(connection, params);
        stmt = StatementLogger.newInstance(stmt, statementLog, queryStack);//创建代理对象
        return stmt;
      } else {
        return method.invoke(connection, params);
      }
    } catch (Throwable t) {
      throw ExceptionUtil.unwrapThrowable(t);
    }
  }
```

## 3.5 ResultSetLogger
调用栈如下：
```
  private ResultSetLogger(ResultSet rs, Log statementLog, int queryStack) {
    super(statementLog, queryStack);
    this.rs = rs;
  }
```
```
  public static ResultSet newInstance(ResultSet rs, Log statementLog, int queryStack) {
    InvocationHandler handler = new ResultSetLogger(rs, statementLog, queryStack);
    ClassLoader cl = ResultSet.class.getClassLoader();
    return (ResultSet) Proxy.newProxyInstance(cl, new Class[]{ResultSet.class}, handler);
  }
```
是在PreparedStatementLogger中调用：
```
public final class PreparedStatementLogger extends BaseJdbcLogger implements InvocationHandler {

  private final PreparedStatement statement;

  private PreparedStatementLogger(PreparedStatement stmt, Log statementLog, int queryStack) {
    super(statementLog, queryStack);
    this.statement = stmt;
  }

  @Override
  public Object invoke(Object proxy, Method method, Object[] params) throws Throwable {
    try {
      if (Object.class.equals(method.getDeclaringClass())) {
        return method.invoke(this, params);
      }          
      if (EXECUTE_METHODS.contains(method.getName())) {
        if (isDebugEnabled()) {
          debug("Parameters: " + getParameterValueString(), true);
        }
        clearColumnInfo();
        if ("executeQuery".equals(method.getName())) {
          ResultSet rs = (ResultSet) method.invoke(statement, params);
          return rs == null ? null : ResultSetLogger.newInstance(rs, statementLog, queryStack);
        } else {
          return method.invoke(statement, params);
        }
      } else if (SET_METHODS.contains(method.getName())) {
        if ("setNull".equals(method.getName())) {
          setColumn(params[0], null);
        } else {
          setColumn(params[0], params[1]);
        }
        return method.invoke(statement, params);
      } else if ("getResultSet".equals(method.getName())) {
        ResultSet rs = (ResultSet) method.invoke(statement, params);
        return rs == null ? null : ResultSetLogger.newInstance(rs, statementLog, queryStack);
      } else if ("getUpdateCount".equals(method.getName())) {
        int updateCount = (Integer) method.invoke(statement, params);
        if (updateCount != -1) {
          debug("   Updates: " + updateCount, false);
        }
        return updateCount;
      } else {
        return method.invoke(statement, params);
      }
    } catch (Throwable t) {
      throw ExceptionUtil.unwrapThrowable(t);
    }
  }
```


# 4.静态代理与动态代理的区别？动态代理动态在哪个地方？
根据代理类的生成时间不同可以将代理分为静态代理和动态代理两种。 
* 静态代理在编译期间就已经确定
* 动态代理类的源码是在程序运行期间由JVM根据反射等机制动态的生成，所以不存在代理类的字节码文件。代理类和委托类的关系是在程序运行时确定。 


## 4.1 静态代理

缺点：
*  代理对象的一个接口只服务于一种类型的对象，如果要代理的方法很多，势必要为每一种方法都进行代理，静态代理在程序规模稍大时就无法胜任了。 
*  如果接口增加一个方法，除了所有实现类需要实现这个方法外，所有代理类也需要实现此方法。增加了代码维护的复杂度。 

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/静态代理.jpg" width="320px" > </div><br>


```
/**  
 * 代理接口。处理给定名字的任务。 
 */  
public interface Subject {  
  /** 
   * 执行给定名字的任务。 
    * @param taskName 任务名 
   */  
   public void dealTask(String taskName);   
}
```

```
public class RealSubject implements Subject {  
  
 /** 
  * 执行给定名字的任务。这里打印出任务名，并休眠500ms模拟任务执行了很长时间 
  * @param taskName  
  */  
   @Override  
   public void dealTask(String taskName) {  
      System.out.println("正在执行任务："+taskName);  
      try {  
         Thread.sleep(500);  
      } catch (InterruptedException e) {  
         e.printStackTrace();  
      }  
   }  
} 
```

```
public class ProxySubject implements Subject {  
 //代理类持有一个委托类的对象引用  
 private Subject delegate;  
   
 public ProxySubject(Subject delegate) {  
  this.delegate = delegate;  
 }  
  
 /** 
  * 将请求分派给委托类执行，记录任务执行前后的时间，时间差即为任务的处理时间 
  *  
  * @param taskName 
  */  
 @Override  
 public void dealTask(String taskName) {  
  long stime = System.currentTimeMillis();   
  //将请求分派给委托类处理  
  delegate.dealTask(taskName);  
  long ftime = System.currentTimeMillis();   
  System.out.println("执行任务耗时"+(ftime - stime)+"毫秒");  
    
 }  
}
```

```

public class SubjectStaticFactory {  
 //客户类调用此工厂方法获得代理对象。  
 //对客户类来说，其并不知道返回的是代理类对象还是委托类对象。  
 public static Subject getInstance(){   
  return new ProxySubject(new RealSubject());  
 }  
}
```

## 4.2 动态代理
* 优点：动态代理与静态代理相比较，最大的好处是接口中声明的所有方法都被转移到调用处理器一个集中的方法中处理（InvocationHandler.invoke）。这样，在接口方法数量比较多的时候，我们可以进行灵活处理，而不需要像静态代理那样每一个方法进行中转。在本示例中看不出来，因为invoke方法体内嵌入了具体的外围业务（记录任务处理前后时间并计算时间差），实际中可以类似Spring AOP那样配置外围业务。 
* 缺点：始终无法摆脱仅支持 interface 代理的桎梏，因为它的设计注定了这个遗憾。回想一下那些动态生成的代理类的继承关系图，它们已经注定有一个共同的父类叫 Proxy。Java 的继承机制注定了这些动态代理类们无法实现对 class 的动态代理，原因是多继承在 Java 中本质上就行不通。 