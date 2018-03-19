---
layout: post
title: java基础语法积累
date: 2018-03-19
categories: 语法积累
tags: [Java]
description: java基本语法积累。
---

### StringBuffer与StringBuilder之间的区别
- StringBuffer支持并发操作，线程安全
- StringBuilder费线程安全，但是单线程执行中效率比StringBuffer高

### HashMap与HashTable之间的区别
- HashMap非线程安全，单线程执行效率比HashTable高
- HashTable线程安全，在多线程场景下使用

### set集合

> 无序，且唯一的集合，常用于取两个集合中的并，交，差集

```
public class SetCollection {

    /**
     * 取两个HashSet的交集
     * @param set1
     * @param set2
     * @return Set<String>
     */
    public static Set<String> getIntersection(Set<String> set1, Set<String> set2) {
        Set<String> result = new HashSet<>();
        result.clear();
        result.addAll(set1);
        result.retainAll(set2);
        return result;
    }

    /**
     * 取两个hashSet中的并集
     * @param set1
     * @param set2
     * @return
     */
    public static Set<String> getUnion(Set<String> set1, Set<String> set2) {
        Set<String> result = new HashSet<>();
        result.clear();
        result.addAll(set1);
        result.addAll(set2);
        return result;
    }

    /**
     * 取两个hashSet之间的差集
     * @param set1
     * @param set2
     * @return
     */
    public static Set<String> getDiff(Set<String> set1, Set<String> set2) {
        Set<String> result = new HashSet<>();
        result.clear();
        result.addAll(set1);
        result.removeAll(set2);
        return result;
    }
}
```
**HashSet和TreeSet之间的区别**
- HashSet是无序的，唯一，只能存在一个null值
- TreeSet是有序的，不能存在null值

### arrayList和LinkedList之间的区别
- arrayList是基于动态数组的数据结构,LinkedList是基于链表的数据结构
- 查询操作建议使用arrayList(LinkedList要移动指针)
- 增减操作时建议使用LinkedList(arrayList要移动数据)

### Iterator的用法
```
public class IteratorTest {

    @org.junit.Test
    public void test1() {
        List<String> list = Lists.newArrayList();
        list.add("A");
        list.add("B");
        list.add("C");
        list.add("D");
        //使用Iterator + while
        Iterator<String> it = list.iterator();
        while (it.hasNext()) {
            System.out.println(it.next());
        }

        //使用Iterator + for
        for(Iterator<String> it2 = list.iterator(); it2.hasNext();) {
            System.out.println(it2.next());
        }
    }
}
```

### Java8 optional类用法
**描述**
> 调用一个方法得到返回值去不能直接将这个返回值作为参数去调用另外一个方法，通常情况下要对这个参数进行判空。

**of静态方法**
> 为非null的对象创建一个optional,创建对象时传入的独享不能为null,否则会抛NullPointException
```
Optional<String> op = Optional.of("abc");
//返回 Optional.of(abc)
```

**fromNullable静态方法**
> 为指定的对象创建一个optional,如果对象为null，则返回一个空的optional,与of方法之间的区别是对象可以为null;
```
Optional<String> empty = Optional.fromNullable(null);
//返回 Optional.absent()
```

**isPresent非静态方法**
> 如果optional值存在，返回true,否则返回false;
```
Optional<String> empty = Optional.fromNullable(null);
empty.isPresent();
//返回true
```

**get非静态方法**
> 获取optional值，如果不存在，抛出NoSuchElementException
```
Optional<String> op = Optional.fromNullable("abc");
op.get();
//返回abc
```