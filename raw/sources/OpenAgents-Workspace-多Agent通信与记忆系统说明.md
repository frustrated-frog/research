# OpenAgents Workspace 项目阅读说明：多 Agent 通信、记忆系统与架构亮点

> 阅读对象：`/Users/machengqian.1/code/multi-agent-team`  
> 阅读日期：2026-07-05  
> 关注问题：项目如何实现多 Agent 通信、记忆系统如何设计、以及其它值得借鉴的工程亮点。

## 1. 项目定位与整体分层

这个项目可以理解为一个“多 Agent 协作工作空间”系统。它不是单纯的聊天 UI，也不是单纯的 Python SDK，而是由三层共同构成：

1. **Python SDK / Network / Mod 层**
   - 主要在 `sdk/src/openagents/`。
   - 提供统一事件模型、AgentNetwork、EventGateway、Mod 系统、传输层、工作区持久化、各种协作 Mod。
   - 这是项目里最核心的“多 Agent 通信协议与运行时”。

2. **Node agent-connector 层**
   - 主要在 `packages/agent-connector/src/`。
   - 负责把真实 CLI Agent 接入 Hosted Workspace，例如 Claude、Codex、OpenClaw、Cursor、OpenCode、Copilot、Gemini、Cline、Amp、Aider、Goose、Hermes 等。
   - 它通过 HTTP Workspace API 加入工作区、发心跳、拉取事件、把消息派发给本地 CLI 进程，再把结果发回工作区。

3. **Hosted Workspace 后端 / 前端层**
   - 后端主要在 `workspace/backend/`，前端在 `workspace/frontend/`。
   - 提供 `/v1/join`、`/v1/heartbeat`、`/v1/events`、文件、浏览器、todo、timer、routine、knowledge 等工作区 API。
   - 这一层把多 Agent 协作呈现为一个在线工作台，并保存工作区级事件与状态。

所以，项目的核心抽象不是“Agent 彼此直接调用”，而是：

```text
Agent / CLI Agent
  -> Connector / SDK Client
  -> Unified Event
  -> Network EventGateway
  -> Mod Pipeline
  -> Queue / Workspace API / Notification
  -> Target Agent / Channel / Group / Mod / System
```

## 2. 多 Agent 通信的核心思想：统一事件总线

项目用一个统一的 `Event` 模型替代了传统的 direct message、broadcast message、mod message 等多种消息结构。核心定义在：

- `sdk/src/openagents/models/event.py`

`Event` 的关键字段包括：

- `event_name`：事件名，要求是层级式小写格式，例如 `thread.channel_message.post`、`shared_cache.create`、`system.register_agent`。
- `event_id`：事件唯一 ID，默认 UUID。
- `source_id`：来源，支持 `agent:xxx`、`mod:xxx`、`system:system`；如果没有前缀，默认按 Agent 处理。
- `destination_id`：目标，支持 Agent、Channel、Group、Mod、System。
- `payload`：事件业务内容。
- `metadata`：运行时元信息，例如 Hosted Workspace 里会把 `session_id` 放在这里。
- `visibility`：可见性。
- `thread_name`：Agent 本地按 thread 组织事件上下文。
- `relevant_mod`：旧式字段，部分代码仍用它限制相关 Mod。
- `requires_response` / `response_to`：请求-响应关系。
- `secret`：Agent 发事件时的认证 secret。
- `source_agent_group`：网络侧会根据 topology 自动补充来源 Agent 所在 group。

地址模型是这个系统的通信基础。项目支持以下目标形式：

```text
agent:alice          # 发给指定 Agent
agent:broadcast      # 广播给全部 Agent
channel:general      # 发给频道成员
group:research       # 发给 Agent group 成员
mod:shared_cache     # 只交给某个 Mod 处理
system:system        # 交给系统命令处理器
```

可见性由 `EventVisibility` 表达：

- `PUBLIC`：公开。
- `NETWORK`：网络内可见，默认值。
- `CHANNEL`：频道内可见。
- `DIRECT`：只在源和目标之间可见。
- `RESTRICTED`：只允许 `allowed_agents` 中的 Agent 可见。
- `MOD_ONLY`：只允许指定 Mod 处理。

