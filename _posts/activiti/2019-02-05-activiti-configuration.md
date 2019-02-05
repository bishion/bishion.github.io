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