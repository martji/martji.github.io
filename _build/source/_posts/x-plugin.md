---
title: MySQL X Plugin
date: 2020-07-17 22:33:10
tags:
- MySQL
categories: 
- 数据库
---

随着文档型数据库的持续火热，MySQL 也不甘寂寞，开始打起了文档型数据库的主义，全面支持 JSON 数据格式是其中一点，还有一点就是 X Plugin。

<!-- more -->

## What is X Plugin

官方对于 X Plugin 的介绍非常含糊，说了一堆使用的方式，就是不说到底是什么东西。Percona 的一篇文章对此有一个说明，引用如下：

> https://www.percona.com/blog/2019/01/07/understanding-mysql-x-all-flavors/

What does the X stand for? Basically, it is a way to name the crossover between relational and document models with extended capabilities, and the X is used for naming the three components we are describing: the Plugin, the Protocol and the DevAPI.

### X Plugin

This is the actual interface between MySQL server and the clients. By clients we can consider a variety of clients, not only the MySQL shell. It has to be installed in MySQL 5.7 versions via the INSTALL PLUGIN command but comes installed by default in MySQL 8. The plugin adds all the functionality, configuration variables, and status counters we need to use it.

It has the ability to work with both traditional SQL and Document objects, and also supports CRUD (Create, Read, Update, Delete) operations, asynchronous query execution and so on – this provides a great capacity to extend the current way we work with MySQL.

### X Protocol

This is a new client protocol created to talk between the X Plugin and Clients. I think it is fair to say this is an eXtended version of the MySQL protocol.

It was designed with the idea of having the capacity for asynchronous calls, meaning that you can send more than one query to server from same client without the need of waiting for first query to finish before sending the second and so. This improves the overall execution time by saving network round trips between clients and server.

Additionally, the protocol accepts CRUD operations and, of course, the handling of JSON documents and plain SQL. The protocol is fully implemented in MySQLShell and has several connectors for popular languages (Java and .Net for example)

### X DevAPI

The last piece of this package is the X DevAPI protocol. Probably the best documented of these pieces is the API implemented on the MySQL Shell and connectors that supports the X Protocol. This API is designed to easily write programs from a given client using some popular languages. For example, we can easily create/test a program from MySQL Shell using Python or JavaScript.

通过上面的描述，基本上可以对 X Plugin 有一个比较直观的认识：**X Plugin 是 MySQL 支持文档模型的一个扩展插件，为了保证对文档模型的操作友好，MySQL 提供了一种全新的协议 X Protocol，并且实现了一套相应的开发接口 X DevAPI**。官方的介绍文档地址如下：

- X Plugin : https://dev.mysql.com/doc/refman/8.0/en/x-plugin.html
- X Protocol : https://dev.mysql.com/doc/internals/en/x-protocol.html
- X DevAPI : https://dev.mysql.com/doc/x-devapi-userguide/en/

## How to use

关于 X Plugin 的使用，一个简单的例子如下：

> MySQL Shell Command : https://dev.mysql.com/doc/mysql-shell/8.0/en/mysql-shell-configuring.html

```shell
#mysqlsh
MySQL Shell 8.0.21

Copyright (c) 2016, 2020, Oracle and/or its affiliates. All rights reserved.
Oracle is a registered trademark of Oracle Corporation and/or its affiliates.
Other names may be trademarks of their respective owners.

Type '\help' or '\?' for help; '\quit' to exit.
 MySQL  JS > \py
Switching to Python mode...
 MySQL  Py >
 MySQL  Py > from mysqlsh import mysqlx
 MySQL  Py > mySession = mysqlx.get_session('test1:123456@127.0.0.1:49320')
 MySQL  Py > myDb = mySession.get_schema('mytest')
 MySQL  Py > myColl = myDb.create_collection('my_collection')
 MySQL  Py > myColl.add({'_id': '1', 'name': 'Laurie', 'age': 19}).execute()
Query OK, 1 item affected (0.0076 sec)
 MySQL  Py > myColl.add({'_id': '2', 'name': 'Nadya', 'age': 54}).execute()
Query OK, 1 item affected (0.0011 sec)
 MySQL  Py > myColl.add({'_id': '3', 'name': 'Lukas', 'age': 32}).execute()
Query OK, 1 item affected (0.0010 sec)
 MySQL  Py >
 MySQL  Py > myColl.find().execute()
{
    "_id": "1",
    "age": 19,
    "name": "Laurie"
}
{
    "_id": "2",
    "age": 54,
    "name": "Nadya"
}
{
    "_id": "3",
    "age": 32,
    "name": "Lukas"
}
3 documents in set (0.0004 sec)
 MySQL  Py >
 MySQL  Py > \sql
Switching to SQL mode... Commands end with ;
 MySQL  SQL >
 MySQL  SQL > \connect test1@127.0.0.1:4932
Creating a session to 'test1@127.0.0.1:4932'
Please provide the password for 'test1@127.0.0.1:4932': ******
Fetching schema names for autocompletion... Press ^C to stop.
Your MySQL connection id is 31
Server version: 8.0.18-rds-dev Source distribution
No default schema selected; type \use <schema> to set one.
 MySQL  127.0.0.1:4932 ssl  SQL > use mytest
Default schema set to `mytest`.
Fetching table and column names from `mytest` for auto-completion... Press ^C to stop.
 MySQL  127.0.0.1:4932 ssl  mytest  SQL > select * from my_collection;
+-------------------------------------------+-----+
| doc                                       | _id |
+-------------------------------------------+-----+
| {"_id": "1", "age": 19, "name": "Laurie"} | 1   |
| {"_id": "2", "age": 54, "name": "Nadya"}  | 2   |
| {"_id": "3", "age": 32, "name": "Lukas"}  | 3   |
+-------------------------------------------+-----+
3 rows in set (0.0006 sec)
 MySQL  127.0.0.1:4932 ssl  mytest  SQL > \q
Bye!
```



注意事项：

1. mysqlsh 是 MySQL 提供的一个新的命令行工具，下载地址见：https://dev.mysql.com/downloads/shell/
2. mysqlsh 提供了丰富的命令操作，详见：https://dev.mysql.com/doc/mysql-shell/8.0/en/mysql-shell-commands.html
3. 使用 mysqlsh 连接时，可能出现如下报错：

```shell
Traceback (most recent call last):
  File "<string>", line 1, in <module>
mysqlsh.DBError: MySQL Error (5001): mysqlx.get_session: Capability prepare failed for 'tls'
```

这是因为 MySQL 没有配置 ssl，本地测试的话可以修改以下两个参数，自动生成证书：

```shell
auto_detect_certs = ON
auto_generate_certs = ON
```