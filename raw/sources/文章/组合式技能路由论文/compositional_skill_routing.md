# LLM Agent 的组合式技能路由：分解、检索与组合

**作者：** Xueping Gao  
**机构：** Alibaba Cloud，中国杭州  
**邮箱：** hellogxp@gmail.com

---

## 摘要

LLM Agent 越来越依赖外部技能——可复用的工具规范——但真实世界任务往往需要组合多个技能，而不只是选择一个技能。

我们将其形式化为**组合式技能路由**问题：给定一个复杂用户查询和一个大型技能库，将查询分解为原子子任务，为每个子任务检索合适技能，并组合出一个可执行计划。

我们提出 **SKILLWEAVER**，一个“分解-检索-组合”框架，它结合了基于 LLM 的任务分解器、带 FAISS 索引的双编码器技能检索器，以及依赖感知的 DAG 规划器。

为支持评估，我们引入 **COMPSKILLBENCH**，这是一个由 300 个组合式查询构成的基准，覆盖 2,209 个真实 MCP 服务器技能，横跨 24 个功能类别，来源于公开 MCP 生态系统。

我们的实验表明，任务分解质量是主要瓶颈：标准 LLM 分解在步骤级别只能达到 34.2% 的类别召回率。为解决这一问题，我们提出**迭代式技能感知分解**（Iterative Skill-Aware Decomposition, SAD），这是一个检索增强的反馈循环，能够迭代地使分解结果与可用技能对齐。

SAD 在单次迭代中将分解准确率从 51.0% 提高到 67.7%（+32.7%，Wilcoxon p < 10^-6）；基于 DA 条件的分析确认，正确粒度是有效检索的前提（当 DA=1 时，CatR@1 从 34% 上升到 41%）。SKILLWEAVER 将上下文窗口消耗降低超过 99%，迁移实验也确认了其泛化能力（即便目标类别不在检索池中，相对 DA 仍提升 +35.6%）。

---

## 1 引言

大语言模型（LLM）的 Agent 范式已经超越了单轮生成，扩展到工具使用、规划和多步骤任务执行。

现代 LLM Agent 中出现的一个关键架构模式是使用**技能**：模块化、可复用的工具规范，它们定义具体能力，并附带何时以及如何调用这些能力的说明（Anthropic，2025）。

我们按照 Anthropic 的 SKILL.md 规范使用“skill”这一概念；技能不同于传统 API，区别在于它更强调结构化自然语言文档和可组合性元数据。随着 Agent 技能库不断增长——已有仓库包含数千个社区贡献技能——一个基础路由问题出现了：给定一个用户查询，Agent 应该调用哪些技能？

已有工作将技能路由视为单技能选择（Zheng 等，2025），但真实世界查询经常需要多个技能。例如，“下载数据集、转换它，并创建可视化报告”需要 API 客户端、数据处理器和可视化工具。

我们将其形式化为**组合式技能路由**（图 1）：给定查询 q 和技能库 S，生成一个有序技能序列 `[s1, ..., sk]`，其中每个 `si` 处理一个原子子任务。

我们提出 **SKILLWEAVER**，这是一个三阶段框架，用来解决这一问题：

1. **分解（Decompose）：** 基于 LLM 的任务分解器，将复杂查询拆分为原子子任务，每个子任务恰好需要一个技能。
2. **检索（Retrieve）：** 双编码器检索器，基于技能元数据上的语义相似度，为每个子任务识别候选技能。
3. **组合（Compose）：** 一个兼容性感知的规划器草图（公式 4），利用技能间兼容性为每一步选择技能。我们通过一个试点执行研究验证端到端可行性（附录 I，76.7% 链路完成率），同时将受控评估重点放在已识别的瓶颈上，即分解-检索阶段。

为了评估组合式技能路由，我们构建了 **COMPSKILLBENCH**，这是该任务的第一个专门基准。

COMPSKILLBENCH 包含 300 个组合式查询，覆盖 2,209 个真实技能，横跨 24 个功能类别，并带有真实技能链和三个难度等级。技能来自公开 MCP 服务器生态系统（2,200+ 注册服务器），并经过去重以保证质量。

我们的实验得到几个关键发现：

- **分解是瓶颈：** 标准 LLM 分解在 2,209 个真实技能组成的池上只达到 34.2% CatR@1。基于 DA 的条件分析表明，正确步骤数是门控因素（当 DA=1 时 CatR@1 上升到 41.2%），这确认了分解粒度是主要限制因素。
- **SAD 缩小差距：** 我们提出的迭代式技能感知分解（SAD）是一个检索增强反馈循环，使分解结果与可用技能词汇对齐。它在一次迭代中将 DA 从 51.0% 提高到 67.7%（+32.7%，p < 10^-6）。剩余的 CatR@1 差距（37% 对比 @10 上限 72%）被一个 LLM listwise 重排序器试点部分弥合（@1 相对提升 +10.3%，p<0.01；附录 K），从而将“作为未来工作的交叉编码器重排序”转变为一个经实证验证的杠杆。
- **元数据足以用于检索：** 仅编码元数据即可达到 69.0% 的 CatR@10，说明即使在 2,209 个技能之间，简洁的技能元数据也携带强判别信号。
- **SAD 可泛化到未见技能：** 迁移实验表明，在类别级留出（相对 DA 提升 +35.6%）和随机技能留出（+23.2%）两种情况下，SAD 都保留优势，确认其学习的是词汇层面的引导，而不是特定技能层面的记忆。

---

## 2 相关工作

### 工具选择与路由

API 检索（Patil 等，2024；Qin 等，2023）、文档匹配（Hao 等，2024）和层次化路由（Zheng 等，2025）研究单工具选择。

与我们工作最接近的 SkillRouter（Zheng 等，2025）使用双编码器进行单技能路由。层次化/自反思 Agent（Du 等，2024）和工具创建框架（Yuan 等，2025）扩展了工具使用，但仍将选择视为单工具问题或逐步问题。CRAFT（Yuan 等，2025）与我们的组合阶段最相关：它通过 LLM 驱动地过滤大型 API 池，为每个查询创建专门工具集。然而，CRAFT 不执行显式多步骤分解——它假设一个扁平的查询到工具集映射——并通过单轮任务上的执行成功率进行评估。相比之下，SKILLWEAVER 处理需要有序多技能链的组合式查询，SAD 在分解和检索之间提供跨阶段反馈，这在 CRAFT 流水线中没有对应物。这些方法都没有联合优化组合式任务中的分解粒度、检索和技能间兼容性。

### 工具增强 LLM 基准

API-Bank（Li 等，2023）、ToolQA（Zhuang 等，2024）和 TaskBench（Shen 等，2023b）对工具使用进行基准测试，但使用的是固定或小规模工具集。我们的 COMPSKILLBENCH 是第一个针对数千个技能上的组合式路由的基准。

### 任务分解与规划

提示策略（Wei 等，2022；Zhou 等，2022）、Decomposed Prompting（Khot 等，2023）、规划框架（Huang 等，2022；Wang 等，2023；LangChain，2023）以及 Agent 系统（Yao 等，2023；Shen 等，2023a）探索了使用静态模板进行 LLM 分解。SAD 与已有检索增强方法的区别在于反馈方向：Self-RAG（Asai 等，2024）、ReAct（Yao 等，2023）和 Reflexion（Shinn 等，2023）将检索到的证据输入生成或动作步骤（输出侧），在固定计划下优化模型生成内容；SAD 则将检索到的技能反馈到分解输入中（输入侧），在检索最终确定之前纠正计划粒度。输入侧反馈是更困难的设计选择——它要求模型根据与不完美第一轮候选的部分关键词重叠来修订计划——但它特别适合组合式技能路由，因为这里的瓶颈是让分解词汇与技能池匹配，而不是优化单个生成步骤。

