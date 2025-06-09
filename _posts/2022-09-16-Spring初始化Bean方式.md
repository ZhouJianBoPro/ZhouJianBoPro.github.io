---
layout: post
title: Spring初始化Bean方式
date: 2022-09-16
tags: [spring]
---

#### spring初始化bean的三种方式
1. @PostConstruct方法注解
2. 实现InitializingBean接口重写afterPropertiesSet方法
3. init-method

#### 三种方式的执行顺序
@PostConstruct > InitializingBean > init-method

#### @PostConstruct注解
当bean被spring容器初始化后会调用@PostConstruct注解标注的方法
```java
@Component
public class ExampleBean {
    @PostConstruct    
    public void init(){
        //开始初始化证书...
    }  
}
```

#### init-method
基于xml配置的spring bean，可以在bean标签内添加init-method属性初始化bean
```xml
<bean id="defaultMQProducer" class="org.apache.rocketmq.client.producer.DefaultMQProducer" init-method="start" destroy-method="shutdown">
        <property name="producerGroup" value=""/>
        <property name="namesrvAddr" value=""/>
        <property name="vipChannelEnabled" value="false"/>
        <property name="retryTimesWhenSendFailed" value="3"/>
        <property name="instanceName" value="defaultMQProducer"/>
    </bean>
```

#### 实现InitializingBean接口
实现InitializingBean接口，并重写afterPropertiesSet方法
```java
public class XxlJobAdminConfig implements InitializingBean{

    private static XxlJobAdminConfig adminConfig = null;

    public static XxlJobAdminConfig getAdminConfig() {
        return adminConfig;
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        adminConfig = this;
    }
}
```
