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

## 4.3 缓存模块

## 4.4 反射模块