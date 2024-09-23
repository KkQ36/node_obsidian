# 概述
## 为什么需要分布式调度？
我们可以通过 Spring 提供的 @Scheduled 注解，或者整合 Quartz 的方式来实现 单机 的任务调度。
比如使用 Spring 提供的任务调度功能：
```java
@Scheduled("0/20 * * * * ?")
public void doWork() {
	// do something
}
```
这样好像就完美的解决问题了，但是在分布式的架构下，如果每个单机中都有一个这样的任务，就会导致任务的重复执行；所以，我们需要分布式任务调度框架来帮助我们统一的管理和执行这些定时任务。
## XXL-JOB 基本介绍
![[Pasted image 20240922105246.png]]
XXL-JOB：是大众点评开源的，轻量级的分布式任务调度平台，其设计核心是开发迅速、学习简单、轻量级、易拓展。
其在大众点评内部使用超过百万次，表现优异。
并且其应用十分广泛，目前已有很多公司接入了 XXL-JOB，只是在官网上公布的就有六百余家。
官网地址：[https://www.xuxueli.com/xxl-job/](https://www.xuxueli.com/xxl-job/)
架构图：
![[Pasted image 20240922105559.png|800]]
Quartz作为开源作业调度中的佼佼者，是作业调度的首选。但是集群环境中Quartz采用API的方式对任务进行管理，从而可以避免上述问题，但是同样存在以下问题：
- 问题一：调用API的的方式操作任务，不人性化；
- 问题二：需要持久化业务QuartzJobBean到底层数据表中，系统侵入性相当严重。
- 问题三：调度逻辑和QuartzJobBean耦合在同一个项目中，这将导致一个问题，在调度任务数量逐渐增多，同时调度任务逻辑逐渐加重的情况下，此时调度系统的性能将大大受限于业务；
- 问题四：quartz底层以“抢占式”获取DB锁并由抢占成功节点负责运行任务，会导致节点负载悬殊非常大；而XXL-JOB通过执行器实现“协同分配式”运行任务，充分发挥集群优势，负载各节点均衡。
XXL-JOB弥补了quartz的上述不足之处。
# 快速入门
参考官方文档完成快速入门：
官方文档地址：[https://www.xuxueli.com/xxl-job/#1.3%20%E7%89%B9%E6%80%A7](https://www.xuxueli.com/xxl-job/#1.3%20%E7%89%B9%E6%80%A7)
### 2.1 初始化“调度数据库”
请下载项目源码并解压，获取 “调度数据库初始化SQL脚本” 并执行即可。
“调度数据库初始化SQL脚本” 位置为:
```java
/xxl-job/doc/db/tables_xxl_job.sql
```
调度中心支持集群部署，集群情况下各节点务必连接同一个mysql实例;
如果mysql做主从,调度中心集群节点务必强制走主库;
### 2.2 编译源码
解压源码,按照maven格式将源码导入IDE, 使用maven进行编译即可，源码结构如下：
xxl-job-admin：调度中心
xxl-job-core：公共依赖
xxl-job-executor-samples：执行器Sample示例（选择合适的版本执行器，可直接使用，也可以参考其并将现有项目改造成执行器）
     ：xxl-job-executor-sample-springboot：Springboot版本，通过Springboot管理执行器，推荐这种方式；
    ：xxl-job-executor-sample-frameless：无框架版本；
### 2.3 配置部署“调度中心”
1. 调度中心项目：xxl-job-admin
2. 作用：统一管理任务调度平台上调度任务，负责触发调度执行，并且提供任务管理平台。
#### 步骤一：调度中心配置：
调度中心配置文件地址：
1. `/xxl-job/xxl-job-admin/src/main/resources/application.properties`
调度中心配置内容说明：
```properties
### 调度中心部署根地址 [选填]：如调度中心集群部署存在多个地址则用逗号分隔。执行器将会使用该地址进行"执行器心跳注册"和"任务结果回调"；为空则关闭自动注册；  
xxl.job.admin.addresses=http://127.0.0.1:8080/xxl-job-admin  
### 执行器通讯TOKEN [选填]：非空时启用；  
xxl.job.accessToken=  
### 执行器AppName [选填]：执行器心跳注册分组依据；为空则关闭自动注册  
xxl.job.executor.appname=xxl-job-executor-sample  
### 执行器注册 [选填]：优先使用该配置作为注册地址，为空时使用内嵌服务 ”IP:PORT“ 作为注册地址。从而更灵活的支持容器类型执行器动态IP和动态映射端口问题。  
xxl.job.executor.address=  
### 执行器IP [选填]：默认为空表示自动获取IP，多网卡时可手动设置指定IP，该IP不会绑定Host仅作为通讯实用；地址信息用于 "执行器注册" 和 "调度中心请求并触发任务"；  
xxl.job.executor.ip=  
### 执行器端口号 [选填]：小于等于0则自动获取；默认端口为9999，单机部署多个执行器时，注意要配置不同执行器端口；  
xxl.job.executor.port=9999  
### 执行器运行日志文件存储磁盘路径 [选填] ：需要对该路径拥有读写权限；为空则使用默认路径；  
xxl.job.executor.logpath=/data/applogs/xxl-job/jobhandler  
### 执行器日志文件保存天数 [选填] ： 过期日志自动清理, 限制值大于等于3时生效; 否则, 如-1, 关闭自动清理功能；  
xxl.job.executor.logretentiondays=30
```
#### 步骤二：部署项目：
如果已经正确进行上述配置，可将项目编译打包部署。
调度中心访问地址：[http://localhost:8080/xxl-job-admin](http://localhost:8080/xxl-job-admin) (该地址执行器将会使用到，作为回调地址)
默认登录账号 “admin/123456”, 登录后运行界面如下图所示。
![[Pasted image 20240922161009.png|700]]
至此“调度中心”项目已经部署成功。
### 2.4 配置部署“执行器项目”
参考原项目中的：xxl-job-executor-sample-springboot 模块来配置执行器。
执行器的作用为：负责接收“调度中心”的调度并执行；可直接部署执行器，也可以将执行器集成到现有业务项目中。
#### 步骤一：maven依赖
如果是新建项目的话，我们需要将拉取下来的项目执行一下 Maven 的 install，将其安装到本地仓库。
然后将原项目中的 pom 文件复制到我们的新项目中：
```java
<?xml version="1.0" encoding="UTF-8"?>  
<project xmlns="http://maven.apache.org/POM/4.0.0"  
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"  
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">  
    <modelVersion>4.0.0</modelVersion>  
    <parent>  
        <groupId>com.xuxueli</groupId>  
        <artifactId>xxl-job-executor-samples</artifactId>  
        <version>2.4.2-SNAPSHOT</version>  
    </parent>  
    <artifactId>xxl-job-executor-sample-springboot</artifactId>  
    <packaging>jar</packaging>  
  
    <name>${project.artifactId}</name>  
    <description>Example executor project for spring boot.</description>  
    <url>https://www.xuxueli.com/</url>  
  
    <properties>  
    </properties>  
  
    <dependencyManagement>  
        <dependencies>  
            <dependency>  
                <!-- Import dependency management from Spring Boot (依赖管理：继承一些默认的依赖，工程需要依赖的jar包的管理，申明其他dependency的时候就不需要version) -->  
                <groupId>org.springframework.boot</groupId>  
                <artifactId>spring-boot-starter-parent</artifactId>  
                <version>${spring-boot.version}</version>  
                <type>pom</type>  
                <scope>import</scope>  
            </dependency>  
        </dependencies>  
    </dependencyManagement>  
  
    <dependencies>  
        <!-- spring-boot-starter-web (spring-webmvc + tomcat) -->  
        <dependency>  
            <groupId>org.springframework.boot</groupId>  
            <artifactId>spring-boot-starter-web</artifactId>  
        </dependency>  
        <dependency>  
            <groupId>org.springframework.boot</groupId>  
            <artifactId>spring-boot-starter-test</artifactId>  
            <scope>test</scope>  
        </dependency>  
  
        <!-- xxl-job-core -->  
        <dependency>  
            <groupId>com.xuxueli</groupId>  
            <artifactId>xxl-job-core</artifactId>  
            <version>${project.parent.version}</version>  
        </dependency>  
  
    </dependencies>  
  
    <build>  
        <plugins>  
            <!-- spring-boot-maven-plugin (提供了直接运行项目的插件：如果是通过parent方式继承spring-boot-starter-parent则不用此插件) -->  
            <plugin>  
                <groupId>org.springframework.boot</groupId>  
                <artifactId>spring-boot-maven-plugin</artifactId>  
                <version>${spring-boot.version}</version>  
                <executions>  
                    <execution>  
                        <goals>  
                            <goal>repackage</goal>  
                        </goals>  
                    </execution>  
                </executions>  
            </plugin>  
        </plugins>  
    </build>  
  
  
</project>
```
#### 步骤二：执行器配置
执行器配置，配置文件地址：
```java
/xxl-job/xxl-job-executor-samples/xxl-job-executor-sample-springboot/src/main/resources/application.properties
```
执行器配置，配置内容说明：
```properties
## 调度中心部署根地址 [选填]：如调度中心集群部署存在多个地址则用逗号分隔。执行器将会使用该地址进行"执行器心跳注册"和"任务结果回调"；为空则关闭自动注册；  
xl.job.admin.addresses=http://127.0.0.1:8080/xxl-job-admin
  
## 执行器通讯TOKEN [选填]：非空时启用；  
xl.job.accessToken=
  
## 执行器AppName [选填]：执行器心跳注册分组依据；为空则关闭自动注册  
xl.job.executor.appname=xxl-job-executor-sample  
## 执行器注册 [选填]：优先使用该配置作为注册地址，为空时使用内嵌服务 ”IP:PORT“ 作为注册地址。从而更灵活的支持容器类型执行器动态IP和动态映射端口问题。  
xxl.job.executor.address=  
### 执行器IP [选填]：默认为空表示自动获取IP，多网卡时可手动设置指定IP，该IP不会绑定Host仅作为通讯实用；地址信息用于 "执行器注册" 和 "调度中心请求并触发任务"；  
xxl.job.executor.ip=
### 执行器端口号 [选填]：小于等于0则自动获取；默认端口为9999，单机部署多个执行器时，注意要配置不同执行器端口；  
xxl.job.executor.port=9999
### 执行器运行日志文件存储磁盘路径 [选填] ：需要对该路径拥有读写权限；为空则使用默认路径；  
xxl.job.executor.logpath=/data/applogs/xxl-job/jobhandler  
### 执行器日志文件保存天数 [选填] ： 过期日志自动清理, 限制值大于等于3时生效; 否则, 如-1, 关闭自动清理功能；  
xxl.job.executor.logretentiondays=30
```
#### 步骤三：执行器组件配置
新建一个配置类，然后向 Application Context 中注入 Bean：`XxlJobSpringExecutor`。
注意：这里配置的是任务的执行器，不是任务，可以理解为另一个程序。
```java
@Configuration  
public class XxlJobConfig {  
  
    private final Logger logger = LoggerFactory.getLogger(XxlJobConfig.class);  
  
    @Value("${xxl.job.admin.addresses}")  
    private String adminAddresses;  
  
    @Value("${xxl.job.accessToken}")  
    private String accessToken;  
  
    @Value("${xxl.job.executor.appname}")  
    private String appname;  
  
    @Value("${xxl.job.executor.address}")  
    private String address;  
  
    @Value("${xxl.job.executor.ip}")  
    private String ip;  
  
    @Value("${xxl.job.executor.port}")  
    private int port;  
  
    @Value("${xxl.job.executor.logpath}")  
    private String logPath;  
  
    @Value("${xxl.job.executor.logretentiondays}")  
    private int logRetentionDays;  
  
  
    @Bean  
    public XxlJobSpringExecutor xxlJobExecutor() {  
        logger.info(">>>>>>>>>>> xxl-job config init.");  
        XxlJobSpringExecutor xxlJobSpringExecutor = new XxlJobSpringExecutor();  
        xxlJobSpringExecutor.setAdminAddresses(adminAddresses);  
        xxlJobSpringExecutor.setAppname(appname);  
        xxlJobSpringExecutor.setAddress(address);  
        xxlJobSpringExecutor.setIp(ip);  
        xxlJobSpringExecutor.setPort(port);  
        xxlJobSpringExecutor.setAccessToken(accessToken);  
        xxlJobSpringExecutor.setLogPath(logPath);  
        xxlJobSpringExecutor.setLogRetentionDays(logRetentionDays);  
  
        return xxlJobSpringExecutor;  
    }  
  
}
```
#### 步骤四：创建任务类
```java
@Component  
public class HelloJob {  
  
    @XxlJob("demoJobHandler")  
    public void execute() {  
        System.out.println("Hello Job，执行时间" + new Date());  
    }  
  
}
```

#### 步骤五：运行 Hello World 程序
首先，保证在调度中心中注册的执行器和我们上面配置的执行器是相同的：
```properties
## 执行器AppName [选填]：执行器心跳注册分组依据；为空则关闭自动注册  
xl.job.executor.appname=xxl-job-executor-sample  
## 执行器注册 [选填]：优先使用该配置作为注册地址，为空时使用内嵌服务 ”IP:PORT“ 作为注册地址。从而更灵活的支持容器类型执行器动态IP和动态映射端口问题。  
xxl.job.executor.address=  
### 执行器IP [选填]：默认为空表示自动获取IP，多网卡时可手动设置指定IP，该IP不会绑定Host仅作为通讯实用；地址信息用于 "执行器注册" 和 "调度中心请求并触发任务"；  
xxl.job.executor.ip=
### 执行器端口号 [选填]：小于等于0则自动获取；默认端口为9999，单机部署多个执行器时，注意要配置不同执行器端口；  
xxl.job.executor.port=9999
```
像这样配置的话，需要保证执行器的地址为 `127.0.0.1:9999`，调度中心默认的执行器可能不是这个端口。
然后我们运行 Hello World 程序：
![[Pasted image 20240922163329.png]]
我们的应用程序和执行器程序是在两个位置启动的。
参考配置，在 **任务管理**模块，新建一个任务，我这里配置的是每五秒执行一次：
![[Pasted image 20240922163431.png|800]]
之后，就可以在操作中，选择启动、停止、删除 或者 执行一次 等操作。
### 2.5 GLUE 模式
比如我们当前有一个 Service 类，其中有一个 `hello()` 方法
```java
@Service  
public class HelloService {  
  
    public void hello(){  
        System.out.println("HelloService say Hello");  
    }  
  
}
```
我们希望在项目上线之后，再不更新项目的情况下，去执行 `HelloService.hello()` 就可以使用到 GLUE 模式。
首先，还是要新建一个任务：
![[Pasted image 20240922164410.png|700]]
我们在操作中可以找到 GLUE IDE，点击，就可以进入编辑的界面，输入以下代码（import  语句也必须输入），执行这个任务，就可以实现调用：
```java
package com.xxl.job.service.handler;

import com.xxl.job.core.context.XxlJobHelper;
import com.xxl.job.core.handler.IJobHandler;
import javax.annotation.Resource;
import com.ryan.demo.xxl_job_demo.service.HelloService;

public class DemoGlueJobHandler extends IJobHandler {
  
  	@Resource
  	HelloService helloService;

	@Override
	public void execute() throws Exception {
		helloService.hello();
	}

}
```
### 2.6 验证分布式集群
我们启动两个相同的应用，并为他们配置不同的端口：
![[Pasted image 20240922170713.png]]
分别配置为，然后启动两个程序
```
-Dserver.port=8088 -Dxxl.job.executor.port=9998
-Dserver.port=8089 -Dxxl.job.executor.port=9999
```
观察调度中心的 **执行器管理** 模块，查看自动配置是否成功，也就是是否出现了两个注册节点，分别为：
```
http://127.0.0.1:9998/
http://127.0.0.1:9999/
```
如果还未更新，可以自己配置，注意一定要按照上面的格式，只配置 `127.0.0.1/9999`、`127.0.0.1/9998` 这样是不行的。
然后回到 **任务管理** 模块，配置我们前面某一个任务的 **路由策略** 为轮询，此时任务会在两个应用之间轮询执行。
![[Pasted image 20240922171045.png|700]]