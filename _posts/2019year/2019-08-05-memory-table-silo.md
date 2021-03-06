---

layout: post

title: "Silo论文： Speedy Transactions in Multicore In-Memory Databases"

date: 2019-08-05

category: 2019年

keywords: Silo, In Memory Table, OCC, MassTree, 

---

## 摘要



## 简介



## 相关工作



### 非事务系统



### 事务系统



### 事务内存(Transactional Memory)



## 架构

Silo是一个关系型数据库。它的客户端发出one-shot请求：1） 当请求开始时，所有的参数都是可用的； 2） 请求直接完成才会与调用者进行交互。对于one-shot请求，我们使用的是C++来实现写的。

Silo表是使用索引的集合来实现的，包括了1个主索引树和0~n个辅助索引。 主索引树将会指向表记录，而 表记录使用独立分配的内存块进行存储， 要访问主键指向的表记录，就需要使用指定主键要遍历主索引树。 所以必须指定一个唯一 的 键，如若不指定，Silo会自己选择一个。 而辅助索引则仅记录了指向表记录的主键索引记录， 所以通过辅助索引来访问表记录，需要进行两个索引树访问。  所有的键值都以字符串类型对待。

索引是使用Masstree来实现的。Masstree的读写是协作的，会使用版本号和基于fence的同步。 与其他树相比，它还采取了重试的键值比较机制。叶子节点包含一个范围的键值，它可能直接指向一个表记录，也可能指向另一个更低层次的树结构以便继续扫描。 

Silo在现代多核机器上的扩展性是比较好的。 每个one-shot请求将会分发到一个线程上，而这个线程又会运行在某个物理核上。 由于树存储在共享主存内， 所以所有的工作线程都可以访问整个数据库。

尽管数据的主副本是在内存里的，事务仍然是支持日志和持久化的。数据持久化后，才会返回给客户端结果。 

在客户端的决策来看， 只读事务可能运行在数据库最近的一个一致性快照上，而非当前状态上。 这些snapshot transaction返回稍微稳定的结果，决不会由于并发修改而回滚。

## 设计

本小节描述事务执行。一个关键的组织原则是： 通过减少向共享内存的写操作，来减少不必要的竞争。   本小节的关键基础是epoch。

### Epochs

epochs是Silo的基础，每一个epoch都了一个数字对应， 为epoch number。

全局epoch number E对所有线程可见，有一个专门的线程周期性地推进E， 其他线程则在提交事务的时候访问E。 E应该频繁地推进， 因为这个周期性时间 会影响事务时延； 同时， epoch的变化与事务时长相比，应该几乎没有， 所以E值才能够被缓存。 论文实现中， E为40毫秒， 当然更小的数值也是可以的。

每一个工作线程w同样维护着一个本地的epoch number ew。 当该线程计算的时候，值ew会落后于E, 该值同样会用于垃圾回收，以决定什么时候垃圾回收会是安全的。 论文 要求ew与E不能够分叉太远： 对于所有的工作线程， 二者差距必须小于1秒。 如果工作线程确实落得太远，那么专门推进线程也会延后其推进动作。 因而， 工作线程应该周期性地刷新它的本地ew， 以确保整个系统向前进。 

snapshot transaction 用的则是另外一个epoch变量，后面会描述。 

### 事务ID

TID用于标识事务和记录版本号，起到了锁的作用、用于冲突检测。 每一个记录包含了最近修改它的事务TID值。 

TID是一个64位的整数。 高bit位是epoch number， 事务提交 时该数值与全局epoch number相等。 中间bit位用于区分相同epoch number内的不同事务。 低三位为状态位，后面为描述。

与其他系统不同， Silo的TID分配是去中心化的。 工作线程仅在确定所有该事务能够提交， 才会选择一个事务的TID。 在那个点它按照如下方法来计算其中的最小数字：(a) 比该事务读写的所有记录的TID都要大； (b) 比本线程最近选择的TID要大； (c) 位于当前的全局epoch中。  计算结果将写入该事务修改的所有记录中。 

TID顺序经常反映serial order，但并不总是如此。TID可以反映依赖关系（写写冲突和写后读冲突）， 但冻反映反依赖（读后写冲突） 。 不管怎么样，生成的TID是单调递增的，并且与serial order保持一致； 赋与一个特定记录的TID也是单调递增的，与serial order保持一致； 不同epoch的TID顺序与是与serial order保持一致的。 

低三位是状态位，用于并发控制。 这三个状态位主要是加锁位、版本最新位、缺席位。当这条记录有对应键的最新数据时， 版本最新位置1； 当这条记录被更新掉，那么该位清除。 缺席位表示该记录的键是不存在的，此类的记录要么是INSERT创建的，要么是DELETE操作删除的，可进行区分。

