---
title: Spring容器中的核心组件与层次
mermaid: true
math: false
comments: true
hide: false
date: 2023-10-08 11:15:09
tags:
	- spring
categories:
	- Java
---


# Spring的IoC容器

Spring的IoC容器是Spring的核心，IOC（Inversion of Control ）控制反转的。IoC和DI（Dependency Injection）是一个意思，Spring容器会把这些依赖给注入进去。具体来讲，我们可以通过xml文件、注解和java代码的形式来定义对象和它们之间的依赖，Spring会自动帮我们把它们解析成BeanDefinition、注册BeanDefinition，然后将BeanDefinition实例化，填充属性并注入依赖，调用生命周期方法等，最终生成Bean，从而供用户使用。

# 核心概念介绍

Spring是怎么把我们的定义的Java对象变成Spring管理的Bean的呢？这里涉及到Spring中的一些概念，包括Bean、BeanDefinition、BeanFactory、ApplicationContext、BeanDefinitionRegistry、SingletonBeanRegistry、BeanDefinitionReader、BeanFactoryPostProcessor、BeanPostProcessor。

- ****Bean**** 我们这里说的Bean是Spring中的Bean，而不是普通的JavaBean。Spring中的Bean是Spring IoC容器管理的对象，Bean不是普通的java对象，Bean具有生命周期，即创建、初始化和销毁。Spring的容器就是用来管理Bean的。
- ****BeanDefinition**** 在Java中我们使用Class来描述类或接口，但是Class不能描述一个类是否要懒加载、自动装配模式等属性，所以在Spring中使用BeanDefinition来描述Bean实例，BeanDefinition包括Bean属于哪个Class、Bean的Name、Scope、Constructor arguments、Properties、Autowiring mode、Lazy initialization mode、Initialization method、Destruction method。
- ****BeanFactory**** BeanFactory内有一系列的BeanDefination，通过BeanDefination，容器可以返回独立的或共享的实例。支持标准的Bean生命周期接口，即Bean的初始化、创建和销毁。 我们可以看到BeanFactory中提供了许多getBean的方法，此外还提供了判断一个Bean是否是单例、原型等方法。
- ****ApplicationContext**** ApplicationContext是BeanFactory的扩展。除了具备****BeanFactory****的功能外，还具备以下功能：
- - 通过继承ListableBeanFactory和HierarchicalBeanFactory接口，可以访问BeanFactory中的应用组件（也就是Bean）；
    - 通过继承ResourceLoader接口，可以加载资源，例如classPath资源、文件系统资源等；
    - 通过继承ApplicationEventPublisher接口，可以向注册的Listener发布事件；
    - 通过继承MessageSource，可以解析消息，支持国际化；
    - 支持从parent context继承，例如WebApplicationContext。
- ****BeanDefinitionRegistry**** 如果说BeanFactory接口只提供了访问Bean的方法，而BeanDefinitionRegistry则提供了注册BeanDefinition的方法。
- ****SingletonBeanRegistry**** BeanDefinitionRegistry则是向Spring容器中添加BeanDefinition的。则SingletonBeanRegistry接口提供了注册单实例Bean的能力，ConfigurableBeanFactory继承了该接口。
- ****BeanDefinitionReader**** BeanDefinitionReader可以根据传入的String或Resource类型参数加载BeanDefinition。该接口不是一个非必须实现的接口，例如AnnotationConfigApplicationContext（注解开发）中的AnnotatedBeanDefinitionReader就没有实现该接口。
- ****BeanFactoryPostProcessor**** BeanFactoryPostProcessor是Spring提供的一个扩展点，其重要的子接口是BeanDefinitionRegistryPostProcessor。主要是用来修改ApplicationContext中的BeanDefinition而不是Bean实例，允许在BeanFactoryPostProcessor处理BeanDefinition之前注册一些BeanDefinition。 BeanDefinitionRegistryPostProcessor目前唯一的实现是ConfigurationClassPostProcessor，这个类非常重要，主要是用来处理@Configuration标注的配置类的，它处理@Component、@Import、@Bean等注解
- ****BeanPostProcessor**** BeanPostProcessor也是Spring提供的一个扩展点，主要是在Bean实例化前后执行回调

