# Go 垃圾回收原理：三色标记、写屏障与低延迟并发 GC

2026 年 10 月 05 日 20 时 26 分 54 秒

自动垃圾回收把程序员从手动 free 的内存泄漏和悬垂指针中解放出来，但不同语言对 GC 的取舍截然不同。JVM 提供了分代、可压缩、多种风格的回收器，而 **Go 选择了一条优先压低停顿时间的路线**：采用并发的三色标记-清除，让 GC 大部分时间与用户代码同时运行，换来很短且稳定的暂停。理解三色抽象和写屏障，不仅能看懂 Go GC，也是理解几乎所有现代追踪式 GC 的通用钥匙。本文将讲清 GC 的基本问题、三色标记过程、为什么必须有写屏障、一次 GC 的完整阶段，以及 GOGC 调优和与 JVM GC 的对比。

## 一、GC 要解决什么

在堆上动态分配对象后，核心问题是：**哪些对象还活着、哪些已经没用可以回收**。

追踪式 GC 的基本思路是从"**根（GC Roots）**"出发——包括栈上的局部变量、全局变量等——沿着引用关系向外遍历：

- 从根可达的对象视为存活；
- 不可达的对象视为垃圾、可以回收。

这与前文栈与堆、JVM 垃圾回收一脉相承，区别在于不同运行时如何组织和执行这个过程。

## 二、Go GC 的设计目标

Go 的 GC 有几个鲜明特征：

- **低延迟优先**：首要目标是把每次暂停压到亚毫秒级，而不是追求最高吞吐；
- **并发标记**：标记工作主要与用户 goroutine 同时进行；
- **非分代、非移动**：不做新生代/老年代划分，也不移动、不压缩对象，采用标记-清除；
- 这种取舍契合服务端程序：大量 goroutine、对请求尾延迟敏感，宁可牺牲一点吞吐和内存，也要停顿短而稳。

## 三、三色标记抽象

Go 用"三色"形象描述标记过程：

- **白色**：尚未被访问，标记结束时仍是白色的对象将被回收；
- **灰色**：对象自身已被标记，但它引用的对象还没全部扫描；
- **黑色**：对象及其引用都已扫描完毕。

算法从根对象开始（根标灰），不断从灰色集合取出对象、把它引用的对象标灰、再把自己标黑；灰色集合清空时标记完成，剩余白色对象即不可达。这是追踪式标记的通用模型。

## 四、为什么需要写屏障

如果标记时程序完全暂停，三色过程不会出错。但 Go 是**并发标记**——用户程序同时在修改指针，就可能发生"**漏标（对象消失）**"：

- 一个黑色对象新建立了对某白色对象的引用；
- 而该白色对象原本从灰色对象可达的路径又被删除；
- 于是这个本该存活的白色对象再也不会被扫描、被错误回收。

为防止漏标，需要维护三色不变式，做法是在指针写入时插入**写屏障**进行拦截：

- **Dijkstra 插入写屏障**：把新被指向的对象标灰（强不变式）；
- **Yuasa 删除写屏障**：在指针被覆盖时保留线索（弱不变式）；
- Go 1.8 起采用**混合写屏障**，结合两者优点，使栈上指针基本无需屏障、避免了早期需要 STW 重新扫描整个栈的开销。

## 五、一次 GC 的完整阶段

Go 一次回收大致分为四段，其中只有很短的 Stop-The-World：

1. **Sweep Termination**：结束上一轮清扫，做短暂 STW、开启写屏障；
2. **Mark**：与用户程序并发地执行三色标记；
3. **Mark Termination**：短暂 STW、关闭屏障并完成收尾；
4. **Sweep**：并发回收白色对象占用的内存。

得益于并发和混合写屏障，两个 STW 窗口都非常短，对在线服务的影响很小。

## 六、触发机制与内存分配

GC 的触发方式包括：

- 堆使用量按比例增长时触发，由 **GOGC** 控制（默认 100，即存活堆翻倍时触发）；
- 周期性触发，或显式调用 `runtime.GC`；
- Go 1.19 起支持 **GOMEMLIMIT** 设置软内存上限，防止容器中内存失控。

