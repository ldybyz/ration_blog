---
title: "我对Spring Bean的理解"
date: 2022-09-22T16:09:24Z
draft: false
categories: ["编程相关"]
tags: ["java"]
---

## 废话
最近因为在准备面试，决定研究一下Spring,就从Bean开始。找了一些书看，又看了一些博客，好多都是贴一大堆源码，似懂非懂。接着我又试着看源码，打断点调试，有些代码总感觉很杂，用了很多while,嵌套了很多层。我想，这大概是因为bean类型和涉及的范围广，所以才会涉及到那么多类，比如BeanFactoryPostProcessor，BeanFactory，BeanPostProcessor，Aware接口。就让我把是懂非懂的想法写下来，后面有错误的地方会修改。
## 什么是Spring Bean
官方的说法，Bean就是由IOC容器初始化、装配及管理的对象。我的理解是，一个普通的类可以封装成一个Bean，然后放置在Spring容器里，需要用到这个类时，可以直接从容器里把这个对象取出来，一般是根据Bean的唯一名称（源码里DefaultListableBeanFactory是用了Map结构来放置）。

## 怎么定义一个Spring Bean
1.基于xml配置装配：通过XML添加一个bean标签

2.基于Java代码装配：创建一个配置类（JavaConfig），再用@Bean注解用于声明一个Bean（可以声明多个）。

3.基于注解的装配（自动化装配）：使用@Component注解表明该类是一个组件类，它将通知Spring为该类创建一个Bean。再使用@ComponentScan注解启用组件扫描（另外一个类）


## Spring Bean 定义（BeanDefinetion)是一种怎样的结构
我使用Spring Boot Actuator的beans接口拿到了bean的结构，我的理解是bean定义。json结构如下所示。字段分别是别名，作用域，类型全名，来源（一般是JavaConfig类或者没有），依赖（一个类的创建需要其他类，这里用bean名称）
```
{ "applicationTaskExecutor":{        
"aliases":["taskExecutor"],   
"scope":"singleton",  
"type":"org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor",  "resource":"class path resource [org/springframework/boot/autoconfigure/task/TaskExecutionAutoConfiguration.class]",
"dependencies":["taskExecutorBuilder"]    
}}
```
## Bean的作用域
Bean的作用域指的是在程序运行的一段时间内，同一个Bean是多次创建，还是只创建一次，具体是根据什么来区分的。几种常见的作用域如下：

1.singleton:单例，一个Spring容器中只有一个Bean的实例，程序默认配置

2.propotype:每次调用新建一个Bean的实例

3.request:每次web请求都会新建一个Bean的实例

4.session:每个会话都会新建一个Bean的实例

## Bean的生命周期
《Spring Beans in Depth》博客里把生命周期分为11个步骤，也有其他分法的。这里为了便于理解，分成六个步骤：

1.获取bean定义：通过XML,注解获取Bean定义

2.bean实例化： Spring实例化bean对象，并将对象加载到ApplicationContext中
3.填充bean属性：设置bean定义的相关属性，比如name等。同时扫描Aware接口,在属性设置时作一些处理。例如，设置bean名称的时候，可以打印bean名称出来。
4.初始化bean:Spring的BeanPostProcessors在这个阶段开始起作用。这里我们可以在执行前，执行后自定义一些操作。
5.准备好：bean已经创建成功，并注入到所有的依赖里。
6.销毁bean:我们同样可以在这个阶段进行一些处理，比如数据库连接关闭。


## 其他
1.Spring Boot Web程序启动后，一般来说容器里的Bean是已经定义好的。但是通过一些监听器能够重新创建Bean,例如配置的修改触发事件。
2.Spring容器还提供了Bean缓存和Bean懒加载等功能。


## 参考
[Spring Beans in Depth](https://medium.com/javarevisited/spring-beans-in-depth-a6d8b31db8a1)

[Spring Bean 生命周期之“我要到哪里去”？](https://mp.weixin.qq.com/s?__biz=MzkwNzI0MzQ2NQ==&mid=2247488903&idx=1&sn=87b83ebeba6be8250f1c9c8686186d87&source=41#wechat_redirect)
