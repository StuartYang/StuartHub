---
title: SpringBoot启动流程
mermaid: true
math: false
comments: true
hide: false
date: 2022-6-18 10:13:22
tags:
	- spring
categories:
	- Java
---


# 启动入口及其注解说明

```java
@SpringBootApplication
public class SummaryApplication {
    public static void main(String[] args) {
		 SpringApplication.run(SummaryApplication.class, args);  也可简化调用静态方法
    }
}

```

通过源码发现该注解

- @Configuration
- @EnableAutoConfiguration
- @ComponentScan

三个注解的组合，这是在springboot 1.5以后为这三个注解做的一个简写。接下来简单说下这三个注解的功能：

```java
@SpringBootConfiguration // 说明被标注的类是一个配置类
@EnableAutoConfiguration // 开启自动组件扫描，稍后细说
@ComponentScan // 扫描本类所在项目中所有包下的Spring Bean，例如：@Component
public @interface SpringBootApplication {
  ...
}

```

可见，其中最关键的就是`@EnableAutoConfiguration`,我们详细说明一个这个组件：

```java
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {

}

```

该注解也是一个合成注解，由 `@AutoConfigurationPackage` 和 `@Import(AutoConfigurationImportSelector.class)` 构成，其中 ：

- ****@AutoConfigurationPackage**** : 启动时，向`AutoConfigurationPackage`组件中保存一些包路径，方便以后使用。
- ****@Import(AutoConfigurationImportSelector.class)**** : ==该注解也是本次讲解的重点，其目的是将项目以及依赖包下 所有符合条件的`@Configuuration` 配置全部进行加载==。

总结一下：

和我们最想相关的几个组件，其中： ComponentScan用于扫描我们项目中写的@Component等Bean。但是有些Bean我们是自己在配置类中写的呀，该怎么加载进Spring呢？那就用到了AutoConfigurationImportSelector，它将本项目及其依赖中符合条件的配置类（Spring中大量使用到Codition\*注解标注哪些条件下加载我们的配置信息，这就是SPI机制 ）加载进Spring管理，这样手动注入的Bean，也就能让Spring识别啦。可这时，我们的项目本身就不到，不想自己再写一个类作为 配置类，这时候就可以在项目的Main入口类中写入配置信息，那是谁标注这个类 是配置类呢？这就是SpringBootConfiguration起的作用了。

至于AutoConfigurationPackage，就和上文中提及的，用来保存一些全类名包路径，方便spring和我们后续使用。

# SpringBoot的启动流程

SpringBoot项目运行的起点就是Main方法的主启动类。点击运行后，会经历4个阶段：服务构建，环境准备，容器创建，填充容器。

## 服务构建

在SpringApplication的构造方法里，进行服务构建：

```java

public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {

	// 1. 设置 资源加载器
   this.resourceLoader = resourceLoader;
   this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
   // 2. 判断该项目是什么web项目，Tomcat就是Servet类型 ，还有响应式非阻塞服务Reactive，以及其他服务None
   this.webApplicationType = WebApplicationType.deduceFromClasspath();
   // 3. 读取spring.factories文件中的 “上下文初始化” 和“监听器”这个三类配置
   setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
   setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
   // 4. 通过“运行栈”stackTrace判断出main方法所在的类，就是启动类本身，方便后续使用
   this.mainApplicationClass = deduceMainApplicationClass();
}

```

- ApplicationContextInitializer：上下文初始化器
- ApplicationListener：监听器 在`spring-boot`和`spring-boot-autoconfigure`这两个工程中配置了7个“上下文初始化器”和额 8个“监听器”，这些配置信息会****在后续启动过程中使用。****

```shell
--------------- 例：Spring-boot包下的 spring.factories   ---------------------

# Initializers
org.springframework.context.ApplicationContextInitializer=\\\\
org.springframework.boot.autoconfigure.SharedMetadataReaderFactoryContextInitializer,\\\\
org.springframework.boot.autoconfigure.logging.ConditionEvaluationReportLoggingListener

# Application Listeners
org.springframework.context.ApplicationListener=\\\\
org.springframework.boot.autoconfigure.BackgroundPreinitializer

```

总结：

在SpringApplication的构造方法里，主要就干了一件事 就是 将依赖包以及项目下`spring.factories` 中的上下文初始化器和监听器加载进Spring容器中，方便在后续启动过程中使用。

## 环境准备

这个阶段主要在`run`方法 中进行：

```java

public ConfigurableApplicationContext run(String... args) {

   // 1. headless设置为true，表示显示器，键盘等输入设备也可以正常运行
   configureHeadlessProperty();
	/** 2. 在spring.factories中找到SpringApplicationRunListener（EventPublishingRunListener） ,
	并实例化
	**/
   SpringApplicationRunListeners listeners = getRunListeners(args);
   // 启动Spring运行监听器EventPublishingRunListener
   listeners.starting();
   try {
	   // 3. 根据不同的web服务类型会构造不同的运行 环境，默认sevlet
      ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
      // 正式开始构建 环境，并发布环境准备完成事件，之前加载进来的监听器会监听这个事件，并做出处理
      // 例如： CloudFoundryVcapEnvironmentPostProcessor 会去加载spring.factories中的CloudFoundryVcapEnvironmentPostProcessor
      ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
      // 4.  打印banner
      Banner printedBanner = printBanner(environment);
/**

后续是容器创建阶段

*/

/**    context = createApplicationContext();
      exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
            new Class[] { ConfigurableApplicationContext.class }, context);
      prepareContext(context, environment, listeners, applicationArguments, printedBanner);

 **/

/*

填充容器阶段

*/

/**
      refreshContext(context);
      afterRefresh(context, applicationArguments);
      if (this.logStartupInfo) {
         new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
      }
      listeners.started(context);
      callRunners(context, applicationArguments);
   }
   catch (Throwable ex) {
      handleRunFailure(context, ex, exceptionReporters, listeners);
      throw new IllegalStateException(ex);
   }

   try {
      listeners.running(context);
   }
   catch (Throwable ex) {
      handleRunFailure(context, ex, exceptionReporters, null);
      throw new IllegalStateException(ex);
   }
   return context;   **/
}

```

