# 1.为什么要用Mybatis而不用JDBC？
jdbc有大量重复样板式代码：

* step1.注册加载mysql驱动
* step2.获取连接Connection （url,user,password)
* step3.创建sql语句Statement，执行并获取结果ResultSet
* step4.处理结果：将ResultSet转化为bean
* step5.关闭连接


对象关系映射（ORM Obeject Relational Mapping），ORM模型就是数据库的表与简单Java对象（POJO）的映射模型，它主要解决数据库数据和POJO对象的相互映射。

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/Mybatis ORM映射.jpg" width="520px" > </div><br>

# 2.mybatis一次数据库访问流程
<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/一次数据库访问流程.jpg" width="520px" > </div><br>

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

### 3.4.1 一级缓存
MyBatis 包含一个非常强大的查询缓存特性，使用缓存可以使应用更快地获取数据，避免频繁的数据库交互 ；

* 一级缓存 ：
  - 一级缓存默认会启用，想要关闭一级缓存可以在select标签上配置flushCache=“true”；
  - 一级缓存存在于 SqlSession 的生命周期中，在同一个 SqlSession 中查询时， MyBatis 会把执行的方法和参数通过算法生成缓存的键值，将键值和查询结果存入一个 Map对象中。如果同一个 SqlSession 中执行的方法和参数完全一致，那么通过算法会生成相同的键值，当 Map 缓存对象中己经存在该键值时，则会返回缓存中的对象;
  - 任何的 INSERT 、UPDATE 、 DELETE 操作都会清空一级缓存；

### 3.4.2 二级缓存
* 二级缓存 ：
  - 二级缓存存在于 SqlSessionFactory 的生命周期中，可以理解为跨sqlSession；缓存是以namespace为单位的，不同namespace下的操作互不影响。
  - setting参数 cacheEnabled，这个参数是二级缓存的全局开关，默认值是 true，如果把这个参数设置为 false，即使有后面的二级缓存配置，也不会生效；
  - 要开启二级缓存,你需要在你的 SQL 映射文件中添加配置：
  - 字面上看就是这样。这个简单语句的效果如下：
    * 映射语句文件中的所有 select 语句将会被缓存。
    * 映射语句文件中的所有 insert,update 和 delete 语句会刷新缓存。
    * 缓存会使用 Least Recently Used(LRU,最近最少使用的)算法来收回。
    * 根据时间表(比如 no Flush Interval,没有刷新间隔), 缓存不会以任何时间顺序 来刷新。
    * 缓存会存储列表集合或对象(无论查询方法返回什么)的 512个引用。
    * 缓存会被视为是 read/write(可读/可写)的缓存；


Tips:  使用二级缓存容易出现脏读，建议避免使用二级缓存，在业务层使用可控制的缓存代替更好。

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/缓存.jpg" width="520px" > </div><br>

打开方式，虽然全局是打开的，但各命名空间默认是关闭的，可通过在mapper.xml中按如下方式打开：
```
<cache></cache>
```
走二级缓存日志：
```
16:37:14.079 [main] DEBUG software.mybatis.mapper.TUserMapper - Cache Hit Ratio [software.mybatis.mapper.TUserMapper]: 0.0
```

# 4.基于xml配置mapper映射器
作用概述：
  * 写出sql语句；
  * 查出来的结果数据转换成bean

## 4.1 各命名空间的二级缓存配置cache/cache-ref

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

### 4.2.1 关联查询

在关系型数据库中，我们经常要处理一对一 、 一对多的关系 。 例如， 一辆汽车需要有一个引擎，这是一对一的关系。 一辆汽车有 4 个或更多个轮子，这是一对多的关系 。关联元素就是专门用来处理关联关系的。

关联元素：
* association      一对一关系
* collection        一对多关系
* discriminator   鉴别器映射

关联方式：
* 嵌套结果:使用嵌套结果映射来处理重复的联合结果的子集
* 嵌套查询:通过执行另外一个 SQL 映射语句来返回预期的复杂类型

#### 4.2.1.1 一对一：嵌套结果
```
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
```
```
	<resultMap id="userAndPosition1" extends="BaseResultMap" type="TUser">
		<association property="position" javaType="TPosition" columnPrefix="post_">
			<id column="id" property="id"/>
			<result column="name" property="postName"/>
			<result column="note" property="note"/>
		</association>
	</resultMap>
```
查询时组成的sql语句：
```
15:47:58.730 [main] DEBUG s.m.m.T.selectUserPosition1 - ==>  Preparing: select a.id, user_name, real_name, sex, mobile, email, a.note, b.id post_id, b.post_name, b.note post_note from t_user a, t_position b where a.position_id = b.id
```
association标签 嵌套结果方式 常用属性：
* property ：对应实体类中的属性名，必填项。
* javaType ： 属性对应的 Java 类型 。
* resultMap ： 可以直接使用现有的 resultMap ，而不需要在这里配置映射关系。
* columnPrefix ：查询列的前缀，配置前缀后，在子标签配置 result 的 column 时可以省略前缀

