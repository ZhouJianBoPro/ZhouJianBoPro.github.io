---
layout: post
title: spring-boot中redis的使用
date: 2018-04-04
categories: spring
tags: [spring-boot]
description: spring-boot中redis的使用。
---

> spring boot对nosql数据库也进行了封装自动化

**redis的优势**
> redis支持更加丰富的数据结构，同时支持数据持久化；除此之外，redis还提供了一些数据库的特性，比如事物，主从库。

**spring-boot中简单使用redis**<br/>
1.引入spring-boot-starter-redis
```xml
<dependency>  
    <groupId>org.springframework.boot</groupId>  
    <artifactId>spring-boot-starter-redis</artifactId> 
     <version>1.3.2.RELEASE</version>
</dependency>  
```
2.application.properties中增加redis的配置
```properties
## Redis 配置
## Redis数据库索引（默认为0）
spring.redis.database=0
## Redis服务器地址
spring.redis.host=116.62.57.144
## Redis服务器连接端口
spring.redis.port=6379
## Redis服务器连接密码（默认为空）
spring.redis.password=
## 连接池最大连接数（使用负值表示没有限制）
spring.redis.pool.max-active=8
## 连接池最大阻塞等待时间（使用负值表示没有限制）
spring.redis.pool.max-wait=-1
## 连接池中的最大空闲连接
spring.redis.pool.max-idle=8
## 连接池中的最小空闲连接
spring.redis.pool.min-idle=0
## 连接超时时间（毫秒）
spring.redis.timeout=0
```
3.自动注入RedisTemplate模板bean<br/>
```java
@Service
public class CityServiceImpl implements CityService {

    private static final Logger LOGGER = LoggerFactory.getLogger(CityServiceImpl.class);

    @Autowired
    private CityDao cityDao;

    @Autowired
    private RedisTemplate redisTemplate;

    public City findCityById(Long id) {
        // 从缓存中获取城市信息
        String key = "city_" + id;
        ValueOperations<String, City> operations = redisTemplate.opsForValue();

        // 缓存存在
        boolean hasKey = redisTemplate.hasKey(key);
        if (hasKey) {
            City city = operations.get(key);

            LOGGER.info("CityServiceImpl.findCityById() : 从缓存中获取了城市 >> " + city.toString());
            return city;
        }

        // 从 DB 中获取城市信息
        City city = cityDao.findById(id);

        // 插入缓存
        operations.set(key, city, 10, TimeUnit.SECONDS);
        LOGGER.info("CityServiceImpl.findCityById() : 城市插入缓存 >> " + city.toString());

        return city;
    }
}
```
<font color="#dd0000">出现的问题：</font>RedisTemplate<K, V>模板类在操作redis时默认使用JdkSerializationRedisSerializer来进行序列化,
导致存储的key和value前缀有多余类似\xac\xed\x00\x05t\x00字符

<font color="#dd0000">解决方法：</font>将JdkSerializationRedisSerializer序列化修改为StringRedisSerializer
```html
@Autowired
private RedisTemplate redisTemplate;

@Autowired(required = false)
public void setRedisTemplate(RedisTemplate redisTemplate) {
    RedisSerializer stringSerializer = new StringRedisSerializer();
    redisTemplate.setKeySerializer(stringSerializer);
    redisTemplate.setValueSerializer(stringSerializer);
    redisTemplate.setHashKeySerializer(stringSerializer);
    redisTemplate.setHashValueSerializer(stringSerializer);
    this.redisTemplate = redisTemplate;
}
```

