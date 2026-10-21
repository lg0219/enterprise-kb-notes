# 企业知识库方案笔记

> 面向**私有化部署**（数据不出内网）、**企业级**要求的知识库选型笔记。
> 整理自 2026-10-21 的方案讨论，已核对相关产品与本机实测数据。
>
> 约定：编排层由 **OpenClaw** 统一承担，本文聚焦其下方的数据与检索层。

---

## 0. 一页速览

| 决策点 | 结论 |
|---|---|
| 编排层 | OpenClaw（已定，不在本文展开） |
| 解析层（PDF/Word/Excel → 文字） | **MinerU**（输出 Markdown，可直接接 OpenClaw 已有管道） |
| 嵌入模型 | 本地 **BGE-M3**（中文强、8192 token 上限、MIT 许可）；备选 EmbeddingGemma-300M |
| 向量库 | 现阶段 OpenClaw 内置 SQLite 够用；上量后换 **Qdrant** 或 **pgvector** |
| 结构化数据 | **不走 RAG**，走 NL2-SQL / 语义层 |
| 多模态 | 暂不作为主线；Qwen3-VL-Embedding-2B 是开源最强，但需另配 GPU 机器 |
| 邮件 | 暂不纳入；接入前需先确认商业邮件产品与授权模型 |

---

## 1. 三个基础概念（先分清，别混）

初学者最容易混淆的三件事，用图书馆打比方：

### 1.1 解析层 —— 把书变成可搜索的文字

原始资料是 PDF / Word / Excel，但向量检索**只能处理纯文字**。解析层负责：

- 扫描件 → 文字（OCR）
- Word → 干净文字（去页眉页脚、批注）
- Excel → 表格结构转文字（"2026年3月，华东区销售额 120 万"）
- PDF 内嵌图表 → 文字描述

> **这是整条链路的瓶颈。** 解析错了，后面模型再强也答不出来。

### 1.2 嵌入模型 —— 把文字翻译成"语义指纹"

读一段话，输出一串数字（768 / 1024 / 2048 维）。

> **关键性质：意思相近的话，数字也相近。**
> "迟到要扣多少钱" 与 "旷工如何处罚" 的指纹几乎重叠。

这正是"用意思搜"能成立的原因，而不是只能字面匹配。

### 1.3 向量数据库 —— 存指纹的仓库 + 语义搜索引擎

两件事：存所有指纹；提问时把问题也翻译成指纹，按**语义距离**排序，找出最像的片段。

### 1.4 重要澄清

> **嵌入模型 ≠ 向量数据库。**
> 嵌入模型是"翻译官"（算指纹），向量库是"仓库+搜索引擎"（存和找指纹）。
> 两者可自由搭配：换翻译官不用换仓库，换仓库不用换翻译官。

---

## 2. 完整链路

```
文档 → [解析层] → 干净文字 → [切块] → chunk → [嵌入模型] → 指纹 → [向量库] 存放
                                                                    ↑
用户提问 → [嵌入模型] → 问题指纹 → [向量库] 找最像的 → 文字片段 → [大模型] → 答案
                                              ↑
                              OpenClaw 负责编排这一段（判断查不查、怎么组织答案）
```

---

## 3. 数据分类决定走哪条路（架构骨架）

**不要试图用一套东西全包。**

| 数据 | 形态 | 正确路径 | 备注 |
|---|---|---|---|
| Excel / Word / PDF | 非结构化 | 解析 → 切块 → 向量检索 | 适用 |
| 邮件（分散在员工邮箱） | 非结构化 + **强权限** | 连接器同步 → 检索，**必须带权限过滤** | 暂不纳入 |
| 私有云盘（API 访问） | 非结构化 | 连接器同步 → 检索 | 适用 |
| 数据库 | **结构化** | **NL2-SQL / 语义层** | ❌ 不要塞进向量库 |

