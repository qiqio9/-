# 分布式锁原理：Redis、Redisson、RedLock 与 ZooKeeper、etcd 的实现与取舍

2026 年 07 月 18 日 15 时 55 分 20 秒

单机环境下，synchronized、ReentrantLock 等锁能很好地解决线程互斥，但它们只在一个 JVM 进程内有效。当一个服务被部署成多个实例，库存扣减、定时任务、账户操作可能在不同节点上并发执行，要保证同一资源在跨进程条件下仍被互斥访问，就需要**分布式锁**。它的本质是借助一个所有客户端都能访问、且自身可信的协调组件来达成"谁持有锁"的共识。本文将梳理分布式锁应满足的条件，重点剖析 Redis、Redisson、RedLock 以及 ZooKeeper、etcd 的实现原理、争议与选型。

## 一、为什么需要分布式锁

考虑几个典型场景：

- 同一商品库存被多个服务实例同时扣减；
- 定时任务在每个实例都有调度，必须保证只有一个实例真正执行；
- 同一账户、同一订单在多节点被并发修改。

单机锁无法跨进程生效，数据库虽然能做约束，但在高频、复杂临界区场景下不够灵活。分布式锁通过一个共享协调点，让所有节点对"锁是否已被持有"形成一致判断。

## 二、一把合格的分布式锁应满足什么

- **互斥性**：任意时刻只有一个客户端持有锁；
- **防死锁**：持有者宕机或网络异常时，锁能因超时自动释放；
- **只能释放自己的锁**：客户端不能误删他人持有的锁；
- **可重入**：同一客户端可重复获取自己已持有的锁；
- **高可用、高性能**：锁服务本身不易故障，获取释放延迟低；
- **可感知获取结果**：加解锁的状态明确。

## 三、Redis 分布式锁

Redis 因性能高、部署普遍，成为最常用的分布式锁方案。

**加锁**：使用一条原子命令 `SET key value NX PX 过期时间`，NX 保证"不存在才设置"，PX 保证到期自动释放，避免锁永久悬挂。

**解锁**：value 必须是客户端生成的唯一标识（如 UUID 加线程信息），释放时通过 Lua 脚本先比对 value、匹配才删除，防止误删别人的锁。

**锁过期与业务未完成**：若业务执行时间超过过期时间，锁会提前失效。Redisson 的**看门狗（watchdog）**机制会在持有期间后台自动续期；其可重入锁用哈希结构记录线程与重入次数，全部通过 Lua 保证原子。

**主从切换的隐患**：Redis 主从复制是异步的，锁写入主节点后若未同步就宕机、从节点提升，新主上可能没有这把锁，导致互斥失效。

## 四、RedLock 及其争议

为解决单主从丢锁，Redis 作者提出 **RedLock**：在多个相互独立的 Redis Master 上尝试加锁，超过半数成功且总耗时小于有效期才算获得锁，释放时向所有节点发送解锁。

该算法引发了著名争论：研究者 Martin Kleppmann 指出，RedLock 依赖系统时钟、且无法应对 GC 停顿和进程暂停，安全性仍可能被破坏，并建议使用带单调递增 **fencing token（栅栏令牌）**的锁；Antirez 则进行了回应。工程实践中，绝大多数业务使用普通 Redis 锁并配合业务幂等兜底，RedLock 实际采用较少。

## 五、ZooKeeper 与 etcd 锁

**ZooKeeper** 利用临时顺序节点实现锁：

- 客户端在锁节点下创建**临时顺序节点**，会话断开节点自动删除（天然防死锁）；
- 只监听自己前一个节点，前驱释放后获得锁，避免所有客户端同时被唤醒的羊群效应；
- ZooKeeper 基于 ZAB 提供强一致，可靠性高，代价是吞吐低于 Redis。

**etcd** 通过 **Lease 租约 + 事务（CAS）**实现锁：把锁绑定到会过期的租约，配合事务原子判断 key 是否存在，底层 Raft 保证一致，正是前文 Raft 协议的典型应用。