这个设计的好处是：所有通信都变成“命名事件 + 地址 + payload + 元数据”，因此后续扩展频道、私聊、共享缓存、Wiki、Forum、任务委托、MCP 工具等能力时，都不需要发明新的底层通信协议。

## 3. 网络侧事件处理链路

网络核心主要在：

- `sdk/src/openagents/sdk/network.py`
- `sdk/src/openagents/sdk/event_gateway.py`
- `sdk/src/openagents/sdk/event_processor.py`

### 3.1 入口：`AgentNetwork.process_external_event`

外部事件进入网络后，首先走 `AgentNetwork.process_external_event(event)`。

这一步做两件重要事情：

1. **系统事件豁免或认证**
   - 部分无 secret 的系统事件可以直接进入，例如注册类事件。
   - 轮询和注销仍需要认证。
   - 普通 Agent 事件会通过 `SecretManager` 验证 `event.secret`。

2. **交给 EventGateway**
   - 认证通过后，调用 `self.event_gateway.process_event(event)`。
   - 内部 Mod 生成的事件可以走 `AgentNetwork.process_event(event)`，绕过外部认证。

### 3.2 EventGateway：系统命令、Mod 管线、最终投递

`EventGateway.process_event()` 的处理顺序是：

1. 更新事件时间戳。
2. 如果来源是 Agent，自动补充 `source_agent_group`。
3. 记录 Agent 活动计数，例如 `messages_sent`。
4. 如果 `event_name` 以 `system.` 开头，交给 `SystemCommandProcessor`。
5. 普通事件交给 `ModEventProcessor`。
6. 如果系统或 Mod 返回了 `EventResponse`，说明事件被拦截处理，直接返回，不再做普通投递。
7. 如果没有任何 Mod 拦截，才按 `destination_id` 投递到 Agent 队列。

这套逻辑很关键：**Mod 既可以观察事件，也可以拦截事件并替代默认投递行为**。例如 Workspace Messaging Mod 接管 `thread.channel_message.post` 后，会先写历史、处理频道成员，再生成 notification 事件给具体 Agent，而不是让原事件简单广播。

### 3.3 EventGateway 的队列与路由

`EventGateway` 内部维护：

- `processed_event_ids`：去重，避免事件循环重复处理。
- `agent_subscriptions`：Agent 对事件模式的订阅。
- `channel_members`：频道成员表。
- `agent_event_queues`：每个 Agent 一个 `asyncio.Queue`。
- `_activity_counters`：用于记录 Agent 发送/接收活动。

投递逻辑在 `deliver_event(event)`：

- `channel:xxx`：发给该频道所有成员，不发给发送者。
- `group:xxx`：通过 topology 找 group 成员，不发给发送者。
- `agent:broadcast`：发给全部 Agent 队列，不发给发送者。
- `agent:xxx`：发给指定 Agent 队列。
- 其它目标：忽略。

Agent 获取消息时走 `poll_events(agent_id)`，网关会把该 Agent 队列里现有事件一次性取出。

### 3.4 ModEventProcessor：有序可拦截管线

`ModEventProcessor` 的设计很干净：

- 如果 `destination_id` 是 `mod:xxx`，只交给目标 Mod。
- 否则按网络加载的 Mod 顺序逐个调用 `mod.process_event(event)`。
- 如果事件来自某个 Mod，则跳过同名源 Mod，避免自处理循环。
- 第一个返回非 `None` 的 Mod 视为已处理，后续 Mod 和默认投递都不会继续。
- 如果所有 Mod 都返回 `None`，事件交回 EventGateway 做普通目标投递。

这类似浏览器事件里的“中间件 + stop propagation”，也是这个项目最可复用的架构点之一。

## 4. Agent 侧收发链路

Agent SDK 客户端在：

- `sdk/src/openagents/sdk/client.py`

### 4.1 发送事件：`AgentClient.send_event`

发送流程是：

1. Agent 构造 `Event`。
2. 事件依次经过所有本地 `BaseModAdapter.process_outgoing_event()`。
3. 任一 Adapter 返回 `None`，事件就被本地过滤，不再发送。
4. 通过 connector 发给网络。
5. 发送成功后，客户端把 outgoing event 存入 `_event_id_map`。
6. 如果事件没有 `thread_name`，客户端会按 `event_name` 自动推导：

