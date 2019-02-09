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
Spring集成还有发布资源的功能。在流程引擎配置中，你可以指定一组资源，引擎在启动时，可以自动扫描这些资源并发布。为了避免重复部署，它还有一个过滤器。当资源被修改的时候，新的发布会被加载到Activiti的数据库中。在Spring容器需要经常重启的场景(比如测试)中，这个功能非常有用。  
这里有个例子：
```xml
<bean id="processEngineConfiguration" class="org.activiti.spring.SpringProcessEngineConfiguration">
  ...
  <property name="deploymentResources"
    value="classpath*:/org/activiti/spring/test/autodeployment/autodeploy.*.bpmn20.xml" />
</bean>

<bean id="processEngine" class="org.activiti.spring.ProcessEngineFactoryBean">
  <property name="processEngineConfiguration" ref="processEngineConfiguration" />
</bean>
```
默认的，上面的配置会将符合过滤器的所有资源放在一起发布。在一次发布过程中，符合过滤条件的所有资源都会被加载。在一些场景中，这可能不符合你的期望。比如，你用这种方式发布了一组流程资源，哪怕只有一个资源文件修改了，所有的流程定义都要重新发布一次。  
如果你想自定义部署方式，你可以给**SpringProcessEngineConfiguration**设定一个额外的属性**deploymentMode**。这个属性决定了符合过滤条件的资源以何种方式发布，系统默认支持三个值：
- default：将符合过滤条件的所有资源放到一个单独的发布中，这个是默认值，如果你不配置的话，生效的就是这个。
- single-resource：给符合过滤条件的每个资源都单独分配一个发布。这个值可以让你将每个流程发布独立出来，在某个流程修改的时候，只有它自己会重新发布。
- resource-parent-folder：符合过滤条件的资源会按照所在文件夹分组，每组共享一个发布。这种方式可以达到对大多数资源进行发布隔离又共享的目的。  
下面是一个对将**deploymentMode**配置成**single-resource**的例子：
```xml
<bean id="processEngineConfiguration"
    class="org.activiti.spring.SpringProcessEngineConfiguration">
  ...
  <property name="deploymentResources" value="classpath*:/activiti/*.bpmn" />
  <property name="deploymentMode" value="single-resource" />
</bean>
```
除了使用**deploymentMode**的几个值之外，你可能还想自定义发布的行为。你可以创建一个**SpringProcessEngineConfiguration**的子类，并重写**getAutoDeploymentStrategy(String deploymentMode)**方法。这个方法将就是使用**deploymentMode**的配置来决定的发布策略的。
## 单元测试
集成了Spring，可以使用基本的[Activiti测试特性](https://www.activiti.org/userguide/index.html#apiUnitTesting)来非常方便地测试业务流程。下面就是一个典型的基于Spring的业务流程单测用例：
```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration("classpath:org/activiti/spring/test/junit4/springTypicalUsageTest-context.xml")
public class MyBusinessProcessTest {

  @Autowired
  private RuntimeService runtimeService;

  @Autowired
  private TaskService taskService;

  @Autowired
  @Rule
  public ActivitiRule activitiSpringRule;

  @Test
  @Deployment
  public void simpleProcessTest() {
    runtimeService.startProcessInstanceByKey("simpleProcess");
    Task task = taskService.createTaskQuery().singleResult();
    assertEquals("My Task", task.getName());

    taskService.complete(task.getId());
    assertEquals(0, runtimeService.createProcessInstanceQuery().count());

  }
}
```
注意，要想让它正确运行，你还需要定义一个*org.activiti.engine.test.ActivitiRule*的bean(上面的例子里，它使用*@Autowired*注入)
```xml
<bean id="activitiRule" class="org.activiti.engine.test.ActivitiRule">
  <property name="processEngine" ref="processEngine" />
</bean>
```
## 使用 Hibernate4.2 JPA
如果你在服务任务或者监听器逻辑中使用Hibernate4.2 JPA，你需要引入Spring ORM，如果Hibernate4.1及更低版本就不需要了。引入方式如下：
```xml
<dependency>
  <groupId>org.springframework</groupId>
  <artifactId>spring-orm</artifactId>
  <version>${org.springframework.version}</version>
</dependency>
```
## Spring Boot
按照[Spring官网](http://projects.spring.io/spring-boot/)说法，*Spring Boot是一个你可以轻松创建可直接运行的、独立的、生产级的、基于Spring的应用程序的框架。它天然集成了Spring平台和第三方库，从而让你能轻松上手。大多数的Spring Boot项目只需要依赖极少的Spring配置就能运行。*  
你可以点击http://projects.spring.io/spring-boot/查看关于SpringBoot的更多信息。  
现阶段，Activiti对Spring Boot的集成还在实验，它一直有Spring提交者进行开发，但仍处于早期阶段。我们欢迎所有的人尝试使用它并提供反馈。
### 兼容性
Spring Boot 需要JDK7，其他的你可以查看SpringBoot的官方文档
### 开始
Spring Boot倡导约定大于配置，一开始，你只需要在你的项目中添加*spring-boot-starters-basic*。下面是一个Maven的例子：
```xml
<dependency>
	<groupId>org.activiti</groupId>
	<artifactId>activiti-spring-boot-starter-basic</artifactId>
	<version>${activiti.version}</version>
</dependency>
```
有了它，Activiti和Spring的依赖项都会被引入到项目中。现在就可以开始编写SpringBoot代码了：
```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.EnableAutoConfiguration;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;

@Configuration
@ComponentScan
@EnableAutoConfiguration
public class MyApplication {

    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }

}
```
Activiti还需要一个数据库来存储数据。如果你运行上面的代码，它会抛一个异常，告诉你你需要添加对数据库驱动的依赖。我们来添加H2的数据库依赖：
```xml
<dependency>
	<groupId>com.h2database</groupId>
	<artifactId>h2</artifactId>
	<version>1.4.183</version>
</dependency>
```
现在程序就能完全启动了，输出日志如下：
```properties
 .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v1.1.6.RELEASE)

MyApplication                            : Starting MyApplication on ...
s.c.a.AnnotationConfigApplicationContext : Refreshing org.springframework.context.annotation.AnnotationConfigApplicationContext@33cb5951: startup date [Wed Dec 17 15:24:34 CET 2014]; root of context hierarchy
a.s.b.AbstractProcessEngineConfiguration : No process definitions were found using the specified path (classpath:/processes/**.bpmn20.xml).
o.activiti.engine.impl.db.DbSqlSession   : performing create on engine with resource org/activiti/db/create/activiti.h2.create.engine.sql
o.activiti.engine.impl.db.DbSqlSession   : performing create on history with resource org/activiti/db/create/activiti.h2.create.history.sql
o.activiti.engine.impl.db.DbSqlSession   : performing create on identity with resource org/activiti/db/create/activiti.h2.create.identity.sql
o.a.engine.impl.ProcessEngineImpl        : ProcessEngine default created
o.a.e.i.a.DefaultAsyncJobExecutor        : Starting up the default async job executor [org.activiti.spring.SpringAsyncExecutor].
o.a.e.i.a.AcquireTimerJobsRunnable       : {} starting to acquire async jobs due
o.a.e.i.a.AcquireAsyncJobsDueRunnable    : {} starting to acquire async jobs due
o.s.j.e.a.AnnotationMBeanExporter        : Registering beans for JMX exposure on startup
MyApplication                            : Started MyApplication in 2.019 seconds (JVM running for 2.294)
```


虽然你只是添加了依赖，加了个*@EnableAutoConfiguration*注解，但是后台还是发生了很多事情：    
- 自动创建一个内存数据库(因为引用了H2的驱动)，然后将连接配置发给Activiti引擎
- 创建并暴露 Activiti ProcessEngine
- 所有的Activiti服务都被当做Spring的bean暴露出来
- 创建Spring Job Executor  

同时，*processes*目录下的所有BPMN2.0流程定义都会被自动发布。所以你需要创建一个*processor*目录，然后添加一个虚拟的流程(文件名为*one-task-process.bpmn20.xml*)：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<definitions
        xmlns="http://www.omg.org/spec/BPMN/20100524/MODEL"
        xmlns:activiti="http://activiti.org/bpmn"
        targetNamespace="Examples">

    <process id="oneTaskProcess" name="The One Task Process">
        <startEvent id="theStart" />
        <sequenceFlow id="flow1" sourceRef="theStart" targetRef="theTask" />
        <userTask id="theTask" name="my task" />
        <sequenceFlow id="flow2" sourceRef="theTask" targetRef="theEnd" />
        <endEvent id="theEnd" />
    </process>

</definitions>
```
你还要添加以下代码来验证发布是否正常运行。*CommandLineRunner*是一个特殊的Spring bean，它在应用启动的时候就会被执行：
```java
@Configuration
@ComponentScan
@EnableAutoConfiguration
public class MyApplication {

    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }

    @Bean
    public CommandLineRunner init(final RepositoryService repositoryService,
                                  final RuntimeService runtimeService,
                                  final TaskService taskService) {

        return new CommandLineRunner() {
            @Override
            public void run(String... strings) throws Exception {
                System.out.println("Number of process definitions : "
                	+ repositoryService.createProcessDefinitionQuery().count());
                System.out.println("Number of tasks : " + taskService.createTaskQuery().count());
                runtimeService.startProcessInstanceByKey("oneTaskProcess");
                System.out.println("Number of tasks after process start: " + taskService.createTaskQuery().count());
            }
        };

    }

}
```
不出意外的话，会输出如下信息：
```properties
Number of process definitions : 1
Number of tasks : 0
Number of tasks after process start : 1
```
### 修改数据库和连接池
前文讲，Spring Boot提倡约定大于配置。默认情况下，Activiti 约定使用H2内存数据库。  
如果想修改数据源，只需要提供一个数据源的Bean就可以覆盖原有的。我们这里使用Spring Boot 自带的*DataSourceBuilder*来创建数据源。如果类路径下有Tomcat、HikariCP或者Commons
 DBCP，系统就会选择其中一个数据源(按照这个顺序，会第一个找Tomcat)。我们以MySQL数据库举例：
 ```java
@Bean
public DataSource database() {
    return DataSourceBuilder.create()
        .url("jdbc:mysql://127.0.0.1:3306/activiti-spring-boot?characterEncoding=UTF-8")
        .username("alfresco")
        .password("alfresco")
        .driverClassName("com.mysql.jdbc.Driver")
        .build();
}
```
移除Maven中对H2的依赖，然后添加MySQL的驱动和Tomcat连接池依赖：
```xml
<dependency>
	<groupId>mysql</groupId>
	<artifactId>mysql-connector-java</artifactId>
	<version>5.1.34</version>
</dependency>
<dependency>
	<groupId>org.apache.tomcat</groupId>
	<artifactId>tomcat-jdbc</artifactId>
	<version>8.0.15</version>
</dependency>
```
系统启动的时候，你可以通过日志看到它使用了MySQL作为数据库(Tomcat作为连接池)：
```properties
org.activiti.engine.impl.db.DbSqlSession   : performing create on engine with resource org/activiti/db/create/activiti.mysql.create.engine.sql
org.activiti.engine.impl.db.DbSqlSession   : performing create on history with resource org/activiti/db/create/activiti.mysql.create.history.sql
org.activiti.engine.impl.db.DbSqlSession   : performing create on identity with resource org/activiti/db/create/activiti.mysql.create.identity.sql
```
### 支持REST
通常，系统如果嵌入了Activit引擎，那就需要在上面包一层REST API(以便于跟公司其他服务进行交互)。Spring Boot让它变得非常简单，只需要添加下面的依赖即可：
```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-web</artifactId>
	<version>${spring.boot.version}</version>
</dependency>
```
创建一个新的类并加上*@Service*注解，然后给它添加两个方法：一个用来启动流程，一个用来查询任务。我们这里只是简单包装一下Activiti的调用，但是现实场景肯定比这个要复杂很多。
```java
@Service
public class MyService {

    @Autowired
    private RuntimeService runtimeService;

    @Autowired
    private TaskService taskService;

	@Transactional
    public void startProcess() {
        runtimeService.startProcessInstanceByKey("oneTaskProcess");
    }

	@Transactional
    public List<Task> getTasks(String assignee) {
        return taskService.createTaskQuery().taskAssignee(assignee).list();
    }

}
```
现在你可以给*@RestController*注解的类创建一个REST接口。这里，我们只是在里面调用了一下上面的服务。
```java
@RestController
public class MyRestController {

    @Autowired
    private MyService myService;

    @RequestMapping(value="/process", method= RequestMethod.POST)
    public void startProcessInstance() {
        myService.startProcess();
    }

    @RequestMapping(value="/tasks", method= RequestMethod.GET, produces=MediaType.APPLICATION_JSON_VALUE)
    public List<TaskRepresentation> getTasks(@RequestParam String assignee) {
        List<Task> tasks = myService.getTasks(assignee);
        List<TaskRepresentation> dtos = new ArrayList<TaskRepresentation>();
        for (Task task : tasks) {
            dtos.add(new TaskRepresentation(task.getId(), task.getName()));
        }
        return dtos;
    }

    static class TaskRepresentation {

        private String id;
        private String name;

        public TaskRepresentation(String id, String name) {
            this.id = id;
            this.name = name;
        }

         public String getId() {
            return id;
        }
        public void setId(String id) {
            this.id = id;
        }
        public String getName() {
            return name;
        }
        public void setName(String name) {
            this.name = name;
        }

    }

}
```
*@Service*和*RestController*注解的类都能被自动扫描到(*@ComponentScan*)。重新启动这个应用，我们就可以通过示例中的 curl 命令来跟REST API交互了：
```shell
curl http://localhost:8080/tasks?assignee=kermit
[]

curl -X POST  http://localhost:8080/process
curl http://localhost:8080/tasks?assignee=kermit
[{"id":"10004","name":"my task"}]
```
### 支持JPA
你可以在 Spring Boot 中给 Activiti 添加JPA支持，只需要添加下面的依赖：
```xml
<dependency>
	<groupId>org.activiti</groupId>
	<artifactId>activiti-spring-boot-starter-jpa</artifactId>
	<version>${activiti.version}</version>
</dependency>
```
它将会添加JPA需要的Spring 配置和bean。默认情况下，由 Hibernate提供JPA服务。  
接着我们创建一个简单的实体类：
```java
@Entity
class Person {

    @Id
    @GeneratedValue
    private Long id;

    private String username;

    private String firstName;

    private String lastName;

    private Date birthDate;

    public Person() {
    }

    public Person(String username, String firstName, String lastName, Date birthDate) {
        this.username = username;
        this.firstName = firstName;
        this.lastName = lastName;
        this.birthDate = birthDate;
    }

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getUsername() {
        return username;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    public String getFirstName() {
        return firstName;
    }

    public void setFirstName(String firstName) {
        this.firstName = firstName;
    }

    public String getLastName() {
        return lastName;
    }

    public void setLastName(String lastName) {
        this.lastName = lastName;
    }

    public Date getBirthDate() {
        return birthDate;
    }

    public void setBirthDate(Date birthDate) {
        this.birthDate = birthDate;
    }
}
```
默认情况下，如果你不是使用内存数据库，系统不会自动创建表。在类路径下，新建一个名叫 *application.properties* 的文件，然后加上下面这个属性：
```properties
spring.jpa.hibernate.ddl-auto=update
```
然后添加下面这个类：
```java
public interface PersonRepository extends JpaRepository<Person, Long> {

    Person findByUsername(String username);

}
```
这是一个Sping 的 repository，它提供开箱即用的CRUD功能，我们还给它添加了一个通过username查找Person的方法。Spring会默认给它自动创建一个实现类。  
接着我们进一步增强这个service的服务：
- 添加一个*@Transactional*注解。注意，通过添加JPA依赖，我们之前使用的DataSourceTransactionManager 被自动替换成 JpaTransactionManager.
- *startProcess*接收一个username来查询Person，然后将该Person的JPA对象作为一个流程变量传给流程实例。  
- 添加一个创建虚拟用户的方法，CommanLineRunner用它来将数据保存到数据库。
```java
@Service
@Transactional
public class MyService {

    @Autowired
    private RuntimeService runtimeService;

    @Autowired
    private TaskService taskService;

    @Autowired
    private PersonRepository personRepository;

    public void startProcess(String assignee) {

        Person person = personRepository.findByUsername(assignee);

        Map<String, Object> variables = new HashMap<String, Object>();
        variables.put("person", person);
        runtimeService.startProcessInstanceByKey("oneTaskProcess", variables);
    }

    public List<Task> getTasks(String assignee) {
        return taskService.createTaskQuery().taskAssignee(assignee).list();
    }

    public void createDemoUsers() {
		 if (personRepository.findAll().size() == 0) {
            personRepository.save(new Person("jbarrez", "Joram", "Barrez", new Date()));
            personRepository.save(new Person("trademakers", "Tijs", "Rademakers", new Date()));
        }
    }

}
```
*CommandLineRunner*现在长这样：
```java
@Bean
public CommandLineRunner init(final MyService myService) {

	return new CommandLineRunner() {
    	public void run(String... strings) throws Exception {
        	myService.createDemoUsers();
        }
    };

}
```
为了配合上面的更改，*RestController*也需要做一些调整(这里仅展示新添加的方法)，HTTP POST 现在需要有个包含处理人用户名的body：
```java
@RestController
public class MyRestController {

    @Autowired
    private MyService myService;

    @RequestMapping(value="/process", method= RequestMethod.POST)
    public void startProcessInstance(@RequestBody StartProcessRepresentation startProcessRepresentation) {
        myService.startProcess(startProcessRepresentation.getAssignee());
    }

   ...

    static class StartProcessRepresentation {

        private String assignee;

        public String getAssignee() {
            return assignee;
        }

        public void setAssignee(String assignee) {
            this.assignee = assignee;
        }
    }
```
最后，为了使用Spring-JPA-Activiti，我们流程定义中Person的JPA对象的id来分配任务：
```xml
<userTask id="theTask" name="my task" activiti:assignee="${person.id}"/>
```
现在我们就可以通过在POST 的body中传一个用户名，就可以启动一个新的流程实例：
```shell
curl -H "Content-Type: application/json" -d '{"assignee" : "jbarrez"}' http://localhost:8080/process
```
而通过用户id也可以查到他的任务列表：
```shell
curl http://localhost:8080/tasks?assignee=1

[{"id":"12505","name":"my task"}]
```
### 深入阅读
很显然，Spring Boot还有很多东西我们并没有涉及到，比如很简单的JTA集成，或者如何构建可在很多服务器上运行的war包。而且，Spring Boot集成还有很多东西：
- 对 Actuator 的支持
- 对 Spring Integration 的支持
- 对 REST API 的支持：在Spring 应用中启动Activiti Rest API
- 对 Spring Security 的支持  

第一版中，我们对这些功能只是有个大致了解，但是将来会对它们做更深层次的应用