---
layout: wiki
title: 事务相关
categories: 事务
description: spring 的事务传播机制，数据库的事务隔离级别
keywords: spring, transaction
---
# Spring 事务传播
Spring 支持的事务粒度是方法级别。
如果事务方法 methodA() 中调用了 事务方法 methodB()，就要针对特定的场景，告诉 spring 这两个事务是如何作用的。
两个事务的作用机制，就是事务的传播机制。
不过 spring 的事务传播，主要看 methodB() 的事务如何处理，这里还包含 methodA() 没有事务的情况

# spring 事务传播属性(TransactionDefinition)
| 属性 | 值 | 含义 |
| ---- | ---- | -------------- |
| PROPAGATION_REQUIRED | 0 | 如果当前没有事务，就新建一个；如果有，就将自己加入当前的事务。这个是默认设置 |
| PROPAGATION_SUPPORTS | 1 | 如果当前没有事务，那就算了；如果有，就将自己加入当前的事务 |
| PROPAGATION_MANDATORY | 2| 如果当前没有事务，就抛出异常；如果当前有事务，就将自己加入当前的事务 |
| PROPAGATION_REQUIRES_NEW | 3 | 如果当前没有事务，就新建一个事务；如果当前有事务，就将当前事务挂起 |
| PROPAGATION_NOT_SUPPORTED | 4 | 如果当前没有事务，就正常执行；如果当前有事务，就将当前事务挂起 |
| PROPAGATION_NEVER | 5 | 如果当前没有事务，就正常执行；如果当前有事务，就抛出异常 |
| PROPAGATION_NESTED | 6 | 如果当前没有事务，就新建一个；如果有，就在嵌套事务里执行 |
| ISOLATION_DEFAULT | -1 | 使用数据库默认的事务传播机制 |

# spring 数据库事务隔离级别
| 属性 | 值 | 含义 | 影响 |
| ---- | ---- | -------------- | ---- |
| ISOLATION_DEFAULT | -1 | 使用数据库的事务隔离级别 | 参考下方具体设置
| TRANSACTION_NONE | 0 | 不使用事务 | 容易丢失数据
| ISOLATION_READ_UNCOMMITTED | 1 | 能读别人未提交的数据，会出现脏读
| TRANSACTION_READ_COMMITTED | 2 | 只能读别人提交的数据，读的时候，别人可以写，只是没提交的值，你读不到。会造成不可重复读
| TRANSACTION_REPEATABLE_READ | 4 | 可重复读，即读的时候别人不允许修改，但是别人可以插入值。会有幻读
| TRANSACTION_SERIALIZABLE | 8 | 序列化，可以认为是单线程
