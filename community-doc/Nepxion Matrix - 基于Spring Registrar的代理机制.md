## Nepxion Matrix - 基于Spring Registrar的代理机制

### 前言
Nepxion Matrix是一款集成Spring AutoProxy，Spring Registrar和Spring Import Selector三种机制的AOP框架，具有很高的通用性、健壮性、灵活性和易用性

更多内容请访问 [https://github.com/Nepxion/Matrix](https://github.com/Nepxion/Matrix)

### 主题
本文主要阐述基于Spring Registrar的代理机制，实现类似Spring Cloud中@FeignClient伪代理的功能，只有接口没有实现类，就能实现注入和动态代理，源码参考和剥离Spring Cloud相关代码而成。它的特点是
- 如果本地只有接口并加相关的注解，那么执行对应的切面调用方式
- 如果本地有接口（不管是否加注解），并也有实现类，那么执行对应的实现类的逻辑

### 源代码
我们通过源代码来讲解（具体代码位于matrix-aop工程下的com.nepxion.matrix.registrar）

#### 抽象登记类 - AbstractRegistrar
应用启动后通过进行包扫描，当接口含有自定义的注解（用法类似于@FeignClient），根据注解以及其相关信息，创建BeanDefinition相应范畴的一系列对象，并通过RegistrarFactoryBean进行委托，最后全部注入到Spring Ioc容器中
```java
public abstract class AbstractRegistrar implements ImportBeanDefinitionRegistrar, ResourceLoaderAware, BeanClassLoaderAware, EnvironmentAware {
    private static final Logger LOG = LoggerFactory.getLogger(AbstractRegistrar.class);

    static {
        System.out.println("");
        System.out.println("╔═╗╔═╗   ╔╗");
        System.out.println("║║╚╝║║  ╔╝╚╗");
        System.out.println("║╔╗╔╗╠══╬╗╔╬═╦╦╗╔╗");
        System.out.println("║║║║║║╔╗║║║║╔╬╬╬╬╝");
        System.out.println("║║║║║║╔╗║║╚╣║║╠╬╬╗");
        System.out.println("╚╝╚╝╚╩╝╚╝╚═╩╝╚╩╝╚╝");
        System.out.println("Nepxion Matrix - Registrar  v2.0.1");
        System.out.println("");
    }

    private ResourceLoader resourceLoader;
    private ClassLoader classLoader;
    private Environment environment;

    public AbstractRegistrar() {

    }

    @Override
    public void setResourceLoader(ResourceLoader resourceLoader) {
        this.resourceLoader = resourceLoader;
    }

    @Override
    public void setBeanClassLoader(ClassLoader classLoader) {
        this.classLoader = classLoader;
    }

    @Override
    public void setEnvironment(Environment environment) {
        this.environment = environment;
    }

    @Override
    public void registerBeanDefinitions(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
        registerAnnotations(metadata, registry);
    }

    public void registerAnnotations(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
        ClassPathScanningCandidateComponentProvider scanner = getScanner();
        scanner.setResourceLoader(this.resourceLoader);

        // 扫描带有自定义注解的类
        AnnotationTypeFilter annotationTypeFilter = new AnnotationTypeFilter(getAnnotationClass());
        scanner.addIncludeFilter(annotationTypeFilter);
        // 确定扫描的包路径列表
        Set<String> basePackages = getBasePackages(metadata);

        // 循环扫描，并把根据注解信息，进行相关注册
        for (String basePackage : basePackages) {
            Set<BeanDefinition> candidateComponents = scanner.findCandidateComponents(basePackage);
            for (BeanDefinition candidateComponent : candidateComponents) {
                if (candidateComponent instanceof AnnotatedBeanDefinition) {
                    // verify annotated class is an interface
                    AnnotatedBeanDefinition beanDefinition = (AnnotatedBeanDefinition) candidateComponent;
                    AnnotationMetadata annotationMetadata = beanDefinition.getMetadata();

                    Map<String, Object> attributes = annotationMetadata.getAnnotationAttributes(getAnnotationClass().getCanonicalName());
                    registerAnnotation(registry, annotationMetadata, attributes);
                }
            }
        }
    }

    private void registerAnnotation(BeanDefinitionRegistry registry, AnnotationMetadata annotationMetadata, Map<String, Object> attributes) {
        String className = annotationMetadata.getClassName();

        LOG.info("Found annotation [{}] in {} ", getAnnotationClass().getSimpleName(), className);

        BeanDefinitionBuilder definition = BeanDefinitionBuilder.genericBeanDefinition(getBeanClass());
        AbstractBeanDefinition beanDefinition = definition.getBeanDefinition();

        // 第三方可以扩展增加更多的自定义属性
        customize(registry, annotationMetadata, attributes, definition);

        // 把interfaze属性赋予RegistrarFactoryBean
        try {
            definition.addPropertyValue("interfaze", Class.forName(className));
        } catch (ClassNotFoundException e) {
            LOG.error("Get interface for name error", e);
        }
        // 把interceptor属性赋予RegistrarFactoryBean		
        definition.addPropertyValue("interceptor", getInterceptor(beanDefinition.getPropertyValues()));

        definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);

        String alias = className;

        // 创建BeanDefinitiond相应范畴的一系列对象，最后注入到Spring Ioc容器中		
        BeanDefinitionHolder holder = new BeanDefinitionHolder(beanDefinition, className, new String[] { alias });
        BeanDefinitionReaderUtils.registerBeanDefinition(holder, registry);
    }

    protected ClassPathScanningCandidateComponentProvider getScanner() {
        // Spring的工具类，实现按自定义的类型，查找classpath下符合要求的class文件
        // false参数的作用是关闭默认TypeFilter，因为默认TypeFilte的模式会查询出不符合要求的class名
        return new ClassPathScanningCandidateComponentProvider(false, this.environment) {
            // 确定成为候选类的先决条件
            @Override
            protected boolean isCandidateComponent(AnnotatedBeanDefinition beanDefinition) {
                if (beanDefinition.getMetadata().isIndependent()) {
                    // 注解的目标类必须是接口等相关的判断逻辑
                    if (beanDefinition.getMetadata().isInterface() && beanDefinition.getMetadata().getInterfaceNames().length == 1 && Annotation.class.getName().equals(beanDefinition.getMetadata().getInterfaceNames()[0])) {
                        try {
                            Class<?> target = ClassUtils.forName(beanDefinition.getMetadata().getClassName(), AbstractRegistrar.this.classLoader);

                            return !target.isAnnotation();
                        } catch (Exception ex) {
                            LOG.error("Could not load target class: " + beanDefinition.getMetadata().getClassName(), ex);
                        }
                    }

                    return true;
                }

                return false;
            }
        };
    }

    protected Set<String> getBasePackages(AnnotationMetadata importingClassMetadata) {
        Map<String, Object> attributes = importingClassMetadata.getAnnotationAttributes(getEnableAnnotationClass().getCanonicalName());

        Set<String> basePackages = new HashSet<>();
        for (String pkg : (String[]) attributes.get("value")) {
            if (StringUtils.hasText(pkg)) {
                basePackages.add(pkg);
            }
        }
        for (String pkg : (String[]) attributes.get("basePackages")) {
            if (StringUtils.hasText(pkg)) {
                basePackages.add(pkg);
            }
        }
        for (Class<?> clazz : (Class[]) attributes.get("basePackageClasses")) {
            basePackages.add(ClassUtils.getPackageName(clazz));
        }

        // 如果扫描目录未设定，则取当前目录做为扫描目录		
        if (basePackages.isEmpty()) {
            basePackages.add(ClassUtils.getPackageName(importingClassMetadata.getClassName()));
        }

        return basePackages;
    }

    protected void customize(BeanDefinitionRegistry registry, AnnotationMetadata annotationMetadata, Map<String, Object> attributes, BeanDefinitionBuilder definition) {
        // 第三方覆盖customize方法，可以扩展增加更多的自定义属性
        for (Map.Entry<String, Object> attribute : attributes.entrySet()) {
            definition.addPropertyValue(attribute.getKey(), attribute.getValue());
        }
    }

    // 返回自定义Enable注解类，用法类似于@EnableFeignClients
    protected abstract Class<? extends Annotation> getEnableAnnotationClass();

    // 返回自定义注解类，用法类似于@FeignClient
    protected abstract Class<? extends Annotation> getAnnotationClass();

    // 返回RegistrarFactoryBean委托类
    protected abstract Class<?> getBeanClass();

    // 返回MethodInterceptor拦截类	
    protected abstract MethodInterceptor getInterceptor(MutablePropertyValues annotationValues);
}
```

#### 抽象委托类 - RegistrarFactoryBean
它是一个FactoryBean类，作为代理机制的委托和桥梁作用。它在AbstractRegistrar被注册到Spring Ioc容器中。在容器初始化后，执行afterPropertiesSet，确定确定实现类和接口的代理关系，并创建代理对象
```java
public class RegistrarFactoryBean implements ApplicationContextAware, FactoryBean<Object>, InitializingBean, BeanClassLoaderAware {
    private ApplicationContext applicationContext;
    // 接口类型
    private Class<?> interfaze;
    // 切面类
    private MethodInterceptor interceptor;
    // 代理类
    private Object proxy;
    private ClassLoader classLoader;

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }

    public ApplicationContext getApplicationContext() {
        return applicationContext;
    }

    @Override
    public Object getObject() throws Exception {
        // 返回代理对象
        return proxy;
    }

    @Override
    public Class<?> getObjectType() {
        // 返回AbstractRegistrar注册的接口类型
        return interfaze;
    }

    @Override
    public boolean isSingleton() {
        return false;
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        // 确定实现类和接口的代理关系
        ProxyFactory proxyFactory = new ProxyFactory();
        proxyFactory.addInterface(interfaze);
        proxyFactory.addAdvice(interceptor);
        proxyFactory.setOptimize(false);

        // 创建代理对象
        proxy = proxyFactory.getProxy(classLoader);
    }

    @Override
    public void setBeanClassLoader(ClassLoader classLoader) {
        this.classLoader = classLoader;
    }

    public Class<?> getInterfaze() {
        return interfaze;
    }

    public void setInterfaze(Class<?> interfaze) {
        this.interfaze = interfaze;
    }

    public MethodInterceptor getInterceptor() {
        return interceptor;
    }

    public void setInterceptor(MethodInterceptor interceptor) {
        this.interceptor = interceptor;
    }
}
```

#### 抽象切面类 - AbstractRegistrarInterceptor
它是一个Interceptor类，实现最终的业务层面的代理逻辑。MutablePropertyValues是汇集注解属性，供供业务层参考和使用，一般可能用不到
```java
public abstract class AbstractRegistrarInterceptor extends AbstractInterceptor {
    // 注解属性，以供业务层参考和使用
    protected MutablePropertyValues annotationValues;

    public AbstractRegistrarInterceptor(MutablePropertyValues annotationValues) {
        this.annotationValues = annotationValues;
    }

    public MutablePropertyValues getAnnotationValues() {
        return annotationValues;
    }

    public String getInterface(MethodInvocation invocation) {
        return getMethod(invocation).getDeclaringClass().getCanonicalName();
    }
}
```

### 示例
我们通过示例代码来讲解（具体代码位于matrix-spring-boot-registrar-example工程下）

#### EnableMyAnnotation
用法类似于@EnableFeignClients，必须@Import(MyRegistrar.class)
```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(MyRegistrar.class)
public @interface EnableMyAnnotation {
    String[] value() default {};

    String[] basePackages() default {};

    Class<?>[] basePackageClasses() default {};
}
```

#### MyAnnotation
用法类似于@FeignClient
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface MyAnnotation {
    String name();

    String label();

    String description() default "";
}
```

#### MyRegistrarFactoryBean
继承RegistrarFactoryBean，实现MyAnnotation注解属性的委托
```java
public class MyRegistrarFactoryBean extends RegistrarFactoryBean {
    private String name;
    private String label;
    private String description;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getLabel() {
        return label;
    }

    public void setLabel(String label) {
        this.label = label;
    }

    public String getDescription() {
        return description;
    }

    public void setDescription(String description) {
        this.description = description;
    }

    @Override
    public int hashCode() {
        return HashCodeBuilder.reflectionHashCode(this);
    }

    @Override
    public boolean equals(Object object) {
        return EqualsBuilder.reflectionEquals(this, object);
    }

    @Override
    public String toString() {
        return ToStringBuilder.reflectionToString(this, ToStringStyle.MULTI_LINE_STYLE);
    }
}
```

#### MyInterceptor
继承AbstractRegistrarInterceptor，实现业务层的切面拦截
```java
public class MyInterceptor extends AbstractRegistrarInterceptor {
    public MyInterceptor(MutablePropertyValues annotationValues) {
        super(annotationValues);
    }

    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        System.out.println("---------------------代理信息---------------------");
        String interfaze = getInterface(invocation);
        String methodName = getMethodName(invocation);
        Object[] arguments = getArguments(invocation);

        System.out.println("Interface=" + interfaze + ", methodName=" + methodName + ", arguments=" + arguments[0]);

        Class<?> interfaceClass = (Class<?>) annotationValues.get("interfaze");
        String name = annotationValues.get("name").toString();
        String label = annotationValues.get("label").toString();
        String description = annotationValues.get("description").toString();

        System.out.println("Interface class=" + interfaceClass + ", annotation:name=" + name + ", label=" + label + ", description=" + description);
        System.out.println("-------------------------------------------------");

        // 实现业务代码

        return "代理返回 " + arguments[0];
    }
}
```

#### MyRegistrar
继承AbstractRegistrar，实现上面定义的类的返回
```java
public class MyRegistrar extends AbstractRegistrar {
    @Override
    protected Class<? extends Annotation> getEnableAnnotationClass() {
        return EnableMyAnnotation.class;
    }

