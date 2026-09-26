# HTTP 缓存机制详解：强缓存、协商缓存与 ETag、CDN 实践

2026 年 07 月 12 日 11 时 45 分 20 秒

在众多性能优化手段中，HTTP 缓存是性价比最高的一种：浏览器和中间代理把可复用的响应保存下来，后续请求不必再回源，既减少网络往返和带宽消耗，也降低了源站压力。但缓存也是最容易被误解的机制——`no-cache` 并不是"不缓存"，资源更新后用户却"看不到新版"，F5 和 Ctrl+F5 行为完全不同。本文将梳理浏览器私有缓存与 CDN 共享缓存，讲清强缓存、协商缓存的相关头部与决策流程，以及前端工程中"HTML 协商、带 hash 的静态资源长缓存"的最佳实践。

## 一、为什么需要 HTTP 缓存

重复请求同一份未变化的资源（JS、CSS、图片）会带来不必要的网络往返和带宽开销。HTTP 缓存把响应就近保存，命中时直接使用。

缓存按位置分为两类：

- **私有缓存**：用户浏览器本地缓存（内存或磁盘），只服务该用户；
- **共享缓存**：CDN、代理上的缓存，可服务多个用户。

浏览器开发者工具中常见的 `(from memory cache)`、`(from disk cache)` 就是命中本地缓存的表现。

## 二、强缓存：不向服务器发请求

强缓存命中时，浏览器直接使用本地副本、**不发送任何请求**，状态码显示 200 并标注来自缓存。相关头部有两代：

**Expires（HTTP/1.0）**：给出一个绝对过期时间，但依赖客户端时钟，时间不准或时区错乱就会失效。

**Cache-Control（HTTP/1.1，优先级高于 Expires）**，常用指令：

- `max-age=秒数`：资源在多久内算新鲜；
- `s-maxage`：仅对共享缓存（CDN）生效的新鲜时间；
- `no-cache`：**可以缓存，但使用前必须向服务器协商验证**，并非不缓存；
- `no-store`：**真正不存储任何响应**，用于敏感数据；
- `public / private`：是否允许共享缓存保存；
- `must-revalidate`：一旦过期必须回源验证，不得使用过期副本。

## 三、协商缓存：发请求问"还能不能用"

强缓存过期后，浏览器会发起一次**协商请求**，由服务器判断本地副本是否仍有效：

**Last-Modified / If-Modified-Since**：

- 服务器在响应头给出资源最后修改时间；
- 下次请求带回该时间，服务器比较，未变化则返回 **304 Not Modified**（不带响应体），浏览器继续用本地副本。

它的局限是：精度通常只到秒，短时多次修改可能漏判；文件被编辑但内容未实质变化也会被误判为已修改。

**ETag / If-None-Match**：

- 服务器为资源生成内容指纹（如哈希），下次请求带回；
- 指纹一致返回 304，比修改时间更精确；
- 当两组头部同时存在时，**ETag 优先级更高**。

这解释了"为什么有了 Last-Modified 还需要 ETag"：前者基于时间、后者基于内容。

## 四、完整缓存决策流程

一次请求的缓存判断可概括为：

1. 本地无缓存 → 直接请求服务器并按响应头保存；
2. 本地有缓存且仍在新鲜期内 → 命中强缓存、直接使用；
3. 已过期 → 携带协商头发请求；
4. 服务器返回 304 → 复用本地副本并刷新新鲜期；返回 200 → 使用新响应并更新缓存。

## 五、Vary 与不同刷新方式

**Vary**：当同一份 URL 的响应会根据某请求头变化时（如 `Vary: Accept-Encoding`、`Accept-Language`），缓存按该头分别存储，避免把压缩版发给不支持的客户端。

**用户刷新行为**也会影响缓存：

- 普通访问/链接跳转：尽量使用强缓存；
- F5 刷新：通常会携带协商头、走验证；
- Ctrl+F5 强制刷新：忽略本地缓存、直接回源。

## 六、CDN 与共享缓存

