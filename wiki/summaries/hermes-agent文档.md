---
title: Hermes Agent 官方文档
type: source
tags: [hermes-agent, nous-research, agent, llm]
created: 2026-04-07
updated: 2026-04-07
sources: [hermes-agent官方文档]
related: [hermes-agent核心定位, hermes-agent架构, hermes-agent安装配置, hermes-agent记忆系统, hermes-agent技能系统, hermes-agent工具系统]
---

# Hermes Agent 官方文档

> 来源：https://hermes-agent.nousresearch.com/docs/
> 抓取时间：2026-04-07

## 项目定位

Hermes Agent 是 **Nous Research** 开发的一个具有自我改进能力的 AI Agent。核心特点：

- **内置学习循环**：从经验中创建技能、使用中改进技能、主动持久化知识、跨会话深化用户模型
- **运行灵活**：$5 VPS、GPU 集群、无服务器基础设施（Daytona、Modal）均可运行，闲置时几乎零成本
- **多平台接入**：CLI、Telegram、Discord、Slack、WhatsApp、Signal、Matrix、Email、SMS 等 14+ 平台，一个网关统一接入
- **研究导向**：由模型训练团队创建，支持批量处理、轨迹导出、RL 训练

## 核心功能一览

| 功能 | 说明 |
|------|------|
| 记忆系统 | MEMORY.md + USER.md 持久化记忆，Honcho 跨会话记忆 |
| 技能系统 | 按需加载的知识文档，支持技能自创建和 Hub 共享 |
| 工具系统 | 47 个内置工具，20 个工具集，MCP 协议支持 |
| 终端后端 | local、Docker、SSH、Daytona、Modal、Singularity 6 种 |
| 消息网关 | 14+ 平台适配器，统一会话路由 |
| 语音模式 | CLI/Telegram/Discord 实时语音交互 |
| 定时任务 | 内置 Cron，任意平台投递 |
| ACP 集成 | VS Code / Zed / JetBrains 编辑器原生集成 |

## 目录结构

```
hermes-agent/
├── run_agent.py          # AIAgent — 核心对话循环（~9200行）
├── cli.py                # HermesCLI — 交互式终端UI（~8500行）
├── model_tools.py        # 工具发现、schema收集、分发
├── toolsets.py           # 工具分组和平台预设
├── hermes_state.py       # SQLite会话/状态数据库（FTS5）
├── agent/                # Agent内部模块
│   ├── prompt_builder.py        # 系统提示组装
│   ├── context_compressor.py    # 对话压缩算法
│   └── ...
├── gateway/              # 消息平台网关
│   ├── run.py                   # GatewayRunner（~5800行）
│   └── platforms/               # 14个平台适配器
└── skills/              # 内置技能
```

## 数据流

### CLI 会话
```
用户输入 → HermesCLI.process_input()
  → AIAgent.run_conversation()
  → prompt_builder.build_system_prompt()
  → runtime_provider.resolve_runtime_provider()
  → API调用
  → 工具调用循环
  → 最终响应 → 显示 → 保存到SessionDB
```

### 网关消息
```
平台事件 → Adapter.on_message()
  → GatewayRunner._handle_message()
  → 授权用户 → 解析会话key
  → AIAgent.run_conversation()
  → 响应回传
```

## 设计原则

| 原则 | 实践 |
|------|------|
| 提示稳定性 | 系统提示在会话期间不变，不破坏缓存 |
| 可观测执行 | 每个工具调用都通过回调对用户可见 |
| 可中断 | API调用和工具执行可被用户输入或信号取消 |
| 平台无关核心 | 一个 AIAgent 类服务 CLI、网关、ACP、批处理 |
| 松耦合 | MCP、插件、记忆提供者使用注册表模式 |
| Profile隔离 | 每个 profile 有独立的 HERMES_HOME、配置、记忆、会话 |

## 快速安装

```bash
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
source ~/.bashrc  # 或 source ~/.zshrc
hermes            # 开始聊天！
```

## 支持的 LLM 提供商

Nous Portal、OpenAI Codex、Anthropic Claude、OpenRouter、OpenAI、DeepSeek、GitHub Copilot、Hugging Face、Kilo Code、Vercel AI Gateway、Custom Endpoint（VLLM、SGLang、Ollama）等 18+。