**Redis 与 ZK/etcd 的取舍**：Redis 偏向高性能、低延迟，锁语义在极端故障下稍弱；ZK/etcd 偏向强一致、可靠性高，适合对锁安全要求严格的场景。

## 六、数据库方案

低频场景也可直接借助数据库：

- **唯一索引**：插入一条带唯一约束的记录即视为获得锁；
- **乐观锁**：用版本号条件更新，冲突重试；
- **行锁**：`SELECT ... FOR UPDATE` 锁定相关行。

它们实现成本低、易理解，但性能和灵活性有限，适合并发不高的后台场景。

## 七、实践建议与常见坑

- **锁粒度尽量细、key 设计明确**，并为锁设置合理超时；
- **业务必须幂等**：分布式锁无法保证绝对不出错，幂等和对账才是最终兜底；
- **正确处理续期与异常**：业务执行中宕机、超时、异常都要保证锁最终释放；
- **警惕过期持有者继续写入**：高安全场景可引入单调递增的 fencing token，使过期锁的写操作被拒绝；
- **避免过度依赖锁**：能用消息队列、本地锁、乐观锁或天然唯一约束解决的问题，不必引入复杂的分布式锁。

## 写在最后

分布式锁是单机锁在跨进程场景下的延伸，核心始终是"互斥、防死锁、只解自己的锁"：Redis 用 NX+PX、Lua 和看门狗提供高性能锁，RedLock 试图以多数派弥补主从切换漏洞却引发时钟与暂停的争议，ZooKeeper 用临时顺序节点、etcd 用租约与事务提供强一致锁，数据库方案则适合低频场景。理解了每种锁依赖的协调机制及其在故障下的表现，再配合幂等、fencing token 与对账兜底，才能让分布式锁真正成为跨节点互斥的可靠保障，而不是新的一致性隐患。



`https://www.jmmanhua.autos`  
`https://www.jmgw.autos`  
`https://www.ytkxz.autos`  
`https://www.jsjl.autos`  
`https://www.jsjlmh.autos`  
`https://www.jmcomic.autos`  
`https://www.madoutv.autos`  
`https://www.bakamh.autos`  
`https://www.hmw.autos`  
`https://www.cbdj.autos`  
`https://www.dybz.autos`  
`https://www.dybzw.autos`  
`https://www.bjxs.autos`  
`https://www.dybzxs.autos`  
`https://www.dybzxsw.autos`  
`https://www.swxs.autos`  
`https://www.bjz.autos`  
`https://www.httv.autos`  
`https://www.qqcsp.autos`  
`https://www.cmzxgk.autos`  
`https://www.xkcm.autos`  
`https://www.mrcgds.autos`  
`https://www.dxcm.autos`  
`https://www.jysp.autos`  
`https://www.dyxs.autos`  
`https://www.momomh.autos`  
`https://www.mmmomomh.autos`  
`https://www.mdcm.autos`  
`https://www.mdzxgk.autos`  
`https://www.gdcm.autos`  
`https://www.mdzx.autos`  
`https://www.mdtv.autos`  
`https://www.mdsp.autos`  
`https://www.hxsp.autos`  
`https://www.gjwz.autos`  
`https://www.wyfl.autos`  
`https://www.wysp.autos`  
`https://www.wyyy.autos`  
`https://www.maomaomh.autos`  
`https://www.hmzx.autos`  
`https://www.yymh.autos`  
`https://www.mwyandex.autos`  
`https://www.hmmh.autos`  
`https://www.hxc.autos`  
`https://www.hxcsp.autos`  
`https://www.hxczx.autos`  
`https://www.hxcwz.autos`  
`https://www.hxcyjs.autos`  
`https://www.mhk.autos`  
`https://www.samh.autos`  
`https://www.hanman.autos`  
`https://www.hmdq.autos`  
`https://www.hmzxgk.autos`  
`https://www.waiwaimh.autos`  
`https://www.kmanhua.autos`  
`https://www.hgmh.autos`  
