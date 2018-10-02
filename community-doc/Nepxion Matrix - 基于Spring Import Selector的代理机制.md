## Nepxion Matrix - 基于Spring Import Selector的代理机制

### 前言
Nepxion Matrix是一款集成Spring AutoProxy，Spring Registrar和Spring Import Selector三种机制的AOP框架，具有很高的通用性、健壮性、灵活性和易用性

更多内容请访问 [https://github.com/Nepxion/Matrix](https://github.com/Nepxion/Matrix)

### 主题
本文主要阐述基于Spring Import Selector的代理机制，实现类似Spring Cloud中@EnableXXX的注解，并配合配置文件中的配置项，达到开关功能，源码参考和剥离Spring Cloud相关代码而成。它的特点是
- 入口加上@EnableXXX，并提供在spring.factories定义@EnableXXX和Configuration类的关联，达到通过注解的在入口配置与否，来控制对应功能的开启和关闭
- 提供在application.properties配置参数，和入口@EnableXXX配置，达到复合控制

### 源代码
我们通过源代码来讲解（具体代码位于matrix-aop工程下的com.nepxion.matrix.selector）

#### 抽象ImportSelector - AbstractImportSelector
```java
public abstract class AbstractImportSelector<T> implements DeferredImportSelector, BeanClassLoaderAware, EnvironmentAware {
    private static final Logger LOG = LoggerFactory.getLogger(AbstractImportSelector.class);

    static {
        System.out.println("");
        System.out.println("╔═╗╔═╗   ╔╗");
        System.out.println("║║╚╝║║  ╔╝╚╗");
        System.out.println("║╔╗╔╗╠══╬╗╔╬═╦╦╗╔╗");
        System.out.println("║║║║║║╔╗║║║║╔╬╬╬╬╝");
        System.out.println("║║║║║║╔╗║║╚╣║║╠╬╬╗");
        System.out.println("╚╝╚╝╚╩╝╚╝╚═╩╝╚╩╝╚╝");
        System.out.println("Nepxion Matrix - Import Selector  v2.0.1");
        System.out.println("");
    }

    private ClassLoader beanClassLoader;
    private Class<T> annotationClass;
    private Environment environment;

    @SuppressWarnings("unchecked")
    protected AbstractImportSelector() {
        this.annotationClass = (Class<T>) GenericTypeResolver.resolveTypeArgument(this.getClass(), AbstractImportSelector.class);
    }

    @Override
    public String[] selectImports(AnnotationMetadata metadata) {
        if (!isEnabled()) {
            return new String[0];
        }

        AnnotationAttributes attributes = AnnotationAttributes.fromMap(metadata.getAnnotationAttributes(this.annotationClass.getName(), true));

        Assert.notNull(attributes, "No " + getSimpleName() + " attributes found. Is " + metadata.getClassName() + " annotated with @" + getSimpleName() + "?");

        // Find all possible auto configuration classes, filtering duplicates
        List<String> factories = new ArrayList<>(new LinkedHashSet<>(SpringFactoriesLoader.loadFactoryNames(this.annotationClass, this.beanClassLoader)));

        if (factories.isEmpty() && !hasDefaultFactory()) {
            throw new IllegalStateException("Annotation @" + getSimpleName() + " found, but there are no implementations. Did you forget to include a starter?");
        }

        if (factories.size() > 1) {
            // there should only ever be one DiscoveryClient, but there might be more than one factory
            LOG.warn("More than one implementation " + "of @" + getSimpleName() + " (now relying on @Conditionals to pick one): " + factories);
        }

        return factories.toArray(new String[factories.size()]);
    }

    protected boolean hasDefaultFactory() {
        return false;
    }

    protected String getSimpleName() {
        return this.annotationClass.getSimpleName();
    }

    protected Class<T> getAnnotationClass() {
        return this.annotationClass;
    }

    protected Environment getEnvironment() {
        return this.environment;
    }

    @Override
    public void setEnvironment(Environment environment) {
        this.environment = environment;
    }

    @Override
    public void setBeanClassLoader(ClassLoader classLoader) {
        this.beanClassLoader = classLoader;
    }

    // 由实现类对应注解的开关
    protected abstract boolean isEnabled();
}
```

### 示例
我们通过示例代码来讲解（具体代码位于matrix-spring-boot-selector-example工程下）

#### EnableMyAnnotationImportSelector
继承AbstractImportSelector
- selectImports方法一般不必去覆盖，本例为了演示复杂场景而已
- 只有当入口加了@EnableMyAnnotation注解，并且com.nepxion.myannotation.enabled定义在配置文件中值为true（或者缺失），对应的功能才会启动
- 注意Spring boot 1.x.x和2.x.x的用法区别（因为RelaxedPropertyResolver在2.x.x已经被删除）
```java
@Order(Ordered.LOWEST_PRECEDENCE - 100)
public class EnableMyAnnotationImportSelector extends AbstractImportSelector<EnableMyAnnotation> {
    // 如下方法适合EnableXXX注解上带有参数的情形，一般用不到
    @Override
    public String[] selectImports(AnnotationMetadata metadata) {
        // 获取父类的Configuration列表
        String[] imports = super.selectImports(metadata);

        // 从注解上获取参数
        AnnotationAttributes attributes = AnnotationAttributes.fromMap(metadata.getAnnotationAttributes(getAnnotationClass().getName(), true));
        boolean extension = attributes.getBoolean("extension");

        if (extension) {
            // 如果EnableMyAnnotation注解上的extension为true，那么去装载MyConfigurationExtension，即初始化里面的MyBeanExtension
            List<String> importsList = new ArrayList<>(Arrays.asList(imports));
            importsList.add(MyConfigurationExtension.class.getCanonicalName());
            imports = importsList.toArray(new String[0]);
        } else {
            // 如果EnableMyAnnotation注解上的extension为false，那么你可以把该参数动态放到属性列表里
            Environment environment = getEnvironment();
            if (ConfigurableEnvironment.class.isInstance(environment)) {
                ConfigurableEnvironment configurableEnvironment = (ConfigurableEnvironment) environment;
                LinkedHashMap<String, Object> map = new LinkedHashMap<>();
                map.put("extension.enabled", false);
                MapPropertySource propertySource = new MapPropertySource("nepxion", map);
                configurableEnvironment.getPropertySources().addLast(propertySource);
            }
        }

        return imports;
    }

    @Override
    protected boolean isEnabled() {
        // Spring boot 1.x.x版本的用法
        return new RelaxedPropertyResolver(getEnvironment()).getProperty("com.nepxion.myannotation.enabled", Boolean.class, Boolean.TRUE);

        // Spring boot 2.x.x版本的用法
        // return getEnvironment().getProperty("com.nepxion.myannotation.enabled", Boolean.class, Boolean.TRUE);
    }
}
```

#### EnableMyAnnotation
必须@Import(EnableMyAnnotationImportSelector.class)
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Import(EnableMyAnnotationImportSelector.class)
public @interface EnableMyAnnotation {
    boolean extension() default true;
}
```

#### MyConfiguration
```java
@Configuration
public class MyConfiguration {
    @Bean
    public MyBean myBean1() {
        return new MyBean("MyBean1");
    }
}
```

#### spring.factories
```xml
com.nepxion.matrix.selector.example.aop.EnableMyAnnotation=\
com.nepxion.matrix.selector.example.Configuration.MyConfiguration
```

#### MyApplication
- 本例展示在Spring Boot入口加上@EnableMyAnnotation，自动初始化对应的Configuration
- 通过spring.factories定义注解对应的配置类，当@EnableMyAnnotation加上，同时application.properties里com.nepxion.myannotation.enabled=true（或者不配置），那么MyBean myBean = MyContextAware.getBean(MyBean.class);才能取到
```java
@SpringBootApplication
@EnableMyAnnotation(extension = false)
public class MyApplication {
    public static void main(String[] args) throws Exception {
        ConfigurableApplicationContext applicationContext = SpringApplication.run(MyApplication.class, args);

        try {
            System.out.println(applicationContext.getBean("myBean1"));
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```
运行MyApplication，从最终输出的结果，随着注解或者配置项的不同逻辑，引起applicationContext.getBean("myBean1")中myBean1实例在容器中存在或者不存在，证明MyConfiguration的配置是根据注解或者配置项的不同逻辑决定是否需要装配