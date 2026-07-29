---
title: Claude Code 与 DeerFlow 设计思想精读
type: synthesis
tags: [claude-code, deerflow-book, agent架构, 设计思想, 对比分析]
created: 2026-05-25
updated: 2026-05-25
sources:
  - raw/sources/claude-code/claude-code-deep-dive-main/
  - raw/sources/deer-flow/deerflow-book-main/
related:
  - agentic对话循环机制
  - 工具架构与注册机制
  - 权限模型与审批流程
  - 流式响应与事件处理
  - 上下文压缩策略
  - 沙箱隔离机制
  - Skills技能扩展
  - Sub-Agent机制
  - LeadAgent大脑
  - 中间件管道
  - 子智能体
  - 沙箱
  - Skills系统
  - 记忆系统
---

# Claude Code 与 DeerFlow 设计思想精读

> 这不是两套项目的功能清单，而是从 Claude Code 和 DeerFlow 中抽取出来的 Agent 工程设计思想。
> 目标是读完之后，你能回答一个更重要的问题：如果我要设计一个迷你 Agent，我应该把哪些问题显式建模？

## 总判断

Claude Code 和 DeerFlow 的共同点，是它们都不再把 LLM 当作“聊天模型”，而是把 LLM 放进一个可执行、可恢复、可扩展、可治理的行动系统里。

它们的差异在于设计中心不同：

| 项目 | 设计中心 | 最值得学的地方 |
|------|----------|----------------|
| Claude Code | 终端原生 coding agent | 如何让一个拥有本地文件、Shell、Git、工具调用权的 Agent 安全而高效地行动 |
| DeerFlow | Super Agent Harness | 如何把长时程、多工具、多 agent、多用户场景组织成可生产化的运行时 |

一句话概括：

- **Claude Code 关心的是：一个强 Agent 在开发者本机上怎么不失控、不中断、不浪费上下文。**
- **DeerFlow 关心的是：一个长任务 Agent 系统怎么被拆成稳定的 harness、middleware、sandbox、memory、skills 和 sub-agent。**

所以学习它们时，不应该只问“有哪些模块”，而应该问：

> 这些模块是在替系统显式承担什么风险、什么状态、什么生命周期？

---

## 设计思想一：Agent 不是聊天流，而是可恢复的行动循环

最容易误解 Agent 的地方，是把它看成“用户问一句，模型答一句”。在 Claude Code 和 DeerFlow 里，真正的核心都不是问答，而是一个反复执行的行动循环：

```text
用户目标
  -> 构造上下文
  -> 模型推理
  -> 产生文本 / 工具调用 / 计划
  -> 执行工具
  -> 写回工具结果
  -> 判断是否继续
  -> 直到完成、失败、等待用户或被中断
```

Claude Code 里，这个循环集中体现在 `QueryEngine -> query() -> StreamingToolExecutor`。`QueryEngine` 负责会话级状态，`query()` 是单次请求里的 agentic loop，`StreamingToolExecutor` 管理工具执行和并发状态。

DeerFlow 里，这个循环体现在 `Lead Agent + LangGraph + ThreadState`。Lead Agent 是大脑，LangGraph 负责图式执行，ThreadState 保存跨轮次状态。

这背后的思想是：

> Agent 的主循环应该被设计成状态机，而不是一段“调用模型然后看看结果”的临时代码。

如果你设计迷你 Agent，第一步不是接入工具，而是先问：

- 一轮执行有哪些状态？
- 工具调用结果写回哪里？
- 中断之后能否继续？
- 模型输出包含多个工具调用时怎么处理？
- 什么时候继续，什么时候停止？

能回答这些问题，才算真正开始设计 Agent。

延伸阅读：

- Claude Code：[[../concepts/claude-code/agentic对话循环机制|agentic对话循环机制]]、[[../entities/claude-code/queryengine对话引擎|queryengine对话引擎]]
- DeerFlow：[[../entities/deerflow-book/LeadAgent大脑|LeadAgent大脑]]、[[../concepts/deerflow-book/ReAct循环|ReAct循环]]、[[../entities/deerflow-book/LangGraph引擎|LangGraph引擎]]

---

## 设计思想二：工具不是函数，而是 Agent 能力边界

很多人会把 tool 理解成“模型可以调用的函数”。Claude Code 和 DeerFlow 都告诉我们：这太浅了。

在 Claude Code 里，一个工具不是只有 `call()`。它还带有 schema、权限检查、UI 渲染、并发安全标记、只读/破坏性标记、提示生成、进度回调等。也就是说，工具同时承担：

- 能力描述：模型知道它能做什么
- 参数协议：schema 约束输入
- 安全边界：执行前能否放行
- 并发语义：能否和其他工具并行
- 用户体验：如何展示工具调用和结果
- 可观测性：如何报告进度

DeerFlow 的工具也不是扁平函数。Lead Agent 的工具来自多层：内置工具、Sub-agent 工具、社区工具、MCP 工具，并通过 ToolFilterMiddleware 根据模型能力和场景过滤。

这背后的思想是：

> 工具是 Agent 与世界交互的协议边界，不是普通函数调用。

所以工具设计最重要的不是“能不能跑”，而是“这个动作的语义是否完整”：

| 维度 | 应该显式表达的问题 |
|------|--------------------|
| 输入 | 参数是否可验证？ |
| 权限 | 执行前是否需要审批？ |
| 副作用 | 是否会写文件、联网、花钱、删除东西？ |
| 并发 | 是否可以和其他工具同时执行？ |
| 结果 | 结果是给模型继续推理，还是给用户展示？ |
| 恢复 | 工具中断后，协议状态如何补齐？ |

如果一个工具只暴露函数签名，却不表达这些语义，它就只是“函数”，还不是 Agent 工程里的“工具”。

延伸阅读：

- Claude Code：[[../concepts/claude-code/工具架构与注册机制|工具架构与注册机制]]、[[../concepts/claude-code/权限模型与审批流程|权限模型与审批流程]]
- DeerFlow：[[../entities/deerflow-book/LeadAgent大脑|LeadAgent大脑]] 中的工具注入、[[../entities/deerflow-book/中间件管道|中间件管道]] 中的 ToolFilter / SubagentLimit 思想

---

## 设计思想三：流式输出不是 UI 技巧，而是执行架构

Claude Code 的流式机制很值得细看。模型返回的不是一整段文本，而是一系列事件：`message_start`、`content_block_start`、`content_block_delta`、`content_block_stop`、`message_delta`。系统逐步累积 text、thinking、tool_use、input_json_delta，最后形成完整 assistant message。

这个设计带来的价值不只是“用户能早点看到字”。更重要的是：

