### **事务的四大特征ACID**

**①：原子性（Atomicity）**

原子性：事务是数据库的逻辑工作单位，事务中包含的诸多操作要么全做、要么不做。因故障未能做完的，需要有一套机制用于“撤销”那一部分已经做了的

 

**②：一致性（Consistency）**

一致性：事务执行的结果必须是使数据库从一个一致性状态变到另一个一致性状态。

- 一致性状态：数据库中只包含成功事务提交的结果
- 不一致状态：数据库中包含事务未完成时的状态

事务操作前后，数据总量不变。保证隔离性才能确保一致性。

例如银行转账业务，账户 A 转1万元到账户 B ,该事务包含两个操作：首先是 A 减少一万元，其次是 B 增加一万元，这两个操作要么全部做要么全不做，如果只做其中一个就会发生逻辑错误，数据库就处于不一致状态了

 

**③：隔离性（Isolation）**

隔离性：一个事务不能被其他事务干扰。也即一个事务的内部操作及使用的数据对其他并发事务是隔离的，并发执行的各个事务之间不能互相干扰

比如，下列两个并发执行的事务T1和T2，如按表中所示顺序执行，则事务T1的修改被T2覆盖了，即T2干挠了T1。违背了事务的隔离性，是错误的调度

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240317134503291.png" alt="image-20240317134503291" style="zoom:50%;" />



**④：持续性（Durability）**

持续性：一个事务一旦提交，它对数据库中数据的改变就是永久性的。接下来的其他操作或故障不应该对其执行结果有任何影响

 

#### **破坏ACID的因素**

主要有两类

**故障：**没有执行完；虽然没有完，但是存储介质故障。破坏了ACID中的ACD

**并发干扰：**多个事务并行运行时，不同事务的操作交叉执行，互相干扰。破坏了ACID中的I，因此这就是DBMS的恢复机制和并发控制机制需要解决的问题



 

### **并发控制**

并发操作带来的数据不一致性问题

主要有四类数据不一致性问题

- 丢失修改
- 读脏数据
- 不可重复读
- 幻读



#### **丢失修改**

丢失修改：两个以上事务从数据库中读入同一数据并修改，其中后提交事务的提交结果破坏了先提交事务的提交结果，导致了先提交事务对数据库的修改丢失。

在上面例子中，两个事务 T1和T2读入同一数据并修改，但是T2提交的结果破坏了T1提交的结果，导致T1的修改被丢失。

解决方法：加X锁



#### **读脏数据**

读脏数据：事务1修改某一数据，并将其写回磁盘；事务2读取同一数据后，事务1由于某种原因被撤销，这时事务1已修改过的数据被恢复为原值，事务2读到的不稳定的瞬间数据就与数据库中的数据产生了不一致，是不正确的数据，又称为脏数据。

解决方法：快照读（根据已提交的事务更新ReadView）

 

#### **不可重复读**

不可重复读：事物1读取数据后，事物2执行更新操作，使事物1无法再现前一次读取结果。

**共有三种情况**

- 事物2修改了事物1所读数据，当事物1再次读该数据时，得到了与前一次不同的值
- 事务2删除了其中部分记录，当事务1再次按相同条件读取数据时，发现某些记录消失了
- 事务2插入了一些记录，当事务1再次按相同条件读取数据时，发现多了一些记录

 解决方法：快照读(保持一个事务的ReadView不变)

#### **幻读**

幻读是指当事务不是独立执行时发生的一种现象，例如第一个事务对一个表中的数据进行了修改，比如这种修改涉及到表中的“全部数据行”。同时，第二个事务也修改这个表中的数据，这种修改是向表中插入“一行新数据”。那么，以后就会发生操作第一个事务的用户发现表中还存在没有修改的数据行，就好象发生了幻觉一样。

幻读并不是说两次读取获取的结果集不同，幻读侧重的方面是某一次的 select 操作得到的结果所表征的数据状态无法支撑后续的业务操作。

解决方法：串行化，next-key锁，可重复读（快照可解决一部分）



### **事务的隔离级别**

概念：多个事务之间隔离的，相互独立的。但是如果多个事务操作同一批数据，则会引发一些问题，设置不同的隔离级别就可以解决这些问题。

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/image-20240304160800719.png" alt="image-20240304160800719" style="zoom: 67%;" />

 

#### **read uncommitted：读未提交**

当前读的数据其他事务未提交

产生的问题：丢失修改（更新数据不加X锁）、脏读、不可重复读、幻读

**<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/clip_image014.png" alt="img" style="zoom: 67%;" />**