### MCP 生态与工具发现

MCP 协议（Anthropic，2024）用 10,000+ 个服务器标准化 Agent-工具集成。渐进式发现（Qin 等，2023）系统性地处理工具过载问题。最近的零样本工具发现工作（Wang 等，2025）通过协议级优化显著减少 token；ToolACE（Liu 等，2025）整理了用于微调的大规模工具调用数据集。TaskWeaver（Qiao 等，2024）这类代码优先的 Agent 框架解决执行编排问题，但不解决技能检索问题。这些工作是互补的：它们解决 Agent 如何访问工具，而我们解决给定查询时应组合哪些技能。

### 检索增强生成

我们将双编码器检索（Karpukhin 等，2020）适配到技能上，并将其向上游扩展，用检索提示来告知分解过程。

---

## 图 1：SKILLWEAVER 概览

示例查询：

> “下载数据集、转换它，并创建可视化报告”

阶段 1：分解（LLM）

- `t1`：下载数据集
- `t2`：转换数据
- `t3`：创建报告

技能库：`N = 2,209`

阶段 2：检索（双编码器 + FAISS）

- top-k：`api-client`, `http-fetch`, ...
- top-k：`csv-parser`, `etl-pipeline`, ...
- top-k：`chart-gen`, `dashboard`, ...

阶段 3：组合（DAG + 兼容性）

- `s1`：api-client
- `s2`：csv-parser
- `s3`：chart-gen

图 1 展示了 SKILLWEAVER 的整体流程。一个查询被分解为子任务，每个子任务通过双编码器检索匹配到技能，然后组合为 DAG。虚线箭头表示 SAD 反馈循环（§4.4）。

---

## 3 问题形式化

### 技能库

一个技能库 `S = {s1, ..., sN}` 包含 N 个技能。每个技能 `si` 是一个元组 `(ni, di, bi, Ci)`，其中 `ni` 是名称，`di` 是自然语言描述，`bi` 是完整规范正文（说明、示例、配置），`Ci ⊆ C` 是来自分类体系 C 的一组功能类别。

### 组合式技能路由

给定一个需要多种能力的复杂查询 q，目标是生成：

1. 一个由 K 个原子子任务组成的分解：`D(q) = [t1, ..., tK]`。
2. 一个技能分配：`σ : [t1, ..., tK] → S^K`，将每个子任务映射到一个技能。
3. 一个执行计划（DAG）：`G = (V, E)`，指定步骤之间的依赖关系。

组合式路由函数为 `f : q → (D, σ, G)`，其优化目标是：

```text
max_{D,σ,G} α Σ_{k=1}^{K} rel(tk, σ(tk)) + (1-α) Σ_{(i,j)∈E} compat(σi, σj)     (1)
```

其中，`rel(·)` 衡量子任务-技能相关性，`compat(·)` 衡量技能间兼容性，`α ∈ [0,1]` 控制相关性与兼容性的权衡（在公式 4 中实例化）。虽然对公式 1 进行联合优化一般是不可处理的，但我们的级联流水线（§4）提供了一个可处理的近似；SAD（§4.4）进一步通过将检索信号反馈给分解来收紧这一近似。

---

## 4 方法：SKILLWEAVER

SKILLWEAVER 通过三个级联阶段实现组合式技能路由（图 1）。

### 4.1 阶段 1：任务分解

给定复杂查询 q，任务分解器使用一个指令微调 LLM 生成一个有序的原子子任务列表：

```text
D(q) = LLM(psys, puser(q)) = [t1, ..., tK]     (2)
```

其中 `psys` 指示模型以 JSON 字符串数组形式输出子任务，每个子任务恰好需要一个技能。

### 4.2 阶段 2：技能检索

对于每个子任务 `tk`，我们使用双编码器（all-MiniLM-L6-v2，384 维）检索 top-m 候选：

```text
cand(tk) = top-m_{s∈S} cos(Eq(tk), Es(s))     (3)
```

我们比较两种表示：仅元数据表示 `(ns ⊕ ds)`，以及包含正文的表示 `(ns ⊕ ds ⊕ bs[:2000])`。嵌入经过 L2 归一化，并使用 FAISS（Johnson 等，2019）建立索引以进行精确内积搜索。未来工作可以探索领域适配编码器或交叉编码器重排序替代方案（§8）。

### 4.3 阶段 3：组合

给定每一步检索到的候选，组合阶段选择最终技能分配。选择目标结合了检索相关性与步骤间兼容性：

```text
σ(tk) = arg max_{s∈cand(tk)} α · sim(tk, s) + (1-α) · c̄k(s)     (4)
```

其中 `c̄k(s)` 是与前序步骤的平均兼容性分数（通过 I/O 类型强制转换、类别 Jaccard 和关键词共现来衡量），并且 `α = 0.5`（在 `[0.3, 0.7]` 上稳健；见附录 E）。步骤之间的依赖通过语言标记和 I/O 重叠检测，从而在可能时生成用于并行执行的 DAG。

**当前评估范围。** 本文聚焦于分解-检索阶段，我们将其识别为主要瓶颈（§7）。组合阶段（公式 4）作为框架的架构性补全提出；对它进行单独评估需要真实兼容性标注，而我们当前基准不提供这些标注。我们通过试点执行研究验证端到端可行性（附录 I），其中 SAD 路由的计划达到 76.7% 链路完成率。

### 4.4 技能感知分解（SAD）

一个关键洞察是，LLM 分解器会生成与技能元数据对齐较差的通用描述。我们提出**技能感知分解**（SAD），这是一种迭代对齐过程：给定第 i 次迭代的分解 `D(i)(q)`，为每个子任务检索顶部候选，构建提示集合 `H(i)`，然后重新分解：

```text
D(i+1)(q) = LLM(psys, pSAD(q, H(i)))     (5)
```

这在有限的技能提示集合空间上定义了一个不动点迭代：由于 `|H(i)| = H`，且每个元素来自有限技能库 S，序列 `{H(i)}` 必然收敛。实践中，我们发现一次迭代足以使 DA 收敛（§7.8），因此两遍变体成为默认设置。

#### 算法 1：迭代式技能感知分解（SAD）

**输入：** 查询 q、技能库 S、检索器 R、提示数量 H=15、最大迭代次数 T、收敛阈值 τ=0.6  
**输出：** 精炼后的分解 `D(T)(q)`

```text
1: D(0)(q) ← LLM(psys, q)                         {普通分解}
2: for i = 0 to T - 1 do
3:     candk ← R.retrieve(tk, H) for each tk ∈ D(i)
4:     H(i) ← 从 ⋃k candk 中取 top-H 技能
5:     if i > 0 and J(H(i), H(i-1)) > τ then
6:         return D(i)(q)                          {已收敛}
7:     end if
8:     D(i+1)(q) ← LLM(psys, pSAD(q, H(i)))
9: end for
10: return D(T)(q)
```

即使 `D(0)` 较差，SAD 仍能工作：不精确描述仍然可以通过部分关键词重叠浮现相关技能，从而提供一个词汇桥梁（算法 1）。

---

## 5 基准：COMPSKILLBENCH

### 5.1 技能池构建

我们从公开 MCP（Model Context Protocol）服务器生态系统（Anthropic，2024）构建技能池，该生态系统收录了 2,200+ 个社区注册工具服务器。我们从 curated `awesome-mcp-servers` 注册表中抽取技能条目，该注册表聚合了带有描述、类别和源 URL 的 MCP 服务器。我们应用以下整理流程：

