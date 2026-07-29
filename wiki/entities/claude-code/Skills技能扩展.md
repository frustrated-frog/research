---
title: Skills 技能扩展
type: entity
tags: [claude-code, skills, prompt-as-capability, 扩展机制]
created: 2026-05-25
updated: 2026-05-25
sources: [claude-code-deep-dive-main/26-Skills技能扩展.md]
related: [工具架构与注册机制, agentic对话循环机制]
---

# Skills 技能扩展

> Claude Code 的 Skill 是可复用工作流，不是普通工具。它把“怎么完成某类任务”的经验封装成 Markdown、元数据和可选资源。

## Skill 的本质

Skill 与 Tool 的区别：

| 抽象 | 本质 | 例子 |
|------|------|------|
| Tool | 原子动作 | Read、Bash、Grep、Edit |
| Skill | 工作流 / 方法论 | code review、simplify、批量重构 |

Tool 解决“能做什么动作”，Skill 解决“遇到某类任务应该怎么做”。

这背后的思想是：

> Prompt 即能力。专业经验可以被封装成可复用的操作手册。

## Skill 来源

Claude Code 的技能可以来自多个层级：

| 来源 | 特点 |
|------|------|
| Bundled Skills | 内置核心技能，随系统发布 |
| 项目技能 | `.claude/skills/`，团队共享 |
| 用户技能 | `~/.claude/skills/`，个人全局可用 |
| 管理策略 | 组织级强制技能 |
| MCP Skills | 由 MCP Server 动态提供 |

这种多来源设计让 Skill 同时支持产品内置、项目定制、个人习惯和组织治理。

## Skill 文件结构

典型 Skill 是一个目录加 `SKILL.md`：

```markdown
---
name: code-review
description: 系统性代码审查
when_to_use: 用户想审查代码、找 bug、做 code review
allowed-tools:
  - Read
  - Grep
  - Glob
context: fork
---

# 代码审查工作流

1. 收集变更
2. 分析风险
3. 输出 findings
```

Frontmatter 负责触发和权限，正文负责工作流。

## 触发机制

Skill 最关键的字段是 `when_to_use` 或 description 类触发说明。模型不是靠硬编码 if/else 选择 Skill，而是根据用户请求语义匹配。

这意味着 Skill 的触发描述本身就是接口设计。好的触发描述要说明：

- 什么用户意图应该触发
- 什么场景不应该触发
- 是否替代某些默认工具或流程
- 输出应该是什么

## Inline 与 Fork

Claude Code 的 Skill 有两种执行模式：

| 模式 | 特点 | 适合场景 |
|------|------|----------|
| Inline | 在主对话中执行，共享上下文 | 轻量任务、快速查询 |
| Fork | 在独立子 Agent 中执行，隔离上下文 | 长任务、大规模重构、深度审查 |

Fork 模式的价值在于：中间探索不会污染主对话，主线程只接收最终结果。

## 权限边界

Skill 不是无限信任的 prompt。Claude Code 对 Skill 有安全控制：

- allowed-tools 限制可用工具
- 安全属性白名单
- 非白名单属性触发权限请求
- MCP Skills 有更严格限制
- 项目/用户/组织来源有不同信任级别

这说明 Skill 虽然是 Markdown，但仍然属于 Agent 的能力扩展边界，需要治理。

## 与 DeerFlow 的对照

DeerFlow 的 [[../../entities/deerflow-book/Skills系统]] 更强调渐进式加载：

```text
name + description + location
  -> 按需读取 SKILL.md
  -> 按需读取 scripts / references / assets
```

Claude Code 更强调多来源、权限、Inline/Fork 执行模式。两者共同说明：

> Skill 是把经验变成系统能力的载体。

## 设计启发

设计一个好 Skill，要回答：

- 它解决哪类任务？
- 它何时触发？
- 它按什么步骤执行？
- 它能用哪些工具？
- 它是否需要隔离上下文？
- 它如何输出可复用结果？

如果一个能力需要多步判断、多工具配合、稳定输出格式，就应该考虑做成 Skill，而不是继续塞进系统提示。

