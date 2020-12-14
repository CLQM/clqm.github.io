---
layout: post
title: "MySQL 查询语句如何执行的"
subtitle: 'MySQL select'
author: "cslqm"
header-style: text
tags:
  - MySQL
---

# 查询语句如何执行的

## 建立连接

数据库，长连接是指连接成功后，如果客户端持续有请求，则一直使用同一个连接。短连接则是每次执行完很少的几次查询就断开连接。

建立连接的过程通常比较复杂，一般推荐用长连接。

全部用长连接后，MySQL 内存占用涨得快，MySQL 在执行过程中临时使用的内存是管理在连接对象里面的。资源只有在连接断开后才释放。
如何解决内存占用问题呢？
1.定期断开长连接。使用一段时间，或者程序里面判断执行过一个占用内存的大查询后，断开连接。
2.使用 MySQL>=5.7，可以在每次执行一个大操作后，通过执行 mysql_reset_connection 来重新初始化连接资源。


## 查询缓存

MySQL 拿到一个查询请求后，会先到查询缓存看看，之前是否执行过这个语句。之前执行过的语句及其结果可能会以 key-value 对的形式，被缓存在内存中。key 是查询的语句，value 是查询的结果。

大多数情况是不建议使用查询缓存。

查询缓存的失效非常频繁，只要有对一个表的更新，这个表的所有查询缓存都会被清空。对于更新压力大的数据库，查询缓存的命中率非常低。如果业务就是一个静态表，很长时间会更新一次，比如系统配置表。

MySQL 8.0 将查询缓存功能删除了。

## 分析器
词法分析 语法分析

## 优化器
表内有多个索引的时候，决定使用那个索引；或者在一个语句有多表关联（join）的时候，决定各个表的顺序。

``` sql
mysql> select * from t1 join t2 using(ID) where t1.c=10 and t2.d=20;
```

这个语句，既可以先从表 t1 里面取出 c=10 的记录的 ID 值，再根据 ID 值关联到表 t2，再判断 t2 里面 d 的值是否等于 20。也可以先从表 t2 里面取出 d=20 的记录的 ID 值，再根据 ID 值关联到 t1，再判断 t1 里面 c 的值是否等于 10。


## 执行器

开始执行时，先判断你是否对这个表有没有执行查询的权限。如果是命中查询缓存，会在返回缓存时检查权限。

select * from T where ID=10;

1. 调用 InnoDB 引擎接口取这个表的第一行，判断 ID 值是否为 10，不是就跳过，是则将这行存在结果集中；
2. 调用引擎接口取“下一行”，重复相同的判断逻辑，直到最后表的一行；
3. 执行器将所有满足的行组合成的结果集返回客户端。