内存分配器采用 mcache/mcentral/mheap 的分层缓存（每个 P 有本地缓存，思想类似池化），减少锁竞争。由于 GC 不移动对象，长期运行可能产生碎片，这是不压缩方案的固有代价。

## 七、与 JVM GC 的对比

- **JVM**：分代、可移动压缩，提供从吞吐优先到低延迟的多种回收器，调优参数丰富；
- **Go**：非分代、不移动、并发标记-清除，停顿短且行为可预测，几乎"开箱即用"；
- 前者给你大量选择、也要求更多调优，后者用统一设计换取简单和稳定的低延迟。

## 八、实践建议

- 从源头减少堆分配：优先值类型、理解逃逸分析（呼应栈与堆），让小对象留在栈上；
- 用 `sync.Pool` 复用高频临时对象、复用 buffer、预分配 slice 容量，都是"池化"思想；
- 必要时通过 GOGC、GOMEMLIMIT 平衡 CPU、内存与停顿；
- 用 pprof 分析分配热点和 GC 行为，而不是凭感觉调参；
- 监控 GC 停顿频率与持续时间，避免指针过密、过小对象泛滥增加标记负担。

## 写在最后

Go GC 用三色标记把"找出存活对象"的过程清晰地表达出来：灰色对象是待办工作、黑色对象已扫描完毕、白色对象最终被清除；而并发标记下的漏标风险，由写屏障（乃至混合写屏障）来封堵，使大部分回收能与用户代码并行，只留下极短的暂停。配合 GOGC、GOMEMLIMIT 和分层分配器，它实现了简单而稳定的低延迟。把它与 JVM 的分代压缩式 GC 对照，再结合逃逸分析和 sync.Pool，就能在不同语言的内存管理之间看清共同原理与各自取舍，写出对 GC 友好的高性能代码。




`https://www.dydm.ac.cn`  
`https://www.lfdq.org.cn`  
`https://www.dsxyy.ac.cn`  
`https://www.dsxys.sh.cn`  
`https://www.dstr.net.cn`  
`https://www.dmw.gd.cn`  
`https://www.dmbz.hk.cn`  
`https://www.dhsp.org.cn`  
`https://www.lfzx.sh.cn`  
`https://www.dddyw.bj.cn`  
`https://www.cyg.ac.cn`  
`https://www.lfdmdq.cn`  
`https://www.cgtt.org.cn`  
`https://www.bzmh.ac.cn`  
`https://www.aicrmj.net.cn`  
`https://www.aidm.hk.cn`  
`https://www.aidj.ac.cn`  
`https://www.aigm.ac.cn`  
`https://www.aimj.hk.cn`  
`https://www.aimh.ac.cn`  
`https://www.gzkf.org.cn`  
`https://www.gzzf.hk.cn`  
`https://www.gzzm.org.cn`  
`https://www.hgdj.ac.cn`  
`https://www.hgmfmj.com.cn`  
`https://www.hgmj.net.cn`  
`https://www.hgmj.org.cn`  
`https://www.hlmj.org.cn`  
`https://www.lfb.org.cn`  
`https://www.lfk.net.cn`  
`https://www.lfwz.ac.cn`  
`https://www.mzys.org.cn`  
`https://www.npdm.hk.cn`  
`https://www.okdm.hk.cn`  
`https://www.okdmw.hk.cn`  
`https://www.okrhdm.com.cn`  
`https://www.qkcy.net.cn`  
`https://www.sm.hk.cn`  
`https://www.zxkp.hk.cn`  
`https://www.yzyqeq.org.cn`  
`https://www.yzyq.ac.cn`  
`https://www.yysp.net.cn`  
`https://www.xtzf.hk.cn`  
`https://www.yqeq.net.cn`  
`https://www.wrczdm.com.cn`  
`https://www.wrcz.org.cn`  
`https://www.wmh.bj.cn`  
`https://www.wmh.ac.cn`  
`https://www.wlgkmh.net.cn`  
`https://www.wldm.net.cn`  