- UI 可以实时显示文本和 thinking
- 工具参数可以增量累积
- 工具执行器可以在流式响应期间更早准备
- usage、stop_reason、request id 等元数据可以回写到同一个消息对象
- transcript 和异步写队列能保持引用一致

DeerFlow 也把流式输出作为产品体验的一部分：Gateway 通过 SSE 暴露 reasoning、工具调用、Sub-agent 进度、最终结果。用户看到的不是“等很久然后得到答案”，而是“Agent 正在行动”。

这背后的思想是：

> 流式不是渲染层优化，而是 Agent 执行过程的事件化表达。

当 Agent 任务变长之后，用户真正需要的不是更快的最终答案，而是持续的可见性：

- 它现在在干什么？
- 它调用了什么工具？
- 它卡住了吗？
- 它是否在等待外部结果？
- 它是否已经偏离任务？

因此，一个好的 Agent 系统应该把执行过程设计成事件流，而不是只设计最终返回值。

延伸阅读：

- Claude Code：[[../concepts/claude-code/流式响应与事件处理|流式响应与事件处理]]、[[../concepts/claude-code/agentic对话循环机制|agentic对话循环机制]]
- DeerFlow：[[../entities/deerflow-book/LeadAgent大脑|LeadAgent大脑]] 的 SSE 执行入口、[[../entities/deerflow-book/子智能体|子智能体]] 的任务进度事件

---

## 设计思想四：安全不是一个开关，而是多层治理

Claude Code 对安全的设计非常典型：权限系统和沙箱系统分层。

权限系统是应用层治理。它通过 Allow / Ask / Deny、规则优先级、工具自身检查、Hook 拦截、权限模式等，决定某个动作是否应该发生。

沙箱系统是执行层治理。它通过 macOS `sandbox-exec` 或 Linux `bubblewrap`，限制命令的文件系统、网络和环境访问。即使权限系统放行了，沙箱仍然可以限制实际破坏范围。

DeerFlow 的沙箱思想更偏 harness 化：定义统一的 Sandbox 抽象，提供 Local、Docker/aio-sandbox、K8s 等实现，并用虚拟路径 `/mnt/user-data/` 屏蔽底层环境差异。上层工具只知道 read/write/bash/list，不关心底层是在本机、容器还是远程 Pod。

这背后的思想是：

> 权限负责“该不该做”，沙箱负责“即使做了也只能在边界内做”。

这两层不能互相替代：

- 只有权限，没有沙箱：一旦误判，破坏是真实的
- 只有沙箱，没有权限：用户无法理解和治理 Agent 的意图
- 权限 + 沙箱：既能做决策，又能限制后果

如果你设计自己的 Agent，不要只写一个 `confirm()`。要把安全拆成至少三层：

```text
意图层：模型想做什么？
审批层：这个动作是否允许？
执行层：即使允许，它能影响哪些资源？
```

这才是工程系统里的安全。

延伸阅读：

- Claude Code：[[../concepts/claude-code/权限模型与审批流程|权限模型与审批流程]]、[[../concepts/claude-code/沙箱隔离机制|沙箱隔离机制]]
- DeerFlow：[[../entities/deerflow-book/沙箱|沙箱]]、[[../concepts/deerflow-book/虚拟路径|虚拟路径]]、[[../concepts/deerflow-book/延迟初始化|延迟初始化]]

---

## 设计思想五：长任务可靠性来自显式状态，而不是更长 Prompt

Agent 做短任务时，靠上下文窗口就够了。做长任务时，真正的问题不是模型不知道，而是系统没有显式保存关键状态。

Claude Code 在多处体现了这个思想：

- QueryEngine 保存会话级消息、权限拒绝、文件读取缓存、token 使用量
- 工具调用必须保持 `tool_use -> tool_result` 的协议完整性
- 压缩时要保护 tool pair，不能把 tool_result 留下却删掉对应 tool_use
- 权限拒绝会被记录，避免模型反复尝试同一个被拒动作
- Todo 和任务状态帮助模型从“聊天”转为“推进任务”

DeerFlow 的 ThreadState 更明显：messages、artifacts、viewed_images、todos、sandbox、task_count 都被放进状态容器，并通过 Checkpointer 跨轮次持久化。即使进程重启，只要 session_id 不变，Agent 就能恢复。

DeerFlow 的 DanglingToolCallMiddleware 也很有代表性：当用户中断对话，历史里可能留下“AI 发起了工具调用但没有 ToolMessage”的悬空状态。中间件会自动修复，避免下次模型 API 因消息格式不合法而失败。

这背后的思想是：

> 可恢复系统不要只保存对话文本，要保存足以恢复协议状态的账本。

你自己写学习笔记时可以记住一句话：

> Memory 不是聊天记录，State 不是变量堆；它们共同构成 Agent 行动协议的账本。

延伸阅读：

- Claude Code：[[../entities/claude-code/queryengine对话引擎|queryengine对话引擎]]、[[../concepts/claude-code/agentic对话循环机制|agentic对话循环机制]]、[[../concepts/claude-code/上下文压缩策略|上下文压缩策略]]
- DeerFlow：[[../entities/deerflow-book/LeadAgent大脑|LeadAgent大脑]] 的 ThreadState、[[../entities/deerflow-book/中间件管道|中间件管道]] 的 DanglingToolCallMiddleware / TodoMiddleware

---

## 设计思想六：上下文压缩不是删除历史，而是分层存储

Claude Code 的上下文压缩很值得学，因为它不是简单“摘要旧消息”。

它至少有三层：

| 层级 | 思想 | 代价 |
|------|------|------|
| MicroCompact | 清理旧工具结果、图片、文档等可再获取内容 | 低，不需要模型 |
| Session Memory Compact | 用已提取的 session memory 替代完整历史 | 中，不需要摘要模型 |
| API 摘要压缩 | 调用模型生成摘要 | 高，需要额外模型调用 |

这里最关键的不是压缩算法，而是信息分类：

- 文件内容可以重新读，所以旧 Read 结果可以清
- 搜索结果可以重新查，所以旧 Grep/Glob/WebSearch 结果可以清
- 工具协议不能破坏，所以 tool_use/tool_result 必须成对保留
- 最近对话要保留 tail，因为它包含当前任务的局部语境
- 长期事实应该进入 memory，而不是挤在 prompt 里

DeerFlow 也体现了分层思想：SummarizationMiddleware 压缩消息历史，TodoMiddleware 在摘要后重新注入任务列表，MemoryMiddleware 异步提取长期记忆，memory.json 则保存用户画像、当前焦点和事实库。

这背后的思想是：

