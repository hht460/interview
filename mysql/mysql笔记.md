# 一条SQL更新语句是如何执行的？

## 重要的日志模块：redo log

1. MySQL 里经常说到的 WAL 技术，WAL 的全称是 Write-Ahead Logging，它的关键点就是先写日志（redo log），再写磁盘（磁盘数据页），也就是先写粉板，等不忙的时候再写账本。
2. InnoDB 的 redo log 是固定大小的，写满后只能清除，再继续写入；redo log 写入是顺序写入，并且是一组一组写入，提高磁盘IO效率；
3. 有了 redo log，InnoDB 就可以保证即使数据库发生异常重启，之前提交的记录都不会丢失，这个能力称为**crash-safe**；

## 重要的日志模块：binlog

​    MySQL 整体来看，其实就有两块：一块是 Server 层，它主要做的是 MySQL 功能层面的事情；还有一块是引擎层，负责存储相关的具体事宜。上面 redo log 是 InnoDB 引擎特有的日志，而 Server 层也有自己的日志，称为 binlog（归档日志）；

## redo log和binlog比较

1. redo log 是 InnoDB 引擎特有的；binlog 是 MySQL 的 Server 层实现的，所有引擎都可以使用。
2. redo log 是**物理日志**，记录的是**“在某个数据页上做了什么修改”**；/只有mysql可以识别得语言
3. binlog 是**逻辑日志**，记录的是**这个语句的原始逻辑**，比如“给 ID=2 这一行的 c 字段加 1 ”。所有数据库都可以识别
4. redo log 是循环写的，空间固定会用完；
5. binlog 是可以追加写入的。“追加写”是指 binlog 文件写到一定大小后会切换到下一个，并不会覆盖以前的日志。

##  InnoDB 引擎更新语句流程

1. 执行器先找引擎取 ID=2 这一行。ID 是主键，引擎直接用树搜索找到这一行。如果 ID=2 这一行所在的数据页本来就在内存中，则从内存中取到该行直接返回给执行器；否则，需要先从磁盘将当前行所在数据页数据读入内存，取到该行然后再返回。
2. 执行器拿到引擎返回的行数据，把这个值加上 1，比如原来是 N，现在就是 N+1，得到新的一行数据，再调用引擎接口写入这行新数据。
3. 引擎将这行新数据更新到内存中（数据页也在内存中，此时为脏页/与磁盘中数据页相比），同时将这个更新操作记录到 redo log 里面，此时 redo log 处于 prepare 状态。然后告知执行器执行完成了，随时可以提交事务。
4. 执行器生成这个操作的 binlog，并把 binlog 写入磁盘。
5. 执行器调用引擎的提交事务接口，引擎把刚刚写入的 redo log 改成提交（commit）状态，更新完成。

![](D:\github\interview\mysql\image-md\update 语句执行流程.png)

## 两阶段提交

**怎样让数据库恢复到半个月内任意一秒的状态**

binlog 会记录所有的逻辑操作，并且是采用“追加写”的形式，通过binlog可以恢复；

**mysql异常重启（crash-safe）**

1. **先写 redo log 后写 binlog**。假设在 redo log 写完，binlog 还没有写完的时候，MySQL 进程异常重启。由于我们前面说过的，redo log 写完之后，系统即使崩溃，仍然能够把数据恢复回来，所以恢复后这一行 c 的值是 1。
   但是由于 binlog 没写完就 crash 了，这时候 binlog 里面就没有记录这个语句。因此，之后备份日志的时候，存起来的 binlog 里面就没有这条语句。
   然后你会发现，如果需要用这个 binlog 来恢复临时库的话，由于这个语句的 binlog 丢失，这个临时库就会少了这一次更新，恢复出来的这一行 c 的值就是 0，与原库的值不同。
2. **先写 binlog 后写 redo log**。如果在 binlog 写完之后 crash，由于 redo log 还没写，崩溃恢复以后这个事务无效，所以这一行 c 的值是 0。但是 binlog 里面已经记录了“把 c 从 0 改成 1”这个日志。所以，在之后用 binlog 来恢复的时候就多了一个事务出来，恢复出来的这一行 c 的值就是 1，与原库的值不同。

