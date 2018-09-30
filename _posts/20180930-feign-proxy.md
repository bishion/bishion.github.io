---
layout: post
title: Spring Cloud FeignClient 代理方式探究
categories: springcloud
description: 探索 Spring Cloud FeignClient 的代理方式
keywords: springcloud, 微服务, 源码, feign, feignclient
---
# Spring Cloud FeignClient 代理方式探究
## 背景
每次跟人讲起 feignClient 的大致原理，我都是含糊其词：
> 程序启动时，Spring 会为每个加了 *@FeignClient(name="provider")* 注解的接口生成一个代理 bean， 名称为注解的 name 属性（本例为 *provider* ），方法就为接口的方法列表。等你执行接口某个方法的时候，代理的方法就会帮你做请求和响应的参数拼接以及 HTTP 请求

直到最近被一个同事打破砂锅问到底：
1. 生成的代理类的类型是啥样的
2. 使用的代理组件是 jdk，还是 cglib

我也一脸懵逼，虽然上面那套说辞大体上能自圆其说，但是很多细节都没有考虑到：
1. jdk 动态代理

