# 1.ORM框架查询数据过程

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/ORM框架查询数据过程.jpg" width="520px" > </div><br>

# 2.反射核心类

* ObjectFactory：MyBatis每次创建结果对象的新实例时，它都会使用对象工厂（ObjectFactory）去构建POJO；
* ReflectorFactory：创建Reflector的工厂类，Reflector是mybatis反射模块的基础，每个Reflector对象都对应一个类，在其中缓存了反射操作所需要的类元信息；
* ObjectWrapper：对对象的包装，抽象了对象的属性信息，他定义了一系列查询对象属性信息的方法，以及更新属性的方法；
* ObjectWrapperFactory： ObjectWrapper 的工厂类，用于创建ObjectWrapper；
* MetaObject：封装了对象元信息，包装了mybatis中五个核心的反射类。也是提供给外部使用的反射工具类，可以利用它可以读取或者修改对象的属性信息


<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/MetaObject.png" width="520px" > </div><br>

```
public class MetaObject {

  //原始的java对象
  private final Object originalObject;
  //对对象的包装，抽象了对象的属性信息，他定义了一系列查询对象属性信息的方法，以及更新属性的方法；
  private final ObjectWrapper objectWrapper;
  private final ObjectFactory objectFactory;
  private final ObjectWrapperFactory objectWrapperFactory;
  private final ReflectorFactory reflectorFactory;
```

## 2.1 实例化目标对象

默认使用resultMap/resultType的默认构造方法，对于配置了constructor的resultMap，则使用带有参数的构造方法。

```
public class DefaultObjectFactory implements ObjectFactory, Serializable {

  private static final long serialVersionUID = -8855120656740914948L;

  @Override
  public <T> T create(Class<T> type) {
    return create(type, null, null);
  }

  @SuppressWarnings("unchecked")
  @Override
  public <T> T create(Class<T> type, List<Class<?>> constructorArgTypes, List<Object> constructorArgs) {
	//判断类是不是集合类，如果是集合类指定具体的实现类
    Class<?> classToCreate = resolveInterface(type);
    // we know types are assignable
    return (T) instantiateClass(classToCreate, constructorArgTypes, constructorArgs);
  }

  @Override
  public void setProperties(Properties properties) {
    // no props for default
  }

  private  <T> T instantiateClass(Class<T> type, List<Class<?>> constructorArgTypes, List<Object> constructorArgs) {
    try {
      Constructor<T> constructor;
      //通过无参构造函数创建对象
      if (constructorArgTypes == null || constructorArgs == null) {
        constructor = type.getDeclaredConstructor();
        if (!constructor.isAccessible()) {
          constructor.setAccessible(true);
        }
        return constructor.newInstance();
      }
      //根据指定的参数列表查找构造函数，并实例化对象
      constructor = type.getDeclaredConstructor(constructorArgTypes.toArray(new Class[constructorArgTypes.size()]));
      if (!constructor.isAccessible()) {
        constructor.setAccessible(true);
      }
      return constructor.newInstance(constructorArgs.toArray(new Object[constructorArgs.size()]));
    } catch (Exception e) {
      StringBuilder argTypes = new StringBuilder();
      if (constructorArgTypes != null && !constructorArgTypes.isEmpty()) {
        for (Class<?> argType : constructorArgTypes) {
          argTypes.append(argType.getSimpleName());
          argTypes.append(",");
        }
        argTypes.deleteCharAt(argTypes.length() - 1); // remove trailing ,
      }
      StringBuilder argValues = new StringBuilder();
      if (constructorArgs != null && !constructorArgs.isEmpty()) {
        for (Object argValue : constructorArgs) {
          argValues.append(String.valueOf(argValue));
          argValues.append(",");
        }
        argValues.deleteCharAt(argValues.length() - 1); // remove trailing ,
      }
      throw new ReflectionException("Error instantiating " + type + " with invalid types (" + argTypes + ") or values (" + argValues + "). Cause: " + e, e);
    }
  }

  protected Class<?> resolveInterface(Class<?> type) {
    Class<?> classToCreate;
    if (type == List.class || type == Collection.class || type == Iterable.class) {
      classToCreate = ArrayList.class;
    } else if (type == Map.class) {
      classToCreate = HashMap.class;
    } else if (type == SortedSet.class) { // issue #510 Collections Support
      classToCreate = TreeSet.class;
    } else if (type == Set.class) {
      classToCreate = HashSet.class;
    } else {
      classToCreate = type;
    }
    return classToCreate;
  }

  @Override
  public <T> boolean isCollection(Class<T> type) {
    return Collection.class.isAssignableFrom(type);
  }

}
```

