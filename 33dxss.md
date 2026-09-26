# 动态代理原理：JDK Proxy、CGLIB 与 Spring AOP、MyBatis 的底层机制

2026 年 07 月 22 日 14 时 40 分 30 秒

很多框架能力看起来像"魔法"：一个接口没有实现类却能执行 SQL，方法上加一个注解就自动开启事务，调用一个本地方法却完成了远程通信。这些"魔法"背后几乎都有同一项技术——**动态代理（Dynamic Proxy）**。它在运行时自动生成代理对象，拦截目标方法调用，在不修改原有代码的前提下织入事务、日志、限流、远程调用等逻辑。本文将从静态代理的局限讲起，剖析 JDK 动态代理与 CGLIB 的原理和差异，介绍 Byte Buddy 等字节码工具，并说明 Spring AOP、MyBatis Mapper 和 RPC 客户端是如何利用代理工作的。

## 一、代理模式与静态代理的局限

代理模式给目标对象提供一个替身，代理与目标实现相同接口、内部持有目标引用；调用方先接触代理，代理在方法前后加入逻辑（如权限、计时、缓存），再调用目标。

这种手写的**静态代理**逻辑直观，但有明显问题：

- 每个接口、每个类都要单独写代理，方法一多就类爆炸；
- 一旦接口变更，代理也要同步修改；
- 无法在运行时根据需要动态生成。

于是需要在程序运行时**自动创建代理类**，这就是动态代理。

## 二、JDK 动态代理

JDK 内置的动态代理**只能基于接口**生成代理，核心 API 是 `Proxy.newProxyInstance`，需要三个参数：

- 类加载器（用于加载生成的代理类）；
- 目标实现的接口列表；
- 一个 `InvocationHandler`（调用处理器）。

当代理对象的方法被调用，请求会转到处理器的 `invoke(proxy, method, args)`，开发者在此编写增强逻辑，再通过 `method.invoke(target, args)` 以反射方式调用目标方法。

运行时 JVM 会动态生成一个形如 `$Proxy0` 的类（它实现了指定接口），这与前文介绍的类加载机制直接相关——生成的字节码同样需要被加载。它的限制是：**目标必须实现接口**，对没有接口的普通类无能为力。

## 三、CGLIB 动态代理

**CGLIB** 采用不同思路：它基于 ASM 字节码框架，在运行时生成目标类的**子类**，通过重写非 final 方法来拦截调用。

- 用 `Enhancer` 设置父类和 `MethodInterceptor`；
- 拦截方法 `intercept` 中可前后增强，并通过 `MethodProxy` 调用父类原始方法；
- CGLIB 用 FastClass 机制为方法建立索引，避免大量反射，调用速度快。

它的优点是**不要求接口**，普通类也能代理；限制是：

- **final 类、final 方法、private/static 方法无法被代理**（final 方法不能被重写）；
- 由于是继承，创建代理时构造逻辑会更复杂。

## 四、其他字节码生成方案

除 JDK Proxy 与 CGLIB，常见字节码工具还有：

- **Javassist**：API 简单，可用接近 Java 源码的方式操作类；
- **Byte Buddy**：流式 API、易用且强大，被 Mockito、Hibernate、SkyWalking 等广泛使用；
- **ASM**：底层、性能高但需要直接操作字节码指令。

它们在"创建代理的速度、方法调用速度、使用复杂度"上各有侧重，框架会按场景选择。

## 五、Spring AOP 中的代理

Spring AOP 是动态代理最典型的应用：

- 若目标实现了接口，默认可用 JDK 动态代理；没有接口则用 CGLIB；Spring Boot 2.x 起默认统一采用 CGLIB（proxyTargetClass）；
- 切面（Aspect）、切点（Pointcut）、通知（Advice）最终都由代理对象在方法调用时执行；
- `@Transactional`、`@Async`、缓存、限流、日志等能力都建立在代理之上。

