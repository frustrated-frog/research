# `/api/chat` A2UI + 对话统一 Checkpoint 回退接口文档

本文档说明 `/api/chat` 在 A2UI 生成、编辑场景下如何返回 checkpoint，以及前端如何用 checkpoint 恢复 A2UI 页面和对话历史。

这里的 checkpoint 是业务版本点，和文件系统 `/rollback` 不是同一套能力。一个 checkpoint 同时绑定：

- 当前 A2UI JSON。
- 当前线程上传图片上下文。
- 当前对话历史的消息截断点。

前端只需要保存后端返回的 `checkpoint_id`，回退时调用 restore 接口即可；不需要自己理解或传递对话截断位置。

## 1. 接口总览

| 接口 | 方法 | 作用 |
| --- | --- | --- |
| `/api/chat` | `POST` | 对话、A2UI 生成、A2UI 编辑；通过 SSE 返回 checkpoint 信息。 |
| `/api/threads/{thread_id}/a2ui` | `GET` | 获取当前线程 A2UI JSON，同时返回当前页面对应的 `latest_checkpoint_id`。 |
| `/api/threads/{thread_id}/a2ui` | `PUT` | 替换当前线程 A2UI JSON，并自动生成页面保存 checkpoint。 |
| `/api/threads/a2ui` | `PUT` | 未指定 thread 时替换 A2UI JSON，由后端生成 thread，并自动生成页面保存 checkpoint。 |
| `/api/threads/{thread_id}/a2ui/checkpoints` | `GET` | 获取当前线程 checkpoint 列表。 |
| `/api/threads/{thread_id}/a2ui/checkpoints/{checkpoint_id}/restore` | `POST` | 回退到指定 checkpoint，同时恢复 A2UI JSON 并截断错误对话历史。 |

## 2. 通用约定

### 2.1 认证

如果服务端配置了 `API_SERVER_KEY` 或 `platforms.api_server.key`，请求需要带 Bearer Token：

```http
Authorization: Bearer <API_SERVER_KEY>
```

本地未配置 key 时，接口允许直接访问。

### 2.2 thread_id

`thread_id` 是 A2UI 页面状态和对话历史的统一线程 ID。

`/api/chat` 支持请求体字段：

| 字段 | 说明 |
| --- | --- |
| `threadId` | 当前前端正式字段。 |
| `thread_id` | 兼容字段。 |

如果请求没有传 thread，后端会生成或复用默认 thread。`/api/chat` 响应头会返回：

```http
X-Thread-Id: <thread_id>
Set-Cookie: HERMES_A2UI_THREAD_ID=<thread_id>
```

前端刷新页面后，可以继续使用同一个 `thread_id` 获取 A2UI 状态和 checkpoint 列表。

### 2.3 checkpoint_id

`checkpoint_id` 形如：

```text
cp_2d0f0c4a59c748f4b7c8d0fdc0a4e311
```

它由后端生成，前端不要自行拼接。

### 2.4 latest_checkpoint_id 的语义

字段名是 `latest_checkpoint_id`，但前端应把它理解为“当前页面状态对应的 checkpoint”。

正常生成或编辑完成后，它通常就是最新创建的 checkpoint。执行 restore 后，它会指向被恢复的 checkpoint，即使列表里还有更晚创建的“回退前自动保存” checkpoint。

## 3. `POST /api/chat`

### 3.1 请求

```http
POST /api/chat
Content-Type: application/json
Accept: text/event-stream
```

请求体示例：

```json
{
  "threadId": "t1",
  "modelName": "jd-chat:xxx",
  "message": "把当前页面的按钮改成红色",
  "imageList": []
}
```

