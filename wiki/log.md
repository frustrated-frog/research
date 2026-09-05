# 研究知识库 - 操作日志

> 格式说明：每个条目以 `## [YYYY-MM-DD]` 开头

---

## [2026-09-03] ingest | HarnessDev: Can LLMs Create and Evolve Their Own Agent Harness?

- 类型：source / concept evidence update
- 来源：`raw/sources/文章/字节论文：自进化框架/HarnessDev- Can LLMs Create and Evolve Their Own Agent Harness?.pdf`（41 页）
- 新建：`summaries/HarnessDev：LLM能否创建并演化自己的Agent Harness.md`
- 更新：Harness Engineering 主题索引、`Harness六大组件`、`过早完成声明`、`可观测性`、全局来源索引与 manifest
- 备注：论文区分 Creation 与 Evolution，单独报告反馈集、held-out 泛化、执行 token 成本与跨 executor 迁移；未将可见反馈增益表述为稳定自我进化。

## [2026-04-11] ingest | Redis核心技术与实战 + Redis源码剖析与实战

- 来源：
  - `Redis核心技术与实战-完成/Redis核心技术与实战.md`（8193行）
  - `Redis源码剖析与实战/Redis源码剖析与实战1.md`（5254行）
  - `Redis源码剖析与实战/Redis源码剖析与实战2.md`（2898行）
- 操作：
  - Redis核心技术与实战：完整精读8193行原材料（第01-24讲+答疑），基于准确内容重写摘要和概念条目
  - Redis源码剖析与实战：采样关键段落建摘要
- 完成项（Redis核心技术与实战）：
  - `summaries/Redis核心技术与实战.md` — 1篇结构化摘要
  - `concepts/` — 4个概念（高性能主线/高可靠主线/高可扩展主线/IO多路复用）
  - `entities/` — 3个实体（RESP协议/RedisCluster/缓存机制）
  - `index.md` + `log.md` + `sources/来源摘要.md`
- 完成项（Redis源码剖析与实战）：
  - `summaries/上篇.md` + `summaries/下篇.md` — 2篇结构化摘要
  - `concepts/` — 6个概念（SDS/Hash表/事件驱动/单线程/主从复制/哨兵）
  - `entities/` — 2个实体（Redis持久化/网络通信模块）
  - `index.md` + `log.md`
- 涉及页面：19 个新 wiki 页面
- 备注：原材料体量巨大（合计16,345行），本次采样精读关键章节建立概念框架，概念抽取未完整（跳表/bio/持久化细节等尚待补充）

---

## [2026-04-11] update | 索引原则重构

- 操作：根据与 AI 的讨论重构全局索引原则
- 完成项：
  - 重写 `wiki/index.md` —— 加入「索引即导航」核心原则，说明靠索引导航不靠记忆导航
  - 新增「项目分索引」板块，明确每个项目的入口文件
  - 更新全局内容总表，补全 deerflow-book 的 9 个 concept 和 7 个 entity
  - 更新统计信息（Concepts 16 / Entities 13 / Sources 7 / Synthesis 2）
  - 新增 `hermes-agent/index.md` —— Hermes Agent 项目子索引
  - 重写 `deer-flow/README.md` 和 `harness-engineering/README.md`
  - 在 `wiki/index.md` 末尾新增「健康检查」段落，记录待实现的自动化孤岛检测脚本计划
- 涉及页面：5 个索引文件
- 核心变化：从单一 All-Concepts 索引 → 全局入口 + 项目分索引 + 自动化健康检查

---

## [2026-04-07] ingest | Hermes Agent 官方文档

- 来源：https://hermes-agent.nousresearch.com/docs/
- 操作：
  - 抓取官方文档（Installation、Quickstart、Architecture、Memory、Skills、Tools）
  - 提炼为 hermes-agent 主题的 wiki 页面