如果不使用“两阶段提交”，那么数据库的状态就有可能和用它的日志恢复出来的库的状态不一致。

1. 判断redo log是否完整（已提交），若是已提交，则直接用redo log恢复数据；
2. 若redo log 是预提交状态，此时需要判断binlog是否完整：
   - binglog完整（写入完成）：直接提交redo log更改为提交状态，再恢复数据；
   - 不完整（未写入完成）:回滚事务；

# 索引

## 索引覆盖

对于联合索引，当查询返回字段刚好属于联合索引字段，那么此时走普通索引检索记录，就不需要回表查主键索引了

（**覆盖索引是指，索引上的信息足够满足查询请求，不需要再回到主键索引上去取数据。**）

## 最左前缀原则

## 索引下推

# MYSQL锁

数据库锁设计初衷是为了处理并发问题

## 锁分类

全局锁、表锁、行锁

### 全局锁

对整个数据库实例加锁；mysql提供一个全局读锁方法，命令：flush tables with read lock（ftwrl）

### 表级锁

表级锁有两种：一种是表锁，一种是元数据锁（meta data lock, MDL）;

表锁：显示指定使用 lock table/unlock table

MDL锁：MDL 不需要显式使用，在访问一个表的时候会被**自动加上**；

1. 当对一个表做增删改查操作时，加MDL读锁；
2. 当对一个表结构做变更操作时，加MDL写锁；

- **读锁之间不互斥**，因此你可以有多个线程同时对一张表增删改查。

- **读写锁之间、写锁之间是互斥的**，用来保证变更表结构操作的安全性。因此，如果有两个线程要同时给一个表加字段，其中一个要等另一个执行完才能开始执行

  MDL锁获取时期：

  - 显式添加begin/start...开启事务 在sql之前，此时直接拿取mdl读/写锁
  - 未加显式开启事务，直接执行sql，在执行sql语句时获取mdl读/写锁

### 行锁

1. 两阶段锁协议：

   ​		在 InnoDB 事务中，行锁是在需要的时候才加上的，但并不是不需要了就立刻释放，而是要等到事务结束（commit）时才释放。这个就是两阶段锁协议；

2. 如果你的事务中需要锁多个行，要把最可能造成锁冲突、最可能影响并发度的锁尽量往后放。

3. 当出现死锁以后，有两种策略：

一种策略是，直接进入等待，直到超时。这个超时时间可以通过参数 innodb_lock_wait_timeout 来设置。

另一种策略是，发起死锁检测，发现死锁后，主动回滚死锁链条中的某一个事务，让其他事务得以继续执行。将参数 innodb_deadlock_detect 设置为 on，表示开启这个逻辑。

# 普通索引、唯一索引

## 查询过程

- 对于普通索引来说，查找到满足条件的第一个记录 (5,500) 后，需要查找下一个记录，直到碰到第一个不满足 k=5 条件的记录。
- 对于唯一索引来说，由于索引定义了唯一性，查找到第一个满足条件的记录后，就会停止继续检索。

​    注：innoDB 的数据是按数据页为单位来读写的，当需要读一条记录的时候，并不是将这个记录本身从磁盘读出来，而是以页为单位，将其整体读入内存。在 InnoDB 中，每个数据页的大小默认是 16KB。

## 更新过程

**涉及技术：change buffer**

​    注：当需要更新一个数据页时，如果数据页在内存中就直接更新，而如果这个数据页还没有在内存中的话，在不影响数据一致性的前提下，InooDB 会将这些更新操作缓存在 change buffer 中，这样就不需要从磁盘中读入这个数据页了（读入过程涉及磁盘随机IO访问）。在**下次查询**需要访问这个数据页的时候，将数据页读入内存，然后执行 change buffer 中与这个页有关的操作。通过这种方式就能保证这个数据逻辑的正确性。

**change buffer 中的操作应用到原数据页，得到最新结果的过程称为 merge；**

**merge触发条件：**

- 下次查询访问到这个数据页时，将该页读入内存进行merge
- 系统有后台线程会定期 merge
- 数据库正常关闭（shutdown）的过程中，也会执行 merge 操作

**什么条件下可以使用 change buffer 呢？**