**最后一条是最容易踩的坑。** 把数据库表做成 RAG，问"上月华东区销售额多少"，模型会从几段文本里拼数字 —— 看起来像答案，但它是错的。结构化数据必须直查库。

真正的架构是：**一个 Agent 入口 → 判断问题类型 → 分流到「文档检索」或「SQL 查询」两条链路。**

---

## 4. 分层选型

```
用户提问
   ↓
① 编排层（OpenClaw：判断"查文档"还是"查库"，多轮追问）
   ↓
   ├─→ ② 连接器层：云盘 API / 文件上传 / 数据库连接
   ↓
   └─→ ③ 检索层：向量检索 + 全文检索 + 权限过滤
              ↑
   ④ 解析层：PDF/Word/Excel → 结构化文字
   ↓
⑤ 存储层：向量库 + 搜索引擎 + 关系库
```

### 各层私有化产品（星数/许可已核实，2026-10）

| 层 | 产品 | 备注 |
|---|---|---|
| 编排+知识库一体 | **Dify** ★159k | 生态最全；许可有附加条款，商用前需确认 |
| | **RAGFlow** ★92k, Apache-2.0 | 文档解析最强；需 ES+MySQL+MinIO+Redis 全套 |
| | **WeKnora**（腾讯）★35k | 腾讯云生态内更顺 |
| | **FastGPT** ★30k | 知识库+工作流，国内案例多 |
| | **MaxKB** ★23k, GPL-3.0 | 最轻量 |
| | **BISHENG** ★12k, Apache-2.0 | 流程编排 |
| 企业搜索型 | **Onyx** ★32k | 前身 Danswer，**连接器最多、ACL 权限模型最成熟** |
| 结构化数据 | **DB-GPT** ★20k, MIT | 专做数据库问答 / AWEL 工作流 |
| 解析层 | **MinerU** ★82k | 中文 PDF 表格/公式提取强 |
| | **Unstructured** ★15k, Apache-2.0 | 通用文档解析生态好 |
| 向量库 | **Milvus** ★46k, Apache-2.0 | 百万级以上才需要 |
| | **Qdrant** | 单二进制，内存占用小，**过滤表达力强**（适合权限过滤） |
| | **pgvector** | 复用已有 PostgreSQL，零新增组件 |
| | **sqlite-vec** | OpenClaw 内置即用，小规模够 |

---

## 5. 三个真正的难点

### 难点 1：文档解析质量 = 问答质量的天花板

若 PDF 里的表格被解析成乱序文本，后面向量检索再准也没用。

- **一体化**：RAGFlow 自带深度解析（复杂排版、表格、公式）
- **分层**：解析交给 **MinerU**（专门做这层，质量更高），输出 Markdown 喂给任意检索层
- Excel 要注意：多 sheet、合并单元格、公式 —— 需要专门的表格解析路径，不能当纯文本

### 难点 2：权限必须"检索时过滤"，不是"生成后过滤"

企业知识库最致命的翻车方式：AI 把 A 部门的薪资表回答给了 B 部门。

> 正确逻辑：**检索时**先按「谁在问 + 他的可见范围」过滤候选块，模型根本看不到无权内容。

要求：索引中每条块都带**来源 + 权限标签**，检索 SQL 里就带权限条件。Onyx 这类企业搜索产品最成熟；纯 Dify/RAGFlow 知识库需自己补这一层。

### 难点 3：邮件的授权模型（暂不纳入，但记录）

邮件分散在员工个人邮箱，是企业级最贵的一块。三条路：

1. **逐用户 OAuth 授权** —— Exchange/Office365 → Microsoft Graph API（有 delta 增量查询，最理想）；Gmail → push notification 增量；通用 IMAP → 只能轮询
2. **域管理员统一授权 + 应用身份模拟** —— 成本低，合规审计更敏感
3. **全员邮箱汇聚到统一归档**（企业常已有邮件归档/合规网关）→ 直接接归档最划算

> 选型关键：**先确认公司用的商业邮件产品**。有 Graph/API → 接 API；只有归档导出 → 接归档。开源连接器在 Onyx 里最全。

