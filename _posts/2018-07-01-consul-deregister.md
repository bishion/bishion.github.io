---
layout: post
title: sc-consul 停服反注册功能
categories: 微服务
description: consul 健康检查有时间窗口, 窗口期停服时怎么办
keywords: consul, 微服务
---
# 疑问
Consul 作为服务治理中心, 有健康检查功能. 服务注册时,可以指定健康检查窗口时间.  
sc-consul 中, 默认的健康检查频率是每隔 10s 一次.
那么问题来了, 如果服务在窗口期挂了, 是不是在 Consul 下一次健康检查之前, 客户端都无法感知呢？

# 实验
1. 启动 Consul 服务
2. 启动服务提供方, 设置健康检查窗口期为十分钟：  
spring.cloud.consul.discovery.health-check-interval: 10m
3. 停掉服务方, 查看 Consul 页面中的服务列表

# 实验结果
1. 如果是在 IDE 中直接点击“停止服务”, 服务会直接从 Consul 的 service 列表中移除
2. 如果是通过 “ctrl + C” 的方式停服, 服务会直接从 Consul 的 service 列表中移除
3. 如果是通过 “kill pid” 的方式停服, 服务会直接从 Consul 的 service 列表中移除
4. 如果是通过 “kill -9 pid” 的方式停服，服务**过一会儿**显示 Critical

# 分析
1. 服务从 Consul 的 service 列表中移除, 表明服务是被主动下线
2. 服务显示 “Critical”, 表明服务健康检查不通过; **过一会儿**, 表明 Consul 不能及时发现服务方出问题

# 思考
实验结果3 验证了我们一开始的顾虑, 倒不足为虑, 实验结果1,2 是怎么回事呢

## 观察
通过 Consul 页面和 Consul 的日志, 我们能看到服务被停掉时, Consul 是收到反注册请求的：
```
2018/07/01 17:56:24 [DEBUG] http: Request GET /v1/catalog/services?wait=2s&index=253 (2.12136395s) from=127.0.0.1:35524
2018/07/01 17:56:25 [DEBUG] agent: removed check "service:consul-provider-8081"
2018/07/01 17:56:25 [INFO] agent: Deregistered service "consul-provider-8081"
2018/07/01 17:56:25 [INFO] agent: Deregistered check "service:consul-provider-8081"
```
于是, 我们能知道, 很有可能是服务方主动发起的反注册请求.   
我们知道, 一般在点击 IDE 的 “停止” 按钮时, 系统还是会打一段日志, 会不会是那个时候触发的反注册呢？
经过仔细检索日志, 我们发现：
```
2018-07-01 18:21:32.540  INFO o.s.s.c.ThreadPoolTaskScheduler : Shutting down ExecutorService 'configWatchTaskScheduler'
2018-07-01 18:21:32.540  INFO o.s.c.c.s.ConsulServiceRegistry : Deregistering service with consul: consul-provider-8081
2018-07-01 18:21:32.543  INFO o.s.s.c.ThreadPoolTaskScheduler : Shutting down ExecutorService 'catalogWatchTaskScheduler'
2018-07-01 18:21:32.545  INFO o.s.s.c.ThreadPoolTaskScheduler : Shutting down ExecutorService 'taskScheduler'
```
sc-consul 中, 反注册的代码我们能找到:ConsulServiceRegistry.deregister(), 日志中也确实显示了服务是被主动下线.
```java
public class ConsulServiceRegistry implements ServiceRegistry<ConsulRegistration> {
	...........
	
	@Override
	public void deregister(ConsulRegistration reg) {
		if (ttlScheduler != null) {
			ttlScheduler.remove(reg.getInstanceId());
		}
		if (log.isInfoEnabled()) { // 这里就是上文日志打印的地方
			log.info("Deregistering service with consul: " + reg.getInstanceId());
		}
		client.agentServiceDeregister(reg.getInstanceId(), properties.getAclToken());
	}
    ...........
}

```
经过代码跟踪, 我们发现这个方法是被AbstractAutoServiceRegistration.destroy() 调用的:
```java
public abstract class AbstractAutoServiceRegistration<R extends Registration>
		implements AutoServiceRegistration, ApplicationContextAware {
	............
	
	@PreDestroy   // 该注解表示, 在退出服务(exit)时执行
	public void destroy() {
		stop();
	}
	............
}
```
# 结论
1. sc-cloud 通过**在程序退出时执行反注册方法**的方式保证下线的服务不会被发现  
2. Spring Boot 退出时执行的方式有很多, 有兴趣的同学可以自行查阅一下
3. 如果服务被强制 kill, 反注册方法不会执行, Consul 窗口期的问题便会显现
4. 对于使用 sc-cloud 自注册的情况, 停服命令建议使用 “kill pid” 的方式