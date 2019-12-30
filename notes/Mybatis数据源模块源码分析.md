# 1.数据源模块需求
* 常见的数据源组件都实现了javax.sql.DataSource接口；
* MyBatis不但要能集成第三方的数据源组件，自身也提供了数据源的实现；
* 一般情况下，数据源的初始化过程参数较多，比较复杂。


# 2.工厂模式

工厂模式（Factory Pattern）属于创建型模式，它提供了一种创建对象的最佳方式。定义一个创建对象的接口，让其子类自己决定实例化哪一个工厂类，工厂模式使其创建过程延迟到子类进行。

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/工厂模式.jpg" width="520px" > </div><br>


为什么要使用工厂模式？
* 一般情况下创建对象的方式
  - 使用new关键字直接创建对象；
  - 通过反射机制创建对象；
  - 通过工厂类创建对象。
* new和反射机制的缺点
  - 对象创建和对象使用使用的职责耦合在一起，违反单一原则；
  - 当业务扩展时，必须修改代业务代码，违反了开闭原则
* 工厂类模式的优点
  - 把对象的创建和使用的过程分开，对象创建和对象使用使用的职责解耦；
  - 如果创建对象的过程很复杂，创建过程统一到工厂里管理，既减少了重复代码，也方便以后对创建过程的修改维护；
  - 当业务扩展时，只需要增加工厂子类，符合开闭原则
* Spring将对象创建和使用分离的原则发挥到了机制


<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/数据源模块类图.png" width="520px" > </div><br>

# 3.数据源模块源码分析
* PooledConnection：使用动态代理封装了真正的数据库连接对象；
* PoolState：用于管理PooledConnection对象状态的组件，通过两个list分别 管理空闲状态的连接资源和活跃状态的连接资源
* PooledDataSource：一个简单，同步的、线程安全的数据库连接池


<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/datasource包.jpg" width="320px" > </div><br>


## 3.1 DataSourceFactory

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/PooledDataSourceFactory.png" width="320px" > </div><br>

```
public interface DataSourceFactory {

  //设置DataSource的相关属性
  void setProperties(Properties props);

  //获取数据源
  DataSource getDataSource();

}
```

### 3.1.1 UnpooledDataSourceFactory

```
//不用连接池的数据连接工厂
public class UnpooledDataSourceFactory implements DataSourceFactory {

  private static final String DRIVER_PROPERTY_PREFIX = "driver.";
  private static final int DRIVER_PROPERTY_PREFIX_LENGTH = DRIVER_PROPERTY_PREFIX.length();

  protected DataSource dataSource;

  public UnpooledDataSourceFactory() {
    this.dataSource = new UnpooledDataSource();
  }

  @Override
  public void setProperties(Properties properties) {
    Properties driverProperties = new Properties();
    //创建DataSource相应的metaObject，方便赋值
    MetaObject metaDataSource = SystemMetaObject.forObject(dataSource);
    //遍历properties，将属性设置到DataSource中
    for (Object key : properties.keySet()) {
      String propertyName = (String) key;
      if (propertyName.startsWith(DRIVER_PROPERTY_PREFIX)) {
        String value = properties.getProperty(propertyName);
        driverProperties.setProperty(propertyName.substring(DRIVER_PROPERTY_PREFIX_LENGTH), value);
      } else if (metaDataSource.hasSetter(propertyName)) {
        String value = (String) properties.get(propertyName);
        Object convertedValue = convertValue(metaDataSource, propertyName, value);
        metaDataSource.setValue(propertyName, convertedValue);
      } else {
        throw new DataSourceException("Unknown DataSource property: " + propertyName);
      }
    }
    //设置DataSource.driverProperties属性
    if (driverProperties.size() > 0) {
      metaDataSource.setValue("driverProperties", driverProperties);
    }
  }

  @Override
  public DataSource getDataSource() {
    return dataSource;
  }

  private Object convertValue(MetaObject metaDataSource, String propertyName, String value) {
    Object convertedValue = value;
    Class<?> targetType = metaDataSource.getSetterType(propertyName);
    if (targetType == Integer.class || targetType == int.class) {
      convertedValue = Integer.valueOf(value);
    } else if (targetType == Long.class || targetType == long.class) {
      convertedValue = Long.valueOf(value);
    } else if (targetType == Boolean.class || targetType == boolean.class) {
      convertedValue = Boolean.valueOf(value);
    }
    return convertedValue;
  }

}

```

