---
layout: post
title: spring整合mybatis底层原理
date: 2024-10-24
tags: [spring]
---

#### @MapperScan
用于扫描指定包下的Mapper接口，这样就不需要为每个Mapper接口单独配置一个MapperFactoryBean，简化了配置过程。
MapperScan注解上会导入MapperScannerRegistrar， 用于将扫描到的Mapper接口生成代理对象，并注册为spring的bean。
```java
@Import(MapperImportBeanDefinitionRegistrar.class)
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface MapperScan {

    String path();
}
```

#### MapperFactoryBean
通过实现FactoryBean接口，重写getObject方法，实现自定义bean的创建逻辑。
MapperFactoryBean中注入了mybatis的SqlSession，用于创建Mapper接口的代理对象。
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

#### ClassPathMapperScanner
该mapper包扫描器实现了ClassPathBeanDefinitionScanner接口。重写isCandidateComponent方法，只对扫描到的接口进行bean的注册；
并重写doScan方法，将扫描到的mapper接口封装为BeanDefinition， BeanDefinition的类型要修改为MapperFactoryBean（通过FactoryBean#getObject给mapper接口生成代理对象），
由于MapperFactoryBean构造方法需要传入mapperClass，因此BeanDefinition构造函数需要传入mapper接口
```java
public class ClassPathMapperScanner extends ClassPathBeanDefinitionScanner {

    public ClassPathMapperScanner(BeanDefinitionRegistry registry) {
        super(registry);
    }

    @Override
    protected boolean isCandidateComponent(AnnotatedBeanDefinition beanDefinition) {
        // 只扫描接口，与spring bean生命周期扫描策略不同，spring bean生命周期只扫描.class文件
        return beanDefinition.getMetadata().isInterface();
    }

    @Override
    protected Set<BeanDefinitionHolder> doScan(String... basePackages) {

        Set<BeanDefinitionHolder> beanDefinitionHolders = super.doScan(basePackages);

        for (BeanDefinitionHolder beanDefinitionHolder : beanDefinitionHolders) {

            BeanDefinition beanDefinition = beanDefinitionHolder.getBeanDefinition();
            // 设置bean构造函数参数，参数为mapper接口的全类名
            beanDefinition.getConstructorArgumentValues().addGenericArgumentValue(beanDefinition.getBeanClassName());
            // 设置bean的class为MapperFactoryBean, spring会调用FactoryBean#getObject方法，通过mybatis的SqlSession为mapper接口生成代理对象
            beanDefinition.setBeanClassName(MapperFactoryBean.class.getName());
        }
        return beanDefinitionHolders;
    }
}
```

#### MapperScannerRegistrar
实现了ImportBeanDefinitionRegistrar接口，并重写registerBeanDefinitions方法，该类通常被导入在@MapperScan注解上。
从@MapperScan注解中获取到扫描路径，然后创建MapperClassPathScanner扫描器，调用扫描器的scan方法完成mapper接口的扫描和注册。
```java
public class MapperScannerRegistrar implements ImportBeanDefinitionRegistrar {
    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry, BeanNameGenerator importBeanNameGenerator) {

        // 从类的元数据中获取指定mapper扫描注解的属性
        Map<String, Object> annotationAttributes = importingClassMetadata.getAnnotationAttributes(MapperScan.class.getName());
        // 该注解只有一个属性，即key=path
        String scanPath = (String) annotationAttributes.get("path");

        // 创建扫描
        ClassPathMapperScanner mapperScanner = new ClassPathMapperScanner(registry);
        // 添加过滤器，该过滤器接受所有类
        mapperScanner.addIncludeFilter((metadataReader, metadataReaderFactory) -> true);

        // 执行mapper扫描，并对mapper接口生成代理对象放入到spring容器中
        mapperScanner.doScan(scanPath);
    }
}
```

#### SqlSessionFactory
```java
@MapperScan(path = "com.proj.mapper")
@ComponentScan("com.proj")
public class AppConfig {

    @Bean
    public SqlSessionFactory sqlSessionFactory() throws IOException {
        // 创建SqlSessionFactory bean
        InputStream is = Resources.getResourceAsStream("mybatis-config.xml");
        return new SqlSessionFactoryBuilder().build(is);
    }
}
```

#### mybatis配置文件
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">

<configuration>
    <!-- 配置环境 -->
    <environments default="development">
        <environment id="development">
            <!-- 事务管理器 -->
            <transactionManager type="JDBC"/>
            <!-- 数据源 -->
            <dataSource type="POOLED">
                <property name="driver" value="com.mysql.cj.jdbc.Driver"/>
                <property name="url" value="jdbc:mysql://localhost:3306/test?useSSL=false&amp;serverTimezone=UTC"/>
                <property name="username" value="root"/>
                <property name="password" value="123456"/>
            </dataSource>
        </environment>
    </environments>
</configuration>
```

#### 完整流程
1. @MapperScan指定mapper接口的包路径。@MapperScan注解上会导入MapperScannerRegistrar
2. MapperScannerRegistrar会获取到扫描路径，然后创建ClassPathMapperScanner扫描器，调用doScan方法
3. ClassPathMapperScanner扫描器会扫描指定包下的mapper接口（重写ClassPathBeanDefinition的isCandidateComponent方法过滤接口），并封装为BeanDefinition，修改BeanDefinition的class类型为MapperFactoryBean, 并且将BeanDefinition的构造函数参数设置为mapper接口的全类名
4. MapperBeanFactory通过mybatis的SqlSession给构造函数传入的mapper接口生成代理对象










