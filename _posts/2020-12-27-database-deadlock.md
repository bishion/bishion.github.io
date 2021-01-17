---
layout: post
title: 记一次死锁的排查过程
categories: database
description: 非主键索引引起的死锁问题排查
keywords: database, mysql, deadlock, index merge
---
# 一次死锁的排查

前几天，生产环境报了一个死锁异常：Deadlock found when trying to get lock; try restarting transaction。经过排查，我发现其实是业务逻辑无脑使用 mybatis 代码插件生成的 sql 导致的，这里简单叙述下当时的排查经历，希望其他同学后续能避免踩坑。
# 背景
## 数据库信息
| 数据库及版本 | 数据库引擎 | 事务隔离级别 | 数据库中间件 |
| --- | --- | --- | --- |
| MySQL 5.7.14.5 | InnoDB | READ-COMMITTED | TDDL5 |

## 数据信息
### 表信息
| 表名称 | 主键 | 索引：idx_student_id | 索引2（idx_subject_no） |
| --- | --- | --- | --- |
| exam | id | student_id | class_no |

### 表数据
| id | subject | score | student_id | subject_no | error_msg |
| --- | --- | --- | --- | --- | --- |
| 123 | 语文 | 70 | 2333 | 5 | 补考 |
| 456 | 语文 | 80 | 6666 | 5 |  |

### 死锁sql
```sql
update student set id = 123 , subject='语文', score = 80,student_id = 2333, subject_no = 5 
where student_id = 2333 and subject_no = 5; --- sql1

update student set id = 456 , subject='语文', score = 90,student_id = 6666, subject_no = 5 
where student_id = 6666 and subject_no = 5; --- sql2
```
## 业务处理场景

1. exam 表是学生成绩表，其中 student_id(学生ID) 和 subject_no(科目代码) 具有唯一性
1. 考试当场考试成绩下来后，会将学生及其成绩信息插入到 exam表中
1. 一次语文考试标准答案错误，系统需要要将一部分学生的语文成绩进行修改
1. 历史原因，系统处理时，是先根据 student_id 和 subject_no 查询到所有信息，修改部分字段后，然后统一 update 
1. 由于使用了 mybatis 代码生成插件，所以 sql 语句中有 _set id = ** _的情况
# 排查
## 猜想是否是查询条件死锁
刚好系统有两个查询条件，又刚好出现了死锁，因此，我们第一个反应是，两个事务获取两个查询条件的锁时，互相拿到了对方想要的字段，则死锁就出现了。
很明显，这里 student_id 值是不一样的，不存在 _对方想要 _这一条件，所以该假设不成立。
有同学可能会问了，那如果 student_id 一样，是否就会死锁呢？这个问题我们后面回答。
## 分析数据库死锁日志
```
2020-12-16T16:28:12.888779+08:00 2450685 [Note] InnoDB: *** (1) TRANSACTION:
*** (1) HOLDS THE LOCK(S):
RECORD LOCKS space id 28 page no 395 n bits 744 index idx_subject_no of table `school`.`exam` trx id 409558534 lock_mode X locks rec but not gap
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 28 page no 449 n bits 128 index PRIMARY of table `school`.`exam` trx id 409558534 lock_mode X locks rec but not gap waiting

2020-12-16T16:28:12.889469+08:00 2450685 [Note] InnoDB: *** (2) TRANSACTION:
*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 28 page no 449 n bits 120 index PRIMARY of table `school`.`exam` trx id 409558535 lock_mode X locks rec but not gap
*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 28 page no 395 n bits 744 index idx_subject_no of table `school`.`exam` trx id 409558535 lock_mode X locks rec but not gap waiting
```
经过查看死锁日志，我们很容易发现：
1. 死锁出现在行级锁（RECORD LOCKS）中
1. 该 sql 没有间隙锁（not gap）问题
1. 事务1 拿到了索引 idx_subject_no 的锁，等待主键索引的锁
1. 事务2 拿到了主键索引的锁，等待 idx_subject_no 的锁

