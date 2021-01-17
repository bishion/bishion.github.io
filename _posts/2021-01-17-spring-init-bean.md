---
layout: post
title: BeanPostProcessor 在单测中的应用
categories: spring
description: 使用 BeanPostProcessor 实现单测中的 mock 功能
keywords: spring, junit, mock
---
# BeanPostProcessor 在单测中的应用
## 背景
项目的核心功能依赖第三方服务，封装在接口 *RemoteService* 的实现类 *RemoteServiceImpl* 中，并注册为 spring 的一个 bean。
和所有第三方服务一样，该服务并不稳定，这就要求单测时，必须要对该服务进行 mock。
### 相关基础代码如下
```java
/** 封装远程调用服务接口*/
public interface RemoteService {
    String getRepFromRemote(Integer param);
}

@Service
public class RemoteServiceImpl implements RemoteService {
    public String getRepFromRemote(Integer param) {
        return "Hello, I come from remote";
    }
}
/** 业务服务 */
public interface BizService {
    String doSthByRemote();
}

@Service
public class BizServiceImpl implements BizService{
    @Resource
    private RemoteService remoteService;

    public String doSthByRemote() {
        return "Remote says: "+remoteService.getRepFromRemote(123);
    }
}
```

## 分析
这个场景很普通，应该是所有 mock 组件的基本功能。
## Mockito?
我选择了常见的Mockito 来解决这个问题。
```java
@SpringBootTest
@RunWith(SpringRunner.class)
public class RemoteServiceTest1 {

    @Resource
    BizService bizService;
    @MockBean
    RemoteService remoteService;
    @Test
    public void getRepFromRemote() {
        Integer param = 123;
        String mockResult = "Hello world";
        initData(param,mockResult);
        Assert.assertEquals("Remote says: "+mockResult,bizService.doSthByRemote());
    }
    
    private void initData(Integer param, String mockResult){
        Mockito.when(remoteService.getRepFromRemote(param)).thenReturn(mockResult);
    }
}
```
### 缺点
1. 有小伙伴觉得引入了一个新组件，有学习成本
2. 大多数情况下，待测桩模块都在该服务上面好几层，*@MockBean* 容易被忽略
3. 每次还要在那里when().then 地重复构造数据，太累
4. 有没有更简单无脑的方式呢？

### 思考
因为远程服务是被封装为 bean，我们可以在 RemoteService 注入到容器前，将它用另外一个 mock 的类替换。

在 spring bean 创建过程中，有两个方法刚好复合这个场景：
1. InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation(...)
2. BeanPostProcessor.postProcessBeforeInitialization(...)

所以我们只需要新建一个 mock 类，实现 RemoteService 和上述两个接口方法，然后在目标 bean 注入到容器时做替换即可
## 方案一：使用 postProcessBeforeInstantiation
该方法在类创建之前执行
方法参数为目标 bean 对应的 *beanClass* 和 *beanName*，默认返回是 null 对象, 表示该 bean 没有代理对象，正常创建即可。

如果返回是一个非 null 对象，则表示目标 bean 被该返回对象所代理，后续的创建工作会立即停止，容器直接使用该对象。
后续 *postProcessAfterInstantiation* 不再执行，会直接跳到 *BeanPostProcessor.postProcessAfterInitialization*

### 实现代码
#### mock 代码
```java
@Primary // 注意，该字段必须设置，否则可能会报【检测到多个匹配的 bean 】的异常
public class MockRemoteService1 implements RemoteService , InstantiationAwareBeanPostProcessor {
    public String getRepFromRemote(Integer param) {
        return "Hello, I am Mock1";
    }

    public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {

        if (beanName.startsWith("remoteService")){
            return this;
        }
        return null;

    }
}
```
#### 测试代码
```java
@SpringBootTest(classes = {Application.class,MockRemoteService1.class})
@RunWith(SpringRunner.class)
public class RemoteServiceTest {
    @Resource
    BizService bizService;

    @Test
    public void getRepFromRemote() {
        Assert.assertEquals("Remote says: Hello, I am Mock1",bizService.doSthByRemote());
    }
}
```
## 方案二：使用 postProcessBeforeInitialization
方法在 bean 刚被创建完成之后，初始化（比如调用 init-method）之前执行。
该方法参数为目标 bean 对应的 *bean* 和 *beanName*，默认返回是 bean 对象，不做任何处理。

### 实现代码
#### mock 代码
```java
@Primary
public class MockRemoteService2 implements RemoteService , InstantiationAwareBeanPostProcessor {
    public String getRepFromRemote(Integer param) {
        return "Hello, I am Mock2";
    }

    public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {

        if (beanName.startsWith("remoteService")){
            return this;
        }
        return null;

    }
}
```
#### 测试代码
参考方法一

## 总结
1. 在当前场景下，两种替换方式等效的，基本上没有什么差别
2. MockService 不能直接使用 *@Service* 注解，因为会有 bean 冲突，而且因为单测特殊性，本身也需要在不同 case 中指定mock数据
3. 在当前项目背景下，该方式比 Mockito 要简约很多
4. 如果不想用 @Primary 注解，可以单独写一个MockService实现 remoteService，然后直接在替换时 new 出来即可
5. 通过 bean 替换的方式来做 mock 还是挺巧妙

