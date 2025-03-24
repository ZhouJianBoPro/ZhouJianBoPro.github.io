---
layout: post
title: SpringBoot启动流程及原理
date: 2025-03-20
tags: [springboot]
---

#### SpringApplication初始化
```java
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
        this.sources = new LinkedHashSet();
        this.bannerMode = Mode.CONSOLE;
        // 应用启动时输出日志：SpringBoot版本，Java版本，Web应用类型等
        this.logStartupInfo = true;
        // 设置为true时会将命令行参数添加到spring环境中
        this.addCommandLineProperties = true;
        // 类型转换及数据绑定，如在处理 REST 请求时，将 URL 参数或请求体中的数据绑定到 Java 对象
        this.addConversionService = true;
        // 无头模式，会仅用所有与图形相关功能，如AWT和Swing，从而减少内存占用
        this.headless = true;
        // 在JVM关闭或中断时，优雅的关闭spring应用上下文，如关闭数据库连接池，线程池等
        this.registerShutdownHook = true;
        // 用于指定额外的Spring Profile，可以指定多个
        this.additionalProfiles = Collections.emptySet();
        // 使用springboot默认提供的Environment
        this.isCustomEnvironment = false;
        // 控制Spring应用上下文是否以懒加载方式启动，默认情况下springboot在启动时尽可能多初始化Bean，以便提前发现潜在的问题
        this.lazyInitialization = false;
        this.applicationContextFactory = ApplicationContextFactory.DEFAULT;
        // 启动性能监控，帮助开发者了解springboot应用在启动过程中各个阶段的时间消耗和顺序执行，可以通过自定义实现ApplicationStartup接口，并设置到SpringApplication中，以获取更详细的启动信息
        this.applicationStartup = ApplicationStartup.DEFAULT;
        this.resourceLoader = resourceLoader;
        Assert.notNull(primarySources, "PrimarySources must not be null");
        this.primarySources = new LinkedHashSet(Arrays.asList(primarySources));
        // 推断应用类型， NONE, SERVLET, REACTIVE，通过类路径判断
        this.webApplicationType = WebApplicationType.deduceFromClasspath();
        // 对引导注册表进行初始化，可以在应用上下文初始化前进行一些提前配置，比如注册基本Bean
        this.bootstrapRegistryInitializers = new ArrayList(this.getSpringFactoriesInstances(BootstrapRegistryInitializer.class));
        // 从spring.factories中获取应用上下文初始化器，也可以自定义：1. 通过springApplication.addInitializers；2. 在/META-INF/spring.factories中配置自定义的初始化器
        this.setInitializers(this.getSpringFactoriesInstances(ApplicationContextInitializer.class));
        // 从spring.factories中获取事件监听器，属于观察者角色；而SpringApplicationRunListener属于被观察者角色。也可以自定义：1. 通过springApplication.addListeners；2. 在/META-INF/spring.factories中配置自定义的监听器
        this.setListeners(this.getSpringFactoriesInstances(ApplicationListener.class));
        // 推断主类，从当前线程栈中获取包含main方法的类
        this.mainApplicationClass = this.deduceMainApplicationClass();
}

static WebApplicationType deduceFromClasspath() {
    // 类路径中存在DispatcherHandler，并且不存在DispatcherServlet和ServletContainer
    if (ClassUtils.isPresent("org.springframework.web.reactive.DispatcherHandler", (ClassLoader)null) && !ClassUtils.isPresent("org.springframework.web.servlet.DispatcherServlet", (ClassLoader)null) && !ClassUtils.isPresent("org.glassfish.jersey.servlet.ServletContainer", (ClassLoader)null)) {
        return REACTIVE;
    } else {
        String[] var0 = SERVLET_INDICATOR_CLASSES;
        int var1 = var0.length;
        for(int var2 = 0; var2 < var1; ++var2) {
            String className = var0[var2];
            // 类路径中不存在javax.servlet.Servlet和org.springframework.web.context.ConfigurableWebApplicationContext
            if (!ClassUtils.isPresent(className, (ClassLoader)null)) {
                return NONE;
            }
        }
        // 默认为SERVLET
        return SERVLET;
    }
}
```


