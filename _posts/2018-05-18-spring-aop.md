---
layout: post
title: spring aop实现原理
date: 2018-05-18
categories: spring
tags: [spring]
description: spring面向切面编程
---

**aop存在的价值**
![](/images/aop.png)
如上图所示，将三个方法相同的代码部分重新定义成一个方法，然后在三个方法中分别调用；只要修改一个地方，
调用方就不用修改了；但是有些特殊的情况：应用需要调用方彻底与被调用方分离，比如为其中两个方法增加事务管理，
安全检查、日志记录等需求，这时得修改多个调用方的代码，aop已经成为一种良好的解决方案


**概念**
> spring aop需要对目标类进行增强，生成新的aop代理类，spring aop无需使用特殊命令对java源代码
进行编译，它在运行时动态的，在内存中临时生成代理类的方式来生成aop代理

**aop相关术语**<br/>

|术语|中文|描述|
|---|---|---|
|joinpoint|连接点|被拦截到的点，在spring中称之为方法|
|pointcut|切入点|需要被配置joinpoint|
|advice|通知|拦截到joinpoint要做的操作，前置通知，后置通知，异常通知，最终通知，环绕通知|
|aspect|切面|切入点和通知的结合|
|target|目标对象|需要被代理的对象|
|wevaing|织入|把通知应用到目标对象来创建代理对象|
|introduction|引介|一种特殊的通知，在不修改类代码前提下，在运行的时候动态的增加一些方法或字段（不常用）|

**AspectJ五大通知（advice）**
- @Before：前置通知，在方法执行之前执行
- @After：后置通知，在方法执行之后执行
- @AfterReturning：返回通知，在方法返回结果之后执行
- @AfterThrowing：异常通知，方法执行异常之后执行
- @Around：环绕通知，围绕着方法执行

**自动增强**
> spring会判断一个或多个方面（事物管理、日志录入、安全检查，拦截器等）是否需要多目标进行增强，并自动生成相应的代理，从而使增强处理在合适
的时候被调用

1.为了启用spring对@Aspect方面配置的支持，并保证spring容器中的目标Bean能够被一个或多个方面自动增加，在spring配置中增加以下配置
```xml
<?xml version="1.0" encoding="GBK"?> 
<beans xmlns="http://www.springframework.org/schema/beans"
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
xmlns:aop="http://www.springframework.org/schema/aop"
xsi:schemaLocation="http://www.springframework.org/schema/beans 
http://www.springframework.org/schema/beans/spring-beans-3.0.xsd 
http://www.springframework.org/schema/aop 
http://www.springframework.org/schema/aop/spring-aop-3.0.xsd"> 
<!-- 启动 @AspectJ 支持 -->
<aop:aspectj-autoproxy/> 
</beans>
```
2.如果不打算使用Spring的xml schema配置方式，则应该在spring配置文件中增加如下片段来启用@AspectJ支持
```xml
<!-- 启动 @AspectJ 支持 -->
<bean class="org.springframework.aop.aspectj.annotation.AnnotationAwareAspectJAutoProxyCreator"/>
```
3.当启用了@AspectJ支持后，只要我们在spring 容器中配置一个@Aspect注解的Bean，spring会自动识别该Bean，
并会将该Bean作为方面Bean处理
```java
// 使用 @Aspect 定义一个方面类
@Aspect 
public class LogAspect 
{ 
// 定义该类的其他内容
}
```
方面Bean和其他Bean一样，可以定义常量方法，还能对切入点（pointcut），增强（advice）处理进行定义，
当spring容器检测到某个方面Bean被@Aspect注解后，spring不会对该方面Bean进行增强，，方面Bean是对目标Bean进行增强的<br/>


**示例：**
```java
//目标Bean
public class TargetBean {
    
    public void method() {
        //逻辑处理
    }
}
```
提供上面的目标Bean后，接下来要对目标Bean的方法进行日志记录，事物控制，可以考虑使用AfterReturning通知进行增强处理<br/>

定义方面Bean采用AfterReturning通知为目标类增强日志记录：
```java
@Aspect
public class LogAspectBean {
    
    //匹配com.commons.service.impl包下所有类
    //所有方法的执行作为切入点
    @AfterReturning(returning = "obj", pointcut = "execution(* com.commons.service.impl.*.*(...))")
    public void log(Object obj) {
        System.out.println("获取目标方法的返回值：" + obj);
        System.out.println("模拟记录日志功能...");
    }
    
}
```
Client客户端调用
```java
public class Cleint {
    
    public static void main(String[] args) {
        ApplicationContext ctx = new ClassPathXmlApplicationContext("beam.xml");
        TargetBean tb = ctx.getBean("targetBean", TargetBean.class);
        tb.method1();
        System.out.println(tb.getClass());//com.commons.service.impl.TargetBean$$EnhancerByCGLIB$$290441d2
    }
}
```

总结：<br/>
1. 上面的Aspect类使用了@Aspect修饰，这样spring容器会将该Bean处理为方面Bean，com.commons.service.impl包下
所有的方法都会自动织入log(Object obj)方法
2. 虽然程序是调用了TargetBean的方法，但是实际执行的绝对不是TargetBean对象的方法，而是aop代理的方法，
也就是说，spring aop为TargetBean生成了aop代理类
3. 由tb.getClass()打印可以知道该代理类是有cglib生成的

如果将上面的程序稍作修改：只要让上面的业务逻辑类实现一个任意的接口（更符合spring的面向接口编程），
假设程序为TargetBean实现ITargetBean接口：
```java
public interface ITargetBean {
    
    void method1();
}

public class TargetBean implements ITargetBean {
    
    @Override
    public void method1() {
        //逻辑处理
    }
}
```
接下来Client类是面向ITargetBean编程，而不是面向对象编程，即将Clinet类改成以下格式：
```java
public class Client {
    
    public static void main(String[] args) {
        ApplicationContext ctx = new ClassPathXmlApplicationContext("bean.xml");
        ITargetBean p = ctx.getBean("targetBean", ITargetBean.class);
        System.out.println(p.getClass());//class $Proxy7
    }
}
```
总结：
1. 如果目标类实现了接口，那么spring aop会采用JDK动态代理的方式来生成代理类
2. 如果目标类没有实现接口，那么spring aop会采用cglib动态代理的方式来生成代理类
3. 被final修饰的类不能通过spring aop框架动态生成代理类

**spring aop原理剖析**
> aop代理其实是由aop框架动态生成的一个对象，该对象可作为目标对象使用，aop代理包含了目标对象的全部方法，
aop代理中的方法在特定切入点增加了增强处理

spring的aop代理由spring ioc容器负责生成，管理，aop代理可以直接使用容器中的其他Bean实例作为目标对象，
这种关系可由ioc容器的依赖注入提供<br/>

aop编程，程序员只需关注一下几点：
- 定义普通业务组件
- 定义切入点，一个切入点可以横切多个业务组件
- 定义增强处理
- 代理对象的方法 = 增强处理 + 被代理的目标对象
- aop实现原理：aop框架负责动态生成aop代理，该代理类的方法由advice，目标对象的方法组成