```text
thread:thread.channel_message
thread:shared_cache
thread:system
```

7. 事件追加到本地 `_event_threads[thread_name]`，因此 Agent 能在自己的上下文里看到自己发出的消息。

### 4.2 接收事件：`AgentClient._handle_event`

接收流程是：

1. 打印/记录收到的事件。
2. 唤醒 `wait_event()` 等待者。
3. 调用 Agent 级事件 handler。
4. 事件依次经过所有本地 `BaseModAdapter.process_incoming_event()`。
5. 任一 Adapter 返回 `None`，事件就停止向后处理，也不会进入本地 thread。
6. 如果没有被过滤，客户端同样按 `event_name` 自动补 `thread_name`。
7. 存入 `_event_id_map` 和 `_event_threads`。

所以，Agent 端也有一层“轻量中间件”。网络层的 `BaseMod` 管全局状态和路由，Agent 侧的 `BaseModAdapter` 管单个 Agent 的工具、消息转换和本地上下文。

## 5. Mod 架构：网络全局状态与 Agent 本地工具分离

项目的 Mod 体系分两层：

### 5.1 `BaseMod`：网络级 Mod

定义在：

- `sdk/src/openagents/sdk/base_mod.py`

职责：

- 管理网络级全局状态。
- 通过 `@mod_event_handler("pattern")` 注册事件处理器。
- 可以拦截事件，返回 `EventResponse`。
- 可以使用 WorkspaceManager 获得 Mod 专属存储目录。
- 默认监听 Agent 注册/注销通知。

例子：

- Messaging Mod 维护频道、消息历史、线程、反应、文件。
- Shared Cache Mod 维护共享 key/value 和文件缓存。
- Wiki Mod 维护页面、版本、提案。
- Task Delegation Mod 维护任务状态。

### 5.2 `BaseModAdapter`：Agent 级 Adapter

定义在：

- `sdk/src/openagents/sdk/base_mod_adapter.py`

职责：

- 绑定单个 `agent_id`、connector、AgentClient。
- 提供 Agent 可调用工具 `get_tools()`。
- 转换或过滤 incoming/outgoing events。
- 把网络通知转换成 Agent 更容易处理的本地事件。

这是一个很好的职责拆分：**网络 Mod 负责全局事实，Agent Adapter 负责个体能力与上下文体验**。

## 6. Workspace Messaging：频道、私聊、线程、通知

Messaging Mod 是理解项目多 Agent 通信的最佳样本，主要在：

- `sdk/src/openagents/mods/workspace/messaging/mod.py`
- `sdk/src/openagents/mods/workspace/messaging/adapter.py`
- `sdk/src/openagents/mods/workspace/messaging/thread_messages.py`
- `sdk/src/openagents/mods/workspace/messaging/message_storage_helper.py`

它实现了：

- Channel message。
- Direct message。
- Reply / thread。
- Reaction。
- 文件上传/下载。
- Quoted message。
- Announcement。
- 历史分页检索。
- 新消息通知。

### 6.1 频道消息的处理方式

Agent 发出的频道消息通常是：

```text
event_name: thread.channel_message.post
destination_id: mod 或 channel
payload.channel: general
payload.content: ...
```

Messaging Mod 处理后：

1. 校验并标准化为 `ChannelMessage`。
2. 写入 `message_history`。
3. 如果频道不存在，自动创建频道。
4. 把所有 active agents 加入新频道的 EventGateway channel membership。
5. 更新频道消息计数。
6. 对频道成员生成一组新的 notification 事件：

```text
thread.channel_message.notification
```

7. Notification 的目标是具体 Agent：

```text
destination_id: agent:<agent_id>
```

也就是说，频道消息不是简单地在底层广播原始事件，而是被 Messaging Mod 捕获、存储、再转换成面向成员的通知事件。

### 6.2 私聊消息

私聊入口是：

```text
thread.direct_message.send
```

处理方式类似频道消息：

1. 校验 target agent。
2. 写入历史。
3. 生成：

```text
thread.direct_message.notification
```

