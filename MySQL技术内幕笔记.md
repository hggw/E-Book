<center><font size="12">MySQL内幕笔记</font></center>

***

# 1、MySQL体系结构和存储引擎

## 1.1 数据库和数据库实例

- **数据库：**物理操作系统文件或其他形式文件类型的集合。在MySQL数据库中，数据库文件可以是frm、MYD、MYI、ibd结尾的文件。
- **数据库实例：**MySQL数据库由后台线程以及一个共享内存区组成。共享内存可以被运行的后台线程所共享。数据库实例才会真正操作数据库文件。

MySQL是一个**单进程多线程架构**的数据库，与Oracle多进程的架构不同，即MySQL数据库实例在系统上的表现就是一个进程。

当启动实例时，MySQL数据库会去读取配置文件，根据配置文件的参数来启动数据实例。在Linux环境下，配置文件一般放在/etc/my.cnf下。若存在多个配置文件中都有同一个参数，MySQL数据库会**以读取到的最后一个配置文件中的参数为准**。

***

## 1.2 MySQL体系结构

从概念上来说，**数据库是文件的集合，是依照某种数据模型组织起来并存放于耳机存储器的数据集合；数据库实例是程序，是位于用户于操作系统之间的一层数据管理软件**。

![](images/MySQL01.png)

从上图可以看出，MySQL由以下几部分组成：

- **Conncetion Pool(连接池组件)**

  管理、缓冲用户的连接，线程处理等需要缓存的需求

- **Management Services & Utlllties(管理服务和工具组件)**

  系统管理和控制工具，例如备份恢复、MySQL复制、集群等

- **SQL Interface(SQL接口组件)**

  接受用户的SQL命令，并且返回用户需要查询的结果

- **Parser(查询分析器组件)**

  对SQL命令进行验证和解析(权限、语法结构)

- **Optimizer(优化器组件)**

  对进行查询的SQL语句进行优化

- **Cathes & Buffers(缓冲组件)**

  查询缓存有命中的查询结果，查询语句就可以直接去查询缓存中取数据

- **Pluggable Storage Engines(插件式存储引擎)**

  负责如何管理数据

  需要特别注意：**存储引擎是基于表的，而不是数据库**(这一点可以从建表语句中看出)

- **物理文件(Files & Logs)**

***

## 1.3 MySQL存储引擎

由于MySQL数据库的开源特性，用户可以根据MySQL预定于的存储引擎接口编写自己的存储引擎。

### 1.3.1 InnoDB存储引擎(常用)

InnoDB支持事务，主要是面向**在线事务处理(OLTP，On-Line-Transaction-Process)**的应用，其特点是**行锁设计、支持外键**，并支持类似于Oracle的非锁定读，即**默认读取操作不会产生锁**。从4.1版本开始，InnoDB存储引擎会将数据单独存放到一个独立的**ibd文件**中。

InnoDB通过使用**多版本并发控制(MVCC)**来获得**高并发性**，并且实现了SQL标准的4种隔离级别，默认为**REPEATABLE**级别。同时使用一种叫做**next-key locking**的策略来避免幻读现象的产生。

InnoDB还提供了**插入缓冲(insert buffer)、二次写(double write)、自适应哈希索引(adaptive hash index)、预读(read ahead)**等高性能和高可用的功能。

对于表内数据的存储，InnoDB采用了**聚集(clustered)**的方式，因此**每张表的存储都是按主键的顺序进行存放**，若表定义是未显式指定主键，InnoDB会为每行生产一个**6字节的ROWID**，以此作为主键。

### 1.3.2 MyISAM存储引擎

MyISAM存储引擎不支持事务、表锁设计，支持全文索引，主要面向一些OLAP数据库应用。MyISAM存储引擎表有MYD和MYI组成，**MYD用来存放数据文件，MYI用于存放索引文件**。它的缓冲池只缓存索引文件(MYI)，而不缓存数据文件(MYD)。

需要注意，5.0版本之前，MyISAM默认支持的表大小为4GB，若大于4GB，则需要指定MAX_ROWS和AVG_ROW_LENGTH属性。5.0版本后，默认支持256TB的单表数据。

### 1.3.3 NDB存储引擎

NDB存储引擎是一个集群存储引擎，特点是将数据全部放在内存中，因此**主键查找(Primary Key lookups)的速度极快**，并且通过添加NDB数据存储节点(Data Node)可以线性提高数据库性能，是高可用、高性能的集群系统。

由于NDB存储引擎的**连接操作(JOIN)是在MySQL数据库层完成**的，而不是存储引擎层，这意味着，复杂的连接操作需要巨大的网络开销，因此**查询速度很慢**。

### 1.3.4 Memory存储引擎

Memory存储引擎将表中的数据存放在内存中，若数据库重启或发生崩溃，表中的数据都将消失，非常**适用于存储临时数据的临时表以及数据仓库中的纬度表**。

Memory存储引擎**默认使用哈希索引**，而不是B+树索引，**只支持表锁**，并发性能较差，并且不支持TEXT和BLOB列类型。而且，**存储变长字段(varchar)时是按照定长字段(char)进行的**。

MySQL数据库使用Memory存储引擎作为临时表来存放查询的中间结果集(intermediate result)。如果中间结果集大于Memory存储引擎表的容量设置，或中间结果含有TEXT或BLOB列类型字段，则MySQL会将其转换成MyISAM存放到磁盘中。

### 1.3.5 Maria存储引擎

Maria存储引擎可以看作是MyISAM的后续版本，**支持缓存数据和索引文件，应用了行锁设计，提供了MVCC功能，支持事务和非事务安全的选项，以及更好的BLOB字符类型的处理性能**。

***

## 1.4 各存储引擎之间的比较

***

## 1.5 连接MySQL

### 1.5.1 TCP/IP

MySQL数据库在任何平台下都支持TCP/IP连接方式，这也是网络中使用得最多的一种方式。这种方式在TCP/IP连接上建立一个基于网络的连接请求，一般情况下Client在一台服务器上，而MySQL实例(Server)在另一台服务器上，这两台机器通过一个TCP/IP网络连接。

这里需要注意，通过TCP/IP连接到MySQL实例时，MySQL数据库会先检查一张**权限视图**(在mysql架构下，表名为user)，用于判断发起请求的客户端IP是否允许连接到MySQL实例。

### 1.5.2 命名管道和共享内存

若两个需要进行通信的进程在同一台服务器上，可以使用命名管道。MySQL4.1之后，提供了共享内存的连接方式。需要在配置文件中添加**--shared-memory**实现。

### 1.5.3 UNIX域套接字(这个套接字不太懂啥意思)

当MySQL客户端和数据库实例在一台服务器上时，用于可以在配置文件中指定套接字文件的路径，如socket=/tmp/mysql.sock。

***

# 2、InnoDB存储引擎

## 2.1 InnoDB体系架构

![](images/MySQL02.png)

InnoDB存储引擎有多个内存块，这些内存卡组成了一个大的内存池，主要负责以下工作：

- 维护所有进程/线程需要访问的多个内部数据结构
- 缓存磁盘上的数据，方便快速读取，同时对磁盘数据进行修改之前在此缓存
- 重做日志(redo log)缓冲

后台线程主要负责刷新内存池中的数据，保证缓冲池中的内存缓存的是最近的数据，以及将修改后的数据文件刷新到磁盘，同时保证数据库发生异常时I你弄DB能回复到正常运行状态。

### 2.1.1 后台线程

InnoDB存储引擎采用的是多线程的模型，后台有多个不同的后台线程，负责处理不同的任务。

**1）Master Thread**

核心的后台线程，主要负责**将缓冲池中的数据异步刷新到磁盘**，保证数据的一致性，包括**脏页的刷新、合并插入缓冲(INSERT BUUFFER)、UNDO页的回收**等。

**2）IO Thread**

为了提高数据库的性能，InnoDB大量使用了AIO(Async IO)来处理写IO请求，而IO Thread主要负责这些**IO请求的回调(call back)处理**。InnoDB共有4种IO Thread，分别为**write、read、insert buffer和log**。从1.0.x版本开始，**read thread和write thread均增大到了4个**，分别使用**innodb_read_io_thread和innodb_write_io_thread**参数进行设置。

