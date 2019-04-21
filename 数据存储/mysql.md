> ## mysql存储引擎及索引

#### Mysql两种常用存储引擎myisam和innodb

1. 锁的粒度不同：
   myisam 不支持事物，只支持表级锁，不支持外键。
   innodb 支持事务和行级锁，支持外键。

2. 索引不同（见下）：

   

#### mySql 索引（存储原理）

mysql索引使用了b+树这一数据结构，因此在讲解mysql索引前须先介绍b+树。

#### 介绍b+树

首先介绍B树，B树事实上是一种平衡的多叉查找树。每个结点都相当于数据。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190223100155353.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=,size_16,color_FFFFFF,t_70)
而B+树相对于b树的差别在于：

- 非叶结点仅具有索引作用。
- 树的所有叶结点构成一个有序链表。
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190225154900411.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=,size_16,color_FFFFFF,t_70)

#### 



### 正式介绍mysql索引

mysql索引使用了b+树这一数据结构，**实际上每个叶子节点都是一个磁盘页**，同时，mysql索引同时包含了**主键b+树以及辅助键b+树**。

#### innodb和myisam对于索引的区别

innodb和myisam对于索引的不同之处在于

- InnoDB使用的是聚簇索引，数据直接存在磁盘页（叶子节点）
- MyISAM使用的是非聚簇索引，叶子节点存储的是数据所在的地址。

我们通过下例来进行介绍：

例如创建下表：

```
create table person(
id int primary key,
name varchar(64) not null,
company varchar(64) not null
index (company)engine=InnoDB;
```

对于InnoDB，数据直接存在叶子节点，下图中的左半部分就是innoDB的索引结构，叶子节点存放的就是最下一行的数据。

于此同时，我们可以看到，每一个引擎都有两颗索引树，主键B+树中叶子节点存放的是行数据（数据库中一行的所有数据），而辅助键B+树只存储辅助键和主键。若使用"where id = 14"这样的条件查找主键，则在主索引B+树中获取id对应的数据。若对Name列进行条件搜索，则需要两个步骤：从辅助键b+树中获取name对应的id。第二步在主索引B+树中获取id对应的数据。

MyISAM使用的是非聚簇索引，非聚簇索引的两棵B+树没什么不同，这两颗B+树的叶子节点都使用一个地址指向真正的表数据，通过辅助键检索无需访问主键的索引树。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190211111428326.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=,size_16,color_FFFFFF,t_70)

#### 两个引擎索引的对比：

**实际上，大多数情况下推荐使用innodb**。但有两种情况推荐使用mysaim。

1、因为innodb索引将数据直接存在磁盘页。所以在非末尾插入数据的话，**会造成叶子节点的行数据所在磁盘位置移动**，插入的效率低，因此innodb建议使用递增的值作为主键（例如自增id），不建议使用无序主键（例如UUID）

个人认为好的解决方法：主键使用自增id，辅助键使用uuid

2、同样，innodb的更新效率也不高，因此如果将数据库用于**日志记录**时，因为日志信息会一直刷，因此数据库的更新频率就会过高，造成大量的磁盘IO。


&nbsp;

&nbsp;

> ## sql优化

#### sql语句查询慢的解决思路

1. 开个慢查询记录，定位到具体sql

```sql
mysql> show variables like 'long%';
long_query_time | 10.000000
mysql> set long_query_time=1;
mysql> show variables like 'slow%';
slow_launch_time    | 2
slow_query_log      | ON
slow_query_log_file | /tmp/slow.log
mysql> set global slow_query_log='ON'
```

2. 使用explain，然后进行分析：
   比方说分析后发现该语句没有用到索引：a、建立索引  b、优化sql语句
   explain教程：https://blog.csdn.net/why15732625998/article/details/80388236
3. 如果最后确定是因为数据量太大导致，就分表分库

#### 引起全表查询的几个操作（应尽量避免）

https://zhidao.baidu.com/question/693081430483912364.html



#### 数据库如何应对大规模写入和读取？

1. 读写分离
   读写分离就是在**主服务器上修改数据**，数据会同步到从服务器，**从服务器**只能**读取数据**，不能写入。
   ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190207212322366.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=,size_16,color_FFFFFF,t_70)
2. 分库分表
   **分表**（解决单表数据量过大带来的查询效率下降）：
   水平切分：**不修改数据库表结构**，通过**对表中数据的拆分**而达到分片的目的，一般水平拆分在查询数据库的时候可能会用到 union 操作（多个结果并）。
    **可以根据hash或者日期进行进行分表。**
    垂直切分：**修改表结构**，按照访问的差异**将某些列拆分出去**，一般查询数据的时候可能会用到 join 操作；把常用的字段放一个表,不常用的放一个表，把字段比较大的比如text的字段拆出来放一个表里面。
    **分库**（提升并发能力）：采用关键字取模的方法
    ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190207155240141.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=,size_16,color_FFFFFF,t_70)
3. 使用nosql

#### 数据库事务及其隔离级别

1. 特性：
   （1）原子性（atomicity） ：表示一个事务内的所有操作都是一个整体，要么全部成功，要么全部失败。
   （2）一致性（consistency） ：也就是不管事务有多少，系统的数据都会保持着一致（不管几个用户在转账，系统的总额是不变的）。
    （3）隔离性(isoation) ：多个并发事务之间的操作不会干扰。
    （4）持久性(durability) ：事务完成之后，它对于系统的影响是永久性的。
    **数据库的ACID是一种强一致性模型，即写入了什么数据，就能读到什么数据。
    不同于数据库，分布式理论中BASE则不强求强一致性，而是选择通过牺牲强一致性来获得可用性。**（此处可引导面试关让你讲分布式理论）