TID术语表示一个单纯的事务ID， 而TID字则是在事务ID的基础上包含了状态位。

### 数据层次

一条记录饮食三个部分：

1. TID字； 
2. 前一个版本的指针。 如果不存在前一个版本，那么指针为NULL。 此字段用于支持snapshot transaction;
3. 记录数据部分。

更新事务总是就地修改，可以减少内存的分配，加速短事务的执行性能。那么读事务就必须有一个版本检验协议来确认它们读取到的是一致性的数据。

### 提交协议

这部分主要讨论读后更新已有记录的操作， 后续会再讨论插入、删除和范围查询的操作。

工作线程会维护两个集合：读集合，同时会保留访问时记录对应的TID；写集合。一般来讲， 写集合中的记录必然会出现在读集合中。 提交协议会3步进行：

1. 第一阶段

   对写集合中的所有记录进行加锁。 为了避免死锁，可以按照一定的全局顺序进行加锁，例如内存地址的增序。

   工作线程使用单条内存访问fence来获得一个全局epoch数字。内存fence用来确保全局数值是最新的，而非cache中过时的；同时从逻辑上来讲， 之前的内存访问操作已经结束，之后的内存访问操作还不会开始。

   这个全局epoch数字的快照对于事务提交是一个serialization point。

2. 第二阶段

   检验读集合中的记录没有被修改。 下列3种情形视作修改：1） TID与执行过程中不同 ； 2） KEY不再是最新版本； 3） 被其他事务锁定 。 那么事务释放锁，并且回滚。

   如果读集合中的所有记录都未更改， 允许事务提交。 第1阶段中所获得的TID将赋于所有记录。

3. 第三阶段

   本事务将其修改写入到全局树上， 写完后立即放锁。这样， 新TID立即可见。

该提交协议是serializable的，因为 (1) 在校验读记录TID之前，对所有写的记录加锁 ；(2) 将已加锁记录视作要修改，事务回滚； (3) 结束第1阶段的fence可以保证TID校验是可以看到并发更新的。 该提交协议是遵循严格2PL的。

该提交协议同样确保epoch边界与serial order是一致的。epochs obey both dependencies and anti-dependencies.

### 数据库操作

**读写操作**

当一个事务提交时，如若可能会对修改记录进行覆盖写操作，这样会提升性能。 如若不行， 需要创建一个新内存来存储新记录，将旧记录标识为不再最新， 再修改索引树指向最新记录。原地更新意味着并发读可能读到不一致的数据。 下面的版本校验协议用于检测此问题。

 在提交协议第3阶段， 要修改记录的数据，持有锁的工作线程需要：(a) 更新记录数据； (b) 执行一个内存fence， (c) 存储TID再放锁。  一致性的要求是， 如果并发读看到这个已放锁的记录，它一定会看到新数据和新TID。 步骤(b) 可以保证首先看到新数据。 因为锁是在TID字上， 所以步骤(c) 原子操作设置TID、放锁两个动作。

在提交协议之外，执行中的事务去访问记录数据， 工作线程会(a) 读取TID字， 直到清除锁标识后将其spin； (b) 检查该记录是否为最新版本； (c) 读取数据； (d) 执行一个内存fence， (e) 再次检查TID字。 如果步骤(b) 不成立、或者 TID字在步骤(a)/(e)之间变改了，那么工作线程可以重试或者 回滚 。 

**删除**

Snapshot transactions要求被删除 的记录呆在索引树中。 所以删除操作只是在记录上打了一个缺席的标识，然后将该记录注册在后续的垃圾回收中。客户端视缺席的记录为键缺少， 而在内部， 缺席的记录则认为存在的记录读时需要校验。 因为垃圾回收的问题，对于缺席的记录数据不会做覆盖写操作。 

**插入**

提交协议 的第2 阶段处理了写写冲突，这是由对写记录加锁来实现的。 但是，如果记录不存在 的话，无法加锁。 为解决此问题， 在开始提交协议之前， INSERT操作会插入一个新记录。 

假设插入的键值是k， 如果插入的键值映射到了一个存在的记录，那么插入失败。 否则 ，一个新的记录会插入树中，其状态为absent， 其TID为0， 操作原语为insert-if-absent。 同时该记录会同时写入到读集合和写集合中。 第2阶段的校验操作会确保没有别的事务可以使用该占位。 

提交成功的话， 会对新记录的TID和数值进行正确重写。 如果提交失败， 则将该记录注册为缺席记录，以便后续回收。

### 范围查询和幻读