**3）Purge Thread**

事务被提交后，其所使用的undolog可能不再需要，Purge Thread主要用于**回收已经使用并分配的undo页**。用户可以在配置文件中设置启用独立的Purge Thread：

- innodb_purge_thread=1

从1.2版本之后，InnoDB支持多个Purge Thread，可以进一步加快undo页的回收。由于Purge Thread需要离散读取undo页，多个Purge Thread能更进一步利用磁盘的随机读取性能。

**4）Page Cleaner Thread**

负责**脏页的刷新**操作，原先由Master Thread处理，为了减轻原Master Thread的工作级用户查询线程的阻塞，将其放入到单独的线程中来处理。

### 2.1.2 内存

**1）缓冲池**

InnoDB存储引擎是基于磁盘存储的，并将其中的记录**按照页的方式**进行管理。缓冲池其实就是通过一块内存区域来弥补磁盘速度较慢对数据库性能的影响。

数据库读取页时，首先将从磁盘读到的页存放在缓冲池中，下次再读相同页时候，先判断是否再缓冲池中，如在，则直接读取该页(也称该页被命中)，否则从磁盘读取。

对数据进行修改时，首先修改在缓冲池中的页，然后以一定的频率刷新到磁盘上。这里需要注意，**页从缓冲池刷新回磁盘的操作并不是在每次页数据发生更新时触发**，而是通过**CheckPoint机制**刷新回磁盘。

缓冲池中缓存的数据页类型主要有：**索引页、数据页、undo页、插入缓冲(insert buffer)、自适应哈希索引、锁信息(lock info)、数据字典信息(data dictionary)**等。

![](images/MySQL03.png)

InnoDB的缓冲池配置可以通过**innodb_buffer_pool_size**进行设置。从1.0.x版本开始，允许有多个缓冲池实例。每个页根据哈希值平均分配到不同缓冲池实例中，减少数据库内部的资源竞争，提高并发性能。可以通过**innodb_buffer_pool_instances**进行设置。

**2）LRU List、Free List和Flush List**

缓冲池是通过**LRU(Lastest Recent Used)**算法进行管理，即使用**最频繁的页在LRU列表的前端，最少使用的在LRU列表的尾端**。当缓冲池无法存放新读取到的页时，将**首先释放LRU列表中尾端的页**。

InnoDB缓冲池页的大小默认是16KB。InnoDB对传统LRU做了一些优化，在LRU列表中加入了**midpoint**位置。**新读取的页不直接放入LRU列表的首部，而是放入LRU列表的midpoint位置**，称为**midpoint insertion strategy**。该位置之前的列表成为new列表，之后的成为old列表。

默认配置下，**midpoint在LRU列表长度的5/8处**，由**innodb_old_blocks_pct**控制(以距离尾部为基准，采用百分比)。

这里稍微解释以下为什么不直接放入LRU列表首部？

> 若直接放入首部，某些SQL操作可能会使缓冲池中的页被刷新出，从而影响缓冲池效率。
>
> 常见的操作由索引或数据的扫描操作。这类操作需要访问表中的许多页甚至全部页，而这些页只是在某次查询操作中需要，如放入首部，很可能会将热点数据页从LRU列表一处。

总的来说，一切都是为了缓冲池中数据页的命中率。

引入midpoint的同时也带来了一个问题，需要多久midpoint可以加入LRU列表的首部(热端)？

> InnoDB引入参数innodb_old_blocks_time，用于表示页读取到mid位置后等待多久才会被加入LRU列表的热端。

当页从LRU列表的old部分加入new部分时，称为**pages made young**，而因innodb_old_blocks_time设置导致页没有从old移动到new部分，称为**pages not made young**。

InnoDB从1.0.x版本开始支持压缩页功能，即将16KB的页压缩为1KB、2KB、4KB和8KB。对于非16KB的页，是通过**unzip_LRU**列表进行管理。

对于压缩页的表，压缩比率可能各不相同。首先unzip_LRU列表对不同压缩页大小的页进行分别管理。再通过伙伴算法进行内存的分配。以从缓冲池申请页为4KB的大小为例：

> 1.检查4KB的unzip_LRU列表，检查有无可用的空闲页，若有，直接使用，否则检查8KB的unzip_LRU列表；
>
> 2.若能够得到8KB空闲页，将页分成2个4KB页，存放到4KB的unzip_LRU列表，否则，从LRU中申请一个16KB的页，分为8KB+4KB+4KB，分别存入对于的unzip_LRU列表中

当LRU列表中的页被修改后，称为**脏页(dirty page)**，即缓冲池中的页和磁盘上的页不一致。此时数据库通过CheckPoint机制将脏页刷新会磁盘，而**Flush列表中的页即为脏页列表**。**LRU列表用来管理缓冲池中页的可用性，Flush列表用来管理将页刷回磁盘**，脏页既存于LRU，也存于Flush，二者互不影响。

### 2.1.3 重做日志缓冲(redo log buffer)

InnoDB存储引擎**首先将重做日志信息先放到这个缓冲区，然后按一定频率将其刷新到重做日志文件**。redo log buffer一般不需要设置得很大，一般情况下每间隔1秒会将其刷新到日志文件，故只需保证每秒发生的事务量在这个缓冲大小之内。可由配置参数innodb_log_buffer_size控制，默认为8MB。

redo log buffer的内容刷新到外部磁盘的重做日志文件，通常有以下三种情况：

- **Master Thread每间隔一秒将重做日志缓冲刷新到重做日志文件；**
- **每个事务提交时会刷新到重做日志文件；**
- **当重做日志缓冲池剩余空间小于1/2，刷新到重做日志文件**

### 2.1.4 额外的内存池

InnoDB存储引擎中，内存管理是通过内存堆的方式进行的，对一些本身位于内存的数据进行分配时，需要从**额外的内存池**中进行申请，当该区域内存不够时，会从缓冲池中进行申请。

***

## 2.2 CheckPoint技术

上面提到了当缓冲池中的数据页发生变化，会出现缓存数据与磁盘数据不一致的情况，即脏页。若刷新到磁盘的过程中发生故障，则修改过的数据便无法恢复。针对这类问题，当前事务数据库系统普遍采用**Write Ahead Log策略**，即**当事务提交时，先写重做日志，再修改页**。当发生数据丢失时，通过重做日志来完成数据的恢复。

**CheckPoint(检查点)**技术的目的就是解决以下几个问题：

- **缩短数据库的恢复时间；**
- **缓冲池内存不足时，将脏页刷新到磁盘；**
- **重做日志不可用时，刷新脏页。**

当数据库发生宕机时，数据库不需要重做所有的日志，只需**对CheckPoint后的重做日志进行恢复**，因为之前的页都已经刷回磁盘。当缓冲池内存不足时，根据LRU算法移除的数据页，若为脏页，则需**强制执行CheckPoint**，将脏页刷回磁盘。

InnoDB是通过**LSN(Log Sequence Number)**来标记版本的。LSN是8字节的数字，每个页、重做日志和CheckPoint都带有LSN。

InnoDB内部有两种CheckPoint，分别为**Sharp CheckPoint和Fuzzy CheckPoint**。

**1）Sharp CheckPoint**

发生在数据库关闭时，将所有的脏页刷新回磁盘，这是默认的工作方式，参数为innodb_fast_shutdown=1。

**2）Fuzzy CheckPoint**

Fuzzy CheckPoint与Sharp CheckPoint的区别在于**只刷新一部分脏页，而不是刷新所有的脏页回磁盘**。

InnoDB中大致有以下几种Fuzzy CheckPoint:

- **Master Thread CheckPoint**

  每秒或每十秒从缓冲池的脏页列表中刷新一定比例的数据页回磁盘，同时InnoDB可以进行其他操作，用户查询线程不会阻塞。

- **FLUSH_LRU_LIST CheckPoint**

  当LRU列表中剩余空闲页不足时，InnoDB需将LRU列表尾端的页移除，若这些页中存在脏页，则需要进行CheckPoint，由于这些页来自LRU列表，故称为FLUSH_LRU_LIST CheckPoint。这个检查被放在一个单独线程Page Cleaner线程中(上一章节有提及)。

