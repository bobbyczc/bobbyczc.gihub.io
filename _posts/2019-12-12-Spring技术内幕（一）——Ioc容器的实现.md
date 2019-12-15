---
layout: post
title: 'Spring技术内幕（一）——Ioc容器的实现'
subtitle: 'Spring Framework的核心'
date: 2019-12-12
categories: Spring
cover: 'http://on2171g4d.bkt.clouddn.com/jekyll-theme-h2o-postcover.jpg'
tags: spring java
---

# Ioc容器的实现

本系列文章基于对《Spring技术内幕》(第二版)的阅读理解

## 1. Spring Ioc容器概述

### 1.1 Ioc容器和依赖反转模式

**依赖反转**：在面向对象系统中，对象封装了数据和对数据的处理，对象的依赖关系常常体现在对数据和方法的以来上。依赖反转就是把对象的依赖注入交给框架或者Ioc容器来完成。举个例子来说，在新建对象的时候不需要在代码中调用对象的构造函数以及setter函数就能将对象的各种数据注入到新的对象中，极大地减少了重复代码量以及代码耦合度。

**IOC容器**：在Spring中，Ioc容器是实现依赖反转控制的载体，它可以在对象生成或者初始化时直接将数据注入到对象中，也可以通过将对象引用注入到对象数据域中的方式来注入对方法调用的依赖。**这样可以将面向对象编程中需要执行的诸如新建对象，为对象引用赋值等操作交给容器统一完成。**

## 2. Ioc容器系列的设计与实现：BeanFactory和ApplicationContext

在Spring IoC容器的设计中，有两个主要的容器系列：

- 实现了 `BeanFactory ` 接口的简单容器系列：只实现了容器的基本功能
- `ApplicationContext` 应用上下文：作为容器的高级形态而存在，在简单容器的基础上，增建了许多面向框架的特性，同时对应用环境作了许多适配

### 2.1 Spring的IoC容器系列

就像商品需要有产品规格一样，作为IoC容器，也需要为它的具体实现指定基本的功能规范，这个功能规范的设计表现为接口类 `BeanFactory`，它体现了Spring为提供用户使用的IoC容器所设定的基本的功能规范。

就比如一个水桶，必须有最基本的功能，就是能够装水；对于Spring 的具体IoC容器实现，也必须实现最基本的功能，也是就 `BeanFactory ` 接口中定义的基本特性。

### 2.2 Spring IoC容器的设计

下面这张图描述了IoC容器的主要接口设计：简单分析一下

![](https://icon.qiantucdn.com/20191215/d62ce60c653aee1f3bd95505e19273ef2)



1. 接口 `BeanFactory-->HierarchicalBeanFactory-->ConfigurableBeanFactory`，是一条主要的 `BeanFactory` 设计路径。`BeanFactory` 提供最基本的容器规范， `HierarchicalBeanFactory` 增加了双亲IoC容器的管理功能，`ConfigurableBeanFactory` 则在上面的基础上增加了配置功能。
2. 第二条接口设计主线是以 `ApplicationContext` 应用上下文接口为核心的接口设计。`BeanFactory-->ListableBeanFactory-->ApplicationContext-->WebApplicationContext/ConfigrableApplicationContext` 。通过继承 `MessageSource` 、`ResourceLoader` 等接口，在基本容器规范的基础上添加了许多高级容器的功能。

#### 2.2.1 BeanFactory的使用场景

`BeanFactory` 的接口定义如下：提供了一些最基本的IoC容器功能

```java 
public interface BeanFactory {
    //使用转义字符“&”来获取BeanFactory本身这个Bean，用来区分工厂Bean和工厂Bean产生的对象Bean
	String FACTORY_BEAN_PREFIX = "&";
    //通过各种参数获取容器中的Bean
	Object getBean(String name) throws BeansException;
	<T> T getBean(String name, Class<T> requiredType) throws BeansException;
	Object getBean(String name, Object... args) throws BeansException;
	<T> T getBean(Class<T> requiredType) throws BeansException;
	<T> T getBean(Class<T> requiredType, Object... args) throws BeansException;
    //判断容器中是否含有指定name的bean
	boolean containsBean(String name);
    //判断指定名字的bean是否是Singleton类型的，可以在BeanDefinition中指定
	boolean isSingleton(String name) throws NoSuchBeanDefinitionException;
      //判断指定名字的bean是否是Prototype类型的，可以在BeanDefinition中指定
	boolean isPrototype(String name) throws NoSuchBeanDefinitionException;
    //判断指定名字的bean的class类型是不是特定的类型
	boolean isTypeMatch(String name, ResolvableType typeToMatch) throws NoSuchBeanDefinitionException;
    //判断指定名字的bean的class类型是不是特定的class类型
	boolean isTypeMatch(String name, Class<?> typeToMatch) throws NoSuchBeanDefinitionException;
    //获取指定名字的bean的class类型
	Class<?> getType(String name) throws NoSuchBeanDefinitionException;
    //获取指定名字bean的所有别名，别名可以在BeanDefinition里指定
	String[] getAliases(String name);

}
```

`BeanFactory` 接口定义了IoC容器的最基本的形式，并没有给出具体实现，像 `XmlBeanFactory` 等都可以看做是容器附加了某种功能的具体实现。

#### 2.2.2 BeanFactory容器的设计原理

以 `XmlBeanFactory` 的实现为例，简单介绍IoC容器的设计原理

![](https://icon.qiantucdn.com/20191215/a422a425aedf1d0d5f140026ab568e4c2)

`XmlBeanFactory` 顾名思义，就是一个可以读取以XML文件方式定义的 `BeanDefinition` 的IoC容器。源码如下，很简单明了：

```java
public class XmlBeanFactory extends DefaultListableBeanFactory {
	private final XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(this);
	public XmlBeanFactory(Resource resource) throws BeansException {
		this(resource, null);
	}
	public XmlBeanFactory(Resource resource, BeanFactory parentBeanFactory) throws BeansException {
		super(parentBeanFactory);
		this.reader.loadBeanDefinitions(resource);
	}

}
```

- 通过 `XmlBeanDefinitionReader` 读取Resource，该resource是以xml文件格式定义的 `BeanDefinition` 信息
- 通过 `loadBeanDefinitions()` 方法解析resource中的 `BeanDefinition` 信息
- 将获取到的bean注册并载入到容器中

#### 2.2.3 ApplicationContext的应用场景

`ApplicationContext` 在 `BeanFactory` 的基础上添加了新的功能，属于高级形态意义的IoC容器：

- 支持不同的信息源。 扩展了 `MessageResource` 接口
- 访问资源。体现在对 `ResourceLoader` 以及 `Resource` 接口的支持上，可以从不同地方获取Bean定义资源。
- 支持应用事件。继承了接口 `ApplicationEventPublisher` 从而在上下文中引入了事件机制。
- 在 `ApplicationContext` 中提供的附加服务。面向框架，推荐使用 `Applicationcontext` 作为IoC容器的基本形式。

#### 2.2.4 ApplicationContext容器的设计原理

以常用的 `FileSystemXmlApplicationContext` 的实现为例：

