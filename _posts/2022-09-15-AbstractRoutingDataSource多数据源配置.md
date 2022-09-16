---
layout: post
title: AbstractRoutingDataSource多数据源配置
date: 2022-09-15
tags: [spring]
---

#### 多数据源使用场景
1. 一个应用需要使用多个数据源
2. 数据库主从部署，同时要求读写分离

#### spring-jdbc的AbstractRoutingDataSource实现原理
- InitializingBean初始化AbstractRoutingDataSource，初始化了所有的数据源，并设置默认数据源
    ```java
    public abstract class AbstractRoutingDataSource extends AbstractDataSource implements InitializingBean {
        
        //包含所有的数据源
        private Map<Object, Object> targetDataSources;
        //默认数据源
        private Object defaultTargetDataSource;
    
        private Map<Object, DataSource> resolvedDataSources;
    
        private DataSource resolvedDefaultDataSource;
    
        @Override
        public void afterPropertiesSet() {
            if (this.targetDataSources == null) {
                throw new IllegalArgumentException("Property 'targetDataSources' is required");
            }
            this.resolvedDataSources = new HashMap<Object, DataSource>(this.targetDataSources.size());
            //解析配置的数据源，key-value格式
            for (Map.Entry<Object, Object> entry : this.targetDataSources.entrySet()) {
                //获取数据源对应的key
                Object lookupKey = resolveSpecifiedLookupKey(entry.getKey());
                //获取key对应的DataSource
                DataSource dataSource = resolveSpecifiedDataSource(entry.getValue());
                this.resolvedDataSources.put(lookupKey, dataSource);
            }
            //设置默认数据源
            if (this.defaultTargetDataSource != null) {
                this.resolvedDefaultDataSource = resolveSpecifiedDataSource(this.defaultTargetDataSource);
            }
        }
    
    }
    ```
- determineCurrentLookupKey抽象方法用于获取需要使用数据源的key，扩展时需要重写该方法
    ```java
    public abstract class AbstractRoutingDataSource extends AbstractDataSource implements InitializingBean {
        
        protected abstract Object determineCurrentLookUpKey();
    
    }
    ```
- determineTargetDataSource方法返回当前需要使用到的数据源DataSource。方法中调用抽象方法determineCurrentLookUpKey获取
目标数据源的key，然后通过key获取到对应的数据源
    ```java
    public abstract class AbstractRoutingDataSource extends AbstractDataSource implements InitializingBean {
            
            protected abstract Object determineCurrentLookUpKey();
    
            protected DataSource determineTargetDataSource() {
                Assert.notNull(this.resolvedDataSources, "DataSource router not initialized");
                //获取当前需要使用数据源的key
                Object lookupKey = determineCurrentLookupKey();
                //通过key获取对应的DataSource
                DataSource dataSource = this.resolvedDataSources.get(lookupKey);
                if (dataSource == null && (this.lenientFallback || lookupKey == null)) {
                    dataSource = this.resolvedDefaultDataSource;
                }
                if (dataSource == null) {
                    throw new IllegalStateException("Cannot determine target DataSource for lookup key [" + lookupKey + "]");
                }
                return dataSource;
            }
    }
    ```
