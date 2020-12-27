---
title: 记一次mysql的insert死锁
date: 2020-12-05
comments: true
categories:
- mysql
tags:
- 高并发
- mysql
---

先update再insert的并发死锁问题分析。

<!-- more -->

## 背景

“如果库里有对应记录，就更新，没有就插入”

很简单的一个逻辑，相信很多人都会遇到。 

最近看一个工程里实现代码是这样的，mysql数据库走的是默认的事务级别:可重复读。包在一个事务中执行：
```sql
if update更新结果>0
  then return "成功";
else
  insert一条新记录
```

看起来似乎没什么问题，线上频频出现insert死锁。

这里总结下，分享下实验SQL，便于有兴趣研究的同学去复现。
```sql
CREATE TABLE `activity_log`(
  `id`            bigint(20) unsigned NOT NULL AUTO_INCREMENT,
  `user_id`       varchar(16) NOT NULL COMMENT '用户id',
  `activity_id`   varchar(32) NOT NULL COMMENT '活动id',
  `time_count`    int(11)     DEFAULT 0 COMMENT '活动参与次数',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uniq_uid_activity_id` (`user_id`,`activity_id`))ENGINE=InnoDB CHARSET=utf8;

//更新语句
UPDATE activity_log SET time_count=time_count+1 WHERE `user_id`='5' AND `activity_id`='2020';
//插入语句
INSERT INTO activity_log(`user_id`,`activity_id`) VALUES('5','2020');
```

## 并发执行现场

![](https://gitee.com/geqiandebei/picture/raw/master/2020-12-6/1607270021456-QQ20201205-0.png)

## 死锁日志分析

> 查看最近的死锁日志mysql命令：show engine innodb status

```sql
2020-12-04 19:35:01 7f7aeb30c700
*** (1) TRANSACTION:
TRANSACTION 25906, ACTIVE 35 sec inserting 
mysql tables in use 1, locked 1
LOCK WAIT 3 lock struct(s), heap size 360, 2 row lock(s), undo log entries 1
MySQL thread id 5509, OS thread handle 0x7f7b0d6f2700, query id 51970 localhost root update
INSERT INTO activity_log(`user_id`,`activity_id`) VALUES('4','2020')
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 44 page no 4 n bits 72 index `uniq_uid_activity_id` of table `test`.`activity_log` trx id 25906 lock_mode X insert intention waiting
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;

*** (2) TRANSACTION:
TRANSACTION 25907, ACTIVE 26 sec inserting
mysql tables in use 1, locked 1
3 lock struct(s), heap size 360, 2 row lock(s), undo log entries 1
MySQL thread id 5510, OS thread handle 0x7f7aeb30c700, query id 51971 localhost root update
INSERT INTO activity_log(`user_id`,`activity_id`) VALUES('5','2020')
*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 44 page no 4 n bits 72 index `uniq_uid_activity_id` of table `test`.`activity_log` trx id 25907 lock_mode X
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;

*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 44 page no 4 n bits 72 index `uniq_uid_activity_id` of table `test`.`activity_log` trx id 25907 lock_mode X insert intention waiting
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;

*** WE ROLL BACK TRANSACTION (2)
------------
```

**mysql死锁日志中,不同类型锁对应的日志信息如下**

1. **记录锁**(LOCK_REC_NOT_GAP): lock_mode X locks rec but not gap
2. **间隙锁**(LOCK_GAP): lock_mode X locks gap before rec
3. **Next-key**锁(LOCK_ORNIDARY): lock_mode X
4. **插入意向锁**(LOCK_INSERT_INTENTION): lock_mode X locks gap before rec insert intention

**例外情况**

如果在supremum record 上加锁，locks gap before rec 会省略掉，间隙锁会显示成 lock_mode X，插入意向锁会显示成 lock_mode X insert intention。譬如上面的
```js
RECORD LOCKS space id 44 page no 4 n bits 72 index `uniq_uid_activity_id` of table `test`.`activity_log` trx id 25907 lock_mode X
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;
```

看起来像是 Next-key 锁,但是看下面的 heap no 1 表示这个记录是 supremum record（另外.infimum record 的 heap no 为 0），所以这个锁应该看作是一个间隙锁。

>在InnoDB存储引擎中，每个数据页中有两个虚拟的行记录，用来界定记录的边界。Infimum 是比该页中任何主键值都要小的值。Supremum 指的是比任何可能打的值还要大的值。这两个值在页创建时被建立，并且任何情况下不会删除。


**最终分析** 

两个事务update不存在的记录，先后获得了间隙锁（gap锁），gap锁之间是兼容的所以在update环节不会阻塞。

二者都持有gap锁，然后去竞争插入意向锁。当存在其他会话持有gap锁的时候，当前会话申请不了插入意向锁，导致死锁。


**推荐阅读**: 

[《记一次mysql的update死锁》](https://www.qianshan.tech/mysql/%E8%AE%B0%E4%B8%80%E6%AC%A1mysql%E7%9A%84update%E6%AD%BB%E9%94%81.html)

![](https://gitee.com/geqiandebei/picture/raw/master/2020-12-6/1607270117025-QQ20201205-1.png)