2. 隔离问题：
   脏读：一个事务读取了另一个事务没有提交的数据（这个数据可能会回滚）
   不可重复读：在一个事务里面读取了两次某个数据，读出来的数据不一致（update）
   虚读(幻读)：两次读取，发现多了/少了一些数据（insert，delete）
3. 隔离级别：
   read uncommitted：读未提交。存在3个问题
   read committed：读已提交。解决脏读(写数据只会锁住相应的行)，存在2个问题 
   repeatable read：（mysql默认）可重复读。解决：脏读、不可重复读，存在1个问题。
   serializable ：串行化。都解决，但并发效率很低。

#### 的最左匹配原则（即从左往右匹配）

在联合索引中，从左往右进行匹配。
例如建立(a,b,c,d)顺序的索引，那么匹配顺序是a，b，c，d
KEY test_col1_col2_col3 on test(col1,col2,col3);
联合索引 test_col1_col2_col3 实际建立了(col1)、(col1,col2)、(col,col2,col3)三个索引。
`SELECT * FROM test WHERE col1=“1” AND clo2=“2” AND clo4=“4”`
上面这个查询语句执行时会依照最左前缀匹配原则，检索时会使用索引(col1,col2)进行数据匹配。
注：下面两条语句效果实际上是一样的，因为第二条语句会被mysql优化器调整第一条

```
SELECT * FROM test WHERE col1=“1” AND clo2=“2”
SELECT * FROM test WHERE col2=“2” AND clo1=“1”
```

#### 为什么不适用二叉查找树，而是使用B树（包括B+）

1、B树包括B+树的设计思想都是尽可能的降低树的高度，以此降低磁盘IO的次数，因为一个索引节点就表示一个磁盘页，页的换入换出次数越多，表示磁盘IO次数越多，越低效。
2、B树的每个节点可以存储多个关键字，它将节点大小设置为磁盘页的大小，充分利用了磁盘预读的功能。每次读取磁盘页时就会读取一整个节点。

```
磁盘预读：磁盘往往不是严格按需读取，而是每次都会预读，即使只需要一个字节，
磁盘也会从这个位置开始，顺序向后读取一定长度的数据放入内存。
这样做的理论依据是计算机科学中著名的局部性原理： 
当一个数据被用到时，其附近的数据也通常会马上被使用。 
程序运行期间所需要的数据通常比较集中。 
```

具体深度能降到多少？
之前看到一个，说5亿条数据大概也就3-4层。



#### 数据库的排它锁和共享锁

首先说明：**数据库的增删改操作默认都会加排他锁，而查询不会加任何锁**。
注意：
1.**单纯的select语句不会加任何锁**。

2. **如果不通过索引条件检索数据，那么InnoDB将对表中所有数据加锁，实际效果跟表锁一样。**

|-- 共享锁（读锁）：自己和其他人都可以同时查询，这个锁的意义在于**加上共享锁查询的过程中，无法再加排它锁**，语法为：

select * from table lock in share mode

|-- 排他锁（写锁）：对某一资源加排他锁，自身可以进行增删改查，其他人无法进行任何操作。语法为：

select * from table for update   （For update是行级锁）
例1：

```sql
T1:select * from table lock in share mode（假设查询会花很长时间，下面的例子也都这么假设）
T2:update table set column1='hello'

过程：
T1运行（并加共享锁)
T2运行
If T1还没执行完
T2等......
else 锁被释放
T2执行
```



T2 之所以要等，是因为 T2 在执行 update 前，试图对 table 表加一个排他锁，而数据库规定同一资源上不能同时共存共享锁和排他锁。所以 T2 必须等 T1 执行完，释放了共享锁，才能加上排他锁，然后才能开始执行 update 语句。

 例2：

```sql
T1:select * from table lock in share mode
T2:select * from table lock in share mode

这里T2不用等待T1执行完，而是可以马上执行。
```

作者：冉椿林博客 
来源：CSDN 
原文：https://blog.csdn.net/localhost01/article/details/78720727 
版权声明：本文为博主原创文章，转载请附上博文链接！
+

#### 数据库的乐观锁和悲观锁（抽象的，不真实存在这两个锁）

使用悲观锁（其实说白了也就是排他锁）

|-- 程序A在查询库存数时使用排他锁（select * from table where id=10 for update）

|-- 然后进行后续的操作，包括更新库存数，最后提交事务。

|-- 程序B在查询库存数时，如果A还未释放排他锁，它将等待。

使用乐观锁（靠表设计和代码来实现）

|-- 一般是在该商品表添加version版本字段或者timestamp时间戳字段

|-- 程序A查询后，执行更新变成了：
    update table set num=num-1 where id=10 and version=23  

这样，保证了修改的数据是和它查询出来的数据是一致的，而其他执行程序未进行修改。当然，如果更新失败，表示在更新操作之前，有其他执行程序已经更新了该库存数，那么就可以尝试重试来保证更新成功。为了尽可能避免更新失败，可以合理调整重试次数（阿里巴巴开发手册规定重试次数不低于三次）。

------

作者：冉椿林博客 
来源：CSDN 
原文：https://blog.csdn.net/localhost01/article/details/78720727 
版权声明：本文为博主原创文章，转载请附上博文链接！



#### PreparedStatement可以防止sql注入的原因

sql注入的例子： where id = 03 or 1=1。
PreparedStatement中sql语句是预编译的，参数都先用？表示，之后只能设置？的值，但不能改变sql的结构，因此or 1=1在此便行不通。
