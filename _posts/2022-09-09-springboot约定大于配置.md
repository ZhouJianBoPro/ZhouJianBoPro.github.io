---
layout: post
title: springboot约定大于配置
date: 2022-09-09
tags: [springboot]
---

#### 什么是springboot
springboot用于简化spring应用的搭建，使用特定的方式进行配置，可以简化开发过程，提升开发效率

#### 约定大于配置理解
1. springboot配置文件默认放在src/main/resource下，application命名的properties，yml，yaml配置文件
2. 可以通过spring.profile.active指定运行环境所需的配置
2. 当要集成第三方组件时，添加相关的starter依赖，然后按照约定进行配置即可

#### springboot的优点
1. 独立运行：springboot内嵌了servlet、tomcat等容器，是一个可以独立运行的jar包，无需将war包部署到tomcat
2. 快速集成第三方组件：只需要添加组件的starter maven依赖，再添加约定的配置即可快速集成组件，无需繁琐的xml配置
3. 简化maven配置：比如使用spring或springmvc需要添加大量的依赖，spring-boot-web-starter包含了springmvc相关依赖及内置的tomcat容器
4. 自动配置spring bean：使用JavaConfig方式自动配置bean。比如需要将一个普通的类让spring管理，我们可以用@Configuration和@Bean注解完成，无需使用@Service或xml方式

#### springboot核心注解
启动类的@SpringBootApplication是springboot核心注解，由以下注解组成
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
public @interface SpringBootApplication {}
```
- @SpringBootConfiguration：是springboot对@Configuration的封装，作用等同于@Configuration，标明这是个配置类
- @EnableAutoConfiguration：自动配置注解，也可以关闭某个自动配置的选项，比如多数据源场景下需要关闭数据源的自动配置
```java
@SpringBootApplication(exclude = {DataSourceAutoConfiguration.class})
```
- @ComponentScan：定义的扫描路径，把符合规则的类装配到spring容器中，这些类将被spring管理
```java
@SpringBootApplication(scanBasePackages = {"cn.com.tsfa.fpm.persist", "cn.com.tsfa.batch"})
```