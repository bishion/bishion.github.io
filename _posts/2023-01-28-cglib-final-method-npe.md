---
layout: post
title: 模板模式中，子类自定义方法添加 @Transactional 导致的 NPE 问题
categories: bug, cglib, final, npe
description: 使用 cglib 作为动态代理, 如果被代理类中有 final 方法，则容易出现空指针问题
keywords: cglib, final
---
# 模板方法添加 @Transactional 导致的 NPE 问题

## 背景
1. 项目中使用了模板模式，模板方法加了个 final 以示专业
2. 在一次需求中，子类方法使用了 @Transactional 注解来做事务管理
3. 运行时喜提空指针

### 大致代码如下：
```java
/**
 * 模板类，先保存通用订单信息，然后调用子类实现去保存个性化扩展信息
 */
public abstract class AbstractOrderService {

    public abstract void saveExtendData(String data);

    @Resource
    private OrderMapper orderMapper;


    public final void createOrder(String data) {
        orderMapper.insertOrder(data); // 空指针报错处，OrderMapper 为null
        saveExtendData(data);

    }
}

/**
 * 保存书籍订单的扩展信息
 */
@Service
public class BookOrderService extends AbstractOrderService {
    /**
     * 这里使用了声明式事务做后续处理
     */
    @Override
    @Transactional
    public void saveExtendData(String data) {
        // 
    }
}


```

## 问题排查
1. 因空指针在基类中出现，导致一开始认为是 *OrderMapper* 注入出现问题，但是而基类并没有被修改过
2. 其他子类正常运行，只有 *BookOrderService* 才有该问题
3. debug 发现，*BookOrderService* 类名为 *BookOrderServic$$EnhancerBySpringCGLIB$$45a96d9a*, 表示被 cglib 做了代理
4. cglib 生成的代理类实际上是继承了目标类 *BookOrderService*, *createOrder()* 方法因为是 *final* 修饰的，无法被 cglib 重写
5. 执行的 *bookOrderService.createOrder()* 并没有被代理，而是直接执行，此时类中所有的成员变量均为 null
6. 参考资料: https://www.liaoxuefeng.com/wiki/1252599548343744/1339039378571298

## spring 环境下，cglib 执行原理分析
1. spring 在 *BookOrderService* 初始化完毕时，会判断该类是否需要被代理: *org.springframework.aop.framework.autoproxy.AbstractAutoProxyCreator#wrapIfNecessary*
2. 如果该类需要被代理，则会生成目标类的代理类(*DynamicAdvisedInterceptor*), 并将上一步初始化好的 *BookOrderService* 放入targetSource 中，目标类的成员变量会被忽略。参考下图1
3. 代理类中会生成一个跟目标类相同的方法名, 方法实现则为调用 *DynamicAdvisedInterceptor.intercept()*, **但是 final 方法除外**
4. *intercept()* 最终会通过反射，调用到已经初始化完毕的 *BookOrderService* 对应方法。参考下图2
5. 如果我们将日志级别调整为 debug，则可以在启动日志中看到如下内容：
> Final method [public final void io.github.bishion.demo01.cglib.AbstractOrderService.createOrderWithFinal(java.lang.String)] cannot get proxied via CGLIB: Calls to this method will NOT be routed to the target instance and might lead to NPEs against uninitialized fields in the proxy instance.

##### 图1: 代理类的成员变量均为null
![图1: 代理类的成员变量均为null](/images/20230128-cglib-proxy-class-field.png)

##### 图2: 被代理方法在最终执行时，用的是目标类的 bean
![图2: 被代理方法在最终执行时，用的是目标类的 bean](/images/20230128-cglib-proxy-method-has-fields.png)   

## 如何查看代理类的源码
### 方法一：使用sun.jvm.hotspot.HSDB
具体可以参考博客： https://blog.csdn.net/qq_39504520/article/details/106086491

### 方法二：设置参数，保存cglib生成的class文件
在启动方法中，添加如下代码即可：
```java
public static void main(String[] args){

    Properties property = System.getProperties();
    // proxyClassFolder 为要保存的代理类的目录
    System.setProperty(DebuggingClassWriter.DEBUG_LOCATION_PROPERTY, proxyClassFolder);
    property.put("net.sf.cglib.core.DebuggingClassWriter.traceEnabled", "true");

    SpringApplication.run(Application.class, args);
}

``` 
## demo源码地址
https://github.com/bishion/daily-demo/tree/main/demo01-cglib-final-npe