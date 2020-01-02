# 1.源码包导入
手动下载，然后mvn安装，这样可以进行注释。

* step1.下载MyBatis的源码
* step2.检查maven的版本，必须是3.2.5以上，建议使用maven的最新版本
* step3.mybatis的工程是maven工程，在开发工具中导入，工程必须使用jdk1.8以上版本；
* step4.把mybatis源码的pom文件中<optional>true</optional>，全部改为false；
* step5.加入如下插件，安装时生成源码包
```
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-source-plugin</artifactId>
        <executions>
          <execution>
            <id>attach-sources</id>
            <goals>
              <goal>jar</goal>
            </goals>
          </execution>
        </executions>
      </plugin>
```
* step6.在工程目录下执行 mvn clean install -Dmaven.test.skip=true,将当前工程安装到本地仓库（pdf插件报错的话，需要将这个插件屏蔽）；
* step7.其他工程依赖此工程


# 2.阅读源码的步骤

* step1.精心挑选要阅读的源码项目；
* step2.饮水思源——官方文档，先看文档再看源码；
* step3.下载源码，安装到本地，保证能编译运行；
* step4.从宏观到微观，从整体到细节；
* step5.找到入口，抓主放次，梳理核心流程；
* step6.源码调试，找到核心数据结构和关键类；
* step7.勤练习，多折腾；


# 3.源码整体结构
<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/MyBatis源码结构.png" width="620px" > </div><br>

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/MyBatis源码分层结构.jpg" width="520px" > </div><br>

# 4.基础层

## 4.1 日志模块

