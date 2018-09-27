---
layout: post
title: adapter-parttner
date: 2018-05-16
categories: design patterns
tags: [patterns]
description: 深入了解适配器模式
---

**适配器模式使用场景**
- 想要建立一个可以重复使用的类，将一些彼此之间没有太多关联的类联系一起工作
- 如有两种usb接口互不兼容，需要有个usb转换器能够同时兼容这两种usb接口，这个usb转换器被称之为适配器
 
**模式结构**
> 适配器模式包含的角色

- Target：目标抽象类（需要兼容适配者类接口）
- Adapter：适配器类
- Adaptee：适配者类
- Client：客户类
 
 **适配器的两种实现方式**<br/>
1.类适配器：在适配器类中调用适配者类的方法，适配者类的方式对客户端来说是透明的，由于类单继承，类适配器只能将一个适配者类适配到目标类中
![类适配器](/images/class_adapter.png)

```java
//Target目标抽象类
public interface Target {
    
    public void method();
}

//Adaptee适配者类
public class Adaptee {
    
    public void method1() {
        //逻辑处理
    }
}

//Adapter适配器类
public class Adapter extends Adaptee implements Target {
     
    @Override
    public void method() {
         //逻辑实现
     }
}

//Client类
public class Client {
    
    public static void main(String[] args) {
        
        Target adapter = new Adapter();
        adapter.method();
        adapter.method1();
    }
}

```

2.对象适配器：接口多实现可以把多个适配者类适配到同一个目标
![对象适配器](/images/obj_adapter.png)
```java
//目标类
public interface Target {
    
    void method();
}

//适配者类1
public class Adaptee1 {
    
    public void method1() {
        //逻辑处理
    }
}

//适配者类2
public class Adaptee2 {
    
    public void method2() {
        //逻辑处理
    }
}

//适配器类
public class Adapter implements Target {
    
    private Adaptee1 adaptee1;
    
    private Adaptee2 adaptee2;
    
    public Adapter() {}
    
    public Adapter(Adaptee1 adaptee1, Adaptee2 adaptee2) {
        adaptee1 = new Adaptee1();
        adaptee2 = new Adaptee2();
    }
    
    @Override
    public void method() {
        //要点，将两个适配者类的始实现适配到目标类中
        adaptee1.method1();
        adaptee2.method2();
    }
}

//Client类
public class Client {
    
    public static void main(String[] args) {
        
        Adapter adapter = new Adapter(new Adaptee1(), new Adaptee2());
        adapter.method1();
    }
}
```

**适配器模式的优点**
- 将目标类与适配者解耦
- 客户端对适配者类的调用是透明的

**对象适配器的优点**
> 一个目标类可以被多个适配者类适配，而类适配器的目标类只能被一个适配者类适配