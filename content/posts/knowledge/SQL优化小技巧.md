---
title: "SQL优化小技巧"
date: "2022-11-26"
tags: ['SQL']
categories: ['编程知识']
---

本篇文章主要是一些SQL优化的小技巧，为了提升SQL编写规范及性能

## 避免使用select *

在实际业务场景中，我们不一定需要查出整张表中的所有数据。如果查出了许多不必要的数据，这样会白白浪费了数据库的资源，如：内存或CPU。此外，多查出来的数据，通过网络IO传输的过程中，也会增加数据传输的时间。

还有一个最重要的问题是：`select *` 不会走覆盖索引，会出现大量的回表操作，而从导致查询sql的性能很低。

所以以后查表时，只查需要用到的字段！

```sql
select name,age from user where id=1;
```

## 用union all代替union

`union` : 可以获取排重后的数据

`union all` : 可以获取所有数据，包含重复的数据

排重的过程需要遍历、排序和比较，它更耗时，更消耗cpu资源。所以如果能用union all的时候，尽量不用union。

```sql
(select * from user where id=1)
union all
(select * from user where id=2);
```

## 小表驱动大表

小表驱动大表：也就是说用小表的数据集驱动大表的数据集。

假如有order和user两张表，其中order表有10000条数据，而user表有100条数据。

这时如果想查一下，所有有效的用户下过的订单列表。

可以使用`in`关键字实现：

```sql
select * from order
where user_id in (select id from user where status=1)
```

也可以使用`exists`关键字实现：

```sql
select * from order
where exists (select 1 from user where order.user_id = user.id and status=1)
```

在`in`与`exists`中该如何抉择呢？

因为如果SQL语句中包含了in关键字，则它会优先执行in里面的`子查询语句`，然后再执行in外面的语句。如果in里面的数据量很少，作为条件查询速度更快。

而如果sql语句中包含了exists关键字，它优先执行exists左边的语句（即主查询语句）。然后把它作为条件，去跟右边的语句匹配。如果匹配上，则可以查询出数据。如果匹配不上，数据就被过滤掉了。

这个需求中，order表有10000条数据，而user表有100条数据。order表是大表，user表是小表。如果order表在左边，则用in关键字性能更好。

总结一下：
- in 适用于左边大表，右边小表。
- exists 适用于左边小表，右边大表。

不管是用`in`，还是`exists`关键字，其核心思想都是用小表驱动大表。

## 批量操作

在需要批量插入数据时，我们不应该使用`for`循环进行逐条插入数据，因为该操作需要多次请求数据库，才能完成批量数据插入。

但众所周知，我们在代码中，每次远程请求数据库，是会消耗一定性能的。而如果我们的代码需要请求多次数据库，才能完成本次业务功能，势必会消耗更多的性能。

这时候我们就需要创建一个批量插入数据的方法：

```sql
-- orderMapper.insertBatch(list):
insert into order(id,code,user_id)
values(123,'001',100),(124,'002',100),(125,'003',101);
```

这样只需要远程请求一次数据库，sql性能会得到提升，数据量越多，提升越大。

但需要注意的是，不建议一次批量操作太多的数据，如果数据太多数据库响应也会很慢。批量操作需要把握一个度，建议每批数据尽量控制在500以内。如果数据多于500，则分多批次处理。

## 多用limit

有时候，我们需要查询某些数据中的第一条，比如：查询某个用户下的第一个订单，想看看他第一次的首单时间。


```sql
select id, create_date
 from order
where user_id=123
order by create_date asc
limit 1;
```

使用 `limit 1`，只返回该用户下单时间最小的那一条数据即可。

> 此外，在删除或者修改数据时，为了防止误操作，导致删除或修改了不相干的数据，也可以在sql语句最后加上limit

```sql
update order set status=0,edit_time=now(3)
where id>=100 and id<200 limit 100;
-- 这样即使误操作，比如把id搞错了，也不会对太多的数据造成影响
```

## 增量查询

有时候，我们需要通过远程接口查询数据，然后同步到另外一个数据库。

如果直接获取所有的数据，然后同步过去。这样虽说非常方便，但是带来了一个非常大的问题，就是如果数据很多的话，查询性能会非常差。

