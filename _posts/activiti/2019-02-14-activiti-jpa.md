---
layout: post
title: Activiti 6.0 用户文档 - 第十章：JPA
categories: activiti
description: 流程引擎 Activiti 第十章：JPA
keywords: 流程引擎, activiti
---
# JPA
你可以使用 JPA 实体作为流程变量，来完成下面这些事：
- 根据流程变量更新现有JPA实体。这些流程变量可以在用户任务中填写，也可以在服务任务中生成
- 复用现有的域模型，而无需编写显式服务来查找和更新值
- 让网关基于现有的实体属性来做决策

## 先决条件
仅支持符合以下条件的实体：
- 使用JPA注解的实体，系统支持字段和属性访问。也可以使用映射的父类
- 实体必须有一个由 **@Id** 注解的主键，而且不支持联合主键(**EmbeddedId**和**IdClass**)。Id 字段/属性 可以是JPA要求的任意类型：基本类型和它们的包装类(boolean 除外)，**String, BigInteger, BigDecimal, java.util.Date 和 java.sql.Date**.
## 配置
要想使用JPA实体，引擎必须持有 **EntityManagerFactory**的引用。可以通过配置这个引用或者提供持久化单元名称来达到这个目的。作为变量的JPA实体可以被自动检测并作相应处理。  
我们通过使用jpaPersistenceUnitName方式配置举例：
```xml
<bean id="processEngineConfiguration"
  class="org.activiti.engine.impl.cfg.StandaloneInMemProcessEngineConfiguration">

<!-- Database configurations -->
<property name="databaseSchemaUpdate" value="true" />
<property name="jdbcUrl" value="jdbc:h2:mem:JpaVariableTest;DB_CLOSE_DELAY=1000" />

<property name="jpaPersistenceUnitName" value="activiti-jpa-pu" />
<property name="jpaHandleTransaction" value="true" />
<property name="jpaCloseEntityManager" value="true" />

<!-- job executor configurations -->
<property name="jobExecutorActivate" value="false" />

<!-- mail server configurations -->
<property name="mailServerPort" value="5025" />
</bean>
```
下一个是使用我们自己定义的 **EntityManagerFactory** 的方式(本例中，是作为一个 open-jpa 管理器)举例。注意，代码片段中值保留了示例中需要的bean，其他的都省略了。完整的使用 open-jpa 实体管理器的可运行示例放在 activiti-spring-examples项目中(**/activiti-spring/src/test/java/org/activiti/spring/test/jpa/JPASpringTest.java**)
```xml
<bean id="entityManagerFactory" class="org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean">
  <property name="persistenceUnitManager" ref="pum"/>
  <property name="jpaVendorAdapter">
    <bean class="org.springframework.orm.jpa.vendor.OpenJpaVendorAdapter">
      <property name="databasePlatform" value="org.apache.openjpa.jdbc.sql.H2Dictionary" />
    </bean>
  </property>
</bean>

<bean id="processEngineConfiguration" class="org.activiti.spring.SpringProcessEngineConfiguration">
  <property name="dataSource" ref="dataSource" />
  <property name="transactionManager" ref="transactionManager" />
  <property name="databaseSchemaUpdate" value="true" />
  <property name="jpaEntityManagerFactory" ref="entityManagerFactory" />
  <property name="jpaHandleTransaction" value="true" />
  <property name="jpaCloseEntityManager" value="true" />
  <property name="jobExecutorActivate" value="false" />
</bean>
```
使用代码创建引擎也可以达到相同的配置目的：
```java
ProcessEngine processEngine = ProcessEngineConfiguration
.createProcessEngineConfigurationFromResourceDefault()
.setJpaPersistenceUnitName("activiti-pu")
.buildProcessEngine();
```
配置属性：
- jpaPersistenceUnitName：要使用的持久化单元的名称(确定该持久化单元在类路径下能找到。在本例中，默认的位置是**/META-INF/persistence.xml**)。既可以使用**jpaEntityManagerFactory**或**jpaPersistenceUnitName**
- jpaEntityManagerFactory：指向**javax.persistence.EntityManagerFactory**实现bean的引用，用来加载 Entitis 并写入更新。使用 *jpaEntityManagerFactory*或*jpaPersistenceUnitName*
- jpaHandleTransaction：标识引擎可以在 *EntityManager*实体上提交/回滚事务。当使用了*Java Transaction API(JTA)*时，该属性要设置成false。
- jpaCloseEntityManager：标识引擎需要关闭从**EntityManagerFactory**获取到的**EntityManager**。当 *EntityManager* 是容器托管状态时(比如使用不支持单事务的 Extended Persistence Context时)，该属性要设置成false
## 应用
### 简单的例子
在Activiti源码的 JPAVariableTest 类中有使用JPA 变量的示例。我们可以一步一步解释**JPAVariableTest.testUpdateJPAEntityValues**介绍。  
首先，我们为基于 **META-INF/persistence.xml**的持久化单元创建一个 *EntityManagerFactory*。它包括了持久化单元需要的类和一些供应商特定配置。  
我们在测试用例中使用了一个简单的实体，它有一个id和一个 **String**类型的变量。在运行测试前，我们需要创建一个这样的实体：
```java
@Entity(name = "JPA_ENTITY_FIELD")
public class FieldAccessJPAEntity {

  @Id
  @Column(name = "ID_")
  private Long id;

  private String value;

  public FieldAccessJPAEntity() {
    // Empty constructor needed for JPA
  }

  public Long getId() {
    return id;
  }

  public void setId(Long id) {
    this.id = id;
  }

  public String getValue() {
    return value;
  }

  public void setValue(String value) {
    this.value = value;
  }
}
```
我们启动一个流程定义，将该实体作为一个流程变量加进去。该实体跟其他变量一起保存在引擎的数据库中。当下次需要这个变量时，**EntityManager*将通过存储的类和Id加载到这个实体。
```java
Map<String, Object> variables = new HashMap<String, Object>();
variables.put("entityToUpdate", entityToUpdate);

ProcessInstance processInstance = runtimeService.startProcessInstanceByKey("UpdateJPAValuesProcess", variables);
```
我们流程定义的第一个节点包含一个 **serviceTask**，它会执行 **entityToUpdate**的**setValue**方法。该节点会解析之前流程实例启动时设置的JPA变量，然后被当前流程的上下文相关的**EntityManager**加载。  
```xml
<serviceTask id='theTask' name='updateJPAEntityTask'
  activiti:expression="${entityToUpdate.setValue('updatedValue')}" />
```
按照流程定义，当该服务任务完成时，流程实例在一个用户任务中等待，然后我们就可以看到它了。此时，**EntityManager** 已经刷新，对实体的更新也被写入数据库。当我们获取 **entityToUpdate**变量的值，它会重新加载，然后我们会看到该变量的值已经从 **'value'** 变成了 **'updatedValue'**
```java
// Servicetask in process 'UpdateJPAValuesProcess' should have set value on entityToUpdate.
Object updatedEntity = runtimeService.getVariable(processInstance.getId(), "entityToUpdate");
assertTrue(updatedEntity instanceof FieldAccessJPAEntity);
assertEquals("updatedValue", ((FieldAccessJPAEntity)updatedEntity).getValue());
```
### 查询JPA流程变量
你可以查询将某个JPA实体作为变量的**ProcessInstance** 和 **Execution**。 **注意，对于 ProcessInstanceQuery和ExecutionQuery ，JPA-Entities仅支持 variableValueEquals(name, entity)** 。不支持**variableValueNotEquals, variableValueGreaterThan, variableValueGreaterThanOrEqual, variableValueLessThan 和 variableValueLessThanOrEqual**中的方法，如果传给这些方法一个 JPA实体，系统会抛一个**ActivitiException**异常
```java
ProcessInstance result = runtimeService.createProcessInstanceQuery()
    .variableValueEquals("entityToQuery", entityToQuery).singleResult();
```
### Spring bean和JPA的一些高级示例
在 **activiti-spring-examples** 有一个高级示例**JPASpringTest**，它实现了下面的简单用例：
- 已经有了一个使用JPA实体的Spring bean，提供保存贷款请求的功能
- 通过该 bean，我们可以使用Activiti操作实体，然后将它们当做流程变量使用。我们按照下面的步骤定义流程：
  - 当启动一个任务时(比如可以使用一个表单启动),服务任务使用 **LoanRequestBean**里的变量创建贷款申请.创建好的实体类作为一个变量被保存,使用的是**activiti:resultVariable** 将结果作为一个表达式保存下来
  - 用户任务中有一个 bean 叫 **approvedByManager**，保存着经理接到这个请求选择通过/不通过的结果
  - 服务任务更新该贷款请求，以便实体与流程同步
  - 排他网关根据实体属性 **approved** 来决定走哪条通道：同意就直接流程结束，否则就执行剩下的流程（发送拒绝邮件），这样用户就会收到一风拒绝信。

