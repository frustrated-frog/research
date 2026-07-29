
# 自我总结

本次探索 skill 总结心得如下

1. 官方文档说明：![[file-20260413140240780.png]]
官方的说明清晰定义了建议优先放在我们自己工作空间的目录下，并且如果有其他agent新安装，那么也可以直接将workspace中的skill迁移过来

2. openclaw 源代码库里面的skills目录下放置了openclaw内置的所有技能，在他里面的public文件夹里面，放置了我自定义的skills，并且要加上配置：
![[file-20260413140240779.png]]

代表着要进行扫描，并且扫描这两个目录下的skills的优先级高于在工作空间创建的一切skills，所以如果遇到了重名的skill，一定是以先扫描到的为准
# OpenClaw Skills 加载机制与实战总结

> 来源：官方文档 + 实际操作经验总结
> 时间：2026-04-08

---

## 1. 核心概念：Skill 是什么

Skill 是一组告诉 Agent **何时使用**、**如何使用**某种工具的指令。每个 Skill 是一个目录，里面必须有 `SKILL.md` 文件，包含 YAML frontmatter 元数据 + Markdown 指令体。

Skill 与工具的区别：
- **工具（Tool）**：Agent 能执行的具体动作（如 `exec`、`browser`）
- **Skill**：教会 Agent 在什么场景下用哪个工具、以及怎么用

---

## 2. Skill 的目录位置与优先级

### 2.1 官方规定的 Skill 目录

| 位置 | 优先级 | 作用域 |
|------|--------|--------|
| `<workspace>/skills/` | 最高 | 单个 Agent 工作区 |
| `<workspace>/.agents/skills/` | 高 | 单个工作区 Agent |
| `~/.agents/skills/` | 中 | 跨工作区共享 |
| `~/.openclaw/skills/` | 中 | 所有 Agent 共享 |
| `skills.load.extraDirs` 配置路径 | 低 | 自定义共享文件夹 |
| OpenClaw 内置 Bundled Skills | 最低 | 全局内置 |

### 2.2 我们的环境现状

```
~/.openclaw/skills/         ← 原来只有这个（但没有 skill 目录）
~/.openclaw/openclaw.json   ← skills.entries 在这里配置

# 我们后来添加的路径：
skills.load.extraDirs:
  - /Users/machengqian.1/code/openclaw/skills/public    ← 我们创建 skill 的地方
  - /Users/machengqian.1/code/openclaw/skills           ← 内置 skills 所在

skills.entries:             ← 每个 entry 对应一个 skill
  code-study-write-doc: { enabled: true }
  knowledge-wiki: { enabled: true }
  1password: { enabled: true }
  obsidian: { enabled: true }
```

### 2.3 优先级规则

当同一 skill 名称出现在多个位置时，**优先级高的覆盖优先级低的**：

```
<workspace>/skills> (最高)
    → <workspace>/.agents/skills
    → ~/.agents/skills
    → ~/.openclaw/skills
    → bundled skills
    → skills.load.extraDirs (最低)
```

---

## 3. Skill 文件格式

### 3.1 必须包含的内容

```markdown
---
name: skill-name           # 唯一标识符，snake_case
description: 一句话描述，显示给 Agent，用于决定何时触发此 skill
---

# Skill 主体

Markdown 指令，教导 Agent 何时、如何使用。
```

### 3.2 可选的 frontmatter 字段

```markdown
---
name: image-lab
description: 生成或编辑图片的工作流
metadata:
  {
    "openclaw": {
      "os": ["darwin"],                         # 仅在 macOS 加载
      "requires": { "bins": ["uv"] },           # PATH 上必须有这些命令
      "always": true,                           # 跳过所有过滤器
      "emoji": "🖼️",                            # UI 显示用
      "homepage": "https://example.com"         # UI "Website" 链接
    }
  }
---
```

### 3.3 关键规则

- **description 是主要触发机制**：OpenClaw 的 Agent 根据 description 决定是否触发该 skill
- frontmatter 中的 `metadata` 必须是单行 JSON 对象
- `name` 必须是 snake_case（不能用空格或特殊字符）

---

## 4. Config 配置详解

### 4.1 `skills.entries`

每个启用的 skill 都需要在这里声明：

```json
{
  "skills": {
    "entries": {
      "skill-name": {          // 匹配 SKILL.md 中的 name 字段
        "enabled": true         // true = 启用，false = 禁用
      }
    }
  }
}
```