1. **抽取：** 解析 2,228 个服务器条目，包含名称、描述、类别和仓库 URL。
2. **质量过滤：** 移除描述短于 15 个字符或主要由徽章图片组成的条目，将数量减少到 2,213 个条目。
3. **去重：** 合并标准化名称完全相同的条目，得到 2,209 个唯一技能。
4. **分类：** 通过人工整理映射，将注册表的 49 个细粒度标签映射为 24 个规范功能类别（表 1）。

### 5.2 查询生成

组合式查询通过组合来自不同类别的技能来生成多步骤任务。

**难度等级：**

- **简单**（150 个查询）：2 个技能，2 个类别。
- **中等**（100 个查询）：3 个技能，3 个类别。
- **困难**（50 个查询）：4-5 个技能，4-5 个类别。

每个查询都关联真实子任务描述、真实技能 ID、所需类别和顺序执行顺序。该基准总计 300 个查询，覆盖 23 个类别（技能数 ≥5 的类别）。

**查询构造。** 查询由跨类别组合的模板动词短语生成。真实子任务描述使用类别特定动词短语（例如“查询数据库”“发送通知”），不会直接复制技能名称或描述，从而确保检索成功需要真正的语义匹配，而不是词汇重叠。

### 表 1：COMPSKILLBENCH 中前 10 个技能类别

| 类别 | 数量 | 示例 |
|---|---:|---|
| Developer Tools | 357 | eslint-mcp, github-actions |
| Finance | 270 | stripe-mcp, plaid-server |
| Integrations | 229 | zapier-mcp, n8n-server |
| Knowledge Mgmt | 180 | notion-mcp, obsidian-server |
| Search/Extraction | 140 | firecrawl, serper-mcp |
| Security | 122 | snyk-mcp, vault-server |
| Communication | 109 | slack-mcp, email-server |
| Databases | 104 | postgres-mcp, redis-server |
| Cloud Infra | 87 | aws-mcp, terraform-server |
| Code Execution | 69 | jupyter-mcp, sandbox-server |
| 另外 14 个类别 | 542 total | - |

表 1：COMPSKILLBENCH 中前 10 个技能类别（总共 24 个）。完整技能池包含来自公开 MCP 生态系统的 2,209 个技能。

### 5.3 评估指标

我们在三个粒度上进行评估。

**步骤级指标：**

- **Skill Recall@k（R@k）：** 真实技能出现在 top-k 候选中的步骤比例。
- **Category Recall@k（CatR@k）：** 正确类别中的任意技能出现在 top-k 中的步骤比例。这个宽松指标更实际，因为一个类别中的许多技能在功能上可以互换。

**链路级指标：**

- **Chain Exact Match：** 所有步骤都选择精确真实技能的查询比例。
- **Chain Category Match（Chaincat）：** 每个查询中选择正确类别技能的步骤平均比例。

**分解准确率（DA）。** 预测子任务数量与真实值完全匹配的查询比例。注意，DA 是一个严格结构指标；一个真实步骤数为 3 的查询如果被分解成 4 步（其中多出一个有效中间步骤），仍然得到 DA=0。

**宽松 DA（DA±1）。** 预测步骤数在真实值 ±1 范围内的查询比例。这捕捉了分解粒度大致正确但由于任务边界含糊而相差一步的情况（例如隐式认证步骤）。

我们主要使用 DA 来诊断分解粒度；CatR@1 是主要检索质量指标。

---

## 6 实验设置

**LLM 分解器。** Qwen2.5-7B-Instruct（Qwen Team，2024）作为主要分解器。生成设置：`τ = 0.1`，`top_p = 0.9`，最大 256 token。

**检索器。** all-MiniLM-L6-v2（384 维）作为双编码器，使用 FAISS IndexFlatIP 在 2,209 个技能上进行精确内积搜索。索引构建耗时 15 秒；每个查询批次的检索延迟 <15ms。除非另有说明，我们设置 `k = 10` 用于检索。

**对比方法。** 我们比较：

- **Vanilla：** 不带技能提示的标准分解。
- **+SAD（H=15）：** 单次迭代的技能感知分解。
- **Iterative SAD：** 最多 3 次额外迭代，并进行收敛监控。

**硬件。** 实验在单张 NVIDIA V100-SXM2-16GB GPU 上运行。7B 模型完全放入 GPU 内存（15GB VRAM）。

---

## 7 结果

### 7.1 主要结果

表 2 展示了所有配置下的主要实验结果。

### 表 2：COMPSKILLBENCH 主要结果

| 方法 | DA | DA±1 | CatR@1 | CatR@10 | Chaincat |
|---|---:|---:|---:|---:|---:|
| **基线（qwen-max，50 个查询）** ||||||
| LLM-Direct（展示 100 个技能） | 0.900 | 0.960 | 0.211 | - | - |
| ReAct-style（迭代式）† | 0.000 | 0.040 | 0.154 | - | - |
| **完整流水线（SKILLWEAVER）——Qwen2.5-7B，300 个查询** ||||||
| Vanilla | 0.510 | 0.713 | 0.342 | 0.686 | 0.040 |
| + SAD（H=15） | **0.677** | **0.843** | **0.370** | **0.703** | **0.073** |
| **SKILLWEAVER + SAD——qwen-max，50 个查询** ||||||
| qwen-max Vanilla | 0.660 | 0.820 | 0.359 | - | - |
| qwen-max + SAD | **0.920** | **0.980** | **0.394** | - | - |

表 2：COMPSKILLBENCH 上的主要结果（2,209 个技能，24 个类别，300 个查询）。DA：严格分解准确率（步骤数完全匹配）。DA±1：允许预测步骤数在真实值 ±1 内的宽松 DA，用来捕捉粒度大致正确的情况。CatR@k：正确类别中的技能出现在 top-k 的步骤比例。Chaincat：所有步骤都选择正确类别技能的查询比例。SAD 的 DA 提升高度显著（Wilcoxon p < 10^-6，n=300）；∆DA 的 bootstrap 95% CI 为 [+10.3%, +23.0%]。CatR@1 呈方向性提升（p=0.17；CI: [-0.005, +0.062]）。†ReAct 不产生显式分解；DA=0 反映协议不匹配，而非系统失败。

**关键发现。** 在 2,209 个真实 MCP 技能组成的池上，普通分解达到 CatR@1 = 34.2%、DA = 51.0%（DA±1 = 71.3%）。SAD 将 DA 提高到 67.7%（相对 +32.7%，p < 10^-6），将 DA±1 提高到 84.3%（+18.2%），并将 CatR@1 方向性提高到 37.0%（+8.2%；统计细节见 §8）。这确认了分解粒度是主要瓶颈——一旦模型生成正确数量的子任务，检索质量就随之提升（以 DA=1 为条件时 CatR@1 上升到 41.2%）。68.6-70.3% 的 CatR@10 表明，检索器在大多数步骤的 top-10 中都能浮现一个正确类别技能；通过重排序缩小 @10 到 @1 的差距是自然的下一步（§8）。

### 7.2 难度分析

SAD 的提升在所有难度等级上都一致：简单查询 DA 从 44.7% 提高到 63.3%（+41.6%），中等查询从 66.0% 提高到 78.0%（+18.2%），困难查询从 40.0% 提高到 60.0%（+50.0%）。困难查询上的最大相对提升确认了：随着任务复杂度增长，分解变得越来越重要，SAD 也越来越有价值。CatR@1 的增益更温和（相对 +5-16%），说明即便分解改进，在完整的 2,209 技能池上，检索精度仍然具有挑战性。

### 7.3 基线

