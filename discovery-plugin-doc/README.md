# 关于我
Neptune（微信：Nepxion），十多年Java EE从业经验，专注于Dubbo和SpringCloud等分布式微服务框架的研究和应用
```xml
Hompage：http://www.imicroservice.com，http://www.nepxion.com
Github：https://github.com/Nepxion
```

# 前言
Spring Cloud如火如荼发展了好几年，作为一款基础的分布式微服务框架，它集成服务注册发现，负载均衡、断路器、API网关等，对于中小型互联网公司来说是一种福音。既然是隶属基础性的范畴，那么对业务应用方面的更细粒度需求考虑的有所欠缺。
笔者尝试从服务注册发现方面去挖掘更多实际性的应用，例如你是否遇到过下面的情况：
```xml
如果你是运维负责人，是否会经常发现，你掌管的测试环境中的服务注册中心，被一些不负责的开发人员把他本地开发环境注册上来，造成测试人员测试失败。你希望可以把本地开发环境注册给屏蔽掉，不让注册
如果你是运维负责人，生产环境的某个微服务集群下的某个实例，暂时出了问题，但又不希望它下线。你希望可以把该实例给屏蔽掉，暂时不让被调用
如果你是业务负责人，鉴于业务服务的快速迭代性，微服务集群下的实例发布不同的版本。你希望根据版本管理策略进行路由，提供给下游微服务区别调用，达到多版本灰度访问控制
```
那么，笔者将提供一套服务注册发现的增强插件贡献给大家 - Nepxion Discovery，支持Eureka、Consul和Zookeeper。现有的Spring Cloud服务可以方便引入该插件，使用者不需要对业务代码做任何修改，只需要做三个非常容易的事情
```xml
引入Plugin Starter依赖到pom.xml
为服务定义一个版本号在application.properties里，相信很多使用者本身就已经这么做了
如果采用了远程配置中心集成的话，那么只需要在那里修改规则（XML），触发推送；如果未集成，可以通过客户端工具（例如Postman）推送修改的规则（XML）
```

# 简介
```xml
实行服务注册层面的控制，基于黑/白名单的IP地址过滤机制禁止对相应的微服务进行注册
实现服务发现层面的控制，基于黑/白名单的IP地址过滤机制禁止对相应的微服务被发现
实行通过对消费端和提供端可访问版本对应关系的配置，进行多版本灰度访问控制
```

