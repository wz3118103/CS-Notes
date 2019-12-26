# 1.为什么要用Mybatis而不用JDBC？
jdbc有大量重复样板式代码：

* step1.注册加载mysql驱动
* step2.获取连接Connection （url,user,password)
* step3.创建sql语句Statement，执行并获取结果ResultSet
* step4.处理结果：将ResultSet转化为bean
* step5.关闭连接


对象关系映射（ORM Obeject Relational Mapping），ORM模型就是数据库的表与简单Java对象（POJO）的映射模型，它主要解决数据库数据和POJO对象的相互映射。

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/Mybatis ORM映射.jpg" width="320px" > </div><br>

# 2.mybatis一次数据库访问流程
<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/一次数据库访问流程.jpg" width="320px" > </div><br>

* SqlSessionFactoryBuilder：读取配置信息创建SqlSessionFactory，建造者模式，方法级别生命周期；
* SqlSessionFactory：创建Sqlsession，工厂单例模式，存在于程序的整个生命周期；
* SqlSession：代表一次数据库连接，可以直接发送SQL执行，也可以通过调用Mapper访问数据库；线程不安全，要保证线程独享（方法级）；
* SQL Mapper：由一个Java接口和XML文件组成，包含了要执行的SQL语句和结果集映射规则。方法级别生命周期；

```
public class MybatisQuickStart {
    private SqlSessionFactory sqlSessionFactory;

    @Before
    public void init() throws IOException {
        String resource = "software/mybatis/mybatis-config.xml";
        InputStream inputStream = Resources.getResourceAsStream(resource);
        sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
        inputStream.close();
    }

    @Test
    public void quickStart() {
        SqlSession sqlSession = sqlSessionFactory.openSession();
        TUserMapper mapper = sqlSession.getMapper(TUserMapper.class);
        TUser user = mapper.selectByPrimaryKey(1);
        System.out.println(user);
    }
}
```

mybatis-config.xml
```
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration PUBLIC "-//mybatis.org//DTD Config 3.0//EN" "http://mybatis.org/dtd/mybatis-3-config.dtd">

<configuration>
	<properties resource="software/mybatis/db.properties" />

	<settings>
		<setting name="mapUnderscoreToCamelCase" value="true" />
	</settings>


	<!--配置environment环境 -->
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

	<!-- 映射文件，mapper的配置文件 -->
 	<mappers>
		<mapper resource="software/mybatis/sqlmapper/TUserMapper.xml" />
	</mappers>

</configuration>
```

TUserMapper.xml
```
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd" >
<mapper namespace="software.mybatis.mapper.TUserMapper">
	<resultMap id="BaseResultMap" type="software.mybatis.entity.TUser">

		<id column="id" property="id" jdbcType="INTEGER" />
		<result column="user_name" property="userName" jdbcType="VARCHAR" />
		<result column="real_name" property="realName" jdbcType="VARCHAR" />
		<result column="sex" property="sex" jdbcType="TINYINT" />
		<result column="mobile" property="mobile" jdbcType="VARCHAR" />
		<result column="email" property="email" jdbcType="VARCHAR" />
		<result column="note" property="note" jdbcType="VARCHAR" />
		<result column="position_id" property="positionId" jdbcType="INTEGER" />
	</resultMap>


	<sql id="Base_Column_List">
		id, user_name, real_name, sex, mobile, email, note,
		position_id
	</sql>

	<select id="selectByPrimaryKey" resultMap="BaseResultMap"
		parameterType="java.lang.Integer">
		select
		<include refid="Base_Column_List" />
		from t_user
		where id = #{id,jdbcType=INTEGER}
	</select>
	
</mapper>
```

# 3.mybatis-config.xml各配置作用
<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/mybatis-config配置项.jpg" width="520px" > </div><br>

## 3.1 setting配置
<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/setting配置项1.jpg" width="520px" > </div><br>

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/setting配置项2.jpg" width="520px" > </div><br>

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/setting配置项3.jpg" width="520px" > </div><br>

