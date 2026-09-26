# 容器到底是什么：从 namespace、cgroup 到镜像与 Docker 原理

2026 年 09 月 01 日 20 时 30 分 40 秒

Docker 让"容器"这个词变得家喻户晓，但很多人对容器的理解仍停留在"轻量级虚拟机"上。这个比喻有助于入门，却没有触及本质：容器里并没有运行一个完整的操作系统，也没有模拟硬件，它就是宿主机上一个**被隔离了视图、被限制了资源、还被切换了文件系统根目录的普通进程**。理解了这句话，容器的快速启动、镜像分层、共享内核带来的风险等现象就都能解释得通。本文将从容器与虚拟机的区别出发，依次讲清 Linux namespace 如何制造隔离、cgroup 如何约束资源、rootfs 与分层镜像如何提供独立环境，以及一个容器被创建出来的完整过程。

## 一、容器不是虚拟机

先对比两种虚拟化方式。

**虚拟机**：通过 Hypervisor 在物理硬件之上虚拟出 CPU、内存、网卡等，每台虚拟机安装并运行一个**完整的客户机操作系统**。它隔离彻底，但镜像大、启动慢（通常分钟级）、资源开销高。

**容器**：不虚拟硬件、不运行独立内核，所有容器**共享宿主机的操作系统内核**，容器内的进程本质上仍是宿主机内核调度的普通进程，只是被加上了几道"边界"。因此容器启动快（秒级甚至更快）、体积小、密度高。

这几道边界由三类 Linux 能力共同构成，可以概括为容器的公式：

**容器 = namespace（隔离视图） + cgroup（限制资源） + rootfs（独立文件系统）**

## 二、namespace：让进程"看到"一个独立的世界

namespace（命名空间）负责回答"进程能看到什么"。它对进程的系统视图进行隔离，使进程以为自己运行在一个独立的系统中。Linux 主要提供以下几种 namespace：

- **PID namespace**：进程号空间隔离。容器内第一个进程的 PID 是 1，可以有自己独立的进程树，容器内看不到宿主机和其他容器的进程；
- **Network namespace**：拥有独立的网卡、IP 地址、端口空间、路由表和回环接口，因此不同容器可以同时监听各自的 80 端口；
- **Mount namespace**：隔离挂载点和文件系统视图，使容器拥有自己的目录结构；
- **UTS namespace**：隔离主机名和域名；
- **IPC namespace**：隔离信号量、消息队列等进程间通信资源；
- **User namespace**：隔离用户与用户组，可将容器内的 root 映射为宿主机上的普通用户；
- **Cgroup namespace**：隐藏宿主机的 cgroup 层级信息。

这些隔离通过 `clone()`、`unshare()`、`setns()` 等系统调用实现。需要强调的是，namespace 隔离的是"**视图**"而非真正的物理边界——容器进程仍然由同一个宿主机内核处理。这也是容器与虚拟机相比，隔离强度较弱、存在内核逃逸风险的根本原因。

## 三、cgroup：限制进程"能用多少"

如果说 namespace 管的是"看得到什么"，那么 **cgroup（控制组）**管的就是"用得了多少"。它把一组进程组织起来，对其资源使用进行**限制、统计和优先级控制**，避免某个容器吃光整台宿主机资源。

常见的 cgroup 子系统（控制器）包括：

- **cpu / cpuset**：限制 CPU 使用配额、绑定特定核心、设置调度权重；
- **memory**：限制内存使用上限，超出且无法回收时，容器中的进程会触发 OOM 被内核杀死；
- **blkio**：限制磁盘 IO 带宽；
- **pids**：限制可创建的进程数量，防止 fork 炸弹。

管理员通过 cgroup 暴露在 `/sys/fs/cgroup` 下的接口进行配置（存在 cgroup v1 与 v2 两代实现，v2 采用统一层级）。正是 cgroup 让"一台机器上跑很多容器、彼此不抢资源"成为可能。

## 四、rootfs 与镜像：让进程"自带运行环境"

仅有隔离和限制还不够，进程还需要一个独立、完整的文件系统环境，这就是 **rootfs（根文件系统）**。

容器的 rootfs 包含应用程序、运行依赖、配置和目录结构，但**不包含内核**（内核由宿主机共享）。rootfs 让应用把运行环境一起打包，实现"在我机器上能跑、在你机器上也能跑"的一致性。

Docker 镜像在 rootfs 基础上引入了**分层与联合文件系统**：

- 镜像由多个**只读层**叠加而成，底层是基础系统，往上是依赖和应用；
- 运行时通过 **UnionFS（常用 OverlayFS）** 把这些只读层（lower）和一个为容器新建的**可写层**（upper）联合挂载成统一视图（merged）；
- 容器对文件的修改采用**写时复制（Copy-on-Write）**：首次修改某文件时，先把它从只读层复制到可写层再改，原始镜像层不被破坏。

这种分层带来了层缓存和镜像复用——多个容器可以共享相同的基础层，节省空间并加速分发。可以把镜像理解为"类"、容器理解为"实例"：一个镜像可以同时启动多个相互独立的容器。