#### 多数据源实现示例
- JavaConfig方式配置DataSource1与DataSource2
    ```java
    @Configuration
    public class DataSourceConfigure {
    
        @Value("${datasource1.connection.url}")
        private String datasource1Url;
    
        @Value("${datasource2.connection.url}")
        private String datasource2Url;
    
        @Value("${datasource1.connection.userName}")
        private String datasource1UserName;
    
        @Value("${datasource2.connection.userName}")
        private String datasource2UserName;
    
        @Value("${datasource1.connection.password}")
        private String datasource1Password;
    
        @Value("${datasource2.connection.password}")
        private String datasource2Password;
    
        @Value("${druid.filters}")
        private String druidFilters;
    
        @Value("${druid.maxActive}")
        private int druidMaxActive;
    
        @Value("${druid.initialSize}")
        private int druidInitialSize;
    
        @Value("${druid.maxWait}")
        private long druidMaxWait;
    
        @Value("${druid.minIdle}")
        private int druidMinIdle;
    
        @Value("${druid.timeBetweenEvictionRunsMillis}")
        private long druidTimeBetweenEvictionRunsMillis;
    
        @Value("${druid.minEvictableIdleTimeMillis}")
        private long druidMinEvictableIdleTimeMillis;
    
        @Value("${druid.testWhileIdle}")
        private boolean druidTestWhileITdle;
    
        @Value("${druid.testOnBorrow}")
        private boolean druidTestOnBorrow;
    
        @Value("${druid.testOnReturn}")
        private boolean druidTestOnReturn;
    
        @Value("${druid.poolPreparedStatements}")
        private boolean druidPoolPreparedStatements;
    
        @Value("${druid.maxPoolPreparedStatementPerConnectionSize}")
        private int druidMaxPoolPreparedStatementPerConnectionSize;
    
        @Bean(name = "dataSource1")
        public DruidDataSource dataSource1() throws SQLException {
            return buildDruidDataSource(datasource1Url, datasource1UserName, datasource1Password);
        }
    
        @Bean(name = "dataSource2")
        public DruidDataSource dataSource2() throws SQLException {
            return buildDruidDataSource(datasource2Url, datasource2UserName, datasource2Password);
        }
        
        private DruidDataSource buildDruidDataSource(String url, String userName, String password) throws SQLException {
            
            DruidDataSource druidDataSource = new DruidDataSource();
            druidDataSource.setUrl(url);
            druidDataSource.setUsername(userName);
            druidDataSource.setPassword(password);
            druidDataSource.setFilters(druidFilters);
            druidDataSource.setMaxActive(druidMaxActive);
            druidDataSource.setInitialSize(druidInitialSize);
            druidDataSource.setMaxWait(druidMaxWait);
            druidDataSource.setMinIdle(druidMinIdle);
            druidDataSource.setTimeBetweenEvictionRunsMillis(druidTimeBetweenEvictionRunsMillis);
            druidDataSource.setMinEvictableIdleTimeMillis(druidMinEvictableIdleTimeMillis);
            druidDataSource.setTestWhileIdle(druidTestWhileITdle);
            druidDataSource.setTestOnBorrow(druidTestOnBorrow);
            druidDataSource.setTestOnReturn(druidTestOnReturn);
            druidDataSource.setPoolPreparedStatements(druidPoolPreparedStatements);
            druidDataSource.setMaxPoolPreparedStatementPerConnectionSize(druidMaxPoolPreparedStatementPerConnectionSize);
            druidDataSource.init();
            return druidDataSource;
        }
    }
    ```
    ```properties
    druid.initialSize=10
    druid.minIdle=10
    druid.maxActive=50
    druid.maxWait=60000
    druid.timeBetweenEvictionRunsMillis=60000
    druid.minEvictableIdleTimeMillis=300000
    druid.testWhileIdle=true
    druid.testOnBorrow=false
    druid.testOnReturn=false
    druid.poolPreparedStatements=true
    druid.maxPoolPreparedStatementPerConnectionSize=20
    druid.filters=stat
    ```
- 写一个数据源类型管理类，使用ThreadLocal共享线程变量，保证线程安全
    ```java
    //数据源类型枚举，用作存储数据源信息的key
    public enum DataSourceEnum {
        DATASOURCE1, DATASOURCE2;
    }
  
    //数据源类型管理
    public class DataSourceTypeManager {
    
        private static final ThreadLocal<DataSourceEnum> dataSourceTypes = new ThreadLocal<DataSourceEnum>() {
            @Override
            protected DataSourceEnum initialValue(){
                //DataSource1为默认数据源
                return DataSourceEnum.DATASOURCE1;
            }
        };
    
        public static DataSourceEnum get(){
            return dataSourceTypes.get();
        }
    
        public static void set(DataSourceEnum dataSourceType){
            dataSourceTypes.set(dataSourceType);
        }
    }
    ```
- 写一个ThreadLocalRoutingDataSource类，该类继承AbstractRoutingDataSource，并实现determineCurrentLookUpKey方法。
该类的作用是获取当前需要使用的数据源key
    ```java
    public class ThreadLocalRoutingDataSource extends AbstractRoutingDataSource {
    
        @Override
        protected Object determineCurrentLookupKey() {
            return DataSourceTypeManager.get();
        }
    
    }
    ```
