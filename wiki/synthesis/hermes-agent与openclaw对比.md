---
title: Hermes Agent 与 OpenClaw 对比
type: synthesis
tags: [hermes-agent, openclaw, 对比分析, agent平台]
created: 2026-04-07
updated: 2026-04-07
related: [hermes-agent核心定位, 工具架构, 记忆系统, 技能系统]
---

# Hermes Agent 与 OpenClaw 对比

> 两个开源 Agent 平台的架构和功能全面对比

## 概览

| 维度 | Hermes Agent | OpenClaw |
|------|-------------|----------|
| **开发团队** | Nous Research | OpenClaw 社区 |
| **定位** | 自我改进的通用 Agent | 终端原生 Agent + 框架 |
| **核心差异** | 内置学习循环，越用越强 | 注重工具生态和 MCP 集成 |
| **代码规模** | ~50,000+ 行 Python | 多语言混合 |
| **平台支持** | 14+ 消息平台 | 多 channel 支持 |

## 架构对比

### Hermes Agent 架构

```
Entry Points → AIAgent（统一核心）
               ↓
  ┌───────────┼───────────┐
  ↓           ↓           ↓
Prompt    Provider    Tool
Builder   Resolution  Dispatch
  ↓           ↓           ↓
Compression  3 API      47 Tools
& Caching   Modes      37 Toolsets
```

**特点：**
- 单核心多入口（AIAgent）
- Provider 运行时统一解析
- 工具注册表模式（自注册）
- 渐进式压缩

### OpenClaw 架构

```
Gateway / CLI → Agent (main session)
                    ↓
              Tools + Skills + Memory
                    ↓
              Multiple Providers
```

**特点：**
- Gateway 统一路由
- Session 隔离
- Skills 系统（SKILL.md）
- Memory 分层

## 核心功能对比

### 记忆系统

| 维度 | Hermes | OpenClaw |
|------|--------|----------|
| **内置记忆** | MEMORY.md + USER.md | MEMORY.md + memory/*.md |
| **容量** | 2,200 + 1,375 字符 | 无硬性限制 |
| **外部提供者** | 8个插件（Honcho等） | 插件系统 |
| **会话搜索** | SQLite + FTS5 | Session 历史 |
| **主动 nudges** | ✓ 内置 | 心跳机制 |

### 技能系统

| 维度 | Hermes | OpenClaw |
|------|--------|----------|
| **格式** | SKILL.md（YAML头） | SKILL.md |
| **自创建** | skill_manage 工具 | 无内置 |
| **Hub** | skills.sh, ClawHub, 官方 | ClawHub |
| **渐进加载** | 三级（list/view/ref） | 按需读取 |
| **条件激活** | fallback_for_toolsets | 无 |

### 工具系统

| 维度 | Hermes | OpenClaw |
|------|--------|----------|
| **内置工具** | 47 个 | 基础工具集 |
| **MCP 支持** | ✓ 内置 | ✓ 支持 |
| **终端后端** | 6种（local/docker/ssh/modal/daytona/singularity） | 基础执行 |
| **浏览器自动化** | 11 个工具 | 基础 |
| **后台进程** | process 工具 | ✓ |
| **Skill 扩展** | Hub + 自创建 | Skills |

### 消息平台

| 维度 | Hermes | OpenClaw |
|------|--------|----------|
| **平台数量** | 14+ | 多 channel |
| **接入方式** | Adapter 模式 | 统一 Gateway |
| **授权** | Allowlist + Pairing | 配置式 |
| **Slash 命令** | ✓ | ✓ |

## 终端后端对比

### Hermes 6 种终端后端

```
local → docker → ssh → singularity → modal → daytona
```

| 后端 | 用途 |
|------|------|
| local | 开发/信任任务 |
| docker | 安全隔离 |
| ssh | 远程沙箱 |
| singularity | HPC/无root |
| modal | 无服务器弹性 |
| daytona | 持久云开发 |

### OpenClaw

- 基础终端执行
- Node 集成
- 设备节点控制

## 自我改进能力

### Hermes 的自我改进循环

```
经验 → 记忆沉淀 → 技能自创建 → 使用中改进 → 跨会话进化
```

**关键特性：**
- Agent 自动创建 skills（5+ 工具调用后）
- 周期性 nudges 驱动主动持久化
- Honcho 跨会话用户建模
- 技能自改进（patch 机制）

### OpenClaw 的记忆

- 定期心跳检查
- MEMORY.md 长期记忆
- 日记式 memory/YYYY-MM-DD.md
- Session 历史追溯

**差异：Hermes 有内建的学习循环机制，OpenClaw 更偏向人工驱动记忆维护。**

## 安装和配置

### Hermes

```bash
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
hermes  # 60秒安装
```

**配置：** .env + config.yaml，hermes setup 向导

### OpenClaw

```bash
openclaw gateway start
```

**配置：** openclaw config 系统

## 适用场景

### 选择 Hermes 当：

- 需要**自我改进**的 Agent（越用越聪明）
- 需要**多消息平台**接入（Telegram/Discord/Slack 全覆盖）
- 需要**强大的终端隔离**（docker/ssh/modal 多后端）
- 需要**技能 Hub** 生态（社区共享技能）
- 研究用途（RL 训练、批量处理轨迹导出）

### 选择 OpenClaw 当：

- 需要**MCP 深度集成**
- 需要**设备节点控制**（手机通知、摄像头等）
- 需要**多 Agent 协作**（subagent 编排）
- 偏好**轻量级框架**
- 需要**高度可定制**的 Agent 行为

## 总结

两个平台都是优秀的开源 Agent 框架，各有侧重：

- **Hermes** 更像是一个**完整的、面向生产的 Agent 产品**，有内建的学习循环、丰富的平台接入、强大的隔离执行环境
- **OpenClaw** 更像一个**灵活的 Agent 框架和运行时**，注重工具生态、节点控制和 MCP 扩展

Hermes 的自我改进循环是它最独特的优势，而 OpenClaw 的节点控制和多 Agent 协作能力是它的强项。
