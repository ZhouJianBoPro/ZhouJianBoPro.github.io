---
layout: post
title: mysql乐观锁学习总结
date: 2018-03-22
categories: 高并发
tags: [高并发]
description: 深入了解乐观锁。
---

**乐观锁介绍**
> 乐观锁认为数据一般情况下不会造成冲突，所以在数据进行更新的时候，才会正式对数据的冲突与否进行检测，如果发生冲突了，返回错误信息，让用户决定如何去做。

**实现乐观锁的两种方式**

<font color="#660066">使用数据版本（version）记录机制实现：</font>
1. 在数据库表中增加数字类型的version的字段；
2. 当读取数据时，将version字段的值一同读出，数据每更新一次，对此version值加一。
3. 当我们提交更新的时候，判断数据库表对应记录的当前版本信息与第一次取出来的version值进行比对，如果数据库表当前版本号与第一次取出来的version值相等，则予以更新，否则认为是过期数据。

<font color="#dd0000">使用时间戳（timestamp）记录机制实现：</font>
1. 在乐观锁控制的table中增加一个timestamp类型的字段，字段名自己定义;
2. 和上面的version操作类似，也是在更新提交的时候检查当前数据库中数据的时间戳和自己更新前取到的时间戳进行对比，如果一致则OK，否则就是版本冲突。

**举例说明**
> 商品goods表中有一个字段status，status为1代表商品未被下单，status为2代表商品已经被下单，那么我们对某个商品下单时必须确保该商品status为1。假设商品的id为1。

<font color="#660066">操作步骤：</font>
```$xslt
#1.查询出商品信息
select (status,status,version) from t_goods where id=#{id}

#2.根据商品信息生成订单

#3.修改商品status为2
update t_goods set status=2,version=version+1 where id=#{id} and version=#{version};
```

**优势与不足**
1. 乐观锁是相信事物之间的竞争的概率是比较少的，因此可以到提交的时候去锁定，进行数据校验。
2. 乐观并发控制不会产生死锁和任何锁。
3. 并发量较大的时候有可能存在风险，例如两个事务都读取了数据库的某一行，经过修改以后写回数据库，这时就遇到了问题。