4. 通知事件发给目标 Agent。

Agent 侧 `messaging/adapter.py` 会把 direct notification 再转换回 Agent 更自然处理的 direct message 事件。

### 6.3 Reply 和 Thread

Reply 由 `MessageThread` 维护树结构：

- 每个 thread 有 `root_message_id`。
- `replies` 是 `parent_id -> [reply_events]`。
- `message_levels` 记录每条消息在树中的层级。
- `add_reply()` 限制最大层级为 5 层，即 `0-4`。

需要注意：代码顶部注释写明支持 “5-level nested threading”，`add_reply()` 也确实支持 0 到 4 的嵌套层级。如果 README 或外部文档说只支持单层 thread，则和当前实现存在不一致。

Reply 处理逻辑：

1. 根据 `reply_to_id` 找原消息。
2. 如果原消息已经属于某个 thread，就加入该 thread。
3. 否则创建新 thread，以原消息为 root。
4. 更新 `message_to_thread`。
5. 给相关参与者发送：

```text
thread.reply.notification
```

### 6.4 Reaction

Reaction 不是独立表，而是直接写进目标消息 payload 的 `reactions` 字段。相关操作包括 add/remove/toggle/list 等。变更后会给相关 Agent 发送：

```text
thread.reaction.notification
```

### 6.5 Announcement

公告事件：

```text
thread.announcement.set
thread.announcement.get
```

`set` 要求来源 Agent 属于 `admin` group，否则返回 forbidden。公告存在 `channel_announcements` 字典中。

## 7. Node agent-connector：把真实 CLI Agent 接入工作区

Python SDK 解决了事件网络，Node `agent-connector` 解决的是另一个问题：如何把真实命令行 Agent 变成工作区里的可协作成员。

关键文件：

- `packages/agent-connector/src/workspace-client.js`
- `packages/agent-connector/src/adapters/base.js`
- `packages/agent-connector/src/daemon.js`
- `packages/agent-connector/src/adapters/*.js`
- `packages/agent-connector/src/adapters/workspace-prompt.js`

### 7.1 WorkspaceClient：HTTP API 客户端

`workspace-client.js` 封装 Hosted Workspace API：

- `POST /v1/join`：加入工作区。
- `POST /v1/heartbeat`：心跳。
- `POST /v1/events`：发送事件。
- `GET /v1/events`：轮询事件。
- `/v1/discover`：发现信息。
- 文件、浏览器、todos、timers、routines、knowledge 等工作区能力。

它还有一个重要机制：`session_id`。

- Agent 每次 join 都会获得新的 `session_id`。
- 发送事件时会把 `session_id` 放进 `event.metadata.session_id`。
- 心跳也会携带 `session_id`。
- 如果同名 Agent 有新的客户端 join，旧客户端的 session 会被判定为 stale，服务端返回 `session_revoked`，旧 adapter 应停止运行。

这解决了“同名 Agent 多实例同时在线导致重复响应”的问题。

### 7.2 BaseAdapter：连接、心跳、轮询、去重、排队

`packages/agent-connector/src/adapters/base.js` 是所有 CLI Agent Adapter 的基类。它负责：

- join workspace。
- 保存服务端返回的 `_sessionId`。
- 同步 workspace 管理的 skills。
- 每 30 秒心跳。
- 启动前跳到事件流 head，避免处理历史消息。
- 主轮询循环 `_pollLoop()`。
- 调用 `pollPending()` 拉取目标消息。
- 用 `_processedIds` 去重。
- 忽略 status 消息。
- 支持 queue cancel。
- 维护每个 channel 的 busy 状态和队列，避免同一频道并发把上下文打乱。
- 轮询 control events，例如 mode change、stop、restart、status。
- 自动给新 session 生成标题。
- 把 Agent 输出以 status/thinking/response/todos/error 等形式发回工作区。

这种设计等于给每个 CLI Agent 套了一个“工作区运行时外壳”：CLI 只需要处理 prompt 和输出，连接、路由、去重、心跳、并发控制由 Adapter 统一处理。

### 7.3 pollPending 的路由语义

`pollPending()` 会从工作区事件流中过滤出应该由当前 Agent 处理的消息：

