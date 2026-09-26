# 数据库事务隔离级别与 MVCC：并发访问下如何保证数据读得对

2026 年 09 月 15 日 16 时 25 分 30 秒

数据库最迷人也最容易出问题的场景，是多个事务同时读写同一份数据。一个转账操作在业务上要求"扣减"和"入账"要么都成功、要么都失败，而在它执行的同时，报表查询、用户下单可能正在读取相关数据。为了让并发事务互不干扰又不至于完全排队，数据库引入了事务隔离级别；而 MySQL InnoDB 又通过 MVCC（多版本并发控制），让普通查询在不加锁的情况下也能读到一致的数据。本文将从事务的 ACID 特性出发，讲清脏读、不可重复读、幻读的成因，四种隔离级别的含义，以及 MVCC 借助版本链与 ReadView 实现一致性读的底层原理。

## 一、事务与 ACID：本文聚焦"隔离性"

事务是由一条或多条 SQL 组成的、不可分割的逻辑工作单元，其特性常被概括为 **ACID**：

- **原子性（Atomicity）**：事务内操作要么全部成功，要么全部回滚，主要由 undo log 支撑；
- **一致性（Consistency）**：事务前后数据满足业务约束，是其他特性共同保障的最终目标；
- **隔离性（Isolation）**：并发事务之间相互隔离、互不干扰；
- **持久性（Durability）**：提交后的数据修改永久保存，主要由 redo log 保障。

本文聚焦其中的隔离性。它的难点在于：隔离得越严格，数据越正确，但并发能力往往越差；放宽隔离，并发上去了，又可能出现各种"读异常"。隔离级别本质上是在**正确性与并发性**之间做权衡。

## 二、并发事务可能出现的三类读问题

设想两个事务同时运行，按照严重程度，可能出现以下问题。

**脏读（Dirty Read）**：一个事务读到了另一个事务**尚未提交**的修改。若对方随后回滚，本次读到的就是一条从未真正存在过的数据，基于它做判断极其危险。

**不可重复读（Non-Repeatable Read）**：同一事务内，两次读取同一行，结果却不同，因为中间有另一个事务修改并提交了该行。它侧重的是**已有数据被修改**导致前后不一致。

**幻读（Phantom Read）**：同一事务内，两次执行相同的范围查询，第二次多出或少了若干行，如同出现"幻影"，因为中间有事务**插入或删除**并提交。它侧重的是**结果集行数变化**。

此外还有**丢失更新**问题：两个事务同时基于旧值修改同一行，后提交的覆盖先提交的结果，使一次更新"丢失"，通常需要乐观锁（版本号）或加锁来避免。

## 三、四种隔离级别：在正确性与并发间取舍

SQL 标准定义了四级隔离，级别越高、允许的异常越少、并发越低：

| 隔离级别 | 脏读 | 不可重复读 | 幻读 |
| --- | --- | --- | --- |
| 读未提交（Read Uncommitted） | 可能 | 可能 | 可能 |
| 读已提交（Read Committed） | 避免 | 可能 | 可能 |
| 可重复读（Repeatable Read） | 避免 | 避免 | 可能（标准层面） |
| 串行化（Serializable） | 避免 | 避免 | 避免 |

需要注意"标准"与"具体实现"的差异：

- **读未提交**几乎不做隔离，实际很少使用；
- **读已提交（RC）**只允许读到已提交数据，是 Oracle、PostgreSQL 等的默认级别；
- **可重复读（RR）**保证同一事务内多次读同一行结果一致，是 **MySQL InnoDB 的默认级别**，并通过后续介绍的机制在很大程度上也防止了幻读；
- **串行化**通过强制事务串行执行消除所有异常，但并发能力最差，仅在极少数强一致场景使用。

## 四、纯加锁方案的局限

最直观的隔离手段是加锁：读操作加共享锁、写操作加排他锁，必要时锁住范围，直到事务结束才释放（两阶段锁协议）。

