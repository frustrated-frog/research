---
title: Sub-Agent 机制
id: entity/claude-code/Sub-Agent机制
type: entity
status: active
tags: [claude-code, sub-agent, coordinator, worker, 并行执行]
aliases: []
created: 2026-05-25
updated: 2026-05-25
sources: [claude-code-deep-dive-main/21-Sub-Agent机制.md]
related: [agentic对话循环机制, Skills技能扩展, queryengine对话引擎]
valid_from: null
valid_to: null
superseded_by: null
---

# Sub-Agent 机制

> Claude Code 的 Sub-Agent 机制是受控并行，而不是“多开几个模型”。核心是 Coordinator / Worker 分工、任务通信协议和隔离执行环境。

## 核心结构

```text
用户请求
  -> Coordinator Agent
      -> Worker Agent 1
      -> Worker Agent 2
      -> Worker Agent 3
  -> Worker 通过通知返回结果
  -> Coordinator 综合并回复用户
```

Coordinator 负责：

- 分解任务
- 启动 Worker
- 给 Worker 写自包含提示
- 接收任务通知
- 综合结果
- 决定继续、停止或新建 Worker

Worker 负责：

- 独立完成被委派的任务
- 使用允许的工具
- 报告结果、失败、进度或产物

## 协调器模式

Claude Code 的 Coordinator Mode 会限制主 Agent 的工具范围，让它专注于协调，而不是自己直接执行大量文件操作。

典型工具包括：

- Agent：创建 Worker
- SendMessage：继续已有 Worker
- TaskStop：停止 Worker
- SyntheticOutput：输出综合结果

这体现了一个重要思想：

> 协调器的价值不是亲自干活，而是正确分解、调度和综合。

## Worker 提示必须自包含

Worker 看不到完整用户对话，也不能假设知道协调器脑中的上下文。因此每个 Worker prompt 必须包含：

- 任务背景
- 具体目标
- 文件路径或范围
- 完成标准
- 可用约束
- 输出格式

坏提示：

```text
继续处理我们刚才讨论的 bug。
```

好提示：

```text
检查 src/auth/validate.ts 中 session 过期时 user 为空导致的错误。
请定位相关测试，修复空值处理，运行相关测试，并报告改动摘要。
```

## 任务通信协议

Worker 完成后，不是随便写一段文本给 Coordinator，而是通过结构化通知返回：

```xml
<task-notification>
  <task-id>...</task-id>
  <status>completed|failed|killed</status>
  <summary>...</summary>
  <result>...</result>
  <usage>...</usage>
</task-notification>
```

这让 Coordinator 可以可靠解析状态、结果和使用情况。

核心思想：

> Sub-agent 的返回值也应该是协议，而不是自由文本。

## 隔离策略

Claude Code 支持多种隔离：

| 隔离方式 | 价值 |
|----------|------|
| 本地进程 | 轻量快速 |
| Git Worktree | 文件修改互不干扰 |
| 远程环境 | 长任务、强资源、环境一致 |

Worktree 隔离尤其适合 coding agent：多个 Worker 可以在不同工作树中研究、实现、验证，避免同时写主工作区。

## 并行边界

Sub-agent 适合：

- 多角度研究
- 多模块探索
- 独立验证
- 互不依赖的实现任务

Sub-agent 不适合：

- 用户意图还不明确
- 步骤强依赖
- 多个 Worker 会写同一批文件
- 一次工具调用即可完成的小任务

并行的关键不是数量，而是任务边界。

## 与 DeerFlow 的对照

DeerFlow 的 [[../../entities/deerflow-book/子智能体]] 采用 Lead Agent + `task` 工具 + SubagentExecutor：

## 关联页面

- [[entities/claude-code/Skills技能扩展|Skills技能扩展]]
- [[entities/claude-code/queryengine对话引擎|queryengine对话引擎]]

- 每个 Sub-agent 独立消息历史
- SubagentExecutor 管理任务状态机
- `SubagentLimitMiddleware` 限制并发数量
- 前端可显示 task_started / task_running / task_completed

Claude Code 更强调 Coordinator/Worker 和 worktree/remote 隔离；DeerFlow 更强调 harness 内的轻量任务执行单元。共同点是：Sub-agent 都是受控并行，不是无限分裂。

## 设计启发

设计 Sub-agent 时先问：

- 任务是否真的可并行？
- Worker 是否有独立上下文？
- 返回结果是否结构化？
- 失败后是继续同一个 Worker，还是新建 Worker？
- 多个 Worker 的写操作如何隔离？
- 是否有并发上限？

没有这些约束的多 Agent，只是更贵、更乱的单 Agent。