> 上下文工程不是把所有东西塞给模型，而是把不同生命周期的信息放到不同层。

一个成熟 Agent 至少应该区分：

| 信息类型 | 应该放哪里 |
|----------|------------|
| 当前任务最近几步 | 工作上下文 |
| 已完成工具结果 | 可压缩历史或外部文件 |
| 用户稳定偏好 | 长期记忆 |
| 当前任务列表 | 显式状态 |
| 系统规则和能力说明 | System Prompt / Skill |
| 可重复获取的信息 | 引用或占位符 |

当你看到“上下文压缩”时，不要只想到摘要。真正的设计问题是：这条信息未来还会以什么方式被使用？

延伸阅读：

- Claude Code：[[../concepts/claude-code/上下文压缩策略|上下文压缩策略]]
- DeerFlow：[[../entities/deerflow-book/记忆系统|记忆系统]]、[[../concepts/deerflow-book/记忆三层架构|记忆三层架构]]、[[../entities/deerflow-book/中间件管道|中间件管道]]

---

## 设计思想七：扩展能力最好挂在生命周期上

DeerFlow 的中间件管道是它最值得学习的设计之一。11 层中间件不是随便堆功能，而是把 Agent 生命周期切成明确时机：

- `before_agent`：注入 thread、uploads、sandbox 等运行前信息
- `before_model`：在模型调用前修正或补充上下文
- `after_model`：在工具执行前检查模型输出，比如限制 Sub-agent 数量
- `after_agent`：在执行完成后做摘要、标题、记忆、图片管理、澄清处理

这使得很多横切能力不必侵入 Lead Agent 主循环：

- 上传文件处理
- 沙箱初始化与释放
- 悬空工具调用修复
- Todo 恢复
- Sub-agent 并发限制
- 长期记忆捕获
- 会话标题生成
- 澄清流程处理

Claude Code 也有类似思想：工具权限检查、PreToolUse hooks、Skills、MCP、权限模式都在不同生命周期点介入系统。

这背后的思想是：

> Agent 主循环应该保持稳定，复杂能力应该挂到明确的生命周期点上。

如果没有生命周期扩展点，系统会很快变成这样：

```text
主循环里塞满：
if 有上传文件...
if 要记忆...
if 要压缩...
if 是图片...
if 是 subagent...
if 要审批...
if 要恢复...
```

中间件和 Hook 的价值，就是把这些能力从主循环中抽离出来，让它们变成可组合的治理层。

延伸阅读：

- Claude Code：[[../concepts/claude-code/工具架构与注册机制|工具架构与注册机制]]、[[../entities/claude-code/Skills技能扩展|Skills技能扩展]]、[[../concepts/claude-code/权限模型与审批流程|权限模型与审批流程]]
- DeerFlow：[[../entities/deerflow-book/中间件管道|中间件管道]]、[[../entities/deerflow-book/LeadAgent大脑|LeadAgent大脑]]

---

## 设计思想八：Skill 是“工作流封装”，不是小工具

Claude Code 和 DeerFlow 都有 Skills，这说明 Skills 已经成为 Agent 系统的重要抽象。

但 Skill 和 Tool 的区别必须分清：

| 抽象 | 本质 | 例子 |
|------|------|------|
| Tool | 一个可调用动作 | read_file、bash、web_search |
| Skill | 一套完成任务的方法 | code review、deep research、data analysis |

Claude Code 的 Skills 体现了“Prompt 即能力”：一个 code review skill 不需要自己实现审查引擎，它把审查顺序、关注点、输出格式、工具约束封装成 Markdown，让模型按专业流程行动。

DeerFlow 的 Skills 更强调渐进式加载：

```text
第一层：name + description + location
第二层：按需读取 SKILL.md 正文
第三层：按需读取 scripts / references / assets
```

这解决了一个关键问题：如果系统有 17 个内置 Skill，不能把所有操作手册都塞进上下文。模型只需要先知道“有哪些能力”，等确定要用时再加载细节。

这背后的思想是：

> 复杂能力应该被封装成可触发、可阅读、可携带资源的工作流，而不是全部硬编码进系统提示。

一个好的 Skill 至少要回答：

- 什么时候应该用它？
- 使用它时按什么步骤走？
- 能用哪些工具？
- 有哪些参考资料或脚本？
- 输出应该是什么形态？
- 是否需要隔离执行？

这也是你学习 agent 项目时最值得留意的地方：项目把“经验”放在哪里？是散落在 prompt 里，还是被组织成可复用 Skill？

延伸阅读：

- Claude Code：[[../entities/claude-code/Skills技能扩展|Skills技能扩展]]
- DeerFlow：[[../entities/deerflow-book/Skills系统|Skills系统]]、[[../concepts/deerflow-book/渐进式加载|渐进式加载]]

---

## 设计思想九：Sub-agent 不是越多越好，而是并行性的受控表达

Claude Code 和 DeerFlow 都支持子 Agent，但它们都没有把 Sub-agent 当成魔法并行。

Claude Code 的协调器模式非常强调职责边界：Coordinator 不直接做大量文件操作，而是负责分解任务、启动 Worker、接收 XML 通知、综合结果。Worker 的提示必须自包含，因为它看不到协调器和用户的完整上下文。

它还强调一个很重要的原则：

> 并行性是超能力，但只适用于真正独立的任务。

Claude Code 还通过 worktree、remote environment 等方式隔离 Worker，避免多个 Agent 同时写同一份代码导致冲突。

DeerFlow 的 Sub-agent 更像 harness 里的轻量执行单元。Lead Agent 通过 `task` 工具创建 Sub-agent；SubagentExecutor 管理状态机；每个 Sub-agent 有独立消息历史；SubagentLimitMiddleware 限制一次最多触发 3 个任务，防止资源爆炸。

这背后的思想是：

> Sub-agent 的核心不是“多几个模型”，而是把可并行的问题变成有边界、有状态、有结果协议的子任务。

什么时候适合 Sub-agent？

- 多角度研究
- 多模块代码理解
- 多竞品比较
- 多数据源调查
- 验证与实现可以独立进行

什么时候不适合？

- 用户意图还不清楚
- 任务有强顺序依赖
- 一次工具调用就能完成
- 多个 Agent 会写同一批文件

Sub-agent 的设计重点是任务边界，而不是数量。

延伸阅读：

- Claude Code：[[../entities/claude-code/Sub-Agent机制|Sub-Agent机制]]
- DeerFlow：[[../entities/deerflow-book/子智能体|子智能体]]、[[../concepts/deerflow-book/上下文隔离|上下文隔离]]、[[../concepts/deerflow-book/长时程Agent|长时程Agent]]

---

