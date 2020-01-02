# 1.需求分析

* Mybatis缓存的实现是基于Map的，从缓存里面读写数据是缓存模块的核心基础功能；
* 除核心功能之外，有很多额外的附加功能，如：防止缓存击穿，添加缓存清空策略（fifo、lru）、序列化功能、日志能力、定时清空能力等；
* 附加功能可以以任意的组合附加到核心基础功能之上


# 2.缓存穿透、缓存击穿、缓存雪崩区别和解决方案

## 2.1 缓存穿透
缓存穿透是指缓存和数据库中都没有的数据，而用户不断发起请求，如发起为id为“-1”的数据或id为特别大不存在的数据。这时的用户很可能是攻击者，攻击会导致数据库压力过大。

缓存穿透，是指查询一个数据库一定不存在的数据。正常的使用缓存流程大致是，数据查询先进行缓存查询，如果key不存在或者key已经过期，再对数据库进行查询，并把查询到的对象，放进缓存。如果数据库查询对象为空，则不放进缓存。

解决方案：
* 接口层增加校验，如用户鉴权校验，id做基础校验，id<=0的直接拦截；
* 从缓存取不到的数据，在数据库中也没有取到，这时也可以将key-value对写为key-null，缓存有效时间可以设置短点，如30秒（设置太长会导致正常情况也没法使用）。这样可以防止攻击用户反复用同一个id暴力攻击

## 2.2 缓存击穿（并发查同一条数据）
缓存击穿，是指一个key非常热点，在不停的扛着大并发，大并发集中对这一个点进行访问，当这个key在失效的瞬间，持续的大并发就穿破缓存，直接请求数据库，就像在一个屏障上凿开了一个洞。

缓存击穿是指缓存中没有但数据库中有的数据（一般是缓存时间到期），这时由于并发用户特别多，同时读缓存没读到数据，又同时去数据库去取数据，引起数据库压力瞬间增大，造成过大压力。

解决方案：
* 设置热点数据永远不过期。
* 加互斥锁，防止都去数据库重复取数据，重复往缓存中更新数据情况出现。能根据key值加锁就更好了，就是线程A从数据库取key1的数据并不妨碍线程B取key2的数据。

## 2.3 缓存雪崩
缓存雪崩是指缓存中数据大批量到过期时间（缓存雪崩，是指在某一个时间段，缓存集中过期失效。），而查询数据量巨大，引起数据库压力过大甚至down机。和缓存击穿不同的是，缓存击穿指并发查同一条数据，缓存雪崩是不同数据都过期了，很多数据都查不到从而查数据库。

解决方案：

* 缓存数据的过期时间设置随机，防止同一时间大量数据过期现象发生。
* 如果缓存数据库是分布式部署，将热点数据均匀分布在不同搞得缓存数据库中。
* 设置热点数据永远不过期。


# 3.装饰器模式

Q.怎么样优雅的为核心功能添加附加能力？
* 使用继承的办法扩展附加功能？继承的方式是静态的，用户不能控制增加行为的方式和时机。另外，新功能的存在多种组合，使用继承可能导致大量子类存在。

装饰器模式（Decorator Pattern）允许向一个现有的对象添加新的功能，是一种用于代替继承的技术，无需通过继承增加子类就能扩展对象的新功能。使用对象的关联关系代替继承关系，更加灵活，同时避免类型体系的快速膨胀。

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/装饰器模式.png" width="520px" > </div><br>


相对于继承，装饰器模式灵活性更强，扩展性更强；
* 灵活性：装饰器模式将功能切分成一个个独立的装饰器，在运行期可以根据需要动态的添加功能，甚至对添加的新功能进行自由的组合；
* 扩展性：当有新功能要添加的时候，只需要添加新的装饰器实现类，然后通过组合方式添加这个新装饰器，无需修改已有代码，符合开闭原则

装饰器模式使用实例：
* IO中输入流和输出流的设计
* Servlet API中提供了一个request对象的Decorator设计模式的默认实现类HttpServletRequestWrapper，HttpServletRequestWrapper类增强了request对象的功能。
* Mybatis的缓存组件


# 4.缓存模块源码分析

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/cache包.jpg" width="320px" > </div><br>

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/PerpetualCache.png" width="520px" > </div><br>

