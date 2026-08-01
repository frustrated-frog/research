---
title: AIAgent 核心循环
id: entity/hermes-agent/aiagent核心循环
type: entity
status: active
tags: [hermes-agent, 核心引擎, run-agent-py]
aliases: []
created: 2026-04-07
updated: 2026-04-07
related: [hermescli, 网关系统, 工具架构, 会话存储]
sources: []
valid_from: null
valid_to: null
superseded_by: null
---

# AIAgent 核心循环

> 文件：run_agent.py，~9,200 行 | 核心编排引擎

## 定位

AIAgent 是 Hermes 的**同步编排引擎**，负责：

- Provider 选择
- Prompt 构建
- 工具执行
- 重试和 fallback
- 回调处理
- 压缩和持久化

一个 AIAgent 类同时服务 CLI、Gateway、ACP、Batch 和 API Server。

## 架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│ Entry Points                                                          │
│ CLI (cli.py) | Gateway (gateway/run.py) | ACP | Batch Runner | API  │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│ AIAgent (run_agent.py)                                                │
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐               │
│  │ Prompt       │  │ Provider     │  │ Tool         │               │
│  │ Builder      │  │ Resolution   │  │ Dispatch     │               │
│  │              │  │              │  │              │               │
│  │ (prompt_     │  │ (runtime_    │  │ (model_      │               │
│  │  builder.py) │  │  provider.py)│  │  tools.py)   │               │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘               │
│         │                 │                 │                       │
│  ┌──────┴───────┐  ┌──────┴───────┐  ┌──────┴───────┐               │
│  │ Compression  │  │ 3 API Modes  │  │ Tool Registry│               │
│  │ & Caching    │  │ chat_compl.  │  │ (registry.py)│               │
│  │              │  │ codex_resp.  │  │ 47 tools     │               │
│  │              │  │ anthropic    │  │ 37 toolsets  │               │
│  └──────────────┘  └──────────────┘  └──────────────┘               │
└─────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌───────────────────┐  ┌──────────────────────┐
│ Session Storage   │  │ Tool Backends        │
│ (SQLite + FTS5)  │  │ Terminal (6 backends)│
│                  │  │ Browser (5 backends) │
│ hermes_state.py  │  │ Web (4 backends)     │
│ gateway/session.py│  │ MCP (dynamic)        │
└───────────────────┘  │ File, Vision, etc.   │
                       └──────────────────────┘
```

## 三种 API 模式

| 模式 | 文件 | 说明 |
|------|------|------|
| chat_completions | 通用对话补全 | OpenAI 兼容格式 |
| codex_responses | Codex 专用 | GitHub Copilot 格式 |
| anthropic_messages | Claude 专用 | Anthropic Messages API |

## 核心子系统

### 1. Prompt Builder

文件：`agent/prompt_builder.py`

组装系统提示，包含：

- Personality（SOUL.md）
- Memory（MEMORY.md、USER.md）
- Skills
- Context Files（AGENTS.md、.hermes.md）
- Tool-use guidance
- Model-specific instructions

### 2. Context Compressor

文件：`agent/context_compressor.py`

当对话超过阈值时，摘要中间对话轮次，保持上下文精简。

### 3. Provider Runtime Resolution

文件：`hermes_cli/runtime_provider.py`

运行时解析器，在 CLI、Gateway、Cron、ACP、Auxiliary 调用间共享。映射 `(provider, model)` → `(api_mode, api_key, base_url)`。

支持 18+ 提供商，包括 OAuth 流程、凭证池、别名解析。

## 数据流

```
用户输入 → HermesCLI.process_input()
  → AIAgent.run_conversation()
  → prompt_builder.build_system_prompt()
  → runtime_provider.resolve_runtime_provider()
  → API调用 (chat_completions / codex_responses / anthropic_messages)
  → tool_calls?
    → model_tools.handle_function_call()
    → 循环
  → 最终响应
  → display
  → 保存到 SessionDB
```

## 设计原则

| 原则 | 实现 |
|------|------|
| 提示稳定性 | 系统提示在会话期间不变，不破坏 LLM 前缀缓存 |
| 可观测执行 | 每个工具调用通过回调对用户可见 |
| 可中断 | API 调用和工具执行可被用户输入或信号取消 |
| 平台无关核心 | 一个 AIAgent 类服务所有入口 |
| 松耦合 | 可选子系统使用注册表模式和 check_fn 门控 |

## 关联页面

- [[entities/hermes-agent/hermescli|hermescli]]
- [[entities/hermes-agent/网关系统|网关系统]]
- [[concepts/hermes-agent/工具架构|工具架构]]
- [[entities/hermes-agent/会话存储|会话存储]]
