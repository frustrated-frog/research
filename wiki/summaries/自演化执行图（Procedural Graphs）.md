---
id: source/procedural-graphs-self-evolving-execution-structures
title: Procedural Graphs：自演化执行图
type: source
status: active
tags: [agent, procedural-graph, self-evolution, execution-structure, harness]
aliases: [Procedural Graphs, Self-Evolving Execution Structures for LLM Agents, PG]
created: 2026-09-13
updated: 2026-09-13
sources:
  - path: raw/sources/文章/自演化执行图论文/Self-Evolving Execution Structures for LLM Agents.pdf
    locator: "第 1–11 页；附录 B、D、E"
    claim_type: extracted
    confidence: high
related:
  - id: concept/hermes-agent/自我改进循环
    relation: extends
  - id: index/harness-engineering
    relation: provides_evidence_for
valid_from: null
valid_to: null
superseded_by: null
---

# Procedural Graphs：自演化执行图

**来源**：Yuxing Lu、Yicheng Chen、Shanchan Wu、Sercan Ö. Arık，*Procedural Graphs: Self-Evolving Execution Structures for LLM Agents*。

## 核心内容

[原文事实] 论文将程序知识表示为有向、带属性的图 \(G=(V,R,E,\Phi)\)。节点可对应工具、推理步骤或状态；边表示可允许的程序转移，并带有 condition、guidance、pitfalls。在线时，系统匹配 Agent 最近动作到活跃节点，取 2 跳局部邻域，由 guidance LLM 将图与最近 3 步轨迹转成下一步提示；求解器仍保有动作选择权（第 3–5 页）。

[原文事实] 离线自演化从成功／失败轨迹提出 add/delete node/edge 编辑，候选图需经过结构校验和独立验证集评估；验证均分不低于保留图时才提交，失败候选作为 rejection memory 供后续 refiner 避免重复（第 5–6 页；附录 B.6）。

## 结果与边界

[原文事实] 表 1 在 4 个 LLM、6 项主基准的 24 个设置中报告 PG 有 21 次第一或并列第一；相对每格最强基线为 19 胜、2 平、3 负，作者报告单侧符号检验 \(p=4.3\times10^{-4}\)。HotpotQA 上的增益范围则为 −0.90 至 +1.30 pt，收益随任务变化（第 7 页）。

[原文事实] 在 EnterpriseArena 的 20/20/20 演化配置中，按验证集选出的最终图测试全周期存活为 85%，无图基线为 0%；单轮选择受 20 episodes 影响，论文要求将其看作搜索轨迹而非逐轮显著性测试（第 10 页；附录 E，表 11）。

[原文事实] 局部子图生成指导胜过完整图注入／完整图指导，但相对无图基线在 GDPval 与 ALFWorld 的总 token 仍分别高 33.4% 与 55.4%（第 10 页，表 3）。

## 与知识库的连接

- [[concepts/hermes-agent/自我改进循环]]：PG 提供了将“执行经验”由文本／技能工件进一步结构化为可编辑转移图、并以验证闸门回滚的实证方案。
- [[index/harness-engineering]]：为 Agent Harness 补充“局部化执行指导、结构化编辑、保留版本与拒绝记忆”的可审计控制层视角。
- [[summaries/HarnessDev：LLM能否创建并演化自己的Agent Harness]]：两篇研究都讨论 Agent 工件演化；HarnessDev 强调全 harness 的 held-out 与跨 executor 评测，PG 重点验证图工件的局部结构演化。

## 进一步阅读

- 详细中文解读：[[raw/sources/文章/自演化执行图论文/自演化执行图论文解读]]
