---
layout: post
title: session-cookie
date: 2018-03-30
categories: basic
tags: [basic]
description: session及cookie深入了解。
---

**cookie机制**<br/>
1.cookie的基本特点：
- 以key-value形式存储在客户端
- 只能保存字符串，不能保存对象
- 需要客户端支持，客户端可以禁用cookie
- 可以设置过期时间（Cookie.setMaxAge()）

2.cookie类型区分：（过期时间)<br/>
- 0不支持cookie
- 负数会话cookie
- 正数普通cookie

3.cookie编程<br/>
```java
//cookie的创建
Cookie cookie = new Cookie(key, value);

//设置cookie过期时间，单位秒
cookie.setMaxAge(-1); //关闭浏览器即失效
cookie.setMaxAge(60); //60s后没有访问服务器失效
```

**session机制**
> 每次客户端发送请求，服务端都会检查是否含有sessionId；如果有，根据sessionId检索出session进行处理，
如果没有，则创建一个session,并绑定一个不重复的sessionId

1.基本特点:
- 状态信息保存在服务端，安全性更高
- 能够支持任何类型的对象
- 可以设置有效时间

2.保存sessionId的技术：
- 通过cookie，默认方式，利用加解密技术在客户端与服务端中传递JsessionId,但是客户端可能禁用cookie
- 重写url，在url传递JsessionId属性，
- 通过隐藏域保存

3.session编程：
```java
//保存信息
session.setAttribute(String key, Object value);
//获取session信息
session.getAttribute(key);
//设置有效时间
session.setMaxInactiveInterval(int min);//0表示不支持session，负值表示永不失效

```