**当前读：**会产生脏读，不可重复都读，幻读

**共享行锁：**当前事务更新的数据不能被其他事务修改，但能被其他事务读

 

 

#### **read committed：读已提交 （Oracle）**

产生的问题：不可重复读、幻读

**<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/clip_image017.png" alt="img" style="zoom:67%;" />**

**快照读：**每次查询都生成一个readview，根据readview检查活跃的事务，当前读的数据其他事务已提交，可以访问；当前读的数据其他事务未提交，不能访问，迭代访问下一个readview，直到找到可以访问的数据。

**X行锁：**当前事务更新数据不能被其他事务写也不能被其他事务读，因此只能读到更新后的数据，解决了丢失修改

 

#### **repeatable read：可重复读 （MySQL默认）**

事务查询第一次时创建一个readview，后续每次查询都用这个readview，不存在不可重复读问题

可能产生的问题：幻读

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/clip_image020.png" alt="img" style="zoom:67%;" />

**快照读：**只会在首次查询是生成一个readview，后面的查询共用这个readview，可以解决不可重复读

**Next-Key行锁：**在一个范围的行，其他事务不能更新和查询，解决了部分幻读的问题

 





#### **serializable：串行化**

可以解决所有的问题，但效率很低

**<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/clip_image023.png" alt="img" style="zoom:67%;" />**

**注意：**隔离级别从小到大安全性越来越高，但是效率越来越低



### **可重复读隔离下为什么会产生幻读？**

在可重复读隔离级别下，普通的查询是快照读，是不会看到别的事务插入的数据的。因此，幻读在**当前读**下才会出现。

什么是快照读，什么是当前读？

快照读读取的是快照数据。不加锁的简单的 SELECT都属于快照读，需要借助MVCC的机制。比如这样：

```
SELECT * FROM .... WHERE ...
```

如下例子：

查找年龄为20岁的作者，并把姓名改成G0

