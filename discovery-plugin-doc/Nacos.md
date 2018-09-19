# Nepxion Discovery - 整合Nacos到Spring Cloud做灰度发布（一）

## 引子
Nepxion Discovery是一款对Spring Cloud Discovery服务注册发现、Ribbon负载均衡、Feign和RestTemplate调用的增强中间件，其功能包括灰度发布（包括切换发布和平滑发布）、服务隔离、服务路由、服务权重、黑/白名单的IP地址过滤、限制注册、限制发现等，支持Eureka、Consul、Zookeeper和阿里巴巴的Nacos为服务注册发现中间件，支持阿里巴巴的Nacos、携程的Apollo和Redis为远程配置中心，支持Spring Cloud Api Gateway（Finchley版）、Zuul网关和微服务的灰度发布，支持多数据源的数据库灰度发布等客户特色化灰度发布，支持用户自定义和编程灰度路由策略（包括RPC和REST两种调用方式），兼容Spring Cloud Edgware版和Finchley版（不支持Dalston版，因为它的生命周期将在2018年12月结束，如果您无法回避使用Dalston版，请自行修改源码或者联系我）。现有的Spring Cloud微服务很方便引入该中间件，代码零侵入

更多内容请访问 [https://github.com/Nepxion/Discovery](https://github.com/Nepxion/Discovery)

## 前言
那么如何整合Nacos到Spring Cloud框架中做灰度发布呢？内容分为三部分讲解，但只涉及到整合Nacos的部分，涉及到灰度发布的逻辑则不在本文的讲述范围内，请自行访问Github进行研究。由于Nacos具有非常好的用户易用性，所以整合起来代码量很少，也很简单
- 整合Nacos服务发现到Spring Cloud做灰度发布
- 利用Nacos配置中心到Spring Cloud做灰度发布
- 利用Nacos控制台做Spring Cloud的灰度发布

## 整合Nacos服务发现到Spring Cloud做灰度发布
### 初始化类
对NacosServiceRegistry对象进行拦截，执行装饰者模式，由NacosServiceRegistryDecorator去代理；对NacosDiscoveryProperties对象进行拦截，并把灰度发布所要用到的Metadata数据植入，并注册到Nacos服务器上
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
            metadata.put(DiscoveryConstant.SPRING_APPLICATION_DISCOVERY_PLUGIN, NacosConstant.DISCOVERY_PLUGIN);
            metadata.put(DiscoveryConstant.SPRING_APPLICATION_REGISTER_CONTROL_ENABLED, PluginContextAware.isRegisterControlEnabled(environment).toString());
            metadata.put(DiscoveryConstant.SPRING_APPLICATION_DISCOVERY_CONTROL_ENABLED, PluginContextAware.isDiscoveryControlEnabled(environment).toString());
            metadata.put(DiscoveryConstant.SPRING_APPLICATION_CONFIG_REST_CONTROL_ENABLED, PluginContextAware.isConfigRestControlEnabled(environment).toString());
            metadata.put(DiscoveryConstant.SPRING_APPLICATION_GROUP_KEY, PluginContextAware.getGroupKey(environment));
            metadata.put(DiscoveryConstant.SPRING_APPLICATION_CONTEXT_PATH, PluginContextAware.getContextPath(environment));

            return bean;
        } else {
            return bean;
        }
    }
}
```

### 装饰类
NacosServiceRegistryDecorator继承NacosServiceRegistry，并实现对它的核心方法进行拦截和装饰，从而实现在注册层面的“黑/白名单的IP地址注册的过滤规则”、“最大注册数的限制的过滤规则”等功能
```java
public class NacosServiceRegistryDecorator extends NacosServiceRegistry {
    private static final Logger LOG = LoggerFactory.getLogger(NacosServiceRegistryDecorator.class);

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
        Boolean registerControlEnabled = PluginContextAware.isRegisterControlEnabled(environment);
        if (registerControlEnabled) {
            try {
                RegisterListenerExecutor registerListenerExecutor = applicationContext.getBean(RegisterListenerExecutor.class);
                registerListenerExecutor.onRegister(registration);
            } catch (BeansException e) {
                LOG.warn("Get bean for RegisterListenerExecutor failed, ignore to executor listener");
            }
        }