**只有在这里声明的 skill 才会在 UI 中显示！**

即使 skill 目录放在正确的路径下，如果 `entries` 里没有它，**UI 也不会显示**。

### 4.2 `skills.load.extraDirs`

额外的 skill 搜索路径：

```json
{
  "skills": {
    "load": {
      "extraDirs": [
        "/path/to/custom-skills-folder"
      ]
    }
  }
}
```

这对于想要统一管理 skill、但 skill 目录不在默认路径下时非常有用。

### 4.3 `skills.load.watch`

是否监听 skill 文件变化（热重载）：

```json
{
  "skills": {
    "load": {
      "watch": true,              // 默认 true，文件变化时自动重新加载
      "watchDebounceMs": 250      // 防抖窗口（毫秒）
    }
  }
}
```

---

## 5. 实战总结：我们遇到的问题及解决

### 5.1 问题一：skill 目录放对了位置但 UI 不显示

**现象**：把 skill 目录放到 `/Users/machengqian.1/code/openclaw/skills/public/` 下，但 web UI 的 skill 列表里看不到。

**原因**：`openclaw.json` 中 `skills.entries` 没有声明这个 skill 名称。

**解决**：在 `skills.entries` 中添加：

```json
{
  "skills": {
    "entries": {
      "code-study-write-doc": { "enabled": true }
    }
  }
}
```

### 5.2 问题二：Gateway 重启不生效，双进程问题

**现象**：修改了 `openclaw.json`，但 Gateway 没有重新加载配置。

**原因**：Gateway 进程没有完全重启，或者有多个旧进程残留。

**解决**：

```bash
# 方法一：用 gateway 工具重启
→ 调用 gateway 工具的 restart action

# 方法二：彻底杀死所有旧进程后重启
ps aux | grep openclaw-gateway  # 查看所有进程
kill -9 <pid1> <pid2>           # 强制杀死
/Users/machengqian.1/code/openclaw/scripts/openclaw.sh gateway start  # 重启
```

### 5.3 问题三：skill 在多个目录有副本，不知道哪个生效

**现象**：最初 `knowledge-wiki` 同时存在于两个目录：
- `/Users/machengqian.1/code/openclaw/skills/knowledge-wiki/`
- `/Users/machengqian.1/code/openclaw/skills/public/knowledge-wiki/`

**原因**：skill-creator 脚本在 `skills/public/` 初始化了一个，同时原来 `skills/` 下也有一个（更早之前创建的）。

**解决**：统一放到 `skills/public/` 下（因为这是 `extraDirs` 中明确指定的路径）。

### 5.4 问题四：skill 目录权限问题

**现象**：Gateway 运行用户（`machengqian.1`）无法读取 skill 目录。

**排查**：用 `ls -la` 检查目录权限。

**解决**：确保 skill 目录及其父目录对运行用户可读。

---

## 6. 完整操作流程

### 6.1 创建新 Skill 的标准步骤

**Step 1：创建 skill 目录**

```bash
mkdir -p /Users/machengqian.1/code/openclaw/skills/public/my-new-skill
```

**Step 2：编写 SKILL.md**

```markdown
---
name: my_new_skill
description: 描述这个 skill 是做什么的，什么时候应该触发它。
---

# My New Skill

当用户要求 XXX 时使用本 skill。

## 工作流程

1. ...
2. ...
```

**Step 3：注册到 openclaw.json**

在 `skills.entries` 下添加：

```json
{
  "skills": {
    "entries": {
      "my_new_skill": { "enabled": true }
    }
  }
}
```

**Step 4：重启 Gateway 使配置生效**

通过 gateway 工具重启，或者彻底杀死进程后重启。

**Step 5：验证**

```bash
openclaw skills list   # 验证 skill 是否被加载
```

### 6.2 使用 init_skill.py 初始化（推荐）

OpenClaw 提供了脚本快速创建标准 skill 结构：

```bash
python3 /Users/machengqian.1/code/openclaw/skills/skill-creator/scripts/init_skill.py \
  my-new-skill \
  --path /Users/machengqian.1/code/openclaw/skills/public \
  --resources references,scripts
```

会自动创建：
- `my-new-skill/SKILL.md`（含 frontmatter 模板）
- `my-new-skill/references/`
- `my-new-skill/scripts/`

---

## 7. Session 快照机制

OpenClaw 在 session 启动时会对所有 eligible skills 做快照，**当前 session 内的所有对话都复用同一个 skill 列表**。

