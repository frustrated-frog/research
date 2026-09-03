---
title: Harness 六大组件
id: concept/harness-engineering/Harness六大组件
type: concept
status: active
tags: [Harness, AI Agent, 文件系统, 沙箱, 记忆, Web Search, MCP, 上下文工程, 编排, Hooks]
aliases: []
created: 2026-04-08
updated: 2026-09-03
related: [上下文工程, 自我改进循环]
valid_from: null
valid_to: null
superseded_by: null
sources:
  - source_id: source-harness-six-components
    path: raw/sources/harness-engineering/Harness六大组件深度解读.md
    locator: "全文"
    claim_type: extracted
    confidence: high
  - source_id: source-harnessdev-20260902
    path: raw/sources/文章/字节论文：自进化框架/HarnessDev- Can LLMs Create and Evolve Their Own Agent Harness?.pdf
    locator: "第 4–7 页、附录 C"
    claim_type: extracted
    confidence: high
---

# Harness 六大组件

> 来源：李伟山《一文讲透如何构建Harness——六大组件全解析》

## 核心公式

> **Agent = Model + Harness**
> 模型提供智能，Harness 让智能变得有用。

Harness 是模型之外的一切工程基础设施。模型是马，Harness 是挽具。

---

## 裸模型的四个硬伤与应对

| 硬伤 | 缺失能力 | Harness 组件 |
|------|---------|------------|
| 无法维持跨会话状态 | 长期记忆 | 文件系统 + AGENTS.md |
| 无法执行代码 | 行动能力 | Bash + 沙箱 |
| 无法获取实时知识 | 感知能力 | Web Search + MCP |
| 无法搭建工作环境 | 环境操控 | 文件系统 + 上下文工程 + 编排 |

---

## 组件一：文件系统

**三大核心能力：**
1. **工作空间**：存储中间结果，突破上下文窗口限制
2. **协作基础**：多 Agent 通过文件共享白板，异步协作
3. **版本追踪**：与 Git 集成，支持错误回滚和分支实验

**设计原则：** 目录结构清晰约定俗成、文件粒度适中（一个模块一个文件）、元数据丰富（README + ADR）。

---

## 组件二：Bash + 沙箱

**核心价值：** 从"生成代码"到"执行代码"的质变，让 Agent 拥有**自我验证循环**（写→跑→看→修→再来）。

实测数据：具备自我验证循环的 Agent 任务完成率比一次性生成**高出 40%–60%**。

**沙箱必要性：** 没有沙箱不敢让 Agent 执行代码。沙箱提供：
- 资源限制（CPU/内存/磁盘上限）
- 网络隔离
- 文件系统隔离
- 超时机制

**技术选型：** Docker（主流）、gVisor/Firecracker（轻量）、WASM（边缘）、Nix（可重现）

---

## 组件三：记忆（AGENTS.md）

**工作机制：** 工作中写入知识 → 存文件 → 下次自动注入上下文

**核心洞察：上下文注入 = 不改权重给模型加知识**

传统方式需要微调（成本高、有灾难性遗忘风险）。AGENTS.md 提供了"即插即用"的知识扩展——今天写进，明天 Agent 自动具备，不需要重新训练。

**最佳实践：**
- 层次化存放：根目录放全局知识，子目录放局部知识
- 结构化书写：项目架构、技术约束、编码规范、Gotchas、ADR
- 定期清理：过时知识比没知识更危险
- 双向可编辑：人类可直接编辑来传达偏好和约束

---

## 组件四：Web Search + MCP

**Web Search** 解决"知识过时"问题，好坏取决于：查询构建、结果筛选、内容提取、信息整合。

**MCP（Model Context Protocol）** 是 Anthropic 推出的开放协议，可理解为"AI 世界的 USB 接口"——即插即用地连接任何外部工具和数据源（数据库、内部 Wiki、Jira、GitHub、监控系统）。

**协同效应示例：** 修复生产 Bug 时，MCP 连接监控系统获取日志 + 连接代码仓库查看变更 + Web Search 搜索解决方案 + MCP 连接 Jira 检查已知问题——Agent 像资深 SRE 一样在多个信息源间穿梭。

---

## 组件五：上下文工程

**Context Rot（上下文腐烂）：** 随着对话进行，冗余信息不断堆积，导致信噪比下降、矛盾信息累积、Token 浪费、推理质量退化。

**四大核心策略：**

| 策略 | 说明 |
|------|------|
| **压缩** | 历史上下文摘要压缩，工具调用记录替换为简洁结果摘要 |
| **工具输出卸载** | 大段输出存文件系统，上下文只保留摘要和文件引用 |
| **Skills 渐进加载** | 不同任务阶段动态加载/卸载相关 Skill，避免一次性塞满上下文 |
| **分层上下文结构** | 核心层（始终保留）、工作层（按需更新）、历史层（逐渐压缩） |

**特殊地位：** 上下文工程不是独立模块，而是影响所有其他组件的**元能力**。Harness 本质上就是好的上下文工程的交付机制。

---

## 组件六：编排 + Hooks

**编排解决"谁做什么"：** 子 Agent 调度、任务分发、模型路由（简单任务用小模型省钱）、结果聚合。

**Hooks 解决"做得对不对"：**

| Hook 类型 | 作用 |
|---------|------|
| Lint 检查 | 代码生成后自动运行 linter，不合规则要求修复 |
| 截断续接 | 检测输出截断则自动触发续接 |
| 格式约束 | JSON Schema 验证、Markdown 结构检查 |
| 安全过滤 | 敏感操作前权限检查和审计 |
| 成本控制 | 监控 token 消耗，超预算则警告 |

**核心价值：** 用**确定性的规则约束概率性的模型输出**。这种"概率性生成 + 确定性校验"是当前 Agent 工程最有效的质量保障策略。

**编排模式演进：** 线性管道 → DAG 编排（支持并行）→ 动态编排（最灵活但难控）→ 层级编排（适合超大规模）。

---

## System Prompt：贯穿所有组件的"神经系统"

System Prompt 在 Harness 中扮演四个角色：
1. **定义角色边界**：告诉模型"你是谁"和"你不是谁"
2. **注入领域知识**：项目背景、技术栈等直接写入，确保第一轮就具备必要知识
3. **约束安全规则**：安全策略的第一道防线
4. **贯穿所有组件**：决定所有组件的工作方式和行为规范

**与 AGENTS.md 的关系：** System Prompt 装"必须知道的"（容量有限），AGENTS.md 装"最好知道的"（无容量限制）。

---

## 核心结论

> **模型决定 Agent 能力的下限，Harness 决定 Agent 能力的上限。**

真正在生产环境中创造价值的，往往不是那个最强的模型，而是那个最好的 Harness。

如果你不是模型本身，那你就是 Harness 的一部分——System Prompt 是神经系统，工具链是四肢，记忆机制是长期记忆，上下文策略是注意力管理，Hook 规则是质量底线。

## 关联页面

- [[concepts/hermes-agent/自我改进循环|自我改进循环]]
- [[concepts/harness-engineering/Harness驾驭层|Harness驾驭层]]
- [[summaries/HarnessDev：LLM能否创建并演化自己的Agent Harness]]：以 Creation 基准验证执行、工具、上下文、状态、生命周期和验证是可单独评测的控制职责；其 Code harness 结果也显示状态/记忆最容易停留在未进入执行主路径的“声明层”。