## 设计思想十：记忆不是越多越好，而是有结构地变成系统资产

DeerFlow 的记忆系统特别适合学习，因为它没有把 memory 简化成“把历史聊天存起来”。

它把记忆拆成三类：

- user：用户长期背景、沟通风格
- topOfMind：用户当前关注的项目和任务
- facts：可追踪来源和置信度的事实

同时用 MemoryMiddleware 捕获对话，用 MemoryUpdateQueue 做 30 秒防抖，用 MemoryUpdater 调 LLM 抽取并合并，用 confidence_threshold 过滤噪声，再通过原子写入保存到 memory.json。

这个设计的成熟点在于：

- 记忆更新是异步的，不阻塞当前对话
- 记忆有置信度，不是所有内容都进入长期状态
- 记忆有来源，未来可以追溯
- 记忆有层次，不把偏好、当前项目、事实混在一起
- 记忆注入是格式化的，而不是原样塞回 prompt

Claude Code 也有项目内存、Session Memory、上下文压缩等机制。它们共同说明：

> 记忆系统的目标不是保存更多文本，而是让未来的 Agent 行动拥有更好的先验。

如果你未来设计自己的 agent 记忆，应该先区分：

```text
这条信息是用户偏好？
是项目事实？
是当前任务状态？
是工具执行结果？
是可以重新查询的材料？
是一次性的中间过程？
```

不同答案应该进入不同存储层。

延伸阅读：

- Claude Code：[[../concepts/claude-code/上下文压缩策略|上下文压缩策略]]、[[../entities/claude-code/queryengine对话引擎|queryengine对话引擎]]
- DeerFlow：[[../entities/deerflow-book/记忆系统|记忆系统]]、[[../concepts/deerflow-book/记忆三层架构|记忆三层架构]]、[[../concepts/deerflow-book/置信度阈值|置信度阈值]]

---

## 设计思想十一：生产化 Agent 需要 Harness，而不是只需要 Framework

DeerFlow 反复体现一个核心观点：Agent 能力不是只来自模型和框架，而是来自 harness。

Framework 解决“怎么搭一个 Agent”；Harness 解决“怎么让 Agent 在真实任务中持续可靠地行动”。

DeerFlow 的 harness 包括：

- Gateway：统一外部入口
- Lead Agent：核心决策循环
- Middleware：生命周期治理
- Sandbox：执行环境
- Memory：长期状态
- Skills：能力扩展
- Sub-agent：并行执行
- Config：模型、工具、部署差异
- SSE：可观测的执行过程

Claude Code 作为 terminal-native agent，也有自己的 harness：

- REPL / CLI：交互入口
- QueryEngine：会话管理
- Tool Registry：能力边界
- Permission：审批系统
- Sandbox：执行隔离
- Worktree：开发隔离
- Context Compact：长会话维持
- Transcript：恢复和审计
- Skills / MCP：能力扩展

这背后的思想是：

> 模型是大脑，但 Harness 才是让大脑可靠行动的身体、环境和制度。

只学模型调用 API，会停留在 demo。学 Harness，才会理解为什么真实 agent 项目有那么多看似“不性感”的模块：权限、日志、压缩、恢复、沙箱、状态、配置、清理、观测。

这些模块不是外围，它们就是 Agent 从玩具变成系统的关键。

延伸阅读：

- Claude Code：[[架构全景分析]]、[[../concepts/claude-code/agentic对话循环机制|agentic对话循环机制]]、[[../concepts/claude-code/沙箱隔离机制|沙箱隔离机制]]、[[../entities/claude-code/Skills技能扩展|Skills技能扩展]]
- DeerFlow：[[../concepts/deerflow-book/Harness与Framework|Harness与Framework]]、[[../entities/deerflow-book/LeadAgent大脑|LeadAgent大脑]]、[[../entities/deerflow-book/中间件管道|中间件管道]]

---

## 设计思想十二：好架构会把隐藏的不变量变成命名组件

Claude Code 和 DeerFlow 都有很多看起来很工程化的小组件，比如：

- pending tool
- dangling tool call
- task notification
- ThreadState
- PermissionResult
- ToolFilterMiddleware
- SubagentLimitMiddleware
- SummarizationMiddleware
- SandboxProvider
- Session Memory Compact

这些名字背后都有一个共同点：它们把原本隐含在代码里的不变量显式化了。

比如：

- `dangling tool call` 表示消息历史里有未闭合的工具协议
- `SubagentLimitMiddleware` 表示模型不能无限生成并发任务
- `PermissionResult` 表示执行动作前必须有明确裁决
- `ThreadState` 表示跨轮次状态不是临时变量
- `SandboxProvider` 表示执行环境的创建、复用、释放需要统一抽象

这背后的思想是：

> 当某个错误模式反复出现，就应该把它命名，并让系统结构承认它。

这也是读开源项目最应该训练的能力。不要只看代码“怎么写”，要问：

- 这个组件在防止哪类失败？
- 它把什么隐含约束变成了显式对象？
- 如果没有它，系统会在哪些场景下崩？

能回答这些问题，才是真的读到了设计。

延伸阅读：

- Claude Code：[[../concepts/claude-code/流式响应与事件处理|流式响应与事件处理]]、[[../concepts/claude-code/上下文压缩策略|上下文压缩策略]]、[[../entities/claude-code/Sub-Agent机制|Sub-Agent机制]]
- DeerFlow：[[../entities/deerflow-book/中间件管道|中间件管道]]、[[../entities/deerflow-book/子智能体|子智能体]]、[[../entities/deerflow-book/沙箱|沙箱]]

---

## 对比总结：两个项目分别教什么

| 设计问题       | Claude Code 的答案                                    | DeerFlow 的答案                                             | 可迁移原则            |
| ---------- | -------------------------------------------------- | -------------------------------------------------------- | ---------------- |
| Agent 怎么行动 | QueryEngine + query loop + StreamingToolExecutor   | Lead Agent + LangGraph + ThreadState                     | 主循环要状态机化         |
| 工具怎么组织     | Tool 是 schema、权限、渲染、并发语义的复合体                       | 多层工具 + ToolFilterMiddleware                              | 工具是能力协议          |
| 怎么保证安全     | Allow/Ask/Deny + OS sandbox + denial tracking      | Sandbox 抽象 + 虚拟路径 + 多实现                                  | 权限和隔离分层          |
| 怎么维持长任务    | transcript、context compact、tool pair invariant     | Checkpointer、ThreadState、Todo、DanglingToolCallMiddleware | 保存协议状态，而非只保存文本   |
| 怎么压缩上下文    | MicroCompact、Session Memory、API 摘要                 | Summarization + Todo 恢复 + Memory                         | 按生命周期分层存储        |
| 怎么扩展能力     | Skills、MCP、Hooks、Worktree、Sub-agent                | Skills、Middleware、MCP、Sub-agent                          | 扩展点要挂在生命周期       |
| 怎么并行       | Coordinator / Worker / XML notification / Worktree | task tool / SubagentExecutor / limit middleware          | 并行要有边界和结果协议      |
| 怎么生产化      | 本地终端产品化 + 权限治理 + 恢复                                | Gateway + harness + sandbox + memory + config            | Agent 需要 harness |

