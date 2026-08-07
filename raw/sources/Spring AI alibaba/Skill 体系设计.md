
核心源码入口如下：

|阅读顺序|源码|作用|
|---|---|---|
|1|[SkillsConfig.java](sandbox:/workspace/scratch/944227610cf9/spring-ai-alibaba-src/examples/multiagent-patterns/skills/src/main/java/com/alibaba/cloud/ai/examples/multiagents/skills/SkillsConfig.java)|看业务侧如何把 Skills 接入 ReactAgent|
|2|[sales_analytics/SKILL.md](sandbox:/workspace/scratch/944227610cf9/spring-ai-alibaba-src/examples/multiagent-patterns/skills/src/main/resources/skills/sales_analytics/SKILL.md)|看一个真实 Skill 的定义|
|3|[SkillScanner.java](sandbox:/workspace/scratch/944227610cf9/spring-ai-alibaba-src/spring-ai-alibaba-graph-core/src/main/java/com/alibaba/cloud/ai/graph/skills/registry/filesystem/SkillScanner.java)|扫描目录、解析 YAML、校验元数据|
|4|[SkillRegistry.java](sandbox:/workspace/scratch/944227610cf9/spring-ai-alibaba-src/spring-ai-alibaba-graph-core/src/main/java/com/alibaba/cloud/ai/graph/skills/registry/SkillRegistry.java)|Skills 注册中心抽象|
|5|[ClasspathSkillRegistry.java](sandbox:/workspace/scratch/944227610cf9/spring-ai-alibaba-src/spring-ai-alibaba-graph-core/src/main/java/com/alibaba/cloud/ai/graph/skills/registry/classpath/ClasspathSkillRegistry.java)|从 Spring classpath 和 JAR 加载 Skills|
|6|[SkillsInterceptor.java](sandbox:/workspace/scratch/944227610cf9/spring-ai-alibaba-src/spring-ai-alibaba-agent-framework/src/main/java/com/alibaba/cloud/ai/graph/agent/interceptor/skills/SkillsInterceptor.java)|向模型注入技能目录、动态开放工具|
|7|[ReadSkillTool.java](sandbox:/workspace/scratch/944227610cf9/spring-ai-alibaba-src/spring-ai-alibaba-agent-framework/src/main/java/com/alibaba/cloud/ai/graph/agent/hook/skills/ReadSkillTool.java)|模型按需读取完整 Skill|
|8|[SkillsAgentHook.java](sandbox:/workspace/scratch/944227610cf9/spring-ai-alibaba-src/spring-ai-alibaba-agent-framework/src/main/java/com/alibaba/cloud/ai/graph/agent/hook/skills/SkillsAgentHook.java)|把注册中心、拦截器和工具组装到 Agent|

整个源码可以分成“启动加载”和“运行时激活”两条链路。

启动时的加载链路

示例中的 `SkillsConfig` 首先创建 `ClasspathSkillRegistry`。它默认扫描 `resources/skills`，每一个一级子目录代表一个 Skill，每个目录必须包含 `SKILL.md`。

`ClasspathSkillRegistry` 构造时默认开启 `autoLoad`，于是调用 `loadSkillsToRegistry()`。它先通过 ClassLoader 找到 `skills` 目录，然后区分两种运行环境：开发环境一般是普通 `file:` 路径，生产环境可能是 `jar:` 路径。对于 JAR，它还要创建一个 NIO `FileSystem`，这样才能像遍历普通目录一样遍历 JAR 内部资源。

找到目录以后，`SkillScanner.scan()` 会遍历所有 Skill 子目录，再由 `loadSkill()` 读取 `SKILL.md`。它主要完成四件事：解析 YAML frontmatter、提取 `name/description/allowed_tools`、校验 Skill 名称规范、去掉 frontmatter 得到完整指令正文。最终构造为 `SkillMetadata`，放入注册中心的 `Map<String, SkillMetadata>`。

文件系统实现还支持两级覆盖：

```text
用户级 ~/saa/skills
        ↓ 同名时被覆盖
项目级 ./skills
```

项目级 Skill 的优先级更高。它的实现非常直接：先把用户 Skill 放进 Map，再把项目 Skill按相同名称写入 Map，利用 `put` 完成覆盖。

运行时的激活链路

`SkillsAgentHook` 是整套机制的装配入口。它做了三件事：创建 `SkillsInterceptor`，向 Agent 注册 `read_skill/search_skills/disable_skill` 三个工具，并根据配置决定是否在每次 Agent 执行前重新加载注册中心。

