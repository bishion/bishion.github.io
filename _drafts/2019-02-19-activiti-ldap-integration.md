---
layout: post
title: Activiti 6.0 用户文档 - 第十五章：配置
categories: activiti
description: 流程引擎 Activiti 第十五章：配置
keywords: 流程引擎, activiti
---
# LDAP 集成
公司通常都会有一套 LDAP 系统来存储用户和用户组。在5.14版之后，针对与 LDAP 系统对接的问题，Activiti提供一套开箱即用的解决方案。  
在5.14版之前，Activiti也能与LDAP集成。但是，从5.14版开始，配置就大大简化了，但是还是会兼容以前的配置方式。具体地说，简化版的配置只是对*原有基础架构*的包装。  
## 用法
如果想将LDAP集成到你的项目中，只需要在pom.xml中添加下面的依赖：
```xml
<dependency>
  <groupId>org.activiti</groupId>
  <artifactId>activiti-ldap</artifactId>
  <version>latest.version</version>
</dependency>
```
## 使用场景
LDAP 集成目前有两个主要场景：
- 允许通过 IdentityService 做认证。使用 IdentityService做任何操作是非常有意义的。
- 获取用户组。这在一些场景非常有用，比如查询一个某个特性用户可以看到的任务（比如某个候选用户组）

## 配置
通过给流程引擎配置的**configurators**注入 **org.activiti.ldap.LDAPConfigurator** 实例就可以集成 LDAP 系统。该类有很高的扩展性：方法很容易被重写，依赖的bean也是可插拔的，如果你对默认实现不满意也可以替换。  
下面是一个配置的示例（注意：使用代码方式创建引擎跟这个方式也是类似）。暂时不要被那么多属性吓到了，我们在下章会详细分析它们。
```xml
<bean id="processEngineConfiguration" class="...SomeProcessEngineConfigurationClass">
    ...
    <property name="configurators">
        <list>
            <bean class="org.activiti.ldap.LDAPConfigurator">

            <!-- Server connection params -->
            <property name="server" value="ldap://localhost" />
            <property name="port" value="33389" />
            <property name="user" value="uid=admin, ou=users, o=activiti" />
            <property name="password" value="pass" />

            <!-- Query params -->
            <property name="baseDn" value="o=activiti" />
            <property name="queryUserByUserId" value="(&(objectClass=inetOrgPerson)(uid={0}))" />
            <property name="queryUserByFullNameLike" value="(&(objectClass=inetOrgPerson)(|({0}=*{1}*)({2}=*{3}*)))" />
            <property name="queryGroupsForUser" value="(&(objectClass=groupOfUniqueNames)(uniqueMember={0}))" />

            <!-- Attribute config -->
            <property name="userIdAttribute" value="uid" />
            <property name="userFirstNameAttribute" value="cn" />
            <property name="userLastNameAttribute" value="sn" />
            <property name="userEmailAttribute" value="mail" />


            <property name="groupIdAttribute" value="cn" />
            <property name="groupNameAttribute" value="cn" />

            </bean>
        </list>
    </property>
</bean>
```
## 属性
下面这些属性可以在 **org.activiti.ldap.LDAPConfigurator** 中设置：  
|属性名|描述|类型|默认值|
|-----|---|----|-----|
|server|LDAP系统的服务地址。比如：ldap://localhost:33389|String||
|port|LDAP系统运行的端口|int||
|user|连接LDAP系统的用户名|String||
|password|连接LDAP系统的密码|String||
|initialContextFactory|连接LDAP系统用到的initialContextFactory名字|com.sun.jndi.ldap.LdapCtxFactory|
|securityAuthentication|连接LDAP系统用到的*java.naming.security.authentication*属性值|String|simple|
|customConnectionParameters|不需要声明setter就可以设置 LDAP连接属性。可以参考http://docs.oracle.com/javase/tutorial/jndi/ldap/jndi.html 查看高级自定义属性。这些属性可以配置连接池，特定的安全设置等。所有的参数均被用来创建LDAP连接|Map<String, String>||
|baseDn|可以从中查询用户和用户组的*专有名称*|String||
|userBaseDn|可以从中查询用户的*专有名称*，如果没提供，就使用上述的 baseDN|String||
|groupBaseDn|可以从中查询用户组的*专有名称*，如果没提供，就使用上述的 baseDN|String||
|searchTimeLimit|查询LDAP的超时时间设置（毫秒数）|long|一小时|
|queryUserByUserId|通过用户用户Id查询用户的时候，调用此方法。比如：(&(objectClass=inetOrgPerson)(uid={0}))这里，LDAP中符合*uid* 条件的*inetOrgPerson*的对象都会被返回。 就像示例中的那样，userId通过{0}注入。如果该参数无法满足你的要求，你还可以使用LDAPQueryBuilder，它支持自定义配置。|String||
|queryUserByFullNameLike|使用名称查询用户的时候调用该方法。比如：(& (objectClass=inetOrgPerson) (({0}={1})({2}={3})) )。此时，LDAP中所有匹配对应姓和名的*inetOrgPerson*均会被返回。注意{0}表示firstNameAttribute（前文定义好的），{1}和{3}表示搜索内容，{2}表示lastNameAttribute。如果该参数无法满足你的要求，你还可以使用LDAPQueryBuilder，它支持自定义配置。|String||
|queryGroupsForUser|查询用户所在用户组时调用该方法。比如(&(objectClass=groupOfUniqueNames)(uniqueMember={0}))，这里，LDAP中DN（user的DN）是*uniqueMember*的*groupOfUniqueNames*对象都会被返回。就像示例里展示的那样，userId通过{0}注入。如果该参数无法满足你的要求，你还可以使用LDAPQueryBuilder，它支持自定义配置。|String||
|userIdAttribute|userId对应的参数名。当LDAP对象和ActivitiUser对象的映射完成之后，可以使用此对象来查询User对象|String||
|userFirstNameAttribute|用户firstname对应的参数名。当LDAP对象和ActivitiUser对象的映射完成之后，可以使用此对象来查询User对象|String||
|userLastNameAttribute|用户lastname对应的参数名。当LDAP对象和ActivitiUser对象的映射完成之后，可以使用此对象来查询User对象|String||
|groupIdAttribute|用户组id对应的参数名。当LDAP对象和ActivitiUser对象的映射完成之后，可以使用此对象来查询Group 对象|String||
|groupNameAttribute|用户组名称对应的参数名。当LDAP对象和ActivitiUser对象的映射完成之后，可以使用此对象来查询Group 对象|String||
|groupTypeAttribute|用户组类型对应的参数名。当LDAP对象和ActivitiUser对象的映射完成之后，可以使用此对象来查询Group 对象|String||