- **Async/Sync Flush CheckPoint**

  当重做日志文件不可用时，需要强制将一些页刷新回磁盘，此时脏页是从脏页列表中选取的。若将已经写入到重做日志的LSN记为**redo_lsn**，将已经刷新回磁盘最新页的LSN记为**checkpoint_lsn**，则：

  > checkpoint_age = redo_lsn - checkpoint_lsn

  定义以下变量：

  > async_water_mark = 75% * total_redo_log_file_size
  >
  > sync_water_mark = 90% * total_redo_log_file_size

  假定有两个重做日志文件，每个文件大小为1GB。则：

  - 当checkpoint_age < async_water_mark时，不需要刷新任何脏页到磁盘；
  - 当async_water_mark < checkpoint_age < sync_water_mark 时触发Async Flush，从Flush列表中刷新足够的脏页回磁盘，使刷新后满足checkpoint_age < async_water_mark；
  - 当checkpoint_age > async_water_mark时，一般很少发生，一般是因为设置的重做日志文件太小，并且在进行类似LOAD DATA 的BULK INSERT操作。此时触发Sync Flush操作，从Flush列表中刷新足够的脏页回磁盘，使刷新后满足checkpoint_age < async_water_mark；

- **Dirty Page too much CheckPoint**

  当脏页数量过多时，导致InnoDB强制进行CheckPoint，主要是为了保证缓冲池中有足够的空闲页，由innodb_max_dirty_pages_pct控制，默认配置为75。

***

## 2.3 Master  Thread工作方式

Master Thread具有最高的线程优先级别，其内部由多个循环(loop)组成：**主循环(Loop)、后台循环(Backgroup Loop)、刷新循环(Flush Loop)、暂停循环(Suspend Loop)**。

**1）主循环(Loop)**

主循环通过**Thread Sleep**来实现，这意味这每秒一次或每十秒一次的操作不是严格精确的。

**每秒一次**的操作包括：

- **日志缓冲刷新到磁盘，即使这个事务未提交(总是)**

- **合并插入缓冲(可能)**

  判断当前一秒内发生的IO次数是否小于5此，若小于，则执行合并插入缓冲的操作

- **至多刷新100个InnoDB的缓冲池中的脏页到磁盘(可能)**

  判断当前缓冲池中脏页的比例(buf_get_modified_ratio_pct)是否超过配置文件中innodb_max_dirty_pages_pct(默认为90，代表90%)，若超过这个阈值，则需要做磁盘同步操作。

- **若当前没有用户活动，则切换到Backgroup Loop(可能)**

**每十秒一次**的操作包括：

- **刷新100个脏页到磁盘(可能)**
- **合并之多5个插入缓冲(总是)**
- **将日志缓冲刷新到磁盘(总是)**
- **删除无用的Undo页(总是)**
- **刷新100个或者10个脏页到磁盘(总是)**

> InnoDB会判断过去10秒内磁盘的IO操作是否小于200次，若小于，则任务当前磁盘IO操作能力足够，将100个脏页刷新到磁盘。
>
> 接着，InnoDB会合并插入缓冲，不同于每秒一次的操作，这里的合并插入缓冲操作总会在这个阶段进行。
>
> 之后会再进行一次将日志缓冲刷到磁盘的操作。
>
> 进一步，InnoDB会执行**full purge操作**，即删除无用的Undo页。对表进行update、delete这类操作时，原先的行被标记为删除，出于一致性读的原则，需保留这些信息。但在full purge过程中，InnoDB会判断当前事务系统中已被删除的行是否可以删除。源代码中，在执行full purge操作时，每次**最多尝试回收20个undo页**。
>
> 然后判断缓冲池中脏页的比例(buf_get_modified_ratio_pct)，若超过70%的脏页，则刷新100个脏页到磁盘，若小于，只需刷新10%的脏页到磁盘。

**2）后台循环(Backgroup Loop)**

- **删除无用的Undo页(总是)**
- **合并20个插入缓冲(总是)**
- **跳回到主循环(总是)**
- **不断刷新100个页知道符合条件(可能跳转到flush loop中完成)**

Master Thread完整的伪代码如下：

```c
void master_thread() {
    goto loop;
loop:
    for(int i = 0; i < 10; i++) {
        thread_sleep(1);
        do log buffer flush to disk;
        if(last_one_second_ios < 5% innodb_io_capacity) {
            do merge 5% innodb_io_capacit insert buffer;
        }
        if(buf_get_modified_ratio_pct > innodb_max_dirty_pages_pct) {
            do buffer pool flush 100% innodb_io_capacity dirty page;
        }
        else if enable adaptive flush:
        	do buffer pool flush desired amount dirty page;
        if(no user activity) {
            goto backgroup loop;
        }
    }
    if(last_ten_second_ios < innodb_io_capacity) {
        do buffer pool flush 100% innodb_io_capacity dirty page;
    }
    do merge 5% innodb_io_capacit insert buffer;
    do log buffer flush to disk;
    do full purge;
    if(buf_get_modified_ratio_pct > 70%) {
        do buffer pool flush 100% innodb_io_capacity dirty page;
    } else {
        do buffer pool flush 10% innodb_io_capacity dirty page;
    }
    goto loop;
backgroup loop:
    do full purge;
    do merge 100% innodb_io_capacity insert buffer;
    if not idle:
    	goto loop;
    else:
    	goto flush loop;
flush loop:
    do buffer pool flush 100% innodb_io_capacity dirty page;
    if(buf_get_modified_ratio_pct > innodb_max_dirty_pages_pct) {
        goto flush loop;
    	goto suspend loop;
	}
suspend loop:
    suspend_thread();
    waiting event;
    goto loop;
}
```

***

## 2.4 InnoDB关键特性

### 2.4.1 插入缓冲

**1）Insert Buffer**

InnoDB存储引擎中，主键是行唯一的标识符，通常应用程序中行记录的插入是按照主键递增的顺序进行插入的。因此，插入聚集索引(Primary Key)一般是顺序的，不需要磁盘的随机读取。但不可能每张表上只有一个**聚集索引**，一张表上通常由多个非聚集的**辅助索引(secondary index)**。

对于非聚集索引叶子节点的插入不再是顺序的，而需要**离散地访问非聚集索引页**，即随机读取，导致插入操作性能下降，这是B+树的特性决定的。

InnoDB存储引擎开创性地设计了Insert Buffer，对于非聚集索引的插入或更新操作，先**判断插入的非聚集索引页是否在缓冲池中**，若在，则直接插入；若不在，则先放入一个Insert Buffer对象中。然后再**以一定的频率和情况进行Insert Buffer和辅助索引页子节点的merge操作**，通常能将多个插入合并到一个操作中(在一个索引页中)，大大提高对于非聚集索引插入的性能。

使用Insert Buffer需同时满足以下两个条件：**辅助索引且不是唯一的**。当应用程序进行大量插入操作时，若此时MySQL数据库发生宕机，势必由大量的Insert Buffer并没有合并到实际的非聚集索引中去，这时恢复可能需要很长的时间。

目前Insert Buffer在写密集的情况下，插入缓冲会占用过多的缓冲池内存(innodb_buffer_pool)，默认最大可以占用到**1/2的缓冲池内存**。可以通过修改IBUF_POOL_SIZE_PER_MAX_SIZE对插入缓冲进行控制。

**2）Change Buffer**

InnoDB对DML操作(**INSERT、DELETE、UPDATE**)进行缓冲，分为为**Insert Buffer、Delete Buffer、Purge Buffer**。

Change Buffer适用的对象依然是非唯一的辅助索引，对一条数据进行UPDATE操作可能分为两个过程：

- **将记录标记为已删除**
- **真正将记录删除**

Delete Buffer对应UPDATE操作的第一个过程，Purge Buffer对应UPDATE操作的第二个过程。InnoDB提供了参数**innodb_change_buffering**，用于开启各种Buffer选项。可选参数值为：**inserts、deletes、purges、changes、all、none**。purges表示启用insert、delete、purge，changes表示启用insert和delete，all表示启用所有，none表示都不启用。默认值为all。

