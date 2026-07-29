# OpenClaw 与 Hermes Agent 全面对比

> 本文档来源：Wiki 已有信息 + 原始使用文档整理
> 整理时间：2026-04-16
> 资料来源：
> - `/Users/machengqian.1/Documents/研究/wiki/synthesis/hermes-agent与openclaw对比.md`
> - `/Users/machengqian.1/Documents/研究/raw/sources/openclaw经验总结/`
> - `/Users/machengqian.1/Documents/研究/raw/sources/hermes-agent-docs/hermes-agent自我改进循环源码解读.md`

---

## 一、项目背景与定位

### 1.1 基本信息

| 维度 | Hermes Agent | OpenClaw |
|------|-------------|----------|
| **开发团队** | Nous Research | OpenClaw 社区 |
| **开源协议** | Apache 2.0 | MIT |
| **编程语言** | Python | 多语言混合（TypeScript 核心 + Python/Shell） |
| **代码规模** | ~50,000+ 行 Python | 轻量级，核心 ~10,000 行 |
| **定位** | 自我改进的通用 Agent（越用越强） | 终端原生 Agent + 灵活框架 |
| **核心差异** | 内置学习闭环，跨会话积累经验 | 注重工具生态、MCP 集成、节点控制 |

### 1.2 设计哲学

**Hermes Agent**：
> "让 Agent 随时间积累经验，越来越懂你和你的项目。"
- 默认情况下 Agent 是"无状态"的，每次对话独立
- Hermes 通过多子系统协作，让 Agent 跨越会话积累知识
- 强调**声明性知识**（MEMORY.md）vs **程序性知识**（Skills）的分层存储

**OpenClaw**：
> "灵活的 Agent 框架和运行时，注重工具生态和节点控制。"
- 强调框架的**可扩展性**和**模块化**
- 注重**多 Agent 协作**（subagent 编排）和**设备节点控制**
- 偏向**人工驱动**的记忆维护

---

## 二、架构对比

### 2.1 Hermes 架构

```
Entry Points → AIAgent（统一核心）
               ↓
  ┌───────────┼───────────┐
  ↓           ↓           ↓
Prompt    Provider    Tool
Builder   Resolution  Dispatch
  ↓           ↓           ↓
Compression  3 API      47 Tools
& Caching   Modes      37 Toolsets
```

**核心特点**：
- **单核心多入口**：AIAgent 是统一的同步编排引擎
- **Provider 运行时统一解析**：支持 3 种 API 调用模式
- **工具注册表模式**：工具自注册到工具集
- **渐进式压缩**：context 压缩 + token 缓存

**主要组件**：
| 组件 | 文件 | 功能 |
|------|------|------|
| AIAgent | run_agent.py (~9423行) | 核心循环 + Background Review |
| Memory Tool | memory_tool.py (~560行) | MemoryStore 持久化记忆 |
| Skill Manager | skill_manager_tool.py (~747行) | 技能自创建与更新 |
| Memory Manager | memory_manager.py (~366行) | 记忆系统编排器 |
| Context Compressor | context_compressor.py (~696行) | 对话压缩算法 |
| Session Search | session_search_tool.py (~505行) | 历史会话全文检索 |

### 2.2 OpenClaw 架构

```
Gateway / CLI → Agent (main session)
                    ↓
              Tools + Skills + Memory
                    ↓
              Multiple Providers
```