### 3.1.2 PooledDataSourceFactory

```
public class PooledDataSourceFactory extends UnpooledDataSourceFactory {

  public PooledDataSourceFactory() {
    this.dataSource = new PooledDataSource();
  }

}
``` 

## 3.2 DataSource

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/DataSource.png" width="520px" > </div><br>


### 3.2.1 UnpooledDataSource

#### Q.为什么Class.forName("com.mysql.jdbc.Driver")后，驱动就被注册到DriverManager?
* 因为Class.forName加载驱动是，驱动会自动向DriverManager进行注册。所以加载UnpooledDataSource时，可以从DriverManager获取到已经注册的数据库驱动，并放入到缓存registeredDrivers。

```
public class UnpooledDataSource implements DataSource {
  
  private ClassLoader driverClassLoader;//驱动类的类加载器
  private Properties driverProperties;//数据库连接相关配置信息
  private static Map<String, Driver> registeredDrivers = new ConcurrentHashMap<>();//缓存已注册的数据库驱动类

  private String driver;
  private String url;
  private String username;
  private String password;

  private Boolean autoCommit;//是否自动提交
  private Integer defaultTransactionIsolationLevel;//事务隔离级别

  
  //提问：为什么Class.forName("com.mysql.jdbc.Driver")后，驱动就被注册到DriverManager?
  static {
    Enumeration<Driver> drivers = DriverManager.getDrivers();
    while (drivers.hasMoreElements()) {
      Driver driver = drivers.nextElement();
      registeredDrivers.put(driver.getClass().getName(), driver);
    }
  }
```


再来查看com.mysql.cj.jdbc.Driver（也即com.mysql.jdbc.Driver）：
* 其在类加载时，会执行静态代码块，此时会向DriverManager注册驱动类

```
package com.mysql.cj.jdbc;

import java.sql.DriverManager;
import java.sql.SQLException;

public class Driver extends NonRegisteringDriver implements java.sql.Driver {
    public Driver() throws SQLException {
    }

    static {
        try {
            DriverManager.registerDriver(new Driver());
        } catch (SQLException var1) {
            throw new RuntimeException("Can't register driver!");
        }
    }
}
```


#### Q.是怎样获取连接的？


```
  @Override
  public Connection getConnection() throws SQLException {
    return doGetConnection(username, password);
  }
```

```
  private Connection doGetConnection(String username, String password) throws SQLException {
    Properties props = new Properties();
    if (driverProperties != null) {
      props.putAll(driverProperties);
    }
    if (username != null) {
      props.setProperty("user", username);
    }
    if (password != null) {
      props.setProperty("password", password);
    }
    return doGetConnection(props);
  }

```

从如下代码可以看出，与手动获取连接的方式是一样的：
* step1.区别加载了数据库驱动
* step2.从DriverManager.getConnection获取连接
* step3.设置事务自动提交和隔离的属性


```
  //从这个代码可以看出，unpooledDatasource获取连接的方式和手动获取连接的方式是一样的
  private Connection doGetConnection(Properties properties) throws SQLException {
    initializeDriver();
    Connection connection = DriverManager.getConnection(url, properties);
    //设置事务是否自动提交，事务的隔离级别
    configureConnection(connection);
    return connection;
  }
```

initializeDriver()还会确保驱动已经注册了，如果没有，再重新注册一次。

