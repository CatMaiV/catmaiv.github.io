---
layout:     post
title:     我的SpringUtil怎么失效了！？
subtitle:  记一次开发过程中多数据源配置排错过程，记录SpringUtil无法从IOC容器中获取Bean实例的排查过程，对比ApplicationContext和ConfigurableListableBeanFactory的差别。
date:       2025-09-02
author:     catmai
header-img: img/post-bg-re-vs-ng2.jpg
catalog: true
tags:
    - Java
    - Spring
---



# 最前

某一天我像往常一样改造项目中的数据源部分，把单一数据源改造成双数据源。参考了若依的多数据源实现，然而当我实际运行时发现，数据源并没有被切换过来，目标sql依然去主库执行了（报了表不存在的错误），由此引发了后续排错的过程。



# 排错过程

## 多数据源配置

首先能想到的肯定时多数据源配错了，虽然配置过很多次了不过也难免会翻车。于是开始检查配置是否正确....

``` yaml
spring:
    datasource:
        type: com.alibaba.druid.pool.DruidDataSource
        driverClassName: com.mysql.cj.jdbc.Driver
        druid:
            # 主库数据源
            master:
                url: jdbc:mysql://xxxxx?useUnicode=true&characterEncoding=UTF-8&serverTimezone=Asia/Shanghai&allowPublicKeyRetrieval=true&verifyServerCertificate=false&useSSL=false
                username: xxx
                password: xxx
            # 从库数据源
            slave:
                # 从数据源开关/默认关闭
                enabled: true
                url: jdbc:mysql://xxxxx?useUnicode=true&characterEncoding=UTF-8&serverTimezone=Asia/Shanghai&allowPublicKeyRetrieval=true&verifyServerCertificate=false&useSSL=false
                username: xxx
                password: xxxx
            # 初始连接数
            initialSize: 5
            # 最小连接池数量
            minIdle: 10
            # 最大连接池数量
            maxActive: 20
            # 配置获取连接等待超时的时间
            maxWait: 60000
            # 配置连接超时时间
            connectTimeout: 30000
            # 配置网络超时时间
            socketTimeout: 60000
            # 后面略了
```

数据源切换：

``` java
public class DynamicDataSourceContextHolder
{
    public static final Logger log = LoggerFactory.getLogger(DynamicDataSourceContextHolder.class);

    /**
     * 使用ThreadLocal维护变量，ThreadLocal为每个使用该变量的线程提供独立的变量副本，
     * 所以每一个线程都可以独立地改变自己的副本，而不会影响其它线程所对应的副本。
     */
    private static final ThreadLocal<String> CONTEXT_HOLDER = new ThreadLocal<>();

    /**
     * 设置数据源的变量
     */
    public static void setDataSourceType(String dsType)
    {
        log.info("切换到{}数据源", dsType);
        CONTEXT_HOLDER.set(dsType);
    }

    /**
     * 获得数据源的变量
     */
    public static String getDataSourceType()
    {
        return CONTEXT_HOLDER.get();
    }

    /**
     * 清空数据源变量
     */
    public static void clearDataSourceType()
    {
        CONTEXT_HOLDER.remove();
    }
}
```

动态数据源：

``` java
public class DynamicDataSource extends AbstractRoutingDataSource
{
    public DynamicDataSource(DataSource defaultTargetDataSource, Map<Object, Object> targetDataSources)
    {
        super.setDefaultTargetDataSource(defaultTargetDataSource);
        super.setTargetDataSources(targetDataSources);
        super.afterPropertiesSet();
    }

    @Override
    protected Object determineCurrentLookupKey()
    {
        return DynamicDataSourceContextHolder.getDataSourceType();
    }
}
```

Druid配置：

