---
layout: post
title: Activiti 6.0 用户文档 - 第四章：API
categories: activiti
description: 流程引擎 Activiti 第四章：API
keywords: 流程引擎, activiti
---
# Activiti API
## 流程引擎的 API 和 服务
API 是使用Activiti的最常用方式。 核心的切入点是 **ProcessEngine**，这个类有很多创建方式，我们在上文[配置](https://www.activiti.org/userguide/index.html#configuration)那一章已经介绍过。你可以通过 ProcessEngine 提供的方法获得各种工作流/BPM服务。ProcessEngine和服务对象都是现成安全的，所以你可以放心在全局使用同一个对象。  
![Activiti服务示例](/images/activiti/api.services.png)   
```java
ProcessEngine processEngine = ProcessEngines.getDefaultProcessEngine();

RuntimeService runtimeService = processEngine.getRuntimeService();
RepositoryService repositoryService = processEngine.getRepositoryService();
TaskService taskService = processEngine.getTaskService();
ManagementService managementService = processEngine.getManagementService();
IdentityService identityService = processEngine.getIdentityService();
HistoryService historyService = processEngine.getHistoryService();
FormService formService = processEngine.getFormService();
DynamicBpmnService dynamicBpmnService = processEngine.getDynamicBpmnService();
```
**ProcessEngines.getDefaultProcessEngine()** 该方法只在第一次调用的时候创建并初始化一个流程引擎，以后的调用都是返回同一个引擎对象。可以使用**ProcessEngine.init()**和**ProcessEngines.destroy()**来创建和关闭所有流程引擎。  
*ProcessEngines*会扫描所有的**activiti.cfg.xml**和**activiti-context.xml**文件。对于**activiti.cfg.xml**，系统会按照Activiti的典型风格来创建流程引擎：**ProcessEngineConfiguration.createProcessEngineConfigurationFromInputStream(inputStream).buildProcessEngine()**。对于 **activiti-context.xml** 文件，系统会使用spring的方式创建流程引擎。Spring的上下文会先加载，然后会将流程引擎管理起来。  
所有的服务都是无状态的。这意味着你可以轻松地多节点运行Activiti，虽然所有节点操作同一个数据，你也不用担心重复执行的问题。所有的服务都不管在哪儿执行的，都做了幂等。  
如果你想使用Activiti 引擎，首先需要的就是**RepositoryService**。通过它，你可以管理和维护 **发布** 和 **流程定义**。**流程定义**就是BPMN2.0中的流程，我们这里不做过多介绍，它表示每个步骤的结构和行为。**发布**是Activiti引擎的打包单位，一个发布可以包含若干BPMN2.0 xml文件和一些别的资源，从一个单独的BPMN2.0 xml流程文件到流程以及相关资源(比如一个 hr 流程的发布可以包含跟hr流程有关的所有东西)的整体打包，具体包含哪些东西取决于开发人员。**RepositoryService** 允许**发布**各种包。发布意味着要将流程加载到引擎中，所有的流程在保存到数据库之前都会被校验和解析。从这个时候起，系统就知道这次发布，里面所有的流程就开始启动了。  
此外，RepositoryService 还允许：
- 查询引擎中的发布和流程定义
- 挂起和激活整个发布或某个流程定义。挂起的意思说，接下来不能做任何操作了。激活的意思刚好相反。
- 检索各种资源，例如发布中包含的文件或引擎自动生成的流程图
- 检索流程定义中的POJO版本，该版本可以允许使用Java而不是xml来跟踪流程
**RepositoryService**是对静态信息(不会变动的数据，至少不会经常变)的描述，而**RuntimeService**刚好相反，它处理的是流程定义后启动的新流程实例。按照上文所说，**流程定义**定义的是流程的框架和流程中每一步的动作。而流程实例是流程定义的一种执行方式。对于某一个流程定义，一般都会有很多实例在同时执行。**RuntimeService**还提供撤回和存储**流程变量**的功能。这些特定流程实例的数据可以被流程中的各种构造器使用(比如，独立网关通常使用流程变量来决定接下来选择那个路径来及继续流程)。通过**RuntimeService**我们还可以查询流程实例和执行情况。执行情况对应着BPMN2.0中"**token**"的概念。*执行* 其实是一个指向当前流程实例的一个引用。一个流程实例可以拥有很多**wait states(等待状态)**，RuntimeService里面有很多操作可以向实例发送信号，然后实例收到这些外部请求后就可以继续往下走了。  
BPMN引擎(如Activiti)的核心是那些需要由真实的人执行的任务。所有这些任务相关的东西都由**TaskService**管理，比如：
- 查询用户或者用户组的任务
- 创建一个*独立的*任务。这些任务不属于任何一个流程示例
- 维护任务归属的用户或者以某种方式参与到任务中的用户
- 领取并完成一个任务。领取的意思是说某个用户决定接收并完成某个任务。完成的意思是说，去做任务要求的工作，一般是填写各种各样的表格。  
**IdentityService**很简单，它可以维护(增删改查...)用户组和用户。你一定要明白，Activiti在运行的时候，不会对用户做任何校验。比如，一个任务可以被分配给任何用户，但是引擎不会校验该用户是不是系统的用户。这就是Activiti要跟LDAP，Active Dictionary等服务结合使用的原因。  
**FormService**是一个可选服务，没有它，Activiti也完全可以正常运行。它引入了*启动表单*和*任务表单*的概念。*启动表单*是一种在流程实例启动之前就展示给用户的表单，而*任务表单*在用户完成表单的时候展示的表单。Activiti允许用户在BPMN2.0流程中定义这些表单。它以一种很简单的方式暴露自己的数据。但要再声明一下，这个服务是可选的，因为表单不需要嵌入到流程定义中。  
**HistoryService**提供了引擎收集的历史数据服务。当流程执行的时候，比如流程实例启动的时间，谁执行了什么操作，流程用了多少时间走完，实例走了哪个路径等很多数据都被流程引擎保存下来了(这个是可配置的)。HistoryService提供很多查询这些数据的服务。  
使用Activiti编写自定义程序时，通常不需要**ManagementService**。它允许用户检索数据库表和表的元数据信息。此外，他还开放了作业的查询和管理功能。Activiti使用工作来做各种事情，比如定时器，异步扩展，延时挂起/激活等。稍后，我们会更详细地讨论它。  
**DynamicBpmnService**可以在不重新发布的情况下修改流程定义。比如你可以在流程定义中修改任务接收人，或者修改服务任务的类名。  
点击[接口文档](https://www.activiti.org/javadocs/index.html)查看更多服务操作和引擎API。
## 异常策略
Activiti中异常类的基类是**org.activiti.engine.ActivitiException**，这是一个非检查类。API可以在任何时候抛出这个异常，但是在一些特定的方法中，会抛出来预期的一些异常(参考[文档](http://activiti.org/javadocs/index.html))。下面是 **TaskService**中的一个例子：
```java
/**
 * 当任务正常执行时该方法会被调用。
 * @param taskId 要完成的任务的 id，不能为null.
 * @throws ActivitiObjectNotFoundException 当该id不存在的时候抛出此异常.
 */
 void complete(String taskId);
```
在上面的例子中，当传过来的id查不到任务的时候，就会抛异常。而且，因为接口文档中已经声明**taskId不能为null，所以当你的传参是null的时候，就会抛一个ActivitiIllegalArgumentException**。  
虽然我们也不想将异常层次设计太复杂，但是为了应对特殊情况，我们也不可避免地添加了下面的几个异常子类。在流程运行或者API调用中抛的异常要么就是普通的 **ActivitiExceptions**，要么就是下面这几种：
- ActivitiWrongDbException：当Activiti发现自己的版本跟数据库版本不匹配时，就会报这个异常
- ActivitiOptimisticLockingException：当多线程访问同一个实体对象导致数据库报乐观锁异常时，会报这个异常
- ActivitiClassLoadingException：当要加载的类不存在时(比如配置的 JavaDelegates, TaskListeners等)，就会报这个异常
- ActivitiObjectNotFoundException：当请求或者操作的对象不存在时，会报这个异常
- ActivitiIllegalArgumentException：把一个非法的参数传给API，或者配置给引擎，或者提供给流程定义使用或传递，均会报这个错
- ActivitiTaskAlreadyClaimedException：在一个任务已经声明后再次调用**taskService.claim(…​)**就会报这个异常
## 使用 Activiti 服务
按前文所述，我们可以通过**org.activiti.engine.ProcessEngine**的实例提供的服务来使用Activiti。下面的代码片段假定你已经有一个运行起来的Activiti环境，比如你有权限访问**org.activiti.engine.ProcessEngine**。如果你只是想简单地运行下面的代码，你可以下载或者clone [Activiti单测模板](https://github.com/Activiti/activiti-unit-test-template)，将它引用到你的IDE，然后在你的**org.activiti.MyUnitTest**单测中添加**testUserguideCode()**方法。  
本教程的最终目的是创建一个可运行的仿公司假期申请的业务流程：  
![假期管理流程](/images/activiti/api.vacationRequest.png)  
### 发布该流程
通过**RepositoryService**我们可以访问与*静态数据*(比如流程定义信息)的所有内容。从概念上讲，这些数据都是Activiti引擎*存储库*的内容。  
在 **src/test/resources/org/activiti/test**目录(如果没用单测模板，那就按你的规则放也行)下创建一个**VacationRequest.bpmn20.xml**，将下文中的内容拷贝进去。注意本章并不会介绍案例中的xml的框架。如果你想熟悉这些xml的结构，你可以点击阅读[BPMN2.0相关章节](https://www.activiti.org/userguide/index.html#bpmn20)。
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<definitions id="definitions"
             targetNamespace="http://activiti.org/bpmn20"
             xmlns="http://www.omg.org/spec/BPMN/20100524/MODEL"
             xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xmlns:activiti="http://activiti.org/bpmn">

  <process id="vacationRequest" name="Vacation request">

    <startEvent id="request" activiti:initiator="employeeName">
      <extensionElements>
        <activiti:formProperty id="numberOfDays" name="Number of days" type="long" value="1" required="true"/>
        <activiti:formProperty id="startDate" name="First day of holiday (dd-MM-yyy)" datePattern="dd-MM-yyyy hh:mm" type="date" required="true" />
        <activiti:formProperty id="vacationMotivation" name="Motivation" type="string" />
      </extensionElements>
    </startEvent>
    <sequenceFlow id="flow1" sourceRef="request" targetRef="handleRequest" />

    <userTask id="handleRequest" name="Handle vacation request" >
      <documentation>
        ${employeeName} would like to take ${numberOfDays} day(s) of vacation (Motivation: ${vacationMotivation}).
      </documentation>
      <extensionElements>
         <activiti:formProperty id="vacationApproved" name="Do you approve this vacation" type="enum" required="true">
          <activiti:value id="true" name="Approve" />
          <activiti:value id="false" name="Reject" />
        </activiti:formProperty>
        <activiti:formProperty id="managerMotivation" name="Motivation" type="string" />
      </extensionElements>
      <potentialOwner>
        <resourceAssignmentExpression>
          <formalExpression>management</formalExpression>
        </resourceAssignmentExpression>
      </potentialOwner>
    </userTask>
    <sequenceFlow id="flow2" sourceRef="handleRequest" targetRef="requestApprovedDecision" />

    <exclusiveGateway id="requestApprovedDecision" name="Request approved?" />
    <sequenceFlow id="flow3" sourceRef="requestApprovedDecision" targetRef="sendApprovalMail">
      <conditionExpression xsi:type="tFormalExpression">${vacationApproved == 'true'}</conditionExpression>
    </sequenceFlow>

    <task id="sendApprovalMail" name="Send confirmation e-mail" />
    <sequenceFlow id="flow4" sourceRef="sendApprovalMail" targetRef="theEnd1" />
    <endEvent id="theEnd1" />

    <sequenceFlow id="flow5" sourceRef="requestApprovedDecision" targetRef="adjustVacationRequestTask">
      <conditionExpression xsi:type="tFormalExpression">${vacationApproved == 'false'}</conditionExpression>
    </sequenceFlow>

    <userTask id="adjustVacationRequestTask" name="Adjust vacation request">
      <documentation>
        Your manager has disapproved your vacation request for ${numberOfDays} days.
        Reason: ${managerMotivation}
      </documentation>
      <extensionElements>
        <activiti:formProperty id="numberOfDays" name="Number of days" value="${numberOfDays}" type="long" required="true"/>
        <activiti:formProperty id="startDate" name="First day of holiday (dd-MM-yyy)" value="${startDate}" datePattern="dd-MM-yyyy hh:mm" type="date" required="true" />
        <activiti:formProperty id="vacationMotivation" name="Motivation" value="${vacationMotivation}" type="string" />
        <activiti:formProperty id="resendRequest" name="Resend vacation request to manager?" type="enum" required="true">
          <activiti:value id="true" name="Yes" />
          <activiti:value id="false" name="No" />
        </activiti:formProperty>
      </extensionElements>
      <humanPerformer>
        <resourceAssignmentExpression>
          <formalExpression>${employeeName}</formalExpression>
        </resourceAssignmentExpression>
      </humanPerformer>
    </userTask>
    <sequenceFlow id="flow6" sourceRef="adjustVacationRequestTask" targetRef="resendRequestDecision" />

    <exclusiveGateway id="resendRequestDecision" name="Resend request?" />
    <sequenceFlow id="flow7" sourceRef="resendRequestDecision" targetRef="handleRequest">
      <conditionExpression xsi:type="tFormalExpression">${resendRequest == 'true'}</conditionExpression>
    </sequenceFlow>

     <sequenceFlow id="flow8" sourceRef="resendRequestDecision" targetRef="theEnd2">
      <conditionExpression xsi:type="tFormalExpression">${resendRequest == 'false'}</conditionExpression>
    </sequenceFlow>
    <endEvent id="theEnd2" />

  </process>

</definitions>
```
你必须*发布*这个流程来让流程引擎知道它的存在。发布的意思就是流程引擎将这个BPMN2.0 xml解析成可执行的逻辑，然后*发布*中每一个流程定义都会对应着数据库的一条记录。通过这种方式，当流程引擎重启的时候，它也会拿到所有*发布*过的流程。
```java
ProcessEngine processEngine = ProcessEngines.getDefaultProcessEngine();
RepositoryService repositoryService = processEngine.getRepositoryService();
repositoryService.createDeployment()
  .addClasspathResource("org/activiti/test/VacationRequest.bpmn20.xml")
  .deploy();

Log.info("Number of process definitions: " + repositoryService.createProcessDefinitionQuery().count());
```
关于发布，你可以点击[发布章节](https://www.activiti.org/userguide/index.html#chDeployment)了解更多内容。
### 启动一个流程实例
当一个流程定义被部署到Activiti之后，我们可以启动一个新的流程实例。每个流程定义都对应着若干流程实例，可以理解为，流程定义是*蓝本*，然后一个流程实例是它的一次运行。  
使用**RuntimeService**可以查到所有流程相关的运行状态。启动一个流程实例的方式有很多，下面的代码片段中，我们使用xml中定义的流程key来启动流程引擎，在启动的时候，我们还提供了一些流程变量，因为第一个用户任务要用在表达式中用到这些变量。流程变量经常会被用到，因为它们让特定的流程定义变得有意义。一般来说，流程实例不相同是因为它们的流程变量不同。  
```java
Map<String, Object> variables = new HashMap<String, Object>();
variables.put("employeeName", "Kermit");
variables.put("numberOfDays", new Integer(4));
variables.put("vacationMotivation", "I'm really tired!");

RuntimeService runtimeService = processEngine.getRuntimeService();
ProcessInstance processInstance = runtimeService.startProcessInstanceByKey("vacationRequest", variables);

// 验证我们真的启动了一个流程实例
Log.info("Number of process instances: " + runtimeService.createProcessInstanceQuery().count());
```
### 完成任务
当这个流程启动的时候，首先是一个用户任务。这一步必须由系统的用户来执行。通常，系统用户都有一个*收件箱*，里面展示了该用户需要完成的所有任务。下面的代码片段展示了如何执行这类查询：
```java
// 列出 "management" 用户组所有的任务
TaskService taskService = processEngine.getTaskService();
List<Task> tasks = taskService.createTaskQuery().taskCandidateGroup("management").list();
for (Task task : tasks) {
  Log.info("Task available: " + task.getName());
}
```
为了让流程实例继续往下流转，我们必须完成当前的任务。对于Activiti，这意味着你需要**完成**这个任务。下面的代码片段展示了要怎么操作：
```java
Task task = tasks.get(0);

Map<String, Object> taskVariables = new HashMap<String, Object>();
taskVariables.put("vacationApproved", "false");
taskVariables.put("managerMotivation", "We have a tight deadline!");
taskService.complete(task.getId(), taskVariables);
```
这样流程实例就会流转到下一步。在这个例子中，下一步允许员工根据自己的请假请求填写一个表格。员工可以重新提交休假请求，这将使流程重新回到启动任务那里。
### 挂起和激活流程
我们可以挂起一个流程。当流程被挂起的时候，它就不能创建新的流程实例了(会抛异常)。我们可以通过**RepositoryService**挂起一个流程定义：
```java
repositoryService.suspendProcessDefinitionByKey("vacationRequest");
try {
  runtimeService.startProcessInstanceByKey("vacationRequest");
} catch (ActivitiException e) {
  e.printStackTrace();
}
```
想重新激活这个流程，调用**repositoryService.activateProcessDefinitionXXX**中的一个方法就就可以。  
我们还可以挂起一个流程实例，此时这个流程就不能往下走了(调用完成任务时会报异常)，而且作业(比如定时器)也不会被执行。调用**runtimeService.suspendProcessInstance**就可以挂起一个流程实例，当然，你也可以调用**runtimeService.activateProcessInstanceXXX**重新激活它  
### 补充说明
上面章节中，我们都没有深挖Activiti的特性。我们接下来会对这些章节做扩展，介绍更多的Activiti API。当然了，与任何开源项目一样，最好的学习方法是阅读源码和Javadoc。
## 查询API
有两种方式查询引擎的数据：API查询和本地查询。通过API，我们可以编写类型安全的查询代码。你可以添加各式的查询条件(所有的条件会被使用AND拼接)和一个排序条件。下面是一个简单示例：
```java
List<Task> tasks = taskService.createTaskQuery()
    .taskAssignee("kermit")
    .processVariableValueEquals("orderId", "0815")
    .orderByDueDate().asc()
    .list();
```
有时，你需要更强的查询语句，比如使用 OR 运算符或者使用QueryAPI无法表达的限制条件。针对这种情况，我们引入了本地查询，可以让用户执行自己的sql。返回值是由你使用的Query对象决定的，数据会被映射到正确的对象中，比如：Task, ProcessInstance, Execution等等。因为查询语句需要用到数据库表和列名，这就要求你对数据库的结构很了解，我们建议要谨慎使用本地查询。数据库表名可以通过API检索，从而尽可能降低对硬编码的依赖。
```java
List<Task> tasks = taskService.createNativeTaskQuery()
  .sql("SELECT count(*) FROM " + managementService.getTableName(Task.class) + " T WHERE T.NAME_ = #{taskName}")
  .parameter("taskName", "gonzoTask")
  .list();

long count = taskService.createNativeTaskQuery()
  .sql("SELECT count(*) FROM " + managementService.getTableName(Task.class) + " T1, "
    + managementService.getTableName(VariableInstanceEntity.class) + " V1 WHERE V1.TASK_ID_ = T1.ID_")
  .count();
```
## 变量
每一个流程实例需要数据来执行各个步骤。在Activiti，这些数据称为*变量*，这些变量保存在数据库中。变量可以在表达式(比如，在排他网关中选择合适的流出路径)中使用，也可以在java服务任务调用外部服务时(比如提供输入值或者保存服务调用的结果)使用等。  
不仅流程实例可以拥有变量(叫做*流程变量*)，*执行*(指向流程活动位置的特定引用)和用户任务也都可以拥有变量。一个流程实例可以拥有任意数量的变量。每个变量对应着*ACT_RU_VARIABLE*表中的一条记录。  
任意*startProcessInstanceXXX*方法都有一个可选参数，用于给流程实例的创建和启动提供参数。比如，在*RuntimeService*中：
```java
ProcessInstance startProcessInstanceByKey(String processDefinitionKey, Map<String, 
```
变量还可以在流程执行的时候添加。比如(*RuntimeService*)：
```java
void setVariable(String executionId, String variableName, Object value);
void setVariableLocal(String executionId, String variableName, Object value);
void setVariables(String executionId, Map<String, ? extends Object> variables);
void setVariablesLocal(String executionId, Map<String, ? extends Object> variables);
```
注意，变量还可以被设置成针对某个执行(别忘了，一个流程实例是由一个执行数组成)是私有的(*local*)。该变量只在那个执行中可见，其他的执行不可见。如果某个重要数据不能在流程实例中传递，或者该数据在流经一个特定路径的时会变化(比如在平行路径中)，这个功能就就很有用了。  
变量可以像下面这样反复取得。注意，相似的方法在*TaskService*中也有。这就表示，任务和执行一样，在执行期间可以获取到局部变量。  
```java
Map<String, Object> getVariables(String executionId);
Map<String, Object> getVariablesLocal(String executionId);
Map<String, Object> getVariables(String executionId, Collection<String> variableNames);
Map<String, Object> getVariablesLocal(String executionId, Collection<String> variableNames);
Object getVariable(String executionId, String variableName);
<T> T getVariable(String executionId, String variableName, Class<T> variableClass);
```
变量经常在[Java代理](https://www.activiti.org/userguide/index.html#bpmnJavaServiceTask)、[表达式](https://www.activiti.org/userguide/index.html#apiExpressions)、执行或任务监听器、脚本等场景中使用。在这些结构中，当前的*执行*或者*任务*可以看见并且设置或者检索这些变量。最简单的方法是下面这些：
```java
execution.getVariables();
execution.getVariables(Collection<String> variableNames);
execution.getVariable(String variableName);

execution.setVariables(Map<String, object> variables);
execution.setVariable(String variableName, Object value);
```
注意，上面这些方法也可以看到局部(*local*)变量。 
处于历史(和向后兼容)的原因，当执行上述调用的时候，**所有**的变量都会从数据库取。 这意味着，如果你有10个变量，并且只通过*getVariable("myVariable")*去获取一个变量，后台还是会缓存另外9个变量。这样处理可以防止后续的调用再去访问数据库。比如当你的流程定义有3个顺序的服务任务(和一个数据库事务)时，一次调用就获取到所有的变量比每个服务任务都获取一次要更好。注意，获取和设置变量**也是这样**。  
当然了，如果你有大量变量或者想严格控制数据库查询次数和数据量，这种处理方式并不适合。Activiti5.17之后，我们增加了一个方法来对这个情况加以控制，该方法有一个参数来标记是否想要缓存所有的变量：
```java
Map<String, Object> getVariables(Collection<String> variableNames, boolean fetchAllVariables);
Object getVariable(String variableName, boolean fetchAllVariables);
void setVariable(String variableName, Object value, boolean fetchAllVariables);
```
当*fetchAllVariables*为*true*时，程序就会像上述那样，当获取或修改一个参数时，所有的参数都会被查出来并缓存。  
但是，当该字段是*false*的时候，系统就会执行特定的查询语句，不会查询其他参数，系统也只会缓存当前字段。
## 瞬态(Transient) 变量
瞬态变量用起来跟普通变量一样，只是它不会被存储。通常，瞬态变量为了一些高级场景使用的(如果你不知道怎么用，就去用普通变量)。  
瞬态变量特性如下：
- 所有的瞬态变量的历史数据都不会被保存
- 类似*普通*变量，瞬态变量在设置时放在*最高*父级上。这意味着，在给一个执行设置变量时，瞬态变量也会被存储在流程实例的执行上。就像普通变量一样，如果想要在一个特定的执行上设置变量，那就使用*局部*变量
- 瞬态变量只能在下一个等待状态之前被访问，然后就没了。等待状态的意思是：流程实例被存储在数据库中。注意，*异步*活动也是*等待状态*的一种。
- 瞬态变量只能被*setTransientVariable(name, value)*设置，但是当你调用*getVariable(name)*(我们有一个*getTransientVariable(name)*方法，它只会去查找瞬态变量)时，瞬态变量也会被返回。这么处理可以让表达式编写变得更容易，而且兼容现有的变量使用逻辑。  
- 瞬态变量会*影响*同名的普通变量。这意味着，当你在流程实例上设置同名的两种变量时，调用*getVariable("someVariable")*返回的是瞬态变量   

大多数情况下，常规变量使用的地方瞬态变量都可以使用：
- *JavaDelegate*的实现中的*DelegateExecution*
- *ExecutionListener*的实现中的*DelegateExecution*，*TaskListener*实现中的*DelegateTask*
- 在*execution*对象的脚本任务中
- 通过运行时服务启动流程实例的时候
- 当完成一个任务的时候
- 当调用*runtimeService.trigger*的时候  

这些方法遵循普通流程变量的命名约定：
```java
void setTransientVariable(String variableName, Object variableValue);
void setTransientVariableLocal(String variableName, Object variableValue);
void setTransientVariables(Map<String, Object> transientVariables);
void setTransientVariablesLocal(Map<String, Object> transientVariables);

Object getTransientVariable(String variableName);
Object getTransientVariableLocal(String variableName);

Map<String, Object> getTransientVariables();
Map<String, Object> getTransientVariablesLocal();

void removeTransientVariable(String variableName);
void removeTransientVariableLocal(String variableName);
```
下面的BPMN图形举了个典型的例子：  
![瞬态变量使用距离](/images/activiti/api.transient.variable.example.png)
我们假设，*Fetch Data*任务调用了一个远程服务(比如使用REST)。我们再假设其他一些配置参数也要在流程启动的时候提供。而且，这些参数对于历史审计并不重要，因此我们将它作为瞬态变量传递：
```java
ProcessInstance processInstance = runtimeService.createProcessInstanceBuilder()
       .processDefinitionKey("someKey")
       .transientVariable("configParam01", "A")
       .transientVariable("configParam02", "B")
       .transientVariable("configParam03", "C")
       .start();
```
注意，这些变量在用户任务到达并且入库之前都是可见的。比如，在*Additional Work*用户任务中，这些变量就不可见了。你还要注意，如果*Fetch Data*是异步的，在这一步执行完之后，这些变量也不可见了。  
*Fetch Data*(简化版)可能是这样的：
```java
public static class FetchDataServiceTask implements JavaDelegate {
  public void execute(DelegateExecution execution) {
    String configParam01 = (String) execution.getVariable(configParam01);
    // ...

    RestReponse restResponse = executeRestCall();
    execution.setTransientVariable("response", restResponse.getBody());
    execution.setTransientVariable("status", restResponse.getStatus());
  }
}
```
如果需要的话，*Process Data*会获取到瞬态变量中的值，将这些数据解析并存储到流程变量中。  
是否使用瞬态变量作为排他网关的判断条件都无所谓(本例是使用瞬态变量*status*)：
```xml
<conditionExpression xsi:type="tFormalExpression">${status == 200}</conditionExpression>
```
## 表达式
Activiti 使用 UEL 作为解析表达式。UEL(Unified Expression Language)表示*统一表达式语言*，是EE6规范的一部分(详情参考[EE6规范](http://docs.oracle.com/javaee/6/tutorial/doc/gjddd.html))。为了在所有环境中支持最新的UEL规范，我们使用它的一个变更版本JUEL。  
表达式可以在诸如[Java服务任务]、[执行监听器]、[任务监听器]和[条件流]等场景中。虽然有值表和方法表达式两种，Activiti对它们做了一层抽象，所以它们同时作为 **express**使用。  
- 值表达式：解析成一个值。默认的所有的流程变量都可以拿来用。而且所有的spring bean(如果你用了spring)也可以使用。简单示例如下：
```js
${myVar}
${myBean.myProperty}
```
- 方法表达式：执行一个方法，带不带参数都行。**如果是执行一个不带参数的方法，请确认方法名后面一定要有一个空括号(这是为了将它跟值表达式区分开)。**参数可以是固定值，也可以是表达式，它会自己解析。比如：
```java
${printer.print()}
${myBean.addNewOrder('orderName')}
${myBean.doSomething(myVar, execution)}
```
注意，这些表达式支持解析原语(也包含比较它们)，bean，list，array，map。  
除了流程变量之外，还有一些默认的对象可以用于表达式：
- execution: 存有要流向的执行的扩展信息的**DelegateExecution**
- task：存有当前任务的扩展信息的**DelegateTask**。**注意：这个仅仅适用于从任务监听器的表达式**
- authenticatedUserId：当前登录的用户id，如果用户没有登录，这个变量不可见。  

更多实际应用和示例，可以参考：[spring表达式](https://www.activiti.org/userguide/index.html#springExpressions)，[Java服务任务](https://www.activiti.org/userguide/index.html#bpmnJavaServiceTaskXML)，[执行监听器](https://www.activiti.org/userguide/index.html#executionListeners)，[任务监听器](https://www.activiti.org/userguide/index.html#taskListeners)或者[条件流程](https://www.activiti.org/userguide/index.html#conditionalSequenceFlowXml)  
## 单元测试
业务流程是软件产品的一部分，所以它也要跟其他普通业务逻辑一样做单元测试。因为Activiti是一个嵌入式的Java引擎，所以测试业务流程和其他普通单测一样。  
Activiti支持JUnit3和JUnit4风格的单测。在JUnit3风格下，单测一定要继承**org.activiti.engine.test.ActivitiTestCase**。这样就能获取到里面的protected成员变量：ProcessEngine。在用例的**setUp()**方法中，系统会默认按照 **activiti.cfg.xml**中的配置初始化 ProcessEngine。如果想指定其他配置文件，可以重写*getConfigurationResource()*。如果配置文件一样，流程引擎会被缓存，以供所有的测试用例使用。  
通过继承**ActivitiTestCase**，你可以使用**org.activiti.engine.test.Deployment**来注解测试方法。在测试执行前，测试类同目录下的**testClassName.testMethod.bpmn20.xml**会被发布。在测试运行完之后，该发布以及相关的流程实例、任务等数据均会被删除。**Deployment**注解还支持自定义资源路径，详情可以参考这个类的源码。  
鉴于上面所有这些，一个JUnit3风格的测试用例如下：
```java
public class MyBusinessProcessTest extends ActivitiTestCase {

  @Deployment
  public void testSimpleProcess() {
    runtimeService.startProcessInstanceByKey("simpleProcess");

    Task task = taskService.createTaskQuery().singleResult();
    assertEquals("My Task", task.getName());

    taskService.complete(task.getId());
    assertEquals(0, runtimeService.createProcessInstanceQuery().count());
  }
}
```
使用JUnit4实现上述功能的单测，必须要用到一个规则：**org.activiti.engine.test.ActivitiRule**。通过这个规则流程引擎和服务都可以通过getter方式获取到。和前文所述**ActivitiTestCase**一样，通过使用这个**Rule**可以激活**org.activiti.engine.test.Deployment**注解(参考上文中关于它的使用和配置介绍)，然后它将会在类加载路径中加载默认配置文件。如果各个单测使用的是相同的配置资源，流程引擎将会被缓存起来共享。  
下面的代码片段展示了JUnit4风格的**ActivitiRule**的用法示例：
```java
public class MyBusinessProcessTest {

  @Rule
  public ActivitiRule activitiRule = new ActivitiRule();

  @Test
  @Deployment
  public void ruleUsageExample() {
    RuntimeService runtimeService = activitiRule.getRuntimeService();
    runtimeService.startProcessInstanceByKey("ruleUsage");

    TaskService taskService = activitiRule.getTaskService();
    Task task = taskService.createTaskQuery().singleResult();
    assertEquals("My Task", task.getName());

    taskService.complete(task.getId());
    assertEquals(0, runtimeService.createProcessInstanceQuery().count());
  }
}
```
## 调试单测
使用内存型H2数据库时，通过下面展示的方式可以很容易地在调试期间查看到Activiti的数据。这些截图用的是Eclipse，不过对于其他IDE来说，机制是一样的。  
假设我们在单测上添加了一个断点。在Eclipse中，双击代码左边的边框就可以达到：  
![Eclipse断点图](/images/activiti/api.test.debug.breakpoint.png)   
现在我们以调试的方式运行测试用例(右键测试类，选择*Run*->*JUnit test*)，测试流程就会在我们的断点那里停下来，这个时候我们就可以查看测试中的变量，就像图中的右上角的面板展示的那样：  
![debug界面图](/images/activiti/api.test.debug.view.png)  
要想查看Activiti的数据，打开*'Display'*窗口(如果这个窗口不在那，选择 Window→Show View→Other and select Display)并输入(代码会自动补全)**org.h2.tools.Server.createWebServer("-web").start()**
![debug Display图](/images/activiti/api.test.debug.start.h2.server.png)  
选择你输入的那一行，然后右键选择*Display*(或者使用快捷方式代替右键)  
![debug Display图2](/images/activiti/api.test.debug.start.h2.server.2.png)  
现在打开浏览器，并输入 http://localhost:8082，接着输入内存数据库的JDBC 连接(默认是**jdbc:h2:mem:activiti**)，然后点击 "Connect" 按钮。  
![debug login图](/images/activiti/api.test.debug.h2.login.png)  
这个时候你就可以查看Activiti的数据，然后通过它来了解你的单测是如何以及为何执行的。  
![debug表数据图](/images/activiti/api.test.debug.h2.tables.png)
## web 应用里的流程引擎
**ProcessEngine**是线程安全的类，所以它可以被多个线程共享。这表示，在web应用中流程引擎可以随着容器的启动被创建一次，然后跟着容器的销毁而关闭。  
下面的代码片段展示了如果写一个简单的**ServletContextListener**，让它跟着容器的Servlet环境一起创建和销毁。
```java
public class ProcessEnginesServletContextListener implements ServletContextListener {

  public void contextInitialized(ServletContextEvent servletContextEvent) {
    ProcessEngines.init();
  }

  public void contextDestroyed(ServletContextEvent servletContextEvent) {
    ProcessEngines.destroy();
  }

}
```
**contextInitialized**方法里将会执行**ProcessEngines.init()**，它会寻找类加载路径中的**activiti.cfg.xml**文件，然后根据它的配置创建**ProcessEngine**(很多jar中都有配置文件)。如果你的类加载路径中有很多资源文件，请确保它们的名称不同。当需要流程引擎的时候，可以使用如下方式拿到：
```java
ProcessEngines.getDefaultProcessEngine()
```
或者
```java
ProcessEngines.getProcessEngine("myName");
```
当然了，就像[配置]描述的那样，有很多其他方式创建流程引擎。  
上下文监听器中的**contextDestroyed**方法调用了**ProcessEngines.destroy()**，它会准确地关闭所有创建的流程引擎。