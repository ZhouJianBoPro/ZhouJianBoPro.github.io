---
layout: post
title: BeanPostProcessor后置处理器
date: 2022-09-16
tags: [spring]
---

#### BeanPostProcessor作用
常用于在bean初始化前后对bean进行修改。
如业务系统之间的调用可以用挡板来模拟返回报文，是否启动挡板可以通过配置来决定，
通常会提供两个bean用于区分挡板环境与真实环境，这俩个bean实现同一接口，使用时注入businessBean。
此时，我们可以通过BeanPostProcessor在启动挡板时将businessBean替换为挡板mockBusinessBean

#### BeanPostProcessor实现示例
```java
@Component
public class MockBeanPostProcess implements BeanPostProcessor, ApplicationContextAware {

    private ApplicationContext applicationContext;

    @Value("@[baffle.switch]")
    private boolean baffleSwitch;

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {

        if ("businessService".equals(beanName) && baffleSwitch) {
            return applicationContext.getBean("mockBusinessService");
        }
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        return bean;
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }
}

```