## 4.1 PerpetualCache

实际的缓存实现，使用HashMap实现缓存功能。

```
public class PerpetualCache implements Cache {

  private final String id;

  private Map<Object, Object> cache = new HashMap<>();

  public PerpetualCache(String id) {
    this.id = id;
  }
```

## 4.2 装饰器

### 4.2.1 BlockingCache

阻塞版本的缓存装饰器，保证只有一个线程到数据库去查找指定的key对应的数据。每个key都对应一个锁，提高了并发。

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/每个key对应一个锁.jpg" width="620px" > </div><br>

Q.这里为什么putObject没有加锁？
* 因为写缓存的时机是读取数据库的线程设置的，因为针对同一key只有一个线程读取，所以无需加锁。

```
public class BlockingCache implements Cache {

  //阻塞的超时时长
  private long timeout;
  //被装饰的底层对象，一般是PerpetualCache
  private final Cache delegate;
  //锁对象集，粒度到key值
  private final ConcurrentHashMap<Object, ReentrantLock> locks;

  public BlockingCache(Cache delegate) {
    this.delegate = delegate;
    this.locks = new ConcurrentHashMap<>();
  }

  @Override
  public void putObject(Object key, Object value) {
    try {
      delegate.putObject(key, value);
    } finally {
      releaseLock(key);
    }
  }

  @Override
  public Object getObject(Object key) {
    acquireLock(key);//根据key获得锁对象，获取锁成功加锁，获取锁失败阻塞一段时间重试
    Object value = delegate.getObject(key);
    if (value != null) {//获取数据成功的，要释放锁
      releaseLock(key);
    }        
    return value;
  }

  @Override
  public Object removeObject(Object key) {
    // despite of its name, this method is called only to release locks
    releaseLock(key);
    return null;
  }

  private ReentrantLock getLockForKey(Object key) {
    ReentrantLock lock = new ReentrantLock();//创建锁
    ReentrantLock previous = locks.putIfAbsent(key, lock);//把新锁添加到locks集合中，如果添加成功使用新锁，如果添加失败则使用locks集合中的锁
    return previous == null ? lock : previous;
  }
  
//根据key获得锁对象，获取锁成功加锁，获取锁失败阻塞一段时间重试
  private void acquireLock(Object key) {
	//获得锁对象
    Lock lock = getLockForKey(key);
    if (timeout > 0) {//使用带超时时间的锁
      try {
        boolean acquired = lock.tryLock(timeout, TimeUnit.MILLISECONDS);
        if (!acquired) {//如果超时抛出异常
          throw new CacheException("Couldn't get a lock in " + timeout + " for the key " +  key + " at the cache " + delegate.getId());  
        }
      } catch (InterruptedException e) {
        throw new CacheException("Got interrupted while trying to acquire lock for key " + key, e);
      }
    } else {//使用不带超时时间的锁
      lock.lock();
    }
  }
  
  private void releaseLock(Object key) {
    ReentrantLock lock = locks.get(key);
    if (lock.isHeldByCurrentThread()) {
      lock.unlock();
    }
  }

```

## 4.3 CacheKey

Mybatis中涉及到动态SQL的原因，缓存项的key不能仅仅通过一个String来表示，所以通过CacheKey来封装缓存的Key值，CacheKey可以封装多个影响缓存项的因素；判断两个CacheKey是否相同包括比较（hashCode、checksum、count、以及各元素）是否相等。

构成CacheKey的对象：
* mappedStatment的id
* 指定查询结果集的范围（分页信息）
* 查询所使用的SQL语句
* 用户传递给SQL语句的实际参数值

```
public class CacheKey implements Cloneable, Serializable {

  private static final long serialVersionUID = 1146682552656046210L;

  public static final CacheKey NULL_CACHE_KEY = new NullCacheKey();

  private static final int DEFAULT_MULTIPLYER = 37;
  private static final int DEFAULT_HASHCODE = 17;

  private final int multiplier;//参与hash计算的乘数
  private int hashcode;//CacheKey的hash值，在update函数中实时运算出来的
  private long checksum;//校验和，hash值的和
  private int count;//updateList的中元素个数
  // 8/21/2017 - Sonarlint flags this as needing to be marked transient.  While true if content is not serializable, this is not always true and thus should not be marked transient.
  //该集合中的元素觉得两个CacheKey是否相等
  private List<Object> updateList;
```