#### 运行 SpringApplication
1. 创建BootstrapContext（引导上下文
2. 加载应用启动监听器
3. 发布应用启动事件
4. [创建并准备环境](#prepareEnvironment)
5. 打印banner
6. [创建并准备应用上下文](#prepareContext)
7. [刷新应用上下文](#refreshContext)
```java
// 启动类中main方法中的args是用于接受命令行参数，命令行配置参数将会覆盖其他相同的配置
public ConfigurableApplicationContext run(String... args) {
        long startTime = System.nanoTime();
        // 1. 创建引导上下文，用于在应用启动前提供一些基础支持
        DefaultBootstrapContext bootstrapContext = this.createBootstrapContext();
        ConfigurableApplicationContext context = null;
        this.configureHeadlessProperty();
        // 2. 从spring.factories中获取应用启动监听器（被观察者）EventPublishingRunListener，它的主要作用是在spring boot启动的不同阶段发布对应的事件
        SpringApplicationRunListeners listeners = this.getRunListeners(args);
        // 3. 被观察者向所有的观察者发布应用开始启动事件，LoggingApplicationListener（日志观察者）订阅到事件后记录日志
        listeners.starting(bootstrapContext, this.mainApplicationClass);

        try {
            ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
            
            // 4. 创建并准备应用环境
            ConfigurableEnvironment environment = this.prepareEnvironment(listeners, bootstrapContext, applicationArguments);
            this.configureIgnoreBeanInfo(environment);
            // 5. 打印banner
            Banner printedBanner = this.printBanner(environment);
            // 6. 创建应用上下文，通过ApplicationContextFactory创建
            context = this.createApplicationContext();
            context.setApplicationStartup(this.applicationStartup);
            // 7. 创建并准备应用上下文
            this.prepareContext(bootstrapContext, context, environment, listeners, applicationArguments, printedBanner);
            // 8. 刷新应用上下文
            this.refreshContext(context);
            this.afterRefresh(context, applicationArguments);
            Duration timeTakenToStartup = Duration.ofNanos(System.nanoTime() - startTime);
            if (this.logStartupInfo) {
                (new StartupInfoLogger(this.mainApplicationClass)).logStarted(this.getApplicationLog(), timeTakenToStartup);
            }

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


#### 环境准备过程 {#prepareEnvironment}
1. 创建Environment对象
2. 配置Environment
3. 发布环境准备事件
4. 将SpringBoot配置源添加到Environment中
5. 将默认配置属性源移到最后面
6. spring.main配置绑定
```java
private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners, DefaultBootstrapContext bootstrapContext, ApplicationArguments applicationArguments) {
        // 1. 创建Environment对象，根据应用类型创建相应的Environment实例对象，会包含环境变量属性源和Java系统属性源
        ConfigurableEnvironment environment = this.getOrCreateEnvironment();
        // 2. 配置Environment，先加载默认属性，然后再加载命令行参数
        this.configureEnvironment((ConfigurableEnvironment)environment, applicationArguments.getSourceArgs());
        // 3. 将SpringBoot的配置属性源添加到Environment对象中
        ConfigurationPropertySources.attach((Environment)environment);
        // 4. 事件发布监听器（被观察者）发布环境准备事件，被观察者（ConfigFileApplicationListener）订阅到事件后加载application.properties等配置文件
        listeners.environmentPrepared(bootstrapContext, (ConfigurableEnvironment)environment);
        // 5. 将默认配置属性源移到最后面，优先级最低
        DefaultPropertiesPropertySource.moveToEnd((ConfigurableEnvironment)environment);
        Assert.state(!((ConfigurableEnvironment)environment).containsProperty("spring.main.environment-prefix"), "Environment prefix cannot be set via properties.");
        // 6. 将环境中spring.main开头的配置与SpringApplication对象中的字段进行绑定，如spring.main.banner-mode（bannerMode）
        this.bindToSpringApplication((ConfigurableEnvironment)environment);
        if (!this.isCustomEnvironment) {
            EnvironmentConverter environmentConverter = new EnvironmentConverter(this.getClassLoader());
            environment = environmentConverter.convertEnvironmentIfNecessary((ConfigurableEnvironment)environment, this.deduceEnvironmentClass());
        }

        ConfigurationPropertySources.attach((Environment)environment);
        return (ConfigurableEnvironment)environment;
    }
```


##### SpringBoot 配置文件优先级规则（从低到高，后加载的配置会覆盖前面加载的配置）
1. 类路径根目录：classpath:/
2. 类路径下的/conf目录：classpath:/conf/
3. 当前目录：file:./
4. 当前目录下的/conf目录：file:./conf/


##### SpringBoot 配置优先级规则（从低到高）
1. 默认属性：通过 SpringApplication.setDefaultProperties 设置的属性
2. 配置文件：如application.properties
3. 操作系统属性和Java系统属性
4. 命令行参数


#### 应用上下文准备过程 {#prepareContext}
1. 应用上下文后置处理器
2. 执行加载到的初始化器
3. 发布上下文准备事件
4. 关闭BootstrapContext
5. 获取并配置BeanFactory
6. 加载启动类
7. 发布应用上下文刷新事件
```java
private void prepareContext(DefaultBootstrapContext bootstrapContext, ConfigurableApplicationContext context, ConfigurableEnvironment environment, SpringApplicationRunListeners listeners, ApplicationArguments applicationArguments, Banner printedBanner) {
        context.setEnvironment(environment);
        // 1. 应用上下文后置处理器，对应用上下文进行自定义扩展，如自定义BeanNameGenerator，在创建SpringApplication对象中设置
        this.postProcessApplicationContext(context);
        // 2. 执行所有的初始化器，如ContextIdApplicationContextInitializer（为ApplicationContext设置一个唯一的上下文ID），ConfigurationWarningsApplicationContextInitializer（检测和报告配置可能出现的问题）等
        this.applyInitializers(context);
        // 3. 发布应用上下文准备事件，如LoggingApplicationListener消费到事件后会记录日志
        listeners.contextPrepared(context);
        // 4. 关闭BootstrapContext，并释放相关资源，防止内存泄露
        bootstrapContext.close(context);
        if (this.logStartupInfo) {
            this.logStartupInfo(context.getParent() == null);
            this.logStartupProfileInfo(context);
        }

        // 5. BeanFactory是Spring容器的核心接口，负责管理bean的生命周期，依赖注入
        ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
        // 将命令行参数对象注册到spring容器中，spring中的所有bean都是单例的
        beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
        // 将Banner对象注册到spring容器中
        if (printedBanner != null) {
            beanFactory.registerSingleton("springBootBanner", printedBanner);
        }

        if (beanFactory instanceof AbstractAutowireCapableBeanFactory) {
            // 允许循环引用
            ((AbstractAutowireCapableBeanFactory)beanFactory).setAllowCircularReferences(this.allowCircularReferences);
            // 允许覆盖bean定义
            if (beanFactory instanceof DefaultListableBeanFactory) {
                ((DefaultListableBeanFactory)beanFactory).setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
            }
        }
        // springboot默认为非懒加载方式加载bean，以便提前发现潜在的问题
        if (this.lazyInitialization) {
            context.addBeanFactoryPostProcessor(new LazyInitializationBeanFactoryPostProcessor());
        }
        context.addBeanFactoryPostProcessor(new PropertySourceOrderingBeanFactoryPostProcessor(context));
        // source为启动类
        Set<Object> sources = this.getAllSources();
        Assert.notEmpty(sources, "Sources must not be empty");
        // 6. 加载启动类，将启动类中的@ComponentScan包扫描路径下的类注册为Bean
        this.load(context, sources.toArray(new Object[0]));
        // 7. 发布应用上下文加载事件
        listeners.contextLoaded(context);
}
```



#### 应用上下文刷新过程 {#refreshContext}
1. [准备刷新上下文](#prepareRefresh)
2. 获取并进一步配置BeanFactory
3. 执行[BeanFactory后置处理器](#BeanFactoryPostProcessor)
4. 注册[Bean后置处理器](#BeanPostProcessor)
```java
public void refresh() throws BeansException, IllegalStateException {
        synchronized(this.startupShutdownMonitor) {
            // 启动性能监控，它可以帮助开发者了解应用启动每个关键步骤的耗时情况
            StartupStep contextRefresh = this.applicationStartup.start("spring.context.refresh");
            // 1. 准备刷新上下文，用于加载自定义属性源及校验非空属性配置
            this.prepareRefresh();
            // 2.1. 准备应用上下文阶段配置的BeanFactory
            ConfigurableListableBeanFactory beanFactory = this.obtainFreshBeanFactory();
            // 2.2. 准备BeanFactory，对BeanFactory进行进一步配置：1.BeanFactory在自动装配过程中会忽略一些属性依赖注入（如忽略ApplicationContextAware）；2.注册一些默认的bean（如environment）, 以便在其他地方能够依赖注入这些bean
            this.prepareBeanFactory(beanFactory);
            try {
                // 3.1 BeanFactory后置处理器，允许在BeanFactory配置完成后，但在Bean实例化前，对BeanFactory进行自定义配置或扩展。可以通过实现BeanFactoryPostProcessor接口并重写postProcessBeanFactory方法
                this.postProcessBeanFactory(beanFactory);
                StartupStep beanPostProcess = this.applicationStartup.start("spring.context.beans.post-process");
                // 3.2 执行BeanFactory后置处理器
                this.invokeBeanFactoryPostProcessors(beanFactory);
                // 5. 注册Bean后置处理器
                this.registerBeanPostProcessors(beanFactory);
                beanPostProcess.end();
                this.initMessageSource();
                this.initApplicationEventMulticaster();
                this.onRefresh();
                this.registerListeners();
                this.finishBeanFactoryInitialization(beanFactory);
                this.finishRefresh();
            } catch (BeansException var10) {
                if (this.logger.isWarnEnabled()) {
                    this.logger.warn("Exception encountered during context initialization - cancelling refresh attempt: " + var10);
                }
                this.destroyBeans();
                this.cancelRefresh(var10);
                throw var10;
            } finally {
                this.resetCommonCaches();
                contextRefresh.end();
            }
        }
}
```



#### 准备刷新上下文 {#prepareRefresh}
```java
protected void prepareRefresh() {
    // 获取容器启动时间
    this.startupDate = System.currentTimeMillis();
    // 设置容器状态为激活
    this.closed.set(false);
    this.active.set(true);
    // 加载属性源，默认无实现，可自定义子类实现
    this.initPropertySources();
    // 验证必填属性：哪些属性被设置为必填（@Value，@ConfigurationProperties 标注的属性）
    this.getEnvironment().validateRequiredProperties();
    if (this.earlyApplicationListeners == null) {
        this.earlyApplicationListeners = new LinkedHashSet(this.applicationListeners);
    } else {
        this.applicationListeners.clear();
        this.applicationListeners.addAll(this.earlyApplicationListeners);
    }

    this.earlyApplicationEvents = new LinkedHashSet();
}
```



#### BeanFactory后置处理器 {#BeanFactoryPostProcessor}
```java
@Component
public class MyCustomBeanFactoryPostProcessor implements BeanFactoryPostProcessor {
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory configurableListableBeanFactory) throws BeansException {
        // 注册一个userService2de bean
        configurableListableBeanFactory.registerSingleton("userService2", new UserService());
    }
}
```


#### Bean后置处理器 {#BeanPostProcessor}
```java
@Component
public class MyCustomBeanPostProcessor implements BeanPostProcessor {

    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("MyCustomBeanPostProcessor.postProcessBeforeInitialization()");
        return bean;
    }

    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("MyCustomBeanPostProcessor.postProcessAfterInitialization()");
        return bean;
    }
}
```

