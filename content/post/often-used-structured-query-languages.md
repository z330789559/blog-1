---
title: "Often Used Structured Query Languages"
date: 2021-09-21T18:52:13+08:00
---

## 概 述

在开发过程中，作为一个`crud boy`来说会使`sql`来操作数据库增删改查是必不可少的，这篇文章将写写日常开发中常用的`sql`语句。数据是一个抽象的的定义，所谓的数据库就是把一些元数据通过特定规律整理到一起管理起来，方便通过他特定`DDL`，`DCL`，`DML`这些特定的语句来方便管理数据。

- `DDL (data definition languages)` 方便开发者通过`SQL`来定义存储的数据格式，组织数据。
- `DML (data manipulation languages)` 允许用户对数据进行`create`、`delete`、`insert`、`updated`操作。
- `DCL (data control languages)` 可以来管理用户对数据的访问操作权限，例如检索，更新，删除 ...

## 查询语句

在日常开发过程中查询数据是应用场景最多的，查询就要使用`select`关键字，如下：

```sql
select * from tableName;
```

例如有一个表结构如下：

![用户表](https://tva1.sinaimg.cn/large/008i3skNgy1guogri7g9hj60f106zglu02.jpg)

上面是查询整张表的所有字段数据，如果需要筛选就需要制定其`field name`如下：

```sql
select name as "用户名" from TableName;
```
并且上面使用了`as`关键字并进行了别名，查询出来数据就按照别名进行展示。

如果想查询指定的数据,就要使用`where`子语句，如下：

```sql
select *  from TableName where ID = 1;
```
那么上的`SQL`语句查询出来的结果就是上图中`ID`等于`1`的一条数据。

如果需要整理结果集去重复的话在前面加入`distinct`关键字，就可以去重了。

## 查询过滤

有时候我们通过条件查询到的数据需要进行其他筛选，这个时候我们就要使用到`limit`或者`order by`进行过滤或者重新整理了。

例如我要查询表中年龄最小那个用户的名字，这里不排除年龄是重复的情况，如下：

```sql
select username from TableName  order by age desc limit 1;
```
上面就是对 `age`进行降序然后取一条数据。

```sql
select * from TableName order by age ASC;
```
对表进行按照`age`升序排序整理输出结果。

`limit`支持几种方式，`limit 2,5`意思是从第`2`行开始，往后查询`5`条数据，`limit 2 offset 3`意思是从`3`开始往后查询`2`条数据。

有时候我们查询需要使用运算符，`sql`也是支持运算符的`<`、`>`、`<>`、`<=`、'>='、`!<`、`!>`，这些都可以加在条件语句中。

字段类型也支持数学运算例如：

```sql
select * from TableName where (age+10) = 32;
```
上面一条语句就是查询`age`加`10`以后等于`32`的数据。
