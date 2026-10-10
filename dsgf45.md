# 分布式 ID 生成器：UUID、号段模式与雪花算法 Snowflake

2026 年 10 月 10 日 21 时 36 分 17 秒

订单号、支付流水、消息 ID、对象主键，这些编号看似不起眼，却必须在整个系统范围内绝不重复。单机时代靠数据库自增主键就能解决，可一旦分库分表，每个分片各自自增就会产生冲突；连续自增的 ID 还会暴露业务量、能被人按顺序遍历。**分布式 ID 生成器**正是为全局唯一、趋势递增、高性能发号而生，它是订单、短链接、分库分表等系统的地基。本文将讲清一个好 ID 的要求、UUID、数据库号段、Redis 自增与雪花算法 Snowflake，以及雪花算法最棘手的时钟回拨问题。

## 一、为什么需要分布式 ID

单库自增主键在规模化后会遇到几类问题：

- **分库分表后冲突**：多个分片各自从 1 开始自增，主键不再全局唯一；
- **暴露业务信息**：连续自增 ID 能被推算出用户量、订单量，且可被逐一遍历，存在安全风险；
- **跨系统对接**：订单、支付、物流之间需要一个各方都认可的唯一编号。

因此需要一个独立于单台数据库、能稳定发号的组件。

## 二、一个好 ID 应满足什么

- **全局唯一**：这是不可妥协的底线；
- **趋势递增**：对 B+ 树聚簇索引友好（呼应 B+ 树篇），无序主键会造成大量页分裂、拉低写入性能；
- **高性能、高可用**：发号延迟低、发号器本身不易宕机；
- **信息安全**：编号不连续、难以被猜测和遍历；
- **长度可控**：尽量用数字、占用空间小；部分业务还希望编号中隐含时间信息。

## 三、UUID

UUID 是最容易获得的全局唯一标识：

- 完全本地生成、无中心依赖、性能极高；
- 有多个版本：v1 基于时间与 MAC、v4 完全随机、较新的 v7 按时间有序；
- 缺点明显：128 位过长、传统 v4 完全无序，作为数据库聚簇主键会引发频繁页分裂，且字符串存储、可读性差。

因此 UUID 更适合 traceId、文件名、令牌这类不依赖数据库顺序的场景，通常不推荐作为业务表主键。

## 四、数据库自增与号段模式

**多主自增**：为不同实例设置不同起始值和相同步长（auto_increment_offset / increment），错开 ID；实现简单，但扩展不灵活、仍受单机自增上限约束。

**号段模式（Segment）**是更实用的方案：

- 发号器一次从数据库取一整段（如 max_id 到 max_id+1000），在应用内存中逐个消费，用完再取下一段；
- 常用**双 buffer**：当前号段用到一定比例就后台预加载下一段，避免发号时同步阻塞；
- 数据表记录 biz_tag、max_id、step 等字段；
- 美团 Leaf-segment 是典型实现。

它的优点是 ID 短、趋势递增、数据库压力小；缺点是依赖数据库，且重启可能跳过未用完的号（产生空洞，但不影响唯一性）。

## 五、Redis 自增

利用 Redis 单线程的 INCR 原子自增，性能高、实现简单；但要保证持久化和高可用，主从切换、未同步时可能丢号或重号，适合对编号连续性要求不高、且已有可靠 Redis 的场景。

## 六、雪花算法 Snowflake

Snowflake 把一个 64 位 long 整数切分为几段，本地即可生成：

- **1 位符号位**：恒为 0（保证正数）；
- **41 位时间戳**：毫秒级，相对自定义纪元，可使用约 69 年；
- **10 位工作机器位**：通常拆成数据中心 ID + 机器 ID，最多约 1024 个节点；
- **12 位序列号**：同一毫秒内自增，每毫秒最多 4096 个。

生成逻辑大致如下：

    if 当前毫秒 == 上次毫秒:
        序列号 = (序列号 + 1) & 4095
        if 序列号溢出: 等待下一毫秒
    else:
        序列号 = 0
    id = (时间戳 << 22) | (机器ID << 12) | 序列号

它的优势是**本地生成、无网络开销、性能极高、ID 趋势递增且隐含时间**，单机理论上每秒可发数十万号，是目前业务主键最主流的方案。

## 七、关键问题：时钟回拨

