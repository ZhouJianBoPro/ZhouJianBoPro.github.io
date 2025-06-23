---
layout: post
title: spring整合mybatis底层原理
date: 2024-10-24
tags: [spring]
---

#### BeanFactoryPostProcessor的作用
> 在Spring容器加载了Spring内部的一些基础设施类BeanDefinition（尚未实例化任何Bean）时，允许修改或新添加BeanFactory中的BeanDefinition，
> 这个阶段非常适合用来做自动扫描，如ConfigurationClassPostProcessor解析@ComponentScan扫描出的类
> MapperScannerConfigurer扫描Mybatis Spring接口并注册为BeanDefinition

#### ConfigurationClassPostProcessor作用
> ConfigurationClassPostProcessor实现了BeanFactoryPostProcessor接口，在容器刷新阶段refresh() -> invokeBeanFactoryPostProcessor()中被调用，
> 它主要作用是处理标注了@Configuration、@ComponentScan、@Import、@Bean等注解的配置类，
> 并扫描出这些类所定义的BeanDefinition。它是Spring容器启动过程中解析配置类的核心文件
- 解析@Configuration注解的类：将@Bean注解的方法解析为BeanDefinition
- 处理@ComponentScan注解的类：将路径下的@Component、@Service、@Repository、@Controller注解的类解析为BeanDefinition
- 处理@Import注解：将@Import注解的类解析为BeanDefinition

#### Spring集成Mapper过程
> 1. SpringBoot启动时，refresh -> invokeBeanFactoryPostProcessors，执行所有的BeanFactoryPostProcessor
> 2. 先执行ConfigurationClassPostProcessor，处理@Import({MapperScannerRegistrar.class})，会将@MapperScan中设置的属性设置到MapperScannerConfigurer中
> 3. 然后在执行MapperScannerConfigurer进行Mapper接口的扫描，并注册到BeanFactory中的BeanDefinition（类型设置为MapperFactoryBean）
> 4. refresh -> finishBeanFactoryInitialization（Bean初始化过程中），会执行MapperFactoryBean#getObject方法，生成Mapper接口的代理对象（SqlSession.getMapper(mapperInterface)），并返回给spring容器

#### MapperFactoryBean
> Mybatis的Mapper接口没有实现类，不能直接通过构造函数实例化，需要通过动态代理生成代理类。
> 实现FactoryBean接口，重写getObject方法，实现自定义bean的创建逻辑。
> MapperFactoryBean中注入了mybatis的SqlSession，用于创建Mapper接口的代理对象（底层是通过MapperProxyFactory中的MapperProxy）
```java
public class MapperFactoryBean implements FactoryBean {

    // mapper接口
    private Class mapperClass;

    // mybatis中的SqlSession，用于生成mapper接口的代理对象
    private SqlSession sqlSession;

    // 构造方法注入mapper
    public MapperFactoryBean(Class mapperClass) {
        this.mapperClass = mapperClass;
    }

    // set注入sqlSession
    @Autowired
    public void setSqlSession(SqlSessionFactory sqlSessionFactory) {
        sqlSessionFactory.getConfiguration().addMapper(mapperClass);
        this.sqlSession = sqlSessionFactory.openSession();
    }

    // 获取mybatis生成的代理对象
    @Override
    public Object getObject() {
        // 使用sqlSession生成mapper接口的代理对象
        return sqlSession.getMapper(mapperClass);
    }

    @Override
    public Class<?> getObjectType() {
        return mapperClass;
    }
}
```










