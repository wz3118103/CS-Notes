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

# 2.源码整体结构
<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/MyBatis源码结构.png" width="620px" > </div><br>

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/MyBatis源码分层结构.jpg" width="520px" > </div><br>

# 3.日志模块
三个核心问题以及Mybatis的解决方案：
* MyBatis没有提供日志的实现类，需要接入第三方的日志组件，但第三方日志组件都有各自的Log级别，且各不相同，二MyBatis统一提供trace、debug、warn、error四个级别；
  - 使用了适配器设计模式
* 自动扫描日志实现，并且第三方日志插件加载优先级如下：slf4J → commonsLoging → Log4J2 → Log4J → JdkLog;
* 日志的使用要优雅的嵌入到主体功能中；
  - 使用了动态代理