- 完成项：
  - `hermes-agent/README.md` — 主题总览
  - `hermes-agent/sources/hermes-agent文档.md` — 来源摘要
  - `hermes-agent/concepts/自我改进循环.md` — 核心概念
  - `hermes-agent/concepts/记忆系统.md` — 记忆机制
  - `hermes-agent/concepts/技能系统.md` — 技能系统
  - `hermes-agent/concepts/工具架构.md` — 工具架构
  - `hermes-agent/entities/aiagent核心循环.md` — 核心引擎
  - `hermes-agent/entities/hermescli.md` — CLI界面
  - `hermes-agent/entities/网关系统.md` — 消息网关
  - `hermes-agent/entities/会话存储.md` — 会话存储
  - `synthesis/hermes-agent与openclaw对比.md` — 综合对比
  - 更新 `wiki/index.md`（总索引）
- 涉及页面：11 个新 wiki 页面
- 备注：重写源码解读文档——全中文Mermaid图表替代ASCII图，详细逐段代码注释，补充Background Review完整代码解析、Nudge触发时机流程图、整体数据流时序图等，约45K字

## [2026-04-06] ingest | Claude Code 源码深度解读（27 篇）

- 来源：`raw/sources/claude-code-deep-dive-main/`
- 操作：
  - 首次摄入：将 27 篇 Claude Code 源码解读文档摄入知识库
  - 结构重组：按主题（claude-code）组织 wiki 结构
  - 命名规范：全部使用简体中文命名
- 完成项：
  - `claude-code/README.md` — Claude Code 总览
  - `claude-code/index.md` — 本主题索引
  - `claude-code/concepts/` — 3 个概念页
  - `claude-code/entities/` — 2 个实体页
  - `claude-code/sources/` — 1 个来源摘要
  - `claude-code/synthesis/` — 1 个综合分析
- 涉及页面：7 个 wiki 页面
- 备注：沙箱隔离机制（sandbox）因与 Claude Code 无关未纳入

## [2026-04-06] init | 知识库初始化

- 操作：创建知识库基础结构
- 完成项：
  - 创建目录结构（raw/sources, wiki, scripts）
  - 编写 AGENTS.md（维护手册）
  - 创建 wiki/index.md（内容索引）
  - 创建 wiki/log.md（操作日志）
- 备注：知识库框架搭建完成

---

## [2026-04-11] restructure | 目录结构重构

- 操作：根据更新后的 knowledge-wiki skill 重构目录结构
- 完成项：
  - 将项目从 `wiki/{project}/` 移动到 `wiki/concepts/{project}/`
  - 将 `wiki/synthesis/` 移动到 `wiki/summaries/`
  - 新增 `wiki/index/` 目录（目前为空）
  - 新增 `wiki/All-Concepts.md` — 全局概念索引表（17 个概念）
  - 新增 `wiki/All-Sources.md` — 全局来源索引表（4 个来源）
  - 更新 `wiki/index.md` — 全局项目索引
  - 追加重构日志到 `wiki/log.md`
- 涉及页面：目录重组，无内容修改
- 备注：后端存储实战课在巩固目录（`/Users/machengqian.1/Documents/巩固/wiki/`），不在研究目录下

---

## [2026-04-13] ingest | Anthropic Managed Agents 工程博客文章

- 来源：https://www.anthropic.com/engineering/managed-agents
- 操作：
  - 抓取 Anthropic Engineering Blog 文章《Scaling Managed Agents: Decoupling the brain from the hands》
  - 保存为 `raw/sources/managed-agents-scaling.md`
- 完成项：
  - `raw/sources/managed-agents-scaling.md` — 原文完整存档
- 涉及页面：1 个源文档
- 核心内容：
  - Managed Agents：Anthropic 的托管式长时序 Agent 服务
  - Pets vs Cattle：单体容器耦合带来的"宠物"问题
  - 解耦设计：将 brain（Claude + harness）、hands（sandbox/tools）、session 分离为独立接口
  - 核心接口：execute(name, input) → string、provision({resources})、wake(sessionId)、getSession(id)、emitEvent(id, event)、getEvents()
  - 安全边界：凭证从不暴露给 sandbox，MCP 代理模式
  - Many brains/many hands 架构
  - TTFT 优化：p50 降低 60%，p95 降低 90%
