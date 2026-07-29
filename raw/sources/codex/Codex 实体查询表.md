可以。先给你一个总原则：Codex 里这些实体大概分成四类：

```
控制面实体：thread / turn / request / response
core 调度实体：Session / Submission / Op / EventMsg / Task
模型协议实体：Prompt / ResponseEvent / ResponseItem / ResponseInputItem
产品展示实体：TurnItem / notification / thread state
```

下面这张表先把最核心的实体摆清楚。

| 实体                  | 所在层               | 可以怎么理解                 | 主要作用                                                                          |
| ------------------- | ----------------- | ---------------------- | ----------------------------------------------------------------------------- |
| `Thread`            | app-server / core | 一个长期会话容器               | 保存配置、历史、权限、cwd、模型、rollout，可 resume/fork/archive                               |
| `Turn`              | app-server / core | 一次用户任务或一次活跃执行          | 一个 thread 里可以有多个 turn；每个 turn 有开始、流式 item、完成状态                                |
| `Session`           | core              | 活着的 agent runtime      | 持有 history、配置、队列、工具服务、状态、事件输出，是 core 的核心对象                                    |
| `Submission`        | core              | 投递给 session 的一条命令信封    | 包着一个 `Op`，带 submission id，用于进入 Submission Queue                               |
| `Op`                | core              | 外部对 agent 的操作命令        | 用户输入、中断、审批结果、compact、动态工具响应等都用 `Op` 表示                                        |
| `Event`             | core              | core 发给外界的事件信封         | 包着 `EventMsg`，带 id，让外部知道某个 turn/submission 发生了什么                              |
| `EventMsg`          | core protocol     | agent 执行过程中发生的事实       | 如 `TurnStarted`、assistant message、tool begin/end、approval request、token count |
| `RegularTask`       | core task         | 普通 agent turn 的执行任务    | 收到普通用户输入且没有 active turn 时启动，内部调用 `run_turn`                                   |
| `TurnContext`       | core              | 本轮 turn 的运行配置快照        | 固化本轮模型、cwd、权限、sandbox、mode、环境、token window 等                                  |
| `StepContext`       | core              | 某一次模型采样时的世界视图          | 一个 turn 里可能多次采样；每次采样前捕获工具/MCP/插件等状态                                           |
| `Prompt`            | model client      | 真正发给模型的一次请求            | 包含历史 input、模型可见工具 specs、base instructions、输出 schema                           |
| `ResponseEvent`     | model client      | 模型流式返回的一小段事件           | 如 item added、text delta、tool arg delta、completed                              |
| `ResponseItem`      | model protocol    | 模型输出或历史里的结构化 item      | assistant message、reasoning、function call、tool output、web search call 等       |
| `ResponseInputItem` | model protocol    | host 写回给模型的输入 item     | 工具执行结果通常会变成这个，下一次采样时模型能看到                                                     |
| `TurnItem`          | app/UI view       | 前端展示用的 item            | agent message、reasoning、command execution、file change 等展示对象                   |
| `ToolRouter`        | core tools        | 本次采样的工具路由器             | 同时持有模型可见工具说明和 host 侧执行注册表                                                     |
| `ToolRegistry`      | core tools        | host 真正能 dispatch 的工具表 | 根据 tool name 找到对应 runtime/handler 执行                                          |
| `ToolCallRuntime`   | core tools        | 工具调用执行器                | 处理并发、取消、生命周期、调用具体 handler                                                     |
| `RolloutItem`       | persistence       | 可重放历史日志里的记录            | 保存 response item、turn context、compaction、metadata，用于恢复/fork/debug             |

**控制面实体**

