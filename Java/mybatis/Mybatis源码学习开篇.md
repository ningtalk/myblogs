# Mybatis源码学习开篇

## 一、前言
Mybatis是一款开源的优秀的持久层框架，也是国内当前主流的持久层框架。  
那么什么是持久层？  
所谓"持久"就是将数据保存到可断电式存储设备中以便今后使用，简单的说，就是将内存中的数据保存到关系型数据库、文件系统、消息队列等提供持久化支持的设备中。  
持久层就是系统中专注于实现数据持久化的相对独立的层面。  
作为持久层框架，Mybatis之所以能被广泛使用的主要原因：  
* 使用方便，上手容易
* 消除了jdbc代码冗余，实现sql语句与代码的分离
* 很好的与spring框架融合使用

> Mybatis发展历史：2002年由cliton Begin开发的ibatis框架，不久后捐献给Apache软件基金会；2010年，改名为Mybatis.

## JDBC
JDBC，Java Database Connectivity的缩写，Java提供的访问关系型数据库的接口。  
使用JDBC操作数据源的主要步骤是，建立数据源连接 -> 执行SQL语句 -> 处理SQL执行结果 -> 关闭连接。  
整个流程中涉及到的主要接口包括Connection、Statement、ResultSet。

> Connection  
表示JDBC驱动与数据源建立的连接，有两种方式可以获取Connection对象，可以通过JDBC提供的DriverManager类获取；也可以通过DataSource接口的实现类获取。
常用的是通过后者获取。

> Statement
Statement接口用于执行SQL语句，子类有PreparedStatement和CallableStatement。

> ResultSet
用于检索和操作SQL执行结果。


