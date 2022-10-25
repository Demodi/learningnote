# 一、InnoDB 一棵 B+ 树可以存放多少行数据？

- InnoDB 一棵 B+ 树可以存放多少行数据？**约 2 千万**。
- 计算机在存储数据的时候，有**最小存储单元**
- 在计算机中磁盘存储数据最小单元是扇区，一个扇区的大小是 512 字节，而文件系统（例如 XFS/EXT4）他的最小单元是块，一个块的大小是 4k，而对于我们的 InnoDB 存储引擎也有自己的最小储存单元 —— 页（Page），一个页的大小是 16K。

# 二、理解最小存储单元

- 文件系统中一个文件大小只有 1 个字节，但不得不占磁盘上 4KB 的空间。
- Innodb 的所有数据文件（后缀为 ibd 的文件），他的大小始终都是 16384（16k）的整数倍。![img](https://cdn.learnku.com/uploads/images/202111/03/34535/M3YyCRBOnu.jpeg!large)
- 磁盘扇区、文件系统、InnoDB 存储引擎都有各自的最小存储单元。![img](https://cdn.learnku.com/uploads/images/202111/03/34535/IgdkOPrScE.jpeg!large)
- 在 MySQL 中我们的 **InnoDB 页的大小默认是 16k**，当然也可以通过参数设置：![img](https://cdn.learnku.com/uploads/images/202111/03/34535/CMCTNvdeHR.jpeg!large)
- 数据表中的数据都是存储在页中的，所以一个页中能存储多少行数据呢？假设一行数据的大小是 1k，那么一个页可以存放 16 行这样的数据。
- 如果数据库只按这样的方式存储，那么如何查找数据就成为一个问题，因为我们不知道要查找的数据存在哪个页中，也不可能把所有的页遍历一遍，那样太慢了。
- 用 B+ 树的方式组织这些数据。![img](https://cdn.learnku.com/uploads/images/202111/03/34535/54b6M3Sa2H.jpeg!large)
- 