---
layout: post
title: Docker容器安装vim及telnet
date: 2022-10-24
tags: [docker]
---

1. 更新软件源
    ```dockerfile
    RUN echo "deb [check-valid-until=no] http://cdn-fastly.deb.debian.org/debian jessie main" > /etc/apt/sources.list.d/jessie.list
    RUN echo "deb [check-valid-until=no] http://archive.debian.org/debian jessie-backports main" > /etc/apt/sources.list.d/jessie-backports.list
    RUN sed -i '/deb http:\/\/deb.debian.org\/debian jessie-updates main/d' /etc/apt/sources.list
    RUN apt-get -o Acquire::Check-Valid-Until=false update
    ```
2. 安装vim
    ```dockerfile
    RUN apt-get install -y vim
    ```
3. 安装telnet
    ```dockerfile
    RUN apt-get install -y telnet
    ```


