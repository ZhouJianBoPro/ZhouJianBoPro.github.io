---
layout: post
title: mysql悲观锁学习总结
date: 2018-03-22
categories: 高并发
tags: [高并发]
description: 深入了解悲观锁。
---

**描述**
> 它指的是对数据被外界修改持保守状态，因此，在获准悲观锁的事物提交之前，其他事物不能获准所有锁，不能对数据进行修改，悲观锁其实就是建立在行级排它锁之上；

**使用场景举例**
> 商品goods表中有一个字段status，status为1代表商品未被下单，status为2代表商品已经被下单，那么我们对某个商品下单时必须确保该商品status为1。假设商品的id为1。
  
1.如果不采用锁，操作方法如下：

```$xslt
//1.查询出商品信息
select status from t_goods where id=1;

//2.根据商品信息生成订单
insert into t_orders (id,goods_id) values (null,1);

//3.修改商品status为2
update t_goods set status=2;
```
<font color="#dd0000">上面这种场景在高并发情况下很容易出现问题，在执行第三步之前有可能该商品的status已经被修改，如果再次修改的话会造成重复下单；</font><br /> 

2.使用悲观锁来实现：
- 当我们在查询出goods信息后就把当前的数据锁定，直到我们修改完毕后再解锁。那么在这个过程中，因为goods被锁定了，就不会出现有第三者来对其进行修改了。
- <font color="#dd0000">注：要使用悲观锁，我们必须关闭mysql数据库的自动提交属性，因为MySQL默认使用autocommit模式，也就是说，当你执行一个更新操作后，MySQL会立刻将结果进行提交。</font><br /> 

```$xslt
//1.查看数据库默认的自动提交属性命令
show variables  like 'autocommit';

//2.关闭自动提交属性
set autocommit=0;

//3.开始事物
begin;

//4.查询出商品信息,并加上排它锁
select status from t_goods where id=1 for update;

//5.修改商品status为2
update t_goods set status=2;

//6.提交事物
commit;

```
<font color="#dd0000">注：需要注意的是，在事务中，只有SELECT ... FOR UPDATE 或LOCK IN SHARE MODE 同一笔数据时会等待其它事务结束后才执行，一般SELECT ... 则不受此影响</font>

**锁定级别与索引之间的关系**
- 明确指定索引，并且有此数据，row lock
- 明确指定索引，若查无此数据，无lock
- 无索引，table lock
- 索引不明确（非=），table lock

**优点与不足**
1. 保证多并发过程中数据的完整性及一致性。
2. 处理加锁机制会让数据库产生额外的消耗。
3. 有可能产生死锁。
4. 由于悲观锁锁定数据时其他事物不能对该数据进行操作，并行效果较差.
5. 在只读型事物中没必要使用悲观锁，数据不会造成冲突。