![img](https://pic2.zhimg.com/80/v2-ab809605698a47bf3c42174e63859935_720w.webp)

来分析下情形：

- T1时刻 读取年龄为20的数据， Session1拿到了2条记录。
- T2时刻 另一个进程Session2插入了一条新的记录，年龄也为20
- T3时刻，Session1再次读取年龄为20的数据，发现还是2条数据，貌似 Session2新插入的数据并未影响到Session1的事务读取。

然而再其更新数据时，却发现了影响了3行数据，因为update语句会把其他事务insert的数据更新为自己的版本号所以能读取到该新增的数据，产生幻读。

**为什么事务自己修改后就能读取到修改后的数据？**

当一个事务修改数据时，数据库会为这个修改创建一个新版本，并且这个新版本会与当前事务的事务ID关联。这个新版本成为当前事务的私有版本，只有在当前事务内部是可见的。这种机制确保了在可重复读隔离级别下，同一个事务内的修改对自己是可见的，而对其他事务则是隔离的。

**结论：**如果事务中都使用快照读（即在RR隔离级别下，只使用select *from...where...语句），那么就不会产生幻读现象，因为快照读只在第一次select时生成一个readView，之后每次读取都是用这个readView，所以即使其他事务新增的数据也不会影响快照读，但是**快照读和当前读混用就会产生幻读。**



### **MySQL 是怎么解决幻读的？**

MySQL InnoDB 引擎的默认隔离级别虽然是「可重复读」，但是它很大程度上避免幻读现象（并不是完全解决了，详见这篇[文章 (opens new window)](https://xiaolincoding.com/mysql/transaction/phantom.html)），解决的方案有两种：

- 针对**快照读**（普通 select 语句），是**通过 MVCC 方式解决了幻读**，因为可重复读隔离级别下，事务执行过程中看到的数据，一直跟这个事务启动时看到的数据是一致的，即使中途有其他事务插入了一条数据，是查询不出来这条数据的，所以就很好了避免幻读问题。
- 针对**当前读**（select ... for update 等语句），是**通过 next-key lock（记录锁+间隙锁）方式解决了幻读**，因为当执行 select ... for update 语句的时候，会加上 next-key lock，如果有其他事务在 next-key lock 锁范围内插入了一条记录，那么这个插入语句就会被阻塞，无法成功插入，所以就很好了避免幻读问题。

**当前读**

当前读就是读取最新数据，而不是历史版本的数据，加锁的 SELECT，或者对数据进行增删改都会进行当前读。

如下的操作都会进行**当前读**。

```
SELECT * FROM player LOCK IN SHARE MODE;
SELECT * FROM player FOR UPDATE;
INSERT INTO player values ...
DELETE FROM player WHERE ...
UPDATE player SET ...
```


说白了，快照读就是普通的读操作，而当前读包括了加锁的读取 和 DML（DML只是对表内部的数据操作，不涉及表的定义，结构的修改。主要包括insert、update、deletet） 操作。

在InnoDB中，当前读都会添加行锁操作，**因此使用当前读进行select时，会默认加next-key锁，自然也就避免了幻读。**







### **undo log与MVCC**

####  **undo log**

撤销日志，在数据库事务开始之前，MYSQL会去记录更新前的数据到undo log文件中。如果事务回滚或者数据库崩溃时，可以利用undo log日志中记录的日志信息进行回退。同时也可以提供多版本并发控制下的读(MVCC)。

 

**undo log生命周期**

undo log产生： 在事务开始之前生成

undo log销毁： 当事务提交之后，undo log并不能立马被删除，而是放入待清理的链表，由purge线程判断是否由其他事务在使用undo段中表的上一个事务之前的版本信息，决定是否可以清理undo log的日志空间。

注意： undo log也会生产redo log，undo log也要实现持久性保护。

 

**undo log日志可以实现事务的回滚操作**

我们在进行数据更新操作的时候，不仅会记录redo log，还会记录undo log，如果因为某些原因导致事务回滚，那么这个时候MySQL就要执行回滚（rollback）操作，利用undo log将数据恢复到事务开始之前的状态。

 

如我们执行下面一条删除语句：

```
delete from book where id = 1;
```

那么此时undo log会生成一条与之相反的insert 语句【反向操作的语句】，在需要进行事务回滚的时候，直接执行该条sql，可以将数据完整还原到修改前的数据，从而达到事务回滚的目的。

 

再比如我们执行一条update语句：

```
update book set name = "三国" where id = 1;  ---修改之前name = 西游记
```

此时undo log会记录一条相反的update语句，如下：

```
update book set name = "西游记" where id = 1;  ---11.3undolog和MVCC机制
```

如果这个修改出现异常，可以使用undo log日志来实现回滚操作，以保证事务的一致性。

 

**uodo log的工作原理**

**![img](https://palepics.oss-cn-guangzhou.aliyuncs.com/img/clip_image002.gif)**

如上图所示：

当事务A进行一个update操作，将id=1修改成id=2。首先会修改buffer pool中的缓存数据，同时会将旧数据备份到undo log buffer中，记录的是还原操作的sql语句。

此时如果事务B要查询修改的数据，但是事务A还没有提交，那么事务B就会从undo log buffer中，查询到事务A修改之前的数据，也就是id=1。此时undo log buffer会将数据持久化到undo log日志中(落盘操作)。

undo日志持久化之后，才会将数据真正写入磁盘中，也就是写入IBD的文件中，最后才会执行事务的提交。

 

**uodo log的存储机制**

**![img](https://palepics.oss-cn-guangzhou.aliyuncs.com/img/clip_image004.gif)**

 

 

 

#### **MVCC**

MVCC，全称Multi-Version， Concurrency Control，即多版本并发控制。MVCC是一种并发控制的方法，一般在数据库管理系统中，实现对数据库的并发访问，在编程语言中实现事务内存。



**undo log版本链**

在undo log日志里，每条数据除了自有的那些字段(表id、日志类型、数据页号等等)，其实还会有两个隐藏字段，一个是trx_id，另一个是roll_pointer。

- 这个trx_id就是最近一次更新的事务id

- roll_pointer是指向你更新这个事务之前生成的undo log数据。

  

**例子：**

假设有一个事务A，插入了一个数据A，此时的undo log数据结构如下：

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/clip_image006.gif" alt="img" style="zoom:80%;" />

 

 

因为事务id是10，所以这条数据的trx_id=10。因为是插入数据，所以没有下一个undo log数据，roll_pointer是空的。

 

接着，此时有一个事务B需要执行，事务B的id=20，那么执行完之后就会新生成一条undo log日志数据，trx_id=20，roll_pointer就会指向实际的回滚日志，也就是值A那条数据。结构如下图所示：

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/clip_image008.gif" alt="img" style="zoom:80%;" />

 

 

以此类推，在这个多个事务中，每个事务新生成的undo log日志数据的roll_pointer都会指向前一个undo log日志数据，一次行程undo log版本链。

 

 

**基于undo log多版本链实现的ReadView机制**

简单来讲，ReadView就是执行一个事务时，会生成一个ReadView，这里面会有比较关键的4个字段：

- m_ids：记录有哪些事务在mysql里还没有提交。
- min_trx_id：m_ids里的最小值。如果当前行的trx_id小于活跃事务的最小值，说明已经提交。
- max_trx_id：下一个mysql要生成的事务id。处于min_trx_id和max_trx_id之间的事务即为活跃事务
- creator_trx_id：就是当前事务本身的事务id。

 

 

#### **ReadView解决读已提交隔离级别（RC隔离级别）**

假设mysql里有个数据，很早之前就有事务插入了，事务id是20，如下图所示：

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/clip_image010.gif" alt="img" style="zoom:67%;" />

 

此时，有两个事务并发过来执行，分别是事务Aid=30，要读取这行数据；事务Bid=35

当事务A要读数据时，就会生成一个ReadView，当前活跃事务m_ids的为30，35，min_trx_id=30，max_trx_i=36，表示30之前的所有事务已经提交了

此时事务A会判断当前行的trx_id是否小于ReadView中的min_trx_id。发现20<30，所以可以得知在事务A开启之前，当前行的事务就已经提交了，因此事务A可以直接读取这条数据。

![img](https://palepics.oss-cn-guangzhou.aliyuncs.com/img/clip_image012.gif)

 

接着事务B开始操作，他把初始值修改成了值B，trx_id设置为自己的事务id，也就是35，同时roll_pointer指向了之前生成的undo log，然后事务B提交了。如下图：

![img](https://palepics.oss-cn-guangzhou.aliyuncs.com/img/clip_image014.gif)

 

这个时候，事务A再查询，就会发现一个问题，事务A就会发现trx_id变成了35，那么trx_id大于min_trx_id=30，同时小于max_trx_id=36，说明这个事务是活跃的事务。

然后就会查看这个trx_id是否在m_ids中，在m_ids中发现了35的id，那么就证明当前的数据是和自己同一时间段并发启动的事务修改的（但是仍未提交），所以按道理这条数据不能让他看到（读已提交隔离级别）

然后顺着roll_pointer找之前的undo log数据，发现trx_id=20，小于min_trx_id，说明这条数据是在事务A提交之前就完成的，符合查询条件，读取这条数据

![img](https://palepics.oss-cn-guangzhou.aliyuncs.com/img/clip_image016.gif)

 

**注意：**读已提交隔离级别中，事务每次执行，都会重新生成一个ReadView，因为只有这样才能获取到最新的事务ID数据。

例如：如果此时将事务B提交，事务A就会**重新生成一个ReadView**，那么35就不在该ReadView的m_ids里，事务A就可以读取到事务B修改后的数据

 



#### **ReadView机制是如何实现可重复读隔离级别（RR隔离级别）**

在RR隔离级别里，当前事务读取一条数据，无论读取多少次，都是一个值，ReadView也一样，别的事务哪怕事务提交了，也不能看到修改后的值，这样就避免了不可重复读的问题。

例子：

首先假设有个数据，是事务id=50之前就插入进去的，现在活跃着两个事务，事务Aid=60，事务Bid=70。如下图：

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/clip_image018.gif" alt="img" style="zoom: 80%;" />

事务A发起查询操作，这时候会生成一个ReadView，这是creator_trx_id=60，m_ids=60、71，min_trx_id=60，max_trx_id=71。如下图：

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/clip_image020.gif" alt="img" style="zoom:80%;" />

 

这个时候当前数据的trx_id=50，小于事务A的60，证明当前事务早在事务A之前提交了，所以事务A可以查询到初始值。

 

接着就是事务B开始执行修改操作，此时trx_id=70，初始值改为值B，同时生成一个undo log，并且事务B提交了，也就是说此时事务B已经结束了。如下图：

<img src="https://palepics.oss-cn-guangzhou.aliyuncs.com/img/clip_image022.gif" alt="img" style="zoom:80%;" />

 

**那么此时事务A再次进行查询操作，m_ids的值是多少呢？**

答案是m_ids=60，70。因为在RR隔离级别中，ReadView一旦生成，就不会改变，这个时候，虽然事务B已经提交了，但是事务A中的ReadView里，还是会有60、70两个活跃事务id。

那么此时，事务A会判断trx_id是否大于60，很明显70>60，然后再查看m_ids中存在trx_id=70，所以这时候事务A还是认为事务B此时还是处于未提交状态，因此不会被允许查看事务B的值，他会根据roll_pointer找到上一条undo log数据，再次判断，因此事务A查到的数据还是初始值。

 

 

 

 

 

 

 

 

 

 

 

 

 













