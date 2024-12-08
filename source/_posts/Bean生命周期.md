---
title: Bean生命周期
mermaid: true
math: false
comments: true
hide: false
excerpt: Bean生命周期的主要目的就是帮助使用者妥善 管理和使用这些Bean。
date: 2022-1-08 11:09:23
tags: 
	- spring
categories:
	- Java
---


Bean生命周期的主要目的就是帮助使用者妥善 管理和使用这些Bean。

1. 生产
2. 使用
3. 销毁

## 生产

1. 加载Bean定义-loadbeanDefinitions，用xml，等 各种方法扫描到，放在BeanDefinitionMap
2. 遍历放在BeanDefinitionMap集合，利用createBean方法，创建一个个bean对象。
3. 1. 构造对象：
    2. 1. 通过createBeanInstance方法进行对象的构造，先用反射机制从Bean定义 中的beanClass拿到这个类的构造方法。获取构造方法规则：如果只有一个就拿这一个，无论有没有入参，当有多个 ，拿到@AutoWired的。多个注解就 报错，如果都没有@AutoWired，优先拿没有 入参 的 ，如果多个没有入参的就报错 。
        2. 确定完构造方法之后，就需要准备 这个构造方法的参数了，在单例池中通过Class类进行查找，找到多个，就通过参数名一一匹配。没有找到报酬。最后 就可以通过 反射 对Bean进行构造 。—— 就是实例化
    3. 填充对象：通过populateBean方法 为Bean内部所需的属性进行赋值填充。通常就浴室@autowired注解的这些变量。Spring会通过三级缓冲机制进行填充，就是 所谓的依赖注入。然后就要初始化实例。
    4. 初始化实例：
    5. 初始化容器相关信息前，先通过invokeAwareMethods方法，为实现了各种Aware接口的Bean设置诸如beanNameAare，beanfactoryAware等容器信息。
    6. 通过invokeInitMethods方法执行 Bean的初始化 方法，这个初始化方法 是永久通过实现initalzingBean接口而实现 的 afterPropertiesSet方法（表示 Bean填充属性后执行）。
    7. 执行用户与在 bean上 自定 的@bean方式设置 的initMethed方法。
    8. 在执行initalzingBean方法 之前和之后会对 Bean后置 处理器 BeanPostProcessors进行处理。设置了后置处理器后，Spring会通过applybeanPostProcessorsBeforeInitialzation和applybeanPostProcessorsafterInitialzation方法处理各种Bean的后置处理器。这些处理器包括AOP，负责构造后@postConstruct和销毁前@PreDestroy处理的InitDestroyAnnotatinbeanPostProcessor。这些处理器可以 同priorityOrdered或者Orderd接口进行 执行优先级，小数先先执行。
    9. 注册销毁 ：前面步骤走完时，bean就可以正常使用了，为了让bean正常走完生命周期，Spring会通过registerDisposableBean方法，将实现了销毁接口DisposableBean的Bean进行注册，这样在 销毁时候就可以执行destroy方法了。
4. 创建好的Bean放入单例池中，就可以被获取和使用了。

## 使用

用户就可以在项目中使用Spring管理的Bean了。

## 销毁

当Spring关闭时，和生产时类似，会先执行销毁前处理器PostprocessBeforeDestruction. 就会执行Bean@PreDestroy注解的方法。然后 通过distroyBeans方法逐一销毁所有的 Bean。最后执行 执行 自定义Bean中实现DisposableBean接口 里destroyMethod的方法。