字段说明：

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `message` | `string` | 否 | 用户输入。`message` 和 `imageList` 至少传一个。 |
| `imageList` | `string[]` | 否 | 图片 URL 或 data URL。兼容字段：`images`、`image_list`。 |
| `threadId` | `string` | 否 | 当前线程 ID。兼容字段：`thread_id`。 |
| `modelName` | `string` | 否 | 前端选择的 JD 模型。兼容字段：`model_name`、`model`。 |
| `secrets` | `object` | 否 | 敏感配置透传，例如 `DECO_MCP_TOKEN`。和 checkpoint 无直接关系。 |

### 3.2 SSE 响应事件

`/api/chat` 返回 `text/event-stream`。每个事件格式：

```text
event: <event_name>
data: <json>
```

checkpoint 相关事件有 3 个。

#### 3.2.1 `meta`

第一条业务事件。用于告诉前端本轮对话的 thread，以及本轮操作前的恢复点。

```text
event: meta
data: {"thread_id":"t1","base_checkpoint_id":"cp_before"}
```

字段说明：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `thread_id` | `string` | 后端最终使用的线程 ID。 |
| `base_checkpoint_id` | `string \| null` | 本轮开始前的恢复点。当前线程没有 A2UI 页面时为 `null`。 |

生成规则：

- 如果本轮开始前已有 A2UI 页面，后端会创建或复用一个 `api_chat_pre_turn` checkpoint。
- 如果已有当前 checkpoint 且快照内容一致，后端会复用它。
- 如果本轮开始前没有 A2UI 页面，`base_checkpoint_id` 为 `null`。

前端用途：

- 用户在本轮生成或编辑过程中发现改错，可以用 `base_checkpoint_id` 回到本轮操作前。
- 如果 `base_checkpoint_id` 为 `null`，说明本轮之前没有可恢复的 A2UI 页面版本。

#### 3.2.2 `checkpoint`

当本轮 `/api/chat` 生成或编辑了 A2UI，并且最终 A2UI 状态相对本轮开始时发生变化，后端会在结束前返回一个新的 checkpoint 事件。

```text
event: checkpoint
data: {
  "thread_id": "t1",
  "checkpoint_id": "cp_after",
  "created_at": 1778760000.123,
  "label": "本轮编辑后",
  "source": "api_chat_post_turn",
  "message_cutoff_id": 42,
  "message_count": 12,
  "a2ui_message_count": 5
}
```

字段说明：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `thread_id` | `string` | 所属线程。 |
| `checkpoint_id` | `string` | 新 checkpoint ID。 |
| `created_at` | `number` | 创建时间，Unix 秒时间戳。 |
| `label` | `string \| null` | 展示文案，例如 `本轮生成后`、`本轮编辑后`。 |
| `source` | `string` | checkpoint 来源。`/api/chat` 结束后通常是 `api_chat_post_turn`。 |
| `message_cutoff_id` | `number \| null` | 后端对话消息截断点。前端不需要传回。 |
| `message_count` | `number` | checkpoint 创建时该线程持久化对话消息数。 |
| `a2ui_message_count` | `number` | checkpoint 中 A2UI 顶层消息数量。 |

注意：

- 普通聊天不会产生新的 `checkpoint` 事件。
- A2UI 没变化时不会产生新的 `checkpoint` 事件。
- 前端收到该事件后，应把它作为当前页面最新可恢复版本保存。

#### 3.2.3 `done`

本轮 SSE 结束事件。

```text
event: done
data: {"thread_id":"t1","latest_checkpoint_id":"cp_after"}
```

字段说明：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `thread_id` | `string` | 当前线程。 |
| `latest_checkpoint_id` | `string \| null` | 本轮结束后当前页面对应的 checkpoint。 |

场景说明：

| 场景 | `checkpoint` 事件 | `done.latest_checkpoint_id` |
| --- | --- | --- |
| 本轮生成了新 A2UI 页面 | 有 | 新生成的 checkpoint。 |
| 本轮编辑了已有 A2UI 页面 | 有 | 新生成的 checkpoint。 |
| 本轮普通聊天，已有 A2UI 页面 | 无 | 已有当前 checkpoint。 |
| 本轮普通聊天，没有 A2UI 页面 | 无 | `null`。 |
| Agent 执行出错但已有 checkpoint | 通常无 | 当前已有 checkpoint。 |