## 2.2 怎样实现对象属性赋值？

### 2.2.1 DefaultReflectorFactory及Reflector

创建Reflector工程类，并带有缓存功能。

```
public class DefaultReflectorFactory implements ReflectorFactory {
  private boolean classCacheEnabled = true;
  private final ConcurrentMap<Class<?>, Reflector> reflectorMap = new ConcurrentHashMap<>();

  public DefaultReflectorFactory() {
  }

  @Override
  public boolean isClassCacheEnabled() {
    return classCacheEnabled;
  }

  @Override
  public void setClassCacheEnabled(boolean classCacheEnabled) {
    this.classCacheEnabled = classCacheEnabled;
  }

  @Override
  public Reflector findForClass(Class<?> type) {
    if (classCacheEnabled) {
            // synchronized (type) removed see issue #461
      return reflectorMap.computeIfAbsent(type, Reflector::new);
    } else {
      return new Reflector(type);
    }
  }

}

```
Reflector将Class信息按照读写分门别类组织好。

```
public class Reflector {

  private final Class<?> type;//对应的class
  private final String[] readablePropertyNames;//可读属性的名称集合，存在get方法即可读
  private final String[] writeablePropertyNames;//可写属性的名称集合，存在set方法即可写
  private final Map<String, Invoker> setMethods = new HashMap<>();//保存属性相关的set方法
  private final Map<String, Invoker> getMethods = new HashMap<>();//保存属性相关的get方法
  private final Map<String, Class<?>> setTypes = new HashMap<>();//保存属性相关的set方法入参类型
  private final Map<String, Class<?>> getTypes = new HashMap<>();//保存属性相关的get方法返回类型
  private Constructor<?> defaultConstructor;//class默认的构造函数

  //记录所有属性的名称集合
  private Map<String, String> caseInsensitivePropertyMap = new HashMap<>();

  public Reflector(Class<?> clazz) {
    type = clazz;
    addDefaultConstructor(clazz);//获取clazz的默认构造函数
    addGetMethods(clazz);//处理clazz中的get方法信息，填充getMethods、getTypes
    addSetMethods(clazz);//处理clazz中的set方法信息，填充setMethods、setTypes
    addFields(clazz);//处理没有get、set方法的属性
    //根据get、set方法初始化可读属性集合和可写属性集合
    readablePropertyNames = getMethods.keySet().toArray(new String[getMethods.keySet().size()]);
    writeablePropertyNames = setMethods.keySet().toArray(new String[setMethods.keySet().size()]);
    //初始化caseInsensitivePropertyMap
    for (String propName : readablePropertyNames) {
      caseInsensitivePropertyMap.put(propName.toUpperCase(Locale.ENGLISH), propName);
    }
    for (String propName : writeablePropertyNames) {
      caseInsensitivePropertyMap.put(propName.toUpperCase(Locale.ENGLISH), propName);
    }
  }

```

### 2.2.2 ObjectWrapperFactory和ObjectWrapper

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/ObjectWrapper.png" width="620px" > </div><br>

查看BeanWrapper，封装了两个信息：
* 实例化的对象object
* 所属类信息metaClass

