---
title: Harness Engineering
type: index
tags: [harness, agent, AI-engineering]
created: 2026-05-14
updated: 2026-05-14
---

# Harness Engineering

Harness Engineering 是让 AI coding agent 从不可预测变得可信赖的系统化工程方法论。

## 核心命题

> **模型能力强不等于执行可靠。** 同一模型在空白环境和有完整 harness 的环境中表现有本质差异。

## 内容索引

### 概念页
| 页面 | 描述 |
|------|------|
| [[concepts/harness-engineering/Harness驾驭层]] | 模型权重之外的工程基础设施 |
| [[concepts/harness-engineering/仓库即事实来源]] | 仓库作为权威信息源 |
| [[concepts/harness-engineering/上下文连续性]] | 跨会话状态持久化 |
| [[concepts/harness-engineering/任务边界与WIP限制]] | WIP=1 防止过度延伸 |
| [[concepts/harness-engineering/功能清单]] | harness 的脊梁骨 |
| [[concepts/harness-engineering/过早完成声明]] | 防止 agent 提前交卷 |
| [[concepts/harness-engineering/端到端测试]] | 验证系统级正确性 |
| [[concepts/harness-engineering/可观测性]] | 双层可观测性架构 |
| [[concepts/harness-engineering/初始化阶段]] | 先打地基再砌墙 |
| [[concepts/harness-engineering/会话交接]] | 清洁状态五条件 |

### 实体页
| 页面 | 描述 |
|------|------|
| [[entities/harness-engineering/AGENTS.md文件]] | agent 着陆页 |
| [[entities/harness-engineering/进度文件]] | 会话日记本 |
| [[entities/harness-engineering/功能清单文件]] | 机器可读状态机 |
| [[entities/harness-engineering/冲刺合同]] | sprint 验收协议 |

### 来源
| 页面 | 描述 |
|------|------|
| [[summaries/Harness_Engineering_完整课程]] | 完整12讲课程摘要 |

## 五分钟理解 Harness Engineering

1. **第1-2讲**：Harness 是什么？——马鞍，不是模型本身
2. **第3-4讲**：怎么组织？——仓库即事实来源，指令要拆分
3. **第5-6讲**：跨会话怎么办？——状态持久化 + 初始化阶段
4. **第7-9讲**：怎么做任务？——WIP=1 + 功能清单 + 防止假胜利
5. **第10-12讲**：怎么验证？——E2E + 可观测性 + 清洁交接

## 延伸阅读

- [OpenAI: Harness Engineering](https://openai.com/index/harness-engineering/)
- [Anthropic: Effective Harnesses for Long-Running Agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- [walkinglabs/learn-harness-engineering](https://github.com/walkinglabs/learn-harness-engineering)