``` java
@Slf4j
@Configuration
public class DruidConfig {
    @Bean
    @ConfigurationProperties("spring.datasource.druid.master")
    public DataSource masterDataSource(DruidProperties druidProperties) {
        DruidDataSource dataSource = DruidDataSourceBuilder.create().build();
        return druidProperties.dataSource(dataSource);
    }

    @Bean(name = "slaveDataSource")
    @ConfigurationProperties("spring.datasource.druid.slave")
    @ConditionalOnProperty(prefix = "spring.datasource.druid.slave", name = "enabled", havingValue = "true")
    public DataSource slaveDataSource(DruidProperties druidProperties) {
        DruidDataSource dataSource = DruidDataSourceBuilder.create().build();
        return druidProperties.dataSource(dataSource);
    }

    @Bean(name = "dynamicDataSource")
    @Primary
    public DynamicDataSource dataSource(DataSource masterDataSource) {
        Map<Object, Object> targetDataSources = new HashMap<>();
        targetDataSources.put(DataSourceType.MASTER.name(), masterDataSource);
        setDataSource(targetDataSources, DataSourceType.SLAVE.name(), "slaveDataSource");
        return new DynamicDataSource(masterDataSource, targetDataSources);
    }

    /**
     * 设置数据源
     *
     * @param targetDataSources 备选数据源集合
     * @param sourceName        数据源名称
     * @param beanName          bean名称
     */
    public void setDataSource(Map<Object, Object> targetDataSources, String sourceName, String beanName) {
        try {
            DataSource dataSource = SpringContextUtil.getBean(beanName);
            targetDataSources.put(sourceName, dataSource);
        } catch (Exception e) {
            
        }
    }

}
```

数据源注解：

```java
@Target({ ElementType.METHOD, ElementType.TYPE })
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
public @interface DataSource
{
    /**
     * 切换数据源名称
     */
    public DataSourceType value() default DataSourceType.MASTER;
}
```

切面实现：

```java
@Aspect
@Order(-1)
@Component
public class DataSourceAspect
{
    protected Logger logger = LoggerFactory.getLogger(getClass());

    @Pointcut("@annotation(com.zjca.common.database.annotation.DataSource)"
            + "|| @within(com.zjca.common.database.annotation.DataSource)")
    public void dsPointCut()
    {

    }

    @Around("dsPointCut()")
    public Object around(ProceedingJoinPoint point) throws Throwable
    {
        DataSource dataSource = getDataSource(point);

        if (StringUtils.isNotNull(dataSource))
        {
            DynamicDataSourceContextHolder.setDataSourceType(dataSource.value().name());
        }

        try
        {
            return point.proceed();
        }
        finally
        {
            // 销毁数据源 在执行方法之后
            DynamicDataSourceContextHolder.clearDataSourceType();
        }
    }

    /**
     * 获取需要切换的数据源
     */
    public DataSource getDataSource(ProceedingJoinPoint point)
    {
        MethodSignature signature = (MethodSignature) point.getSignature();
        DataSource dataSource = AnnotationUtils.findAnnotation(signature.getMethod(), DataSource.class);
        if (Objects.nonNull(dataSource))
        {
            return dataSource;
        }

        return AnnotationUtils.findAnnotation(signature.getDeclaringType(), DataSource.class);
    }
}
```



OK，一顿排查下来，我确认了配置是没有问题的。

证据：

1.在Druid控制台可以看到数据源有两个，地址和数据库都是yaml里定义的。

2.在切换数据源的时候，切换的日志也正常打印了。



问题出现在DruidConfig.class中，再精确一点是DynamicDataSource出现了问题。



## 问题暴露

在项目启动时，需要由Spring帮我们创建DynamicDataSource，于是在DruidConfig中打断点发现：创建dynamicDataSource时，没有获取到slaveDataSource的实例。

```java
@Bean(name = "dynamicDataSource")
@Primary
public DynamicDataSource dataSource(DataSource masterDataSource) {
    Map<Object, Object> targetDataSources = new HashMap<>();
    targetDataSources.put(DataSourceType.MASTER.name(), masterDataSource);
    //这里没有获取到
    setDataSource(targetDataSources, DataSourceType.SLAVE.name(), "slaveDataSource");
    return new DynamicDataSource(masterDataSource, targetDataSources);
}
```

所以至始至终targetDataSources里面只有一个数据源。所以哪怕切面能够切换到slave数据源，也“查无此人”，切换失败了。

在DynamicDataSource中determineCurrentLookupKey正确变更了也无济于事。因为targetDataSources里面压根就没有。

