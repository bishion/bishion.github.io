---
layout: post
title: Activiti 6.0 用户文档 - 第五章：集成Spring
categories: activiti
description: 流程引擎 Activiti 第五章：集成Spring
keywords: 流程引擎, activiti
---
# 集成 Spring
虽然你完全可以脱离Spring使用Activiti，但是我们本章还是要介绍下Activiti中一些非常不错的集成功能。
## ProcessEngineFactoryBean
可以将**ProcessEngine**当做一个普通的Spring bean来配置。集成Spring的切入点是类**org.activiti.spring.ProcessEngineFactoryBean**。这个bean持有一个可以创建流程引擎的配置。这就是说，在Spring中创建和配置所用到的属性跟文档中[配置章节](https://www.activiti.org/userguide/index.html#configuration)中展示的一样。集成Spring的配置和引擎bean看起来是下面这样的:
```xml