### 3.3 `/api/chat` SSE 完整示例

#### 生成或编辑 A2UI

```text
event: meta
data: {"thread_id":"t1","base_checkpoint_id":"cp_before"}

event: text
data: {"content":"..."}

event: a2ui
data: {"surfaceUpdate":{"surfaceId":"main","components":[]}}

event: checkpoint
data: {"thread_id":"t1","checkpoint_id":"cp_after","created_at":1778760000.123,"label":"本轮编辑后","source":"api_chat_post_turn","message_cutoff_id":42,"message_count":12,"a2ui_message_count":5}

event: done
data: {"thread_id":"t1","latest_checkpoint_id":"cp_after"}
```

#### 普通聊天

```text
event: meta
data: {"thread_id":"t1","base_checkpoint_id":"cp_current"}

event: text
data: {"content":"当前页面包含一个 Banner 和两个按钮。"}

event: done
data: {"thread_id":"t1","latest_checkpoint_id":"cp_current"}
```

## 4. `GET /api/threads/{thread_id}/a2ui`

获取当前线程的 A2UI JSON，同时返回当前页面对应的 checkpoint。

### 4.1 请求

```http
GET /api/threads/t1/a2ui
```

### 4.2 响应

```json
{
  "thread_id": "t1",
  "a2ui_json": [
    {
      "beginRendering": {
        "surfaceId": "main",
        "root": "root"
      }
    }
  ],
  "latest_checkpoint_id": "cp_current"
}
```

字段说明：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `thread_id` | `string` | 当前线程。 |
| `a2ui_json` | `object[]` | 当前页面 A2UI JSON。 |
| `latest_checkpoint_id` | `string \| null` | 当前页面对应的 checkpoint。没有 A2UI checkpoint 时为 `null`。 |

前端用途：

- 页面刷新后，先调这个接口恢复当前画布。
- 同时把 `latest_checkpoint_id` 作为当前版本 ID。

## 5. `GET /api/threads/{thread_id}/a2ui/checkpoints`

获取当前线程的 checkpoint 列表，用于展示版本列表或刷新后恢复版本选择器。

### 5.1 请求

```http
GET /api/threads/t1/a2ui/checkpoints?limit=50
```

查询参数：

| 参数 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `limit` | `number` | 否 | 返回数量，默认 `50`，最大 `200`，最小 `1`。 |

### 5.2 响应

```json
{
  "thread_id": "t1",
  "checkpoints": [
    {
      "checkpoint_id": "cp_after",
      "thread_id": "t1",
      "created_at": 1778760000.123,
      "label": "本轮编辑后",
      "source": "api_chat_post_turn",
      "message_cutoff_id": 42,
      "message_count": 12,
      "a2ui_message_count": 5
    },
    {
      "checkpoint_id": "cp_before",
      "thread_id": "t1",
      "created_at": 1778759900.456,
      "label": "本轮开始前",
      "source": "api_chat_pre_turn",
      "message_cutoff_id": 40,
      "message_count": 10,
      "a2ui_message_count": 4
    }
  ]
}
```

列表按 `created_at` 倒序返回。

注意：restore 后，列表第一条可能是 `api_a2ui_restore_pre`，而当前页面实际指向的 checkpoint 应以 `GET /api/threads/{thread_id}/a2ui` 的 `latest_checkpoint_id` 为准。

## 6. `POST /api/threads/{thread_id}/a2ui/checkpoints/{checkpoint_id}/restore`

回退到指定 checkpoint。该接口会同时处理 A2UI 页面和对话历史。

### 6.1 请求

```http
POST /api/threads/t1/a2ui/checkpoints/cp_before/restore
```

请求体为空。

### 6.2 成功响应

