---
layout: post
title: Activiti 6.0 用户文档 - 第五章：集成Spring
categories: activiti
description: 流程引擎 Activiti 第五章：集成Spring
keywords: 流程引擎, activiti
---
# 集成 Spring
虽然你完全可以脱离Spring使用Activiti，但是我们本章还是要介绍下Activiti中一些非常不错的集成功能。
## ProcessEngineFactoryBean
可以将**ProcessEngine**当做一个普通的Spring bean来配置。集成Spring的切入点是类**org.activiti.spring.ProcessEngineFactoryBean**。这个bean持有一个可以创建流程引擎的配置。这就是说，在Spring中创建和配置所用到的属性跟文档中[配置章节](https://www.activiti.org/userguide/index.html#configuration)中展示的一样。集成Spring的配置和引擎bean看起来是下面这样的:
```xml
<bean id="processEngineConfiguration" class="org.activiti.spring.SpringProcessEngineConfiguration">
    ...
</bean>

<bean id="processEngine" class="org.activiti.spring.ProcessEngineFactoryBean">
  <property name="processEngineConfiguration" ref="processEngineConfiguration" />
</bean>
```
注意这里的**processEngineConfiguration**bean使用的是**org.activiti.spring.SpringProcessEngineConfiguration**。
## 事务
我们将逐步解释下源码中集成Spring的示例中**SpringTransactionIntegrationTest**。下面是案例中我们用的Spring的配置文件(你可以在SpringTransactionIntegrationTest-context.xml中找到)，它包括dataSource, transactionManager, processEngine 和流程引擎的各种服务。   
Activiti在内部使的**org.springframework.jdbc.datasource.TransactionAwareDataSourceProxy**来将包装过的数据源传给**SpringProcessEngineConfiguration**(使用属性"dataSource")。这样做是为了确保Spring事务和数据源中检测到的数据库连接能很好地协同工作。这说明你不需要自己配置数据源代理，虽然它也允许你将**TransactionAwareDataSourceProxy**传给**SpringProcessEngineConfiguration**。这种情况下，就不需要额外的封装了。  
**当你自己在Spring配置中声明一个*TransactionAwareDataSourceProxy*时，请确保不要将它用于已经获取到Spring事务的资源(例如：DataSourceTransactionManager 和 JPATransactionManager 就需要未代理的数据源)。
```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:tx="http://www.springframework.org/schema/tx"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans   http://www.springframework.org/schema/beans/spring-beans.xsd
                           http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-2.5.xsd
                           http://www.springframework.org/schema/tx      http://www.springframework.org/schema/tx/spring-tx-3.0.xsd">

  <bean id="dataSource" class="org.springframework.jdbc.datasource.SimpleDriverDataSource">
    <property name="driverClass" value="org.h2.Driver" />
    <property name="url" value="jdbc:h2:mem:activiti;DB_CLOSE_DELAY=1000" />
    <property name="username" value="sa" />
    <property name="password" value="" />
  </bean>

  <bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="dataSource" />
  </bean>

  <bean id="processEngineConfiguration" class="org.activiti.spring.SpringProcessEngineConfiguration">
    <property name="dataSource" ref="dataSource" />
    <property name="transactionManager" ref="transactionManager" />
    <property name="databaseSchemaUpdate" value="true" />
    <property name="asyncExecutorActivate" value="false" />
  </bean>

  <bean id="processEngine" class="org.activiti.spring.ProcessEngineFactoryBean">
    <property name="processEngineConfiguration" ref="processEngineConfiguration" />
  </bean>

  <bean id="repositoryService" factory-bean="processEngine" factory-method="getRepositoryService" />
  <bean id="runtimeService" factory-bean="processEngine" factory-method="getRuntimeService" />
  <bean id="taskService" factory-bean="processEngine" factory-method="getTaskService" />
  <bean id="historyService" factory-bean="processEngine" factory-method="getHistoryService" />
  <bean id="managementService" factory-bean="processEngine" factory-method="getManagementService" />

...
</beans>
```
该配置文件还包含我们将在此特定案例中使用的bean和配置：
```xml
<beans>
  ...
  <tx:annotation-driven transaction-manager="transactionManager"/>

  <bean id="userBean" class="org.activiti.spring.test.UserBean">
    <property name="runtimeService" ref="runtimeService" />
  </bean>

  <bean id="printer" class="org.activiti.spring.test.Printer" />

</beans>
```
首先，使用任意一种Spring的方式创建应用上下文。在这个案例中，你可以使用类路径下的XML资源来配置Spring应用上下文：
```java
ClassPathXmlApplicationContext applicationContext = new ClassPathXmlApplicationContext(
	"org/activiti/examples/spring/SpringTransactionIntegrationTest-context.xml");
````
或者如果是测试的话，可以用这个：
```java
	@ContextConfiguration("classpath:org/activiti/spring/test/transaction/SpringTransactionIntegrationTest-context.xml")
```
此时，我们就可以获取到服务的beans，然后调用它们的方法。ProcessEngineFactoryBean 将使用一个额外的拦截器为Activiti的服务方法上添加Propagation.REQUIRED事务机制。因此，我们可以像下面这样使用repositoryService 发布一个流程：
```java
RepositoryService repositoryService =
  (RepositoryService) applicationContext.getBean("repositoryService");
String deploymentId = repositoryService
  .createDeployment()
  .addClasspathResource("org/activiti/spring/test/hello.bpmn20.xml")
  .deploy()
  .getId();
```
另外一种around的方式也是可行的。在本例中，Spring事务将会包裹住 userBean.hello()，然后Activiti服务方法在调用的时候也会加入这个事务
```java
UserBean userBean = (UserBean) applicationContext.getBean("userBean");
userBean.hello();
```
UserBean长得像如下这样。别忘了，在Spring bean配置中我们已经将repositoryService注入给userBean
```java
public class UserBean {

  /** 通过Spring注入 */
  private RuntimeService runtimeService;

  @Transactional
  public void hello() {
    // 这里你可以做一些有事务属性的事情，它会跟startProcessInstanceByKey的事务一起加入到 Activiti RuntimeService的事务中
    runtimeService.startProcessInstanceByKey("helloProcess");
  }

  public void setRuntimeService(RuntimeService runtimeService) {
    this.runtimeService = runtimeService;
  }
}
```
## 表达式
使用ProcessEngineFactoryBean时，BPMN流程中所有的[表达式](https://www.activiti.org/userguide/index.html#apiExpressions)默认都会看到所有的Spring bean。我们可以通过配置来限制表达式可以看到的bean，甚至可以让它一个bean也看不到。下面的案例中，就是暴露一个bean(printer)，表达式可以通过 key "printer"来获取它。**如果不想暴露任何bean，那就给SpringProcessEngineConfiguration的*bean*属性传递一个空的list。如果你不设置*bean*属性，就是暴露所有的bean给上下文**
```xml
<bean id="processEngineConfiguration" class="org.activiti.spring.SpringProcessEngineConfiguration">
  ...
  <property name="beans">
    <map>
      <entry key="printer" value-ref="printer" />
    </map>
  </property>
</bean>

<bean id="printer" class="org.activiti.examples.spring.Printer" />
```
此时就可以在表达式中使用暴露的bean。比如，SpringTransactionIntegrationTest *hello.bpmn20.xml*就展示了UEL的方法表达式是如何调用bean的方法：
```xml
<definitions id="definitions">

  <process id="helloProcess">

    <startEvent id="start" />
    <sequenceFlow id="flow1" sourceRef="start" targetRef="print" />

    <serviceTask id="print" activiti:expression="#{printer.printMessage()}" />
    <sequenceFlow id="flow2" sourceRef="print" targetRef="end" />

    <endEvent id="end" />

  </process>

</definitions>
```
*Printer*长这样：
```java
public class Printer {

  public void printMessage() {
    System.out.println("hello world");
  }
}
```
Bean的配置(上面也展示过了)长这样：
```xml
<beans>
  ...

  <bean id="printer" class="org.activiti.examples.spring.Printer" />

</beans>
```
## 资源自动发布
