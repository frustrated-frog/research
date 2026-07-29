# Harness 与 Framework

DeerFlow 在官方文档中对自己的定位经历了一次重要修正——从"AI Framework"到"AI Harness"。这个措辞变化不是营销用语，而是精确描述了 DeerFlow 的技术本质和设计边界。

## Framework 的定义与局限

**Framework（框架）** 在软件开发中意味着：

> 你必须在 Framework 写好的"插槽"里填入你的代码。Framework 定义了控制流（Control Flow），你的代码在框架的调度下被执行。

典型的 AI Framework 如 LangChain、LlamaIndex，提供了一整套抽象——模型调用链、Prompt 模板、输出解析器、向量数据库连接器。开发者通过组合这些组件构建应用，但这种组合本身受到框架设计的约束。

Framework 的问题是：**当你需要的workflow与框架设计不符时，你会发现自己一直在"绕过框架"而非"使用框架"**。

## Harness 的定义与优势

**Harness（挽具/ harness系统）** 原本是测试领域的术语——Test Harness 是驱动被测代码执行的测试基础设施。同样，**AI Harness** 是驱动 AI 模型执行复杂任务的**基础设施**，而非约束 AI 行为的框架：

> Harness 不写你的 AI 逻辑，但让你的 AI 能够稳定、持续、可观测地执行这个逻辑。

DeerFlow 0.2 → 0.3 的重写就是这一理念的体现：没有试图发明新的 Agent 编排范式，而是基于 LangGraph（一个标准的图执行引擎）构建稳定可靠的多 Agent 协作基础设施。LangGraph 定义了图的节点和边（ReAct 循环的执行流程），DeerFlow 在此基础上填入自己的 Agent 逻辑、中间件系统、工具生态和记忆架构。

## 两者的关键区别

| 维度 | Framework | Harness |
|------|-----------|---------|
| **控制权归属** | Framework 定义控制流，你的代码在被调用时执行 | Harness 提供执行环境，你的代码驱动整个流程 |
| **抽象层次** | 高级抽象（Prompt 模板、向量存储） | 底层能力（进程隔离、文件系统、多 Agent 协作） |
| **定制边界** | 定制受框架设计约束 | 定制不受限制，只受底层能力约束 |
| **学习曲线** | 学习框架的概念和 API | 学习如何有效地组织你的任务 |
| **适用场景** | 相对标准的工作流 | 高度定制化的复杂工作流 |

## 实践中的含义

对于 DeerFlow 开发者，这意味着：

**你可以且应该用自己的代码填满 DeerFlow 提供的能力空间。** Skill 系统让你定义任意的工作流程——不只是"用哪个工具"，而是"以什么步骤、什么判断标准、什么输出格式"完成任务。Skill 的 SKILL.md 就是你对 DeerFlow 说的话："当你遇到 X 类任务时，按我的方式做。"

**DeerFlow 不会试图取代你的判断。** 它提供了执行环境（沙箱）、记忆系统（跨会话上下文）、多 Agent 协作（Sub-agent 编排），但这些能力如何使用、由谁使用、在什么场景触发，都由开发者通过 Skill 和 System Prompt 来定义。

**切换成本低。** 因为 DeerFlow 基于标准组件（LangGraph、LangChain 的模型抽象），如果有一天需要迁移到其他基础设施，迁移的主要是 Skill 定义和 Prompt 配置，而非 DeerFlow 特有的 API 调用模式。

## 在 DeerFlow 源码中的体现

这一理念在源码中有明确的体现：

- **Skill 系统**：纯文本配置（SKILL.md），零 DeerFlow 特有 API，完全是"告知 Agent 怎么做"而非"驱动框架怎么做"
- **沙箱抽象**：`Sandbox` 基类定义了统一的接口，三种实现（Local/aio-sandbox/K8s）可以替换而不影响上层工具代码
- **中间件管道**：11 层中间件是可插拔的，每个中间件都是独立的类，可以按需添加、移除或替换
- **MCP 扩展**：不修改 DeerFlow 源码，只需在配置文件中声明 MCP 服务器地址，即可让 Agent 调用任意外部服务

## 小结

DeerFlow 选择"Harness"而非"Framework"作为自我定义，是一次精准的技术定位。这个定位意味着：DeerFlow 把自己放在了"能力提供者"而非"行为规范者"的位置。开发者得到的是一套稳定的多 Agent 执行环境，而非一套需要学习的 Agent 编排语言。这解释了为什么 DeerFlow 能如此自然地支持 Skill 扩展、MCP 集成和自定义工具——因为它的设计目标从来不是"做 AI 能做的所有事"，而是"让任何 AI 任务能在它的环境中稳定运行"。

---

_关联概念：[[entities/LangGraph引擎]] [[entities/Skills系统]] [[concepts/长时程Agent]]_