Change Buffer最大使用内存的数量可以通过innodb_change_buffer_max_size来控制。默认值为25，表示最多使用1/4的缓冲池空间。

**3）Insert Buffer的内部实现**（有点难，暂缓**）P64

Insert Buffer的数据结构是一颗B+树，**全局只有一棵Insert Buffer B+树**，负责对**所有表的辅助索引**进行Insert Buffer，存放在**共享表空间**中，默认也是就是**ibdata1**中。因此，试图通过独立表空间idb文件恢复表中数据时，往往会导致CHECK TABLE失败，因为辅助索引数据可能还在Insert Buffer中，故还需要REPAIR TABLE操作重建表上所有的辅助索引。

Insert Buffer B+树也由叶节点和非叶节点组成。非叶节点存放的是查询的**search key(键值)**，共占用**9个字节**，由**space(4字节)、marker(1字节)和offset(4字节)**组成。其中space表示待插入记录所在表的表空间id，在InnoDB中每张表都有一个唯一的space id，通过space id可以查询得知是哪张表。marker是用于兼容老版本的Insert Buffer，offset表示页所在的偏移量。

<img src="images/MySQL04.png" style="zoom: 67%;" />

当一个辅助索引要插入到页(space，offset)时，如该页不在缓冲池中，则InnoDB**首先根据上述规则构造一个search key，再查询Insert Buffer B+树，然后将这条记录插入到Insert Buffer B+树的叶子节点中**。

对Insert Buffer B+树插入叶子节点，并不是直接插入，而是需要根据一定规则进行构造：

<img src="images/MySQL05.png" style="zoom: 50%;" />

叶子节点中额外有一个**metadata**字段，存储内容如下：

|         名称          | 字节 |
| :-------------------: | :--: |
| IBUF_REC_OFFSET_COUNT |  2   |
| IBUF_REC_OFFSET_TYPE  |  1   |
| IBUF_REC_OFFSET_FLAGS |  1   |

IBUF_REC_OFFSET_COUNT用于排序每个记录进入Insert Buffer的顺序。

为保证每次Merge Insert Buffer页成功，采用 **Insert Buffer Bitmap**用于标记**每一个辅助索引页的可用空间**。结构如下：

|         名称         | 大小 | 说明                                                         |
| :------------------: | :--: | :----------------------------------------------------------- |
|   IBUF_BITMAP_FREE   |  2   | 表示该辅助索引页中的可用空间数量，可取值为：<br />0：无可用剩余空间<br />1：剩余空间大于1/32页<br />2：剩余空间大于1/16页<br />3：剩余空间大于1/8页 |
| IBUF_BITMAP_BUFFERED |  1   | 1表示该辅助索引页有记录被缓存在Insert Buffer B+树中          |
|   IBUF_BITMAP_IBUF   |  1   | 1表示该页为Insert Buffer B+树的索引页                        |

**4）Merge Insert Buffer**

当需要插入的辅助索引页不在缓冲池中，则需要先插入到Insert Buffer B+树中，后续还需要**合并**到真正的辅助索引中。Merge Insert Buffer主要在以下几种情况发生：

- **辅助索引页被读取到缓冲池**

  检查Insert Buffer Bitmap页，确认该辅助索引页是否有记录存放于Insert Buffer B+树中。

- **Insert Buffer Bitmap页追踪到该辅助索引页无可用空间**

  若插入辅助索引记录时检测到插入后，可用空间会小于1/32页，则强制进行一次合并操作。

- **Master Thread**

  Master Thread线程中每秒或每十秒会进行一次Merge Insert Buffer操作，区别在于每次merge的页数量不同

### 2.4.2 两次写

当数据库宕机，InnoDB存储引擎可能正在写入某个页到表中，而这个页可能只写了一部分，造成**部分写失效(partial page write)**。如果发生写失效，那可以通过重做日志恢复。但如果这个页本身发生损坏，那进行重做显然没有意义。

在应用(apply)重做日志前，需要一个页的副本，发生写入失效时，**先通过页的副本来还原该页，再进行重做**，称为**doublewrite**。

doublewrite由两部分组成，一部分是内存中的**doublewrite buffer**(大小为2MB)，另一部分是**物理磁盘上共享表空间中连续的128个页**(2个区，大小为2MB)。对缓冲池脏页进行刷新时，不直接写入磁盘，而是**先通过memcpy函数将脏页复制到内存的doublewrite buffer**，再通过doublewrite buffer分别**每次1MB顺序地写入共享表空间的物理磁盘**。然后调用fsync函数，同步磁盘，避免缓冲写带来的问题。由于页是连续的，并且是顺序写入，因此开销不大。

### 2.4.3 自适应哈希索引

InnoDB存储引擎会监控对表示各索引页的查询，会自动根据访问的频率和模式来自动地为某些热点页建立哈希索引。若观察到建立哈希索引可以带来速度提升，则会建立哈希索引，这就是**自适应哈希索引(Adaptive Hash Index，AHI)**。AHI通过缓冲池的B+树页构造，建立速度快，且无需对整张表构建哈希索引。

AHI的建立要求：

- **对同一个页的连续访问模式必须是一致的**
- **以该模式访问了100次**
- **页通过该模式访问了N次，其中N=页种记录*1/16**

需要注意，**哈希索引只能用来搜索等值的查询**，对于其他查找类型，是无法使用的。

### 2.4.4 异步IO(Asynchronous IO，AIO)

当用户发出一条索引扫描的查询，这条SQL语句可能需要扫描多个索引页，即需要进行多次IO操作。若采用Sync IO，则每扫描一个页并等待其完成后再进行下一次扫描，很显然是没必要的。用户可以**发出一个IO请求后立即发出下一个IO请求，直至发送完全部请求，等待IO操作的完成，这就是AIO**。同时AIO可以进行IO Merge操作，将多个IO合并为一个，可以提供IOPS性能。

### 2.4.5 刷新邻接页(Flush Neighbor Page)

当刷新一个脏页时，InnoDB会检测该页所在区的所有页，如果是脏页，则一起进行刷新。这样可以通过AIO将多个IO写入操作合并为一个IO操作。这个工作机制在传统机械键盘下有着显著的优势，对于具有超高IOPS性能的SSD磁盘，建议关闭此特性。

## 2.5 启动、关闭与恢复

在数据库关闭时，InnoDB的操作涉及参数innodb_fast_shutdown，可取值为0，1，2，默认为1。

- 0表示关闭数据库时，InnoDB需要完成所有的full puege和merge insert buffer，并将所有脏页刷新回磁盘；
- 1为默认值，不需要完成上述操作，但在缓冲池中的一些数据脏页还是会刷新回磁盘；
- 2表示不完成full puege和merge insert buffer操作，也不讲缓冲池中的数据脏页写回磁盘，儿是将日志都写入日志文件。

InnoDB存储引擎恢复涉及参数innodb_force_recovery，默认取0，表示当需要恢复时，进行所有的恢复操作。

- 1(SRV_FORCE_IGNORE_CORRUPT)：忽略检查到的corrupt页
- 2(SRV_FORCE_NO_BACKGROUD)：阻止Master Thread线程的运行
- 3(SRV_FORCE_NO_TRX_UNDO)：不进行事务的回滚操作
- 4(SRV_FORCE_NO_IBUF_MERGE)：不进行插入缓冲的合并操作
- 5(SRV_FORCE_NO_UNDO_LOG_SCAN)：不查看撤销日志(Undo Log)，InnoDB存储引擎会将未提交的事务视为已提交
- 6(SRV_FORCE_NO_LOG_REDO)：不进行前滚操作

***

# 3、文件

## 3.1 配置参数文件

理论上，MySQL实例可以不需要配置参数文件，此时所有参数值取决于编译MySQL时指定的默认值和源代码中指定参数的默认值。

### 3.1.1 参数类型

MySQL数据库的参数分为**动态参数**和**静态参数**两种。动态参数在MySQL实例**运行过程中允许修改**，可以通过**SET命令**进行修改，但静态参数在实例的**整个生命周期内都不得进行更改**。

SET语法如下：

```mysql
SET [global | session] system_var_name = expr
```

**global和session关键字**表明参数的修改是基于当前会话还是整个实例的生命周期。

