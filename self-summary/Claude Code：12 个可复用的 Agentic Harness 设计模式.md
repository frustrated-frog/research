
# 深度拆解 Claude Code：12 个可复用的 Agentic Harness 设计模式

1. 持久化指令文件模式（Persistent Instruction File Pattern）

这个模式的做法其实很直接：放一个项目级的配置文件，每次会话自动加载。里面写清楚构建命令、测试方式、架构规则、命名约定这些内容。文件跟着代码仓库走，而不是靠人每次复制粘贴。

**适用场景**：需要在多个会话里反复处理同一个代码库。

**权衡点**：有维护成本。这个文件需要跟着项目一起更新，一旦过时，反而会误导 Agent，还不如没有。

2. 作用域上下文组装模式（Scoped Context Assembly Pattern）

这个模式的思路是把指令拆到不同作用域里：组织级、用户级、项目根目录、父目录、子目录。Agent 会根据当前所在的位置，动态加载对应的规则。这样既能保持全局一致，又能允许局部有差异。

另外，通过导入的方式，可以把大的指令集拆开管理，避免重复。

**适用场景**：Monorepo、多语言项目，或者不同目录有不同规范的代码库。

**权衡点**：可读性会变差。规则分散在多个文件里后，很难一眼看清 Agent 实际加载了哪些内容，不同作用域之间也可能出现冲突。

3. 分层记忆模式（Tiered Memory Pattern）

这个模式的做法是把记忆分层：一个精简的索引始终放在上下文里（比如控制在几百行以内），和当前任务相关的内容按需加载，完整的历史记录则留在磁盘上，需要时再去查。

![[file-20260424114543739.png]]

**适用场景**：需要跨多次会话保留偏好、决策或状态的 Agent。

**权衡点**：实现会更复杂。需要想清楚信息该放哪一层，什么时候上升或下沉，以及怎么保证索引和实际数据是同步的。

4. 记忆整合模式（Dream Consolidation Pattern）

这个模式的思路是加一个后台整理机制，在空闲时定期做清理：去重、删旧、重组结构，让记忆保持干净、可用。可以理解为给 Agent 做一次「垃圾回收」。

![[file-20260424224533910.png]]

**适用场景**：Agent 会长期运行、持续积累记忆，而且不方便靠人工去维护。

**权衡点**：整理本身也要消耗 token，而且不一定完全准确。如果清理太激进，可能会把有用的信息一起删掉。

5. 渐进式上下文压缩模式（Progressive Context Compaction Pattern）

这个模式的做法是「**分层压缩**」：新的对话尽量保留细节，稍旧的内容做轻量总结，再往前的就逐步压缩，甚至折叠成很短的摘要。可以理解为越久远的信息，保留得越粗。像源码里提到的几层压缩，本质上也是这个思路，只是做得更细。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/CBgB44gdva29S0icHnSC438kIOjiadoBHjd2Kx98GNuNLYxS0iaqnT4k0bRmdMv1icVJf2FOZ2dyHvDxb9MKDRKst1ib0TN5icjmtPbyM5pogeYIc/640?wx_fmt=png&from=appmsg&tp=wxpic&wxfrom=5&wx_lazy=1#imgIndex=5)

**适用场景**：对话轮次比较多（比如 20～30 轮以上）的任务。

**权衡点**：压缩一定是有损的。信息在一轮轮总结中会丢失，如果后面又需要这些细节，Agent 可能会「编」而不是承认不知道。

## 工作流与编排

这一组模式的核心其实就是一个词：**分离**。

把读取和写入拆开，把「查资料」和「改代码」的上下文拆开，把顺序执行和并行执行也拆开。这样做的好处是，随着任务变复杂，系统不会越来越乱。

大多数 Agent 的默认做法是把这些事情混在一起，刚开始可能没问题，但任务一多，质量就很容易下降。

6. 探索-规划-行动循环模式（Explore-Plan-Act Loop Pattern）

这个模式的做法是把流程拆成三步，而且权限逐步放开：

- • 先探索，只读代码、查信息、摸清结构
    
- • 再规划，和用户对齐思路
    
- • 最后再动手改代码
    

本质上就是先搞清楚，再决定怎么做，最后再执行。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/CBgB44gdva3YQibznj5fa3M2SX5BhqNmE0G6KSc3q6NzzKKT2lQSib1jPfkFF61sckB9FhbTiayfE88ictjxficYmTZ8UH8K9aXuErShEoENDosk/640?wx_fmt=png&from=appmsg&tp=wxpic&wxfrom=5&wx_lazy=1#imgIndex=6)

**适用场景**：不熟悉的代码库，或者涉及多个文件的复杂修改。

**权衡点**：会慢一点。多了探索和规划这两步，小任务会显得有点「流程过重」。

### 7. 上下文隔离子智能体模式（Context-Isolated Subagents Pattern）

这个模式的做法是把任务拆给不同的子 Agent，每个都有自己的上下文和权限：

- • 做调研的只负责看和分析，不能改代码
    
- • 做规划的只负责设计方案
    
- • 真正执行的才有完整工具权限
    