```
  private synchronized void initializeDriver() throws SQLException {
    if (!registeredDrivers.containsKey(driver)) {
      Class<?> driverType;
      try {
        if (driverClassLoader != null) {
          driverType = Class.forName(driver, true, driverClassLoader);
        } else {
          driverType = Resources.classForName(driver);
        }
        // DriverManager requires the driver to be loaded via the system ClassLoader.
        // http://www.kfu.com/~nsayer/Java/dyn-jdbc.html
        Driver driverInstance = (Driver)driverType.newInstance();
        DriverManager.registerDriver(new DriverProxy(driverInstance));
        registeredDrivers.put(driver, driverInstance);
      } catch (Exception e) {
        throw new SQLException("Error setting driver on UnpooledDataSource. Cause: " + e);
      }
    }
  }
```


### 3.2.2 PooledDataSource

连接池数据源还是以UnPooledDataSource作为基础的。

```
public class PooledDataSource implements DataSource {

  private static final Log log = LogFactory.getLog(PooledDataSource.class);

  private final PoolState state = new PoolState(this);

  //真正用于创建连接的数据源
  private final UnpooledDataSource dataSource;

  // OPTIONAL CONFIGURATION FIELDS
  //最大活跃连接数
  protected int poolMaximumActiveConnections = 10;
  //最大闲置连接数
  protected int poolMaximumIdleConnections = 5;
  //最大checkout时长（最长使用时间）
  protected int poolMaximumCheckoutTime = 20000;
  //无法取得连接是最大的等待时间
  protected int poolTimeToWait = 20000;
  //最多允许几次无效连接
  protected int poolMaximumLocalBadConnectionTolerance = 3;
  //测试连接是否有效的sql语句
  protected String poolPingQuery = "NO PING QUERY SET";
  //是否允许测试连接
  protected boolean poolPingEnabled;
  //配置一段时间，当连接在这段时间内没有被使用，才允许测试连接是否有效
  protected int poolPingConnectionsNotUsedFor;
  //根据数据库url、用户名、密码生成一个hash值，唯一标识一个连接池，由这个连接池生成的连接都会带上这个值
  private int expectedConnectionTypeCode;

  public PooledDataSource() {
    dataSource = new UnpooledDataSource();
  }

```

#### Q.是怎样获取连接的？

```
  @Override
  public Connection getConnection() throws SQLException {
    return popConnection(dataSource.getUsername(), dataSource.getPassword()).getProxyConnection();
  }
```

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/getConnection.png" width="620px" > </div><br>