需要注意，对变量的全局值进行修改，MySQL实例本身不会对配置参数文件中的值进行修改，即下次启动MySQL实例还是会读取原参数。

***

## 3.2 日志文件

MySQL常见的日志文件有：**错误日志(error log)**、**二进制日志(binary log)**、**慢查询日志(slow query log)**和**查询日志(log)**。

### 3.2.1 错误日志

错误日志对MySQL的启动、运行和关闭过程进行了记录，不仅记录了所有的错误信息，也记录一些警告信息或正确的信息。

```mysql
SHOW VARIABLES LIKE 'error_log';
```

### 3.2.2 慢查询日志

慢查询日志可以帮助定位可能存在问题的SQL语句，从而进行SQL优化。通过配置**long_query_time**参数设置一个阈值，将运行时间**超过**该值的所有SQL都记录到满查询日志文件中，默认是10s。这里需要注意的是，运行时间正好等于这个阈值的情况不会被记录下。

```mysql
SHOW VARIABLES LIKE 'long_query_time';
SHOW VARIABLES LIKE 'log_slow_queries';
```

还有一个与慢查询日志有关的参数log_queries_not_using_indexes，若运行的SQL没有使用索引，则MySQL会将其记录值慢查询日志。

```mysql
SHOW VARIABLES LIKE 'log_queries_not_using_indexes';
```

5.6.5版本开始新增了一个参数**log_throttle_queries_not_using_indexes**，用于表示每分钟允许记录到slow log的且未使用索引的SQL语句次数，默认为0，表示没有限制。

**MySQL5.1**开始将慢查询的日志记录放入**mysql架构**下的**slow_log**中，这使得用户的查询更加方便和直观。

### 3.2.3 查询日志

查询日志记录了所有对MySQL数据库请求的信息，无论请求是否正确执行，默认文件名为主机名.log。**MySQL5.1**开始将查询日志记录放入**mysql架构**下的**general_log**中。

### 3.2.4 二进制日志

二进制日志记录了**对MySQL数据库执行更改的所有操作**，但**不包括SELECT和SHOW操作**。主要作用如下：

- **恢复(recovery)**：某些数据的恢复需要二进制日志，例如通过二进制日志进行point-in-time的恢复
- **复制(replication)**：通过复制和执行二进制日志使远程的MySQL的slave数据库与master数据库进行实时同步。
- **审计(audit)**：用户通过二进制日志中的信息进行审计，判断是否有对数据库进行注入的攻击。

***

## 3.3 pid文件

MySQL实例启动时会将进程ID写入pid文件，该文件由参数pid_file控制，文件名为主机名.pid。

***

## 3.4 表结构定义文件

MySQL数据的存储是**以表为单位**的，物理采用何种存储引擎，都有一个**frm为后缀名**的文件，记录该表的表结构定义，**frm还用于存放视图的定义**。

***

## 3.5 InnoDB存储引擎文件

### 3.5.1 表空间文件

InnoDB中存储的数据是按照**表空间(tablespace)**进行存放的。默认配置下会有一个初始大小为10MB，名为ibdata1的文件，

### 3.5.2 重做日志文件

默认在InnoDB存储引擎的数据目录下会有两个名为**ib_logfile0**和**ib_logfile1**的文件，这就是**重做日志文件(redo log file)**。

每个InnoDB存储引擎至少有1个**重做日志文件组**，每个组下至少有两个重做日志文件。为了更高的可靠性，用户可以设置多个**镜像日志组(mirrored log groups)**放在不同的磁盘上，以此提高重做日志的高可用性。在日志组中，每个重做日志文件的大小一致，并以循环写入的方式运行。InnoDB先写重做日志文件1，当写到文件的最后时，会切换至重做日志文件2，当文件2也被写满时，再切换到文件1中。

**二进制日志与重做日志文件对比：**

> 1、二进制日志记录索引与MySQL数据库有关的日志，而InnoDB存储引擎的重做日志只记录有关该存储引擎本身的**事务日志**；
>
> 2、无论二进制日志文件格式是什么，记录的都是关于一个**事务的具体操作内容**，即逻辑日志，而InnoDB的重做日志记录的是关于**每个页更改的物理情况**；
>
> 3、二进制日志**仅在事务提交前进行提交**，即只写磁盘一次，而事务进行的过程中，会不断有重做日志条目被写入重做日志文件中。

写入重做日志的操作并不是直接写入，而是先写入一个**重做日志缓冲(redo log buffer)**，然后按照**一定顺序**写入日志文件。

<img src="images/MySQL06.png" style="zoom:67%;" />

从重做日志缓冲写入磁盘时，按照一个扇区512字节的大小进行写入。由于主线程(master thread)每秒会将重做日志缓冲写入磁盘的重做日志文件中，无论事务是否提交。

另一个触发写磁盘的过程是由参数**innodb_flush_log_at_trx_commit**控制，0表示当提交事务时，不会同步将事务的重做日志写入磁盘，而是等待主线程每秒的刷新；1表示在提交时同步写入磁盘，2表示异步写入磁盘。

***

# 4、表

## 4.1 索引组织表

InnoDB存储引擎中的表是根据主键顺序组织存放的，这种存储方式的表称为**索引组织表**。

采用InnoDB存储引擎的表中，每张表都有个**主键(Primary Key)**，若建表时未显式定义主键，则：

- 首先判断表中是否由**非空的唯一索引(Unique NOT NULL)**，若有，则为主键，否则自动创建一个6字节大小的指针

当表中有多个非空唯一索引时，将会选择建表时**第一个定义的非空唯一索引**为主键，这里根据的式**定义索引的顺序**。

***

## 4.2 InnoDB逻辑存储结构

InnoDB将所有数据逻辑存放在一个**表空间**中，表空间由**段(segment)、区(extent)、页(page)**组成。

![](images/MySQL07.png)

### 4.2.1 表空间

在第3章中已经提到，在默认情况下，InnoDB存储引擎有一个共享表空间ibdata1，即**所有数据都存放在这个表空间内**。

若启用了innodb_file_per_table，则每张表的数据单独放到一个表空间内，其中存放的只是**数据、所有和插入缓冲Bitmap页**。其他类的数据，如**回滚(undo)信息、插入缓冲索引页、系统事务信息、二次写缓冲(Double write buffer)等还是存放在原共享表空间内**。

### 4.2.2 段

数据段为索引B+树的叶子节点(**Leaf node segment**)，索引段为B+树的非索引节点(**Non-leaf node segment**)。

### 4.2.3 区

区是连续页组成的空间，每个区的大小为1MB，InnoDB一次向磁盘申请4~5个区，默认InnoDB页的大小为16KB，即默认**一个区中有64个连续页**。页的大小支持用参数**KEY_BLOCK_SIZE**进行设置，对应的每个区内页的数量也会发生变化。

### 4.2.4 页

InnoDB中常见的页类型有：

- **数据页(B-tree Node)**
- **undo页(undo Log Page)**
- **系统页(System Page)**
- **事务数据页(Transaction system Page)**
- **插入缓冲位图页(Insert Buffer Bitmap)**
- **插入缓冲空闲列表页(Insert Buffer Free List)**
- **未压缩的二进制大对象页(Uncompressed BLOB Page)**
- **压缩的二进制大对象页(compressed BLOB Page)**

### 4.2.5 行

InnoDB存储引擎是面向列的，数据按行进行存放，每个页存最多允许存放16KB/2~200行的记录。

***

## 4.3 InnoDB行记录格式



## 4.4 InnoDB数据页结构

## 4.5 Named File Fromats机制

## 4.6 约束

## 4.7 视图

## 4.8 分区表

***

# 5、索引与算法

InnoDB存储引擎支持以下几种常见的索引：

- B+树索引
- 全文索引
- 哈希索引

哈希索引前文已经介绍过，InnoDB存储引擎会根据表的使用情况自动生成，无法人为干预是否在一张表中生产哈希索引。

B+树索引构造类似于二叉树，根据键值快速找到数据。需要注意的是，**B+树索引并不能找到一个给定键值所在的具体行，只是找到被查找数据行躲在的页，然后数据库将该页读入到内存，再从内存中进行查找，最后得到要查找的数据。**

## 5.1 B+树

