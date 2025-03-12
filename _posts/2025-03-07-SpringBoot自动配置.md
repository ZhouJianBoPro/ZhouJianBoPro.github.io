---
layout: post
title: SpringBoot自动配置
date: 2025-03-07
tags: [springboot]
---

#### 什么是自动配置
springboot可以基于自动配置特性来快速集成组件，只需要引入相关依赖，进行简单的配置或无需配置即可集成。相比于spring无需繁杂的配置及定义相关的bean.

#### 自动配置核心概念
- 约定大于配置：springboot提供了默认的配置，开发者只需少量配置或无需配置即可集成组件，比如spring-boot-starter-web, 集成了web应用所需的各种依赖和配置。
- 条件化配置：使用条件化注解如（@ConditionalOnClass，@ConditionalOnMissBean）来决定是否应用某个配置类或bean
- 自动配置类：自动配置是带有@Configuration的类，位于META-INF/spring.factories文件中，文件中的key为EnableAutoConfiguration类的全限定名，value为自动配置类的全限定名(XXXAutoConfiguration)
    ```properties
    org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
    org.redisson.spring.starter.RedissonAutoConfiguration
    ```
- 配置属性：通过@ConfigurationProperties将配置文件属性绑定到Java对象

#### 自动配置类条件注解（是否加载配置类或bean）
- @ConditionalOnClass：当classpath下存在指定的类时，才加载配置类
- @ConditionalOnMissingClass: 当classpath下不存在指定的类时，才加载配置类
- @ConditionalOnBean：当容器中存在指定的bean时，才加载配置类
- @ConditionalOnMissingBean：当容器中不存在指定的bean时，才加载配置类
- @ConditionalOnProperty：当指定的属性存在时，才加载配置类 @ConditionalOnProperty(name = "my.feature.enabled", havingValue = "true")

#### 自动配置类
- @Configuration：标记该类为一个配置类，Spring 会将其作为一个 Bean 定义的来源。
- @Conditional及其派生注解：条件注解用于控制自动配置类是否生效
- @AutoConfigureAfter 和 @AutoConfigureBefore：控制自动配置类的加载顺序
- @EnableConfigurationProperties：将外部配置（如 application.properties 或 application.yml）绑定到 Java 对象，并注册为 Spring Bean
    ```java
    @Configuration
    @EnableConfigurationProperties(DataSourceProperties.class)
    public class DataSourceAutoConfiguration {
        // 自动配置类的定义
    }
    ```
- 自动配置类命名规范：配置类名+AutoConfiguration，例如DataSourceAutoConfiguration

#### 自动配置工作原理
1. @SpringBootApplication注解包含了@EnableAutoConfiguration注解，这是自动配置的核心注解
2. @EnableAutoConfiguration注解导入了AutoConfigurationImportSelector类，@Import(AutoConfigurationImportSelector.class）
3. AutoConfigurationImportSelector实现了DeferredImportSelector接口（用于延迟加载自动配置类，因为自动配置类中有条件注解，如@ConditionalOnBean，需要让其他bean先加载） 
4. AutoConfigurationImportSelector#getAutoConfigurationEntry方法加载自动配置类
```java
//AnnotationMetadata注解元数据指的是springboot启动类
protected AutoConfigurationEntry getAutoConfigurationEntry(AnnotationMetadata annotationMetadata) {
        if (!this.isEnabled(annotationMetadata)) {
            return EMPTY_ENTRY;
        } else {
            // 1. 获取自动配置注解属性，exclude = {}, excludeName = {}, 有可能需要排除一些自动配置类
            AnnotationAttributes attributes = this.getAttributes(annotationMetadata);
            // 2. 通过SpringFactoriesLoader加载META-INF/spring.factories文件中的配置类（仅获取自动配置类）
            List<String> configurations = this.getCandidateConfigurations(annotationMetadata, attributes);
            // 3. 对扫描到的自动配置类去重
            configurations = this.removeDuplicates(configurations);
            // 4. 过滤掉在注解中排除的自动配置类
            Set<String> exclusions = this.getExclusions(annotationMetadata, attributes);
            this.checkExcludedClasses(configurations, exclusions);
            configurations.removeAll(exclusions);

            // 5. 过滤（条件注解的判断在此阶段进行），AutoConfigurationImportFilter#mach 通过条件注解判断是否加载自动配置类
            configurations = this.getConfigurationClassFilter().filter(configurations);
            
            // 6. 触发自动配置导入事件
            this.fireAutoConfigurationImportEvents(configurations, exclusions);
            return new AutoConfigurationEntry(configurations, exclusions);
        }
}

// 获取候选的自动配置类
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
    // 通过spring spi中的SpringFactoriesLoader加载META-INF/spring.factories文件中的自动配置类，key为EnableAutoConfiguration类全限定名
   List<String> configurations = new ArrayList(SpringFactoriesLoader.loadFactoryNames(this.getSpringFactoriesLoaderFactoryClass(), this.getBeanClassLoader()));
   ImportCandidates.load(AutoConfiguration.class, this.getBeanClassLoader()).forEach(configurations::add);
   Assert.notEmpty(configurations, "No auto configuration classes found in META-INF/spring.factories nor in META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports. If you are using a custom packaging, make sure that file is correct.");
   return configurations;
}

// 获取SpringFactoriesLoaderFactoryClass， 用于过滤出spring.factories中key为EnableAutoConfiguration类全限定名的自动配置类
protected Class<?> getSpringFactoriesLoaderFactoryClass() {
    return EnableAutoConfiguration.class;
}

// 获取自动配置注解属性，exclude = {}, excludeName = {}
protected AnnotationAttributes getAttributes(AnnotationMetadata metadata) {
    // 获取@EnableAutoConfiguration 注解属性exclude = {}, excludeName = {}
   String name = this.getAnnotationClass().getName();
   AnnotationAttributes attributes = AnnotationAttributes.fromMap(metadata.getAnnotationAttributes(name, true));
   Assert.notNull(attributes, () -> {
      return "No auto-configuration attributes found. Is " + metadata.getClassName() + " annotated with " + ClassUtils.getShortName(name) + "?";
   });
   return attributes;
}

protected Class<?> getAnnotationClass() {
   return EnableAutoConfiguration.class;
}
```

