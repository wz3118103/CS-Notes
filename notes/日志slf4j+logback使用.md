# 1.Logback的主要模块
Logback是由log4j创始人设计的另一个开源日志组件。它当前分为下面三个模块：
* logback-core：其它两个模块的基础模块
* logback-classic：它是log4j的一个改良版本，同时它完整实现了slf4j API使你可以很方便地更换成其它日志系统如log4j或JDK14 Logging
* logback-access：访问模块与Servlet容器集成提供通过Http来访问日志的功能

# 2.Logback的主要标签
* logger 存放日志对象，定义日志类型级别
* appender 指定日志输出的目的地
* layout 格式化日志信息

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/日志.jpg" width="520px" > </div><br>
# 3.logback使用
## 3.1 logback加载配置文件的顺序
* java -Dlogback.configurationFile=xxx/xxx.xml
* 查找该目录下面logback-test.xml
* 就会去查找classpath（也即resources目录下面）logback.groovy文件
* 查找该目录下面logback.xml
* 如果还没有，如果是jdk1.6以上，就会去查找com.qos.logback.classic.spi.Configurator的第一个实现类
* 如果还没有，就使用ch.qos.logback.classic.BasicConfigurator，功能是在控制台直接输出日志信息


## 3.2 logback.xml配置文件
* scan如果为true，表示如果配置信息如果改变，会自动重新加载配置文件信息
  - scanPeriod表示多长时间去感知配置是否有变化
* debug，如果为true会打印logback内部的信息。因此一般设置为false。
* 定义参数常量：日志级别、日志存放时长、日志存放的文件、日志格式
* ConsoleAppender
  - encoder 将日志信息转换成字符串，并将其输出到日志文件中
* RollingFileAppender
  - file 日志文件
  - rollingPolicy 日志滚动策略（TimeBasedRollingPolicy）
    * fileNamePattern 压缩日志文件格式
    * maxHistory 保留日志文件个数（超过后会删除最高的）
  - encoder 将日志信息转换成字符串，并将其输出到日志文件中
  - filter 过滤器，可以按照日志级别进行过滤
* Logger存放日志对象，指定关注哪个package下面的信息
  - name表示关注的package
  - level表示logback仅记录哪个级别以上的信息
  - 与appender-ref对象绑定，然后logback往appender指定的地方输出日志信息
  - additivity="true"，会将root里面的appender-ref放到自己的appender-ref里面，并且其level不是按照root的level，而是按照自己的level进行日志信息输出
  - 业务里面一个类只能定义一个Logger
* root
  - 如果上面的Logger没有配置level，则其继承该处root定义的level信息
  - appender-ref，一般设置为consoleAppender


```
<?xml version="1.0" encoding="UTF-8"?>
<configuration scan="true" scanPeriod="60 seconds" debug="false">
    <!--定义参数常量-->
    <property name="log.level" value="debug"/>
    <property name="log.maxHistory" value="30"/>
    <property name="log.filePath" value="${catalina.base}/logs/webapps"/>
    <property name="log.pattern"
              value="%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} %msg%n"/>

    <appender name="consoleAppender" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>${log.pattern}</pattern>
        </encoder>
    </appender>
    <appender name="debugAppender" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${log.filePath}/debug.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${log.filePath}/debug/debug.%d{yyyy-MM-dd}.log.gz</fileNamePattern>
            <maxHistory>${log.maxHistory}</maxHistory>
        </rollingPolicy>
        <encoder>
            <pattern>${log.pattern}</pattern>
        </encoder>
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>DEBUG</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
    </appender>
    <appender name="infoAppender" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${log.filePath}/info.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${log.filePath}/info/info.%d{yyyy-MM-dd}.log.gz</fileNamePattern>
            <maxHistory>${log.maxHistory}</maxHistory>
        </rollingPolicy>
        <encoder>
            <pattern>${log.pattern}</pattern>
        </encoder>
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>INFO</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
    </appender>
    <appender name="errorAppender" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${log.filePath}/error.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${log.filePath}/error/error.%d{yyyy-MM-dd}.log.gz</fileNamePattern>
            <maxHistory>${log.maxHistory}</maxHistory>
        </rollingPolicy>
        <encoder>
            <pattern>${log.pattern}</pattern>
        </encoder>
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>ERROR</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
    </appender>
    <!--additivity为true时，会继承root的appender放到logger下面，并且会按照
      logger中的level-->
    <logger name="com.imooc.o2o" level="${log.level}" additivity="true">
        <appender-ref ref="debugAppender"/>
        <appender-ref ref="infoAppender"/>
        <appender-ref ref="errorAppender"/>
    </logger>
    <root level="info">
        <appender-ref ref="consoleAppender"/>
    </root>
</configuration>
```

# 4.slf4j调用logback
```
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class AreaServiceImpl implements AreaService {
    @Autowired
    private AreaDao areaDao;
    @Autowired
    private JedisUtil.Keys jedisKeys;
    @Autowired
    private JedisUtil.Strings jedisStrings;

    private static Logger logger = LoggerFactory.getLogger(AreaServiceImpl.class);

    @Override
    @Transactional
    public List<Area> getAreaList() {
        String key = AREALISTKEY;
        List<Area> areaList = null;
        ObjectMapper mapper = new ObjectMapper();
        if (!jedisKeys.exists(key)) {
            areaList = areaDao.queryArea();
            String jsonString;
            try {
                jsonString = mapper.writeValueAsString(areaList);
                jedisStrings.set(key, jsonString);
            } catch (JsonProcessingException e) {
                e.printStackTrace();
                logger.error(e.getMessage());
                throw new AreaOperationException(e.getMessage());
            }
        } else {
            String jsonString = jedisStrings.get(key);
            JavaType javaType = mapper.getTypeFactory().constructParametricType(ArrayList.class,
                    Area.class);
            try {
                areaList = mapper.readValue(jsonString, javaType);
            } catch (IOException e) {
                e.printStackTrace();
                logger.error(e.getMessage());
                throw new AreaOperationException(e.getMessage());
            }
        }
        return areaList;
    }
```
<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/slf4j绑定不同的日志打印类.png" width="520px" > </div><br>



# 参考
* [logback的使用和logback.xml详解](https://www.cnblogs.com/warking/p/5710303.html)
* [slf4j+logback的配置及使用](https://www.jianshu.com/p/696444e1a352)