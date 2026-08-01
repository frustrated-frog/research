---
title: 任务边界与WIP限制
id: concept/harness-engineering/任务边界与WIP限制
type: concept
status: active
tags: [WIP, task-boundary, scope, agent]
aliases: []
created: 2026-05-14
updated: 2026-05-14
sources: []
related: []
valid_from: null
valid_to: null
superseded_by: null
---

# 任务边界与WIP限制

**WIP=1** 是 agent harness 的默认安全设置：任何时刻只允许一个任务处于"进行中"状态，做完一个再做下一个。

## 问题背景

Agent 天生有"多做一点"的冲动。看到相关的事情就顺手一起做：
- "顺便把这个也改了"
- "既然看到这个问题一起修了吧"
- 同时启动 5 个功能，结果每个都是半成品

数学：假设上下文容量为 C，同时激活 k 个任务，每个任务平均获得 C/k 的推理资源。当 C/k 低于完成单个任务所需的最小阈值时，所有任务都做不完。

## 核心概念

### 过度延伸（Overreach）
agent 在一次会话中激活的任务数量超过最优值。量化：同时做 5 个功能但 0 个跑通 = overreach。

### 不足完成（Under-finish）
已启动的任务中，通过端到端验证的比例低于阈值。写了代码但没跑通测试 = under-finish。

### WIP 限制（Work-in-Progress Limit）
来自 Kanban 方法论。核心思想：限制同时在进行的任务数量。对于 agent，WIP=1 是最安全的默认值。

### 完成证据（Completion Evidence）
一个任务从"进行中"变成"已完成"必须满足的可验证条件。没有这个，agent 会用"代码看起来没问题"代替"行为通过测试"。

### 范围表面（Scope Surface）
一个 DAG 结构，每个节点是一个工作单元，边是依赖关系。状态只有四种：未开始、进行中、阻塞、已通过。

## 实践方法

### 强制 WIP=1
在 CLAUDE.md 或 AGENTS.md 里写：
```
## 工作规则
- 每次只做一个功能点
- 当前功能点端到端验证通过后，才能开始下一个
- 不要在实现功能 A 时"顺便"重构功能 B
```

### 给每个任务定义显式完成证据
```
F01: 用户注册
  验证: curl -X POST /api/register -d '{"email":"test@example.com"}' | jq .status == 201
  状态: passing
```

### 范围表面外部化
用机器可读文件（JSON 或 Markdown）记录所有任务状态。任何新会话都能直接读。

### 监控验证完成率（VCR）
VCR = 已通过验证的任务数 / 已启动的任务数。VCR < 1.0 时，阻止新任务启动。

## 数据支撑

Anthropic 实验数据：使用"WIP=1 小下一步"策略的 agent，任务完成率比使用宽泛提示的 agent 高 37%。Agent 生成的代码行数和实际完成的功能数量呈弱负相关——写得越多，完成得越少。

## 与其他概念的关系

_关联概念：[[concepts/harness-engineering/功能清单]] [[concepts/harness-engineering/Harness驾驭层]] [[concepts/harness-engineering/过早完成声明]]_
