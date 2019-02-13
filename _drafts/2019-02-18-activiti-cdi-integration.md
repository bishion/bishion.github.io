---
layout: post
title: Activiti 6.0 用户文档 - 第十四章：CDI 集成
categories: activiti
description: 流程引擎 Activiti 第十四章：CDI 集成
keywords: 流程引擎, activiti
---
# CDI 集成
activiti-cdi 模块同时使用了 Activiti 的可配置性和cdi的可扩展性，它最突出的特性如下：
- 支持@BusinessProcessScoped注解的bean（Cdi 生成的生命周期绑定到流程实例上的bean）
- 可以从流程中解析 Cdi bean（包括EJB）的自定义EI解析器
- 使用注解对流程实例的声明式控制
- Activiti 与 cdi 事件总线相连
- 适用于 Java EE和Java SE，还有Spring
- 支持单元测试

```xml
<dependency>
	<groupId>org.activiti</groupId>
	<artifactId>activiti-cdi</artifactId>
	<version>5.x</version>
</dependency>
```
## 设置activiti-cdi
Activiti cdi 可以在各种环境中配置。这一章，我们将对这些配置选项做一个简单介绍。
### 查找 Process Engine
cdi 扩展需要能访问到 ProcessEngine，因此，系统在运行的时候会去查找 **org.activiti.cdi.spi.ProcessEngineLookup** 的实现。cdi 模块自带了一个默认的实现，叫做 **org.activiti.cdi.impl.LocalProcessEngineLookup**，该实现使用 **ProcessEngines**-Utility 类来查找 ProcessEngine，在默认的配置中，**ProcessEngines#NAME_DEFAULT** 就是用来做这个的。该类可以被继承，做一些自定义操作。注意：我们还需要在类路径下配置一个 **activiti.cfg.xml**。  
Activiti cdi 使用 java.uti.ServitLoader SPI 解析 **org.activiti.cdi.spi.ProcessEngineLookup** 的实例。为了给该接口提供一个自定义的实现，我们需要为我们的发布新建一个文件：**META-INF/services/org.activiti.cdi.spi.ProcessEngineLookup**，里面指定实现类的全路径名称。
> 如果你不提供自定义的 **org.activiti.cdi.spi.ProcessEngineLookup**，Activiti 默认使用 **LocalProcessEngineLookup**。这样你就只需要在类路径下提供一个 activiti.cfg.xml就可以（参考下文）
### 配置 Process Engine
用户选择的 ProcessEngineLookup策略（参考上一节）决定了配置。这里，我们将重点放在与LocalProcessEngineLookup结合使用的配置选项上，它需要我们在类路径上提供 Spring activiti.cfg.xml文件。  
Activiti 提供不同的ProcessEngineConfiguration实现，主要依赖于底层的事务管理策略。activiti-cdi 模块并不关心事务，这意味着我们可以使用任何事务管理策略(甚至是Spring 事务)。方便起见，cdi模块提供了两个自定义的ProcessEngineConfiguration实现：
- org.activiti.cdi.CdiJtaProcessEngineConfiguration：activiti JtaProcessEngineConfiguration的子类，如果Activiti 需要用到JTA 托管的事务，就需要用到这个类
- org.activiti.cdi.CdiStandaloneProcessEngineConfiguration：activiti StandaloneProcessEngineConfiguration 的子类，如果Activiti 需要用到纯JDBC事务，就需要用到该类。下面是 JBoss 7下的activiti.cfg.xml文件示例：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

	<!-- lookup the JTA-Transaction manager -->
	<bean id="transactionManager" class="org.springframework.jndi.JndiObjectFactoryBean">
		<property name="jndiName" value="java:jboss/TransactionManager"></property>
		<property name="resourceRef" value="true" />
	</bean>

	<!-- process engine configuration -->
	<bean id="processEngineConfiguration"
		class="org.activiti.cdi.CdiJtaProcessEngineConfiguration">
		<!-- lookup the default Jboss datasource -->
		<property name="dataSourceJndiName" value="java:jboss/datasources/ExampleDS" />
		<property name="databaseType" value="h2" />
		<property name="transactionManager" ref="transactionManager" />
		<!-- using externally managed transactions -->
		<property name="transactionsExternallyManaged" value="true" />
		<property name="databaseSchemaUpdate" value="true" />
	</bean>
