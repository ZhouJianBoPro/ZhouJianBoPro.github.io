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
1. 创建BootstrapContext（引导上下文）
2. 加载应用启动监听器
3. 发布应用启动事件
4. [创建并准备环境](#prepareEnvironment)
5. 打印banner
6. [创建并准备应用上下文](#prepareContext)
7. [刷新应用上下文](#refreshContext)
8. 发布应用上下文启动完成事件
9. 执行所有[自定义Runner](#callRunners)
10. 发布应用准备就绪事件
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
            // 6.1 创建应用上下文，通过ApplicationContextFactory创建
            context = this.createApplicationContext();
            context.setApplicationStartup(this.applicationStartup);
            // 6.2 创建并准备应用上下文
            this.prepareContext(bootstrapContext, context, environment, listeners, applicationArguments, printedBanner);
            // 7. 刷新应用上下文
            this.refreshContext(context);
            this.afterRefresh(context, applicationArguments);
            Duration timeTakenToStartup = Duration.ofNanos(System.nanoTime() - startTime);
            if (this.logStartupInfo) {
                (new StartupInfoLogger(this.mainApplicationClass)).logStarted(this.getApplicationLog(), timeTakenToStartup);
            }

            // 8. 发布应用上下文启动完成事件，事件中包含了应用启动事件
            listeners.started(context, timeTakenToStartup);
            // 9. 运行所有的自定义Runner，自定义Bean实现ApplicationRunner#run或CommandLineRunner#run
            this.callRunners(context, applicationArguments);
        } catch (Throwable var12) {
            this.handleRunFailure(context, var12, listeners);
            throw new IllegalStateException(var12);
        }

        try {
            // 10. 发布应用准备就绪事件，如dubbo中的服务提供者向注册中心完成注册服务是在该阶段
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
5. 初始化[事件广播器](#eventMulticaster)
6. 注册事件监听器
7. 完成[Bean的初始化](#finishBeanFactoryInitialization)
8. 完成上下文刷新，发布上下文刷新完成事件
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
                // 4. 注册Bean后置处理器：从BeanFactory获取所有的实现了BeanPostProcessor的Bean定义，并注册到BeanFactory中。区分内部/外部处理器，而且外部处理器有顺序优先级
                this.registerBeanPostProcessors(beanFactory);
                beanPostProcess.end();
                // MessageSource：用于处理国际化，多语言，多国语言等场景，默认使用ResourceBundleMessageSource
                this.initMessageSource();
                // 5. 初始化事件广播器。默认为SimpleApplicationEventMulticaster，用于广播事件，观察者模式实现事件广播和监听
                this.initApplicationEventMulticaster();
                this.onRefresh();
                // 6. 注册事件监听器，事件监听器实现了ApplicationListener接口，用于监听事件广播器发送的事件
                this.registerListeners();
                // 7. 完成Bean的初始化，实例化所有非懒加载的Bean（会触发完整Bean的生命周期：实例化 -> 属性填充 -> BeanPostProcessor前置处理 -> 初始化方法 (@PostConstruct, InitializingBean) -> BeanPostProcessor后置处理）
                this.finishBeanFactoryInitialization(beanFactory);
                // 8. 完成上下文刷新，发布上下文已经刷新的事件
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
- 获取实现了BeanPostProcessor接口的Bean定义
- Bean后置处理器分类：[内部处理器实现方式](#internalPostProcessor)，[外部处理器实现方式](#outBeanPostProcessor)
- 优先级排序（外部处理器）：PriorityOrdered > Ordered > Non-Ordered
```java
public static void registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory, AbstractApplicationContext applicationContext) {
        // 1. 从BeanFactory中获取所有实现了BeanPostProcessor接口的Bean名称
        String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);
        // 2 定义内部Bean后置处理器集合（优先级高于所有的外部处理器）
        List<BeanPostProcessor> internalPostProcessors = new ArrayList();
        // 3.1 定义优先级最高的外部Bean后置处理器集合
        List<BeanPostProcessor> priorityOrderedPostProcessors = new ArrayList();
        // 3.2 定义实现了Ordered接口的外部处理器集合（优先级在外部处理器中第二）
        List<String> orderedPostProcessorNames = new ArrayList();
        // 3.3 定义没有实现Ordered接口的外部处理器集合（优先级最低）
        List<String> nonOrderedPostProcessorNames = new ArrayList();
        String[] var8 = postProcessorNames;
        int var9 = postProcessorNames.length;

        String ppName;
        BeanPostProcessor pp;
        for(int var10 = 0; var10 < var9; ++var10) {
            ppName = var8[var10];
            if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
                // 4.1 处理实现了 PriorityOrdered 接口的 BeanPostProcessor
                pp = (BeanPostProcessor)beanFactory.getBean(ppName, BeanPostProcessor.class);
                priorityOrderedPostProcessors.add(pp);
                if (pp instanceof MergedBeanDefinitionPostProcessor) {
                    internalPostProcessors.add(pp);
                }
            } else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
                // 4.2 处理实现了 Ordered 接口的 BeanPostProcessor
                orderedPostProcessorNames.add(ppName);
            } else {
                // 4.3 处理没有实现 Ordered 接口的 BeanPostProcessor
                nonOrderedPostProcessorNames.add(ppName);
            }
        }

        // 5. 首先注册优先级最高的外部Bean后缀处理器（按照getOrder中设置的值排序）
        sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
        registerBeanPostProcessors(beanFactory, (List)priorityOrderedPostProcessors);
        
        // 6. 然后注册实现了Ordered接口的Bean处理器，并且按照getOrder中设置的值排序
        List<BeanPostProcessor> orderedPostProcessors = new ArrayList(orderedPostProcessorNames.size());
        Iterator var14 = orderedPostProcessorNames.iterator();
        while(var14.hasNext()) {
            String ppName = (String)var14.next();
            BeanPostProcessor pp = (BeanPostProcessor)beanFactory.getBean(ppName, BeanPostProcessor.class);
            orderedPostProcessors.add(pp);
            if (pp instanceof MergedBeanDefinitionPostProcessor) {
                // 当处理器实现了MergedBeanDefinitionPostProcessor时，被认为内部处理器
                internalPostProcessors.add(pp);
            }
        }
        sortPostProcessors(orderedPostProcessors, beanFactory);
        registerBeanPostProcessors(beanFactory, (List)orderedPostProcessors);

        // 7. 注册优先级最低的Bean后置处理器
        List<BeanPostProcessor> nonOrderedPostProcessors = new ArrayList(nonOrderedPostProcessorNames.size());
        Iterator var17 = nonOrderedPostProcessorNames.iterator();
        while(var17.hasNext()) {
            ppName = (String)var17.next();
            pp = (BeanPostProcessor)beanFactory.getBean(ppName, BeanPostProcessor.class);
            nonOrderedPostProcessors.add(pp);
            // 当处理器实现了MergedBeanDefinitionPostProcessor时，被认为内部处理器
            if (pp instanceof MergedBeanDefinitionPostProcessor) {
                internalPostProcessors.add(pp);
            }
        }
        registerBeanPostProcessors(beanFactory, (List)nonOrderedPostProcessors);
        
        // 8. 注册内部Bean后置处理器
        sortPostProcessors(internalPostProcessors, beanFactory);
        registerBeanPostProcessors(beanFactory, (List)internalPostProcessors);
        beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(applicationContext));
    }
```

#### 外部Bean后置处理器实现方式 {#outBeanPostProcessor}
```java
/**
 * 只实现BeanPostProcessor接口，优先级最低
 */
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

/**
 * 分别实现BeanPostProcessor和Ordered，优先级比只实现BeanPostProcessor后置处理器要高。且重写了Ordered的getOrder方法，值越小，优先级越高
 */
@Component
public class MyCustomOrderedBeanPostProcessor implements BeanPostProcessor, Ordered {

    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("MyCustomOrderedBeanPostProcessor.postProcessBeforeInitialization()");
        return bean;
    }

    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("MyCustomOrderedBeanPostProcessor.postProcessAfterInitialization()");
        return bean;
    }

    /**
     * 重写Ordered接口的getOrder方法，值越小，优先级越高
     * @return
     */
    @Override
    public int getOrder() {
        return 0;
    }
}

/**
 * 分别实现BeanPostProcessor和PriorityOrdered，优先级最高。且重写了Ordered的getOrder方法，值越小，优先级越高
 */
@Component
public class MyCustomPriorityOrderedBeanPostProcessor implements BeanPostProcessor, PriorityOrdered {

    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("MyCustomPriorityOrderedBeanPostProcessor.postProcessBeforeInitialization()");
        return bean;
    }

    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("MyCustomPriorityOrderedBeanPostProcessor.postProcessAfterInitialization()");
        return bean;
    }

    /**
     * 重写Ordered接口的getOrder方法，值越小，优先级越高
     * @return
     */
    @Override
    public int getOrder() {
        return 0;
    }
}
```

#### 内部Bean后置处理器实现方式 {#internalPostProcessor}
需要实现MergedBeanDefinitionPostProcessor接口，内部Bean后置处理器执行顺序在外部Bean后置处理器之前。并且支持更细粒度的操作，可以拿到RootBeanDefinition
```java
@Component
public class MyCustomMergedBeanDefinitionPostProcessor implements MergedBeanDefinitionPostProcessor {
    @Override
    public void postProcessMergedBeanDefinition(RootBeanDefinition rootBeanDefinition, Class<?> aClass, String s) {
        System.out.println("MyCustomMergedBeanDefinitionPostProcessor.postProcessMergedBeanDefinition");
    }

    @Override
    public void resetBeanDefinition(String beanName) {
        MergedBeanDefinitionPostProcessor.super.resetBeanDefinition(beanName);
    }
}
```

#### 事件广播器实现方式 {#eventMulticaster}
1. 创建自定义事件
    ```java
    public class CustomApplicationEvent extends ApplicationEvent {
    
        private String message;
    
        public CustomApplicationEvent(Object source, String message) {
            super(source);
            this.message = message;
        }
    
        public String getMessage() {
            return message;
        }
    }
    ```
2. 创建事件监听器
    ```java
    @Component
    public class MyCustomApplicationListener implements ApplicationListener<CustomApplicationEvent> {
        @Override
        public void onApplicationEvent(CustomApplicationEvent event) {
            System.out.println("收到事件：" + event.getMessage());
        }
    }
    ```
3. 发布事件（ApplicationEventPublisher#publishEvent）
    ```java
    @Component
    public class EventPublishService {
    
        @Autowired
        private ApplicationEventPublisher applicationEventPublisher;
    
        public void publishEvent() {
            applicationEventPublisher.publishEvent(new CustomApplicationEvent(this, "Hello, Spring event!"));
        }
    }
    ```
   
#### 完成Bean初始化过程 {#finishBeanFactoryInitialization}
简化流程：getBean() -> doGetBean() -> createBean() -> doCreateBean()，重点来分析doCreateBean方法
1. bean实例化
2. 执行bean的内部处理器
3. 将提前暴露bean的工厂方法放入到三级缓存中
4. [bean属性填充](#populateBean)
5. [初始化bean](#initializeBean)
6. [解决循环依赖](#cirularDependency)
7. [注册销毁逻辑的bean](#disposableBean)
```java
protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) throws BeanCreationException {
        BeanWrapper instanceWrapper = null;
        if (mbd.isSingleton()) {
           // 某些情况下Bean定义可能发生变更，移除的目的是为了保证Bean实例是最新的
            instanceWrapper = (BeanWrapper)this.factoryBeanInstanceCache.remove(beanName);
        }

        if (instanceWrapper == null) {
            // 1. 创建bean实例。默认情况下，spring容器会使用无参构造方法实例化。若没有无参构造方法，则会选择有参构造方法（有多个有参构造方法时需要指定使用哪个，可以通过@Autowired注解指定，否则spring不知道使用哪个会抛出异常）
            instanceWrapper = this.createBeanInstance(beanName, mbd, args);
        }

        Object bean = instanceWrapper.getWrappedInstance();
        Class<?> beanType = instanceWrapper.getWrappedClass();
        if (beanType != NullBean.class) {
            mbd.resolvedTargetType = beanType;
        }

        synchronized(mbd.postProcessingLock) {
            // 检查是否已经执行过Bean内部处理器
            if (!mbd.postProcessed) {
                try {
                    // 2. 执行Bean内部处理器
                    this.applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
                } catch (Throwable var17) {
                    throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Post-processing of merged bean definition failed", var17);
                }

                mbd.postProcessed = true;
            }
        }

        // 3. 允许循环时（spring.main.allow-circular-references: true），将正在创建的单例bean放入到三级缓存中，三级缓存是用于生成提前暴露bean的工厂方法（lambda表达式）
        boolean earlySingletonExposure = mbd.isSingleton() && this.allowCircularReferences && this.isSingletonCurrentlyInCreation(beanName);
        if (earlySingletonExposure) {
            if (this.logger.isTraceEnabled()) {
                this.logger.trace("Eagerly caching bean '" + beanName + "' to allow for resolving potential circular references");
            }
            
            this.addSingletonFactory(beanName, () -> {
                return this.getEarlyBeanReference(beanName, mbd, bean);
            });
        }

        Object exposedObject = bean;

        try {
            // 4. 属性填充
            this.populateBean(beanName, mbd, instanceWrapper);
            // 5. 初始化bean
            exposedObject = this.initializeBean(beanName, exposedObject, mbd);
        } catch (Throwable var18) {
            if (var18 instanceof BeanCreationException && beanName.equals(((BeanCreationException)var18).getBeanName())) {
                throw (BeanCreationException)var18;
            }

            throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Initialization of bean failed", var18);
        }

        // 6. 解决循环依赖，从三级缓存中获取bean
        if (earlySingletonExposure) {
            Object earlySingletonReference = this.getSingleton(beanName, false);
            if (earlySingletonReference != null) {
                if (exposedObject == bean) {
                    exposedObject = earlySingletonReference;
                } else if (!this.allowRawInjectionDespiteWrapping && this.hasDependentBean(beanName)) {
                    String[] dependentBeans = this.getDependentBeans(beanName);
                    Set<String> actualDependentBeans = new LinkedHashSet(dependentBeans.length);
                    String[] var12 = dependentBeans;
                    int var13 = dependentBeans.length;

                    for(int var14 = 0; var14 < var13; ++var14) {
                        String dependentBean = var12[var14];
                        if (!this.removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
                            actualDependentBeans.add(dependentBean);
                        }
                    }

                    if (!actualDependentBeans.isEmpty()) {
                        throw new BeanCurrentlyInCreationException(beanName, "Bean with name '" + beanName + "' has been injected into other beans [" + StringUtils.collectionToCommaDelimitedString(actualDependentBeans) + "] in its raw version as part of a circular reference, but has eventually been wrapped. This means that said other beans do not use the final version of the bean. This is often the result of over-eager type matching - consider using 'getBeanNamesForType' with the 'allowEagerInit' flag turned off, for example.");
                    }
                }
            }
        }

        try {
            // 7. 注册实现了销毁逻辑的bean。在bean初始化完成后，spring会检查该bean在容器关闭时是否需要销毁，并注册到销毁队列中。当容器关闭时，spring会按顺序依次执行这些bean的销毁逻辑
            this.registerDisposableBeanIfNecessary(beanName, bean, mbd);
            return exposedObject;
        } catch (BeanDefinitionValidationException var16) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Invalid destruction signature", var16);
        }
}
```

#### bean属性填充过程 {#populateBean}
1. 获取BeanDefinition定义的属性
2. 执行[注解驱动的依赖注入](#InstantiationAwareBeanPostProcessor)
3. 进行依赖检查
4. 将属性应用在Bean实例中
```java
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {

      // 1. 获取BeanDefinition定义的属性
      PropertyValues pvs = mbd.hasPropertyValues() ? mbd.getPropertyValues() : null;

      // 2. 处理注解驱动的依赖注入；用于处理@Value，@Autowired，@Resource等注解
      boolean hasInstAwareBpps = this.hasInstantiationAwareBeanPostProcessors();
      boolean needsDepCheck = mbd.getDependencyCheck() != 0;
      PropertyDescriptor[] filteredPds = null;
      if (hasInstAwareBpps) {
         if (pvs == null) {
            pvs = mbd.getPropertyValues();
         }

         PropertyValues pvsToUse;
         for(Iterator var9 = this.getBeanPostProcessorCache().instantiationAware.iterator(); var9.hasNext(); pvs = pvsToUse) {
            InstantiationAwareBeanPostProcessor bp = (InstantiationAwareBeanPostProcessor)var9.next();
            // 这里会处理@Autowired、@Value，@Resource等注解
            pvsToUse = bp.postProcessProperties((PropertyValues)pvs, bw.getWrappedInstance(), beanName);
         }
      }

      // 3. 进行依赖检查，检查Bean的所有依赖项是否可用
      if (needsDepCheck) {
         if (filteredPds == null) {
            filteredPds = this.filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
         }

         this.checkDependencies(beanName, mbd, filteredPds, (PropertyValues)pvs);
      }

      // 4. 将属性应用在Bean实例中
      if (pvs != null) {
         this.applyPropertyValues(beanName, mbd, bw, (PropertyValues)pvs);
      }
   }
```

#### 注解驱动的依赖注入 {#InstantiationAwareBeanPostProcessor}
- AutowiredAnnotationBeanPostProcessor：处理@Autowired和@Value注解
- CommonAnnotationBeanPostProcessor：处理@Resource，及生命周期注解（@PostConstruct和@PreDestroy）

#### 初始化Bean {#initializeBean}
1. 初始化前先处理Aware回调
2. 执行Bean后置处理器初始化前方法
3. 调用初始化方法
4. 执行Bean后置处理器初始化后方法
```java
protected Object initializeBean(String beanName, Object bean, @Nullable RootBeanDefinition mbd) {

        // 1. 处理Aware回调（3种），BeanNameAware（可以获取到beanName）, BeanClassLoaderAware（可以获取到CLassLoader），BeanFactoryAware（可以获取到BeanFactory）
        invokeAwareMethods(beanName, bean);

		Object wrappedBean = bean;
		if (mbd == null || !mbd.isSynthetic()) {
            // 2. 执行Bean后置处理器初始化前方法，实现了BeanPostProcessor#postProcessBeforeInitialization的bean，这里同时也会处理 @PostConstruct（由 CommonAnnotationBeanPostProcessor 触发）
			wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
		}

		try {
            // 3. 调用初始化方法，实现了InitializingBean#afterPropertiesSet接口的bean
			invokeInitMethods(beanName, wrappedBean, mbd);
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					(mbd != null ? mbd.getResourceDescription() : null),
					beanName, "Invocation of init method failed", ex);
		}
		if (mbd == null || !mbd.isSynthetic()) {
            // 4. 执行Bean后置处理器的初始化后方法，实现了BeanPostProcessor#postProcessAfterInitialization的bean
			wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
		}

		return wrappedBean;
}
```

#### 解决循环依赖 {#cirularDependency}
1. 先从一级缓存中获取bean
2. 再尝试从二级缓存中获取提前暴露的bean
3. 前面两级缓存都没有获取到时，从第三级缓存中获取bean的工厂方法
4. 从工厂方法中获取到提前暴露的bean，放入二级缓存中，并从三级缓存移除
```java
@Nullable
	protected Object getSingleton(String beanName, boolean allowEarlyReference) {
        // 1. 从单例池中获取bean（一级缓存）
		Object singletonObject = this.singletonObjects.get(beanName);
		if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
            // 2. 一级缓存中不存在bean时，从二级缓存中获取普通对象/代理对象
			singletonObject = this.earlySingletonObjects.get(beanName);
			if (singletonObject == null && allowEarlyReference) {
				synchronized (this.singletonObjects) {
					singletonObject = this.singletonObjects.get(beanName);
					if (singletonObject == null) {
						singletonObject = this.earlySingletonObjects.get(beanName);
						if (singletonObject == null) {
                            // 3. 二级缓存中不存在bean时，从三级缓存中获取bean的工厂方法
							ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
							if (singletonFactory != null) {
                                // 4. 从工厂方法中获取到提前暴露的bean，放入二级缓存中，并从三级缓存移除
								singletonObject = singletonFactory.getObject();
								this.earlySingletonObjects.put(beanName, singletonObject);
								this.singletonFactories.remove(beanName);
							}
						}
					}
				}
			}
		}
		return singletonObject;
	}
```

#### 销毁逻辑bean的实现方式 {#disposableBean}
1. Bean 的方法标注了 @PreDestroy 注解
   ```java
   @Component
   public class AService {
       @PreDestroy
          public void destroy() {
              System.out.println("AService destroy");
      }
   }
   ```
2. Bean实现了DisposableBean接口
   ```java
   @Component
   public class AService implements DisposableBean {
   
       @Override
       public void destroy() throws Exception {
           System.out.println("AService destroy");
       }
   }
   ```
3. Bean的destroy-method属性配置了方法名
   ```java
   public class CService {
      public void destroy() {
         System.out.println("CService destroy");
      }
   }
   
   @Configuration
   public class AppConfig {
      @Bean(destroyMethod = "destroy")
      public CService cService() {
         return new CService();
      }
   }
   ```
   
#### 自定义Runners {#callRunners}
- 实现ApplicationRunner#run
   ```java
   @Component
   public class CustomApplicationRunner implements ApplicationRunner {
       @Override
       public void run(ApplicationArguments args) throws Exception {
           System.out.println("执行自定义ApplicationRunner, args = " + args);
       }
   }
   ```
- 实现CommandLineRunner#run
   ```java
   @Component
   public class CustomCommandLineRunner implements CommandLineRunner {
       @Override
       public void run(String... args) throws Exception {
           System.out.println("自定义CustomCommandLineRunner， args = " + args);
       }
   }
   ```

