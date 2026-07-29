---
title: HermesCLI
type: entity
tags: [hermes-agent, cli, 交互界面]
created: 2026-04-07
updated: 2026-04-07
related: [aiagent核心循环, 网关系统]
---

# HermesCLI

> 文件：cli.py，~8,500 行 | Hermes 的交互式终端 UI

## 定位

HermesCLI 是用户与 Hermes Agent 交互的主要界面，提供：
- 交互式终端 UI
- Setup 向导
- 配置管理
- 子命令系统

## 主要子命令

| 命令 | 说明 |
|------|------|
| `hermes` | 开始聊天 |
| `hermes model` | 选择 LLM 提供商和模型 |
| `hermes tools` | 配置工具集启用/禁用 |
| `hermes setup` | 全配置向导 |
| `hermes gateway` | 启动消息网关 |
| `hermes skills` | 技能管理（搜索/安装/查看） |
| `hermes doctor` | 诊断问题 |
| `hermes status` | 检查配置状态 |
| `hermes version` | 查看版本 |
| `hermes update` | 更新到最新版本 |
| `hermes chat` | 带参数的聊天（--toolsets, -q 等） |
| `hermes -c / --continue` | 恢复上一个会话 |
| `hermes acp` | 启动 ACP 服务器 |

## CLI 聊天模式

```bash
# 使用特定工具集
hermes chat --toolsets "web,terminal"

# 单次查询
hermes chat -q "Hello! What tools do you have available?"

# 带工具集的单次查询
hermes chat --toolsets skills -q "What skills do you have?"
```

## 命令注册表

核心文件：`hermes_cli/commands.py`

包含 `COMMAND_REGISTRY` — 所有 slash 命令的中央定义。

## 交互功能

- **Slash 命令**：输入 `/` 查看所有可用命令的自动完成下拉
- **多行输入**：Alt+Enter 或 Ctrl+J 添加新行
- **中断 Agent**：输入新消息按 Enter，或 Ctrl+C
- **会话恢复**：退出时打印 resume 命令

## 配置系统

核心文件：`hermes_cli/config.py`

- DEFAULT_CONFIG：默认配置
- OPTIONAL_ENV_VARS：可选环境变量
- migration：配置迁移逻辑

## 主题系统

文件：`hermes_cli/skin_engine.py`

支持 CLI 界面主题定制。

## 终端回调

文件：`hermes_cli/callbacks.py`

处理：
- clarify：请求用户澄清
- sudo：密码提升请求
- approval：命令审批