## 五、一个容器是如何被创建出来的

容器的构建和运行遵循 **OCI（开放容器标准）**，Docker 生态中各组件分工明确：Docker CLI 接收命令，containerd 负责容器的生命周期管理，最终由 **runc** 这样的底层运行时真正创建容器。

创建一个容器，底层大致经历以下步骤：

1. 拉取并解压镜像，准备好分层的 rootfs；
2. 通过 `clone()` 配合 namespace 标志，创建一组处于新命名空间中的进程；
3. 配置 cgroup，为其设置 CPU、内存等资源限额；
4. 通过 `pivot_root`（或 chroot）把进程的根目录切换到准备好的 rootfs，并完成必要挂载；
5. 在新的命名空间和文件系统视图中 `exec` 启动用户指定的入口程序。

容器启动后，其第一个进程（PID 1）承担特殊责任：它要正确处理信号、回收子进程，否则容器内可能产生无人回收的僵尸进程。这也是为什么简单地用 shell 启动程序可能出现信号传递不到、`docker stop` 等待超时等问题。

## 六、安全边界与工程实践

**共享内核意味着风险**：容器逃逸、内核漏洞会影响宿主机和其他容器。对强隔离、多租户不可信负载，可结合虚拟机式容器（Kata Containers、Firecracker）获得接近虚拟机的隔离。

**默认收紧权限**：尽量不以 root 运行应用，裁剪 Linux capabilities，使用只读根文件系统，配合 seccomp 限制可调用的系统调用，以及 AppArmor/SELinux 等强制访问控制。

**资源限制必须显式设置**：不设内存、CPU 限制的容器在异常时可能拖垮整台宿主机。

**容器与编排协同**：单机的容器解决了"打包与运行"，而容器的调度、服务发现、弹性伸缩则由 Kubernetes 等编排平台承接，二者构成云原生的两层基础。

## 写在最后

容器的神奇之处，恰恰在于它并没有发明什么全新的执行模型，而是把 Linux 早已具备的几项能力组合了起来：namespace 给进程一个"世界只有这么大"的视图，cgroup 给它划定资源的边界，rootfs 和分层镜像让它把运行环境随身携带。看穿"容器即被约束的普通进程"这一本质，就能理解它为何轻快、为何要警惕共享内核、为何 PID 1 如此重要，也能在安全加固、资源配置和故障排查时，从现象一路追溯到这些底层机制。


`https://www.hgdy.ac.cn`  
`https://www.hgdyw.net.cn`  
`https://www.hgmh.ac.cn`  
`https://www.hmmh.com.cn`  
`https://www.hmtt.com.cn`  
`https://www.hmw.sh.cn`  
`https://www.hmwz.net.cn`  
`https://www.hmzxgk.com.cn`  
`https://www.hmzxyd.com.cn`  
`https://www.hmzyk.com.cn`  
`https://www.hongtao.ac.cn`  
`https://www.hongtaotv.net.cn`  
`https://www.hongtaoys.org.cn`  
`https://www.hsdm.ac.cn`  
`https://www.hsxs.com.cn`  
`https://www.hybz.net.cn`  
`https://www.hyrzbz.com.cn`  
`https://www.jmmh.org.cn`  
`https://www.khyy.ac.cn`  
`https://www.klmh.net.cn`  
`https://www.larw.com.cn`  
`https://www.larwhm.com.cn`  
`https://www.llss.net.cn`  
`https://www.manwa.org.cn`  
`https://www.mmjxmh.com.cn`  
`https://www.qjtd.net.cn`  
`https://www.rrmj.org.cn`  
`https://www.ssmh.org.cn`  
`https://www.ttmh.ac.cn`  
`https://www.waiwaimh.com.cn`  
`https://www.waman.org.cn`  
`https://www.wamh.com.cn`  
`https://www.wmtt.net.cn`  
`https://www.wwmh.net.cn`  
`https://www.wynmh.com.cn`  
`https://www.xemh.com.cn`  
`https://www.zzzyk.com.cn`  
`https://www.yzyjrb.net.cn`  
`https://www.ysdaquan.net.cn`  
`https://www.xkcm.org.cn`  
`https://www.xingkongyy.com.cn`  
`https://www.xingkongys.net.cn`  
`https://www.xfdm.net.cn`  
`https://www.xcyy.ac.cn`  
`https://www.xcys.ac.cn`  
`https://www.wamh.net.cn`  
`https://www.tydm.net.cn`  
`https://www.wam.ac.cn`  
`https://www.tangxcm.net.cn`  
`https://www.sywz.net.cn`  
`https://www.rhzq.net.cn`  
`https://www.qztv.net.cn`  
`https://www.ntdm.ac.cnnt`  
`https://www.ngyy.org.cn`  
`https://www.ngys.org.cn`  
`https://www.ngdy.com.cn`  
`https://www.mwwz.net.cn`  
`https://www.mwfzs.com.cn`  
`https://www.mwamh.com.cn`  
