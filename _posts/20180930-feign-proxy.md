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

直到最近被一个同事打破砂锅问到底，我反倒一脸懵逼。虽然上面那套说辞大体上能自圆其说，但是真相还是需要亲身去探索。

## 程序入口
我们都知道，如果想使用 FeignClient，需要在启动主类上添加 *@EnableFeignClients* 注解。那么 Feign 的一套体系生效应该就在于这个注解上了。我们打开 *EnableFeignClients* 看下源码：
```java
@Import(FeignClientsRegistrar.class)
public @interface EnableFeignClients {
    。。。。
}
```
```java
class FeignClientsRegistrar implements ImportBeanDefinitionRegistrar,ResourceLoaderAware,EnvironmentAware {
    @Override
	public void registerBeanDefinitions(AnnotationMetadata metadata,BeanDefinitionRegistry registry) {
		。。。。
		registerFeignClients(metadata, registry);
	}
    public void registerFeignClients(AnnotationMetadata metadata,BeanDefinitionRegistry registry) {
        。。。。
        for (BeanDefinition candidateComponent : candidateComponents) {
            // 注意这里，开始寻找添加 @FeignClient 注解的接口，并在后续进行注册
            Map<String, Object> attributes = annotationMetadata.getAnnotationAttributes(FeignClient.class.getCanonicalName());

            String name = getClientName(attributes);
            registerClientConfiguration(registry, name,attributes.get("configuration"));

            registerFeignClient(registry, annotationMetadata, attributes);
        }
        。。。。
     }
}
```
上文代码印证了我们之前的猜测：Feign 组件通过 *EnableFeignClients* 作为入口生效。具体执行过程如下：  
1. 在用户添加了该注解后，该注解引入了 *FeignClientsRegistrar* 配置
2. *FeignClientsRegistrar* 实现了 *ImportBeanDefinitionRegistrar* 接口，然后在 *registerBeanDefinitions* 方法中将加上 *@FeignClient* 注解的接口均在 spring 中做了注册（这句表述不太准确，用户完全可以自定义扫描包之类的，简单起见不做扩展）

##  生成代理
被 *@FeignClient* 注解的接口被抛给了 spring 去注册，但是注册的代理对象是在 *FeignClientFactoryBean* 返回的:  
```java
class FeignClientFactoryBean implements FactoryBean<Object>, InitializingBean,ApplicationContextAware {
    @Override
	public Object getObject() throws Exception {
		FeignContext context = applicationContext.getBean(FeignContext.class);
		Feign.Builder builder = feign(context);
        // 如果没配置 url 属性，则需要走服务发现逻辑
		if (!StringUtils.hasText(this.url)) {
			String url = "http://" + this.name;
			return loadBalance(builder, context, new HardCodedTarget<>(this.type,this.name, url));
		}
        // 注意，这里的 Client 非常重要，它是 feign 网络请求组件接口，默认是 Default
		Client client = getOptional(context, Client.class);
		if (client != null) {
			if (client instanceof LoadBalancerFeignClient) {
				// not load balancing because we have a url,
				// but ribbon is on the classpath, so unwrap
				client = ((LoadBalancerFeignClient)client).getDelegate();
			}
			builder.client(client);
		}

		Targeter targeter = get(context, Targeter.class);
		return targeter.target(this, builder, context, new HardCodedTarget<>(this.type, this.name, url));
	}
}
```
由上文代码我们可以看出，在生成代理的时候分了两种情况：如果 *@FeignClient* 注解配置了 *url* 属性，则直接生成代理；如果没有配置，证明需要走服务发现逻辑，则交给 Ribbon 去拼接 url 并生成代理。  
我们这里以简单的 url 配置方式为例，看看最终的代理对象是如何生成的：
```java
public class ReflectiveFeign extends Feign {

  @Override
  public <T> T newInstance(Target<T> target) {
    Map<Method, MethodHandler> methodToHandler = new LinkedHashMap<Method, MethodHandler>();

    for (Method method : target.type().getMethods()) {
        // 注意，这里的逻辑比源码删减很多
        methodToHandler.put(method, nameToHandler.get(Feign.configKey(target.type(), method)));
    }
    InvocationHandler handler = factory.create(target, methodToHandler);
    // 使用 jdk 代理生成对象，注意这里的 target 并非被 @FeignClient 注解的接口，而是 HardCodedTarget，但是它里面存储了被注解接口的信息
    T proxy = (T) Proxy.newProxyInstance(target.type().getClassLoader(), new Class<?>[]{target.type()}, handler);

    return proxy;
  }
```
其实，源码中跳转了几次：  
1. FeignClientFactoryBean 使用 Targeter.target() 去生成被代理对象
2. Targeter 使用 Feign.target() 生成被代理对象
3. Feign 中的 target() 调用抽象方法 newInstance() 来生成对象
4. ReflectiveFeign 继承自 Feign， 并实现了 newInstance() 方法
5. ReflectiveFeign.newInstance() 中使用 jdk 代理，将被  *@FeignClient* 注解的接口的所有方法通过 jdk 代理来实现。

## 总结
经过上面的代码分析，我们大致能将整个流程串了起来：  
1. 在用户加上 *EnableFeignClients* 注解后，整个 Feign 组件被启用
2. *FeignClientsRegistrar* 在启动时，将被加了 *FeignClient* 注解的(而且符合配置条件的)接口注册成 bean
3. *FeignClientFactoryBean* 梳理 url 信息，并选择 HTTP 组件，将信息一并交给 *ReflectiveFeign* 生成代理类
4. *ReflectiveFeign* 收到请求后，以 HardCodedTarget 为蓝本，使用 jdk 动态代理，生成代理对象



