---
layout: post
title: Activiti 6.0 用户文档 - 第一章：介绍
categories: activiti
description: 流程引擎 Activiti 第一章：介绍
keywords: 流程引擎, activiti
---
# 介绍

## 开源许可
Activiti 根据 [Apache V2](http://www.apache.org/licenses/LICENSE-2.0.html) 开源许可分发
## 下载
http://activiti.org/download.html
## 源码
Activiti 以 jar 的方式开源了绝大多数的源码.  这些源码都放在 https://github.com/Activiti/Activiti 上面
## 软件要求
### JDK7 以上
Activiti 运行在 JDK7 或者更高的版本上面. 用户可以去[Java 官网](http://www.oracle.com/technetwork/java/javase/downloads/index.html)下载 JDK. 下载页面还提供了安装说明. 你可以在命令行运行 *java -version* 来确认是否安装成功. 如果安装正确的话, 运行结果应该是你安装的 jdk 版本 
### IDE 
你可以选用自己喜欢的 IDE 来完成基于 Activiti 的开发. 如果你想使用 Activiti 设计器, 那就需要使用 Eclipse Kepler 或 Luna 版本. 你可以去 [Eclipse 下载页面](http://www.eclipse.org/downloads/) 下载你想要的 eclipse 版本. 将下载报解压后, 直接在 eclipse 目录下运行 eclipse 可执行文件. 本文档后面有一章单独介绍 [如何在 eclipse 安装 Activiti设计插件](#eclipseDesignerInstallation//TODO)
## 报告问题
每一个有追求的程序员都需要阅读这篇文章: [提问之道](http://www.catb.org/~esr/faqs/smart-questions.html)  
等你读完了, 你就可以在[用户论坛](http://forums.activiti.org/en/viewforum.php?f=3)在提交你的意见和建议 你也可以在我们的[JIRA](https://activiti.atlassian.net/)上提交 issue ,以便追踪.
> 虽然 Activiti 托管在 GitHub, 但是你不能在 GitHub 的 issue 系统上提交 issue,. 如果你想提交 issue, 请使用我们的 [JIRA](https://activiti.atlassian.net/)

## 实验性功能
一些标记为 **[EXPERIMENTAL]** 的章节表示该功能不是稳定版本  
路径中有 *.impl.* 的包,里面的类都是内部实现类, 将来都可能被修改的. 但是, 本文档中提及的配置类可以作为稳定版本, 我们会提供长期支持. 
## 内部实现类
在 jar 包中, 所有在 *.impl.* 包(比如: org.activiti.engine.impl.db)下的类都是一些内部实现类, 里面的类和接口可能在版本升级中被修改
