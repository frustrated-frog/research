---
title: Harness Engineering 完整课程
type: source
tags: [harness, agent, AI-engineering, openai, anthropic]
created: 2026-05-14
updated: 2026-05-14
sources: []
related: []
---

# Harness Engineering 完整课程

## 核心内容

Harness Engineering 是由 walkinglabs 出品的12讲系统课程，阐述如何通过"驾驭层"（Harness）让 AI coding agent 从不可预测变得可信赖、可追踪、可协作。核心命题：**模型能力强不等于执行可靠**，同一模型在空白环境和有完整 harness 的环境中表现有本质差异。

## 关键要点

### 第1讲：模型能力强 ≠ 执行可靠
- SWE-bench Verified 上最强 coding agent 通过率约 50-60%，真实场景更低
- Anthropic 对照实验：同一模型（Opus 4.5）+ 裸跑 = 游戏跑不起来；同一模型 + 完整 harness = 游戏可游玩
- **关键结论**：换模型是成本最高的选择，大多数失败是 harness 问题而非模型问题

### 第2讲：Harness 五子系统
harness 由五个子系统组成，类比厨房的五个功能区：
1. **指令子系统**（菜谱架）→ AGENTS.md / CLAUDE.md
2. **工具子系统**（刀具架）→ shell / 文件 / 测试
3. **环境子系统**（灶台）→ 依赖 / 服务 / 版本
4. **状态子系统**（备菜台）→ PROGRESS.md / git commits
5. **反馈子系统**（出菜检查口）→ test / lint / build

### 第3讲：仓库即事实来源
- agent 看不到 Slack/Confluence/工程师脑子里的信息
- 所有必要上下文必须沉淀到仓库
- 用"冷启动测试"检验仓库质量

### 第4讲：指令拆分到不同文件
- 巨型指令文件会触发"中间迷失效应"（LLM 对中间内容利用效率最低）
- 入口文件 50-200 行 + 专题文档路由
- 渐进式披露原则

### 第5讲：上下文连续性
- 上下文窗口是有限的，长任务必然跨会话
- 连续性工件：进度文件 + 决策日志 + git 检查点
- 重建成本：好 harness 能把新会话恢复时间从 15 分钟压到 3 分钟

### 第6讲：初始化阶段
- 初始化和实现的优化目标不同，混在一起互相拖后腿
- 自举契约四条件：能启动、能测试、能看进度、能接手下一步
- 热启动优于冷启动

### 第7讲：任务边界
- agent 天然"过度延伸"——同时做太多事，每件都做不好
- WIP=1：做完一个再做下一个
- 完成证据必须是可执行的

### 第8讲：功能清单是原语
- 功能清单是 harness 的"脊梁骨"，不是备忘录
- 每个功能项 = (行为描述, 验证命令, 当前状态) 三元组
- 状态机：not_started → active → blocked / passing

### 第9讲：防止过早宣告完成
- agent 系统性过度自信（置信度校准偏差）
- 解决方案：把"干活的人"和"检查的人"分开
- 三层终止校验：语法/静态分析 → 运行时行为 → 系统级确认

### 第10讲：端到端测试
- 单元测试通过 ≠ 任务完成
- 端到端测试不仅检测缺陷，还改变 agent 的编码行为
- 架构规则必须可执行（从文档变成 CI 检查）

### 第11讲：可观测性
- 双层可观测性：运行时信号（系统做了什么）+ 过程可观测性（为什么这样做）
- 冲刺合同：任务开始前的短期协议
- Anthropic 三 agent 架构：Planner + Generator + Evaluator

### 第12讲：会话交接
- 清洁状态五条件：构建通过、测试通过、进度已记录、无过时工件、启动路径可用
- 熵增是默认状态，"以后再清理"等于永远不清理
- 定期简化 harness：随模型能力提升移除不再必要的组件

## 详细结构

- 12讲完整课程，每讲包含：核心论点、案例数据、核心概念、延伸阅读、练习
- 来源：walkinglabs/learn-harness-engineering (GitHub)
- 参考：OpenAI Harness Engineering、Anthropic Effective Harnesses

## 相关页面

_关联概念：[[concepts/harness-engineering/Harness驾驭层]] [[concepts/harness-engineering/上下文连续性]] [[concepts/harness-engineering/任务边界与WIP限制]] [[concepts/harness-engineering/功能清单]] [[concepts/harness-engineering/过早完成声明]] [[concepts/harness-engineering/端到端测试]] [[concepts/harness-engineering/可观测性]] [[concepts/harness-engineering/初始化阶段]] [[concepts/harness-engineering/会话交接]]_