每个子 Agent 只接触自己需要的信息，避免被「流噪声」影响。

![[file-20260425233235087.png]]

**适用场景**：长会话、多阶段流程，或者不同阶段对上下文要求差异很大的任务。

**权衡点**：需要额外协调。主 Agent 要决定每一步传什么信息，传少了会丢细节，传多了又回到上下文污染的问题。

### 8. 分支-合并并行模式（Fork-Join Parallelism Pattern）

这个模式的思路是把任务拆成多个分支并行处理：每个子 Agent 在独立的代码副本里工作（比如用 `git worktree`），互不干扰。等都完成后，再把结果合并回来。

![[file-20260425233228378.png]]

**适用场景**：可以拆成多个互不依赖子任务的场景。

**权衡点**：合并会更复杂。如果不同分支改到了同一部分代码，冲突可能比顺序处理更难解决。

## 工具与权限

9. 渐进式工具扩展模式（Progressive Tool Expansion Pattern）

这个模式的做法是先给一小部分常用工具，够用就行；其他工具按需再打开。比如读写文件、搜索这些作为默认能力，复杂一点的工具等用到再加载。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/CBgB44gdva0kmzppJYfzXAFn6QypdUNYdibdlLe8HoBicJqMnJhFJxBZanVtcwQaztnXrcTQEhn953kQ4unLu5NXDeh3v1BdgExghRJJQmQ0o/640?wx_fmt=png&from=appmsg&tp=wxpic&wxfrom=5&wx_lazy=1#imgIndex=9)

**适用场景**：工具很多，但大多数任务其实只用到一小部分。

**权衡点**：需要额外判断什么时候该开新工具。如果开得太晚，Agent 可能已经走了弯路，浪费了一些轮次。

10. 命令风险分类模式（Command Risk Classification Pattern）

这个模式的做法是在执行前做一层「风险判断」：低风险的命令自动放行，高风险的才需要人工确认或直接拦截。

实现上通常是对命令做解析（比如看做什么操作、带了哪些参数、影响范围多大），再结合规则去判断风险等级。

![图片](https://mmbiz.qpic.cn/mmbiz_png/CBgB44gdva0NG8y0hkXBnDWC4icMIbiaUoibhR2stIITWQQMicIN2aX3TOic5S4dMYltiaZEOdLP7ULibfOPRZn33c31VF4wyP5mlsVRthJfKziaovU/640?wx_fmt=png&from=appmsg&tp=wxpic&wxfrom=5&wx_lazy=1#imgIndex=10)

**适用场景**：Agent 能执行 shell 命令，或者会操作外部系统。

**权衡点**：规则不可能覆盖所有情况，需要不断调整；有时候会误判，要么放过风险操作，要么拦住本来安全的命令。

11. 单用途工具设计模式（Single-Purpose Tool Design Pattern）

这个模式的做法是把常见操作拆成专门的工具，比如读文件、改文件、搜索、匹配路径，各自都有明确的输入和边界。这样不仅更好理解，也更容易限制权限。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/CBgB44gdva0nmJ533p4jF0nLwpOL4SnvhaZGmhugngmQu4SicmJsRcjSdUACPAvO5agnyQExYOniaqRJm1bBOibgzrzDqvBYtmVkoeGibrrJCsM/640?wx_fmt=png&from=appmsg&tp=wxpic&wxfrom=5&wx_lazy=1#imgIndex=11)

**适用场景**：需要频繁做文件操作或搜索的 Agent。

**权衡点**：灵活性会受限。专用工具不可能覆盖所有情况，所以还是需要保留通用 shell 作为兜底。

## 自动化

最后这一类可以单独拿出来说，因为它其实贯穿前面的所有部分。

不管是记忆、工作流，还是工具，本质上都有一个共同问题：有些步骤是每次都必须执行的，但不能指望模型每次都记得。

### 12. 确定性生命周期钩子模式（Deterministic Lifecycle Hooks Pattern）

这个模式的做法是：把这些动作挂到 Agent 生命周期的关键节点上自动执行，完全不依赖提示词。比如工具调用前后、会话开始、工作目录变化时，系统都会触发对应的钩子。

简单来说，凡是「不能出错、不能漏」的事情，都不该交给模型记，而应该交给系统兜底。

![图片](https://mmbiz.qpic.cn/mmbiz_png/CBgB44gdva3MaWoDhvDeJFduOGibWe3GA1e5TMP4k7mJ3uK6XVs0SmVIQ4WGfkoTt0GricXvyXbwxKJUj1lImXDq7psXwBlswUPD6ETbyQXRY/640?wx_fmt=png&from=appmsg&tp=wxpic&wxfrom=5&wx_lazy=1#imgIndex=12)

**适用场景**：存在必须严格执行、不能遗漏的步骤

**权衡点**：出了问题不太好排查，因为这些逻辑是在对话之外跑的

## 结语：Harness

这些模式不是空谈的理论，而是从生产级代码中提炼出来的架构智慧。

**内存怎么分层、上下文怎么压缩、权限怎么控制、哪些流程必须自动执行**——这些本质上都是架构层面的决策。模型会变，工具也会换，但这些东西不会很快过时。
