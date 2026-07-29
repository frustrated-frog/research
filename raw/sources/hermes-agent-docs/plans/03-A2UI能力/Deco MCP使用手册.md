# Deco MCP 使用手册

本文档整理自 Joyspace 页面《Deco MCP（Beta）使用手册》，并补充本项目 `/api/chat` 接入 Deco MCP 时的使用约定。用于后续排查、测试和接入 A2UI 生成链路时快速查看。

原始文档地址：<https://joyspace.jd.com/pages/t8fYI98GQurPV72X9tOI>

## 1. Deco MCP 是什么

Deco MCP 是 Relay 官方提供的设计稿转代码能力，核心目标是把 Relay 设计稿中的图层或在线会话结果交给大模型使用，让大模型先拿到设计上下文，再生成前端页面或 A2UI 页面。

MCP 是 Model Context Protocol。对大模型来说，MCP 工具就是一组可调用的外部能力。Deco MCP 目前主要暴露两类工具：

| 工具 | 用途 | 典型场景 |
| --- | --- | --- |
| `getCode` | 获取 Relay 选中图层的设计稿生成代码 | 从某个设计稿图层生成 Taro、React、Vue 代码，再让大模型继续生成页面 |
| `getOnlineCode` | 获取在线会话修改后的代码文件 | 在线工具中已经有代码会话，需要把某个版本的代码取回来 |

## 2. 最重要的使用原则

Deco MCP 不支持直接把 Relay 普通网页 URL 当入参。

必须先在 Relay 页面中按步骤复制标准 MCP 请求索引，再把索引交给大模型或 `/api/chat`。如果直接输入类似 `https://relay.jd.com/file/design?...` 的普通网页地址，MCP 工具无法知道要读取哪个图层或哪个在线会话。

标准请求索引常见有两类：

```text
Relay://Deco?generateId=1971482321706033153&inspectRatio=1&inspectType=react-taro&inspectUnit=px
```

```text
//UI2Code?generateId=2008188023988027393&inspectRatio=1&rect=0_0_375_812&styleUnit=px&type=vue
```

在线会话索引示例：

```text
Relay://chatId=1983449749704953858&version=1
```

历史资料里可能出现 `chatld` 这种拼写，接入侧可以做兼容，但新用例建议统一使用 `chatId`。

## 3. Token 获取和有效期

Deco MCP 需要个人 Token 认证。通用流程如下：

1. 访问并登录 Relay。
2. 打开浏览器开发者工具。
3. 在控制台粘贴并执行官方文档提供的 Token 获取脚本。
4. 从控制台输出中复制 Token。
5. 妥善保存 Token，不要写入公开文档、截图或代码仓库。

注意事项：

- Chrome 控制台如果禁止粘贴，需要先输入 `allow pasting` 后再粘贴脚本。
- 新版 Token 有效期为 90 天。
- 如果出现登录过期或认证失败，优先重新获取并配置 Token。
- 当前 Deco MCP 只需要个人 Token 校验，不需要额外配置 ERP Cookie。

## 4. 本项目中的 Token 传入方式

本项目 `/api/chat` 按需注册 Deco MCP 服务。`DECO_MCP_TOKEN` 的唯一来源是用户输入，不从 `.env`、进程环境变量、Hermes memory 或其它持久化配置里读取。

允许的来源只有三种：

1. 本轮请求体里的 `secrets.DECO_MCP_TOKEN`。
2. 本轮用户消息里直接输入的 `DECO_MCP_TOKEN=...` 或 `DECO_MCP_TOKEN是...`。
3. 同一 thread 内由上述用户输入产生的短期临时缓存。

拿到用户输入的 Token 后，服务端会把 Token 拼到 Deco MCP SSE URL 的 `token` 查询参数里。Token 仍然只来自用户输入，不允许固化到代码、文档或持久化配置中。

本项目当前使用的 Deco MCP SSE 地址格式：

```text
http://mcp-gateway.jd.com/mcp/deco-mcp-server-prod/sse?token=用户本轮输入的DECO_MCP_TOKEN
```

等价请求配置示意：

```json
{
  "deco-prod-mcp": {
    "url": "http://mcp-gateway.jd.com/mcp/deco-mcp-server-prod/sse?token=用户本轮输入的DECO_MCP_TOKEN",
    "transportType": "sse",
    "tools": {
      "include": ["getCode", "getOnlineCode"]
    }
  }
}
```

在 Hermes `/api/chat` 中，推荐通过请求体 `secrets.DECO_MCP_TOKEN` 提供 Token。不要把真实 Token 固化到代码、测试数据、文档、环境变量或 memory 里。

## 5. getCode 使用方式

`getCode` 用于读取 Relay 设计稿中选中图层的生成代码。

