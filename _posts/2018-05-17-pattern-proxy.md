---
layout: post
title: 代理模式
date: 2018-05-17
tags: [designMode]
description: 深入了解代理模式
---

**代理模式的优点**
> 职责清晰，真实角色只需要关注业务逻辑实现，非业务逻辑部分可后期交给代理角色实现

**简单示例说明**
> 小张是个程序员，有一天客户向他提出一个需求需要近期完成，小张没有立即答应，而是将情况汇报给项目经理，
有项目经理决定该需求是否在范围之内；

**静态代理**
- 抽象角色
- 真实角色（完成任务的角色）
- 代理角色

抽象角色：基于面向对象思想，首先定义一个接口
```java
public interface ICoder {
 
    public void implDemands(String demandName);
 
}
```

真实角色：用于完成逻辑处理
```java
public class JavaCoder implements ICoder{
 
    private String name;
 
    public JavaCoder(String name){
        this.name = name;
    }
 
    @Override
    public void implDemands(String demandName) {
        System.out.println(name + " implemented demand:" + demandName + " in JAVA!");
    }
}
```

代理角色：实现抽象角色
```java
public class ProxyCoder implements ICoder {
    
    private ICoder coder;
    
    public ProxyCoder(ICoder coder) {
        this.coder = coder;
    }
    
    @Override
    public void implDemands(String demandName) {
        
        //代理角色也可以对任务进行处理，比如不接受该任务
        if(demandName.startsWith("Add")){
            System.out.println("No longer receive 'Add' demand");
            return;
        }
        coder.implDemands(demandName);
    }
}
```

**动态代理**
> 与静态代理相比，抽象代理的抽象角色和真实角色并没有发生改变，改变的是代理角色，代理角色由静态代理变为动态代理，
动态代理类要求实现InvocationHandler接口，并重写invoke方法；

```java
public class CoderDynamicProxy implements InvocationHandler{
     //被代理的实例
    private ICoder coder;
 
    public CoderDynamicProxy(ICoder _coder){
        this.coder = _coder;
    }
 
    //调用被代理的方法，method标识了调用代理类的那个方法，args是参数
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println(System.currentTimeMillis());
        Object result = method.invoke(coder, args);
        System.out.println(System.currentTimeMillis());
        return result;
    }
}
```

通过场景类动态生成代理类：
```java
public class DynamicClient {
 
     public static void main(String args[]){
            //要代理的真实对象
            ICoder coder = new JavaCoder("Zhang");
            //创建中介类实例
            InvocationHandler  handler = new CoderDynamicProxy(coder);
            //获取类加载器
            ClassLoader cl = coder.getClass().getClassLoader();
            //动态产生一个代理类
            ICoder proxy = (ICoder) Proxy.newProxyInstance(cl, coder.getClass().getInterfaces(), handler);
            //通过代理类，执行doSomething方法；
            proxy.implDemands("Modify user management");
        }
}
```

**动态代理**
1. 创建抽象角色
2. 创建真实角色
3. 通过实现InvocationHandler创建中介类（重写invoke方法）
4. 通过场景类，动态生成代理类