B+树是为磁盘或其他直接存取辅助设备设计的一种平衡查找树，所有记录节点都是**按键值的大小顺序存放在同一层的叶子节点**上，由各叶子节点指针进行连接。

![](images/MySQL08.png)

### 5.1.1 B+树的插入操作

B+树的插入必须保证插入后叶子节点中的记录依然排序，主要有以下三种情况：

**1）Leaf Page和Index Page均未满**

- 直接将记录插入到叶子节点

![](images/MySQL09.png)

**2）Leaf Page满，Index Page未满**

- 拆分Leaf Page
- 将中间的节点放入Index Page
- 小于中间节点的记录放左边，大于等于中间的记录放右边

![](images/MySQL10.jpg)

**3）Leaf Page和Index Page均满**

- 拆分Leaf Page
- 小于中间节点的记录放左边，大于等于中间的记录放右边
- 拆分Index Page
- 小于中间节点的记录放左边，大于中间的记录放右边
- 中间节点放入上一层Index Page

![](images/MySQL11.jpg)

上述的操作可能需要做大量的拆分页，意味着大量的磁盘操作，因此B+树提供了类型于平衡二叉树的旋转功能。

![](images/MySQL12.jpg)

### 5.1.2 B+树的删除操作

B+树使用**填充因子(fill factor)**控制树的删除变化，50%是填充因子可设的最小值。删除操作根据填充因子的变化有以下三种：

**1）叶子节点和中间节点均不小于填充因子**

直接将记录从叶子节点删除，如果该节点为Index Page的节点，则用该节点的右节点代替

例如删除70：

![](images/MySQL13.jpg)

删除25：

![](images/MySQL14.jpg)

**2）叶子节点小于填充因子，中间节点大于等于填充因子**

合并叶子节点和它的兄弟节点，同时更新Index Page

**3）叶子节点和中间节点均小于填充因子**

合并叶子节点和它的兄弟节点，更新Index Page，合并Index Page和它的兄弟节点

例如删除60：

![](images/MySQL15.jpg)

***

## 5.2 B+树索引

在数据库中，B+树的高度一般都在**2~4层**，也就是查找某一键值的行记录最多需要2到4次IO。数据库中的B+树索引可以分为**聚集索引(clustered index)和辅助索引(secondary index)**，主要区别在于叶子节点存放的是否为一整行的信息。

### 5.2.1 聚集索引

聚集索引**按照每张表的主键**构造B+树，同时叶子节点中存放的是整张表的行记录数据，称为数据页。实际的数据页只能按照一棵B+树进行排序，因此每张表只能拥有一个聚集索引。在多数情况下，查询优化器倾向于采用聚集索引。

聚集索引存储逻辑上连续，通过双向链表链接，页按照主键顺序排序，每个页中的记录也是通过双向链表进行维护。

聚集索引对于**主键的排序查找和范围查找**速度非常快，通过叶子节点的上层中间节点可以得到页的范围，之后直接读取数据页即可。

### 5.2.2 辅助索引

辅助索引的叶子节点不包含行记录的全部数据，除了包含键值以为，每个叶子节点的索引行中还包含一个**书签(bookmark)**，用于告诉InnoDB去哪里找到于索引相对于的行数据，即**相应行数据的聚集索引键**。

当通过辅助索引来寻找数据时，InnoDB存储引擎会遍历辅助索引，并通过叶级别的指针获得指向主键索引的主键，然后通过主键索引来找到一个完整的行记录。

### 5.2.3 B+树索引的分裂(有点懵，没看懂)

B+树索引页的分裂并不总是从页的中间记录开始，插入是根据自增顺序进行的，InnoDB存储引擎的**Page Header**中有几个部分用于保存插入的顺序信息：

- PAGE_LAST_INSERT
- PAGE_DIRECTION
- PAGE_N_DIRECTION

通过这些信息，InnoDB可以决定向左还是向右分类，同时决定分裂点记录为哪一个。若随机插入，则取页的中间记录作为分裂点的记录。若往同一方向插入的数量为5，并且当前定位到的记录之后还有3条记录，则分裂点的记录为定位到的记录之后的第三条记录。

### 5.2.4 B+树索引的管理

**1）索引管理**

索引的创建和删除有两个方法：

- **ALTER TABLE**
- **CREATE/DROP INDEX**

可以设置对整个列的数据进行索引，也可以只索引一个列的开头部分数据。查看索引命令：

```mysql
SHOW INDEX FROM tbl_name;
```

**2）快速索引创建(Fast Index Creation)**

对于辅助索引的创建，InnoDB支持使用Fast Index Creation(FIC)，会对创建索引的表加上一个S锁，不需要重建表。由于FIC在索引创建过程中对表加上了**S锁**，因此在创建的过程中，该表只**支持读操作，不支持写操作**。删除辅助索引，InnoDB只需要更新内部视图，并将辅助索引的空间标记为可用，同时删除MySQL数据库内部视图上对该表的索引定义即可。

FIC只限定于辅助索引的创建，对于**主键的创建和删除**，MySQL操作过程为：

- 创建一张新的临时表，表结构为通过命令ALTER TABLE新定义的结构
- 将原表中的数据导入到临时表
- 删除原表
- 将临时表重命名为原表名

**3）在线架构改变(Online Schema Change，OSC)**

所谓“在线”是指在事务的创建过程中，可以有读写事务对表进行操作，这提供了原有MySQL数据库在DDL操作时的并发性。但OSC存在一定的局限性，要求进行修改的表一定要有主键，且表本身不能存在外键和触发器。

**4）Online DDL**



***

## 5.3 Cardinality

Cardinality用于评估索引是否具有高选择性，表示索引中不重复记录数量的**预估值**。在实际应用中Cardinality应尽可能地接近1。

生产环境中，索引的更新操作可能是非常频繁的，每次索引发生操作时就对其进行Cardinality的统计，显然会给数据库带来很大的负担，因此数据库对Cardinality的统计都是通过**采样(Sample)**的方法来完成的。

在InnoDB中，Cardinality统计信息的更新发生在两个操作中：**Insert**和**Update**。其内部**更新Cardinality的策略**为：

- **表中1/16的数据发生过变化**
- **stat_modified_counter > 2000000000**

InnoDB内部同样是通过采样的方法，对Cardinality进行统计及更新。默认对**8个叶子节点**进行采样，过程为：

- 取得B+树索引叶子节点的数量，记为A
- 随机取得B+树索引中的8个叶子节点。统计每个页不同记录的个数，记为P1，P2，...，P3。
- 根据采样信息给出Cardinality的预估值：Cardinality = (P1+P2+...+P8)*A/8

***

## 5.4 B+树索引的使用

### 5.4.1 联合索引

联合索引是指对表上的多个列进行索引，本质上，就是B+树的键值数量不是1，而是大于等于2。同样地，**多个键值依次都进行了排序处理(最左匹配原则)**。

### 5.4.2 覆盖索引

从辅助索引中可以直接得到需要查询的记录，称为覆盖索引。使用覆盖索引不需要查询聚集索引中的记录，辅助索引不包含整行记录的所有信息，可以减少大量的IO操作。类似于：

```mysql
SELECT primary key1 FROM table WHERE key1=xxx;
SELECT COUNT(*) FROM table;
```

### 5.4.3 Muti-Range Read 优化

MRR优化的目的是为了减少磁盘的随机访问，并且将随机访问转化为较为顺序的数据访问，适用于**range，ref，eq_ref类型的查询**。

### 5.4.4 Index Condition Pushdown(ICP)优化

当进行索引查询时候，MySQL首先根据索引来查找记录，然后再根据WHERE条件来过滤。次啊用ICP优化后，MySQL会在取出索引的同时，判断是否可以进行WHERE条件的过滤，即**将Where的部分过滤操作放在了存储引擎层**。

ICP优化**支持range、ref、eq_ref、ref_or_null类型的查询**，当前支持MyISAM和InnoDB存储引擎。**当优化器选择ICP优化时，执行计划的Extra会显示Using index condition**。

***

## 5.5 哈希算法

