---
title: AGENTS.md文件
type: entity
tags: [AGENTS.md, CLAUDE.md, instruction, harness]
created: 2026-05-14
updated: 2026-05-14
sources: []
related: []
---

# AGENTS.md文件

**AGENTS.md**（或 **CLAUDE.md**）是 AI agent 的"着陆页"——进入项目后第一个读取的文件，提供项目概览、运行命令、硬约束和专题文档链接。

## 在系统中的位置

```
项目根目录/
├── AGENTS.md              ← agent 的"着陆页"
├── src/
│   └── ...
├── docs/
│   ├── api-patterns.md    ← 专题文档（按需加载）
│   ├── database-rules.md
│   └── testing-standards.md
└── ...
```

## 核心内容

一个好的 AGENTS.md 应该：

### 必须包含（5-15 条，不可违反）
```markdown
# 项目概览
Python 3.11 FastAPI 后端，PostgreSQL 15 数据库。

## 快速开始
- 安装：make setup
- 测试：make test
- 完整验证：make check

## 硬约束
- 所有 API 必须走 OAuth 2.0 认证
- 所有数据库查询必须用 SQLAlchemy 2.0 语法
- 所有 PR 必须通过 pytest + mypy --strict + ruff check

## 专题文档
- [API 设计规范](docs/api-patterns.md) — 添加新端点时必读
- [数据库操作约束](docs/database-rules.md) — 涉及数据库修改时必读
```

### 不是百科全书，而是路由器
- 50-200 行就够了
- 放不下就拆分到 docs/ 目录
- 让 agent 按需去读

## 常见错误

1. **巨型文件**：塞进 600 行，所有规则混在一起
2. **缺少验证命令**：告诉 agent 做什么，但不告诉 agent 怎么验证
3. **缺少硬约束**：只有建议性语言，没有不可违反的规则
4. **过期内容**：文档和代码脱节，agent 遵守了但代码已经变了

## 与其他组件的关系

_关联概念：[[concepts/harness-engineering/仓库即事实来源]] [[concepts/harness-engineering/Harness驾驭层]] [[concepts/harness-engineering/功能清单]]_
