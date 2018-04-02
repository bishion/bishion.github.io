---
layout: wiki
title: spring 中调用多个 dao 时的事务控制
categories: 数据库
description: spring service 中调用多个 dao 时的事务控制
keywords: spring，dao, transaction, 
---
# 问题
spring 中，每个 dao 都会使用一个 connection，它怎么保证事务

# 答案
在 service 加了事务注解后，spring对于里面的多个 dao，在获取 connection 时给同一个线程返回相同的 connection

# DataSourceUtils
## doGetConnection() 代码
```java
public static Connection doGetConnection(DataSource dataSource) throws SQLException {
    Assert.notNull(dataSource, "No DataSource specified");

    ConnectionHolder conHolder = (ConnectionHolder) TransactionSynchronizationManager.getResource(dataSource);
    if (conHolder != null && (conHolder.hasConnection() || conHolder.isSynchronizedWithTransaction())) {
        conHolder.requested();
        if (!conHolder.hasConnection()) {
            logger.debug("Fetching resumed JDBC Connection from DataSource");
            conHolder.setConnection(fetchConnection(dataSource));
        }
        return conHolder.getConnection();
    }
    // Else we either got no holder or an empty thread-bound holder here.

    logger.debug("Fetching JDBC Connection from DataSource");
    Connection con = fetchConnection(dataSource);
}
```
## 代码分析
从代码中可以看出来，在获取数据库连接的时候，spring 从 TransactionSynchronizationManager 获取连接资源，获取到了之后就将该连接返回。

# TransactionSynchronizationManager
## doGetResource() 代码
```java
private static final ThreadLocal<Map<Object, Object>> resources = new NamedThreadLocal<>("Transactional resources");
private static Object doGetResource(Object actualKey) {
    Map<Object, Object> map = resources.get();
    if (map == null) {
        return null;
    }
    Object value = map.get(actualKey);
    ......
    return value;
}
```
## 代码分析
从 resources 的 ThreadLocal 类型可以看出来，它为每个线程保存了一个连接。
当获取连接的请求过来时，如果当前线程已经创建过连接，则直接将当前连接返回
那 resources 是什么时候初始化的呢

## bindResource() 代码
```java
public static void bindResource(Object key, Object value) throws IllegalStateException {
    Object actualKey = TransactionSynchronizationUtils.unwrapResourceIfNecessary(key);
    Assert.notNull(value, "Value must not be null");
    Map<Object, Object> map = resources.get();
    // set ThreadLocal Map if none found
    if (map == null) {
        map = new HashMap<>();
        resources.set(map);
    }
    ......
}
```
## 代码分析
在事务开启后，TransactionAspectSupport 会去新建事务，里面就有一个 createTransactionIfNecessary() 方法。
该方法里面在第一次开启事务连接时，会去调用bindResource() 方法，将当前的 dataSource 给 resources