这意味着：
- 新增或修改 skill 不会影响当前 session
- 需要开启新 session（`/new`）或重启 Gateway 才能看到变化

**但**：如果启用了 `skills.load.watch`，skill 文件的修改会在下一个 agent turn 自动生效（热重载）。

---

## 8. Skill 在 Prompt 中的注入

当 skill 满足以下条件时，会被注入到 system prompt：

1. `skills.entries` 中 `enabled: true`
2. 所有 `metadata.openclaw.requires` 条件满足（OS、PATH 命令、环境变量等）
3. 没有被 `skills.allowBundled` 阻止

注入格式（XML 列表）：

```
<skills>
  <skill>
    <name>skill-name</name>
    <description>描述</description>
    <location>skills/my-new-skill</location>
  </skill>
</skills>
```

Token 消耗估算：
- 基础开销：195 字符
- 每条 skill：~97 字符 + name/description/location 长度

---

## 9. 常见问题速查

| 问题 | 原因 | 解决 |
|------|------|------|
| UI 看不到 skill | `skills.entries` 没有声明 | 添加 `"skill-name": { enabled: true }` |
| skill 放哪个目录 | 看用途和共享范围 | `skills/public` 用于共享，`workspace/skills` 用于单 Agent |
| 修改后不生效 | Gateway 没重启或没开新 session | 重启 Gateway 或 `/new` |
| Gateway 重启不生效 | 有多个旧进程残留 | `kill -9` 所有进程后重启 |
| 权限报错 | 目录不可读 | `ls -la` 检查并修复权限 |
| 依赖命令不存在 | `metadata.requires.bins` 未满足 | 确保命令在 PATH 中 |

---

## 10. 我们最终的配置

```json
{
  "skills": {
    "load": {
      "extraDirs": [
        "/Users/machengqian.1/code/openclaw/skills/public",
        "/Users/machengqian.1/code/openclaw/skills"
      ]
    },
    "entries": {
      "1password": { "enabled": true },
      "obsidian": { "enabled": true },
      "openai-whisper": { "enabled": false },
      "code-study-write-doc": { "enabled": true },
      "knowledge-wiki": { "enabled": true }
    }
  }
}
```

**最终效果**：
- `code-study-write-doc`：在 `skills/public/` 下，新增 skill
- `knowledge-wiki`：在 `skills/public/` 下，修复后的 skill
- 两者都在 web UI 的 skill 列表中可见


# OpenClaw Skill 配置与排查问答整理

## 1. 背景

你在 OpenClaw 中配置了自定义 skill，并观察到一个现象：

- skill 明明已经复制到 `workspace` 目录下
    
- 但在技能列表中显示的 `Source` 仍然不是 `workspace`
    
- 后来又发现 `workspace/skills` 下放的是 `knowledge-wiki.md`，而不是 skill 目录结构
    

围绕这个问题，逐步确认了 OpenClaw 的 skill 加载机制、目录要求、来源显示逻辑，以及排障方法。

---

## 2. skill 的来源为什么可能不是 workspace

OpenClaw 的 skill 并不是只从 `workspace` 一个地方加载，而是可能从多个来源扫描：

1. 内置 bundled skills
    
2. `extraDirs` 中配置的目录
    
3. `workspace/skills` 目录
    

因此，即使你把某个 skill 复制到了 `workspace` 下，如果 OpenClaw 在前面的来源中已经找到同名 skill，它仍然会优先使用前面的那个来源。

### 结论

`Source` 显示的是 **实际加载来源**，不是“文件最终所在位置”。

---

## 3. 为什么删除 extraDirs 后，workspace 里的 skill 还没显示

你后来把 `extraDirs` 里的那个 skill 删除了，并把 skill 放到了 `workspace/skills` 下，但列表里仍然没有出现。这个时候重点排查以下几件事：

- skill 是否真的放在了正确的目录下
    
- skill 是否有正确的文件结构
    
- OpenClaw 是否重新加载了 skill 索引
    
- skill 的文件名是否符合规范
    

---

## 4. 关键问题：你放的是 `.md` 文件，而不是 skill 目录

你执行 `ls ~/.openclaw/workspace/skills` 后看到的是：

```text
knowledge-wiki.md
```

这说明当前结构是一个 **单独的 Markdown 文件**，而不是一个 skill 目录。

### OpenClaw 识别 skill 的标准结构（重要）

OpenClaw 需要的是：