---

## 如果要设计一个迷你版 Agent，应该先有这些组件

读完 Claude Code 和 DeerFlow 后，一个迷你 Agent 的核心不应该只有“LLM + tools”。至少应该有：

```text
UserInput
  -> TurnLoop
  -> MessageLedger
  -> ContextBuilder
  -> ModelClient
  -> EventStream
  -> ToolRegistry
  -> PermissionGate
  -> SandboxRunner
  -> ToolResultLedger
  -> ContinueOrStopPolicy
  -> Compactor
  -> MemoryWriter
```

每个组件负责一个明确问题：

| 组件 | 负责的问题 |
|------|------------|
| TurnLoop | 一轮 Agent 行动如何推进和停止 |
| MessageLedger | 对话、工具调用、工具结果如何形成可恢复账本 |
| ContextBuilder | 当前应该给模型哪些信息 |
| EventStream | 执行过程如何被观察 |
| ToolRegistry | Agent 有哪些动作能力 |
| PermissionGate | 动作执行前如何裁决 |
| SandboxRunner | 动作执行时如何限制影响范围 |
| ToolResultLedger | 工具结果如何写回协议 |
| Compactor | 上下文压力过大时如何分层压缩 |
| MemoryWriter | 哪些信息值得进入长期记忆 |

这比“我调用模型，然后解析 tool_call”复杂，但这正是 Agent 工程和 demo 的分界线。

---

## 最值得带走的句子

1. Agent 不是聊天流，而是可恢复的行动循环。
2. 工具不是函数，而是 Agent 与世界交互的协议边界。
3. 流式输出不是 UI 技巧，而是执行过程的事件化表达。
4. 权限决定该不该做，沙箱限制做了以后能影响什么。
5. 长任务可靠性来自显式状态，而不是更长 Prompt。
6. 上下文压缩不是删除历史，而是按生命周期分层存储。
7. 主循环应该稳定，复杂能力应该挂到生命周期扩展点上。
8. Skill 是工作流封装，不是小工具。
9. Sub-agent 的核心是任务边界和结果协议，不是模型数量。
10. 记忆不是保存更多文本，而是让未来行动拥有更好的先验。
11. 模型是大脑，Harness 是让大脑可靠行动的身体、环境和制度。
12. 好架构会把隐藏的不变量变成命名组件。

---

## 后续可继续扩展的问题

这篇文档只对 Claude Code 和 DeerFlow 做综合精读。后续可以把其他项目继续接入同一张设计地图：

- AgentScope Java：生命周期 Hook、pending tool、结构化输出协议
- Hermes Agent：自我改进循环、技能自创建、多平台接入
- OpenClaw：Skills 加载、Sub-agent 实战、Gateway 配置
- OpenAI Agents SDK：轻量框架如何表达 handoff、guardrail、tracing

每加入一个项目，都不只是补“项目介绍”，而是回答：

> 它在 Agent 公共问题上，给出了什么不同的设计答案？

### 一. 上下文工程不是把所有东西塞给模型，而是把不同生命周期的信息放到不同层。

这句话的意思是：

**上下文工程不是“模型这次回答需要什么，我就全塞进 prompt”，而是先判断每类信息的生命周期，再决定它应该放在哪个存储层、什么时候取出来给模型。**

这里的“不同层”，可以理解成 Agent 系统里的不同信息位置。

比如一个长任务 Agent 里，信息至少有这些层：

| 层 | 放什么 | 生命周期 |
|---|---|---|
| **当前 Prompt 层** | 这一步马上要用的信息 | 几秒到一轮对话 |
| **工作记忆层** | 当前任务目标、计划、todo、最近几轮上下文 | 一个任务周期 |
| **工具协议账本层** | tool_use、tool_result、pending tool、执行状态 | 直到工具链闭合 |
| **外部文件 / artifact 层** | 大文件、代码、搜索结果、分析产物 | 可长期保存，可按需读取 |
| **摘要层** | 旧对话的压缩版、阶段性结论 | 跨多轮任务 |
| **长期记忆层** | 用户偏好、稳定事实、长期项目背景 | 跨会话 |
| **索引层** | 哪些资料在哪里、该去哪里找 | 长期导航 |

所以“不同层”的核心不是物理目录，而是：**不同信息有不同的使用频率、有效期、成本和风险。**

举个例子。

假设 Agent 读了一个 5000 行源码文件。错误做法是：

```text
把整个文件一直塞在 prompt 里。
```

这会浪费上下文，而且后面很多内容未必还用得上。

更好的做法是分层：

```text
当前 Prompt 层：
- 只放这一步要修改的函数片段

工作记忆层：
- 记录“当前在修复登录态过期 bug”

工具协议账本层：
- 记录刚才 Read 了哪个文件、Edit 了哪里、测试有没有跑

外部文件层：
- 完整源码仍然在磁盘，需要时重新 Read

摘要层：
- 记录“auth 模块的 session 校验逻辑在 validate.ts”

长期记忆层：
- 如果这是用户长期项目，记录“用户正在维护 auth 系统”
```

这样模型每一轮看到的是“刚好够用”的上下文，而不是所有历史。

Claude Code 的上下文压缩就是这个思想：旧的 `Read`、`Grep`、`Bash` 结果可以清掉，因为它们能重新获取；但 `tool_use -> tool_result` 这种协议关系不能随便删，因为删坏了 Agent 就无法继续。

DeerFlow 也是类似：`ThreadState` 存当前任务状态，`TodoMiddleware` 负责恢复任务列表，`MemoryMiddleware` 把长期信息写进 memory，上传文件和沙箱文件则在外部层保存。

所以这句话最简单的理解是：

> 不要把 prompt 当仓库。  
> Prompt 只是当前工作台。  
> 文件、记忆、摘要、任务状态、工具结果账本，才是完整的信息系统。

真正的上下文工程，是设计“什么信息什么时候进入模型，什么时候离开模型，离开后放在哪里，需要时怎么找回来”。

### 二. 它们把原本隐含在代码里的不变量显式化了

