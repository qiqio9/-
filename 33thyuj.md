# 深入理解虚拟内存：地址空间、页表、缺页中断与 swap、mmap

2026 年 08 月 08 日 16 时 25 分 15 秒

每个运行中的程序都以为自己独占了整台机器的内存，地址从固定位置开始、连续可用，却看不到其他进程的存在。这一"幻觉"由操作系统的**虚拟内存**机制提供：进程操作的是虚拟地址，由硬件和内核动态翻译到物理内存，暂时用不到的数据还能被换出到磁盘。虚拟内存既是进程隔离和安全的基础，也解释了 fork 为何快、malloc 为何能"超额分配"、容器为何会被 OOM Killed。本文将系统梳理虚拟地址空间、分页与页表、缺页中断、页面置换以及 mmap、大页等关键机制。

## 一、为什么需要虚拟内存

如果进程直接使用物理内存地址，会带来一系列问题：

- 多个进程可能使用相同地址、互相冲突，一个越界写就破坏其他程序甚至内核；
- 程序必须一次性全部装入内存，无法运行大于物理内存的任务；
- 内存分配和管理极其困难，安全性无法保证。

引入虚拟内存后，每个进程都拥有**独立、连续的虚拟地址空间**，通常又划分为用户空间与共享的内核空间。它带来四个核心好处：

- **隔离与保护**：进程无法访问彼此内存，页表还可标记只读、不可执行；
- **按需分配**：只在真正访问时才占用物理内存；
- **共享与映射**：多个进程可映射同一份共享库或共享内存；
- **简化编程**：程序无需关心物理内存布局。

## 二、虚拟地址到物理地址：分页与页表

虚拟内存普遍采用**分页**机制：地址空间被切成固定大小的**页（page）**，物理内存切成同样大小的**页帧（frame）**，常见页大小为 4KB，另有 2MB、1GB 的大页。

地址翻译由硬件 **MMU（内存管理单元）**完成：

- 虚拟地址被分为"页号 + 页内偏移"，页号通过**页表**查到物理页帧号，偏移保持不变；
- 由于单级页表本身极其庞大且稀疏，现代系统使用**多级页表**，只为实际使用的部分建立映射；
- 翻译结果缓存在 **TLB（转译旁视缓存）**中，避免每次访存都走多级页表。

这解释了一个前文多次出现的现象：**进程切换要更换页表、使 TLB 失效**，而线程共享同一地址空间则没有这笔开销，这正是进程切换比线程切换更昂贵的重要原因。

## 三、缺页中断：数据不在内存时怎么办

页表项中有一个"是否在内存"的标志。当访问的页尚未加载，会触发**缺页中断（Page Fault）**，由内核分类处理：

- **合法但未分配/未加载**：例如 malloc 申请后第一次写入（按需/惰性分配）、页被换出到 swap、文件映射尚未读入——内核分配物理页或从磁盘加载，更新页表后让指令重新执行，进程几乎无感；
- **非法访问**：访问未授权地址，内核向进程发送段错误信号（Segmentation Fault）。

还有两类与缺页密切相关的经典机制：

- **写时复制（Copy-on-Write）**：fork 出子进程后，双方共享只读页；任一方写入时才触发缺页、复制出独立副本，因此创建进程很快；
- **惰性分配**：malloc/申请大块内存时只承诺虚拟地址，物理内存延迟到真正触碰时才分配，所以进程的"虚拟内存"常远大于实际占用（RSS）。

缺页有成本之分：已在内存即可解决的是次要缺页（minor），需要磁盘 IO 的是主要缺页（major），后者昂贵得多。

## 四、swap、页面置换与颠簸

当物理内存不足，操作系统会把不常用的页写到磁盘上的 **swap 区**，腾出页帧给当前需要的程序；这些页再次被访问时再换入。

选择换出哪些页由**置换算法**决定，理想是 LRU，实际多用时钟算法等近似实现。如果内存严重不足、换入后很快又被换出，系统会陷入**颠簸（Thrashing）**，CPU 大量时间消耗在磁盘搬运上、近乎卡死。

因此生产环境对 swap 往往很谨慎：服务更希望内存不足时由 **OOM Killer** 明确释放进程，而不是接受不可预测的延迟；在容器中，内存限制触顶通常直接导致容器被 OOM Killed。

## 五、mmap、共享内存与大页

**mmap** 把文件或匿名内存映射到进程地址空间：

- 文件映射可像读写内存一样访问文件，配合缺页按需加载，这也是"零拷贝"中 mmap 方案的基础；
- 匿名映射可用于分配大块内存；多个进程映射同一对象即形成**共享内存**，是最快的进程间通信方式；
- 动态共享库正是通过 mmap 被多个进程共享同一份代码。

