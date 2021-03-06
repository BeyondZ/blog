
关于"幻读"现象的一些说法好像并不统一，下面摘一下来自百度百科引用。

> 幻读是指当事务不是独立执行时发生的一种现象，例如第一个事务对一个表中的数据进行了修改，比如这种修改涉及到表中的“全部数据行”。同时，第二个事务也修改这个表中的数据，这种修改是向表中插入“一行新数据”。那么，以后就会发生操作第一个事务的用
    户发现表中还存在没有修改的数据行，就好象发生了幻觉一样.一般解决幻读的方法是增加范围锁RangeS，锁定检索范围为只读，这样就避免了幻读。


这里我也说一下我对幻读现象的理解，或者说MySQL的InnoDB引擎的幻读现象是如何处理的，
这篇主要就只讨论RR隔离级别下的幻读。基本不会对比着其他隔离级别以及脏读、不可重复读等现象。

- [1. 一些相关的技术背景](https://github.com/BBLLMYD/blog/blob/master/blogs/%E6%B5%85%E6%9E%90InnoDB%E4%B8%8B%E7%9A%84%E5%B9%BB%E8%AF%BB.md#1-%E4%B8%80%E4%BA%9B%E7%9B%B8%E5%85%B3%E7%9A%84%E6%8A%80%E6%9C%AF%E8%83%8C%E6%99%AF)
- [2. InnoDB引擎下的"幻读"](https://github.com/BBLLMYD/blog/blob/master/blogs/%E6%B5%85%E6%9E%90InnoDB%E4%B8%8B%E7%9A%84%E5%B9%BB%E8%AF%BB.md#2-innodb%E5%BC%95%E6%93%8E%E4%B8%8B%E7%9A%84%E5%B9%BB%E8%AF%BB)
- [3. 其他 & 总结](https://github.com/BBLLMYD/blog/blob/master/blogs/%E6%B5%85%E6%9E%90InnoDB%E4%B8%8B%E7%9A%84%E5%B9%BB%E8%AF%BB.md#3-%E5%85%B6%E4%BB%96--%E6%80%BB%E7%BB%93)

---

### 1. 一些相关的技术背景

- MVCC

MySQL多版本并发控制实现了MVCC机制，MVCC通过MVCC使用时间戳，或"自动增量的事务ID"等手段，意图解决读写锁造成的多个、长时间的读操作饿死写操作等问题。
其基础是快照读和当前读机制，所以每一行记录都有着一些隐藏列如DATA_TRX_ID、DATA_ROLL_PTR、DB_ROW_ID等。

- 相关的锁

**幻读现象出现的本质是因为没法锁住不存在的行。** 
大家知道在数据库标准的RR隔离级别下，是不可以避免幻读的，
而MySQL的InnoDB引擎通过Next-Key锁（行锁+Gap锁）来避免幻读（其实是在特定场景下能避免幻读）。

这也是导致说法不统一的一个重要原因。或者说，InnoDB是可以一定程度的避免幻读。

在一个事务内的读是要分为**快照读**和**当前读**的，
**如果在快照读操作（select）后面跟着当前读操作（select for update），那就是无法避免幻读的，是因为第二次当前读之前这时候Next-Key锁还没来得及发挥作用。**
也就是说这时候的"读"其实已经不仅仅是"读"了。

---

### 2. InnoDB引擎下的"幻读"

分别举个例子（**默认都在MySQL InnoDB的RR隔离级别下**）：

- 2.1 快照读造成的事务执行失败eg

|     T1     |     T2    |
|      :-        |     :-      |
|      `start transaction;`        |          |
|      1. `select * from users where id = 1;`        |           |
|             |     1. `insert into users(id, name) values (1, 'skrT2');`      |
|      2. `insert into users(id, name) values (1, 'skrT1');`     |           |
|      `commit;`     |           |

"skrT1"值会插入失败。但是这在T1事务内部看起来好像是自洽的，但是执行到T1.2步骤的时候会因为主键冲突无法插入，因为T2的事务操作结果此过程对T1是不可见的。

**看不到并不代表没有。**

--- 

- 2.2 "解决"上述流程的问题eg

|     T1     |     T2    |
|      :-        |     :-      |
|      `start transaction;`        |     `start transaction;`      |
|      1. `select * from users where id = 1;`        |           |
|             |     `insert into users(id, name) values (1, 'skrT2');`      |
|             |     `commit;`      |
|      2. `select * from users where id = 1 for update;`     |           |
|      3. `insert into users(id, name) values (1, 'skrT1');`     |           |
|      `commit;`     |           |

这次在T1 insert之前执行了一次for update的select，此时的读是当前读会读取到T2插入的数据，这时候虽然也会插入失败，好像T1事务内看起来至少合理了一些。
但是这种操作明显不太合自然逻辑了。

**当前读可以看到最新的数据。**

--- 

- 2.3 再次"解决"eg


|     T1     |     T2    |
|      :-        |     :-      |
|      `start transaction;`        |     `start transaction;`      |
|      1. `select * from users where id = 1 for update;`        |           |
|             |     `insert into users(id, name) values (1, 'skrT2');`（阻塞）      |
|      2. `insert into users(id, name) values (1, 'skrT1');`     |           |
|      `commit;`     |           |


而这种情况如果是在T1.1操做的时刻做for update的select**看起来貌似是没意义的**，因为上面提到了**锁此时没法锁住不存在的行**。
但是实际上这种操作会阻塞T2事务进行，T2事务此时是无法插入数据的。**因为这种情况下T1.1操作锁定的是索引范围，与实体记录是否真实存在就没有关系了。**

此时Gap锁生效，阻止多个事务将记录插入到同一范围内的可能性。

RR隔离级别下的Next-Key锁存在一个类似升降级的过程，就是**当索引字段上的等值查询并满足定位到具体行会升级为行锁，此时无Gap锁生效，否则会增量Gap间隙锁机制来避免幻读**。

---

- 2.4 Gap导致的"死锁"eg

**PS：假设表中目前缺失id = 17的记录**

|     T1     |     T2    |
|      :-        |     :-      |
|      `start transaction;`        |          |
|      1. `select * from users where id = 17 for update;`        |           |
|             |     `start transaction;`      |
|             |      1. `select * from users where id = 17 for update;`         |
|      | 2. `insert into users(id, name) values (17, 'skrT2');`（阻塞）     |           |
|        2. `insert into users(id, name) values (17, 'skrT1');`（阻塞）      |

由于Gap锁并不互斥，T1.1释放之前T2.1也获取到了相同的间隙锁，
此时T1的持有的锁不允许T2.2的insert操作执行，T2的锁也不允许T1.2执行，也就产生了死锁。

间隙锁有几个特点，一是非排他锁，二是锁住一段范围后不允许其他事务在当前锁住的范围内进行DDL操作。
这种机制的目的是为了避免幻读出现，但是为了不影响整体读操作的吞吐量，同时也导致了死锁的可能性。

此时的死锁是由间隙锁导致，不管是间隙锁还是行锁，死锁都会被引擎检测到，
然后根据一些成本条件当作变量权衡出其中一个事务回滚，将另一个事务提交，被回滚的事务会返回给 Client 一个 Deadlock 异常。

---

我理解的 for update 或是 lock in share mode 下的的 select 操作更像是提供给用户用来拿到新版本数据的一个"钩子"，虽然这看起来和隔离的思想有些相悖了，但是这种设计必然有他的价值和更适合应用场景。

所以此时只要不能阻塞T2的情况下，只要T2率先插入记录，T1是必然要回滚或者失败的。除非采用读写冲突的Serializable隔离级别下。

这里举的例子是简单情况下的insert操作，实际update操作也是同样的道理，锁只是手段，本质上都是**当前读和快照读的不一致，或者快照读读到的旧版本数据导致**。

- insert时的异常情况是，我本来要插入本应该能够合理（依据刚刚读出的逻辑）插入的数据，却出现了不能正常插入的现象（要插入的数据有可能被其他线程插入了）。<br>
- update时的异常现象是，我本来要更新本应该能够合理（依据刚刚读出的逻辑）更新的数据，却出现了没有正常更新的情况（要更新的数据有可能被其他线程给删了）。

---

### 3. 其他 & 总结

数据库的四个隔离级别是一套规范标准，MySQL有MySQL的实现，Oracle有Oracle的实现；
到了MySQL的不同引擎还有着不同引擎的具体实现。

数据库的锁就是实现这些隔离级别的工具，根据锁的资源的粒度和读写场景有很多锁的实现，
MySQL的InnoDB引擎下与"幻读"现象更相关的包括读锁，写锁，Next-Key锁等，Next-Key只有在RR隔离级别下才工作。

到这里其实可以发现，实际上更多是取决于我们怎么定义"幻读"以及怎么利用或者避免，而目的是在此场景下如何能保证程序的执行能够按正确的、我们想要的逻辑来执行。
因为避免幻读的本质其实是**根据写操作来验证前面读的内容是幻象。** 也就是说 InnoDB 的 RR 隔离级别下其实更灵活，提供给你了可以"有选择性幻读"的条件。








