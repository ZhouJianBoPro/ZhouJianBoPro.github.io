---
layout: post
title: mysql表连接查询
date: 2018-05-21
categories: database
tags: [database]
description: mysql表连接查询
---

**常见的四种连接查询方式**
- 内连接（inner join on）
- 左外连接[左连接]（left outer join on）
- 右外连接[右连接]（right outer join on）
- 完全连接（full join）
- 交叉连接（cross join on）

**例子**<br/>
![](/images/q1.png)

1.内连接：两张表条件相同的记录连接起来
![](/images/q2.png)

2.左外连接：左表中每条记录按条件去右表全表匹配，存在则将右表记录拼接在左表该条记录之后,不存在则拼null
![](/images/q3.png)

3.右外连接：右表中每条记录按条件去左表全表匹配，存在则将左表记录拼接在右表该条记录之后,不存在则拼null
![](/images/q4.png)

4.完全连接：取两张表的并集
![](/images/q5.png)

5.交叉连接：取两张表的笛卡尔积

**总结**
1. 查询两表关联列相等的数据用内连接
2. 右表是左表的子集用左外连接
3. 左表是右表的子集用右外连接