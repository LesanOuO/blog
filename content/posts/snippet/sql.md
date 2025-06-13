---
title: "SQL 代码片段"
date: "2022-06-16"
tags: ["SQL"]
categories: ["代码片段"]
weight:
---

## 行转列

- 使用PIVOT实现

```sql
SELECT *
FROM student
PIVOT (
    SUM(score) FOR subject IN (语文, 数学, 英语)
)
```

- 分组后使用case进行条件判断处理

当然我们也可以用 CASE WHEN 得到同样的结果，就是写起来麻烦一点。

```sql
SELECT name,
  MAX(
  CASE
    WHEN subject='语文'
    THEN score
    ELSE 0
  END) AS "语文",
  MAX(
  CASE
    WHEN subject='数学'
    THEN score
    ELSE 0
  END) AS "数学",
  MAX(
  CASE
    WHEN subject='英语'
    THEN score
    ELSE 0
  END) AS "英语"
FROM student
GROUP BY name
```

使用 CASE WHEN 可以得到和 PIVOT 同样的结果，没有 PIVOT 简单直观。

## 列转行

- 使用UNPIVOT实现

```sql
SELECT *
FROM student1
UNPIVOT (
    score FOR subject IN ("语文","数学","英语")
)
```

- 分组后使用case进行条件判断处理

我们也可以使用下面方法得到同样结果

```sql
SELECT
    NAME,
    '语文' AS subject ,
    MAX("语文") AS score
FROM student1 GROUP BY NAME
UNION
SELECT
    NAME,
    '数学' AS subject ,
    MAX("数学") AS score
FROM student1 GROUP BY NAME
UNION
SELECT
    NAME,
    '英语' AS subject ,
    MAX("英语") AS score
FROM student1 GROUP BY NAME
```

- UNION & UNION ALL

UNION 操作符用于合并两个或多个 SELECT 语句的结果集。

请注意，UNION 内部的 SELECT 语句必须拥有相同数量的列。列也必须拥有相似的数据类型。同时，每条 SELECT 语句中的列的顺序必须相同。

**SQL UNION 语法**

```sql
SELECT column_name(s) FROM table_name1
UNION
SELECT column_name(s) FROM table_name2
```

**注释：**默认地，UNION 操作符选取不同的值。如果允许重复的值，请使用 UNION ALL。

**SQL UNION ALL 语法**

```sql
SELECT column_name(s) FROM table_name1
UNION ALL
SELECT column_name(s) FROM table_name2
```

另外，UNION 结果集中的列名总是等于 UNION 中第一个 SELECT 语句中的列名。