- 备注：尚待建立 wiki 概念页面

---

## [2026-04-12] restructure | 扁平化公共层重构

- 操作：将项目从 `wiki/concepts/{project}/` 的嵌套结构迁移到扁平公共层结构
- 完成项：
  - 创建顶层 `concepts/`、`entities/`、`summaries/`、`synthesis/`、`index/` 目录
  - 迁移所有概念到 `concepts/{theme}/`（17 个概念）
  - 迁移所有实体到 `entities/{theme}/`（15 个实体）
  - 迁移来源摘要到 `summaries/`（4 个）
  - 迁移综合分析到 `synthesis/`（2 个）
  - 创建 `index/{theme}/index.md` 主题入口（5 个主题）
  - 更新 `All-Concepts.md`（32 个条目）和 `All-Sources.md`（6 个条目）
  - 更新全局 `index.md`
  - 追加日志到 `log.md`
- 涉及页面：全部页面结构重组
- 备注：新结构：concepts/{theme}/、entities/{theme}/、summaries/、synthesis/、index/{theme}/ 五个公共顶层目录

---

## [2026-05-14] ingest | Harness Engineering 完整课程

- 来源：
  - `raw/sources/harness-engineering/Harness_Engineering_完整课程.md`（12讲，146,320字）
- 操作：
  - 精读12讲完整课程内容
  - 创建来源摘要 + 10个概念页 + 4个实体页 + 1个主题入口
- 完成项：
  - `summaries/Harness_Engineering_完整课程.md` — 1篇结构化摘要
  - `concepts/harness-engineering/` — 10个概念（Harness驾驭层/仓库即事实来源/上下文连续性/任务边界与WIP限制/功能清单/过早完成声明/端到端测试/可观测性/初始化阶段/会话交接）
  - `entities/harness-engineering/` — 4个实体（AGENTS.md文件/进度文件/功能清单文件/冲刺合同）
  - `index/harness-engineering/index.md` — 主题入口
  - 更新 `All-Concepts.md`（+10概念 +4实体）、`All-Sources.md`（+1摘要）、`index.md`（统计更新）
- 涉及页面：16 个新 wiki 页面
- 备注：walkinglabs 12讲系统课程，核心命题"模型能力强 ≠ 执行可靠"，同模型配完整 harness 表现有本质差异

---

## [2026-05-25] synthesis | Claude Code 与 DeerFlow 设计思想精读

- 类型：synthesis
- 来源：
  - `raw/sources/claude-code/claude-code-deep-dive-main/`
  - `raw/sources/deer-flow/deerflow-book-main/`
- 涉及页面：
  - `wiki/synthesis/ClaudeCode与DeerFlow设计思想精读.md`
  - `wiki/index.md`
  - `wiki/All-Sources.md`
- 核心内容：
  - 从 Claude Code 与 DeerFlow 中提炼 12 个 Agent 工程设计思想
  - 重点比较行动循环、工具协议、流式事件、安全治理、显式状态、上下文压缩、生命周期扩展、Skills、Sub-agent、记忆、Harness 与架构不变量
  - 形成后续比较其他 Agent 项目的设计地图

---

## [2026-05-25] enrich | Claude Code 与 DeerFlow 设计思想精读下钻链接

- 类型：concept/entity/synthesis
- 操作：
  - 为 `wiki/synthesis/ClaudeCode与DeerFlow设计思想精读.md` 的 12 个设计思想补充“延伸阅读”链接
  - 新增 Claude Code 侧缺失的 5 个细节页
  - 更新 `wiki/index.md`、`wiki/index/claude-code/index.md`、`wiki/All-Concepts.md`
  - 修正 `wiki/entities/deerflow-book/Skills系统.md` 中的旧式/空目标链接
- 新增页面：
  - `wiki/concepts/claude-code/流式响应与事件处理.md`
  - `wiki/concepts/claude-code/上下文压缩策略.md`
  - `wiki/concepts/claude-code/沙箱隔离机制.md`
  - `wiki/entities/claude-code/Skills技能扩展.md`
  - `wiki/entities/claude-code/Sub-Agent机制.md`