​    唯一索引：不适用；更新操作都要先判断这个操作是否违反唯一性约束（*判断现在表中是否已经存在 k=4 的记录*），这必须要将数据页读入内存才能判断，数据已经在内存中，直接更新更快，没必要使用change buffer；

**innoDB引擎插入一条数据处理流程**（*eg: insert into (4,400)* ）

第一种情况是，**这个记录要更新的目标页在内存中**。这时，InnoDB 的处理流程如下：

- 对于唯一索引来说，找到 3 和 5 之间的位置，**判断到没有冲突**，插入这个值，语句执行结束；
- 对于普通索引来说，找到 3 和 5 之间的位置，插入这个值，语句执行结束。

第二种情况是，**这个记录要更新的目标页不在内存中**。这时，InnoDB 的处理流程如下：

- 对于唯一索引来说，需要将数据页读入内存，找到 3 和 5 之间的位置，**判断到没有冲突**，插入这个值，语句执行结束；

- 对于普通索引来说，则是将更新记录在 change buffer，语句执行就结束了。

  **注：将数据从磁盘读入内存涉及随机 IO 的访问，是数据库里面成本最高的操作之一。change buffer 因为减少了随机磁盘访问；**

**change buffer 使用场景**

​    因为 merge 的时候是真正进行数据更新的时刻，而 change buffer 的主要目的就是将记录的变更动作缓存下来，所以在一个数据页做 merge 之前，change buffer 记录的变更越多（也就是这个页面上要更新的次数越多），收益就越大;

对于**写多读少**的业务来说，页面在写完以后马上被访问到的概率比较小，此时 change buffer 的使用效果最好;

**索引选择和实践**

- 普通索引和唯一索引在查询能力没有差别，主要在更新操作，普通索引做了优化，建议考虑普通索引；
- 如果更新操作后马上伴随对这条记录得查询，此时应该关闭change buffer；其它情况下change buffer能起到提升性能作用；

**change buffer 和redo log**

在表上执行新增操作：*insert into t(id,k) values(id1,k1),(id2,k2);* *当前 k 索引树的状态，查找到位置后，k1 所在的数据页在内存 (InnoDB buffer pool) 中，k2 所在的数据页不在内存中*

![](D:\github\interview\mysql\image-md\带 change buffer 的更新过程.png)

这条更新语句做了如下的操作（按照图中的数字顺序）：

- Page 1 在内存中，直接更新内存；

- Page 2 没有在内存中，就在内存的 change buffer 区域，记录下“我要往 Page 2 插入一行”这个信息

- 将上述两个动作记入 redo log 中（图中 3 和 4）。

  注：从执行过程分析，执行这条更新语句的成本很低，就是写了两处内存（page1，change buffer），然后写了一处磁盘（两次操作合在一起写了一次磁盘/redo log），而且还是顺序写的。

接着执行查询操作：select * from t where k in (k1, k2)

![](D:\github\interview\mysql\image-md\带 change buffer 的读过程.png)

- 读 Page 1 的时候，直接从内存返回。

  *有几位同学在前面文章的评论中问到，WAL 之后如果读数据，是不是一定要读盘，是不是一定要从 redo log 里面把数据更新以后才可以返回？其实是不用的。你可以看一下图 3 的这个状态，虽然磁盘上还是之前的数据，但是这里直接从内存返回结果，结果是正确的。*

- 要读 Page 2 的时候，需要把 Page 2 从磁盘读入内存中，然后应用 change buffer 里面的操作日志，生成一个正确的版本并返回结果。

**redo log 和change buffer对比：**

redo log 主要节省的是随机写磁盘的 IO 消耗（转成合并顺序写）；

change buffer 主要节省的则是随机读磁盘的 IO 消耗（不立马读取磁盘数据，而是写入change buffer）。

# 字符串字段创建索引

## 创建方式

- 直接创建完整索引，这样可能比较占用索引空间；

- 创建前缀索引，节省索引空间，但会增加查询扫描次数，并且不能使用覆盖索引；

  前缀索引：截取字符串前几位创建索引；

  导致结果：由于索引区分度降低，检索时需要回表检索主键索引树，判断结果值是否相等，同时破坏覆盖索引功能；

  区分度检测方法：*select count(distinct email) as L from SUser;*  统计结果和总行数比较