```text
~/.openclaw/workspace/skills/
└── knowledge-wiki/
    ├── SKILL.md
    ├── scripts/
    └── references/
```

也就是说：

- skill 必须是一个目录
    
- 目录中必须有 `SKILL.md`
    
- 其他文件夹如 `scripts/`、`references/` 是可选的
    

**不能被识别的情况**

下面这种结构通常不会被当作 skill：

```text
~/.openclaw/workspace/skills/
└── knowledge-wiki.md
```

因为它不是目录，也不是 `SKILL.md`。

---

## 5. 正确的修复方式

### 第一步：创建 skill 目录

```bash
mkdir -p ~/.openclaw/workspace/skills/knowledge-wiki
```

### 第二步：把 Markdown 文件改成 `SKILL.md`

如果你原来有：

```text
~/.openclaw/workspace/skills/knowledge-wiki.md
```

需要移动并改名为：

```text
~/.openclaw/workspace/skills/knowledge-wiki/SKILL.md
```

命令示例：

```bash
mv ~/.openclaw/workspace/skills/knowledge-wiki.md ~/.openclaw/workspace/skills/knowledge-wiki/SKILL.md
```

### 第三步：确认目录结构

执行：

```bash
ls ~/.openclaw/workspace/skills/knowledge-wiki
```

应该至少看到：

```text
SKILL.md
```

### 第四步：重启 OpenClaw

让 OpenClaw 重新扫描 skills：

```bash
pnpm openclaw gateway run
```

---

## 6. 为什么官方更推荐目录结构

OpenClaw 的 skill 设计本身就是围绕“一个 skill 一个目录”来组织的。

这种结构的好处是：

- 便于放说明文档 `SKILL.md`
    
- 便于放脚本 `scripts/`
    
- 便于放参考资料 `references/`
    
- 便于后续扩展和维护
    

如果只有单独一个 `.md` 文件：

- OpenClaw 很难把它识别为完整 skill
    
- 也不便于后续加入脚本和辅助文件
    

---

## 7. skill 加载来源和 workspace 的关系

你之前遇到过一个常见误区：

> “我已经把 skill 放到 workspace 了，为什么来源还是不是 workspace？”

原因通常有两个：

### 7.1 之前在别的目录已经有同名 skill

如果 `extraDirs` 或内置目录中已经存在同名 skill，OpenClaw 可能优先加载那个版本。

### 7.2 workspace 下的结构不符合 skill 规范

比如：

- 只有 `.md` 文件，没有目录
    
- 没有 `SKILL.md`
    
- 多套了一层目录
    

这些都会导致它没有被识别。

---

## 8. 推荐的最终目录结构

你可以把 OpenClaw 的 skill 和工作区结构整理成下面这样：

```text
~/.openclaw/
├── openclaw.json
└── workspace/
    ├── skills/
    │   └── knowledge-wiki/
    │       ├── SKILL.md
    │       ├── scripts/
    │       └── references/
    ├── vault/
    │   └── obsidian/
    └── wiki/
```

这个结构的优点：

- skill 放在 workspace 中，便于统一管理
    
- Obsidian vault 和知识库目录分离
    
- 后续可以继续增加更多 skill
    

---

## 9. 排查 skill 不显示的通用步骤

如果以后再遇到 skill 不显示，可以按下面顺序检查：

9.1 看目录是否正确

确认 skill 是不是一个目录，而不是一个 `.md` 文件。

9.2 看文件名是否正确

必须是 `SKILL.md`。

9.3 看是否有重复来源

检查 `extraDirs` 和 bundled skills 里是否也存在同名 skill。

9.4 看配置是否还指向旧目录

确认 `openclaw.json` 里是否还有旧的 `extraDirs`。

9.5 重启服务

修改目录或配置后，需要重启 OpenClaw 才会重新扫描。

9.6 查看技能列表

通过技能列表确认 skill 是否被加载。

---

## 10. 这次问题的最终结论

这次 skill 没有显示，核心原因是：

- 你放进去的是 `knowledge-wiki.md`
    
- OpenClaw 需要的是 `knowledge-wiki/SKILL.md`
    
- skill 必须以目录形式存在
    
- 仅有 Markdown 文件不会被识别为完整 skill
    

### 正确做法

把：

```text
~/.openclaw/workspace/skills/knowledge-wiki.md
```

改成：

```text
~/.openclaw/workspace/skills/knowledge-wiki/SKILL.md
```

然后重启 OpenClaw 即可。



