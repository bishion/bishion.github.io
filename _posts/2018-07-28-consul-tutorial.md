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
服务总会有一些运行时修改的配置，比如状态标识或者开关等，需要实时生效。使用配置系统分发这些配置或者重启服务一般都需要几分钟甚至数小时。在推送过程中，框架一致性很难保证，服务配置也可能出错。  
![image](/images/consul/use-case-config-challenge.svg)
### 解决方案
分布式应用的实时配置中心  
Consul 支持上千节点的配置数据实时更新。配置存储在树状的 Key/Value 结构中，高效的边界触发器可以将更新快速地推送出去。  
![image](/images/consul/use-case-config-solution.svg)
### 特点
#### Key/Value 存储结构
开关，状态等动态服务配置信息采用富 Key/Value 存储结构。  
![image](/images/consul/use-case-config-ui-kv.webp)
#### 事务支持
Key/Value 支持读或者写事务，它允许多个 key 原子地读写。服务配置的修改可以原子操作，从而避免一致性问题。
[了解更多](#api-kv)  
```bash
$ curl http://localhost:8500/v1/txn \
  --request PUT \
  --data \
  '[
    {
      "KV": {
        "Verb": "set",
        "Key": "lock",
        "Value": "MQ=="
      }
    },
    {
      "KV": {
        "Verb": "cas",
        "Index": 10,
        "Key": "configuration",
        "Value": "c29tZS1jb25maWc="
      }
    }
  ]'
```
#### 阻塞查询和边界触发请求
Consul API 支持阻塞查询，允许边界触发更新。客户端通过这种方式可以在数据更新时收到通知。Consul-Template 或者类似工具允许配置文件在被修改时重新初始化。  
[了解更多](#api-kv)  
```bash
$ curl http://localhost:8500/v1/kv/web/config/rate_limit?wait=1m&index=229
[
    {
        "LockIndex": 0,
        "Key": "web/config/rate_limit",
        "Flags": 0,
        "Value": "NjAw",
        "CreateIndex": 229,
        "ModifyIndex": 234
    }
]
```
#### 监控
监控组件使用阻塞查询来管理配置信息和健康状态的更新，在更新发生时去执行用户事先定义好的特定脚本。这给建造一个响应式架构带来了很大便利。  
[了解更多](#guides-agent-watches)  
```bash
$ consul watch \
      -type=key \
      -key=web/config/rate_limit \
      /usr/local/bin/record-rate-limit.sh
```
#### 分布式锁和信号量
Key/Value 存储支持分布式锁和信号量。应用可以使用这个功能做leader选举或者控制共享资源的访问。
[了解更多](#guides-semaphore)  
![image](/images/consul/use-case-lock-semaphore.svg)
# 介绍
本文是开始 Consul 旅程的最佳起点。我们将为大家介绍 Consul 是什么，它能解决什么问题，它跟现有类似框架有什么区别，以及如何使用它。如果你已经熟悉 Consul 的基本功能，[文档](#documentation)有对Consul功能点更具体的介绍。
## 什么是 Consul
Consul 是 service mesh(服务网格)的一个解决方案，它提供了诸如服务发现，配置和隔离等功能的一整套控制平面(control plane)。开发人员可以根据需要单独使用这些功能点，也可以将他们整合成为一个完整的service mesh。Consul 需要一个数据平面(data plane)，并支持代理和本地集成模型。Consul 自带了一个简单的代理以实现开箱即用的功能，不过它也支持集成第三方代理框架，比如Envoy。  
Consul 的核心功能如下：  
- **服务发现：** Consul 客户端可以注册一个服务，比如 api 接口或者 mysql 服务，其他的客户端可以通过 Consul 来发现这些服务的提供方。通过 DNS 或者 HTTP，引用可以很方便地找到它所依赖的服务。  
- **健康检查：** Consul 客户端可以提供任意数量的健康检查，无论是特定服务(服务是否返回200状态)还是本地节点(比如内存使用率是否大于90%)。运维人员可以通过这些信息管理集群的健康情况，服务发现组件也可以使用这些信息过滤掉不健康节点。  
- **KV 存储：** 应用可以使用 Consul 的树状 key/value 存储做很多事情，比如动态配置，开关，协作，leader 选举等等。KV 存储使用的是 HTTP 接口，非常简单易用。
- **服务通信加密：** Consul 可以生成并分发 TLS 证书给服务，从而保证服务之间使用 TLS 通信。我们可以使用 [intentions 组件](#docs-connect-intention)来自定义允许哪些服务访问。使用 intentions 可以实时地操作服务隔离，这比使用复杂的网络拓扑和静态防火墙规则要简单很多。  
- **多数据中心** Consul 支持开箱即用的多数据中心功能。这意味着当工作区域扩展至多个的时候，用户不需要费心去创建额外的抽象层来满足多中心需求。  
Consul 设计时就考虑到了对 DevOps 社区和应用开发者友好，这让它非常适合新兴的弹性架构。  
## Consul 的基础架构
Consul 是一个分布式高可用的框架。本节我们省略那些不必要的细节，只讲述基本的知识点，让读者快速了解 Consul 是如何运转的。如果你想看更多的细节，可以参考[Consul 内部架构概览](#)。  

每一个 Consul 节点都要运行一个 Consul agent(Consul 代理)。运行一个 agent 不需要做服务发现或者设置 key/value 数据。这个代理只是负责对当前节点自己还有它上面的服务做健康检查。  

agent 与一个或多个 Consul servers(Consul 服务)通信。Consul servers 是数据存储和备份的地方。servers 内部选举一个 leader。虽然 Consul 可以只有一个 server，但是为了避免单节点故障造成数据丢失，我们建议最好部署 3-5 个 server 节点。每个数据中心都需要一个 Consul server 集群。  

你的服务发现组件可以通过 Consul server或者任意一个 Consul agent 来查询服务或者节点。agent 可以将这些查询自动转发给 server。  

每个数据中心都要运行一个 Consul server 集群。当一个跨数据中心的服务发现或者配置请求发出时，本地的 Consul server 会将这些请求转发给远程的数据中心，然后将结果返回。
## Consul 与其他框架对比 
Consul 可以解决很多问题，但是每个问题市面上都有很多解决方案。虽然没有一个系统可以囊括 Consul 所有的特性，但是遇到其中一些问题的时候，你还是有许多别的方案可以选择。  
在本章节中，我们将把 Consul 和有类似功能的系统一起做比较。在大多数的场景中，Consul 与那些系统并不冲突。 
### Consul 与 zookeeper，doozerd，etcd
Zookeeper，doozerd 和 etcd 在架构上很相似。它们三个都对服务节点数量有要求。他们都具有强一致性的特点，我们可以通过客户端应用集成的工具库调用它们对外开放的 API 来构建复杂的分布式系统。  
在一个单独的数据中心中，Consul 也有多个服务节点(server nodes)。在每个数据中心，Consul server 需要一个法定人数来运作和提供强一致性。但是 Consul 本身支持多数据中心，并且提供一个功能丰富的 gossip 系统来连接服务端和客户端节点。 
所有这些系统在提供 key/value 存储上面实现方式大致相同：强一致性的读和为了应对网络分区而牺牲的可用性。但是，在一些复杂场景中，他们的区别非常明显。  
当构建服务发现系统时，这些系统提供的特性很有诱惑力，但是需要强调的是，这些特性必须手动构建。Zookeeper 它们仅提供一个原始的K/V存储功能，它需要应用开发者以此手动构建服务发现功能。而Consul自带服务发现框架，大大减轻了客户端和开发人员的工作。服务端只需要简单地注册自己的服务，客户端就可以通过 DNS 或者 HTTP 接口来做服务发现。其他几个系统都需要自己实现一套解决方案。  
好的服务发现框架必须包含健康检查和失败方案。如果节点 A 挂了或者它上面的服务崩溃了，哪怕知道节点 A 提供了 Foo 服务也没啥用。有些系统简单地使用定期更新和TTL(time to live)做心跳检测。这些方案需要在固定的服务器上对日益增长的节点发出健康检测请求。而且，检测失败的窗口期跟 TTL 的时间一样长。  
Zookeeper 使用键值对来充当临时节点，当客户端断开连接时，对应的键值对就会被删除。这种方式要比心跳检测的方式要高级很多，但是它还是有固有的扩展性问题，而且增加了客户端复杂性。所有的客户端必须保持与 Zookeeper 的连接。另外，它增加了客户端的复杂度并给调试带来挑战。  
Consul 使用完全不同的架构来实现健康检查。相对于那些只有服务端节点的方案，Consul 在集群的每一个节点还有一个客户端。客户端是[gossip 池](#docs-internals-gossip)的一部分，提供分布式健康检查等功能。gossip 协议实现了一个高效的故障侦测机制，它支持集群扩展到任意大小而不会将工作量集中于特定的几个服务节点。客户端还支持本地执行更丰富的健康检查，而 Zookeeper 只能提供很基础的连接检查。有了 Consul，客户端可以检查一个 web 服务是否是 200 状态，内存使用率是否正常，磁盘空间是否够用等等。Consul 客户端对外暴露的是简单的 HTTP 接口，而不是像 Zookeeper 那样把系统的复杂性直接暴露给客户端。  
Consul 的主要功能是服务发现，健康检查，K/V 存储和多数据中心。除非你需要的是简单的 K/V 存储，否则其他系统都需要额外的工具和库来构建。但是有了客户端， Consul 提供的都是很简单的 API 接口。而且，如果你使用了配置文件或者 DNS 接口来做为服务发现的解决方案，你甚至都不要这些 API ，当然也不需要任何开发工作了。  
### Consul 与 Chef, Puppet等
人们使用 Chef，Puppet或者其他配置管理工具来构建服务发现系统已经不是什么新鲜事了。在定期执行的汇聚过程中，通过查询全局状态来给每个节点创建配置文件，以达到服务发现的目的。  
不幸的是，这种方式有很多的弊端。配置信息是静态的，而且更新频率取决于汇聚的频率，通常都是几分钟甚至几小时一次。而且，并没有一个机制能在配置中管理系统的状态：流量进入不健康的节点会让问题更加严重。这种方式让多数据中心面临更大的挑战，因为必须有一组服务集群去维护所有的数据中心。  
Consul 在设计上就着重考虑了服务发现功能。因此，它能动态而且快速反馈集群的状态。节点可以注册和反注册它提供的服务，让调用方应用和服务能更快地查到所有的提供方。通过内置的健康检查功能，Consul 可以将流量绕过不健康的节点，从而系统和服务可以更优雅地恢复。Consul 使用动态的 key/value 存储来代替静态配置，这样应用配置信息可以不经过慢吞吞的汇聚就能被更新掉。最后要说的是，因为每个数据中心是单独运行的，所以用户可以像管理单个数据中心那样管理多个数据中心。  
也就是说，Consul 并不是配置管理工具的替代品。这些工具包括 Consul 对于设置应用程序仍然至关重要。静态配置最好由现有的工具管理，而动态的状态信息和注册信息最好则可以交给 Consul。配置管理和集群管理的分离也有其他好处：Chef 和 Puppet 不需要维护全局状态时就变得很简单了，不需要运行一个定时器来应用随着服务和配置的修改，并且由于配置管理工具不依赖全局状态运行，整个架构就更加稳定了。  
### Consul 与 Nagios，Sensu
Nagios 和 Sensu 都是监控工具，在系统出现问题时，他们可以快速通知到运维人员。  
Nagios 在服务集群中执行对远程主机的健康检查。这种方式扩展性不高，因为大型集群很容易达到垂直扩展的瓶颈，而 Nagios 也很难水平扩展。在新型的 devOps 架构和配置管理工具方面，Nagios 也是出名的难用，因为服务集群增减机器的时候，本地配置也必须更新。  
Sensu 有一些更先进的设计理念，它依赖本地 agent 执行健康检查，然后将结果推送给消息队列。服务集群从消息队列中接收并处理健康检查信息。这种模式比 Nagio 更加有弹性，它能很好地支持水平扩缩容，弱化了 server 和 agent之间的关联性。但是，中央代理(central broker)有扩缩容瓶颈而且是系统的单点故障。  
Consul 提供了跟 Nagios 和 Sensu 相同的健康检查功能，但是它对流行的 DevOps 更加友好，而且不会有后两者的扩缩容问题。Consul 只在本地执行健康检查，就像 Sensu 那样，从而避免了将压力集中在那几台服务节点上。健康检查的结果是由 Consul 服务端维护的，服务端考虑到了容错性，并不会有单点故障。最后，Cosnul 使用了边界触发更新模式，从而可以扩展到很大的检查规模。边界触发更新(edge-triggered update)的意思是，只有当检查结果是从“passing”变成“failing”或者“failing”变成“passing”时，agent 才会向 server 同步更新。  
在大规模集群下，绝大多数的检查结果都是“passing”，而且剩下的“failing”也大都是持久性的。Consul 只处理变更的状态，从而减少了健康检测所消耗的网络和计算资源，继而让系统由更好的扩展性。  
聪明的读者可能会注意到，如果 Consul agent 挂掉，那么就不会再有状态变更更新，对于其他节点来说，所有的健康检查都是稳定的。当然，Cosnul 也考虑到这点，Consul 在客户端和服务端都内置了 [gossip 协议](#docs-internals-gossip)，以作为分布式故障探测器。一旦一个 Consul gent 挂掉，信息很快就会被侦测到。这种故障侦测器将工作分配到整个集群，而且保证了边界触发框架正常运行。 
### Consul 与 SkyDNS
SkyDNS 是一个服务发现工具，它有一个强一致性的服务集群来保证容错性。节点通过 HTTP 接口注册服务，然后使用 HTTP 或者 DNS 来做服务发现。  
Consul 跟它十分类似，但是提供了更多的特性。Consul 也是依赖一组服务集群提供强一致性服务并保证容错性。节点可以使用 HTTP 接口或者 agent 注册服务，并提供 HTTP 或者 DNS 的方式提供查询功能。  
但是，两个系统还是有很多不同的地方。Consul 提供了更丰富的健康检查特性，不仅支持简单的检查方式，还支持高扩展的失败检测。SkyDNS 依赖于简单的心跳检测和 TTL，这种方式有很多我们熟知的扩展性问题。而且，相对于 Consul 那么丰富的健康检查方式，心跳检测提供了有限的存活校验。  
通过“regions”，SkyDNS 也支持多数据中心，但是数据是被一个单独的集群管理并提供查询服务的。如果两个 SkyDNS 服务器是跨数据中心的，数据复制协议就不得不忍受漫长的提交时间，如果它们在同一个数据中心，网络故障就会导致整个数据中心失去可用性。而且，就算没有网络问题，所有的请求必须在另外一个数据中心执行，我们也不得不忍受它带来的查询性能问题。  
Consul 支持开箱即用的多数据中心功能，每个数据中心只维护自己的数据。这意味着，每个数据中心管理着独立的服务集群。如果要跨数据中心查询，则请求会被转发给远程数据中心，而同一个数据中心的请求则在局域网内执行，所以数据中心之间的网络问题不会影响到数据中心内部的服务，而且一个数据中心不可用了，也不会对别的数据中心造成影响。  
### Consul 与 SmartStack
SmartStack 是一个服务发现工具。它有一个由四个组件组成的独特架构：Zookeeper，HAProxy，Synapse 和 Nerve。Zookeeper 服务负责存储集群状态并保证其一致性和容错性。SmartStack 集群的每一个节点都会运行 Nerve 和 Synapse。Nerve 负责将服务注册给 ZooKeeper 并执行单个服务的健康检查。Synapse 负责向 ZooKeeper 查询服务提供方并且动态配置 HAProxy。最终，用户调用 HAProxy，HAProxy 提供服务方之间的健康检查和负载均衡。  
Consul 比 SmartStack 要简单很多，而且不依赖扩展组件。Consul 使用内置的 [gossip 协议](#docs-internals-gossip)去做节点监控和服务发现。不像SmartStack，使用 Consul 后，我们并不需要硬编码服务提供方的地址，也不需要总是更新。  
Consul 和 Nerves 的服务注册都可以通过一个配置文件来完成，但是 Consul 还支持在运行中通过 API 动态修改服务和检查结果。  
为了实现服务发现，SmartStack 客户端必须使用 HAProxy，而且要求 Synapse 提前配置好所有需要的端点。Consul 客户端提供的是 DNS 或者 HTTP 接口，不需要事先做任何配置。Consul 还提供“tag”抽象，允许服务提供元数据比如版本号，优先级，或不透明标签来做查询过滤。客户端可以选择只请求符合对应“tag”的服务提供方。  
这两个系统还有一个区别就是他们管理健康检查的方式。Nerve 跟 Consul agent 类似，是执行本地的健康检查。但是，Consul 维护着单独的目录和健康系统。
## <span id="started">正式入门</span>
### <span id="started-service">服务</span>
#### <span id="started-service-query">服务查询</span>
#### <span id="started-service-http">HTTP 接口</span>
### <span id="started-health-check">健康检查</span>
# 指南
## <span id="guides-datacenter">多数据中心</span>
## <span id="guides-semaphore">信号量</span>
# <span id="documentation">文档</span>
## <span id="docs-internals">原理</span>
### <span id="docs-internals-architecture">架构</span>
### <span id="docs-internals-gossip">gossip</span>
## 代理
### <span id="docs-agent-watches">监控</span>
## <span id="docs-connect">连接</span>
### <span id="docs-connect-intention">Intentions</span>
### <span id="docs-connect-proxies">代理</span>
### <span id="docs-connect-ca">认证管理</span>
### <span id="docs-connect-security">安全</span>
# API
### <span id="api-kv">KV 存储</span>
# 社区
# <span id="download">下载</span>