---
layout: post
title: 单例模式 
date: 2018-05-16
tags: [设计模式]
---

**单例模式的优缺点**
1. 单例对象的类必须保证只有一个全局的实例存在，可以节约系统资源，避免频繁创建销毁对象
2. 不适用于变化的对象，会造成数据紊乱

**单例模式使用场景**
1. 应用配置的读取，配置文件是共享的资源
2. 数据库连接池的设计，因为频繁创建和关闭数据库连接会造成资源浪费
3. 线程池的设计，方便对线程池中的线程进行管理

**单例模式的实现方式**<br/>
1.单线程实现单例模式：构造方法私有化，保证只能在类内部实例化对象,多线程会导致重复创建实例
```java
public class Singleton {
    
    private static Singleton singleton = null;
    
    private Singleton singleton() {}
    
    public static Singleton getSingleton() {
        if(singleton == null) {
            return singleton = new Singleton();
        }
        return singleton;
    }
    
}
```
2.多线程实现单例模式（双重检查锁）：对象引用进行两次null判断，由于volatile关键字在jdk1.5之前无法解决指令重排序问题0，java5之前无法完全保证线程安全
```java
public class Singleton {
    
    private static volatile Singleton singleton = null;
    
    private Singleton singleton() {}
    
    public static Singleton getSingleton() {
        if(singleton == null) {
            synchronized (Singleton.class) {
                if(singleton == null) {
                    singleton = new Singleton();
                }
            }
        }
        return singleton;
    }
}
```
3.静态内部类法：将类的实例化放在静态内部类中，避免静态实例在Singleton类加载时就创建对象，实现了延时加载，
并且静态内部类只会加载一次，保证了线程安全
```java
public class Singleton {
    
    private Singleton singleton() {}
    
    private static class Holder {
        private static Singleton singleton = new Singleton();
    } 
    
    private static Singleton getSingleton () {
        return Holder.singleton;
    }
}
```

4.枚举写法
```java
public enum Singleton {
    
    INSTANCE;
    
    private String name;
    
    private Singleton Singleton() {}
    
    public String getName() {
        return name();
    }
    
    public void setName() {
        this.name = name;
    }
    
}
```


