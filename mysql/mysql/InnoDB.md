# InnoDB解析

### 数据库的定义

很多开发者在最开始时其实都对数据库有一个比较模糊的认识，觉得数据库就是一堆数据的集合，但是实际却比这复杂的多，数据库领域中有两个词非常容易混淆，也就是数据库和实例：

	* 数据库：物理操作文件系统或其他形式文件类型的集合；
	* 实例：MySQL 数据库由后台线程以及一个共享内存区组成；

### 数据库和实例

在 MySQL 中，实例和数据库往往都是一一对应的，而我们也无法直接操作数据库，而是要通过数据库实例来操作数据库文件，可以理解为数据库实例是数据库为上层提供的一个专门用于操作的接口。

在 Unix 上，启动一个 MySQL 实例往往会产生两个进程，mysqld 就是真正的数据库服务守护进程，而 mysqld_safe 是一个用于检查和设置 mysqld 启动的控制程序.
它负责监控 MySQL 进程的执行，当 mysqld 发生错误时，mysqld_safe 会对其状态进行检查并在合适的条件下重启。

### 数据的存储

在 InnoDB 存储引擎中，所有的数据都被逻辑地存放在表空间中
`表空间`（tablespace）是存储引擎中最高的存储逻辑单位，在表空间的下面又包括

    段（segment）、
    区（extent）、
    页（page）：

![|center](./src/innodb-1.png)

同一个数据库实例的所有表空间都有相同的页大小；
默认情况下，表空间中的页大小都为 16KB，
当然也可以通过改变 innodb_page_size 选项对默认大小进行修改，
需要注意的是不同的页大小最终也会导致区大小的不同：

![|center](./src/innodb-2.png)


在 InnoDB 存储引擎中，一个区的大小最小为 1MB，页的数量最少为 64 个。

### 如何存储表

MySQL 使用 InnoDB 存储表时，会将表的定义和数据索引等信息分开存储，其中前者存储在 .frm 文件中，后者存储在 .ibd 文件中，这一节就会对这两种不同的文件分别进行介绍。

![|center](./src/innodb-3.png)


#### .frm 文件
无论在 MySQL 中选择了哪个存储引擎，所有的 MySQL 表都会在硬盘上创建一个 .frm 文件用来描述表的格式或者说定义；.frm 文件的格式在不同的平台上都是相同的。

#### .ibd 文件
InnoDB 中用于存储数据的文件总共有两个部分，一是系统表空间文件，包括 ibdata1、ibdata2 等文件，其中存储了 InnoDB 系统信息和用户数据库表数据和索引，是所有表公用的。
当打开 innodb_file_per_table 选项时，.ibd 文件就是每一个表独有的表空间，文件存储了当前表的数据和相关的索引数据。

### 索引
索引是数据库中非常非常重要的概念，它是存储引擎能够快速定位记录的秘密武器，对于提升数据库的性能、减轻数据库服务器的负担有着非常重要的作用；索引优化是对查询性能优化的最有效手段，它能够轻松地将查询的性能提高几个数量级。

#### 索引的数据结构

在上一节中，我们谈了行记录的存储和页的存储，在这里我们就要从更高的层面看 InnoDB 中对于数据是如何存储的；InnoDB 存储引擎在绝大多数情况下使用 B+ 树建立索引，这是关系型数据库中查找最为常用和有效的索引，但是 B+ 树索引并不能找到一个给定键对应的具体值，它只能找到数据行对应的页，然后正如上一节所提到的，数据库把整个页读入到内存中，并在内存中查找具体的数据行。

![|center](./src/innodb-4.png)


B+ 树是平衡树，它查找任意节点所耗费的时间都是完全相同的，比较的次数就是 B+ 树的高度；

### 锁

我们都知道锁的种类一般分为`乐观锁`和`悲观锁`两种。
InnoDB 存储引擎中使用的就是`悲观锁`，而按照锁的粒度划分，也可以分成`行锁`和`表锁`。


