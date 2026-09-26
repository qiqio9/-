# LSM 树：写密集型存储引擎的原理与 B+ 树的对照

2026 年 08 月 05 日 11 时 15 分 40 秒

谈到数据库索引，B+ 树几乎是默认答案，它读取稳定、范围查询方便，但在写入密集场景下会暴露随机写、页分裂和刷脏页的成本。另一类被 RocksDB、HBase、Cassandra、TiDB 广泛采用的存储结构——**LSM 树（Log-Structured Merge-Tree，日志结构合并树）**——反其道而行：把所有写入转化为顺序追加，再在后台逐层合并，用读放大和合并开销换取极高的写入吞吐。本文将从 B+ 树的写入瓶颈讲起，梳理 MemTable、SSTable、Compaction 的工作机制，以及读放大、写放大、空间放大之间的权衡。

## 一、B+ 树在写入时的瓶颈

B+ 树的数据按页组织、按键排序，这对查询非常友好，但写入时要维持"有序"：

- 更新分散在不同位置的行，产生**随机写**；机械硬盘上随机写依赖寻道，速度远低于顺序写；
- 页写满后发生**页分裂**，需要移动数据、产生碎片；
- 缓冲池中的脏页要按策略刷盘，还涉及 WAL 保证崩溃恢复；
- 在 SSD 上，随机小写还会带来硬件层面的写放大。

对于"写多读少、允许短暂不一致"的场景（如时序、消息、海量更新），B+ 树并非最优，LSM 树正是针对这一痛点设计。

## 二、LSM 的核心思想：先写内存与日志，再批量落盘

LSM 不就地更新已有数据，而是把修改当作新记录追加，一次写入的典型路径如下：

1. **先写 WAL（预写日志）**：把修改顺序记录到日志文件，保证宕机重启后内存中尚未落盘的数据可以恢复；
2. **写入内存中的 MemTable**：它采用有序结构（通常是跳表，正是前文介绍的数据结构），支持快速插入、查找和有序遍历；
3. 当 MemTable 写满，将其冻结为 **Immutable MemTable**（只读），同时新建一个 MemTable 继续承接写入；
4. 后台任务把冻结的 MemTable **顺序 flush** 到磁盘，生成 **SSTable** 文件。

由于全程是顺序追加，LSM 把 B+ 树的随机写转化为顺序写，写入吞吐通常高得多，尤其适合高并发写入。

## 三、SSTable 与层级组织

**SSTable（Sorted String Table）**是不可变、按键排序的键值文件，内部按块组织并带索引，通常还为每个文件配备**布隆过滤器**，用于快速判断某个 key 是否可能存在。

随着数据不断 flush，磁盘上会积累大量 SSTable，它们按不同策略分层管理，常见有两类：

- **大小分层（Tiered，STCS）**：把大小相近的 SSTable 攒在一起再合并，合并次数少、写放大低，但同层可能有重叠、空间和读放大较高；
- **分层（Leveled，LCS）**：除最上层外，每一层内部数据互不重叠、有序覆盖整个键空间，读放大低、空间占用小，但合并更频繁、写放大较高。

## 四、读取：如何在多个版本中找到值

LSM 的数据可能同时存在于 MemTable、Immutable 和多层 SSTable 中，且同一 key 可能有多个版本。一次读取按"**从新到旧**"依次查找：

1. 先查当前 MemTable，再查 Immutable MemTable；
2. 然后查最新一层的 SSTable，逐层向下，直到找到该 key；
3. 借助每个 SSTable 的**布隆过滤器和块索引**快速排除不含该 key 的文件；
4. 若查到的是删除标记（**tombstone**），则视为不存在。

这正是 LSM 的**读放大**来源：最坏情况下要访问多个文件，而 B+ 树通常沿固定 3~4 层路径即可命中。

## 五、Compaction：合并、去重与删除

后台 **Compaction（合并）**是 LSM 的核心维护动作：把多个 SSTable 读取、归并排序后写成新文件，在这一过程中：

- 丢弃被覆盖的旧版本，只保留最新值；
- 真正移除被 tombstone 标记删除的数据；
- 控制文件数量，维持层级结构。

Compaction 会消耗 IO 和 CPU，且与在线读写争抢资源，因此需要限速、错峰。它直接决定了 LSM 的三类放大：

- **写放大**：同一份数据在多次合并中被反复重写；
- **读放大**：一次查询读取的文件/块数量；
- **空间放大**：磁盘上旧版本、碎片导致的额外占用。

这三者通常此消彼长（也称 RUM 权衡），没有一种策略能同时最优，需要按业务是写敏感还是读敏感来选择合并方式。

## 六、LSM 树与 B+ 树的对照

