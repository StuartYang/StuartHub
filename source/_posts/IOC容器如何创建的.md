---
title: IOC容器如何创建的
mermaid: true
math: false
comments: true
hide: false
date: 2024-12-08 11:16:10
tags:
	- spring
categories:
	- Java
---


又叫自动装配bean的原理。

有三中方式创建IOC容器：

1. ****ClassPathXmlApplicationContext****：从项目的根目录下加载配置文件。传统的xml
2. ****FileSystemXmlApplicationContext****：从磁盘中的加载配置文件。不常用
3. ****AnnotationConfigApplicationContext****：当使用注解配置容器对象时使用此类进行注解读取，创建容器。Springboot的注解开发

## Spring IoC容器的创建流程

环境准备preparefresh方法：

1. preparefresh方法：准备相关sevlet的环境evirment。
2. 对servlet初始化参数进行设置。
3. 检验所设的参数是否为必要参数 ，然后将部分资源，例如mysql的host设置为启动时必须要整的环境变量。
4. 完成监听器和事件的初始化。

通过`obtainFreshBeanFactory方法`和`prepareBeanFactory方法`在获取容器同时在使用BeanFactory方法之前进行一些准备工作：

由于Boot使用`ServletWebServerApplicationContext`作为容器,

之前已经构建好？？？ BeanFactory了？？？—— 存疑

所以obtainFreshBeanFactory方法不不做任何处理。

作为传统的spring项目，很多时候会选择ClassPathXmlApplicationContext作为容器。

每次执行obtainFreshBeanFactrioy时，会通过他的 refreshBeanFactroy方法进行重新构建beanfactory。并且重新加载bean定义BeanDefinitoion。

prepareBeanFactory方法方法中，主要准备类加载器BeanClassLoader，表达啥解析器“BeanExcepessResolver”，配置文件处理器PropertyEditorRegistrar等系统级处理器。 以及两个Bean后置处理器：用来解析Aware接口的ApplicationContextAwarePrecessor和用来自定义监听器注册和销毁的ApplicationListenerDetector。同时会注册一些特殊bean和系统级Bean，例如：

- 容器本身Beanfacotory和ApplicationContext
- 系统环境environment
- 系统属性systemProperties 将他们放入特殊对象池和 单例池中

第四部 通过PostPrecessBeanfactory方法对Beanfactory进行额外设置或者修改，主要定义了request，session在内的servlet相关作用于scopes。同时注册羽 sevlet相关的特殊Bean。sevletRequest，Respose等。

第五步，执行核心的 invokeBeanFactoryPostPrecesssors方法。 ApplicationContextAwarePrecessor的子类BeanFactoryPostProcessor的 子类ConfiigurationClassPostProcessor，通过她加载所有@configuration配置类，同时检索指定的bean扫描路径@componentScans，以及ClassPathDefinitionScanner.doScan()方法，也会扫描@Bean，@import所对应的bean定义。扫描到的Bean存放到Bean定义池BeanDefinitionMap中。后续就可以通过这些bean定义够着相应 的Bean对象啦。

第六部registerBeanPostProcessors方法，检索所有的Bean后置处理器。 同时更具指定的order为他们进行排序。然后放假后置处理器池，beanPostProcessors中。

- 每一个后置处理器 都会在 Bean初始化之前和之后分别执行对应的逻辑。

第七部八，从单例池中获取两个Bean放在ApplicationConetxt中。

- 一个国际化的bean——initMessageSource，可以继承实现自己的。
- 自定义事件广播器-initApplicationEventMulticaster

九-onfresh

先查找 上下了ServletWebServerFactory这个接口的应用服务器Bean。

默认是tomcat，stating方法，服务器开始 运行。

十-registerListeners方法 在Bean中查找所有的监听器Bean，将他们注册到第八构造的消息广播器中。

11 通过finishBeanfactoryinitialzation来生产我们所有的Bean 了。Bean的生命周期。整体分为：

- 构造对象
- 填充属性
- 实例化实例
- 注册对象 Bean生成之后会放假单例池中-singletonObjects中。

12- finishRefresh方法构造并注册“生命周期管理器”lifecycleProcessor。调用所有实现了生命 周期接口LifeCycle的Bean中的start方法。当然，在容器关闭时候也会自动调用对应的stop方法。 ![[Pasted image 20240221225429.png]]