- 倒序存储，将字符串倒序，截取前几位创建前缀索引，用于绕过字符串本身前缀的区分度不够的问题；

- 创建 hash 字段索引，查询性能稳定，有额外的存储和计算消耗，跟第三种方式一样，都不支持范围扫描。

**倒序存储和使用 hash 字段**比较：

相同点：都不支持范围查询；

不同点：

- 从占用的额外空间来看，倒序存储方式在主键索引上，不会消耗额外的存储空间，而 hash 字段方法需要增加一个字段。当然，倒序存储方式使用 倒序字符串前缀长度应该较长，这个消耗跟额外这个 hash 字段也差不多抵消了。
- 在 CPU 消耗方面，倒序方式每次写和读的时候，都需要额外调用一次 reverse 函数，而 hash 字段的方式需要额外调用一次 crc32() 函数（hash函数）。如果只从这两个函数的计算复杂度来看的话，reverse 函数额外消耗的 CPU 资源会更小些。
- 从查询效率上看，使用 hash 字段方式的查询性能相对更稳定一些。因为 crc32 算出来的值虽然有冲突的概率，但是概率非常小，可以认为每次查询的平均扫描行数接近 1。而倒序存储方式毕竟还是用的前缀索引的方式，也就是说还是会增加扫描行数。

# 数据库“抖”（数据刷新）

1. 内存里的数据写入磁盘的过程，术语就是 flush；
2. 当内存数据页跟磁盘数据页内容不一致的时候，我们称这个内存页为“脏页”。内存数据写入到磁盘后，内存和磁盘上的数据页的内容就一致了，称为“干净页”。
3. MySQL 偶尔“抖”一下的那个瞬间，可能就是在刷脏页（flush）；

## 触发mysql刷脏页动作：

1. InnoDB 的 redo log 写满了。这时候系统会停止所有更新操作，把 checkpoint 往前推进，redo log 留出空间可以继续写；
2. 系统内存不足。当需要新的内存页，而内存不够用的时候，就要淘汰一些数据页，空出内存给别的数据页使用。如果淘汰的是“脏页”，就要先将脏页写到磁盘；
3.  MySQL 认为系统“空闲”的时候。当然，MySQL“这家酒店”的生意好起来可是会很快就能把粉板记满的，所以“掌柜”要合理地安排时间，即使是“生意好”的时候，也要见缝插针地找时间，只要有机会就刷一点“脏页”；
4. MySQL 正常关闭的情况。这时候，MySQL 会把内存的脏页都 flush 到磁盘上，这样下次 MySQL 启动的时候，就可以直接从磁盘上读数据，启动速度会很快。

# 表数据删掉一半，表文件大小不变？

**表数据既可以存在共享表空间里，也可以是单独的文件。这个行为是由参数 innodb_file_per_table 控制**

1. 这个参数设置为 OFF 表示的是，表的数据放在系统共享表空间，也就是跟数据字典放在一起；
2. 这个参数设置为 ON 表示的是，每个 InnoDB 表数据存储在一个以 .ibd 为后缀的文件中（推荐）。

## 表数据删除

行删除，仅做个删除标记，后续可以继续复用；只限于符合范围条件的数据，插入或者更新，可以直接复用这个空间

页删除，整个数据页做个标记，后续继续复用；当整个页从 B+ 树里面摘掉以后，可以复用到任何位置

## 重建表

​    经过大量**增删改**的表，都是可能是存在空洞的。所以，如果能够把这些空洞去掉，就能达到收缩表空间的目的；

​    **重建表，就可以达到这样的目的**

​    **可以使用 alter table A engine=InnoDB 命令来重建表**；重建表流程和Mysql版本有关，5.5版本之前是非Online DDl；5.5版本之后是Online DDL；

### 非Online DDL

​    可以新建一个与表 A 结构相同的表 B，然后按照主键 ID 递增的顺序，把数据一行一行地从表 A 里读出来再插入到表 B 中。由于表 B 是新建的表，所以表 A 主键索引上的空洞，在表 B 中就都不存在了。显然地，表 B 的主键索引更紧凑，数据页的利用率也更高。如果我们把表 B 作为临时表，数据从表 A 导入表 B 的操作完成后，用表 B 替换 A，从效果上看，就起到了收缩表 A 空间的作用。

