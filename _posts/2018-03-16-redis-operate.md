---
layout: post
title: redis对数据类型的操作
date: 2018-03-16
categories: cache
tags: [cache]
description: redis对数据类型的操作。
---

**key操作命令**
- exists(key)：判断该key是否存在，Integer
- type(key)：返回值的类型
- del(key)：删除key
- keys(parttern)：返回满足parttern格式的所有key
- expire(key, seconds)：给key设置活动时间，超过该事件key失效
- ttl key：获取key的活动时间

**string类型**
- setex：设置ttl
- setnx：当key不存在保存，key存在时不覆盖value

**map类型**
- hmset：存储map
- hmget：取出map

**list类型**
- lpush：一般将对象转换成字符串存入
- lrang：通过下标取出

```html
jedis.lpush(key, new Gson().toJson(object));
jedis.expire(key, EXPIRE_SECONDS);

List<String> values = jedis.lrange(key, 0, -1);
List<Object> = Lists.transform(values, new Function<String, Object>() {
    @Override
    public Object apply(String json) {
        return new Gson().fromJson(json, Object.class);
    }
});
```
**set类型**
- sadd：列表设置一个或多个值
- smembers: 取出所有成员

**sorted set类型**
- zadd：有序列表中设置一个或多个值
- zrange key 0 -1 [WITHSCORES]



