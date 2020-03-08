---
title: 事务注解@Transactional
date: 2019-09-10
comments: true
categories:
- springboot
tags:
- 高并发
- mysql
excerpt: Transactional注解的使用姿势
---

<!-- more -->

# 基本使用

* 作用在方法上
* 作用在类头部
* 作用在接口中

## 隔离级别

| 隔离级别 | 说明
| -- | --
isolation = Isolation.READ_UNCOMMITTED | 读取未提交数据
isolation = Isolation.READ_COMMITTED | 读取已提交数据
isolation = Isolation.REPEATABLE_READ | 可重复读
isolation = Isolation.SERIALIZABLE | 串行化

## 传播行为

| 事务传播行为 | 说明
| -- | -- 
propagation=Propagation.REQUIRED | 如果有事务， 那么加入事务， 没有的话新建一个(默认情况)
propagation=Propagation.NOT_SUPPORTED | 容器不为这个方法开启事务
propagation=Propagation.REQUIRES_NEW | 不管是否存在事务，都创建一个新的事务，原来的挂起，新的执行完毕，继续执行老的事务
propagation=Propagation.MANDATORY | 必须在一个已有的事务中执行，否则抛出异常
propagation=Propagation.NEVER | 必须在一个没有的事务中执行，否则抛出异常(与Propagation.MANDATORY相反)
propagation=Propagation.SUPPORTS | 如果其他bean调用这个方法，在其他bean中声明事务，那就用事务。如果其他bean没有声明事务，那就不用事务


# 注解失效场景

实际使用中经常出现注解失效或者未按照设想方案生效的，整理如下:

1. @Transactional作用方法非public类型的。
2. @Transactional写在接口上, 但工程选择了cglib方式的动态代理。实际使用时建议注解写在具体的类服务方法上。  
(详情可了解spring动态代理的两种实现:**JDK动态代理**和**CGLIB动态代理**)。
3. 类中方法自调用，同一类中未加注解的方法A调用加注解的方法B导致。只有目标方法由外部调用，目标方法才由Spring生成的代理对象来管理,同一个类中的方法调用未经过事务代理逻辑包装。  
示例1:  
```java
@Transactional(isolation = Isolation.REPEATABLE_READ)
    public void test2(){
        test1();
    }

    @Transactional(rollbackFor = Exception.class,isolation = Isolation.REPEATABLE_READ,propagation = Propagation.REQUIRES_NEW)
    public void test1() {
        Account account = new Account("1993","千山");
        AccountLog log=new AccountLog();
        /***** 略 *****/
        accountRepo.insert(account);
        accountLogRepo.insert(log);
    }
```  
test2调用test1的时候，属于自调用，并未经过事务，test1的事务属性配置均不生效，**自调用也就不存在嵌套事务的概念**。 

示例2:  
```java
    public void test2(){
        test1();
    }

    @Transactional(rollbackFor = Exception.class)
    public void test1() {
        Account account = new Account("1993","千山");
        AccountLog log=new AccountLog();
        /***** 略 *****/
        accountRepo.insert(account);
        accountLogRepo.insert(log);
    }
```  
外部服务调用test2方法，因为test2没有事务注解，未经代理对象的逻辑，处理实际是没有经过事务的

# 回滚机制

默认情况下，@Transactional针对unchecked类型异常进行回滚， 即： Error 和 RuntimeException 类型异常  
需要针对其他异常回滚，通过rollbackFor注解选项配置,如   
```@Transactional( rollbackFor = Exception.class)```

![](https://img-blog.csdnimg.cn/20200130135825755.png)

***
![](https://img-blog.csdnimg.cn/20191221203110149.png)