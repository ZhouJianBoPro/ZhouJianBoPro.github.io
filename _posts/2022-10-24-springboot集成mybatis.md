---
layout: post
title: springboot集成mybatis
date: 2022-10-24
tags: [mybatis]
---

#### 引入mybatis的springboot启动器
```xml
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>1.3.2</version>
</dependency>
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>${mysql.version}</version>
</dependency>
```
#### springboot配置文件配置数据源及xml路径
```yaml
# 配置xml路径
mybatis:
  type-aliases-package: cn.com.pro.provider.db.model
  mapper-locations: classpath:mapper/*.xml

# 配置数据源
spring:
  datasource:
    druid:
      url: jdbc:mysql://127.0.0.1:3306/test?seUnicode=true&characterEncoding=utf-8&userSSL=false&serverTimezone=GMT%2B8
      driver-class-name: com.mysql.jdbc.Driver
      username: admin
      password: 123456
      initial-size: 10
      min-idle: 10
      max-active: 50
      max-wait: 60000
      time-between-eviction-runs-millis: 60000
      min-evictable-idle-time-millis: 300000
      test-while-idle: true
      test-on-borrow: false
      test-on-return: false
      pool-prepared-statements: true
      filters: stat,wall,log4j
      max-pool-prepared-statement-per-connection-size: 20
      connection-properties: druid.stat.mergeSql=true;druid.stat.slowSqlMillis=5000
      use-global-data-source-stat: true
      stat-view-servlet:
        login-username: admin
        login-password: admin
        reset-enable: false
        url-pattern: /druid/*
      web-stat-filter:
        url-pattern: /*
        exclusions: "*.js,*.gif,*.jpg,*.png,*.css,*.ico,/druid/*"
```
#### 配置Mapper接口扫描
> 1. springboot启动类使用@MapperScan注解配置Mapper接口扫描路径
> 2. 使用@Mapper接口标注Mapper接口类
> 3. 若同时使用@MapperScan与@Mapper接口，@Mapper作用将失效

#### mybatis-spring-boot-starter启动器
```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
<dependency>
  <groupId>org.mybatis.spring.boot</groupId>
  <artifactId>mybatis-spring-boot-autoconfigure</artifactId>
</dependency>
<dependency>
  <groupId>org.mybatis</groupId>
  <artifactId>mybatis</artifactId>
</dependency>
<dependency>
  <groupId>org.mybatis</groupId>
  <artifactId>mybatis-spring</artifactId>
</dependency>
```
> mybatis的spring启动器依赖了mybatis、spring-mybatis、及自动配置器mybatis-spring-boot-autoconfigure

#### mybatis-spring-boot-autoconfigure自动装配
> 自动装配器包的META-INF目录中有一个spring.factories文件，根据springboot的spi机制，springboot会主动扫描spring.factories
中配置的EnableAutoConfiguration的子类，并将该子类中通过JavaConfig方式配置的bean自动装配进spring容器

