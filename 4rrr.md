# 跨域与 CORS：同源策略、预检请求、凭证携带与常见解决方案

2026 年 06 月 25 日 15 时 50 分 20 秒

前端开发者几乎都遇到过控制台那行红字：`No 'Access-Control-Allow-Origin' header is present...`，这就是典型的跨域报错。很多人把"跨域"理解成请求被服务器拒绝，其实它是**浏览器的安全机制**：请求往往已经发出、服务器也处理了，只是浏览器不把响应结果交给脚本。理解同源策略、CORS 的简单请求与预检请求、携带 Cookie 时的特殊限制，能让我们快速定位并正确解决跨域问题，而不是盲目加注解或关闭安全策略。本文将系统梳理这些机制以及开发代理、Nginx、JSONP 等方案的取舍。

## 一、什么是同源策略

两个 URL 只有在**协议、域名、端口三者完全相同**时才算同源。例如 `https://a.com` 与 `http://a.com`（协议不同）、`a.com` 与 `api.a.com`（域名不同）、`a.com:80` 与 `a.com:8080`（端口不同）都属于跨源。

**同源策略**是浏览器最核心的安全机制之一，它限制页面中的脚本读取来自其他源的响应数据，目的是防止恶意网站在用户不知情时读取其在其他网站的私有数据。

需要强调：跨域限制发生在**浏览器端**，针对的是"脚本能否读取响应"，而不是服务器之间的通信。

## 二、跨域请求到底发生了什么

一个常见误区是"跨域请求根本没发出去"。实际上：

- 对多数请求，浏览器照常把请求发给服务器，服务器也可能已经执行了写操作；
- 浏览器在拿到响应后，检查其中是否带有允许该源的 CORS 头，若没有就**拦截响应、不交给 JavaScript**，并在控制台报错。

这意味着用跨域 POST 做下单等操作，可能"前端报错、后端却已处理"，所以后端不能因为"浏览器会拦"就忽略幂等。

此外，`<img>`、`<script>` 等标签加载资源、普通表单提交并不受同源策略约束——这正是早期 JSONP 能工作的原因。

## 三、简单请求

CORS 把请求分为两类。满足以下条件的称为**简单请求**：

- 方法为 GET、HEAD 或 POST；
- 只使用安全的标准请求头；
- POST 的 `Content-Type` 仅限特定值（如表单类型，不含 `application/json`）。

简单请求会**直接发送**，并自动带上 `Origin` 头说明来源；服务器在响应中返回 `Access-Control-Allow-Origin`（允许的源，或 `*`），浏览器校验通过后才让脚本读取结果。

## 四、预检请求（非简单请求）

当请求"不简单"——例如使用 PUT/DELETE 方法、`Content-Type: application/json`、或带有自定义头（如 Authorization）——浏览器会先自动发一个 **OPTIONS 预检请求**：

- 用 `Access-Control-Request-Method`、`Access-Control-Request-Headers` 询问服务器是否允许接下来的真实请求；
- 服务器通过 `Access-Control-Allow-Methods`、`Access-Control-Allow-Headers`、`Access-Control-Max-Age`（预检结果缓存时间）应答；
- 预检通过后，浏览器才发送真实请求。

预检的意义在于：在真正可能产生副作用的请求发出之前先征求许可，避免非简单请求直接作用于服务器。因此开发者工具里常看到每个跨域调用对应"OPTIONS + 真实请求"两条记录。

## 五、携带凭证 Cookie

默认情况下，跨域请求**不携带 Cookie**。要让浏览器带上凭证，需要两端配合：

- 前端设置 `withCredentials = true`；
- 后端返回 `Access-Control-Allow-Credentials: true`；
- 此时 `Access-Control-Allow-Origin` **不能再用 `*`**，必须回显具体的源。

这与前文的 Cookie-Session、CSRF 防护直接相关，配置遗漏是跨域登录态丢失的常见原因。

