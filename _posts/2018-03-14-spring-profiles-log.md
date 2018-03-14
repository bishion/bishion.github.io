---
layout: post
title: spring cloud 根据不同环境加载不同 log 配置
categories: 微服务
description: spring cloud 根据不同环境加载不同 log 配置
keywords: 微服务，logback
---

# 问题
目前微服务中，logback的配置是写死的（如下）。这样会导致一个问题，本地需要看控制台时，就把控制台日志放开；
提交代码时，又需要手动把控制台日志注释掉
经常有项目组忘记这个，导致生产应用还在打控制台日志
``` xml
<root>
   <level value="INFO" />
   <appender-ref ref="STDOUT" />
</root>
```

# 解决
1. 将logback.xml改名为：logback-spring.xml
2. 用<springProfile>将<root>标签包裹（如下）
3. 本地启动时，加上-Dspring.active.profiles=dev的参数（如下截图）
``` xml
<springProfile name="dev">
   <root>
      <appender-ref ref="STDOUT" />
   </root>
</springProfile>
<springProfile name="!dev">
   <root>
      <level value="INFO" />
   </root>
</springProfile>
```
