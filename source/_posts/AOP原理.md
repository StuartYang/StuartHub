---
title: AOP原理
mermaid: true
math: false
comments: true
hide: false
date: 2023-12-08 11:17:02
tags:
	- spring
categories:
	- Java
---


# 序言

排队，洗头，****理发****，洗头，吹头 ，收费

除了理发外，其他的都是重复公共的流程，做成 公共代码，Spring 利用AOP将公共流程代码切入到主流代码中，每次主流代码的调用，都隐形的完成公共流程代码的执行。

# AOP如何实现的

利用代理，就和理发一样，用户理发时候通过助理完成一系列的服务操作 。代理可以伪装成“主体”，“骗”过调用方，在执行主体前夹带公共流程服务 。 这个 模型在AOP中：

- 助理：代理
- 公共服务 ：增强advice，advices的统称切面
- 在剪发前后 做哪些服务的时机：连接点joinPoint
- 只有 剪发行为 需要洗头等 增强，修剪刘海 就不需要洗头了，哪些欣慰需要被增强，这个规则：切点pointCut

使用时候，通过三个注解就可以轻松使用AOP：@adviced ，@joinpoint ，@pointCut

在实现AOP代理有哪些途径呢？

- AspectJ框架的 静态代理：在 代码运行前通过修改class文件完成代理，功能强大，效率高，运行效率高，但是复杂。有三种实现方式：
- - 编译前：编译前通过aspectJ生成新的代理类，将新类作为正常得类加载到JVM
    - 编译后 ：通过字节码技术重新构建class和jar文件，通常在代理第三方jar的时候会使用到。
    - 加载时：在JVM加载累的时候植入代理。
- Spring AOP的动态代理：在代码运行的时候进行代理。功能受限，有额外的开销，但是使用简单。
- - cglib工具：通过字节码积水在运行时候生成主要类的 代理类。
    - JDK代理：通过实现主类接口，构造一个能够伪装成主类的代理类，****它有一个限制就是代理的主类必修实现接口才可以。****

在原始的Spring中， 如果 主类实现了任意接口，就会用JDK代理；如果没有，就是用cglib工具。

在SpringBoot中 ，无论什么情况都默认使用cglib进行代理。

注意的是，SpringAOP中也会使用到AspectJ的一个注解， 但是在Spring 中这些注解的实现和AspectJ没有什么关系。

# 哪些应用场景会使用到AOP

- 性能统计
- 事务处理
- 缓存层
- 统一异常处理
- 日志
- 权限控制
- ....

# SpringAOP的底层实现

## 使用AOP在 Spring中

通知有五种类型，分别是：

- 前置通知（@Before）：在目标方法调用之前调用通知
- 后置通知（@After）：在目标方法完成之后调用通知
- 环绕通知（@Around）：在被通知的方法调用之前和调用之后执行自定义的方法
- 返回通知（@AfterReturning）：在目标方法成功执行之后调用通知
- 异常通知（@AfterThrowing）：在目标方法抛出异常之后调用通知

__定义切面__

```java
// 定义切面
@Aspect
@Component
public class AopAdvice {

	// 切点
	/**
	execution *.*(..))表示：包路径.任意类.任意方法.任意参数
	**/
    @Pointcut("execution (* com.demo.aop.controller.*.*(..))")
    public void test() {

    }

	// 定义通知
    @Before("test()")
    public void beforeAdvice() {
        System.out.println("beforeAdvice...");
    }
    @Around("test()")
    public void aroundAdvice(ProceedingJoinPoint proceedingJoinPoint) {
        System.out.println("before");
        try {
            proceedingJoinPoint.proceed();
        } catch (Throwable t) {
            t.printStackTrace();
        }
        System.out.println("after");
    }

}

```

__主类__

```java

package com.demo.aop.controller;

@RestController
public class AopController {

    @RequestMapping("/hello")
    public String sayHello(){
        System.out.println("hello");
        return "hello";
    }
}

```

****总结：**** 这样，代码在运行时，会在主体前后，根据通知类型不同，添加公共服务处理。

## 底层实现

部分参见[[Bean的生命周期]]

Bean在Spring 生产中会经历三个阶段：

- 构造对象
- 填充属性
- 初始化实例

所有的后置处理器都在Bean构造完并且填充属性之后执行，其中 就有AOP后置处理器。 在Bean构造完以后都会调用这个后置处理器postPreocessInitialization方法，在这个方法中，Spring为使用AOP的Bean创建代理对象：

1. 通过getAdvicesAndAdvisorsForBean方法获取所有的增强Advice
2. 判断当前bean是否满足我们的配置切面条件
3. 满足条件的Bean，构造代理对象来实现AOP
4. 为了统一进行AOP，Spring专门搭建一个构建生产代理对象的工厂proxyFactory，Spring会告诉这个工厂选择哪种方式进行代理，分别是CGlib或者JDK代理，用户可以在Springboot入口类上添加@EnableAspectproxy注解，并且将其中的proxyTargetClass配置改为true，来强制使用cglib。当然Springboot默认使用cglib，如果proxyTargetClass配置改为false，并且被代理的类实现了任意接口，才能使用JDK代理，否则还会使用cglib。
5. proxyFactory知道用那种方式进行构建代理对象，就会构造出CglibAopProxy对象。然后就可以利用CglibAopProxy对象的getProxy方法获得真正的代理对象。
6. cglib：在CglibAopProxy对象的getProxy方法中，用增强器Enhancer来设置 代理基本 信息以及 增强 方法的调用链， 接下来利用增强器Enhancer的create( )方法来 生成代理对象 。在调用 被代理方法时，会先执行代理对象的intercept方法，通过责任链来执行所有的方法 增强。
