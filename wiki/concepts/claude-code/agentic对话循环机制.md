---
title: Agentic 对话循环机制
type: concept
tags: [claude-code, agentic-loop, 对话系统, 流式处理]
created: 2026-04-06
updated: 2026-04-06
sources: [claude-code-deep-dive-main/05-Agentic对话循环机制.md]
related: [工具架构与注册机制, 权限模型与审批流程, query-ts实现, queryengine对话引擎]
---

# Agentic 对话循环机制

> Claude Code 的核心——能够自主规划、执行工具调用、并根据结果调整策略的持续交互系统

## 三层循环结构

```
┌─────────────────────────────────────────────┐
│ 会话层 - QueryEngine（整个会话期间存活）       │
│  管理：消息历史、权限拒绝、文件读取缓存         │
└─────────────────────┬───────────────────────┘
                      │ submitMessage()
                      ▼
┌─────────────────────────────────────────────┐
│ 对话轮次层 - query()（单次用户请求）          │
│  管理：messages、toolUseContext、turnCount    │
└─────────────────────┬───────────────────────┘
                      │ yield 工具调用
                      ▼
┌─────────────────────────────────────────────┐
│ 工具执行层 - StreamingToolExecutor           │
│  管理：TrackedTool 队列、并发状态              │
└─────────────────────────────────────────────┘
```

## 核心组件

### QueryEngine
会话级状态管理器，每个会话一个实例持续到会话结束。
- `mutableMessages`：跨轮次累积的完整消息历史
- `permissionDenials`：SDK 报告用的权限拒绝记录
- `totalUsage`：累计 token 使用量
- `readFileState`：文件读取缓存

### query() — Agentic Loop 心脏
1732 行的 `AsyncGenerator`，核心是 `while(true)` 主循环：
1. **上下文预处理**：applyToolResultBudget → snipCompact → microcompact → contextCollapse → autocompact
2. **API 调用**：deps.callModel() 流式请求
3. **工具执行**：StreamingToolExecutor 并行/串行执行
4. **循环判定**：needsFollowUp 决定继续或终止

### StreamingToolExecutor
流式并发执行器，在 API 流式响应期间就开始执行工具：
- **并发安全工具**（Read/Grep/Glob）：并行执行
- **非并发工具**（Edit/Write/Bash）：串行执行
- 通过 `pendingProgress` 实时报告长时间运行工具的进度

## 对话循环三大分支

| 分支 | 条件 | 动作 |
|------|------|------|
| **工具执行** | API 响应包含 tool_use 块 | 并行/串行执行工具，continue |
| **错误恢复** | 可恢复错误（413/max_output_tokens 等） | 延迟揭示，先尝试恢复 |
| **终止** | 无 tool_use / 不可恢复错误 | return Terminal |

## 错误恢复机制（优先级从低到高）

| 错误类型 | 第一次恢复 | 第二次恢复 | 最终 |
|---------|-----------|-----------|------|
| `prompt_too_long` | Context Collapse | Reactive Compact | return |
| `media_size_error` | Reactive Compact | — | return |
| `max_output_tokens` | 8k→64k 重试 | 多轮继续 | yield 错误 |
| `overloaded_error` | 指数退避重试 | — | throw |

**核心设计：延迟揭示** — 可恢复错误先尝试恢复，只有全部失败才揭示给调用者。

## 四级上下文压缩

| 层级 | 触发条件 | 压缩效果 | 可逆 |
|------|---------|---------|------|
| **Snip** | 历史消息超 50 轮 | 移除早期消息 | ✗ |
| **Microcompact** | 相同文件被读多次 | 折叠重复读取 | ✓ |
| **Context Collapse** | 检测到可折叠模式 | 保留原始+投影视图 | ✓ |
| **Autocompact** | Token 超限 90% | 生成摘要替换 | ✗ |

## 关键设计原则

1. **分层状态管理**：会话层 / 轮次层 / 工具层各司其职
2. **延迟计算**：工具在流式接收期间就开始执行
3. **错误恢复优于错误报告**：多层 fallback + 延迟揭示
4. **并发安全优先**：写操作强制串行化
5. **可观测性**：每个关键操作都有 checkpoint，transition 字段记录决策原因
