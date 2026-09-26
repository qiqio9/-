# 分布式链路追踪：Trace、Span、上下文传播与 OpenTelemetry

2026 年 07 月 01 日 16 时 25 分 30 秒

在单体应用里，一个请求自始至终在一个进程内完成，查日志就能定位问题。可一旦系统拆成微服务，一次"下单"可能依次经过网关、订单、库存、支付、消息队列等十几个服务，日志散落在不同机器，谁慢了、哪一跳报错、依赖关系如何，很难靠肉眼拼凑。**分布式链路追踪（Distributed Tracing）** 正是为还原这种跨服务调用路径而生。本文将讲清 Trace 与 Span 等核心概念、上下文如何跨进程与异步传播、自动埋点与 OpenTelemetry 标准、采样与存储方案，以及链路追踪如何与日志、指标共同构成可观测性三支柱。

## 一、为什么需要链路追踪

设想一次请求链路：

`网关 → 订单服务 → 库存服务 → 支付服务 → 消息队列 → 下游消费`

当用户反馈"下单很慢"或"偶尔失败"，我们需要回答：

- 时间主要耗在哪一跳？
- 错误发生在哪个服务、哪个方法？
- 服务之间的真实依赖和调用次数是怎样的？

没有链路追踪，这些信息分散在各服务日志里、缺乏共同标识，排查如同盲人摸象。链路追踪给每次请求分配全局 ID，把跨进程的片段串成一条完整路径。

## 二、核心概念：Trace 与 Span

- **Trace（追踪）**：一次完整请求从入口到结束的全部调用过程，用一个全局唯一的 **traceId** 标识；
- **Span（跨度）**：链路中的一个最小操作单元，如一次 RPC、一次数据库查询；
- 每个 Span 有 **spanId** 和 **parentSpanId**，通过父子关系组织成一棵树；
- Span 记录开始/结束时间、耗时、标签（tag，如实例、版本）、事件（log）和状态（是否出错）。

多个 Span 最终以瀑布图形式呈现，调用顺序、并行关系和每段耗时一目了然。

## 三、上下文传播

链路要串起来，关键在于把 trace 上下文不断传递下去：

- 请求在入口（网关）生成 traceId，之后随每次调用透传；
- **HTTP** 通过请求头传递（如 W3C 的 `traceparent`），RPC 通过 attachment，**消息队列**通过消息属性透传；
- **跨线程、线程池、异步任务**需要显式传递上下文，否则会出现"断链"——这与线程池的使用密切相关；
- 除了 traceId，还可用 **baggage** 在整条链路上携带业务字段（如租户、灰度标记）。

上下文在哪一跳丢失，链路就从哪里断裂，这是接入时最常见的问题。

## 四、埋点与数据采集

生成 Span 的方式有两种：

- **手动埋点**：在代码中显式创建 Span，精确但侵入业务；
- **自动埋点**：通过框架拦截、Java agent 字节码增强等方式无侵入地为 HTTP、RPC、数据库调用自动生成 Span。SkyWalking、OpenTelemetry 的自动探针都采用这一思路，其底层正是前文的动态代理与类加载机制。

**OpenTelemetry** 已成为事实标准：它统一了埋点 API、SDK，并通过 Collector 负责数据接收、处理和导出，使后端存储可替换，避免被单一厂商绑定。

## 五、采样、存储与展示

全量采集每一次请求会带来明显的性能和存储成本，因此需要**采样**：

- **头部采样**：请求进入时就决定是否记录，简单但可能漏掉错误请求；
- **尾部采样**：等链路完成后再决定，可做到"错误请求全采、正常请求按比例采"，但需要临时缓存；
- 高流量系统通常按比例采样，并对慢请求、异常请求提高采样率。

常见后端有 Jaeger、Zipkin、SkyWalking、Tempo 等，提供瀑布图、服务依赖拓扑和检索能力。

## 六、可观测性三支柱

链路追踪不是孤立的，它与另外两者共同构成可观测性：

- **Logs（日志）**：描述离散、详细的事件；
- **Metrics（指标）**：聚合的数值，如 QPS、延迟、错误率；
- **Traces（链路）**：描述一次请求经过的完整路径。

最佳实践是把 **traceId 写入日志上下文（如 MDC）**，从指标发现异常、点进某条 Trace、再跳到对应日志，三者用同一标识互相打通。

## 七、实践建议

- 统一接入标准（OpenTelemetry），在入口生成 traceId 并保证全链路透传；
- 为 Span 补充关键标签（用户、订单号、错误码），便于检索和归因；
- 重点处理线程池、异步、定时任务和消息队列等容易断链的场景；
- 根据流量设置合理采样率，并监控追踪带来的性能开销；
- 在系统建设初期就接入，而不是故障频发后才补救。

## 写在最后

分布式链路追踪用一个 traceId 把跨服务、跨进程甚至跨异步任务的调用片段重新组织成完整的 Trace：Span 记录每一跳的耗时与状态，上下文传播保证链路不断，自动埋点和 OpenTelemetry 让接入标准化、可移植，采样则在可观测性与成本之间取得平衡。当它与日志、指标通过同一标识关联起来，微服务排障就从"在多台机器的日志里碰运气"变成"沿着调用链逐级定位"。理解了 Trace、Span 与上下文传播这条主线，就能为复杂分布式系统建立起真正可追溯、可观测的运维能力。



`https://www.huarenyy.autos`  
`https://www.hrzb.autos`  
`https://www.hsxy.autos`  
`https://www.jzdasldxy.autos`  
`https://www.xyzdyh.autos`  
`https://www.glj.autos`  
`https://www.rjgljmh.autos`  
`https://www.rjglj.autos`  
`https://www.jjwx.autos`  
`https://www.jjwxc.autos`  
`https://www.jjxs.autos`  
`https://www.xswang.autos`  
`https://www.xswz.autos`  
`https://www.mmdzy.autos`  
`https://www.mmdzydy.autos`  
`https://www.szdzy.autos`  
`https://www.mmdzyzxgk.autos`  
`https://www.htwxc.autos`  
`https://www.ylxz.autos`  
`https://www.ylxzxs.autos`  
`https://www.jjdpy.autos`  
`https://www.pydjj.autos`  
`https://www.plgjj.autos`  
`https://www.jqxs.autos`  
`https://www.thw.autos`  
`https://www.thsq.autos`  
`https://www.tanhuasp.autos`  
`https://www.tczx.autos`  
`https://www.tczxdyzxgk.autos`  
`https://www.bncm.autos`  
`https://www.bncmdm.autos`  
`https://www.cmxs.autos`  
`https://www.cmny.autos`  
`https://www.nycm.autos`  
`https://www.snzxs.autos`  
`https://www.sxxs.autos`  
`https://www.cmbn.autos`  
`https://www.91pr.autos`  
`https://www.blxs.autos`  
`https://www.qdxs.autos`  
`https://www.91lt.autos`  
`https://www.91th.autos`  
`https://www.2048kjd.autos`  
`https://www.91jx.autos`  
`https://www.hsck.autos`  
`https://www.hsckgw.autos`  
`https://www.91dsj.autos`  
`https://www.91av.autos`  
`https://www.91dasheng.autos`  
`https://www.91mfk.autos`  
`https://www.91llq.autos`  
`https://www.91anw.autos`  
`https://www.91s.autos`  
`https://www.91h.autos`  
`https://www.91hs.autos`  
`https://www.91dh.autos`  
