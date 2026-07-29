---
title: query.ts 实现
type: entity
tags: [claude-code, 核心组件, query-loop, async-generator]
created: 2026-04-06
updated: 2026-04-06
sources: [claude-code-deep-dive-main/05-Agentic对话循环机制.md]
related: [queryengine对话引擎, agentic对话循环机制]
---

# query.ts — Agentic Loop 实现

> Claude Code 的心脏，1732 行的 AsyncGenerator，实现"自主多轮执行"的核心语义

## 定位

`query()` 是**对话轮次层**的核心，每次用户请求创建一个调用。它是一个 AsyncGenerator，通过 `yield` 逐步产出各类事件。

## State 对象

所有跨迭代可变状态封装在 State 中（避免 9 个独立可变变量）：

```typescript
type State = {
  messages: Message[]                          // 当前对话历史
  toolUseContext: ToolUseContext              // 工具执行上下文
  autoCompactTracking: AutoCompactTrackingState | undefined  // 自动压缩追踪
  maxOutputTokensRecoveryCount: number        // max_output_tokens 恢复计数
  hasAttemptedReactiveCompact: boolean        // 是否已尝试反应式压缩
  maxOutputTokensOverride: number | undefined // 强制输出 token 限制
  pendingToolUseSummary: Promise | undefined  // 工具摘要
  stopHookActive: boolean | undefined         // Stop Hook 是否激活
  turnCount: number                           // 当前对话轮次计数
  transition: Continue | undefined            // 上次迭代为何继续
}
```

## 核心循环结构

```typescript
async function* query(params) {
  // 初始化
  let state = initState()
  
  while (true) {
    // 1. 上下文预处理
    yield* compact(state)  // snip → microcompact → contextCollapse → autocompact
    
    // 2. API 调用
    const response = await deps.callModel(state.messages)
    
    // 3. 处理流式响应
    for (const event of response) {
      if (event.type === 'tool_use') {
        executor.addTool(event)
      }
      yield event
    }
    
    // 4. 执行工具
    if (executor.hasPendingTools()) {
      const results = await executor.executeAll()
      state.messages.push(...results)
      state.turnCount++
      continue
    }
    
    // 5. 检查终止条件
    return { reason: 'end_turn' }
  }
}
```

## 上下文预处理管道

```
applyToolResultBudget
    ↓
snipCompact（历史修剪，超过 50 轮移除早期消息）
    ↓
microcompact（折叠相同文件路径的重复读取）
    ↓
contextCollapse（提交可折叠模式到压缩存储）
    ↓
autocompact（Token 超限 90% 时生成摘要替换）
    ↓
yield compact_boundary
```

## 循环三大分支

| 分支 | 条件 | 处理 |
|------|------|------|
| **工具执行** | `needsFollowUp === true` | StreamingToolExecutor 执行，continue |
| **错误恢复** | 可恢复错误 | 延迟揭示，先尝试多级恢复 |
| **终止** | 无 tool_use 或不可恢复 | return Terminal |

## 错误恢复策略

| 错误 | 第一次 | 第二次 | 最终 |
|------|--------|--------|------|
| `prompt_too_long` | Context Collapse | Reactive Compact | return |
| `media_size_error` | Reactive Compact | — | return |
| `max_output_tokens` | 8k→64k 重试 | 多轮继续 | yield 错误 |
| `overloaded_error` | 指数退避 | — | throw |

## 终止条件

```typescript
return { reason: 'end_turn' }           // 自然结束
return { reason: 'aborted_streaming' }  // 用户中断
return { reason: 'prompt_too_long' }    // 上下文超限
return { reason: 'max_turns' }          // 达到轮次上限
return { reason: 'budget_exceeded' }    // 预算耗尽
```