**LLM-Direct（上限估计）。** 我们向 qwen-max（一个远大于我们 7B 分解器的专有模型）提供 100 个技能名称（包含真实技能），并要求它直接为查询选择工具。尽管 DA 接近完美（90%——强模型容易正确分解），CatR@1 只有 21.1%，远低于 SKILLWEAVER 的 37.0%。这一上限估计确认，仅在 prompt 中列出技能是不够的——即使是强得多的模型也无法匹配带 SAD 的检索式路由，说明技能匹配挑战不仅仅是模型容量问题。

**ReAct-style。** 一个迭代式思考-动作-观察 Agent（qwen-max）达到 DA=0%，因为 think-act-observe 循环在没有显式分解指导的情况下，将多步骤任务折叠为单个动作。这确认了组合式路由需要显式结构化分解。

### 7.4 改写鲁棒性

为验证结果不是由模板查询模式夸大造成的，我们使用 qwen-max（temperature=0.7）改写 50 个查询，并重新运行流水线。SAD DA 从 66.0% 小幅下降到 62.0%（-4 个百分点；注意：66.0% 是 50 查询子集基线，而表 2 中完整 300 查询为 67.7%）；原始查询和改写查询之间的逐查询 DA 一致率为 72%，说明跨表面形式变化时分解质量稳定。CatR@1 也稳定（改写后 38.2%，原始 38.3%）。为进一步验证，我们扩展到由 7B 模型自身改写的另外 150 个查询（更严格，因为同一模型生成并评估）；SAD DA 从 65.3% 下降到 59.3%（-6 个百分点），一致率为 66%，CatR@1 保持稳定（34.5%→33.4%）。在两个集合合计 200 个改写查询上，DA 下降幅度较小（≤6 个百分点），确认 SAD 的收益不是表面形式记忆的产物。

SAD 的收益还延伸到与技能池零文本重叠的人类风格查询（表 6）：宽松 DA±1 从 30.5% 提高到 50.5%（相对 +66%），确认即使在开放式步骤边界且严格 DA 自然较低的情况下，它也能泛化到模板模式之外。

### 7.5 跨模型验证

为验证 SAD 的收益不是模型特定的，我们在 50 查询子集上使用两个额外模型进行评估。Qwen2.5-14B-Instruct 的 Vanilla DA=32.0%，但 SAD DA=68.0%（+36 个百分点），CatR@1 从 29.0% 上升到 42.4%。qwen-max（一个可类比 GPT-4 的专有模型）达到 Vanilla DA=66.0%、SAD DA=92.0%（相对 +39.4%）。14B Vanilla DA（32%）反而低于 7B Vanilla（51%）这一反直觉结果，反映了 14B 更强的过度分解倾向：14B Vanilla 平均每个查询生成 4.72 个预测步骤（真实均值 2.94），而 7B 为 3.62。SAD 将 14B 平均值降低到 3.18 步，暴露出分解粒度是一种与模型能力正交的失败模式。SAD 的提示将 14B 输出锚定回正确的词汇粒度，从而获得最大绝对增益——这是 SAD 是粒度纠正器而不是能力增强器的最清晰证据。

### 7.6 消融：粒度 vs. 质量

**DA 是检索前提。** 以 DA=1 的查询为条件进行分析表明，正确分解是有效检索的前提：CatR@1 从 34.2%（无条件）跳到 41.2%（仅 DA=1），CatR@10 达到 81.6%。这意味着当分解器产生正确步骤数时，检索已经相当有效——瓶颈在于达到这一点。

**SAD 的机制。** SAD 修复了 75 个查询（25%）中 Vanilla 分解产生错误步骤数的问题。在这些被修复的查询上，CatR@1 从 23.6%（坏分解）提高到 37.0%（正确分解）。关键的是，在 128 个两种方法都产生正确 DA 的查询上，它们的 CatR@1 在统计上相同（41.7% vs 40.9%，p=0.97）。这说明 SAD 的 CatR@1 增益完全来自通过粒度纠正解锁正确检索，而不是来自词汇对齐本身。

**步骤数约束基线。** 为进一步将粒度与语义对齐隔离，我们在全部 300 个查询上使用一个 oracle 步骤数 prompt 运行 Vanilla 7B（“分解为恰好 K* 个原子子任务”，其中 K* 是真实值）。这个受约束基线达到 DA=99.3%（几乎完美粒度）和 CatR@1=39.8%，与 SAD 的 DA=1 条件 CatR@1=41.2% 非常接近（∆=1.4 个百分点）。由此得出两个结论：（i）SAD 的主要机制确实是粒度纠正——oracle 步骤数信号恢复了大部分 CatR@1 增益；（ii）即使有 oracle 粒度，CatR@1 也在接近 40% 处平台化，而 CatR@10 达到 79.1%，暴露出一个独立的表示层瓶颈（40% top-1 vs 79% top-10），这推动我们将交叉编码器重排序作为未来工作。

### 7.7 上下文窗口分析

暴露全部 2,209 个技能会消耗约 884K token；SKILLWEAVER 将其减少到每个查询 2-5 个技能（表 3）。

### 表 3：上下文窗口消耗

| 策略 | 工具数 | 估计 token | 降低比例 |
|---|---:|---:|---:|
| 所有工具（朴素） | 2,209 | 约 884K | - |
| Top-k 检索 | 10 | 约 4,000 | 99.5% |
| SKILLWEAVER（平均） | 2.9 | 约 1,160 | 99.9% |

表 3：上下文窗口消耗。“估计 token”只计算暴露给任务执行 LLM（§4）的工具，假设每个序列化技能约 400 token；它不包括 SAD 分解器第二遍输入，其中 H=15 个提示会增加固定约 1,100 token，并在所有查询中共享。组合式路由将任务时上下文减少两个数量级。

### 7.8 收敛分析

算法 1 允许多次迭代；我们评估额外轮次是否能在标准单次迭代 SAD 之外进一步改进路由。表 4 报告了全部 300 个查询上的逐轮指标（Qwen2.5-7B，H=15）。

第 1 轮捕捉到完整 DA 提升（51.3%→67.0%），第 2-3 轮没有进一步增益；CatR@1 在第 2 轮达到峰值（38.9%），之后在第 3 轮下降（36.1%）。提示 Jaccard 单调上升（0.32→0.47→0.52），说明逐步稳定——与较小池相比收敛更慢，反映了更大的词汇空间 `C(2209,15)`。对于延迟敏感部署，我们默认 `T=1`；当检索精度关键时使用 `T=2`。

### 表 4：迭代式 SAD 收敛

| 轮次 | DA | CatR@1 | CatR@10 | Chaincat | Jaccard |
|---|---:|---:|---:|---:|---:|
| 0（Vanilla） | 0.513 | 0.351 | 0.690 | 0.040 | - |
| 1（SAD-1） | 0.670 | 0.370 | 0.704 | 0.060 | 0.324 |
| 2（SAD-2） | 0.653 | 0.389 | 0.690 | 0.073 | 0.473 |
| 3（SAD-3） | 0.653 | 0.361 | 0.695 | 0.077 | 0.524 |

表 4：迭代式 SAD 收敛（7B，H=15，n=300，2,209 个技能）。与表 2 的小差异（例如第 0 轮 DA=0.513 vs 0.510）来自迭代流水线中的步骤对齐差异；表 2 是权威结果。第 1 轮捕捉大部分 DA 增益。提示 Jaccard 单调上升，说明技能词汇逐渐稳定。DA 在第 1 轮后平台化，而 CatR@1 在第 2 轮达到峰值，说明一次迭代足以改善 DA，但可选第二次迭代提高检索精度。

