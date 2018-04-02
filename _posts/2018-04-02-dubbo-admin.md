---
layout: post
title: dubbo-admin管理平台搭建
date: 2018-04-02
categories: linux
tags: [linux]
description: linux中搭建dubbo-admin。
---

**1.前提**<br/>
> centOS中安装jdk，tomcat,安装过程忽略

**2.zookeeper部署**
- 官网下载地址:http://www.apache.org/dyn/closer.cgi/zookeeper/
- 解压安装包:tar -zxvf  zookeeper-3.4.11.tar.gz
- 文件重命名:mv zookeeper-3.4.11.tar.gz zookeeper
- 进入zookeeper配置：cd zookeeper/conf
- 将zoo_sample.cfg重命名为zoo.cfg
- 修改zoo.cfg配置并保存：dataDir=/root/zookeeper/data
- 运行zookeeper：cd /root/zookeeper/bin, ./zkServer.sh start
- 查看运行状态:./zkServer.sh status

**3.部署dubbo-admin**
- dubbo-admin clone: https://github.com/alibaba/dubbo
- 针对dubbo-master目录下的子项目dubbo-admin编译打包：mvn package -Dmaven.skip.test=true
- 将子项目dubbo-admin新生成的target目录下的dubbo-admin-2.5.4-SNAPSHOT上传至tomcat的webapp中，并修改名称为ROOT
- 进入ROOT/WEB-INF中可以修改root和guest账户的登录密码
- 启动tomcat后在浏览器中访问：http://116.62.57.144:8080
- 输入用户名密码进行登录

**常见问题**
1. zookeeper和tomcat启动正常，在浏览器中访问不了：可能是8080端口没有开放访问权限









