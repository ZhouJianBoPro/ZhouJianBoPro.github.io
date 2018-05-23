---
layout: post
title: c3p0数据库连接池
date: 2018-03-18
categories: database
tags: [database]
description: c3p0数据库连接池
---

**传统模式下的JDBC数据库连接池**<br/>

1.工作流程：
- 加载数据库驱动
- 建立数据库连接
- 执行sql并返回结果
- 关闭数据库连接

2.缺点：<br/>
> 每来一个请求就得重新创建数据库连接，频繁的创建和关闭连接占用大量的系统资源，严重的话会造成
服务器的崩溃

**数据库连接池**
1. 为数据库连接建立一个缓冲池，预先在数据库连接池中放入一定数量的连接（最小数据库连接数），当需要建立连接时从连接池中
取出连接即可，用完之后将连接释放至数据库连接池中
2. 当请求连接数大于最小数据数时，数据库连接池中增加连接数，最大不会超过最大连接数
3. 当数据库连接池中的连接数量大于最小连接数时，且有空闲的连接，可以设置最大空闲时间，超过该时间没有被使用即关闭
4. 当请求的连接数超过最大连接数，将请求放入等待队列中，排队等待空闲连接

**c3p0数据库连接池**
```java
//初始化核心池
ComboPooledDataSource dataSource=new ComboPooledDataSource();
dataSource.setJdbcUrl("jdbc:mysql:///test");//设置url
dataSource.setDriverClass("com.mysql.jdbc.Driver");//设置驱动
dataSource.setUser("root");//mysql的账号
dataSource.setPassword("123456");//mysql的密码
dataSource.setInitialPoolSize(6);//初始连接数，即初始化6个连接
dataSource.setMaxPoolSize(50);//最大连接数，即最大的连接数是50
dataSource.setMaxIdleTime(60);//最大空闲时间

//从数据库连接池中获取连接对象，并发情况下要使用同步
Connection con=dataSource.getConnection();

//执行sql
String sql="select * from user ";
PreparedStatement ps=con.prepareStatement(sql);

//获取结果集
ResultSet rs=ps.executeQuery();
List<User> list=new ArrayList<User>();
while(rs.next()){
    User user=new User();
    user.setId(rs.getInt("id"));
    user.setName(rs.getString("name"));
    list.add(user);
}
```