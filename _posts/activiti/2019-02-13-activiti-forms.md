---
layout: post
title: Activiti 6.0 用户文档 - 第九章：表单
categories: activiti
description: 流程引擎 Activiti 第九章：表单
keywords: 流程引擎, activiti
---
# 表单
Activiti 提供了一种方便且灵活的方式来为业务流程的手动步骤添加表单。我们支持两种表单使用策略：内置表单属性渲染，外部表单属性渲染。  
## 表单属性
与业务流程相关的所有信息都包含在流程变量或它们的引用中。Activiti支持将复杂的Java对象存储为流程对象，比如**Serializable**对象，JPA实体或者String类型的整个XML文档。  
启动流程和完成用户任务是人们参与流程的地方。使用一些UI技术来呈现表单，然后通过它与人们交互。为了简化多种UI技术，流程定义里有一种逻辑，可以将流程变量中复杂的Java对象转化成 **properties**类型的 **Map<String，String>**。
使用Activiti API 中暴露属性信息的方法，任何一种UI技术都可以在这些属性之上构建表单。这些属性可以为流程变量提供专用(且更局限)的视图。比如，**FormData**返回值中提供了展示表单所需要的属性。  
```java
StartFormData FormService.getStartFormData(String processDefinitionId)
```
或者
```java
TaskFormdata FormService.getTaskFormData(String taskId)
```
默认的，内置的form引擎，*会查看*属性和流程变量。因此，如果任务表单的属性与流程变量是一一对应的，就无需声明它们。比如，下面的声明：
```xml
<startEvent id="start" />
```
当执行到达 startEvent 时，所有的流程变量都是可以看到的，但是：
```java
formService.getStartFormData(String processDefinitionId).getFormProperties()
```
会返回一个空值，因为没有定义一个特定的mapping。  
在上面的例子中，所有提交的属性都会被存储为流程变量。这就意味着，简单地在表单中增加一个input输入框，一个新的变量就会被保存。  
属性通过流程变量传递，但是他们没有必须要作为流程变量保存。比如，一个流程变量可能是一个JPA实体中的类Address。而且UI技术中使用的表单属性**StreetName**可以与表达式 **#{address.street}**链接。  
类似地，用户需要在表单中提交的属性可以作为流程变量或者带有URL值表达式(比如 **#{address.street}**)的某个流程变量的嵌套属性保存。  
类似地，提交的属性默认以流程变量的形式存储，除非额外声明**formProperty**  
处理表单属性和流程变量还要考虑类型转换的问题。  
比如：
```xml
<userTask id="task">
  <extensionElements>
    <activiti:formProperty id="room" />
    <activiti:formProperty id="duration" type="long"/>
    <activiti:formProperty id="speaker" variable="SpeakerName" writable="false" />
    <activiti:formProperty id="street" expression="#{address.street}" required="true" />
  </extensionElements>
</userTask>
```
