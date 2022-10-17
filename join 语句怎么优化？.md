在选择Join算法时，会有优先级，理论上会优先判断能否使用 INLJ、BNLJ：
**Index Nested-LoopJoin > Block Nested-Loop Join > Simple Nested-Loop Join**



# Simple Nested-Loop Join(简单嵌套循环联接)

- 当我们进行连接操作时，左边的表是**「驱动表」**，右边的表是**「被驱动表」**
- Simple Nested-Loop Join 这种连接操作是从驱动表中取出一条记录然后逐条匹配被驱动表的记录，如果条件匹配则将结果返回。然后接着取驱动表的下一条记录进行匹配，直到驱动表的数据全都匹配完毕
- **「因为每次从驱动表取数据比较耗时，所以MySQL并没有采用这种算法来进行连接操作」**
- ![图片](https://mmbiz.qpic.cn/mmbiz_png/OQmPiaEUnhd6j6yEia3zBgp5uYRKyV3ibOz5ZDJBxpaFXTlDQ0nFya5RBrmCYU3UBWOFDPT9Dr2htL7BXy9F2gdibA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



# Block Nested-Loop Join(块嵌套循环联接)

- 既然每次从驱动表取数据比较耗时，那我们每次从驱动表取一批数据放到内存中，然后对这一批数据进行匹配操作。这批数据匹配完毕，再从驱动表中取一批数据放到内存中，直到驱动表的数据全都匹配完毕
- 批量取数据能减少很多IO操作，因此执行效率比较高，这种连接操作也被MySQL采用
- ![图片](https://mmbiz.qpic.cn/mmbiz_png/OQmPiaEUnhd6j6yEia3zBgp5uYRKyV3ibOzuo8BYV5NuEqpz7gFbFI5DgjnzbV0HbPrPM7mIugiaLNwF9qOxQMGpMQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)
- 驱动表存取数据的这块内存在MySQ中有一个专有的名词，叫做 join buffer，我们可以执行如下语句查看 join buffer 的大小 `show variables like '%join_buffer%'`
- 如果直接使用 join 语句，MySQL优化器可能会选择表 t1 或者 t2 作为驱动表，这样会影响我们分析sql语句的过程，所以我们用 straight_join 让mysql使用固定的连接方式执行查询 `select * from t1 straight_join t2 on (t1.common_field = t2.common_field)`
- 在Extra列中看到了 Using join buffer ，说明连接操作是基于 **「Block Nested-Loop Join」** 算法
- ![图片](https://mmbiz.qpic.cn/mmbiz_png/OQmPiaEUnhd6j6yEia3zBgp5uYRKyV3ibOzqOHUpa5KPeq7iaIzRE1DcibAHYlviaHN0N1RuMShAOF8kT7Y4RC5s5BbA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



#  Index Nested-Loop Join(索引嵌套循环联接)

- 了解了 **「Block Nested-Loop Join」** 算法之后，可以看到驱动表的每条记录会把被驱动表的所有记录都匹配一遍，非常耗时
- 提高效率的算法是，给被驱动表连接的列加上索引，这样匹配的过程就非常快，如图所示
- ![图片](https://mmbiz.qpic.cn/mmbiz_png/OQmPiaEUnhd6j6yEia3zBgp5uYRKyV3ibOz3mGLWRBEuESlqB044llTXtJkUIKpsdPZkGkAhAh3BWvW6NIIQmGtKQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)
-  `select * from t1 straight_join t2 on (t1.id = t2.id)` ，执行时间为0.001秒，可以看到比基于普通的列进行连接快了不止一个档次，执行计划如下![图片](https://mmbiz.qpic.cn/mmbiz_png/OQmPiaEUnhd6j6yEia3zBgp5uYRKyV3ibOzdZpC9c3po35D1N17rHZlK36fua5icUbVAUxJhicCgYqjRWETJMzPeCzw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)
- **「驱动表的记录并不是所有列都会被放到 join buffer，只有查询列表中的列和过滤条件中的列才会被放入 join buffer，因此我们不要把 \* 作为查询列表，只需要把我们关心的列放到查询列表就好了，这样可以在 join buffer 中放置更多的记录」**



# Batched Key Access（批量索引访问）

- BKA算法原理：将外层循环的行/结果集存入join buffer，内存循环的每一行数据与整个buffer中的记录做比较，可以减少内层循环的扫描次数。
- MySQL 5.6 开始支持。



# 如何选择驱动表？

- **「如果是 Block Nested-Loop Join 算法：」**
  1. 当 join buffer 足够大时，谁做驱动表没有影响
  2. 当 join buffer 不够大时，应该选择小表做驱动表（小表数据量少，放入 join buffer 的次数少，减少表的扫描次数）
- **「如果是 Index Nested-Loop Join 算法」**
  - 假设驱动表的行数是M，因此需要扫描驱动表M行，被驱动表的行数是N，每次在被驱动表查一行数据，要先搜索索引a，再搜索主键索引。
  - 每次搜索一颗树近似复杂度是以2为底N的对数，所以在被驱动表上查一行的时间复杂度是 2 * log2^N
  - 驱动表的每一行数据都要到被驱动表上搜索一次，整个执行过程近似复杂度为 M + M * 2 * log2^N
  - **「显然M对扫描行数影响更大，因此应该让小表做驱动表。当然这个结论的前提是可以使用被驱动表的索引」**
  - **「总而言之，我们让小表做驱动表即可」**
- **「当 join 语句执行的比较慢时，我们可以通过如下方法来进行优化」**
  1. 进行连接操作时，能使用被驱动表的索引
  2. 小表做驱动表
  3. 增大 join buffer 的大小
  4. 不要用 * 作为查询列表，只返回需要的列



