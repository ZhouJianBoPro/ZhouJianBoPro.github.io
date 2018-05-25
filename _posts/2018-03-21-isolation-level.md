---
layout: post
title: 深入了解事物的隔离级别
date: 2018-03-16
categories: high concurrency
tags: [concurrency]
description: 深入分析事物的隔离级别。
---
**事物的ACID特性简称：**
- 原子性(atomicity)：每一个简单的 SQL 语句即包含在一个事务中，具有原子性，多个sql不能同时执行；
- 一致性(consistency)：终极目标，数据不会被破坏，两句UPDATE语句，从A账户转账到B账户，不管成功失败，A和B账户的总额是不变的；
- 隔离性(isolation)：当多个事物处理同一数据时，多个事物之间时互不影响的，所以，在多个事物并发操作过程中，控制不好事物隔离级别会造成数据的脏读，幻读，不可重复读等读现象；
- 持久性(durability)：事物一旦被提交，数据可以被持久化；

**隔离性中的问题：**
<font color="#dd0000">脏读：A事物读取了B事物未提交更改的数据：</font>
![脏读示例](/images/dirtyRead.png)

<font color="#0000dd">不可重复读：A事物读取了B事物已提交更改的数据，造成两次出现数据不一致</font>
![不可重复读示例](/images/unrepeatableRead.png)

<font color="#660066">幻读：A事物读取了B事物已提交新增或删除的数据：</font>
![幻读示例](/images/fantasyRead.png)


**隔离性**
> 数据操作过程中可以利用数据库锁机制提高隔离级别，但是随着数据库的隔离级别越高，数据的并发能力也就下降了。

**查看事物的隔离级别**
```$xslt
select @@global.tx_isolation;
```

**事物四种隔离级别的比较**

|隔离级别/读数据一致性及允许的并发副作用|脏读|不可重复读|幻读|
|---|---|---|---|---|
|未提交读(read uncommited)|允许|允许|允许|
|已提交读(read commited)|禁止|允许|允许|
|可重复读(repeatable read)|禁止|禁止|允许|
|可序列化(serializable)|禁止|禁止|禁止|

**JDBC事物实战**

```$xslt
public class TransactionLevels extends BaseJDBC {
    public static void main(String[] args) {
        try {
            // 加载数据库驱动
            Class.forName(DRIVER);
            // 数据库连接
            Connection conn = DriverManager.getConnection(URL,USER,PWD);
            // 数据库元数据
            DatabaseMetaData metaData = conn.getMetaData();
 
            // 是否支持事务
            boolean isSupport = metaData.supportsTransactions();
            System.out.println(isSupport);
            // 是否支持的事务
            boolean isSupportLevel = metaData.supportsTransactionIsolationLevel(Connection.TRANSACTION_SERIALIZABLE);
            System.out.println(isSupportLevel);
            // 获取默认事务
            int defaultIsolation = metaData.getDefaultTransactionIsolation();
            System.out.println(defaultIsolation);
 
            /** 关闭数据库连接 */
            if (conn != null) {
                try {
                    conn.close();
                } catch (SQLException e) {
                    e.printStackTrace();
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
 
    }
}
```


    

