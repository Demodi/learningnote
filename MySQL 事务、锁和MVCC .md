# 怎么理解 InnoDB 引擎中的事务?

- 事务可以使「一组操作」要么全部成功，要么全部失败
- 事务其目的是为了「保证数据最终的一致性」
- ![图片](https://cdn.learnku.com/uploads/images/202205/25/34535/9MJAOvJF8k.jpeg!large)

# 事务的几大特性

- 就是 ACID 嘛，分别是原子性（Atomicity）、一致性（Consistency）、隔离性（Isolation）、持久性（Durability）。
- 事务主要是为了实现一致性，具体是通过AID，即原子性、隔离性和持久性来达到一致性的目的

## 原子性 Atomicity

  - ![图片](https://cdn.learnku.com/uploads/images/202205/25/34535/uQ65uRKbVS.jpeg!large)
  - 当前事务的操作要么同时成功，要么同时失败。底层依赖  undo log 来实现
  - 原子性由 undo log 日志来保证，因为 undo log 记载着数据修改前的信息。
  - 如果执行事务过程中出现异常的情况，那执行「回滚」。
  - InnoDB 引擎就是利用 undo log 记录下的数据，来将数据「恢复」到事务开始之前

## 隔离性 Consistency

  - ![图片](https://cdn.learnku.com/uploads/images/202205/25/34535/xHkDhxFdfd.jpeg!large)
  - 在事务「并发」执行时，他们内部的操作不能互相干扰。
  - 如果多个事务可以同时操作一个数据，那么就会产生脏读、重复读、幻读的问题。
  - 于是，事务与事务之间需要存在「一定」的隔离。
  - 在 InnoDB 引擎中，定义了四种隔离级别供我们使用：分别是：read uncommit (读未提交)、read commit (读已提交)、repeatable read (可重复复读)、serializable (串行)
  - 不同的隔离级别对事务之间的隔离性是不一样的（级别越高事务隔离性越好，但性能就越低），而隔离性是由 MySQL 的各种锁来实现的，只是它屏蔽了加锁的细节。

## 持久性 Isolation

  - ![图片](https://cdn.learnku.com/uploads/images/202205/25/34535/EkKada3YQB.jpeg!large)
  - 一旦提交了事务，它对数据库的改变就应该是永久性的。说白了就是，会将数据持久化在硬盘上。
  - 而持久性由 redo log 日志来保证，当我们要修改数据时，MySQL 是先把这条记录所在的「页」找到，然后把该页加载到内存中，将对应记录进行修改。
  - 为了防止内存修改完了，MySQL 就挂掉了（如果内存改完，直接挂掉，那这次的修改相当于就丢失了）。
  - MySQL 引入了 redo log，内存写完了，然后会写一份 redo log，这份 redo log 记载着这次在某个页上做了什么修改。
  - 即便 MySQL 在中途挂了，我们还可以根据 redo log 来对数据进行恢复。
  - redo log 是顺序写的，写入速度很快。并且它记录的是物理修改（xxxx 页做了 xxx 修改），文件的体积很小，恢复速度也很快。

## 一致性 Durability

  - ![图片](https://cdn.learnku.com/uploads/images/202205/25/34535/0XX9aM7elz.jpeg!large)
  - 「一致性」可以理解为我们使用事务的「目的」，而「隔离性」「原子性」「持久性」均是为了保障「一致性」的手段，保证一致性需要由应用程序代码来保证
  - 比如，如果事务在发生的过程中，出现了异常情况，此时你就得回滚事务，而不是强行提交事务来导致数据不一致。

# MySQL 锁

- 在 InnoDB 引擎下，按锁的粒度分类，可以简单分为行锁和表锁。
- 行锁实际上是作用在索引之上的。当我们的 SQL 命中了索引，那锁住的就是命中条件内的索引节点（这种就是行锁），如果没有命中索引，那我们锁的就是整个索引树（表锁）。
- 简单来说就是：锁住的是整棵树还是某几个节点，完全取决于 SQL 条件是否有命中到对应的索引节点。
- 而行锁又可以简单分为读锁（共享锁、S 锁）和写锁（排它锁、X 锁）。
- 读锁是共享的，多个事务可以同时读取同一个资源，但不允许其他事务修改。
- 写锁是排他的，写锁会阻塞其他的写锁和读锁。
- ![图片](https://cdn.learnku.com/uploads/images/202205/25/34535/hgu6Q4CLgO.jpeg!large)

# MySQL 的四种隔离级别

## read uncommit (读未提交)

- read uncommit 隔离级别下会产生脏读
- 事务 B 读取到了事务 A 还没提交的数据，这种用专业术语来说叫做「脏读」。
- 对于锁的维度而言，其实就是**在 read uncommit 隔离级别下，读不会加任何锁，而写会加排他锁。读什么锁都不加，这就让排他锁无法排它了。**
- **对于更新操作而言，InnoDB 是肯定会加写锁的**（数据库是不可能允许在同一时间，更新同一条记录的）。而**读操作，如果不加任何锁，那就会造成上面的脏读。**
- 脏读在生产环境下肯定是无法接受的，那如果读加锁的话，那意味着：当更新数据的时，就没办法读取了，这会极大地降低数据库性能。

## read commit (读已提交) 

- read commit (读已提交) 隔离级别解决了脏读

- 技术思想：在读取的时候生成一个” 版本号”，等到其他事务 commit 了之后，才会读取最新已 commit 的” 版本号” 数据。
- 比如说：事务 A 读取了记录 (生成版本号)，事务 B 修改了记录 (此时加了写锁)，事务 A 再读取的时候，是依据最新的版本号来读取的 (当事务 B 执行 commit 了之后，会生成一个新的版本号)，如果事务 B 还没有 commit，那事务 A 读取的还是之前版本号的数据。
- 通过「版本」的概念，这样就解决了脏读的问题，而「版本」其实就是对应快照的数据。
- read commit (读已提交) 隔离级别会存在不可重复读的问题
- 「不可重复读」：一个事务读取到另外一个事务已经提交的数据，也就是说一个事务可以看到其他事务所做的修改。
- 不可重复读的例子：A 查询数据库得到数据，B 去修改数据库的数据，导致 A 多次查询数据库的结果都不一样【危害：A 每次查询的结果都是受 B 的影响的】

##  repeatable read (可重复复读) 

- repeatable read (可重复复读) 隔离级别避免了不可重复读的问题
- repeatable read (可重复复读) 隔离级别是「事务级别」的快照！每次读取的都是「当前事务的版本」，即使当前数据被其他事务修改了 (commit)，也只会读取当前事务版本的数据。
- repeatable read (可重复复读) 隔离级别会存在幻读的问题

- repeatable read (可重复复读) 隔离级别会存在幻读的问题
- 「幻读」指的是指在一个事务内读取到了别的事务插入的数据，导致前后读取不一致。
- 在 InnoDB 引擎下的的 repeatable read (可重复复读) 隔离级别下，快照读 MVCC 影响下，已经解决了幻读的问题（因为它是读历史版本的数据）

- 如果是当前读（指的是 select * from table for update），则需要配合间隙锁来解决幻读的问题。

## serializable (串行) 

- 它是最高的隔离级别，相当于不允许事务的并发，事务与事务之间执行是串行的，它的效率最低，但同时也是最安全的。

事务隔离级别下，针对于 read commit (读已提交) 隔离级别，它生成的就是语句级快照，而针对于 repeatable read (可重复读)，它生成的就是事务级的快照。

# MVCC  (Multi-Version Concurrency Control) 多版本并发控制

- ![图片](https://cdn.learnku.com/uploads/images/202205/25/34535/nYFypi62d5.jpeg!large)
- 在 MySQL InnoDB 引擎层面，又有新的解决方案（解决加锁后读写性能问题），叫做 MVCC (Multi-Version Concurrency Control) 多版本并发控制
- 多版本并发控制，其实指的是一条记录会有多个版本，每次修改记录都会存储这条记录被修改之前的版本，多版本之间串联起来就形成了一条版本链。
- 这样不同时刻启动的事务可以无锁地获得不同版本的数据(普通读)。此时读(普通读)写操作不会阻塞，写操作可以继续写，无非就是多加了一个版本，历史版本记录可供已经启动的事务读取。
- ![图片](https://mmbiz.qpic.cn/mmbiz_png/eSdk75TK4nFQRcyKribkGLAdbs84qGjBwkHficC7MXgWKfnhYR91Pv0g2lBumXOvIgmFCmJjo57MQibaBL2kTiaaDA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)
- MVCC 用来**实现读已提交和可重复读**。首先隔离级别如果是读未提交的话，直接读最新版本的数据就行了，压根就不需要保存以前的版本。可串行化隔离级别事务都串行执行了，所以也不需要多版本，因此 MVCC 是用来实现读已提交和可重复读的。
- 在 MVCC 下，就可以做到读写不阻塞，且避免了类似脏读这样的问题。
- MVCC 通过生成数据快照（Snapshot)，并用这个快照来提供一定级别（语句级或事务级）的一致性读取
- 回到事务隔离级别下，事务隔离级别下，针对于 read commit (读已提交) 隔离级别，它生成的就是语句级快照，而针对于 repeatable read (可重复读)，它生成的就是事务级的快照。
- ![图片](https://cdn.learnku.com/uploads/images/202205/25/34535/nYFypi62d5.jpeg!large)
- **MVCC 思想其实很简单：在读取的时候生成一个” 版本号”，等到其他事务 commit 了之后，才会读取最新已 commit 的” 版本号” 数据。**
- 比如说：事务 A 读取了记录 (生成版本号)，事务 B 修改了记录 (此时加了写锁)，事务 A 再读取的时候，是依据最新的版本号来读取的 (当事务 B 执行 commit 了之后，会生成一个新的版本号)，如果事务 B 还没有 commit，那事务 A 读取的还是之前版本号的数据。
- 通过「版本」的概念，这样就解决了脏读的问题，而「版本」其实就是对应快照的数据。
- MVCC 的主要是**通过 read view 和 undo log 来实现**的
- ![图片](https://cdn.learnku.com/uploads/images/202205/25/34535/Xk4CpiWpZV.jpeg!large)
- 如果没有 MVCC 读写操作之间就会冲突。
- 想象一下有一个事务1正在执行，此时一个事务2修改了记录A，还未提交，此时事务1要读取记录A，因为事务2还未提交，所以事务1无法读取最新的记录A，不然就是发生脏读的情况，所以应该读记录A被事务2修改之前的数据，但是记录A已经被事务2改了呀，所以事务1咋办？只能用锁阻塞等待事务2的提交，这种实现叫 LBCC(Lock-Based Concurrent Control)。
- ![图片](https://mmbiz.qpic.cn/mmbiz_png/eSdk75TK4nFQRcyKribkGLAdbs84qGjBwR1TdchQiczmBfIC55Bn9kQdZ0ahvuDeOn79jfFCgtePJnPEG6gwP0Sg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)
- 如果有多版本的话，就不一样了。事务2修改的记录 A，还未提交，但是记录 A 被修改之前的版本还在，此时事务1就可以读取之前的版本数据，这样读写之间就不会阻塞啦，所以说 MVCC 提高了事务的并发度，提升数据库的性能。
- undo log  
  - undo log  会记录修改数据之前的信息，事务中的原子性就是通过 undo log 来实现的。
  - 所以，有 undo log 可以帮我们找到「版本」的数据
  - 实际上 InnoDB 不会真的存储了多个版本的数据，只是借助 undolog 记录每次写操作的反向操作，所以索引上对应的记录只会有一个版本，即最新版本。只不过可以根据 undolog 中的记录反向操作得到数据的历史版本，所以看起来是多个版本。
  - ![图片](https://mmbiz.qpic.cn/mmbiz_png/eSdk75TK4nFQRcyKribkGLAdbs84qGjBwsHFHKAicumYvibqWcMBH1uUSUTPzicNcgVL3WlrLKo6hHGWxic3qyKibI9w/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)
  - trx_id：当前事务ID。
  - roll_pointer：指向 undo log 的指针。
  - 以 `insert （1，XX）` 这条语句举例，成功插入之后数据页的记录上不仅存储 ID 1，name XX，还有 trx_id 和 roll_pointer 这两个隐藏字段
  - ![图片](https://mmbiz.qpic.cn/mmbiz_png/eSdk75TK4nFQRcyKribkGLAdbs84qGjBwCrqYufDicwJkAgKcv6pUJqicRWlyHmGefufNBe9iaF2W0tXiaWLicXzGlsw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)
  - 从图中可以得知此时插入的事务ID是1，此时插入会生成一条 undolog ，并且记录上的 roll_pointer 会指向这条 undolog ，而这条 undolog  是一个类型为 `TRX_UNDO_INSERT_REC` 的 log，代表是 insert 生成的，里面存储了主键的长度和值(还有其他值，不提)。
  - 所以 InnoDB 可以根据 undolog  里的主键的值，找到这条记录，然后把它删除来实现回滚(复原)的效果。因此可以简单地理解 undolog 里面存储的就是当前操作的反向操作，所以认为里面存了个 `delete 1` 就行。
  - 此时事务1提交，然后另一个 ID 为 5 的事务再执行 `update NO where id 1` 这个语句，此时的记录和 undolog 就如下图所示：
  - ![图片](https://mmbiz.qpic.cn/mmbiz_png/eSdk75TK4nFQRcyKribkGLAdbs84qGjBwzEUm7OnNVym2n5j0cL8Bp7SbIqTPTwmDDMKIZmagicLCUPc1LbjpMAw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)
  - 
- read view 
  - 而 read view 实际上就是在查询时，InnoDB 会生成一个 read view
  - read view 有几个重要的字段，分别是：trx_ids（尚未提交 commit 的事务版本号集合），up_limit_id（下一次要生成的事务 ID 值），low_limit_id（尚未提交版本号的事务 ID 最小值）以及 creator_trx_id（当前的事务版本号）
  - 在每行数据有两列隐藏的字段，分别是 DB_TRX_ID（记录着当前 ID）以及 DB_ROLL_PTR（指向上一个版本数据在 undo log 里的位置指针）
- MVCC 其实就是**靠「比对版本」来实现读写不阻塞**，而**版本的数据存在于 undo log 中**。
- 针对于不同的隔离级别（read commit 和 repeatable read），无非就是 **read commit 隔离级别下，每次都获取一个新的 read view，repeatable read 隔离级别则每次事务只获取一个 read view**。

# 总结

- 事务为了保证数据的最终一致性

- 事务有四大特性，分别是原子性、一致性、隔离性、持久性

- 原子性由 undo log 保证

- 持久性由 redo log 保证

- 隔离性由数据库隔离级别供我们选择，分别有 read uncommit, read commit, repeatable read, serializable

- 一致性是事务的目的，一致性由应用程序来保证

- 事务并发会存在各种问题，分别有脏读、重复读、幻读问题。上面的不同隔离级别可以解决掉由于并发事务所造成的问题，而隔离级别实际上就是由 MySQL 锁来实现的

- 频繁加锁会导致数据库性能低下，引入了 MVCC 多版本控制来实现读写不阻塞，提高数据库性能

- MVCC 原理即通过 read view 以及 undo log 来实现