并发控制机制乐观锁和悲观锁其实都是并发控制的机制，同时它们在原理上就有着本质的差别；

	* 乐观锁是一种思想，它其实并不是一种真正的『锁』，它会先尝试对资源进行修改，在写回时判断资源是否进行了改变，如果没有发生改变就会写回，否则就会进行重试，在整个的执行过程中其实都没有对数据库进行加锁；
	* 悲观锁就是一种真正的锁了，它会在获取资源前对资源进行加锁，确保同一时刻只有有限的线程能够访问该资源，其他想要尝试获取资源的操作都会进入等待状态，直到该线程完成了对资源的操作并且释放了锁后，其他线程才能重新操作资源；

虽然乐观锁和悲观锁在本质上并不是同一种东西，一个是一种思想，另一个是一种真正的锁，但是它们都是一种并发控制机制。

![|center](./src/innodb-5.png)

乐观锁不会存在死锁的问题，但是由于更新后验证，所以当冲突频率和重试成本较高时更推荐使用悲观锁，而需要非常高的响应速度并且并发量非常大的时候使用乐观锁就能较好的解决问题，在这时使用悲观锁就可能出现严重的性能问题；在选择并发控制机制时，需要综合考虑上面的四个方面（冲突频率、重试成本、响应速度和并发量）进行选择。

### 锁的种类

对数据的操作其实只有两种，也就是读和写，而数据库在实现锁时，也会对这两种操作使用不同的锁；
InnoDB 实现了标准的行级锁，也就是共享锁（Shared Lock）和互斥锁（Exclusive Lock）；
共享锁和互斥锁的作用其实非常好理解：

	* 共享锁（读锁）：允许事务对一条行数据进行读取；
	* 互斥锁（写锁）：允许事务对一条行数据进行删除或更新；

而它们的名字也暗示着各自的另外一个特性，共享锁之间是兼容的，而互斥锁与其他任意锁都不兼容：

稍微对它们的使用进行思考就能想明白它们为什么要这么设计，因为共享锁代表了读操作、互斥锁代表了写操作，所以我们可以在数据库中并行读，但是只能串行写，只有这样才能保证不会发生线程竞争，实现线程安全。

### 锁的粒度

无论是共享锁还是互斥锁其实都只是对某一个数据行进行加锁，
InnoDB 支持多种粒度的锁，也就是行锁和表锁；
为了支持多粒度锁定，InnoDB 存储引擎引入了意向锁（Intention Lock），
意向锁就是一种`表级`锁。

	* 意向共享锁：事务想要在获得表中某些记录的共享锁，需要在表上先加意向共享锁；
	* 意向互斥锁：事务想要在获得表中某些记录的互斥锁，需要在表上先加意向互斥锁；

意向锁其实不会阻塞全表扫描之外的任何请求，它们的主要目的是为了表示是否有人请求锁定表中的某一行数据。


>有的人可能会对意向锁的目的并不是完全的理解，我们在这里可以举一个例子：如果没有意向锁，当已经有人使用行锁对表中的某一行进行修改时，如果另外一个请求要对全表进行修改，那么就需要对所有的行是否被锁定进行扫描，在这种情况下，效率是非常低的；不过，在引入意向锁之后，当有人使用行锁对表中的某一行进行修改之前，会先为表添加意向互斥锁（IX），再为行记录添加互斥锁（X），在这时如果有人尝试对全表进行修改就不需要判断表中的每一行数据是否被加锁了，只需要通过等待意向互斥锁被释放就可以了。

### 死锁的发生

既然 InnoDB 中实现的锁是悲观的，那么不同事务之间就可能会互相等待对方释放锁造成死锁，最终导致事务发生错误；
想要在 MySQL 中制造死锁的问题其实非常容易：

![|center](./src/innodb-6.png)