范围查询因为幻读的问题而变得复杂 。 在扫描查询 的过程中， 该范围内的键值不应该发生增加或者减少。 典型的解法是 next-key locking。 

silo的解法则是利用了底层B树叶子节点上的版本号信息。 它可保证，对树节点的结构修改会引起所有相关节点的版本号变化。 所以，范围扫描的过程中， 除了将所有记录放入读集合中，还 会维护一个集合node-set。 在扫描的过程中，将范围有关的叶子节点及其版本号放入到节点集中。 同理， 在提交第2阶段， 还需要对节点集中所有节点的版本号进行校验是否未发生变化 ， 这样可以保证在扫描过程中没有新的键增加、或者将已有键删除。 

幻读的问题还有可能是因为查找和删除的失败，因为树中并不存在对应 的键值。 无论何种情况， 只要在提交的时候， 键值仍然是不存在于树中的，事务就能够提交。 这种情况下， 只要将可能包含缺少键值的节点放入到节点集中即可。 

需要注意插入是会触发结构型修改。我们需要区分由并发事务引起的结构型修改（必然导致事务回滚）和当前事务引起的修改（不应该引发事务回滚）。 解决方法为：假设记录插入前后的版本号分别为Vold/Vnew， 那么插入操作需要更新节点集，将版本号为Vold的节点更新其版本号为Vnew。 这样， 在校验阶段，版本号不一样的节点会使得事务回滚。 

### 辅助索引

辅助索引的键值为包含主键的记录。 如果辅助索引访问的记录发生变化 的话， 使用辅助索引的事务也会回滚。 

### GC垃圾回收

需要回收的垃圾数据有两类：索引数据和记录数据。 Silo并没有使用引用计数的方法， 而是使用了基于epoch的回收方案RCU。 

在产生垃圾记录的时候， 工作线程会将要回收的对象以其reclamation epoch记录在列表中。 The reclamation epoch is the epoch after which no thread could possibly access the object; once it is reached, we can free the object. 可以使用后台线程或者工作线程自己去做此事， 我们选择使用工作线程来完成， 以此来减少特定线程的数目，以及数据在物理核之间的迁移。 

记录对象不同使用的回收策略不同，但类似。 树节点的回收策略最是简单。 事务过程中释放掉的树节点， 其reclamation epoch为ew(见前面章节)。 可以保证没有线程会再去访问*e* ≤ min *ew* − 1 释放掉的树节点。 推进线程会周期性地检查所有线程的ew值，并将设置一个全局的tree reclamation epoch为min *ew* − 1，那么，小于等于此全局值的垃圾树节点就可以安全地释放掉了。 

### Snapshot Transaction

We support running read-only transactions on recent-past snapshots by retaining additional versions for records.  这些版本形成一个一致性快照，它包含了serial order中某个点前面所有事务的修改，同时不包含该点之后事务的任何修改。 快照对于snapshot transaction和检查点是有用的。 管理快照有两个取巧部分： 确保快照是一致的、完整的， 确保快照内存是可最终回收的。

通过snapshot epochs来提供一致性快照。 snapshot epochs边界与epoch的边界是对齐 的， 所以在serial order中也是一个一致性点。 但是， snapshot epoch比epoch推进得要慢； 这是考虑到生成快照并不是免费的。 论文 中， 一秒钟取一次新快照。 

推进线程会周期性地生成一个全局的snapshot epoch, 每一个工作线程同样也维护着一个本地值； 在事务开始时， 它会设置本地值为当前全局值。 使用此值可以判定记录的可见性。 对于记录r， 相近版本是小于等于本地值的、最大最近的那个。 当一个snapshot transaction完成时， 它不需要检查就可以提交； 它决不会回滚，因为这个快照是一致的，决不会被修改的。 

Read/write transactions must not delete or overwrite record versions relevant for a snapshot. 考虑一个在E时刻提交的事务。 当在阶段3对记录r进行修改或者删除时， 该事务需要比较E对应的快照和记录r.tid对应的快照。 如果两个值相等， 那么该事务可以安全地覆盖此记录， 那么新版本将会在所有的快照中替代旧版本记录。 如果不相等， 则旧版本的记录必须保留，以便当前的或者后续的snapshot transaction的事务访问。 那么，读事务会新加一个记录， 该记录的前向指针字段会记录旧版本记录。  通过此链接， snapshot transaction可以访问到更旧版本的记录 。

当不再有事务访问 时， 记录版本也应该进行释放。 事务生成的快照版本内存，带其快照信息会进行注册。 推进线程会周期性地计算snapshot reclamation epoch， 其值为min *se* − 1。  小于等于该值 的快照版本就可以安全地释放了。 对于前向版本指针，则可不作调整，因为其指向的更旧版本不会再被任何snapshot transaction事务所访问了， 它们更倾向于更新的版本。 

