# Nepxion Discovery
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg?label=license)](https://github.com/Nepxion/Discovery/blob/master/LICENSE)
[![Maven Central](https://img.shields.io/maven-central/v/com.nepxion/discovery.svg?label=maven%20central)](http://search.maven.org/#search%7Cga%7C1%7Cg%3A%22com.nepxion%22%20AND%20discovery)
[![Javadocs](http://www.javadoc.io/badge/com.nepxion/discovery.svg)](http://www.javadoc.io/doc/com.nepxion/discovery)
[![Build Status](https://travis-ci.org/Nepxion/Discovery.svg?branch=master)](https://travis-ci.org/Nepxion/Discovery)

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
    <artifactId>discovery-plugin-config-center-extension-redis</artifactId>
</dependency>
```

#### 添加配置
```xml
# Eureka config
eureka.instance.metadataMap.version=1.0
eureka.instance.metadataMap.group=example-service-group

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

### 搭建独立控制台
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
- 引入控制台依赖
```xml
<dependency>
    <groupId>com.nepxion</groupId>
    <artifactId>discovery-console-starter</artifactId>
</dependency>
```
- 引入Redis远程配置中心扩展依赖
```xml
<dependency>
    <groupId>com.nepxion</groupId>
    <artifactId>discovery-console-extension-redis</artifactId>
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
- 运行独立控制台
- 运行您的微服务、Zuul或者Spring Cloud Api Gateway（F版）
- 运行图形化灰度发布桌面程序
  - 请访问[http://www.iqiyi.com/w_19rzwzovrl.html](http://www.iqiyi.com/w_19rzwzovrl.html)，视频清晰度改成720P，然后最大化播放
  - 请访问[https://pan.baidu.com/s/1eq_N56VbgSCaTXYQ5aKqiA](https://pan.baidu.com/s/1eq_N56VbgSCaTXYQ5aKqiA)，获取更清晰的视频，注意一定要下载下来看，不要在线看，否则也不清晰