- 人类消息。
- 系统/控制消息。
- 指向当前 Agent 的 Agent 消息。
- `metadata.target_agents` 包含当前 Agent 的消息。
- 跳过自己发出的消息。

Hosted Workspace 使用 `workspace.message.posted` 这类事件作为 UI/Connector 层消息格式；底层 SDK 则使用 `Event` 模型。这两套名字不同，但思路一致：事件流 + 目标路由 + metadata 控制。

### 7.4 CLI Agent 的会话记忆

不同 CLI Agent Adapter 会额外维护自己的会话映射：

- Codex Adapter：把每个 channel 对应的 thread/session 信息保存到 `~/.openagents/sessions/${workspaceId}_${agentName}_codex.json`；直接 API fallback 里还有 `_conversationHistory`，上限约 `MAX_HISTORY_ENTRIES = 50`。
- Amp Adapter：类似地按 workspace、agent、channel 保存 thread 映射。
- Claude / OpenCode / Hermes / Gemini 等 Adapter 会保存或恢复各自 CLI 支持的 session id。
- README 中也提到 Aider 的 per-channel chat history、Goose 基于 `(workspace, agent, channel)` hash 的 `--resume`。

这意味着项目有两种上下文记忆：

- 工作区级事件历史：大家共享。
- Agent CLI 自身 session 记忆：某个 Agent 在某个 channel 内连续工作。

## 8. 记忆系统设计：不是一个 Memory 类，而是多层持久化

这个项目的“记忆系统”不是集中在一个 `Memory` 抽象里，而是分层分域设计：

```text
WorkspaceManager SQLite
  -> 网络事件、Agent registry、network_state、event_queue

Mod isolated storage
  -> 每个 Mod 独立目录，保存自己的领域对象

Messaging hot memory + dumps + archive
  -> 聊天历史、线程、反应、文件、频道状态

Hosted Workspace backend DB
  -> UI/HTTP 工作区事件、成员、session、文件、浏览器、knowledge 等

Agent adapter local sessions
  -> 每个 CLI Agent 自己的 per-channel session / conversation history
```

### 8.1 WorkspaceManager：基础持久化层

定义在：

- `sdk/src/openagents/sdk/workspace_manager.py`

初始化工作区目录时创建：

```text
<workspace>/
  network.db
  mods/
  logs/
```

SQLite 表包括：

- `events`：事件持久化。
- `agents`：Agent registry。
- `network_state`：任意 key/value 网络状态。
- `event_queue`：未处理事件队列。

同时提供：

```python
get_mod_storage_path(mod_name)
```

每个 Mod 都拿到自己的独立目录：

```text
<workspace>/mods/<mod_name>/
```

这是一种简单但实用的隔离策略：通用网络状态进 SQLite，领域数据由 Mod 自己选择 JSON、文件、gzip、目录结构等方式保存。

### 8.2 Messaging 的热历史、dump 与归档

Messaging Mod 的历史存在两层：

1. 内存：

```python
message_history: Dict[str, Event]
threads: Dict[str, MessageThread]
message_to_thread: Dict[str, str]
channels: Dict[str, ...]
```

2. 文件：

```text
message_history.json
message_dump_YYYYMMDD_HHMMSS.json
daily_archives/*.json.gz
```

相关配置在 `MessageStorageConfig`：

- `max_memory_messages`：内存消息上限。
- `memory_cleanup_minutes`：内存清理间隔。
- `dump_interval_minutes`：周期 dump 间隔。
- `hot_storage_days`：热存储天数。
- `archive_retention_days`：归档保留天数。

清理策略：

- 周期性保存主历史文件。
- 额外生成带时间戳的 dump，避免崩溃丢数据。
- 热存储超过天数的消息先归档再从内存移除。
- 超过消息上限时，按时间删除最旧消息，降到上限的 80%。
- 归档过期后清理。

这里的设计偏工程实用主义：不用上复杂向量库或事件溯源框架，而是把聊天协作场景需要的“热历史 + 可恢复 + 可归档”先做扎实。

### 8.3 Shared Cache：共享 KV / 文件缓存

位置：

- `sdk/src/openagents/mods/core/shared_cache/mod.py`
- `sdk/src/openagents/mods/core/shared_cache/README.md`

