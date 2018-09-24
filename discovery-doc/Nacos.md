# Nepxion Discovery - 基于Nacos实现Spring Cloud灰度发布和路由

## 前言
Nepxion Discovery是一款对Spring Cloud Discovery服务注册发现、Ribbon负载均衡、Feign和RestTemplate调用的增强中间件，其功能包括灰度发布（包括切换发布和平滑发布）、服务隔离、服务路由、服务权重、黑/白名单的IP地址过滤、限制注册、限制发现等，支持Eureka、Consul、Zookeeper和阿里巴巴的Nacos为服务注册发现中间件，支持阿里巴巴的Nacos、携程的Apollo和Redis为远程配置中心，支持Spring Cloud Api Gateway（Finchley版）、Zuul网关和微服务的灰度发布，支持多数据源的数据库灰度发布等客户特色化灰度发布，支持用户自定义和编程灰度路由策略（包括RPC和REST两种调用方式），兼容Spring Cloud Edgware版和Finchley版（不支持Dalston版，因为它的生命周期将在2018年12月结束，如果您无法回避使用Dalston版，请自行修改源码或者联系我）。现有的Spring Cloud微服务很方便引入该中间件，代码零侵入

更多内容请访问 [https://github.com/Nepxion/Discovery](https://github.com/Nepxion/Discovery)

## 主题
那么如何基于Nacos实现Spring Cloud灰度发布和路由呢？主要分为如下三部分
- 整合Nacos服务注册发现机制，实现Spring Cloud的灰度发布和路由
- 利用Nacos配置中心，实现Spring Cloud的灰度发布和路由规则的推送、订阅
- 利用Nacos控制台，实现Spring Cloud的灰度发布和路由规则的配置

无论是原生的Nacos Client，还是Nacos Spring、Nacos SpringBoot，或者Nacos SpringCloud都具有非常好的用户易用性和扩展性，尤其是Spring系列，紧紧遵循Spring生态的规范，所以大家可以看到整合起来代码量相对较少、也比较简单。本文考虑到篇幅，只介绍涉及到整合Nacos的部分，涉及到具体灰度发布和路由的逻辑则不在讲述范围内，请自行访问Github相关代码和文档进行研究。本文涉及的代码跟Github相关代码有较大出入，有些甚至是伪代码，其目的是避免繁琐代码，力求简单说明概念和问题

## 整合Nacos服务注册发现机制，实现Spring Cloud的灰度发布和路由
本模块是基于spring-cloud-alibaba-nacos-discovery（见 [https://github.com/spring-cloud-incubator/spring-cloud-alibaba](https://github.com/spring-cloud-incubator/spring-cloud-alibaba)）标准化的服务注册发现机制而实现的，所以我们可以完全可以象扩展Eureka、Consul或者Zookeeper Discovery组件一样，去扩展Nacos组件做灰度发布和路由，下文主要讲述几个扩展步骤，对所有的服务注册发现组件都是大体一致，细节有所区别

### 装饰类
服务注册层面的装饰类 - NacosServiceRegistryDecorator继承和装饰NacosServiceRegistry，实现通过RegisterListenerExecutor注册监听执行器对它的核心方法进行拦截，从而实现在注册层面的“黑/白名单的IP地址注册的过滤规则”、“最大注册数的限制的过滤规则”等功能
```java
public class NacosServiceRegistryDecorator extends NacosServiceRegistry {
    private NacosServiceRegistry serviceRegistry;
    private ConfigurableApplicationContext applicationContext;
    private ConfigurableEnvironment environment;

    public NacosServiceRegistryDecorator(NacosServiceRegistry serviceRegistry, ConfigurableApplicationContext applicationContext) {
        this.serviceRegistry = serviceRegistry;
        this.applicationContext = applicationContext;
        this.environment = applicationContext.getEnvironment();
    }

    @Override
    public void register(NacosRegistration registration) {
        // 注册之前，registerListenerExecutor.onRegister方法里，执行如下操作：
        // 1. 触发黑/白名单的IP地址注册的过滤规则。如果不符合，中断注册抛出异常
        // 2. 触发最大注册数的限制的过滤规则。如果不符合，中断注册抛出异常
        // 上述规则，可以同时启用，也可以单独存在
        RegisterListenerExecutor registerListenerExecutor = applicationContext.getBean(RegisterListenerExecutor.class);
        registerListenerExecutor.onRegister(registration);

        // 执行NacosServiceRegistry的逻辑
        serviceRegistry.register(registration);
    }

    @Override
    public void deregister(NacosRegistration registration) {
        // 反注册之前，执行registerListenerExecutor.onDeregister方法里，可执行相关逻辑，目前是空实现，可以让用户自行扩展。下同
        RegisterListenerExecutor registerListenerExecutor = applicationContext.getBean(RegisterListenerExecutor.class);
        registerListenerExecutor.onDeregister(registration);

        // 执行NacosServiceRegistry的逻辑。下同
        serviceRegistry.deregister(registration);
    }

    @Override
    public void setStatus(NacosRegistration registration, String status) {
        RegisterListenerExecutor registerListenerExecutor = applicationContext.getBean(RegisterListenerExecutor.class);
        registerListenerExecutor.onSetStatus(registration, status);

        serviceRegistry.setStatus(registration, status);
    }

    @Override
    public <T> T getStatus(NacosRegistration registration) {
        return serviceRegistry.getStatus(registration);
    }

    @Override
    public void close() {
        RegisterListenerExecutor registerListenerExecutor = applicationContext.getBean(RegisterListenerExecutor.class);
        registerListenerExecutor.onClose();

        serviceRegistry.close();
    }

    public ConfigurableEnvironment getEnvironment() {
        return environment;
    }
}
```

服务发现层面的装饰类 - NacosServerListDecorator继承NacosServerList，实现通过LoadBalanceListenerExecutor负载均衡监听执行器对它的核心方法进行拦截和过滤，从而实现在负载均衡层面的“版本访问的灰度路由规则”、“版本权重的灰度路由规则”、“区域权重的灰度路由规则”等功能
```java
public class NacosServerListDecorator extends NacosServerList {
    private ConfigurableEnvironment environment;

    private LoadBalanceListenerExecutor loadBalanceListenerExecutor;

    public NacosServerListDecorator() {
        super();
    }

    public NacosServerListDecorator(String serviceId) {
        super(serviceId);
    }

    @Override
    public List<NacosServer> getInitialListOfServers() {
        // 获取初始化服务列表的时候，做过滤和拦截
        List<NacosServer> servers = super.getInitialListOfServers();

        filter(servers);

        return servers;
    }

    @Override
    public List<NacosServer> getUpdatedListOfServers() {
        // 定时更新服务列表的时候，做过滤和拦截
        List<NacosServer> servers = super.getUpdatedListOfServers();

        filter(servers);

        return servers;
    }

    private void filter(List<NacosServer> servers) {
        // loadBalanceListenerExecutor.onGetServers方法将触发
        // 1. 触发版本访问的灰度路由的过滤规则，即通过微服务的版本号比对，去掉列表中不符合要求的实例
        // 2. 触发版本权重的灰度路由的过滤规则，即通过微服务版本号对应的权重流量比对，去掉列表中不符合要求的实例
        // 3. 触发区域权重的灰度路由的过滤规则，即通过微服务所在的区域对应的权重流量比对，去掉列表中不符合要求的实例
        // 上述规则，可以同时启用，也可以单独存在		
        String serviceId = getServiceId();
        loadBalanceListenerExecutor.onGetServers(serviceId, servers);
    }

    public void setEnvironment(ConfigurableEnvironment environment) {
        this.environment = environment;
    }

    public void setLoadBalanceListenerExecutor(LoadBalanceListenerExecutor loadBalanceListenerExecutor) {
        this.loadBalanceListenerExecutor = loadBalanceListenerExecutor;
    }
}
```

### 适配类
由于在不同的服务注册发现组件（Eureka、Consul、Zookeeper、Nacos）中，获得Metadata的方法是实现在Server的子类上，所以我们要做一层适配，做一次强制转换
Metadata的数据在灰度发布和路由中起着至关重要的作用，比如灰度发布中涉及到的版本（Version）、组（Group）和区域（Region）都是通过Metadata方式提供，例如
```xml
spring.cloud.nacos.discovery.metadata.version=1.0
spring.cloud.nacos.discovery.metadata.group=example-service-group
spring.cloud.nacos.discovery.metadata.region=dev
```

```java
public class NacosAdapter extends AbstractPluginAdapter {
    @Override
    public Map<String, String> getServerMetadata(Server server) {
        if (server instanceof NacosServer) {
            NacosServer nacosServer = (NacosServer) server;

            return nacosServer.getMetadata();
        }

        throw new DiscoveryException("Server instance isn't the type of NacosServer");
    }
}
```

### 初始化类
ApplicationContextInitializer是在Spring容器初始化的时候执行，可以对Spring容器中的Bean进行拦截和替换。对NacosServiceRegistry对象进行拦截，由NacosServiceRegistryDecorator去代理；对NacosDiscoveryProperties对象进行拦截，并把相关的Metadata数据植入，并注册到Nacos服务器上
```java
public class NacosApplicationContextInitializer extends PluginApplicationContextInitializer {
    @Override
    protected Object afterInitialization(ConfigurableApplicationContext applicationContext, Object bean, String beanName) throws BeansException {
        if (bean instanceof NacosServiceRegistry) {
            NacosServiceRegistry nacosServiceRegistry = (NacosServiceRegistry) bean;

            return new NacosServiceRegistryDecorator(nacosServiceRegistry, applicationContext);
        } else if (bean instanceof NacosDiscoveryProperties) {
            ConfigurableEnvironment environment = applicationContext.getEnvironment();

            NacosDiscoveryProperties nacosDiscoveryProperties = (NacosDiscoveryProperties) bean;

            Map<String, String> metadata = nacosDiscoveryProperties.getMetadata();
            metadata.put("xxx", "yyy");
            ... 

            return bean;
        } else {
            return bean;
        }
    }
}
```

### 配置类
NacosRibbonClientConfiguration里的ribbonServerList方法的返回类型，用NacosServerListDecorator装饰类替换NacosServerList，放入灰度发布的负载均衡拦截执行器
```java
@Configuration
@AutoConfigureAfter(NacosRibbonClientConfiguration.class)
public class NacosLoadBalanceConfiguration {

    @Autowired
    private ConfigurableEnvironment environment;

    @Autowired
    private LoadBalanceListenerExecutor loadBalanceListenerExecutor;

    @Bean
    public ServerList<?> ribbonServerList(IClientConfig config) {
        NacosServerListDecorator serverList = new NacosServerListDecorator();
        serverList.initWithNiwsConfig(config);
        serverList.setEnvironment(environment);
        serverList.setLoadBalanceListenerExecutor(loadBalanceListenerExecutor);

        return serverList;
    }
}
```
指定RibbonClients注解的配置类列表，包含PluginLoadBalanceConfiguration和NacosLoadBalanceConfiguration。PluginLoadBalanceConfiguration封装了通用灰度发布逻辑（这里不展开了）
```java
@Configuration
@RibbonClients(defaultConfiguration = { PluginLoadBalanceConfiguration.class, NacosLoadBalanceConfiguration.class })
public class NacosAutoConfiguration {
    @Bean
    public PluginAdapter pluginAdapter() {
        return new NacosAdapter();
    }
}
```
spring.factories，在starter的src/main/resources/META-INF/spring.factories中加入自动配置
```xml
org.springframework.context.ApplicationContextInitializer=\
com.nepxion.discovery.plugin.framework.context.NacosApplicationContextInitializer

org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.nepxion.discovery.plugin.framework.configuration.PluginAutoConfiguration,\
com.nepxion.discovery.plugin.framework.configuration.NacosAutoConfiguration,\
com.nepxion.discovery.plugin.configcenter.configuration.ConfigAutoConfiguration,\
com.nepxion.discovery.plugin.admincenter.configuration.AdminAutoConfiguration
```

## 利用Nacos配置中心，实现Spring Cloud的灰度发布和路由规则的推送、订阅
配置中心并没有直接用spring-cloud-alibaba-nacos-config（见 [https://github.com/spring-cloud-incubator/spring-cloud-alibaba](https://github.com/spring-cloud-incubator/spring-cloud-alibaba)），因为灰度规则各项操作相对较复杂，所以采用了原生的Nacos Client Api（见 [https://github.com/alibaba/nacos](https://github.com/alibaba/nacos)）来实现

### Common层实现
NacosOperation，封装了几乎所有对Nacos配置中心的操作逻辑，包括
- 根据微服务所在的组和应用名，获取配置
- 根据微服务所在的组和应用名，删除配置
- 根据微服务所在的组和应用名，发布配置
- 根据微服务所在的组和应用名，订阅配置
  - 参数Executor，Nacos提供的接口比较人性化，可以直接使用他们内置的线程池（传入null），也允许使用用户自己定义的线程池，两种方式有效保证配置变更的异步订阅
  - 参数NacosSubscribeCallback，对Nacos订阅的Callback的接口封装
- 根据微服务所在的组和应用名，反订阅配置
```java
public class NacosOperation {
    @Autowired
    private ConfigService nacosConfigService;

    @Autowired
    private Environment environment;

    public String getConfig(String group, String serviceId) throws NacosException {
        long timeout = environment.getProperty(NacosConstant.TIMEOUT, Long.class, NacosConstant.DEFAULT_TIMEOUT);

        return nacosConfigService.getConfig(serviceId, group, timeout);
    }

    public boolean removeConfig(String group, String serviceId) throws NacosException {
        return nacosConfigService.removeConfig(serviceId, group);
    }

    public boolean publishConfig(String group, String serviceId, String config) throws NacosException {
        return nacosConfigService.publishConfig(serviceId, group, config);
    }

    public Listener subscribeConfig(String group, String serviceId, Executor executor, NacosSubscribeCallback subscribeCallback) throws NacosException {
        Listener configListener = new Listener() {
            @Override
            public void receiveConfigInfo(String config) {
                subscribeCallback.callback(config);
            }

            @Override
            public Executor getExecutor() {
                return executor;
            }
        };

        nacosConfigService.addListener(serviceId, group, configListener);

        return configListener;
    }

    public void unsubscribeConfig(String group, String serviceId, Listener configListener) {
        nacosConfigService.removeListener(serviceId, group, configListener);
    }
}
```

NacosSubscribeCallback，对Nacos订阅的Callback的接口封装
```java
public interface NacosSubscribeCallback {
    void callback(String config);
}
```

NacosAutoConfiguration，通过AutoConfiguration初始化ConfigService和NacosOperation
- 通过@ConditionalOnMissingBean的方式，允许用户通过自己实现的ConfigService进行注入，来代替内置方式
- 如果通过内置方式，那么用户只需要在配置文件里，填入相关配置，即可完成初始化。如下配置除了url必填之外，其它也可以由用户自行去定义
```xml
# Nacos config
nacos.url=localhost:8080
# nacos.discovery.namespace=application
# nacos.discovery.timout=30000
```

```java
@Configuration
public class NacosAutoConfiguration {
    @Autowired
    private Environment environment;

    @Bean
    @ConditionalOnMissingBean
    public ConfigService nacosConfigService() throws NacosException {
        Properties properties = new Properties();

        String url = environment.getProperty(NacosConstant.URL);
        if (StringUtils.isNotEmpty(url)) {
            properties.put(NacosConstant.SERVER_ADDR, url);
        } else {
            throw new IllegalArgumentException("Url can't be null or empty");
        }

        String namespace = environment.getProperty(NacosConstant.NAMESPACE);
        if (StringUtils.isNotEmpty(namespace)) {
            properties.put(NacosConstant.NAMESPACE, namespace);
        }

        return NacosFactory.createConfigService(properties);
    }

    @Bean
    public NacosOperation nacosOperation() {
        return new NacosOperation();
    }
}
```

### 微服务端实现
NacosConfigAdapter，继承实现ConfigAdapter（处理灰度发布配置的适配器），主要有三个方法，代码实例通过伪代码方式呈现
- getConfig，用于微服务端在启动的时候，向Nacos服务器请求灰度配置
- subscribeConfig，用于微服务端在启动的时候，完成初始化对灰度配置的监听行为
- close，用于当微服务端断开和Nacos服务器连接或者Spring Bean销毁的时候，执行反订阅和线程池销毁
```java
@Configuration
public class NacosConfigAdapter extends ConfigAdapter {
    private ExecutorService executorService = new ThreadPoolExecutor(2, 4, 0, TimeUnit.MILLISECONDS, new ArrayBlockingQueue<Runnable>(1), new NamedThreadFactory("nacos-config"), new ThreadPoolExecutor.DiscardOldestPolicy());

    @Autowired
    private NacosOperation nacosOperation;

    private Listener configListener;

    @Override
    public String getConfig() throws Exception {
        // 获取Spring Cloud的Metadata中的组名
        String group = ...;
        // 获取Spring Cloud的服务名
        String serviceId = ...;

        return nacosOperation.getConfig(group, serviceId);
    }

    @PostConstruct
    public void subscribeConfig() {
        // 获取Spring Cloud的Metadata中的组名
        String group = ...;
        // 获取Spring Cloud的服务名
        String serviceId = ...;

        configListener = nacosOperation.subscribeConfig(group, serviceId, executorService, new NacosSubscribeCallback() {
            @Override
            public void callback(String config) {
                // 订阅逻辑实现
            }
        });
    }

    @Override
    public void close() {
        if (configListener != null) {
            // 获取Spring Cloud的Metadata中的组名
            String group = ...;
            // 获取Spring Cloud的服务名
            String serviceId = ...;

            nacosOperation.unsubscribeConfig(group, serviceId, configListener);
        }

        executorService.shutdownNow();
    }
}
```

NacosConfigAutoConfiguration，通过AutoConfiguration初始化NacosConfigAdapter
```java
@Configuration
public class NacosConfigAutoConfiguration {
    @Bean
    public NacosConfigAdapter configAdapter() {
        return new NacosConfigAdapter();
    }
}
```

spring.factories，在微服务端的src/main/resources/META-INF/spring.factories中加入自动配置
```xml
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.nepxion.discovery.common.nacos.configuration.NacosAutoConfiguration,\
com.nepxion.discovery.plugin.configcenter.nacos.configuration.NacosConfigAutoConfiguration
```

### 控制平台实现
控制平台的作用是当用户自行研发第三方管理界面的时候，可以通过微服务的方式发布Nacos服务器配置操作和汇聚的接口（我们统称它为控制平台）。对于本系统来说，目前它的作用是为Java Desktop图形化界面提供接口，您也可以使用它自行研发符合您口味的灰度发布界面
NacosConfigAdapter，继承实现ConfigAdapter（控制平台操作配置的适配器），该类和“服务端”的类同名，但并不是同一个，主要有三个方法
- updateConfig，用于用户界面更新配置
- clearConfig，用于用户界面清楚配置
- getConfig，用于用户界面获取配置
```java
public class NacosConfigAdapter implements ConfigAdapter {
    @Autowired
    private NacosOperation nacosOperation;

    @Override
    public boolean updateConfig(String group, String serviceId, String config) throws Exception {
        return nacosOperation.publishConfig(group, serviceId, config);
    }

    @Override
    public boolean clearConfig(String group, String serviceId) throws Exception {
        return nacosOperation.removeConfig(group, serviceId);
    }

    @Override
    public String getConfig(String group, String serviceId) throws Exception {
        return nacosOperation.getConfig(group, serviceId);
    }
}
```

spring.factories，在控制平台的src/main/resources/META-INF/spring.factories中加入自动配置
```xml
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.nepxion.discovery.common.nacos.configuration.NacosAutoConfiguration,\
com.nepxion.discovery.console.nacos.configuration.NacosConfigAutoConfiguration
```

## 利用Nacos控制台，实现Spring Cloud的灰度发布和路由规则的配置
敬请期待Nacos 0.3.0版本，推出Nacos控制台