### 7.9 对未见技能的泛化

为测试 SAD 是否过拟合特定技能池，我们在两种留出条件下评估（表 5）。

1. **类别迁移：** 移除 24 个类别中的 2 个（security、code-execution；191 个技能）后，剩下 62 个查询至少有一个目标类别不在索引中。SAD 仍然在这些查询上相对提高 DA +35.6%，说明即使确切目标类别缺失，来自相关类别的提示也能提供足够的词汇脚手架。
2. **技能级留出：** 随机移除 20% 技能（442/2,209）会影响 139 个查询（评估 100 个）。SAD 在受影响查询上获得 +23.2% 相对 DA 增益，而完整池上为 +32.7%——这表明存在适度退化，但收益持续存在，确认 SAD 利用的是技能库的结构性词汇，而不是记忆特定技能-提示映射。

### 表 5：迁移实验

| 条件 | 模式 | DA | CatR@1 | ∆DA 相对 |
|---|---|---:|---:|---:|
| **留出 2 个类别（移除 security + code-exec）** |||||
| 缩减池（n=62） | Vanilla | 0.452 | 0.195 | - |
| 缩减池（n=62） | +SAD | 0.613 | 0.213 | +35.6% |
| **80/20 技能切分（留出 442 个技能）** |||||
| 80% 池（n=100） | Vanilla | 0.560 | 0.348 | - |
| 80% 池（n=100） | +SAD | 0.690 | 0.366 | +23.2% |
| **完整池（参考）** |||||
| 完整池（n=300） | Vanilla | 0.510 | 0.342 | - |
| 完整池（n=300） | +SAD | 0.677 | 0.370 | +32.7% |

表 5：迁移实验（7B，H=15，2,209 个技能）。即使目标技能或类别不在检索池中，SAD 也能改进路由。在类别级留出下（移除 2/24 个类别，2,018 个训练技能），SAD 实现 +35.6% 相对 DA 增益。在随机技能留出下（移除 442/2,209 个），增益为 +23.2%，确认 SAD 的词汇引导可泛化到特定技能池之外。

### 7.10 错误分析与 SAD 机制

Vanilla 失败案例（检查 50 个）分为过度分解（36%）、通用描述（28%）、词汇不匹配（22%）和分解不足（14%）；Oracle R@1 = 99.5% 将分解隔离为瓶颈。SAD 的提示提供技能级语义引导——具体工具名称和描述——将子任务表述锚定到可检索词汇，并且提示集合在第 2 轮稳定（Jaccard >0.52），说明它进行的是一致词汇识别，而不是随机探索（完整分类见附录 J）。

---

## 8 讨论

**级联瓶颈。** 我们的 DA 条件分析（§7.5）揭示了一个级联结构：分解粒度控制检索，正确 DA 将 CatR@1 从 34% 提高到 41%。SAD 是一个粒度纠正器，而不是词汇对齐学习器——约 75% 的 CatR@1 增益来自 Vanilla 产生错误步骤数的查询；在 DA 匹配查询上，SAD 的逐步增益在统计上为零（p=0.97）。步骤数约束的 oracle 基线也确认了这一点：将 K 固定为真实值可恢复 DA=99.3%，但 CatR@1 只有 39.8%（距离 @10 上限仍有 36 个百分点残差差距），这确定了下一个瓶颈是表示层重排序，而不是更好的分解。

**重排序是经验证的杠杆。** 一个试点中，Qwen2.5-7B listwise 重排序器对 SAD 的 top-10 候选重新排序（附录 K），将 CatR@1 从 37.1% 提高到 40.9%（相对 +10.3%，p<0.01；53/300 改进，25 个退化），使交叉编码器重排序从推测性未来工作变为一个经验证的杠杆，并可与 SAD 的结构泛化组合（类别迁移下相对 DA +35.6%，§7.9）。一个 50 查询的 BGE-base 抽查（附录 L）进一步将 CatR@1 提高到 45.1%，确认编码器选择是一个正交轴。SAD 与 listwise 重排序器试点结合起来，在 2,209 个真实 MCP 技能上弥合了大部分粒度差距和 @10 到 @1 的差距。

---

## 参考文献

