---
title: 记一次mysql的update死锁
date: 2019-11-20
comments: true
categories:
- mysql
tags:
- 高并发
- mysql
---

2019.11.20 账户系统死锁问题排查分析。

<!-- more -->

## 问题出现
项目出现死锁告警，日志中出现报错:  
```
Error updating database. Cause: com.mysql.jdbc.exceptions.jdbc4.MySQLTransactionRollbackException: Deadlock found when trying to get lock; try restarting transaction
```   
联系dba查看死锁日志如下:  
![死锁日志](https://img-blog.csdnimg.cn/20191121125954480.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2dlcWlhbmRlYmVp,size_16,color_FFFFFF,t_70)

> mysql对死锁有两种处理方式:  
1.连接等待直到超时。  
2.开启死锁监测，发现死锁时主动回滚当前事务，让其他事务可以正常执行。（innodb_deadlock_detect=on)   
>   
>mysql查看最近的死锁日志命令: show engine innodb status;

## 背景
对业务逻辑不做赘述，简单理解为账户交易采用TCC的模式，先冻结，再确认入账或撤销。
对应的有三个事务,
* 请求处理事务: 锁定账户、请求记录入库、更新账户(冻结金额+ 可用余额 -)
* 入账确认事务: 锁定请求记录、锁定账户、更新账户(冻结金额-)
* 入账取消事务: 锁定请求记录、锁定账户、更新账户(冻结金额- 可用余额 +)

对应主要有两张表：账户主体表(account)和请求记录表(trans_log)，因为问题主要出在账户表操作上,在此只贴出账户表关键字段。如下

```sql
CREATE TABLE `account` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT COMMENT '主键字段',
  `account_no` varchar(32) NOT NULL COMMENT '账户号',
  `customer_no` varchar(32) NOT NULL COMMENT '客户号',
  `balance` decimal(15,2) NOT NULL DEFAULT '0.00' COMMENT '有效余额',
  `frozen_amount` decimal(15,2) NOT NULL DEFAULT '0.00' COMMENT '冻结余额',
  `status` varchar(32) NOT NULL COMMENT '账户状态(正常:NORMAL 冻结:FROZEN 失效:DISABLED)',
  `account_type` varchar(32) NOT NULL COMMENT '账户类型码',
  PRIMARY KEY (`id`) COMMENT '主键',
  UNIQUE KEY `uk_account_no` (`account_no`),
  UNIQUE KEY `uk_customer_acctype` (`customer_no`,`account_type`)
) ENGINE=InnoDB AUTO_INCREMENT=4491 DEFAULT CHARSET=utf8mb4 COMMENT='账户主体表';

INSERT INTO `account`(`id`,`account_no`,`customer_no`,`balance`, `frozen_amount`, `status`, `account_type`) 
VALUES (1, '888888','123456',10.00, 0.00, 'NORMAL', 'BALANCE');
```
**账户表有两个唯一索引，一是根据账户号索引，二是根据客户id和客户类型联合索引。**

![请求处理事务入账确认事务](https://img-blog.csdnimg.cn/20191121143405797.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2dlcWlhbmRlYmVp,size_16,color_FFFFFF,t_70)


## 问题排查

检测到死锁抛出异常的地方在请求处理事务的update的时候，根据死锁日志查看引起死锁的是PRIMARY主键索引和uk_uk_account_no索引。（虽然insert动作也会有加锁动作，但根据死锁日志排除，以下对insert不用关注）  

采用以下时序复现了死锁场景:
![导致死锁的事务执行时序](https://img-blog.csdnimg.cn/20191121143418584.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2dlcWlhbmRlYmVp,size_16,color_FFFFFF,t_70)

时序分析：
1. 请求处理事务: 先根据uk_customer_acctype索引进行锁定，实际上锁定了uk_customer_acctype索引簇记录和主键索引簇上对应的记录。
2. 入账确认事务: 根据唯一索引uk_account_no锁定,先对uk_account_no索引簇记录加X锁成功，对主键记录上锁时阻塞等待
3. 请求处理事务: update更新时因为是根据account_no条件进行更新，所以尝试对uk_account_no索引簇加锁,而对应的锁已经被入账确认事务占用，形成死锁环。
![死锁环](https://img-blog.csdnimg.cn/20191121144514512.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2dlcWlhbmRlYmVp,size_16,color_FFFFFF,t_70)

## 总结

注意for update对二级索引（非聚簇索引）加X锁时,也会同时对主键索引（聚簇索引）记录加锁。

![](https://oscimg.oschina.net/oscnet/be1412c6890280b7976893810e491212df3.jpg)