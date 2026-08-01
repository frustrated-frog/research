---
title: QueryEngine
id: entity/claude-code/queryengine对话引擎
type: entity
status: active
tags: [claude-code, 核心组件, 对话引擎]
aliases: []
created: 2026-04-06
updated: 2026-04-06
sources: [claude-code-deep-dive-main/04-核心架构总览.md, 05-Agentic对话循环机制.md]
related: [query-ts实现, agentic对话循环机制]
valid_from: null
valid_to: null
superseded_by: null
---

# QueryEngine — 对话引擎

> 会话级状态管理器，Claude Code 的中枢神经

## 定位

`QueryEngine` 是整个对话系统的**根对象**，每个会话创建一个实例，持续到会话结束。

## 核心状态成员

```typescript
class QueryEngine {
  config: QueryEngineConfig              // 配置项（不可变）
  mutableMessages: Message[]             // 完整消息历史（跨轮次累积）
  abortController: AbortController       // 中断控制器
  permissionDenials: SDKPermissionDenial[]  // 权限拒绝记录
  totalUsage: NonNullableUsage          // 累计 token 使用量
  readFileState: FileStateCache         // 文件读取缓存
  discoveredSkillNames: Set<string>     // 本轮发现的技能名称
  loadedNestedMemoryPaths: Set<string>  // 已加载的嵌套内存路径
}
```

## 核心职责

| 职责 | 说明 |
|------|------|
| **跨轮次状态持久化** | 管理 mutableMessages，跨 turn 累积 |
| **用户输入预处理** | processUserInput 处理斜杠命令、附件 |
| **会话持久化** | recordTranscript 早期写入 transcript |
| **成本追踪** | accumulateUsage / getTotalCost |
| **权限上下文管理** | canUseTool 包装 + 拒绝跟踪 |

## submitMessage 流程

```
用户输入
    ↓
清理 discoveredSkillNames
    ↓
包装 canUseTool（跟踪权限拒绝）
    ↓
processUserInput（处理斜杠命令、附件）
    ↓
mutableMessages.push(...messages)
    ↓
recordTranscript（早期持久化）
    ↓
加载 skills 和 plugins
    ↓
yield system_init 消息
    ↓
query(params) 或 返回本地命令输出
```

## 设计亮点

1. **早期持久化**：在进入 query() 循环之前就写入 transcript，进程被杀也能恢复
2. **双模式持久化**：交互模式同步（~4-30ms），`--bare` 模式异步
3. **权限拒绝跟踪**：自动记录所有被拒工具调用，用于 SDK 报告

## 关联页面

- [[entities/claude-code/query-ts实现|query-ts实现]]
- [[concepts/claude-code/agentic对话循环机制|agentic对话循环机制]]