InnoDB存储引擎是哟个哈希算法来对字典进行查找，其**冲突机制采用链表方式**，**哈希函数采用除法散列**。对于缓冲池的哈希表来说，在缓冲池中的Page页都有一个**chain指针**，它指向相同哈希函数值的页。

InnoDB存储引擎的表空间都有一个space_id，用户所需查询的都是**某个表空间的某个连续16	KB的页**，即偏移量offset。将space_id左移20位，然后加上space_id和offset，即**关键字K=spac_id<<20+space_id+offset**，然后通过除法散列到各个槽中。

***

## 5.6 全文索引

在某些需求场景中，数据库需要支持**全文检索(Full-Text Search)**，将存储于数据库中的整本书或整篇文章中的人员内容信息查找出来。

### 5.6.1 倒排索引

**全文检索通常使用倒排索引(inverted index)来实现**，在**辅助表(auxiliary table)**中存储了**单词与单词自身在一个或多个文档中位置之间的映射**，通常采用**关联数组**实现，表现形式有以下两种：

- **inverted file index，其表现形式为{单词，单词所在文档的ID}**
- **full inverted index，其表现形式为{单词，(单词所在文档的ID，在具体文档中的位置)}**

举例说明：

![](images/MySQL16.jpg)

DocumentId表示全文检索文档的Id，Text表示存储的内容。关联数组若采用inverted file index，则如下：

![](images/MySQL17.jpg)

若采用full inverted index，则如下：

![](images/MySQL18.jpg)

### 5.6.2 InnoDB全文索引

InnoDB存储引擎从1.2.x版本之后支持全文检索，采用的是**full inverted index的方式**，**将(DocumentId，Position)视为一个“ilist”**。在全文检索的表中，有word和ilist两个字段，并在**word字段上设有索引**。

由于倒排索引需要将word存放到一张辅助表(Auxiliary Table)中，在InnoDB中共有**6张Auxiliary Table**，持久化在磁盘上。

InnoDB为提高全文检索性能还引入了**FTS Index Cache(全文检索索引缓存)**，采用红黑树结构，根据(word，ilist)进行排序。InnoDB会**批量对Auxiliary Table进行更新**，而不是每次插入数据后就更新，这意味着插入数据已经更新了对应的表，但对全文索引的更新可能在分词操作后还在FTS Index Cache中。

当数据库正常关闭时，数据库会将当前FTS Index Cache中的数据同步到Auxiliary Table(持久化)中。若数据库异常宕机，FTS Index Cache可能未被同步到磁盘上，则再启动数据库时，当用户对表进行全文检索时，InnoDB会自动读取未完成的文档，重新进行相关全文索引的操作。

**对全文检索进行查询时，首先会将FTS Index Cache中对应的word字段合并到Auxiliary Table之后再进行查询。**

***

# 6、锁

锁机制用于管理共享资源的并发访问，InnoDB存储引擎会在**行级别**上对表数据上锁，提供一致性的**非锁定读、行级锁支持**。行级锁没有相关额外的开销，可以同时得到并发性和一致性。

## 6.1 lock与latch

**latch**一般称为**闩锁(轻量级锁)**，要求锁定的时间必须非常短。在InnoDB中，latch可以分为**mutex(互斥量)**和**rwlock(读写锁)**，用来保证并发线程操作临界资源的正确性，且**通常没有死锁检测的机制**。

**lock的对象是事务**，用来**锁定数据库中的对象，如表、页、行**，且一般lock的对象**仅在事务commit或rollback后进行释放**(不同事务隔离级别释放的时间可能不同)。

|          | lock                                                  | latch                                                        |
| -------- | ----------------------------------------------------- | ------------------------------------------------------------ |
| 对象     | 事务                                                  | 线程                                                         |
| 保护     | 数据库内容                                            | 内存数据结构                                                 |
| 持续时间 | 整个事务过程                                          | 临界资源                                                     |
| 模式     | 行锁、表锁、意向锁                                    | 读写锁、互斥量                                               |
| 死锁     | 通过waits-for graph、time out等机制进行死锁检测与处理 | 无死锁检测与处理机制，仅通过应<br />用程序加锁顺序(lock leveling)保证无死锁情况发生 |
| 存在于   | Lock Manager的哈希表中                                |                                                              |

```mysql
// 查看InnoDB的latch
SHOW ENGINE INNODB MUTEX;
```

***

## 6.2 InnoDB存储引擎中的锁

### 6.2.1 锁的类型

InnoDB存储引擎实现了两种行级锁：

- **共享锁(S Lock)**，允许事务读一行数据
- **排他锁(X Lock)**，允许事务删除或更新一行数据

假设事务T1已获得行r的共享锁，则事务T2可以立即获得行r的共享锁，这种情况也称为**锁兼容(Lock Compatible)**。若此时事务T3想获得行r的排他锁，则必须等待事务T1、T2释放行r上的共享锁，即锁不兼容。**排他锁与任何锁都不兼容，而共享锁仅跟共享锁兼容**。

此外，InnoDB支持**多粒度(granular)锁定**，允许事务在行级上的锁和表级上的锁同时存在。为了支持在不同粒度上的加锁操作，InnoDB支持将锁定的对象分为多个层次，即**意向锁(Intention Lock)****，意味着事务希望在更细粒度上进行加锁。

![](images/MySQL19.jpg)

若事务t需要对页上的记录r上X锁，则需要分别对数据库A、表、页加上意向锁IX，最后对记录r加X锁。如果在其中一个环节导致等待，则该操作需要等待粗粒度锁的完成。例如，在对记录r加X锁前，已有其他事务对表加了S表锁，则事务t无法对表加IX锁，需等待表锁释放之后，再加上IX锁。

**意向锁为表级别的锁**，设计目的主要是为了在一个事务中揭示下一行将被请求的锁类型，有以下两种：

- **意向共享锁(IS Lock)**，获得一张表中某几行的共享锁
- **意向排他锁(IX Lock)**，获得一张表中某几行的排他锁

|      | IS     | IX     | S      | X      |
| ---- | ------ | ------ | ------ | ------ |
| IS   | 兼容   | 兼容   | 兼容   | 不兼容 |
| IX   | 兼容   | 兼容   | 不兼容 | 不兼容 |
| S    | 兼容   | 不兼容 | 兼容   | 不兼容 |
| X    | 不兼容 | 不兼容 | 不兼容 | 不兼容 |

### 6.2.2 一致性非锁定读

一致性的非锁定读(consistent nonlocking read)是指InnoDB存储引擎通过**行多版本控制(multi versioning)**的方式读取当前指向时间数据库中行的数据。即**若读取的行正在执行写操作，这时读取操作不会等待行锁的释放**，而是取读取行的一个**快照数据**，通过undo段来完成。

快照数据其实就是当前行数据的历史版本，每行记录可能有多个版本，称为**行多版本技术**，由此带来的并发控制称为**多版本并发控制(Multi Version Concurrency Control，MVCC)**。

在事务隔离级别**READ COMMITED和REPEATABLE READ**下，InnoDB使用非锁定的一致性读。但在**READ COMMITED**下，非一致性读总是**读取被锁定行的最新一份快照数据**，而在**REPEATABLE READ**下，非一致性读总是**读取事务开始时的行数据版本**。

### 6.2.3 一致性锁定读

InnoDB对**SELECT语句支持两种一致性的锁定读(locking read)**操作：

- **SELECT ... FOR UPDATE**

  对读取的行记录加一个X锁，其他事务不能对已锁定的行加上任何锁。

- **SELECT ... LOCK N SHARE MODE**

  对读取的行记录加一个S锁，其他事务可以向被锁定的行加S锁，若加X锁，则会阻塞。

### 6.2.4 自增长与锁

在InnoDB的内存结构中，对每个含有自增长值得表都有一个自增长计数器(auto-increment counter)，当进行插入操作是，计数器会被初始化，执行：

```mysql
SELECT MAX(auto_inc_col) FROM t FOR UPDATE;
```

依据计数器的值加1赋予自增长列，称为AUTO-INC Locking，这种表锁机制为了提高插入性能，**不在一个事务完成后释放锁，而是在完成对自增长值插入的SQL语句后立即释放锁**。

从5.1.22版本开始，InnoDB通过对自增长的插入进行分类，提供了一种轻量级互斥量的自增长实现机制，并提供了innodb_autoinc_lock_mode参数控制自增长的模式，默认值为1。

