---
layout: post
title: spring cloud feign 的 https bug
categories: 微服务
description: spring cloud feign 的 https bug
keywords: 微服务，feign
---
# 问题描述
服务方一切正常，客户端在调用的时候却报错：

```
com.netflix.hystrix.exception.HystrixRuntimeException: queryAeraAddress failed and no fallback available.
Caused by: feign.RetryableException: Unrecognized SSL message, plaintext connection? executing GET http://app-name/api/a/v1/common/areas?parentCode=123
Caused by: javax.net.ssl.SSLException: Unrecognized SSL message, plaintext connection?
com.netflix.hystrix.exception.HystrixRuntimeException: queryAeraAddress failed and no fallback available.
Caused by: feign.RetryableException: Read timed out executing GET http://app-name/api/a/v1/common/areas?parentCode=123
Caused by: java.net.SocketTimeoutException: Read timed out
Servlet.service() for servlet [dispatcherServlet] in context with path [] threw exception [Request processing failed; nested exception is com.netflix.hystrix.exception.HystrixRuntimeException: queryAeraAddress failed and no fallback available.] with root cause
java.net.SocketTimeoutException: Read timed out
Servlet.service() for servlet dispatcherServlet threw exception
java.lang.IllegalStateException: getOutputStream() has already been called for this response
.....
````
# 问题分析
日志显示，问题应该跟https有关系，导致请求服务超时。

因日志打印出来的是 http://app-name/***, 所以是不是服务方提供的是https服务呢。

经检查，服务方是一个简单的微服务应用，没有做 https 配置，并且用 https 访问反而报错

由此可得：应该是客户端访问服务方时用了 https

# 问题跟踪
客户端是使用简单的 feignClient 访问：@FeignClient(
    name = "app-name"
)，并无异常

程序也没有对程序做过 https 配置，相同配置的其他服务都一切正常，那问题肯定是在调用前，服务修改了请求协议

# 问题定位
经过代码跟踪，终于在 FeignLoadBalancer 中找到一段代码：

```java
@Override
public URI reconstructURIWithServer(Server server, URI original) {
	String scheme = original.getScheme();
	if (!"https".equals(scheme) && (this.serverIntrospector.isSecure(server) 
	|| this.clientConfig.get(CommonClientConfigKey.IsSecure, false))) {
	    original = UriComponentsBuilder.fromUri(original).scheme("https").build().toUri();
	}
	return super.reconstructURIWithServer(server, original);
}
```
这段代码的功能是拼接用来访问服务端的 url 。引人注目的是，它特意为 https 留下了不小的篇幅。

我们看下 serverIntrospector.isSecure(server) 做了什么

```java
public class DefaultServerIntrospector implements ServerIntrospector {
	@Override
	public boolean isSecure(Server server) {
		// Can we do better?
		return (""+server.getPort()).endsWith("443");
	}
}
```
# 答案揭晓
1. 因为 https 默认的端口是 443，tomcat 默认的SSL服务端口是 8443，因此作者为了图省事，就以服务端口是否以 "443" 结尾来决定服务方提供的是否是 https 服务。不过他也有过犹豫，自嘲地幽了一默：“Can we do better?”
2. 服务端 Docker 实例在启动时被随机分配了某个以 "443" 结尾的主机端口(我遇到的是30443)
3. 客户端在调用时，发现服务方的端口是以 443 结尾，认定服务方是https接口
4. 于是一个 http 服务被当作 https 服务调用，报了文章开头的错误

# 解决方法
1. 自行实现 ServerIntrospector 接口，并注入到spring容器中
2. 升级 spring-cloud-netflix 包。spring 从 1.4.0.M1 版本开始就修复了这个 bug