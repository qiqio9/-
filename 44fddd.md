# 限流算法详解：计数器、滑动窗口、漏桶、令牌桶与分布式限流

2026 年 07 月 20 日 10 时 20 分 40 秒

任何系统的处理能力都有上限，当请求速率超过承载能力，轻则响应变慢，重则线程池打满、数据库压垮、整体雪崩。限流（Rate Limiting）就是在请求进入系统前主动控制速率，把流量约束在安全范围内，它与熔断、降级一起构成高可用系统的稳定性三件套。限流听起来简单，实现方式却各有讲究：固定窗口为何会在边界突刺、滑动窗口如何平滑、漏桶和令牌桶的差异是什么、集群下如何统一限流。本文将逐一剖析这些算法的原理、优缺点与工程落地。

## 一、为什么需要限流

限流的价值主要体现在三类场景：

- **保护系统**：大促、热点事件导致流量超过容量，主动限制以避免被压垮；
- **防刷防爬**：对单个用户、IP、接口设置调用频率；
- **保障核心服务**：资源紧张时优先保证核心链路，限制非核心请求。

限流可以作用在多个维度：QPS、并发连接数、线程数，以及按用户、IP、接口、租户分别配额。被限流后的处置也需要设计——直接拒绝、排队等待，还是返回降级结果。

## 二、固定窗口计数器

最简单的限流是**固定时间窗口计数**：在每个窗口（如 1 秒）内维护计数器，请求到来加一，超过阈值就拒绝，窗口结束后清零。

它实现简单、内存占用小，但存在明显的**临界突刺问题**：如果在一个窗口的末尾和下一个窗口的开头瞬间各打满一次配额，那么在两个窗口交界处，极短时间内实际通过了约 2 倍流量，限流被绕过。窗口越大、精度越粗，这个问题越明显。

## 三、滑动窗口

**滑动窗口**把一个大窗口切分为若干更细的子窗口，统计时只计算"当前时刻向前一个完整周期"内所有子窗口的计数；随着时间推进，窗口不断向前滑动、丢弃最旧的部分。

由于统计的始终是最近一段时间、且粒度更细，它有效缓解了固定窗口的边界突刺；子窗口划分越细，结果越平滑、越精确，代价是需要维护更多计数。可以把滑动窗口理解为"细化并连续化的计数器"，TCP 的流量控制也采用了滑动窗口的思想。

## 四、漏桶算法

**漏桶（Leaky Bucket）**把请求想象成倒入桶中的水：桶底以**恒定速率**漏出请求进行处理；请求过快导致桶满时，新请求被排队或丢弃。

它的核心作用是**流量整形**：无论上游请求多么突发，下游得到的始终是匀速流量，非常适合下游处理能力稳定、不能被突发冲击的场景。

缺点也来自"恒定"：即使桶长期空闲，处理速率也不能超过漏出速度，无法承接合理的瞬时突发。

## 五、令牌桶算法

**令牌桶（Token Bucket）**采用相反的机制：系统以恒定速率向桶中放入令牌，请求到来时必须取到令牌才能通过；桶有容量上限，未用完的令牌可以累积。

它的突出优点是**允许突发**：空闲期攒下的令牌，可以在高峰时让一批请求瞬时通过，同时长期平均速率仍受放令牌速度约束，因此成为最常用的限流算法。Guava 的 RateLimiter（提供平滑突发与预热两种模式）、Sentinel 等都实现了令牌桶思想。

**漏桶与令牌桶的核心区别**：漏桶强制输出匀速、不允许突发；令牌桶限制平均速率、但允许一定程度的突发。

## 六、分布式限流

单机限流只在单个实例内有效，当服务部署多副本、或需要全局配额时，必须集中统计：

- **Redis + Lua**：把取令牌、计数和过期判断写成一段原子 Lua 脚本，所有实例共享结果，是最常见的分布式限流实现；
- **网关层限流**：在 Nginx（`limit_req` 为漏桶、`limit_conn` 限并发）、API 网关统一做全局限流；
- **Sentinel 集群限流、Resilience4j** 等提供成熟方案。