```
  private PooledConnection popConnection(String username, String password) throws SQLException {
    boolean countedWait = false;
    PooledConnection conn = null;
    long t = System.currentTimeMillis();//记录尝试获取连接的起始时间戳
    int localBadConnectionCount = 0;//初始化获取到无效连接的次数

    while (conn == null) {
      synchronized (state) {//获取连接必须是同步的
        if (!state.idleConnections.isEmpty()) {//检测是否有空闲连接
          // Pool has available connection
          //有空闲连接直接使用
          conn = state.idleConnections.remove(0);
          if (log.isDebugEnabled()) {
            log.debug("Checked out connection " + conn.getRealHashCode() + " from pool.");
          }
        } else {// 没有空闲连接
          if (state.activeConnections.size() < poolMaximumActiveConnections) {//判断活跃连接池中的数量是否大于最大连接数
            // 没有则可创建新的连接
            conn = new PooledConnection(dataSource.getConnection(), this);
            if (log.isDebugEnabled()) {
              log.debug("Created connection " + conn.getRealHashCode() + ".");
            }
          } else {// 如果已经等于最大连接数，则不能创建新连接
            //获取最早创建的连接
            PooledConnection oldestActiveConnection = state.activeConnections.get(0);
            long longestCheckoutTime = oldestActiveConnection.getCheckoutTime();
            if (longestCheckoutTime > poolMaximumCheckoutTime) {//检测是否已经以及超过最长使用时间
              // 如果超时，对超时连接的信息进行统计
              state.claimedOverdueConnectionCount++;//超时连接次数+1
              state.accumulatedCheckoutTimeOfOverdueConnections += longestCheckoutTime;//累计超时时间增加
              state.accumulatedCheckoutTime += longestCheckoutTime;//累计的使用连接的时间增加
              state.activeConnections.remove(oldestActiveConnection);//从活跃队列中删除
              if (!oldestActiveConnection.getRealConnection().getAutoCommit()) {//如果超时连接未提交，则手动回滚
                try {
                  oldestActiveConnection.getRealConnection().rollback();
                } catch (SQLException e) {//发生异常仅仅记录日志
                  /*
                     Just log a message for debug and continue to execute the following
                     statement like nothing happend.
                     Wrap the bad connection with a new PooledConnection, this will help
                     to not intterupt current executing thread and give current thread a
                     chance to join the next competion for another valid/good database
                     connection. At the end of this loop, bad {@link @conn} will be set as null.
                   */
                  log.debug("Bad connection. Could not roll back");
                }  
              }
              //在连接池中创建新的连接，注意对于数据库来说，并没有创建新连接；
              conn = new PooledConnection(oldestActiveConnection.getRealConnection(), this);
              conn.setCreatedTimestamp(oldestActiveConnection.getCreatedTimestamp());
              conn.setLastUsedTimestamp(oldestActiveConnection.getLastUsedTimestamp());
              //让老连接失效
              oldestActiveConnection.invalidate();
              if (log.isDebugEnabled()) {
                log.debug("Claimed overdue connection " + conn.getRealHashCode() + ".");
              }
            } else {
              // 无空闲连接，最早创建的连接没有失效，无法创建新连接，只能阻塞
              try {
                if (!countedWait) {
                  state.hadToWaitCount++;//连接池累计等待次数加1
                  countedWait = true;
                }
                if (log.isDebugEnabled()) {
                  log.debug("Waiting as long as " + poolTimeToWait + " milliseconds for connection.");
                }
                long wt = System.currentTimeMillis();
                state.wait(poolTimeToWait);//阻塞等待指定时间
                state.accumulatedWaitTime += System.currentTimeMillis() - wt;//累计等待时间增加
              } catch (InterruptedException e) {
                break;
              }
            }
          }
        }
        if (conn != null) {//获取连接成功的，要测试连接是否有效，同时更新统计数据
          // ping to server and check the connection is valid or not
          if (conn.isValid()) {//检测连接是否有效
            if (!conn.getRealConnection().getAutoCommit()) {
              conn.getRealConnection().rollback();//如果遗留历史的事务，回滚
            }
            //连接池相关统计信息更新
            conn.setConnectionTypeCode(assembleConnectionTypeCode(dataSource.getUrl(), username, password));
            conn.setCheckoutTimestamp(System.currentTimeMillis());
            conn.setLastUsedTimestamp(System.currentTimeMillis());
            state.activeConnections.add(conn);
            state.requestCount++;
            state.accumulatedRequestTime += System.currentTimeMillis() - t;
          } else {//如果连接无效
            if (log.isDebugEnabled()) {
              log.debug("A bad connection (" + conn.getRealHashCode() + ") was returned from the pool, getting another connection.");
            }
            state.badConnectionCount++;//累计的获取无效连接次数+1
            localBadConnectionCount++;//当前获取无效连接次数+1
            conn = null;
            //拿到无效连接，但如果没有超过重试的次数，允许再次尝试获取连接，否则抛出异常
            if (localBadConnectionCount > (poolMaximumIdleConnections + poolMaximumLocalBadConnectionTolerance)) {
              if (log.isDebugEnabled()) {
                log.debug("PooledDataSource: Could not get a good connection to the database.");
              }
              throw new SQLException("PooledDataSource: Could not get a good connection to the database.");
            }
          }
        }
      }

    }

    if (conn == null) {
      if (log.isDebugEnabled()) {
        log.debug("PooledDataSource: Unknown severe error condition.  The connection pool returned a null connection.");
      }
      throw new SQLException("PooledDataSource: Unknown severe error condition.  The connection pool returned a null connection.");
    }

    return conn;
  }
```