****总结：****

- BeanDefinition：提供Bean的注册。存放Bean原始的class信息。
- BeanFactory：负责从BeanDefinition中获取到Bean的class信息 ，实例化成Bean对象。
- ApplicationContext：封装很多spring的信息，主体是BeanFactory，给用户使用。

下面介绍一下BeanDefinition、BeanFactory、ApplicationContext接口的层次。

## BeanDefinition的简单层次

- ****BeanDefinition**** ：用来描述Bean定义信息，包括属性值、构造器参数值以及它的具体实现提供的额外的信息。
- ****AnnotatedBeanDefinition****： BeanDefinition的直接子接口，主要是可以获取到AnnotationMetadata。
- ****AbstractBeanDefinition****：BeanDefinition接口的抽象类，提取出了BeanDefinition的公共属性，例如：scope，dependsOn，destroyMethodName等。
- ****GenericBeanDefinition****：AbstractBeanDefinition的子类，通常使用它是为了注册用户可见的BeanDefinition。这些BeanDefinition可能会进一步被PostProcessor处理，也可能被重新设置parentName
- ****AnnotatedGenericBeanDefinition****：GenericBeanDefinition的子类，在将@Configuration标注的类（配置类）转化成BeanDefinition时，使用的就是AnnotatedGenericBeanDefinition。
- RootBeanDefinition：一个RootBeanDefinition可能是由多个有继承关系的BeanDefinition创建而来的，
- ChildBeanDefinition：ChildBeanDefinition使得Bean可以从他的parent继承一些属性。使用RootBeanDefinition和ChildBeanDefinition可以表达预先确定好的父子关系。

****总结：AnnotatedGenericBeanDefinition****是****BeanDefinition****的最终实现，由于继承和实现了很多接口，所以扩充了许多额外功能，但是核心功能就是****BeanDefinition****，用来描述Bean的定义信息。

## BeanFactory的层次

- ****BeanFactory****：BeanFactory接口是Spring IoC容器最基本的接口，可以从Spring容器中获取到Bean。BeanFactory只提供了获取Bean的接口，而没有提供注册Bean的接口。****注册Bean的接口是BeanDefinitionRegistry****，****也就是说注册和获取Bean的接口是隔离的。****
- - ****ListableBeanFactory****：ListableBeanFactory是BeanFactory的直接子接口，主要是提供“批量”的能力，例如批量获取BeanName和Bean实例。
    - ****HierarchicalBeanFactory**** ：HierarchicalBeanFactory是BeanFactory的直接子接口，主要提供支持父子容器的能力，即可以获取当前容器的父容器。
    - ****AutowireCapableBeanFactory****：AutowireCapableBeanFactory是BeanFactory的直接子接口，主要提供自动注入的能力。
- ****AbstractBeanFactory****：AbstractBeanFactory是BeanFactory的抽象基类，提供了基本的实现
- ****ConfigurableListableBeanFactory****和****ConfigurableBeanFactory****：为BeanFactory的一系列配置方法。例如：setBeanClassLoader、addBeanPostProcessor....
- ****DefaultListableBeanFactory****：ConfigurableListableBeanFactory和BeanDefinitionRegistry接口的默认实现，在AnnotationConfigApplicationContext中会用到。

## ApplicationContext的层次

- ****ApplicationContext****：ApplicationContext是配置一个应用的中心接口。
- ****WebApplicationContext**** ：WebApplicationContext为web应用提供配置，在ApplicationContext的基础上增加了getServletContext方法，可以获取到标准的javax.servlet.ServletContext。
- ****AbstractApplicationContext****：AbstractApplicationContext是ApplicationContext的抽象实现，提供了通用能力。
- ****GenericApplicationContext****：我们常用的基于注解的AnnotationConfigApplicationContext是GenericApplicationContext的子类。
- ****AnnotationConfigApplicationContext****：AnnotationConfigApplicationContext支持以@Configuration标注的类、@Component注解作为BeanDefinition源。如果有多个@Configuration标注的类，那么后面的类中定义的@Bean方法会覆盖前面的类。利用这点可以通过增加额外的@Configuration类来覆盖某些BeanDefinition。
