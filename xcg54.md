# IO 多路复用：从 select、poll 到 epoll 的原理与演进

2026 年 09 月 12 日 11 时 30 分 45 秒

一台服务器如何同时维护上万个客户端连接？这正是著名的 C10K 问题。如果沿用"一个连接配一个线程"的阻塞式模型，万级连接意味着万级线程，光是栈内存和上下文切换就足以压垮系统；而把连接设为非阻塞后反复轮询，又会让 CPU 空转。IO 多路复用给出的答案是：用一次系统调用同时盯住大量文件描述符，只在其中某些连接真正就绪时才返回处理。本文将依次剖析 select、poll、epoll 三种实现的工作方式与性能差异，重点讲清 epoll 借助红黑树、就绪链表和事件回调实现高效的底层逻辑，以及水平触发与边缘触发的区别。

## 一、从阻塞 IO 的困境说起

在传统阻塞 IO（BIO）模型中，服务端在 `accept()` 等待新连接、在 `read()` 等待数据时都会阻塞当前线程。为了同时服务多个连接，常见做法是每个连接分配一个线程。

这种方式在连接数少时简单可靠，但连接规模一大就难以为继：

- 每个线程都要占用独立的栈空间（通常 MB 级），万级连接的内存成本很高；
- 大量线程带来频繁的上下文切换，CPU 时间被调度而非业务消耗；
- 多数连接在大部分时间里并没有数据，线程却依然为它们空占资源。

把套接字设为非阻塞可以避免线程被卡住，但程序需要不断轮询每个连接"有没有数据"，绝大多数询问都返回"未就绪"，造成 CPU 忙等。

理想的机制应当是：把"哪些连接需要关注"一次性告诉内核，由内核在后台监控，应用只在有连接就绪时被唤醒并拿到就绪名单。这就是 **IO 多路复用**。

## 二、什么是 IO 多路复用

IO 多路复用通过 `select`、`poll`、`epoll_wait` 这样的系统调用，让一个线程同时监听多个文件描述符（fd）；当其中任意一个 fd 就绪（可读、可写或发生异常），调用就返回，线程再针对就绪的 fd 进行处理。

需要明确的是，IO 多路复用仍属于**同步 IO**：它只负责通知"哪个 fd 就绪了"，真正的数据读取仍由应用自己调用 `read` 完成；与之相对，异步 IO（如 Windows 的 IOCP）会由内核把数据拷贝到应用缓冲区后再通知，二者层次不同。

Linux 上 IO 多路复用主要经历了 select、poll、epoll 三代实现。

## 三、select：最早的多路复用

select 使用一个位图（fd_set）来表示要监听的 fd 集合，调用时传入关注读、写、异常的三个集合以及超时时间，内核返回后通过位图标记哪些 fd 就绪。

它的设计在今天看来有几个明显限制：

**连接数上限**：位图大小受 `FD_SETSIZE` 约束，通常只能监听 1024 个 fd，难以支撑海量连接。

**每次调用都要全量拷贝**：fd 集合每次调用都要从用户态重新拷贝到内核，调用返回后还需把集合拷贝回来。

**内核线性扫描**：内核通过遍历整个 fd 集合来检查就绪状态，复杂度为 O(N)。

**用户态再次遍历**：select 返回后只告诉你"有 fd 就绪"，却不告诉你具体是哪些，应用必须再遍历一遍全部 fd 才能找到就绪者。

此外，位图在每次调用后会被内核改写，应用每次都要重新初始化。select 的优点是**跨平台性好**，几乎所有系统都支持，但在高并发场景下效率不足。

## 四、poll：去掉上限，却未根治瓶颈

poll 用 `pollfd` 数组替代位图，每个元素明确记录 fd、关注的事件和返回的事件。它解决了 select 的 1024 连接上限，也不再需要每次重新初始化位图。

但 poll 的核心工作方式并未改变：

- 每次调用仍需把整个 pollfd 数组从用户态拷贝到内核；
- 内核依然要线性遍历所有 fd 判断就绪，复杂度仍是 O(N)；
- 返回后应用仍需遍历数组找出就绪 fd。

因此，poll 可以看作对 select 的一次有限改良，在连接数较多时同样会随监听总数增长而变慢。

## 五、epoll：事件驱动与内核就绪表

epoll 不再用"每次传全量集合、每次全量扫描"的模式，而是把"注册关注"和"等待就绪"拆成三个独立接口：

- `epoll_create()`：创建一个 epoll 实例；
- `epoll_ctl()`：向实例中**注册、修改或删除**关注的 fd 及事件，这些信息常驻内核，通常只需注册一次；
- `epoll_wait()`：等待并获取当前已就绪的事件列表。

在内核实现上，epoll 维护了两个关键结构：

**一棵红黑树**：保存所有被注册监听的 fd（兴趣列表），支持高效的增删改，复杂度约为 O(log N)，避免重复全量拷贝。

**一个就绪双向链表**：当某个 fd 状态就绪时，内核通过注册在其上的**回调函数**把它加入就绪链表；`epoll_wait` 只需从该链表中取出就绪事件返回，而不必扫描全部监听 fd。

这意味着 epoll 的开销主要与"**活跃连接数**"相关，而不是"监听连接总数"。在连接很多但同时就绪的很少这一典型网络场景下，优势极为明显。

## 六、水平触发 LT 与边缘触发 ET

epoll 提供两种就绪通知模式，这是使用中最需要理解的细节。

