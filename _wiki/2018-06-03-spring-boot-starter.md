---
layout: wiki
title: 自定义 spring boot starter
categories: spring
description: 自定义 spring boot starter
keywords: spring
---
# pom
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-autoconfigure</artifactId>
    <version>1.5.7.RELEASE</version>
</dependency>
```
# java
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

#spring.factories
文件位置： /resources/META-INF/factories
```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=com.bizi.starter.BiziAutoConfiguration
```