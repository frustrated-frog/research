---
id: entity/deerflow-book/Skills系统
title: Skills 系统
type: entity
status: active
tags: [wiki, deerflow-book]
aliases: []
created: 2026-07-29
updated: 2026-08-01
sources: []
related: []
valid_from: null
valid_to: null
superseded_by: null
---

# Skills 系统

**Skills 系统** 是 DeerFlow 的能力扩展核心——通过编写 `SKILL.md` 文件，开发者可以定义任何复杂度的 Agent 工作流程，并将这些工作流程作为 DeerFlow 的内置能力来使用。Skills 系统让 DeerFlow 成为一个"可以学新技能的 AI 助手"，而非"只能做固定事情的工具"。

## Skill 的本质

一个 Skill 就是一个**目录加上一个 `SKILL.md` 文件**：

```
skills/
├── public/                    # 内置 Skill
│   ├── deep-research/
│   │   └── SKILL.md           # 必须：元数据 + 操作指令
│   │   ├── scripts/           # 可选：DuckDB 分析脚本等
│   │   ├── references/        # 可选：按需读取的参考文档
│   │   └── assets/           # 可选：模板、图片等静态资源
│   └── data-analysis/
│       ├── SKILL.md
│       └── scripts/
│           └── analyze.py
└── custom/                    # 用户自定义 Skill
    └── my-skill/
        └── SKILL.md
```

`SKILL.md` 包含两部分：YAML frontmatter（必填） + Markdown 正文（操作手册）。

## SKILL.md 格式

```markdown
---
name: skill-name              # 必填：唯一标识符（kebab-case）
description: 触发描述          # 必填：Agent 据此决定是否加载本 Skill
license: MIT                  # 可选：许可证
---

# Skill 操作手册（Markdown）

## 概述
本 Skill 的核心能力和适用场景

## 工作流程
分步骤的操作指南

## 参考资源
当需要时读取 references/ 目录下的文档
```

`name` 和 `description` 是解析的必填字段，缺少任何一个会导致 Skill 被静默跳过。

## 加载机制

### 发现

`load_skills()` 函数递归扫描 `skills/public/` 和 `skills/custom/` 目录：

```python
def load_skills():
    for category in ["public", "custom"]:
        category_path = skills_path / category
        for current_root, dir_names, file_names in os.walk(category_path):
            if "SKILL.md" not in file_names:
                continue
            skill = parse_skill_file(skill_file, category=category)
```

### 解析

`parse_skill_file()` 用简单的正则表达式提取 YAML frontmatter（零外部依赖）：

```python
front_matter_match = re.match(r"^---\s*\n(.*?)\n---\s*\n", content, re.DOTALL)
# 简单的 key-value 解析（无 PyYAML）
for line in front_matter.split("\n"):
    if ":" in line:
        key, value = line.split(":", 1)
        metadata[key.strip()] = value.strip()
```

### 启用控制

通过 `extensions_config.json` 控制运行时启用状态：

```json
{
  "skills": {
    "deep-research": { "enabled": true },
    "video-generation": { "enabled": false }
  }
}
```

未明确配置时，`public` 和 `custom` 类别的 Skill 默认启用。

## 渐进式加载

Skills 系统的核心创新——三层渐进式加载（详见[[../../concepts/deerflow-book/渐进式加载|渐进式加载]]）：

```
第一层：name + description + location
       ↓ 始终在 System Prompt 中（约 100 词）
       Agent 决定触发哪个 Skill
       ↓
第二层：SKILL.md 正文（按需读取）
       ↓ Agent 主动调用 read_file
第三层：scripts/、references/、assets/
       ↓ 执行过程中按需读取
```

17 个内置 Skill 不会把数千行操作手册全部塞进上下文，而是让 Agent 按需加载。

## 内置 Skill 全览

| Skill | 用途 |
|-------|------|
| `deep-research` | 系统化多角度网络研究 |
| `data-analysis` | DuckDB SQL 数据分析 |
| `chart-visualization` | 26 种图表类型的智能可视化 |
| `frontend-design` | 高质量前端界面代码生成 |
| `web-design-guidelines` | UI 代码审查（基于 Vercel 指南） |
| `image-generation` | 结构化图像生成 |
| `video-generation` | 结构化视频生成 |
| `podcast-generation` | 双主持人播客音频生成 |
| `ppt-generation` | PowerPoint 文件生成 |
| `github-deep-research` | GitHub 仓库多轮深度分析 |
| `consulting-analysis` | 咨询级专业分析报告 |
| `skill-creator` | 创建和评估 Skill 的元技能 |
| `bootstrap` | 生成个性化 SOUL.md |
| `find-skills` | 发现和安装可用 Skill |
| `surprise-me` | 创造性 Skill 组合 |
| `claude-to-deerflow` | HTTP API 平台交互 |
| `vercel-deploy` | Vercel 部署 |

## Skill 的 description 即触发器

`description` 字段是 Agent 决定是否加载 Skill 的唯一依据，编写质量直接影响触发准确率。最佳实践：

```markdown
description: >
    Use this skill instead of WebSearch for ANY question requiring
    web research. Trigger on queries like "what is X", "explain X",
    "compare X and Y", "research X", or before content generation tasks.
```

推动式写法（"instead of WebSearch"、"Trigger on queries like"）比被动描述更有效。

## Skill vs. MCP

| 对比维度 | Skill | MCP Server |
|---------|-------|------------|
| 本质 | 行为模式（怎么做） | 工具连接器（能用什么） |
| 形式 | Markdown 文件 + 脚本 | 运行中的服务进程 |
| 作用 | 提供工作流、方法论、操作手册 | 暴露具体的可调用工具 |
| 加载方式 | Agent 按需读取文件 | 启动时注册工具到 Agent |

两者互补：Skill 定义"做什么以及怎么做"，MCP 提供"需要的工具"。最强大的组合是 Skill（流程编排）+ MCP（工具能力）。

## 小结

Skills 系统是 DeerFlow 能力的边界——通过将工作流程封装为 `SKILL.md` 文件，实现了无需修改框架代码即可扩展 Agent 能力的架构目标。零外部依赖的解析器、三层渐进式加载、按需读取的外部资源，以及 public/custom 双目录体系，共同构成了一个既强大又精简的可扩展系统。17 个内置 Skill 证明了这一架构的实用性，而 Skill 与 MCP 的互补关系则提供了完整的能力扩展图谱。

---

_关联概念：[[../../concepts/deerflow-book/渐进式加载|渐进式加载]] [[#Skill vs. MCP|Skill vs MCP]] [[LeadAgent大脑]]_