```json
{
  "success": true,
  "thread_id": "t1",
  "checkpoint_id": "cp_before",
  "a2ui_json": [
    {
      "beginRendering": {
        "surfaceId": "main",
        "root": "root"
      }
    }
  ],
  "removed_message_count": 2
}
```

字段说明：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `success` | `boolean` | 是否恢复成功。 |
| `thread_id` | `string` | 当前线程。 |
| `checkpoint_id` | `string` | 被恢复的 checkpoint。 |
| `a2ui_json` | `object[]` | 恢复后的 A2UI JSON。前端可以直接用它刷新画布。 |
| `removed_message_count` | `number` | 后端从对话历史里删除的消息数量。 |

### 6.3 restore 的后端行为

restore 成功后，后端会做这些事：

1. 如果当前线程有 A2UI 页面，先创建一个 `api_a2ui_restore_pre` checkpoint，保存“回退前”的页面，方便用户反向找回。
2. 用目标 checkpoint 中的 A2UI JSON 覆盖当前页面。
3. 恢复目标 checkpoint 中记录的上传图片上下文。
4. 将当前页面 checkpoint 指针改为被恢复的 `checkpoint_id`。
5. 从对话历史中删除 `message_cutoff_id` 之后的消息。

因此 restore 后，下一轮 `/api/chat` 传给 Agent 的 `conversation_history` 不会包含 checkpoint 之后的错误对话。

### 6.4 失败响应

checkpoint 不存在：

```json
{
  "error": {
    "message": "Checkpoint not found"
  }
}
```

HTTP 状态码：`404`。

缺少 checkpoint_id：

```json
{
  "error": {
    "message": "Missing checkpoint_id"
  }
}
```

HTTP 状态码：`400`。

## 7. `PUT /api/threads/{thread_id}/a2ui`

该接口用于前端直接保存页面 A2UI JSON。保存成功后，后端会自动生成 `api_a2ui_put` checkpoint。

### 7.1 请求

```http
PUT /api/threads/t1/a2ui
Content-Type: application/json
```

```json
{
  "a2ui_json": [
    {
      "beginRendering": {
        "surfaceId": "main",
        "root": "root"
      }
    }
  ]
}
```

兼容字段：

```json
{
  "a2uiJson": []
}
```

### 7.2 响应

```json
{
  "success": true,
  "thread_id": "t1"
}
```

为了兼容旧前端，PUT 响应暂不直接返回 checkpoint。前端保存后如果需要最新版本 ID，可以再调用：

```http
GET /api/threads/t1/a2ui
```

读取 `latest_checkpoint_id`。

## 8. `PUT /api/threads/a2ui`

和上一节类似，但不要求 URL 里带 `thread_id`。后端会生成或解析线程 ID。

### 8.1 请求

```http
PUT /api/threads/a2ui
Content-Type: application/json
```

```json
{
  "a2ui_json": []
}
```

### 8.2 响应

```json
{
  "success": true,
  "thread_id": "auto_thread_id"
}
```

同样会自动生成 `api_a2ui_put` checkpoint。需要版本 ID 时，继续用 `GET /api/threads/{thread_id}/a2ui` 查询。

## 9. checkpoint 来源说明

| source | 产生时机 | label |
| --- | --- | --- |
| `api_chat_pre_turn` | `/api/chat` 本轮开始时，线程已有 A2UI 页面，作为操作前恢复点。 | `本轮开始前` |
| `api_chat_post_turn` | `/api/chat` 本轮结束时，A2UI 状态发生变化。 | `本轮生成后` 或 `本轮编辑后` |
| `api_a2ui_put` | 前端调用 PUT 接口直接保存 A2UI 页面。 | `页面保存` |
| `api_a2ui_restore_pre` | restore 前自动保存当前页面，防止回退后无法找回。 | `回退前自动保存` |

## 10. 前端推荐接入流程

### 10.1 打开或刷新页面

1. 获取当前页面：

```http
GET /api/threads/{thread_id}/a2ui
```