**水平触发（Level Triggered，LT，默认模式）**：只要 fd 上还有未读完的数据，每次 `epoll_wait` 都会持续通知，直到数据被处理完。它编程简单、不易漏事件，select 和 poll 本质上也都是这种语义。

**边缘触发（Edge Triggered，ET）**：只在 fd 状态**发生变化的那一刻**通知一次，即使数据没有读完也不会再次通知。使用 ET 时必须：

- 把 fd 设为**非阻塞**；
- 在收到通知后循环读取，直到返回 `EAGAIN/EWOULDBLOCK`，确认数据已读尽；
- 否则剩余数据可能再也得不到通知，造成"饿死"。

ET 减少了事件被重复通知的次数，在特定高吞吐场景下能降低 epoll 唤醒开销，但对编程正确性要求更高；LT 更稳妥，是多数框架的默认选择。

## 七、实践建议与常见误区

**epoll 并非万能**：如果连接数很少，或所有连接都长期高度活跃，epoll 相对 select/poll 的优势并不明显；而在需要跨平台时，select 仍有价值。不同系统也有对应机制，如 BSD/macOS 的 kqueue，Windows 则提供真正异步的 IOCP。

**ET 必须配非阻塞 IO**：在 ET 模式下使用阻塞读写，一次未预期的阻塞就可能卡住整个处理线程。

**与 Reactor 模式配合**：epoll + 非阻塞 IO + 事件分发（Reactor）是 Nginx、Redis、Netty 等高性能网络组件的共同底座；epoll 负责高效感知就绪，线程或协程负责并发处理，二者缺一不可。

**注意边界问题**：例如多个进程/线程同时 `accept` 同一监听套接字可能产生惊群；连接到达过快、文件描述符耗尽时 `accept` 返回错误，都需要专门处理。较新的内核还提供了 `EPOLLEXCLUSIVE` 等机制缓解惊群。

**不要神化单次性能**：IO 多路复用解决的是"高效管理大量连接"，真正的吞吐还取决于业务逻辑、线程模型、内存拷贝（如零拷贝）等整条链路。

## 写在最后

从 select 到 poll 再到 epoll，演进的主线非常清晰：先解除连接数限制，再把"每次全量拷贝、全量扫描"改为"一次注册、事件回调、只取就绪"。红黑树让关注集合可动态维护，就绪链表让 `epoll_wait` 的成本只取决于活跃连接，LT/ET 则在通知的及时性与编程复杂度之间提供了取舍。理解了这套机制，就能明白为什么高并发网络程序几乎都构建在 epoll 之上，也能在面对"连接数上不去、CPU 却被轮询打满"之类问题时，准确判断症结所在。


`https://dmw.dmw.ac.cn`  
`https://dydh.dydh.com.cn`  
`https://fcdmw.fcdmw.net.cn`  
`https://gywz.gywz.net.cn`  
`https://hhdm.hhdm.net.cn`  
`https://hjcg.hjcg.net.cn`  
`https://hmzx.hmzx.net.cn`  
`https://jrtrk.jrtrk.com.cn`  
`https://llp.llp.net.cn`  
`https://mejdw.mejdw.com.cn`  
`https://mhwz.mhwz.net.cn`  
`https://mmhm.mmhm.net.cn`  
`https://npdm.npdm.net.cn`  
`https://omzx.omzx.com.cn`  
`https://rbdm.rbdm.net.cn`  
`https://rsp.rsp.ac.cn`  
`https://sryy.sryy.org.cn`  
`https://trdm.trdm.net.cn`  
`https://trmh.trmh.net.cn`  
`https://trxs.trxs.net.cn`  
`https://trxsw.trxsw.com.cn`  
`https://txsjz.txsjz.com.cn`  
`https://txsp.txsp.org.cn`  
`https://xdy.xiaody.com.cn`  
`https://xydmk.xydmk.net.cn`  
`https://ysbz.ysbz.net.cn`  
`https://crmh.crmh.org.cn`  
`https://zwzmzx.zwzmzx.com.cn`  
`https://ystrmh.ystrmh.com.cn`  
`https://xlsp.xlsp.com.cn`  
`https://wzlkmh.wzlkmh.com.cn`  
`https://tzsp.taozisp.ac.cn`  
`https://txzx.txzx.org.cn`  
`https://txvlog.txvlog.ac.cn`  
`https://ttyy.tiantianyy.net.cn`  
`https://ttyy.tiantangyy.net.cn`  
`https://ttdm.tiantiandm.net.cn`  
`https://sxh.suxiaohan.com.cn`  
`https://smdy.smdy.net.cn`  
`https://rhzxsp.rhzxsp.com.cn`  
`https://rhyq.rhyq.com.cn`  
`https://ppyy.pipiyy.net.cn`  
`https://ntw.nantongw.net.cn`  
`https://mrhl.mrhl.net.cn`  
`https://mrds.mrdasai.net.cn`  
`https://mrcg.mrcg.net.cn`  
`https://mimijx.mimijx.com.cn`  
`https://mfdy.mianfeidy.com.cn`  
`https://kpsq.kpsq.com.cn`  
`https://kpkr.kpkr.com.cn`  
`https://jskd.jiusekd.com.cn`  
`https://jsjlmh.jsjlmh.com.cn`  
`https://htsp.hongtaosp.net.cn`  
`https://hemays.hemays.com.cn`  
`https://gzys.guaziys.net.cn`  
`https://gpys.guapiys.net.cn`  
`https://gmzrbz.gmzrbz.com.cn`  
`https://fulip.fulip.com.cn`  
`https://fqys.fqys.com.cn`  