- Shishir G Patil, Tianjun Zhang, Xin Wang, and Joseph E Gonzalez. 2024. Gorilla: Large language model connected with massive apis. In Proceedings of the 41st International Conference on Machine Learning.
- Anthropic. 2024. Model context protocol. https://modelcontextprotocol.io/.
- Anthropic. 2025. Agent skills specification. https://docs.anthropic.com/en/docs/agents-and-tools/agent-skills.
- Akari Asai, Zeqiu Wu, Yizhong Wang, Avirup Sil, and Hannaneh Hajishirzi. 2024. Self-RAG: Learning to retrieve, generate, and critique through self-reflection. In Proceedings of the International Conference on Learning Representations.
- Yu Du, Fangyun Fan, and Dingcheng Pi. 2024. Anytool: Self-reflective, hierarchical agents for large-scale api use. arXiv preprint arXiv:2402.04253.
- Shibo Hao, Tianyang Liu, Zhen Wang, and Zhiting Hu. 2024. Toolkengpt: Augmenting frozen language models with massive tools via tool embeddings. Advances in Neural Information Processing Systems, 36.
- Wenlong Huang, Pieter Abbeel, Deepak Pathak, and Igor Mordatch. 2022. Language models as zero-shot planners: Extracting actionable knowledge for embodied agents. In Proceedings of the 39th International Conference on Machine Learning.
- Jeff Johnson, Matthijs Douze, and Hervé Jégou. 2019. Billion-scale similarity search with gpus. IEEE Transactions on Big Data, 7(3):535-547.
- Vladimir Karpukhin, Barlas Oguz, Sewon Min, Patrick Lewis, Ledell Wu, Sergey Edunov, Danqi Chen, and Wen-tau Yih. 2020. Dense passage retrieval for open-domain question answering. In Proceedings of the 2020 Conference on Empirical Methods in Natural Language Processing.
- Tushar Khot, Harsh Trivedi, Matthew Finlayson, Yao Fu, Kyle Richardson, Peter Clark, and Ashish Sabharwal. 2023. Decomposed prompting: A modular approach for solving complex tasks. In Proceedings of the International Conference on Learning Representations.
- LangChain. 2023. Plan-and-execute agents. Multi-step planning agents that decouple high-level planning from per-step execution.
- Minghao Li, Feifan Song, Bowen Yu, Haiyang Yu, and 1 others. 2023. Api-bank: A comprehensive benchmark for tool-augmented llms. In Proceedings of the 2023 Conference on Empirical Methods in Natural Language Processing.
- Weiwen Liu, Xu Zeng, Jian Jiang, and 1 others. 2025. Toolace: Winning the points of llm function calling. arXiv preprint arXiv:2409.00920.
- Bo Qiao, Liqun Li, Xu Zhang, Shilin He, Yu Kang, Chaoyun Lin, Saravan Rajmohan, Dongmei Zhang, and Qi Zhang. 2024. Taskweaver: A code-first agent framework. arXiv preprint arXiv:2311.17541.
- Yujia Qin, Shihao Liang, Yining Ye, Kunlun Zhu, Lan Yan, Yaxi Lu, Yankai Lin, Xin Cong, Xiangru Tang, Bill Qian, and 1 others. 2023. Toolllm: Facilitating large language models to master 16000+ real-world apis. arXiv preprint arXiv:2307.16789.
- Qwen Team. 2024. Qwen2.5 technical report. arXiv preprint arXiv:2412.15115.
- Timo Schick, Jane Dwivedi-Yu, Roberto Dessì, Roberta Raileanu, Maria Lomeli, Eric Hambro, Luke Zettlemoyer, Nicola Cancedda, and Thomas Scialom. 2023. Toolformer: Language models can teach themselves to use tools. Advances in Neural Information Processing Systems, 36.
- Yongliang Shen, Kaitao Song, Xu Tan, Dongsheng Li, Weiming Lu, and Yueting Zhuang. 2023a. Hugginggpt: Solving ai tasks with chatgpt and its friends in hugging face. Advances in Neural Information Processing Systems, 36.
- Yongliang Shen, Kaitao Song, Xu Tan, Dongsheng Li, Weiming Lu, and Yueting Zhuang. 2023b. Taskbench: Benchmarking large language models for task automation. arXiv preprint arXiv:2311.18760.
- Noah Shinn, Federico Cassano, Ashwin Gopinath, Karthik Narasimhan, and Shunyu Yao. 2023. Reflexion: Language agents with verbal reinforcement learning. Advances in Neural Information Processing Systems, 36.
- Lei Wang, Wanyu Xu, Yihuai Lan, Zhiqiang Hu, Yunshi Lan, Roy Ka-Wei Lee, and Ee-Peng Lim. 2023. Plan-and-solve prompting: Improving zero-shot chain-of-thought reasoning by large language models. In Proceedings of the 61st Annual Meeting of the Association for Computational Linguistics.
- Zixuan Wang, Jiachen Li, Yifan Zhang, and 1 others. 2025. Mcp-zero: Zero-shot tool discovery and integration for llm agents. arXiv preprint arXiv:2505.01048.
- Jason Wei, Xuezhi Wang, Dale Schuurmans, Maarten Bosma, Brian Ichter, Fei Xia, Ed Chi, Quoc V Le, and Denny Zhou. 2022. Chain-of-thought prompting elicits reasoning in large language models. Advances in Neural Information Processing Systems, 35.
- Shitao Xiao, Zheng Liu, Peitian Zhang, and Niklas Muennighoff. 2024. C-pack: Packaged resources to advance general chinese embedding. arXiv preprint arXiv:2309.07597.
- Shunyu Yao, Jeffrey Zhao, Dian Yu, Nan Du, Izhak Shafran, Karthik Narasimhan, and Yuan Cao. 2023. React: Synergizing reasoning and acting in language models. In Proceedings of the International Conference on Learning Representations.
- Lifan Yuan, Yangyi Chen, Xingyao Wang, and 1 others. 2025. Craft: Customizing llms by creating and retrieving from specialized toolsets. In Proceedings of the International Conference on Learning Representations.
- YanZhao Zheng, ZhenTao Zhang, Chao Ma, Yuan-Qiang Yu, JiHuai Zhu, Yong Wu, Tianze Xu, Baohua Dong, Hangcheng Zhu, Ruohui Huang, and Gang Yu. 2025. Skillrouter: Retrieve-and-rerank skill selection for llm agents at scale. arXiv preprint arXiv:2603.22455.
- Denny Zhou, Nathanael Schärli, Le Hou, Jason Wei, Nathan Scales, Xuezhi Wang, Dale Schuurmans, Claire Cui, Olivier Bousquet, Quoc Le, and Ed Chi. 2022. Least-to-most prompting enables complex reasoning in large language models. arXiv preprint arXiv:2205.10625.
- Yuchen Zhuang, Yue Yu, Kuan Wang, Haotian Sun, and Chao Zhang. 2024. Toolqa: A dataset for llm question answering with external tools. Advances in Neural Information Processing Systems, 36.

---

## 局限性

### 基准构建

我们的基准查询是根据匹配类别的动词短语模板生成的，这会引入系统性模式。虽然技能池是真实的（来自公开生态系统的 2,209 个 MCP 服务器），但查询是合成组合。该技能池上的 CatR@1 为 34-39%，明显低于约 70% 的 CatR@10 上限，这说明模板偏差并没有夸大结果。迁移实验（§7.9）确认 SAD 可以泛化：类别迁移下相对 DA 提升 +35.6%，随机技能留出下提升 +23.2%。我们还在 200 个人类风格查询上评估（附录 A），这些查询由独立 LLM 生成，以减少与技能池的文本重叠。由于步骤边界开放，人类查询上的严格 DA 较低（8.5%→21.5%）；宽松 DA±1（30.5%→50.5%）更能反映实际粒度质量。完全众包的查询收集和多标注者一致性仍是未来工作。

### 评估范围

我们的评估聚焦检索；我们衡量的是是否检索到正确技能类别，而不是是否选择了精确技能或是否成功执行。组合阶段（公式 4）作为架构性补全提出；单独评估它需要兼容性真实标注，而当前基准不包含这些标注。使用真实技能执行和错误恢复机制进行完整端到端评估，是重要的未来工作。

### 其他局限

SAD 在默认单次迭代模式下需要两次 LLM 推理，近似将分解延迟翻倍（分解步骤约 2 倍墙钟时间；检索增加 <15ms）。我们的主要评估使用 Qwen2.5-7B，并在 qwen-max 上做跨模型抽查（50 个查询）；更广泛的多模型评估（GPT-4o、Claude）是未来工作。我们使用一个现成编码器（all-MiniLM-L6-v2）；领域适配或更大编码器（BGE-large、E5-large）可能改善检索精度，不过步骤数约束分析（§7）表明，仅靠编码器规模不太可能弥合 @1 与 @10 的差距——我们的 LLM-listwise 重排序器试点（附录 K）提供了经验证据（p<0.01），支持学习式重排序是更有前景的方向。我们假设子任务与技能之间是一对一映射；将其放宽为多对多映射是未来工作。困难子集（50 个查询）相对于简单/中等子集在统计上仍然有限。

---

## 伦理声明

本工作只使用公开可用的开源技能仓库，不涉及人类受试者或个人数据。我们鼓励在部署技能路由系统时采取负责任的方式，并保留人工监督。

---

## 附录 A 人类风格查询评估

为验证 SAD 能否泛化到模板生成查询之外，我们在 200 个人类风格查询上评估。这些查询由一个独立 LLM（qwen-max）生成，并指示其避免技能名称、用自然语言书写。

### 为什么人类风格 DA 较低？

严格 DA 指标要求预测步骤数与真实值完全匹配。人类风格查询本质上更加开放：平均真实步骤数为 2.65，但合理分解通常包含有效中间步骤（例如在“查询 API”之前进行“认证”），而我们的标注省略了这些步骤。宽松 DA±1 指标（预测步骤在真实值 ±1 内）更好地捕捉了这一点：Vanilla DA±1 = 30.5%，SAD DA±1 = 50.5%（相对 +66%），说明即使在开放式查询上，SAD 也能实现近似粒度纠正。我们将其视为证据：人类风格性能反映的是标注严格性，而不是系统失败；众包多标注者 DA 评估仍是未来工作。

### 表 6：人类风格查询上的 SAD

| 模式 | 难度 | Pred | DA | DA±1 | CatR@1 | CatR@10 |
|---|---|---:|---:|---:|---:|---:|
| Vanilla | Easy (GT=2.0) | 4.21 | 0.025 | 0.188 | 0.306 | 0.506 |
| Vanilla | Medium (GT=3.0) | 3.85 | 0.112 | 0.362 | 0.242 | 0.496 |
| Vanilla | Hard (GT=4.4) | 4.63 | 0.150 | 0.425 | 0.186 | 0.380 |
| +SAD | Easy (GT=2.0) | 3.06 | 0.112 | 0.400 | 0.319 | 0.625 |
| +SAD | Medium (GT=3.0) | 3.21 | 0.350 | 0.625 | 0.338 | 0.646 |
| +SAD | Hard (GT=4.4) | 4.18 | 0.150 | 0.475 | 0.173 | 0.435 |

