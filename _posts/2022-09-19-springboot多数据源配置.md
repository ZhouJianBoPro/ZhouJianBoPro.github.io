---
layout: post
title: springboot多数据源配置
date: 2022-09-19
tags: [springboot]
---

#### 实现方式
基于springboot自动配置特性，引入dynamic-datasource-spring-boot-starter，再结合内部的@DS注解，可快速实现数据源的切换，
该注解可作用于类和方法中，方法上的注解优于类上的注解，当并不使用@DS注解时，使用默认数据源，默认数据源配置可在配置文件中指定
```properties
spring.datasource.dynamic.primary=DataSource1
```

#### 实现过程
- 引入dynamic-datasource-spring-boot-starter依赖
    ```xml
    <dependency>
      <groupId>com.baomidou</groupId>
      <artifactId>dynamic-datasource-spring-boot-starter</artifactId>
      <version>3.1.1</version>
    </dependency>
    ```
- 引入druid-spring-boot-starter连接池依赖
    ```xml
    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>druid-spring-boot-starter</artifactId>
        <version>1.1.9</version>
    </dependency>
    ```
- 配置数据源DataSource1、DataSource2及全局连接池
    ```properties
    # 设置默认数据源
    spring.datasource.dynamic.primary=DataSource1
    
    # druid全局连接池配置
    spring.datasource.dynamic.druid.filters=stat
    spring.datasource.dynamic.druid.max-active=50
    spring.datasource.dynamic.druid.initial-size=10
    spring.datasource.dynamic.druid.max-wait=60000
    spring.datasource.dynamic.druid.min-idle=10
    spring.datasource.dynamic.druid.time-between-eviction-runs-millis=60000
    spring.datasource.dynamic.druid.min-evictable-idle-time-millis=300000
    spring.datasource.dynamic.druid.test-while-idle=true
    spring.datasource.dynamic.druid.test-on-borrow=false
    spring.datasource.dynamic.druid.test-on-return=false
    spring.datasource.dynamic.druid.pool-prepared-statements=true
    spring.datasource.dynamic.druid.max-pool-prepared-statement-per-connection-size=20
    
    # DataSource1配置
    spring.datasource.dynamic.datasource.DataSource1.url=
    spring.datasource.dynamic.datasource.DataSource1.username=
    spring.datasource.dynamic.datasource.DataSource1.password=
    spring.datasource.dynamic.datasource.DataSource1.driver-class-name=com.mysql.cj.jdbc.Driver
    
    # DataSource2配置
    spring.datasource.dynamic.datasource.DataSource2.url=
    spring.datasource.dynamic.datasource.DataSource2.username=
    spring.datasource.dynamic.datasource.DataSource2.password=
    spring.datasource.dynamic.datasource.DataSource2.driver-class-name=com.mysql.cj.jdbc.Driver
    ```
- springboot启动类禁用数据源连接池自动配置
    ```java
    @SpringBootApplication(exclude = DruidDataSourceAutoConfigure.class)
    ```
- 在数据库操作类Mapper或Service中的类或方法上添加@DS注解，不加时选择默认数据源
    ```java
    @DS("DataSource2")
    ```
#### @DS注解失效场景
1. A、B方法都添加了@DS注解，此时B方法被A方法调用，此时B方法的@DS失效
2. 当一个事务中使用到不同的数据源时会导致数据源无法切换
3. Mapper中存在@Select等注解时@DS会失效