```
public class BeanWrapper extends BaseWrapper {

  private final Object object;
  private final MetaClass metaClass;

  public BeanWrapper(MetaObject metaObject, Object object) {
    super(metaObject);
    this.object = object;
    this.metaClass = MetaClass.forClass(object.getClass(), metaObject.getReflectorFactory());
  }

```

以getSetterType为例：

```
  @Override
  public Class<?> getSetterType(String name) {
    PropertyTokenizer prop = new PropertyTokenizer(name);
    if (prop.hasNext()) {
      MetaObject metaValue = metaObject.metaObjectForProperty(prop.getIndexedName());
      if (metaValue == SystemMetaObject.NULL_META_OBJECT) {
        return metaClass.getSetterType(name);
      } else {
        return metaValue.getSetterType(prop.getChildren());
      }
    } else {
      return metaClass.getSetterType(name);
    }
  }
```

最终调用reflector.getSetterType()方法：

```
  public Class<?> getSetterType(String name) {
    PropertyTokenizer prop = new PropertyTokenizer(name);
    if (prop.hasNext()) {
      MetaClass metaProp = metaClassForProperty(prop.getName());
      return metaProp.getSetterType(prop.getChildren());
    } else {
      return reflector.getSetterType(prop.getName());
    }
  }
```

**重点：BeanWrapper可以给对象的属性赋值**

```
  @Override
  public void set(PropertyTokenizer prop, Object value) {
    if (prop.getIndex() != null) {
      Object collection = resolveCollection(prop, object);
      setCollectionValue(prop, collection, value);
    } else {
      setBeanProperty(prop, object, value);
    }
  }


  private void setBeanProperty(PropertyTokenizer prop, Object object, Object value) {
    try {
      Invoker method = metaClass.getSetInvoker(prop.getName());
      Object[] params = {value};
      try {
        method.invoke(object, params);
      } catch (Throwable t) {
        throw ExceptionUtil.unwrapThrowable(t);
      }
    } catch (Throwable t) {
      throw new ReflectionException("Could not set property '" + prop.getName() + "' of '" + object.getClass() + "' with value '" + value + "' Cause: " + t.toString(), t);
    }
  }


```

## 2.3 模拟反射赋值实例

```
	@Test
	public void reflectionTest(){
		
		// step1.创建实例对象
		ObjectFactory objectFactory = new DefaultObjectFactory();
		TUser user = objectFactory.create(TUser.class);

        // step2.反射工具类初始化
		ObjectWrapperFactory objectWrapperFactory = new DefaultObjectWrapperFactory();
		ReflectorFactory reflectorFactory = new DefaultReflectorFactory();
		MetaObject metaObject = MetaObject.forObject(user, objectFactory, objectWrapperFactory, reflectorFactory);
	

		// step3.模拟数据库行数据转化成对象
		//1）模拟从数据库读取数据
		Map<String, Object> dbResult = new HashMap<>();
		dbResult.put("id", 1);
		dbResult.put("user_name", "lison");
		dbResult.put("real_name", "李晓宇");
		TPosition tp = new TPosition();
		tp.setId(1);
		dbResult.put("position_id", tp);
		//2）模拟映射关系
		Map<String, String> mapper = new HashMap<String, String>();
		mapper.put("id", "id");
		mapper.put("userName", "user_name");
		mapper.put("realName", "real_name");
		mapper.put("position", "position_id");
		
		// step3.获取对象和类信息对应的BeanWrapper
		BeanWrapper objectWrapper = (BeanWrapper) metaObject.getObjectWrapper();
		
        // step4.根据数据查出来的数据给对象的属性赋值
		Set<Entry<String, String>> entrySet = mapper.entrySet();
		for (Entry<String, String> colInfo : entrySet) {
			String propName = colInfo.getKey();
			Object propValue = dbResult.get(colInfo.getValue());
			PropertyTokenizer proTokenizer = new PropertyTokenizer(propName);
			objectWrapper.set(proTokenizer, propValue);
		}
		System.out.println(metaObject.getOriginalObject());
			
```