分布式限流要关注网络开销和 Redis 故障，实践中常在中心限流不可用时退化为本地限流兜底，并做好时钟与 key 过期设计。

## 七、限流、熔断与降级的分工

这三者常被一起提及，侧重点不同：

- **限流**：事前控制请求进入的速率，防止过载；
- **熔断**：当下游服务持续异常时主动切断调用，避免级联故障，待其恢复再尝试；
- **降级**：资源紧张时舍弃非核心功能、返回兜底结果，保住核心链路。

它们通常配合使用：限流挡住入口洪峰，熔断隔离故障依赖，降级维持核心可用。

## 八、实践建议

- **先压测、后定阈值**：限流值应基于系统真实容量，而不是拍脑袋；
- **多层、多维度限流**：网关层做粗粒度全局限流，服务层按接口、用户做细粒度控制；
- **明确被限流的体验**：给用户清晰的"操作过于频繁"提示或排队机制，避免静默失败；
- **监控限流命中率与拒绝量**，据此动态调整阈值，既防过载也防误伤；
- 区分场景选算法：需要匀速整形用漏桶，需要弹性突发用令牌桶。

## 写在最后

限流算法的演进，是一条在"简单、平滑、突发、全局"之间不断取舍的路径：固定窗口简单却有临界突刺，滑动窗口用更细粒度换来平滑，漏桶强制匀速整形，令牌桶在平均速率约束下允许合理突发，而 Redis+Lua 与网关方案把单机能力扩展为集群全局控制。理解了每种算法真正约束的是什么，再结合熔断与降级，就能为系统建立起一道既挡得住恶意与洪峰、又不过度打扰正常用户的稳定防线。



`https://www.mjtv.autos`  
`https://www.mjzxgk.autos`  
`https://www.meijuw.autos`  
`https://www.jmtv.autos`  
`https://www.rhzxgk.autos`  
`https://www.rbll.autos`  
`https://www.rblldy.autos`  
`https://www.njsrj.autos`  
`https://www.xlss.autos`  
`https://www.xlsszxgk.autos`  
`https://www.mmdpy.autos`  
`https://www.mmdpydy.autos`  
`https://www.mmdpyhgdy.autos`  
`https://www.sqmh.autos`  
`https://www.pydmm.autos`  
`https://www.pydmmzxgk.autos`  
`https://www.sldxyz.autos`  
`https://www.sldxyzdy.autos`  
`https://www.hddj.autos`  
`https://www.ddj.autos`  
`https://www.hgdjai.autos`  
`https://www.aidj.autos`  
`https://www.djw.autos`  
`https://www.aidjzxgk.autos`  
`https://www.djzxgk.autos`  
`https://www.hdmgdj.autos`  
`https://www.djmfgk.autos`  
`https://www.aicrdj.autos`  
`https://www.hgdj.autos`  
`https://www.honggousp.autos`  
`https://www.honggoudj.autos`  
`https://www.fqdj.autos`  
`https://www.hgmj.autos`  
`https://www.kpkr.autos`  
`https://www.fldy.autos`  
`https://www.kpsq.autos`  
`https://www.qpzy.autos`  
`https://www.pzzywz.autos`  
`https://www.nnys.autos`  
`https://www.hqw.autos`  
`https://www.dyls.autos`  
`https://www.xtv.autos`  
`https://www.xbsp.autos`  
`https://www.dxtv.autos`  
`https://www.mwgw.autos`  
`https://www.ytkgv.autos`  
`https://www.xlspgtv.autos`  
`https://www.xiaolansp.autos`  
`https://www.ntsp.autos`  
`https://www.ncsp.autos`  
`https://www.ncyy.autos`  
`https://www.jpm.autos`  
`https://www.xjpm.autos`  
`https://www.jpmdy.autos`  
`https://www.jpmmh.autos`  