详细分析参见：
* [Mybatis日志模块源码解析](https://github.com/wz3118103/CS-Notes/blob/master/notes/Mybatis%E6%97%A5%E5%BF%97%E6%A8%A1%E5%9D%97%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.md)
* [日志slf4j+logback使用](https://github.com/wz3118103/CS-Notes/blob/master/notes/%E6%97%A5%E5%BF%97slf4j%2Blogback%E4%BD%BF%E7%94%A8.md)
* [日志slf4j+logback源码分析](https://github.com/wz3118103/CS-Notes/blob/master/notes/%E6%97%A5%E5%BF%97slf4j%2Blogback%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90.md)

三个核心问题以及Mybatis的解决方案：
* MyBatis没有提供日志的实现类，需要接入第三方的日志组件，但第三方日志组件都有各自的Log级别，且各不相同，二MyBatis统一提供trace、debug、warn、error四个级别；
  - 使用了适配器设计模式，logging包里面定义了针对各种日志的适配器
* 自动扫描日志实现，并且第三方日志插件加载优先级如下：slf4J → commonsLoging → Log4J2 → Log4J → JdkLog;
  - 使用org.apache.ibatis.logging.LogFactory的静态代码在类加载时按顺序选择，该LogFactory在MappedStatement里会进行调用
  - slf4j是个接口，它在加载时会去找实现了org/slf4j/impl/StaticLoggerBinder.class的日志类并进行加载
  - Logback.StaticLoggerBinder的静态代码块在类加载时会读取配置文件logback.xml并将配置信息解析出来放在ch.qos.logback.classic.Logger中
* 日志的使用要优雅的嵌入到主体功能中；
  - 使用了jdk动态代理，以Connection为例，会创建Connection的动态代理，该代理的InvocationHandler是ConnectionLogger，其invoke方法里面调用了statemengLog（从上面MappedStatement里面获取的）在调用Connection具体方法前打印日志

## 4.2 数据源模块

详细分析参见：

* [Mybatis数据源模块源码分析](https://github.com/wz3118103/CS-Notes/blob/master/notes/Mybatis%E6%95%B0%E6%8D%AE%E6%BA%90%E6%A8%A1%E5%9D%97%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90.md)

数据源模块的核心问题及Mybatis解决方法：
* MyBatis不但要能集成第三方的数据源组件，自身也提供了数据源的实现
  - 面向接口编程即可，数据源需要实现DataSourceFactory和DataSource接口
* 一般情况下，数据源的初始化过程参数较多，比较复杂
  - 引入工厂模式将对象的构造和使用分离开来，通过DataSourceFactory创建DataSource并对DataSource进行属性参数的设置（这里调用了反射模块进行设置）
* 分析Mybatis提供的简单池化数据源类PooledDataSourceFactory和PooledDataSource
  - PooledDataSource.getConnection()对实际的DriverManager.getConnection()做了一层处理，其在PoolState中管理了空闲连接池idleConnections和活跃的连接池activeConnections，类型都是List<PooledConnection>。通过这两个集合以及最大限制来管理连接池
  - PooledDataSource.getConnection()会首先获得PooledConnection，PooledConnection封装真正的DriverManager.getConnection()，并且也封装了实现了Connection接口的动态代理类proxyConnection，proxyConnection的InvocationHandler就是PooledConnection。然后返回其proxyConnection
  - 当该proxyConnection调用close()时，会触发PooledConnection.invoke()，其走的是dataSource.pushConnection(this)池化的连接池归还方案

## 4.3 缓存模块

详细分析参见：
* [Mybatis缓存模块分析](https://github.com/wz3118103/CS-Notes/blob/master/notes/Mybatis%E7%BC%93%E5%AD%98%E6%A8%A1%E5%9D%97%E5%88%86%E6%9E%90.md)

缓存模块的核心问题及Mybatis解决方案：
* Mybatis缓存的实现是基于Map的，从缓存里面读写数据是缓存模块的核心基础功能；
  - PerpetualCache使用HashMap实现缓存功能
* 除核心功能之外，有很多额外的附加功能，如：防止缓存击穿，添加缓存清空策略（fifo、lru）、序列化功能、日志能力、定时清空能力等；
  - 使用装饰器模式附加各种功能
* 附加功能可以以任意的组合附加到核心基础功能之上
  - 装饰器模式是的功能可以以任意的组合附加到核心功能之上
* 缓存分为二级缓存和一级缓存
  - 代码中先使用二级缓存判断，二级缓存根据配置会使用各种装饰器，二级缓存存放在MappedStatement.cache中
  - 一级缓存存放在BaseExecutor中，直接使用的是PerpetualCache

## 4.4 反射模块

详细分析参见：

* [Mybatis反射模块源码分析](https://github.com/wz3118103/CS-Notes/blob/master/notes/Mybatis%E5%8F%8D%E5%B0%84%E6%A8%A1%E5%9D%97%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90.md)


反射模块的核心目的及解决方案：

* 实例化目标对象
  - 使用ObjectFactory创建目标对象
* 根据xml文件配置给对象属性赋值
  - 使用ReflectorFactory创建Reflector对对象进行赋值，最后都会通过查找Reflector相应属性的方法，然后通过反射调用method.invoke(object, params)对对象属性进行赋值
* 目标对象和类信息进行关联
  - 使用ObjectWrapperFactory创建的ObjectWrapper来进行关联。ObjectWrapper包含实例化的对象object和所属类信息metaClass（由ReflectorFactory和Reflector组成）

# 5.Mybatis核心流程

详细分析参见：

* [Mybatis核心流程分析](https://github.com/wz3118103/CS-Notes/blob/master/notes/Mybatis%E6%A0%B8%E5%BF%83%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90.md)

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/Mybatis核心流程三阶段.jpg" width="520px" > </div><br>

各流程阶段需要解决的问题：

* 初始化——配置加载阶段
  - SqlSessionFactoryBuilder.build("mybatis-config.xml")方法解析配置文件，并生成SqlSessionFactory，该SqlSessionFactory在Configuration中包含完整的配置信息
  - XMLConfigBuilder解析mybatis-config.xml
  - XMLMapperBuilder解析mapper.xml
  - XMLStatementBuilder解析sql语句
  - 包括如下关键类：
    * Configuration ： Mybatis启动初始化的核心就是将所有xml配置文件信息加载到Configuration对象中， Configuration是单例的，生命周期是应用级的；
    * MapperRegistry：mapper接口动态代理工厂类的注册中心。在MyBatis中，通过mapperProxy实现InvocationHandler接口，MapperProxyFactory用于生成动态代理的实例对象；
    * ResultMap：用于解析mapper.xml文件中的resultMap节点，使用ResultMapping来封装id，result等子元素；
    * MappedStatement：用于存储mapper.xml文件中的select、insert、update和delete节点，同时还包含了这些节点的很多重要属性；
    * SqlSource：mapper.xml文件中的sql语句会被解析成SqlSource对象，经过解析SqlSource包含的语句最终仅仅包含？占位符，可以直接提交给数据库执行
* 代理阶段——生成mapper的代理对象
  - SqlSession sqlSession = sqlSessionFactory.openSession()此时会生成Executor，然后根据Executor生成DefaultSqlSession
    * 这里Executor生成有三层
    * 第一层根据executorType值——BATCH、RESUE、SIMPLE，生成BatchExecutor、ReuseExecutor或者SimpleExecutor
    * 第二层根据是否开启二级缓存，executor = new CachingExecutor(executor)
    * 第三层添加插件executor = (Executor) interceptorChain.pluginAll(executor)
  - sqlSession.getMapper()会从自身的configuration中调用mapperRegistry.getMapper(type, sqlSession)，也即获取指定类型的MapperProxyFactory生成实现指定Mapper接口（例如TUerMapper接口）的动态代理对象
  - 该动态代理对象的InvocationHandler是MapperProxy
* 执行阶段——调用Mapper接口动态代理对象进行执行
  - 该执行会调用至InvocationHandler也即MapperProxy的invoke方法，invoke方法最终会调用MapperMethod.execute(sqlSession, args)
  - MapperMethod.execute会根据sql语句类型和返回参数调用至SqlSession的相应方法
* 最终调用至SqlSession
  - DefaultSqlSession.selectList()方法最终会调用Executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER)
  - Executor执行顺序如下
    * step1.各插件代理对象的处理
    * step2.CachingExecutor.query()进行二级缓存查询
    * step3.调用SimpleExecutor/BatchExecutor的query()方法，该方法在父类BaseExecutor中实现，进行一级缓存查询
    * step4.最后才父类BaseExecutor.query()中调用doQuery()，才开始调用SimpleExecutor/BatchExecutor的doQuery()方法
* 执行具体查询的四大组件，在SimpleExecutor.doQuery()方法中，Executor是个指挥官，它调度三个小弟（StatementHandler、ParameterHandler和ResultSetHandler）工作
  - 驱动加载在配置文件解析阶段解析environments节点时就完成了，其会创建数据源工程，比如UnpooledDataSourceFactory，在其构造方法中会创建数据源UnpooledDataSource，在创建该数据源是，其静态代码块就加载了驱动
  - 在SimpleExecutor.prepareStatement()中，首先调用Connection connection = getConnection(statementLog);获取连接，并使用动态代理增强Connection，使得其有日志打印能力
    * BaseExecutor.getConnection()会调用transaction.getConnection()
    * JdbcTransaction.getConnection()调用该类的openConnection()
    * 进而调用connection = dataSource.getConnection()，这就与前面的数据源模块关联起来了
  - 会首先调用PreparedStatementHandler的父类BaseStatementHandler.prepare()，该方法又会调用子类PreparedStatementHandler.instantiateStatement(connection)方法，该方法会调用connection.prepareStatement(sql)生成Statement
  - 然后调用PreparedStatementHandler.parameterize(stmt)，其调用DefaultParameterHandler.setParameters((PreparedStatement) statement)对sql语句的占位符进行处理，也即设置参数
  - 最后调用PreparedStatementHandler.query()，执行查询语句，并使用反射模块将查询到的结果通过DefaultResultSetHandler.handleResultSets()赋值给POJO对象的相应属性
  
Executor中执行查询的流程就是JDBC流程：

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/JDBC流程.jpg" width="520px" > </div><br>


## 5.1 配置加载阶段

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/所有Builder都将信息汇总到Configuration.jpg" width="620px" > </div><br>

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/配置文件与Configuration映射关系.jpg" width="620px" > </div><br>

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/Mybatis初始化.png" width="820px" > </div><br>

## 5.2 代理阶段和执行阶段

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/动态代理增强.jpg" width="620px" > </div><br>

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/binding模块.png" width="420px" > </div><br>

* MapperRegistry ： mapper接口和对应的代理对象工厂的注册中心；
* MapperProxyFactory：用于生成mapper接口动态代理的实例对象；
* MapperProxy：实现了InvocationHandler接口，它是增强mapper接口的实现；
* MapperMethod：封装了Mapper接口中对应方法的信息，以及对应的sql语句的信息；它是mapper接口与映射配置文件中sql语句的桥梁

## 5.3 最终调用至SqlSession

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/SqlSession查询接口嵌套关系.jpg" width="720px" > </div><br>

## 5.4 执行查询的四大组件

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/Executor内部运作过程.png" width="620px" > </div><br>

通过对SimpleExecutor doQuery()方法的解读发现，Executor是个指挥官，它在调度三个小弟工作：

* StatementHandler：它的作用是使用数据库的Statement或PrepareStatement执行操作，启承上启下作用；
* ParameterHandler：对预编译的SQL语句进行参数设置，SQL语句中的的占位符“？”都对应BoundSql.parameterMappings集合中的一个元素，在该对象中记录了对应的参数名称以及该参数的相关属性
* ResultSetHandler：对数据库返回的结果集（ResultSet）进行封装，返回用户指定的实体类型


<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/CallableStatementHandler.png" width="620px" > </div><br>


# 6.插件

详细分析参见：

* [Mybatis插件](https://github.com/wz3118103/CS-Notes/blob/master/notes/Mybatis%E6%8F%92%E4%BB%B6.md)

插件本质也是使用jdk的动态代理进行增强，并在创建四大对象时调用interceptorChain.pluginAll()生成代理：

* newExecutor
* newStatementHandler
* newParameterHandler
* newResultSetHandler