那么我们就知道了，死锁是在 subject_no 与 id 产生的。但是问题也来了，查询条件中并没有 id ，怎么会有死锁呢？
## 为何要获取 id 的锁
本来查询条件中并没有 id ，现在却要去获取 id 的锁。问题在哪儿呢，难道是因为 sql 中有 _set id = 123_ 的操作？
参考一篇文章（[http://www.hollischuang.com/archives/914](http://www.hollischuang.com/archives/914)）所说：
> 在MySQL中，行级锁并不是直接锁记录，而是锁索引。索引分为主键索引和非主键索引两种，如果一条sql语句操作了主键索引，MySQL就会锁定这条主键索引；**如果一条语句操作了非主键索引，MySQL会先锁定该非主键索引，再锁定相关的主键索引**。 在UPDATE、DELETE操作时，MySQL不仅锁定WHERE条件扫描过的所有索引记录，而且会锁定相邻的键值，即所谓的next-key locking。
> 

> 当两个事务同时执行，一个锁住了主键索引，在等待其他相关索引。另一个锁定了非主键索引，在等待主键索引。这样就会发生死锁。

从引用资料里这里加粗文字就很容易看出，获取 id 的锁是因为我们操作了非主键索引。而且，非主键索引的操作，加锁方式会分为两步：先锁该索引，再锁主键。
联系到前文的死锁信息（idx_subject_no 与 PRIMARY），我们还是有疑问：明明 student_id 字段也有索引，而且在查询条件中排在前面，为何却是 subject_no 引发了死锁呢？

## 同一条查询 sql 会走两个索引吗
参考一篇文章（[https://www.cnblogs.com/digdeep/archive/2015/11/18/4975977.html](https://www.cnblogs.com/digdeep/archive/2015/11/18/4975977.html)）
> MySQL5.0之前，一个表一次只能使用一个索引，无法同时使用多个索引分别进行条件扫描。但是从5.1开始，引入了 index merge 优化技术，对同一个表可以使用多个索引分别进行条件扫描。

这里我们可以看出，在 mysql 5.1 后一条查询 sql 会走两个索引（虽然在 AND 条件下有点画蛇添足）。另外经过执行计划验证可知，优化器会自动将 **区分度高的索引字段** 自动放在前面。这里就回答了上文说的 student_id 相同并不会死锁的问题，因为加锁的顺序并不会乱，因而就不会死锁

## 为什么不是 student_id
虽然 student_id 作为非主键索引，而且还排在前面，但是在当前业务场景中，它们要改的是完全不同的数据库记录，所以不会引发 id 锁的冲突。
反而是 subject_no 是相同的，其对应多条 id ，则有可能引发冲突

# 死锁的过程
我们先捋一下整个加锁的过程：
1. 因 index merge, 该 sql 会将两个非主键索引分别加锁
1. 事务1 锁了 idx_student_id(2333)
1. 事务1 锁了 idx_student_id 相应的 id(123)
1. **事务1 锁了 idx_subject_no(5) --- 节点A**
1. **事务1 等待 idx_subject_no 对应的 id(456, 不用关心123，因为已经被锁了)  -- 节点B**
1. 事务2 锁了 idx_student_id(6666)
1. **事务2 锁了 idx_student_id 相应的 id(456)  --- 节点C**
1. **事务2 等待 idx_subject_no(5)  --- 节点D**

为了直观，我们可以将两个事务的加锁过程用表格对应起来：

| 序号\事务 | 事务1 | 事务2 |
| --- | --- | --- |
| 1 | 锁 idx_student_id(2333) |  |
| 2 | 锁 idx_student_id 对应的 id(123) | 锁 idx_student_id(6666) |
| 3 | 锁 idx_subject_no(5) | 锁 idx_student_id 相应的 id(456) |
| 4 | 等待锁 idx_subject_no 对应id(456) | 等待锁 idx_subject_no(5) |

通过上面的梳理，我们很容易发现，student_id 不是死锁的直接原因 , 却是最重要的推手。
# 问题根源

1. mybatis 自动生成的 sql 比较机械，比如更新字段的判空，要么都判要么都不判，甚至还把 id 也放进更新字段
1. 开发过程中，没有仔细考虑性能问题以及异常情况，机械使用自动生成的语句
1. 数据库的 index merge 会对 sql 做优化，将区分度高的索引条件放前面
# 解决方案

1. 就目前这个案例来说，只需要关闭 index merge 即可
1. 为避免后续未知问题，我们生产程序还是使用 id 作为更新条件
1. 考虑到规范和性能，这里不再使用自动生成的模板，而是结合具体场景做自定义优化
# 总结

1. innodb 的行锁是通过锁 id 来实现的
   1. 查询条件中有 id ，就直接锁 id
   1. 查询条件中有非主键索引，就先锁该索引，然后找到对应的 id 后，再锁 id
   1. 查询条件中没有任何索引，则直接锁表
2. innodb 对于查询条件中的索引加锁是从左至右的，不会乱序,不会乱序
2. 事务隔离级别为 Read-Commit 的时候，不会使用间隙锁
2. mybatis 自动生成插件在项目初始化时比较省事，但是后续使用时不要为了适配它去修改自己的实现逻辑