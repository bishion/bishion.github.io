---
layout: post
title: spring boot 集成 camunda
categories: camunda
description: 流程引擎 Camunda 入门
keywords: 流程引擎, camunda
---

## 需求
最近要启动一个故障自动化处理的项目。即在系统收到故障告警后，按照一定的规则进行故障的筛查和处理。

## 思考
乍一看，这个需求让人一头雾水：
1. 一定的规则是啥意思？
2. 故障筛查和处理怎么做？
3. 我怎么知道每次的故障是啥？
4. ？？？

## 分析
冷静下来之后，我们发现，如果忽略故障的具体筛查和处理过程，那么这个系统核心的问题并不复杂：流程处理。  
收到故障信息后，系统按照故障的特征，去匹配某一个流程，然后通过流程的每个节点去一步步做对应的操作。  
那么问题来了，这个流程用什么实现呢？  
答案很简单，流程引擎。

## 流程引擎
对于流程引擎，我们一开始是拒绝的，毕竟流程引擎对于目前这个场景来说有点重，规则引擎或者自己走配置文件就可以解决。  
但是考虑了很久，还是打算用流程引擎，原因如下：  
1. 它能解决我们目前关于规则处理的需求
2. 很多流程引擎工具都有可视化配置工具
3. 系统将来可能用到审批功能
4. 故障处理将来可能会引入人工干预

## Camunda
关于 Activiti、Camunda 、Flowable的爱恨纠葛，我们这里就不做分析，网上已经有很多了。  
这三个组件都可以满足我们当前要求，但是因为历史原因，我们选择了Camunda。  
本文主要介绍我们使用 Camunda 过程中遇到的一些问题以及如何解决的。

## 接入
### pom.xml
```xml
<!-- camunda start -->
<dependency>
	<groupId>org.camunda.bpm</groupId>
	<artifactId>camunda-engine-spring</artifactId>
	<version>7.11.0</version>
</dependency>
<!-- 该配置不是必须，如果没有下文中某个报错信息，可以不作配置 -->
<dependency>
	<groupId>org.mybatis</groupId>
	<artifactId>mybatis</artifactId>
	<version>3.5.2</version>
</dependency>
<!-- camunda end -->
```

### spring 配置
```java
@Configuration
public class CamundaConfig {

    /**
     * spring 流程引擎配置，做如下操作：
     * 1. 关联数据源
     * 2. 关联事务管理器
     * 3. 配置启动时是否检查数据库
     * 4. 配置流程历史信息保留级别。
     * @param dataSource 数据源
     * @return spring 流程配置器
     */
    @Bean
    public SpringProcessEngineConfiguration processEngineConfiguration(DataSource dataSource) {
        SpringProcessEngineConfiguration config = new SpringProcessEngineConfiguration();

        config.setDataSource(dataSource);
        config.setTransactionManager(new DataSourceTransactionManager(dataSource));

        config.setDatabaseSchemaUpdate("true");
        config.setHistory("audit");
        config.setJobExecutorActivate(true);

        return config;
    }
    /**
     * 配置流程引擎实例
     * @param springProcessEngineConfiguration spring流程引擎配置
     * @return 流程引擎实例
     */
    @Bean
    public ProcessEngineFactoryBean processEngine(SpringProcessEngineConfiguration springProcessEngineConfiguration) {
        ProcessEngineFactoryBean factoryBean = new ProcessEngineFactoryBean();
        factoryBean.setProcessEngineConfiguration(springProcessEngineConfiguration);
        return factoryBean;
    }

    /**
     * 流程历史服务，查询曾经运行过的流程实例信息
     * @param processEngine 流程引擎实例
     * @return 流程历史服务
     */
    @Bean
    public HistoryService historyService(ProcessEngine processEngine) {
        return processEngine.getHistoryService();
    }
    /**
     * 流程存储服务，
     * @param processEngine 流程引擎实例
     * @return 流程历史服务
     */
    @Bean
    public RepositoryService repositoryService(ProcessEngine processEngine) {
        return processEngine.getRepositoryService();
    }

    /**
     * 流程部署器，在部署流程时使用。
     * @param repositoryService 流程存储服务
     * @return 流程历史服务
     */
    @Bean
    public DeploymentBuilder deploymentBuilder(RepositoryService repositoryService) {
        return repositoryService.createDeployment();
    }
    /**
     * 流程运行时服务，在启动流程实例、查询正在运行的流程实例等场景使用。
     * @param processEngine 流程存储服务
     * @return 流程历史服务
     */
    @Bean
    public RuntimeService runtimeService(ProcessEngine processEngine) {
        return processEngine.getRuntimeService();
    }
}
```

