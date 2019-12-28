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


核心问题：

3.静态代理与动态代理的区别？动态代理动态在哪个地方？
4.isDebugEnabled()这个开关是怎么打开的
5.JdbcLogger是怎样使用动态代理模式将日志功能侵入jdbc操作？它是怎样将日志功能作为增强加入的？
6.JdbcLogger增强了哪些地方？是怎样获取Logger进行打印的？只不过是将日志打印功能放到了代理类的增强里面