测试连接是否有效：
```
  public boolean isValid() {
    return valid && realConnection != null && dataSource.pingConnection(this);
  }

```

```
  protected boolean pingConnection(PooledConnection conn) {
    boolean result = true;

    try {
      result = !conn.getRealConnection().isClosed();
    } catch (SQLException e) {
      if (log.isDebugEnabled()) {
        log.debug("Connection " + conn.getRealHashCode() + " is BAD: " + e.getMessage());
      }
      result = false;
    }

    if (result) {
      if (poolPingEnabled) {
        if (poolPingConnectionsNotUsedFor >= 0 && conn.getTimeElapsedSinceLastUse() > poolPingConnectionsNotUsedFor) {
          try {
            if (log.isDebugEnabled()) {
              log.debug("Testing connection " + conn.getRealHashCode() + " ...");
            }
            Connection realConn = conn.getRealConnection();
            try (Statement statement = realConn.createStatement()) {
              statement.executeQuery(poolPingQuery).close();
            }
            if (!realConn.getAutoCommit()) {
              realConn.rollback();
            }
            result = true;
            if (log.isDebugEnabled()) {
              log.debug("Connection " + conn.getRealHashCode() + " is GOOD!");
            }
          } catch (Exception e) {
            log.warn("Execution of ping query '" + poolPingQuery + "' failed: " + e.getMessage());
            try {
              conn.getRealConnection().close();
            } catch (Exception e2) {
              //ignore
            }
            result = false;
            if (log.isDebugEnabled()) {
              log.debug("Connection " + conn.getRealHashCode() + " is BAD: " + e.getMessage());
            }
          }
        }
      }
    }
    return result;
  }
```


#### Q.是怎样归还连接的？

这里使用了Java动态代理模式：
* step1.在popConection中创建PooledConnection，并使用代理模式创建了proxyConnection

```
conn = new PooledConnection(dataSource.getConnection(), this);

conn = new PooledConnection(oldestActiveConnection.getRealConnection(), this);
```

```
private static final Class<?>[] IFACES = new Class<?>[] { Connection.class };

  public PooledConnection(Connection connection, PooledDataSource dataSource) {
    this.hashCode = connection.hashCode();
    this.realConnection = connection;
    this.dataSource = dataSource;
    this.createdTimestamp = System.currentTimeMillis();
    this.lastUsedTimestamp = System.currentTimeMillis();
    this.valid = true;
    this.proxyConnection = (Connection) Proxy.newProxyInstance(Connection.class.getClassLoader(), IFACES, this);
  }
```

* step2.在getConnection中返回的PooledConnection的proxyConnection

```
  @Override
  public Connection getConnection() throws SQLException {
    return popConnection(dataSource.getUsername(), dataSource.getPassword()).getProxyConnection();
  }
```

```
  public Connection getProxyConnection() {
    return proxyConnection;
  }
```

* step3.当调用conn.close的时候，会触发proxyConnection的InvocationHandler.invoke方法，而这里InvocationHandler就是proxyConnection

```
  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    String methodName = method.getName();
    if (CLOSE.hashCode() == methodName.hashCode() && CLOSE.equals(methodName)) {//如果是调用连接的close方法，不是真正的关闭，而是回收到连接池
      dataSource.pushConnection(this);//通过pooled数据源来进行回收
      return null;
    } else {
      try {
    	  //使用前要检查当前连接是否有效
        if (!Object.class.equals(method.getDeclaringClass())) {
          // issue #579 toString() should never fail
          // throw an SQLException instead of a Runtime
          checkConnection();//
        }
        return method.invoke(realConnection, args);
      } catch (Throwable t) {
        throw ExceptionUtil.unwrapThrowable(t);
      }
    }
  }
```

* step4.这里会调用pushConnection来进行归还

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/pushConnection.png" width="620px" > </div><br>