|实体|重点解释|
|---|---|
|`thread/start`|创建一个长期执行容器。它不是问模型，而是创建 thread/session，准备配置、权限、历史、队列、listener。|
|`turn/start`|在某个 thread 上提交一次用户输入。它做校验、映射 input、处理 cwd/model/permissions 覆盖，然后生成 `Op::UserInput`。|
|`ThreadRequestProcessor`|app-server 里处理 thread 生命周期请求的地方，比如 start/resume/fork/archive。|
|`TurnRequestProcessor`|app-server 里处理 turn 请求的地方，比如 start/steer/interrupt/review。|
|`ThreadManager`|app-server 管理活跃 core thread/session 的对象。它负责创建、查找、恢复、监听 core session。|
|`CodexThread`|app-server 持有的 core thread handle。外部通过它 submit op、读取 event。|
|`Thread state`|app-server 给产品 API 用的状态视图。它不是 core 真相本身，而是 core events 投影出来的当前状态。|

源码锚点：

- [turn_processor.rs (line 462)](/Users/machengqian.1/code/codex/codex-rs/app-server/src/request_processors/turn_processor.rs:462)
- [thread_processor.rs (line 451)](/Users/machengqian.1/code/codex/codex-rs/app-server/src/request_processors/thread_processor.rs:451)

**core 调度实体**

|实体|重点解释|
|---|---|
|`Session`|Codex agent 的执行内核。你可以把它理解成“活着的 agent 进程内对象”。|
|`Submission Queue`|外部给 core 投递命令的队列。`Op::UserInput`、approval、interrupt 都从这里进。|
|`Event Queue`|core 对外发送执行事实的队列。TUI、exec、app-server listener 都消费这些事件。|
|`submission_loop`|外层事件循环。不断读取 `Submission`，根据 `Op` 类型分发处理。|
|`ActiveTurn`|当前正在运行的 turn 状态。用于判断新输入是 steer 当前任务，还是启动新任务。|
|`InputQueue`|管理 pending input、用户插话、多 agent mailbox 等输入。|
|`SessionTask`|core 里一类可运行任务的抽象。普通回答是 `RegularTask`，还有 review、compact、shell task 等。|
|`RegularTask`|普通用户任务。它发出 `TurnStarted`，然后调用 `run_turn`，必要时继续处理 pending input。|

源码锚点：

- [session/mod.rs (line 532)](/Users/machengqian.1/code/codex/codex-rs/core/src/session/mod.rs:532)
- [handlers.rs (line 712)](/Users/machengqian.1/code/codex/codex-rs/core/src/session/handlers.rs:712)
- [regular.rs (line 37)](/Users/machengqian.1/code/codex/codex-rs/core/src/tasks/regular.rs:37)

**模型与消息实体**

|实体|重点解释|
|---|---|
|`Prompt`|一次模型请求的完整输入。包含历史、工具 specs、base instructions、输出 schema。|
|`ResponseEvent`|模型流式返回的事件，不一定是完整消息。比如文本增量、工具参数增量、item 完成。|
|`ResponseItem`|完整的模型输出 item，也是历史里的核心单位。一个 assistant message、tool call、reasoning 都是 item。|
|`OutputItemAdded`|模型开始输出一个 item。Codex 会据此让 UI 开始展示某个 item。|
|`OutputTextDelta`|assistant 文本的流式片段。前端看到打字机效果主要靠它。|
|`ToolCallInputDelta`|工具调用参数的流式片段。比如模型正在逐步生成 shell 命令参数。|
|`OutputItemDone`|一个 item 完整结束。这里 Codex 才能判断它是普通消息还是工具调用。|
|`Completed`|本次模型 response 完成。注意一次 turn 里可能有多次 response。|
|`ResponseInputItem`|host 写回给模型看的输入 item。工具输出会变成它，进入下一轮 prompt。|

这里最容易混淆的是：

```
ResponseEvent 是流式事件。
ResponseItem 是完整结构化 item。
TurnItem 是给 UI 展示的 item。
```

也就是说：

```
模型流里来了很多 ResponseEvent
-> 拼出或完成 ResponseItem
-> core 记录到历史
-> app-server/UI 投影成 TurnItem/notification
```

源码锚点：

- [turn.rs (line 1937)](/Users/machengqian.1/code/codex/codex-rs/core/src/session/turn.rs:1937)
- [stream_events_utils.rs (line 319)](/Users/machengqian.1/code/codex/codex-rs/core/src/stream_events_utils.rs:319)