这种方式确实能保证正确性，但会带来明显的读写阻塞：一个长事务修改某行并持有写锁时，所有想读这行的事务都被迫排队；范围更新时锁住大片数据，更会严重拖累并发。串行化级别就是这种思路的极端体现。

理想的状态是**读不阻塞写、写不阻塞读**：查询能读到一个一致的历史版本，而不必等待当前写入完成。InnoDB 实现这一目标的关键，正是 **MVCC（多版本并发控制）**。

## 五、MVCC：用多版本实现无锁一致性读

MVCC 的核心思想是：数据不只保留最新值，而是通过版本链保存其在不同时刻的多个版本，查询依据规则找到对自己"可见"的那一版。它由三个要素协同实现。

**隐藏字段**：每行数据除业务字段外，还隐含两个关键字段——`trx_id`（最近一次修改该行的事务 ID，事务 ID 随事务开始而递增）和 `roll_pointer`（回滚指针，指向 undo log 中该行的上一个版本）。

**undo log 版本链**：每次修改，旧版本不会立即删除，而是被记入 undo log，并通过回滚指针串联成一条由新到旧的版本链，每个版本都记录了产生它的事务 ID。

**ReadView（读视图）**：事务进行查询时生成的一份"可见性快照"，其中包含生成时所有**活跃（尚未提交）事务**的 ID 列表 `m_ids`，以及最小活跃事务 ID、下一个将分配的事务 ID 和当前事务 ID。判断某版本是否可见的规则可概括为：

- 若该版本的 `trx_id` 等于当前事务 ID，说明是自己改的，可见；
- 若 `trx_id` 小于活跃列表中的最小值，说明修改在快照生成前已提交，可见；
- 若 `trx_id` 大于等于快照上界，说明是快照之后才开始的事务，不可见；
- 若 `trx_id` 处于活跃事务列表中，说明生成快照时该事务尚未提交，不可见；
- 不可见时，就沿回滚指针取上一版本，重复判断，直到找到可见版本。

**RC 与 RR 的关键差异，正来自 ReadView 的生成时机**：

- 在 **RC** 下，**每次**普通 SELECT 都生成新的 ReadView，因此总能看到此前已提交的最新数据——这解释了不可重复读；
- 在 **RR** 下，事务**第一次**普通 SELECT 时生成 ReadView 并在整个事务期间**复用**，因此无论其他事务如何提交修改，看到的始终是同一份快照——这就实现了可重复读。

## 六、快照读、当前读与幻读的最终解决

MVCC 作用的是**快照读**，即普通的 SELECT，它读的是版本链中的历史快照，不加锁。在 RR 下，由于快照固定，其他事务新插入的行不会出现在后续快照读中，因此快照读天然避免了幻读。

但还有一类**当前读**：`SELECT ... FOR UPDATE`、`SELECT ... LOCK IN SHARE MODE` 以及 `UPDATE`、`DELETE`，它们必须读取**最新版本**并对其加锁，否则可能基于旧数据做修改。对于当前读，InnoDB 通过锁机制防止幻读：

- **记录锁（Record Lock）**锁住具体索引记录；
- **间隙锁（Gap Lock）**锁住索引记录之间的空隙，防止其他事务在该范围插入新行；
- **Next-Key Lock** 是记录锁与间隙锁的组合，既锁记录又锁区间。

RR 级别下，当前读默认使用 Next-Key Lock 扫描并锁定范围，使其他事务无法在范围内插入"幻影行"，从而在当前读场景也解决了幻读。这正是 InnoDB 的 RR 比 SQL 标准定义"更强"的地方。

## 七、工程实践建议

**RR 还是 RC**：RR 一致性最好，但间隙锁在范围查询、插入热点场景可能降低并发、增加死锁概率；不少互联网公司将隔离级别设置为 RC，配合应用层校验和乐观锁，换取更高并发与更简单的锁行为，应结合业务选择。

