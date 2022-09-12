---
layout: post
title: 使用 update 条数作为业务逻辑导致的bug
categories: bug, concurrent
description: 使用 update 的影响条数作为业务逻辑判断条数，会有 bug
keywords: bug, useAffectedRows
---
# 使用update 条数作为业务逻辑判断条件导致的 bug

## 背景
1. 一张用户信息扩展表中，有一个字段为 user_no ，业务要求一个用户只有一条扩展信息
2. 运管页面，运营同学可以批量导入用户的扩展信息，以 user_no 作为索引字段
3. 因为有些用户没有扩展信息，所以在导入的时候，需要先判断用户是否有扩展信息，再选择是更新还是插入
4. 生产发现，用户连续点击的时候，会导致重复插入数据
5. 代码如下：

```java
@Resource
private UserInfoExtendMapper extendMapper;
public void saveOrUpdateExtend(UserInfoExtend userInfoExtend){
    // 业务校验略
    // 根据 userNo 直接进行更新, 如果更新到数据，则皆大欢喜；如果没更新到，则证明新数据，执行插入
    if(extendMapper.updateByUserNo(userInfoExtend) == 0){ // 标注1
        userInfoExtend.insert(userInfoExtend);
    }
}
```

## 问题原因
### 原因一：没有加锁
上述代码 [标注1] 的地方没有加锁，导致并发情况下，如果数据库里不存在该 userNo 的数据，则两个线程均判断为 true，进而都执行 insert 操作。

这个解决方式很简单，加上防重复提交即可。

### 原因二： useAffectedRows 配置有误
在日常使用 mysql 客户端执行 update 语句时，在执行完毕时，经常会有 *Affected rows: 0*  字样。

这个现象产生的原因有两种情况：

1. 根据 update 的查询条件没有查询到数据（rows matched）
2. 更新前后的字段值一样，即为无数据更新（rows changed）

navicat 客户端默认使用的是 rows changed，但是 jdbc 默认使用的是 rows matched。如果之前有人改过数据库连接配置为下文示例，则在单线程情况下也会出现前文错误。

```text
jdbc:mysql://jdbc.host/{jdbc.db}?useAffectedRows=true
```

## 总结
1. 经常使用 navicate 的缘故，让我在报错时竟然先忽略了并发因素
2. 应该没有人真的会在项目里使用 useAffectedRows = true 吧
3. 为降低未知因素影响，对于业务系统来说，还是不要使用这种方式做逻辑处理