```sql
select * from user
where id>#{lastId} and create_time >= #{lastCreateTime}
limit 100;
```

按id和时间升序，每次只同步一批数据，这一批数据只有100条记录。每次同步完成之后，保存这100条数据中最大的id和时间，给同步下一批数据的时候用。

通过这种增量查询的方式，能够提升单次查询的效率。

## 高效的分页

有时候，列表页在查询数据时，为了避免一次性返回过多的数据影响接口性能，我们一般会对查询接口做分页处理。

在MySQL中分页一般用`limit`关键字，如果表中数据量少，用limit关键字做分页，没啥问题。但如果表中数据量很多，用它就会出现性能问题。

例如：

```sql
select id,name,age
from user limit 1000000,20;
-- MySQL会查到1000020条数据，然后丢弃前面的1000000条，只查后面的20条数据，这个是非常浪费资源的
```

这时候，海量数据该如何分页呢？

```sql
select id,name,age
from user where id > 1000000 limit 20;
```

先找到上次分页最大的id，然后利用id上的索引查询。不过该方案，要求id是连续的，并且有序的。

还能使用between优化分页：

```sql
select id,name,age
from user where id between 1000000 and 1000020;
-- 需要注意的是between要在唯一索引上分页，不然会出现每页大小不一致的问题
```

## 用连接查询代替子查询

MySQL中如果需要从两张以上的表中查询出数据的话，一般有两种实现方式：`子查询` 和 `连接查询`。

子查询的例子如下：

```sql
select * from order
where user_id in (select id from user where status=1)
```

子查询语句可以通过`in`关键字实现，一个查询语句的条件落在另一个select语句的查询结果中。程序先运行在嵌套在最内层的语句，再运行外层的语句。

子查询语句的优点是简单，结构化，如果涉及的表数量不多的话。

但缺点是MySQL执行子查询时，需要创建临时表，查询完毕后，需要再删除这些临时表，有一些额外的性能消耗。

这时可以改成连接查询。具体例子如下：

```sql
select o.* from order o
inner join user u on o.user_id = u.id
where u.status=1
```

## join的表不宜过多

根据阿里巴巴开发者手册的规定，join表的数量不应该超过 `3` 个。

如果join太多，MySQL在选择索引的时候会非常复杂，很容易选错索引。并且如果没有命中中，nested loop join 就是分别从两个表读一行数据进行两两对比，复杂度是 n^2。

如果实现业务场景中需要查询出另外几张表中的数据，可以在a、b、c表中`冗余专门的字段`，比如：在表a中冗余d_name字段，保存需要查询出的数据。

不过有些ERP系统，并发量不大，但业务比较复杂，需要join十几张表才能查询出数据。

所以join表的数量要根据系统的实际情况决定，不能一概而论，尽量越少越好。

## join时要注意

在涉及到多张表联合查询的时候，一般会使用join关键字。

而join使用最多的是left join和inner join。

- left join：求两个表的交集外加左表剩下的数据。
- inner join：求两个表交集的数据。

使用inner join的示例如下：

```sql
select o.id,o.code,u.name
from order o
inner join user u on o.user_id = u.id
where u.status=1;

-- 如果两张表使用inner join关联，MySQL会自动选择两张表中的小表，去驱动大表，所以性能上不会有太大的问题
```

使用left join的示例如下：

```sql
select o.id,o.code,u.name
from order o
left join user u on o.user_id = u.id
where u.status=1;

-- 如果两张表使用left join关联，MySQL会默认用left join关键字左边的表，去驱动它右边的表。如果左边的表数据很多时，就会出现性能问题
```

> 要特别注意的是在用left join关联查询时，左边要用小表，右边可以用大表。如果能用inner join的地方，尽量少用left join

## 控制索引的数量

众所周知，索引能够显著的提升查询SQL的性能，但索引数量并非越多越好。

因为表中新增数据时，需要同时为它创建索引，而索引是需要额外的存储空间的，而且还会有一定的性能消耗。

阿里巴巴的开发者手册中规定，单表的索引数量应该尽量控制在 `5` 个以内，并且单个索引中的字段数不超过 `5` 个。

