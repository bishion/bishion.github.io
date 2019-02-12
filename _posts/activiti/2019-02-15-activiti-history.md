---
layout: post
title: Activiti 6.0 用户文档 - 第十一章：历史
categories: activiti
description: 流程引擎 Activiti 第十一章：历史
keywords: 流程引擎, activiti
---
# 历史
在API 中，我们可以查询到5种History实体。HistoryService暴露了对应的5个方法： **createHistoricProcessInstanceQuery(), createHistoricVariableInstanceQuery(), createHistoricActivityInstanceQuery(), createHistoricDetailQuery() 和 createHistoricTaskInstanceQuery().**  
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
下面这个例子演示了如何查询 id 为 123 的流程中所有更新过的变量，该方法只会返回 **HistoricVariableUpdates**。注意，一个变量名有多个 **HistoricVariableUpdates** 实例是很有可能的，因为它在流程中每修改一次就有产生一个实体。你可以使用 **orderByTime**（变量修改时间）或者**orderByVariableRevision**（变量修改版本）来对这些实体排序。
```java
historyService.createHistoricDetailQuery()
  .variableUpdates()
  .processInstanceId("123")
  .orderByVariableName().asc()
  .list()
```
下面这个例子演示了如何查询id为“123”的流程启动时或者在任务中提交的所有表单属性，该方法只返回 **HistoricFormProperties**。
```java
historyService.createHistoricDetailQuery()
  .formProperties()
  .processInstanceId("123")
  .orderByVariableName().asc()
  .list()
```
最后一个例子演示了如何获取任务id为“123”的所有更新过的变量。该方法返回任务中设置过的所有变量（任务的本地变量） **HistoricVariableUpdates**，**而不是**流程实例中的。
```java
historyService.createHistoricDetailQuery()
  .variableUpdates()
  .taskId("123")
  .orderByVariableName().asc()
  .list()
```
在 **TaskListener**中使用 **TaskService** 或者 **DelegateTask**可以设置任务的本地变量。
```java
taskService.setVariableLocal("123", "myVariable", "Variable value");
```
```java
public void notify(DelegateTask delegateTask) {
  delegateTask.setVariableLocal("myVariable", "Variable value");
}
```
### HistoricTaskInstanceQuery
查询已经完成的耗时最久的前十个 **HistoricTaskInstance**
```java
historyService.createHistoricTaskInstanceQuery()
  .finished()
  .orderByHistoricTaskInstanceDuration().desc()
  .listPage(0, 10);
```
查询最后一个审批人是 *kermit*的，删除原因包括“invalid”的前十个**HistoricTaskInstance**
```java
historyService.createHistoricTaskInstanceQuery()
  .finished()
  .taskDeleteReasonLike("%invalid%")
  .taskAssignee("kermit")
  .listPage(0, 10);
```
## 历史配置
历史的级别可以可以通过程序配置：*org.activiti.engine.impl.history.HistoryLevel（或者在5.11版本之前，使用 **ProcessEngineConfiguration**里面的常量 HISTORY ）*
```java
ProcessEngine processEngine = ProcessEngineConfiguration
  .createProcessEngineConfigurationFromResourceDefault()
  .setHistory(HistoryLevel.AUDIT.getKey())
  .buildProcessEngine();
```
该级别可以在 activiti.cfg.xml或者 spring-context 中配置：
```xml
<bean id="processEngineConfiguration" class="org.activiti.engine.impl.cfg.StandaloneInMemProcessEngineConfiguration">
  <property name="history" value="audit" />
  ...
</bean>
```
可以设置为下面几个值：
- none：忽略是所有的历史归档。对于流程执行来说，它的效率是最高的，但是历史信息都被丢了
- activity：所有的实例流程和活动的实例都会被归档。在流程实例结束时，顶级流程实例变量的最新值将复制到历史变量实例中。不归档具体信息
- audit：默认值。它保存所有的流程实例，活动实例，维持变量值同步的信息和提交的表单属性，以便通过表单进行的所有用户交互都可以被跟踪而且被审计到。
- full：这个是历史数据归档的最高等级，性能也是最低的。在当前级别下，除了**audit**存储的那些数据之外，系统还保存了可能出现的流程变量更新的详细数据

**Activiti 5.11之前，历史数据级别是存储在数据库中的（ACT_GE_PROPERTY表，属性名是 historyLevel）。从5.11版本以后，数据库中的该值就被忽略/删除，不再使用。现在历史数据可以在引擎的两次启动之间修改，而不会抛出异常。**
## 用与审计的历史数据
当[配置](https://www.activiti.org/userguide/#historyConfig)了 **audit**以上的级别时，通过 **FormService.submitStartFormData(String processDefinitionId, Map<String, String> properties) 和 FormService.submitTaskFormData(String taskId, Map<String, String> properties)**提交的所有属性都会被存储。  
表单属性可以通过下面的查询API 检索：
```
historyService
      .createHistoricDetailQuery()
      .formProperties()
      ...
      .list();
```
在上面这种情况下，只有**HistoricFormProperty**类型的历史详情被返回。  
如果你在使用 **IdentityService.setAuthenticatedUserId(String)**调用提交方法之前，设置了认证用户，那么该提交表单的认证用户将有权限在历史数据中使用**HistoricProcessInstance.getStartUserId()**访问启动表单，还有权限使用 **HistoricActivityInstance.getAssignee()** 访问任务表单