- 核心内容：
  - 将综合页升级为可下钻的学习地图
  - 将流式事件、上下文压缩、沙箱、Skills、Sub-Agent 五个 Claude Code 设计点补成独立细节页

---

## [2026-05-26] enrich | Claude Code 流式响应源码下钻

- 类型：concept
- 来源：
  - `raw/sources/claude-code/claude-code-deep-dive-main/06-流式响应与事件处理.md`
  - `/Users/machengqian.1/code/typescriptProject/claude-code-main/src/services/api/claude.ts`
  - `/Users/machengqian.1/code/typescriptProject/claude-code-main/src/services/tools/StreamingToolExecutor.ts`
  - `/Users/machengqian.1/code/typescriptProject/claude-code-main/src/query.ts`
  - `/Users/machengqian.1/code/typescriptProject/claude-code-main/src/QueryEngine.ts`
- 涉及页面：
  - `wiki/concepts/claude-code/流式响应与事件处理.md`
  - `raw/sources/claude-code/claude-code-deep-dive-main/06-流式响应与事件处理.md`
- 核心内容：
  - 补充 raw stream 选择、事件循环状态、content block 初始化、delta 类型守卫、双 yield、message_delta 引用回写
  - 补充 query 层如何边流式边启动工具，以及 fallback 时 tombstone/discard 的协议保护
  - 补充 StreamingToolExecutor 的并发安全判断、队列规则、结果顺序、progress 实时 yield、Bash 错误兄弟取消机制

---

## [2026-05-26] enrich | Claude Code 上下文压缩源码下钻

- 类型：concept
- 来源：
  - `raw/sources/claude-code/claude-code-deep-dive-main/19-上下文压缩策略.md`
  - `/Users/machengqian.1/code/typescriptProject/claude-code-main/src/query.ts`
  - `/Users/machengqian.1/code/typescriptProject/claude-code-main/src/services/compact/microCompact.ts`
  - `/Users/machengqian.1/code/typescriptProject/claude-code-main/src/services/compact/apiMicrocompact.ts`
  - `/Users/machengqian.1/code/typescriptProject/claude-code-main/src/services/compact/timeBasedMCConfig.ts`
  - `/Users/machengqian.1/code/typescriptProject/claude-code-main/src/services/compact/sessionMemoryCompact.ts`
  - `/Users/machengqian.1/code/typescriptProject/claude-code-main/src/services/compact/autoCompact.ts`
  - `/Users/machengqian.1/code/typescriptProject/claude-code-main/src/services/compact/compact.ts`
  - `/Users/machengqian.1/code/typescriptProject/claude-code-main/src/services/compact/grouping.ts`
  - `/Users/machengqian.1/code/typescriptProject/claude-code-main/src/services/compact/postCompactCleanup.ts`
- 涉及页面：
  - `raw/sources/claude-code/claude-code-deep-dive-main/19-上下文压缩策略.md`
- 核心内容：
  - 在原文结尾追加源码下钻增补，补充 query 主循环中的上下文变换顺序
  - 解释 MicroCompact、API MicroCompact、Session Memory Compact、AutoCompact、CompactConversation、Reactive Compact、Post-compact cleanup 的真实职责边界
  - 总结分层降级、协议不变量优先、状态再水合、transcript 与 prompt projection 分离、API-round 分组、缓存感知、熔断与 telemetry 等设计思想

---

## [2026-05-27] source | ClawTeam 通用 Agent 协作设计思想精读

- 类型：source
- 来源：
  - `raw/sources/clawteam/ClawTeam通用Agent协作设计思想精读.md`
  - `https://github.com/HKUDS/ClawTeam`
  - `/tmp/clawteam-inspect/clawteam/spawn/prompt.py`
  - `/tmp/clawteam-inspect/clawteam/team/models.py`
  - `/tmp/clawteam-inspect/clawteam/store/file.py`
  - `/tmp/clawteam-inspect/clawteam/team/mailbox.py`
  - `/tmp/clawteam-inspect/clawteam/transport/file.py`
  - `/tmp/clawteam-inspect/clawteam/harness/phases.py`
  - `/tmp/clawteam-inspect/clawteam/harness/orchestrator.py`