CDN 作为共享缓存，使用 `s-maxage` 控制边缘节点的缓存时长，未命中时回源；源站通过 Cache-Control 决定哪些内容可被 CDN 缓存。提高 CDN 命中率、减少回源，是静态资源分发的关键，这与 DNS 智能调度共同构成内容分发链路。

## 七、前端工程实践

现代前端普遍采用分层缓存策略：

- **HTML 入口文件**：设置 `no-cache`，每次协商，保证总能发现新版本；
- **JS/CSS/图片等静态资源**：文件名中加入**内容 hash**，配合很长的 `max-age`（甚至 `immutable`）；内容变化时文件名随之改变，自然会被当作新资源请求；
- 这种"入口协商、带指纹资源长缓存"的组合，同时解决了"缓存命中率"和"更新及时性"。

对 API 响应则要谨慎缓存，涉及用户数据时使用 private 或 no-store；热点公共接口可短缓存以减轻源站压力。

## 八、常见误区

- 把 `no-cache` 当成不缓存——真正不存的是 `no-store`；
- 强缓存期内修改资源用户看不到更新——应通过 hash 文件名或更改 URL 解决，而不是指望原 URL 立即生效；
- 多台源服务器生成不一致的 ETag，导致缓存反复失效；
- 以为 POST 会像 GET 一样被默认缓存——通常不会；
- 对敏感信息仍设置可被共享缓存的 public。

## 写在最后

HTTP 缓存用一套"新鲜度判断 + 协商验证"的机制，让未变化的资源尽量不再回源：Cache-Control 主导强缓存，ETag 与 Last-Modified 负责过期后的精确协商，CDN 把这套逻辑扩展到离用户更近的边缘节点，而内容 hash 长缓存则是前端工程化的落地答案。理解了强缓存与协商缓存的分工、no-cache 与 no-store 的区别以及不同刷新行为，就能在资源更新不生效、缓存命中率低等问题面前，沿着缓存链路快速定位并做出正确配置。



`https://www.wfmao.autos`  
`https://www.ttdyy.autos`  
`https://www.ttdyw.autos`  
`https://www.xldytt.autos`  
`https://www.dyg.autos`  
`https://www.tktt.autos`  
`https://www.tksp.autos`  
`https://www.tkmh.autos`  
`https://www.btys.autos`  
`https://www.xptt.autos`  
`https://www.cltt.autos`  
`https://www.bttt.autos`  
`https://www.clm.autos`  
`https://www.tiantangdy.autos`  
`https://www.ttdongman.autos`  
`https://www.dytiantang.autos`  
`https://www.ngys.autos`  
`https://www.dygc.autos`  
`https://www.ngyy.autos`  
`https://www.ngdy.autos`  
`https://www.ysgongchang.autos`  
`https://www.ysdaquan.autos`  
`https://www.yingshigc.autos`  
`https://www.qjtd.autos`  
`https://www.qjw.autos`  
`https://www.dyttwang.autos`  
`https://www.x91sp.autos`  
`https://www.yhkf.autos`  
`https://www.bwpk.autos`  
`https://www.mxdongman.autos`  
`https://www.xgzm.autos`  
`https://www.fddm.autos`  
`https://www.okdmw.autos`  
`https://www.ays.autos`  
`https://www.adys.autos`  
`https://www.admwdfls.autos`  
`https://www.gaishidm.autos`  
`https://www.xjw.autos`  
`https://www.xjdongman.autos`  
`https://www.xjmanhua.autos`  
`https://www.xiangjiaoys.autos`  
`https://www.xiangjiaosp.autos`  
`https://www.91xj.autos`  
`https://www.91xjsp.autos`  
`https://www.pzsp.autos`  
`https://www.pyw.autos`  
`https://www.qww.autos`  
`https://www.wmxz.autos`  
`https://www.wmw.autos`  
`https://www.dldl.autos`  
`https://www.nmsp.autos`  
`https://www.smdy.autos`  
`https://www.smdyrk.autos`  
`https://www.mmyjs.autos`  
`https://www.hyrzbz.autos`  
