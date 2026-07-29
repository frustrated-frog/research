# LangGraph 引擎

**LangGraph 引擎** 是 DeerFlow 的执行骨架——基于 LangGraph 库的图形化 Agent 编排框架，将 Lead Agent 的 ReAct 循环建模为一个状态图，通过图的遍历驱动 Agent 的思考-行动-观察迭代。DeerFlow 0.3 的重写将 Agent 编排从 LangChain 的 Chain 模式迁移到 LangGraph，核心动机是获得**持久化检查点（Checkpointing）**和**图的确定性执行**。

## 为什么选择 LangGraph

LangChain 的 `Chain` 模式是线性的——输入 → Prompt → LLM → Output。适合简单场景，但无法表达"Agent 在工具调用结果上循环推理"这类需要状态回环的复杂工作流。

LangGraph 的核心改进是**将 Agent 建模为图**：

```
节点（Node）：状态转换函数（Agent 节点 + 工具节点）
边（Edge）：条件跳转逻辑（是否继续执行工具调用）
循环（Cycle）：ReAct 的"推理-行动-观察"回环
```

这带来了两个关键能力：

### 1. 持久化检查点（Checkpointing）

LangGraph 的 Checkpointer 可以在任意节点处保存完整状态（`ThreadState`），支持从断点恢复执行：

```python
agent = create_react_agent(
    model,
    tools,
    checkpointer=AsyncCheckpointerProvider.get_instance(),
)
# 下次调用时传入 thread_id，自动恢复上次的 messages、todos 等状态
agent.stream({"messages": [...]}, session_id="thread_abc")
```

即使进程崩溃，只要 session_id 不变，Agent 就能从上次中断的地方继续。

### 2. 中间件钩子

LangGraph 的 `BaseMiddleware`（DeerFlow 的 11 层中间件基于此）提供了在图的各个执行阶段注入逻辑的能力：
- `before_agent`：Agent 执行前
- `before_model`：模型推理前
- `after_model`：模型推理后、工具执行前
- `after_agent`：Agent 执行完成后

这让 DeerFlow 得以实现工具调用的截断、消息的注入、状态的修改等精细控制。

## DeerFlow 中的图结构

Lead Agent 的图结构极为精简——只有 1 个主节点（`agent`），所有中间件都是围绕这个节点的外部钩子：

```
┌──────────────────────────────────────────────────────┐
│                    LangGraph 图                         │
│                                                      │
│   ┌────────────────────────────────────────────┐   │
│   │              agent 节点                       │   │
│   │  ┌─────────────────────────────────────┐  │   │
│   │  │  中间件管道（11 层，before/after钩子）   │  │   │
│   │  └─────────────────────────────────────┘  │   │
│   │  ┌─────────────────────────────────────┐  │   │
│   │  │  LLM（模型推理 + 工具调用生成）          │  │   │
│   │  └─────────────────────────────────────┘  │   │
│   │              ↓ tool_calls                      │
│   │  ┌─────────────────────────────────────┐  │   │
│   │  │  工具执行节点（Tools节点）              │  │   │
│   │  └─────────────────────────────────────┘  │   │
│   └────────────────────────────────────────────┘   │
│                     ↓ no more tool_calls                  │
│              [结束，返回最终回复给用户]                    │
└──────────────────────────────────────────────────────┘
```

LangGraph 内部负责：
1. 管理 `messages` 列表的更新（通过 `merge_messages` reducer）
2. 调用 Checkpointer 保存/恢复状态
3. 判断何时结束循环（`END` 路由）

DeerFlow 在此基础上扩展：
1. 注入 System Prompt（`messages[0]`）
2. 串联 11 层中间件
3. 扩展 ThreadState（`artifacts`、`todos`、`viewed_images` 等）

## ThreadState 的 Reducer

LangGraph 的 `Annotated` 类型允许为状态字段指定 reducer 函数，DeerFlow 定义了三个关键 reducer：

### merge_messages

```python
Annotated[list[BaseMessage], merge_messages]
```

合并新消息到列表，同时处理去重（同一 tool_call_id 的 ToolMessage 只保留最后一个）。

### merge_artifacts

```python
Annotated[list[str], merge_artifacts]
```

`present_files` 工具追加新文件路径，`merge_artifacts` 将新列表与旧列表合并。

### merge_viewed_images

```python
Annotated[dict[str, Any], merge_viewed_images]
```

`view_image` 工具产生的 base64 图片数据与现有字典合并。

## 异步 Checkpointer

DeerFlow 支持两种 Checkpointer：

### AsyncCheckpointerProvider

用于生产环境，异步持久化状态到磁盘：
- 写操作不阻塞 Agent 执行
- 支持高并发多 session
- 底层使用 SQLite 或 PostgreSQL（通过 LangGraph 适配器）

### CheckpointerProvider

用于开发环境，同步持久化：
- 简单直接，调试方便
- 性能要求不高的场景

两者都实现 `BaseCheckpointer` 接口，`create_agent()` 根据配置选择使用哪个。

## 与 LangChain 的关系

```
LangChain 0.3.x
    │
    ├── langchain-core：核心抽象（BaseMessage、BaseTool）
    ├── langchain：Chain 模式（线性执行）
    │
    └── langgraph：图模式（循环 + 状态持久化）← DeerFlow 使用
```

DeerFlow 不是"在 LangChain 之上封装"，而是"基于 LangGraph 的图执行引擎构建自己的 Agent 逻辑"。LangGraph 提供了状态机、循环执行和检查点持久化，DeerFlow 提供：模型工厂、中间件管道、工具生态和记忆架构。

## 小结

LangGraph 引擎是 DeerFlow 0.3 重写的核心基础设施——通过将 ReAct 循环建模为图结构，获得了可靠的状态持久化和可控制的执行流程。DeerFlow 在 LangGraph 的标准执行框架之上，添加了 System Prompt 注入、11 层中间件管道和扩展的 ThreadState，实现了"框架提供骨架，自研提供灵魂"的架构分工。理解 LangGraph 的图执行模型，是理解 DeerFlow 如何实现稳定可靠的 Agent 长时程执行的关键。

---

_关联概念：[[entities/LeadAgent大脑]] [[concepts/ReAct循环]] [[entities/中间件管道]] [[concepts/延迟初始化]]_