proxyFactory使用工具CGLIB/JAVASSIST，即使POJO没有get/set方法，也能成功设置属性。

## 3.2 environments

* environment 元素是配置一个数据源的开始，属性id是它的唯一标识
* transactionManager 元素配置数据库事务，其中type属性有三种配置方式
  - jdbc，采用jdbc的方式管理事务；
  - managed，采用容器的方式管理事务，在JNDI数据源中使用；
  - 自定义，自定义数据库事务管理办法；
* dataSource 元素配置数据源连接信息，type属性是连接数据库的方式配置，有四种配置方式
  - UNPOOLED 非连接池方式连接
  - POOLED   使用连接池连接
  - JNDI  使用JNDI数据源
  - 自定义数据源

## 3.3 Mybatis配置mapper

* 用classPath下资源引用（推荐）
```
  <mappers>
  <!--直接映射到相应的mapper文件 -->
  <mapper resource="sqlmapper/TUserMapper.xml" />
  </mappers>
```

* 用类注册方式引用
```
<mappers>
<!—通过类扫描mapper文件 -->
<mapper class="com.enjoylearning.mybatis.mapper.TUserMapper" />
</mappers>
```

* 使用包名引入引射器名
```
<mappers>
<!—扫描包下所有的mapper文件 -->
<package name="com.enjoylearning.mybatis.mapper"/>
</mappers>
```

* 用文件的全路径引用（不推荐）

## 3.4 缓存
一级缓存 select语句中；与SqlSession生命周期一致

二级缓存：与SqlSessionFfactory生命周期一样，namespace区隔

为什么不用二级缓存，容易出现脏读（使用代码验证）

# 4.基于xml配置mapper映射器
作用概述：
  * 写出sql语句；
  * 查出来的结果数据转换成bean

## 4.1 cache/cache-ref

cache – 给定命名空间的缓存配置。
cache-ref – 其他命名空间缓存配置的引用。


## 4.2 resultMap

resultMap 元素是 MyBatis 中最重要最强大的元素。它可以让你从 90% 的 JDBC ResultSets 数据提取代码中解放出来,在对复杂语句进行联合映射的时候，它很可能可以代替数千行的同等功能的代码。


ResultMap 的设计思想是，简单的语句不需要明确的结果映射，而复杂一点的语句只需要描述它们的关系就行了。

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/resultMap属性.jpg" width="520px" > </div><br>

* constructor - 用于在实例化类时，注入结果到构造方法中
  - idArg - ID 参数;标记出作为 ID 的结果可以帮助提高整体性能
  - arg - 将被注入到构造方法的一个普通结果
* id – 一个 ID 结果;标记出作为 ID 的结果可以帮助提高整体性能
* result – 注入到字段或 JavaBean 属性的普通结果
* association – 一个复杂类型的关联;许多结果将包装成这种类型
  - 嵌套结果映射 – 关联可以指定为一个 resultMap 元素，或者引用一个
* collection – 一个复杂类型的集合
  - 嵌套结果映射 – 集合可以指定为一个 resultMap 元素，或者引用一个
* discriminator – 使用结果值来决定使用哪个 resultMap
  - case – 基于某些值的结果映射
    * 嵌套结果映射 – 一个 case 也是一个映射它本身的结果,因此可以包含很多相 同的元素，或者它可以参照一个外部的 resultMap


阿里编程实践：
> （四）ORM映射-3.【强制】
> 不要用 resultClass 当返回参数，即使所有类属性名与数据库字段一一对应，也需要定义；反过来，每一个表也必然有一个 POJO 类与之对应。
说明：配置映射关系，使字段与 DO 类解耦，方便维护。

### 4.2.1 id & result