能力：

- `shared_cache.create`
- `shared_cache.get`
- `shared_cache.update`
- `shared_cache.delete`
- `shared_cache.file.upload`
- `shared_cache.file.download`

它提供 Agent 之间共享的 key/value 和文件缓存，并支持 group ACL。实际代码里的存储路径是 Mod 独立目录下再嵌套：

```text
<workspace>/mods/<shared_cache_mod>/shared_cache/cache_data.json
<workspace>/mods/<shared_cache_mod>/shared_cache/files/
```

需要注意：README 对路径的描述可能更简化，实际代码是基于 `get_mod_storage_path()` 的 Mod 隔离目录。

### 8.4 Shared Artifact：可协作产物

位置：

- `sdk/src/openagents/mods/workspace/shared_artifact/mod.py`

能力：

- `shared_artifact.create`
- `shared_artifact.get`
- `shared_artifact.update`
- `shared_artifact.delete`
- `shared_artifact.list`

它把 Agent 协作产生的文本或二进制产物保存为文件，并用 `.metadata.json` 维护元信息。典型路径：

```text
<mod_storage>/shared_artifact/artifacts/
<mod_storage>/shared_artifact/artifacts/.metadata.json
```

适合保存比 cache 更“正式”的协作产物，例如文档、代码片段、分析结果、数据文件。

### 8.5 Wiki：页面、版本、编辑提案

位置：

- `sdk/src/openagents/mods/workspace/wiki/mod.py`
- `sdk/src/openagents/mods/workspace/wiki/README.md`

Wiki 是更结构化的长期知识层，特点：

- 页面存在 `pages/`。
- 版本历史存在 `versions/`。
- 编辑提案存在 `proposals/`。
- 元数据存在 `metadata.json`。
- Owner 可直接编辑。
- 非 Owner 可提交 proposal。
- 支持 search、list、version history、revert。

这个 Mod 的价值在于：它把 Agent 协作产生的知识从“聊天流”提升成“可维护页面”，并且保留版本与提案机制。

### 8.6 Forum：话题、评论、投票

位置：

- `sdk/src/openagents/mods/workspace/forum/mod.py`
- `sdk/src/openagents/mods/workspace/forum/README.md`

Forum 提供：

- Topic create/edit/delete。
- Comment / nested reply，最高 5 层。
- Upvote/downvote。
- Topic search。
- recent/popular/votes 排序。
- group visibility。

实现上是 storage-first：

```text
topics/<topic_id>.json
votes.json
metadata.json
```

注意：如果 README 中说 Forum 是 in-memory，当前代码已经不是纯内存，而是以文件为主、按需加载。

### 8.7 Feed：不可变公告流

位置：

- `sdk/src/openagents/mods/workspace/feed/mod.py`

Feed 面向 one-way announcement/update/info/alert，不像 Forum 那样讨论。它适合记录：

- 系统公告。
- 进展更新。
- 重要 alert。
- 只读信息流。

存储结构是：

```text
posts/<post_id>.json
metadata.json
```

每个 post 更像不可变事件，配合 group ACL 做可见性控制。

### 8.8 Task Delegation：任务状态记忆

位置：

- `sdk/src/openagents/mods/coordination/task_delegation/mod.py`

它提供 A2A-compatible 的任务委托模型：

- 本地或外部任务委托。
- 任务进度历史。
- timeout checker。
- 状态更新。
- task persistence。

典型存储：

```text
tasks/*.json
```

从“记忆系统”的角度看，Task Delegation 保存的不是聊天内容，而是协作执行状态：谁委托了什么、当前进度如何、是否超时、历史更新是什么。

### 8.9 Hosted Workspace 后端的持久化

Hosted Workspace 后端在 `workspace/backend/` 还有自己的数据库模型与迁移：

- workspace events。
- workspace members。
- member `session_id`。
- files。
- browser sessions/contexts。
- knowledge。
- routines/timers。

这层更多服务 UI 和 HTTP Connector。它和 Python SDK 的 WorkspaceManager 不是同一个存储层，但共同构成完整产品的记忆体系。

## 9. 通信与记忆的组合范式

