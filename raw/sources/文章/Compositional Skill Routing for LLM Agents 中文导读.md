# Compositional Skill Routing for LLM Agents：中文导读

> 说明：原论文页面标注为 CC BY-NC-ND 4.0。该许可证中的 NoDerivatives 条款限制分发改编作品，完整翻译通常属于改编。因此这里提供的是不替代原文的中文导读、结构化摘要和术语说明，不是全文逐句译文。

## 论文信息

- 标题：Compositional Skill Routing for LLM Agents: Decompose, Retrieve, and Compose
- 中文参考题名：面向 LLM 智能体的组合式技能路由：分解、检索与组合
- 作者：Xueping Gao
- 机构：Alibaba Cloud, Hangzhou, China
- arXiv：2606.18051v1
- 提交日期：2026-06-16
- 领域：cs.CL
- 页数：14 页
- 许可证：CC BY-NC-ND 4.0

## 一句话概括

这篇论文研究的是：当一个 LLM 智能体面对复杂任务时，如何把任务拆成多个原子步骤，为每一步找到合适的 skill/tool，并把这些 skill 组合成可执行计划。作者提出了 SkillWeaver 框架，并构建了 CompSkillBench 基准来评估组合式技能路由。

## 核心问题

现代 LLM Agent 不再只是生成文本，而是会调用外部工具、技能和 API。过去的技能路由通常把问题简化成“从技能库里选一个最合适的 skill”。但现实任务往往需要多个技能协同完成，例如：

- 下载数据集
- 转换或清洗数据
- 生成图表或报告

因此，作者把问题形式化为 Compositional Skill Routing：给定复杂用户查询和大型 skill 库，系统需要输出一个有序的 skill 序列，每个 skill 对应一个原子子任务。

## 作者提出的框架：SkillWeaver

SkillWeaver 采用三阶段流程：

1. Decompose：用 LLM 将复杂查询拆成原子子任务。理想情况下，每个子任务只需要一个 skill。
2. Retrieve：用 bi-encoder 检索器和 FAISS 索引，为每个子任务检索候选 skill。
3. Compose：用依赖感知的 DAG planner，把多个 skill 组织成可执行的组合计划。

这套流程的关键不只是“能找到 skill”，而是要先把任务拆到正确粒度。如果拆得太粗，一个步骤可能需要多个技能；如果拆得太细，会产生不必要的步骤并增加检索噪声。

## 主要贡献

1. 定义了 Compositional Skill Routing 这一问题，强调真实任务通常需要多技能组合，而不是单技能选择。
2. 提出 SkillWeaver：一个“分解 - 检索 - 组合”的框架。
3. 构建 CompSkillBench：包含 300 个组合查询，覆盖 2,209 个真实 MCP server skills，跨 24 个功能类别。
4. 指出任务分解质量是瓶颈：普通 LLM 分解在步骤层面的 category recall 只有 34.2%。
5. 提出 Iterative Skill-Aware Decomposition，简称 SAD，通过检索反馈迭代修正任务分解。
6. 实验显示 SAD 可将分解准确率从 51.0% 提升到 67.7%，单轮相对提升 32.7%。
7. SkillWeaver 可将上下文窗口消耗降低超过 99%。

## 方法细节

### 任务分解

任务分解器接收用户复杂请求，输出一组原子子任务。每个子任务应满足：

- 语义上独立
- 可由一个 skill 完成
- 与其他步骤存在清晰依赖关系

作者指出，分解质量直接决定后续检索效果。若分解步骤本身错误，即使检索器很强，也难以找对 skill。

### Skill 检索

SkillWeaver 使用 bi-encoder 架构，将子任务和 skill metadata 编码到同一向量空间中，再用 FAISS 做近邻搜索。这样可以避免把整个 skill 库塞进上下文窗口。

这也是上下文节省超过 99% 的来源：系统只把少量候选 skill 暴露给 LLM，而不是把 2,209 个 skill 的说明全部放入 prompt。

### 组合规划

检索完成后，系统需要把每个步骤的 skill 组合成一个可执行链。论文将这一过程建模为带依赖关系的 DAG 规划问题，并考虑 skill 之间的兼容性。

### SAD：迭代式技能感知分解

SAD 的核心思想是：先让 LLM 拆任务，再检索候选 skill；如果检索结果显示当前拆分和 skill 库不匹配，就把反馈提供给 LLM，让它重新调整分解。

这相当于让任务分解过程“知道 skill 库里实际有什么”，而不是在真空中拆任务。

## 实验与结果

### 基准数据集

CompSkillBench 包含：