Tips:
* resultMap可以通过使用extends实现继承关系，简化很多配置工作量；
* 关联的表查询的类添加前缀是编程的好习惯；
* 通过添加完整的命名空间，可以引用其他xml文件的resultMap；

#### 4.2.1.2 一对一：嵌套查询
```
		List<TUser> list2 = mapper.selectUserPosition2();
		for (TUser tUser : list2) {
			System.out.println(tUser.getId());
		}
```
```
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
	<resultMap id="userAndPosition2" extends="BaseResultMap" type="TUser">
		<association property="position" fetchType="lazy"  column="position_id" select="software.mybatis.mapper.TPositionMapper.selectByPrimaryKey" />
	</resultMap>
```
association标签 嵌套查询方式 常用属性： 
* select ：另 一个映射查询的 id, MyBatis 会额外执行这个查询获取嵌套对象的结果 。
* column ：列名（或别名），将主查询中列的结果作为嵌套查询的参数。
* fetchType ：数据加载方式，可选值为 lazy 和 eager，分别为延迟加载和积极加载 ，这个配置会覆盖全局的 lazyLoadingEnabled 配置；

Tips：“N+1 查询问题”
概括地讲,N+1 查询问题可以是这样引起的:
* 你执行了一个单独的 SQL 语句来获取结果列表(就是“+1”)。
* 对返回的每条记录,你执行了一个查询语句来为每个加载细节(就是“N”)。

这个问题会导致成百上千的 SQL 语句被执行。这通常不是期望的。
解决办法：使用“fetchType=lazy”并且全局setting进行改善：
<setting name="aggressiveLazyLoading" value="false"/>

如果是tUser.getPosition()，则会调用嵌套查询：
```
16:00:55.445 [main] DEBUG s.m.m.T.selectUserPosition2 - ==>  Preparing: select a.id, a.user_name, a.real_name, a.sex, a.mobile, a.position_id from t_user a 
16:00:55.500 [main] DEBUG s.m.m.T.selectUserPosition2 - ==> Parameters: 
16:00:55.668 [main] DEBUG s.m.m.T.selectUserPosition2 - <==      Total: 10
16:00:55.670 [main] DEBUG s.m.m.T.selectByPrimaryKey - ==>  Preparing: select id, post_name, note from t_position where id = ? 
16:00:55.671 [main] DEBUG s.m.m.T.selectByPrimaryKey - ==> Parameters: 1(Integer)
16:00:55.748 [main] DEBUG s.m.m.T.selectByPrimaryKey - <==      Total: 1
software.mybatis.entity.TPosition@e350b40
16:00:55.748 [main] DEBUG s.m.m.T.selectByPrimaryKey - ==>  Preparing: select id, post_name, note from t_position where id = ? 
16:00:55.748 [main] DEBUG s.m.m.T.selectByPrimaryKey - ==> Parameters: 2(Integer)
16:00:55.802 [main] DEBUG s.m.m.T.selectByPrimaryKey - <==      Total: 1
software.mybatis.entity.TPosition@2794eab6
software.mybatis.entity.TPosition@e350b40
software.mybatis.entity.TPosition@e350b40
null
null
null
software.mybatis.entity.TPosition@e350b40
null
null
```

因为tUser.getId()只会用到t_user，不用查询t_position，最终只会查询一次数据库：
```
15:56:35.426 [main] DEBUG s.m.m.T.selectUserPosition2 - ==>  Preparing: select a.id, a.user_name, a.real_name, a.sex, a.mobile, a.position_id from t_user a 
15:56:35.477 [main] DEBUG s.m.m.T.selectUserPosition2 - ==> Parameters: 
15:56:35.634 [main] DEBUG s.m.m.T.selectUserPosition2 - <==      Total: 10
1
2
3
126
127
128
129
131
132
133
```

#### 4.2.1.3 一对多
* collection 支持的属性以及属性的作用和 association 完全相同
* mybatis会根据id标签，进行字段的合并，合理配置好ID标签可以提高处理的效率；

```
	<select id="selectUserJobs1" resultMap="userAndJobs1">
		select
		a.id,
		a.user_name,
		a.real_name,
		a.sex,
		a.mobile,
		b.comp_name,
		b.years,
		b.title
		from t_user a,
		t_job_history b
		where a.id = b.user_id

	</select>
```
```
	<resultMap id="userAndJobs1" extends="BaseResultMap" type="TUser">
		<collection property="jobs"
			ofType="software.mybatis.entity.TJobHistory" >
			<result column="comp_name" property="compName" jdbcType="VARCHAR" />
			<result column="years" property="years" jdbcType="INTEGER" />
			<result column="title" property="title" jdbcType="VARCHAR" />
		</collection>
	</resultMap>
```
以上为嵌套结果，如下为嵌套查询：
```
	<select id="selectUserJobs2" resultMap="userAndJobs2">
		select
		a.id,
		a.user_name,
		a.real_name,
		a.sex,
		a.mobile
		from t_user a
	</select>
```
```
	<resultMap id="userAndJobs2" extends="BaseResultMap" type="TUser">
		<collection property="jobs" fetchType="lazy" column="id"
			select="software.mybatis.mapper.TJobHistoryMapper.selectByUserId" />
	</resultMap>
```