![](https://raw.githubusercontent.com/CatMaiV/BlogImage/main/img/2025/20250902102615.png)

那么为什么slaveDataSource会获取失败呢？我们跳转到setDataSource方法，通过debug发现确实是获取不到。

![](https://raw.githubusercontent.com/CatMaiV/BlogImage/main/img/2025/20250902103105.png)

这里竟然拿不到？？我的Spring上下文容器失效了？

```java
SpringContextUtil.getBean("slaveDataSource")
```



# ApplicationContext和ConfigurableListableBeanFactory

由于项目中已经有上下文工具了，我就直接用了没有看具体的差别。

本项目的：

```java
@Component
public class SpringContextUtil implements ApplicationContextAware {

    private static ApplicationContext applicationContext;

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        SpringContextUtil.applicationContext = applicationContext;
    }

    
    /**
     * 获取applicationContext
     */
    public static ApplicationContext getApplicationContext() {
        return applicationContext;
    }
    
    
    @SuppressWarnings("unchecked")
    public static <T> T getBean(String name) throws BeansException
    {
        return (T) getApplicationContext().getBean(name);
    }
    
    
}
```

在其他项目中使用的Spring上下文工具：

```java
@Component
public final class SpringUtils implements BeanFactoryPostProcessor, ApplicationContextAware 
{
    /** Spring应用上下文环境 */
    private static ConfigurableListableBeanFactory beanFactory;

    private static ApplicationContext applicationContext;

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException 
    {
        SpringUtils.beanFactory = beanFactory;
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException 
    {
        SpringUtils.applicationContext = applicationContext;
    }
    
    /**
     * 获取对象
     *
     * @param name
     * @return Object 一个以所给名字注册的bean的实例
     * @throws BeansException
     *
     */
    @SuppressWarnings("unchecked")
    public static <T> T getBean(String name) throws BeansException
    {
        return (T) beanFactory.getBean(name);
    }
    
}
```

同样是getBean(String name)的方法，二者获取的渠道不同。





## ApplicationContext

先看定义

```java
public interface ApplicationContext extends EnvironmentCapable, ListableBeanFactory, HierarchicalBeanFactory,
       MessageSource, ApplicationEventPublisher, ResourcePatternResolver {


}
```

继承了很多接口，这里看下官方的注释怎么说的：

> 为应用程序提供配置的中央接口。该接口在应用程序运行时为只读，但如果实现支持，则可以重新加载。
> ApplicationContext 提供：
> 用于访问应用程序组件的 Bean 工厂方法。继承自 ListableBeanFactory。
> 以通用方式加载文件资源的能力。继承自 org.springframework.core.io.ResourceLoader 接口。
> 向已注册的监听器发布事件的能力。继承自 ApplicationEventPublisher 接口。
> 解析消息的能力，支持国际化。继承自 MessageSource 接口。
> 继承自父级上下文。子级上下文中的定义始终优先。例如，这意味着整个 Web 应用程序可以使用单个父级上下文，而每个 Servlet 都有其自己的子级上下文，并且这些子级上下文与其他 Servlet 的上下文无关。
> 除了标准的 org.springframework.beans.factory.BeanFactory 生命周期功能之外，ApplicationContext 实现还可以检测和调用 ApplicationContextAware bean 以及 ResourceLoaderAware、ApplicationEventPublisherAware 和 MessageSourceAware bean。

获取Bean的方法是通过ListableBeanFactory实现的。

```java
public interface ListableBeanFactory extends BeanFactory {
    
}
```

ListableBeanFactory的注释：

> BeanFactory 接口的扩展，由能够枚举所有 Bean 实例的 Bean 工厂实现，而不是像客户端请求那样逐个按名称查找 Bean。预加载所有 Bean 定义的 BeanFactory 实现（例如基于 XML 的工厂）可以实现此接口。
> 如果这是一个 HierarchicalBeanFactory，则返回值将不考虑任何 BeanFactory 层次结构，而仅与当前工厂中定义的 Bean 相关。使用 BeanFactoryUtils 辅助类可以同时考虑祖先工厂中的 Bean。
> 此接口中的方法仅遵循此工厂的 Bean 定义。它们将忽略通过其他方式（例如 org.springframework.beans.factory.config.ConfigurableBeanFactory 的 registerSingleton 方法）注册的任何单例 Bean，但 getBeanNamesForType 和 getBeansOfType 除外，这两个方法也会检查这些手动注册的单例 Bean。当然，BeanFactory 的 getBean 方法也允许透明地访问这些特殊的 Bean。然而，在典型情况下，所有 Bean 都会由外部 Bean 定义定义，因此大多数应用程序无需担心这种区分。
> 注意：除了 getBeanDefinitionCount 和 containsBeanDefinition 之外，此接口中的方法并非设计用于频繁调用。实现速度可能会很慢。



## ConfigurableListableBeanFactory

同样先看定义

```java
public interface ConfigurableListableBeanFactory
       extends ListableBeanFactory, AutowireCapableBeanFactory, ConfigurableBeanFactory {
    
}
```

可以看到继承的接口比ApplicationContext要少，并且在beanFactory这里，双方都继承了ListableBeanFactory这个接口。除此之外，额外继承了AutowireCapableBeanFactory和ConfigurableBeanFactory，这两个工厂都是实例化对象的。

可以简单理解为：

ListableBeanFactory  提供了 getBean的方法

AutowireCapableBeanFactory和ConfigurableBeanFactory还额外提供了createBean的方法。



以下是AutowireCapableBeanFactory的createBean方法说明。

```java
/**
 * Fully create a new bean instance of the given class.
 * <p>Performs full initialization of the bean, including all applicable
 * {@link BeanPostProcessor BeanPostProcessors}.
 * <p>Note: This is intended for creating a fresh instance, populating annotated
 * fields and methods as well as applying all standard bean initialization callbacks.
 * Constructor resolution is based on Kotlin primary / single public / single non-public,
 * with a fallback to the default constructor in ambiguous scenarios, also influenced
 * by {@link SmartInstantiationAwareBeanPostProcessor#determineCandidateConstructors}
 * (e.g. for annotation-driven constructor selection).
 * @param beanClass the class of the bean to create
 * @return the new bean instance
 * @throws BeansException if instantiation or wiring failed
 */
<T> T createBean(Class<T> beanClass) throws BeansException;
```

> 完全创建给定类的新 Bean 实例。
> 执行 Bean 的完整初始化，包括所有适用的 BeanPostProcessors。
> 注意：这旨在创建一个新的实例，填充带注解的字段和方法，以及应用所有标准的 Bean 初始化回调。构造函数解析基于 Kotlin 的 primary / single public / single non-public 原则，在不明确的情况下会回退到默认构造函数，同时也受 SmartInstantiationAwareBeanPostProcessor.determineCandidateConstructors 的影响（例如，用于注解驱动的构造函数选择）。





## 差异解析

SpringBoot 启动过程中，`ApplicationContext` 是分阶段完成初始化的：

1. **创建 `ApplicationContext` 实例**
2. **准备阶段**
   - 加载并解析配置类。
   - 注册 BeanDefinition 到 `BeanFactory`（此时只是“配方”，还没实例化）。
3. **调用 BeanFactoryPostProcessor**
   - 这里能拿到 `ConfigurableListableBeanFactory`，可以操作 BeanDefinition，甚至 `getBean()` 强制触发实例化。
   - 此时大部分 Bean **还没有实例化**。
4. **实例化单例 Bean**
   - 按需/按顺序实例化 Bean，并完成依赖注入、`@PostConstruct` 等回调。
5. **完成 ApplicationContext.refresh()**
   - 这个时候 ApplicationContext 才完全可用，所有非懒加载单例 Bean 都初始化好了。



所以在项目启动过程中，ApplicationContext并没有完全准备好，此时我们尝试获取就会可能获取不到。而在ConfigurableListableBeanFactory中，如果我们获取不到，会触发createBean的相关代码进行创建，所以此时一定会获取到需要的bean。

阶段对比：

| 阶段                         | BeanFactory.getBean() | ApplicationContext.getBean() |
| ---------------------------- | --------------------- | ---------------------------- |
| **BeanDefinition 注册后**    | ✅ 可以触发实例化      | ❌ 获取不到，可能会抛出异常   |
| **BeanFactoryPostProcessor** | ✅ 常用来提前获取/修改 | ❌ 获取不到                   |
| **Bean 实例化完成后**        | ✅ 一直可用            | ✅ 这时才安全可用             |



# 总结

所以在各类文章中提到，ApplicationContext.getBean()通常在程序运行时获取Spring容器中的bean，而ConfigurableListableBeanFactory一般在自定义Starter、框架集成时会用到，此时需要插队或者提前获取某些定义的bean。

| 特性 | ApplicationContext.getBean()                     | ConfigurableListableBeanFactory.getBean() |
| ---- | ------------------------------------------------ | ----------------------------------------- |
| 层级 | 高层容器接口                                     | 底层容器接口                              |
| 面向 | 应用开发者                                       | 框架/扩展开发者                           |
| 功能 | 包含 BeanFactory 功能 + 国际化、事件、资源加载等 | 只专注 Bean 的定义和生命周期管理          |
| 用途 | 日常获取 Bean，业务代码调用                      | 容器扩展、框架级代码操作 BeanDefinition   |
| 关系 | 内部也是委托给 BeanFactory 实现                  | 实际的底层实现类                          |

因此在DruidConfig中，项目初始化时获取slaveDataSource配置需要通过ConfigurableListableBeanFactory获取，此时ApplicationContext还没有完全准备好。















