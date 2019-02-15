---
layout: post
title: Activiti 6.0 用户文档 - 第十七章：工具
categories: activiti
description: 流程引擎 Activiti 第十七章：工具
keywords: 流程引擎, activiti
---
# 工具
## JMX
### 介绍
我们还可以使用 Java 管理扩展(Java Management Extensions,JMX) 来连接Activiti引擎，这样就可以查询信息或者修改它的行为。任何标准的JMX客户端都可以做到这一点。不用写一行代码，我们就可以使用JMX做很多事情，比如启用和禁用Job Executor，发布和删除新的流程定义文件。
### 快速开始
默认情况下，JMX是禁用状态。如果想使用默认配置启动JMX，你只需要使用Maven或者其他工具将activiti-jmx包引用到你的项目中去。如果你使用的Maven，你可以将合适的依赖加到你的pom.xml中去：
```xml
<dependency>
  <groupId>org.activiti</groupId>
  <artifactId>activiti-jmx</artifactId>
  <version>latest.version</version>
</dependency>
```
添加完依赖，在流程引擎创建完之后，JMX连接就可以使用了。你只需要执行JDK包中的jconsole就可以。在本地的进程列表中，你可以看到JVM已经包含了 Activiti。如果因为某些原因，在 "本地进程"区域没有找到对应的JVM，你可以在 "远程进程"菜单下使用下面这个URL连接：
```properties
service:jmx:rmi:///jndi/rmi://localhost:1099/jmxrmi/activiti
```
你可以在你的日志文件中找到准确的本地URL。连接之后，你可以看到标准的JVM统计和MBeans。你可以在右边的面板选择 "MBeans"->"org.activiti.jmx.Mbeans"，这样就可以看到Activiti 的MBeans。你可以选择任何一个 MBean，来查询对应的信息并修改配置。下面的截图展示了jconsole长什么样子[此处缺失截图-译者注]：  
不仅仅是jconsole，任何JMX客户端都可以访问MBeans。很多数据中心监控工具都可以连接到JMX MBeans。
### 属性和方法
下面是一组现在可以用的属性和方法列表。如果需要的话，该列表在未来的版本中可能会扩展。
|MBean|类型|名称|描述|
|-----|---|----|---|
|ProcessDefinitionsMBean|属性|processDefinitions|发布流程定义需要的属性列表**Id, Name, Version, IsSuspended**|
||属性|deployments|当前发布用到的属性 **Id, Name, TenantId**|
||方法|getProcessDefinitionById(String id)|根据id查询流程定义中的属性 **Id, Name, Version and IsSuspended**|
||方法|deleteDeployment(String id)|根据**id**删除发布|
||方法|suspendProcessDefinitionById(String id)|根据 **id**挂起一个流程定义|
||方法|activatedProcessDefinitionById(String id)|根据 **id**激活一个流程定义|
||方法|suspendProcessDefinitionByKey(String id)|根据 **key**挂起一个流程定义|
||方法|activatedProcessDefinitionByKey(String id)|根据 **key**激活一个流程定义|
||方法|deployProcessDefinition(String resourceName, String processDefinitionFile)|根据文件发布一个流程定义|
|JobExecutorMBean|属性|isJobExecutorActivated|如果作业执行程序被激活，则返回true，否则返回false|
||方法|setJobExecutorActivate(Boolean active)|根据参数激活或者取消激活作业执行程序|
### 配置
JMX中有很多很多默认配置，这让经常用的配置可以轻松发布。但是，我们也可以很轻松地使用代码或者配置文件修改默认配置。下面就是一个在配置文件中修改默认配置的示例：
```xml
<bean id="processEngineConfiguration" class="...SomeProcessEngineConfigurationClass">
  ...
  <property name="configurators">
    <list>
	  <bean class="org.activiti.management.jmx.JMXConfigurator">

	    <property name="connectorPort" value="1912" />
        <property name="serviceUrlPath" value="/jmxrmi/activiti" />

		...
      </bean>
    </list>
  </property>
</bean>
```
下面的表格就显示了可以配置的参数以及它们的默认值：
|名称|默认值|描述|
|---|-----|----|
|disabled|false|如果设置了true，即使在依赖中有了jmx的包，JMX也不会启动|
|domain|org.activiti.jmx.Mbeans|MBean的 Domain|
|createConnector|true|如果是true，给启动的 MbeanServer创建一个连接|
|MBeanDomain|DefaultDomain|MBean server 的 domain|
|registryPort|1099|服务 URL 中的注册端口|
|serviceUrlPath|/jmxrmi/activiti|服务URL|
|connectorPort|-1|如果大于0，将在服务URL中显示为连接器端口|
### JMX 服务URL
JMX 服务的 URL 有如下的格式：
```properties
service:jmx:rmi://<hostName>:<connectorPort>/jndi/rmi://<hostName>:<registryPort>/<serviceUrlPath>
```
**hostName** 自动设置为机器的网络名称。可以配置**connecorPort，registryPort 和 serviceUrlPath** 的值。  
如果 **connectionPort** 比0小，那么服务URL中的相应部分就省略掉，如下所示：
```properties
service:jmx:rmi:///jndi/rmi://:<hostname>:<registryPort>/<serviceUrlPath>
```
## Maven 脚手架
### 创建测试用例
开发过程中，在动手开发实际逻辑之前，创建一个小的测试用例来测试一个想法或者特性是很有帮助的，它有助于将被测试的东西隔离。对于传递bug报告和功能请求，Junit 单测用例也是一个首选工具。将测试用例附加到bug报告或jira问题上，可以大大缩短其修复时间。   
为了加快测试用例的创建时间，我们提供了一个maven脚手架。通过使用该脚手架，你可以很快地创建一个标准的测试用例。该脚手架必须已经在本地maven库中，如果没有，你可以在 **tooling/archtypes** 目录中执行 **mvn install**，从而将该脚手架安装到你本地maven 仓库目录。  
下面的就名就可以创建一个单测项目：
```shell
mvn archetype:generate \
-DarchetypeGroupId=org.activiti \
-DarchetypeArtifactId=activiti-archetype-unittest \
-DarchetypeVersion=<current version> \
-DgroupId=org.myGroup \
-DartifactId=myArtifact
```
上面的每一个参数都会在下表中解释：
|行|参数|解释|
|-|----|---|
|1|archetypeGroupId|脚手架中的groupId，必须是 **org.activiti**|
|2|archetypeArtifactId|脚手架中的 artifactId，比如是 **activiti-archetype-unittest**|
|3|archetypeVersion|在测试项目中使用到的Activiti版本|
|4|groupId|生成项目的 groupId|
|5|artifactId|生成项目的 artifactId|
生成的项目目录机构可能是下面这样的：
```shell
.
├── pom.xml
└── src
    └── test
        ├── java
        │   └── org
        │       └── myGroup
        │           └── MyUnitTest.java
        └── resources
            ├── activiti.cfg.xml
            ├── log4j.properties
            └── org
                └── myGroup
                    └── my-process.bpmn20.xml
```
你可以修改或新增单测用例和它相应的流程模型。如果你使用该项目来验证bug或者某个特性，单测初始化时应该是失败的。在对应的bug已经修复，特性已经实现之后，单测就会运行成功。在发送它之前，请务必确保项目已经执行 **mvn clean** clean过了。  


原文档最后更新时间 2017-05-25 10:52:43