#### 4.2.1.4 discriminator 鉴别器映射

在特定的情况下使用不同的pojo进行关联， 鉴别器元素就是被设计来处理这个情况的。鉴别器非常容易理解,因为它的表现很像 Java 语言中的 switch 语句；

* discriminator 标签常用的两个属性如下：
  - column：该属性用于设置要进行鉴别比较值的列 。
  - javaType：该属性用于指定列的类型，保证使用相同的 Java 类型来比较值。
* discriminator 标签可以有1个或多个 case 标签， case 标签包含以下三个属性 。
  - value ： 该值为 discriminator 指定 column 用来匹配的值 。
  - resultMap ： 当column的值和value的值匹配时，可以配置使用resultMap指定的映射，resultMap优先级高于 resultType 。
  - resultType ： 当 column 的值和 value 的值匹配时，用于配置使用 resultType指定的映射。
  
```
	<select id="selectUserHealthReport" resultMap="userAndHealthReport">
		select
		<include refid="Base_Column_List" />
		from t_user a
	</select>
```
```
	<resultMap id="userAndHealthReportMale" extends="userAndHealthReport" type="TUser">
		<collection property="healthReports" column="id"
			select= "software.mybatis.mapper.THealthReportMaleMapper.selectByUserId"></collection>
	</resultMap>
	
	<resultMap id="userAndHealthReportFemale" extends="userAndHealthReport" type="TUser">
		<collection property="healthReports" column="id"
			select= "software.mybatis.mapper.THealthReportFemaleMapper.selectByUserId"></collection>
	</resultMap>
	
	<resultMap id="userAndHealthReport" extends="BaseResultMap" type="TUser">
				 
		<discriminator column="sex"  javaType="int">
			<case value="1" resultMap="userAndHealthReportMale"/>
			<case value="2" resultMap="userAndHealthReportFemale"/>
		</discriminator>
	</resultMap>
```
查询的过程：
```
[main] DEBUG s.m.m.T.selectUserHealthReport - ==>  Preparing: select id, user_name, real_name, sex, mobile, email, note, position_id from t_user a 
16:17:26.852 [main] DEBUG s.m.m.T.selectUserHealthReport - ==> Parameters: 
16:17:26.975 [main] DEBUG s.m.m.T.selectByUserId - ====>  Preparing: select id, check_project, detail, user_id from t_health_report_male where user_id = ? 
16:17:26.976 [main] DEBUG s.m.m.T.selectByUserId - ====> Parameters: 1(Integer)
16:17:27.051 [main] DEBUG s.m.m.T.selectByUserId - <====      Total: 3
16:17:27.054 [main] DEBUG s.m.m.T.selectByUserId - ====>  Preparing: select id, check_project, detail, user_id from t_health_report_male where user_id = ? 
16:17:27.055 [main] DEBUG s.m.m.T.selectByUserId - ====> Parameters: 2(Integer)
16:17:27.110 [main] DEBUG s.m.m.T.selectByUserId - <====      Total: 0
16:17:27.113 [main] DEBUG s.m.m.T.selectByUserId - ====>  Preparing: select id, item, score, user_id from t_health_report_female where user_id = ? 
16:17:27.113 [main] DEBUG s.m.m.T.selectByUserId - ====> Parameters: 3(Integer)
```

#### 4.2.1.5 多对多
* 先决条件一：多对多需要一种中间表建立连接关系；
* 先决条件二：多对多关系是由两个一对多关系组成的，一对多可以也可以用两种方式实现；

```
	<select id="selectUserRole" resultMap="userRoleInfo">
		select a.id, 
		      a.user_name,
		      a.real_name,
		      a.sex,
		      a.mobile,
		      a.note,
		      b.role_id,
		      c.role_name,
		      c.note role_note
		from t_user a,
		     t_user_role b,
		     t_role c
		where a.id = b.user_id AND 
		      b.role_id = c.id
     </select>	
```
```
	<resultMap type="TUser" id="userRoleInfo" extends="BaseResultMap">
		<collection property="roles" ofType="TRole" columnPrefix="role_">
			<result column="id" property="id" />
			<result column="Name" property="roleName" />
			<result column="note" property="note" />
		</collection>
	</resultMap>
```


阿里编程实践：
> (三)SQL语句-6.【强制】
> 不得使用外键与级联，一切外键概念必须在应用层解决。
说明：以学生和成绩的关系为例，学生表中的student_id是主键，那么成绩表中的student_id则为外键。如果更新学生表中的student_id，同时触发成绩表中的student_id更新，即为级联更新。外键与级联更新适用于单机低并发，不适合分布式、高并发集群；级联更新是强阻塞，存在数据库更新风暴的风险；外键影响数据库的插入速度。


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