**我的理解是：** 原本一些口口相传或者默契约定的规则，在这两个项目中被设计成了具体的组件，不需要人特定的去记住他，而是当作组件来维护，特定场景的agent有不同的面临的问题，可以每个场景根据情况来设计特定的组件，不在口口相传

“隐含的不变量”就是：**系统必须一直遵守的规则，但这个规则没有被单独命名，也没有被单独建模，只是散落在代码逻辑里，靠程序员记住。**

“显式化”就是：**把这个规则变成一个有名字的概念、组件、字段、状态机或检查器，让系统自己维护它。**

举个最典型的 Agent 例子：工具调用必须闭合。

模型发起工具调用后，消息历史里必须出现对应工具结果：

```text
assistant: tool_use(id="call_1", name="read_file")
tool:      tool_result(tool_use_id="call_1", content="...")
```

这个规则就是一个“不变量”：

> 每个 `tool_result` 必须对应一个存在的 `tool_use`；每个未完成的 `tool_use` 不能被当作已经完成。

如果代码里只是到处写：

```text
执行前检查一下有没有 tool result
压缩前注意别删 tool_use
恢复时看看有没有缺失消息
```

那这个规则就是**隐含的**。它存在，但没有一个清晰名字。新人读代码时不一定知道这是系统级规则，只会看到很多零散判断。

但如果系统把它命名成：

```text
pending tool
dangling tool call
ToolResultLedger
DanglingToolCallMiddleware
adjustIndexToPreserveAPIInvariants()
```

那它就被**显式化**了。

意思是：系统承认“未闭合工具调用”是一类真实状态，并专门设计组件处理它。

再举几个例子。

**例子一：Sub-agent 不能无限创建**

隐含写法：

```text
如果模型一次生成太多 task，就只执行几个吧。
```

这个规则如果散在工具执行代码里，就是隐含的。

显式化之后：

```text
SubagentLimitMiddleware
max_concurrent = 3
```

它变成一个明确组件：专门负责限制 Sub-agent 并发。

**例子二：Agent 当前任务列表不能因为摘要丢失**

隐含规则：

> 即使旧消息被压缩，Agent 也不能忘记当前 todo。

如果只是靠 prompt 里提醒“请记住任务”，这是隐含且脆弱的。

显式化之后：

```text
ThreadState.todos
TodoMiddleware
```

任务列表被单独保存；摘要后再重新注入上下文。系统不靠模型记忆，而靠结构维护。

所以这句话可以翻译成更直白的话：

> 好架构会把“大家心里知道必须成立的规则”，变成代码里有名字、有结构、有入口、有测试可能性的东西。

为什么这很重要？

因为隐含规则靠人记，容易漏；显式规则靠系统维护，容易复用、检查和调试。

你以后读 Agent 项目时，可以专门问一句：

> 这个组件是不是在给某个失败模式起名字？

比如：

- `pending tool`：给“工具调用还没还债”起名字
- `DanglingToolCallMiddleware`：给“工具调用断链”起名字
- `ThreadState`：给“跨轮次状态不能靠局部变量”起名字
- `SandboxProvider`：给“执行环境有生命周期”起名字
- `SubagentLimitMiddleware`：给“模型会过度并发”起名字

这就是“把隐含的不变量显式化”。

### 三. 在这两个项目当中，有没有涉及到主agent和子agent的情况？如果有的话，主agent是如何做意图识别和规划的？

有，而且这两个项目都把“主 Agent / 子 Agent”作为重要能力。但它们给出的设计答案不一样。

**先说结论**

| 项目          | 主 Agent                 | 子 Agent      | 核心设计答案                                |
| ----------- | ----------------------- | ------------ | ------------------------------------- |
| Claude Code | Coordinator / 主会话 Agent | Worker Agent | 主 Agent 专注“分解、调度、综合”，Worker 做研究/实现/验证 |
| DeerFlow    | Lead Agent              | Sub-agent    | Lead Agent 是统一大脑，用 `task` 工具动态委派子任务   |

更重要的是：**它们都没有把“意图识别”做成一个单独的传统分类器，**  它们更像是：

```text
System Prompt 给角色和规则
工具列表给可行动作
模型在 ReAct 循环中判断用户意图
如果需要并行/委派，就调用 Agent/task 工具
中间件/限制器兜底，防止模型乱调度
```

---

**Claude Code：Coordinator / Worker 模式**

Claude Code 里主 Agent 可以进入 Coordinator Mode。这个模式下，主 Agent 不主要负责亲自读写文件，而是负责：

```text
理解用户目标
  -> 判断任务是否复杂
  -> 拆成研究 / 实现 / 验证等子任务
  -> 用 Agent 工具启动 Worker
  -> 接收 Worker 的 task-notification
  -> 综合结果
  -> 必要时继续某个 Worker 或停止 Worker
  -> 回复用户
```

结构大概是：

```text
User
  ↓
Coordinator Agent
  ├── Worker 1：研究代码结构
  ├── Worker 2：实现某个改动
  ├── Worker 3：独立验证测试
  ↓
task-notification XML
  ↓
Coordinator 综合
  ↓
User
```

它的意图识别主要靠 Coordinator 的系统提示。提示会告诉主 Agent：

- 你是协调者，不是普通执行者
- 能直接回答的问题直接答
- 复杂任务要分解
- 独立任务要并行
- Worker prompt 必须自包含
- 不要让一个 Worker 检查另一个 Worker
- 研究完成后，要由 Coordinator 自己综合理解

它的规划流程很像：

```text
1. 识别任务类型
   简单问答？直接回答
   复杂代码任务？进入分解
   需要计划？可用 Plan Agent
   需要研究/实现/验证？启动 Worker

2. 拆任务
   哪些可以并行？
   哪些有顺序依赖？
   哪些需要隔离 worktree？
   哪些 Worker 可以继续复用？

3. 委派
   用 Agent Tool 创建 Worker
   prompt 必须包含背景、目标、文件范围、完成标准

4. 汇报
   Worker 用 XML task-notification 返回结果

5. 综合
   Coordinator 汇总发现，决定继续、修正、停止或回答用户
```

Claude Code 的重点是：**主 Agent 像项目经理/协调器，子 Agent 像独立执行者。**

对应文档可以看：[Sub-Agent机制.md](/Users/machengqian.1/Documents/研究/wiki/entities/claude-code/Sub-Agent机制.md)

---

**DeerFlow：Lead Agent / Sub-agent 模式**

DeerFlow 更明确地叫 Lead Agent。Lead Agent 是整个系统的大脑，Sub-agent 是通过 `task` 工具动态创建的执行单元。

结构是：