---

## 6. 本机现状（2026-10-21 实测）

### 6.1 硬件约束

| 项 | 值 | 影响 |
|---|---|---|
| GPU | **无** | 淘汰所有 2B+ 多模态模型的本地实时推理 |
| CPU | 4 核 | 嵌入/解析能跑，但慢 |
| 内存 | 7.6 GB 总，繁忙时可用 ~2.7 GB | 限制并行度与模型规模 |
| 磁盘 | 116 GB，可用 67 GB | 充足 |

### 6.2 OpenClaw 侧已跑通的部分

```
Memory index: 105/105 files · 3869 chunks
Vector store: ready      Semantic vectors: ready      Vector dims: 768
FTS: ready               Embeddings: ready
Store: ~/.openclaw/agents/main/agent/openclaw-agent.sqlite
向量扩展: sqlite-vec (vec0.so)
```

**语义检索已真正上线**（此前为 `fts-only`，即只有关键词索引）。验证方式：用不含原文关键词的查询（"小模型在本地跑图灵测试"）检索，命中项 `textScore=0` 但 `vectorScore≈0.58`，证明是纯语义召回。

### 6.3 切块现状（实测数据库）

| 项 | 值 |
|---|---|
| 总块数 | 3878 |
| 平均块长 | **814 字符** |
| 最大块长 | **1567 字符**（没有一块超过 2000） |
| memory 源 | 98 文件 / 916 块 / 平均 946 字符 |
| sessions 源 | 7 文件 / 2958 块 / 平均 773 字符 |

**切块规则**（源码 `chunkMarkdownText()`）：

