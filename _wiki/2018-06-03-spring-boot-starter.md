---
layout: wiki
title: 自定义 spring boot starter
categories: spring
description: 自定义 spring boot starter
keywords: spring
---
# 背景
spring cloud 提供了很多“开箱即用”的功能：默认不需要任何配置，只要引入相关包即可生效  
如果将我们的功能模块也加上这样的特性，需要做哪些事情呢？

# 第一：pom
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-autoconfigure</artifactId>
    <version>1.5.7.RELEASE</version>
</dependency>
```

# 第二：java
```java
@Configuration
public class BiziAutoConfiguration {
    @Bean
    public MyService myService(){
        return new MyService();
    }
}
public class MyService {
    public void testService(){
        System.out.println("===========");
    }
}
```

# 第三：spring.factories
文件位置： /resources/META-INF/spring.factories
```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=com.bizi.starter.BiziAutoConfiguration
```