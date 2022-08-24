---
layout: post
title: 阿里云 oss sts 使用不当引起的 oom 问题排查
categories: oss, sts
description: 使用阿里云 oss sts功能时，忘记关闭旧的ossclient，导致系统 oom
keywords: oss, sts, oom
---
# 阿里云 oss sts 使用不当引起的 oom 问题排查

## 背景
每隔几天，应用就会重启一次，监控发现应用内存一直在涨。下图是连续两天的系统dump日志，我们发现 *DefaultServiceClient* 占用的内存涨势惊人：

![第一天dump](/images/20220822-mat-oss-914.png)
![第二天dump](/images/20220822-mat-oss-915.png)

## 问题原因
看到问题的第一反应就是：难道 ossClient 并非线程安全？不过很快打消念头，程序运行几个月了并未异常。而且官方文档中也查到了，[OSS Java SDK 多线程安全](https://help.aliyun.com/document_detail/32024.html#section-flp-kbr-h92)。

经过代码排查，我们发现我们用到了 oss 的 [sts](https://help.aliyun.com/document_detail/100624.html) 特性，而使用不当，会很容易导致应用 oom。


## sts 使用流程
1. 先调用服务方接口，获取一个临时访问的token, 以及过期时间
2. 在过期时间内使用 token 创建一个ossClient
3. 如果 token 过期，则再重新执行步骤 1

该流程实现起来非常简单，上代码：

> 原版代码，团队小朋友使用的是缓存，但是看着太复杂，这里简化处理

```java
    // 记录当前在使用的 ossClient
    private OSSClient ossClient;
    // 记录当前 ossClient 的过期时间
    private Long expireTime = System.currentTimeMillis();
    public OSSClient getOssClient() {
        if (expireTime < System.currentTimeMillis()){
            synchronized (this){
                if(expireTime < System.currentTimeMillis()){
                    // 调用对方接口获取临时访问凭据
                    TempOssTokenDTO tempOssTokenDTO = this.applyTempOssToken(tempTokenReq);
                    // 将过期时间降低五分钟，防止拿到临过期的连接
                    this.expireTime = tempOssTokenDTO.getExpireTime() - 30000;
                    this.ossClient = (OSSClient) new OSSClientBuilder().build(endpoint,tempOssTokenDTO.getAccessKeyId(), tempOssTokenDTO.getAccessKeySecret(), tempOssTokenDTO.getSecurityToken());
                }
            }
        }
        return ossClient;  
    }
```

## 存在的问题
上述代码有个很大的问题，就是没有关闭已经过期的 ossClient，随着程序的运行，未释放的连接越来越多，进而程序 oom

## 解决思路
这个问题从表象上来看，很容易去解决：在生成新的 ossClient 的时候，顺便将老的 ossClient 关闭就可以了。但是，一个比较棘手的问题在于，如果我们直接关闭连接会导致正在使用该连接的操作报错，进而引发故障。

所以，解决问题的核心思路变成了：如何在生成新的连接时，等旧的连接都不用了，再关闭。

### 方法一：使用计数器
这里提供思路，不具备业务环境可操作性
1. 使用 Map<OssClient, Integer> 存放 ossClient 被引用的次数
2. 当线程获取到 ossClient 时，计数器加 1
3. 当线程使用 ossClient 完毕时，计数器减 1
3. 等到原有 ossClient 临近过期时，丢给一个线程，线程在等待该 ossClient 引用数为0的时候，执行关闭操作

### 方法二：使用 jdk 延时队列
如果不考虑那么精确的话，其实我们可以预估一个时间（比如 5 分钟），在获取新的连接后，旧的连接只给一个收尾时间，然后就关闭。

所以，解决问题的核心思路变成了：如何在生成新的连接时，延迟一段时间关闭旧的连接？对的，延时队列。

很多队列组件都可以支持该功能，JDK 自带了一个延时队列，DelayQueue，可以用用看：

```java
    // 使用延时队列，存放快过期的 OssClient
    private DelayQueue<OssClientDelayed> ossClientQueue = new DelayQueue<>();
    public OSSClient getOssClient() {
        if (expireTime < System.currentTimeMillis()){
            synchronized (this){
                if(expireTime < System.currentTimeMillis()){
                    // ... 
                    // 将旧的 ossClient 放到延时队列中去
                    ossClientQueue.offer(new OssClientDelayed(ossClient,expireTime));
                    this.ossClient = (OSSClient) new OSSClientBuilder().build(endpoint,tempOssTokenDTO.getAccessKeyId(), tempOssTokenDTO.getAccessKeySecret(), tempOssTokenDTO.getSecurityToken());
                    // 触发释放旧的连接
                    this.getAndShutdown();

                }
            }
        }
        return ossClient;  
    }
    private void getAndShutdown(){
        // 这里防止阻塞，用了poll，如果旧的还没过期，就忽略
        OssClientDelayed ossClientDelayed = ossClientQueue.poll();
        if(ossClientDelayed !=null){
            ossClientDelayed.getOssClient().shutdown();
        }
    }
```

### 方法三：使用缓存的失效回调功能
上述方法二为了实现简单，只在创建新的连接时，才触发释放旧的连接，会导致释放过于延后（可以理解为，在创建第三个连接时，才真正释放掉第二个连接，想想为啥）。

除了延时队列，缓存失效回调功能，也可以用在这个场景。

无论是 Redis 还是 guava Cache 都会有 key 失效回调功能。所以这里，我们选择使用 guava 来做：
```java
    private Cache<String, OSSClient> ossClientCache = CacheBuilder.newBuilder()
            // 放到缓存里，5 分钟后过期
            .expireAfterWrite(300, TimeUnit.SECONDS)
            // 过期后，执行过期方法：关闭 ossClient
            .removalListener((RemovalListener<String, OSSClient>) notification -> notification.getValue().shutdown())
            .build();

    // 提前十分钟就开始获取下一次的 token
    private static final Long EXPIRE_INTERVAL = 1000 * 60 * 10L;
    public OSSClient getClient() {
        if (expireTime - System.currentTimeMillis() < EXPIRE_INTERVAL) {
            synchronized (this) {
                if (expireTime - System.currentTimeMillis() < EXPIRE_INTERVAL) {
                    // 将老的 ossClient 放到缓存中，让缓存管理它的后续释放工作
                    if (Objects.nonNull(ossClient)) {
                        this.ossClientCache.put(this.ossClient.toString(), this.ossClient);
                    }
                    // 其他照旧 。。。。
                }
            }
        }
        return ossClient;
    }
    
```
## 后记
1. 这个场景针对使用 sts 访问 ossClient 的情况，日常将 ossClient 注入到 spring 容器中使用没有问题
2. ossClient 导致 oom, 排查过程比较偶然，我也是无意间翻到了大对象列表中有 oss 的身影，就多看了几眼
3. 团队最终采用的是方案三，不过我觉得方案二也可以接受
4. 潜意识里觉得可能还有更好的方案，只能等未来某天的顿悟或者大神的指教了