真正控制渐进式加载的是 `SkillsInterceptor.interceptModel()`。模型每次调用前，拦截器都会从注册中心获得所有 Skill，但不会把完整 `SKILL.md` 全部放进上下文，只把技能名称、描述和路径组装成一段 Skills System Prompt。

因此模型第一次看到的大致只是：

```text
sales_analytics：用于销售数据分析
inventory_management：用于库存管理
```

当用户问“查询最近一个季度收入最高的客户”时，模型根据 `sales_analytics` 的描述判断它可能适用，然后产生一个 `read_skill` 工具调用。`ReadSkillTool` 接收 `skill_name` 或 `skill_path`，通过注册中心找到对应的 `SkillMetadata`，再把完整 Skill 正文作为工具结果返回给模型。

下一次 ReAct 循环中，模型上下文里已经包含完整的表结构、业务口径和查询规则，于是才能生成符合要求的 SQL。完整执行过程是：

```text
用户请求
→ 拦截器注入技能目录
→ 模型判断需要 sales_analytics
→ 模型调用 read_skill
→ 注册中心返回完整 SKILL.md
→ 模型读取业务规则
→ 生成最终 SQL
```

动态工具是怎么随 Skill 激活的

这份源码不只支持加载知识，还支持“激活某个 Skill 后才开放对应工具”。

`SkillsInterceptor` 会扫描历史 `AssistantMessage`，寻找已经发生过的 `read_skill` ToolCall，从调用参数中解析出 Skill 名称。随后通过 `groupedTools` 或 Skill frontmatter 中的 `allowed_tools` 找到对应 `ToolCallback`，加入本轮的 `dynamicToolCallbacks`。

这里有一个很容易忽略的时序细节：工具不是在模型决定读取 Skill 的同一瞬间注入，**而是在 `read_skill` 执行完成后的下一次模型调用前注入。**

例如：

```text
第一轮模型调用：只能看到 read_skill
→ 调用 read_skill("sales_analytics")
→ 工具结果返回

第二轮模型调用：拦截器发现历史 read_skill 调用
→ 注入 sales_analytics 对应的数据库工具
→ 模型开始调用数据库工具
```

这才是真实工程里的“Skill 激活”：不仅把知识正文交给模型，还动态改变模型在后续执行阶段能够看到的工具集合。

源码中值得深入思考的地方

第一，这个项目所谓的“渐进式加载”，主要是模型上下文层面的渐进式披露，不完全是存储层面的懒加载。`SkillScanner` 启动时已经调用 `Files.readString()` 读取了完整 `SKILL.md`，并把 `fullContent` 放进 `SkillMetadata`。只是首次模型调用时没有把完整内容塞进 Prompt，直到调用 `read_skill` 才把它返回给模型。

因此它节省的是上下文 Token，而不是启动时的文件 IO 和 JVM 内存。如果 Skill 数量达到几千个、单个 Skill 又很大，生产系统可以进一步改成：启动时只解析 frontmatter，完整正文在第一次 `read_skill` 时读取并写入缓存。

第二，它没有额外训练一个 Skill 分类器。Skill 路由依赖模型阅读 `name + description` 后自主决定是否调用 `read_skill`。这是一种实现简单、可解释性较好的方案，但 Skill 数量很大时，技能列表本身会膨胀，也容易发生描述相似导致的错误选择。更完整的企业实现通常会在注册中心前增加 BM25、向量召回或规则路由，只向模型暴露 TopK 候选 Skill。

第三，它没有显式维护 `activeSkills` 状态，而是通过扫描历史消息中的 `read_skill` 调用反推出已经激活的 Skill。这种设计不需要修改 Agent State，接入成本低，但也有一个明显隐患：如果长对话经过上下文裁剪，把早期 `read_skill` ToolCall 删除了，对应动态工具就可能不再被注入。生产环境更可靠的做法，是把已激活 Skill 的名称和版本单独持久化到运行状态中。

第四，注册中心的热更新设计比较值得学习。`reload()` 使用 `synchronized` 防止并发重载，技能集合使用 `volatile Map`，重载时先在局部变量里构建完整的新 Map，最后一次性替换引用。这样读请求不需要一直加锁，也不会看见加载到一半的 Map，这是一种简单的 Copy-on-Write 思路。

这套源码非常适合作为你的第一套 Skills 源码学习对象。建议先从 `SkillsConfig → SkillScanner → SkillsInterceptor → ReadSkillTool` 四个文件开始，它们已经能串起完整链路。下一步最值得仔细拆的是 `SkillsInterceptor.interceptModel()`，因为 Skills 的识别、上下文注入和工具动态开放都集中在这里。