1. 按 Markdown 结构边界切（标题、段落）—— 主切法
2. 代码块（```）整体保留，不从中间劈开
3. 块超长时：**换行优先 → 空格次之**；中文另有标点断点处理
4. 最后按**嵌入模型 token 上限**兜底再切一刀

**硬上限来源**：`DEFAULT_LOCAL_EMBEDDING_MAX_INPUT_TOKENS = 2048`（local 嵌入模型输入上限）。
→ 通用嵌入模型是 8192，**说明当前模型让块被切得比"本可以"更碎**。换 BGE-M3 可切出更大的块，检索时上下文更完整。

**当前不可配置**：`memory.search` 下没有 chunkSize / maxChars 类配置项，属内置默认值。

### 6.4 现有可复用资产

| 资产 | 位置 | 状态 |
|---|---|---|
| 语义检索 | OpenClaw 内置 SQLite + sqlite-vec | ✅ 已通 |
| 嵌入模型 | `~/.openclaw/models/llama.cpp/embeddinggemma-300m-qat-Q8_0.gguf` | ✅ 已跑通（Google EmbeddingGemma，300M，768 维） |
| wiki vault | `~/.openclaw/wiki/main` | bridge 模式，100 页 / 98 份导出物 |
| 每日记忆 | `workspace/memory/*.md` + MEMORY.md + USER.md | ✅ 在索引内 |
| 会话历史 | `sessions/main/*.jsonl` | ✅ 在索引内（7 文件 / 2958 块） |
| 制度 RAG（自建） | `lg0219/rag-kb` | 代码在 GitHub；资产路径写的是旧机器 `/root/...`，**尚未迁移** |

---

## 7. 嵌入模型选型

### 7.1 三条现实路径

| 方案 | 优点 | 缺点 |
|---|---|---|
| **本地 EmbeddingGemma-300M**（现有） | 免费、离线、中文可用、已跑通 | 300M 小模型语义上限低；**2048 token 上限导致块过碎** |
| **本地 BGE-M3** | 中文语义检索强，1024 维，8192 token，MIT 许可 | 需额外部署，占内存（CPU 可跑） |
| **Gemini Embedding 001/002**（API） | 质量最高；002 为多模态（文本↔图片统一空间） | **数据出内网，本场景直接出局**；按量计费 |

### 7.2 开源多模态模型（供后续参考）

| 模型 | 参数 | 维度 | 许可 | 本机可行性 |
|---|---|---|---|---|
| **Qwen3-VL-Embedding-2B**（阿里） | 2B | 2048 | 宽松 | ⚠️ 能跑但慢（无 GPU） |
| Qwen3-VL-Embedding-8B（阿里） | 8B | 4096 | 宽松 | ❌ |
| **Jina Embeddings v4** | 3.8B | 2048 | ⚠️ Qwen Research License，**非商用需另购许可** | ❌ + 许可坑 |
| Nomic Embed Text v2 MoE | 475M | 768 | Apache 2.0 | ✅ 但**纯文本** |

Milvus 公开榜单：`Qwen3-VL-Embedding-2B` 跨模态检索 R@1 = **0.945**，优于闭源的 Gemini Embedding 2（0.928）。

### 7.3 务实判断

> **企业知识库的多模态需求，通常 90% 是假需求。**

用户问的是**文字内容**。PDF 里的图表，99% 的价值在"里面的数字"，而非"图长什么样"。

**推荐路径**（成本低、见效快、全部内网）：

1. **解析**：MinerU
2. **嵌入**：BGE-M3（先用现有 EmbeddingGemma 跑通也行）
3. **向量库**：Qdrant（或 pgvector）
4. **多模态作为增强**：换 Qwen3-VL-Embedding-2B，并**单独准备带 GPU 的机器**只做嵌入计算

**替代方案**（不额外要 GPU）：用多模态模型在**解析阶段**给每张图生成一句文字描述
（「柱状图：1-3月华东区销售额逐月上升，最高120万」），这些描述当普通文字走文本嵌入。
**效果接近多模态，成本低一个数量级。**

---

## 8. 推荐架构

```
── 入库流（离线 / 定时）──────────────────────────
私有云盘 API / 本地文档
     ↓
[MinerU]      ← 解析层（新增）
     ↓
  Markdown
     ↓
[自建切分]     ← 企业制度场景建议自己接管（见下）
     ↓
   chunk
     ↓
[BGE-M3]       ← 嵌入（换掉 EmbeddingGemma）
     ↓
   指纹
     ↓
[Qdrant]       ← 向量库（独立于 OpenClaw 的 memory 库）

── 查询流（在线）───────────────────────────────
[编排层]  OpenClaw / Codex / workbuddy / 自建 agent   ← 可随时替换
   │      MCP (streamable-http)  或  HTTP API
   ↓
┌────────── 知识服务（独立进程，与编排层解耦）──────────┐
│  MCP tools   : kb_search / kb_get_source              │
│  身份校验    : 调用方 token → 可见范围                 │
│  权限过滤    : 检索时裁剪候选（不是生成后过滤）           │
│  HTTP API    : /ask /search /health（给系统与批处理用）  │
│  审计        : 哪个 agent、代表谁、问了什么、命中哪些块   │
└────────────────────────────────────────────────────┘
   ↓
 检索结果 → 编排层 → 大模型 → 答案
```

### 关键设计取舍

**为什么 MinerU 而不是 RAGFlow？**
RAGFlow 自带切分 + 向量库 + 检索 API + WebUI。既然已定"OpenClaw 统一编排"，用它等于**两套检索、两套切分、两套向量库**，OpenClaw 只能通过 API 调它，反而多一层、多一套运维。MinerU 输出正是 Markdown，**能直接喂进 OpenClaw 已有的那半截管道**。

**为什么切分可能需要自己接管？**
制度类知识库对切分很敏感。OpenClaw 内置按 Markdown 结构切，对"一条制度一块"这种诉求无法控制，可能切出半条制度、缺上下文。**企业制度场景建议在 OpenClaw 外面自建切分。**

**为什么向量库独立于 OpenClaw？**
OpenClaw 的 memory 库装的是"我的记忆"；企业知识库存的是"公司文件"。职责不同、生命周期不同、权限模型不同。

---

## 9. 编排层可换：知识服务的解耦设计

> 本章回答：**如果不是由 OpenClaw 编排，而由 Codex / workbuddy / 自建 agent 编排，架构要怎么变？**
> 结论：**这反而是更好的架构**。知识库应做成独立"知识服务"，用标准接口暴露，编排层只是它的一个消费者。

### 9.1 目标形态

```
                    ┌─ OpenClaw   (mcp.servers)
                    ├─ Codex      (codex mcp add)
知识服务 (MCP/HTTP) ─┼─ workbuddy
                    └─ 自建 agent
```

**一个实现，多个编排器** —— 这是最大的好处。换编排器 = 改一行连接配置，知识库代码零改动。

### 9.2 为什么 MCP 是最合适的接口（已核实）

| 事实 | 证据来源 |
|---|---|
| Codex 原生支持接 MCP server | `codex mcp add <NAME> --url <url>`（本机 codex-cli 0.162.1） |
| OpenClaw 可作为 MCP 客户端 | 顶层 `mcp.servers` 支持 `command`/`args`/`env`/`cwd`/`url`/`transport`/`headers`/`auth`/`oauth`/`sslVerify`/`clientCert`/`clientKey`/`toolFilter` |
| OpenClaw 还能反向暴露自己 | `openclaw mcp serve` —— 把 OpenClaw 频道以 MCP stdio 暴露给别的 MCP 客户端 |
| **修正**：顶层 `mcp.servers` **有 `oauth` 字段** | config schema 确认。与 `nodeHost.mcp.servers`（v1 路径）**明确不支持 OAuth** 是两条不同路径，不可混为一谈 |

### 9.3 什么变、什么不变

| 层 | 是否受编排层影响 |
|---|---|
| 解析（MinerU）、切块、嵌入（BGE-M3）、向量库（Qdrant） | ❌ **完全不变** |
| 权限模型、增量同步、索引、审计 | ❌ 不变 |
| **"谁来调检索"** | ✅ 变了：从编排器内部调用 → 标准接口调用 |

**唯一需要重新设计的是「身份传递」**：多个编排器时，"谁在问"必须能传进去，否则权限过滤无从谈起。
做法：**每个调用方带自己的身份 token，服务端据此判定可见范围**。这也顺带解决了审计 —— 服务端天然能记录"哪个 agent、代表谁、问了什么、命中了哪些块"。

### 9.4 三种实现方式

**A. HTTP API + 每个编排器配一个 skill**
- ✅ 已有基础：`lg0219/rag-kb` 的 `api_server.py`（`/ask`、`/health`）
- 优点：最简单、最通用，任何 agent 都能用 HTTP 工具调
- 缺点：每个编排器要写一小段调用逻辑

**B. MCP Server（推荐）**
- 用 Python / Node 的 MCP SDK 把检索逻辑包成 tool（如 `kb_search`、`kb_get_source`）
- 优点：**工具自动发现**，编排器零配置即可用；换编排器不用改知识库代码
- 缺点：多一层协议

**C. 把企业知识库塞进 OpenClaw 自己的 memory 体系**
- ⚠️ **不推荐**（若要保持编排层可换）
- OpenClaw 的 memory 绑在它的会话 / agent 模型上（MEMORY.md、workspace、dreaming），换编排器就用不了了

### 9.5 关键决策

> **MCP 与 HTTP API 两个都留，不要互相替代。**
> MCP 面向 **agent**（需要工具自动发现）；HTTP 面向 **系统**（同步任务、批处理、其他服务调用、运维排查）。
> `rag-kb` 现有的 HTTP API 保持不动，MCP 只是套一层壳。

**并且：OpenClaw 自己的 memory 体系保持原样**，专门放"我的个人记忆"，不要把企业知识混进去。
两者职责不同、生命周期不同、权限模型不同。

### 9.6 各编排器接入方式

**OpenClaw（作为客户端）**
```bash
openclaw mcp add corp-kb \
  --url https://kb.internal/mcp \
  --transport streamable-http \
  --include 'kb_search,kb_get_source'
openclaw mcp doctor corp-kb --probe     # 保存定义 ≠ 可达，probe 才算数
```
（顶层 `mcp.servers`；与 `nodeHost.mcp.servers` 是两处不同配置。）

**Codex（作为客户端）**
```bash
codex mcp add corp-kb --url https://kb.internal/mcp
```

**自建 agent**
- 优先用 MCP SDK 直连
- 集成成本最低时用 HTTP API + 一个轻量 skill

**OpenClaw（作为服务端，被别的客户端编排）**
```bash
openclaw mcp serve --token-file /path/to/token
```

---

## 10. 两条落地路线

### 路线 A：最小闭环（推荐先做）

1. 修通/复用现有嵌入与向量存储（已通）
2. MinerU 解析一小批制度（10 份）→ Markdown
3. 灌进向量库 → 暴露检索
4. OpenClaw 编排：问制度走知识库，其他走 memory
5. **做评测**：准备 20-30 个真实问题，看命中率，迭代切分

### 路线 B：正式建设

- 解析层独立服务化（MinerU API 化）
- 增量同步机制（云盘文件变更 → 只处理变动的）
- 权限标签体系 + 检索时过滤
- 多租户 / 审计日志
- 结构化数据侧接 DB-GPT / NL2SQL
- 评测回归测试集上线（改一个解析参数要知道整体是变好还是变坏）

---

## 11. 现实成本提醒

- **GPU**：本地 embedding 够用；**重排（rerank）和本地大模型才是显存大头**
- **运维**：RAGFlow 那套（ES+MySQL+MinIO+Redis+主服务）起步 5+ 容器；Onyx 也是全套
- **同步**：邮件/云盘必须做增量 + 定时，否则索引会烂
- **评测**：上线后要有问答回归测试集，否则改参数不知整体好坏
- **许可**：MinerU 是 Apache 2.0 + 附加条款，**达到商用阈值需另购许可**，部署前确认

---

## 12. 未决问题（下次继续）

1. **商业邮件服务具体是哪个**（Exchange/Graph？腾讯企业邮？Gmail？有没有邮件归档系统？）—— 决定邮件能否接、怎么接
2. **结构化数据库类型、库数量、是否多租户/行级权限** —— 决定 NL2SQL 方案复杂度
3. **是否允许另购 GPU 机器** —— 决定多模态是主线还是增强
4. **MinerU 商用许可阈值是否触及** —— 决定能否直接用
5. **制度文档的切分粒度要求** —— 决定切分是否必须自建
6. **编排层最终选谁** —— 决定知识服务的接口优先级（MCP 为主还是 HTTP 为主）
7. **是否需要多个编排器共用同一份索引** —— 决定身份 token 与审计模型的设计

---

## 附：关键事实来源

- OpenClaw 文档 `docs/nodes/mcp-and-skills.md`（node-hosted MCP v1，**明确不支持 OAuth**）
- OpenClaw 文档 `docs/tools/mcp.md`（顶层 `mcp.servers` 接入方式；含 `openclaw mcp serve` 反向暴露）
- OpenClaw config schema `.properties.mcp.properties.servers`（含 `oauth`/`toolFilter`/`codex` 等字段）
- 本机 `codex mcp add --help`（codex-cli 0.162.1，支持 `--url` 接 streamable HTTP）
- OpenClaw 文档 `docs/gateway/config-agents/models.md`（`agents.defaults.imageModel` 等模型路由）
- 本机实测：`openclaw memory status --index`、SQLite `memory_index_chunks` 表统计
- 源码：`dist/chunk-*.mjs`（切块）、`dist/embedding-chunk-limits-*.mjs`（token 上限）
- Milvus 2026 嵌入模型横评（跨模态 R@1 榜单）
- Jina / Qwen3-VL / BGE-M3 / Nomic 官方模型页
