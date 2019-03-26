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
处理表单属性和流程变量之间的问题还要考虑类型转换。  
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
- 表单属性 **room** 将映射成一个 String 类型的流程变量
- 表单属性 **duration** 将作映射成一个 java.lang.Lang 类型的的流程变量
- 表单属性 **speaker** 将映射成为流程变量里的 **speakerName**。它只能在 TaskFormData 对象中访问。如果要提交speaker属性，系统会抛出ActivitiException。类似地，如果属性 **readable="false"**，该属性可以从FormData中被排除，但是仍可以被提交。
- 表单属性 **street** 将映射成流程变量 **address**的一个String类型属性：**street**，而且该表单属性设置了**required="true"**，如果该字段为空，提交的时候就会报异常。  

从一些方法(**StartFormData FormService.getStartFormData(String processDefinitionId)**，**FormService.getTaskFormData(String taskId)**)返回的 FormData中，你还能找到一些类型信息。  
我们支持下面这些表单属性类型：
- string (org.activiti.engine.impl.form.StringFormType)
- long (org.activiti.engine.impl.form.LongFormType)
- enum (org.activiti.engine.impl.form.EnumFormType)
- date (org.activiti.engine.impl.form.DateFormType)
- boolean (org.activiti.engine.impl.form.BooleanFormType)
当每个表单属性声明以后，通过 **List<FormProperty> formService.getStartFormData(String processDefinitionId).getFormProperties()** 和 **List<FormProperty> formService.getTaskFormData(String taskId).getFormProperties()** 我们能拿到 **FormProperty** 信息。  
```java
public interface FormProperty {
  /** the key used to submit the property in {@link FormService#submitStartFormData(String, java.util.Map)}
   * or {@link FormService#submitTaskFormData(String, java.util.Map)} */
  String getId();
  /** the display label */
  String getName();
  /** one of the types defined in this interface like e.g. {@link #TYPE_STRING} */
  FormType getType();
  /** optional value that should be used to display in this property */
  String getValue();
  /** is this property read to be displayed in the form and made accessible with the methods
   * {@link FormService#getStartFormData(String)} and {@link FormService#getTaskFormData(String)}. */
  boolean isReadable();
  /** is this property expected when a user submits the form? */
  boolean isWritable();
  /** is this property a required input field */
  boolean isRequired();
}
```
比如
```xml
<startEvent id="start">
  <extensionElements>
    <activiti:formProperty id="speaker"
      name="Speaker"
      variable="SpeakerName"
      type="string" />

    <activiti:formProperty id="start"
      type="date"
      datePattern="dd-MMM-yyyy" />

    <activiti:formProperty id="direction" type="enum">
      <activiti:value id="left" name="Go Left" />
      <activiti:value id="right" name="Go Right" />
      <activiti:value id="up" name="Go Up" />
      <activiti:value id="down" name="Go Down" />
    </activiti:formProperty>

  </extensionElements>
</startEvent>
```
所有这些信息都可以通过API拿到。类型名称可以使用**formProperty.getType().getName()**获取，甚至日期类型也可以通过 **formProperty.getType().getInformation("datePattern")** 获得，枚举类型可以使用 **formProperty.getType().getInformation("values")** 拿到。  
Actviti Explorer 支持表单表达式，可以根据定义渲染表单。下面的XML代码片段：
```xml
<startEvent>
  <extensionElements>
    <activiti:formProperty id="numberOfDays" name="Number of days" value="${numberOfDays}" type="long" required="true"/>
    <activiti:formProperty id="startDate" name="First day of holiday (dd-MM-yyy)" value="${startDate}" datePattern="dd-MM-yyyy hh:mm" type="date" required="true" />
    <activiti:formProperty id="vacationMotivation" name="Motivation" value="${vacationMotivation}" type="string" />
  </extensionElements>
</userTask>
```
在 Activiti Explorer 中被渲染成一个流程启动表单：  
![表单浏览器图](/images/activiti/forms.explorer.png)  
## 表单渲染扩展
你还可以通过API来使用外部的任务表单渲染器，下面介绍使用该API的步骤。  
实际上，渲染表单所需要的所有数据都是这两个方法中的一个组装的：**StartFormData FormService.getStartFormData(String processDefinitionId)**和**TaskFormdata FormService.getTaskFormData(String taskId)**。  
使用 **ProcessInstance FormService.submitStartFormData(String processDefinitionId, Map<String,String> properties)** 和 **void FormService.submitTaskFormData(String taskId, Map<String,String> properties)**可以提交表单属性。  
你可以参阅[表单属性](https://www.activiti.org/userguide/index.html#formProperties)可以知道表单属性是怎么映射成流程变量的。  
如果你想使用流程来保存表单模板资源的版本信息，可以将它们放在你发布的业务包中。在发布中，它就是一种资源，你还能使用**String ProcessDefinition.getDeploymentId()**和**InputStream RepositoryService.getResourceAsStream(String deploymentId, String resourceName);**获取到该资源。你的这些模板定义文件可以在你自己的项目中渲染/展示。  
除了访问任务表单之外，你还可以使用访问发布资源的特性做一些别的事情。  
通过API **String FormService.getStartFormData(String processDefinitionId).getFormKey()**和**String FormService.getStartFormData(String processDefinitionId).getFormKey()**可以暴露 **<userTask activiti:formKey="…​"** 信息。在发布中，你可以使用这个特性存储模板文件的整个名称(比如 **org/activiti/example/form/my-custom-form.xml**)，当然了，这个不是必须的。比如，你还可以在表单属性中保存通用主键，然后通过算法或者转换来拿到实际使用的模板。当你想为不同的UI框架展示不同的表单的时候，这就很方便了。比如，你可以为PC用户展示正常屏幕大小的表单，为移动用户展示小屏幕的表单，甚至可以是用于 IM 的表单或者邮件中的表单。