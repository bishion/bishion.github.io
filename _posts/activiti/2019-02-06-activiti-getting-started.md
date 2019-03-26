---
layout: post
title: Activiti 6.0 用户文档 - 第二章：开始
categories: activiti
description: 流程引擎 Activiti 第二章：开始
keywords: 流程引擎, activiti
---
# 正式开始
## 一分钟预览
在 [Activiti 官网](http://www.activiti.org/) 下载好 Activiti UI WAR 包之后,按照如下几个步骤就可以将 demo 以默认配置启动. 你还需要 [Java 环境](http://java.sun.com/javase/downloads/index.jsp) 和 [Tomcat](http://tomcat.apache.org/download-80.cgi) (事实上,任何一个 web 容器都可以, 我们只是依赖 servlet 功能,但是 我们只在 Tomcat 上面测试过)

- 将下载的 activiti-app.war 包放到 Tomcat 的 webapps 目录下面
- 运行 Tomcat bin 目录下的 startup.bat 或者 startup.sh 脚本
- 当 Tomcat 启动后, 打开浏览器, 在地址栏输入: http://localhost:8080/activiti-app , 输入用户名 admin 和密码 test

一切搞定! Activiti UI 默认使用的是 H2 内存型数据库, 如果你想配置其他的数据, 你可以阅读[启动设置](https://www.activiti.org/userguide/index.html#activiti.setup)
## 2.2 Activiti 设置
在运行 Activiti 之前,你需要安装 [Java 环境](http://java.sun.com/javase/downloads/index.jsp) 和 [Tomcat](http://tomcat.apache.org/download-80.cgi). 而且,确认已经设置好 ***JAVA_HOME*** 的环境变量. 设置环境变量的方式取决于你的操作系统, 这里就不细说了   
运行 Activiti UI 和 REST web 应用只需要将下载好的 WAR 包放到 Tomcat 下的 webapps 目录下. UI 应用默认使用的是一个内存数据库  
案例用户:

| 用户名 | 密码 | 用户角色 |
| ----- | ----- | ----- |
| amin | test | admin |

接着你就可以进入到下面这个 web 应用了
| 应用名称 | URL | 描述 |
| ----- | ----- | ----- |
| Activiti UI | http://localhost:8080/activiti-app | 用户操作页面. 使用它你可以新建一个流程, 分配、查看和审批任务 |

注意, 启动 Activiti UI 应用只是一种简单快速地展示 Activiti 特性和功能的方式. 它不是使用 Activiti 的唯一方式. 因为 Activiti 只是一个 jar 包, 可以被引入到任何 Java 环境中, 无论是 Java Swing 项目还是在 Tomcat, JBoss, WebSphere 等容器中. 你也完全可以将 Activiti 当作一个独立的服务来部署. 总之, 只要 Java 支持, Activiti 就可以!
## Activiti 数据库设置 
按前文所说, Activiti UI 默认使用内存型数据 H2. 如果你想使用单独的 H2 或者其他类型数据库, 你可以去 Activiti UI 应用, 找到  WEB-INF/classes/META-INF/activiti-app 修改 activiti.properties 文件
## 引用 Activiti jar 和它的依赖
我们推荐使用 [Maven](http://maven.apache.org/)(或 [Ivy](http://ant.apache.org/ivy/))来引用 Activiti 包和它的依赖, 这样可以简化你我双方的依赖管理工作. 你可以按照 http://www.activiti.org/community.html#maven.repository 上的说明来引用你需要的 jar 包  
或者, 如果你不想使用 Maven,你也可以手动将这些 jar 引入到你的项目里. 你下载的 Activiti 压缩包中有一个 libs 文件夹, 里面包含了所有的 Activiti jar(也包含他们的源码包). 但是, 他们的依赖包却不在其中. Activiti 引擎所需要的依赖包如下(使用 mvn dependency:tree 命令得出的):
```
org.activiti:activiti-engine:jar:6.x
+- org.activiti:activiti-bpmn-converter:jar:6.x:compile
|  \- org.activiti:activiti-bpmn-model:jar:6.x:compile
|     +- com.fasterxml.jackson.core:jackson-core:jar:2.2.3:compile
|     \- com.fasterxml.jackson.core:jackson-databind:jar:2.2.3:compile
|        \- com.fasterxml.jackson.core:jackson-annotations:jar:2.2.3:compile
+- org.activiti:activiti-process-validation:jar:6.x:compile
+- org.activiti:activiti-image-generator:jar:6.x:compile
+- org.apache.commons:commons-email:jar:1.2:compile
|  +- javax.mail:mail:jar:1.4.1:compile
|  \- javax.activation:activation:jar:1.1:compile
+- org.apache.commons:commons-lang3:jar:3.3.2:compile
+- org.mybatis:mybatis:jar:3.3.0:compile
+- org.springframework:spring-beans:jar:4.1.6.RELEASE:compile
|  \- org.springframework:spring-core:jar:4.1.6.RELEASE:compile
+- joda-time:joda-time:jar:2.6:compile
+- org.slf4j:slf4j-api:jar:1.7.6:compile
+- org.slf4j:jcl-over-slf4j:jar:1.7.6:compile
```
注意: mail.jar 只在你使用[邮件任务](#bpmnEmailTask//TODO)时,才必须要引入
你可以在对应的 [Activiti 源码包](https://github.com/Activiti/Activiti)的对应模块中执行 *mvn dependency:copy-dependencies* 来轻松下载依赖  
## 下一步
多操作几遍 [Activiti UI](#activitiUI) web 应用你就能很快熟悉 Activiti 的概念和功能. 但是, Activiti 的核心目的当然是让你在自己的项目中使用到强大的 BPM 和工作流功能. 接下来的章节将会帮你熟悉如何在你的项目中使用 Activiti 编程:
- [配置章节](https://www.activiti.org/userguide/index.html#configuration) 将会教你如何设置 Activiti, 如何初始化一个 ***ProcessEngine*** 实例. 这个类是你了解 Activiti 所有引擎功能的一个核心切入点.
- [API 章节](https://www.activiti.org/userguide/index.html#chapterApi) 将指导你了解 Activiti
 API 后面的服务. 这些服务简单有力地支撑着 Activiti 引擎, 而且可以在任何 Java 环境中使用
- BPMN 2.0 是 Activiti 引擎所使用的标准, 你是否有兴趣深入了解它呢? 如果是的话, 你可以阅读 [BPMN 2.0 章节](https://www.activiti.org/userguide/index.html#bpmn20)