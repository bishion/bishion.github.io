---
layout: post
title: RxJava 与 观察者模式
categories: 设计模式
description: RxJava 的观察者模式
keywords: 设计模式, 观察者模式, java
---
# 观察者模式
> The observer pattern is a software design pattern in which an object, called the subject,
 maintains a list of its dependents, called observers, and notifies them automatically of any state changes,
  usually by calling one of their methods. --Wikipedia

> 翻译：观察者模式是一种软件设计模式，它是由被观察对象（subject）、观察者(observer)组成。其中，被观察对象中维护了一个观察者的列表，当被观察者的状态发生改变时，它就自动通知到所有的观察者。--维基百科

# 代码
```java
public class subject{
    private String state;
    private List<Observer> observers = new ArrayList<>();
    
    public void updateState(String state){
        this.state = state;
        for(Observer observer : observers){
            observer.doSth(state);
        }
    }
    
    public void register(Observer observer){
        this.observers.add(observer)
    }
}

public interface Observer{
    public void doSth(String state);
}

public class Observer1 implements Observer{
    public void doSth(String state){
        System.out.println("the new state is :"+ state);
    }
}
```
# 分析
- 观察者模式并不是说观察者和被观察者完全解耦，被观察者中确实维护了一组观察者
- 观察者模式并不是说被观察者只需要关注自己的事情，被观察者状态改变后，它还要自己一个个通知到观察者
- 这个模式的名字并不贴切，因为并没有人在做『观察』的动作。所谓的观察者，只需要忙自己的事情，直到自己关心的事件找上门
- 观察者模式更应该叫『发布-订阅模式』，就像你关注了某个公众号，每当有新文章发布，你就能收到推文。
- 不过遗憾的是，『发布-订阅模式』这个名字被用到了另外一个场景中：消息中间件。该模式把通知的动作抽离到消息调度中心，由消息中间件来做,真正实现了发送方和接收方的解耦

# RxJava
> RxJava is a Java VM implementation of ReactiveX (Reactive Extensions): a library for composing asynchronous and event-based programs by using observable sequences.
 
> 翻译：RxJava 是一个基于 JVM 的 ReactiveX(响应式扩展)：一个使用可观察对象序列来编写异步和基于事件的程序库
 
由定义可知，RxJava 并不是一个观察者模式的实现框架，而是利用观察者模式来实现事件响应(可同步可异步)的工具库
 
## RxJava 的基本使用方法
```java
 @Test
 public void test2(){
     Observer<String> observer = new Observer<String>() {
         @Override
         public void onCompleted() {
             System.out.println("complete");
         }

         @Override
         public void onError(Throwable e) {
             System.out.println("error"); // 事件报错后自动触发
         }

         @Override
         public void onNext(String s) {
             System.out.println("next:"+s);
         }
     };
     Observable<String> observable = Observable.create(new Observable.OnSubscribe<String>() {
                 @Override
                 public void call(Subscriber<? super String> subscriber) {
                     System.out.println("called");
                     subscriber.onNext("teste");    // 这里调用的是观察者的 onNext() 方法
                     subscriber.onCompleted();      // 这里调用的是观察者的 onCompleted() 方法, onCompleted(),之后的消息就忽略了
                     subscriber.onError(new RuntimeException()); // 这句可以触发 onError() 方法
                     throw new RuntimeException()   // 这句可以触发 onError() 方法
                     
                 }
             });
    observable.subscribe(observer);  // 这里字面意思是：被观察者订阅观察者。Rxjava这么写是对链式调用的折衷，目前还体现不出来它的优势
}         
```
## RxJava 执行过程 
- 创建一个观察者 observer，订阅了三个方法：OnNext(), OnError(), OnCompleted()
- 创建一个被观察者 observable, 执行相关逻辑，将信息通知给观察者
- 开始注册：observable.subscribe(observer)。一旦订阅语句执行，被观察者的 call() 方法立即被触发，观察者就能获取到相关信息
 
## RxJava 分析
- RxJava 为了更好地对事件的各个维度进行响应，对观察者模式进行了纵向扩展：
    - 观察者只有一个
    - 响应事件有多个，onNext() 可以认为是updateState()，OnError()、OnCompleted() 都是扩展的方法
    - 观察者被观察者在事件来临时，都要重新初始化一次
- RxJava 不应该作为观察者的框架来学习，但是作为反应器模式(Reactor pattern)的框架还是很值得一看的
- 事件监听机制在 APP 中应用比较多，在 web 服务端的应用场景还有待开发
- feign 使用 RxJava 来实现失败重试就是事件机制的一个很典型的巧妙应用

# Reactor pattern
> The reactor design pattern is an event handling pattern for 
handling service requests delivered concurrently to a service handler by one or more inputs.
The service handler then demultiplexes the incoming requests and dispatches them synchronously 
to the associated request handlers. --Wikipedia

> 翻译：反应器模式是一种事件响应模式，它将一个或多个请求同时传给服务方。服务方在收到请求后将他们分发给相关的处理逻辑。 --维基百科

## 反应器模式与观察者模式
- 从定义上看，反应器模式就是事件驱动模式，也可以翻译为监听模式
- 反应器模式跟观察者模式很像，都是接到请求后，将信息传递给关注者
- 反应器模式更像是观察者模式的特例，反应器模式更多地关注事件本身
- 反应器模式在 NIO 中应用比较多

# 后记
本文从观察者模式谈到了 RxJava, 接着提及了事件响应，然后是反应器模式，接着到了 NIO。

其实对于 IO 的多路复用，除了 Reactor 模式之外，还有一种 Proactor 模式。有兴趣的同学可以看下
