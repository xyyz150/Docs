# Nepxion Discovery
[![Total lines](https://tokei.rs/b1/github/Nepxion/Discovery?category=lines)](https://github.com/Nepxion/Discovery)
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg?label=license)](https://github.com/Nepxion/Discovery/blob/master/LICENSE)
[![Maven Central](https://img.shields.io/maven-central/v/com.nepxion/discovery.svg?label=maven%20central)](http://search.maven.org/#search%7Cga%7C1%7Cg%3A%22com.nepxion%22%20AND%20discovery)
[![Javadocs](http://www.javadoc.io/badge/com.nepxion/discovery-plugin-framework.svg)](http://www.javadoc.io/doc/com.nepxion/discovery-plugin-framework)
[![Build Status](https://travis-ci.org/Nepxion/Discovery.svg?branch=master)](https://travis-ci.org/Nepxion/Discovery)
[![Codacy Badge](https://api.codacy.com/project/badge/Grade/8e39a24e1be740c58b83fb81763ba317)](https://www.codacy.com/project/HaojunRen/Discovery/dashboard?utm_source=github.com&amp;utm_medium=referral&amp;utm_content=Nepxion/Discovery&amp;utm_campaign=Badge_Grade_Dashboard)

## 入门教程
- 本教程以Eureka（Consul、Zookeeper和Nacos同理，配置不同而已）为服务注册发现，Apollo和Nacos（Redis同理）为远程配置中心为例展开讲解，旨在帮您快速把本框架集成到您的业务系统中
- 本教程略掉其它非必须或者锦上添花的功能，如您需要，请参考[主页](https://github.com/Nepxion/Discovery/blob/master/README.md)和[示例演示](https://github.com/Nepxion/Docs/blob/master/discovery-doc/README_EXAMPLE.md)
- 本教程略掉Spring Cloud基础部分，例如如何搭建微服务、Eureka、Zuul或者Spring Cloud Gateway等不在本文介绍范围内

## 目录
- [服务快速集成](#服务快速集成)
  - [服务-引入依赖](#服务-引入依赖)
  - [服务-添加配置](#服务-添加配置)
  - [服务-更多信息](#服务-更多信息) 
- [搭建控制平台](#搭建控制平台)
  - [控制平台-引入依赖](#控制平台-引入依赖)
  - [控制平台-添加配置](#控制平台-添加配置)
  - [控制平台-更多信息](#控制平台-更多信息)
- [搭建远程配置中心](#搭建远程配置中心)
  - [搭建Apollo服务器](#搭建Apollo服务器)
  - [搭建Nacos服务器](#搭建Nacos服务器)  
- [运行服务](#运行服务) 
- [界面操作](#界面操作)
  - [运行图形化灰度发布桌面程序](#运行图形化灰度发布桌面程序)
  - [运行图形化灰度发布Web程序](#运行图形化灰度发布Web程序)
  - [运行Apollo配置界面](#运行Apollo配置界面)
  - [运行Nacos配置界面](#运行Nacos配置界面)
  - [运行Swagger或者Postman方式](#运行Swagger或者Postman方式)
  - [更多信息](#更多信息)

## 服务快速集成
集成到微服务、Zuul或者Spring Cloud Gateway

### 服务-引入依赖
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
- 引入Apollo或者Nacos远程配置中心扩展依赖，必须选择一个引入
```xml
<dependency>
    <groupId>com.nepxion</groupId>
    <artifactId>discovery-plugin-config-center-starter-apollo</artifactId>	
    <artifactId>discovery-plugin-config-center-starter-nacos</artifactId>
</dependency>
```
- :exclamation:如果需要，引入用户自定义和编程灰度路由扩展依赖（三个依赖分别是服务端，网关Zuul端，网关Spring Cloud Gateway端，对应选择一个引入）
```xml
<dependency>
    <groupId>com.nepxion</groupId>
    <artifactId>discovery-plugin-strategy-starter-service</artifactId>
    <artifactId>discovery-plugin-strategy-starter-zuul</artifactId>
    <artifactId>discovery-plugin-strategy-starter-gatewway</artifactId>
</dependency>
```

### 服务-添加配置
```xml
# Eureka config for discovery
eureka.instance.metadataMap.version=1.0
eureka.instance.metadataMap.group=example-service-group
eureka.instance.metadataMap.region=dev

# Apollo config for rule
app.id=discovery
apollo.meta=http://localhost:8080
# apollo.plugin.namespace=application

# Nacos config for rule
nacos.server-addr=localhost:8080
# nacos.plugin.namespace=application

# Admin config
# 关闭访问Rest接口时候的权限验证
management.security.enabled=false
# E版配置方式
management.port=5100
# F版或以上配置方式
management.server.port=5100
```

:star:如果只想要“用户自定义和编程灰度路由”功能，而不想要灰度发布功能
- “用户自定义和编程灰度路由”是不需要接入远程配置中心的，所以建议去除远程配置中心包的引入
```xml
<dependency>
    <groupId>com.nepxion</groupId>
    <artifactId>discovery-plugin-config-center-starter-xxx</artifactId>
</dependency>
```
- “用户自定义和编程灰度路由”是不会对服务注册发现等逻辑产生影响，所以建议下面两项配置改为false
```xml
# 开启和关闭服务注册层面的控制。一旦关闭，服务注册的黑/白名单过滤功能将失效，最大注册数的限制过滤功能将失效。缺失则默认为true
spring.application.register.control.enabled=false
# 开启和关闭服务发现层面的控制。一旦关闭，服务多版本调用的控制功能将失效，动态屏蔽指定IP地址的服务实例被发现的功能将失效。缺失则默认为true
spring.application.discovery.control.enabled=false
```

### 服务-更多信息
- 请参考discovery-springcloud-example-service、discovery-springcloud-example-zuul、discovery-springcloud-example-gateway三个工程

## 搭建控制平台
:warning:如果您通过“图形化灰度发布桌面程序”进行规则操作，那么必须搭建控制平台；如果您依靠第三方配置平台界面（例如Apollo或者Nacos）进行规则操作，那么不需要搭建控制平台
### 控制平台-引入依赖
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
- 引入Apollo（敬请期待）或者Nacos远程配置中心扩展依赖，必须选择一个引入
```xml
<dependency>
    <groupId>com.nepxion</groupId>
    <artifactId>discovery-console-starter-nacos</artifactId>
</dependency>
```
- 引入Eureka Client依赖

### 控制平台-添加配置
```xml
# Nacos config for rule
nacos.server-addr=localhost:8080
# nacos.plugin.namespace=application

# Admin config
# 关闭访问Rest接口时候的权限验证
management.security.enabled=false
# E版配置方式
management.port=3333
# F版或以上配置方式
management.server.port=3333
```

### 控制平台-更多信息
- 请参考discovery-springcloud-example-console工程

## 搭建远程配置中心
### 搭建Apollo服务器
- Apollo服务器版本，推荐用最新版本，从[https://github.com/ctripcorp/apollo/releases](https://github.com/ctripcorp/apollo/releases)获取
- 参考Apollo主页，搭建环境

### 搭建Nacos服务器
- Nacos服务器版本，推荐用最新版本，从[https://github.com/alibaba/nacos/releases](https://github.com/alibaba/nacos/releases)获取
- Windows下运行bin/startup.cmd，Linux下运行bin/startup.sh即可

## 运行服务
- 运行Eureka服务端，可以从discovery-springcloud-example-eureka获取
- 运行控制平台（如果需要）
- 运行您的微服务、Zuul或者Spring Cloud Gateway

## 界面操作
如下五种方式可以达到同等效果的灰度发布效果，请任选一种
### 运行图形化灰度发布桌面程序
- 桌面程序对Windows和Mac操作系统都支持，但在Mac操作系统中界面显示有点瑕疵，但不影响功能使用
- Clone [https://github.com/Nepxion/Discovery.git](https://github.com/Nepxion/Discovery.git)获取源码（注意master和Edgware分支）
- 通过IDE启动
  - 运行discovery-console-desktop\ConsoleLauncher.java启动
- 通过脚本启动
  - 在discovery-console-desktop目录下执行mvn clean install，target目录下将产生discovery-console-desktop-[版本号]-release的目录
  - 进入discovery-console-desktop-[版本号]-release，请修改config/console.properties中的url，该地址指向控制平台的地址
  - 运行“Discovery灰度发布控制台.bat”，启动桌面程序
  - 如果您是操作系统，请参考“Discovery灰度发布控制台.bat”，自行编写“Discovery灰度发布控制台.sh”脚本，启动桌面程序
- 操作界面
  - 登录认证，用户名和密码为admin/admin或者nepxion/nepxion。顺便说一下，控制台支持简单的认证，用户名和密码配置在discovery-springcloud-example-console\bootstrap.properties中，您可以自己扩展AuthenticationResource并注入，实现更专业的认证功能
![Alt text](https://github.com/Nepxion/Docs/blob/master/discovery-doc/Console0.jpg)
  - 点击“显示服务拓扑”按钮，弹出“服务集群组过滤”对话框，列表是以服务所在的集群组列表（例如：eureka.instance.metadataMap.group=example-service-group），选择若干个并点击“确定”按钮，如果使用者想获取全部的服务集群（可能会耗性能），则直接点击“取消”按钮
![Alt text](https://github.com/Nepxion/Docs/blob/master/discovery-doc/Console4.jpg)
  - 从服务注册发现中心获取服务拓扑
![Alt text](https://github.com/Nepxion/Docs/blob/master/discovery-doc/Console5.jpg)
  - 执行灰度路由，选择一个服务，右键菜单“执行灰度路由”
![Alt text](https://github.com/Nepxion/Docs/blob/master/discovery-doc/Console6.jpg)
  - 通过“服务列表”切换，或者点击增加和删除服务按钮，确定灰度路由路径，点击“执行路由”
![Alt text](https://github.com/Nepxion/Docs/blob/master/discovery-doc/Console7.jpg)
![Alt text](https://github.com/Nepxion/Docs/blob/master/discovery-doc/Console2.jpg)
  - 推送模式设置，“异步推送”和“同步推送”，前者是推送完后立刻返回，后者是推送完后等待推送结果（包括规则XML解析的异常等都能在界面上反映出来）；“规则推送到远程配置中心”和“规则推送到服务或者服务集群”，前者是推送到配置中心（持久化），后者是推送到一个或者多个服务机器的内存（非持久化，重启后丢失）
![Alt text](https://github.com/Nepxion/Docs/blob/master/discovery-doc/Console8.jpg)
  - 执行灰度发布，选择一个服务或者服务组，右键菜单“执行灰度发布”，前者是通过单个服务实例执行灰度发布，后者是通过一组服务实例执行灰度发布
![Alt text](https://github.com/Nepxion/Docs/blob/master/discovery-doc/Console9.jpg)
  - 灰度发布，包括“更改版本”和“更改规则”，前者通过更改版本号去适配灰度规则中的版本匹配关系，后者直接修改规则。“更改版本”是推送到一个或者多个服务机器的内存（非持久化，重启后丢失），“更改规则”是根据不同的推送模式，两种方式都支持
![Alt text](https://github.com/Nepxion/Docs/blob/master/discovery-doc/Console10.jpg)
  - 全链路灰度发布，所有在同一个集群组（例如：eureka.instance.metadataMap.group=example-service-group）里的服务统一做灰度发布，即一个规则配置搞定所有服务的灰度发布。点击“全链路灰度发布”按钮，弹出“全链路灰度发布”对话框
![Alt text](https://github.com/Nepxion/Docs/blob/master/discovery-doc/Console11.jpg)
![Alt text](https://github.com/Nepxion/Docs/blob/master/discovery-doc/Console12.jpg)
  - 刷新灰度状态，选择一个服务或者服务组，右键菜单“刷新灰度状态”，查看某个服务或者服务组是否正在做灰度发布
![Alt text](https://github.com/Nepxion/Docs/blob/master/discovery-doc/Console13.jpg)
- 操作视频
  - 灰度发布-版本访问策略
    - 请访问[https://pan.baidu.com/s/1eq_N56VbgSCaTXYQ5aKqiA](https://pan.baidu.com/s/1eq_N56VbgSCaTXYQ5aKqiA)，获取更清晰的视频，注意一定要下载下来看，不要在线看，否则也不清晰
    - 请访问[http://www.iqiyi.com/w_19rzwzovrl.html](http://www.iqiyi.com/w_19rzwzovrl.html)，视频清晰度改成720P，然后最大化播放
  - 灰度发布-版本权重策略
    - 请访问[https://pan.baidu.com/s/1VXPatJ6zrUeos7uTQwM3Kw](https://pan.baidu.com/s/1VXPatJ6zrUeos7uTQwM3Kw)，获取更清晰的视频，注意一定要下载下来看，不要在线看，否则也不清晰
    - 请访问[http://www.iqiyi.com/w_19rzs9pll1.html](http://www.iqiyi.com/w_19rzs9pll1.html)，视频清晰度改成720P，然后最大化播放
  - 灰度发布-全链路策略
    - 请访问[https://pan.baidu.com/s/1XQSKCZUykc6t04xzfrFHsg](https://pan.baidu.com/s/1XQSKCZUykc6t04xzfrFHsg)，获取更清晰的视频，注意一定要下载下来看，不要在线看，否则也不清晰
    - 请访问[http://www.iqiyi.com/w_19s1e0zf95.html](http://www.iqiyi.com/w_19s1e0zf95.html)，视频清晰度改成720P，然后最大化播放

### 运行图形化灰度发布Web程序
- 参考[https://github.com/Nepxion/DiscoveryUI](https://github.com/Nepxion/DiscoveryUI)
- 操作过程跟“运行图形化灰度发布桌面程序”类似

### 运行Apollo配置界面
![Alt text](https://github.com/Nepxion/Docs/blob/master/discovery-doc/Apollo1.jpg)
- 参考Apollo官方文档[https://github.com/ctripcorp/apollo](https://github.com/ctripcorp/apollo)相关文档，搭建Apollo环境，以及熟悉相关的基本操作
- 根据上图，做如下步骤操作
  - 设置页面中AppId和配置文件里面app.id一致
  - 设置页面中Namespace和配置文件里面apollo.plugin.namespace一致，如果配置文件里不设置，那么页面默认采用内置的“application”
  - 在页面中添加配置
    - 局部配置方式：一个服务集群（eureka.instance.metadataMap.group和spring.application.name都相同的服务）对应一个配置文件，通过group+serviceId方式添加，Key为“group-serviceId”，Value为Xml或者Json格式的规则内容。group取值于配置文件里的eureka.instance.metadataMap.group配置项，serviceId取值于spring.application.name配置项目
    - 全局配置方式：一组服务集群（eureka.instance.metadataMap.group相同，但spring.application.name可以不相同的服务）对应一个配置文件，通过group方式添加，Key为“group-group”，Value为Xml或者Json格式的规则内容。group取值于配置文件里的eureka.instance.metadataMap.group配置项
    - 强烈建议局部配置方式和全局配置方式不要混用，否则连使用者自己都无法搞清楚到底是哪种配置方式在起作用
  - 其他更多参数，例如evn, cluster等，请自行参考Apollo官方文档，保持一致

### 运行Nacos配置界面
![Alt text](https://github.com/Nepxion/Docs/blob/master/discovery-doc/Nacos2.jpg)
- 参考Nacos官方文档[https://github.com/alibaba/nacos](https://github.com/alibaba/nacos)相关文档，搭建Nacos环境，以及熟悉相关的基本操作
- 添加配置步骤跟Apollo配置界面中的“在页面中添加配置”操作项相似

### 运行Swagger或者Postman方式
- Swagger界面
![Alt text](https://github.com/Nepxion/Docs/blob/master/discovery-doc/Swagger1.jpg)
![Alt text](https://github.com/Nepxion/Docs/blob/master/discovery-doc/Swagger2.jpg)
- Swagger界面
  - [Postman操作集合](https://github.com/Nepxion/Docs/blob/master/discovery-doc/Nepxion.postman_collection.json)，请下载到本地，导入到Postman中执行相关操作

### 更多信息
- 规则文件rule.xml或者rule.json的样例，请到相应的discovery-springcloud-example-xxx工程的\src\main\resources下获取，如何了解和使用规则文件，请阅读[主页](https://github.com/Nepxion/Discovery/blob/master/README.md)