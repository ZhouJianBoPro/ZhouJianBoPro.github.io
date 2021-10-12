---
layout: post
title: 事务管理
date: 2019-11-11
tags: [spring技术]
---

**事务的特性**

|特性|英文|说明|
|---|---|---|
|原子性|Atomicity|操作要么全部成功要么全部失败,事物在执行过程中发生异常时会回滚到事物开始前的状态|
|一致性|Consistency|无论操作结果是成功还是失败，数据保持一致性|
|隔离性|Isolation|多个事物是互相隔离的，一个事物的操作不影响到另外一个事物的结果|
|持久性|Durability|事物提交后，对数据的修改是永久的|

**事务的传播特性**

|传播特性|行为描述|
|propagation_required|spring模式事务，若没有事务则创建新的事务，存在事务加入到这个事务中|
|propagation_supports|支持当前事务，没有事务时以非事务方法执行|
|propagation_required_new|新建事务，若当前方法存在事务，将事务挂起|
|propagation_not_supports|以非事务方式执行该方法，若该方法存在事务时将事务挂起|
|propagation_never|以非事务方式执行，存在事务抛出异常|
|propagation_mandatory|使用当前事务，若当前方法没有事务抛出异常|
|propagation_nested|若方法存在事务则以嵌套事物方式执行，没有的话就创建新的事物|

**事物的隔离级别**

|英文|描述|并发问题|场景|
|read uncommitted|读未提交（一个事物可以读取另一个事物未提交的数据）|脏读|客户余额为1000元，此时准备发起自动缴费代扣（并未结束），客户在此刻看到余额为1000|
|read committed|读已提交（一个事物要等另一个事物提交后才能读取数据）|不可重复读|客户余额1000，准备去消费时，账户此时缴费代扣成功1000，买单时提示余额不足|
|repeatable read|可重复读（读取数据时不再允许修改数据）|幻读（事物提交之前Insert操作）|客户消费期间不会发生缴费代扣。幻读：客户妻子查询账户账单时发现只有一笔消费记录，但是此时客户又（新增）一笔消费记录，妻子打印账单时发现了两笔记录|
|Serializable|序列化（事物按顺序串行执行）|隔离级别最高，避免发生脏读，不可重复读，幻读|效率较低，极大损耗数据库性能，一般数据库默认事务隔离级别为repeatable read|

**事务的实现方式**

- 编程式事务：在代码中调用beginTransaction(), commit(), rollback()等事务管理方法，弃用。
- 基于TransactionProxyTransaction的声明式事务，配置繁琐，不推荐使用
- 基于@Transactional注解实现声明式事务
- 基于AspectJ AOP配置事务

**基于@Transactional注解实现声明式事务**

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

2.在applicationContext.xml文件中配置事物管理器和事务注解驱动
```xml
    <!-- 事务管理器 -->
    <bean class="org.springframework.jdbc.datasource.DataSourceTransactionManager" id="txManager">
        <property name="dataSource" ref="dataSource"></property>
    </bean>
 
    <!-- 事务注解驱动，标注@Transactional的类和方法将具有事务性 -->
    <tx:annotation-driven transaction-manager="txManager"></tx:annotation-driven>
```

3.给具体业务方法(service层)增加@transactional注解
```html
@Transactional(isolation=Isolation.DEFAULT,propagation=Propagation.REQUIRED,rollbackFor=DefineException.class)

```

4.将业务bean交给spring管理，业务bean配置在xml文件中(直接new出来的对象不会使用到事物)
```xml
<bean class="com.loongshawn.service.TestService" id="testService"></bean>
```