如果你想要自定义默认属性或者了解组缓存，可以看看下面这些属性：
|属性名|描述|类型|默认值|
|-----|---|----|-----|
|ldapUserManagerFactory|如果默认的实现不满足要求，你可以自己实现LDAPUserManagerFactory|LDAPUserManagerFactory的实例||
|ldapGroupManagerFactory|如果默认的实现不满足要求，你可以自己实现 LDAPGroupManagerFactory|LDAPGroupManagerFactory的实例||
|ldapMemberShipManagerFactory|如果默认的实现不满足要求，你可以自己实现 LDAPMembershipManagerFactory。注意，鉴于成员的关系是由LDAP系统自己维护的，该种方式比较难实现|LDAPMembershipManagerFactory |LDAPMembershipManagerFactory的实例|
|ldapQueryBuilder|如果默认的实现不满足要求，你可以使用该属性。当LDAPUserManager或LDAPGroupManage对LDAP系统执行实际查询时，会用到 LDAPQueryBuilder 的实例。默认的实现类使用该实例上设置的属性，比如queryGroupsForUser 和 queryUserById|org.activiti.ldap.LDAPQueryBuilder的实例||
|groupCacheSize|设置用户组缓存的大小。它是一个保存了用户对应用户组的LRU缓存，来减小对LDAP系统的查询。当该值小于0时，不会实例化缓存。默认是-1，所以不会缓存。|-1|
|groupCacheExpirationTime|用户组缓存失效的毫秒数。当要查询某个用户所在组的时候，如果设置了缓存，那么该用户组就会被放到缓存中，并存活该设置的时间。比如，超时时间是30分钟，当用户组是在0:00被查询到的，那么0:30以后再来查询该用户组时，就不会去缓存而是去LDAP中查找。同样地，0:00-0:30内对该用户所在组的查询都是从缓存中获取的。|long|一小时|

当使用Activiti Directory时请注意：Activiti论坛的同学反馈说，对于Activiti Directory，需要将InitialDirContext设置为Context.REFERRAL。该属性可以通过上述的customConnectionParameters映射来传递。