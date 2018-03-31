---
layout: post
title: feign 的重试机制
categories: 微服务
description: feign 的重试机制
keywords: 微服务，feign, retry
---
# 问题描述
feign 默认会重试，经常会造成数据重复，怎么关闭呢
# 分析
重试机制在很多业务场景下都扮演着重要角色，比如接口查询、多供应商服务选择等

良好的重试设计可以使系统可用性大大增强

但是，不合理的重试也会带来数据重复甚至服务方雪崩

feign 作为 spring cloud 默认的服务调用框架，肯定有令人惊叹的重试机制

## feign 如何做重试
### SynchronousMethodHandler.invoke() 源码
```java
@Override
  public Object invoke(Object[] argv) throws Throwable {
    RequestTemplate template = buildTemplateFromArgs.create(argv);
    Retryer retryer = this.retryer.clone(); // 这句很重要
    while (true) {
      try {
        return executeAndDecode(template);
      } catch (RetryableException e) {
        retryer.continueOrPropagate(e);
        if (logLevel != Logger.Level.NONE) {
          logger.logRetry(metadata.configKey(), logLevel);
        }
        continue;
      }
    }
  }

```
### 分析
从代码上来看，feign 的重试机制还是比较简单的：
1. 进来一个 while (true) 死循环，然后去请求远程服务
2. 如果服务正常运行，立即返回
3. 如果服务运行异常，就在异常捕获中执行 Retryer.continueOrPropagate(e) 逻辑
4. 跳出死循环的条件在于：程序正常返回，或者 continueOrPropagate(e) 也抛异常
5. 该循环还有一个终止条件：executeAndDecode() 方法抛出非 RetryableException 异常，不过程序肯定对这个异常类型做了封装，我们待会儿再看

我们看下该方法做了哪些：
```java
public void continueOrPropagate(RetryableException e) {
  if (attempt++ >= maxAttempts) {
    throw e;
  }

  long interval;
  if (e.retryAfter() != null) {
    interval = e.retryAfter().getTime() - currentTimeMillis();
    if (interval > maxPeriod) {
      interval = maxPeriod;
    }
    if (interval < 0) {
      return;
    }
  } else {
    interval = nextMaxInterval();
  }
  try {
    Thread.sleep(interval);
  } catch (InterruptedException ignored) {
    Thread.currentThread().interrupt();
  }
  sleptForMillis += interval;
}
```
1. 如果重试次数已经超过允许的值，就抛异常，按照上文分析，此时死循环结束
2. 如果还可以重试，就歇一会儿，按照上文分析，歇完会再次请求远程服务，从而实现重试

### 注意
1. 上文是 Retryer.DEFAULT 的 continueOrPropagate(e) 的源码，Restryer 是一个接口，不止这一个实现类
2. 因为 Restryer 需要记录一些状态信息(比如重试间隔，重试次数)，所以使用前，必须执行 clone() 方法，初始化一个新的实例

# 问题解决
自己实现 Retryer, 然后自定义 continueOrPropagate(e)，这样就可以随意控制是否重试了。

# 还有问题
feign 默认使用的 Retryer 的实现类就是 Retryer.NEVER_RETRY，而且经排查，程序生效的就是它（排除其他 jar 包偷偷实现重试机制的情况）。

```java
@Bean
@ConditionalOnMissingBean
public Retryer feignRetryer() {
    return Retryer.NEVER_RETRY;
}

@Bean
@Scope("prototype")
@ConditionalOnMissingBean
public Feign.Builder feignBuilder(Retryer retryer) {
    return Feign.builder().retryer(retryer);
}
```
但是程序还是**依然在重试!!!**

## 隐藏的重试

除了前文的 Retryer.continueOrPropagate(e) 之外，程序唯一能动手脚的就是那句 executeAndDecode(template) 了

查看源码，才发现本来一个简单的执行 Request 请求的方法，被 feign 诠释得异常复杂

### AbstractLoadBalancerAwareClient.executeWithLoadBalancer() 源码
```java
// AbstractLoadBalancerAwareClient.executeWithLoadBalancer
public T executeWithLoadBalancer(final S request, final IClientConfig requestConfig) throws ClientException {
    RequestSpecificRetryHandler handler = getRequestSpecificRetryHandler(request, requestConfig);
    LoadBalancerCommand<T> command = LoadBalancerCommand.<T>builder().withLoadBalancerContext(this).withRetryHandler(handler).withLoadBalancerURI(request.getUri()).build();
    try {
        return command.submit(
        new ServerOperation<T>() {
            @Override
            public Observable<T> call(Server server) {
                URI finalUri = reconstructURIWithServer(server, request.getUri());
                S requestForServer = (S) request.replaceUri(finalUri);
                try {
                    return Observable.just(AbstractLoadBalancerAwareClient.this.execute(requestForServer, requestConfig));
                } 
                catch (Exception e) {
                    return Observable.error(e);
                }
            }
        }).toBlocking().single();
    } catch (Exception e) {
        Throwable t = e.getCause();
        if (t instanceof ClientException) {
            throw (ClientException) t;
        } else {
            throw new ClientException(e);
        }
    }
}
```
### executeWithLoadBalancer() 分析
1. 从方法名可知，feign 在请求服务端时，用的是客户端负载的方式(使用 Ribbon 从 consul 返回的服务列表中选取一个来执行)
2. 在这个方法里，就将所有的异常包装成 ClientException 了。不难想象，肯定有地方将它再包装成前文提到的 RetryableException
3. 这个方法用到了 rx.Observable, 一个强大的响应式工具包。
> RxJava is a Java VM implementation of ReactiveX (Reactive Extensions): a library for composing asynchronous and event-based programs by using observable sequences.