- 涉及页面：
  - `raw/sources/clawteam/ClawTeam通用Agent协作设计思想精读.md`
- 核心内容：
  - 新建 ClawTeam source 目录和源码精读文档
  - 聚焦通用 Agent 协作设计思想，而非 git/worktree 等特定实现
  - 提炼 Agent-facing API、外部化状态、持久异步消息、协议注入、阶段门、运行时适配、事件化 harness 等模式

---

## [2026-05-28] source | Ragent 企业级 Agentic RAG 源码精读

- 类型：source
- 来源：
  - `raw/sources/ragent/Ragent企业级Agentic RAG源码精读.md`
  - `https://github.com/nageoffer/ragent`
  - `/tmp/ragent-inspect-5yu4Bz/bootstrap/src/main/java/com/nageoffer/ai/ragent/rag/service/pipeline/StreamChatPipeline.java`
  - `/tmp/ragent-inspect-5yu4Bz/bootstrap/src/main/java/com/nageoffer/ai/ragent/rag/core/rewrite/MultiQuestionRewriteService.java`
  - `/tmp/ragent-inspect-5yu4Bz/bootstrap/src/main/java/com/nageoffer/ai/ragent/rag/core/intent/IntentResolver.java`
  - `/tmp/ragent-inspect-5yu4Bz/bootstrap/src/main/java/com/nageoffer/ai/ragent/rag/core/intent/DefaultIntentClassifier.java`
  - `/tmp/ragent-inspect-5yu4Bz/bootstrap/src/main/java/com/nageoffer/ai/ragent/rag/core/guidance/IntentGuidanceService.java`
  - `/tmp/ragent-inspect-5yu4Bz/bootstrap/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieve/RetrievalEngine.java`
  - `/tmp/ragent-inspect-5yu4Bz/bootstrap/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieve/MultiChannelRetrievalEngine.java`
  - `/tmp/ragent-inspect-5yu4Bz/bootstrap/src/main/java/com/nageoffer/ai/ragent/rag/core/mcp/LLMMcpParameterExtractor.java`
  - `/tmp/ragent-inspect-5yu4Bz/bootstrap/src/main/java/com/nageoffer/ai/ragent/rag/core/prompt/RAGPromptService.java`
  - `/tmp/ragent-inspect-5yu4Bz/bootstrap/src/main/java/com/nageoffer/ai/ragent/ingestion/engine/IngestionEngine.java`
  - `/tmp/ragent-inspect-5yu4Bz/infra-ai/src/main/java/com/nageoffer/ai/ragent/infra/chat/RoutingLLMService.java`
- 涉及页面：
  - `raw/sources/ragent/Ragent企业级Agentic RAG源码精读.md`
- 核心内容：
  - 新建 Ragent source 目录和源码精读文档
  - 聚焦 RAG 与 Agentic RAG 的通用设计思想，弱化 Java 语言细节
  - 梳理查询改写、意图树路由、歧义澄清、多通道检索、MCP 工具调用、Prompt 场景化、入库流水线、模型路由、流式协议、会话记忆、Trace 等模式

---

## [2026-05-29] enrich | Ragent 多通道检索源码下钻

