---
layout: post
title: mysql悲观锁学习总结
date: 2018-03-21
categories: 高并发
tags: [高并发]
description: 深入了解悲观锁。
---

**描述**
> 它指的是对数据被外界修改持保守状态，因此，在获准悲观锁的事物提交之前，其他事物不能获准所有锁，不能对数据进行修改，悲观锁其实就是建立在行级排它锁之上；

**使用场景举例**
> 商品goods表中有一个字段status，status为1代表商品未被下单，status为2代表商品已经被下单，那么我们对某个商品下单时必须确保该商品status为1。假设商品的id为1。
  
如果不采用锁，操作方法如下：

```$xslt
//1.查询出商品信息
select status from t_goods where id=1;

//2.根据商品信息生成订单
insert into t_orders (id,goods_id) values (null,1);

//3.修改商品status为2
update t_goods set status=2;
```
<font color="#dd0000">上面这种场景在高并发情况下很容易出现问题，在执行第三步之前有可能该商品的status已经被修改，如果再次修改的话会造成重复下单；</font><br /> 




