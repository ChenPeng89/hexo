---
title: Mybatis学习笔记：一、入门
date: 2016-05-31 10:03:09
tags: Mybatis
---
## 背景
Mybatis是支持普通SQL查询，存储过程和高级映射的优秀的持久层框架。它的前身是ibatis。
## 优势
- Mybatis框架相对简单很容易上手
- 可以针对sql进行更详细的优化
- 具有延迟加载策略
- 拥有强大易用的缓存策略

## 劣势
- 相比于Hibernate，mybatis的功能还是有些简陋
- Mybatis的延迟加载是全局设置的
- Mybatis的底层SQL需要自己定义，因此移植性不好。

## 结构
Mybatis分为三层：
![](http://i.imgur.com/aFHaNt5.jpg)
- 接口层：提供外部调用的接口，然后调用数据处理层处理数据请求。
- 数据处理层：完成参数映射，SQL解析，SQL执行，和结果的映射。主要是调用请求来完成一次数据操作。
- 基础支撑层：负责最基础的功能支撑，包括连接管理、事务管理、配置加载和缓存处理。

## 生命周期
![](http://i.imgur.com/HGZhpvi.jpg)

1. Mybatis的入口程序是SqlSessionFactoryBuilder，它通过xml配置文件来创建Configuration对象，然后通过build()来创建SqlSessionFactory对象。SqlSessionFactoryBuilder可以在创建SqlSessionFactory后就回收掉，不用一直保存。
2. SqlSessionFactory通过SqlSessionFactoryBuilder创建的Configuration对象并调用openSession()来创建SqlSession对象。
   SqlSessionFactory一旦被创建，在运行期内应该一直存在的，可以使用静态单例模式来创建它。
3. SqlSession类似于数据库的Session，不是线程安全的，每次使用后都应该关闭它。SqlSession的默认实现类是DefaultSqlSession，它有两个必须配置的属性：Configuration和Executor。SqlSession对数据库的操作都是通过Executor来完成的。
   应用程序是通过访问Mapper对象来访问sqlsession的。Mapper对象是Mybatis根据传入的接口类型和对应的mapper.xml配置文件来生成的一个代理对象。
4. Executor对象是在创建Configuration对象时创建的，它缓存在Configuration中，它的主要功能是调用数据库的statementHandler来访问数据库，并将查询结果缓存起来。
5. statement是真正访问数据库的对象，它调用ResultSetHandler来处理查询结果。

本文为笔者学习笔记，很多都是在网上查阅的资料，写出来供有需要的朋友参考。
参考：[《深入浅出Mybatis》](http://blog.csdn.net/hupanfeng/article/category/1443955)