```
	<select id="selectBySymbol" resultMap="BaseResultMap">
		select
		#{inCol}
		from ${tableName} a
		where a.sex = #{sex}
		order by ${orderStr}
	</select>
```
```
11:15:08.445 [main] DEBUG s.m.m.TUserMapper.selectBySymbol - ==>  Preparing: select ? from t_user a where a.sex = ? order by sex,user_name 
11:15:08.506 [main] DEBUG s.m.m.TUserMapper.selectBySymbol - ==> Parameters: id, user_name, real_name, sex, mobile, email, note(String), 1(Byte)
```
```
	<select id="selectBySymbol" resultMap="BaseResultMap">
		select
		#{inCol}
		from #{tableName} a
		where a.sex = #{sex}
		order by ${orderStr}
	</select>

```
```
11:46:11.396 [main] DEBUG s.m.m.TUserMapper.selectBySymbol - ==>  Preparing: select ? from ? a where a.sex = ? order by sex,user_name 
11:46:11.474 [main] DEBUG s.m.m.TUserMapper.selectBySymbol - ==> Parameters: id, user_name, real_name, sex, mobile, email, note(String), t_user(String), 1(Byte)

mysql> select id, user_name, real_name, sex, mobile, email, note, position_id from "t_user" where id = 1;
ERROR 1064 (42000): You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near '"t_user" where id = 1' at line 1
```

阿里编程实践：
> （四）ORM映射-4.【强制】sql.xml 配置参数使用：#{}，#param# 不要使用${} 此种方式容易出现 SQL 注入。

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
* selectKey（mysql在插入数据库之后才能获取，所以order="after"；oracle实在插入之前，所以使用order="before"）

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
* choose只会按照顺序第一个条件，其后的条件不处理

```
	<select id="selectChooseOper" resultMap="BaseResultMap">
		select
		<include refid="Base_Column_List" />
		from t_user a
		<where>
			<choose>
				<when test="email != null and email != ''">
					and a.email like CONCAT('%', #{email}, '%')
				</when>
				<when test="sex != null">
					and a.sex = #{sex}
				</when>
				<otherwise>
					and 1=1
				</otherwise>
			
			
			</choose>
		</where>
	</select>
```



## 4.7 批量操作

* 通过foreach标签去拼接动态sql语句（需要数据库支持这种语法）
* 过BatchExecutor——defaultExecutorType

## 4.8 注解方式
```
public interface TJobHistoryAnnoMapper {
	
    int deleteByPrimaryKey(Integer id);


    int insertSelective(TJobHistory record);

    TJobHistory selectByPrimaryKey(Integer id);

    int updateByPrimaryKeySelective(TJobHistory record);

    int updateByPrimaryKey(TJobHistory record);
    
    
    @Results(id="jobInfo",value={
    		@Result(property="id",column="id",id = true),
    		@Result(property="userId",column="user_id"),
    		@Result(property="compName",column="comp_name"),
    		@Result(property="years",column="years"),
    		@Result(property="title",column="title")
    })
    @Select("select id, user_id, comp_name, years, title from t_job_history"
    		+ "	where user_id = #{userId}")
    List<TJobHistory> selectByUserId(int userId);
    
    @ResultMap("jobInfo")
    @Select("select id, user_id, comp_name, years, title from t_job_history")
    List<TJobHistory> selectAll();
    
    @Insert("insert into t_job_history (id, user_id, comp_name,	years, title)"
    		+ "	values (#{id,jdbcType=INTEGER}, #{userId,jdbcType=INTEGER},"
    		+ "#{compName,jdbcType=VARCHAR},"
    		+ "#{years,jdbcType=INTEGER}, #{title,jdbcType=VARCHAR})")
    @Options(useGeneratedKeys=true,keyProperty="id")
    int insert(TJobHistory record);
}
```
必须要添加到mappers中去，否则会报错
```
	<mappers>
		<!--直接映射到相应的mapper文件 -->
		<mapper resource="software/mybatis/sqlmapper/demo/TUserMapper.xml" />
		<mapper resource="software/mybatis/sqlmapper/demo/TJobHistoryMapper.xml" />
		<mapper resource="software/mybatis/sqlmapper/demo/TPositionMapper.xml" />
		<mapper resource="software/mybatis/sqlmapper/demo/THealthReportFemaleMapper.xml" />
		<mapper resource="software/mybatis/sqlmapper/demo/THealthReportMaleMapper.xml" />
		<mapper class="software.mybatis.mapper.TJobHistoryAnnoMapper"/>
	</mappers>
```
```
org.apache.ibatis.binding.BindingException: Type interface software.mybatis.mapper.TJobHistoryAnnoMapper is not known to the MapperRegistry.

```
另外，测试时，必须commit，否则不会添加到数据库中去：
```
		TJobHistory job = new TJobHistory();
		job.setTitle("产品经理");
		job.setUserId(1);
		job.setCompName("美团");
		job.setYears(3);
		
		mapper.insert(job);
		System.out.println(job.getId());
		sqlSession.commit();
```