- 配置多个数据源及其事务、mybatis扫描的包
    ```java
    @EnableTransactionManagement
    @Configuration
    public class DataSourceConfigure {
    
        @Bean(name = "dataSource1")
        public DruidDataSource dataSource1() throws SQLException {
            return buildDruidDataSource(datasource1Url, datasource1UserName, datasource1Password);
        }
    
        @Bean(name = "dataSource2")
        public DruidDataSource dataSource2() throws SQLException {
            return buildDruidDataSource(datasource2Url, datasource2UserName, datasource2Password);
        }
    
        @Bean(name = "multiDataSource")
        public ThreadLocalRountingDataSource multiDataSource(@Qualifier("dataSource1") DruidDataSource dataSource1,
                                                             @Qualifier("dataSource2") DruidDataSource dataSource2) {
    
            ThreadLocalRountingDataSource threadLocalRountingDataSource = new ThreadLocalRountingDataSource();
            threadLocalRountingDataSource.setDefaultTargetDataSource(dataSource1);
    
            Map<Object, Object> targetDataSources = Maps.newHashMap();
            targetDataSources.put(DataSourceEnum.DATASOURCE1, dataSource1);
            targetDataSources.put(DataSourceEnum.DATASOURCE2, dataSource2);
            threadLocalRountingDataSource.setTargetDataSources(targetDataSources);
            return threadLocalRountingDataSource;
        }
    
        @Bean(name = "bizTransactionManager")
        public DataSourceTransactionManager bizTransactionManager(ThreadLocalRountingDataSource multiDataSource) {
    
            DataSourceTransactionManager transactionManager = new DataSourceTransactionManager();
            transactionManager.setDataSource(multiDataSource);
            return transactionManager;
        }
    
        @Bean(name = "bizSqlSessionFactory")
        public SqlSessionFactoryBean bizSqlSessionFactory(ThreadLocalRountingDataSource multiDataSource) throws IOException {
    
            SqlSessionFactoryBean sqlSessionFactory = new SqlSessionFactoryBean();
            sqlSessionFactory.setDataSource(multiDataSource);
            sqlSessionFactory.setConfigLocation(new PathMatchingResourcePatternResolver().getResource("classpath:mybatis/mybatis-config.xml"));
            sqlSessionFactory.setMapperLocations(new PathMatchingResourcePatternResolver().getResources("classpath:cn/com/mysql/mapper/**/*.xml"));
            return sqlSessionFactory;
        }
    
        @Bean(name = "mapperScannerConfigurer")
        public MapperScannerConfigurer mapperScannerConfigurer() {
    
            MapperScannerConfigurer mapperScannerConfigurer = new MapperScannerConfigurer();
            mapperScannerConfigurer.setBasePackage("cn.com.mysql.dao");
            mapperScannerConfigurer.setSqlSessionFactoryBeanName("bizSqlSessionFactory");
        }
    }
    ```
- 以AOP方式配置数据源拦截器，当执行到某些特定的方法，需要切换数据源去执行。默认使用DataSource1， 
在其他数据源执行完成之后或执行时出现异常，需要自动切换到默认数据源
    ```java
    @Aspect
    @Component
    public class DataSourceInterceptor {
    
        /**
         * 设置DataSource2的切点
         */
        @Pointcut("execution(public * cn.com.mysql.dao.datasource2.*(..))")
        public void dataSource2Point(){};
    
        /**
         * 执行dataSource2的方法之前，先切换数据源为dataSource2
         */
        @Before("dataSource2Point()")
        public void beforeDataInSource2(JoinPoint jp) {
            DataSourceTypeManager.set(DataSourceEnum.DATASOURCE2);
        }
    
        /**
         * 执行dataSource2的方法之后，切换数据源位dataSource1
         */
        @AfterReturning("dataSource2Point()")
        public void afterReturningInDataSource2(JoinPoint jp) {
            DataSourceTypeManager.set(DataSourceEnum.DATASOURCE1);
        }
    
        /**
         * dataSource2的方法抛出异常之后，切换数据源位dataSource1
         */
        @AfterThrowing("dataSource2Point()")
        public void afterThrowingInDataSource2(JoinPoint jp) {
            DataSourceTypeManager.set(DataSourceEnum.DATASOURCE1);
        }
    }
    ```