然后删除 操作是要特殊处理的。 删除掉的记录按道理 应该从树上取下来， 但是snapshot transaction又需要此记录来找到对应 的更旧版本。 所以，提交协议创建了一个缺席的记录，其状态位为已删除， 数据部分则直接存储相关键值。 再将该记录带回收信息snap(E)进行注册。 当到达回收时间点时， 清理线程会修改树、移除对此记录的引用（使用键值）， 再将该记录基于tree reclamation epoch进行注册； 该记录并不能够 立即释放， 因为可能有并发的事务在访问它。 然而， 如果缺席的记录被一个后来的插入记录所占据时， 该树不应该再被修改。 清理线程需要检查该记录是否依然版本最新， 若不是则简单忽略此记录。 不再需要进一步动作， 因为插入事务会标识该记录进行后续回收。 

### 持久化

Since epoch boundaries are consistent with the serial order, Silo treats whole epochs as the durability commit units. 

所有属于epoch e的事务的日志写在一起。在故障之后， 系统将会检查日志，然后找到durable epoch D,   此为成功记日志的事务所在的最近epoch。 然后系统恢复所有小于等于D的事务，其他的不管。 不恢复再多这一点很重要： 尽管整个epoch是一致的， 但是一个epoch之内的serial order却是无法从日志中恢复的。  所以， 重放一个epoch中事务的一部分日志，会产生非一致的状态。 这同样意味着， epoch周期直接影响着事务提交 的平均时延。 

Silo中有一定数量的线程用于处理日志， 每一个日志线程负责不同的工作纯种子集。 每一个线程写入的日志位于不同的磁盘上。 

当一个工作线程提交一个事务时， 它会创建一个日志记录，包含了事务TID、记录信息（表、键值、数值）。 日志直接以磁盘格式记录在本地内存缓存中。 当缓存满或者新epoch开始的时候，工作线程会把本地缓存发布到对应的日志线程上， 然后通过写入一个全局变量ctid w来发布其最后的已提交TID。 

日志线程以持续循环执行，每一个迭代都会计算出d之前的事务日志是可提交的。  在迭代开始之前， 首先会计算出该日志线程对应的工作线程的最小值t = min(ctid w); 从此值计算出本地durable epoch d = epoch(t) -1。 所有小于等于d的、与其关注的工作线程事务必然已经发布， 日志是完整的。 日志线程将所有日志缓存、再加一条包含d信息的数据追加到日志文件 末尾， 并且等待写盘的结束。 之后， 日志线程将d发布到对应 的全局变量dl上（每个日志线程对应一个变量）， 之后日志缓存可以被工作线程所重复使用。 

有一个线程会周期性地计算并发布全局durable epoch值D=min(dl)。 所以小于等于D的事务则必定日志持久化。 那么，小于等于D的事务就可以给客户端以响应了。

Silo使用记录级的redo日志，而非undo日志或者操作日志。 

恢复时， 每一个日志线程会计算出最大最近的dl值， 再计算出D = min(dl)， 然后回放日志。 相同记录的日志必须以TID的顺序进行回放，以确保结果等于最新版本。 不同的记录可以并发回放。

Silo并没有实现完整的检查点机制。

## 下一步阅读论文

索引相关的论文

* Cache craftiness for fast multicore key-value storage
* Cache- conscious concurrency control of main-memory indexes on shared-memory multiprocessor systems 
* PALM: Parallel architecture-friendly latch-free modifications to B+ trees on many-core processors
* Efficient locking for concurrent operations on B-trees
* The Bw-tree: A B-tree for new hardware
* LLAMA: A cache/storage subsystem for modern hardware

锁相关的论文

- A key-value locking method for concurrency control of multiaction transactions operating on B-tree indexes

垃圾回收

- Practical lock freedom
- Performance of memory reclamation for lockless syn- chronization 

并发控制相关

- On optimistic methods for concurrency control 
- High-performance concurrency control mechanisms for main-memory databases
- A critique of ANSI SQL isolation levels
- Low over- head concurrency control for partitioned main memory databases 
- The notions of consistency and predicate locks in a database system 
- Distributed optimistic concurrency control with reduced rollback
- Observations on optimistic concurrency con- trol schemes 
- Optimistic concurrency control revisited
- Hekaton: SQL Server’s memory-optimized OLTP engine 

分区相关

- Skew-aware automatic database partitioning in shared-nothing
- PLP: page latch-free shared-everything OLTP
- Spanner: Google’s globally- distributed database 