# Redis 核心原理：数据结构、对象编码、RDB/AOF 持久化与内存淘汰

2026 年 06 月 28 日 11 时 30 分 45 秒

Redis 几乎是后端开发最常打交道的中间件，缓存、分布式锁、限流、排行榜、会话存储都离不开它。但很多人对它的理解停留在"一个放在内存里的大 Map"。实际上，Redis 的高性能来自内存、单线程模型、IO 多路复用和一系列为不同数据规模精心设计的底层编码；它还通过 RDB、AOF 在内存与磁盘之间平衡性能与数据安全，并用过期和淘汰策略管理有限内存。本文将剖析 Redis 为什么快、五种基本类型背后的编码、键过期与内存淘汰机制，以及 RDB/AOF 持久化的原理与取舍。

## 一、Redis 为什么快

Redis 的高吞吐由几个因素共同决定：

- **纯内存操作**：数据主要在内存，避免磁盘 IO；
- **单线程处理命令**：核心命令在一个线程内顺序执行，没有多线程锁竞争和上下文切换；注意 Redis 6.0 起把**网络读写**改为多线程，命令执行仍为单线程；
- **IO 多路复用**：基于 epoll 等机制用单线程管理大量连接（前文已讲）；
- **高效的底层数据结构**：针对小数据和大数据采用不同编码。

正因为命令单线程顺序执行，像 `KEYS`、大 value 操作这类阻塞命令会卡住整个实例，生产环境应避免。

## 二、五种基本类型与底层编码

Redis 对外暴露 String、List、Hash、Set、ZSet 等类型，内部采用"对象 + 编码"的双层设计，会根据数据量自动切换编码：

- **String**：底层用 **SDS（简单动态字符串）**，记录长度和空闲空间，二进制安全、支持高效扩容，避免 C 字符串的缺陷；
- **List**：采用 **quicklist**（双向链表结合压缩结构），兼顾两端操作和内存占用；
- **Hash**：字段少时用紧凑的 **listpack/ziplist** 连续存储以省内存，超过阈值转为 **hashtable**；
- **Set**：全是小整数时用 **intset**，否则用 hashtable；
- **ZSet**：元素少时用 listpack，数据量大时用**跳表（skiplist）配合哈希表**——哈希表用于按成员取分、跳表支持按分数范围查询，这正是前文跳表原理的典型应用。

这种设计的思想是：**小数据用紧凑编码省内存，大数据用复杂结构保性能**，并在达到阈值时自动转换。

## 三、键的过期策略

设置 TTL 后，Redis 并不会为每个键挂一个精确定时器（那样开销太大），而是组合两种方式：

- **惰性删除**：访问键时检查是否过期，过期才删；
- **定期删除**：周期性随机抽样一批带过期的键检查，控制删除量。

两者配合在 CPU 与内存间折中；若大量过期键长期不被访问、抽样又没覆盖，就需要靠内存淘汰兜底。

## 四、内存淘汰机制

当内存达到 `maxmemory` 上限，Redis 按配置的策略淘汰键：

- **noeviction**：不淘汰，写操作直接报错（默认）；
- **allkeys-random / volatile-random**：随机淘汰，或只在设了过期的键中随机；
- **volatile-ttl**：优先淘汰剩余时间最短的键；
- **allkeys-lru / volatile-lru**：淘汰最久未使用的键；Redis 实现的是**近似 LRU**——通过少量采样而非维护全局链表，以很小代价逼近 LRU 效果；
- **LFU（4.0 起）**：按访问频率淘汰，比单纯看时间更适合周期性热点。

实践中纯缓存常用 allkeys-lru，需要保留某些永久键时用 volatile-* 策略。

## 五、RDB 快照持久化

**RDB** 在某一时刻把内存数据生成紧凑的二进制快照：

