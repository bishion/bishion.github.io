---
layout: post
title: Activiti 6.0 用户文档 - 第六章：发布
categories: activiti
description: 流程引擎 Activiti 第六章：发布
keywords: 流程引擎, activiti, deployment
---
# 发布
## 业务包
多个流程需要需要打包成一个业务包才能被发布，它是Activiti引擎的部署单元。一个业务包就是一个zip文件。它里面可以包含：BPMN2.0流程，任务表单，规则和其他文件类型。通常，业务包包含一组命名资源。  
当一个业务包发布的时候，系统就会扫描以 **.bpmn20.xml**或 **.bpmn**结尾的BPMN文件。每个文件都会被解析，里面可能包含多个流程定义。
> 业务包中的Java类**不会被添加到类路径**中。为了正常运行流程，流程定义使用到的所有自定义类(比如Java服务任务或者监听器)都应该在放在Activiti Engine类路径下  

### 通过程序发布
可以用下面这个方式发布一个 zip 业务包：
```java
String barFileName = "path/to/process-one.bar";
ZipInputStream inputStream = new ZipInputStream(new FileInputStream(barFileName));

repositoryService.createDeployment()
    .name("process-one.bar")
    .addZipInputStream(inputStream)
    .deploy();
```
我们也可以通过独立的资源文件来创建一个发布。详细信息可以参考javadoc。
## 外部资源
流程定义都是保存在Activiti的数据库中。当使用Spring Task、执行监听器或Spring bean的时侯，这些流程定义可以通过Activiti配置文件指向委托类。这些类和Spring配置文件可以执行流程定义。
### Java 类
流程中用到的所有自定义的的类(比如服务任务或事件监听器、任务监听器等等)必须在引擎实例启动的时候出现在流程引擎的类路径中。  
但是，如果你在发布业务包的时候，系统不强制你将这些类放在类路径中。这意味着当你使用Ant发布一个业务包时，委托类不一定要放在类路径中。  
当你启动我们的demo并且要往里面添加你的自定义类的时候，你需要将这些类的jar包放到 activiti-explore 或 activiti-rest 应用下的lib文件夹中。同时也不要忘了将这些自定义类加到你的依赖中。或者，你可以将这些依赖放到Tomcat的安装路径里：**${tomcat.home}/lib**  
### 在流程中使用Spring bean
当表达式或者脚本使用Spring bean的时候，引擎在执行流程定义的时候需要有这些bean的访问权限。如果你是自己创建的web项目，你需要按照[spring集成](https://www.activiti.org/userguide/index.html#springintegration)这一章介绍的方式来配置你的流程引擎。但是要时刻注意，如果你用到了Activiti rest项目的上下文，你还需要修改它。修改方式是：使用一个包含你的Spring 上下文配置的 **activiti.cfg.xml**来替换掉 **activiti-rest/lib/activiti-cfg.jar**中的**activiti.cfg.xml**。  
### 创建一个单独的app
不用去管是不是流程用到的委托类都放在了类路径下而且都是用了正确的Spring配置，你更应该考虑的是将Activiti rest应用放在自己的webapp下然后确认只有一个 **ProcessEngine**
## 流程定义的版本控制
BPMN中并没有版本控制的概念。但是它却是个好东西，因为一个可执行的BPMN流程文件可能作为开发项目的一部分接受版本控制系统(比如Subversion、Git或Mercurial)的管理。在发布过程中，Activiti会在将**ProcessEngine**保存到数据库之前给它分配一个版本。  
对于业务包中的每个流程定义，在初始化**key**，**version**，**name**和**id**这些属性的时候，会执行下面这几步：
- XML 文件中的 **id** 属性会用来给流程定义中的**key**赋值
- XML 文件中的 **name** 属性会用来给流程定义中的**name**赋值。如果name属性没有指定，就用属性id代替
- 当某个 key 的流程第一次发布后，它的版本就是 1。当相同key的流程定义再发布的时候，版本就会加1。key 用来区分流程定义
- id的属性设置为 {processDefinitionKey}:{processDefinitionVersion}:{generated-id}，这里的 {generated-id}是添加的唯一编号，来保证集群环境中缓存的流程Id的唯一性

下面就是一个流程的示例：
```xml
<definitions id="myDefinitions" >
  <process id="myProcess" name="My important process" >
    ...
```
当发布这个流程定义的时候，它在数据库的样子是这样的：
|id|key|name|version|
|--|---|----|-------|
|myProcess:1:676|myProcess|My important process|1|

假设我们将这个流程的更新版本(比如修改了一些用户任务，**id**属性不变)再发布一次，那么这个流程定义在数据库中的数据就是这样的：
|id|key|name|version|
|--|---|----|-------|
|myProcess:1:676|myProcess|My important process|1|
|myProcess:1:870|myProcess|My important process|2|
当**runtimeService.startProcessInstanceByKey("myProcess")**被调用时，它将会使用版本号为**2**的流程定义，因为它是最新的。  
我们可以按照下面的方式再创建一个流程，发布之后，第三条数据就会被添加到表中。
```xml
<definitions id="myNewDefinitions" >
  <process id="myNewProcess" name="My important process" >
    ...
```
表中的数据是这样的：
|id|key|name|version|
|--|---|----|-------|
|myProcess:1:676|myProcess|My important process|1|
|myProcess:1:870|myProcess|My important process|2|
|myNewProcess:1:1033|myNewProcess|My important process|1|
注意新流程的key跟第一个流程已经不一样了。虽然它们的name相同(当然了，我们也可以修改它)，但是Activiti 只通过 **id**来区分流程，所以发布时，它的版本号是1。
## 提供一个流程图
可以给发布添加一个流程图。这个图片会被Activiti库保存，我们可以通过API访问，可以通过它在Activiti Explorer中展示流程。  
假设我们类路径下有一个流程：**org/activiti/expenseProcess.bpmn20.xml**，它里面有个流程叫*expense*。流程图的命名规则就像下面这样(按照如下顺序)：
- 如果发布中有一个图片，名称由BPMN2.0 XML名称加上流程key加上图片后缀构成的，那么就直接使用这个图片。在本示例中，它的名称为，**org/activiti/expenseProcess.expense.png**(或者.jpg/gif)。如果你在BPMN2.0 XML 文件中定义了多个图像，那么你就会发现这个方法最好用：每个图片都在自己的文件名中包含流程key
- 如果该图片不存在，那么就会去查找跟BPMN2.0 XML 同名的图像。在本示例中，就会去查找**org/activiti/expenseProcess.png**。注意，这意味着同一个BPMN2.0文件中的**每一个**流程定义使用共享同一个流程图。如果BPMN2.0XML文件中只有一个流程定义，这样处理也没什么问题。

使用代码做一个发布的例子如下：
```java
repositoryService.createDeployment()
  .name("expense-process.bar")
  .addClasspathResource("org/activiti/expenseProcess.bpmn20.xml")
  .addClasspathResource("org/activiti/expenseProcess.png")
  .deploy();
```
然后可以通过API来检索图片：
```java
ProcessDefinition processDefinition = repositoryService.createProcessDefinitionQuery()
  .processDefinitionKey("expense")
  .singleResult();

String diagramResourceName = processDefinition.getDiagramResourceName();
InputStream imageStream = repositoryService.getResourceAsStream(
    processDefinition.getDeploymentId(), diagramResourceName);
```
## 生成一个流程图
像[上文](https://www.activiti.org/userguide/index.html#providingProcessDiagram)说的那样，如果发布中没有提供图片，而流程定义中包含*生成图片*配置，那么Activiti就会生成一个流程图。  
可以使用部署时[指定的方式](https://www.activiti.org/userguide/index.html#providingProcessDiagram)来检索该流程图。  
![生成流程图](/images/activiti/deployment.image.generation.png)  
处于某种原因，没必要或者不想在发布过程中生成流程图，可以在流程引擎配置中将**isCreateDiagramOnDeploy**属性设置成false：
```xml
<property name="createDiagramOnDeploy" value="false" />
```
这样就不会生成流程图了。
## 分类
发布和流程定义中都可以设置分类。在BPMN文件中，流程定义的类别的可以由相关属性初始化：**<definitions …​ targetNamespace="yourCategory" …​**  
使用API中指定发布的类别的方式如下：
```java
repositoryService
    .createDeployment()
    .category("yourCategory")
    ...
    .deploy();
```