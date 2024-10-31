---
layout: post
title: SpringBootApplication注解解析
date: 2024-10-30
tags: [springboot]
---

#### @SpringBootApplication注解
```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(
        excludeFilters = {@Filter(
                type = FilterType.CUSTOM,
                classes = {TypeExcludeFilter.class}
        ), @Filter(
                type = FilterType.CUSTOM,
                classes = {AutoConfigurationExcludeFilter.class}
        )}
)
public @interface SpringBootApplication {
}
```
@SpringBootApplication是一个复合注解，它同时包含@SpringBootConfiguration, @EnableAutoConfiguration, @ComponentScan注解

#### @SpringBootConfiguration
```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Configuration
@Indexed
public @interface SpringBootConfiguration {
    @AliasFor(
        annotation = Configuration.class
    )
    boolean proxyBeanMethods() default true;
}
```
@SpringBootConfiguration本质上是@Configuraion注解，它的作用是标识当前类是一个配置类，可以在这个类中通过@Bean注解自定义一些Bean。
@Configuration 中有一个属性proxyBeanMethods, 默认为true，用于保证该bean的单例

#### @EnableAutoConfiguration
```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import({AutoConfigurationImportSelector.class})
public @interface EnableAutoConfiguration {
    String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";

    Class<?>[] exclude() default {};

    String[] excludeName() default {};
}
```
开启自动配置功能注解，内部使用了@Import注解导入了AutoConfigurationImportSelector类，该类负责收集和选择
需要自动配置的类。springboot通过META-INF/spring.factories文件来指定需要自动配置的类

#### @ComponentScan
```java
@ComponentScan(
    excludeFilters = {@Filter(
    type = FilterType.CUSTOM,
    classes = {TypeExcludeFilter.class}
), @Filter(
    type = FilterType.CUSTOM,
    classes = {AutoConfigurationExcludeFilter.class}
)}
)
public @interface SpringBootApplication {
    
}
```
定义spring容器扫描路径，默认为当前类所在的包。@ComponentScan注解默认会排除掉一些类：
1. 通过@Configuration注解标注的类，并且该类在spring.factories中的EnableAutoConfiguration属性对应values中存在。因为spring.factories中配置的类本身就会被spring注册，所以@Configuration注解的类不需要重复注册
2. 自定义排除器，继承TypeExcludeFilter类并重写方法。需要注意的是，自定义的排除器需要在spring初始化器中注册，不能直接用@Component注解标记，执行包路径扫描是在bean生命周期之前。
```java
public class MyTypeExcludeFilter extends TypeExcludeFilter {
    @Override
    public boolean match(MetadataReader metadataReader, MetadataReaderFactory metadataReaderFactory) throws IOException {
        return metadataReader.getClassMetadata().getClassName().equals(UserService.class.getName());
    }
}


public class MyApplicationContextInitializer implements ApplicationContextInitializer {
    @Override
    public void initialize(ConfigurableApplicationContext applicationContext) {
        applicationContext.getBeanFactory().registerSingleton("myTypeExcludeFilter", new MyTypeExcludeFilter());
    }
}
```

spring.factories中定义初始化器
```properties
org.springframework.context.ApplicationContextInitializer=com.proj.exclude.MyApplicationContextInitializer
```

