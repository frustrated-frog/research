---
title: Pi Agent Harness 主题索引
id: index/pi-agent-harness
type: index
status: active
tags: [Pi, Agent, Harness, 编码Agent]
aliases: []
created: 2026-07-29
updated: 2026-07-29
sources: [raw/sources/pi/Pi Agent Harness源码设计精读.md]
related: [summaries/Pi Agent Harness源码设计精读.md]
valid_from: null
valid_to: null
superseded_by: null
---

# Pi Agent Harness

Pi 是一个分层 Agent Harness 项目：底层统一多 Provider 模型协议，中层实现工具调用循环和有状态 Agent，编码会话层组合技能、扩展、系统提示、工具、压缩、重试与追加式会话树，并将同一运行时投射到 TUI、打印、JSON、RPC 和服务端监督环境。

## 深度技术文档

- [Pi Agent Harness 源码设计精读](../../../raw/sources/pi/Pi%20Agent%20Harness源码设计精读.md) — 完整解释流程、状态所有权、工具执行、消息投影、会话树、压缩、扩展和 Durable Harness 设计

## 来源摘要

- [[summaries/Pi Agent Harness源码设计精读]] — 核心设计结论与文档结构

## 与现有知识的连接

- [[concepts/harness-engineering/Harness驾驭层]] — Harness 为模型不确定性提供确定执行边界
- [[concepts/harness-engineering/上下文连续性]] — 追加式会话、摘要和保存点共同维持长时程连续性
- [[concepts/claude-code/流式响应与事件处理]] — 流式事件既驱动 UI，也形成持久化和生命周期屏障
- [[concepts/claude-code/工具架构与注册机制]] — 工具定义、启用集合、策略钩子和执行包装的关系
- [[concepts/claude-code/上下文压缩策略]] — 安全切点、结构化摘要与上下文溢出恢复

---

_关联概念：[[summaries/Pi Agent Harness源码设计精读]] [[concepts/harness-engineering/Harness驾驭层]] [[concepts/claude-code/工具架构与注册机制]]_