MySQL使用的B+树的结构来保存索引的，在insert、update和delete操作时，需要更新B+树索引。如果索引过多，会消耗很多额外的性能。

那么，问题来了，如果表中的索引太多，超过了5个该怎么办？

这个问题要辩证的看，如果你的系统并发量不高，表中的数据量也不多，其实超过5个也可以，只要不要超过太多就行。

但对于一些高并发的系统，请务必遵守单表索引数量不要超过5的限制。

那么，高并发系统如何优化索引数量？

能够建联合索引，就别建单个索引，可以删除无用的单个索引。

将部分查询功能迁移到其他类型的数据库中，比如：Elastic Seach、HBase等，在业务表中只需要建几个关键索引即可。

## 选择合理的字段类型

`char` ：表示固定字符串类型，该类型的字段存储空间的固定的，会浪费存储空间。

`varchar` ：表示变长字符串类型，该类型的字段存储空间会根据实际数据的长度调整，不会浪费存储空间。

如果是长度固定的字段，比如用户手机号，一般都是11位的，可以定义成char类型，长度是11字节。

但如果是企业名称字段，假如定义成char类型，就有问题了。

如果长度定义得太长，比如定义成了200字节，而实际企业长度只有50字节，则会浪费150字节的存储空间。

如果长度定义得太短，比如定义成了50字节，但实际企业名称有100字节，就会存储不下，而抛出异常。

所以建议将企业名称改成varchar类型，变长字段存储空间小，可以节省存储空间，而且对于查询来说，在一个相对较小的字段内搜索效率显然要高些。

我们在选择字段类型时，应该遵循这样的原则：

1. 能用数字类型，就不用字符串，因为字符的处理往往比数字要慢。
2. 尽可能使用小的类型，比如：用bit存布尔值，用tinyint存枚举值等。
3. 长度固定的字符串字段，用char类型。
4. 长度可变的字符串字段，用varchar类型。
5. 金额字段用decimal，避免精度丢失问题。

## 提升group by的效率

我们有很多业务场景需要使用group by关键字，它主要的功能是去重和分组。

通常它会跟`having`一起配合使用，表示分组后再根据一定的条件过滤数据，使用它就会有一些性能上的隐患。

```sql
select user_id,user_name from order
group by user_id
having user_id <= 200;
-- 这种写法性能不好，它先把所有的订单根据用户id分组之后，再去过滤用户id大于等于200的用户
```

分组是一个相对耗时的操作，为什么我们不先缩小数据的范围之后，再分组呢？

```sql
select user_id,user_name from order
where user_id <= 200
group by user_id
```

使用where条件在分组前，就把多余的数据过滤掉了，这样分组时效率就会更高一些。

> 其实这是一种思路，不仅限于group by的优化。我们的sql语句在做一些耗时的操作之前，应尽可能缩小数据范围，这样能提升sql整体的性能

## 索引优化

很多时候SQL语句，走了索引，和没有走索引，执行效率差别很大。所以`索引优化`被作为SQL优化的首选。

索引优化的第一步是：检查sql语句有没有走索引。

可以使用[explain命令](https://mp.weixin.qq.com/s?__biz=MzkwNjMwMTgzMQ==&mid=2247490262&idx=1&sn=a67f610afa984ecca130a54a3be453ab&source=41#wechat_redirect)，查看mysql的执行计划。

例如：
```sql
explain select * from `order` where code='002';
```

结果：
![](https://cdn.jsdelivr.net/gh/LesanOuO/images@master/img/20221114110010.png)

通过这几列可以判断索引使用情况，执行计划包含列的含义如下图所示：
![](https://cdn.jsdelivr.net/gh/LesanOuO/images@master/img/20221114110045.png)

说实话，sql语句没有走索引，排除没有建索引之外，最大的可能性是索引失效了。

下面说说索引失效的常见原因：
![](https://cdn.jsdelivr.net/gh/LesanOuO/images@master/img/20221114110125.png)

如果不是上面的这些原因，则需要再进一步排查一下其他原因。

此外，你有没有遇到过这样一种情况：明明是同一条sql，只有入参不同而已。有的时候走的索引a，有的时候却走的索引b？

没错，有时候MySQL会选错索引。

必要时可以使用`force index`来强制查询sql走某个索引。