        serviceRegistry.register(registration);
    }

    @Override
    public void deregister(NacosRegistration registration) {
        Boolean registerControlEnabled = PluginContextAware.isRegisterControlEnabled(environment);
        if (registerControlEnabled) {
            try {
                RegisterListenerExecutor registerListenerExecutor = applicationContext.getBean(RegisterListenerExecutor.class);
                registerListenerExecutor.onDeregister(registration);
            } catch (BeansException e) {
                LOG.warn("Get bean for RegisterListenerExecutor failed, ignore to executor listener");
            }
        }

        serviceRegistry.deregister(registration);
    }

    @Override
    public void setStatus(NacosRegistration registration, String status) {
        Boolean registerControlEnabled = PluginContextAware.isRegisterControlEnabled(environment);
        if (registerControlEnabled) {
            try {
                RegisterListenerExecutor registerListenerExecutor = applicationContext.getBean(RegisterListenerExecutor.class);
                registerListenerExecutor.onSetStatus(registration, status);
            } catch (BeansException e) {
                LOG.warn("Get bean for RegisterListenerExecutor failed, ignore to executor listener");
            }
        }

        serviceRegistry.setStatus(registration, status);
    }

    @Override
    public <T> T getStatus(NacosRegistration registration) {
        return serviceRegistry.getStatus(registration);
    }

    @Override
    public void close() {
        Boolean registerControlEnabled = PluginContextAware.isRegisterControlEnabled(environment);
        if (registerControlEnabled) {
            try {
                RegisterListenerExecutor registerListenerExecutor = applicationContext.getBean(RegisterListenerExecutor.class);
                registerListenerExecutor.onClose();
            } catch (BeansException e) {
                LOG.warn("Get bean for RegisterListenerExecutor failed, ignore to executor listener");
            }
        }

        serviceRegistry.close();
    }

    public ConfigurableEnvironment getEnvironment() {
        return environment;
    }
}
```

NacosServerListDecorator继承NacosServerList，并实现对它的核心方法进行拦截和过滤，从而实现在负载均衡层面的“版本访问的灰度路由规则”、“版本权重的灰度路由规则”、“区域权重的灰度路由规则”等功能
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
        List<NacosServer> servers = super.getInitialListOfServers();

        filter(servers);

        return servers;
    }

    @Override
    public List<NacosServer> getUpdatedListOfServers() {
        List<NacosServer> servers = super.getUpdatedListOfServers();

        filter(servers);

        return servers;
    }

    private void filter(List<NacosServer> servers) {
        Boolean discoveryControlEnabled = PluginContextAware.isDiscoveryControlEnabled(environment);
        if (discoveryControlEnabled) {
            String serviceId = getServiceId();
            loadBalanceListenerExecutor.onGetServers(serviceId, servers);
        }
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
由于在不同的服务注册发现插件（例如，Eureka、Consul、Zookeeper、Nacos），或者Metadata是在Server的实现类上，所以我们要做一层适配
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

### 配置类
替换NacosRibbonClientConfiguration里的ribbonServerList方法，植入灰度发布的负载均衡执行器
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

## 利用Nacos配置中心到Spring Cloud做灰度发布
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

### 服务端实现
NacosConfigAdapter，继承实现ConfigAdapter（处理灰度发布配置的适配器），主要有三个方法，代码实例通过伪代码方式呈现
- getConfig，用于服务端在启动的时候，向Nacos服务器请求灰度配置
- subscribeConfig，用于服务端在启动的时候，完成初始化对灰度配置的监听行为
- close，用于当服务端断开和Nacos服务器连接或者Spring Bean销毁的时候，执行反订阅和线程池销毁
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
控制平台的作用是当用户自行研发基于Nacos配置界面的时候，可以通过微服务的方式发布Nacos服务器配置的操作接口
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

## 利用Nacos控制台做Spring Cloud的灰度发布
敬请期待Nacos 0.3.0版本