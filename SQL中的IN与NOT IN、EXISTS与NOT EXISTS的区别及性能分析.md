# in 和 exists

- in 是把外表和内表作 hash 连接，而 exists 是对外表作 loop 循环，每次 loop 循环再对内表进行查询，一直以来认为 exists 比 in 效率高的说法是不准确的。
- **如果查询的两个表大小相当，那么用 in 和 exists 差别不大；如果两个表中一个较小一个较大，则子查询表大的用 exists，子查询表小的用 in；**



# not in 和 not exists

- not in 逻辑上不完全等同于 not exists，如果你误用了 not in，小心你的程序存在致命的 BUG
- 请尽量不要使用 not in (它会调用子查询)，而尽量使用 not exists（它会调用关联子查询）。
- 如果子查询中返回的任意一条记录含有空值，则查询将不返回任何记录。
- 如果子查询字段有非空限制，这时可以使用 not in，并且可以通过提示让它用 hasg_aj 或 merge_aj 连接。
- 如果查询语句使用了 not in，那么对内外表都进行全表扫描，没有用到索引；
- 而 not exists 的子查询依然能用到表上的索引。所以无论哪个表大，用 not exists 都比 not in 要快。



# in 与 = 的区别

` name in('zhang','wang','zhao')`  与 `name='zhang' **or** name='wang' **or** name='zhao'`的结果是相同的。



# 其他分析：

EXISTS 的执行流程

```sql
select * from t1 where exists ( select null from t2 where y = x ) 
```

可以理解为

```
for x in ( select * from t1 ) loop 
  if ( exists ( select null from t2 where y = x.x ) then 
  	OUTPUT THE RECORD 
  end if 
end loop 
```



### 1. 对于 in 和 exists 的性能区别

-  如果子查询得出的结果集记录较少，主查询中的表较大且又有索引时应该用 in, 反之如果外层的主查询记录较少，子查询中的表大，又有索引时使用 exists。
- 其实我们区分 in 和 exists 主要是造成了驱动顺序的改变（这是性能变化的关键），如果是 exists，那么以外层表为驱动表，先被访问，如果是 IN，那么先执行子查询，所以我们会以驱动表的快速返回为目标，那么就会考虑到索引及结果集的关系了
- 另外 IN 时不对 NULL 进行处理 `select 1 from dual where null in (0,1,2,null)` 为空



### 2. NOT IN 与 NOT EXISTS

NOT EXISTS 的执行流程

```
select ..... from rollup R  where not exists ( select 'Found' from title T where R.source_id = T.Title_ID);
```

可以理解为

```
for x in ( select * from rollup ) loop 
  if ( not exists ( that query ) ) then 
  	OUTPUT 
  end if; 
end loop; 
```

**NOT EXISTS 与 NOT IN 不能完全互相替换，看具体的需求。如果选择的列可以为空，则不能被替换。**

### 对于 not in 和 not exists 的性能区别：

- not in 只有当子查询中，select 关键字后的字段有 not null 约束或者有这种暗示时用 not in, 另外如果主查询中表大，子查询中的表小但是记录多，则应当使用 not in, 并使用 anti hash join.
- 如果主查询表中记录少，子查询表中记录多，并有索引，可以使用 not exists, 另外 not in 最好也可以用 `/*+ HASH_AJ */` 或者外连接 + is null
- NOT IN 在基于成本的应用中较好
- IN 适合于外表大而内表小的情况；EXISTS 适合于外表小而内表大的情况。

```
select * 
from t1, ( select distinct y from t2 ) t2 
where t1.x = t2.y;
```

—— 如果你有一定的 SQL 优化经验，从这句很自然的可以想到 t2 绝对不能是个大表，因为需要对 t2 进行全表的 “唯一排序”，如果 t2 很大这个排序的性能是 不可忍受的。但是 t1 可以很大，为什么呢？最通俗的理解就是因为 t1.x=t2.y 可以走索引。

但这并不是一个很好的解释。试想，如果 t1.x 和 t2.y 都有索引，我们知道索引是种有序的结构，因此 t1 和 t2 之间最佳的方案是走 merge join。另外，如果 t2.y 上有索引，对 t2 的排序性能也有很大提高。

```
select * from t1 where exists ( select null from t2 where y = x ) 
```

可以理解为

```
for x in ( select * from t1 ) 
  loop 
  if ( exists ( select null from t2 where y = x.x ) 
    then 
    OUTPUT THE RECORD! 
  end if 
end loop
```

—— 这个更容易理解，t1 永远是个表扫描！因此 t1 绝对不能是个大表，而 t2 可以很大，因为 y=x.x 可以走 t2.y 的索引。