</beans>
```
如果使用的是 Glassfish3.1.1，该文件就应该是下面示例（假设已经正确配置名为jdbc/activiti的数据源）：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

	<!-- lookup the JTA-Transaction manager -->
	<bean id="transactionManager" class="org.springframework.jndi.JndiObjectFactoryBean">
		<property name="jndiName" value="java:appserver/TransactionManager"></property>
		<property name="resourceRef" value="true" />
	</bean>

	<!-- process engine configuration -->
	<bean id="processEngineConfiguration"
		class="org.activiti.cdi.CdiJtaProcessEngineConfiguration">
		<property name="dataSourceJndiName" value="jdbc/activiti" />
		<property name="transactionManager" ref="transactionManager" />
		<!-- using externally managed transactions -->
		<property name="transactionsExternallyManaged" value="true" />
		<property name="databaseSchemaUpdate" value="true" />
	</bean>
</beans>
```
注意上面的配置需要"spring-context"模块：
```xml
<dependency>
	<groupId>org.springframework</groupId>
	<artifactId>spring-context</artifactId>
	<version>3.0.3.RELEASE</version>
</dependency>
```
在JavaSE中的配置跟[创建一个ProcessEngine](https://www.activiti.org/userguide/#configuration)章节提供的示例差不多，只是需要用"StandaloneProcessEngineConfiguration"替换"CdiStandaloneProcessEngineConfiguration"。
### 发布流程
使用基础的 activiti-api(**RepositoryService**)就可以发布流程。此外，activiti-cdi还可以根据类路径下的 **processes.xml** 中列出的流程对它们进行自动发布。下面是 processes.xml文件的示例：
```xml
<?xml version="1.0" encoding="utf-8" ?>
<!-- list the processes to be deployed -->
<processes>
	<process resource="diagrams/myProcess.bpmn20.xml" />
	<process resource="diagrams/myOtherProcess.bpmn20.xml" />
</processes>
```
#### 使用CDI执行上下文流程
在本小节，我们简单介绍下Activiti cdi 扩展使用的上下文流程执行模块。一个BPMN业务流程一般都是长时间运行，包括用户和系统任务。在运行时，一个流程被用户或者应用逻辑分成一组独立的工作单元。在activiti-cdi中，流程实例可以跟cdi工作区关联，该关联表示工作单元。如果一个工作单元很复杂，这种处理就非常有用了，比如UserTask的实例是不同形式的复杂序列，并且在交互期间还要保持"非流程范围"的状态。  
在默认配置中，流程实例之间保持"最大"活动范围的关联，随着会话启动，如果会话的上下文不活动了，就会回滚到请求状态。
### 将会话与流程实例相关联
当解析 @BusinessProcessScoped或者注入流程变量时，我们需要一个激活的cdi 范围与流程实例相关联。Activiti-cdi 提供 **org.activiti.cdi.BusinessProcess** bean来管理该关联，最突出的是：
- *startProcessBy(…​)*与 Activiti **RuntimeService**暴露的方法对应，它允许启动和关联一个业务流程
- **resumeProcessById(String processInstanceId)**允许关联一个已知id的流程实例
- **resumeTaskById(String taskId)** 允许关联指定id的任务（和通过扩展得到的相应的流程实例）

一旦一个工作单元（比如一个UserTask）完成了，**completeTask()**方法就会被调用，然后与流程实例终止会话/请求。这会通知Activiti当前任务已经完成，然后让流程实例继续执行。  
注意 **BusinessProcess** bean 是一个被 **@Named** 注解的bean，这表示我们可以使用表达式语言调用它暴露的方法，比如通过 JSF页面调用。下面的 JSF2 代码片段表示开启一个新的会话，然后将它与一个用户任务实例关联，任务的id被当做一个请求参数传递(比如：**pageName.jsf?taskId=XX**)：
```xml
<f:metadata>
<f:viewParam name="taskId" />
<f:event type="preRenderView" listener="#{businessProcess.startTask(taskId, true)}" />
</f:metadata>
```
### 声明式控制流程
Activiti-cdi 使用注解声明启动流程实例和完成任务。**@org.activiti.cdi.annotation.StartProcess**注解允许我们通过 "key"或者"name"来启动一个流程实例。注意，流程实例是在被注解的方法返回*之后*才启动的。比如：
```java
@StartProcess("authorizeBusinessTripRequest")
public String submitRequest(BusinessTripRequest request) {
	// do some work
	return "success";
}
```
按照Activiti的配置，被注解的方法代码和流程实例的启动在同一个事务中关联。**@org.activiti.cdi.annotation.CompleteTask**注解也是以相同的模式工作：
```java
@CompleteTask(endConversation=false)
public String authorizeBusinessTrip() {
	// do some work
	return "success";
}
```
**@CompleteTask**注解可以终止当前会话。默认情况下，Activiti返回时，会话就终止了。我们可以禁用会话终止，就像上面例子中展示的那样。
### 在流程中引用Bean
Activiti-cdi 使用一个自定义的解析器，将 CDI的bean暴露给Activiti EI。这让我们可以在流程中引用bean：
```xml
<userTask id="authorizeBusinessTrip" name="Authorize Business Trip"
			activiti:assignee="#{authorizingManager.account.username}" />
```
其中 "authorizingManager" 可以是生产者方法提供的一个bean：
```java
@Inject	@ProcessVariable Object businessTripRequesterUsername;

@Produces
@Named
public Employee authorizingManager() {
	TypedQuery<Employee> query = entityManager.createQuery("SELECT e FROM Employee e WHERE e.account.username='"
		+ businessTripRequesterUsername + "'", Employee.class);
	Employee employee = query.getSingleResult();
	return employee.getManager();
}
```
我们可以使用该特性在一个服务任务中用 **activiti:expression="myEjb.method()"** 表达式调用EJB的一个业务方法。注意，它还需要在 **MyEjb**类上加一个 **@Named**注解
### 使用@BusinessProcessScoped 注解的bean
使用activiti-cdi可以将bean的生命周期与流程实例绑定。为此，系统提供了名为BusinessProcessContext的自定义上下文实现。BusinessProcessScoped的实例被作为流程变量在当前的流程实例中存储。BusinessProcessScoped 注解的bean需要是PassivationCapable(可钝化的)，比如可序列化的。下面就是一个流程范围的bean实例：
```java
@Named
@BusinessProcessScoped
public class BusinessTripRequest implements Serializable {
	private static final long serialVersionUID = 1L;
	private String startDate;
	private String endDate;
	// ...
}
```
有时，在缺少流程实例时(比如在流程启动之前)，我们需要使用流程范围的bean。如果当前没有活动的流程实例，BusinessProcessScoped 的实例被临时存储在本地范围内(比如会话或者请求中，这取决于上下文)。如果该作用域接下来关联上了一个业务流程实例，该bean就会刷入到流程实例中。
### 注入流程实例
流程实例还可以被注入。Activiti-CDI支持下面几种：
- 使用 **@Inject \[additional qualifiers\] Type fieldName** 注入类型安全的 **BusinessProcessScoped**
- 使用 **ProcessVariable(name?)** 注入类型不安全的其他流程变量：
```java
@Inject @ProcessVariable Object accountNumber;
@Inject @ProcessVariable("accountNumber") Object account
```
为了使用EL引用流程变量，我们可以更简单的选择：
- **@Named @BusinessProcessScoped**注解的bean可以直接引用
- 使用 **ProcessVariables**bean可以引用其他流程变量
```javascript
#{processVariables['accountNumber']}
```