### 部署并启动一个流程
部署一个流程有四种方式：
1. 流程引擎配置在字符串中：deploymentBuilder.addString(**)
1. 流程引擎配置在项目路径中：deploymentBuilder.addClasspathResource(**);
1. 流程引擎配置在文件流中：deploymentBuilder.addInputStream(**);
1. 流程引擎配置在zip包中：deploymentBuilder.addZipInputStream(**)

下面以流程放在字符串的方式举例，毕竟流程自动部署的场景中，我第一个想到的就是字符串的方式：
```java
deploymentBuilder.addString("myfirstBpmn.bpmn",bpmnTest).deploy()
/**
 * @param bpmnKey 流程引擎的 id，对应 bpmn 配置的  <bpmn:process id="bpmnKey">
 * @param businessKey 业务关键字，这个可以算是流程引擎的扩展字段，用来跟业务关联。你可以通过它来查询对应的业务信息 
 * @param params 流程引擎启动时的参数
 */
runtimeService.startProcessInstanceByKey(bpmnKey, businessKey, params);
```

### 流程节点要执行某个calss
```java
public class MyTaskService implements JavaDelegate {
    @Override
    public void execute(DelegateExecution execution) {
        // 你的业务逻辑....
    }
}
```

### 出错与解决
#### getLanguageDriver 方法不存在
```properties
***************************
APPLICATION FAILED TO START
***************************

Description:

An attempt was made to call a method that does not exist. The attempt was made from the following location:

    com.baomidou.mybatisplus.core.MybatisMapperAnnotationBuilder.getLanguageDriver(MybatisMapperAnnotationBuilder.java:369)

The following method did not exist:

    com.baomidou.mybatisplus.core.MybatisConfiguration.getLanguageDriver(Ljava/lang/Class;)Lorg/apache/ibatis/scripting/LanguageDriver;

The method's class, com.baomidou.mybatisplus.core.MybatisConfiguration, is available from the following locations:

    jar:file:/Users/guofangbi/.m2/repository/com/baomidou/mybatis-plus-core/3.2.0/mybatis-plus-core-3.2.0.jar!/com/baomidou/mybatisplus/core/MybatisConfiguration.class

It was loaded from the following location:

    file:/Users/guofangbi/.m2/repository/com/baomidou/mybatis-plus-core/3.2.0/mybatis-plus-core-3.2.0.jar


Action:

Correct the classpath of your application so that it contains a single, compatible version of com.baomidou.mybatisplus.core.MybatisConfiguration
```
#### 解决办法
这个错是因为 Camunda 与 baomidou 中的 mybatis 插件的 jar 冲突导致的。  
我看 baomidou 的 issues 中有人说将版本从 3.2.0 降到 3.1.0 即可，但是我试了没成功 。  
我是通过升了 mybatis 的版本解决的。

```xml
<dependency>
	<groupId>org.mybatis</groupId>
	<artifactId>mybatis</artifactId>
	<version>3.5.2</version>
</dependency>
```
### addString 不生效
通过 **deploymentBuilder.addClasspathResource()** 的方式可以正常发布流程，但是 **addString** 就不生效。  
发布时不报任何错，而且 *act_re_deployment* 表中有数据，只有 *act_re_procdef* 表没数据。  

#### 解决办法
我后来发现，是因为调用 *addString* 时指定的 **resourceName** 没有以 **.bpmn** 结尾，导致 Camunda 无法通过的后缀来判断使用的是哪种流程配置标准，继而无法对流程配置做相关的解析。不过这种不报任何错的方式让我排查了很久才找到原因。  

## 总结
Camunda 整体用起来还是很流畅的。而且还配套了 **Camunda Modeler** 作为可视化配置工具。  
其实流程引擎还是有很多 js 插件来满足用户在 web 页面进行流程可视化配置的需求，有兴趣的读者可以试着搜索 *bpmn-io* 、 *bpmn-js* 来找找看。