# 5.代码生成器MBG
MyBatis Generator：MyBatis 的开发团队提供了一个很强大的代码生成器，代码包含了数据库表对应的实体类 、Mapper 接口类、 Mapper XML 文件和 Example 对象等，这些代码文件中几乎包含了全部的单表操作方法，使用 MBG 可以极大程度上方便我们使用 MyBatis，还可以减少很多重复操作。


* generatorConfiguration – 根节点
  - properties – 用于指定一个需要在配置中解析使用的外部属性文件；
  - classPathEntry - 在MBG工作的时候，需要额外加载的依赖包；
  - context -用于指定生成一组对象的环境
    * property (0 个或多个） - 设置一些固定属性
    * plugin (0 个或多个）- 定义一个插件，用于扩展或修改通过 MBG 生成的代码
    * commentGenerator (0 个或 1 个） - 该标签用来配置如何生成注释信息
    * jdbcConnection ( 1 个）- 必须要有的，使用这个配置链接数据库
    * javaTypeResolver ( 0 个或 1 个） - 指定 JDBC 类型和 Java 类型如何转换
    * javaModelGenerator ( 1 个） - java模型创建器
    * sqlMapGenerator (0 个或 1 个）- 生成SQL map的XML文件生成器
    * javaClientGenerator (0 个或 1 个）- 生成Mapper接口
    * table ( 1个或多个） -选择一个table来生成相关文件，可以有一个或多个table


## 5.1 怎么运行MGB

### 5.1.1 从命令提示符 使用 XML 配置文件
* step1.根据generatorConfig.xml建立好目录结构
* step2.将generatorConfig.xml mybatis-generator-core-1.3.5.jar mysql-connector-java-xxx.jar放到与src同一级目录下面
<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/mbg目录.jpg" width="220px" > </div><br>
* step3.同时注意修改generatorConfig.xml引用的<classPathEntry location，修改成当前路径即可
* step4.执行下面的命令
```
java -jar mybatis-generator-core-x.x.x.jar -configfile generatorConfig.xml
```
使用场景：对逆向工程定制较少，项目工程结构比较复杂的情况

劣势——定制麻烦：利于要依赖某个父类，此时这种方法需要将其打包成jar包，然后放在<classPathEntry location中。

优势——如果有很多工程，可以一次性完成

### 5.1.2 作为 Maven Plugin
```
mvn mybatis-generator:generate
```
作为maven插件引入mybatis.generator：
```
            <plugin>
                <groupId>org.mybatis.generator</groupId>
                <artifactId>mybatis-generator-maven-plugin</artifactId>
                <version>1.3.2</version>
                <configuration>
                    <configurationFile>src/main/resources/software/mybatis/generatorConfig.xml</configurationFile>
                    <verbose>true</verbose>
                    <overwrite>true</overwrite>
                </configuration>
            </plugin>
```
注意配置configurationFile的位置。

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/mbg.jpg" width="520px" > </div><br>

generatorConfig.xml：
```
<?xml version="1.0" encoding="UTF-8" ?>  
<!DOCTYPE generatorConfiguration PUBLIC   
"-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN"  
 "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd" >
<generatorConfiguration>
	<!-- 引入配置文件 -->
	<properties resource="software/mybatis/db.properties" />

	<!-- 加载数据库驱动 -->
	<classPathEntry location="${class_path}" />
	
	<!-- context:生成一组对象的环境 
			id:必选，上下文id，用于在生成错误时提示 
			defaultModelType:指定生成对象的样式 
				 1，conditional：类似hierarchical；
				 2，flat：所有内容（主键，blob）等全部生成在一个对象中，推荐使用； 
		  		 3，hierarchical：主键生成一个XXKey对象(key class)，Blob等单独生成一个对象，其他简单属性在一个对象中(record class) 
		  	targetRuntime: 
		  		 1，MyBatis3：默认的值，生成基于MyBatis3.x以上版本的内容，包括XXXBySample； 
		         2，MyBatis3Simple：类似MyBatis3，只是不生成XXXBySample； 
     -->
	<context id="context1" targetRuntime="MyBatis3Simple"	defaultModelType="flat">
	
	    <!-- 生成的Java文件的编码 -->
    	<property name="javaFileEncoding" value="UTF-8"/>
		
		<commentGenerator>
			<!-- 是否去除自动生成的注释 true：是 ： false:否 -->
			<property name="suppressAllComments" value="false" />
			<!-- 阻止注释中包含时间戳 true：是 ： false:否 -->
			<property name="suppressDate" value="true" />
			<!--  注释是否包含数据库表的注释信息  true：是 ： false:否 -->
			<property name="addRemarkComments" value="true" />
		</commentGenerator>
		
		<!--数据库连接的信息：驱动类、连接地址、用户名、密码 -->
		<jdbcConnection driverClass="${jdbc_driver}"
			connectionURL="${jdbc_url}" userId="${jdbc_username}" password="${jdbc_password}" />


        <!-- java模型创建器，是必须要的元素   负责：1，key类（见context的defaultModelType）；2，java类；3，查询类
        	targetPackage：生成的类要放的包，真实的包受enableSubPackages属性控制；
        	targetProject：目标项目，指定一个存在的目录下，生成的内容会放到指定目录中，如果目录不存在，MBG不会自动建目录
     	-->
		<javaModelGenerator targetPackage="software.mybatis.mbg.entity" targetProject="${project_src}">
			<!-- 设置一个根对象，
	                      如果设置了这个根对象，那么生成的keyClass或者recordClass会继承这个类；在Table的rootClass属性中可以覆盖该选项
	                      注意：如果在key class或者record class中有root class相同的属性，MBG就不会重新生成这些属性了，包括：
	                1，属性名相同，类型相同，有相同的getter/setter方法；
	         -->
			<property name="rootClass" value="software.mybatis.mbg.entity.BaseEntity" />
		</javaModelGenerator>


		<!-- 生成SQL map的XML文件生成器，
            targetPackage：生成的类要放的包，真实的包受enableSubPackages属性控制；
        	targetProject：目标项目，指定一个存在的目录下，生成的内容会放到指定目录中，如果目录不存在，MBG不会自动建目录
         -->
		<sqlMapGenerator targetPackage="." targetProject="${project_mapper_xml}">
		</sqlMapGenerator>
		
		
		 <!-- 对于mybatis来说，即生成Mapper接口，注意，如果没有配置该元素，那么默认不会生成Mapper接口 
		        type：选择怎么生成mapper接口（在MyBatis3/MyBatis3Simple下）：
		            1，ANNOTATEDMAPPER：会生成使用Mapper接口+Annotation的方式创建（SQL生成在annotation中），不会生成对应的XML；
		            2，MIXEDMAPPER：使用混合配置，会生成Mapper接口，并适当添加合适的Annotation，但是XML会生成在XML中；
		            3，XMLMAPPER：会生成Mapper接口，接口完全依赖XML；
		        注意，如果context是MyBatis3Simple：只支持ANNOTATEDMAPPER和XMLMAPPER
		    -->		
    	<javaClientGenerator  targetPackage="software.mybatis.mbg.mapper"	targetProject="${project_src}" type="XMLMAPPER" />




		<!-- shema 数据库 tableName表名 -->
		<table schema="${jdbc_username}" tableName="t_health_report_female" >
			<generatedKey column="id" sqlStatement="MySql"/>
		</table>
		<table schema="${jdbc_username}" tableName="t_health_report_male" >
			<generatedKey column="id" sqlStatement="MySql"/>
		</table>
		<table schema="${jdbc_username}" tableName="t_job_history" >
			<generatedKey column="id" sqlStatement="MySql"/>
		</table>
		<table schema="${jdbc_username}" tableName="t_position" >
			<generatedKey column="id" sqlStatement="MySql"/>
		</table>
		<table schema="${jdbc_username}" tableName="t_role" >
			<generatedKey column="id" sqlStatement="MySql"/>
		</table>
		<table schema="${jdbc_username}" tableName="t_role_permission" >
			<generatedKey column="id" sqlStatement="MySql"/>
		</table>
		<table schema="${jdbc_username}" tableName="t_user" >
			<generatedKey column="id" sqlStatement="MySql"/>
		</table>
		<table schema="${jdbc_username}" tableName="t_user_role" >
			<generatedKey column="id" sqlStatement="MySql"/>
		</table>

	</context>
</generatorConfiguration>
```

### 5.1.3 从另一个 Java 程序 使用 XML 配置文件

第二与第三种方式使用场景：对逆向工程定制较多，项目工程结构比较单一的情况

引入依赖：
```
        <dependency>
            <groupId>org.mybatis.generator</groupId>
            <artifactId>mybatis-generator-core</artifactId>
            <version>1.3.5</version>
            <scope>test</scope>
        </dependency>
```
生成方法：
```
	@Test
	public void mybatisGeneratorTest() throws FileNotFoundException{
		List<String> warnings = new ArrayList<String>();  
        boolean overwrite = true;
        String genCfg = "software/mybatis/generatorConfig.xml";
        File configFile = new File(getClass().getClassLoader().getResource(genCfg).getFile());
        ConfigurationParser cp = new ConfigurationParser(warnings);  
        Configuration config = null;  
        try {  
            config = cp.parseConfiguration(configFile);  
        } catch (IOException e) {  
            e.printStackTrace();  
        } catch (XMLParserException e) {  
            e.printStackTrace();  
        }  
        DefaultShellCallback callback = new DefaultShellCallback(overwrite);  
        MyBatisGenerator myBatisGenerator = null;  
        try {  
            myBatisGenerator = new MyBatisGenerator(config, callback, warnings);  
        } catch (InvalidConfigurationException e) {  
            e.printStackTrace();  
        }  
        try {  
            myBatisGenerator.generate(null);  
        } catch (SQLException e) {  
            e.printStackTrace();  
        } catch (IOException e) {  
            e.printStackTrace();  
        } catch (InterruptedException e) {  
            e.printStackTrace();  
        }  
    }
```

# 6.与spring集成
Mybatis-spring 用于帮助你将 MyBatis 代码无缝地整合到 Spring 中。 
* Spring 将会加载必要的 MyBatis 工厂类和 session 类
* 提供一个简单的方式来注入 MyBatis 数据映射器和 SqlSession 到业务层的 bean 中。 
* 方便集成spring事务
* 翻译 MyBatis 的异常到 Spring 的 DataAccessException 异常(数据访问异常)中。

## 6.1 集成配置最佳实践

#### step1.准备spring项目一个

#### step2.在pom文件中添加mybatis-spring的依赖
```
<dependency>
	<groupId>org.mybatis</groupId>
	<artifactId>mybatis-spring</artifactId>
	<version>1.3.0</version>
</dependency>

```

#### step3.配置SqlSessionFactoryBean
在 MyBatis-Spring 中， SqlSessionFactoryBean 是用于创建 Sql SessionFactory 的。
*  dataSource ：用于配置数据源，该属性为必选项，必须通过这个属性配置数据源 ，这里使用了上一节中配置好的 dataSource 数据库连接池 。
*  mapper Locations ： 配置 SqlSessionFactoryBean 扫描 XML 映射文件的路径，可以使用 Ant 风格的路径进行配置。
*  configLocation ：用于配置mybatis config XML的路径，除了数据源外，对MyBatis的各种配置仍然可以通过这种方式进行，并且配置MyBatis settings 时只能使用这种方式。但配置文件中任意环境,数据源 和 MyBatis 的事务管理器都会被忽略；
*  typeAliasesPackage ： 配置包中类的别名，配置后，包中的类在 XML 映射文件中使用时可以省略包名部分 ，直接使用类名。这个配置不支持 Ant风格的路径，当需要配置多个包路径时可以使用分号或逗号进行分隔 。


#### step4.配置MapperScannerConfigurer
通过 MapperScannerConfigurer类自动扫描所有的 Mapper 接口，使用时可以直接注入接口 。

MapperScannerConfigurer中常配置以下两个属性。
*  basePackage ： 用于配置基本的包路径。可以使用分号或逗号作为分隔符设置多于一个的包路径，每个映射器将会在指定的包路径中递归地被搜索到 。
*  annotationClass ： 用于过滤被扫描的接口，如果设置了该属性，那么 MyBatis 的接口只有包含该注解才会被扫描进去


#### step5.配置事务


整体的applicationContext.xml配置：
```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:c="http://www.springframework.org/schema/c"
	xmlns:context="http://www.springframework.org/schema/context"
	xmlns:cache="http://www.springframework.org/schema/cache" xmlns:tx="http://www.springframework.org/schema/tx"
	xmlns:redisson="http://redisson.org/schema/redisson" xmlns:aop="http://www.springframework.org/schema/aop"
	xsi:schemaLocation="http://www.springframework.org/schema/beans
                        http://www.springframework.org/schema/beans/spring-beans.xsd
		                http://www.springframework.org/schema/context 
		                http://www.springframework.org/schema/context/spring-context-4.3.xsd
		                http://www.springframework.org/schema/cache
                        http://www.springframework.org/schema/cache/spring-cache.xsd
                        http://www.springframework.org/schema/tx 
          				http://www.springframework.org/schema/tx/spring-tx-3.2.xsd
          				http://redisson.org/schema/redisson
          				http://redisson.org/schema/redisson/redisson.xsd
          				http://www.springframework.org/schema/aop
        				http://www.springframework.org/schema/aop/spring-aop-4.2.xsd">

	<context:property-placeholder location="classpath:software/mybatis/db.properties"
		ignore-unresolvable="true" />

	<bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource"
		init-method="init" destroy-method="close">
		<!-- 基本属性 url、user、password -->
		<property name="driverClassName" value="${jdbc_driver}" />
		<property name="url" value="${jdbc_url}" />
		<property name="username" value="${jdbc_username}" />
		<property name="password" value="${jdbc_password}" />

		<!-- 配置初始化大小、最小、最大 -->
		<property name="initialSize" value="1" />
		<property name="minIdle" value="1" />
		<property name="maxActive" value="20" />

		<!-- 配置获取连接等待超时的时间 -->
		<property name="maxWait" value="60000" />

		<!-- 配置间隔多久才进行一次检测，检测需要关闭的空闲连接，单位是毫秒 -->
		<property name="timeBetweenEvictionRunsMillis" value="60000" />

		<!-- 配置一个连接在池中最小生存的时间，单位是毫秒 -->
		<property name="minEvictableIdleTimeMillis" value="300000" />

		<property name="validationQuery" value="SELECT 'x'" />
		<property name="testWhileIdle" value="true" />
		<property name="testOnBorrow" value="false" />
		<property name="testOnReturn" value="false" />

		<!-- 打开PSCache，并且指定每个连接上PSCache的大小 -->
		<property name="poolPreparedStatements" value="true" />
		<property name="maxPoolPreparedStatementPerConnectionSize"
			value="20" />
		<!-- 配置监控统计拦截的filters -->
		<property name="filters" value="stat" />
	</bean>


	<!-- spring和MyBatis完美整合，不需要mybatis的配置映射文件 -->
	<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
		<property name="dataSource" ref="dataSource" />
		<property name="typeAliasesPackage" value="software.mybatis.entity" />
		<property name="mapperLocations" value="classpath:software/mybatis/sqlmapper/demo/*.xml" />
	</bean>

	 <bean id="tUserMapper" class="org.mybatis.spring.mapper.MapperFactoryBean">
		<property name="mapperInterface" value="software.mybatis.mapper.TUserMapper"></property>
		<property name="sqlSessionFactory" ref="sqlSessionFactory"></property>
	</bean>
	
	<!-- DAO接口所在包名，Spring会自动查找其下的类 -->
	 <bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
		<property name="basePackage" value="software.mybatis.mapper" />
	</bean> 



	<!-- (事务管理)transaction manager -->
	<bean id="transactionManager"
		class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
		<property name="dataSource" ref="dataSource" />
		<qualifier value="transactionManager" />
	</bean>

	<tx:annotation-driven transaction-manager="transactionManager" />


	<context:component-scan base-package="software.mybatis.*">
	</context:component-scan>

</beans>

```
UserServiceImpl.java：
```
@Service
public class UserServiceImpl implements UserService {
	
	@Resource(name="tUserMapper")
	private TUserMapper userMapper;

	@Override
	public TUser getUserById(Integer id) {
		return userMapper.selectByPrimaryKey(id);
	}
}
```
测试代码：
```
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration("classpath:software/mybatis/applicationContext.xml")
public class MybatisSpringTest {

	@Resource
	private UserService us;
	
	@Test
	public void TestSpringMybatis(){
		System.out.println(us.getUserById(1).toString());
	}
	
}

```


# 7.阿里编码规范
(四) ORM 映射

1. 【强制】在表查询中，一律不要使用 * 作为查询的字段列表，需要哪些字段必须明确写明。
 说明：1）增加查询分析器解析成本。2）增减字段容易与 resultMap 配置不一致。3）无用字段增加网络消耗，尤其是 text 类型的字段。

2. 【强制】POJO 类的布尔属性不能加 is，而数据库字段必须加 is_，要求在 resultMap 中进行字段与属性之间的映射。
说明：参见定义 POJO 类以及数据库字段定义规定，在<resultMap>中增加映射，是必须的。在MyBatis Generator 生成的代码中，需要进行对应的修改。

3. 【强制】不要用 resultClass 当返回参数，即使所有类属性名与数据库字段一一对应，也需要定义；反过来，每一个表也必然有一个 POJO 类与之对应。
说明：配置映射关系，使字段与 DO 类解耦，方便维护。

4. 【强制】sql.xml 配置参数使用：#{}，#param# 不要使用${} 此种方式容易出现 SQL 注入。

5. 【强制】iBATIS 自带的 queryForList(String statementName,int start,int size)不推荐使用。
说明：其实现方式是在数据库取到 statementName 对应的 SQL 语句的所有记录，再通过 subList 取start,size 的子集合。
正例：
```
  Map<String, Object> map = new HashMap<>(); 
  map.put("start", start); 
  map.put("size", size); 
```

6. 【强制】不允许直接拿 HashMap 与 Hashtable 作为查询结果集的输出。
说明：resultClass=”Hashtable”，会置入字段名和属性值，但是值的类型不可控。

7. 【强制】更新数据表记录时，必须同时更新记录对应的 gmt_modified 字段值为当前时间。

8. 【推荐】不要写一个大而全的数据更新接口。传入为 POJO 类，不管是不是自己的目标更新字段，都进行 update table set c1=value1,c2=value2,c3=value3; 这是不对的。执行 SQL时，不要更新无改动的字段，一是易出错；二是效率低；三是增加 binlog 存储。

9. 【参考】@Transactional 事务不要滥用。事务会影响数据库的 QPS，另外使用事务的地方需要考虑各方面的回滚方案，包括缓存回滚、搜索引擎回滚、消息补偿、统计修正等。

10. 【参考】<isEqual>中的 compareValue 是与属性值对比的常量，一般是数字，表示相等时带上此条件；<isNotEmpty>表示不为空且不为 null 时执行；<isNotNull>表示不为 null 值时执行。

# 8.相关问题

Q1.PreparedStatement有什么优势？
* 防止SQL注入
* 提高性能：每当数据库执行一个查询时，它总是首先通过计算来确定查询策略，以便高效地执行查询操作。通过事先准备好查询并多次重用它，可以确保查询所需的准备步骤只被执行一次。