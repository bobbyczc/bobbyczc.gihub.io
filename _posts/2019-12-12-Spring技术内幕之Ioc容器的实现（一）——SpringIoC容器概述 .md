---
layout: post
title: 'Spring技术内幕之Ioc容器的实现（一）——SpringIoC容器概述'
subtitle: 'Spring Framework的核心'
date: 2019-12-12
categories: Spring
cover: 'http://on2171g4d.bkt.clouddn.com/jekyll-theme-h2o-postcover.jpg'
tags: spring java
---

# Spring Ioc容器概述

本系列文章基于对《Spring技术内幕》(第二版)的阅读理解

## 1 Ioc容器和依赖反转模式

**依赖反转**：在面向对象系统中，对象封装了数据和对数据的处理，对象的依赖关系常常体现在对数据和方法的以来上。依赖反转就是把对象的依赖注入交给框架或者Ioc容器来完成。举个例子来说，在新建对象的时候不需要在代码中调用对象的构造函数以及setter函数就能将对象的各种数据注入到新的对象中，极大地减少了重复代码量以及代码耦合度。

**IOC容器**：在Spring中，Ioc容器是实现依赖反转控制的载体，它可以在对象生成或者初始化时直接将数据注入到对象中，也可以通过将对象引用注入到对象数据域中的方式来注入对方法调用的依赖。**这样可以将面向对象编程中需要执行的诸如新建对象，为对象引用赋值等操作交给容器统一完成。**