    @Override
    protected Class<? extends Annotation> getAnnotationClass() {
        return MyAnnotation.class;
    }

    @Override
    protected Class<?> getBeanClass() {
        return MyRegistrarFactoryBean.class;
    }

    @Override
    protected MethodInterceptor getInterceptor(MutablePropertyValues annotationValues) {
        return new MyInterceptor(annotationValues);
    }
}
```

完成上述实现后，我们可以在业务层面进行实现

#### MyService1
用法类似于@FeignClient
```java
@MyAnnotation(name = "a", label = "b", description = "c")
public interface MyService1 {
    String doA(String id);

    String doB(String id);
}
```

#### MyService2
```java
@MyAnnotation(name = "x", label = "y", description = "z")
public interface MyService2 {
    String doC(String id);

    String doD(String id);
}
```

#### MyService3
```java
@MyAnnotation(name = "1", label = "2", description = "3")
public interface MyService3 {
    String doE(String id);

    String doF(String id);
}
```

#### MyService3Impl
```java
@Service
@Primary // 优先Bean注入，用来代替MyService3后置切面拦截调用
public class MyService3Impl implements MyService3 {
    public String doE(String id) {
        return "直接返回 " + id;
    }

    public String doF(String id) {
        return "直接返回 " + id;
    }
}
```

#### MyInvoker
```java
@Component
public class MyInvoker {
    @Autowired
    private MyService1 myService1;