## 六、常见解决方案

- **CORS（标准方案，推荐）**：在后端或 API 网关统一配置允许的源、方法和头，前后端分离的主流做法；
- **开发服务器代理**：本地开发时让 dev server 把 `/api` 请求同源转发到后端，浏览器看到的是同源；
- **Nginx 反向代理**：生产环境让前端与接口在同一域名下、由 Nginx 内部转发，从根本上同源；
- **JSONP**：利用 `<script>` 标签可跨域的特性，只支持 GET 且存在安全隐患，属于已过时的兼容方案；
- **WebSocket** 不受同源策略约束，但服务端仍应校验 Origin 防止未授权连接。

## 七、常见错误与安全注意

- 缺少 Allow-Origin、返回了多个值，或与请求 Origin 不匹配；
- 配置了 `*` 又开启凭证，浏览器直接拒绝；
- **OPTIONS 预检被登录鉴权拦截返回 401/403**，导致真实请求发不出去，鉴权过滤器应放行预检；
- 多源场景应配合 `Vary: Origin`，避免 CDN/缓存把某一源的响应错发给其他源；
- **安全红线**：不要为了省事在开启 credentials 的同时无条件反射任意 Origin，这等于让任何网站都能带用户凭证访问你的接口；应维护可信白名单、精确回显。

## 写在最后

跨域不是后端报错，而是浏览器同源策略在保护用户数据：简单请求直接发送并由响应头授权，非简单请求先经 OPTIONS 预检，携带 Cookie 时必须用具体源而非通配符。CORS 是标准且推荐的解法，开发代理和 Nginx 则通过"制造同源"绕开问题，JSONP 已基本退出舞台。理解了"请求其实已发出、浏览器拦的是响应"以及预检和凭证的规则，既能快速消除控制台报错，也能在放开跨域时守住白名单和凭证这道安全底线。



`https://www.llfh.autos`  
`https://www.cllj.autos`  
`https://www.cld.autos`  
`https://www.btcl.autos`  
`https://www.btss.autos`  
`https://www.btxz.autos`  
`https://www.btzz.autos`  
`https://www.btlm.autos`  
`https://www.btssyq.autos`  
`https://www.btclss.autos`  
`https://www.cldh.autos`  
`https://www.clx.autos`  
`https://www.clp.autos`  
`https://www.clmbt.autos`  
`https://www.clmao.autos`  
`https://www.clhu.autos`  
`https://www.zzb.autos`  
`https://www.ciliss.autos`  
`https://www.cldd.autos`  
`https://www.lwcl.autos`  
`https://www.lwlt.autos`  
`https://www.slwltdlw.autos`  
`https://www.clxm.autos`  
`https://www.xmcl.autos`  
`https://www.clxgw.autos`  
`https://www.clbao.autos`  
`https://www.zzss.autos`  
`https://www.clbgw.autos`  
`https://www.wqcl.autos`  
`https://www.skrbt.autos`  
`https://www.lwltgw.autos`  
`https://www.cilitt.autos`  
`https://www.nmcl.autos`  
`https://www.dypjb.autos`  
`https://www.crks.autos`  
`https://www.dycrb.autos`  
`https://www.91dy.autos`  
`https://www.crdyin.autos`  
`https://www.crduoyin.autos`  
`https://www.dyzxgk.autos`  
`https://www.91dyin.autos`  
`https://www.91khl.autos`  
`https://www.91khlrk.autos`  
`https://www.kdw.autos`  
`https://www.kdsq.autos`  
`https://www.huangguosp.autos`  
`https://www.hgdjgw.autos`  
`https://www.hyyx.autos`  
`https://www.hgame.autos`  
`https://www.galagame.autos`  
`https://www.galgame.autos`  
`https://www.galgameyxwz.autos`  
`https://www.galgamezywz.autos`  
`https://www.hywz.autos`  