SpringApplicationRunListeners启动后，就可以利用监听器（保护初始化的8个监听器ApplicationListener）监听Spring走到哪一步了，然后做出自己的逻辑判断。等待所有监听器串行处理完成 之后，继续后续逻辑。

## 容器创建阶段

```java

public ConfigurableApplicationContext run(String... args) {

/**
   SpringApplicationRunListeners listeners = getRunListeners(args);
   listeners.starting();
   try {
	  ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
      ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
      Banner printedBanner = printBanner(environment);
**/
/**

后续是容器创建阶段

*/
    // 1. 创建容器——Spring上下文，通过上下文可以获取关于Spring容器中的一切 (创建容器的代码在下一段)
      context = createApplicationContext();
    //  2. 容器创建成功后，创建生产 ， 存放我们bean实例的额“Bean工厂” ——  SpringFactories
    //  创建用来解析@Component，@ComponetScan等租界的泪痣处理器，ConfigurationClassPostcessor
    // 创建用来解析@autowired @value。@inject等注解的自动注解Bean后置解析器AuutoAnnotationBeanPostPrecessor
      exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
            new Class[] { ConfigurableApplicationContext.class }, context);

	/**
		3. 把他们都放入容器中后，通过prepareContext方法对容器中的部分部分属性进行初始化，
		还记得在Springboot构造器中加载进环境的ApplicationContextInitializer和ApplicationListener吗，
		在这里都会传入上下文容器中，完成对容器的初始化。

		完成一系列的加载工作后，会陆续为容器注册bean的引用策略和懒加载策略等，之后通过bean定义加载去启动类在内的资源
		加载到Bean定义池（BeanDefinitionMap）中，方便后续创建实例。

		最后发布完成事件
	**/

 prepareContext(context, environment, listeners, applicationArguments, printedBanner);

/*

后续是填充容器过程

*/

    /**
      refreshContext(context);
      afterRefresh(context, applicationArguments);
      listeners.started(context);
      callRunners(context, applicationArguments);
	  listeners.running(context);
      	**/

      return context;
}

```

容器：指内部有很多不同的属性，集合以及 配套功能的结构体-ApplicationContext

```java
/**

创建容器：根据服务环境 类型的不同，创建出不同的容器，我们这里是默认SERVLET
*/
protected ConfigurableApplicationContext createApplicationContext() {
   Class<?> contextClass = this.applicationContextClass;
   if (contextClass == null) {
      try {
         switch (this.webApplicationType) {
         case SERVLET:
            contextClass = Class.forName(DEFAULT_SERVLET_WEB_CONTEXT_CLASS);
            break;
         case REACTIVE:
            contextClass = Class.forName(DEFAULT_REACTIVE_WEB_CONTEXT_CLASS);
            break;
         default:
            contextClass = Class.forName(DEFAULT_CONTEXT_CLASS);
         }
      }
   }
   return (ConfigurableApplicationContext) BeanUtils.instantiateClass(contextClass);
}

```

## 填充容器

```java
public ConfigurableApplicationContext run(String... args) {
	 // 1. 自动装配，bean实例化IOC ，同时也会启动一个web服务区，方便项目启动，
      refreshContext(context);
      afterRefresh(context, applicationArguments);
		// 2. 发布Bean对象创建完成事件
      listeners.started(context);
      // 3.  回调我们自定义的额Runner接口，例如ApplicationRunner借口
      callRunners(context, applicationArguments);
	  listeners.running(context);
}

```

在这一步中会生产滋生提供依旧项目中自定义的所有Bean对象，并且放在容器中。这个过程就是我们常说的自动装配。其涵盖了“Bean生命周期管理”。

## SpringBoot启动总结

1. 初始化构造器阶段 ：主要是：判断项目服务类型；从`spring.factories`中加载监听器和初始化器，方便后续启动使用。
2. run方法启动节点：
3. 在spring.factories中找到SpringApplicationRunListener（EventPublishingRunListener）构造Spring环境。
4. Spring环境构造完成后，创建ApplicationContext上下文容器，并且将初始化构造器中加载的监听器，初始化器，以及run方法开始的SpringApplicationRunListener传递进容器，完成上下文的参数设置。最后，将项目和依赖包下的bean定义加载进BeanDefinitionMap，方便后续实例化。
5. 容器完成设置后，开始Bean的生命周期：实例化。最后发布Bean对象创建完成事件，回调我们自定义的额Runner接口，例如ApplicationRunner借口
