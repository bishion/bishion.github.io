---
layout: post
title: 重写 consul 中服务发现逻辑
categories: 微服务
description: consul 中 serverList 功能重写
keywords: consul, 微服务, ribbon
---
# 需求
覆盖 spring-cloud-consul(以下简称 consul，注意它不是日常说的 HashiCorp consul ) 的服务发现逻辑，以满足特殊场景需求  
**不允许**直接改源码

# 分析
consul 中有两处用到了服务发现逻辑
1. ConsulDiscoveryClient: 这个类用的场景比较少，一般是别的组件或者系统需要自己获取服务注册的一些信息时，手动注入该类，然后直接使用
2. ConsulServerList: 我们日常通过 feign 或者 ribbon 间接使用服务发现时，调用的就是这个类的 getServers() 方法  
我们今天重点看一下ConsulServerList  
不出意外，ConsulServerList 是被注入到 Spring 中使用的，而且注入条件是 ConditionalOnMissingBean：
```java
@Configuration
public class ConsulRibbonClientConfiguration {
    @Bean
	@ConditionalOnMissingBean
	public ServerList<?> ribbonServerList(IClientConfig config, ConsulDiscoveryProperties properties) {
		ConsulServerList serverList = new ConsulServerList(client, properties);
		serverList.initWithNiwsConfig(config);
		return serverList;
	}
}
```

# 解决办法一
自己实现 ServerList，并且让它在原生 ConsulServerList 初始化前被注入
```java
@Configuration
public class MyConsulRibbonClientConfiguration {
    @Bean
    public ServerList<?> ribbonServerList(IClientConfig config, ConsulDiscoveryProperties properties) {
        MyConsulServerList serverList = new MyConsulServerList(client, properties);
        serverList.initWithNiwsConfig(config);
        return serverList;
    }
}
public class MyConsulServerList extends AbstractServerList<ConsulServer> {
       ..... 自定义逻辑
}
```

## 运行结果
虽然我们已经定义好了 MyConsulServerList, 但是程序运行时，还是去初始化 ConsulServerList  
而且我们发现原生 ConsulServerList 对于serviceId 的处理很奇怪
```java
public class ConsulServerList extends AbstractServerList<ConsulServer> {
    ....
	private String serviceId; // 服务名，下同，不再备注
	@Override
    public void initWithNiwsConfig(IClientConfig clientConfig) {
        this.serviceId = clientConfig.getClientName();
    }
	....
}
```
对的，你没有看错，serviceId 是放在成员变量里的  
这就有一个问题，绝大多数系统都是不止一个服务提供方，那 serverId 放在成员变量里，就证明 ConsulServerList 不可能是单例  
可是也没有看到明显的 scope 配置，那这个 Bean 是什么时候被初始化的呢

## 多例的 ConsulServerList
经过断点可知，每当请求一个新的 service 时，ConsulRibbonClientConfiguration.ribbonServerList() 就会被执行一次  
而触发该方法的并不是 consul, 而是 ribbon
```java
LoadBalancerFeignClient.execute() -> IClientConfig requestConfig = getClientConfig(options, clientName);
LoadBalancerFeignClient.getClientConfig() -> requestConfig = this.clientFactory.getClientConfig(clientName);
SpringClientFactory.getClientConfig() -> getInstance(name, IClientConfig.class);
SpringClientFactory.getInstance() -> super.getContext(name);
NamedContextFactory.getContext() -> createContext(name);

NamedContextFactory.createContext():
protected AnnotationConfigApplicationContext createContext(String name) { // name 亦即服务名
    AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
    ......
    // 重新注册 RibbonClientConfiguration，这里 defaultConfigType 就是 RibbonClientConfiguration
    context.register(PropertyPlaceholderAutoConfiguration.class,this.defaultConfigType); 
    .....
    context.refresh(); // 刷新 RibbonClientConfiguration, 其下的 serverList 也被重新初始化
    return context;
}
```
核心调用链和代码如上文所示。大致步骤如下：
1. feign 拿着 serviceId 去问 ribbon 要服务端配置
2. ribbon 根据 serviceId 去 RibbonClientConfiguration 的上下文中找是否存在这个名称的 ConsulServerList bean
3. 如果存在就返回
4. 如果不存在，就注册一个，然后刷新上下文初始化

## spring 怎么知道要刷新哪个上下文
它是通过 @RibbonClient 注解来定位的  
这里就不再展开，具体可以参考代码：RibbonClientConfigurationRegistrar.class

# 解决办法二
1. 禁用原有的 ConsulRibbonClientConfiguration: spring.cloud.consul.ribbon.enabled:true
2. 自定义自己的 ConsulRibbonClientConfiguration: MyConsulRibbonClientConfiguration
3. 自定义自己的 ConsulServerList: MyConsulServerList
4. 给自己的逻辑添加入口: @ConditionalOnExpression("${spring.cloud.consul.ribbon.enabled:true}==false")
5. 具体实现代码参考: https://github.com/bishion/microService/tree/master/myConsulTool

# 思考
为什么 ribbon 要这么大费周章地去给每个服务方定义一个单独的 ServerList 呢? 明明在调用的时候，将 serviceId 传给服务发现逻辑就可以了  
这是因为 ribbon 的一个需求决定的: 服务配置个性化  
ribbon 支持对某一个服务单独配置负载，比如负载算法，是否重试等，当然也包括服务发现逻辑  
为每一个服务实例化一个服务发现逻辑，可以最大化地将自由交给实现方。只是目前的 consul 并未用到这个功能而已

# 后记
1. 为每个服务**懒加载**一个独有的实例，ribbon 的实现很巧妙，值得学习
2. 关于 ribbon 对于服务个性化配置的实现，找机会另开一题
3. 如有错误或不当之处，欢迎留言指正