这个项目值得借鉴的地方是：通信和记忆没有分裂成两个完全无关系统，而是通过事件串起来。

以频道消息为例：

```text
Agent 发 thread.channel_message.post
  -> Network 认证
  -> EventGateway
  -> Messaging Mod 拦截
  -> 写 message_history / thread / channel 状态
  -> 生成 thread.channel_message.notification
  -> Network 内部 process_event
  -> 投递到目标 Agent 队列
  -> Agent Adapter 转换/处理
  -> Agent 本地 thread 也保存事件
```

以共享缓存为例：

```text
Agent 发 shared_cache.create
  -> Shared Cache Mod 拦截
  -> 校验 group ACL
  -> 写 cache_data.json 或 files/
  -> 给有权限 Agent 发 notification
```

以 Wiki 为例：

```text
Agent 发 wiki.page.update
  -> Wiki Mod 校验 owner / proposal 规则
  -> 写 page / version / proposal
  -> 返回 response 或通知相关 Agent
```

也就是说，这个系统的“记忆写入”通常发生在 Mod 拦截事件时；“记忆读取”则通过对应 query/retrieve/list/search 事件或 Adapter 工具暴露给 Agent。

## 10. 其它架构亮点

### 10.1 统一事件模型带来的扩展性

所有能力都收敛到 `Event`，扩展新能力只需要：

1. 定义新的 `event_name`。
2. 写一个网络级 Mod 处理该事件。
3. 可选写 Agent 级 Adapter 暴露工具。
4. 可选给 Hosted Workspace UI 增加可视化。

这种模式比“每种消息一种 RPC 接口”更灵活。

### 10.2 动态 Mod 加载 / 卸载

`AgentNetwork` 支持系统事件驱动的 Mod 管理，例如动态 load/unload。相关变更文档也出现在 `changelogs/docs/2025-11-29-dynamic-mod-loading.md`。这使得网络能力可以插件化，而不是固定写死。

### 10.3 多传输支持

SDK 中有 HTTP、gRPC、MCP、A2A 等传输相关代码。`AgentClient` 会通过 health check 检测网络 profile，再选择或校验传输类型。

这意味着同一套事件语义可以跑在不同 transport 上。

### 10.4 MCP Server 暴露工作区能力

`sdk/src/openagents/mcp_server.py` 和 `packages/agent-connector/src/mcp-server.js` 都体现了 MCP 接入思路。工具包括：

- `workspace_get_history`
- workspace agents/status
- files
- browser
- todos
- timers
- routines
- knowledge

这使得 Agent 不只能“聊天”，还可以通过工具读取工作区历史、使用共享浏览器、操作共享文件和任务列表。

### 10.5 Shared Browser Fabric

Workspace 后端有 browser router/manager，Connector prompt 也会注入共享浏览器相关指令。多个 Agent 可以围绕同一个浏览器状态协作，这比单 Agent 自己开一个浏览器更贴合“团队工作空间”。

### 10.6 Control Events 与 Plan/Execute 模式

Node BaseAdapter 轮询 control events，支持 mode changes、stop、restart、status 等控制操作。Adapter 内部 `_mode` 默认为 `execute`，并可根据工作区控制信号改变行为。

这让人类可以在 UI 里控制 Agent，而不是只能通过自然语言提示间接控制。

### 10.7 Skill Hub / Skill Sync

`agent-connector` 在 join 后会读取 workspace 中该 Agent 的 enabled skills，并同步到本地 disabled modules。相关逻辑在：

- `packages/agent-connector/src/adapters/base.js`
- `packages/agent-connector/src/skill-catalog.js`
- `packages/agent-connector/src/skill-installer.js`

这让工作区不仅管理 Agent 在线状态，还能管理 Agent 的能力配置。

### 10.8 Daemon 监督多 Agent

`packages/agent-connector/src/daemon.js` 可以根据配置创建多个 adapter，并监督其进程。这对“多 Agent 团队”产品很重要：实际运行时不是手工开多个 terminal，而是由 daemon 托管。

### 10.9 A2A 任务委托

Task Delegation Mod 与 A2A transport 相关代码说明项目希望兼容更广义的 Agent-to-Agent 协议，而不是只服务自家 UI。

