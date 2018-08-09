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
  - 请访问[https://pan.baidu.com/s/1Ok4NT87giJfJ8F9uidIO9w](https://pan.baidu.com/s/1Ok4NT87giJfJ8F9uidIO9w)获取
  - 解压后，请修改config/console.properties中的url，该地址指向独立控制台的地址
  - 运行“Discovery灰度发布控制台.bat”，启动桌面程序
  - 如果您是Mac系统，有两种方式启动桌面程序
    - 请参考“Discovery灰度发布控制台.bat”，自行编写Discovery灰度发布控制台.sh脚本启动
    - 下载Nepxion Discovery源码，通过IDE启动discovery-console-desktop\ConsoleLauncher.java启动
- 图形化灰度发布桌面程序的操作视频
  - 灰度发布-版本访问策略
    - 请访问[https://pan.baidu.com/s/1eq_N56VbgSCaTXYQ5aKqiA](https://pan.baidu.com/s/1eq_N56VbgSCaTXYQ5aKqiA)，获取更清晰的视频，注意一定要下载下来看，不要在线看，否则也不清晰
    - 请访问[http://www.iqiyi.com/w_19rzwzovrl.html](http://www.iqiyi.com/w_19rzwzovrl.html)，视频清晰度改成720P，然后最大化播放
  - 灰度发布-版本权重策略
    - 请访问[https://pan.baidu.com/s/1VXPatJ6zrUeos7uTQwM3Kw](https://pan.baidu.com/s/1VXPatJ6zrUeos7uTQwM3Kw)，获取更清晰的视频，注意一定要下载下来看，不要在线看，否则也不清晰
    - 请访问[http://www.iqiyi.com/w_19rzs9pll1.html](http://www.iqiyi.com/w_19rzs9pll1.html)，视频清晰度改成720P，然后最大化播放
- 规则文件rule.xml或者rule.json的样例，请到相应的discovery-springcloud-example-xxx工程的\src\main\resources下获取，如何了解和使用规则文件，请阅读[主页](https://github.com/Nepxion/Discovery/blob/master/README.md)
  - 附录一份全链路灰度发布的样例
```xml
<?xml version="1.0" encoding="UTF-8"?>
<rule>
    <!-- 如果不想开启相关功能，只需要把相关节点删除即可，例如不想要黑名单功能，把blacklist节点删除 -->
    <register>
        <!-- 服务注册的黑/白名单注册过滤，只在服务启动的时候生效。白名单表示只允许指定IP地址前缀注册，黑名单表示不允许指定IP地址前缀注册。每个服务只能同时开启要么白名单，要么黑名单 -->
        <!-- filter-type，可选值blacklist/whitelist，表示白名单或者黑名单 -->
        <!-- service-name，表示服务名 -->
        <!-- filter-value，表示黑/白名单的IP地址列表。IP地址一般用前缀来表示，如果多个用“;”分隔，不允许出现空格 -->
        <!-- 表示下面所有服务，不允许10.10和11.11为前缀的IP地址注册（全局过滤） -->
        <blacklist filter-value="10.10;11.11">
            <!-- 表示下面服务，不允许172.16和10.10和11.11为前缀的IP地址注册 -->
            <service service-name="discovery-springcloud-example-a" filter-value="172.16"/>
        </blacklist>

        <!-- <whitelist filter-value="">
            <service service-name="" filter-value=""/>
        </whitelist>  -->

        <!-- 服务注册的数目限制注册过滤，只在服务启动的时候生效。当某个服务的实例注册达到指定数目时候，更多的实例将无法注册 -->
        <!-- service-name，表示服务名 -->
        <!-- filter-value，表示最大实例注册数 -->
        <!-- 表示下面所有服务，最大实例注册数为10000（全局配置） -->
        <count filter-value="10000">
            <!-- 表示下面服务，最大实例注册数为5000，全局配置值10000将不起作用，以局部配置值为准 -->
            <service service-name="discovery-springcloud-example-a" filter-value="5000"/>
        </count>
    </register>

    <discovery>
        <!-- 服务发现的黑/白名单发现过滤，使用方式跟“服务注册的黑/白名单过滤”一致 -->
        <!-- 表示下面所有服务，不允许10.10和11.11为前缀的IP地址被发现（全局过滤） -->
        <blacklist filter-value="10.10;11.11">
            <!-- 表示下面服务，不允许172.16和10.10和11.11为前缀的IP地址被发现 -->
            <service service-name="discovery-springcloud-example-b" filter-value="172.16"/>
        </blacklist>

        <!-- 服务发现的多版本灰度访问控制 -->
        <!-- service-name，表示服务名 -->
        <!-- version-value，表示可供访问的版本，如果多个用“;”分隔，不允许出现空格 -->
        <!-- 版本策略介绍 -->
        <!-- 1. 标准配置，举例如下 -->
        <!--    <service consumer-service-name="a" provider-service-name="b" consumer-version-value="1.0" provider-version-value="1.0;1.1"/> 表示消费端1.0版本，允许访问提供端1.0和1.1版本 -->
        <!-- 2. 版本值不配置，举例如下 -->
        <!--    <service consumer-service-name="a" provider-service-name="b" provider-version-value="1.0;1.1"/> 表示消费端任何版本，允许访问提供端1.0和1.1版本 -->
        <!--    <service consumer-service-name="a" provider-service-name="b" consumer-version-value="1.0"/> 表示消费端1.0版本，允许访问提供端任何版本 -->
        <!--    <service consumer-service-name="a" provider-service-name="b"/> 表示消费端任何版本，允许访问提供端任何版本 -->
        <!-- 3. 版本值空字符串，举例如下 -->
        <!--    <service consumer-service-name="a" provider-service-name="b" consumer-version-value="" provider-version-value="1.0;1.1"/> 表示消费端任何版本，允许访问提供端1.0和1.1版本 -->
        <!--    <service consumer-service-name="a" provider-service-name="b" consumer-version-value="1.0" provider-version-value=""/> 表示消费端1.0版本，允许访问提供端任何版本 -->
        <!--    <service consumer-service-name="a" provider-service-name="b" consumer-version-value="" provider-version-value=""/> 表示消费端任何版本，允许访问提供端任何版本 -->
        <!-- 4. 版本对应关系未定义，默认消费端任何版本，允许访问提供端任何版本 -->
        <!-- 特殊情况处理，在使用上需要极力避免该情况发生 -->
        <!-- 1. 消费端的application.properties未定义版本号，则该消费端可以访问提供端任何版本 -->
        <!-- 2. 提供端的application.properties未定义版本号，当消费端在xml里不做任何版本配置，才可以访问该提供端 -->
        <version>
            <!-- 表示网关z的1.0，允许访问提供端服务a的1.0版本 -->
            <service consumer-service-name="discovery-springcloud-example-gateway" provider-service-name="discovery-springcloud-example-a" consumer-version-value="1.0" provider-version-value="1.0"/>
            <!-- 表示网关z的1.1，允许访问提供端服务a的1.1版本 -->
            <service consumer-service-name="discovery-springcloud-example-gateway" provider-service-name="discovery-springcloud-example-a" consumer-version-value="1.1" provider-version-value="1.1"/>
            <!-- 表示网关z的1.0，允许访问提供端服务a的1.0版本 -->
            <service consumer-service-name="discovery-springcloud-example-zuul" provider-service-name="discovery-springcloud-example-a" consumer-version-value="1.0" provider-version-value="1.0"/>
            <!-- 表示网关z的1.1，允许访问提供端服务a的1.1版本 -->
            <service consumer-service-name="discovery-springcloud-example-zuul" provider-service-name="discovery-springcloud-example-a" consumer-version-value="1.1" provider-version-value="1.1"/>
            <!-- 表示消费端服务a的1.0，允许访问提供端服务b的1.0版本 -->
            <service consumer-service-name="discovery-springcloud-example-a" provider-service-name="discovery-springcloud-example-b" consumer-version-value="1.0" provider-version-value="1.0"/>
            <!-- 表示消费端服务a的1.1，允许访问提供端服务b的1.1版本 -->
            <service consumer-service-name="discovery-springcloud-example-a" provider-service-name="discovery-springcloud-example-b" consumer-version-value="1.1" provider-version-value="1.1"/>
            <!-- 表示消费端服务b的1.0，允许访问提供端服务c的1.0和1.1版本 -->
            <service consumer-service-name="discovery-springcloud-example-b" provider-service-name="discovery-springcloud-example-c" consumer-version-value="1.0" provider-version-value="1.0;1.1"/>
            <!-- 表示消费端服务b的1.1，允许访问提供端服务c的1.2版本 -->
            <service consumer-service-name="discovery-springcloud-example-b" provider-service-name="discovery-springcloud-example-c" consumer-version-value="1.1" provider-version-value="1.2"/>
        </version>

        <!-- 服务发现的多版本权重灰度访问控制 -->
        <!-- service-name，表示服务名 -->
        <!-- version-value，表示版本对应的权重值，格式为"版本值=权重值"，如果多个用“;”分隔，不允许出现空格 -->
        <!-- 权重策略介绍 -->
        <!-- 1. 标准配置，举例如下 -->
        <!--     <service consumer-service-name="a" provider-service-name="b" provider-weight-value="1.0=90;1.1=10"/> 表示消费端访问提供端的时候，提供端的1.0版本提供90%的权重流量，1.1版本提供10%的权重流量 -->
        <!-- 2. 尽量为线上所有版本都赋予权重值 -->
        <weight>
            <!-- 表示消费端服务b访问提供端服务c的时候，提供端服务c的1.0版本提供90%的权重流量，1.1版本提供10%的权重流量 -->
            <service consumer-service-name="discovery-springcloud-example-b" provider-service-name="discovery-springcloud-example-c" provider-weight-value="1.0=90;1.1=10"/>
        </weight>
    </discovery>
</rule>
```