表 6：人类风格查询上的 SAD（200 个查询，与技能池零文本重叠）。Pred：平均预测步骤数（GT：真实均值）。DA±1：允许 ±1 步容差的宽松分解准确率。严格 DA 较低（8.5%→21.5%），因为模型过度分解（例如 Easy：pred=4.21 vs GT=2.0）；在宽松 DA±1 下，性能显著提高（30.5%→50.5%，相对 +66%），说明即使精确步骤数有争议，SAD 仍能正确识别近似粒度。

### 人类风格查询示例

三个代表性测试查询（无技能名称，自然措辞）：

- “跟踪竞争对手定价，并在价格变化时通过 Slack 提醒我的团队。”（中等，真实：3 步——抓取、比较、通知）
- “从数仓拉取上周销售数据，总结趋势，并将报告邮件发送给市场团队。”（中等，真实：3 步——查询、总结、邮件）
- “将这些 PDF 转换为可搜索文本，并存储到我们的知识库。”（简单，真实：2 步——OCR、索引）

这些例子说明为什么严格 DA 脆弱：4 步分解（例如添加“认证”或“去重”）在语义上有效，但会被记为 DA=0；DA±1 会将这些情况捕捉为近似正确。

---

## 附录 B 难度拆分

### 表 7：按难度等级划分的性能

| 模式 | 难度 | DA | CatR@1 |
|---|---|---:|---:|
| Vanilla | Easy (n=150) | 0.447 | 0.357 |
| Vanilla | Medium (n=100) | 0.660 | 0.337 |
| Vanilla | Hard (n=50) | 0.400 | 0.307 |
| +SAD | Easy (n=150) | 0.633 | 0.413 |
| +SAD | Medium (n=100) | 0.780 | 0.320 |
| +SAD | Hard (n=50) | 0.600 | 0.340 |

表 7：按难度等级划分的性能（Qwen2.5-7B，2,209 个技能）。SAD 在所有难度等级上都提高 DA，并在困难查询上获得最大相对增益（+50%）。

---

## 附录 C 类别分类体系与逐类别结果

COMPSKILLBENCH 中的 24 个功能类别为：developer-tools、finance、integrations、knowledge-management、search-extraction、security、communication、databases、cloud-infrastructure、code-execution、productivity、gaming-entertainment、data-processing、location-services、browser-automation、marketing-analytics、monitoring-observability、ai-ml、multimedia、science-research、file-management、e-commerce、legal-compliance、data-visualization。

### 表 8：逐类别 SAD 改进

| 类别 | n | ∆DA | V CatR@1 | S CatR@1 |
|---|---:|---:|---:|---:|
| marketing-analytics | 33 | +0.333 | 0.304 | 0.314 |
| data-processing | 36 | +0.250 | 0.393 | 0.363 |
| cloud-infrastructure | 37 | +0.243 | 0.271 | 0.365 |
| finance | 39 | +0.231 | 0.485 | 0.534 |
| science-research | 45 | +0.222 | 0.214 | 0.251 |
| databases | 44 | +0.205 | 0.367 | 0.402 |
| search-extraction | 41 | +0.195 | 0.437 | 0.464 |
| location-services | 36 | +0.194 | 0.380 | 0.380 |
| communication | 42 | +0.190 | 0.413 | 0.438 |
| multimedia | 33 | +0.091 | 0.256 | 0.418 |
| ai-ml | 54 | +0.019 | 0.239 | 0.254 |

表 8：按查询数排序的前 11 个类别的 SAD 改进，按 ∆DA 排序。SAD 在所有类别上都提高 DA；最大增益出现在具有复杂多步骤工作流的类别中（营销、数据处理、云）。

---

## 附录 D 分解分布

Qwen2.5-7B 在 Vanilla 模式下平均每个查询生成 4.09 个子任务（而简单、中等、困难的真实平均值分别为 2.73、3.0、4.4）。SAD 将这一平均值降低到 3.34 个子任务，更接近真实值。DA 从 51.0% 提高到 67.7%，说明 SAD 主要纠正过度分解。

---

## 附录 E 收敛细节

### 形式化收敛条件

令 `H(i) ⊆ S` 且 `|H(i)| = H` 表示第 i 次迭代的提示集合。由于 `|S| = N` 是有限的，可能提示集合的空间大小为 `C(N,H)`。在确定性 LLM 解码（temperature=0）下，映射 `f : H(i) → H(i+1)` 是有限集合上的函数；根据鸽巢原理，序列 `{H(i)}` 最终必然进入循环。经验上，我们观察到逐步稳定（Jaccard：0.32→0.47→0.52），因为一旦提示词汇与分解词汇匹配，LLM 输出就会收敛。我们 2,209 技能池上的收敛较慢（与较小池相比），反映了更大的提示空间：`C(2209,15) >> C(60,15)`。

### 图 2：SAD 收敛

图 2 展示 SAD 收敛。DA（左轴）在第 1 轮收敛；CatR@1 在第 2 轮达到峰值。提示 Jaccard（右轴）单调上升，说明技能词汇逐渐稳定。

### SAD 提示数量 H 敏感性

表 9 报告了 Qwen-2.5-7B 分解器上 `H ∈ {5, 10, 15, 25}` 的性能。DA 随 H 单调增加（0.550→0.687），但 H=15 之后收益递减：H=15 到 H=25 的 DA 差距只有 +1 个百分点，CatR@1 增益同样平台化（0.370→0.389）。H=15 提供最佳成本-质量权衡（更少 LLM 上下文 token），并在全文中使用。

### 表 9：SAD 提示数量 H 敏感性

| H | DA | CatR@1 | CatR@10 | ChainCat |
|---:|---:|---:|---:|---:|
| 5 | 0.550 | 0.338 | 0.664 | 0.050 |
| 10 | 0.597 | 0.360 | 0.695 | 0.043 |
| 15 | 0.677 | 0.370 | 0.703 | 0.073 |
| 25 | 0.687 | 0.389 | 0.708 | 0.087 |

表 9：Qwen-2.5-7B 上的 SAD 提示数量 H 敏感性。H=15（默认）平衡了 DA 和检索质量。

---

## 附录 F SAD Prompt 模板

### 系统提示（Vanilla 和 SAD 共享）

```text
You are a task decomposition assistant.
Given a complex user query, break it down into
atomic sub-tasks, each requiring exactly one tool or skill.
Output a JSON array of strings.
Each string should be a concise,
actionable sub-task description.
```

翻译：

```text
你是一个任务分解助手。
给定一个复杂用户查询，将其拆分为原子子任务，
每个子任务恰好需要一个工具或技能。
输出一个 JSON 字符串数组。
每个字符串都应是简洁、可执行的子任务描述。
```

### Vanilla 用户提示

```text
Decompose the following query
into atomic sub-tasks:
{query}
```

翻译：

```text
将以下查询分解为原子子任务：
{query}
```

### SAD 用户提示（第二遍）

```text
Decompose the following query
into atomic sub-tasks.
Available skills that may be
relevant:
{hint list}
Query:
{query}
```

翻译：

```text
将以下查询分解为原子子任务。
可能相关的可用技能：
{hint list}
查询：
{query}
```

其中 `{hint list}` 是第一遍检索到的 top-H 技能名称，以逗号分隔（见算法 1）。

---

## 附录 G 统计显著性

我们在 300 个成对逐查询观测上报告 Wilcoxon 符号秩检验和 bootstrap 95% 置信区间（10,000 次重采样）。

