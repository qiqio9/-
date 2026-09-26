# 零拷贝 Zero-Copy：从四次数据拷贝到 mmap、sendfile 的性能演进

2026 年 08 月 27 日 11 时 50 分 20 秒

Kafka 吞吐高、Nginx 发文件快，"零拷贝"是被反复提到的原因之一。但这个名字容易让人误解为"数据完全不复制"，实际上它指的是**在数据传输过程中，尽量减少由 CPU 参与的数据拷贝次数，并减少用户态与内核态之间的上下文切换**；由 DMA 完成的硬件拷贝依然存在。本文将从最普通的"读文件、发网络"流程出发，逐步拆解传统方式下的四次拷贝与四次切换，再对比 mmap、sendfile、splice 各自如何削减开销，最后说明零拷贝的适用场景与常见误区。

## 一、数据是如何在磁盘、内存与网卡间流动的

考虑一个最常见的任务：服务器把一个文件通过网络发送给客户端。这需要两类系统调用配合——先 `read()` 把文件读进内存，再 `write()` 把数据写往套接字。

理解拷贝，先要认识两个概念：

**用户态与内核态**：应用程序运行在用户态，权限受限；磁盘、网卡等硬件必须由操作系统内核操作，应用通过系统调用陷入内核态完成。每次系统调用都会引起一次**上下文切换**。

**DMA（直接内存访问）**：磁盘与内存、网卡与内存之间的大批量数据搬运，由专门的 DMA 控制器完成，不需要 CPU 逐字节复制。CPU 只负责发起和收尾。

因此一次传输中，有些拷贝是 DMA 做的、有些是 CPU 做的，零拷贝要消除的主要是后者以及"数据绕路用户态"的开销。

## 二、传统读写：四次拷贝与四次上下文切换

用传统的 `read()` + `write()` 发送文件，数据经历的路径如下：

1. `read()` 触发：磁盘数据经 **DMA 拷贝**到**内核缓冲区**（Page Cache）；
2. 内核再经 **CPU 拷贝**把数据从内核缓冲区复制到**用户缓冲区**；
3. `write()` 触发：数据经 **CPU 拷贝**从用户缓冲区复制到 **socket 缓冲区**；
4. 最后经 **DMA 拷贝**从 socket 缓冲区送到**网卡**发出。

合计 **4 次数据拷贝**（2 次 DMA + 2 次 CPU），以及 **4 次上下文切换**（`read`、`write` 各进入和返回一次）。

关键问题在于：数据在用户缓冲区里只是"路过"，应用并没有修改它，却付出了两次 CPU 拷贝和绕路用户态的代价。零拷贝的各种方案，正是围绕"能不能不让数据进用户态"展开。

## 三、mmap：让用户态与内核共享数据

**mmap（内存映射文件）**把一个文件映射到用户空间的内存地址，应用读取这段内存时，由缺页机制把文件内容加载进内核缓冲区，而这块内存在用户态可直接访问，相当于用户缓冲区与内核缓冲区合二为一。

用 mmap + write 的路径变为：

1. 磁盘经 DMA 拷贝到内核缓冲区（用户态通过映射直接可见）；
2. 数据经 CPU 拷贝从内核缓冲区到 socket 缓冲区；
3. 再经 DMA 拷贝到网卡。

合计 **3 次拷贝**（2 次 DMA + 1 次 CPU），上下文切换仍为 4 次。它省掉了"内核→用户"那次 CPU 拷贝。

但 mmap 有明显的使用门槛：

- 首次访问会触发缺页中断，文件未在缓存中时仍要等磁盘；
- 映射后若文件被其他进程截断，写入可能触发 `SIGBUS` 信号；
- 需要正确 munmap，映射数量和虚拟地址空间有限；
- 它仍需调用 write 才能发出数据，并未完全脱离用户态。

mmap 适合需要频繁读取同一文件、或进程间共享内存的场景。

## 四、sendfile：让数据完全不经过用户态

**sendfile** 专门为"把文件内容送往套接字"设计，只需一次系统调用，数据在内核内部就从文件描述符流向 socket，应用完全接触不到数据。

在早期实现中：

- 磁盘经 DMA 到内核缓冲区；
- 经 CPU 拷贝到 socket 缓冲区；
- 经 DMA 到网卡。

这是 **3 次拷贝（2 DMA + 1 CPU）、仅 2 次上下文切换**。

而在支持 **SG-DMA（散射—聚集 DMA）** 的 Linux 2.4 及之后版本中，内核不再把数据复制进 socket 缓冲区，而是只把数据位置和长度等描述信息传给网卡，DMA 直接根据描述符从内核缓冲区读取并发出。此时：

- 磁盘经 DMA 到内核缓冲区；
- 网卡经 DMA 直接从内核缓冲区取数据。

全程 **2 次拷贝且都由 DMA 完成，CPU 拷贝为 0，上下文切换仅 2 次**，这才是名副其实的"零 CPU 拷贝"。

sendfile 的限制也很明确：只能用于文件到 socket 的单向传输，传输过程中无法在用户态修改数据；最优效果还需要网卡硬件支持。

## 五、splice 与编程接口