两个会话都持有一个锁，并且尝试获取对方的锁时就会发生死锁，不过 MySQL 也能在发生死锁时及时发现问题，并保证其中的一个事务能够正常工作，这对我们来说也是一个好消息。

### 事务与隔离级别

在介绍了锁之后，我们再来谈谈数据库中一个非常重要的概念 —— 事务；
相信只要是一个合格的软件工程师就对事务的特性有所了解，其中被人经常提起的就是事务的原子性，
在数据提交工作时，要么保证所有的修改都能够提交，要么就所有的修改全部回滚。
但是事务还遵循包括原子性在内的 ACID 四大特性：
    
    原子性（Atomicity）、
    一致性（Consistency）、
    隔离性（Isolation）
    持久性（Durability）；
    
### 几种隔离级别

事务的隔离性是数据库处理数据的几大基础之一，而隔离级别其实就是提供给用户用于在性能和可靠性做出选择和权衡的配置项。
ISO 和 ANIS SQL 标准制定了四种事务隔离级别，而 InnoDB 遵循了 SQL:1992 标准中的四种隔离级别：READ UNCOMMITED、READ COMMITED、REPEATABLE READ 和 SERIALIZABLE；每个事务的隔离级别其实都比上一级多解决了一个问题：

	* RAED UNCOMMITED：使用查询语句不会加锁，可能会读到未提交的行（Dirty Read）；
	* READ COMMITED：只对记录加记录锁，而不会在记录之间加间隙锁，所以允许新的记录插入到被锁定记录的附近，所以再多次使用查询语句时，可能得到不同的结果（Non-Repeatable Read）；
	* REPEATABLE READ：多次读取同一范围的数据会返回第一次查询的快照，不会返回不同的数据行，但是可能发生幻读（Phantom Read）；
	* SERIALIZABLE：InnoDB 隐式地将全部的查询语句加上共享锁，解决了幻读的问题；

MySQL 中默认的事务隔离级别就是 REPEATABLE READ，但是它通过 Next-Key 锁也能够在某种程度上解决幻读的问题。

#### 脏读

当事务的隔离级别为 READ UNCOMMITED 时，我们在 SESSION 2 中插入的未提交数据在 SESSION 1中是可以访问的。

![|center](./src/innodb-7.png)


#### 不可重复读

当事务的隔离级别为 READ COMMITED 时，虽然解决了脏读的问题，但是如果在 SESSION 1 先查询了一个范围的数据，在这之后 SESSION 2 中插入一条数据并且提交了修改，在这时，如果 SESSION 1 中再次使用相同的查询语句，就会发现两次查询的结果不一样。

不可重复读的原因就是，在 READ COMMITED 的隔离级别下，存储引擎不会在查询记录时添加间隙锁，锁定 id < 5 这个范围。

![|center](./src/innodb-8.png)


#### 幻读

重新开启了两个会话 SESSION 1 和 SESSION 2，在 SESSION 1 中我们查询全表的信息，没有得到任何记录；在 SESSION 2 中向表中插入一条数据并提交；由于 REPEATABLE READ 的原因，再次查询全表的数据时，我们获得到的仍然是空集，但是在向表中插入同样的数据却出现了错误。

![|center](./src/innodb-9.png)

这种现象在数据库中就被称作幻读，虽然我们使用查询语句得到了一个空的集合，但是插入数据时却得到了错误，好像之前的查询是幻觉一样。
在标准的事务隔离级别中，幻读是由更高的隔离级别 SERIALIZABLE 解决的，但是它也可以通过 MySQL 提供的 Next-Key 锁解决：

![|center](./src/innodb-10.png)

REPERATABLE READ 和 READ UNCOMMITED 其实是矛盾的，如果保证了前者就看不到已经提交的事务，如果保证了后者，就会导致两次查询的结果不同，MySQL 为我们提供了一种折中的方式，能够在 REPERATABLE READ 模式下加锁访问已经提交的数据，其本身并不能解决幻读的问题，而是通过文章前面提到的 Next-Key 锁来解决。

