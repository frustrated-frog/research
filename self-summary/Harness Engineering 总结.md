
# ClaudeCode 做 harness

1. **要想理解 Claude Code，首先要理解 Agent 架构的三个代际演进：**

- 第一代是 Chatbot，无状态问答；
    
- 第二代是 Workflow，用 n8n、LangChain 这类工具把 LLM 嵌进代码驱动的 DAG 流里，代码决定模型下一步做什么；
    
- 第三代是 Autonomous Agent，模型控制循环，运行时只是执行器。
    

Claude Code，就是属于第三代的商业化产品。

2. **TAOR Loop 设计**

Claude Code 的执行引擎是一个叫 TAOR 的循环：Think-Act-Observe-Repeat。

运行时越笨，架构越稳定。把智能下沉到模型，把确定性留给框架。

**随着模型变得更强，脚手架应该变薄，而不是变厚。硬编码的脚手架应该随着模型能力提升而被主动删除，架构随时间推移越来越薄。如果你每次模型升级都要往框架里加更多脚手架，说明你在对抗模型，而不是利用模型。

3. **Context Window 是稀缺资源，不是越大越好**

**第一层是 Auto-Compaction**。当 Context 使用量达到约 50% 时自动触发，用 LLM 摘要替换原始对话轮次，释放空间的同时保留关键决策。这不是简单地截断历史，而是用摘要压缩，确保重要信息不丢失。这个机制对应的故障模式叫做 Context Collapse，解决方案是：Auto-compaction at ～50% + sub-agents with isolated context windows。

**第二层是 Sub-Agent 隔离**。把重型的探索、研究任务卸载给独立的子 Agent。子 Agent 运行自己独立的 TAOR 循环，有自己的 Context 预算，任务完成后只把摘要返回给主 Agent。这样，无论子任务消耗了多少 token，主 Agent 的 Context 都不会被污染。

子 Agent 运行时：有自己的 maxTurns 上限、有自己的 compaction 机制（独立压缩，不影响主对话）、有自己的 MEMORY.md。主 Agent 派出子 Agent 之后，只等一个 summary 回来，整个子任务的 token 消耗对主 Context 完全透明隔离。

**第三层是 Prompt Cache 经济学。** promptCacheBreakDetection.ts 里追踪了 14 个 cache-break 向量，也就是 14 种会让 prompt 缓存失效的情况。代码里还有一个函数叫 DANGEROUS_uncachedSystemPromptSection()，代表着这里加东西要小心，会破坏缓存。代码里还有多个 sticky latches，防止模式切换破坏 prompt 缓存的锁定机制。

4. **记忆系统的核心是索引，不是存储**

核心设计原则是：**记忆是索引，不是存储。能从代码库中重新推导出的信息，绝不应该被存储。**

从架构上看，Claude Code 的记忆系统分为六层，在每次会话启动时按层加载：

- **Managed Policy**（组织级策略）：企业或团队层面的统一规范
    
- **Project CLAUDE.md**（项目配置）：当前项目的特定指令和上下文
    
- **User Preferences**（用户偏好）：个人层面的习惯和偏好设置
    
- **Auto-Memory**（自动学习模式）：Agent 从历史交互中学到的用户模式
    
- **Session**（会话上下文）：当前会话的临时信息
    
- **Sub-Agent Memory**（子 Agent 记忆）：各子 Agent 独立维护的专项记忆
    

其中，Auto-Memory 循环甚至允许 Agent 学习用户的工作模式，并把这些模式写入 MEMORY.md 供未来会话使用。用户不需要反复解释相同的事情，Agent 会从之前的交互里学习并记住重要信息。

Claude Code 的子 Agent 记忆机制也值得一提。在自定义子 Agent 的配置里，可以设置 memory: user，Agent 会把学到的模式写入 ~/.claude/agent-memory/<name>/MEMORY.md，下次调用时自动加载前 200 行。这意味着每个子 Agent 都可以有自己独立的、持续积累的专项记忆。

这个系统具有主动自我编辑能力。它不仅会记录，还会重写、去重、甚至剪除互相矛盾的信息，过期且无效的记忆在这里被视为「负债」而非资产。

Claude Code 的记忆系统设计，也侧面反映了：在产品层面，记忆不只是一个 Feature，它是决定用户是否继续使用的核心留存机制，因为用户真正期待的是一个「会学习」的 Agent。

