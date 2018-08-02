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
## <span id="more-disvocery">服务发现变得简单</span>
服务注册，内置健康检查，DNS和HTTP接口，让服务发现变得更加简单。  
[下载](#download) [查看文档](#started-service)
### 挑战
在动态的世界中，服务负载变得不再高效   
负载均衡经常被用在服务的上层，然后提供一个静态的IP给客户端调用。它增加成本和延时，引入了单点故障，并且必须随着服务扩缩容而更新。
![image](/images/consul/use-case-discovery-challenge.svg)
### 解决方案
动态架构下的服务发现  
动态架构下，服务发现是取代负载均衡的最佳选择。注册中心来维护一个实时的服务列表，包括服务地址和服务的健康状况。客户端通过注册中心查询服务端的地址，然后直接跟服务端通信。这就达到了脱离负载均衡也能弹性括缩容和优雅容错的目的。
![image](/images/consul/use-case-solution.svg)
### 特点
#### 服务注册中心
Consul 提供了一个注册中心，我们可以看到所有的运行节点和服务还有它们的健康状况。通过它的 HTTP 接口，运维人员可以了解环境，了解那些与动态架构交互的应用程序和自动化运维工具。
[了解更多](#started-service)
![image](/images/consul/ui-health-checks.jpg)
#### DNS 查询接口
Consul 使用内置的 DNS 服务器来实现服务发现。因为几乎所有的应用都支持使用 DNS 来做 IP 解析，这就让现有的应用很容易集成。使用 DNS 代替静态 IP 地址，可以让服务弹性扩缩容和绕过故障节点变得更加简单。
[了解更多](#started-service-query)
```bash

$ dig web-frontend.service.consul. ANY

; <<>> DiG 9.8.3-P1 <<>> web-frontend.service.consul. ANY
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 29981
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;web-frontend.service.consul. IN ANY

;; ANSWER SECTION:
web-frontend.service.consul. 0 IN A 10.0.3.83
web-frontend.service.consul. 0 IN A 10.0.1.109
```
#### 带有边界触发器的 HTTP 接口
Consul 提供了 HTTP 接口，供用户从注册中心查询节点，服务及其健康信息。该接口还支持阻塞查询或者对更改进行轮询。它允许自动化工具对正在注册或者健康状态发生变化的服务实时地修改配置或者重新路由。
[了解更多](#started-service-http)
```bash
$ curl http://localhost:8500/v1/health/service/web?index=11&wait=30s
{
  ...
  "Node": "10-0-1-109",
  "CheckID": "service:web",
  "Name": "Service 'web' check",
  "Status": "critical",
  "ServiceID": "web",
  "ServiceName": "web",
  "CreateIndex": 10,
  "ModifyIndex": 20
  ...
}
```
#### 多数据中心
Consul 支持开箱即用的多数据中心，而且不需要复杂的配置。在另一个数据中心查询服务，跟查询本地服务一样。预查询(Prepared Queries)等高级功能可以自动将失败查询转移到其他数据中心。
[了解更多](#guides-datacenter)
```bash
$ curl http://localhost:8500/v1/catalog/datacenters
["dc1", "dc2"]$ curl http://localhost:8500/v1/catalog/nodes?dc=dc2
[
    {
        "ID": "7081dcdf-fdc0-0432-f2e8-a357d36084e1",
        "Node": "10-0-1-109",
        "Address": "10.0.1.109",
        "Datacenter": "dc2",
        "TaggedAddresses": {
            "lan": "10.0.1.109",
            "wan": "10.0.1.109"
        },
        "CreateIndex": 112,
        "ModifyIndex": 125
    },
...
```
#### 健康检查
将服务发现和健康检查结合，就可以防止请求被路由到不健康的主机，继而轻松实现断路器功能。
[了解更多](#guides-health-check)
![image](/images/consul/ui-health-checks.jpg)
## <span id="more-segmentation">服务隔离变得简单-*新特性*</span>
自动TLS加密和身份认证让服务间访问变得更加安全
[下载](#download) [查看文档](#docs-connect)
### 挑战
防火墙模式在动态配置下伸缩性不强。  
服务间防火墙基于 IP 访问限制来保护入网和出网流量。但是在动态的世界里，服务随时会创建和销毁，防火墙的扩展性很低，因为它带来的是复杂的网络拓扑和随时变换的防火墙规则。
![image](/images/consul/use-case-secure-firewall.svg)
### 解决方案
基于服务隔离的动态服务鉴权。  
服务隔离是让服务自己管理访问限制，而不是依赖网络。Consul 使用服务策略规定哪些服务有权限访问。这些策略摆脱了IP规则和网络架构的束缚，可以在数据中心和大集群之间任意扩展。
![image](/images/consul/use-case-secure-segment.svg)
### 特点
#### 服务调用图
定义和强制服务间的访问使用 Intentions 配置。基于服务的规则比基于 IP 的规则更容易管理变来变去的动态架构。  
[了解更多](#docs-connect-intention)  
![image](/images/consul/ui-intentions-list.webp)
#### 跨平台服务防护
新旧平台之间的安全通信。Sidecar 可以让你在不改代码的情况下将它集成进应用程序，4层协议兼容所有的常见协议。grid_3_1256-1cecf8c0.webp
[了解更多](#docs-connect-proxies)  
![image](/images/consul/grid_3_1256-1cecf8c0.webp)
#### 基于证书的服务识别
使用 TLS 鉴别服务和加密通讯。SPIFFE 格式的证书可以更好地跟其他平台协作。Consul 可以作为一个授权中心来简化开发，或者与 Vault 这样的外部认证中心整合。  
[了解更多](#docs-connect-ca)  
![image](/images/consul/vault.png)![image](/images/consul/spiffe.png)
#### 通信加密
所有的服务间通信都经过 TLS 加密和鉴权。TLS 保证服务能正确识别，并对通信数据进行加密。
[了解更多](#docs-connect-ca)  
```bash
$ consul connect proxy -service web \
        -service-addr 127.0.0.1:8000
        -listen 10.0.1.109:7200
==> Consul Connect proxy starting...
    Configuration mode: Flags
                Service: web
        Public listener: 10.0.1.109:7200 => 127.0.0.1:8000
...
$ tshark -V \
        -Y "ssl.handshake.certificate" \
        -O "ssl" \
        -f "dst port 7200"
Frame 39: 899 bytes on wire (7192 bits), 899 bytes captured (7192 bits) on interface 0
Internet Protocol Version 4, Src: 10.0.1.110, Dst: 10.0.1.109
Transmission Control Protocol, Src Port: 61918, Dst Port: 7200, Seq: 136, Ack: 916, Len: 843
Secure Sockets Layer
    TLSv1.2 Record Layer: Handshake Protocol: Certificate
        Version: TLS 1.2 (0x0303)
        Handshake Protocol: Certificate
          RDNSequence item: 1 item (id-at-commonName=Consul CA 7)
              RelativeDistinguishedName item (id-at-commonName=Consul CA 7)
                  Id: 2.5.4.3 (id-at-commonName)
                  DirectoryString: printableString (1)
                      printableString: Consul CA 7
```
## 更简单的配置
支持富配置中心
### 挑战
运行时的配置管理在大规模集群下性能不高。  

### 解决方案
# 介绍
## 什么是 Consul
## Consul 与其他框架对比 
## <span id="started">正式入门</span>
### <span id="started-service">服务</span>
#### <span id="started-service-query">服务查询</span>
#### <span id="started-service-http">HTTP 接口</span>
### <span id="started-health-check">健康检查</span>
# 指南
## <span id="guides-datacenter">多数据中心</span>
# 文档
## <span id="docs-connect">连接</span>
### <span id="docs-connect-intention">Intentions</span>
### <span id="docs-connect-proxies">代理</span>
### <span id="docs-connect-ca">认证管理</span>
### <span id="docs-connect-security">安全</span>
# API
# 社区
# <span id="download">下载</span>