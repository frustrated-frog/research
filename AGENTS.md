# AGENTS.md - 研究知识库维护手册

> 本知识库基于 LLM Wiki 模式运作：LLM 增量构建和维护一个持久的 wiki，作为你和原始文档之间的中间层。
> Wiki 是**累加的**——每次摄入新 source 或提问，wiki 都会变得更丰富。
> 你负责筛选来源、提问和思考；LLM 负责所有归档、交叉引用和维护工作。

---

## 📁 目录结构

```
研究/
├── raw/
│   └── sources/          # 原始文档（只读不修改）
├── wiki/                 # LLM 生成的 wiki 页面
│   ├── index.md          # 内容索引总览
│   ├── log.md            # 操作日志（append-only）
│   ├── All-Concepts.md   # 全局概念索引（所有概念的汇总）
│   ├── All-Sources.md    # 全局来源索引（所有来源摘要的汇总）
│   ├── concepts/         # 概念页（按项目分子目录）
│   │   ├── [项目名]/
│   │   └── ...
│   ├── entities/         # 实体页（按项目分子目录）
│   │   ├── [项目名]/
│   │   └── ...
│   ├── index/            # 各主题的索引页
│   │   ├── [项目名]/
│   │   └── ...
│   ├── summaries/        # 跨项目综合摘要
│   └── synthesis/        # 综合分析页
└── AGENTS.md             # 本文件
```

> [!note]
> 当前实际结构与早期定义有所不同：按项目分子目录组织概念/实体页，新增了 `All-Concepts.md`、`All-Sources.md`、`index/`、`summaries/`，暂未启用 `sources/` 文件夹。

---

## 📋 页面规范

### 文件命名
- 使用**中文命名**（不用 kebab-case 英文）
- 实体页：`[实体名].md`，如 `工具-Obsidian.md`
- 概念页：`[概念名].md`，如 `RAG（检索增强生成）.md`
- 来源摘要：`[来源标题].md`，如 `Sandbox机制.md`
- 综合分析：`[分析主题].md`，如 `Hermes与OpenClaw对比.md`

### 页面头部（YAML frontmatter）

```yaml
---
title: 页面标题
type: concept | entity | source | synthesis
tags: [tag1, tag2]
created: YYYY-MM-DD
updated: YYYY-MM-DD
sources: []        # 关联的原始文档
related: []        # 关联的 wiki 页面
---
```

---

## 🔄 工作流程

### Ingest（新文档摄入）

当用户将新文档放入 `raw/sources/` 并要求处理时：

1. **读取源文档**，理解核心内容
2. **与用户讨论**关键要点（如果用户参与）
3. **创建来源摘要**：存入 `wiki/summaries/` 下
4. **更新相关页面**：
   - 在 `wiki/concepts/[项目名]/` 或 `wiki/entities/[项目名]/` 下创建/更新相关页面
   - 添加交叉引用
5. **更新 index.md 和 All-Concepts.md/All-Sources.md**：在对应分类下添加新页面条目
6. **追加 log.md**：记录 ingest 事件

单个 source 可能涉及 10-15 个 wiki 页面的联动更新。

### Query（知识查询）

1. 读取 `index.md` 了解当前 wiki 构成
2. 根据用户问题定位相关页面
3. 综合多个页面的内容给出答案
4. **有价值的答案应沉淀回 wiki**：建议用户"要不要把这个分析存成 wiki 页面？"

### Lint（定期体检）

定期（建议每周一次）让 LLM 检查：
- 页面间矛盾之处
- 被新来源更新的过时信息
- 孤立页面（无 inbound 链接）
- 有提及但无专属页面的重要概念
- 可补充的交叉引用

---

## 📊 index.md 结构

```markdown
# 研究知识库 - 内容索引

## 全局索引
- [All-Concepts.md](All-Concepts.md) - 所有概念的汇总
- [All-Sources.md](All-Sources.md) - 所有来源摘要的汇总

## 按项目分类

### Hermes Agent
- 概念页：[concepts/hermes-agent/](concepts/hermes-agent/)
- 实体页：[entities/hermes-agent/](entities/hermes-agent/)
- 索引页：[index/hermes-agent/](index/hermes-agent/)

### DeerFlow Book
...

## Summaries（跨项目综合摘要）
| 页面 | 涉及概念 | 更新 |

## Synthesis（综合分析）
| 页面 | 涉及概念 | 更新 |
```

---

## 📝 log.md 格式

```markdown
## [YYYY-MM-DD] ingest | 文档标题
- 类型：source/concept/entity/synthesis
- 涉及页面：xxx, yyy

## [YYYY-MM-DD] query | 用户问题摘要
- 涉及页面：xxx
- 沉淀为：xxx.md（如果有）

## [YYYY-MM-DD] lint | 检查摘要
- 发现问题：xxx
- 已修复：xxx
```

---

## 🛠️ 工具

- **Obsidian 插件**：Web Clipper（剪藏）、Dataview（动态查询）、Graph View（图谱）
- **搜索**：使用 `search_files` 工具搜索 wiki 页面内容

---

## 💡 维护原则

1. Wiki 是累加的，**不删除**已有页面，只更新或补充
2. 矛盾点用 `> [!note]` 类型的 callout 标注，并注明来源/日期
3. 每个页面至少有一个指向其他 wiki 页面的链接
4. 用户提问的优质答案，鼓励沉淀回 wiki

---

_本文件由 LLM 和用户共同维护，随知识库演进迭代更新。_