**工具实体**

|实体|重点解释|
|---|---|
|`ToolRouter`|本次采样的工具总路由。它同时回答“模型能看到什么”和“host 能执行什么”。|
|`model_visible_specs`|暴露给模型的工具说明。模型只能根据这些 spec 发起 tool call。|
|`ToolRegistry`|host 侧真正执行工具的注册表。模型不可见，但 runtime 需要它 dispatch。|
|`ToolCall`|模型提出的工具调用请求，包含 tool name、call id、参数 payload。|
|`ToolCallRuntime`|执行工具调用的运行时。它处理并发、取消、hooks、错误转模型输出等。|
|`in_flight`|当前模型 response 里已经发起、还没全部完成的工具 future 队列。|
|`FunctionCallOutput`|工具执行结果的一种模型可见输出。下一轮采样时模型会看到。|
|`ToolExposure`|工具暴露策略，例如 Direct、Deferred、Hidden、DirectModelOnly。控制工具是否对模型可见。|

关键关系是：

```
模型看到 model_visible_specs
模型输出 tool call
ToolRouter 解析成 ToolCall
ToolCallRuntime 用 ToolRegistry 找 handler
handler 执行工具
工具结果写回历史
needs_follow_up = true
模型再次采样
```

源码锚点：

- [router.rs (line 35)](/Users/machengqian.1/code/codex/codex-rs/core/src/tools/router.rs:35)
- [spec_plan.rs (line 156)](/Users/machengqian.1/code/codex/codex-rs/core/src/tools/spec_plan.rs:156)
- [parallel.rs (line 42)](/Users/machengqian.1/code/codex/codex-rs/core/src/tools/parallel.rs:42)

**持久化与上下文实体**

| 实体                       | 重点解释                                                   |
| ------------------------ | ------------------------------------------------------ |
| `ContextManager`         | 内存里的历史管理器。负责构造模型 prompt、维护 reference context、处理上下文窗口。  |
| `TurnContextItem`        | 持久化的 turn 上下文基线。用于知道本轮 cwd、权限、模型、环境等状态。                |
| `ContextualUserFragment` | Codex 注入上下文的结构化片段，不是随便拼字符串。                            |
| `Rollout`                | 会话执行日志。记录模型 item、工具调用、工具结果、turn context、compaction 等。  |
| `RolloutItem`            | rollout 里的单条记录。恢复/fork/replay 依赖它。                     |
| `Compaction`             | 上下文压缩记录。不是偷偷改历史，而是在历史里记录“这里发生了一次压缩”。                   |
| `MemoryCitation`         | assistant 使用长期记忆后留下的引用结构。core 会解析并反馈给 memory usage 统计。 |

这组实体解决的是：

```
agent 不只是要跑完当前任务，还要能恢复、fork、审计、压缩、长期记忆反馈。
```

**最关键的流转关系**

最后把这些实体串成一条链：

```
turn/start
-> TurnRequestProcessor
-> Op::UserInput
-> Submission
-> submission_loop
-> user_input_or_turn
-> RegularTask
-> run_turn
-> Prompt
-> ResponseEvent stream
-> ResponseItem
-> ToolCall / assistant message
-> ToolCallRuntime 执行工具
-> ResponseInputItem 写回历史
-> needs_follow_up
-> 下一次 Prompt
-> TurnComplete / EventMsg
-> app-server notification / TurnItem
```

你可以先记住三个“边界”：

|边界|左边|右边|意义|
|---|---|---|---|
|控制面边界|app-server request|core `Op`|API 请求不直接操作模型，先变成 core 命令|
|模型边界|`Prompt`|`ResponseEvent/ResponseItem`|模型只看到 prompt 和工具 specs，host 消费结构化输出|
|展示边界|`EventMsg`|app-server notification / `TurnItem`|core 只产出执行事实，前端决定怎么展示|

如果只用一句话概括：Codex 不是靠一段 prompt 维持 agent loop，而是靠这些结构化实体把“用户输入、模型输出、工具动作、上下文、持久化、UI 展示”拆成清楚的层。