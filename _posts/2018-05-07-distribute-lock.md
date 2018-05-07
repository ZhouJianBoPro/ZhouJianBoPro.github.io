---
layout: post
title: 分布式锁
date: 2018-04-16
categories: distribute
tags: [distribute-lock]
description: 分布式锁实现的几种方式
---

**分布式疼点**
> 分布式的CAP原理告诉我们“任何一个分布式系统不能同时满足一致性（Consistency）,可用性（Availability）及分区容错性（Partition tolerance）”，最多同时满足其中的两项，所以，
很多系统在设计之初要多这三项做出取舍，在绝大多数互联网系统中大多牺牲强一致性来满足系统的高可用性，系统只需要满足最终一致性

**保证系统的最终一致性**
> 使用分布式事物，分布式锁等技术方案来保证数据最终的一致性

**分布式锁需要满足的条件**
1. 互斥性，在任意情况下，只有一个客户能够持有锁
2. 不会发生死锁，即使有一个客户端在持有锁的期间崩溃没有解锁，也能保证后续其他客户端能够加锁
3. 具有容错性，只要大部分节点正常运行，客户端就应该能够正常进行加锁与解锁
4. 可靠性，加锁与解锁必须是同一个客户端，自己的锁不能被其他客户端解锁

**分布式锁的实现方案**
1. [基于缓存实现分布式锁](#cache)
2. [基于数据库实现分布式锁](#database)
3. [基于Zookeeper实现分布式锁](#zookeeper)

<span id="cache"><font color="#dd0000">基于redis实现分布式锁</font><br /></span>
1.加锁代码：
```java
 public class RedisTool {
 
     private static final String LOCK_SUCCESS = "OK";
     private static final String SET_IF_NOT_EXIST = "NX";
     private static final String SET_WITH_EXPIRE_TIME = "PX";
 
     /**
      * 尝试获取分布式锁
      * @param jedis Redis客户端
      * @param lockKey 锁
      * @param requestId 请求标识
      * @param expireTime 超期时间
      * @return 是否获取成功
      */
     public static boolean tryGetDistributedLock(Jedis jedis, String lockKey, String requestId, int expireTime) {
 
         String result = jedis.set(lockKey, requestId, SET_IF_NOT_EXIST, SET_WITH_EXPIRE_TIME, expireTime);
 
         if (LOCK_SUCCESS.equals(result)) {
             return true;
         }
         return false;
     }
 }
```
说明：加锁set()中总共含有五个形参：
* lockKey：用redis中的key当锁，因为key是唯一的
* requestId：value中传的是requestId，requestId可以通过UUID.randomUUID().toString生成，加锁的客户端，我们可以知道这把锁是哪个请求加的，满足了分布式锁的可靠性
* SET_IF_NOT_EXIST：此处填的是NX, 当key不存在的时候加锁，存在时不做任何操作
* SET_WITH_EXPIRE_TIME：此处填的是PX，意思是给该key一个过期的设置，具体是有由expire决定
* expireTime：key过期时间

加锁错误示例：<br/>
使用jedis.setnx()和jedis.expire()组合实现加锁，两条redis命令，不具备原子性，当第一条执行错误时，会导致锁不能释放，造成死锁

2.解锁代码：
```java
public class RedisTool {

    private static final Long RELEASE_SUCCESS = 1L;

    /**
     * 释放分布式锁
     * @param jedis Redis客户端
     * @param lockKey 锁
     * @param requestId 请求标识
     * @return 是否释放成功
     */
    public static boolean releaseDistributedLock(Jedis jedis, String lockKey, String requestId) {

        String script = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end";
        Object result = jedis.eval(script, Collections.singletonList(lockKey), Collections.singletonList(requestId));

        if (RELEASE_SUCCESS.equals(result)) {
            return true;
        }
        return false;
    }

}
```
说明：<br/>
* script字符串是个Lua脚本，并使KEYS[1]赋值为localKey，ARGV[1]赋值为requestId<br/>
* Lua代码功能：获取锁对应的value值，检查是否与requestId相等，如果相等则释放锁，使用Lua语言确保了上述操作是原子性的

错误示例：
```html
public static void wrongReleaseLock1(Jedis jedis, String lockKey) {
    // 任意客户端都能进行解锁，失去可靠性
    jedis.del(lockKey);
}

public static void wrongReleaseLock2(Jedis jedis, String lockKey, String requestId) {
        
    // 判断加锁与解锁是不是同一个客户端
    if (requestId.equals(jedis.get(lockKey))) {
        // 若在此时，这把锁突然不是这个客户端的，则会误解锁
        jedis.del(lockKey);
    }
}
```

<span id="database"><font color="#dd0000">基于数据库实现分布式锁</font><br /></span>


<span id="zookeeper"><font color="#dd0000">基于zookeeper实现分布式锁</font><br /></span>