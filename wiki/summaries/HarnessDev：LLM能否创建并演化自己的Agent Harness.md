---
id: source/harnessdev-20260902
title: HarnessDev：LLM能否创建并演化自己的Agent Harness？
type: source
status: active
tags: [harness-engineering, agent-evolution, benchmark, self-improvement]
aliases: [HarnessDev, Can LLMs Create and Evolve Their Own Agent Harness?]
created: 2026-09-03
updated: 2026-09-03
sources:
  - source_id: source-harnessdev-20260902
    path: raw/sources/文章/字节论文：自进化框架/HarnessDev- Can LLMs Create and Evolve Their Own Agent Harness?.pdf
    locator: "全文（41 页）"
    claim_type: extracted
    confidence: high
related:
  - id: concept/harness-engineering/Harness六大组件
    relation: provides_evidence_for
  - id: concept/harness-engineering/过早完成声明
    relation: provides_evidence_for
  - id: concept/harness-engineering/可观测性
    relation: provides_evidence_for
valid_from: 2026-09-02
valid_to: null
superseded_by: null
---

# HarnessDev：LLM能否创建并演化自己的Agent Harness？

## 核心内容

HarnessDev 将评测对象从单次任务答案改为可运行、可冻结、可复用和可审计的 agent harness。它区分 creator LLM（开发 harness）与 executor LLM（在冻结 harness 中执行下游任务），并以任务能力、执行 token 成本、未见任务泛化和跨 executor 迁移共同评估结果。

该基准分为两个阶段：Creation 要求模型从无策略的弱种子创建完整执行系统；Evolution 要求模型依据下游运行反馈继续修改自己创建的 code harness。论文结论是，前沿模型已能创建可用 harness，且可做局部反馈驱动改进；但创建质量因领域而异，演化的可见分数增益不稳定，且向 held-out 任务和另一 executor 的迁移有限。

## 关键要点与证据

### 1. Harness 的被测职责

论文用 \(H=\langle E,T,C,S,L,V\rangle\) 描述 harness：执行循环、工具、上下文、状态/记忆、生命周期/恢复、验证。Creation 的种子仅提供稳定 CLI、被动原语和审计输出，不含这六类任务求解控制逻辑；未修改时五个下游基准均为 0 分。（第 4–6 页；附录 C）

### 2. 严格分开开发、执行与评分

评测流程为 \((L_C,D)\rightarrow H\)，随后 \((H,L_E,x)\rightarrow y\)，再由 \(J\) 评分。Self-Eval 让 creator 自行执行，测模型–harness 共适配；Unified-Eval 以 Gemini 3.1 Pro 固定执行所有生成 harness，降低 executor 差异。（第 4、6 页；附录 D）

### 3. Creation 覆盖与结果

Creation 覆盖 2,207 个不重复实例：SWE-bench Pro 731、Terminal-Bench 2.1 89、MLE-bench 75、EQ-Bench3 46、BrowseComp 1,266。Self-Eval 下 Opus 4.8 的四项非加权平均为 67.8，人工系统外部参照为 86.2；模型生成 harness 在写作上接近参照，在 MLE-bench 上超过所选参照，但代码与搜索/研究仍有明显差距。（第 6–10 页，Table 2–5）

### 4. 结构与成本发现

18 个 Code harness 均有显式 execution loop，但状态/记忆最薄弱：11 个声明 State 类，只有 1 个提供保存接口、1 个实现周期 checkpoint，26,679 条真实轨迹未见 checkpoint。编辑规模不预测成绩；MLE-bench 上相近 medal 成绩可伴随约 19 倍的 executor token 差异。（第 8–10 页；附录 B.3）

### 5. 跨模型迁移并不自然成立

更换固定 Gemini executor 后，多个生成 harness 的排名与分数变化很大；例如 Opus 的 SWE-Pro 从 69.3 降至 33.0。论文归因于提示、工具协议、预算与终止规则对创建者模型的共适配，并把 Unified-Eval 作为判断 harness 可移植性的必要补充。（第 8、10–11 页）

### 6. Evolution 的真实泛化弱于反馈集增益

Evolution 使用可见的 SWE-Pro-100 与 Terminal-Bench-89 反馈对，随后将每个正式冻结版本跑在 creator 未见的 SWE-Pro-630。Self-runtime 五条最终线在 held-out 上均较 \(H_0\) 提升 +1.43 至 +4.44；固定 Gemini 下只有 Opus 提升，其余三条退化。64 次相邻正式版本切换中，反馈与 held-out 分数仅 34 次（53.1%）同向，只有 2/9 声明最终版本是 held-out 最优。（第 11–14 页，Table 6–7）

### 7. 有效修复需要可解释失败和端到端复验

最强正例是 Opus：它发现 100 次运行中 99 次自报成功、却仅 48 次实际通过，于是定位“过早完成”并加入 completion check。反之，小 probe 常与完整评测不一致；一个 GPT-5.5 候选 5 个 Terminal probe 全过、完整集合仅得 0.584。论文认为反馈可以支持局部程序搜索，但尚不能支持稳定自我进化的强结论。（第 12–13 页）

## 事实、推断与未决问题

- **原文事实**：Creation 在若干写作与 ML 实验设置上可匹配/超过论文选择的外部人工参照；代码与搜索任务仍有差距。
- **原文事实**：Evolution 的未见集提升较反馈集小，并随 executor 改变而明显变化。
- **本 Wiki 的工程推断**：任何“自改 harness”实践都应将冻结快照、完整回归、held-out 评测、轨迹级机制触发证据和跨 executor 试验设为基础设施，而非只据局部任务分数选版本。
- **未决问题**：论文的 Evolution 仅限 code harness，held-out 仅评 SWE-Pro，且每个 creator–runtime 单元只有单条轨迹；目前尚不足以估计稳定性或推广到其他领域。

## 详细结构

- 第 1–3 页：问题动机、FDE 类比与研究结论概览。
- 第 4–7 页：基准定义、弱种子、Creation/Evolution 协议、约束与指标。
- 第 7–11 页：Creation 结果、组件触发证据、token 成本、executor 迁移。
- 第 11–14 页：Evolution 轨迹、held-out 泛化、版本选择与稳定性。
- 第 14–16 页：相关工作、局限、结论与安全说明。
- 第 22–41 页：实验配置、接口合约、系统提示与代表性轨迹。

## 相关页面

- [[concepts/harness-engineering/Harness六大组件]]
- [[concepts/harness-engineering/过早完成声明]]
- [[concepts/harness-engineering/可观测性]]
- 同目录详解：[[raw/sources/文章/字节论文：自进化框架/HarnessDev 论文技术解读]]