### Online DDL

1. 建立一个临时文件，扫描表 A 主键的所有数据页；
2. 用数据页中表 A 的记录生成 B+ 树，存储到临时文件中；
3. 生成临时文件的过程中，将所有对 A 的操作记录在一个日志文件（row log）中，对应的是图中 state2 的状态；
4. 临时文件生成后，将日志文件中的操作应用到临时文件，得到一个逻辑数据上与表 A 相同的数据文件，对应的就是图中 state3 的状态；
5. 用临时文件替换表 A 的数据文件。

# “order by”是怎么工作的

select *city, name, age* from t where city='杭州' order by name limit 1000  ;

sort_buffer ： MySQL 会给每个线程分配一块内存用于排序

## 全字段排序

1. 初始化 sort_buffer，确定放入 name、city、age 这三个字段；
2. 从索引 city 找到第一个满足 city='杭州’条件的主键 id，也就是图中的 ID_X；
3. 到主键 id 索引取出整行，取 name、city、age 三个字段的值，存入 sort_buffer 中；
4. 从索引 city 取下一个记录的主键 id；
5. 重复步骤 3、4 直到 city 的值不满足查询条件为止，对应的主键 id 也就是图中的 ID_Y；
6. 对 sort_buffer 中的数据按照字段 name 做快速排序；
7. 按照排序结果取前 1000 行返回给客户端。

​    但如果排序数据量太大，内存（sort_buffer）放不下，则不得不利用磁盘临时文件辅助排序；MySQL 将需要排序的数据分成 12 份，每一份单独排序后存在这些临时文件中。然后把这 12 个有序文件再合并成一个有序的大文件（外部排序一般使用归并排序算法）。

## rowid 排序

如果 MySQL 认为排序的单行（需要返回字段个数太多）长度太大，mysql会采取rowid排序

**新的算法放入 sort_buffer 的字段，只有要排序的列（即 name 字段）和主键 id**

1. 初始化 sort_buffer，确定放入两个字段，即 name 和 id；
2. 从索引 city 找到第一个满足 city='杭州’条件的主键 id，也就是图中的 ID_X；
3. 到主键 id 索引取出整行，取 name、id 这两个字段，存入 sort_buffer 中；
4. 从索引 city 取下一个记录的主键 id；
5. 重复步骤 3、4 直到不满足 city='杭州’条件为止，也就是图中的 ID_Y；
6. 对 sort_buffer 中的数据按照字段 name 进行排序；
7. 遍历排序结果，取前 1000 行，并按照 id 的值回到原表中取出 city、name 和 age 三个字段返回给客户端。

## 优化1（联合索引）

如果能够保证从 city 这个索引上取出来的行，天然就是按照 name 递增排序的话，是不是就可以不用再排序了；

创建一个 city 和 name 的联合索引；

![](D:\github\interview\mysql\image-md\city和name联合索引示意图.png)

1. 从索引 (city,name) 找到第一个满足 city='杭州’条件的主键 id；
2. 到主键 id 索引取出整行，取 name、city、age 三个字段的值，作为结果集的一部分直接返回；
3. 从索引 (city,name) 取下一个记录主键 id；
4. 重复步骤 2、3，直到查到第 1000 条记录，或者是不满足 city='杭州’条件时循环结束。

## 优化2（索引覆盖）

创建一个 city、name 和 age 的联合索引，刚好覆盖需要返回的所有字段；

1. 从索引 (city,name,age) 找到第一个满足 city='杭州’条件的记录，取出其中的 city、name 和 age 这三个字段的值，作为结果集的一部分直接返回；
2. 从索引 (city,name,age) 取下一个记录，同样取出这三个字段的值，作为结果集的一部分直接返回；
3. 重复执行步骤 2，直到查到第 1000 条记录，或者是不满足 city='杭州’条件时循环结束。

# SQL语句逻辑相同，性能却差异巨大

## 案例一（条件字段函数操作）

对字段做了函数计算，就用不上索引了