5. 权限系统的设计更像是 UX 设计，信任是可组合的

Claude Code 的权限系统被设计为一个五档的信任光谱：

- plan：只读，完全不能写入，信任级别最低
    
- default：编辑和 shell 操作前都需要询问，标准模式
    
- acceptEdits：自动批准文件编辑，shell 操作仍需询问，中等信任
    
- dontAsk：自动批准白名单内的所有操作，高信任
    
- bypassPermissions：跳过所有检查，仅限托管组织使用，最高信任

每个工具调用都经过静态分析层的多层白名单校验。bashSecurity.ts 里有 23 项编号的安全检查，包括：

- 18 个被阻止的 Zsh 内置命令
    
- 防御 Zsh equals expansion：=curl 这种写法可以绕过对 curl 的权限检查
    
- unicode 零宽字符注入
    
- IFS null-byte 注入
    
- 一个在 HackerOne 审查期间发现的恶意 token 绕过

5.1 更巧妙的设计

在 system.ts 文件里，每个 API 请求都包含一个 cch=00000 占位符。在请求真正离开进程之前，Bun 的原生 HTTP 栈（用 Zig 编写，运行在 JavaScript 运行时之下）会把这五个零替换成一个计算出的哈希值。服务端会验证这个哈希，确认请求来自真实的 Claude Code 二进制文件，而不是第三方伪造的客户端。

之所以用等长的占位符，是为了让替换不改变 Content-Length 头部，也不需要缓冲区重新分配，这是一个很细节的工程考量。整个计算过程发生在 JS 层之下，对运行在 JS 里的任何代码都完全不可见。本质上是在 HTTP 传输层实现的 API 调用 DRM。

6. 多 Agent 编排，从子 Agent 到 Agent Teams

Claude Code 的多 Agent 编排采用了横向扩展的方式，分为两层。

**第一层：Sub-Agent**

子 Agent 以独立进程方式运行，有自己的 TAOR 循环、自己的 Context 预算、自己的 maxTurns 上限、自己的记忆。任务完成后，只把摘要返回给主 Agent，主 Agent 的 Context 完全不受影响。

Claude Code 内置了三种预设子 Agent，各有分工：

- **Explore**：用 Haiku 模型（速度快、成本低），只有只读工具（Read、Grep、Glob），专门做文件发现和代码库探索
    
- **Plan**：继承主 Agent 的模型，只有只读工具，专门做代码库研究和规划前的信息收集
    
- **General-purpose**：继承主 Agent 的模型，配备全套工具，处理复杂的多步骤操作
    

自定义子 Agent 通过 .md 文件加 YAML frontmatter 定义，可以指定模型（sonnet/opus/haiku/inherit）、权限模式、maxTurns、可用工具白名单、禁用工具黑名单，甚至可以预加载特定的 Skills。存储位置有三种：~/.claude/agents/（用户级）、.claude/agents/（项目级），或通过 --agents CLI 参数指定。

子 Agent 还支持前台和后台两种执行模式。前台模式会阻塞主对话，权限询问和问题会透传给用户；后台模式则在用户继续工作的同时并发运行，权限在启动前就预先收集，如果遇到没有预批准的权限请求，工具调用直接失败，Agent 继续运行。按 Ctrl+B 可以把正在运行的前台 Agent 切换到后台。

**第二层：Agent Teams**

这不再是主 Agent 派遣子 Agent 的主从关系，而是完全独立的 Claude Code 实例通过共享文件系统协调任务。两者区别：

![[file-20260422205307056.png]]

Agent Teams 的协调机制包括：Shared Task List（所有 Agent 可见任务状态，完成当前任务后自主认领下一个未分配任务）、单播 Message（发给特定 Teammate）、Broadcast（发给所有 Teammate，注意成本随团队规模线性增长）、以及 Automatic Idle Notification（Teammate 完成任务停止时自动通知 Lead）。

同时，还有两个专门针对团队的质量门控 Hook：TeammateIdle（Teammate 即将进入空闲时触发，返回 exit code 2 可以发送反馈让它继续工作）和 TaskCompleted（任务即将被标记完成时触发，返回 exit code 2 可以阻止完成并要求修复）。

但 Agent Teams 目前还是实验性功能，需要通过 CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1 环境变量或 settings.json 启用。