- 文件小、恢复速度快，适合备份和灾难恢复；
- 通过 `BGSAVE` **fork 子进程**完成写盘，利用操作系统的**写时复制（COW）**，父进程继续提供服务；
- 缺点是两次快照之间宕机会丢失这段时间的数据，fork 在内存很大时也有开销。

## 六、AOF 持久化

**AOF（追加式文件）**记录每一条写命令：

- 数据更安全、文件可读，宕机丢失少；
- 刷盘策略 `appendfsync` 有 `always`（每条都刷、最安全最慢）、`everysec`（每秒一次，默认，兼顾性能）、`no`（交给系统）；
- AOF 会定期**重写**：fork 进程用最少命令重建当前数据，压缩不断膨胀的日志；
- 缺点是文件较大、恢复比 RDB 慢。

Redis 4.0 起支持**混合持久化**：重写时前半段用 RDB 格式保存全量、后半段追加增量命令，兼顾恢复速度与安全性。

## 七、对比与实践建议

- **RDB** 适合快速恢复和定期备份但可能丢数据，**AOF** 更安全但体积大、恢复慢，生产中常二者结合；
- 持久化不等于备份，重要数据仍需把快照异地保存；
- 避免大 key、热 key 和阻塞命令，监控内存使用率与淘汰、过期命中情况；
- 合理设置 maxmemory 和淘汰策略，理解缓存穿透、击穿、雪崩以及分布式锁在 Redis 上的行为，才能用好而非"踩中"它。

## 写在最后

Redis 的强大并非只靠"内存快"这一个字：SDS、quicklist、listpack、intset 和跳表等编码在小数据与大数据之间动态取舍，单线程加 IO 多路复用简化了并发模型，惰性与定期删除、近似 LRU/LFU 管理着有限内存，而 RDB 与 AOF（乃至混合持久化）则在性能与数据安全之间提供不同档位。理解了这些内部机制，再去看缓存、限流、分布式锁等上层用法，就能明白它们在什么条件下高效、在什么情况下会失效，从而把 Redis 用得既快又稳。



`https://www.91wz.autos`  
`https://www.bslt.autos`  
`https://www.mfyjwzmf.autos`  
`https://www.flb.autos`  
`https://www.2048lt.autos`  
`https://www.snly.autos`  
`https://www.8x8x.autos`  
`https://www.hseck.autos`  
`https://www.javbus.autos`  
`https://www.mrdsyandex.autos`  
`https://www.yandexmrds.autos`  
`https://www.mrdss.autos`  
`https://www.mrdsy.autos`  
`https://www.mrdsgw.autos`  
`https://www.mrdsmrds.autos`  
`https://www.mrdsmrdasai.autos`  
`https://www.cztz.autos`  
`https://www.cztzmrds.autos`  
`https://www.czds.autos`  
`https://www.91mfsp.autos`  
`https://www.91zxsp.autos`  
`https://www.jpyq.autos`  
`https://www.mrdsyandexgw.autos`  
`https://www.jskd.autos`  
`https://www.stzb.autos`  
`https://www.91n.autos`  
`https://www.91qz.autos`  
`https://www.tt91w.autos`  
`https://www.91niu.autos`  
`https://www.crzb.autos`  
`https://www.cryy.autos`  
`https://www.crys.autos`  
`https://www.cryx.autos`  
`https://www.kmgw.autos`  
`https://www.clwz.autos`  
`https://www.dxj.autos`  
`https://www.cldq.autos`  
`https://www.4hu.autos`  
`https://www.clb.autos`  
`https://www.ttcl.autos`  
`https://www.clzz.autos`  
`https://www.zzcl.autos`  
`https://www.hmcl.autos`  
`https://www.clnm.autos`  
`https://www.clxq.autos`  
`https://www.cll.autos`  
`https://www.clg.autos`  
`https://www.javdb.autos`  
`https://www.fhk.autos`  
`https://www.fhw.autos`  
`https://www.fhben.autos`  
`https://www.fhss.autos`  
`https://www.fhb.autos`  
`https://www.fhg.autos`  
