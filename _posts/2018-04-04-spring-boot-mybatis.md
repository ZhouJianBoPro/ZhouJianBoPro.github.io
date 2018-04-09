---
layout: post
title: spring-boot mybatis整合
date: 2018-04-09
categories: spring
tags: [spring-boot]
description: spring-boot mybatis两种整合方式学习。
---

**实现方式**<br/>
- 映射文件（容易维护，sql都在xml文件中）
- 注解

**映射文件方式**<br/>
直接上代码：<br/>

1.数据源配置：
```properties
spring.datasource.url=jdbc:mysql://116.62.57.144:3306/springbootdb?useUnicode=true&characterEncoding=utf8
spring.datasource.username=root
spring.datasource.password=root
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
```
2.mapper映射文件
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd" >
<mapper namespace="org.spring.springboot.dao.CityDao">
    <resultMap id="BaseResultMap" type="org.spring.springboot.domain.City">
        <result column="id" property="id" />
        <result column="province_id" property="provinceId" />
        <result column="city_name" property="cityName" />
        <result column="description" property="description" />
    </resultMap>

    <parameterMap id="City" type="org.spring.springboot.domain.City"/>

    <sql id="Base_Column_List">
        id, province_id, city_name, description
    </sql>

    <insert id="insert" parameterType="org.spring.springboot.domain.City">
        insert into city(province_id,city_name,description)
        values (#{provinceId},#{cityName}, #{description})
    </insert>

    <select id="findByName" resultMap="BaseResultMap" parameterType="java.lang.String">
        select
        <include refid="Base_Column_List" />
        from city
        where city_name = #{cityName}
    </select>

    <select id="findByProvinceId" resultMap="BaseResultMap" parameterType="java.lang.Integer">
        select
        <include refid="Base_Column_List" />
        from city
        where province_id = #{provinceId}
    </select>

</mapper>

```
3.Dao接口种类会自动实现对sql的注入,方法名要与mapper中的映射对应
```java
public interface CityDao {

    City findByName(String cityName);

    Integer insert(City city);

    List<City> findByProvinceId(Integer provinceId);
}
```

**注解方式**<br/>
```java
@Mapper // 标志为 Mybatis 的 Mapper
public interface CityDao {

    /**
     * 根据城市名称，查询城市信息
     *
     * @param cityName 城市名
     */
    @Select("SELECT * FROM city where city_name = #{cityName}")
    // 返回 Map 结果集
    @Results({
            @Result(property = "id", column = "id"),
            @Result(property = "provinceId", column = "province_id"),
            @Result(property = "cityName", column = "city_name"),
            @Result(property = "description", column = "description"),
    })
    City findByName(@Param("cityName") String cityName);
}
```