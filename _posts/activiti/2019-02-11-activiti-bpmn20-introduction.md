---
layout: post
title: Activiti 6.0 用户文档 - 第七章：BPMN 2.0 介绍
categories: activiti
description: 流程引擎 Activiti 第七章：BPMN 2.0 介绍
keywords: 流程引擎, activiti, bpmn
---
# BPMN2.0 介绍
## 什么是BPMN ?
参考我们的[BPMN2.0答疑入口](http://activiti.org/faq.html#WhatIsBpmn20)
## 定义流程
> 本章首先假设你使用 Eclipse IDE来创建和编辑文件。但是，只有极少数的内容是针对Eclipse的。你可以使用喜欢的任何工具来创建BPMN2.0的xml文件。

创建一个新的XML文件(*右键点击任意项目，选择 New→Other→XML-XML File*)，然后给它起个名字。为了防止发布的时候，引擎无法识别这个文件，请确保这个文件名是**以.bpmn20.xml或.bpmn结尾**。  
![流程定义图](/images/activiti/new.bpmn.procdef.png)  
BPMN2.0模式规定，**definitions**是根元素，它里面可以定义多个流程定义(不过为了简化后续维护发布流程的工作，我们建议每个文件中只定义一个流程)。一个空的流程定义就像下面展示的那样。注意，最小化的**definitions**元素只需要声明**xmlns**和**targetNamespace**。其中 **targetNamespace**可以设置为任何值，它在给流程定义分类的时候非常有用。
```xml
<definitions
  xmlns="http://www.omg.org/spec/BPMN/20100524/MODEL"
  xmlns:activiti="http://activiti.org/bpmn"
  targetNamespace="Examples">

  <process id="myProcess" name="My First Process">
    ..
  </process>

</definitions>
```
你还可以(不强制)添加线上的给BPMN2.0 XML schema位置，以替代Eclipse中的XML配置。
```properties
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
xsi:schemaLocation="http://www.omg.org/spec/BPMN/20100524/MODEL
                    http://www.omg.org/spec/BPMN/2.0/20100501/BPMN20.xsd
```
**process**元素有两个属性：
- id：这个属性是**必须**的，它对应着 Activiti **ProcessDefinition**对象的 **key**属性。**RuntimeService**中的**startProcessInstanceByKey**还通过这个**id**启动一个流程定义的流程实例。这个方法每次都是取流程定义的**最新发布版本**
```java
ProcessInstance processInstance = runtimeService.startProcessInstanceByKey("myProcess");
```
- 这里千万要注意，它跟调用**startProcessInstanceById**方法不一样。这个方法的参数是一个由Activiti发布的时候生成的String类型的id，可以通过调用**processDefinition.getId()**来获取。生成的id的格式是：**key:version**，长度**约束在64个字符**。如果你碰到一个**ActivitiException**显示说生成的ID太长，请减小流程中*key*的长度
- name：该属性是**可选的**，它映射为 **ProcessEngine**的*name*属性。引擎自己并不使用这个字段，举例来说，用户接口中有了它可以更友好地展示信息  

[[十分钟教程]]
## 开始：一个十分钟教程
本节，我们将通过一个(十分简单的)业务流程场景，来介绍一些基本的Activiti 概念和Activiti API。
### 准备工作
这个教程假设你已经运行[Activiti demo程序](https://www.activiti.org/userguide/index.html#demo.setup.one.minute.version)，而且使用的是一个独立的 H2服务。编辑 **db.properties**文件，设定好**jdbc.url=jdbc:h2:tcp://localhost/activiti**，然后根据[H2文档](http://www.h2database.com/html/tutorial.html#using_server)将H2服务起起来。
### 目标
本教程的目标是了解Activiti和一些BPMN2.0的基本概念。教程会引导大家编写一个简单的Java SE程序，这个程序可以发布一个流程定义，然后通过Activiti 的API跟这个流程交互。我们还会接触到一些Activiti的一些周边工具。当然了，你学习的这些东西也可以用来围绕你的业务流程构建你自己的web项目。
### 用例
这个用例很简单：我们有一家公司，就叫它 BPMNCorp 好了。在BPMNCorp，会计部门每个月都要为股东编写财务报告。当报告写完之后，需要一位高管审核之后才能发给所有股东。
### 流程图
上面说的业务流程其实是可以使用[Activiti开发工具](https://www.activiti.org/userguide/index.html#activitiDesigner)画出来的。但是对于本教程，我们会手动编写XML文件，通过这种方式我们才会学到更多东西。我们的流程使用BPMN2.0图形化描述如下：  
![财务流程图](/images/activiti/financial.report.example.diagram.png)  
我们看到的是一个[无开始的事件](https://www.activiti.org/userguide/index.html#bpmnNoneStartEvent)(左边的圆圈)，接着是两个[用户任务](https://www.activiti.org/userguide/index.html#bpmnUserTask)：'Write monthly financial report' 和 'Verify monthly financial report',然后以一个[无结尾事件](https://www.activiti.org/userguide/index.html#bpmnNoneEndEvent)(右边的圆圈)。  
## XML 描述
这个业务流程的XML版本(FinancialReportProcess.bpmn20.xml)如下面所示。我们很容易就能分辨出来我们业务流程中的主要节点(可以点击下文的链接来查看BPMN2.0章节中对应的详细介绍)
- [(无)启动事件](https://www.activiti.org/userguide/index.html#bpmnNoneStartEvent)表示流程的*入口*
- [用户任务]表示流程中人类的任务。注意，第一个任务是分配给*会计部*，而第二个任务是分配给*管理部*。有关如何将用户和用户组分配给用户任务的详细信息，请参阅[用户任务分配](https://www.activiti.org/userguide/index.html#bpmnUserTaskAssignment)部分。
- 当到达[无结束事件](https://www.activiti.org/userguide/index.html#bpmnNoneEndEvent)时，该流程就结束了
- 元素之间通过[序列流](https://www.activiti.org/userguide/index.html#bpmnSequenceFlow)连接。这些序列流都有**源**和**目标**，来定义序列流的方向。
```xml
<definitions id="definitions"
  targetNamespace="http://activiti.org/bpmn20"
  xmlns:activiti="http://activiti.org/bpmn"
  xmlns="http://www.omg.org/spec/BPMN/20100524/MODEL">

	<process id="financialReport" name="Monthly financial report reminder process">

	  <startEvent id="theStart" />

	  <sequenceFlow id="flow1" sourceRef="theStart" targetRef="writeReportTask" />

	  <userTask id="writeReportTask" name="Write monthly financial report" >
	    <documentation>
	      Write monthly financial report for publication to shareholders.
	    </documentation>
	    <potentialOwner>
	      <resourceAssignmentExpression>
	        <formalExpression>accountancy</formalExpression>
	      </resourceAssignmentExpression>
	    </potentialOwner>
	  </userTask>

	  <sequenceFlow id="flow2" sourceRef="writeReportTask" targetRef="verifyReportTask" />

	  <userTask id="verifyReportTask" name="Verify monthly financial report" >
	    <documentation>
	      Verify monthly financial report composed by the accountancy department.
	      This financial report is going to be sent to all the company shareholders.
	    </documentation>
	    <potentialOwner>
	      <resourceAssignmentExpression>
	        <formalExpression>management</formalExpression>
	      </resourceAssignmentExpression>
	    </potentialOwner>
	  </userTask>

	  <sequenceFlow id="flow3" sourceRef="verifyReportTask" targetRef="theEnd" />

	  <endEvent id="theEnd" />

	</process>

</definitions>
```
### 启动一个流程实例
我们现在为业务流程创建了一个**流程定义**，通过它我们可以创建**流程实例**。在这个案例中，一个流程表示某个月份的财务报表的创建和审批。所有的流程实例共享相同的流程定义。  
我们必须首先**发布**一个流程定义，然后才能给它创建流程实例。发布一个流程定义表示两件事：
- 该流程定义要被保存到Activiti配置的数据库中。这样，通过发布我们的业务流程，流程引擎重启了之后也能顺利找到我们之前的流程定义。  
- BPMN2.0的流程文件必须被解析成一个内存对象之后，Activiti API 才能操作它  

有关发布的更多信息，请参阅[发布章节](https://www.activiti.org/userguide/index.html#chDeployment)  
就像发布章节介绍的那样，发布可以有很多种实现，下面的API方式时其中一种。注意，所有与Activiti的交互都是通过它的*service*。
```java
Deployment deployment = repositoryService.createDeployment()
  .addClasspathResource("FinancialReportProcess.bpmn20.xml")
  .deploy();
```
现在你可以使用我们在流程定义(参考XML中的process元素)中使用的**id**来启动一个流程实例。注意这个**id**在Activiti的术语中叫做**key**。
```java
ProcessInstance processInstance = runtimeService.startProcessInstanceByKey("financialReport");
```
这行代码将会创建一个流程实例，而且直接穿过启动事件。启动事件之后，它跟着所有的顺序流(本例中只有一个)流出，然后到达第一个任务(*编写月度财务报告*)。Activiti引擎此时会在数据库中保存一个任务。此时，分配给该任务的用户或用户组将会被解析并存储在数据库中。一定要注意，Activiti引擎会一路往下执行直到遇到一个*等待状态*，比如用户任务。在这个等待状态中，流程实例的当前状态会被保存在数据库中。在用户决定完成这个任务之前，它一直保持这个状态。那时，引擎会继续一路执行，直到它遇到一个新的等待状态或者流程结束。当引擎在这期间重启或者崩溃时，流程的状态还安全地存储在数据库中。  
当任务被创建时，**startProcessInstanceByKey**方法会返回，因为用户任务活动是*等待状态*。在本例中，任务被分配给一个用户组，这意味着组里的每一个成员都是该任务的**候选者**  
现在我们可以将这些都放到一起，然后创建一个简单的Java程序。新建一个新的Eclipse项目，将Activiti 的包和它们的依赖(可以在Activiti项目的*libs*目录下)添加到项目的类路径下。在我们调用Activiti服务之前，我们必须创建一个**ProcessEngine**，然后由它来访问那些服务。这里我们使用*'standalone'*配置，它使用我们demo的数据库来创建一个**ProcessEngine**。  
你可以在[这里](https://www.activiti.org/userguide/images/FinancialReportProcess.bpmn20.xml)下载这个流程定义。这个文件里不仅包含我们上面展示的那些XML信息，还有Activiti工具提供的必要的BPMN[生成图片配置](https://www.activiti.org/userguide/index.html#generatingProcessDiagram)来显示流程信息。
```java
public static void main(String[] args) {

  // Create Activiti process engine
  ProcessEngine processEngine = ProcessEngineConfiguration
    .createStandaloneProcessEngineConfiguration()
    .buildProcessEngine();

  // Get Activiti services
  RepositoryService repositoryService = processEngine.getRepositoryService();
  RuntimeService runtimeService = processEngine.getRuntimeService();

  // Deploy the process definition
  repositoryService.createDeployment()
    .addClasspathResource("FinancialReportProcess.bpmn20.xml")
    .deploy();

  // Start a process instance
  runtimeService.startProcessInstanceByKey("financialReport");
}
```
### 任务列表
我们可以使用下面的逻辑来通过**TaskService**查询任务信息：
```java
List<Task> tasks = taskService.createTaskQuery().taskCandidateUser("kermit").list();
```
注意，我们传给这个方法的用户必须是*会计部*这个部门的用户，因为我们在流程定义中已经做了如下声明：
```xml
<potentialOwner>
  <resourceAssignmentExpression>
    <formalExpression>accountancy</formalExpression>
  </resourceAssignmentExpression>
</potentialOwner>
```
我们还可以通过用户组的名称调用任务查询API来获取同样的结果，我们可以在代码中添加如下逻辑：
```
TaskService taskService = processEngine.getTaskService();
List<Task> tasks = taskService.createTaskQuery().taskCandidateGroup("accountancy").list();
```
因为我们使用了跟demo程序相同的数据库来配置我们的**ProcessEngine**，我们可以登录到[Activiti Explorer](http://localhost:8080/activiti-explorer/)。默认情况下，*会计部*下面没有用户。使用kermit/kermit 登录进系统，点击 Group -> "Create Group"，接着再点击 Users，然后将 fozzie 加到这个部门中。现在使用 fozzie/fozzie登录进系统，选择*Processes*页面，我们就可以启动我们的业务流程，点击 *Monthly financial report'* 对应的 *'Actions'* 列下的 *'Start Process'* 链接。  
![启动一个流程图](/images/activiti/bpmn.financial.report.example.start.process.png)  
就像上面介绍的，流程将会执行到第一个用户任务。如果我们现在使用 kermit登录，那么在流程实例启动后，我们可以看到一个新的待办任务。我们可以点击*Tasks*页面来查看这个任务。注意，就算这个流程是别人启动的，它对于会计部的所有用户都是可见的。  
![待办任务](/images/activiti/bpmn.financial.report.example.task.assigned.png)  
### 申领任务
一个会计师现在需要申领任务。通过申领这个任务，该特定的用户就变成了这个任务的**代理人**，然后这个任务就从会计部其他成员的任务列表中移除。通过程序申领一个任务的代码如下：
```java
taskService.claim(task.getId(), "fozzie");
```
现在这个任务就在**申领该任务的用户的个人任务列表中**
```java
	List<Task> tasks = taskService.createTaskQuery().taskAssignee("fozzie").list();
```
在Activiti UI APP中，点击 *claim* 按钮，也会做相同的操作。该任务就会移到当前登录用户的个人任务列表中。你还能看到任务的受理人就是当前的登录用户。
![申领任务图](/images/activiti/bpmn.financial.report.example.claim.task.png)  
### 完成任务
该会计师现在就要开始写财务报告了，等到他写完，就可以点击 **完成任务**，这样就这个任务的所有事情都做完了。
```java
taskService.complete(task.getId());
```
对于Activiti引擎，这个外部信号告诉自己流程实例要继续往下执行了。任务本身将从运行时数据中删除，紧接着就是一个任务流出操作，将执行交给第二个任务(*'审批报告'*)。第二个任务的分派工作跟上文描述的第一个任务机制基本相同，唯一的不同就是任务将会分配给*管理部*。  
在演示程序中，完成任务是通过在任务列表中点击*complete*按钮。因为 Fozzie 不是一个高管(原文写错成'会计'-译者注)，所以我们需要从Activiti Explorer 登出，然后使用 *kermit*(他是高管)登录，第二个任务就出现在了我们的待办列表中。  
### 结束任务
我们可以用跟之前完全相同的方式来检索和申领工作。完成第二个任务之后，流程就会走到最后，该流程实例终止，所有跟这个流程实例相关的运行时执行数据均会从数据库中删除。  
当你登录到Activiti Explorer之后就能验证这些，存储流程执行信息的数据库表中已经没有数据了。  
![流程结束后的数据图](/images/activiti/bpmn.financial.report.example.process.ended.png)
你还可以使用**historyService**来用代码的方式验证流程已经终止：
```java
HistoryService historyService = processEngine.getHistoryService();
HistoricProcessInstance historicProcessInstance =
historyService.createHistoricProcessInstanceQuery().processInstanceId(procId).singleResult();
System.out.println("Process instance end time: " + historicProcessInstance.getEndTime());
```
### 代码概览
结合前面的代码片段，你就有了下面这样一坨代码(这段代码考虑到你可能会通过Activiti Explorer UI 启动一些流程实例。因此，它总是查询任务列表而不是一个单独的任务)
```java
public class TenMinuteTutorial {

  public static void main(String[] args) {

    // 创建一个流程引擎
    ProcessEngine processEngine = ProcessEngineConfiguration
      .createStandaloneProcessEngineConfiguration()
      .buildProcessEngine();

    // 获取Activiti服务
    RepositoryService repositoryService = processEngine.getRepositoryService();
    RuntimeService runtimeService = processEngine.getRuntimeService();

    // 发布一个流程定义
    repositoryService.createDeployment()
      .addClasspathResource("FinancialReportProcess.bpmn20.xml")
      .deploy();

    // 启动一个流程实例
    String procId = runtimeService.startProcessInstanceByKey("financialReport").getId();

    // 查询第一个任务
    TaskService taskService = processEngine.getTaskService();
    List<Task> tasks = taskService.createTaskQuery().taskCandidateGroup("accountancy").list();
    for (Task task : tasks) {
      System.out.println("Following task is available for accountancy group: " + task.getName());

      // 申领一个任务
      taskService.claim(task.getId(), "fozzie");
    }

    // 验证 fozzie 现在可以检索到任务
    tasks = taskService.createTaskQuery().taskAssignee("fozzie").list();
    for (Task task : tasks) {
      System.out.println("Task for fozzie: " + task.getName());

      // 完成任务
      taskService.complete(task.getId());
    }

    System.out.println("Number of tasks for fozzie: "
            + taskService.createTaskQuery().taskAssignee("fozzie").count());

    // 检索和申领第二个任务
    tasks = taskService.createTaskQuery().taskCandidateGroup("management").list();
    for (Task task : tasks) {
      System.out.println("Following task is available for management group: " + task.getName());
      taskService.claim(task.getId(), "kermit");
    }

    // 完成第二个任务流程终止
    for (Task task : tasks) {
      taskService.complete(task.getId());
    }

    // 验证流程真的结束了
    HistoryService historyService = processEngine.getHistoryService();
    HistoricProcessInstance historicProcessInstance =
      historyService.createHistoricProcessInstanceQuery().processInstanceId(procId).singleResult();
    System.out.println("Process instance end time: " + historicProcessInstance.getEndTime());
  }

}
```
## 将来的改进
很容易就看出来这个业务流程相对于我们实际使用的流程来说太简单了。但是，当你读完Activiti提供的BPMN2.0规范后，你就可以使用下面几种方式来增强你的业务流程了：
- 定义**网关**来做决策。这样，高管可以拒绝会计的财务报告，然后系统会重新为会计创建任务
- 定义和使用 **变量**，以便我们存储或者引用这个报告，然后让它在表单中展示
- 在流程最后定义一个**服务任务**，它会将报告发给每一个股东
- 等等