> 翻译：RxJava 是一个基于 JVM 的 ReactiveX(响应式扩展)：一个使用可观察对象序列来编写异步和基于事件的程序库

从 RxJava 的定义和前文代码可知，feign 使用 rx.Observable 实现了基于观察者模式的重试机制。重试的逻辑在 LoadBalancerCommand.submit() 中

### LoadBalancerCommand.submit() 源码 -- 简化版
```java
public Observable<T> submit(final ServerOperation<T> operation) {
    final int maxRetrysSame = retryHandler.getMaxRetriesOnSameServer(); // 相同 server 重试次数
    final int maxRetrysNext = retryHandler.getMaxRetriesOnNextServer(); // 不同 server 重试次数

    Observable<T> o = selectServer().concatMap(new Func1<Server, Observable<T>>() {

                public Observable<T> call(Server server) {
                    Observable<T> o = Observable.just(server).concatMap(new Func1<Server, Observable<T>>() {
                        public Observable<T> call(final Server server) {
                            return operation.call(server).doOnEach(new Observer<T>() { // 执行前文传入的 ServerOperation.call() 方法
                                ...... // 这段主要是记录日志
                            });
                        }
                    });
                    
                    if (maxRetrysSame > 0) // 相同 server 重试
                        o = o.retry(retryPolicy(maxRetrysSame, true));
                    return o;
                }
            });
        
    if (maxRetrysNext > 0 && server == null){
        o = o.retry(retryPolicy(maxRetrysNext, false));
    }
}
```

### submit() 分析
这段让人眩晕的代码中，最显眼的莫过于 **maxRetrysNext** 和 **maxRetrysSame**了

这两个变量控制着调用服务时的重试机制，而且对于服务方的选择也有了高级定制

经过代码追踪，发现了这两个变量的默认值

### DefaultClientConfigImpl 源码 -- 简化版
```java
public static final int DEFAULT_MAX_AUTO_RETRIES_NEXT_SERVER = 1;  // 默认不同 server 重试次数

public static final int DEFAULT_MAX_AUTO_RETRIES = 0;              // 默认相同 server 重试次数

# CommonClientConfigKey 类
public static final IClientConfigKey<Integer> MaxAutoRetries = new CommonClientConfigKey<Integer>("MaxAutoRetries"){};  
public static final IClientConfigKey<Integer> MaxAutoRetriesNextServer = new CommonClientConfigKey<Integer>("MaxAutoRetriesNextServer"){};
```

## feign 默认重试机制分析
1. feign 默认不会使用 Retryer 重试，如果你希望自定义重试，可以自己实现 Retryer 接口，并注入
2. feign 默认会使用 Observable 做一次不同 server 的重试(如果你只有一个 server，它只能再对这台 server 重试一次)
3. 如果想要彻底关闭重试，需要加上配置：ribbon.MaxAutoRetriesNextServer = 0
4. feign 的重试做的很理性，毕竟无论是不是微服务，只有一个服务提供方也太寒碜

# 扩展
如果我想对不同服务做不同的重试机制怎么办呢
```properties
app1.ribbon.MaxAutoRetriesNextServer=1   // 对 app1 不同 server 重试一次
app1.ribbon.MaxAutoRetries=0             // 对 app1 相同 server 不做重试
app2.ribbon.MaxAutoRetriesNextServer=2   // 对 app1 不同 server 重试两次
app3.ribbon.MaxAutoRetries=1             // 对 app1 相同 server 重试一次    
```

# 思考
重试机制适用场景
- 多个服务方可供选择
- 网络抖动比较频繁
- 对响应结果敏感

重试机制不适用场景
- 数据提交，且系统对重复数据敏感
- 服务方只有一个(重试会加重服务方负载)
- 对响应时间敏感


feign 在重试机制上给了我们不同的选择：
- 统一的重试处理，你可以用 while(true)的方式
- 不同服务方的定制重试，你可以用 Observable 的事件机制

RxJava 建议有兴趣的同学多了解下，相信它会为你打开新世界的大门