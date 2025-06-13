---
title: "SQLServer一些小技巧"
date: "2022-11-26"
tags: ['SQLServer']
categories: ['编程知识']
---

## 多行转成一列（并用","分割）

```sql
SELECT NAME, STUFF (( SELECT ',' + VALUE FROM A WHERE NAME = Test.NAME FOR XML PATH ( '' ) ),1,1,'') VALUE
FROM
	A AS Test
GROUP BY
	NAME;
```

STUFF语句就是为了去掉第一个【逗号】

STUFF用法：（从原字符的第二个开始共三个字符替换为后面的字符）
```sql
SELECT STUFF('abcdef', 2, 3, 'ijklmn');
-- 查询结果：aijklmnef
```

> 其余行列转换用法请参考文章：https://www.cnblogs.com/no27/p/6398130.html

## 根据符号将一列拆分多行

```sql
select
	name,
	SUBSTRING(a.comp,number,CHARINDEX(',',a.comp+',',number)-number) as company,
from data_base a,master..spt_values
where and number >=1
	and number < len(comp)
	and type='p'
	and SUBSTRING(','+comp,number,1)=','
```

1. `SUBSTRING()`从输入字符串中的位置(从1开始计数)开始提取具有指定长度的子字符串
    `SUBSTRING(input_string, start, length)`

2. `CHARINDEX()`函数从指定位置开始搜索字符串内的子字符串
    `CHARINDEX(substring, string [, start_location])`，其中`start_location`是搜索开始的位置，可选

3. `master..spt_values`这个表主要用来保存一些枚举值
   ```sql
    --0~2047 共2048个数字
    SELECT number FROM MASTER..spt_values WHERE TYPE = 'p'
   ```

## 通过`PARSENAME`拆分字符串

注意：`PARSENAME`最多只能拆分成4个字段

`PARSENAME`默认是根据'.'进行拆分的，所以首先要做的是将字段中的其他分隔符（如‘-’）替换成'.'

```sql
DECLARE @ip NVARCHAR(200) = '192;168;1;2';
SELECT
    PARSENAME(REPLACE(@ip,';','.'), 1) AS col1, -- 2
    PARSENAME(REPLACE(@ip,';','.'), 2) AS col2, -- 1
    PARSENAME(REPLACE(@ip,';','.'), 3) AS col3, -- 168
    PARSENAME(REPLACE(@ip,';','.'), 4) AS col4; -- 192
```

## SQL Server 字符串拆分函数`Split`

```sql
create function split(
	@string varchar(255),--待分割字符串
	@separator varchar(255)--分割符
)returns @array table(item varchar(255) COLLATE Chinese_PRC_CI_AS) -- COLLATE分配排序规则
as
begin
	declare @begin int,@end int,@item varchar(255)
	set @begin = 1
	set @end=charindex(@separator,@string,@begin)
	while(@end<>0)
	begin
		set @item = substring(@string,@begin,@end-@begin)
		insert into @array(item) values(@item)
		set @begin = @end+1
		set @end=charindex(@separator,@string,@begin)
	end
	set @item = substring(@string,@begin,len(@string)+1-@begin)
	if (len(@item)>0)
		insert into @array(item) values(substring(@string,@begin,len(@string)+1-@begin))
	return
end
```

## SQL Server CROSS/OUTER APPLY

使用 APPLY 运算符(2005或以上版本)可以为实现查询操作的外部表表达式返回的每个行调用表值函数。表值函数作为右输入，外部表表达式作为左输入。通过对右输入求值来获得左输入每一行的计算结果，生成的行被组合起来作为最终输出。APPLY 运算符生成的列的列表是左输入中的列集，后跟右输入返回的列的列表。

APPLY 有两种形式： CROSS APPLY 和 OUTER APPLY。CROSS APPLY 仅返回外部表中通过表值函数生成结果集的行。OUTER APPLY 既返回生成结果集的行，也返回不生成结果集的行，其中表值函数生成的列中的值为 NULL。

看一下例子：
```sql
select * from table1 join MyFunction(1) on 1=1

-- MyFunction 的参数是一个常量，可以返回一个表。

-- 但有时候我们希望以 table1 的字段作为参数，传进函数去计算，像：

select * from table1 join MyFunction(id) on 1=1

-- 这样是会出错的。这个时候我们就可以用 apply 来实现了。例如：

select * from table1 cross apply MyFunction(id) on 1=1
```
简单的说，apply 允许我们将前面结果集每一行的数据作为参数，传递到后面的表达式，后面的表达式可以是一个表值函数，或者select结果集。

**so，当你在需要将某个字段的值作为参数使用时，或者用join实现起来比较复杂时，就可以考虑apply来实现。**

简单实例：

获得语文第一名，数学前两名，英语前三名的name，学科，分数，用cross apply实现：
```sql
SELECT b.* FROM (
select Subject='Chiness',num=1 union all
select 'Math',2 union all
select 'English',3
)a cross apply (select top(a.num) * from Students where Subject=a.Subject )b
```