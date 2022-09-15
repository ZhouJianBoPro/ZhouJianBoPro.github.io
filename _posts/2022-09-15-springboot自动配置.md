---
layout: post
title: springboot自动配置
date: 2022-09-15
tags: [springboot]
---

#### 自动配置原理
1. springboot启动时先找到main方法，如果有多个main方法，需要在pom.xml指定启动类的全路径，否则会报错
2. springboot在启动时会通过@EnableAutoConfiguration注解找到依赖第三方组件jar包中的/META-INF/spring.factories文件
3. 该文件配置了需要自动配置的类路径，且该类是以AutoConfiguration结尾来命名，springboot对该自动配置类进行加载
4. 该类实际上是以AutoConfiguration结尾来命名的，实际上是以JavaConfig方式配置的Spring Bean
5. 该类通过@EnableConfigurationProperties注解绑定了配置类。该配置类是以Properties结尾命名，实现了配置类属性与配置文件的绑定

#### 自动配置实现(mybatis为例)
- pom.xml中增加mybatis自动配置依赖
    ```xml
    <dependency>
        <groupId>org.mybatis.spring.boot</groupId>
        <artifactId>mybatis-spring-boot-autoconfigure</artifactId>
        <version>1.3.2</version>
    </dependency>
    ```
- application.yml配置mybatis相关属性
    ```properties
    mybatis.mapper-locations=classpath*:mybatis/mapper/*.xml
    ```
- mybatis自动配置依赖对应jar包中/META-INF/spring.factories中配置文件
    ```properties
    # Auto Configure
    org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
    org.mybatis.spring.boot.autoconfigure.MybatisAutoConfiguration
    ```
- 自动配置类MybatisAutoConfiguration
    ```java
    @org.springframework.context.annotation.Configuration
    @ConditionalOnClass({ SqlSessionFactory.class, SqlSessionFactoryBean.class })
    @ConditionalOnBean(DataSource.class)
    /*配置类，与配置文件绑定*/
    @EnableConfigurationProperties(MybatisProperties.class)
    @AutoConfigureAfter(DataSourceAutoConfiguration.class)
    public class MybatisAutoConfiguration {
    
      private static final Logger logger = LoggerFactory.getLogger(MybatisAutoConfiguration.class);
    
      private final MybatisProperties properties;
    
      private final Interceptor[] interceptors;
    
      private final ResourceLoader resourceLoader;
    
      private final DatabaseIdProvider databaseIdProvider;
    
      private final List<ConfigurationCustomizer> configurationCustomizers;
    }
    ```
- 配置类MybatisProperties
    ```java
    @ConfigurationProperties(prefix = MybatisProperties.MYBATIS_PREFIX)
    public class MybatisProperties {
    
      public static final String MYBATIS_PREFIX = "mybatis";
    
      /**
       * Locations of MyBatis mapper files.
       */
      private String[] mapperLocations;
    
      public String[] getMapperLocations() {
        return this.mapperLocations;
      }
    
      public void setMapperLocations(String[] mapperLocations) {
        this.mapperLocations = mapperLocations;
      }
    }
    ```