```text
User
  ↓
Gateway API
  ↓
Lead Agent
  ↓
中间件管道
  ↓
模型推理
  ├── 普通工具调用：bash/read_file/write_file/web_search
  ├── 澄清工具：ask_clarification
  └── 子任务委派：task
        ↓
      SubagentExecutor
        ↓
      Sub-agent ReAct Loop
        ↓
      result 返回 Lead Agent
  ↓
Lead Agent 综合
  ↓
User / 文件产物
```

DeerFlow 的主 Agent 如何做意图识别？

它主要靠 System Prompt 里的几类指令：

1. **先澄清**
   如果用户需求不清楚、缺信息、多种解释，Lead Agent 必须先问清楚，不要直接开干。

2. **先规划**
   对复杂任务，要先思考任务结构，而不是直接调用工具。

3. **Skill First**
   复杂任务先看是否有相关 Skill，可以把专业工作流加载进来。

4. **Sub-agent 编排**
   如果任务可以拆成多个独立子任务，就启动 Sub-agent。

DeerFlow 的 `<subagent_system>` 里有很清楚的三步：

```text
DECOMPOSE：把复杂任务拆成并行子任务
DELEGATE：同时启动多个 subagents
SYNTHESIZE：收集并整合结果
```

同时它明确告诉模型什么时候用 Sub-agent：

适合：

- 复杂研究问题
- 多方面分析
- 大型代码库理解
- 综合调查

不适合：

- 任务不可分解
- 极简单动作
- 需要立即澄清
- 存在顺序依赖

DeerFlow 的规划流程大概是：

```text
1. Lead Agent 接收用户请求

2. 判断是否清楚
   不清楚 -> ask_clarification
   清楚 -> 继续

3. 判断是否复杂
   简单 -> 自己调用工具完成
   复杂 -> 规划

4. 判断能否并行拆解
   不能并行 -> 顺序执行
   可以并行 -> 统计子任务数量

5. 根据 max_concurrent_subagents 分批
   例如最多 3 个：
   第一批 1-3
   第二批 4-6
   最后综合

6. 调用 task 工具
   task(description, prompt, subagent_type)

7. SubagentExecutor 执行
   PENDING -> RUNNING -> COMPLETED / FAILED / TIMEOUT

8. Lead Agent 收到结果

9. 综合所有结果，生成最终回答或产物
```

对应文档可以看：

- [LeadAgent大脑.md](/Users/machengqian.1/Documents/研究/wiki/entities/deerflow-book/LeadAgent大脑.md)
- [子智能体.md](/Users/machengqian.1/Documents/研究/wiki/entities/deerflow-book/子智能体.md)
- [中间件管道.md](/Users/machengqian.1/Documents/研究/wiki/entities/deerflow-book/中间件管道.md)

---

**两个项目的关键差异**

Claude Code 的答案是：

> 主 Agent 变成 Coordinator，重点是分工、隔离、通知协议、综合判断。

DeerFlow 的答案是：

> 保留一个统一 Lead Agent，把规划、澄清、技能加载、工具调用、Sub-agent 编排都放进一个 harness 化的大脑里。

更抽象地说：

```text
Claude Code：
主 Agent = 协调器
子 Agent = 独立 Worker
重点 = 并行软件工程任务、worktree/remote 隔离、结构化通知

DeerFlow：
主 Agent = Lead Agent
子 Agent = task 工具启动的临时执行单元
重点 = long-horizon harness、动态拆解、批次调度、上下文隔离
```

**最值得你抓住的设计思想**

主 Agent 的“意图识别”不是简单分类：

```text
用户说了什么 -> 分类成 A/B/C
```

而是一个动态决策过程：

```text
用户目标是什么？
是否清楚？
是否复杂？
是否需要 Skill？
是否可拆分？
是否可并行？
是否需要子 Agent？
每个子任务的完成标准是什么？
结果回来后如何综合？
```

所以主 Agent 真正做的是：

> 从用户意图中识别“任务结构”。

这句话很关键。

普通聊天 Agent 识别的是“你想问什么”。  
主/子 Agent 架构里的主 Agent 识别的是：

```text
这个任务应该如何被组织成行动系统？
```

这就是它和普通意图分类最大的区别。

### 四. 什么叫做协议不变量优先于 token 目标？

所谓协议不变量就是：协议规定的绝对不能改变的，破坏了约定的协议就会影响agent的反应

这句话的意思是：

**压缩上下文时，省 token 不是第一目标。第一目标是：压缩后的消息序列仍然必须是一个“合法的 Agent 对话”。**

所谓“协议不变量”，就是无论你怎么压缩，都不能破坏的结构规则。

举个最典型的例子：

```text
assistant: 我要调用 Read 工具，tool_use_id = 123
user/tool_result: 这是 Read 工具返回的结果，tool_use_id = 123
assistant: 根据读取结果，我发现...
```

这里有一个不变量：

```text
tool_result 必须能找到对应的 tool_use
```

如果为了省 token，把前面的 `tool_use` 删了，只留下：

```text
user/tool_result: 这是 Read 工具返回的结果，tool_use_id = 123
assistant: 根据读取结果，我发现...
```

token 是省了，但这个对话对 API 来说可能已经不合法了。因为它看到一个工具结果，却找不到是谁发起了这个工具调用。

所以 Claude Code 的设计不是：

> 我要保留最近 10k token，超过就硬切。

而是：

> 我先找一个大概能保留 10k token 的切分点，但如果这个切分点会切坏工具调用对、流式 assistant response、compact boundary，那我宁愿多保留一些 token，也要把结构补完整。

这就是“协议不变量优先于 token 目标”。

再换成人话：

**token 目标是预算，不变量是底线。预算可以超一点，底线不能破。**

在 Agent 系统里，这个思想特别重要。因为 Agent 的上下文不是普通聊天记录，而是一种带协议的执行日志，里面有：

- assistant 发起的工具调用
- tool_result 返回结果
- streaming 拆出来的 assistant 消息块
- compact boundary
- 子 agent / 主 agent 的状态边界
- 文件、计划、skills、MCP 等运行状态

这些结构一旦被压缩算法切坏，模型可能不是“少知道一点”，而是直接进入不合法状态，或者产生很奇怪的行为。

所以这句话可以理解为：

> Claude Code 在压缩时，不是把上下文当成一长串文本来裁剪，而是把它当成一份有结构、有协议、有执行关系的运行记录。压缩可以牺牲一点 token 效率，但不能牺牲协议正确性。

### 五. 在 claudecode 中什么是 reactive compact？

在 Claude Code 里，**reactive compact** 可以理解为：