![](images/MySQL20.png)

![](images/MySQL21.png)

***

## 6.3 锁的算法

### 6.3.1 行锁的3种算法

InnoDB有3种行锁的算法，分别是：

- **Record Lock**：单个行记录上的锁
- **Gap Lock**：**间隙锁**，锁定一个范围，但不包含记录本身
- **Next-Key Lock**：Gap Lock+Record Lock，锁定一个范围，并且锁定记录本身

Record Lock总是会去锁定索引记录，若创建表的时候未设置任何一个索引，则会使用隐式的主键来进行锁定。**InnoDB对于行的查询采用的都是Next-Key Lock算法**，采用的锁定技术称为**Next-Key Locking**。这种锁定技术锁定的不是单个值，而是一个范围，是谓词锁(predict lock)的一种改进。还有**Previous-Key Locking**技术，例如一个索引有10、11、13和20四个值，则Next-Key Locking和Previous-Key Locking的区间分别为：

| Next-Key Locking | Previous-Key Locking |
| ---------------- | -------------------- |
| (-00,10]         | (-00,10)             |
| (10,11]          | [10,11)              |
| (11,13]          | [11,13)              |
| (13,20]          | [13,20)              |
| (20,+00)         | [20,+00]             |

当**查询的索引含有唯一属性**时，InnoDB引擎会对Next-Key Lock进行优化，将其**降级为Record Lock**，即仅锁住索引本身，而不是范围。

### 6.3.2 解决Phantom Problem

**Phantom Problem(幻像问题)**是指在同一事务下，连续执行两次同样的SQL语句可能导致不同的结果，第二次SQL可能会返回之前不存在的行。在默认事务隔离级别下，即REPEATABLE READ下，InnoDB采用Next-Key Locking机制避免幻像问题。

***

## 6.4 锁问题

### 6.4.1 脏读

在缓冲池中已经被修改的页，但还没刷新到磁盘中，称为**脏页(Dirty Read)**。所谓脏数据是值事务对缓冲池中行记录的修改，并且还没被提交(commit)。脏页是因为数据库实例内存和磁盘的异步造成的，不影响数据的一致性。**脏读**指的就是在不同事务下，当前事务可以读到另一事务未提交的数据。

### 6.4.2 不可重复读

不可重复读是指事务T1多次读取同一数据集合，在T1未结束时，事务T2也访问同一个数据集合，并进行DML操作。此时，事务T1中的两次读数据，事务T2的修改，事务T1两次读到的数据可能不同，这种情况称为**不可重复读**。

**不可重复读读到的是已经提交的数据，脏读读到的是未提交的数据。**

### 6.4.3 丢失更新

丢失更新就是事务T1的更新操作被另一个事务T2更新操作所覆盖，导致数据不一致。

## 6.5 阻塞

某些时刻一个事务的锁需要等待另一个事务中的锁释放，造成阻塞。在InnoDB中，参数**innodb_lock_wait_timeout**和**innodb_rollback_on_timeout**控制等待的时间(默认50s)和等待超时时是否对事务进行回滚操作(默认OFF)。innodb_lock_wait_timeout支持动态调整，innodb_rollback_on_timeout不支持。

***

## 6.6 死锁

死锁是指两个或两个以上的事务在执行过程中，因争夺锁资源而造成的一种互相等待的现象。

解决死锁最简单的方法是**超时**，当两个事务互相等待时候，当其中一个事务等待时间超过设置的阈值时，其中一个事务回滚，另一个事务继续进行。若超时的事务所占权重比较大，事务操作更新了很多行，采用FIFO的方式选择回滚对象，就不合适了。

当前数据库普遍采用**wait-for graph(等待图)**进行**死锁检测**，要求数据库保存：

- **锁的信息链表**
- **事务等待链表**

将上述链表构造成一张图，若存在回路则代表存在死锁。

![](images/MySQL22.png)

事务为图中的节点，事务T1指向T2边的定义为：

- 事务T1等待事务T2所占用的资源
- 事务T1最终等待T2所占用的资源，事务之间在等待相同的资源，而T1发生在T2后面

wait-for graph的死锁检测通常采用**深度优先算法**实现。

***

## 6.7 锁升级

InnoDB存储引擎不是根据每个记录产生行锁的，而是根据每个事务访问的每个页对锁进行管理，采用的是位图的方式，锁住一张页中一条记录和多条记录，开销通常都是一致的。

***

# 7、事务

## 7.1 ACID

### 7.1.1 A(Atomicity，原子性) 

整个数据库事务是不可分割的工作单位，只有事务中所有数据库操作都成功，事务才算成功。

### 7.1.2 C(Consistency，一致性)

一致性指事务将数据库从一种状态转变为下一种一致的状态。在事务开始之前和事务结束以后，数据库的完整性约束没有被破坏。

### 7.1.3 I(Isolation，隔离性)

事务的隔离性要求每个读写事务的对象对其他事务的操作对象能相互分离，即该事务提交前对其他事务都不可见，通常使用锁来实现。当前数据库系统种都提供了一种**粒度锁(granular lock)**的策略，允许事务仅是锁住一个实体对象的子集，以此提高事务之间的并发度。

### 7.1.4 D(Durability，持久性)

事务一旦提交，其结果就是永久性的。即使发生宕机等故障，数据库也能将数据恢复，只能从事务本身的角度保证结果的永久性。

## 7.2 事务的分类

### 7.2.1 扁平事务

所有的操作都处于同一层次，其**由BEGIN WORK开始，由COMMIT WORK或ROLLBACK WORK结束**，因此扁平事务是应用程序称为原子操作的基本组成模块。主要限制是**不能提交或者回滚事务的某一部分**。

### 7.2.2 带有保存点的扁平事务

除了支持扁平事务支持的操作外，**允许事务执行过程中回滚到同一事务中较早的一个状态**。**保存点**用来通知系统应该记住事务当前的状态，以便当之后发生错误时，事务能**回到保存点当时的状态**。

对于扁平事务来说，其隐式地设置了一个保存点，整个事务只有这一个保存点，因此回滚只能回滚到事务开始时的状态。**保存点用SAVE WORK函数建立**，通知系统记录当前的处理状态。出现问题时，保存点作为内部的重启动点，根据应用逻辑，决定回到最近的一个保存点还是其他更早的保存点。

### 7.2.3 链事务

当系统崩溃时，扁平事务的保存点都将消失，意味着当进行恢复时，事务需要从头开始执行。链事务的思想是：在提交一个事务时，释放不需要的数据对象，将必要的处理上下文隐式地传给下一个要开始的事务。提交事务操作和开始下一个事务操作将合并为一个原子操作，意味者喜爱一个事务将看到上一个事务的结果。

![](images/MySQL23.png)

带有保存点的扁平事务可以回滚到任意正确的保存点，而**链事务的回滚仅限于当前事务**，即只能恢复到最近的一个保存点。链事务执行**COMMIT后即释放了当前事务所持有的锁**，而带有保存点的扁平事务不影响当前所持有的锁。

### 7.2.4 嵌套事务

由一个**顶层事务(top-level transaction)**控制着各个层次的事务。顶层事务下嵌套着**子事务(subtransaction)**，其控制每个局部的变换。

![](images/MySQL24.png)

- 处于叶节点的事务是扁平事务，但是每个子事务从根到叶节点的距离可以是不同的。
- 事务的前驱称为父事务，事务的下一层称为儿子事务
- 任何子事务需要在顶层事务提交后才真正的提交
- 树中的任意一个事务的回滚会引起所有子事务一同回滚

**实际工作由叶节点完成**，只有叶节点的事务才能访问数据库、发送消息、获取其他类型的资源。**高层的事务仅负责逻辑控制**，决定何时调用相关的子事务。

### 7.2.5 分布式事务

在分布式环境下运行的扁平事务，需要根据数据所在位置访问网络中的不同节点。

## 7.3 事务的实现

### 7.3.1 redo log

重做日志用来实现事务的持久性，即事务ACID中的D，由**内存中的重做日志缓冲(redo log buffer)和重做日志文件(redo log file)**。



