* id 和 result 都将一个列的值映射到一个简单数据类型(字符串,整型,双精度浮点数,日期等)的属性或字段
* 两者之间的唯一不同是， id 表示的结果将是对象的标识属性，这会在比较对象实例时用到。 这样可以提高整体的性能，尤其是缓存和嵌套结果映射(也就是联合映射)的时候

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/id&result属性.jpg" width="520px" > </div><br>

### 4.2.2 constructor

* 一个pojo不存在没有参数的构造方法，就需要使用constructor;
* 为了通过名称来引用构造方法参数，你可以添加 @Param 注解,指定参数名称的前提下，以任意顺序编写 arg 元素

```
<constructor>
	<idArg column="id" javaType="int" />
	<arg column="user_name" javaType="String" />
</constructor>

```

## 4.3 sql元素和参数

sql元素：用来定义可重用的 SQL 代码段，可以包含在其他语句中；

参数：向sql语句中传递的可变参数
* 预编译 #{}：将传入的数据都当成一个字符串，会对自动传入的数据加一个双引号，能够很大程度防止sql注入；
* 传值 ${}：传入的数据直接显示生成在sql中，无法防止sql注入；
* 表名、选取的列是动态的，order by和in操作， 可以考虑使用$


## 4.4 select

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/select属性.jpg" width="520px" > </div><br>

### 4.4.1 自动映射
前提：SQL列名和JavaBean的属性是一致的；
自动映射等级autoMappingBehavior设置为PARTIAL，需要谨慎使用FULL；
使resultType；
如果列名和JavaBean不一致，但列名符合单词下划线分割，Java是驼峰命名法，则mapUnderscoreToCamelCase可设置为true；

### 4.4.2 传递多个查询入参

* 使用map传递参数；可读性差，导致可维护性和可扩展性差，杜绝使用；
* 使用注解传递参数；直观明了，当参数较少一般小于5个的时候，建议使用；
* 使用Java Bean的方式传递参数；当参数大于5个的时候，建议使用；

阿里编程实践：
> （四）ORM映射-6.【强制】
> 不允许直接拿HashMap与Hashtable作为查询结果集的输出。
说明：resultClass=”Hashtable”，会置入字段名和属性值，但是值的类型不可控。


## 4.5 insert, update 和 delete
<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/增改查属性.jpg" width="520px" > </div><br>

获取键id：
* useGeneratedKeys和keyProperty
* selectKey

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/selectKey属性.jpg" width="520px" > </div><br>

```
<selectKey  keyProperty=“id” order= " Before" resultType="int">
	select SEQ_ID.nextval from dual
</selectKey>

```

## 4.6 动态sql
<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/动态sql.jpg" width="520px" > </div><br>

* if 用在where里面 （可以尽量让if少一点，可以提升sql语句性能）
foreach 
* where （如果没有，则不加where；删掉最前面的一个and 或者or）
set (如果没有，则不加set；删掉最后一个逗号）
trim （insert语句使用trim，where和set使用trim实现）


## 4.7 批量操作

* 通过foreach标签去拼接动态sql语句（需要数据库支持这种语法）
* 过BatchExecutor——defaultExecutorType


## 4.8 关联查询
association（嵌套结果；嵌套查询）

N+1查询问题（主表查询1次，然后根据主表数据查询从表N次）
总开关：
cacheEnabled/
lazyloadingEnabled/
aggressiveLazyLoading
语句开关：fetchType=lazy

外键约束对mysql性能影响比较大

collection

discriminator

阿里编程实践：
> (三)SQL语句-6.【强制】
> 不得使用外键与级联，一切外键概念必须在应用层解决。
说明：以学生和成绩的关系为例，学生表中的student_id是主键，那么成绩表中的student_id则为外键。如果更新学生表中的student_id，同时触发成绩表中的student_id更新，即为级联更新。外键与级联更新适用于单机低并发，不适合分布式、高并发集群；级联更新是强阻塞，存在数据库更新风暴的风险；外键影响数据库的插入速度。


# 5.代码生成器MBG

# 6.与spring集成
Mybatis-spring

# 7.阿里编码规范

# 8.关键的问题