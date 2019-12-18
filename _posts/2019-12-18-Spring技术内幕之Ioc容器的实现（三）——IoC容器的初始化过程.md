---
layout: post
title: 'Spring技术内幕之Ioc容器的实现（三）——IoC容器的初始化过程'
subtitle: 'Spring Framework的核心'
date: 2019-12-18
categories: Spring
cover: 'http://on2171g4d.bkt.clouddn.com/jekyll-theme-h2o-postcover.jpg'
tags: spring java
---

# IoC容器的初始化过程

本系列文章基于对《Spring技术内幕》(第二版)的阅读理解

------

IoC容器的启动过程包括 `BeanDefinition` 的Resource定位、载入和注册三个基本过程

- Resource定位。即容器寻找用户定义Bean的资源位置
- BeanDefinition的载入。即将用户定义好的bean表示成容器内部的数据结构（BeanDefinition）
- 向IoC容器注册这些BeanDefinition。通过调用 `BeanDefinitionRegistry` 接口的实现，在IoC容器内部将BeanDefinition注入到一个hashMap里面，而IoC容器也是通过这个hashMap来持有管理这些BeanDefinition数据

### 3.1 BeanDefinition的Resource定位

