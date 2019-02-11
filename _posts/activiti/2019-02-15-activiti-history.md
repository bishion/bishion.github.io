---
layout: post
title: Activiti 6.0 用户文档 - 第十一章：历史
categories: activiti
description: 流程引擎 Activiti 第十一章：历史
keywords: 流程引擎, activiti
---
# 历史
在API 中，我们可以查询到5种History实体。HistoryService暴露这对应的5个方法： **createHistoricProcessInstanceQuery(), createHistoricVariableInstanceQuery(), createHistoricActivityInstanceQuery(), createHistoricDetailQuery() 和 createHistoricTaskInstanceQuery().**  
下面是几个例子，展示查询历史数据的API，API的详细说明可以在 [javadoc](http://activiti.org/javadocs/index.html)中找到，就在 **org.activiti.engine.history**包里面。
### HistoricProcessInstanceQuery
查询流程定义为 *XXX* 的10个耗时最长的历史流程实例 **HistoricProcessInstances**
```java
historyService.createHistoricProcessInstanceQuery()
  .finished()
  .processDefinitionId("XXX")
  .orderByProcessInstanceDuration().desc()
  .listPage(0, 10);
```
### HistoricVariableInstanceQuery
查询所有流程实例ID为 *XXX* 的 **HistoricVariableInstances**，并按照变量名称倒排序。  
```java
historyService.createHistoricVariableInstanceQuery()
  .processInstanceId("XXX")
  .orderByVariableName.desc()
  .list();
```
### HistoricActivityInstanceQuery
查询已经完成的流程定义id为 *XXX* ，类型为 *servictTask* 的最后一个**HistoricActivityInstance**
```java
historyService.createHistoricActivityInstanceQuery()
  .activityType("serviceTask")
  .processDefinitionId("XXX")
  .finished()
  .orderByHistoricActivityInstanceEndTime().desc()
  .listPage(0, 1);
```
### HistoricDetailQuery