操作步骤：

1. 打开 Relay 设计稿。
2. 进入开发模式。
3. 选中需要生成代码的图层。
4. 在右侧操作面板复制图层信息或 MCP 请求索引。
5. 把复制到的标准索引交给大模型。

推荐提示词：

```text
使用 Deco getCode 工具获取设计稿代码 //UI2Code?generateId=2008188023988027393&inspectRatio=1&rect=0_0_375_812&styleUnit=px&type=vue 获取到代码后根据代码生成页面
```

或：

```text
使用 getCode 工具帮我从设计稿 Relay://Deco?generateId=1971482321706033153&inspectRatio=1&inspectType=react-taro&inspectUnit=px 获取代码，并根据代码生成 A2UI 页面
```

在本项目中，模型应优先调用：

```text
mcp_deco_prod_mcp_getCode
```

## 6. getOnlineCode 使用方式

`getOnlineCode` 用于读取在线会话中的代码文件，适合已经在在线工具中修改过代码、并希望大模型拿到某个版本代码继续处理的场景。

操作步骤：

1. 进入 Deco 在线工具页面。
2. 点击右上角「本地开发」。
3. 复制会话信息。
4. 将会话索引交给大模型。

推荐提示词：

```text
通过 getOnlineCode 工具获取代码文件内容 Relay://chatId=1983449749704953858&version=1，并根据代码生成页面
```

在本项目中，模型应优先调用：

```text
mcp_deco_prod_mcp_getOnlineCode
```

## 7. `/api/chat` 调用建议

真实调用 `/api/chat` 时，建议把设计稿索引和页面生成诉求写在 `message` 中，把 Token 放在 `secrets` 中。

示例：

```json
{
  "message": "使用 Deco getCode 工具获取设计稿代码 //UI2Code?generateId=2008188023988027393&inspectRatio=1&rect=0_0_375_812&styleUnit=px&type=vue 获取到代码后根据代码生成页面",
  "thread_id": "deco-demo-thread",
  "secrets": {
    "DECO_MCP_TOKEN": "不要在文档里填写真实 Token"
  }
}
```

预期链路：

1. `/api/chat` 检测到标准 Relay/Deco 索引。
2. `/api/chat` 使用 `DECO_MCP_TOKEN` 注册 `deco-prod-mcp`。
3. 大模型加载 `jd-a2ui-generate` skill。
4. 大模型调用 `mcp_deco_prod_mcp_getCode` 或 `mcp_deco_prod_mcp_getOnlineCode`。
5. 大模型根据 Deco 返回的代码上下文生成 A2UI JSON。
6. 前端收到 `step`、`text`、`a2ui`、`done` 等 SSE 事件。

## 8. 常见问题

### 8.1 输入 Relay 网页 URL 后无法获取代码

原因通常是入参不符合 Deco MCP 要求。需要复制标准 MCP 请求索引，而不是直接输入 Relay 网页 URL。

### 8.2 返回 `RESOURCE_EXHAUSTED`

一般表示背后大模型资源紧张。稍后重试即可。

### 8.3 返回登录过期或认证失败

优先检查 Token 是否过期。新版 Token 有效期为 90 天，过期后需要重新获取并配置。

在本项目中还要确认 Token 是否通过 SSE URL 的 `token` 查询参数传递。如果误放到普通 Bearer Header 或其它 Header 中，可能会被网关识别为登录态异常。

### 8.4 MCP 正常启动，但模型没有调用工具

可能原因：

- 提示词没有包含标准 Relay/Deco 索引。
- 当前模型对 MCP 工具调用支持不好。
- 当前模型上下文窗口太小，没有正确理解工具描述。
- skill 没有加载，导致模型不知道应先读取 Relay/Deco 设计稿。

建议换支持工具调用更稳定的模型，并明确要求使用 `Deco getCode` 或 `Deco getOnlineCode`。

### 8.5 原 Relay MCP 是否还要使用

官方文档说明后续统一到 Deco MCP。原 Relay MCP 不再维护，新接入和新测试优先使用 Deco MCP。

## 9. 快速检查清单

开始排查前，按下面顺序确认：

- 已登录 Relay，并能正常打开设计稿或在线工具。
- Token 是最新获取的 90 天有效 Token。
- Token 没有写进文档、代码或公开日志。
- 本轮请求或当前 thread 临时缓存中已经有用户输入的 `DECO_MCP_TOKEN`。
- 请求发送的是标准 MCP 索引，不是 Relay 普通网页 URL。
- `getCode` 用于设计稿图层，`getOnlineCode` 用于在线会话代码。
- Hermes 注册 Deco MCP 时使用 SSE URL `?token=...` 查询参数。
- A2UI 生成前已加载 `jd-a2ui-generate` skill。