- 类型：source
- 来源：
  - `raw/sources/ragent/Ragent企业级Agentic RAG源码精读.md`
  - `/tmp/ragent-inspect-5yu4Bz/bootstrap/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieve/RetrievalEngine.java`
  - `/tmp/ragent-inspect-5yu4Bz/bootstrap/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieve/MultiChannelRetrievalEngine.java`
  - `/tmp/ragent-inspect-5yu4Bz/bootstrap/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieve/channel/SearchChannel.java`
  - `/tmp/ragent-inspect-5yu4Bz/bootstrap/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieve/channel/SearchContext.java`
  - `/tmp/ragent-inspect-5yu4Bz/bootstrap/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieve/channel/IntentDirectedSearchChannel.java`
  - `/tmp/ragent-inspect-5yu4Bz/bootstrap/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieve/channel/VectorGlobalSearchChannel.java`
  - `/tmp/ragent-inspect-5yu4Bz/bootstrap/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieve/channel/AbstractParallelRetriever.java`
  - `/tmp/ragent-inspect-5yu4Bz/bootstrap/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieve/postprocessor/DeduplicationPostProcessor.java`
  - `/tmp/ragent-inspect-5yu4Bz/bootstrap/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieve/postprocessor/RerankPostProcessor.java`
- 涉及页面：
  - `raw/sources/ragent/Ragent企业级Agentic RAG源码精读.md`
- 核心内容：
  - 扩充“设计思想四：多通道检索是召回架构”章节
  - 补充召回与排序分离、SearchContext 共享协议、SearchChannel 动态启用、意图定向与全局兜底、两层并行检索、SearchChannelResult 元信息、后处理链收敛器等源码解读
  - 追加一次完整 KB 检索的源码运行过程，并指出当前多通道结果缺少精细证据归因的小边界

---

## [2026-05-29] enrich | Ragent 多通道检索 Mermaid 图解

- 类型：source
- 来源：
  - `raw/sources/ragent/Ragent企业级Agentic RAG源码精读.md`
- 涉及页面：
  - `raw/sources/ragent/Ragent企业级Agentic RAG源码精读.md`
- 核心内容：
  - 在“设计思想四：多通道检索是召回架构”章节增量插入 Mermaid 图
  - 补充候选扩张到后处理收敛总览图、RetrievalEngine 子问题并行图、SearchChannel 动态启用决策图、两层并行检索图、后处理链收敛图、源码调用顺序图

---

## [2026-07-29] ingest | Pi Agent Harness 源码设计精读

- 类型：source
- 来源：
  - `raw/sources/pi/Pi Agent Harness源码设计精读.md`
  - `/Users/machengqian.1/code/typescriptProject/pi`
- 操作：
  - 精读 Pi monorepo 的模型协议、Agent Core、Coding Agent、Session、Compaction、Extensions、TUI、RPC、Server 与新 AgentHarness 设计
  - 新建一份不依赖源码路径引用、以技术机制和设计取舍为中心的深度中文文档
  - 区分当前稳定 AgentSession 运行链与正在演进的 Durable AgentHarness，避免混淆已实现能力和设计目标
  - 补充 14 幅 Mermaid 流程图、时序图和状态图
  - 新建来源摘要与主题入口，并更新全局来源索引
- 完成项：
  - `raw/sources/pi/Pi Agent Harness源码设计精读.md` — 深度技术文档
  - `wiki/summaries/Pi Agent Harness源码设计精读.md` — 来源摘要
  - `wiki/index/pi-agent-harness/index.md` — 主题入口
  - `wiki/index.md` — 新增主题并更新统计
  - `wiki/All-Sources.md` — 新增来源并更新统计
- 涉及页面：3 个新页面，3 个索引/日志更新

---

## [2026-07-29] enrich | Pi Agent Loop 源码下钻

- 类型：source
- 来源：
  - `raw/sources/pi/Pi Agent Harness源码设计精读.md`
  - `/Users/machengqian.1/code/typescriptProject/pi/packages/agent/src/agent-loop.ts`
  - `/Users/machengqian.1/code/typescriptProject/pi/packages/agent/test/agent-loop.test.ts`