2. 用 `a2ui_json` 渲染画布。
3. 保存 `latest_checkpoint_id` 作为当前版本 ID。
4. 如果需要展示版本列表，再调用：

```http
GET /api/threads/{thread_id}/a2ui/checkpoints
```

### 10.2 发起一次 `/api/chat`

1. 调用 `/api/chat`。
2. 收到 `meta.base_checkpoint_id` 后，记录“本轮开始前版本”。
3. 正常处理 `text`、`thinking`、`step`、`a2ui` 事件。
4. 如果收到 `checkpoint` 事件，更新当前版本为 `checkpoint.checkpoint_id`。
5. 收到 `done.latest_checkpoint_id` 后，以它作为最终当前版本。

### 10.3 用户点击回退

1. 选择要恢复的 `checkpoint_id`。
2. 调用：

```http
POST /api/threads/{thread_id}/a2ui/checkpoints/{checkpoint_id}/restore
```

3. 用响应里的 `a2ui_json` 立即刷新画布。
4. 把当前版本 ID 更新为响应里的 `checkpoint_id`。
5. 后续继续调用 `/api/chat` 即可，后端已处理错误对话历史截断。

### 10.4 本轮误操作的快速撤销

如果用户刚发出一次编辑指令，模型还没结束或刚结束就发现页面改错：

- 优先使用本轮 `meta.base_checkpoint_id` restore。
- 如果 `base_checkpoint_id` 为 `null`，说明本轮之前没有 A2UI 页面，前端可以提示“暂无可回退版本”。

## 11. 常见问题

### 11.1 A2UI 快照 checkpoint 能和人机对话 checkpoint 共用同一个吗？

可以。当前实现就是同一个业务 checkpoint。

一个 checkpoint 里同时保存 A2UI JSON 和对话消息截断点，所以 restore 时可以让页面状态和下一轮 Agent 看到的对话历史一起回到同一时刻。

### 11.2 `/api/chat` 为什么要返回 checkpoint？

前端如果拿不到 `checkpoint_id`，就无法知道应该 restore 到哪个版本。

因此 `/api/chat` 现在通过：

- `meta.base_checkpoint_id` 告诉前端“本轮开始前版本”。
- `checkpoint.checkpoint_id` 告诉前端“本轮新版本”。
- `done.latest_checkpoint_id` 告诉前端“本轮结束后的当前版本”。

### 11.3 普通聊天会不会制造很多无意义 checkpoint？

不会。

普通聊天不会产生新的 `checkpoint` 事件。只要线程之前已有 A2UI checkpoint，`done.latest_checkpoint_id` 会继续返回当前版本，方便前端保持状态。

### 11.4 restore 后版本列表里为什么多了一个“回退前自动保存”？

这是为了防止用户回退错了之后无法找回。

restore 前，后端会保存当前页面为 `api_a2ui_restore_pre` checkpoint。用户如果想恢复到回退前状态，可以从 checkpoint 列表中选择这个版本。

### 11.5 前端需要处理 message_cutoff_id 吗？

不需要。

`message_cutoff_id` 是后端内部字段，用于截断对话历史。前端只需要传 `checkpoint_id`。

## 12. 联调检查清单

- `/api/chat` 第一条业务事件包含 `meta.base_checkpoint_id`。
- A2UI 生成或编辑完成后，有 `checkpoint` 事件。
- `done.latest_checkpoint_id` 和当前页面版本一致。
- 普通聊天没有 `checkpoint` 事件，但 `done.latest_checkpoint_id` 仍返回已有版本。
- `GET /api/threads/{thread_id}/a2ui` 返回 `latest_checkpoint_id`。
- `GET /api/threads/{thread_id}/a2ui/checkpoints` 可以看到版本列表。
- restore 后，响应里的 `a2ui_json` 是目标版本。
- restore 后，下一轮 `/api/chat` 不再带入 checkpoint 之后的错误对话。

