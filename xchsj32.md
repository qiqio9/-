# 一文读懂 Raft：分布式一致性协议如何通过选举与日志复制达成共识

2026 年 08 月 30 日 16 时 20 分 35 秒

在分布式系统中，多台节点共同维护同一份数据，任何一台都可能宕机，网络还会出现延迟、丢包甚至分区。如何让这些节点在种种故障下依然对数据状态达成一致，是分布式领域最核心的问题之一。长期以来 Paxos 是这一问题的经典答案，却以晦涩难懂著称。2014 年斯坦福大学的 Diego Ongaro 提出了 **Raft**，它明确把"可理解性"作为设计目标，将一致性问题拆解为领导者选举、日志复制和安全性三个相对独立的部分，如今已被 etcd、Consul、TiKV 等广泛采用。本文将沿着这三部分，讲清 Raft 的核心机制。

## 一、为什么需要共识算法

单台服务器简单可靠但终究会故障，系统通常用多副本提升可用性，可副本一多，"以谁的数据为准"就成了难题。

共识算法的目标是：让一个集群在部分节点崩溃、消息延迟、丢失、重复以及网络分区的情况下，仍能对外表现得像一台可靠的机器。Raft 处理的是**崩溃故障**（节点只是停机、延迟、重启，不会恶意造假），而非需要拜占庭容错的恶意节点场景。

达成一致的关键机制是**多数派（Quorum）**：一个决议需要得到超过半数节点（N/2+1）的认可。多数派有一个重要性质——任意两个多数派集合必然存在交集，这保证了集群不会同时通过两个相互冲突的决议。通常集群部署奇数个节点（如 3、5），在容忍故障数和成本之间更划算。

## 二、Raft 的总体思路：强领导者与三大子问题

Raft 采用**强领导者模型**：集群中同一时刻只有一个领导者（Leader），所有写操作都由它处理，再由它复制给其他节点。这种中心化设计让数据流向清晰、易于理解。

它把复杂的一致性问题拆成三块：

1. **领导者选举（Leader Election）**：如何在集群中选出唯一的 Leader；
2. **日志复制（Log Replication）**：Leader 如何把写操作可靠地复制到多数节点；
3. **安全性（Safety）**：通过一组约束保证已确认的数据绝不丢失、不冲突。

## 三、节点角色与任期

Raft 节点在任一时刻处于三种角色之一：

- **Follower（跟随者）**：被动接收 Leader 的心跳和日志，不主动发起请求；
- **Candidate（候选人）**：选举过程中的临时角色，用于争取选票；
- **Leader（领导者）**：处理客户端请求、复制日志、周期性发送心跳维持权威。

Raft 还引入了**任期（Term）**概念，时间被划分为一个个单调递增的任期，每个任期最多有一个 Leader。Term 充当逻辑时钟：节点发现存在更高任期时，会立即更新自己的任期并降级为 Follower，这让"过期的领导者"无法继续发号施令。

## 四、领导者选举的完整过程

集群启动时，所有节点都是 Follower，并各自设置一个随机的**选举超时时间**。正常情况下，Leader 会周期性发送心跳，Follower 收到心跳就重置计时器。

当某个 Follower 在选举超时内没有收到心跳，它会认为 Leader 已故障，于是：

1. 将自己转为 Candidate，当前 Term 加一；
2. 给自己投一票，并向其他节点发送 `RequestVote` 请求拉票；
3. 若获得**多数选票**，就成为新的 Leader，立即广播心跳宣告地位；
4. 若期间发现已有更高 Term 的 Leader，则主动降级为 Follower。

投票有两条关键规则：

- **一个任期内每个节点只能投一票，先到先得**，避免同时选出多个 Leader；
- **只有候选人的日志"至少和自己一样新"时才投票给它**，这是安全性的重要保障。

当多个 Candidate 得票都不过半（**分裂投票**）时，会因各自**随机化的选举超时**在不同时刻重新发起选举，通常一两轮后就能分出胜负。随机超时是 Raft 简单却关键的设计。

## 五、日志复制：写操作如何被确认

Leader 选出后开始承接写请求，日志复制流程如下：

1. 客户端把写命令发给 Leader；
2. Leader 将其作为一条日志（包含 Term、索引号 Index 和命令内容）追加到本地日志；
3. 并行向各 Follower 发送 `AppendEntries`，让它们复制这条日志；
4. 当该日志被**多数节点**写入，Leader 将其标记为**已提交（committed）**，执行到状态机并向客户端返回成功；
5. 随后的心跳会告知 Follower 哪些日志已提交，它们也各自执行。

Raft 保证日志的高度一致性：不同节点上，若两条日志的索引号和 Term 都相同，则命令内容相同，且其之前的所有日志也都相同。

当 Follower 日志落后或与 Leader 冲突时，Leader 会让它**回溯**到双方最后一致的位置，删除之后的冲突条目、补齐缺失条目，最终强制对齐到 Leader 的日志。这保证了以 Leader 为准的收敛。

