---
title: spring-aop
date: 2022-05-25 22:13:01
tags:
  - spring
categories: 
  - spring
description : Spring-AOP详解
date: 2020-01-01 16:02:20
---

### 动态代理
#### JDK动态代理

JDK动态代理一般需要4个条件 : `目标接口` ,`目标类`,`拦截器`，`代理类`。

```java
// 目标接口
public interface SubjectInterface {
    public String doSomething(String arg1,Integer arg2);
    public String doSomething2(String arg1,Integer arg2);
}

// 目标类 实现目标接口
public class SubjectImpl implements SubjectInterface {
    @Override
    public String doSomething(String arg1, Integer arg2) {
        System.out.println("========SubjectImpl-doSomething============");
        return null;
    }
    @Override
    public String doSomething2(String arg1, Integer arg2) {
        return null;
    }
}

// 拦截器
public class SubjectInvocation implements InvocationHandler{
    private Object object;
    public SubjectInvocation(){}
    public SubjectInvocation(Object o) {
        this.object = o;
    }
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    	System.out.println("=========Before-JDK动态代理对业务进行了增强处理=========");
        Object invoke = method.invoke(object, args);
        System.out.println("=========after-JDK动态代理对业务进行了增强处理=========");
        return invoke;
    }
}

// 代理类，使用Proxy.newProxyInstance方法创建代理类
SubjectInterface subject = new SubjectImpl();
SubjectInvocation invacation = new SubjectInvocation(subject);
SubjectInterface subjectProxy = (SubjectInterface)Proxy.newProxyInstance(
                subject.getClass().getClassLoader(),    // 目标类的ClassLoader
                subject.getClass().getInterfaces(),     // 目标类的接口
                invacation                              // 拦截器
);
subjectProxy.doSomething("thing",1);
```

#### cglib动态代理

xxx

### SpringAop

#### 术语

- target(目标类)，例如上面的SubjectImpl类，需要被代理的类就叫目标类。
- proxy(代理类)， 根据目标对象生成的代理对象。
- JoinPoint(连接点)，所谓连接点是指那些被拦截到的点，可以理解成对象中的方法，因为spring中只支持方法类型的连接点。例如上面`doSomething`方法就是一个JointPoint。
- PointCut(切入点)，所谓切入点就是连接点的一部分，即需要被拦截的连接点就是切入点。例如上面的方法`doSomething`,`doSomething2`，他们两个都是JointPoint。如果我们需要对`doSomething`进行拦截，那这个方法就是PointCut。
- Advice(通知)，通知分为前置通知、后置通知、异常通知、最终通知、环绕通知。
- Aspect(切面)，就是`切入点+通知`的结合。例如对`doSomething`方法做前置通知就表示一个切面。



切点表达式

```java
    // com.crab.spring.aop.demo02包下任何类的任意方法
    @Pointcut("execution(* com.crab.spring.aop.demo02.*.*(..))")
    public void m1(){}
    // com.crab.spring.aop.demo02包及其子包下任何类的任意方法
    @Pointcut("execution(* com.crab.spring.aop.demo02..*.*(..))")
    public void m2(){}
    // com.crab.spring.aop包及其子包下IService接口的任意无参方法
    @Pointcut("execution(* com.crab.spring.aop..IService.*(..))")
    public void m3(){}
    // com.crab.spring.aop包及其子包下IService接口及其子类型的任意无参方法
    @Pointcut("execution(* com.crab.spring.aop..IService+.*(..))")
    public void m4(){}
    // com.crab.spring.aop.demo02.UserService类中有且只有一个String参数的方法
    @Pointcut("execution(* com.crab.spring.aop.demo02.UserService.*(String))")    
    public void m5(){}
    // com.crab.spring.aop.demo02.UserService类中参数个数为2且最后一个参数类型是String的方法
    @Pointcut("execution(* com.crab.spring.aop.demo02.UserService.*(*,String))")    
    public void m6(){}
    // com.crab.spring.aop.demo02.UserService类中最后一个参数类型是String的方法
    @Pointcut("execution(* com.crab.spring.aop.demo02.UserService.*(..,String))")    
    public void m7(){}
```



### 参考

- https://zhuanlan.zhihu.com/p/350194631
- https://blog.csdn.net/f641385712/article/details/88925478
- https://baijiahao.baidu.com/s?id=1714736908373150395&wfr=spider&for=pc
- https://tianxiaobo.com/2018/06/17/Spring-AOP-%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E7%B3%BB%E5%88%97%E6%96%87%E7%AB%A0%E5%AF%BC%E8%AF%BB/