**splice** 在 Linux 中用于在文件、管道等之间移动数据而不经过用户态，可借助管道作为中转实现比 sendfile 更通用的零拷贝组合，避免 sendfile 只能"文件→套接字"的限制。

在应用层，Java NIO 的 `FileChannel.transferTo()/transferFrom()` 底层会在 Linux 上调用 sendfile；许多框架的"零拷贝"开关本质就是封装这些系统调用。

需要牢记"零"的准确含义：它是 **CPU 视角**的零拷贝——数据不必复制到用户空间、不必由 CPU 搬运，而不是连 DMA 的硬件搬运也不存在。

## 六、零拷贝为什么能显著提升性能

把几种方式放在一起，收益来源非常清楚：

- **减少 CPU 拷贝**：把 CPU 从逐字节搬运中解放出来去处理业务；
- **减少上下文切换**：sendfile 用一次系统调用替代 read+write，切换减半；
- **降低内存带宽占用**：少了重复搬运，缓解内存总线压力；
- **减少缓存污染**：数据不绕路用户空间，避免无用数据挤占 CPU Cache；
- **与 Page Cache 协同**：文件内容缓存在内核中，重复发送直接命中，配合零拷贝可获得极高吞吐。

这正是消息队列转发日志、静态文件服务器分发大文件时性能突出的重要原因。

## 七、适用场景与常见误区

**适合零拷贝的场景**：静态资源与大文件下载、视频和安装包分发、消息队列（Kafka 依赖 Page Cache + sendfile 快速转发）、对象存储与 CDN 回源等——特点是"数据原样搬运、不需加工"。

**不适合的场景**：传输前需要在应用层压缩、加密、转码或改写内容，因为这些操作必须让数据进入用户态处理。

**避免神化**：

- 零拷贝不等于零开销，文件不在 Page Cache 时仍要等待磁盘 IO；
- 小文件、慢网络下，瓶颈可能在别处，零拷贝收益有限；
- 是否真正达到"零 CPU 拷贝"还取决于内核版本和网卡是否支持 SG-DMA；
- 在容器、虚拟机中，网络路径和驱动能力也会影响实际效果。

## 写在最后

零拷贝并不是某项单一技术，而是一类"让数据在内核中就近完成传输"的优化思想：传统 read/write 让数据四次拷贝、四次切换并绕路用户态，mmap 通过共享映射省掉一次 CPU 拷贝，sendfile 配合 SG-DMA 则把 CPU 拷贝降到零、切换减半，splice 进一步扩展了适用范围。理解了用户态/内核态、DMA 与缓冲区的协作，就能判断哪些链路能从零拷贝中真正受益，而不是把它当作一个到处贴的性能标签。


`https://yhmanhua.com.cn`  
`https://yhshipin.net.cn`  
`https://yhyingshi.com.cn`  
`https://yhyingyuan.com.cn`  
`https://ysbz.ac.cn`  
`https://ysdaquan.org.cn`  
`https://ystr.net.cn`  
`https://ystrdm.com.cn`  
`https://www.zymao.com.cn`  
`https://www.zjmao.com.cn`  
`https://www.yymanhua.org.cn`  
`https://www.ysmao.net.cn`  
`https://www.yrzxmh.com.cn`  
`https://www.xiaolansp.org.cn`  
`https://www.wwmh.org.cn`  
`https://www.wkvip.net.cn`  
`https://www.wfmao.com.cn`  
`https://www.ttmj.net.cn`  
`https://www.smdyy.net.cn`  
`https://www.shengmayy.com.cn`  
`https://www.shengmays.com.cn`  
`https://www.ppzxgk.com.cn`  
`https://www.nnys.net.cn`  
`https://www.maomaomh.com.cn`  
`https://www.jztv.org.cn`  
`https://www.jzsp.org.cn`  
`https://www.jzdm.com.cn`  
`https://www.juzitv.org.cn`  
`https://www.hhmh.org.cn`  
`https://www.heitaosp.com.cn`  
`https://www.dsmao.com.cn`  
`https://www.djrmh.com.cn`  
`https://www.djr.net.cn`  
`https://www.51dm.org.cn`  
`https://www.51manhua.ac.cn`  
`https://www.51shipin.net.cn`  
`https://www.91dongman.org.cn`  
`https://www.91manhua.org.cn`  
`https://www.91shipin.ac.cn`  
`https://www.baiyangyy.com.cn`  
`https://www.ccmh.org.cn`  
`https://www.dianyingdq.com.cn`  
`https://www.dman.com.cn`  
`https://www.dmghg.hk.cn`  
`https://www.dyin.net.cn`  
`https://www.dysp.net.cn`  
`https://www.hanman.net.cn`  
`https://www.hhyy.net.cn`  
`https://www.hmdm.net.cn`  
`https://www.hongtaoys.ac.cn`  
`https://www.madousp.net.cn`  
`https://www.madoutv.org.cn`  
`https://www.mahuays.net.cn`  
`https://www.mdzx.net.cn`  
`https://www.phdy.com.cn`  
`https://www.phyy.net.cn`  
`https://www.renrenys.org.cn`  
`https://www.rman.com.cn`  
`https://www.xbkf.org.cn`  
