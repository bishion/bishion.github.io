---
layout: post
title: spring boot 单测
categories: spring
description: spring boot 常见单测方式介绍
keywords: spring boot
---
# spring boot 项目常见单测方式

## 背景
最近组里在推单测，突然想起自己已经三十多年不写单测了，不由心生惭愧。  
默默打开 spring boot 官网，看了看相关单测的文档，整理了下自己的需要的一些单测场景。  
由于内容比较基础，只管怎么做，不问为什么，出于节省时间的目的，建议写过单测的同学可以不用往下看了。

## 代码环境
- spring cloud Edgware.SR5 版本
- 使用 h2 内置数据库
- 持久层使用 mybatis 1.3.3
- spring cloud 组件只使用了 spring cloud config
- 依赖 spring-boot-starter-test 组件
- 依赖 mybatis-spring-boot-starter-test 组件

源码地址：https://github.com/bishion/springboot-test.git

## 场景一：Controller 的单测
入门场景，测试一个Controller，它没有依赖，没有参数。  
桩代码：
```java
@RestController
public class HealthController {

    @RequestMapping("/health")
    public String health(){
        return "UP";
    }
}
```
测试代码：  
```java

