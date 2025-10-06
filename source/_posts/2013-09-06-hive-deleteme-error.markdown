---
layout: post
title: "Hive 并发情况下报 DELETEME 表不存在的异常"
date: 2013-09-06 11:20
comments: true
categories: [Big Data]
tag: [hive]
published: true
---

在每天运行的 Hive 脚本中，偶尔会抛出以下错误：

```
2013-09-03 01:39:00,973 ERROR parse.SemanticAnalyzer (SemanticAnalyzer.java:getMetaData(1128)) - org.apache.hadoop.hive.ql.metadata.HiveException: Unable to fetch table dw_xxx_xxx
        at org.apache.hadoop.hive.ql.metadata.Hive.getTable(Hive.java:896)
        ...
Caused by: javax.jdo.JDODataStoreException: Exception thrown obtaining schema column information from datastore
NestedThrowables:
com.mysql.jdbc.exceptions.jdbc4.MySQLSyntaxErrorException: Table 'hive.DELETEME1378143540925' doesn't exist
        at org.datanucleus.jdo.NucleusJDOHelper.getJDOExceptionForNucleusException(NucleusJDOHelper.java:313)
        ...
```

查阅了网上的资料，是 DataNucleus 的问题。

## 问题背景
背景1：我们知道 MySQL 中的库表信息是存放在 `information_schema` 库中的，Hive 也有类似的机制，它会将库表信息存放在一个第三方的 RDBMS 中，目前我们线上配置的是本机 MySQL，即：

```bash
$ mysql -uhive -ppassword hive
```

![使用 DBS 表存放元数据](/images/hive-deleteme-error/dbs.png)

<!--more-->

背景2：Hive 使用的是 DataNuclues ORM 库来操作数据库的，而基本上所有的 ORM 框架（对象关系映射）都会提供自动建表的功能，即开发者只需编写 Java 对象，ORM 会自动生成 DDL。DataNuclues 也有这一功能，而且它在初始化时会通过生成临时表的方式来获取数据库的 Catalog 和 Schema，也就是 `DELETEME` 表：

![通过 DELETEME 表获取数据库信息](/images/hive-deleteme-error/deleteme.png)

这样就有一个问题：在并发量大的情况下，`DELETEME` 表名中的毫秒数可能相同，那在 `pt.drop(conn)` 的时候就会产生找不到表的报错。

## 解决办法
解决办法已经可以在代码中看到了：将 `datanucleus.fixedDataStore` 选项置为 `true`，即告知 DataNuclues 该数据库的表结构是既定的，不允许执行 DDL 操作。

这样配置会有什么问题？让我们回忆一下 Hive 的安装步骤：

1. 解压 hive-xyz.tar.gz
2. 在 conf/hive-site.xml 中配置 Hadoop 以及用于存放库表信息的第三方数据库
3. 执行 `bin/hive -e` 即可使用。DataNucleus 会按需创建上述的 `DBS` 等表

这对新手来说很有用，因为不需要手动去执行建表语句，但对生产环境来说，普通帐号是没有 DDL 权限的，我们公司建表也都是提 DB-RT 给 DBA 操作。同理，线上 Hive 数据库也应该采用手工创建的方式，导入 scripts/metastore/upgrade/mysql/hive-schema-0.9.0.mysql.sql 文件即可。这样一来，就可以放心地配置 `datanucleus.fixedDataStore` 以及 `datanecleus.autoCreateSchema` 两个选项了。

这里我们也明确了一个问题：设置 `datanucleus.fixedDataStore=true` 不会影响 Hive 建库建表，因为 Hive 中的库表只是 `DBS`、`TBLS` 表中的一条记录而已。

建议的操作：

1. 在线上导入 hive-schema-0.9.0.mysql.sql，将尚未创建的表创建好（比如我们没有用过 Hive 的权限管理，所以 DataNucleus 没有自动创建 `DB_PRIVS` 表）
2. 在 hive-site.xml 中配置 `datanucleus.fixedDataStore=true`、`datanecleus.autoCreateSchema=false`

这样就可以彻底解决这个异常了。

为什么 HWI(Hive Web Interface) 没有遇到类似问题？因为它是常驻内存的，`DELETEME` 表只会在启动的时候创建，后续的查询不会创建。而我们这里每次调用 `hive` 命令行都会去创建，所以才有这样的问题。

## 参考链接
* http://www.cnblogs.com/ggjucheng/archive/2012/07/25/2608633.html
* https://github.com/dianping/cosmos-hive/issues/10
* https://issues.apache.org/jira/browse/HIVE-1841
