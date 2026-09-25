---
id: index/结构感知检索
title: 结构感知检索
type: index
status: active
tags: [RAG, information-retrieval, document-structure]
aliases: [structure-aware-retrieval]
created: 2026-09-07
updated: 2026-09-07
sources: []
related:
  - concept/结构感知检索/目录结构增强检索
  - source/stair-structure-aware-information-retriever
valid_from: null
valid_to: null
superseded_by: null
---

# 结构感知检索

本主题关注如何把目录、标题树、章节路径等全局文档结构作为检索与路由信息，而非仅将文本切成固定长度片段。

## 概念页

| 页面 | 描述 |
|---|---|
| [[concepts/结构感知检索/目录结构增强检索]] | 先定位结构节点、再进入节点正文的检索思路与边界 |

## 来源

| 页面 | 描述 |
|---|---|
| [[summaries/结构感知信息检索（STAIR）]] | ToC 叶节点生成、SearchTome 基准和 STAIR 的实证结果 |

## 当前边界

现有来源只验证了存在可用 ToC 的教材中的叶章节定位，不应外推为对无结构语料、端到端回答质量或企业规模索引的结论。