**警惕长事务**：长事务会长期持有 ReadView，使版本链上的旧版本无法被清理，导致 undo log 膨胀、空间占用上升，并可能长期持锁。应避免在事务中夹杂外部调用、人为等待，并设置事务超时。

**应对丢失更新**：对"读—改—写"型并发，可在表中加版本号字段，更新时校验版本（乐观锁），或显式使用当前读加行锁。

**隔离级别并非越高越好**：每一级都是正确性与并发能力的交换，理解其底层是加锁还是 MVCC，才能做出符合业务的取舍。

## 写在最后

事务隔离并不是一道简单的选择题，而是一整套由锁与多版本共同支撑的并发控制机制：隔离级别定义了"允许看到什么"，MVCC 用版本链和 ReadView 让普通查询无锁地读到一致快照，Next-Key Lock 则为必须读取最新数据的当前读补上了防幻读的锁。理解了快照读与当前读的分工、以及 RC 与 RR 在 ReadView 生成时机上的差异，那些关于"为什么两次查询结果不同""为什么插入被阻塞"的困惑，就都能在这套原理中找到确切答案。



`https://xxn.xxn.net.cn`  
`https://yckz.yckz.net.cn`  
`https://yqk.yqk.org.cn`  
`https://ysgc.ysgc.net.cn`  
`https://yydm.yydm.net.cn`  
`https://zwzm.zwzm.com.cn`  
`https://cgw.cgw.sh.cn`  
`https://aify.aify.org.cn`  
`https://cgwz.cgwz.com.cn`  
`https://fyai.fyai.net.cn`  
`https://hjcg.hjcg.org.cn`  
`https://hlbd.hlbd.net.cn`  
`https://hls.hls.ac.cn`  
`https://hlznl.hlznl.net.cn`  
`https://htsp.htsp.net.cn`  
`https://jssp.jssp.net.cn`  
`https://mdsp.mdsp.net.cn`  
`https://mgsp.mgsp.org.cn`  
`https://mrcg.mrcg.org.cn`  
`https://mtsp.mtsp.ac.cn`  
`https://nsdm.nsdm.com.cn`  
`https://nspk.nspk.com.cn`  
`https://ppyy.ppyy.org.cn`  
`https://pzsp.pzsp.net.cn`  
`https://trmh.trmh.com.cn`  
`https://trxs.trxs.org.cn`  
`https://xjmh.xjmh.com.cn`  
`https://xxmh.xxmh.com.cn`  
`https://xxsp.xxsp.net.cn`  
`https://zkw.zkw.ac.cn`  
`https://zxai.zxai.org.cn`  
`https://ainy.ainy.ac.cn`  
`https://ailt.ailt.net.cn`  
`https://aizx.aizx.net.cn`  
`https://dydq.dydq.net.cn`  
`https://dytt.dytt.ac.cn`  
`https://dyttw.dyttw.org.cn`  
`https://fldm.fldm.com.cn`  
`https://hpw.hpw.org.cn`  
`https://hzdm.hzdm.com.cn`  
`https://kjw.kjw.org.cn`  
`https://kxw.kxw.net.cn`  
`https://mjtt.mjtt.ac.cn`  
`https://mjw.meijuw.com.cn`  
`https://okdm.okdm.com.cn`  
`https://okdmw.okdmw.net.cn`  
`https://rbdy.rbdy.com.cn`  
`https://rrmj.rrmj.net.cn`  
`https://tmny.tmny.net.cn`  
`https://ttdm.ttdm.ac.cn`  
`https://ttys.ttys.org.cn`  
`https://yhmh.yhmh.net.cn`  
`https://yhsp.yhsp.net.cn`  
`https://yhtv.yhtv.net.cn`  
`https://yhys.yhys.net.cn`  
`https://yhyy.yhyy.org.cn`  
`https://bgjj.bgjj.com.cn`  
`https://ady.ady.net.cn`  
`https://blds.blds.com.cn`  