- 操作：
  - 仅在原文结尾追加“Pi 的 Agent Loop 到底是如何运转的”专题，未修改原有正文
  - 详细解释新 Prompt 与 Continue 两种入口、工作上下文与新增消息双账本、双层循环和 turn 事务边界
  - 下钻流式 AssistantMessage 归约、停止原因矩阵、工具准备/执行/收尾、进度事件结算屏障
  - 解释并行工具预检顺序、完成事件顺序、ToolResult 稳定写回顺序和全体 terminate 规则
  - 说明 Steering、Follow-up、prepareNextTurn、shouldStopAfterTurn 的精确优先级
  - 区分低层 EventStream 的观察语义与 Agent awaited listener 的执行屏障
  - 补充 10 幅 Mermaid 流程图、时序图和状态图
- 完成项：
  - `raw/sources/pi/Pi Agent Harness源码设计精读.md` — 结尾新增 29 节 Agent Loop 深度解析
- 备注：追加前 52,807 字节内容的 SHA-256 保持不变，确认原文未被改动

---

## [2026-08-01] lint | 知识库结构强化

- 类型：lint / maintenance
- 范围：仅更新 `wiki/`；未修改 `raw/sources/` 内容
- 操作：
  - 为 68 个 Wiki Markdown 页面补齐稳定 `id`、状态、别名、有效期和 supersession 元数据
  - 创建 `wiki/manifest.json`，记录 164 个原始文件的 SHA-256、大小、修改时间、摄入状态和页面映射
  - 修复 15 个页面中的 49 条可确定的错误 Wiki 链接
  - 将已有 `related` 关系补成正文 wikilink，强化 20 个页面的 Obsidian 图谱连接
  - 重建 `All-Concepts.md`、`All-Sources.md` 和根索引统计
  - 修正相对路径与裸标题形式的 Obsidian 链接解析遗漏，并补齐 1 个摘要页的稳定 `id`
- 检查结果：
  - Wiki 页面：69 个；概念 30 个；实体 21 个；来源摘要 6 个；综合分析 2 个
  - 缺失 frontmatter：0 个（`log.md` 作为 append-only 操作日志保留独立格式）
  - 重复 `id`：0 个；可解析 Wiki 链接：228 条；断链：0 条（强化前基线为 184 条）
  - 已映射原始文件：39 个；尚未映射原始文件：125 个
  - 概念/实体孤立页面：0 个；仍有 39 个页面缺少具体来源字段，暂不凭空补写证据
- 备注：原始资料保持只读，后续可按来源主题增量补齐证据映射和孤岛关系

---

## [2026-09-03] maintenance | 文章目录按论文归档

- 类型：路径整理 / manifest 更新
- 操作：将 5 组可明确配对的 PDF 与对应 Markdown 移入中文命名的论文子目录；“组合式技能路由论文”同时保留中文导读和中文正文。
- 新目录：`智能体原生记忆系统论文`、`LLM评判与自我改进论文`、`组合式技能路由论文`、`Claude Code设计空间论文`、`生成式技能组合论文`
- 未移动：没有可明确配对 PDF 的独立 Markdown；已完成归档的 `字节论文：自进化框架`。
- 影响：仅改变原始资料路径；文件内容、哈希和来源数量不变；`wiki/manifest.json` 已同步。

---

## [2026-09-04] ingest | Runtime-Independent Persistent Agents

- 类型：source / 论文解读
- 来源：raw/sources/文章/运行时独立持久智能体论文/Runtime-Independent Persistent Agents.pdf
- 操作：
  - 逐页提取并核对架构图、授权迁移协议、provider 表与机制证据表。
  - 新建中文论文解读，明确标注原文事实、作者解释和解读推断；将系统连续性与行为身份保真度分开叙述。
  - 将 PDF 与解读文档归入中文目录“运行时独立持久智能体论文”。
  - 新建来源摘要，并以来源证据更新“上下文连续性”“会话交接”和 Harness Engineering 索引。
- 完成项：
  - raw/sources/文章/运行时独立持久智能体论文/运行时独立持久智能体论文解读.md
  - wiki/summaries/运行时独立持久智能体.md
  - wiki/concepts/harness-engineering/上下文连续性.md
  - wiki/concepts/harness-engineering/会话交接.md
  - wiki/manifest.json、wiki/All-Sources.md、wiki/index.md、wiki/index/harness-engineering/index.md
