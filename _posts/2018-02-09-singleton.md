---
layout: post
title: 单例不简单
categories: 设计模式
description: java 单例的总结
keywords: 设计模式, 单例模式, java
---
# 需求
用java实现一个单例模式

# 分析
单例模式，顾名思义，就是单独实例的模式。它是设计模式中最简单的一种，就像阿飞的剑法，不花哨却很有效。

想让程序中只有一个实例，其实很简单：只允许它被创建一次即可。
# 解决方案

## 方案一
``` java
public class Singleton {
    private static final Singleton singleton = new Singleton();
    private Singleton(){
    }
    public Singleton getInstance(){
        return singleton;
    }
}
```
### 分析
#### 优点
1. 构造方法私有，别人无法创建，满足单例要求
2. 简单易懂

#### 缺点
1. 启动时就要初始化，如果初始化很复杂，启动很慢
2. 如果该实例没有被用到，资源浪费

## 方案二
``` java
public class Singleton {
    private static Singleton singleton;
    private Singleton(){}
    public Singleton getInstance(){
        if(singleton == null){ //标记2A
            singleton = new Singleton();
        }
        return singleton;
    }
}
```
### 分析
#### 优点
1. 简单易懂
2. 在第一次需要使用的时候初始化,达到延迟加载的效果

#### 缺点
1. 没有考虑到多线程问题
2. 如果两个线程同时走到了“标记2A”处，则会导致初始化两次

## 方案三
``` java
public class Singleton {
    private static Singleton singleton;
    private Singleton(){}
    public synchronized Singleton getInstance(){
            if(singleton == null){
                singleton = new Singleton();
        }
        return singleton;
    }
}
```
### 分析
#### 优点
1. 懒加载
2. 支持多线程

#### 缺点
每次获取实例时都会加锁，如果调用频繁，大大降低性能

## 方案四
``` java
public class Singleton {
    private static Singleton singleton;
    private String fileName;
    private Singleton(){
        File file = getRemoteFile(); // 标记4A,这里会执行很久
        this.fileName = file.getName(); 
    }
    public Singleton getInstance(){
        if(singleton == null){  // 标记4B 
            synchronized (Singleton.class){
                if(singleton == null){
                    singleton = new Singleton();
                }
            }
        }
        return singleton;
    }
}
```
### 分析
#### 优点
1. 懒加载
2. 传说中的双重检查锁，支持多线程
3. 频繁调用时也不会有性能损耗

#### 缺点
极端情况下，线程A走到了“标记4A”处时阻塞，singleton实例已经分配了内存空间，但是没有完全初始化；此时线程B走到了“标记4B”处，它发现singleton不为null，于是将没有初始化完成(fileName还为null)的singleton返回

## 方案五
``` java
// 其他跟方案四一样，只是将singlerton用volatile修饰
private static volatile Singleton singleton;
```

### 分析
#### 优点
1. 懒加载
2. 支持多线程
3. volatile保证了每次去读的时候，对象都是最新的副本(注意，这里并不能完全避免对象是半成品，只是减小概率)

#### 缺点
1. 效率低，每次调用时，强制检查singleton是否共享
2. 对象的创建有好多种方式(构造方法,反射,clone,反序列化)，而这里只考虑了构造方法的方式
3. 本身这种方式就比较复杂
4. volatile 是jdk1.5后才支持

## 方案六
``` java
public enum SingletonEnum{
    singleton;
    SingletonEnum(){
        // 复杂的初始化
    }
    public void myMethod(){
        // 一些操作
    }
}
```
### 分析
#### 优点
1. 简单明了
2. 枚举，java底层支持的单例，品质保证
3. 抗序列化，保证线程安全，无法用反射初始化
4. Effective java推荐的单例方式

#### 缺点
1. 不是懒加载(基本也算懒加载了，*复杂的初始*化在第一次调用才会执行)
2. jdk1.5后才支持
3. 失去普通类的特性

## 方案七
``` java
public class Singleton {
    private static class SingletonClassInstance {
        private static final Singleton instance = new Singleton();
    }
    public static Singleton getInstance() {
        return SingletonClassInstance.instance;
    }
    private Singleton() {}
}
```
### 分析
#### 优点
1. 懒加载
2. 不受jdk版本限制

#### 缺点
1. 只限制了构造方法初始化
2. 额外生成了一个永生代的类
3. 增加了程序的复杂性

# 总结
没有最好的实现方式，只有最适合自己的方式