Snowflake 强依赖机器时钟，而 NTP 校时可能让时钟**往回跳**，回拨后生成的时间戳与之前重复，就可能产生重复 ID。常见处理：

- 记录上次发号时间戳，发现回拨时，若幅度很小则**等待**时钟追上来再发；
- 幅度较大则**拒绝发号并告警**，避免冒险；
- workerId 通常借助 ZooKeeper、etcd 或配置中心统一分配，节点启动时还会校验时钟；
- 百度 UidGenerator 用 RingBuffer 并容忍一定回拨，Leaf-snowflake 用 ZooKeeper 注册 worker、持久化时间戳做校验。

## 八、方案对比

| 方案 | 顺序性 | 性能 | 依赖 | 主要问题 |
| --- | --- | --- | --- | --- |
| UUID | 传统版本无序 | 极高 | 无 | 长、索引页分裂 |
| 号段模式 | 趋势递增 | 高 | 数据库 | 重启跳号、依赖 DB |
| Redis INCR | 递增 | 高 | Redis | 故障可能丢号 |
| Snowflake | 趋势递增 | 极高 | 需分配 workerId | 时钟回拨 |

## 九、实践建议

- 业务表主键优先 Snowflake 或号段模式；traceId、文件名用 UUID；
- 上线 Snowflake 前必须落实时钟回拨处理与 workerId 统一管理；
- 可自定义纪元、预留扩展位，为未来分片和机房调整留余地；
- 监控发号延迟、回拨次数和号段消耗，避免发号器成为热点单点；
- 区分“内部主键”和“对外编号”，对外可再做混淆或加业务前缀，避免直接暴露规律。

## 写在最后

分布式 ID 解决的是“在没有单一权威计数器的情况下，如何让编号全局唯一且好用”：UUID 以本地随机换无依赖却牺牲顺序，号段模式用数据库批量取号平衡简单与性能，Redis 提供轻量自增，而 Snowflake 用时间戳、机器位和序列号的精巧切分实现本地高性能发号，唯一需要认真对待的是时钟回拨。理解了每种方案依赖什么、顺序性从哪来，就能在分库分表和订单系统中，为每一条数据拿到既不重复、又对索引友好的可靠编号。

`https://91dy.net.cn`  
`https://91gw.hk.cn`  
`https://91mf.net.cn`  
`https://91mfspwz.cn`  
`https://91shipinwz.com.cn`  
`https://91w.ac.cn`  
`https://91xjsp.com.cn`  
`https://91xsp.net.cn`  
`https://91ys.ac.cn`  
`https://91zxk.net.cn`  
`https://shenmadtygw.com.cn`  
`https://shenmadianyingwang.cn`  
`https://mogushipin.bj.cn`  
`https://lanmeisp.org.cn`  
`https://yhdmgw.ac.cn`  
`https://yinghuasp.hk.cn`  
`https://agedmgwsy.cn`  
`https://xkysgw.com.cn`  
`https://xingchenys.ac.cn`  
`https://xcyy.hk.cn`  
`https://ttdy.hk.cn`  
`https://quanjiw.cn`  
`https://dyttwgw.cn`  
`https://dianyingtiantangw.org.cn`  
`https://dianyingtiantang.hk.cn`  
`https://dashixiongys.org.cn`  
`https://hmmmjx.com.cn`  
`https://91spmfgk.cn`  
`https://hongtaosp.hk.cn`  
`https://hongtaoys.hk.cn`  
`https://jfys.net.cn`  
`https://shyswz.cn`  
`https://shyywz.net.cn`  
`https://xgyy.hk.cn`  
`https://xiguady.ac.cn`  
`https://yinghuays.hk.cn`  
`https://ysc.ac.cn`  
`https://xiaoxiaoys.ac.cn`  
`https://wkyy.net.cn`  
`https://wgdpydyhkdpttbd.com.cn`  
`https://mhmvgkwzbbdmf.com.cn`  
`https://mahuays.hk.cn`  
`https://911blw.ac.cn`  
`https://guaziys.ac.cn`  
`https://crmh.ac.cn`  
`https://crw.net.cn`  
`https://crdy.ac.cn`  
`https://aiaiw.net.cn`  
`https://555ys.ac.cn`  
`https://91cm.ac.cn`  