这里有一个极高频的坑——**同类内部方法自调用不生效**：在一个类的方法 A 中直接用 `this.B()` 调用同类的 B，调用的是原始对象而非代理对象，因此 B 上的事务、缓存注解会被绕过。解决办法包括：把方法拆到另一个 Bean、注入自身代理，或开启 exposeProxy 后显式调用代理。

## 六、MyBatis Mapper 与 RPC Stub

**MyBatis Mapper**：Mapper 接口没有手写实现类，MyBatis 在注册时为其创建 `MapperProxy`；调用接口方法时，代理依据方法上的注解或 XML 找到对应 SQL 执行并映射结果。

**RPC 客户端（Dubbo、Feign 等）**：为远程接口生成代理，调用"本地方法"时，代理负责参数序列化、网络发送、结果反序列化，让远程调用在编码层面像本地方法一样自然。

## 七、常见坑与实践建议

- **this 自调用绕过增强**：这是事务、缓存、限流"明明加了注解却不生效"的首要原因；
- **final/private/static 限制**：使用 CGLIB 时避免把需要增强的方法声明为 final；
- **构造与对象相等**：CGLIB 子类化会带来额外构造行为，代理对象与原始对象也不应简单混用；
- **复用代理对象**：代理只需生成一次，避免每次调用都重新创建；
- **理解代理边界**：代理织入的是方法调用，类内部状态、字段访问并不经过代理。

## 写在最后

动态代理把"增强逻辑"从业务代码中剥离出来，再在运行时自动织入：JDK Proxy 用接口和 InvocationHandler 生成代理，CGLIB 用继承和字节码子类覆盖更广的类，Byte Buddy、ASM 等则提供了底层字节码能力。Spring AOP 的事务与缓存、MyBatis 的 Mapper、RPC 的客户端 Stub，都是这一机制的具体落地。理解了"调用先进代理、代理再决定如何调用目标"这条主线，以及 this 自调用为何会绕过代理，就能看穿大量框架注解背后的真实执行路径，也能在增强不生效时快速定位根因。



`https://www.kekeys.autos`  
`https://www.qnys.autos`  
`https://www.fnyy.autos`  
`https://www.hmdm.autos`  
`https://www.bldm.autos`  
`https://www.ycy.autos`  
`https://www.lmm.autos`  
`https://www.wamh.autos`  
`https://www.lmmzxdm.autos`  
`https://www.cycdm.autos`  
`https://www.aidm.autos`  
`https://www.mddm.autos`  
`https://www.ldm.autos`  
`https://www.admw.autos`  
`https://www.dmd.autos`  
`https://www.mfmj.autos`  
`https://www.hlldb.autos`  
`https://www.bghl.autos`  
`https://www.hlmrds.autos`  
`https://www.mrds.autos`  
`https://www.yrzx.autos`  
`https://www.yrzxmh.autos`  
`https://www.aiaiw.autos`  
`https://www.jciyuan.autos`  
`https://www.shtv.autos`  
`https://www.shsp.autos`  
`https://www.shys.autos`  
`https://www.shyy.autos`  
`https://www.tiantangys.autos`  
`https://www.xxsp.autos`  
`https://www.xxdm.autos`  
`https://www.meijutt.autos`  
`https://www.flsp.autos`  
`https://www.mhdq.autos`  
`https://www.flp.autos`  
`https://www.jsmh.autos`  
`https://www.dmdq.autos`  
`https://www.dwdm.autos`  
`https://www.ecy.autos`  
`https://www.acgdongman.autos`  
`https://www.acgmanhua.autos`  
`https://www.yhacg.autos`  
`https://www.cracg.autos`  
`https://www.acgjlb.autos`  
`https://www.acgmhw.autos`  
`https://www.acgdmw.autos`  
`https://www.dongmantt.autos`  
`https://www.dmhy.autos`  
`https://www.agedongman.autos`  
`https://www.sjpk.autos`  
`https://www.mjwo.autos`  
`https://www.aimeiju.autos`  
`https://www.wjm.autos`  
`https://www.zjw.autos`  
`https://www.kjb.autos`  
`https://www.lzzj.autos`  
