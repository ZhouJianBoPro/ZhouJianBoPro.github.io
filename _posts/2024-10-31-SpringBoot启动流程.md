---
layout: post
title: SpringBoot启动流程
date: 2024-10-31
tags: [springboot]
---

#### SpringApplication对象初始化
```java
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
        this.sources = new LinkedHashSet();
        this.bannerMode = Mode.CONSOLE;
        this.logStartupInfo = true;
        this.addCommandLineProperties = true;
        this.addConversionService = true;
        this.headless = true;
        this.registerShutdownHook = true;
        this.additionalProfiles = Collections.emptySet();
        this.isCustomEnvironment = false;
        this.lazyInitialization = false;
        this.applicationContextFactory = ApplicationContextFactory.DEFAULT;
        this.applicationStartup = ApplicationStartup.DEFAULT;
        this.resourceLoader = resourceLoader;
        Assert.notNull(primarySources, "PrimarySources must not be null");
        this.primarySources = new LinkedHashSet(Arrays.asList(primarySources));
        // 1. 推断应用类型
        this.webApplicationType = WebApplicationType.deduceFromClasspath();
        this.bootstrapRegistryInitializers = new ArrayList(this.getSpringFactoriesInstances(BootstrapRegistryInitializer.class));
        // 2. 获取spring.factories中的ApplicationContextInitializer对象，为初始化器
        this.setInitializers(this.getSpringFactoriesInstances(ApplicationContextInitializer.class));
        // 3. 获取spring.factories中的ApplicationListener对象
        this.setListeners(this.getSpringFactoriesInstances(ApplicationListener.class));
        // 4.推断main方法所在的类
        this.mainApplicationClass = this.deduceMainApplicationClass();
}
```
1. 实例化SpringApplication对象
2. 推断应用类型，通过classpath中的类名判断（DispatcherServlet， DispatcherHandler）：NONE为普通springboot应用，SERVLET为spring web应用
3. 从classpath:META-INF/spring.factories或获取ApplicationContextInitializer实例对象（初始化器），默认有7个初始化器
4. 从spring.factories中获取ApplicationListener实例对象（监听器），默认有8个监听器
5. 推断main方法运行类，通过遍历线程栈方法，找到main方法所在的类

#### 执行run方法
```java
public ConfigurableApplicationContext run(String... args) {
        long startTime = System.nanoTime();
        DefaultBootstrapContext bootstrapContext = this.createBootstrapContext();
        
        // 定义spring容器
        ConfigurableApplicationContext context = null;
        
        // 不会创建图形界面相关的资源
        this.configureHeadlessProperty();
        
        // 1. 从spring.factories中获取SpringApplicationRunnerListener监听器EventPublishingRunListener，它会在容器各个阶段发布对应事件
        SpringApplicationRunListeners listeners = this.getRunListeners(args);
        // 2. 发布容器开始启动的事件
        listeners.starting(bootstrapContext, this.mainApplicationClass);

        try {
            ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
            // 3. 加载环境配置，包括系统属性和配置文件
            ConfigurableEnvironment environment = this.prepareEnvironment(listeners, bootstrapContext, applicationArguments);
            this.configureIgnoreBeanInfo(environment);
            
            // 4. spring容器启动时打印banner
            Banner printedBanner = this.printBanner(environment);
            
            // 5. 根据应用类型创建容器
            context = this.createApplicationContext();
            context.setApplicationStartup(this.applicationStartup);
            
            // 6. 容器启动前处理：给容器设置环境变量，使用ApplicationContextInitializer初始化容器
            this.prepareContext(bootstrapContext, context, environment, listeners, applicationArguments, printedBanner);
            
            // 7. 启动容器：解析自动配置类并生成对应的bean对象，启动webServer，默认为tomcat
            this.refreshContext(context);
            
            // 8. 容器启动后处理：执行ApplicationRunner和CommandLineRunner
            this.afterRefresh(context, applicationArguments);
            Duration timeTakenToStartup = Duration.ofNanos(System.nanoTime() - startTime);
            if (this.logStartupInfo) {
                (new StartupInfoLogger(this.mainApplicationClass)).logStarted(this.getApplicationLog(), timeTakenToStartup);
            }

            // 9. 发布容器启动成功的事件
            listeners.started(context, timeTakenToStartup);
            this.callRunners(context, applicationArguments);
        } catch (Throwable var12) {
            this.handleRunFailure(context, var12, listeners);
            throw new IllegalStateException(var12);
        }

        try {
            Duration timeTakenToReady = Duration.ofNanos(System.nanoTime() - startTime);
            listeners.ready(context, timeTakenToReady);
            return context;
        } catch (Throwable var11) {
            this.handleRunFailure(context, var11, (SpringApplicationRunListeners)null);
            throw new IllegalStateException(var11);
        }
}
```
1. 从spring.factories中获取SpringApplicationRunListener实例对象，默认只有一个EventPublishingRunListener（事件发布监听器）对象，它会在容器各个阶段发布对应的事件
2. 执行runnerListener.starting方法，发布应用开始启动的事件
3. 加载环境的应用配置，包括系统属性，环境变量和应用配置文件
4. 根据应用类型创建spring容器ConfigurableApplicationContext
5. spring容器启动前处理：把环境信息设置给spring容器；使用ApplicationContextInitializer初始化容器等 
6. 启动spring容器：refreshContext，加载所有的自动配置类并创建bean对象，启动webServer（默认是tomcat）
7. 发布容器启动成功事件