**大页与透明大页（THP）**：使用 2MB 等大页可显著减少页表数量和 TLB miss，数据库、JVM 等大内存应用常显式启用；但 THP 的后台整理可能引入延迟抖动，低延迟服务有时反而选择关闭。

## 六、实践建议

- 用 `free`、`top`/`ps` 区分虚拟内存（VIRT）与实际物理占用（RSS），不要因虚拟内存偏大而误判泄漏；
- 在容器中正确设置内存限制并理解 OOM 行为，区分堆、元空间、直接内存和页缓存的占用；
- 关注数据访问的局部性，减少随机访问造成的 TLB miss 和缺页；
- 排查延迟问题时可观察 major fault、swap IO；对延迟敏感服务评估是否关闭 swap/THP；
- 理解内存映射后，再看共享库加载、内存映射文件和零拷贝会更加清晰。

## 写在最后

虚拟内存是操作系统最成功的抽象之一：它用一套"虚拟地址 + 分页翻译 + 按需缺页"的机制，让每个进程都拥有安全、连续、可超额的私有空间，同时通过页缓存、共享内存和 mmap 实现高效共享。多级页表与 TLB 控制了翻译成本，写时复制与惰性分配让进程创建和内存申请变得廉价，而 swap 与 OOM 则是物理资源不足时的两种不同应对。理解了这套机制，进程隔离、fork、内存占用、容器 OOM 以及零拷贝等众多看似分散的知识点，就都能在同一套内存模型中被串联起来。



`https://jjysyy.jjysyy.com.cn`  
`https://jjj.jjj.sh.cn`  
`https://hgsp.hgsp.org.cn`  
`https://gdcm.gdcm.net.cn`  
`https://gcjp.gcjp.net.cn`  
`https://fldy.fldy.com.cn`  
`https://fjbnslxz.fjbnslxz.com.cn`  
`https://dyzxgk.dyzxgk.net.cn`  
`https://dyzx.dyzx.net.cn`  
`https://dyxs.dyxs.net.cn`  
`https://dywz.dywz.ac.cn`  
`https://dydq.dydq.org.cn`  
`https://ddhk.ddhk.com.cn`  
`https://crzxgk.crzxgk.com.cn`  
`https://cmys.cmys.net.cn`  
`https://zxdy.zxdy.net.cn`  
`https://57kpw.57kpw.net.cn`  
`https://91cm.91cm.com.cn`  
`https://91sp.91sp.org.cn`  
`https://91tv.91tv.org.cn`  
`https://91w.91w.org.cn`  
`https://91ys.91ys.net.cn`  
`https://91zxgk.91zxgk.net.cn`  
`https://blsp.blsp.org.cn`  
`https://cmsp.cmsp.ac.cn`  
`https://cmsq.cmsq.net.cn`  
`https://rbyjbkyeq.rbyjbkyeq.com.cn`  
`https://rhjp.rhjp.net.cn`  
`https://sgsp.sgsp.net.cn`  
`https://shxfyy.shxfyy.com.cn`  
`https://shyk.shyk.net.cn`  
`https://smdy.smdy.ac.cn`  
`https://smdyrk.smdyrk.com.cn`  
`https://sywz.sywz.com.cn`  
`https://tmcm.tmcm.org.cn`  
`https://tmsryy.tmsryy.net.cn`  
`https://tmyy.tmyy.org.cn`  
`https://wyyy.wyyy.net.cn`  
`https://xjsp.xjsp.ac.cn`  
`https://xjys.xjys.org.cn`  
`https://xsjys.xsjys.net.cn`  
`https://ylysdq.ylysdq.com.cn`  
`https://ysai.ysai.org.cn`  
`https://yzjp.yzjp.net.cn`  
`https://yzyq.yzyq.net.cn`  
`https://17cgw.17cgw.net.cn`  
`https://17cyqc.17cyqc.com.cn`  
`https://51bl.51bl.org.cn`  
`https://51cg.51cg.org.cn`  
`https://51cghl.51cghl.com.cn`  
`https://51cgsp.51cgsp.com.cn`  
`https://51cgw.51cgw.net.cn`  
`https://51hl.51hl.ac.cn`  
`https://xhp.xhp.net.cn`  
`https://spzxgk.spzxgk.net.cn`  
`https://xgyy.xgyy.org.cn`  
`https://shyy.shyy.ac.cn`  
`https://shys.shys.ac.cn`  
