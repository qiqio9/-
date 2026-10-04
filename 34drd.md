# 数据库锁原理：表锁行锁、间隙锁、临键锁、死锁与乐观锁、悲观锁

2026 年 10 月 04 日 19 时 27 分 28 秒

当多个事务同时修改同一行数据，如果没有约束，就可能出现"两个事务都基于旧值更新、后写覆盖先写"的丢失更新。数据库必须用**锁**让并发写在关键资源上排队。InnoDB 的锁机制不仅是事务隔离级别的实现基础，也与 MVCC 紧密配合：普通查询走多版本、当前读写才加锁。行锁为什么必须走索引、间隙锁和临键锁如何防止幻读、死锁是怎么产生又如何被检测、乐观锁与悲观锁怎么选。本文将系统讲清这些问题。

## 一、为什么数据库需要锁

并发事务同时写同一数据，会造成脏写和丢失更新。锁通过对资源加互斥约束，保证同一时刻写操作的正确性。

要先建立一个整体分工：

- **MVCC** 主要解决"快照读"，让普通 SELECT 读历史版本、不加锁；
- **锁**主要解决"当前读"和写操作，让事务读到最新数据并阻止他人修改；
- 二者共同实现了可重复读等隔离级别。

锁的粒度则是并发能力与加锁开销之间的权衡。

## 二、锁的粒度：表锁与行锁

- **表锁**：开销小、加锁快、不会死锁，但并发度低，常见于 DDL 和 MyISAM；
- **行锁**：并发度高，但开销大、管理复杂、可能死锁，是 InnoDB 的特性。

一个极其重要的结论：**InnoDB 的行锁是加在索引记录上的**。如果更新或查询没有走索引，InnoDB 只能扫描并对经过的记录加锁，极端情况下几乎等同于锁全表，并发骤降。因此"更新条件必须命中索引"是使用行锁的前提。

## 三、共享锁、排他锁与意向锁

- **共享锁（S，读锁）**：多个事务可同时持有，用于 `LOCK IN SHARE MODE`；
- **排他锁（X，写锁）**：与其他锁互斥，更新、删除和 `FOR UPDATE` 使用；
- **意向锁（IS/IX，表级）**：事务在给行加 S/X 锁前，先在表级声明意向。它的作用是让数据库**快速判断表里是否已有行锁**，而不必逐行扫描；意向锁之间彼此兼容，只与表级 S/X 冲突。

## 四、记录锁、间隙锁与临键锁

这是 InnoDB 防止幻读的关键，也是最容易被误解的部分：

- **记录锁（Record Lock）**：锁定单条索引记录；
- **间隙锁（Gap Lock）**：锁定索引记录之间的"空隙"（不含记录本身），阻止其他事务向间隙插入数据；
- **临键锁（Next-Key Lock）**：记录锁 + 前面间隙的组合，锁定一个左开右闭区间，是可重复读级别下的默认行锁形式。

细节上：唯一索引等值查询且命中时，临键锁会退化为记录锁；范围查询则保留临键锁。间隙锁之间并不互斥——多个事务可对同一间隙加锁，但它们都会阻止插入，从而共同防止"幻读"。

## 五、两阶段锁协议

InnoDB 遵循**两阶段锁（2PL）**：事务在执行过程中按需加锁，但所有锁要等到 COMMIT 或 ROLLBACK 时才统一释放。

这给了一个实用启示：应把最可能产生冲突的热点更新放在事务的靠后位置，以缩短锁的实际持有时间，减少与其他事务的冲突和死锁概率。

## 六、死锁

**死锁**指两个或多个事务互相持有对方需要的锁、形成循环等待。

- 常见成因：事务以不同顺序访问多行、范围查询与插入相互等待、间隙锁冲突；
- InnoDB 默认开启**死锁检测**，通过构建等待图（wait-for graph）发现循环，并主动回滚代价较小的事务来打破僵局；
- 若检测关闭，则靠 `innodb_lock_wait_timeout`（锁等待超时）兜底。

避免死锁的常见做法：固定多表/多行的加锁顺序、缩短事务、必要时降低隔离级别、保证走索引、把大批量操作拆小。死锁在高并发下并不罕见，应用层应当捕获并重试，而不是把它当成程序崩溃。

