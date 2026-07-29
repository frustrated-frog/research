# Lead Agent（大脑核心）

**Lead Agent** 是 DeerFlow 的核心编排引擎——它负责理解用户意图、规划执行步骤、调用工具、与 Sub-agent 协作，最终产出用户需要的结果。所有用户请求都经过 Lead Agent 的 ReAct 循环处理，它就像一个项目主管，知道什么时候该亲自动手，什么时候该委派给专业人员。

## 在系统中的位置

```
用户请求
    │
    ▼
Gateway API ──▶ Lead Agent（ReAct 循环）
                    │
                    ├── 工具调用（bash/read_file/write_file/...）
                    │
                    ├── Sub-agent 委派（task 工具）
                    │
                    └── 响应生成
```

Lead Agent 由 `make_lead_agent()` 工厂函数创建，内部调用 `create_agent()` 构建 LangGraph 图实例，并将 11 层中间件依次串联到管道中。

## 核心组件

### 1. 模型工厂

Lead Agent 支持多种模型接入，通过 `_resolve_model_name()` 识别配置中的模型标识符：

```python
def _resolve_model_name(model_name: str) -> str:
    # Claude 系列
    if model_name.startswith("claude"):
        return "anthropic"
    # DeepSeek 系列（Kimi、豆包等使用相同适配类）
    if model_name.startswith("deepseek") or \
       model_name.startswith("kimi") or \
       model_name.startswith("doubao"):
        return "deepseek"
    # Gemini 系列
    if model_name.startswith("gemini"):
        return "google"
    # 默认走 OpenAI 兼容接口
    return "openai"
```

模型实例通过 `create_chat_model()` 工厂函数创建，内部使用 LangChain 的适配器类（如 `ChatAnthropic`、`PatchedChatDeepSeek`）封装各家的 API 差异。

### 2. 工具注入

Lead Agent 可用的工具分三层叠加：

```
基础工具（BUILTIN_TOOLS）
    ├── present_files        # 文件呈现给前端
    └── ask_clarification   # 向用户提问

Sub-agent 工具（SUBAGENT_TOOLS，仅 subagent_enabled=True 时）
    └── task                # 委派任务给 Sub-agent

社区工具（从 config.yaml 加载）
    ├── web_search (Tavily)
    ├── web_fetch (Tavily/Jina)
    └── image_search (DuckDuckGo)

MCP 工具（从 extensions_config.json 加载）
    └── 来自 MCP 服务器的工具
```

工具过滤由 `ToolFilterMiddleware` 控制，可根据模型能力排除不支持的工具（如视觉模型才注入 `view_image`）。

### 3. 中间件管道

11 层中间件依次处理，完整的管道顺序为：

```
MessageInput
    ↓
ThreadDataMiddleware       ── 注入 thread_id、user_id 等元信息
    ↓
UploadsMiddleware          ── 注入用户上传文件的内容块
    ↓
SandboxMiddleware           ── 管理沙箱的获取和释放
    ↓
DanglingToolCallMiddleware  ── 修复中断导致的悬空工具调用
    ↓
[LangGraph: Model]
    ↓
SummarizationMiddleware    ── 自动压缩过长的消息历史
    ↓
TodoMiddleware             ── 摘要后恢复任务列表
    ↓
TitleMiddleware            ── 生成会话标题
    ↓
MemoryMiddleware           ── 捕获对话用于长期记忆
    ↓
ViewImageMiddleware        ── 管理多模态图片注入
    ↓
SubagentLimitMiddleware    ── 截断超额的任务委派调用
    ↓
ClarificationMiddleware    ── 处理用户的澄清回复
    ↓
MessageOutput
```

### 4. 系统提示词模板

Lead Agent 的 System Prompt 由 `apply_prompt_template()` 函数构建，将多个模块化的提示块组合成完整的系统指令：

```python
SYSTEM_PROMPT_TEMPLATE = "\n".join([
    "<role>",           # 角色定义：你是 DeerFlow AI 助手
    "<thinking_style>",# 思考风格：结构化推理
    "<clarification_system>", # 澄清机制说明
    "<skill_system>",  # Skill 系统：17 个内置 Skill 的元数据
    "<subagent_system>",# Sub-agent 编排：如何分解任务
    "<response_style>",# 响应风格：简洁、有条理
])
```

每个模块独立维护，通过函数参数控制是否启用（如 `subagent_enabled=False` 时不注入 `<subagent_system>`）。

## ThreadState：跨轮次持久化的状态容器

Lead Agent 的运行时状态定义在 `ThreadState` 中，通过 Checkpointer 实现跨轮次持久化：

```python
class ThreadState(TypedDict):
    messages: Annotated[list[BaseMessage], merge_messages]
    # 消息历史，merge_messages reducer 保证顺序和去重

    artifacts: Annotated[list[str], merge_artifacts]
    # Agent 生成的产出物路径列表（用于前端渲染）

    viewed_images: Annotated[dict[str, Any], merge_viewed_images]
    # 多模态图片的 base64 数据

    todos: list[dict[str, Any]]
    # 当前任务列表（由 write_todos 工具维护）

    sandbox: dict[str, str] | None
    # 当前沙箱 ID

    task_count: int
    # 已创建的 Sub-agent 任务计数（用于生成唯一 task_id）
```

关键理解：**这些状态通过 Checkpointer 持久化到磁盘**，所以即使进程重启，只要 session_id 不变，Agent 就能从上次中断的地方继续执行。

## 执行入口

外部通过 `agent_executor.py` 的 `stream()` 方法与 Lead Agent 交互：

```python
def stream(input: dict, session_id: str, ...) -> Iterator[dict]:
    # 1. 加载或恢复 ThreadState
    # 2. 注入 System Prompt
    # 3. 执行 LangGraph 图
    # 4. yield 每个中间步骤（工具调用、思考、最终回复）
```

流式输出（SSE）让前端能够实时显示 Agent 的执行过程——包括 `<reasoning>` 标签中的思考、工具调用的参数和结果。

## 小结

Lead Agent 是 DeerFlow 的中枢——它接收用户请求，通过 LangGraph 的 ReAct 循环和 11 层中间件的协同处理，利用工具和 Sub-agent 协作能力完成复杂任务，同时通过 Checkpointer 实现可靠的跨轮次状态持久化。理解 Lead Agent 的结构，是理解 DeerFlow 如何实现"稳定可靠的长时程 Agent 执行"的关键。

---

_关联概念：[[entities/LangGraph引擎]] [[concepts/ReAct循环]] [[entities/中间件管道]] [[concepts/上下文隔离]]_
