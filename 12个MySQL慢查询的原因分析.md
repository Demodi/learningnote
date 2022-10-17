# 12个MySQL慢查询的原因分析

## 1. SQL 没加索引

   - 参考搜索条件，在合适的列**建⽴索引**，尽量避免全表扫描 

## 2. 索引不⽣效的⼗⼤经典场景
### 2.1 隐式的类型转换，索引失效

- 查询条件按照字段类型传值 

### 2.2 查询条件包含 or，可能导致索引失效 

- 其中⼀个有索引，另⼀个没有索引：全表扫描 + 索引扫描 + 合并 -> 直接 **全表扫描** 
- or 的列都加了索引，**索引可能会⾛也可能不⾛**

### 2.3 like 通配符可能导致索引失效 

1. 使⽤覆盖索引 （覆盖索引（covering index ，或称为索引覆盖）即从非主键索引中就能查到的记录，而不需要查询主键索引中的记录，避免了回表的产生减少了树的搜索次数，显著提升性能。）【覆盖索引：在查询的数据列里面，不需要回表去查，直接从索引列就能取到想要的结果。换句话说，你SQL用到的索引列数据，覆盖了查询结果的列，就算上覆盖索引了。】
2. 把 % 放后⾯ 

### 2.4 查询条件不满⾜联合索引的最左匹配原则 

- MySQL 建⽴联合索引时，会遵循最左前缀匹配的原则，即最左优先。如果你建⽴⼀个 （a,b,c）的联合索引，相当于建⽴了 (a)、(a,b)、(a,b,c) 三个索引。 
- 建⽴符合最左前缀匹配的索引 

### 2.5 在索引列上使⽤ MySQL 的内置函数

- **内置函数的逻辑转移到传值⼀侧**，即不直接对数据列使用函数或者计算 

### 2.6 对索引进⾏列运算（如 ：+、-、*、/） 

- **不可以对索引列进⾏运算**，可以在代码处理好，再传参进去

### 2.7 索引字段上使⽤（！= 或者 < >），索引可能失效

- 其实这个也是跟 MySQL 优化器有关，如果优化器觉得即使⾛了索引，还是需要扫描很多很多⾏的哈，它觉得不划算，不如**直接不⾛索引**。平时我们⽤ != 或者 < >，not in 的时候，留点⼼眼哈。

### 2.8 索引字段上使⽤ is null，is not null，索引可能失效

- 很多时候，也是因为数据量问题，导致了 MySQL 优化器放弃⾛索引。同时，平时我们⽤ explain 分析 SQL 的时候，如果 type = range，要注意⼀下哈，因为这个可能因为数据量问题，导致索引⽆效。 
- **两列默认 NULL 的字段⽤ or 连接起来，索引就失效了**

### 2.9 左右连接，关联的字段编码格式不⼀致

- 不同表之间代表相同含义的数据项字段编码格式应该⼀致

### 2.10 优化器选错了索引

- 使⽤ force index 强⾏选择某个索引
- 修改你的 SQl，引导它使⽤我们期望的索引
- 优化你的业务逻辑
- 优化你的索引，新建⼀个更合适的索引，或者删除误⽤的索引 