## 七、乐观锁与悲观锁

- **悲观锁**：假设冲突大概率发生，先用 `SELECT ... FOR UPDATE` 加锁再修改，适合写多、冲突频繁的场景；
- **乐观锁**：假设冲突较少，更新时带版本号条件（`where version=?`），影响行数为 0 说明被他人改过、需要重试；它与 Java 的 CAS、分布式锁思想相通。

冲突频率决定选型：冲突多用悲观锁避免反复重试，冲突少用乐观锁减少加锁开销。

## 八、快照读与当前读

- **快照读**：普通 SELECT，基于 MVCC 读历史版本，不加锁；
- **当前读**：`FOR UPDATE`、`LOCK IN SHARE MODE` 以及 INSERT/UPDATE/DELETE，读取最新版本并加锁。

在可重复读级别下，**MVCC 让快照读看到一致的历史视图，临键锁让当前读无法插入"幻影行"**，两者配合才完整实现了防幻读。

## 九、实践建议与排查

- 更新和加锁查询务必走索引，用 EXPLAIN 确认，避免锁范围扩大；
- 通过 `SHOW ENGINE INNODB STATUS` 查看最近死锁日志和锁等待，监控锁等待时间；
- 缩短事务、避免大事务和长事务，减少锁的生命周期；
- 对热点行可合并更新、串行化处理或引入队列，而不是放任高并发争抢；
- 保持业务幂等，为死锁重试和异常恢复留好兜底。

## 写在最后

数据库锁是事务并发正确性的底线：表锁与行锁决定并发粒度，S/X 锁与意向锁协调读写，记录锁、间隙锁和临键锁在索引上构建起防止幻读的区间，两阶段锁规定了锁的释放时机，而死锁检测、乐观锁与悲观锁则提供了冲突的处理方式。把它与 MVCC、隔离级别、Java 锁和分布式锁对应起来，就能看清"一条更新在数据库里到底锁住了什么、锁多久、为何会和别人互相等待"，从而在高并发写场景下既保证正确，又不至于因锁范围失控而拖垮系统。





`https://www.51cg.gd.cn`  
`https://www.51dm.sh.cn`  
`https://www.51kp.ac.cn`  
`https://www.51sp.sh.cn`  
`https://www.51manhua.bj.cn`  
`https://www.57kpw.org.cn`  
`https://www.91cg.sh.cn`  
`https://www.91cy.ac.cn`  
`https://www.91dm.org.cn`  
`https://www.91kp.net.cn`  
`https://www.91mh.org.cn`  
`https://www.91sp.ac.cn`  
`https://www.acedm.org.cn`  
`https://www.acgmh.net.cn`  
`https://www.agedm.sh.cn`  
`https://www.aqd.hk.cn`  
`https://www.aqdlt.gd.cn`  
`https://www.awtv.org.cn`  
`https://www.bbys.ac.cn`  
`https://www.bdys.net.cn`  
`https://www.bkmh.ac.cn`  
`https://www.lfdm.bj.cn`  
`https://www.kpwz.ac.cn`  
`https://www.kmdsp.net.cn`  
`https://www.kkys.sh.cn`  
`https://www.kbmh.ac.cn`  
`https://www.jpp.net.cn`  
`https://www.jmtt.gd.cn`  
`https://www.jinmantt.org.cn`  
`https://www.jmmh.hk.cn`  
`https://www.hzdm.ac.cn`  
`https://www.htsp.gz.cn`  
`https://www.hptx.org.cn`  
`https://www.hmyyy.cn`  
`https://www.hjw.bj.cn`  
`https://www.hjsq.ac.cn`  
`https://www.hjsp.org.cn`  
`https://www.hjlt.com.cn`  
`https://www.hhmh.sh.cn`  
`https://www.gzys.bj.cn`  
`https://www.gczxgk.ac.cn`  
`https://www.gcsp.org.cn`  
`https://www.fqys.ac.cn`  
`https://www.fqsp.ac.cn`  
`https://www.fcmh.net.cn`  
`https://www.fcdm.gd.cn`  
`https://www.dyzxgk.ac.cn`  
`https://www.dyw.ac.cn`  
`https://www.dytt.gd.cn`  
`https://www.dydq.ac.cn`  