### Q.一级缓存在哪里创建？
* CachingExecutor.query最后调用BaseExecutor.createCacheKey

```
public class CachingExecutor implements Executor {

  private final Executor delegate;


  public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
	//获取sql语句信息，包括占位符，参数等信息
    BoundSql boundSql = ms.getBoundSql(parameterObject);
  //拼装缓存的key值
    CacheKey key = createCacheKey(ms, parameterObject, rowBounds, boundSql);
    return query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }

  @Override
  public CacheKey createCacheKey(MappedStatement ms, Object parameterObject, RowBounds rowBounds, BoundSql boundSql) {
    return delegate.createCacheKey(ms, parameterObject, rowBounds, boundSql);
  }
```

BaseExecutor.createCacheKey：

```
  @Override
  public CacheKey createCacheKey(MappedStatement ms, Object parameterObject, RowBounds rowBounds, BoundSql boundSql) {
    if (closed) {
      throw new ExecutorException("Executor was closed.");
    }
    CacheKey cacheKey = new CacheKey();
    cacheKey.update(ms.getId());
    cacheKey.update(rowBounds.getOffset());
    cacheKey.update(rowBounds.getLimit());
    cacheKey.update(boundSql.getSql());
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    TypeHandlerRegistry typeHandlerRegistry = ms.getConfiguration().getTypeHandlerRegistry();
    // mimic DefaultParameterHandler logic
    for (ParameterMapping parameterMapping : parameterMappings) {
      if (parameterMapping.getMode() != ParameterMode.OUT) {
        Object value;
        String propertyName = parameterMapping.getProperty();
        if (boundSql.hasAdditionalParameter(propertyName)) {
          value = boundSql.getAdditionalParameter(propertyName);
        } else if (parameterObject == null) {
          value = null;
        } else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
          value = parameterObject;
        } else {
          MetaObject metaObject = configuration.newMetaObject(parameterObject);
          value = metaObject.getValue(propertyName);
        }
        cacheKey.update(value);
      }
    }
    if (configuration.getEnvironment() != null) {
      // issue #176
      cacheKey.update(configuration.getEnvironment().getId());
    }
    return cacheKey;
  }
```
CacheKey中的值：
* 1）ms.getId()——software.mybatis.mapper.TUserMapper.selectByEmailAndSex2
* 2）rowBounds——查询结果集的范围（分页信息）
* 3）boundSql.getSql()——sql语句
* 4）parameterMappings——用户传递给SQL语句的实际参数值

### update和equals方法

```
  public void update(Object object) {
	//获取object的hash值
    int baseHashCode = object == null ? 1 : ArrayUtil.hashCode(object); 
    //更新count、checksum以及hashcode的值
    count++;
    checksum += baseHashCode;
    baseHashCode *= count;
    hashcode = multiplier * hashcode + baseHashCode;
    //将对象添加到updateList中
    updateList.add(object);
  }
```

```
  @Override
  public boolean equals(Object object) {
    if (this == object) {//比较是不是同一个对象
      return true;
    }
    if (!(object instanceof CacheKey)) {//是否类型相同
      return false;
    }

    final CacheKey cacheKey = (CacheKey) object;

    if (hashcode != cacheKey.hashcode) {//hashcode是否相同
      return false;
    }
    if (checksum != cacheKey.checksum) {//checksum是否相同
      return false;
    }
    if (count != cacheKey.count) {//count是否相同
      return false;
    }

    //以上都不相同，才按顺序比较updateList中元素的hash值是否一致
    for (int i = 0; i < updateList.size(); i++) {
      Object thisObject = updateList.get(i);
      Object thatObject = cacheKey.updateList.get(i);
      if (!ArrayUtil.equals(thisObject, thatObject)) {
        return false;
      }
    }
    return true;
  }

```


### Q.一级缓存在哪里放入键值对？

CachingExecutor.query

```
  @Override
  public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
	//获取sql语句信息，包括占位符，参数等信息
    BoundSql boundSql = ms.getBoundSql(parameterObject);
  //拼装缓存的key值
    CacheKey key = createCacheKey(ms, parameterObject, rowBounds, boundSql);
    return query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }
```

