---
layout: post
title: SpringBoot监听器
date: 2024-11-01
tags: [springboot]
---

#### SpringBoot中监听器
SpringBoot中的监听器使用了观察者模式，在spring容器各个阶段发布不同的事件。
1. springboot在初始化SpringApplication时会从spring.factories获取默认的观察者列表，这些观察者对象是ApplicationListener的实现类。
2. springboot在启动前会从spring.factories获取SpringApplicationRunListener的实例对象EventPublishingRunListener，SpringApplicationRunListener定义了不同阶段的事件。
3. SpringApplicationRunListener和其实现类EventPublishingRunListener认作被观察者， 其中SpringApplicationRunListener是抽象主题， 该EventPublishingRunListener是具体主题。
4. EventPublishingRunListener被观察者在初始化时会将所有的观察者对象添加到观察者列表中。因此，ApplicationListener可以认作为观察者抽象类，而其实现类为具体的观察者，定义了接受事件的处理逻辑
5. ApplicationListener实现类包含EnvironmentPostProcessorApplicationListener, LoggingApplicationListener等，默认有8个

#### SpringApplicationRunListener主题接口（被观察者抽象接口）
```java
public interface SpringApplicationRunListener {
    
    // 开始启动事件
    default void starting(ConfigurableBootstrapContext bootstrapContext) {
    }

    // 环境配置准备完成事件
    default void environmentPrepared(ConfigurableBootstrapContext bootstrapContext, ConfigurableEnvironment environment) {
    }

    // 上下文准备完成事件
    default void contextPrepared(ConfigurableApplicationContext context) {
    }

    // 上下文加载完成事件
    default void contextLoaded(ConfigurableApplicationContext context) {
    }

    // 容器启动完成事件
    default void started(ConfigurableApplicationContext context, Duration timeTaken) {
        this.started(context);
    }
    
    // 容器启动失败事件
    default void failed(ConfigurableApplicationContext context, Throwable exception) {
    }
}
```

#### EventPublishingRunListener具体主题类（被观察者）
```java
public class EventPublishingRunListener implements SpringApplicationRunListener, Ordered {
    private final SpringApplication application;
    private final String[] args;
    // 定义了观察者实例列表，实现了ApplicationListener抽象观察者接口
    private final SimpleApplicationEventMulticaster initialMulticaster;

    public EventPublishingRunListener(SpringApplication application, String[] args) {
        this.application = application;
        this.args = args;
        this.initialMulticaster = new SimpleApplicationEventMulticaster();

        // 获取spring.factories中定义的监听器（观察者）
        Iterator var3 = application.getListeners().iterator();

        // 将监听器放入观察者列表中
        while(var3.hasNext()) {
            ApplicationListener<?> listener = (ApplicationListener)var3.next();
            this.initialMulticaster.addApplicationListener(listener);
        }

    }

    public int getOrder() {
        return 0;
    }

    // 容器开始启动事件
    public void starting(ConfigurableBootstrapContext bootstrapContext) {
        this.initialMulticaster.multicastEvent(new ApplicationStartingEvent(bootstrapContext, this.application, this.args));
    }

    // 环境配置准备完成事件
    public void environmentPrepared(ConfigurableBootstrapContext bootstrapContext, ConfigurableEnvironment environment) {
        this.initialMulticaster.multicastEvent(new ApplicationEnvironmentPreparedEvent(bootstrapContext, this.application, this.args, environment));
    }
}
```

#### ApplicationListener观察者抽象接口
```java
@FunctionalInterface
public interface ApplicationListener<E extends ApplicationEvent> extends EventListener {
    
    // 定义事件处理逻辑方法
    void onApplicationEvent(E event);

    static <T> ApplicationListener<PayloadApplicationEvent<T>> forPayload(Consumer<T> consumer) {
        return (event) -> {
            consumer.accept(event.getPayload());
        };
    }
}
```