    @Autowired
    private MyService2 myService2;

    @Autowired
    private MyService3 myService3;

    public String invokeMyService1() {
        return myService1.doA("A");
    }

    public String invokeMyService2() {
        return myService2.doC("C");
    }

    public String invokeMyService3() {
        return myService3.doE("E");
    }
}
```

#### MyApplication
- 本例展示在Spring Boot入口加上@EnableMyAnnotation，在接口（不需要实现类）加上MyAnnotation，就可以实现调用拦截的功能，模拟出FeignClient的调用场景
- MySerivice1，MySerivice2头部都加有@EnableMyAnnotation注解，执行MyInterceptor的切面调用
- MySerivice3虽然头部加有注解，但它有实现类，那么优先执行实现类的注入（@Primary），MyInterceptor的切面调用将不起效果
```java
@SpringBootApplication
@EnableMyAnnotation
public class MyApplication {
    public static void main(String[] args) throws Exception {
        ConfigurableApplicationContext applicationContext = SpringApplication.run(MyApplication.class, args);

        MyInvoker myInvoker = applicationContext.getBean(MyInvoker.class);
        System.out.println("调用MyServier1，返回值=" + myInvoker.invokeMyService1());
        System.out.println("调用MyServier2，返回值=" + myInvoker.invokeMyService2());
        System.out.println("调用MyServier3，返回值=" + myInvoker.invokeMyService3());
    }
}
```
运行MyApplication，最终输出结果，从接口我们可以看到，调用MyServier1和MyServier2执行MyInterceptor中的逻辑，调用MyServier3执行它的实现类中的逻辑
```xml
---------------------代理信息---------------------
Interface=com.nepxion.matrix.registrar.example.service.MyService1, methodName=doA, arguments=A
Interface class=interface com.nepxion.matrix.registrar.example.service.MyService1, annotation:name=a, label=b, description=c
-------------------------------------------------
调用MyServier1，返回值=代理返回 A
---------------------代理信息---------------------
Interface=com.nepxion.matrix.registrar.example.service.MyService2, methodName=doC, arguments=C
Interface class=interface com.nepxion.matrix.registrar.example.service.MyService2, annotation:name=x, label=y, description=z
-------------------------------------------------
调用MyServier2，返回值=代理返回 C
调用MyServier3，返回值=直接返回 E
```