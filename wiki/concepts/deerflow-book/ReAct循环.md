---
id: concept/deerflow-book/ReAct循环
title: ReAct 循环
type: concept
status: active
tags: [wiki, deerflow-book]
aliases: []
created: 2026-07-29
updated: 2026-08-01
sources: []
related: []
valid_from: null
valid_to: null
superseded_by: null
---

# ReAct 循环

**ReAct（Reason + Act）** 是 DeerFlow Lead Agent 的核心执行循环——一个将推理（Reason）和行动（Act）交替进行的循环模式，让 Agent 能够像人类一样边想边做、边做边反思，而不是一次性生成全部答案。

## ReAct 的起源

ReAct 最早由 2022 年的论文《ReAct: Synergizing Reasoning and Acting in Language Models》提出，核心洞察是：LLM 的推理能力（从已知信息推导出结论）和行动能力（调用工具、与环境交互）是**相辅相成的**，而非互相独立的。

传统方法的问题：
- **纯推理（Reasoning-only）**：LLM 在"幻觉"中推理，越推理越偏离事实
- **纯行动（Acting-only）**：LLM 盲目执行工具调用，不理解为什么这样做

ReAct 的解决方案：**交替进行推理和行动，用推理指导行动，用行动验证推理。**

## DeerFlow 中的 ReAct 循环

DeerFlow 的 Lead Agent 基于 LangGraph 的 `create_react_agent()` 构建，执行循环如下：

```
┌─────────────────────────────────────────────┐
│              ReAct 循环（LangGraph 图）          │
│                                             │
│  ┌─────────┐   ┌─────────┐   ┌─────────┐   │
│  │  思考   │──▶│  行动   │──▶│  观察   │──▶│
│  │ (Reason)│   │ (Act)   │   │(Observe)│   │
│  └─────────┘   └─────────┘   └─────────┘   │
│       ▲                                 │   │
│       └─────────────────────────────────────┘
│                    （继续循环直到完成）            │
└─────────────────────────────────────────────┘
```

### 循环的四个阶段

**1. 思考（Reason）**：LLM 根据当前状态（messages + memory）决定下一步做什么。System Prompt 中的 `<thinking_style>` 指导 LLM 进行结构化推理：

```
You are a helpful assistant...
重要：Always do reasoning by yourself first.
Your reasoning will be internal (not shown to the user).
When you reason, put your thinking inside <reasoning></reasoning> XML tags.
```

**2. 行动（Act）**：LLM 生成工具调用（tool_calls），如 `read_file`、`bash`、`task`。这一步的输出是一个包含工具名和参数的结构化动作。

**3. 执行（Execute）**：LangGraph 执行实际的工具调用，获取返回结果。

**4. 观察（Observe）**：工具执行结果被包装为 `ToolMessage`，追加到消息历史中。下一轮推理时，LLM 会"看到"上一步的输出，据此决定下一步行动。

### 循环终止条件

ReAct 循环在以下任一条件满足时终止：
- LLM 生成了面向用户的最终回复（不再调用工具）
- 达到 `max_turns` 轮次上限
- 抛出未捕获的异常

## 与简单 Tool Use 的区别

ReAct 循环与"给 LLM 一堆工具让它随便调用"的关键区别在于**结构化的交替模式**：

| 维度 | 简单 Tool Use | ReAct 循环 |
|------|--------------|-----------|
| 推理显示 | 推理混入最终回复 | 推理在 `<reasoning>` 标签中独立可见 |
| 推理-行动顺序 | 混在一起，可能边推理边行动 | 严格交替：推理 → 行动 → 观察 → 推理 |
| 自我纠错 | 依赖 LLM 自发发现错误 | 每轮观察为下一轮推理提供验证材料 |
| 可观测性 | 外部很难理解 LLM 为什么这么做 | `<reasoning>` 标签让推理过程可追踪 |

## DeerFlow 中的两个特殊中间件

ReAct 循环在 DeerFlow 中通过两个中间件得到了额外增强：

**`TodoMiddleware`**：当 ReAct 循环因摘要截断而"失忆"时，自动将任务列表重新注入上下文，确保长时程任务不会因上下文压缩而中断。

**`DanglingToolCallMiddleware`**：当 ReAct 循环因用户中断而留下未完成的工具调用时，自动填充占位 ToolMessage，防止消息格式错误导致下一轮推理失败。

这两个中间件本质上是 ReAct 循环的"韧性层"——处理中断、压缩、异常退出等现实场景，而非仅仅在理想条件下运行。

## ThreadState 中的循环状态

ReAct 循环的每轮状态都保存在 `ThreadState` 中（通过 Checkpointer 持久化）：

```python
class ThreadState(TypedDict):
    messages: Annotated[list[BaseMessage], merge_messages]
    artifacts: Annotated[list[str], merge_artifacts]
    viewed_images: Annotated[dict[str, Any], merge_viewed_images]
    todos: list[dict[str, Any]]  # 当前任务列表（用于 TodoMiddleware）
    sandbox: dict[str, str] | None  # 当前沙箱 ID
    task_count: int  # 已创建的 Sub-agent 任务数
```

这些状态在 Agent 轮次之间保持连贯，使得跨数十轮执行的复杂任务成为可能。

## 小结

ReAct 循环是 DeerFlow Lead Agent 的大脑核心工作模式。它将 LLM 的推理能力和工具执行能力组织为一个交替进行的结构化循环，通过"推理 → 行动 → 观察"的不断迭代，让 Agent 能够处理远超单次生成能力的复杂任务。DeerFlow 在 LangGraph 的标准 ReAct 实现基础上，通过 TodoMiddleware 和 DanglingToolCallMiddleware 两个中间件，为这个循环添加了应对现实中断和上下文压缩的韧性设计。

---

_关联概念：[[entities/deerflow-book/LangGraph引擎]] [[entities/deerflow-book/LeadAgent大脑]] [[concepts/deerflow-book/长时程Agent]]_