## 六、安全性：什么情况下数据才不会出错

仅有选举和复制还不够，Raft 用几条安全约束堵住边界情况。

**选举限制**：Candidate 必须日志足够新才能当选。比较规则是先比最后一条日志的 Term，Term 大者为新；Term 相同再比索引号，索引大者为新。这保证了新 Leader 一定包含所有已提交的日志（**Leader 完整性**）。

**提交旧任期日志的陷阱**：Leader 不能仅凭"某条旧任期日志已复制到多数节点"就提交它，因为在特定时序下它可能被其他节点覆盖；必须通过提交一条**当前任期**的日志来间接提交之前的条目。这是 Raft 论文著名的"图 8"问题。

**网络分区与脑裂**：分区后，少数派一侧若选出 Leader，也无法获得多数派确认，因此不能提交任何写操作；多数派一侧可正常工作并推进 Term。分区恢复后，少数派的旧 Leader 发现更高 Term 会立即降级，其未提交日志被覆盖，集群重新一致。这正是多数派机制防止"脑裂双写"的体现。

## 七、成员变更、日志快照与工程落地

**成员变更**：直接增减节点可能在切换瞬间出现两个互不相交的多数派，Raft 通过联合共识（或单节点逐个变更）保证变更过程中任意时刻的多数派都重叠、安全。

**日志快照（Snapshot）**：日志不能无限增长，节点会定期把已提交日志"压缩"成一份状态机快照，丢弃之前的日志；落后过多的 Follower 通过 `InstallSnapshot` 先补齐快照再追日志。

**典型应用**：etcd 用 Raft 为 Kubernetes 提供一致的元数据存储，Consul 用其支撑服务发现与配置，TiKV 则在每个数据分片上运行一组 Raft（**Multi-Raft**），实现海量数据下的一致性与水平扩展。

## 写在最后

Raft 的优雅在于"分解"与"约束"：用随机超时和多数投票解决"谁是 Leader"，用 AppendEntries 和强制对齐解决"日志如何一致"，再用选举限制、提交规则和任期机制保证"已确认的数据永远正确"。相比 Paxos，它没有牺牲正确性，却让整个协议可以被工程师完整理解和正确实现。掌握了角色、任期、选举、日志复制与安全性这条主线，再看 etcd、Consul 等系统的行为，以及分布式数据库中的多副本一致性，就有了一个清晰而坚实的分析框架。




`https://www.mwagw.com.cn`  
`https://www.mwa.ac.cn`  
`https://www.mtcm.org.cn`  
`https://www.mhdq.org.cn`  
`https://www.mfdyw.net.cn`  
`https://www.mfdy.net.cn`  
`https://www.meijtt.net.cn`  
`https://www.lfwz.com.cn`  
`https://www.dmdq.org.cn`  
`https://www.awdm.com.cn`  
`https://www.91md.com.cn`  
`https://www.91h.net.cn91`  
`https://www.zjuw.com.cn`  
`https://www.xiguayy.net.cn`  
`https://www.xiguays.org.cn`  
`https://www.xiguasp.net.cn`  
`https://www.xiguady.net.cn`  
`https://www.xgkf.com.cn`  
`https://www.taosesp.net.cn`  
`https://www.spwz.net.cn`  
`https://www.qmbw.com.cn`  
`https://www.qjiw.com.cn`  
`https://www.piankuw.net.cn`  
`https://www.mitwz.net.cn`  
`https://www.mitw.net.cn`  
`https://www.mitcm.net.cn`  
`https://www.mitaozx.net.cn`  
`https://www.mitaoys.net.cn`  
`https://www.mitaotv.net.cn`  
`https://www.mitaodm.net.cn`  
`https://www.mitaocss.com.cn`  
`https://www.mimiyjs.net.cn`  
`https://www.mimirk.com.cn`  
`https://www.mimijx.net.cn`  
`https://www.manwa.ac.cn`  
`https://www.lifzx.com.cn`  
`https://www.lifdm.com.cn`  
`https://www.lfzxgk.net.cn`  
`https://www.lfwz.net.cn`  
`https://www.lfdq.net.cn`  
`https://www.kanpwz.net.cn`  
`https://www.kanjuw.com.cn`  
`https://www.kanjub.net.cn`  
`https://www.jmmh.ac.cnjm`  
`https://www.jinmantt.net.cn`  
`https://www.hongtsp.com.cn`  
`https://www.hemays.net.cn`  
`https://www.dytiantang.org.cn`  
`https://www.dxkm.net.cn`  
`https://www.dmlf.com.cn`  
`https://www.cilitt.com.cn`  
`https://www.bttt.net.cnBT`  
`https://www.benziwz.com.cn`  
`https://www.aawang.com.cn`  
`https://www.huangsedm.com.cn`  
`https://www.51dongman.net.cn`  
`https://www.51mhua.com.cn`  
`https://www.aimeiju.net.cn`  
`https://www.chimidy.com.cn`  
