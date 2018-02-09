---
layout: post
title: Sleuth 信息在线程之间的传递
categories: Blog
description: Sleuth 中信息如何在线程之间传递
keywords: 微服务，sleuth
---
# 问题
上期说，*Sleuth作为微服务下的调用链框架，支持traceId在各种多线程情况下的传递*。很抱歉，这个结论是错的，我在这里向大家道歉
# 分析
得出这个结论是因为官方文档的一句话[链接](http://cloud.spring.io/spring-cloud-static/spring-cloud-sleuth/1.3.0.RELEASE/multi/multi__integrations.html#_executor_executorservice_and_scheduledexecutorservice)：
> We’re providing LazyTraceExecutor, TraceableExecutorService and TraceableScheduledExecutorService. Those implementations are creating Spans each time a new task is submitted, invoked or scheduled.

>翻译：我们提供LazyTraceExecutor, TraceableExecutorService和TraceableScheduledExecutorService这三个实现类，以期在一个新~~线程~~任务提交、执行或调度时创建新的Span

这句话很有迷惑性，很容易让人误解(好吧我承认，目前只有我)，认为Sleuth支持各种多线程。遗憾的是，如果你手动创建了一个Thread，调用下一级服务时，Sleuth并不能感知到
# 证明
## 方法一
写个[Demo](https://github.com/bishion/microService)看下，是否TraceId能够正常传过去
## 方法二
我们看下**LazyTraceExecutor**是怎么做的
```java
public class LazyTraceExecutor implements java.util.concurrent.Executor {

	private static final Log log = LogFactory.getLog(MethodHandles.lookup().lookupClass());

	private Tracer tracer;
	private final BeanFactory beanFactory;
	private final Executor delegate;
	private TraceKeys traceKeys;
	private SpanNamer spanNamer;

	public LazyTraceExecutor(BeanFactory beanFactory, Executor delegate) {
		this.beanFactory = beanFactory;
		this.delegate = delegate;
	}

	@Override
	public void execute(Runnable command) {
		if (this.tracer == null) {
			try {
				this.tracer = this.beanFactory.getBean(Tracer.class);
			}
			catch (NoSuchBeanDefinitionException e) {
				this.delegate.execute(command);
				return;
			}
		}
		this.delegate.execute(new SpanContinuingTraceRunnable(this.tracer, traceKeys(), spanNamer(), command));
	}
}
```
使用方式
```java
@Configuration
public class MyConfiguration {
    @Autowired
    BeanFactory beanFactory;
    @Bean
    public Executor executor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        // CUSTOMIZE HERE
        executor.setCorePoolSize(7);
        executor.setMaxPoolSize(42);
        executor.setQueueCapacity(11);
        executor.setThreadNamePrefix("MyExecutor-");
        // DON'T FORGET TO INITIALIZE
        executor.initialize();
        return new LazyTraceExecutor(this.beanFactory, executor);
    }
}
```
所以它能在线程之间传递traceId并不稀奇，因为它根本就是要你使用它的多线程工具。
# 能否让子线程获取父线程信息呢
## 能：InheritableThreadLocal
一般来说，每个线程一个副本，我们都是用ThreaLocal。可是，如果你想要该线程和它的子线程都能读这个副本，那就可以用InheritableThreadLocal了。
用法很简单[Demo](https://github.com/bishion/study/tree/master/threads)：
```java
    private static final ThreadLocal<String> sessionInfoHolder1 = new ThreadLocal<String>();
    private static final ThreadLocal<String> sessionInfoHolder2 = new InheritableThreadLocal<String>();
```
# 思考
既然Sleuth不支持用户自己创建线程，而使用InheritableThreadLocal可以解决这个问题，那么是不是说，**Sleuth这个短板其实是可以解决的**?
## 方案
1. Filter中获取TraceId,存入：InheritableThreadLocal
2. TraceFeignClient中判断当前线程或其父线程中是否有traceId，如果有，就证明是用户new的线程，即可复用Trace信息。
3. 有兴趣的读者可以去Sleuth拉个分支试下，说不定会被管理员merge