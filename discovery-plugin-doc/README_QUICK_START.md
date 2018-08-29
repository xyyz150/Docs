# Nepxion Discovery
[![Total lines](https://tokei.rs/b1/github/Nepxion/Discovery?category=lines)](https://github.com/Nepxion/Discovery)
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg?label=license)](https://github.com/Nepxion/Discovery/blob/master/LICENSE)
[![Maven Central](https://img.shields.io/maven-central/v/com.nepxion/discovery.svg?label=maven%20central)](http://search.maven.org/#search%7Cga%7C1%7Cg%3A%22com.nepxion%22%20AND%20discovery)
[![Javadocs](http://www.javadoc.io/badge/com.nepxion/discovery.svg)](http://www.javadoc.io/doc/com.nepxion/discovery)
[![Build Status](https://travis-ci.org/Nepxion/Discovery.svg?branch=master)](https://travis-ci.org/Nepxion/Discovery)
[![Codacy Badge](https://api.codacy.com/project/badge/Grade/8e39a24e1be740c58b83fb81763ba317)](https://www.codacy.com/project/HaojunRen/Discovery/dashboard?utm_source=github.com&amp;utm_medium=referral&amp;utm_content=Nepxion/Discovery&amp;utm_campaign=Badge_Grade_Dashboard)

## 入门教程
- 本教程以Eureka为服务注册发现，Redis为远程配置中心为例展开讲解，旨在帮您快速把本框架集成到您的业务系统中
- 本教程略掉其它非必须或者锦上添花的功能，如您需要，请参考[主页](https://github.com/Nepxion/Discovery/blob/master/README.md)和[示例演示](https://github.com/Nepxion/Docs/blob/master/discovery-plugin-doc/README_EXAMPLE.md)
- 本教程略掉Spring Cloud基础部分，例如如何搭建微服务、Eureka、Zuul或者Spring Cloud Api Gateway（F版）等不在本文介绍范围内

### 集成到微服务、Zuul或者Spring Cloud Api Gateway（F版）
#### 引入Pom依赖
- 引入全局Pom依赖

插件版本，请参考[主页](https://github.com/Nepxion/Discovery/blob/master/README.md)的“依赖”章节，请根据Spring Cloud不同版本选择正确的插件版本
```xml
<dependency>
    <groupId>com.nepxion</groupId>
    <artifactId>discovery</artifactId>
    <version>${discovery.plugin.version}</version>
    <type>pom</type>
    <scope>import</scope>
</dependency>
```
- 引入Eureka插件依赖
```xml
<dependency>
    <groupId>com.nepxion</groupId>
    <artifactId>discovery-plugin-starter-eureka</artifactId>
</dependency>
```
- 引入Redis远程配置中心扩展依赖
```xml
<dependency>
    <groupId>com.nepxion</groupId>
    <artifactId>discovery-plugin-config-center-starter-redis</artifactId>
</dependency>
```
- :exclamation:如果需要，引入用户自定义和编程灰度路由扩展依赖（三个依赖分别是服务端，网关Zuul端，网关Spring Cloud Api Gateway（F版）端，对应引入）
```xml
<dependency>
    <groupId>com.nepxion</groupId>
    <artifactId>discovery-plugin-strategy-starter-service</artifactId>
    <artifactId>discovery-plugin-strategy-starter-zuul</artifactId>
    <artifactId>discovery-plugin-strategy-starter-gatewway</artifactId>
</dependency>
```

#### 添加配置
```xml
# Eureka config
eureka.instance.metadataMap.version=1.0
eureka.instance.metadataMap.group=example-service-group
eureka.instance.metadataMap.region=dev

# Redis config
spring.redis.host=localhost
spring.redis.port=6379
spring.redis.password=
spring.redis.database=0
spring.redis.pool.max-active=8
spring.redis.pool.max-wait=-1
spring.redis.pool.max-idle=8
spring.redis.pool.min-idle=0

# Admin config
# 关闭访问Rest接口时候的权限验证
management.security.enabled=false
# E版配置方式
management.port=5100
# F版配置方式
management.server.port=5100
```

#### 更多信息
- 请参考master（Finchley）分支或者Edgware分支下的discovery-springcloud-example-service、discovery-springcloud-example-zuul、discovery-springcloud-example-gateway三个工程

### 搭建控制平台
#### 引入Pom依赖
- 引入全局Pom依赖

插件版本，请参考[主页](https://github.com/Nepxion/Discovery/blob/master/README.md)的“依赖”章节，请根据Spring Cloud不同版本选择正确的插件版本
```xml
<dependency>
    <groupId>com.nepxion</groupId>
    <artifactId>discovery</artifactId>
    <version>${discovery.plugin.version}</version>
    <type>pom</type>
    <scope>import</scope>
</dependency>
```
- 引入Redis远程配置中心扩展依赖
```xml
<dependency>
    <groupId>com.nepxion</groupId>
    <artifactId>discovery-console-starter-redis</artifactId>
</dependency>
```
- 引入Eureka Client依赖

#### 添加配置
```xml
# Redis config
spring.redis.host=localhost
spring.redis.port=6379
spring.redis.password=
spring.redis.database=0
spring.redis.pool.max-active=8
spring.redis.pool.max-wait=-1
spring.redis.pool.max-idle=8
spring.redis.pool.min-idle=0

# Admin config
# 关闭访问Rest接口时候的权限验证
management.security.enabled=false
# E版配置方式
management.port=3333
# F版配置方式
management.server.port=3333
```

#### 建立启动类
```java
@SpringBootApplication
@EnableDiscoveryClient
public class ConsoleApplication {
    public static void main(String[] args) {
        new SpringApplicationBuilder(ConsoleApplication.class).run(args);
    }
}
```

#### 更多信息
- 请参考master（Finchley）分支或者Edgware分支下的discovery-springcloud-example-console工程

### 检验成果
- 运行Eureka服务端，可以从discovery-springcloud-example-eureka获取
- 运行控制平台
- 运行您的微服务、Zuul或者Spring Cloud Api Gateway（F版）
- 运行图形化灰度发布桌面程序
  - Clone [https://github.com/Nepxion/Discovery.git](https://github.com/Nepxion/Discovery.git)获取源码（注意master和Edgware分支）
  - 在discovery-console-desktop目录下执行mvn clean install，target目录下将产生discovery-console-desktop-[版本号]-release的目录
  - 进入discovery-console-desktop-[版本号]-release，请修改config/console.properties中的url，该地址指向控制平台的地址
  - 运行“Discovery灰度发布控制台.bat”，启动桌面程序
  - 如果您是Mac系统，有两种方式启动桌面程序
    - 请参考“Discovery灰度发布控制台.bat”，自行编写Discovery灰度发布控制台.sh脚本启动
    - 通过IDE启动discovery-console-desktop\ConsoleLauncher.java启动
- 图形化灰度发布桌面程序的操作视频
  - 灰度发布-版本访问策略
    - 请访问[https://pan.baidu.com/s/1eq_N56VbgSCaTXYQ5aKqiA](https://pan.baidu.com/s/1eq_N56VbgSCaTXYQ5aKqiA)，获取更清晰的视频，注意一定要下载下来看，不要在线看，否则也不清晰
    - 请访问[http://www.iqiyi.com/w_19rzwzovrl.html](http://www.iqiyi.com/w_19rzwzovrl.html)，视频清晰度改成720P，然后最大化播放
  - 灰度发布-版本权重策略
    - 请访问[https://pan.baidu.com/s/1VXPatJ6zrUeos7uTQwM3Kw](https://pan.baidu.com/s/1VXPatJ6zrUeos7uTQwM3Kw)，获取更清晰的视频，注意一定要下载下来看，不要在线看，否则也不清晰
    - 请访问[http://www.iqiyi.com/w_19rzs9pll1.html](http://www.iqiyi.com/w_19rzs9pll1.html)，视频清晰度改成720P，然后最大化播放
  - 灰度发布-全链路策略
    - 请访问[https://pan.baidu.com/s/1XQSKCZUykc6t04xzfrFHsg](https://pan.baidu.com/s/1XQSKCZUykc6t04xzfrFHsg)，获取更清晰的视频，注意一定要下载下来看，不要在线看，否则也不清晰
    - 请访问[http://www.iqiyi.com/w_19s1e0zf95.html(http://www.iqiyi.com/w_19s1e0zf95.html)，视频清晰度改成720P，然后最大化播放
- 规则文件rule.xml或者rule.json的样例，请到相应的discovery-springcloud-example-xxx工程的\src\main\resources下获取，如何了解和使用规则文件，请阅读[主页](https://github.com/Nepxion/Discovery/blob/master/README.md)