### 10.10 测试覆盖方向

仓库里有后端测试、agent-connector 测试、E2E smoke、SDK 地址测试等，例如：

- `workspace/backend/tests/test_network.py`
- `workspace/backend/tests/test_browser_contexts.py`
- `packages/agent-connector/test/*.test.js`
- `tests/e2e/agent-smoke.js`
- `tests/test_onm_addressing.py`

测试关注点包括 session_id、workspace events、browser contexts、workspace client、stream parser 等，这些都对应多 Agent 系统容易出问题的边界。

## 11. 值得注意的实现细节与不一致点

1. **`EventDestination` 字段拼写是 `desitnation_id`**
   - 代码中实际使用这个拼写，例如 `destination.desitnation_id`。
   - 这是兼容性上需要注意的 typo，不能随手改，否则会影响现有调用。

2. **Messaging Mod 可能重复调用 `_add_to_history`**
   - 在部分 channel message 流程中，进入 handler 时会 add history，后续 `_process_channel_message()` 又会 add history。
   - 因为 `message_history` 是 dict，key 是 `event_id`，重复写通常不会产生重复记录，但会触发清理/保存计数等副作用。

3. **README 与实现存在局部不一致**
   - Messaging 当前支持 5 层 nested thread。
   - Forum 当前是 storage-first，不是纯 in-memory。
   - Shared Cache README 路径描述可能与实际 Mod 隔离目录略有差别。

4. **Python SDK 与 Hosted Workspace 有两套事件命名风格**
   - SDK 偏 `thread.channel_message.post`、`shared_cache.create`。
   - Hosted Workspace / Node connector 偏 `workspace.message.posted`。
   - 两者都走事件流模型，但集成时需要注意映射关系。

5. **Agent 本地 thread 不是全局真相**
   - `_event_threads` 是 AgentClient 本地缓存，帮助 Agent 组织上下文。
   - 真正跨 Agent 的共享历史在 Messaging Mod / Workspace backend / Mod storage 中。

6. **session_id 是防重复实例的关键**
   - Node connector 与后端都围绕 session_id 做 stale session rejection。
   - 多 Agent 系统里，同名 Agent 重复启动很常见，这个机制很实用。

## 12. 可借鉴设计总结

这个项目最值得学习的不是某一个具体 Mod，而是它把多 Agent 协作拆成了几个可以组合的稳定抽象：

1. **统一事件模型**
   - 所有交互都用 Event 表达。
   - 事件名、地址、payload、metadata、visibility 共同描述意图。

2. **EventGateway 作为路由和投递核心**
   - 负责认证后的统一入口、系统命令、Mod 管线、队列投递、订阅和去重。

3. **Mod 管线作为能力扩展点**
   - 网络级 Mod 管全局事实和共享状态。
   - Agent 级 Adapter 管本地工具和事件转换。

4. **以事件驱动记忆写入**
   - 聊天、缓存、Wiki、Forum、Feed、任务状态都通过事件触发写入。
   - 读取通过 retrieve/list/search/query 事件或 MCP 工具暴露。

5. **多层记忆而非单一 Memory**
   - 短期：Agent 本地 thread / CLI session。
   - 中期：Messaging 热历史、channel/thread/reaction。
   - 长期：Wiki、Forum、Feed、Shared Artifact、Task records。
   - 基础设施：WorkspaceManager SQLite、Hosted Workspace DB。

6. **真实 CLI Agent 接入能力**
   - Node adapter 把工作区事件转换成 CLI Agent prompt。
   - 统一处理心跳、去重、session、队列、控制事件。
   - 让不同厂商/不同形态的 Agent 能在同一工作区协作。

7. **人类可控的工作空间**
   - UI/HTTP backend、control events、shared browser、files、todos、timers、knowledge 共同构成“团队协作环境”，而不只是 Agent 间消息传递。

如果要用一句话概括：**OpenAgents Workspace 的核心是一套事件驱动的多 Agent 操作系统雏形；通信靠统一 Event + Gateway + Mod pipeline，记忆靠 WorkspaceManager 与各领域 Mod 的分层持久化，真实 Agent 接入靠 agent-connector 把 CLI 工具包装成工作区成员。**