**核心特点**：
- **Gateway 统一路由**：所有消息通过 Gateway 分发
- **Session 隔离**：每个会话独立运行
- **Skills 系统**：基于 SKILL.md 的技能定义
- **Memory 分层**：MEMORY.md + memory/*.md 日记式存储

**主要组件**：
| 组件 | 功能 |
|------|------|
| Gateway | 消息路由、会话管理、配对授权 |
| Agent | 主会话执行、工具调用、技能加载 |
| Skills System | 技能注册、加载、触发 |
| Memory System | 长期记忆、会话历史 |
| MCP Bridge | Model Context Protocol 集成 |

---

## 三、记忆系统深度对比

### 3.1 Hermes 的四层记忆体系

Hermes 使用四个互补的存储机制：

| 存储 | 内容类型 | 容量 | 谁来写 | 怎么用 |
|------|---------|------|--------|--------|
| **MEMORY.md** | 环境事实、项目约定、工具特性、所学教训 | 2,200字符 | LLM主动写入 | 会话开始时注入系统提示 |
| **USER.md** | 用户偏好、沟通风格、工作习惯 | 1,375字符 | LLM主动写入 | 会话开始时注入系统提示 |
| **Skills/** | 具体任务的步骤、最佳实践、已验证方案 | 无限制 | LLM主动创建/更新 | 按需加载，slash命令触发 |
| **SQLite(state.db)** | 所有历史会话原文 | 无限 | 自动记录 | FTS5检索，按需摘要 |

**关键创新**：
- **原子写入**：防止进程崩溃导致文件损坏
- **§ 分隔符**：自然文本中极少出现，不干扰 frontmatter 解析
- **Nudge 机制**：周期性触发后台 review，主动沉淀知识
- **Honcho 集成**：可选的外部用户建模系统

### 3.2 OpenClaw 的记忆体系

| 存储 | 内容类型 | 容量 | 谁来写 | 怎么用 |
|------|---------|------|--------|--------|
| **MEMORY.md** | 长期事实和偏好 | 无硬性限制 | 人工维护 | 会话开始时读取 |
| **memory/*.md** | 日记式记忆 | 无限制 | 定期心跳写入 | 按日期组织 |
| **Session 历史** | 完整对话记录 | 无限 | 自动记录 | 可追溯查询 |

**关键特点**：
- **心跳机制**：定期检查并持久化记忆
- **多目录加载**：workspace/skills、~/.agents/skills、~/.openclaw/skills 等多路径
- **SKILL.md 结构**：必须包含 frontmatter 元数据 + Markdown 指令体
- **优先级规则**：workspace > ~/.agents/skills > ~/.openclaw/skills > bundled

### 3.3 核心差异

| 维度 | Hermes | OpenClaw |
|------|--------|----------|
| **自我改进** | ✓ 内置学习循环，Agent 自动创建 skills | ✗ 更偏向人工驱动 |
| **Nudge 机制** | ✓ 内置周期性主动沉淀 | 心跳机制（较被动）|
| **外部记忆插件** | 8个插件（Honcho、Mem0等） | 插件系统 |
| **会话搜索** | SQLite + FTS5 全文检索 | Session 历史追溯 |

---

## 四、技能系统对比

### 4.1 Hermes 技能系统

**格式**：SKILL.md（YAML frontmatter）

```markdown
---
name: skill-name
description: 一句话描述，用于触发判断
---
# Skill 主体
Markdown 指令...
```

**特点**：
- **自动创建**：Agent 可通过 skill_manage 工具自动创建
- **Hub 生态**：skills.sh、ClawHub、官方技能市场
- **渐进加载**：三级加载（list/view/ref）
- **条件激活**：fallback_for_toolsets 机制
- **自我改进**：Agent 可 patch 已有 skill

**触发机制**：
- Slash 命令触发（如 `/skill`）
- 工具集 fallback
- Agent 主动判断场景触发

### 4.2 OpenClaw 技能系统

**格式**：SKILL.md（YAML frontmatter）

```markdown
---
name: my_new_skill
description: 描述这个 skill 是做什么的，什么时候应该触发它。
metadata:
  {
    "openclaw": {
      "os": ["darwin"],
      "requires": { "bins": ["uv"] },
      "always": true,
      "emoji": "🖼️"
    }
  }
---
# My New Skill
...
```

**技能目录优先级**：

```
<workspace>/skills/           (最高)
    → <workspace>/.agents/skills/
    → ~/.agents/skills/
    → ~/.openclaw/skills/
    → bundled skills
    → skills.load.extraDirs   (最低)
```

**特点**：
- **多来源加载**：从多个目录扫描 skill
- **entries 注册**：必须在 openclaw.json 中声明才生效
- **热重载**：启用 watch 时文件变化自动重新加载
- **Snapshot 机制**：session 启动时快照 skill 列表

### 4.3 关键差异

| 维度 | Hermes | OpenClaw |
|------|--------|----------|
| **自创建能力** | ✓ 内置 skill_manage 工具 | ✗ 无内置，需手动创建 |
| **触发方式** | Slash + 工具集 fallback | description 文本匹配 |
| **配置注册** | 无需注册，文件系统即触发 | 必须在 entries 中声明 |
| **元数据丰富度** | 基础 YAML | 支持 OS/requires/always/emoji |

---

## 五、工具系统对比

### 5.1 Hermes 工具生态

**内置工具**：47 个工具，37 个工具集

| 类别 | 工具数 | 示例 |
|------|--------|------|
| 终端执行 | 6种后端 | local, docker, ssh, singularity, modal, daytona |
| 浏览器自动化 | 11个 | navigate, click, type, snapshot, vision 等 |
| 记忆工具 | 3个 | memory, user, session_search |
| 技能工具 | 2个 | skill_manage, skills_list |
| 文件操作 | 多个 | read_file, write_file, patch, terminal |
| 其他 | ... | cronjob, delegate_task, process 等 |

**终端后端**：

```
local → docker → ssh → singularity → modal → daytona
```

| 后端 | 用途 |
|------|------|
| local | 开发/信任任务 |
| docker | 安全隔离 |
| ssh | 远程沙箱 |
| singularity | HPC/无root |
| modal | 无服务器弹性 |
| daytona | 持久云开发 |

### 5.2 OpenClaw 工具生态

**基础工具集**：
- 终端执行
- Node 集成
- 浏览器自动化
- MCP 工具
- Skill 扩展

**MCP 支持**：
- 内置 Model Context Protocol 支持
- 可连接多种 MCP 服务器
- 工具动态发现

### 5.3 核心差异

| 维度 | Hermes | OpenClaw |
|------|--------|----------|
| **内置工具数** | 47个 | 基础工具集 |
| **终端后端多样性** | 6种（local/docker/ssh/singularity/modal/daytona） | 基础执行 |
| **MCP 支持** | ✓ 内置 | ✓ 支持 |
| **浏览器自动化** | 11个专用工具 | 基础能力 |
| **后台进程** | process 工具 | ✓ 支持 |

---

## 六、消息平台支持对比

### 6.1 Hermes 支持的 14+ 平台

| 平台 | 接入方式 | 特点 |
|------|---------|------|
| Telegram | Adapter | 官方推荐入门平台 |
| Discord | Adapter | Slash 命令支持 |
| Slack | Adapter | 频道/线程隔离 |
| WhatsApp | Adapter | - |
| Signal | Adapter | - |
| Matrix | Adapter | - |
| IRC | Adapter | - |
| ... | ... | 共 14+ 平台 |

**特点**：
- **Adapter 模式**：每平台独立 adapter
- **Allowlist 授权**：基于 pairing 的白名单
- **Slash 命令**：平台原生命令支持

### 6.2 OpenClaw 平台支持

- **多 channel 支持**：通过 Gateway 统一路由
- **配置式授权**：gateway.pairing 配置
- **设备节点控制**：手机通知、摄像头等 IoT 集成

### 6.3 核心差异

| 维度 | Hermes | OpenClaw |
|------|--------|----------|
| **平台数量** | 14+ | 多 channel |
| **接入方式** | Adapter 模式 | 统一 Gateway |
| **授权机制** | Allowlist + Pairing | 配置式 |
| **设备节点** | ✗ | ✓ 手机通知、摄像头等 |

---

## 七、SubAgent 与多 Agent 协作

### 7.1 Hermes 的 SubAgent 支持

- **ACP 服务器**：通过 `hermes acp` 启动
- **多模型支持**：可同时运行多个 LLM provider
- **任务委托**：通过 delegate_task 工具

### 7.2 OpenClaw 的 SubAgent 机制

**配置要求**：
```json
"agents": {
  "defaults": { ... },
  "list": []  // 必须注册主 agent
}
```

**运行时选项**：
| runtime | 说明 |
|---------|------|
| `"subagent"` | 需要 gateway pairing 才可用 |
| `"acp"` | 可并行启动多个 Coding Agent |

**问题排查**：
- `runtime="subagent"` 需要 gateway 完成配对
- `agents_list` 返回 `configured: false` = 主 agent 未注册
- 解决：在配置中添加 `agents.list` 条目

---

## 八、安装与配置对比

### 8.1 Hermes 安装

```bash
# 一键安装
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
hermes  # 60秒安装完成

# 配置向导
hermes setup
```

**配置文件**：
- `.env` 环境变量
- `config.yaml` 主配置
- `~/.hermes/` 数据目录

### 8.2 OpenClaw 安装

```bash
# Gateway 启动
openclaw gateway start

# 配置管理
openclaw config 系统
```

**配置文件**：`openclaw.json`

```json
{
  "skills": {
    "load": {
      "extraDirs": [
        "/path/to/custom-skills-folder"
      ],
      "watch": true
    },
    "entries": {
      "skill-name": { "enabled": true }
    }
  },
  "agents": {
    "defaults": { ... },
    "list": []
  }
}
```

---

## 九、适用场景对比

### 9.1 选择 Hermes 当：

✓ 需要**自我改进**的 Agent（越用越聪明）
- 内置学习闭环，Agent 自动创建和更新 skills
- Nudge 机制驱动主动持久化

✓ 需要**多消息平台**接入
- Telegram/Discord/Slack/WhatsApp 等 14+ 平台全覆盖

✓ 需要**强大的终端隔离**
- docker/ssh/modal/daytona 多后端支持
- 适合敏感任务隔离

✓ 需要**技能 Hub 生态**
- 社区共享技能市场
- 官方维护的优质 skills

✓ 研究用途
- RL 训练
- 批量处理轨迹导出
- SQLite 完整历史记录

### 9.2 选择 OpenClaw 当：

✓ 需要**MCP 深度集成**
- Model Context Protocol 原生支持
- 工具动态发现

✓ 需要**设备节点控制**
- 手机通知
- 摄像头等 IoT 集成

✓ 需要**多 Agent 协作**
- subagent 编排
- 并行处理多任务

✓ 偏好**轻量级框架**
- 代码规模小
- 定制灵活

✓ 需要**高度可定制**
- 自定义 skill 加载路径
- 灵活的 Gateway 配置

---

## 十、总结

### 10.1 核心差异速查

| 维度 | Hermes Agent | OpenClaw |
|------|-------------|----------|
| **学习能力** | ✓ 内置自我改进循环 | 人工驱动 |
| **工具丰富度** | 47工具 + 6种终端后端 | 基础工具集 |
| **平台支持** | 14+ 消息平台 | 多 channel |
| **SubAgent** | ACP 服务器 | subagent runtime（需 pairing）|
| **记忆系统** | 四层存储 + Nudge | MEMORY.md + 日记式 |
| **Skill 自创建** | ✓ 内置 | ✗ 手动 |
| **代码规模** | ~50,000 行 | 轻量级 |
| **设备节点** | ✗ | ✓ |

### 10.2 一句话总结

**Hermes Agent**：更像一个**完整的、面向生产的 Agent 产品**，有内建的学习循环、丰富的平台接入、强大的隔离执行环境，**越用越聪明**。

**OpenClaw**：更像一个**灵活的 Agent 框架和运行时**，注重工具生态、节点控制和 MCP 扩展，**高度可定制**。

### 10.3 未来趋势

| 方向 | Hermes | OpenClaw |
|------|--------|----------|
| 多模态 | 持续增强 | 待观察 |
| 自主学习 | 深化 Nudge 机制 | 待增强 |
| MCP 生态 | 跟进中 | 重点方向 |
| 多 Agent | ACP 生态扩展 | subagent 编排 |
