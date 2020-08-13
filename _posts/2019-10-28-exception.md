---
layout: post
title: 异常块执行顺序
date: 2019-10-28
tags: [Java基础]
---

1. 无异常无返回值时先执行try中的逻辑后执行finally中的逻辑

2. 当try或catch中有返回值时：<br/>

- 返回值类型为基本类型时，返回值不受finally影响
- 返回值类型为引用类型时，最终的返回值会被fianlly影响

```java
public class DemoTest {

    public static void main(String[] args) {
        System.out.println(test());
        System.out.println(test2());
    }

    //返回值为基本数据类型
    public static String test() {

        String a = "1";
        try {
            System.out.println("try :" + a);
            a = a + "2";
            return a;
        } catch(Exception e) {
            e.printStackTrace();
        } finally {
            a = a + "3";
            System.out.println("finally : " + a);
        }
        return null;
    }
    //输出 try :1
          fianlly : 123
          12
          

    //返回值为引用类型
    public static List<Integer> test2() {

        List<Integer> list = new ArrayList<>();
        try {
            list.add(1);
            System.out.println("try : " + list);
            return list;
        } catch(Exception e) {
            e.printStackTrace();
        } finally {
            list.add(2);
            System.out.println("finally : " + list);
        }
        return null;
    }
    //输出：try :[1]
           fianlly : [1, 2]
           [1, 2]
}
```

3. 当fianlly中有return时，会直接在finally退出，try和catch中的return会失效




