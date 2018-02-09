---
layout: post
title: Sleuth 如何支持restTemplate
categories: Blog
description: Sleuth 如何支持restTemplate
keywords: 微服务，sleuth
---
# 问题
《微服务下基于sleuth的参数透传功能探索》这篇文章遗留了几个问题：

> ## 留下的坑
> 1. Sleuth通过LazyTraceExecutor解决多线程下的问题，但是它并没有解决给手动创建的Thread传递信息的问题
> 2. 有机会试试java字节码替换怎么操作
> 3. Sleuth如何重写RestTemplate的
> 4. TraceFeignClient怎么生成Client的实例

今天我们看下第三个：Sleuth是如何重写RestTemplate的

# 手动创建RestTemplate实例
```java
 @RequestMapping("/sayHello")
    public String sayHello(){
        return new RestTemplate().getForObject("http://localhost:9413/sayHello",String.class);
    }
```
然而这种方式，并不能将traceId传到后方。参考官方文档（http://cloud.spring.io/spring-cloud-static/Dalston.SR5/single/spring-cloud.html#_http_client_integration ）：
> ### Important
> You have to register RestTemplate as a bean so that the interceptors will get injected. If you create a RestTemplate instance with a new keyword then the instrumentation WILL NOT work

> 翻译：你必须将Restemplate注册成为一个Bean，这样interceptors才能注入进去。如果你是使用new的方式，调用链就不会奏效。

# 将RestTemplate注册成一个Bean
``` java
@Bean
public RestTemplate restTemplate(){
   	return new RestTemplate();
}
```
经过测试(https://github.com/bishion/microService/tree/master/sleuthconsumer)可知，这种方式可以将trace信息传给下一个服务
# 思考
将RestTemplate注册成Bean的时，只是简简单单地初始化了一个实例，并没有做什么样特殊的配置，那Sleuth是在什么时候将trace信息传给服务方的呢？
## 猜想一
Sleuth定义了一个切面，在RestTempate执行execute()方法时，在request中加入trace信息
### 分析
如果Sleuth是通过切面来解决RestTemplate的参数透传问题，那么它应该能同时支持Bean注册和手动初始化这两种方式。这个跟我们上面发现的现象矛盾
## 猜想二
是否使用了注册RestTemplate为Bean的方式，是trace信息能否传给服务方的关键因素。因此可以推测Sleuth应该是拿到了Bean之后，将Bean中的信息改写，替换成自己的实现方式
### 分析
Sleuth如何拿到Bean呢，无非两种方式：
1. 通过名称restTemplate
2. 通过RestTemplate类型

### 验证
1. 通过名称拿Bean：这种比较好验证，直接在Demo(参考前文github)中修改Bean注册时的名称即可，发现Sleuth还是能支持！分析一错误
2. 通过类型拿Bean：这种不太好验证，只能debug了。经过debug，我们能发现程序执行的时候确实是走了RestTemplate.execute()方法，没有找到使用切面的痕迹。分析二错误

# Debug
Debug过程中，我们发现，程序确实是在Resttemplate中执行的execute()方法。而且通过日志我们能看到，在程序执行完execute()后，服务方立刻就能将trace信息打印出来。令人费解的是，在执行exeute()方法之前，request**还没有**trace信息
``` java
// 节省篇幅，省略无关代码。具体参考：RestTemplate.java
protected <T> T doExecute(URI url, @Nullable HttpMethod method, @Nullable RequestCallback requestCallback,@Nullable ResponseExtractor<T> responseExtractor) throws RestClientException {
		ClientHttpResponse response = null;
		ClientHttpRequest request = createRequest(url, method);
		if (requestCallback != null) {
			requestCallback.doWithRequest(request);
		}
// 执行完这句,服务方就能获取到trace信息;而执行之前,request里没有trace信息!
		response = request.execute(); 
		handleResponse(url, method, response);
		if (responseExtractor != null) {
			return responseExtractor.extractData(response);
		}else {
			return null;
		}
	}
```
## 分析
通过上面信息可知，唯一能对Request做手脚的地方，就在这一句：
> response = request.execute();

就这么一行代码，手动创建的实例Sleuth无法生效，通过Bean初始化的实例却可以。接下来，我们看下这个方法里做了什么：
``` java
// 节省篇幅，省略无关代码。具体参考：InterceptingClientHttpRequest.InterceptingRequestExecution.execute()方法
private final Iterator<ClientHttpRequestInterceptor> iterator;
@Override
public ClientHttpResponse execute(HttpRequest request, byte[] body) throws IOException {
	if (this.iterator.hasNext()) {
		ClientHttpRequestInterceptor nextInterceptor = this.iterator.next();
		return nextInterceptor.intercept(request, body, this);
	}else {
		ClientHttpRequest delegate = requestFactory.createRequest(request.getURI(), request.getMethod());
		......
		return delegate.execute();
	}
}
```
上面代码中的***Interceptor(拦截器)***字样比较引人注意。经过debug，我们发现，当前ClientHttpRequestInterceptor的实现类是：TraceRestTemplateInterceptor。总算找到它了！！！

``` java
@Override
public ClientHttpResponse intercept(HttpRequest request, byte[] body,
			ClientHttpRequestExecution execution) throws IOException {
	publishStartEvent(request);// 就是这里添加的trace信息
	return response(request, body, execution);
}
```
## 思考
1. Slueuth通过在启动时将**所有**注册在Spring的RestTemplate实例加上了一个自己的Interceptor(TraceRestTemplateInterceptor)
2. TraceRestTemplateInterceptor在它的intercept()方法中加上了trace信息(参考：AbstractTraceHttpRequestInterceptor.publishStartEvent(HttpRequest request))
3. RestTemplate并不是http请求的最终执行方,它委托给了：AbstractClientHttpRequest的实现类

那么问题来了，程序如何选定实现类的呢

我们回看RestTemplate源码，最终由哪个Request执行，是在这里实现的：
> ClientHttpRequest request = createRequest(url, method);

## 答案
Sleuth通过给容器里所有的RestTemplate实例加上一个Spring提供的拦截器，实现了支持RestTemplate的功能

# 总结
其实排查到最后，问题变成了：RestTemplate如何自定义Header
> 在Spring3.1之前，我们通过HttpEntity给RestTemplate添加Header；spring3.1中有了 ClientHttpRequestInterceptor，通过实现这个接口，我们能自定义整个request信息

PS：RestTemplate在这里使用了很巧妙的方式，它没有在createRequest()方法里让我们把header信息添进去(当然，如果你愿意，也可以这么做)；它选择在执行http方法时，让我们用拦截器来添加自定义信息。嗯，责任链模式，装饰器模式傻傻分不清楚