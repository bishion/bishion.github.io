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
 - databaseType：正常来说，你没必要设置这个属性， 系统可以根据数据库连接自动判断数据库类型. 设置它主要是为了应对自动判断失效的情况.  可以设置的值:{h2, mysql, oracle， postgres，mssql，db2}。这个设置决定了使用何种 create/drop 数据库脚本。点击[支持的数据库](https://www.activiti.org/userguide/index.html#supporteddatabases)来对我们支持的数据库有个宏观认识。
 - databaseSchemaUpdate：允许用户设置处理引擎在启动和停止时对于数据库的操作策略。
   - false(默认)：在处理引擎启动的时候，检查数据库版本是否跟程序是否匹配，如果不匹配，就抛异常
   - true：在处理引擎构造的时候，检查数据库，如果有需要就更新数据库。如果数据库不存在，就会创建一个。
   - create-drop：当流程引擎启动的时候就创建数据库，关闭的时候就销毁数据库。
## JNDI 数据源配置
默认的情况下，Activiti 的数据库配置是放在每个应用的 WEB-INF/classes下面。这种方式并不完美，因为它需要用户要么修改Activiti的源码中的db.properties 文件然后重新打war 包，要么就在每次发布的时候，解压 war包然后手动去修改该配置文件。  
通过使用 JNDI(Java Naming and Directory Interface，Java 命名和目录接口)来维护数据库连接，连接配置就可以全权交给 Servlet 容器，跟 war 包的发布就完全无关了。 这种方式比 db.properties 提供的连接参数更丰富。
### 配置
JNDI 的配置因你使用的 servlet 容器的不同而不同。 本文我们以 Tomcat 为例，如果你使用的是别的容器应用，还请参考对应的文档。  
如果你使用的是 Tomcat， JNDI 数据源往往配置在 $CATALINA_BASE/conf/[enginename]/[hostname]/[warname].xml(对于 Activiti UI，放在$CATALINA_BASE/conf/Catalina/localhost/activiti-app.xml)文件中。Activiti war 包在第一次发布的时候，会将默认的配置文件拷贝到上述目录下，所以如果它已经存在了，你需要手动替换掉它。你使用如下的配置就可以将 JNDI数据源从 H2 修改成 MySQL：
```java
<?xml version="1.0" encoding="UTF-8"?>
    <Context antiJARLocking="true" path="/activiti-app">
        <Resource auth="Container"
            name="jdbc/activitiDB"
            type="javax.sql.DataSource"
            description="JDBC DataSource"
            url="jdbc:mysql://localhost:3306/activiti"
            driverClassName="com.mysql.jdbc.Driver"
            username="sa"
            password=""
            defaultAutoCommit="false"
            initialSize="5"
            maxWait="5000"
            maxActive="120"
            maxIdle="5"/>
        </Context>
```
### JNDI 属性
你需要设置以下属性来为 Activiti UI 配置 JNDI 数据源：
- datasource.jndi.name：JNDI 名称
- datasource.jndi.resourceRef：设置是否在 J2EE 容器中查找，比如，如果 JNDI 名称中没有包含设置的话，是否要加上"java:comp/env/"前缀。默认值是 "true"
## 支持的数据库
Activiti 支持如下的数据库(大小写敏感)：

| 数据库类型 | JDBC URL 示例 | 备注 |
|--------- |-------------|---|
| h2 |	jdbc:h2:tcp://localhost/activiti | 默认数据库 |
|mysql|jdbc:mysql://localhost:3306/activiti?autoReconnect=true|
|oracle|jdbc:oracle:thin:@localhost:1521:xe|
|postgres|jdbc:postgresql://localhost:5432/activiti|
|db2|jdbc:db2://localhost:50000/activiti|
|mssql|jdbc:sqlserver://localhost:1433;databaseName=activiti<br/>(jdbc.driver=com.microsoft.sqlserver.jdbc.SQLServerDriver) OR jdbc:jtds:sqlserver://localhost:1433/activiti<br/>(jdbc.driver=net.sourceforge.jtds.jdbc.Driver)|使用微软 JDBC 4.0驱动(sqljdbc.jar) 和 JTDS 驱动测试过|
## 创建数据库表格
最容易的创建表格的方式如下：
- 将 activiti-engine 的jar 包引用到你的项目中
- 添加对应的数据库驱动
- 在项目中添加 Activiti 配置文件(activiti.cfg.xml)，配置指向你的数据库(参考[数据库配置](https://www.activiti.org/userguide/index.html#databaseConfiguration))
- 执行 *DbSchemaCreate* 的 main 方法
但是，大多数情况下，只有数据库管理员才能执行 DDL 语句。在生产环境，这个是最聪明的做法。用户可以在 Activiti 下载页面或者 Activiti 源码包下的 *database* 目录下找到数据库 DDL 脚本。这些脚本在 activiti-engine-*.jar 中也有，放在 org/activiti/db/create(在drop目录下有drop脚本)包里面。这些 SQL 文件的命名格式如下：
> activiti.{db}.{create|drop}.{type}.sql

其中，*db* 是[支持的数据库](https://www.activiti.org/userguide/index.html#supporteddatabases)中的一种，*type* 是下面的值之一：
- engine：引擎执行时需要的表，必须有
- identity：这些表里放着用户、用户组，还有两者的关联数据。这些表是可选的，如果你需要用流程引擎自带的认证体系，那就需要这些表。
- history：这些表里存放着历史和审计数据。这些表是可选的，当历史级别设置成 *none* 的时候，这些表就不用设置了。注意，如果你不使用历史表，一些使用历史表存数据的特性(比如任务的备注)也就被禁用了。
MySQL 用户需要注意：版本号低于 5.6.4 的MySQL不支持毫秒级别的时间戳和日期。 更郁闷的是，有些版本甚至在你要创建这些类型的列的时候还会报异常。在做自动创建和删除的时候，流程引擎会在执行之前，修改 DDL 脚本。如果你使用 DDL 的方式，我们有两个脚本：正常版本和 *mysql55* 版本(该版本试用于低于5.6.4的任何版本)。第二个脚本中去掉了对列的毫秒级精度要求。  

下面具体谈下 MySQL 的版本：
- <5.6：毫秒级精度不可用。 你可以使用文件名中有 *mysql55* 字样的 DDL 脚本。自动创建/更新数据库的功能依然开箱即用
- 5.6.0 - 5.6.3：毫秒级精度不可用。系统不能自动创建/更新数据库。我们推荐你更新数据库版本，如果你真要用的话，可以使用手动执行 *mysql55* 的数据库脚本
- 5.6.4+：支持毫秒级精度。DDL 脚本也可以用(文件名有 *mysql* 字样)。自动创建/更新数据库的功能也是开箱即用

千万注意：一旦 Activiti 的表格已经创建或者更新好了之后，你再更新数据库版本，这时候再想列支持毫秒精度，就需要手动去修改列的类型了

## 关于数据库表名的解释
Activiti 的数据库表名都是以 **ACT_** 开头。表名的第二部分用两个字符标识表名的使用场景。这种标识场景的方式基本上也与服务 API 的格式大致吻合  
- ACT_RE_*：*RE* 表示 *repository*。使用这个前缀的表里存放的是一些 *静态*信息，比如流程定义和流程资源(图片，规则等)。
- ACT_RU_*：*RU* 表示 *runtime*。这些表里存放着一些运行时的数据，比如流程实例，用户待办，变量，任务等等。Activiti 只在流程实例运行的时候保存这些数据，当流程运行完之后，数据就会被删除。这样才能保证这些表数据量小运行速度快。
- ACT_ID_*：*ID* 表示 *identity*。这些表里存放着认证数据，比如用户，用户组等等
- ACT_HI_*：*HI* 表示 *history*。这些表里存着历史数据，比如过去的流程实例，变量，任务等等。
- ACT_GE_*：在很多场景中会被用到的*general* 数据。

## 数据库更新
数据库更新前，确保你备份了所有的数据(你自己的数据库备份机制)。  
每个流程引擎在创建的时候，都会默认检查数据库版本。如果 Activiti 发现数据库表的版本跟自己的版本不匹配的时候，就会抛异常。  
如果你想更新，你必须在 *activiti.cfg.xml* 文件中添加如下设置：
```xml
<beans >

  <bean id="processEngineConfiguration" class="org.activiti.engine.impl.cfg.StandaloneProcessEngineConfiguration">
    <!-- ... -->
    <property name="databaseSchemaUpdate" value="true" />
    <!-- ... -->
  </bean>

</beans>
```
**当然了，你还是要将合适的数据库驱动引入到自己的项目中。**更新你项目中的 Activiti 库，或者启动一个新版的 Activiti，然后将数据库配置指向你的旧数据库。因为 **databaseSchemaUpdate** 被设置成 **true**，所以 Activiti 在检测到程序和数据库表版本不同步的时候，就会自动更新数据库(表)到最新版本。
**你还可以使用 DDL 脚本手动更新数据库。**你可以去Activiti的下载页面找到更新脚本，然后执行就可以了。
## 任务执行器(6.0.0 版本之后才有)
Activiti6 只保留了 Activiti5的异步执行器，因为它在流程引擎中执行异步任务时候性能更高而且对数据库友好。Activiti5其他的任务执行器都舍弃了。更多信息可以在本文档的高阶部分讲到。  
此外，如果是在Java EE7下面运行，可以使用符合 JSR-236 的 ManagedAsyncJobExecutor 来让容器管理多线程。像下面这样，将线程工厂注入给 ManagedAsyncJobExecutor 就可以做到：
```xml
<bean id="threadFactory" class="org.springframework.jndi.JndiObjectFactoryBean">
   <property name="jndiName" value="java:jboss/ee/concurrency/factory/default" />
</bean>

<bean id="customJobExecutor" class="org.activiti.engine.impl.jobexecutor.ManagedAsyncJobExecutor">
   <!-- ... -->
   <property name="threadFactory" ref="threadFactory" />
   <!-- ... -->
</bean>
```
如果没有指定线程工厂，则托管实现则使用默认选项。
## 激活任务执行器
**AsyncExector** 是一个管理线程池的组件，可以用来触发定时器和其他异步任务。也可以使用其他组件来实现(比如消息队列，可以参考本文档的高阶部分。)

默认情况下，**AsyncExecutor** 并没有被激活，因此也不会被启动。通过下面的配置，异步执行器就可以跟着 Activiti 引擎一起启动。
```xml
<property name="asyncExecutorActivate" value="true" />
```
*asyncExecutorActivate* 属性让 Activiti 引擎在启动的时候将异步执行器也跟着启动。  
## 邮件服务器配置
我们不强制你配置邮件服务器。Activiti 支持在业务流程中发送邮件。为了让邮件真正的发送出去，一个准确的SMTP邮件服务器配置是必须的。你可以点击[邮件任务](https://www.activiti.org/userguide/index.html#bpmnEmailTaskServerConfiguration)查看配置选项
## 历史数据配置
用户也可以选择性配置保存历史数据，这将允许你自定义流程引擎的[历史功能](https://www.activiti.org/userguide/index.html#history)。点击[历史数据设置]查看更多详细信息。
```xml
<property name="history" value="audit" />
```
## 在表达式和脚本中暴露配置 bean
默认情况下，你在**activiti.cfg.xml**配置或者其他配置文件中的bean，对于表达式或者脚本都是可见的。如果你想限制配置文件中bean的作用域，你可以在流程引擎配置中设置一个叫做 **beans**的属性。**ProcessEngineConfiguration**的 *beans* 属性是一个map。表达式和脚本只能看到你指定的这个map中的bean。bean 通过map中配置的名称被暴露出来。
## 缓存配置
所有的流程定义在被解析后就放到缓存里，从而避免每次访问流程时都要去数据库里取。默认情况下，对于缓存的使用并没有限制。为了限制缓存的流程定义数，你可以配置如下属性：
```xml
<property name="processDefinitionCacheLimit" value="10" />
```
设置这个属性可以将默认的hashmap形式的缓存改成有明确限制的LRU缓存。当然了，这个属性的值取决于你流程的总数和实际运行的流程数。  
你还可以注入你自己的缓存实现。这个实现必须是一个bean，而且这个bean需要实现 *org.activiti.engine.impl.persistence.deploy.DeploymentCache*：
```xml
<property name="processDefinitionCache">
  <bean class="org.activiti.MyCache" />
</property>
```
其实还有两个个类似的属性叫**knowledgeBaseCacheLimit**和**knowledgeBaseCache**，这两个属性是用来配置规则缓存的。如果你的流程中有用到规则任务的时候，它们才需要配置。
## 日志
所有的日志(activiti，spring，mybatis，...)都使用了 SLF4J，这样就允许你按照自己的喜好来选择日志实现。  
**默认情况下，流程引擎的依赖中并没有 SLF4j 的绑定实现，所以你需要手动添加到项目中。** 如果没有加实现，SLF4J 就会使用一个 NOP-logger，不会记录任何日志，而不是去发出一个不打日志的警告。更多信息你可以参考：http://www.slf4j.org/codes.html#StaticLoggerBinder。  
如果使用Maven，添加一个依赖(这里使用 log4j 举例)的格式如下。注意，你还需要给它添加一个版本：
``` xml
<dependency>
  <groupId>org.slf4j</groupId>
  <artifactId>slf4j-log4j12</artifactId>
</dependency>
```
activiti-ui 和 activiti-rest 应用配置的是 log4j。如果你执行 Activiti 的所有模块下的测试用例，我们使用的也是 log4j。  
**如果你的容器里引用了 commons-logging，那么请注意：**为了将spring-logging通过SLF4J路由进来，我们需要一个桥(参考：http://www.slf4j.org/legacy.html#jclOverSLF4J)。如果你的容器提供了一个 commons-logging 的实现，为了程序的稳定，你需要按照http://www.slf4j.org/codes.html#release 的指示来配置。  
使用 Mavne 的设置方式(版本号省略了)：
```java
<dependency>
  <groupId>org.slf4j</groupId>
  <artifactId>jcl-over-slf4j</artifactId>
</dependency>
```
## 映射诊断上下文
Activiti 支持 SLF4j 的映射诊断上下文。下面这些信息将跟着日志一起被传给底层的日志实现：
- mdcProcessDefinitionID表示流程定义id
- mdcProcessInstanceID表示流程实例id
- mdcExecutionId表示执行id
默认情况下，这些日志并不会被记录。你可以按照你想要的格式配置，从而将它们跟你原本的日志一起打印出来。比如，在Log4j中使用下面这个示例就会将上面所有信息都输出出来：
```properties
log4j.appender.consoleAppender.layout.ConversionPattern=ProcessDefinitionId=%X{mdcProcessDefinitionID}executionId=%X{mdcExecutionId} mdcProcessInstanceID=%X{mdcProcessInstanceID} mdcBusinessKey=%X{mdcBusinessKey} %m%n
```
如果你有专门的日志分析器来实时检测日志的时候，这个东西将非常有用。
## 事件处理器
Activiti中的事件机制可以让你在很多事件中得到通知。你可以点击查看[所有支持的事件类型](https://www.activiti.org/userguide/index.html#eventDispatcherEventTypes)来对事件做一个概览。  
你可以针对任何一种事件注册一个监听器来对该事件发生时做出相应的反匮。你不仅可以使用[配置](https://www.activiti.org/userguide/index.html#eventDispatcherConfiguration)或者[API](https://www.activiti.org/userguide/index.html#eventDispatcherConfigurationRuntime)的方式添加引擎范围的监听器，也可以在[BPMN的XML中特定的流程定义](https://www.activiti.org/userguide/index.html#eventDispatcherConfigurationProcessDefinition)中添加事件监听器。  
所有的事件都是**org.activiti.engine.delegate.event.ActivitiEvent**的衍生。事件暴露出来**type(类型)**，**executionId(执行Id)**，**processInstanceId(处理实例Id)**和**processDefinitionId(流程定义Id)**。某些事件包含与事件发生相关的上下文，其他的有效负载的信息可以在[所有支持的事件类型](https://www.activiti.org/userguide/index.html#eventDispatcherEventTypes)中找到。
### 事件监听器的实现
一个事件监听器只需要实现**org.activiti.engine.delegate.event.ActivitiEventListener**。下面就是一个监听器的示例，他会将收到的所有事件输出到控制台，但是不包括与任务执行相关的事件：
```java
public class MyEventListener implements ActivitiEventListener {

  @Override
  public void onEvent(ActivitiEvent event) {
    switch (event.getType()) {

      case JOB_EXECUTION_SUCCESS:
        System.out.println("A job well done!");
        break;

      case JOB_EXECUTION_FAILURE:
        System.out.println("A job has failed...");
        break;

      default:
        System.out.println("Event received: " + event.getType());
    }
  }

  @Override
  public boolean isFailOnException() {
    // 当前监听器中的onEvent方法不会抛异常，所以异常情况也可以被忽略。。。
    return false;
  }
}
```
上述的**isFailOnException()** 方法是在 **onEvent(..)**方法抛出异常时执行的。如果该方法返回**false**，该异常就会被忽略。如果返回的是**true**，异常就会被抛出，从而让当前正在执行的命令以失败终止。如果该事件是API调用(或者其他事务操作比如任务执行)的一部分，事务就要执行回滚。如果该监听器不是业务敏感型的操作，建议直接返回**false**。  
Activiti 提供了一些基础的事件实现，以给常见的事件监听场景提供便利。它们可以作为监听器的基类或者示例参考：
- org.activiti.engine.delegate.event.BaseEntityEventListener：这是一个监听器的基类，可以用来监听特定实体或者所有实体相关的事件。它隐藏了类型检查，并且提供了4个抽象方法：**onCreate(..)**，**onUpdate(..)**和**onDelete(..)**是对实体创建、更新或者删除的时候的监听；其他类型的实体事件发生时，**onEntityEvent(..)**就会被调用。
### 配置和启动
一旦配置了监听器，它就会在流程引擎启动的时候被激活，而且流程引擎重启了它也还在。  
**eventListeners**属性保存的是**org.activiti.engine.delegate.event.ActivitiEventListener**的实例列表。