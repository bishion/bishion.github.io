---
layout: post
title: spring consul 健康检查与 consul 版本匹配问题
categories: spring cloud, consul
description: spring cloud consul 健康检查与 consul 版本匹配问题
keywords: 版本匹配
---
# 问题描述
consul 升级后，原有的项目健康检查 /health 通不过，报 _**Data must not be null**_ 的错误

# 问题排查
## 健康检查源码：ConsulHealthIndicator.class
```java
protected void doHealthCheck(Health.Builder builder) throws Exception {
    try {
        Response<Self> self = consul.getAgentSelf();  // 请求接口：domain:port/v1/agent/self
        Config config = self.getValue().getConfig();
        Response<Map<String, List<String>>> services = consul.getCatalogServices(QueryParams.DEFAULT);
        builder.up()
                .withDetail("services", services.getValue())
                .withDetail("advertiseAddress", config.getAdvertiseAddress()) // 该配置为空，因而报了异常
                ******
    }
    catch (Exception e) {
        builder.down(e);
    }
}
```
## 分析
经源码得知，consul 的健康检测是去拿 agent 的基础配置，继而根据基础配置的一些字段来判断 consul 是否正常运行。
然而，consul 的1.0.0 版本以后，/agent/self 接口返回的信息中便没有了 advertiseAddress 信息，于是健康检查就报了异常。
## 解决方案
### 方案一：升级 spring-cloud-consul 到 1.3.0.RELEASE 版本
```java
protected void doHealthCheck(Health.Builder builder) throws Exception {
    try {  
        Response<String> leaderStatus = consul.getStatusLeader(); // 请求接口：domain:port/v1/status/leader
        Response<Map<String, List<String>>> services = consul.getCatalogServices(QueryParams.DEFAULT);
        builder.up()
                .withDetail("leader", leaderStatus.getValue()) // 只查看 leader 信息，不再拿别的参数了
                .withDetail("services", services.getValue());
    }
    catch (Exception e) {
        builder.down(e);
    }
}
```
#### 优点
1. spring 换了个相对稳定的接口做健康检查，以应对将来可能的接口变化
2. 之后的一段时间，consul 的升级应该不会影响到健康检查
#### 缺点
1. spring-cloud-consul 升级到 1.3.0.RELEASE，意味着 spring-cloud 要升级到 Edgware.SR1，对于很多老项目来说，代价太大
2. 没有了，第一条已经致命了
### 方案二：关闭 consul 的健康检查配置
```properties
management.health.consul.enabled=false
```
#### 优点
1. 快速修改
2. 通过 config-server 可以实现热加载，不需上线
#### 缺点
1. 健康检查排除掉 consul，有一定的隐患
2. 该配置并不支持 spring-cloud-consul 的 1.1.2.RELEASE 以前的版本
#### 分析
因为 consul 作为注册中心，挂掉的几率还是很小的，将“关闭对 consul 的健康检查”作为临时解决方案是可行的  
然而 1.1.2.RELEASE 版本以前的 spring-cloud-consul，是强制对 consul 做健康检查的，并没有开放配置字段  
因此，如果你使用的是旧的版本，该方案不可行  
进一步查看 ConsulHealthIndicator 怎么生效的：
```java
@Bean
@ConditionalOnMissingBean  // 注意该注解，只要你已经有一个 bean，当前 bean 便不再加载
@ConditionalOnEnabledHealthIndicator("consul") // 这一行是 1.3.0.RELEASE 版本以后才加的
public ConsulHealthIndicator consulHealthIndicator(ConsulClient consulClient) {
    return new ConsulHealthIndicator(consulClient);
}
```

### 方案三：重写 consulHealthIndicator
```java
@Service("consulHealthIndicator")
public class MyConsulHealthIndicator extends AbstractHealthIndicator {
	@Override
	protected void doHealthCheck(Health.Builder builder) throws Exception {
		// 自由发挥，可以直接写 builder.up();也可以按照最新的健康检查逻辑拷贝进来
	}
}
```
#### 分析
好在旧版本虽然无法通过配置的方式关掉 consul 的健康检查，但是新旧版本都支持重写 bean _consulHealthIndicator_ 
#### 优点
1. 新旧 spring 版本均支持
2. 自行实现，按需定制
#### 缺点
1. 要上线
2. 维护业务系统无关的代码
# 问题总结
1. 基本上没有很优雅的方式解决这个问题，只能老老实实改代码升级
2. 出现这个问题的根本原因在于 consul 改变了接口实现，导致客户端不兼容，差评
3. spring 在健康检查中的逻辑也并不严谨，依赖了太多信息

# 关于版本信息
1. consul 在 1.0.0 版本及以后，修改了 /v1/agent/self 接口的返回数据
2. spring-cloud-consul 在 1.1.2.RELEASE 版本及以后才支持通过配置的方式开启 consul 的健康检查
3. spring-cloud-consul 在 1.3.0.RELEASE 版本及以后才支持新 consul 的健康检查
4. spring-cloud-parent 在 Edgware.SR1 版本及以后才支持新 consul 的健康检查