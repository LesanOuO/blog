---
title: "通过SQL导出数据库表结构"
date: "2022-11-26"
tags: ["SQL"]
categories: ["实用分享"]
weight:
---

在编写数据库文档时，我们常常需要写表结构文档，这时候就需要一些方法来节约我们写文档的时间。

我通过网络上的资料，总结了 `MySQL` 及 `SQL Server` 的通过SQL语句导出数据库表结构的方法。

## MySQL

```sql
SELECT
	COLUMN_NAME 列名,
	COLUMN_TYPE 数据类型,
	DATA_TYPE 字段类型,
	CHARACTER_MAXIMUM_LENGTH 长度,
	IS_NULLABLE 是否为空,
	COLUMN_DEFAULT 默认值,
	COLUMN_COMMENT 备注
FROM
	INFORMATION_SCHEMA.COLUMNS
	WHERE
	table_schema = '数据库名称' -- 数据库名称
	AND
	table_name = '表名' -- 表名
  -- 如果不写的话，默认会查询出所有表中的数据，这样可能就分不清到底哪些字段是哪张表中的了，所以还是建议写上要导出的名名称
```

## SQL Server

```sql
SELECT
表名 = Case When A.colorder=1 Then D.name Else '' End,
表说明 = Case When A.colorder=1 Then isnull(F.value,'') Else '' End,
字段序号 = A.colorder,
字段名 = A.name,
字段说明 = isnull(G.[value],''),
标识 = Case When COLUMNPROPERTY( A.id,A.name,'IsIdentity')=1 Then '√'Else '' End,
主键 = Case When exists(SELECT 1 FROM sysobjects Where xtype='PK' and parent_obj=A.id and name in (
    SELECT name FROM sysindexes WHERE indid in( SELECT indid FROM sysindexkeys WHERE id = A.id AND colid=A.colid))) then '√' else '' end,
类型 = B.name,
占用字节数 = A.Length,
长度 = COLUMNPROPERTY(A.id,A.name,'PRECISION'),
小数位数 = isnull(COLUMNPROPERTY(A.id,A.name,'Scale'),0),
允许空 = Case When A.isnullable=1 Then '√'Else '' End,
默认值 = isnull(E.Text,'')
FROM
syscolumns A
Left Join
systypes B
On
A.xusertype=B.xusertype
Inner Join
sysobjects D
On
A.id=D.id and D.xtype='U' and D.name<>'dtproperties'
Left Join
syscomments E
on
A.cdefault=E.id
Left Join
sys.extended_properties G
on
A.id=G.major_id and A.colid=G.minor_id
Left Join
sys.extended_properties F
On
D.id=F.major_id and F.minor_id=0
where d.name='表名' --如果只查询指定表,加上此条件
Order By
A.id,A.colorder
```