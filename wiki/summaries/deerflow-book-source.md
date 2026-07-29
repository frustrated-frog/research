---
title: 来源 - deerflow-book
---

# 来源信息

## deerflow-book

**仓库**：[coolclaws/deerflow-book](https://github.com/coolclaws/deerflow-book)  
**协议**：MIT License  
**分支**：main（2.0 持续开发中）

### 章节与源文件映射

| 章节 | 源文件 |
|------|--------|
| 第 1 章　DeerFlow 是什么，为什么重要 | `chapters/01-what-is-deerflow.md` |
| 第 2 章　仓库全景与技术栈 | `chapters/02-repo-overview.md` |
| 第 3 章　快速上手 | `chapters/03-quick-start.md` |
| 第 4 章　LangGraph 引擎 | `chapters/04-langgraph-engine.md` |
| 第 5 章　Lead Agent | `chapters/05-lead-agent.md` |
| 第 6 章　11 层中间件管道 | `chapters/06-middleware-pipeline.md` |
| 第 7 章　Context Engineering | `chapters/07-context-engineering.md` |
| 第 8 章　Sub-Agent 架构总览 | `chapters/08-subagent-overview.md` |
| 第 9 章　SubagentExecutor | `chapters/09-subagent-executor.md` |
| 第 10 章　并发调度与 Orchestration | `chapters/10-orchestration.md` |
| 第 11 章　长期记忆架构 | `chapters/11-memory-architecture.md` |
| 第 12 章　记忆更新流水线 | `chapters/12-memory-pipeline.md` |
| 第 13 章　Sandbox 抽象层 | `chapters/13-sandbox-abstraction.md` |
| 第 14 章　Local Sandbox 与 aio-sandbox | `chapters/14-sandbox-implementations.md` |
| 第 15 章　内置工具与社区工具 | `chapters/15-builtin-tools.md` |
| 第 16 章　MCP 扩展 | `chapters/16-mcp-extensions.md` |
| 第 17 章　Skills 系统 | `chapters/17-skills-system.md` |
| 第 18 章　编写自定义 Skill | `chapters/18-custom-skills.md` |
| 第 19 章　FastAPI Gateway | `chapters/19-fastapi-gateway.md` |
| 第 20 章　IM 渠道系统 | `chapters/20-im-channels.md` |
| 第 21 章　配置体系全解 | `chapters/21-config-system.md` |
| 第 22 章　模型配置与适配 | `chapters/22-model-config.md` |
| 第 23 章　部署与生产化 | `chapters/23-deployment.md` |
| 附录 A　阅读路径指南 | `chapters/appendix-a-reading-path.md` |
| 附录 B　配置字段速查表 | `chapters/appendix-b-config-reference.md` |
| 附录 C　术语表 | `chapters/appendix-c-glossary.md` |

### 源码路径映射

DeerFlow 主仓库：[coolclaws/deer-flow](https://github.com/coolclaws/deer-flow)

| 模块 | 源码路径 |
|------|---------|
| Lead Agent | `backend/src/agents/lead_agent/` |
| Sub-Agent | `backend/src/subagents/` |
| Middleware | `backend/src/agents/middlewares/` |
| Sandbox | `backend/src/sandbox/` |
| Skills | `backend/src/skills/` |
| Memory | `backend/src/memory/` |
| LangGraph | `backend/src/agents/lead_agent/agent.py` |
| Tools | `backend/src/tools/` |
| MCP | `backend/src/mcp/` |
| Gateway | `backend/src/gateway/` |
| Channels | `backend/src/channels/` |

---
_最后更新：2026-04-08_