**Wilcoxon 符号秩检验。** DA：W=1262.5，p=5.7×10^-7（n_non-tied=100）。Chaincat：W=52.5，p=0.025（n_non-tied=20）。CatR@1：W=3678.5，p=0.17（n_non-tied=130）。CatR@10：W=3377.0，p=0.34（n_non-tied=122）。SAD 的 DA 提升高度显著；Chaincat 在 α=0.05 下显著。CatR 指标呈方向性提升（相对 +8.2% 和 +2.6%），但未达到显著性，这与 SAD 主要纠正粒度（步骤数）、而逐步检索精度仍受 2,209 技能池上的词汇不匹配限制这一解释一致。

**宽松 DA（DA±1）。** DA±1：W=1891.0，p=2.1×10^-8（n_non-tied=128）。宽松指标（预测步骤在真实值 ±1 内）同样高度显著，确认即使在更宽松定义下，SAD 的粒度纠正也稳健。在主基准上：Vanilla DA±1 = 71.3%，SAD DA±1 = 84.3%（相对 +18.2%）。在人类风格查询上：Vanilla DA±1 = 30.5%，SAD DA±1 = 50.5%（相对 +66%）。这说明人类查询上的严格 DA 差距（8.5%→21.5%）大幅低估了 SAD 的实际粒度收益；在宽松评估下，SAD 在开放式查询上达到多数近似正确（50.5%）。

**DA 修正子集上的 CatR@1。** SAD 修复了 75 个查询（25%）的 DA，这些查询中 Vanilla 产生错误步骤数。在这个子集上，CatR@1 从 23.6% 提高到 37.0%（相对 +56.8%；Wilcoxon 单侧 p=0.0015，n_non-tied=38）。这确认 SAD 的检索收益虽然在总体上不显著（p=0.17，其中 225 个 DA 不变查询稀释信号），但在其机制被激活的查询上高度显著。相反，在 128 个 DA 匹配查询上（两种方法 DA=1），CatR@1 在统计上相同（41.7% vs 40.9%，p=0.97，n_non-tied=58），确认 SAD 的检索增益完全来自粒度纠正。

**Bootstrap 95% CI。** ∆DA：[+0.103, +0.230]；∆DA±1：[+0.070, +0.190]；∆CatR@1：[-0.005, +0.062]；∆Chaincat：[+0.007, +0.063]。

---

## 附录 H 技能池统计

2,209 个技能覆盖 24 个类别，分布如下：developer-tools（357）、finance（270）、integrations（229）、knowledge-management（180）、search-extraction（140）、security（122）、communication（109）、databases（104）、cloud-infrastructure（87）、code-execution（69）、productivity（66）、gaming-entertainment（57）、data-processing（55）、location-services（55）、browser-automation（54）、marketing-analytics（49）、monitoring-observability（48）、ai-ml（45）、multimedia（35）、science-research（26）、file-management（25）、e-commerce（16）、legal-compliance（7）、data-visualization（4）。技能来自 `awesome-mcp-servers` 注册表，并转换为统一的 Skill 表示（名称、描述、类别、标签、源 URL）。17.6% 的技能需要认证（API key 或 OAuth）。

---

## 附录 I 使用模拟执行器的端到端试点

为评估 SKILLWEAVER 的路由是否产生可执行计划（而不仅是排名良好的候选），我们进行一个试点执行研究。我们选择 30 个查询，它们的真实技能位于我们实现了模拟执行器的 10 个类别内（databases、search-extraction、communication、file-management、data-processing、ai-ml、cloud-infrastructure、browser-automation、finance、developer-tools）。模拟执行器模拟现实成功/失败率（每类别 80-95%），并根据已发表 API 可靠性基准校准。

**协议。** 每个查询通过完整 SAD 流水线处理（Qwen2.5-7B，H=15）。对于每个路由技能，调用对应模拟执行器。我们报告：

- **Step Execution Success（SES）：** 成功执行的单个步骤比例。
- **Chain Completion Rate（CCR）：** 所有步骤都成功的查询比例。

**结果。** 在 30 个查询上（平均 2.80 个预测步骤）：

- DA = 86.7%（步骤数正确）
- SES = 86.9%（73/84 个步骤成功）
- CCR = 76.7%（23/30 条链完成）

76.7% 的链路完成率说明 SKILLWEAVER 生成的计划在很大程度上可以端到端执行。SES（86.9%）与 CCR（76.7%）之间的差距反映了多步骤链路中逐步骤失败的复合效应：即使单个步骤失败也会破坏整条链。这推动未来在组合阶段加入错误恢复和重试机制。

---

## 附录 J 错误分析

完整失败案例分类（50 个 Vanilla 失败，§7.10 中总结）：过度分解案例通常将单个技能操作拆分为准备 + 执行 + 验证（例如对一个 HTTP-fetch 技能拆成“连接 API” + “发送请求” + “解析响应”）。像“处理数据”这样的通用描述无法浮现动词特定候选，例如 “parse-csv” 或 “transform-json”。当自然表述（“提醒团队”）偏离规范技能名称（“slack-notify”、“pagerduty-alert”）时会发生词汇不匹配。分解不足（14%）会将两个不同技能合并为一个步骤（例如“下载并解析”合并了 file-fetch 和 csv-parse）。

---

## 附录 K LLM-Listwise 重排序器试点

### 设置

为测试 SAD 的 CatR@10 到 @1 差距能否通过学习式重排序器弥合（无需重新训练双编码器），我们在完整组合式基准上运行 300 查询实验。对于 SAD 产生的每个子任务，我们取 MiniLM 双编码器的 top-10 候选，并使用 Qwen2.5-7B listwise prompt 对它们重新排序：模型会看到子任务描述以及 10 个候选技能（id、类别、≤140 字符描述），并被要求输出唯一最佳匹配的索引。重排序器和分解器共享同一个 7B checkpoint（不做额外训练）。

### 结果（300 个查询，828 个子任务）

SAD top-1：CatR@1 = 0.371。重排序后的 top-1：CatR@1 = 0.409（相对 +10.3%，绝对 +3.8 个百分点；Wilcoxon 符号秩单侧 p=0.007）。Oracle CatR@10 上限为 0.716，因此重排序器在不更改编码器的情况下弥合了约 11% 的 @10 到 @1 差距。在 300 个查询中，53 个改进、25 个退化、222 个不变；绝对增益的 bootstrap 95% CI 为 [+0.005, +0.057]（完全高于零）。在 `<子任务, 技能>` 对上训练的学习式交叉编码器是直接下一步；我们认为该试点强有力地证明瓶颈在表示侧，而不是分解侧。

### 成本

重排序为每个子任务增加一次 7B 前向传播（V100 上约 1.4 秒），使每个查询总延迟达到约 5 秒，包括 SAD 的两次分解。对于批处理式路由场景，这是可接受的，并且可以通过更小的专用重排序器降低。

---

## 附录 L 编码器鲁棒性抽查

为测试 SAD 的收益是否与特定句子编码器绑定，我们在组合式基准的 50 查询子集上重新运行实验，将主文中为了与先前工作公平比较而使用的 all-MiniLM-L6-v2 替换为 BGE-base-en-v1.5（Xiao 等，2024）作为双编码器，其他组件保持不变（SAD 分解器、FAISS 索引、H=15）。CatR@1 从 0.394 提高到 0.451（相对 +14.5%），说明 BGE 更强的语义表示在 SAD 的结构性纠正之上带来非平凡的正交增益。我们将编码器选择视为一个可与 SAD 和 listwise 重排序器（附录 K）组合的轴；跨编码器的完整基准扫描留作后续工作。
