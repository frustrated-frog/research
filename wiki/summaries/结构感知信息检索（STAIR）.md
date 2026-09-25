---
id: source/stair-structure-aware-information-retriever
title: STAIR：结构感知信息检索
type: source
status: active
tags: [RAG, information-retrieval, table-of-contents, structure-aware-retrieval, DSI]
aliases: [STructure Aware Information Retriever, SearchTome, STAIR]
created: 2026-09-07
updated: 2026-09-07
sources:
  - raw/sources/文章/结构感知信息检索论文/STructure Aware Information Retriever.pdf
related:
  - concept/结构感知检索/目录结构增强检索
  - index/结构感知检索
valid_from: null
valid_to: null
superseded_by: null
---

# STAIR：结构感知信息检索

**来源**：Vineet Kumar 等，*STAIR (STructure Aware Information Retriever): A novel dataset and LLM based retriever for document structure augmentation*。

## 核心内容

[原文事实] STAIR 将长文档的完整目录与查询一起输入微调后的 Mistral，目标是生成可回答该查询的合法 ToC 叶节点。推理用 constrained generation 限制输出到有效叶节点；任务输出是章节定位，而非最终答案（PDF 第 4–5 页）。

[原文事实] SearchTome 含 18 本开放教材、6 个领域；目录从 PDF 提取，段落问题由 Mixtral 8x7B 生成，并标注对应叶标题（第 3–4 页，表 1）。在其平均指标中，STAIR 达到 R@1 82.6、R@3 90.8、nDCG@3 87.5；DSI 为 76.9、85.3、81.9（第 8 页，表 4）。

## 关键机制

- 用目录叶标题取代固定长度 chunk 或不透明 document identifier；
- 将同书完整 ToC 作为全局结构上下文；
- 从 query—leaf 对进行监督微调；
- 以合法节点受约束解码，降低生成无效章节名的概率。

## 证据与边界

[原文事实] 论文报告 DSI 与 STAIR 的 Recall@1 差异在六个领域的 randomization test 中均达到 p < 0.05；STAIR 的无效叶节点输出率约 0.05%，图示显示其在低训练样本叶节点上优于 DSI（第 6、8 页）。

[作者解释] 作者将这一差异归因于 ToC 提供了显式结构，减少了 DSI 从 query—identifier 对中隐式学习结构的负担。

[未决] 这些实验没有评测无目录语料、端到端回答忠实性、真实用户查询、跨书零样本迁移、索引更新成本或企业级规模。论文中的“hallucination”是无效叶节点输出，而非回答事实错误。

## 相关页面

- [[concepts/结构感知检索/目录结构增强检索]]
- [[index/结构感知检索]]
- 详细解读：[[raw/sources/文章/结构感知信息检索论文/结构感知信息检索论文解读]]