> **不是提前预测“快满了”就压缩，而是等真实 API 请求已经失败，比如 prompt too long / 413，再立刻压缩上下文并重试当前请求。**

也就是“反应式压缩”。

它和普通 auto compact 的区别是：

```text
auto compact:
请求前看 token 快超了 -> 主动压缩 -> 再请求模型

reactive compact:
先请求模型 -> API 真的报 prompt too long -> 拦住错误 -> 压缩 -> 用压缩后的上下文重试
```

在源码里入口主要在 [query.ts](/Users/machengqian.1/code/typescriptProject/claude-code-main/src/query.ts:1119)。

流程大概是：

```text
1. query 正常向模型发请求
2. API 返回 prompt-too-long / media-size 这类可恢复错误
3. Claude Code 先不把这个错误展示给用户，而是 withheld
4. 如果 context collapse 开着，先尝试 drain collapse
5. 如果还不行，调用 reactiveCompact.tryReactiveCompact(...)
6. compact 成功后，buildPostCompactMessages(...)
7. 把状态设为 reactive_compact_retry
8. 用压缩后的 messages 重新进入 query 循环
```

对应这段逻辑：

```ts
const compacted = await reactiveCompact.tryReactiveCompact(...)

if (compacted) {
  const postCompactMessages = buildPostCompactMessages(compacted)
  ...
  state = {
    messages: postCompactMessages,
    hasAttemptedReactiveCompact: true,
    transition: { reason: 'reactive_compact_retry' },
  }
  continue
}
```

这里的关键词是 **retry 当前请求**。它不是说“压缩完，下一次用户输入再好用”，而是这一次已经失败的模型调用，会立刻用压缩后的上下文再试一次。

为什么需要它？

因为 token 估算不一定准。Claude Code 请求前可能觉得还没爆，但真实 API 侧算出来超了。尤其是：

- 工具结果很大
- 图片 / 文档内容很大
- prompt cache / context management 后的真实 token 和本地估算有差异
- context collapse、snip、microcompact 多层机制叠加后，本地估算可能滞后

所以 reactive compact 是最后一道保险：

> 你以为还能发，结果 API 说不行，那就现场压缩并重试。

它还有几个很工程化的保护点：

1. **错误先 withheld，不立刻展示给用户**  
   因为系统还有机会恢复。如果恢复成功，用户甚至不需要看到这次 prompt-too-long。

2. **只尝试一次，防止死循环**  
   `hasAttemptedReactiveCompact: true` 会防止反复 reactive compact。否则可能出现：请求超限 -> 压缩 -> 重试 -> 又超限 -> 再压缩 -> 无限循环。

3. **失败后不继续跑 stop hooks**  
   源码注释里说，如果模型根本没有产生有效 response，stop hooks 没有评估对象。继续跑 hook 还可能注入更多 token，让错误恢复链路越修越坏。

4. **和 context collapse 协作**  
   如果 context collapse 开启，Claude Code 会先 drain 已经 staged 的 collapse，因为这更便宜，也更能保留细粒度上下文；不行再 reactive compact。

一句话总结：

**reactive compact 是 Claude Code 的“请求失败后的现场抢救机制”：当真实 API 返回上下文过长时，它拦住错误、压缩上下文、重建 post-compact messages，然后自动重试当前请求。**

它的设计思想是：  **不要只依赖预测式 token 阈值，Agent 系统还要能对真实运行时失败做恢复。**

### 六. claudecode是怎么实现文件只读的权限的？

Claude Code 里的“文件只读权限”不是靠一个简单的 `readonly=true` 开关实现的，而是靠三层组合出来的：

1. **工具分层**
   - `Read` 是只读工具，源码里明确 `isReadOnly() { return true }`：  
     [FileReadTool.ts](/Users/machengqian.1/code/typescriptProject/claude-code-main/src/tools/FileReadTool/FileReadTool.ts:376)
   - `Edit` / `Write` 是写工具，会走写权限检查：  
     [FileEditTool.ts](/Users/machengqian.1/code/typescriptProject/claude-code-main/src/tools/FileEditTool/FileEditTool.ts:125)  
     [FileWriteTool.ts](/Users/machengqian.1/code/typescriptProject/claude-code-main/src/tools/FileWriteTool/FileWriteTool.ts:135)

2. **路径权限**
   - 读文件走 `checkReadPermissionForTool()`。
   - 写文件走 `checkWritePermissionForTool()`。
   - 这两个都在：  
     [filesystem.ts](/Users/machengqian.1/code/typescriptProject/claude-code-main/src/utils/permissions/filesystem.ts:1030)

最核心的规则是：

```text
读：
  在工作目录内 -> 默认允许
  在内部可读目录 -> 允许
  命中 Read deny/ask 规则 -> 拒绝或询问
  在工作目录外 -> 默认询问

写：
  命中 Edit deny 规则 -> 拒绝
  敏感路径 -> 询问
  在 acceptEdits 模式且在工作目录内 -> 允许
  命中 Edit allow 规则 -> 允许
  其他情况 -> 询问
```

也就是说，Claude Code 默认并不是“所有文件都可写”。它更像：

> **工作目录内默认可读，但写入需要更高权限，除非进入 acceptEdits 模式或用户显式授权。**

3. **Bash 命令的只读判断**
   
文件不只会通过 `Read/Edit/Write` 工具访问，还可能通过 Bash，比如：

```bash
cat file.txt
sed -n '1,20p' file.txt
rg "keyword"
git status
```

这些命令如果被证明是只读，就可以自动允许。核心函数是：

[readOnlyValidation.ts](/Users/machengqian.1/code/typescriptProject/claude-code-main/src/tools/BashTool/readOnlyValidation.ts:1876)

它会解析 shell 命令，判断是否都是只读子命令。如果是：

```ts
return {
  behavior: 'allow',
  updatedInput: input,
}
```

如果不能证明只读，就返回 `passthrough`，交给后面的权限系统继续判断，通常就会询问用户。

所以一句话总结：

**Claude Code 的文件只读权限，是通过“只读工具 + 路径规则 + Bash 静态只读命令验证”共同实现的。**

它不是单纯依赖模型自觉，也不是只靠系统 prompt，而是在工具调用前做权限判定：

```text
模型想读文件
  -> ReadTool
  -> checkReadPermissionForTool
  -> 工作目录内默认允许

模型想改文件
  -> EditTool / WriteTool
  -> checkWritePermissionForTool
  -> 默认询问，除非 acceptEdits 或规则允许

模型想用 Bash 读文件
  -> BashTool
  -> checkReadOnlyConstraints
  -> 能证明只读才自动允许
```

这个设计思想很重要：**只读不是一句提示词，而是工具协议层面的能力边界。**