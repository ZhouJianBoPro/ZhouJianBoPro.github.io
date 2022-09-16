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
