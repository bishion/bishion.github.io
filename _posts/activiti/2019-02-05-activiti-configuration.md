# 配置
## 创建 ProcessEngine
Activiti 处理引擎是通过 *activiti.cfg.xml* 来配置的. 注意, 如果你使用[Sprng 风格创建处理引擎](https://www.activiti.org/userguide/index.html#springintegration), 这个配置就不适用了  
*ProcessEngine* 的最方便的创建方式是使用 *org.activiti.engine.ProcessEngines* 类:
```
ProcessEngine processEngine = ProcessEngines.getDefaultProcessEngine()
```
程序将会自动在类加载路径下寻找 *activiti.cfg.xml*, 然后使用该文件的配置来初始化一个处理引擎. 下面的代码段展示了一个配置的示例, 然后我们会对这些配置做一个详细介绍.
```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

  <bean id="processEngineConfiguration" class="org.activiti.engine.impl.cfg.StandaloneProcessEngineConfiguration">

    <property name="jdbcUrl" value="jdbc:h2:mem:activiti;DB_CLOSE_DELAY=1000" />
    <property name="jdbcDriver" value="org.h2.Driver" />
    <property name="jdbcUsername" value="sa" />
    <property name="jdbcPassword" value="" />

    <property name="databaseSchemaUpdate" value="true" />

    <property name="asyncExecutorActivate" value="false" />

    <property name="mailServerHost" value="mail.my-corp.com" />
    <property name="mailServerPort" value="5025" />
  </bean>
</beans>
```
注意: 这个 XML 实际上是一个 Spring 配置. **这并不意味着 Activiti 只能被用在 Spring 环境中!** 我们只是通过 Spring 的依赖注入特性来初始化这个引擎.  
ProcessEngineConfiguration 对象也可以以代码方式通过配置文件来创建. 它还可以通过一个不同的 bean id (比如下面第三行的示例)创建
```java
ProcessEngineConfiguration.createProcessEngineConfigurationFromResourceDefault();
ProcessEngineConfiguration.createProcessEngineConfigurationFromResource(String resource);
ProcessEngineConfiguration.createProcessEngineConfigurationFromResource(String resource, String beanName);
ProcessEngineConfiguration.createProcessEngineConfigurationFromInputStream(InputStream inputStream);
ProcessEngineConfiguration.createProcessEngineConfigurationFromInputStream(InputStream inputStream, String beanName);
```
我们还可不使用配置文件，而是使用默认方式来创建配置信息，(点击[查看更多]())
```java
ProcessEngineConfiguration.createStandaloneProcessEngineConfiguration();ProcessEngineConfiguration.createStandaloneInMemProcessEngineConfiguration(
```
所有类似 *ProcessEngineConfiguration.createXXX() * 的方法均返回一个  *ProcessEngineConfiguration* 对象, 然后你可以按需调整它的属性. 最后, 通过调用 *buildProcessEngine()*, 我们可以得到一个 *ProcessEngine* 对象:
```java
ProcessEngine processEngine = ProcessEngineConfiguration.createStandaloneInMemProcessEngineConfiguration()
 .setDatabaseSchemaUpdate(ProcessEngineConfiguration.DB_SCHEMA_UPDATE_FALSE)
 .setJdbcUrl("jdbc:h2:mem:my-own-db;DB_CLOSE_DELAY=1000")
 .setAsyncExecutorActivate(false)
 .buildProcessEngine();
```
## ProcessEngineConfiguration bean
*activiti.cfg.xml* 必须有个 id 为 *processEngineConfiguration* 的 bean.
```xml
<bean id="processEngineConfiguration" class="org.activiti.engine.impl.cfg.StandaloneProcessEngineConfiguration"/>
```
这个 bean 被用来初始化 *ProcessEngine*. 很多类你都可以用来定义 *
processEngineConfiguration *.这些类适用于根据不同的环境设置不同的默认值. 你需要根据你的环境选择一个(最)合适的类, 从而让你的引擎可以用最小的配置跑起来. 下面是当前有的几个类(未来版本我们会往里面接着添加):
-  org.activiti.engine.impl.cfg.StandaloneProcessEngineConfiguration: 可以独立使用的处理引擎. Activiti 可以处理好事务. 默认情况下, 在引擎启动时才会检查数据库(如果数据库中没有 Activiti 或者版本不对, 就会抛异常)
- org.activiti.engine.impl.cfg.StandaloneInMemProcessEngineConfiguration: 这个是类可以被方便地用来做单测. Activiti 也可以处理好它的事务. 默认情况下, 它使用 H2 内存型数据库. 数据库随着处理引擎启动而创建, 然后在处理引擎关闭时删除. 如果你使用它,基本上不用做什么配置(除非你使用到了任务或者邮箱特性).
- org.activiti.spring.SpringProcessEngineConfiguration: 这个是在 Spring 环境中使用的处理引擎. 更多信息你可以点击[Spring 介绍章节]()
- org.activiti.engine.impl.cfg.JtaProcessEngineConfiguration: 在流程引擎作为一个单独模块运行时使用, 它内置了 JTA 事务
## 数据库配置
Activiti 使用两种方式配置数据库. 一种是定义数据库的 JDBC 配置:
- jdbcUrl: 数据库的 JDBC 路径
- jdbcDriver: 针对数据库的驱动
- jdbcUsername: 数据库用户
- jdbcPassword: 数据库密码
通过 JDBC 属性配置的方式获取到的数据源默认使用[MyBatis]连接池设置. 下面的可选属性可以用来配置这个连接池(搬运于 MyBatis 文档):
- 
jdbcMaxActiveConnections: 最大的活动连接. 默认是 10 个
- jdbcMaxIdleConnections: 连接池维护的最大空闲连接数
- 
jdbcMaxCheckoutTime:获取连接的最大等待毫秒数. 默认是20000(20s)
- 
jdbcMaxWaitTime: 这个是一个低优先级的配置, 用来设置获取连接的最大等待毫秒数,以防止连接池配置出错时, 程序出现假死). 默认是 20000(20s)
数据库配置示例
```xml
<property name="jdbcUrl" value="jdbc:h2:mem:activiti;DB_CLOSE_DELAY=1000" /><property name="jdbcDriver" value="org.h2.Driver" /><property name="jdbcUsername" value="sa" /><property name="jdbcPassword" value="" 
```
我们的测试显示, MyBatis 连接池在高并发请求时性能不是最好的. 因此, 我们推荐使用 *javax.sql.DataSource* 的实现来配置处理引擎(比如, DBCP, C3P0, Hikari, Tocmat 连接池等):
```xml
<bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource" >
 <property name="driverClassName" value="com.mysql.jdbc.Driver" />
 <property name="url" value="jdbc:mysql://localhost:3306/activiti" />
 <property name="username" value="activiti" />
 <property name="password" value="activiti" />
 <property name="defaultAutoCommit" value="false" />
</bean>
<bean id="processEngineConfiguration" class="org.activiti.engine.impl.cfg.StandaloneProcessEngineConfiguration">
 <property name="dataSource" ref="dataSource" />
 ...
 ```
 请注意, 我们并不提供这些数据源的库, 所以你需要自己引用这些依赖.  
 无论你使用 JDBC 还是数据源组件,都可以设置下面这些属性.
 - databaseType: 
