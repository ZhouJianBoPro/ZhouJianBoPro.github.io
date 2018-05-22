---
layout: post
title: mysql索引
date: 2018-05-22
categories: database
tags: [database]
description: mysql索引
---

**使用索引的优缺点**
- 加快查询速度
- 在数据插入及修改时效率较低（索引需要存储在mysql中）

**建表时添加索引**
```sql
CREATE TABLE T_USER (
  'id' int(10) not null auto_increment comment '主键',
  'user_name' VARCHAR (20) not null comment '用户名',
  'password' VARCHAR (100) not null comment '用户密码',
  PRIMARY key('id'),
  index(user_name) #创建单索引
);
```
```sql
   CREATE TABLE T_USER (
     'id' int(10) not null auto_increment comment '主键',
     'user_name' VARCHAR (20) not null comment '用户名',
     'password' VARCHAR (100) not null comment '用户密码',
     PRIMARY key('id'),
     UNIQUE index index_userName(user_name) #创建唯一索引
   );
   ```
   ```sql
   CREATE TABLE T_USER (
     'id' int(10) not null auto_increment comment '主键',
     'user_name' VARCHAR (20) not null comment '用户名',
     'password' VARCHAR (100) not null comment '用户密码',
     PRIMARY key('id'),
     UNIQUE index index_userName(user_name) #创建唯一索引
   );
   ```
   ```sql
   CREATE TABLE T_USER (
     'id' int(10) not null auto_increment comment '主键',
     'user_name' VARCHAR (20) not null comment '用户名',
     'password' VARCHAR (100) not null comment '用户密码',
     PRIMARY key('id'),
     index index_userName_password(user_name, password) #创建联合索引
   );
   ```

**已存在的表创建索引**<br/>
1.创建单列索引：
```sql
ALTER table t_user add index index_name(user_name); 
```
2.创建唯一索引
```sql
ALTER table t_user add UNIQUE index index_name(user_name);
```
3.创建联合索引
```sql
alter TABLE  t_user add index index_name(user_name, password);
```

**删除索引**
```sql
drop index index_name on t_user;
```

**总结**
- 索引一般使用在查询数据量较大的场景下
- 哪些字段建立索引？一般在where、order by，group by后面的字段