# 功能
## 多版本灰度访问控制
笔者设置一个场景，如下图所示
![Alt text](https://github.com/Nepxion/Docs/blob/master/discovery-plugin-doc/Version.jpg)

图中描述的是
```xml
微服务集群部署了3个，分别是A服务集群、B服务集群、C服务集群，分别对应的实例数为1、2、3
微服务集群的调用关系为服务A->服务B->服务C
经过灰度控制，服务B2只允许访问服务C3，不允许访问服务C1和服务C2
```

上面的灰度规则，可能在运行前已经配置好，当然也希望在运行中去动态改变规则，例如通过配置中心改变规则配置，并推送过来。那么我们可以做如下配置，通过XML描述一目了然的看到图中所要表达的规则策略
```xml
<?xml version="1.0" encoding="UTF-8"?>
<rule>
    <discovery>
        <version>
            <!-- 表示消费端服务a的1.0，允许访问提供端服务b的1.0和1.1版本 -->
            <service consumer-service-name="a" provider-service-name="b" consumer-version-value="1.0" provider-version-value="1.0;1.1"/>
            <!-- 表示消费端服务b的1.0，允许访问提供端服务c的1.0和1.1版本 -->
            <service consumer-service-name="b" provider-service-name="c" consumer-version-value="1.0" provider-version-value="1.0;1.1"/>
            <!-- 表示消费端服务b的1.1，允许访问提供端服务c的1.2版本 -->
            <service consumer-service-name="b" provider-service-name="c" consumer-version-value="1.1" provider-version-value="1.2"/>
        </version>
    </discovery>
</rule>
```

## 黑/白名单的IP地址注册的过滤
服务启动的时候，禁止指定的IP地址注册到服务注册发现中心。支持黑/白名单，白名单表示只允许指定IP地址前缀注册，黑名单表示不允许指定IP地址前缀注册
```xml
全局过滤，指注册到服务注册发现中心的所有服务，只有IP地址包含在全局过滤字段的前缀中，都允许注册（对于白名单而言），或者不允许注册（对于黑名单而言）
局部过滤，指专门针对某个服务而言，那么真正的过滤条件是全局过滤+局部过滤结合在一起
```
```xml
<?xml version="1.0" encoding="UTF-8"?>
<rule>
    <register>
        <!-- 表示下面所有服务，不允许10.10和11.11为前缀的IP地址注册（全局过滤） -->
        <blacklist filter-value="10.10;11.11">
            <!-- 表示下面服务，不允许172.16和10.10和11.11为前缀的IP地址注册 -->
            <service service-name="discovery-springcloud-example-a" filter-value="172.16"/>
        </blacklist>
    </register>
</rule>
```

## 黑/白名单的IP地址发现的过滤
服务启动的时候，禁止指定的IP地址被服务发现。它使用的方式和“黑/白名单的IP地址注册的过滤”一致
```xml
<?xml version="1.0" encoding="UTF-8"?>
<rule>
    <discovery>
        <!-- 表示下面所有服务，不允许10.10和11.11为前缀的IP地址被发现（全局过滤） -->
        <blacklist filter-value="10.10;11.11">
            <!-- 表示下面服务，不允许172.16和10.10和11.11为前缀的IP地址被发现 -->
            <service service-name="discovery-springcloud-example-b" filter-value="172.16"/>
        </blacklist>
    </discovery>
</rule>
```

# 依赖
选择相应的插件引入
```xml
<dependency>
    <groupId>com.nepxion</groupId>
    <artifactId>discovery-plugin-starter-eureka</artifactId>
    <version>${discovery.plugin.version}</version>
</dependency>

<dependency>
    <groupId>com.nepxion</groupId>
    <artifactId>discovery-plugin-starter-consul</artifactId>
    <version>${discovery.plugin.version}</version>
</dependency>

<dependency>
    <groupId>com.nepxion</groupId>
    <artifactId>discovery-plugin-starter-zookeeper</artifactId>
    <version>${discovery.plugin.version}</version>
</dependency>
```

# 工程

| 工程名 | 描述 |
| --- | --- | 
| discovery-plugin-framework | 核心框架 |
| discovery-plugin-framework-consul | 核心框架的Consul扩展 |
| discovery-plugin-framework-eureka | 核心框架的Eureka扩展 |
| discovery-plugin-framework-zookeeper | 核心框架的Zookeeper扩展 |
| discovery-plugin-config-center | 配置中心实现 |
| discovery-plugin-router-center | 路由中心实现 |
| discovery-plugin-admin-center | 管理中心实现 |
| discovery-plugin-starter-consul | Consul Starter |
| discovery-plugin-starter-eureka | Eureka Starter |
| discovery-plugin-starter-zookeeper | Zookeeper Starter |

# 核心代码
对DiscoveryClient做装饰模式处理
```java
public class DiscoveryClientDecorator implements DiscoveryClient {
    private DiscoveryClient discoveryClient;
    private ConfigurableApplicationContext applicationContext;
    private ConfigurableEnvironment environment;

    public DiscoveryClientDecorator(DiscoveryClient discoveryClient, ConfigurableApplicationContext applicationContext) {
        this.discoveryClient = discoveryClient;
        this.applicationContext = applicationContext;
        this.environment = applicationContext.getEnvironment();
    }

    @Override
    public List<ServiceInstance> getInstances(String serviceId) {
        List<ServiceInstance> instances = discoveryClient.getInstances(serviceId);

        // 做获取服务实例列表获取的拦截和过滤

        return instances;
    }

    @Override
    public List<String> getServices() {
        List<String> services = discoveryClient.getServices();

        // 做服务列表获取的拦截和过滤

        return services;
    }

    @Deprecated
    @Override
    public ServiceInstance getLocalServiceInstance() {
        return discoveryClient.getLocalServiceInstance();
    }

    @Override
    public String description() {
        return discoveryClient.description();
    }

    public ConfigurableEnvironment getEnvironment() {
        return environment;
    }
}
```

对ServiceRegistry做装饰模式
```java
public class ConsulServiceRegistryDecorator extends ConsulServiceRegistry {
    private ConsulServiceRegistry serviceRegistry;
    private ConfigurableApplicationContext applicationContext;
    private ConfigurableEnvironment environment;

    public ConsulServiceRegistryDecorator(ConsulServiceRegistry serviceRegistry, ConfigurableApplicationContext applicationContext) {
        super(null, null, null, null);

        this.serviceRegistry = serviceRegistry;
        this.applicationContext = applicationContext;
        this.environment = applicationContext.getEnvironment();
    }

    @Override
    public void register(ConsulRegistration registration) {
        // 做服务注册的拦截和处理

        serviceRegistry.register(registration);
    }

    @Override
    public void deregister(ConsulRegistration registration) {
        // 做服务取消注册的拦截和处理

        serviceRegistry.deregister(registration);
    }

    @Override
    public void setStatus(ConsulRegistration registration, String status) {
        // 做服务状态设置的拦截和处理

        serviceRegistry.setStatus(registration, status);
    }

    @Override
    public Object getStatus(ConsulRegistration registration) {
        return serviceRegistry.getStatus(registration);
    }

    @Override
    public void close() {
        // 做服务注册关闭的拦截和处理

        serviceRegistry.close();
    }

    public ConfigurableEnvironment getEnvironment() {
        return environment;
    }
}
```

Spring容器启动时（在上下文初始化前），用装饰器来置换对应的Bean
```java
public class PluginApplicationContextInitializer implements ApplicationContextInitializer<ConfigurableApplicationContext> {
    @Override
    public void initialize(ConfigurableApplicationContext applicationContext) {
        applicationContext.getBeanFactory().addBeanPostProcessor(new InstantiationAwareBeanPostProcessorAdapter() {
            @Override
            public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
                if (bean instanceof DiscoveryClient) {
                    DiscoveryClient discoveryClient = (DiscoveryClient) bean;

                    return new DiscoveryClientDecorator(discoveryClient, applicationContext);
                } else if (...) {
                    ...
                }
            }
        });
    }
}
```

更多内容请参考
```xml
https://github.com/Nepxion/Discovery
```