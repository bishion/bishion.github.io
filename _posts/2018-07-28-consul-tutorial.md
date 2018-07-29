---
layout: post
title: Consul 文档翻译
categories: Consul
description: Consul 官网以及文档翻译
keywords: Consl, 翻译
---
# Consul 文档翻译
Consul 作为一款优秀的服务治理框架，受到的关注似乎并没有想象的多。网上关于它的介绍也是寥寥无几，基本上都是围绕 Spring Cloud Consul 来的。  
公司用的服务发现框架就是 Consul，虽然只用了服务发现功能，但是它出色的性能让我着迷。  
为了更深入地学习 Consul，也让更多人了解 Consul，我准备将 Consul 的文档翻译成中文，但愿自己能坚持下来。
# 首页
## 简化服务网格建设
Consul 是一个分布式服务网格(service mesh)，无论是跨平台还是公私有云，他都可以连接，加密和配置服务。  
[下载](#download) 
[开始学习](#started)
[在线演示](https://demo.consul.io/)

## Consul 命令行演示视频  
<video controls="controls" autoplay="autoplay" height="364" width="573">
<source src="/images/consul/connect-video-cli.mp4" type="video/mp4">
<p>Your user agent does not support the HTML5 Video element.</p>
</video>

## Consul UI 演示视频
<video controls="controls" autoplay="autoplay" height="364" width="573">
<source  src= "/images/consul/connect-video-ui.mp4" type="video/mp4">
<p>Your user agent does not support the HTML5 Video element.</p>
</video>

## 动态架构下的基于服务的通信
静态架构向动态架构迁移过程中，通信方式从静态IP访问变成动态的服务发现，安全防护也从静态防火墙变成了动态的服务隔离。
### 静态架构
基于主机IP的通信  
![image](/images/consul/static-net.svg)
### 动态架构
基于服务的通信  
![image](/images/consul/dynamic-net.svg)
## 使用场景
### 服务发现-通信
服务发现可以让服务注册并发现对方。  
[更多](#more-disvocery)
### 服务隔离-安全
通过动态的TLS加密和身份认证保证服务之间的通信安全。  
[更多](#more-segmentation)
### 服务配置-运行时配置
丰富的Key/Value存储功能让服务配置更简洁。    
[更多](#more-configuration)
## Consul 的原则
### API 驱动
服务注册，健康检查，服务授权策略，故障迁移逻辑等等功能均可以通过API调用实现自动化。  
``` bash
$ curl http://localhost:8500/v1/kv/deployment
[
  {
    "LockIndex": 1,
    "Session": "1c3f5836-4df4-0e26-6697-90dcce78acd9",
    "Value": "Zm9v",
    "Flags": 0,
    "Key": "deployment",
    "CreateIndex": 13,
    "ModifyIndex": 19
  }
]
```
### 随处运行和通信
服务间可以跨平台跨公私有云调用，无论是从Kubernetes到虚拟机，还是从容器到Serverless，服务间都可以通讯。
![image](/images/consul/grid_1_1256-2d5a4cdd.webp)
### 扩展和集成
- 在任何架构上部署集群
- 通过代理，服务可以使用TLS协议通信
- 通过可插拔的认证中心提供TLS证书
![image](/images/consul/grid_2_1256-6d4637c2.webp)
# 使用场景
## <span id="more-disvocery">服务发现</span>
## <span id="more-segmentation">服务隔离</span>
## <span id="more-configuration">服务配置</span>
# 介绍
## 什么是 Consul
## Consul 与其他框架对比
## <span id="started">正式入门</span>
# 指南
# 文档
# API
# 社区
# <span id="download">下载</span>