- 300 个组合式查询
- 2,209 个真实 MCP server skills
- 24 个功能类别
- 来源：公开 MCP 生态系统

### 关键指标

- DA：Decomposition Accuracy，分解准确率
- CatR@1：Category Recall at 1，Top-1 类别召回
- skill-level retrieval：具体 skill 级别检索效果
- context window consumption：上下文窗口消耗

### 主要发现

1. 普通 LLM 的任务分解是主要瓶颈。
2. 标准分解在步骤层面的类别召回只有 34.2%。
3. SAD 单轮迭代将分解准确率从 51.0% 提升到 67.7%。
4. 当 DA=1，也就是分解粒度正确时，CatR@1 从 34% 提升到 41%。
5. SkillWeaver 能减少超过 99% 的上下文窗口消耗。
6. 迁移实验显示，即使目标类别不在检索池中，SAD 仍然带来 +35.6% 的相对 DA 提升。

## 论文的关键洞见

### 1. Agent 的难点不只是会不会调用工具

很多 Agent 系统的问题不在于没有工具，而在于工具太多、任务太复杂，模型不知道应该怎么拆、该找哪些工具、按什么顺序调用。

### 2. 分解比检索更基础

如果第一步任务分解不对，后面的检索和规划会被带偏。论文把分解质量识别为整个 pipeline 的核心瓶颈。

### 3. Skill 库应该反过来指导任务分解

普通 LLM 分解是“凭常识拆任务”。SAD 的思路是让模型根据实际可用 skill 调整拆法，这更贴近真实工程系统。

### 4. Progressive disclosure 很重要

把全部工具说明塞进上下文既昂贵又不可扩展。SkillWeaver 的检索式暴露机制更适合大规模 skill 库。

## 与现有工作的关系

论文把自己放在三个相关方向中：

- Tool use / Agent planning：研究 LLM 如何调用工具、规划步骤、执行任务。
- Skill routing：研究如何从大量 skill 中选择合适 skill。
- Retrieval-augmented systems：用检索减少上下文负担，并提高系统可扩展性。

与单技能选择不同，本文强调组合式、多步骤、多 skill 的路由问题。

## 局限性

论文自身也提到或暗示了几个限制：

- 评估集中在 decompose-retrieve 部分，完整端到端执行只做了 pilot study。
- 真实 Agent 环境中的错误恢复、权限、状态管理、工具调用失败等问题仍未完全覆盖。
- 分解质量虽然通过 SAD 提升明显，但仍远未达到完美。
- benchmark 规模是 300 个查询，适合作为起点，但还不足以覆盖全部真实任务复杂度。

## 对实践的启发

如果你在构建 Agent 或 skill/plugin 系统，这篇论文给出的工程启发是：

1. 不要只做 tool retrieval，还要显式做 task decomposition。
2. 让 decomposition 和 skill library 形成反馈闭环。
3. 不要把所有工具塞进 prompt，要用检索进行 progressive disclosure。
4. 评价 Agent 工具选择时，不只看 final answer，也要看每一步是否拆得对、检索得对、组合得对。
5. 大型 skill 生态需要专门的 benchmark，而不是只用通用问答或单工具调用数据集。

## 术语表

- LLM Agent：能规划、调用工具、执行多步骤任务的大语言模型智能体。
- Skill：可复用的工具规范或能力说明，通常包含何时使用、如何调用、输入输出要求等信息。
- Compositional Skill Routing：组合式技能路由，即为复杂任务选择并组合多个 skill。
- Decompose：任务分解。
- Retrieve：检索，为子任务找候选 skill。
- Compose：组合，把多个 skill 组织成执行计划。
- DAG Planner：基于有向无环图的规划器，用于表达步骤依赖。
- FAISS：高效向量相似度搜索库。
- Bi-encoder：双编码器，将查询和文档分别编码到向量空间后做相似度匹配。
- MCP：Model Context Protocol，论文中的技能来源来自公开 MCP 生态。
- SAD：Iterative Skill-Aware Decomposition，迭代式技能感知分解。
- DA：Decomposition Accuracy，分解准确率。
- CatR@1：Top-1 类别召回率。

## 推荐阅读方式

1. 先读 Abstract 和 Introduction，把握“为什么单技能路由不够”。
2. 再读 Problem Formulation，理解作者如何定义组合式技能路由。
3. 重点读 Method: SkillWeaver，关注 Decompose、Retrieve、Compose 三阶段。
4. 阅读 Benchmark 和 Results，理解 CompSkillBench 的构造和 SAD 的效果。
5. 最后读 Discussion 和 Limitations，判断这套方法离真实工程落地还有多远。