请注意，因为这只是一个单测用例，所以流程里面并没不包含任何表单。  
![JPA流程测试图](/images/activiti/jpa.spring.example.process.png)
```xml
<?xml version="1.0" encoding="UTF-8"?>
<definitions id="taskAssigneeExample"
  xmlns="http://www.omg.org/spec/BPMN/20100524/MODEL"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xmlns:activiti="http://activiti.org/bpmn"
  targetNamespace="org.activiti.examples">

  <process id="LoanRequestProcess" name="Process creating and handling loan request">
    <startEvent id='theStart' />
    <sequenceFlow id='flow1' sourceRef='theStart' targetRef='createLoanRequest' />

    <serviceTask id='createLoanRequest' name='Create loan request'
      activiti:expression="${loanRequestBean.newLoanRequest(customerName, amount)}"
      activiti:resultVariable="loanRequest"/>
    <sequenceFlow id='flow2' sourceRef='createLoanRequest' targetRef='approveTask' />

    <userTask id="approveTask" name="Approve request" />
    <sequenceFlow id='flow3' sourceRef='approveTask' targetRef='approveOrDissaprove' />

    <serviceTask id='approveOrDissaprove' name='Store decision'
      activiti:expression="${loanRequest.setApproved(approvedByManager)}" />
    <sequenceFlow id='flow4' sourceRef='approveOrDissaprove' targetRef='exclusiveGw' />

    <exclusiveGateway id="exclusiveGw" name="Exclusive Gateway approval" />
    <sequenceFlow id="endFlow1" sourceRef="exclusiveGw" targetRef="theEnd">
      <conditionExpression xsi:type="tFormalExpression">${loanRequest.approved}</conditionExpression>
    </sequenceFlow>
    <sequenceFlow id="endFlow2" sourceRef="exclusiveGw" targetRef="sendRejectionLetter">
      <conditionExpression xsi:type="tFormalExpression">${!loanRequest.approved}</conditionExpression>
    </sequenceFlow>

    <userTask id="sendRejectionLetter" name="Send rejection letter" />
    <sequenceFlow id='flow5' sourceRef='sendRejectionLetter' targetRef='theOtherEnd' />

    <endEvent id='theEnd' />
    <endEvent id='theOtherEnd' />
  </process>

</definitions>
```
虽然上面的例子很简单，但是我们已经能看出JPA跟Spring和参数化方法结合使用时功能还是很强大的。流程基本上不用java代码了（当然了，除了Spring的bean），从而大幅提升开发效率