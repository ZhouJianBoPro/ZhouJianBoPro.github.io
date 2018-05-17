---
layout: post
title: 深入了解适配器模式
date: 2018-03-16
categories: design patterns
tags: [patterns]
description: 深入了解适配器模式
---

**适配器模式使用场景**
- 想要建立一个可以重复使用的类，将一些彼此之间没有太多关联的类联系一起工作
- 如有两种usb接口互不兼容，需要有个usb转换器能够同时兼容这两种usb接口，这个usb转换器被称之为适配器
 
**模式结构**
> 适配器模式包含的角色

- Target：目标抽象类（需要兼容的接口）
- Adapter：适配器类
- Adaptee：适配者类
- Client：客户类
 
 **适配器的两种实现方式**
- 类适配器：单一的为一个类而实现的一种设计模式
![类适配器](/images/class_adapter.png)
- 对象适配器
 