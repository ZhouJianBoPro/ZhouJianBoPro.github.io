---
layout: post
title: Spring事物
date: 2019-11-11
tags: [Spring]
description: spring事物管理
---

**事物的特性**

|特性|英文|说明|
|---|---|---|
|原子性|Atomicity|操作要么全部成功要么全部失败|
|一致性|Consistency|无论操作结果是成功还是失败，数据保持一致性|
|隔离性|Isolation|多个事物是互相隔离的，一个事物的操作不影响到另外一个事物的结果|
|持久性|Durability|事物提交后，对数据的修改是永久的|

**spring及mybatis中事物管理配置**

1.配置数据源(DriverManager)和会话工厂(SessionFactory)
```xml
<bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource" init-method="init" destroy-method="close">
        <!-- 基本属性 url、user、password -->
        <property name="url" value="${jd.jdbc.url}"/>
        <property name="username" value="${jd.jdbc.username}"/>
        <property name="password" value="${jd.jdbc.password}"/>
        <!-- 其他配置省略 -->
</bean>

<!-- sessionFactory -->
<bean id="sessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
	 <property name="configLocation" value="classpath:mybatis-config.xml"></property>
	<property name="dataSource" ref="dataSource"/>
</bean>
```

1.在applicationContext.xml文件中配置事物管理器和事物注解驱动
```xml
    <!-- 事务管理器 -->
    <bean class="org.springframework.jdbc.datasource.DataSourceTransactionManager" id="txManager">
        <property name="dataSource" ref="dataSource"></property>
    </bean>
 
    <!-- 事务注解驱动，标注@Transactional的类和方法将具有事务性 -->
    <tx:annotation-driven transaction-manager="txManager"></tx:annotation-driven>
```

2.给具体业务方法(service层)增加@transactional注解<br/>

3.将业务bean交给spring管理，业务bean配置在xml文件中(直接new出来的对象不会使用到事物)
```xml
<bean class="com.loongshawn.service.TestService" id="testService"></bean>
```

**事物的属性**

|特性|英文|
|---|---|
|REQUIRED|默认属性，当前方法有事物加入该事物中，没有的话创建新的事物|
|SUPPORTS|当前方法有事物加入到该事物中，没有的话以非事物执行|
|REQUIRES_NEW|无论当前方法是否有事物都会创建新事物|
|NOT_SUPPORTED|当前方法有事物将事物挂起|
|MANDATORY|不存在事物抛出异常|
|NEVER|存在事物抛出异常|
|NESTED|存在事物在嵌套事物内执行，没有事物创建新的事物|


