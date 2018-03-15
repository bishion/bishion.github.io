---
layout: wiki
title: volatile 不支持非原子操作
categories: 多线程
description: 多线程下，volatile 不支持非原子操作
keywords: java 多线程，multi-threading
---
## 先上代码
```java
public class VolatileTest {

    public volatile int inc = 0; // 注意这里
    
    public void increased() {
        inc++;
    }
    public static void main(String[] args) {
        final VolatileTest test = new VolatileTest();
        for (int i = 0; i < 100; i++) {
            new Thread() {
                public void run() {
                    for (int j = 0; j < 1000; j++){
                        test.increased();
                        try {
                            Thread.sleep(10);
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                }
            }.start();
        }
        while (Thread.activeCount() > 2) { // 保证子线程线程执行完毕后，再终止程序
            Thread.yield();
        }
        System.out.println(test.inc);
    }
}
```
## 执行结果
每次执行结果都不一样，但是都是小于100000的一个数字

## 分析
按说 inc 使用了 volatile 修饰，线程在每次读取 inc 之前，都会拿最新的副本，应该是线程安全的。

但是，上面那句话里有个前提 **线程在每次读取 inc 之前**，也就是说，只是在读的时候去检查数据是不是最新的版本。

极端情况如下：
1. 两个线程 A 和 B，两个线程同时读了最新的副本：800，接着同时做 inc + 1
2. A 线程执行 inc = 801，接着释放，轮到 B 线程执行 inc = 801 
3. 因为 2 步骤只有写入步骤，线程不会检查最新版本的 inc，因此 inc 的值最终为 801