| 维度 | B+ 树 | LSM 树 |
| --- | --- | --- |
| 写入方式 | 随机写、页分裂 | 顺序追加、后台合并 |
| 读取路径 | 路径稳定、3~4 层 | 可能查多层、读放大 |
| 删除/更新 | 就地更新 | 追加标记、合并时清理 |
| 空间 | 较紧凑 | 合并前有旧版本膨胀 |
| 崩溃恢复 | WAL | WAL |
| 适合场景 | 读多写少、事务 | 写多读少、高吞吐 |

简单说：B+ 树把成本主要放在写入时维持有序，LSM 把成本推迟到后台合并与读取时。

## 七、典型系统与实践建议

- **LevelDB / RocksDB**：嵌入式 LSM 引擎，RocksDB 被大量数据库和平台作为底层存储；
- **HBase / Cassandra**：典型的 LSM 分布式存储，前者偏分层、后者提供多种压缩策略；
- **TiDB / TiKV**：基于 RocksDB（后演进为自研引擎）配合 Multi-Raft 实现分布式一致性。

实践调优主要围绕：选择合适的 Compaction 策略、设置 MemTable 与块缓存大小、启用布隆过滤器、控制后台合并速率，并关注 compaction 对延迟的影响。需要注意，Kafka 这类纯追加日志虽然也是顺序写，但不做合并归并，并不是 LSM。

## 写在最后

LSM 树与 B+ 树代表了存储引擎的两种哲学：B+ 树用"写入时维持有序"换取稳定高效的读取，LSM 则用"顺序追加 + 后台合并"换取极高写入吞吐，并在读、写、空间三种放大之间灵活权衡。MemTable 的跳表、SSTable 的布隆过滤器、WAL 的崩溃恢复，又把前面介绍过的多种数据结构组合成一个整体。理解了 LSM 的追加、分层与合并逻辑，就能读懂 RocksDB、HBase、TiDB 等系统为何在写密集场景下表现出色，也能在选型时看清它相对 B+ 树真正付出的代价。




`https://shtv.shtv.org.cn`  
`https://shsp.shsp.ac.cn`  
`https://ncyy.ncyy.net.cn`  
`https://ncsp.ncsp.ac.cn`  
`https://hxc.hxc.net.cn`  
`https://hpzxgk.hpzxgk.com.cn`  
`https://ggyy.ggyy.ac.cn`  
`https://flsp.flsp.net.cn`  
`https://flj.flj.org.cn`  
`https://57kpw.57kpw.com.cn`  
`https://51sp.51sp.ac.cn`  
`https://51mh.51mh.ac.cn`  
`https://51hlcg.51hlcg.com.cn`  
`https://51dm.51dm.com.cn`  
`https://zxcg.zxcg.org.cn`  
`https://yqc.yqc.net.cn`  
`https://xrksp.xrksp.com.cn`  
`https://wycgw.wycgw.net.cn`  
`https://wycg.wycg.org.cn`  
`https://qzsp.qzsp.net.cn`  
`https://qqcsp.qqcsp.com.cn`  
`https://qpgyy.qpgyy.org.cn`  
`https://ppsp.ppsp.net.cn`  
`https://ppp.ppp.sh.cn`  
`https://mtzx.mtzx.bj.cn`  
`https://mtwz.mtwz.net.cn`  
`https://mtw.mtw.sh.cn`  
`https://mgsp.muguasp.org.cn`  
`https://lms.lms.sh.cn`  
`https://llsp.llsp.net.cn`  
`https://jycsp.jycsp.org.cn`  
`https://jyc.jyc.ac.cn`  
`https://hlzx.hlzx.net.cn`  
`https://hlw.hlw.sh.cn`  
`https://hlsq.hlsq.ac.cn`  
`https://hlmrds.hlmrds.com.cn`  
`https://hlcgw.hlcgw.net.cn`  
`https://hlcg.hlcg.org.cn`  
`https://fjbns.fjbns.net.cn`  
`https://cgzx.cgzx.net.cn`  
`https://cgtt.cgtt.net.cn`  
`https://cgsp.cgsp.org.cn`  
`https://cghlw.cghlw.com.cn`  
`https://cghl.cghl.org.cn`  
`https://cgblw.cgblw.net.cn`  
`https://cgbl.cgbl.net.cn`  
`https://cgaw.cgaw.net.cn`  
`https://cg51.cg51.net.cn`  
`https://blw.blw.ac.cn`  
`https://awcg.awcg.net.cn`  
`https://91cgw.91cgw.net.cn`  
`https://91cg.91cg.net.cn`  
`https://51hlw.51hlw.net.cn`  
`https://51kp.51kp.net.cn`  
`https://52cg.52cg.net.cn`  
`https://www.yingtaoshipin.autos`  
`https://www.tydm.autos`  