```
  protected void pushConnection(PooledConnection conn) throws SQLException {

    synchronized (state) {//回收连接必须是同步的
      state.activeConnections.remove(conn);//从活跃连接池中删除此连接
      if (conn.isValid()) {
    	  //判断闲置连接池资源是否已经达到上限
        if (state.idleConnections.size() < poolMaximumIdleConnections && conn.getConnectionTypeCode() == expectedConnectionTypeCode) {
        	//没有达到上限，进行回收
          state.accumulatedCheckoutTime += conn.getCheckoutTime();
          if (!conn.getRealConnection().getAutoCommit()) {
            conn.getRealConnection().rollback();//如果还有事务没有提交，进行回滚操作
          }
          //基于该连接，创建一个新的连接资源，并刷新连接状态
          PooledConnection newConn = new PooledConnection(conn.getRealConnection(), this);
          state.idleConnections.add(newConn);
          newConn.setCreatedTimestamp(conn.getCreatedTimestamp());
          newConn.setLastUsedTimestamp(conn.getLastUsedTimestamp());
          //老连接失效
          conn.invalidate();
          if (log.isDebugEnabled()) {
            log.debug("Returned connection " + newConn.getRealHashCode() + " to pool.");
          }
          //唤醒其他被阻塞的线程
          state.notifyAll();
        } else {//如果闲置连接池已经达到上限了，将连接真实关闭
          state.accumulatedCheckoutTime += conn.getCheckoutTime();
          if (!conn.getRealConnection().getAutoCommit()) {
            conn.getRealConnection().rollback();
          }
          //关闭真的数据库连接
          conn.getRealConnection().close();
          if (log.isDebugEnabled()) {
            log.debug("Closed connection " + conn.getRealHashCode() + ".");
          }
          //将连接对象设置为无效
          conn.invalidate();
        }
      } else {
        if (log.isDebugEnabled()) {
          log.debug("A bad connection (" + conn.getRealHashCode() + ") attempted to return to the pool, discarding connection.");
        }
        state.badConnectionCount++;
      }
    }
  }

```

#### 3.2.2.1 PooledConnection

PooledConnection是proxyConnection的InvocationHandler，其又对真正的realConnection进行了封装。

```
class PooledConnection implements InvocationHandler {

  private static final String CLOSE = "close";
  private static final Class<?>[] IFACES = new Class<?>[] { Connection.class };

  private final int hashCode;
  //记录当前连接所在的数据源对象，本次连接是有这个数据源创建的，关闭后也是回到这个数据源；
  private final PooledDataSource dataSource;
  //真正的连接对象
  private final Connection realConnection;
  //连接的代理对象
  private final Connection proxyConnection;
  //从数据源取出来连接的时间戳
  private long checkoutTimestamp;
  //连接创建的的时间戳
  private long createdTimestamp;
  //连接最后一次使用的时间戳
  private long lastUsedTimestamp;
  //根据数据库url、用户名、密码生成一个hash值，唯一标识一个连接池
  private int connectionTypeCode;
  //连接是否有效
  private boolean valid;

```

#### 3.2.2.2 PoolState

PoolState：用于管理PooledConnection对象状态的组件，通过两个list分别管理空闲状态的连接资源和活跃状态的连接资源

```
public class PoolState {

  protected PooledDataSource dataSource;
  //空闲的连接池资源集合
  protected final List<PooledConnection> idleConnections = new ArrayList<>();
  //活跃的连接池资源集合
  protected final List<PooledConnection> activeConnections = new ArrayList<>();
  //请求的次数
  protected long requestCount = 0;
  //累计的获得连接的时间
  protected long accumulatedRequestTime = 0;
  //累计的使用连接的时间。从连接取出到归还，算一次使用的时间；
  protected long accumulatedCheckoutTime = 0;
  //使用连接超时的次数
  protected long claimedOverdueConnectionCount = 0;
  //累计超时时间
  protected long accumulatedCheckoutTimeOfOverdueConnections = 0;
  //累计等待时间
  protected long accumulatedWaitTime = 0;
  //等待次数 
  protected long hadToWaitCount = 0;
  //无效的连接次数 
  protected long badConnectionCount = 0;
```