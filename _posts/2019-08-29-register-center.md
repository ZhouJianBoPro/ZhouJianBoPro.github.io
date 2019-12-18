---
layout: post
title: 注册中心
date: 2018-08-29
tags: [register]
description: 注册中心
---

**Euraka工作原理**

1. euraka简介：euraka是服务注册发现的工具，spring cloud集成了euraka, euraka可以细分为euraka server和euraka client

2. euraka工作原理：<br/>
- 服务调用方通过服务名来在euraka server中找到服务提供方的地址，同时注册中心会根据负载均衡算法向服务调用方提供最佳节点
- 处于不同节点的euraka通过replicate进行信息同步
-  服务启动后向euraka注册，euraka server会将注册信息replicate至其他euraka server
- 当application client调用application server时，会向euraka获取服务提供方的地址，然后会将改地址缓存在本地，下次调用时直接从本地缓存中取
- 当euraka server 检测到服务提供方不可用时，会将该服务置为down，服务恢复正常后，application client会将服务提供方的地址更新，重新加载到本地内存中
- 服务提供方在启动后会定期向euraka server发送心跳，如果超过一定时间没有发送，该节点的euraka server将该服务置为down

**zk工作原理**

1. zk角色：leader(领导者)，learner(学习者)
- leader:接受leaner请求，leader将请求广播至每个follower，提交其他节点的follower的投票结果返回给请求的follwer
- learner:learner分为：follower(跟随者)，observer(观察者)，follower主要负责接受客户端的请求（可以直接对读请求进行处理，写请求转发至leeader），在选举leader的时候进行投票；observer将客户端的写请求转发给leader节点，不参与投票，提供系统的读操作性能 

2. 流程：
- follower接受到请求后，若该请求为写请求，将该请求转发至leader，读请求该follow自行处理
- 判断leader的状态，若leader服务不可用，重新选举leader
- leader将请求转发给其他节点的follower进行提案投票
- follower将投票结果发送给leader
- leader将结果汇总，如果需要写入，则开始写入并将写入操作发送给所有的follower,然后commit
- follower将请求结果返回给client


**euraka和zk之间的区别**
- euraka保证分布式系统CAP(C:一致性，A:可用性，P:分区容错性)AP，保障了服务的可用性，euraka server各个节点都是平等的，某个节点挂掉并不影响其他节点工作，当服务调用方发现某个euraka注册或连接失败，会自动切换到另一个节点，保障了服务的高可用性，但是会存在强一致性问题
- zk保证了分布式系统中的CP, 选举leader时可能会耗费一定的时间





