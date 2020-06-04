---
title: "MySQL Isolation Level Introduction"
date: 2020-06-04T10:23:52+08:00
draft: true
---

# 事物的隔离级别

当对数据库的操作是并发的，多个事物互相交叉运行，可能会出现不同的事物，对同一个数据进行读写操作，极大的概率会破坏预期。mysql的innodb存储引擎提出了四种隔离级别；

* 读未提交     （read uncommitted）
* 读提交       （read committed）
* 可重复读     （read repeated）
* 串行化       （serializable）

隔离级别定义了事物之间相互影响的关系，比如最严格的串行化，意味着两个事物对同一批数据进行操作，只能是其中一个事物执行，另外的事物“排队”等待，当前面的事物执行完毕，才能执行，这就说明了，事物之间不会存在影响， 其他的三种隔离级别都不满足严格的【事物】定义。


# 脏读

事物1可以读到事物2未提交的数据【脏数据】， 这个叫做脏读。

```sql
mysql> select * from foo;
+----+-------------+------+
| id | name        | year |
+----+-------------+------+
|  1 | li_lei      | 1991 |
|  2 | han_mei_mei | 1992 |
|  3 | shang_hai   | 1993 |
+----+-------------+------+
```

![image](/img/read-uncommitted-read-dirty.png)

# 不可重复读

事物1中多次读取同一条数据的结果不一致，称为不可重复读。这里和脏读会有类似的感觉，两者之间的区别是；
* 脏读读到的是其他事物未提交的数据
* 不可重复读读到的是其他事物已经提交的数据

```sql
mysql> select * from foo;
+----+-------------+------+
| id | name        | year |
+----+-------------+------+
|  1 | li_lei      | 2000 |
|  2 | han_mei_mei | 1992 |
|  3 | shang_hai   | 1993 |
+----+-------------+------+
```

![image](/img/read-unrepeated.png)

# 幻读

幻读，事物1读取到的行数和之前读取到的不一致。

```sql
mysql>  INSERT INTO `foo` (id, name, year) values (4, "li_jin_zhi", 1994);
Query OK, 1 row affected (0.01 sec)

mysql> select count(*) from foo where id < 10;
+----+-------------+------+
| id | name        | year |
+----+-------------+------+
|  1 | li_lei      | 2000 |
|  2 | han_mei_mei | 1992 |
|  3 | shang_hai   | 1993 |
|  4 | li_jin_zhi  | 1994 |
+----+-------------+------+
```

![image](/img/p-r.png)


# MySQL隔离级别
隔离级别低，性能会比较高，但是也会带来严重的数据一致问题，串行化会是去并发性，对于数据一致性要求很高的场景才会使用，综合下来， mysql的默认级别是read repeated，sqlserver默认的级别也是这个。但是sql标准中的RR会有幻读的问题，mysql是如何解决这个问题的呢？

# 多版本并发控制【MVCC】

RR解决脏读、不可重复读、幻读等问题，使用的是MVCC：MVCC全称Multi-Version Concurrency Control，即多版本的并发控制协议。

MVCC最大的优点是读不加锁，因此读写不冲突，并发性能好。InnoDB实现MVCC，多个版本的数据可以共存，主要是依靠数据的隐藏列(也可以称之为标记位)和undo log。其中数据的隐藏列包括了该行数据的版本号、删除时间、指向undo log的指针等等；当读取数据时，MySQL可以通过隐藏列判断是否需要回滚并找到回滚需要的undo log，从而实现MVCC；隐藏列的详细格式不再展开。

下面结合前文提到的几个问题分别说明。

（1）脏读

![image](/img/read-uncommitted-read-dirty.png)


当事务1在T3时间节点读取lilei年纪的时候，会发现数据已被其他事务修改，且状态为未提交。此时事务1读取最新数据后，根据数据的undo log执行回滚操作，得到事务2修改前的数据，从而避免了脏读。

（2）不可重复读

![image](/img/read-unrepeated.png)

当事务1在T2节点第一次读取数据时，会记录该数据的版本号（数据的版本号是以row为单位记录的），假设版本号为1；当事务2提交时，该行记录的版本号增加，假设版本号为2；当事务1在T4再一次读取数据时，发现数据的版本号（2）大于第一次读取时记录的版本号（1），因此会根据undo log执行回滚操作，得到版本号为1时的数据，从而实现了可重复读。

（3）幻读

InnoDB实现的RR通过next-key lock机制避免了幻读现象。

next-key lock是行锁的一种，实现相当于record lock(记录锁) + gap lock(间隙锁)；其特点是不仅会锁住记录本身(record lock的功能)，还会锁定一个范围(gap lock的功能)。当然，这里我们讨论的是不加锁读：此时的next-key lock并不是真的加锁，只是为读取的数据增加了标记（标记内容包括数据的版本号等）；准确起见姑且称之为类next-key lock机制。还是以前面的例子来说明：

![image](/img/p-r.png)

当事务1在T1节点第一次读取id<10数据时，标记的不只是id 1/2/3的数据，而是将范围(0,10)进行了标记，这样当T5时刻再次读取id<10数据时，便可以发现id=4的数据比之前标记的版本号更高(任何版本都比无要高)，此时再结合undo log执行回滚操作，避免了幻读。




