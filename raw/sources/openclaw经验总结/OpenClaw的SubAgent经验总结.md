## 为什么subagent不可使用？

**openclaw答：**

问题找到了，核心原因是：**`runtime="subagent"` 在当前配置下被策略限制，提示 "pairing required"**。

具体分析：

1. **调用方式**：我用 `sessions_spawn` 且 `runtime="subagent"`
2. **错误信息**：后台报错是 `No available agent provider (No subagent provider found, or all subagents are disabled, or agent "subagent" is unknown)`
3. **策略限制**：查看 Tool 定义时注意到这句话：

> *"If pairing is required (gateway.pairing.required) but the gateway is not paired, spawning is not allowed."*

**也就是说，`runtime="subagent"` 需要 gateway 完成配对（pairing）才能使用**，而你的 gateway 虽然在运行，但没有被配对过。

---

### 两个解决方向

#### 方向 A：用 `runtime="acp"` 替代

如果想并行处理多个主题，可以用 `runtime="acp"` 启动一个 ACP harness（比如 Codex/Claude Code），不受 pairing 限制：

```
sessions_spawn(runtime="acp", agentId="codex", ...)
```

这样可以并行开多个 Coding Agent 线程来同时处理多个文档。

#### 方向 B：配置 gateway pairing

给 gateway 配置一个 pairing，让 `runtime="subagent"` 可用。这需要设置 `gateway.pairing` 相关字段。

---

你想走哪个方向？如果 A 更简单直接，我可以现在就试着用 ACP runtime 并行处理剩余的 13 个主题。

现在找到根本原因了。`agents_list` 返回结果：

```json
{
  "agents": [{ "id": "main", "configured": false }]
}
```

**`configured: false` = 主 agent 没有在 `agents.list[]` 中配置**，所以 sub-agent runtime 找不到有效的 agent 配置来启动会话。

---

## 解决步骤

需要在配置中添加 `agents.list` 条目来"注册"主 agent。看配置，当前 `agents.list` 是空的（因为 config 里只有 `agents.defaults`）：

```json
"agents": {
  "defaults": { ... },
  "list": []  // ← 空，main agent 未注册
}
```

我来用 `config.patch` 添加主 agent 条目：

