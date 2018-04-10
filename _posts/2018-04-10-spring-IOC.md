---
layout: post
title: spring依赖注入(DI)及控制反转(IOC)梳理
date: 2018-04-04
categories: spring
tags: [spring]
description: spring依赖注入及控制反转概念梳理。
---

**DI是什么**
1. 依赖注入是控制反转实现的一种方式，将实例变量传入到一个对象中去
2. spring核心就是依赖注入，spring支持的注入方式：setter注入、构造器注入

**IOC是什么**
1. IOC是一种思想，将你设计好的对象交给IOC容器控制，而不是在对象内部通过new主动去创建对象
2. IOC容器控制对象的创建，同时控制了外部对象的获取
3. 正转：传统应用程序是由我们自己在对象中直接获取依赖对象；反转：容器帮我们创建及依赖注入对象，依赖对象的获取被反转了

**代码示例**<br/>
<font color="#dd0000">1.公共代码：</font>
1.1、提供一个DataFinder接口：
```java
public interface DataFinder {
    void getData();
}
```
1.2、MysqlDataFinder实现：
```java
public class MysqlDataFinder implements DataFinder{
 
    @Override
    public void getData() {
        //此处具体实现省略
    }
}
```
1.3、OracleDataFinder实现：
```java
public class OracleDataFinder implements DataFinder{
 
    @Override
    public void getData() {
        //此处具体实现省略
    }
}
```

<font color="#dd0000">2.传统编程实现：</font>
```java
public class Example {
 
    private DataFinder dataFinder;
 
    public Example(){
    }
     
    public Example(int type){
        if(type == 1){
            dataFinder = new MysqlDataFinder();
        }else if(type == 2){
            dataFinder = new OracleDataFinder();
        }
    }
     
    public void doStuff() {
        // 此处具体实现省略
        dataFinder.getData();
    }
}
 
public class Client {
 
    public static void main(String[] args){
         
        Example example1 = new Example(1);
        example1.doStuff();
         
        Example example2 = new Example(2);
        example2.doStuff();
    }
}
```
缺点：Example类依赖MysqlDataFinder和OracleDataFinder具体实现，耦合度较高；实际上Example不应该有这么多控制逻辑，
只负责调用doStuff()即可，不需要关注DataFinder的类型，我们应该试着将调用DataFinder类型类型交给Client类。

<font color="#dd0000">3.IOC实现：</font>
```java
public class Example {

    private DataFinder dataFinder;
 
    public Example(){
    }
     
    public Example(DataFinder dataFinder){
        this.dataFinder = dataFinder;
    }
     
    public void doStuff() {
        // 此处具体实现省略
        dataFinder.getData();
    }
}
 
public class Client {
 
    public static void main(String[] args){
         
        Example example1 = new Example(new MysqlDataFinder());
        example1.doStuff();
         
        Example example2 = new Example(new OracleDataFinder());
        example2.doStuff();
    }
}
```
优势：这样Example就不用依赖具体类的实现，将Example对于选择哪个具体实现DataFinder控制权转移至Client中，有Client决定调用哪个实现类完成任务。

**传统编程与IOC比较**
- 传统编程：决定使用哪个类的控制权是在调用类本身，在编译阶段就确定了
- IOC模式：调用类只依赖与接口，而不依赖具体类的实现，减少了耦合度
- 总而言之，两者区别在于接口调用时是否要依赖其实现类