## 3. limit 深分⻚问题
   - 原因：
     1. limit 语句会先扫描 offset + n ⾏，然后再丢弃掉前 offset ⾏，返回后 n ⾏数据。也就是说 limit 100000,10，就会扫描 100010 ⾏，⽽ limit 0,10，只扫描 10 ⾏。 
     2. limit 100000,10 扫描更多的⾏数，也意味着**回表更多的次数**。 
     3. ![图片](https://cdn.learnku.com/uploads/images/202205/25/34535/0UEXsQ60xC.png!large)
     
   - 解决⽅案： 
     - 标签记录法
       - 就是标记⼀下上次查询到哪⼀条了，下次再来查的时候，从该条开始往下扫描。就好像看书⼀样，上次看到哪⾥了，你就折叠⼀下或者夹个书签，下次来看的时候，直接就翻到啦。这样的话，后⾯⽆论翻多少⻚，性能都会不错的，因为命中了 id 索引。
     
       - **缺陷：需要⼀种类似连续⾃增的字段** 
     
       - 优化后 SQL：
     
         ```sql
         SELECT id, name, balance FROM account WHERE id > 100000 LIMIT 10;
         ```
     
     - 延迟关联法
     
       - 优化思路就是，先通过 idx_create_time ⼆级索引树查询到满⾜条件的主键 id，再与 原表通过主键 id 内连接，这样后⾯直接⾛了主键索引了，同时也减少了回表。 
       - 延迟关联法本质还是利用了覆盖索引，把丢弃数据的步骤放在内存里做，这样讲会更容易理解
       - 原 SQL： 
       
       ```sql
       SELECT id, name, balance FROM account WHERE create_time > '2020-09-19' LIMIT 100000, 10;
       ```
       - 原 SQL 2 ： 
       ```sql
       SELECT DISTINCT goods_id FROM jn_order_goods WHERE TO_DAYS(NOW()) - TO_DAYS(gmt_create) = 1 LIMIT 100, 100; 
       ```
       - 优化后SQL ： 
       ```sql
       	SELECT acct1.id, acct1.name, acct1.balance FROM account acct1 INNER JOIN ( SELECT a.id FROM account a WHERE a.create_time > '2020-09-19' LIMIT 100000, 10 ) acct2 ON acct1.id = acct2.id;
       ```
       - 优化后SQL 2： 
       ```sql
       	SELECT `jn_order_goods`.`goods_id` FROM `jn_order_goods` INNER JOIN ( SELECT `rec_id` FROM `jn_order_goods` WHERE TO_DAYS(NOW()) - TO_DAYS(`gmt_create`) = 1 ORDER BY `goods_id` LIMIT 100, 100 ) `tmp_0` USING (`rec_id`);
       ```

## 4. 单表数据量太⼤

   - 原因：⼀个表的数据量达到好⼏千万或者上亿时，加索引的效果没那么明显啦。性能之所以会变差，是因为维护索引的 B+ 树结构层级变得更⾼了，查询⼀条数据时，需要经历的磁盘 IO 变多，因此查询性能变慢。

### 一棵 B + 树可以存多少数据量

 - InnoDB 存储引擎最小储存单元是页，一页大小就是 `16k`。
 - B + 树叶子存的是数据，内部节点存的是键值 + 指针。索引组织表通过非叶子节点的二分查找法以及指针确定数据在哪个页中，进而再去数据页中找到需要的数据
 - ![图片](https://cdn.learnku.com/uploads/images/202205/25/34535/VNszJ5QBXW.png!large)
 - 假设 B + 树的高度为 `2` 的话，即有一个根结点和若干个叶子结点。这棵 B + 树的存放总记录数为 = 根结点指针数 * 单个叶子节点记录行数。
   - 如果一行记录的数据大小为 1k，那么单个叶子节点可以存的记录数 =16k/1k =16.
   - 非叶子节点内存放多少指针呢？我们假设主键 ID 为 **bigint 类型，长度为 8 字节** (**面试官问你 int 类型，一个 int 就是 32 位，4 字节**)，而指针大小在 InnoDB 源码中设置为 6 字节，所以就是 8+6=14 字节，16k/14B =16*1024B/14B = 1170
 - 因此，一棵高度为 2 的 B + 树，能存放 `1170 * 16=18720` 条这样的数据记录。同理一棵高度为 3 的 B + 树，能存放 `1170 *1170 *16 =21902400`，也就是说，可以存放两千万左右的记录。B + 树高度一般为 1-3 层，已经满足千万级别的数据存储。
 - 如果 B + 树想存储更多的数据，那树结构层级就会更高，查询一条数据时，需要经历的磁盘 IO 变多，因此查询性能变慢。

### 如何解决单表数据量太大，查询变慢的问题

 - 一般超过千万级别，我们可以考虑**分库分表**了。
 - 分库分表可能导致的问题：
   - 事务问题
   - 跨库问题
   - 排序问题
   - 分页问题
   - 分布式 ID
 - 评估是否分库分表
   - 是否可以把部分历史数据归档
   - 是否可以垂直分库分表，把一个大的表按属性拆分成几个小表
   - 是否可以水平分库分表，策略：**range 范围、hash 取模、range+hash 取模混合**等等

## 5. join 或者子查询过多

   - 一般来说，**不建议使用子查询**，可以**把子查询改成 `join` 来优化**。
   - **尽量不要有超过 3 个以上的表连接**。
   - MySQL 中，join 的执行算法，分别是：`Index Nested-Loop Join` 和 `Block Nested-Loop Join`
     - `Index Nested-Loop Join`（索引嵌套循环连接）：这个 join 算法，跟我们写程序时的嵌套查询类似，并且可以用上**被驱动表的索引**。
     - `Block Nested-Loop Join`（缓存块嵌套循环连接）：这种 join 算法，**被驱动表上没有可用的索引** , 它会先把驱动表的数据读入线程内存 `join_buffer` 中，再扫描被驱动表，把被驱动表的每一行取出来，跟 `join_buffer` 中的数据做对比，满足 join 条件的，作为结果集的一部分返回。
   - `join` 过多的问题：
     - 过多的表连接，会大大增加 SQL 复杂度。
     - 如果可以使用被驱动表的**索引**那还好，并且使用**小表来做驱动表**，**查询效率更佳**。
     - 如果被驱动表**没有可用的索引**，join 是在 `join_buffer` 内存做的，如果匹配的数据量比较小或者 `join_buffer` 设置的比较大，速度也不会太慢。
     - 如果 `join` 的数据量比较大时，MySQL 会采用在硬盘上创建临时表的方式进行多张表的关联匹配，这种显然效率就极低，本来磁盘的 IO 就不快，还要关联。
     - 当表做 `join` 查询时候，遍历表的总数 约等于 表中记录总数的总和，算法复杂度为 `O(M + N)`， 其中M，N为关联表的总记录数
   - 解决方案：
     - 如果业务需要的话，关联 `2~3` 个表是可以接受的，但是**关联的字段需要加索引**。
     - 如果需要关联更多的表，建议从代码层面进行拆分，在业务层先查询一张表的数据，然后以关联字段作为条件查询关联表形成 `map`，然后在业务层进行数据的拼装。

## 6. in 元素过多

   - 如果使用了 `in`，**即使后面的条件加了索引，还是要注意 `in` 后面的元素不要过多**。
   - **`in` 元素一般建议不要超过 `500` 个**，如果超过了，建议**分组**，每次 `500` 一组进行。
   - `in` 中含有子查询的算法复杂度为 `O(M * N)`，其中M为外围查询的时间，N为子查询的时间
   - 当表做 `join` 查询时候，遍历表的总数 约等于 表中记录总数的总和，算法复杂度为 `O(M + N)`， 其中M，N为关联表的总记录数
   - 在 `in` 中使用独立子查询(和外围查询没有任何关联)的算法复杂度变成 `O(M * N)`
   - MySQL 碰到 `in` 子查询时，MySQL 的优化器会将 `in` 子查询转化为 `exist` 相关子查询，[算法复杂度为 `O(N^3)`]?
   - https://dev.mysql.com/doc/refman/8.0/en/subquery-optimization-with-exists.html

## 7. 数据库在刷脏页

### 什么是脏页

- 当内存数据页跟磁盘数据页内容不一致的时候，我们称这个内存页为 “**脏页**”。
- 内存数据写入到磁盘后，内存和磁盘上的数据页的内容就一致了，称为 “**干净页**”。
- 一般有更新 SQL 才可能会导致脏页

### 一条更新语句是如何执行的？ 

`update t set c=c+1 where id=666;`

1. 对于这条更新 SQL，执行器会先找引擎取 id=666 这一行。如果这行所在的数据页本来就在内存中的话，就直接返回给执行器。如果不在内存，就去磁盘读入内存，再返回。
2. 执行器拿到引擎给的行数据后，给这一行 C 的值加一，得到新的一行数据，再调用引擎接口写入这行新数据。
3. 引擎将这行新数据更新到内存中，同时将这个更新操作记录到 redo log 里面，但是此时 redo log 是处于 prepare 状态的哈。
4. 执行器生成这个操作的 binlog，并把 binlog 写入磁盘。
5. 执行器调用引擎的提交事务接口，引擎把刚刚写入的 redo log 改成提交（commit）状态，更新完成。
- ![图片](https://cdn.learnku.com/uploads/images/202205/25/34535/7DF1LU01JB.png!large)
- InnoDB 在处理更新语句的时候，只做了写日志这一个磁盘操作。这个日志叫作 `redo log` （重做日志）。
- 平时更新 SQL 执行得很快，其实是因为它只是在写内存和 `redo log` 日志，等到空闲的时候，才把 redo log 日志里的数据同步到磁盘中。
- 写 `redo log` 的过程是顺序写磁盘的。**磁盘顺序写**会减少寻道等待时间，速度比随机写要快很多的。

### 为什么会出现脏页呢？

- 更新 SQL 只是在写内存和 `redo log` 日志，等到空闲的时候，才把 `redo log` 日志里的数据同步到磁盘中。这时内存数据页跟磁盘数据页内容不一致，就出现脏页。

### 什么时候会刷脏页（flush）？

- InnoDB 存储引擎的 `redo log` 大小是固定，且是环型写入的
- ![图片](https://cdn.learnku.com/uploads/images/202205/25/34535/ryOtoEmcjE.png!large)
1. redo log 写满了，要刷脏页。这种情况要尽量避免的。因为出现这种情况时，整个系统就不能再接受更新啦，即所有的更新都必须堵住。
2. 内存不够了，需要新的内存页，就要淘汰一些数据页，这时候会刷脏页
- InnoDB 用缓冲池（buffer pool）管理内存，而当要读入的数据页没有在内存的时候，就必须到缓冲池中申请一个数据页。
- 这时候只能把最久不使用的数据页从内存中淘汰掉：
  - 如果要淘汰的是一个干净页，就直接释放出来复用；
  - 如果是脏页呢，就必须**将脏页先刷到磁盘**，变成干净页后才能复用。
1. MySQL 认为系统空闲的时候，也会刷一些脏页
2. MySQL 正常关闭时，会把内存的脏页都 flush 到磁盘上

   - 为什么刷脏页会导致 SQL 变慢呢？

     1. `redo log` 写满了，要刷脏页，这时候会导致系统所有的更新堵住，写性能都跌为 0 了，肯定慢呀。一般要杜绝出现这个情况。
     2. 一个查询要淘汰的脏页个数太多，一样会导致查询的响应时间明显变长。

## 8. order by 文件排序

### 8.1 order by 的 `Using filesort` 文件排序

- 查看 `explain` 执行计划的时候，可以看到 `Extra` 这一列，有一个 `Using filesort`，它表示用到**文件排序**。
- Extra 这个字段的 **Using index condition** 表示索引条件
- Extra 这个字段的 **Using filesort** 表示用到排序

### 8.2 order by 文件排序效率为什么较低

- ![图片](https://cdn.learnku.com/uploads/images/202205/25/34535/ppuw7E3zFZ.png!large)

- `order by` 排序，分为全字段排序和 rowid 排序。

- `order by` 排序是拿 `max_length_for_sort_data` 和结果行数据长度对比，如果结果行数据长度超过 `max_length_for_sort_data` 这个值，就会走 **rowid 排序**，相反，则走**全字段排序**。

- rowid 排序，sort_buffer可以放更多数据，但是需要再回到原表去取数据，比全字段排序多一次**回表，所以效率会慢一点。**
  - `**select** name,age,city **from** staff **where** city = '深圳' order **by** age limit 10;`
  1.  MySQL 为对应的线程初始化 `sort_buffer`，放入需要排序的 `age` 字段，以及`主键id`;
  2.  从索引树 `idx_city`， 找到第一个满足 `city='深圳’`条件的`主键id`，也就是图中的 `id=9`；
  3.  到`主键id索引树`拿到 `id=9` 的这一行数据， 取 `age和主键id` 的值，存到 `sort_buffer`；
  4.  从索引树 `idx_city` 拿到下一个记录的`主键id`，即图中的 `id=13`；
  5.  重复步骤 3、4 直到 `city` 的值不等于深圳为止；
  6.  前面 5 步已经查找到了所有 `city` 为深圳的数据，在 `sort_buffer` 中，将所有数据根据 age 进行排序；
  7.  遍历排序结果，取前 10 行，并按照 `id` 的值回到原表中，取出 `city、name 和 age` 三个字段返回给客户端。
  -  ![图片](https://cdn.learnku.com/uploads/images/202205/25/34535/FJE3r1Q0gh.png!large)

- 全字段排序，sort_buffer内存不够的话，就需要用到磁盘临时文件，造成**磁盘访问**。

  - MySQL 会给每个查询线程分配一块小内存，用于排序的，称为 `sort_buffer`。什么时候把字段放进去排序呢，其实是通过 `idx_city` 索引找到对应的数据，才把数据放进去啦。
  - 将查询所需的字段全部读取到 sort_buffer 中，就是**全字段排序**。
  - `select name,age,city from staff where city = '深圳' order by age limit 10;`

  1. MySQL 为对应的线程初始化 `sort_buffer`，放入需要查询的 `name、age、city` 字段；
  2. 从索引树 `idx_city`， 找到第一个满足 `city='深圳’`条件的主键 id，也就是图中的 `id=9`；
  3. 到主键 `id索引树`拿到 `id=9` 的这一行数据， 取 `name、age、city` 三个字段的值，存到 `sort_buffer`；
  4. 从索引树 `idx_city` 拿到下一个记录的主键 `id`，即图中的 `id=13`；
  5. 重复步骤 3、4 直到 `city` 的值不等于深圳为止；
  6. 前面 5 步已经查找到了所有 `city` 为深圳的数据，在 `sort_buffer` 中，将所有数据根据 `age` 进行排序；
  7. 按照排序结果取前 10 行返回给客户端。
  - ![图片](https://cdn.learnku.com/uploads/images/202205/25/34535/KxbabGB992.png!large)

- `sort_buffer` 的大小是由一个参数控制的：`sort_buffer_size`。

  - 如果要排序的数据小于 `sort_buffer_size`，排序在 `sort_buffer` 内存中完成
  - 如果要排序的数据大于 `sort_buffer_size`，则借助磁盘文件来进行排序
  - 借助磁盘文件排序的话，**效率就更慢一点**。因为先把数据放入 sort_buffer，当快要满时。会排一下序，然后把 sort_buffer 中的数据，放到临时磁盘文件，等到所有满足条件数据都查完排完，再用归并算法把磁盘的临时排好序的小文件，合并成一个有序的大文件。
  - 借助磁盘临时小文件排序，实际上使用的是**归并排序**算法。

- 我们通过**optimizer_trace**，可以看到是否使用了rowid排序的

  ```sql
  ## 打开optimizer_trace，开启统计
  set optimizer_trace = "enabled=on";
  ## 执行SQL语句
  select name,age,city from staff where city = '深圳' order by age limit 10;
  ## 查询输出的统计信息
  select * from information_schema.optimizer_trace 
  ```

- 一般情况下，对于InnoDB存储引擎，会优先使**用全字段**排序。可以发现 **max_length_for_sort_data** 参数设置为1024，这个数比较大的。一般情况下，排序字段不会超过这个值，也就是都会走**全字段**排序。

### 8.3 如何优化 order by 的文件排序

- 因为数据是无序的，所以就需要排序。如果数据本身是有序的，那就不会再用到文件排序啦。而索引数据本身是有序的，我们通过建立索引来优化 `order by` 语句，就不需要 **Using filesort ** 排序了。
- 我们还可以通过调整 `max_length_for_sort_data`、`sort_buffer_size` 等参数优化；

### 8.4 使用order by 的一些注意点

- 没有where条件，order by字段需要加索引吗
  - **无 where 条件也没有 limit** : 无条件查询的话，即使 create_time 上有索引,也不会使用到。因为 MySQL 优化器认为走普通二级索引，再去回表成本比全表扫描排序更高。所以选择走全表扫描,然后根据全字段排序或者 rowid 排序来进行。
  - **无 where 条件带 limit m**: 如果m值较小，是可以走索引的。因为 MySQL 优化器认为，根据索引有序性去回表查数据，然后得到 m 条数据，就可以终止循环，那么成本比全表扫描小，则选择走二级索引。
  
- 分页limit过大时，会导致大量排序怎么办?
  - 可以记录上一页最后的id，下一页查询时，查询条件带上id，如：where id > 上一页最后id limit 10。（标签记录法）
  - 也可以在业务允许的情况下，限制页数。
  - 延迟关联法，先通过 idx_create_time 二级索引树查询到满足条件的主键 ID，再与原表通过主键 ID 内连接，这样后面直接走了主键索引了，同时也减少了回表。
  
- 索引存储顺序与order by不一致，如何优化？
  ```sql
  select name, age from staff  order by age, name desc limit 10;
  ```
  - ![图片](https://mmbiz.qpic.cn/mmbiz_png/PoF8jo1Pmpwlj1dDtegxsHWfNCSQev65VrmibD2cCNg3v3E87G41k1Dzax3ibEGzKzbNV89aPYnxBIgvaw8Hibndw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)
  
  - 因为 idx_age_name 索引树中，age 从小到大排序，如果 **age 相同，再按 name 从小到大排序**。而 order by 中，是按 age 从小到大排序，如果 **age 相同，再按 name 从大到小排序**。也就是说，索引存储顺序与 order by 不一致。
  
  - 如果MySQL是8.0版本，支持**Descending Indexes**，可以这样修改索引：
  
  - ```sql
    ALTER TABLE `staff` ADD index idx_age_name(`age`,`name` desc) USING BTREE;
    ```
  
  - 不过索引 `(a, b desc)`，就不再满足 `order by a, b` 了，还是会用到 `filesort`。
  
  - MySQL 的排序操作，只有 asc 没有 desc (desc 是假的,未来会做真正的 desc,期待...).
  
- 使用了in条件多个属性时，SQL执行是否有排序过程

  - 如果我们有**联合索引 idx_city_age**，执行这个 SQL 的话，是不会走排序过程的，如下：
  - `select * from staff where city in ('深圳') order by age limit 10;`
  - ![图片](https://mmbiz.qpic.cn/mmbiz_png/PoF8jo1Pmpwlj1dDtegxsHWfNCSQev65UpPoVY8QBJeKXCypp6XxTtmxY5DhwPc2eLfrtFdTL3tUqujToZna2A/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)
  - 但是，如果使用 in 条件，并且有多个条件时，就会有排序过程。
  - `explain select * from staff where city in ('深圳','上海') order by age limit 10;`
  - ![图片](https://mmbiz.qpic.cn/mmbiz_png/PoF8jo1Pmpwlj1dDtegxsHWfNCSQev65mBZzGBAC5xu0PZoPndgHwrV9ELiaibMg07wiaHMc1C9U3suZkOLGniaKnQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)
  - 这是因为: in 有两个条件，在满足深圳时，age 是排好序的，但是把满足上海的 age 也加进来，就不能保证满足所有的 age 都是排好序的。因此需要 Using filesort。


## 9. 拿不到锁

   - 一般这种时候就是表被锁住了，或者要查询的某一行或者几行被锁住了。我们只能慢慢等待锁被释放。
   - 可以用 `show processlist` 命令，看看当前语句处于什么状态哈。

## 10. delete + in 子查询不走索引！

- 当 `delete` 遇到 `in` 子查询时，即使有索引，也是不走索引的。而对应的 `select + in` 子查询，却可以走索引。
- 实际执行的时候，MySQL 对 `select in` 子查询做了优化，把子查询改成 `join` 的方式，所以可以走索引。但是很遗憾，对于 `delete in` 子查询，MySQL 却没有对它做这个优化。
- 如果对某个查询**执行** `EXPLAIN EXTENDED [SQL;] SHOW WARNINGS;` 就可以看到**优化器重构后**的查询语句。
- 可以把 `delete in子查询` 改为 **`join`** 的方式。改用join的方式是**可以走索引的**
- ![图片](https://mmbiz.qpic.cn/mmbiz_png/PoF8jo1PmpzVZucibyYaKWtvDjaIauJCiah8W3JqGcjEbwvLyMCDJLuDSwR6d413iavssjp52ouaPWuCQ4W4hvbFQ/640?wx_fmt=png)
- 给表加别名，也可以**走索引**
  - **LooseScan 是什么呢？** 其实它是一种策略，是 **semi join 子查询**的一种执行策略。
  - 因为子查询改为 join，是可以让 delete in 子查询走索引；**加别名呢**，会走 **LooseScan 策略**，而 LooseScan 策略，本质上就是 **semi join 子查询**的一种执行策略。
  - `semi-join` 是半连接的意思，SQL 层面并没有 semi-join 的语法，这是MySQL内核里面使用的一种方式。就是在某些场景下，优化器可能将子查询优化为 semi-join 半连接的形式，使用连接查询就可以避免使用物化表了。
  - 因此，加别名就可以让delete in子查询走索引啦！


## 11. group by 使用临时表

### 11.1 group by 的执行流程

- `group by` 一般用于**分组统计**，它表达的逻辑就是 `根据一定的规则，进行分组`。
- `select city, count(*) as num from staff group by city;`
- ![图片](https://cdn.learnku.com/uploads/images/202205/25/34535/gSOHYirvdq.png!large)
- Extra 这个字段的 `Using temporary` 表示在执行分组的时候使用了**临时表**
- Extra 这个字段的 `Using filesort` 表示使用了**文件排序**
1. 创建内存临时表，表里有两个字段 `city和num`；
2. 全表扫描 `staff` 的记录，依次取出 `city = 'X'` 的记录。
   1. 判断临时表中是否有为 `city='X'` 的行，没有就插入一个记录 `(X,1)`;
   2. 如果临时表中有 `city='X'` 的行，就将 X 这一行的 num 值加 1；
3. 遍历完成后，再根据字段 `city` 做排序，得到结果集返回给客户端。这个流程的执行图如下：
4. ![图片](https://cdn.learnku.com/uploads/images/202205/25/34535/rQEYPdBBeW.png!large)
- 临时表的排序是怎样的呢？
  - 就是把需要排序的字段，放到 sort buffer，排完就返回。
  - 排序分**全字段排序和 rowid 排序**
    - 如果是全字段排序，需要查询返回的字段，都放入 sort buffer，根据排序字段排完，直接返回
    - 如果是 rowid 排序，只是需要排序的字段放入 sort buffer，然后多一次回表操作，再返回。
    - 怎么确定走的是全字段排序还是rowid 排序排序呢？由一个数据库参数控制的，`max_length_for_sort_data`

### 11.2 group by 可能会慢在哪里？

  - `group by` 使用不当，很容易就会产生慢 SQL 问题。因为它既用到临时表，又默认用到排序。有时候还可能用到磁盘临时表。
  - 如果执行过程中，会发现`内存临时表`大小到达了上限（控制这个上限的参数就是 tmp_table_size），会把内存临时表转成磁盘临时表。
  - 如果数据量很大，很可能这个查询需要的磁盘临时表，就会占用大量的磁盘空间。

### 11.3 where 和 having的区别

- group by + where 的执行流程

  ```sql
  select city ,count(*) as num from staff where age > 30 group by city;
  //加索引
  alter table staff add index idx_age (age);
  
  explain select city ,count(*) as num from staff where age > 30 group by city;
  ```

  - ![图片](https://mmbiz.qpic.cn/mmbiz_png/PoF8jo1Pmpw3lfjrm2iavcLbJViaxTt98DkVZibUZMgZQC2OEicCvUZxQ5y0XO1KgUZzyibtJ1qib9G6rFC2ZzejlGUw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

  - **Using index condition**: 表示索引下推优化，根据索引尽可能的过滤数据，然后再返回给服务器层根据where其他条件进行过滤。

  - 这里单个索引为什么会出现索引下推呢？explain出现并不代表一定是使用了索引下推，只是代表可以使用，但是不一定用了。

  - 根据 group by 和 where 的执行，group by 的联合索引 age 和 city，age 相同的话，city 有序的

  - 执行流程如下：

    1. 创建内存临时表，表里有两个字段 `city` 和 `num`；
    2. 扫描索引树 `idx_age`，找到大于年龄大于 30 的主键 ID
    3. 通过主键 ID，回表找到 city = 'X'

    - 判断**临时表**中是否有为 city='X' 的行，没有就插入一个记录 (X,1);
    - 如果临时表中有 city='X' 的行的行，就将 x 这一行的 num 值加 1；

    4. 继续重复 2,3 步骤，找到所有满足条件的数据；
    5. 最后根据字段 `city` 做**排序**，得到结果集返回给客户端。

- group by + having 的执行流程

  - `select city ,count(*) as num from staff group by city having num >= 3;`

  - ![图片](https://mmbiz.qpic.cn/mmbiz_png/PoF8jo1Pmpw3lfjrm2iavcLbJViaxTt98DruiceXEP2Z44xesnF6cYngaic9O1S36jsVRUvytib8icGEvTYsAwbOKXiaw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

  - `having` 称为分组过滤条件，它对返回的结果集操作。

- 同时有 where、group by 、having 的执行顺序

  - `select city ,count(*) as num from staff where age > 19 group by city having num >= 3;`
  1. 执行 `where`子句查找符合年龄大于 19 的员工数据
  2. `group by ` 子句对员工数据，根据城市分组。
  3. 对`group by` 子句形成的城市组，运行聚集函数计算每一组的员工数量值；
  4. 最后用 `having `子句选出员工数量大于等于 3 的城市组。

- where + having 区别总结

  - `having` 子句用于**分组后筛选**，where 子句用于**行**条件筛选
  - `having` 一般都是配合 `group by` 和聚合函数一起出现如 (`count(),sum(),avg(),max(),min()`)
  - `where` 条件子句中不能使用聚集函数，而 `having `子句就可以。
  - `having `只能用在 group by 之后，where 执行在 group by 之前


### 11.4 使用 group by 注意的问题

- group by 一定要配合聚合函数使用吗？
  - group by 就是**分组统计**的意思，一般情况都是配合聚合函数`如（count(), sum(), avg(), max(), min())`一起使用。
    - count() 数量
    - sum() 总和
    - avg() 平均
    - max() 最大值
    - min() 最小值
  - 如果没有配合聚合函数使用可以吗？我用的是 **Mysql 5.7** ，是可以的。不会报错，并且返回的是，分组的第一行数据。
  - 是否必须与聚合函数使用在 Mysql 5.7.5 版本以后取决于 sql_mode 是否设置 only_full_group_by，默认是设置了 only_full_group_by。开启此设置才会必须与聚合函数使用。
  - 聚合函数是对一组值执行计算，并返回单个值，也被称为组函数。 聚合函数是指对一组值执行计算并返回单一的值的一类函数，它们通常与 group by 子句一起使用，将数据集分组为子集。
  - 反过来，SQL 中只要用到聚合函数就一定要用到 group by 吗? 不一定
    1. 当聚集函数和非聚集函数出现在一起时，需要将非聚集函数进行 group by
    2. 当只做聚集函数查询时候，就不需要进行分组了。
- group by 后面跟的字段一定要出现在 select 中吗？
  - 不一定，这个可能跟不同的数据库，不同的版本有关吧。


### 11.5 group by导致的慢SQL问题

- group by 使用不当，很容易就会产生慢 SQL 问题。因为它既用到临时表，又默认用到排序。有时候还可能用到磁盘临时表。
- 如果执行过程中，会发现内存临时表大小到达了**上限**（控制这个上限的参数就是 `tmp_table_size`），会把**内存临时表转成磁盘临时表**。
- 如果数据量很大，很可能这个查询需要的磁盘临时表，就会占用大量的磁盘空间。


### 11.6 如何优化 group by 呢？

  - 方向 1：既然它默认会排序，我们不给它排是不是就行啦。
  - 方向 2：既然临时表是影响 `group by` 性能的 X 因素，我们是不是可以不用临时表？
  - 执行 group by 语句为什么需要临时表呢？group by 的语义逻辑，就是统计不同的值出现的个数。如果这个**这些值一开始就是有序的**，我们是不是直接往下扫描统计就好了，就**不用临时表来记录并统计结果**啦？
  - 优化方案：
    1. group by 后面的字段加索引 （**加合适的索引**是优化 `group by` 最简单有效的优化方式。（ `where b = x group by a` 加联合索引 `idx_b_a（b, a）` ））
    2. order by null 不用排序 （如果你的需求并**不需要对结果集进行排序**，可以使用 `order by null`）
    3. 尽量只使用内存临时表
       - 如果 `group by` 需要统计的数据不多，我们可以尽量只使用**内存临时表**；
       - 因为如果 group by 的过程因为内存临时表放不下数据，从而用到磁盘临时表的话，是比较耗时的。
       - 因此可以适当调大`tmp_table_size`参数，来避免用到**磁盘临时表**。
    4. 使用 SQL_BIG_RESULT
       - 如果预估数据量比较大，我们使用 `SQL_BIG_RESULT`  这个提示直接用磁盘临时表。
       - MySQL 优化器发现，磁盘临时表是 B+ 树存储，存储效率不如数组来得高。因此会直接用数组来存
       - `select SQL_BIG_RESULT city ,count(*) as num from staff group by city;`
       - ![图片](https://mmbiz.qpic.cn/mmbiz_png/PoF8jo1Pmpw3lfjrm2iavcLbJViaxTt98DxE8wviaAbjPONoicfnXhdaDWKrqOWm60JfPvlBCgdO5ib8Nw6wiavEUA1w/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)
       - 执行流程如下：
         1. 初始化 sort_buffer，放入 city 字段；
         2. 扫描表 staff，依次取出 city 的值,存入 sort_buffer 中；
         3. 扫描完成后，对 sort_buffer 的 city 字段做排序
         4. 排序完成后，就得到了一个有序数组。
         5. 根据有序数组，统计每个值出现的次数。

## 12. 系统硬件或网络资源

- 如果数据库服务器内存、硬件资源，或者网络资源配置不是很好，就会慢一些，可以升级配置
- 如果数据库压力本身很大，比如高并发场景下，大量请求到数据库来，数据库服务器 `CPU` 占用很高或者 `IO利用率`很高，这种情况下所有语句的执行都有可能变慢的

## 13. 参数配置不一致

- 例如：测试环境用了 `index merge`，生产环境配置却把 `index merge` 关闭了
