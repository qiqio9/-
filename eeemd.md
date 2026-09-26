# RAG 检索增强生成：让大模型用上私有知识的完整流程与关键技术

2026 年 06 月 18 日 09 时 20 分 50 秒

大语言模型（LLM）能力很强，但它的知识固化在训练参数里，有明确的截止日期，不了解企业内部文档，还可能"一本正经地编造"（幻觉）。要让模型回答私有、实时、专业的问题，**RAG（检索增强生成）** 成为当前最主流的工程方案：先从知识库中检索出相关内容，再把它们作为上下文交给模型生成答案，相当于让模型"开卷考试"。它把搜索引擎、向量表示和大模型结合起来。本文将讲清 RAG 解决什么问题、离线建库与在线问答的完整流程、文档切分与 Embedding、混合检索与重排，以及防幻觉、评估和工程落地中的关键点。

## 一、为什么需要 RAG

直接使用通用大模型有几个固有局限：

- **知识截止**：训练完成后发生的事情它不知道；
- **不懂私有数据**：企业的制度、产品手册、历史工单不在训练语料中；
- **幻觉问题**：缺乏依据时可能编造看似合理的答案；
- **更新成本高**：靠重新训练或微调来注入新知识，周期长、代价大。

RAG 的核心思想是把"让模型记住所有知识"转变为"**回答前先去知识库查找、基于查到的内容作答**"。与微调相比，RAG 在知识时效性、可溯源性和成本上优势明显；微调更适合改变模型风格或固化某种能力，二者常配合使用。

## 二、RAG 的整体流程

RAG 分为离线建库和在线问答两条链路。

**离线（数据准备）**：

1. 文档加载与解析（PDF、网页、表格等）；
2. 把长文档**切分为片段（chunk）**；
3. 用 Embedding 模型把每个片段转为向量；
4. 连同原文和元数据写入**向量数据库**。

**在线（问答）**：

1. 把用户问题也转为向量；
2. 在向量库中检索最相似的若干片段（top-k）；
3. 把检索内容与问题拼成提示词（prompt）；
4. 交给大模型生成答案，并标注引用来源。

## 三、文档切分（Chunking）

切分是 RAG 中最影响效果、却最容易被草率处理的环节：

- 片段**太大**，检索到的内容掺杂无关信息、浪费上下文窗口；
- 片段**太小**，语义不完整、答案缺少必要背景；
- 常见做法是固定长度切分并保留**重叠（overlap）**，或按标题、段落等结构边界切分；
- 进阶采用"父子块"：用小块检索、命中后返回更大的父块以补充上下文；
- 同时保存元数据（来源、时间、权限），用于过滤和引用。

## 四、Embedding 与向量检索

**Embedding（嵌入）**把文本映射成一个高维数值向量，语义相近的文本在向量空间中距离较近。

- 检索时计算问题向量与各片段向量的**相似度**（常用余弦相似度或点积），取最接近的 top-k；
- 向量数据库通过近似最近邻索引（如 HNSW、IVF）在海量向量中快速检索，而不必逐条计算；
- Embedding 模型和向量维度的选择、多语言支持，都会直接影响召回质量。

## 五、提升检索质量

检索的质量决定了 RAG 的上限——"垃圾进、垃圾出"。常用增强手段：

- **混合检索**：把向量语义检索与 BM25 关键词检索结合（前文搜索原理中的稀疏检索），既懂语义又不漏专有名词、编号；
- **Rerank 重排**：先粗召回较多候选，再用交叉编码器（cross-encoder）对问题与每个片段精细打分、重新排序；
- **查询改写**：把口语问题改写、扩展为多个查询，或用 HyDE 先生成假设答案再检索；
- **元数据过滤与权限控制**：按时间、部门过滤，并保证用户只能检索到有权查看的内容。

## 六、生成与防幻觉

检索只是提供素材，生成阶段仍需约束：

- 在 prompt 中明确要求"**仅根据提供的上下文回答**"，并分配好 token 预算、注意长上下文中的"中间遗忘"；
- 要求模型标注答案引用了哪些片段，便于核验；
- 当检索不到相关内容时，让模型明确回答"不知道"，而不是强行编造；
- 通过流式输出改善用户等待体验。

## 七、工程落地要点

- **增量同步**：源文档更新、删除时向量库要同步更新，可借鉴变更捕获的思路；
- **效果评估**：分别评估检索（命中率、上下文召回）与生成（答案正确性、忠实度），建立可量化的评测集；
- **缓存**：对 Embedding 和高频问答结果做缓存以降低成本和延迟；
- **多轮对话**：需结合历史把追问改写成独立查询再检索；
- **RAG 与 Agent 的区别**：RAG 主要为模型补充知识，Agent 则进一步能规划、调用工具并执行动作，二者可组合。

## 写在最后

RAG 用"先检索、后生成"的方式，把大模型从"闭卷、记忆有限、会编造"转变为"开卷、可引用私有知识、答案可溯源"：文档切分决定证据粒度，Embedding 与向量索引实现语义召回，混合检索和重排把最相关的证据顶到前面，而严格的生成约束和评估体系则抑制幻觉。它不是简单接一个向量库，而是一项涵盖数据处理、检索、生成和持续评测的系统工程。理解了这条链路，就能在大模型应用开发中，让模型既发挥语言能力，又建立在真实、可控的知识基础之上。



`https://www.avbobo.autos`  
`https://www.guifu.autos`  
`https://www.gfzxgk.autos`  
`https://www.ggf.autos`  
`https://www.wyzy.autos`  
`https://www.wyzydm.autos`  
`https://www.wyzyzxgk.autos`  
`https://www.xjie.autos`  
`https://www.xiaojiedy.autos`  
`https://www.rstx.autos`  
`https://www.xjdyzxgk.autos`  
`https://www.alssw.autos`  
`https://www.hmmfk.autos`  
`https://www.mrzx.autos`  
`https://www.manhwa.autos`  
`https://www.yymanhua.autos`  
`https://www.shengqimh.autos`  
`https://www.qmw.autos`  
`https://www.kmxk.autos`  
`https://www.qzfsmh.autos`  
`https://www.qzfs.autos`  
`https://www.rmw.autos`  
`https://www.aqts.autos`  
`https://www.labbgxz.autos`  
`https://www.labbgxzhm.autos`  
`https://www.hmsr.autos`  
`https://www.wmh.autos`  
`https://www.labj.autos`  
`https://www.lajq.autos`  
`https://www.lajqdm.autos`  
`https://www.mgsp.autos`  
`https://www.nantongmh.autos`  
`https://www.nantongdm.autos`  
`https://www.boylove.autos`  
`https://www.ntp.autos`  
`https://www.xmpk.autos`  
`https://www.xxfz.autos`  
`https://www.nbjc.autos`  
`https://www.yljc.autos`  
`https://www.lgjc.autos`  
`https://www.qmjc.autos`  
`https://www.lgsq.autos`  
`https://www.ssjc.autos`  
`https://www.hmqxk.autos`  
`https://www.kfzys.autos`  
`https://www.mhxq.autos`  
`https://www.duoman.autos`  
`https://www.rzxy.autos`  
`https://www.hanmandq.autos`  
`https://www.mhdqmfyd.autos`  
`https://www.manwu.autos`  
`https://www.fangju.autos`  
`https://www.wmwm.autos`  
`https://www.wmxwmdm.autos`  
`https://www.wmxwm.autos`  
`https://www.wmxwmzxgk.autos`  
`https://www.wangmxwm.autos`  
`https://www.fhwang.autos`  
`https://www.ffg.autos`  
