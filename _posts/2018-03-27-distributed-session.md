---
layout: post
title: 分布式环境下session处理机制
date: 2018-03-27
categories: 分布式
tags: [session]
description: 分布式环境下session处理机制
---

**场景**
1. 在分布式环境下，集群中存在A,B两台服务器
2. 用户第一次访问网站时，Nginx通过负载均衡机制将用户请求转发至A服务器，登陆后A服务器会给用户创建一个session
3. 用户第二次访问时，请求被转发至B服务器，此时B服务器没有session，用户得重新登录生成session
4. 造成用户频繁登，在项目中是绝对不能存在的

**处理机制**
1. [粘性session](#m1)
2. [服务器session复制](#m2)
3. [基于分布式缓存的session共享机制](#m3)
4. [session持久化到DB中](#m4)

<span id="m1"><font color="#dd0000">粘性session</font><br /></span>
1.原理<br/> 
> 负载均衡器设置了粘性session，请求被转发到A服务器，以后用户请求与A服务器绑定。

2.优点<br/> 
> 简单，不需要对session做任何操作
 
3.缺陷<br/> 
> 缺乏容错性，如果当前服务器发生故障，该用户请求被转发至另一台服务器上，session信息将失效，得重新登录
 

<span id="m2"><font color="#dd0000">服务器session复制</font><br /></span>
1.原理<br/>
> 任何一个服务器上的session发生改变，该节点会将session所有内容序列化广播至所有其他节点，不管其他节点需不需要session，以此来来保证session同步

2.优点<br/>
> 可容错，各个服务器间的session可实时响应

3.缺陷<br/>
> 用户量大的话会对服务器造成一定的压力，会造成网络堵塞


<span id="m3"><font color="#dd0000">基于分布式缓存的session共享机制</font><br /></span>
1.原理<br/>
> cache做主从复制，写入session都往从cache服务器上写，读取都从主cacche服务器中读取；应用服务器发生故障时，内存中的session会丢失，从持久化的cache中读取

2.优点<br/>
> 可容错，session实时响应


<span id="m4"><font color="#dd0000">session持久化DB中</font><br /></span>
1.原理<br/>
> 创建一个数据库，专门用来存储session，保证session的持久化

2.优点<br/>
> 服务器出现问题，session不会丢失

3.缺陷<br/>
> 网站访问量大，对数据库造成压力

**常用机制**

基于分布式缓存的session共享应用最为广泛，无论是效率还是扩展性都挺好