在query方法中，注意ms.isFlushCacheRequired()会根据MappedStatement.flushCacheRequired是否清除一级缓存，这个是语句配置时候的flushCache属性：

```
  @SuppressWarnings("unchecked")
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
```

最终调用到BaseExecutor的.queryFromDatabase，其中调用localCache.putObject存入缓存

```
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

其中localCache如下，localCache是在创建BaseExecutor时就创建了：

```
public abstract class BaseExecutor implements Executor {

  private static final Log log = LogFactory.getLog(BaseExecutor.class);

  protected PerpetualCache localCache;//一级缓存的实现，PerpetualCache
  protected PerpetualCache localOutputParameterCache;//一级缓存用于缓存输出的结果
  protected Configuration configuration;//全局唯一configuration对象的引用


  protected BaseExecutor(Configuration configuration, Transaction transaction) {
    this.transaction = transaction;
    this.deferredLoads = new ConcurrentLinkedQueue<>();
    this.localCache = new PerpetualCache("LocalCache");
    this.localOutputParameterCache = new PerpetualCache("LocalOutputParameterCache");
    this.closed = false;
    this.configuration = configuration;
    this.wrapper = this;
  }

```


Q.二级缓存在哪里使用？

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/二级缓存调用栈.jpg" width="520px" > </div><br>

CachingExecutor.query()：

```
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

在MappedStatement中：

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
```

Q.二级缓存在哪里创建？

在XMLMapperBuilder中：

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

在MapperBuilderAssistant.useNewCache()：

```
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

会将Cache放到MapperBuilderAssistant.currentCache和Configuration的caches中去。

```
public class Configuration {

 
  /*mapper文件中增删改查操作的注册中心*/
  protected final Map<String, MappedStatement> mappedStatements = new StrictMap<>("Mapped Statements collection");
  
  /*mapper文件中配置cache节点的 二级缓存*/
  protected final Map<String, Cache> caches = new StrictMap<>("Caches collection");
```

Q.二级缓存在哪里放入到MappedStatement中？

在XMLMapperBuilder的configurationElement中：
* 首先会在cacheElement()解析出二级缓存的配置，并放到MapperBuilderAssistant的currentCache中
* 然后在构建语句buildStatementFromContext中，会将currentCache赋值给MappedStatement的cache

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

MapperBuilderAssistant中cache(currentCache)：

```
  public MappedStatement addMappedStatement(
      String id,
      SqlSource sqlSource,
      StatementType statementType,
      SqlCommandType sqlCommandType,
      Integer fetchSize,
      Integer timeout,
      String parameterMap,
      Class<?> parameterType,
      String resultMap,
      Class<?> resultType,
      ResultSetType resultSetType,
      boolean flushCache,
      boolean useCache,
      boolean resultOrdered,
      KeyGenerator keyGenerator,
      String keyProperty,
      String keyColumn,
      String databaseId,
      LanguageDriver lang,
      String resultSets) {

    if (unresolvedCacheRef) {
      throw new IncompleteElementException("Cache-ref not yet resolved");
    }

    id = applyCurrentNamespace(id, false);
    boolean isSelect = sqlCommandType == SqlCommandType.SELECT;

    MappedStatement.Builder statementBuilder = new MappedStatement.Builder(configuration, id, sqlSource, sqlCommandType)
        .resource(resource)
        .fetchSize(fetchSize)
        .timeout(timeout)
        .statementType(statementType)
        .keyGenerator(keyGenerator)
        .keyProperty(keyProperty)
        .keyColumn(keyColumn)
        .databaseId(databaseId)
        .lang(lang)
        .resultOrdered(resultOrdered)
        .resultSets(resultSets)
        .resultMaps(getStatementResultMaps(resultMap, resultType, id))
        .resultSetType(resultSetType)
        .flushCacheRequired(valueOrDefault(flushCache, !isSelect))
        .useCache(valueOrDefault(useCache, isSelect))
        .cache(currentCache);

    ParameterMap statementParameterMap = getStatementParameterMap(parameterMap, parameterType, id);
    if (statementParameterMap != null) {
      statementBuilder.parameterMap(statementParameterMap);
    }

    MappedStatement statement = statementBuilder.build();
    configuration.addMappedStatement(statement);
    return statement;
  }
```