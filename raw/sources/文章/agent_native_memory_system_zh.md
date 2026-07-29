# 我们准备好迎接 Agent 原生记忆系统了吗？

**Wei Zhou**  
上海交通大学  
weizhoudb@sjtu.edu.cn

**Xuanhe Zhou\***  
上海交通大学  
zhouxuanhe@sjtu.edu.cn

**Shaokun Han**  
上海交通大学  
areedd0@sjtu.edu.cn

**Hongming Xu**  
上海交通大学  
muzhihai@sjtu.edu.cn

**Guoliang Li**  
清华大学  
liguoliang@tsinghua.edu.cn

**Zhiyu Li**  
MemTensor（上海）科技有限公司  
lizy@memtensor.cn

**Feiyu Xiong**  
MemTensor（上海）科技有限公司  
xiongfy@memtensor.cn

**Fan Wu**  
上海交通大学  
fwu@cs.sjtu.edu.cn

\* Xuanhe Zhou 为通讯作者。

---

## 摘要

大语言模型（LLM）智能体的记忆，已经从简单的检索增强机制迅速演化为一种数据管理系统。该系统支持在智能体执行全过程中进行持久化信息存储、检索、更新、整合以及动态生命周期治理。尽管出现了这种演化，现有评测仍然主要通过端到端任务成功指标（例如 F1、BLEU）来评测智能体记忆，同时把底层系统视为一个整体式黑盒。因此，包括运行成本、不同记忆模块之间的架构权衡，以及动态知识更新下的鲁棒性在内的关键系统级问题，仍然没有得到充分探索。

在本文中，我们从数据管理视角出发，对智能体记忆进行系统性实验研究。我们提出一个分析框架，将智能体记忆分解为四个核心模块：记忆表示与存储、记忆抽取、记忆检索与路由、记忆维护。在该框架下，我们在覆盖 11 个数据集的 5 类基准工作负载上，评估了 12 个代表性记忆系统和两个参考基线。我们的大规模端到端评测表明，没有任何一种单一架构能够在所有场景中占据主导地位；相反，有效性很大程度上取决于记忆结构与工作负载瓶颈之间的匹配程度。此外，通过细粒度消融研究，我们量化了各模块对表示保真度、检索精度、更新正确性和长程稳定性的独立影响。最后，我们揭示了真实工作负载下的成本-性能权衡，表明局部化维护比全局重组更具成本效率。基于这些发现，我们识别出构建真正 Agent 原生记忆系统的若干有前景方向。代码公开于：https://github.com/OpenDataBox/MemoryData。

---

## 1 引言

大语言模型（LLM）智能体的快速发展，激发了大量关于构建智能体记忆的研究和工业实践。所谓智能体记忆，是指 LLM 智能体的数据管理系统，它支持长程有状态执行和个性化交互 [9, 13, 17, 19, 23, 24, 28]。

如图 1 所示，现有智能体记忆系统覆盖了多样化的架构设计。  
（1）**流式与反思型记忆系统**（例如 MemoryBank [39]）将经验维护为带时间戳的记忆流，并周期性地将其总结为更高层级的反思，再写回到记忆流中。  
（2）**分层层级式记忆系统**（例如 MemGPT [25]）将记忆组织为多个层级，每个层级具有不同容量和访问属性，将核心记忆与归档存储分离，并显式地在层级之间移动记忆，例如驱逐和提升。  
（3）**知识图谱记忆系统**（例如 Mem0g [5]、Zep [26]）以结构化形式表示实体、关系及其时间演化，例如时间知识图谱，并通常包含实体消歧和冲突解决机制。  
（4）**复合混合型记忆系统**（例如 A-MEM [33]）在多个存储基底之间路由具备模式感知的记忆对象，显式区分运行时状态（例如 KV 缓存）和长期存储（例如向量索引、图索引、关键词索引），并由专门的维护模块管理。

然而，这种快速扩散也导致了高度碎片化的系统格局，并且缺乏从数据管理视角出发的系统评估。这自然引出了一个问题：**我们是否已经准备好迎接 Agent 原生记忆系统？**

本文重新审视智能体记忆中的这一问题。具体而言，我们聚焦于文本、结构化，甚至参数化表示之上的系统级记忆 [5, 25, 26, 33]，这是现代自治智能体的一项基础设施组件。我们关注以记忆为核心的系统，而不是那些把记忆作为辅助模块的任务特定智能体框架 [1, 40]。智能体记忆是一种持久化数据管理系统，它维护超出单次推理步骤的信息，例如历史交互、环境观察和中间工具执行，并且与 LLM 的参数权重和易失性上下文窗口相解耦。智能体框架依赖这些外部记忆系统（例如 Mem0 [5]、Letta [25]、Zep [26] 和 A-MEM [33]）主动写入、更新、索引并将相关上下文路由回推理循环。长程智能体的能力在很大程度上取决于这一记忆层的可靠性和效率。采用设计不佳记忆架构的智能体，在连续执行过程中可能遭遇事实矛盾、灾难性遗忘或不可接受的延迟 [6, 38]。

近期基准 [20, 22, 29, 31] 已经评估过智能体记忆，并表明外部记忆可以提升需要事实回忆和长上下文理解的任务表现。然而，这些评测主要根植于自然语言处理范式；当把智能体记忆视为数据管理系统时，它们存在多重局限（见第 2 节）。

第一，它们未能在统一工作负载下评估许多代表性记忆架构，例如 MemoChat、MemTree 和 LightMem 等系统没有被纳入此前评测，因此难以进行有原则的跨系统比较。数据库领域的工作 [32] 又将范围限制在少数以聊天机器人为中心的数据集上，例如 LoCoMo 和 LongMemEval，忽略了复杂智能体执行场景。

第二，现有基准主要依赖单侧、端到端任务成功指标，例如 F1 和 BLEU 分数，而不是全面的评测套件。它们没有显式隔离并度量多维性能指标，例如证据级检索保真度、冲突知识下的动态更新鲁棒性和长程稳定性。

第三，它们很少从系统视角测量关键运行成本，例如索引构建时间和查询延迟，而这些对于生产部署至关重要。

最后，它们把记忆系统当作整体式黑盒，而没有将其分解为基本数据管理模块来进行孤立、细粒度分析。

我们克服上述限制，从数据管理视角进行全面实验和分析。本文贡献如下。

（1）**技术分解与分类法（第 3 节）**。我们将现有智能体记忆系统分解为四个核心组件：（i）记忆表示与存储，（ii）记忆抽取，（iii）记忆检索与路由，（iv）记忆维护。对于每个组件，我们进一步依据底层设计原则对现有方法进行分类，建立结构化分类法，从而支持有原则的比较。

（2）**整体端到端性能评估（第 4 节）**。我们在统一且公平的测试平台下，例如统一的时间开销轨迹，对涵盖 11 个数据集的五种不同基准工作负载进行端到端评测。研究包含 12 个代表性记忆系统，每个系统体现了不同的表示、存储、路由和维护策略组合。我们从五个方面评估它们的性能：任务有效性（RQ1）、检索保真度（RQ2）、动态更新鲁棒性（RQ3）、长程稳定性（RQ4）和运行成本（RQ5）。

（3）**细粒度技术组件评估（第 5 节）**。利用四模块框架，我们对每个技术组件中的代表性策略开展受控且细粒度的实验。通过系统性地生成一次只修改一个模块的受控变体，我们量化其性能权衡，并评估其对表示保真度、路由精度和更新正确性的个体影响。

（4）**洞察性发现**。基于实验结果和深入分析，我们提炼出关于智能体记忆系统成本-性能权衡的一组洞察。

❶ **记忆系统是否在不同智能体请求工作负载上有效？** 没有单一记忆架构可以主导所有场景。复合混合系统在会话问答上领先；图方法在单跳事实回忆中表现优异，但在时间推理上遇到困难。此外，有效的记忆系统在不同 LLM 主干变体下仍然稳健，因为它们在答案生成之前将证据定位外部化。

❷ **记忆系统检索已存证据的准确性如何？** 显式查询规划和平衡的混合搜索能够最大化上下文相关性。然而，随着证据和查询之间时间距离增大，检索准确率显著下降，暴露出基于相似度检索的局限。

❸ **记忆系统在动态更新下是否鲁棒？** 图方法最可靠地处理知识更新，而流行的事实抽取插件和追加式存储难以进行定向覆盖。缺乏生命周期管理的系统会返回陈旧事实，导致“过去的幻觉”。

❹ **记忆系统是否能在长程场景中保持稳定？** 许多追加式记忆存储会随着证据距离变远而发生灾难性退化。对于时间相关查询，原始长上下文检索仍然优于多数记忆支持方法，说明标准语义整合往往会破坏关键的时间顺序线索。

❺ **智能体记忆的运行成本是多少？** 高结构化系统的索引构建时间和查询延迟比轻量存储高出数个数量级，但并不总是带来成比例的准确率收益。

❻ **单个记忆组件何时会出错？** 每一层抽象，例如压缩、总结和事实抽取，都会逐步丢弃信息。此外，细粒度 LLM 抽取可以带来有限的精度增益，但会显著损害多跳推理。最后，保守记忆整合是最佳默认维护策略，而延迟刷写会在表面覆盖率和实际可回答性之间制造一种具有迷惑性的权衡。

---

## 图 1：智能体记忆的典型执行工作流

图 1 展示了四类典型 Agent 记忆执行工作流：

- （a）结构拓扑型：流式日志。用户的观察或经验被写入时间戳记忆流，检索评分器依据新近性、重要性和相关性返回记忆；反思模块周期性地总结并写回。
- （b）结构拓扑型：层级分层。LLM 智能体通过函数调用操作核心记忆、召回存储和归档存储，并支持追加、替换、插入和搜索。
- （c）结构拓扑型：知识图谱。实体抽取、实体消歧、冲突解决和图遍历共同组成时间知识图谱记忆。
- （d）多范式混合记忆系统。消息或轨迹经记忆编码器抽取为结构化对象，经存储路由器写入多引擎存储；混合检索、重排序、上下文组装器和维护控制器共同支持记忆增强上下文。

---

## 2 预备知识

为了支撑本文后续讨论，我们首先从数据管理视角澄清智能体记忆的范围。尽管近期研究已经从认知分类、智能体架构和基于图的组织等视角审视记忆 [6, 10, 30, 32, 34, 36]，其底层概念仍然经常主要被视为 LLM 或智能体流水线中的算法组件 [6, 10, 36]。与此不同，我们将智能体记忆研究为一个独立的数据管理对象和系统基础设施，明确关注它如何在真实智能体工作负载下被表示、存储、检索、更新和维护。在这一视角下，我们给出以下定义。

### 记忆类型

对于 LLM 智能体，多种信息会被产生并可能需要被记忆，包括对话历史、工具执行日志、蒸馏出的事实以及用户偏好 [36]。按照既有认知框架，记忆可以沿两个轴大体组织 [10, 30]。

（1）沿时间轴，短期记忆保存进行中会话的易失性状态，而长期记忆跨会话持久保存。  
（2）沿功能轴，长期记忆进一步分为具体过去事件等信息（情景记忆）、抽象事实知识（语义记忆）[10, 32]、可复用行动策略（程序性记忆）和用户偏好。

### 智能体记忆

我们将智能体记忆 M 定义为持久化数据管理对象 [25, 32]，它维护超出单次推理步骤的累积状态，并使智能体能够在未来推理和行动中访问这些状态 [10, 36]。

### 智能体记忆系统

为了将智能体记忆 M 具体落地，需要一个稳健基础设施 [10, 25]。如表 1 所示，从数据系统视角 [14, 32]，我们将智能体记忆系统形式化为四个模块的元组：M_sys = <R, S, Q, U>，其中每个模块控制记忆生命周期的不同阶段。

（1）**记忆表示与存储 R**：该映射定义逻辑和物理记忆格式，并包含数据模型的两个方面：（a）逻辑表示，从简单原语（离散 token、连续向量）到复杂拓扑（知识图谱、树和复合体）；（b）物理存储，使用瞬时寄存器、专用单引擎数据库或多引擎后端来进行持久化和索引。

（2）**记忆抽取 S**：该机制控制异构输入流（例如多轮对话、工具日志）如何通过原始序列拼接、无模式语义抽取或模式约束结构化抽取等流水线，转换为逻辑记忆原语。

（3）**记忆检索与路由 Q**：该函数基于查询上下文动态识别相关记忆子集，并使用特定路由算法遍历索引。机制包括原生注意力检索、语义 K 近邻搜索、拓扑子图遍历、通过 LLM 规划的自治智能体路由，以及多阶段混合执行。

（4）**记忆维护 U**：该策略控制记忆条目的动态生命周期，被分解为三个子操作：（a）冲突解决与版本控制，通过多版本、失效标记或优先级规则处理矛盾；（b）容量管理，通过基于约束的硬驱逐（例如 FIFO、token 限制）或基于评分的优先级驱逐（例如时间衰减）限制增长；（c）语义整合，使用 LLM 将冗余断言合并为密集摘要，或通过工具调用接口执行 CRUD 操作。

### 与 RAG 和上下文工程的区别

检索增强生成（RAG）[8, 13] 通常作为无状态、只读检索原语运行：给定一个查询，它从静态语料库中取回相关段落来增强单次生成步骤。上下文工程 [2] 是更广义的实践，即在每次推理轮次中整理有限的 LLM 上下文窗口，例如动态选择提示、工具描述和检索到的事实，以缓解上下文腐化 [2]。

相比之下，智能体记忆系统：（1）是管理智能体特定状态的持久且可更新基础设施；（2）控制完整长期记忆生命周期，包括记忆表示、存储、检索和维护，而不仅仅是在当前上下文窗口中打包内容。

### 与传统数据库工作负载的区别

智能体记忆工作负载与传统数据库 OLTP / OLAP 工作负载差异显著 [17, 25]。

第一，记忆访问往往是语义性的，而不是纯谓词式的 [4, 11]。查询通常以自然语言、部分上下文或潜在意图表达，因此依赖近似匹配、查询重写或 LLM 引导检索，而不仅仅是刚性模式上的精确逻辑谓词。

第二，记忆内容在连续且可能冲突的观察下演化。不同于传统事务设置中，更新通常在预定义模式和一致性模型下覆盖元组，智能体记忆必须容纳跨时间、工具和环境收集到的不确定、部分且有时矛盾的信息 [10, 38]。

第三，智能体记忆工作负载在访问模式和粒度上高度异构。单个工作负载可能同时包含长上下文综合、情景回忆、结构化事实查找、时间推理和流式更新。因此，实践系统通常需要混合执行策略，在一种记忆架构中组合语义检索、结构化过滤和拓扑感知遍历 [25, 32]。这些性质将智能体记忆与传统数据库区分开来，并推动了专门抽象和评估方法的发展。

---

## 图 2：记忆表示方法

图 2 展示了三类记忆表示：

1. **Token 级序列**：包括显式离散文本 token 与隐式连续向量 token。例子包括“Kim is vegetarian ...”、嵌入向量 `[0.21, ..., 0.46]`、潜在状态向量等。
2. **图与树拓扑**：包括时间知识图谱和层级树结构。节点可以表示实体，边可以表示关系；根节点可以是摘要，叶子节点可以是原始事实。
3. **异构复合表示**：记忆对象同时包含类型与日期、文本、嵌入和图结构等元数据。

---

## 3 方法概览

本节细致分析现有智能体记忆系统在第 2 节所述四个组件上的设计，并建立一个统一分类法，总结代表性组件方法。

### 3.1 记忆表示与存储

该模块由两个组件组成：（1）逻辑表示，它定义暴露给智能体系统的结构编码和组织方式，直接决定容量、可访问性，以及表达能力、检索粒度和下游推理兼容性之间的权衡；（2）物理存储，它指定持久化和索引结构，例如易失性的上下文寄存器、密集向量引擎或拓扑图数据库。

### 表 1：智能体记忆系统的分类与特征

| 类别 | 方法 | 表示 | 存储 | 记忆抽取 | 记忆检索与查询路由 | 记忆维护 |
|---|---|---|---|---|---|---|
| Sequential Context | MemoChat [18] | Token 级序列（结构化 JSON 备忘录） | 瞬时上下文寄存器 | 模式约束抽取（LLM 主题分割） | 自治智能体路由（LLM 主题选择） | LLM 驱动语义整合（轮次触发） |
| Sequential Context | Mem0 [5] | Token 级序列（离散事实） | 专用单引擎（向量数据库） | 无模式抽取 | 基于语义检索 | LLM 驱动语义整合（工具调用） |
| Sequential Context | MEM1 [41] | Token 级序列 | 瞬时上下文寄存器 | 原始序列拼接 | 原生注意力检索 | 容量驱动的物理驱逐 |
| Sequential Context | MemAgent [35] | Token 级序列 | 瞬时上下文寄存器 | 原始序列拼接（递归摘要） | 原生注意力检索 | 容量驱动物理驱逐（RL 覆盖） |
| Structural Topological | MemTree [27] | 图与树拓扑（层级树） | 专用单引擎（向量数据库） | 无模式抽取（自顶向下嵌入） | 基于语义检索（折叠树） | LLM 驱动语义整合（递归聚合） |
| Structural Topological | Zep [26] | 图与树拓扑（时间知识图谱） | 专用单引擎（图数据库） | 模式约束抽取（三元组） | 多阶段混合执行（Dense + BM25 + BFS） | 基于时间戳的多版本（逻辑失效） |
| Structural Topological | Mem0g [5] | 图与树拓扑（标注图） | 异构多引擎（向量 + 图数据库） | 模式约束抽取（实体-关系） | 拓扑子图遍历 | 基于时间戳的多版本 |
| Structural Topological | Cognee [21] | 图与树拓扑（实体-关系三元组） | 异构多引擎（图 + 向量 + 关系数据库） | 模式约束抽取（通过 Pydantic 的 ECL 流水线） | 拓扑子图遍历（Dense 引导三元组抽取） | 基于时间戳的多版本（基于哈希去重） |
| Multi-Paradigm Hybrid | LightMem [7] | 异构复合（三分模式） | 专用单引擎（关系数据库） | 无模式抽取（熵门控） | 基于语义检索 | 基于时间戳的多版本（追加式日志） |
| Multi-Paradigm Hybrid | SimpleMem [16] | 异构复合 | 异构多引擎（向量数据库 + BM25 + SQL） | 模式约束抽取 | 自治智能体路由（查询扩展） | LLM 驱动语义整合（即时合成） |
| Multi-Paradigm Hybrid | MemOS [15] | 异构复合（MemCube） | 异构多引擎（向量 + 图数据库） | 模式约束抽取（语义解析器） | 多阶段混合执行（布尔 + 语义） | 基于时间戳的多版本（差分写入） |
| Multi-Paradigm Hybrid | MemoryOS [12] | 异构复合（段-页） | 异构多引擎（关键词索引 + 向量数据库） | 模式约束抽取 | 多阶段混合执行（层级路由） | 容量驱动物理驱逐（热度驱逐） |
| Multi-Paradigm Hybrid | A-MEM [33] | 异构复合（原子笔记） | 异构多引擎（向量 + 图数据库） | 模式约束抽取（JSON 属性） | 拓扑子图遍历 | LLM 驱动语义整合（变异与剪枝） |
| Multi-Paradigm Hybrid | Letta [25] | 异构复合（上下文层） | 专用单引擎（关系数据库） | 模式约束抽取 | 自治智能体路由（函数调用） | 容量驱动物理驱逐（队列刷写） |

#### 3.1.1 逻辑表示

如图 2 所示，该组件通过将记忆组织为清晰模型，例如图或向量空间，在原始数据和执行环境之间架起桥梁。它决定系统能够多高效地搜索、组合并使用历史上下文来完成复杂任务。

❶ **Token 级序列表示**。该类别将记忆建模为扁平的一维序列，缺少显式结构抽象，例如图或层级。记忆既可以被表示为离散、可读的自然语言 token，也可以被表示为隐式连续潜在向量 token，例如事实嵌入、隐藏状态或 KV-cache 张量。

- **显式离散文本 token**。该类别将记忆建模为可读字符串或独立事实陈述。例如，Mem0 将记忆隔离为从交互历史中直接抽取的离散自然语言事实。类似地，MemoChat 将多轮对话组织为离散 JSON 块（主题、摘要、原始轮次），以在纯文本范式中维持主题连贯性。虽然这些系统将纯文本记忆外部化，另一些系统则将其保留在活跃处理窗口中：MemAgent 将内部信念状态限制为严格有界的文本序列（例如 1024 个 token），MEM1 则将内部状态摘要封装在专门边界标签内（例如 `<IS>`）。

- **隐式连续向量 token**。不同于可读文本 token，该子类将记忆编码为连续向量。这些向量既可以作为外部嵌入附着于事实和摘要上用于语义检索，也可以作为模型侧潜在状态，例如压缩内部状态和注意力缓存。例如，Mem0 将抽取事实表示为密集语义嵌入，MemoRAG 使用专门初始化的权重矩阵将原始输入压缩为高维 Key-Value（KV）缓存张量。虽然这些向量 token 表示减少了显式 token 化负担，并自然融入检索或推理流水线，但它们牺牲了结构可解释性，并且很难通过细粒度操作进行操纵，例如谓词级过滤或对编码事实进行定向更新。

❷ **图与树拓扑表示**。该类别将记忆抽象为由互连节点和边组成的结构化图与树拓扑，使会话实体、高层概念及其时间或语义关系能够被显式建模并以计算方式遍历。

- **时间知识图谱**。该子类使用图拓扑来映射实体及其连接，天然支持时间推理和冲突检测。例如，Zep 将记忆划分为形式定义的、时间感知知识图谱（例如 episode、entity 和 community 子图）。类似地，Mem0g 将记忆形式化为有向标注图，顶点表示实体，边封装关系三元组（例如 LIVES_IN）。为了辅助时间推理，实体节点会带有结构元数据，例如语义类型、密集嵌入和创建时间戳。

- **层级树结构**。该子类将知识组织为递归层级结构，在终端叶子节点保留高粒度观察，在祖先节点保留广义语义抽象。例如，MemTree 将记忆建模为动态有向树模式。每个节点是一个包含文本内容、密集嵌入、拓扑指针和深度标量的元组。在该拓扑中，深层叶节点保留孤立事实（例如某个球员得分），祖先节点提供高层概念摘要（例如比赛结果），专门的根节点作为确定性入口点。

❸ **异构复合表示**。该类别超越简单 token 序列和标准图，将记忆封装为复杂的多部分数据容器。这些架构直接将非结构化文本与高度结构化元数据（例如时间戳、分类标签、向量嵌入和网络链接）结合成单一功能单元。例如，MemOS 提出 MemCube，这是一种统一数据对象，将记忆组织为三种不同载荷（纯文本、激活记忆和参数记忆），并附带结构化细节（例如 ID 标签）。

#### 3.1.2 物理存储与索引

如图 3 所示，该组件管理数据如何被物理存储和访问，依赖内存缓存、文件、向量引擎或数据库等系统。它设定实际容量限制，并决定记忆操作的速度、吞吐和整体可扩展性。

❶ **瞬时上下文寄存器**。为了消除磁盘 I/O 和外部遍历延迟，该类别将记忆完全保留在活跃硬件状态中，例如动态上下文窗口或 KV 缓存。MemoChat 避免专用外部记忆引擎，并在记忆-检索-响应循环中把结构化 JSON 风格备忘录保存在 LLM 上下文输入中；MemAgent 则通过密集位置嵌入将摘要 token 直接存储为 Key-Value（KV）缓存张量。

❷ **专用单引擎存储**。该类别将形成的记忆单元物理存放在独立、同质的后端中，该后端严格适配记忆的逻辑结构。根据摄入范式，架构会部署不同后端拓扑：（1）密集向量数据库，用于将数据投射到连续高维空间；Mem0 和 MemTree 使用集中式向量存储，Letta 使用带 pgvector 扩展的 PostgreSQL；（2）图数据库，用于执行拓扑约束；Zep 和 Mem0g 都执行预定义 Cypher 查询，将逻辑图组件物理持久化到 Neo4j；（3）关系 SQL 引擎，用于序列化结构和时间模式。LightMem 增量追加事实流，以保持全局关系状态；（4）文件或对象存储，保存原始交互制品，例如会话历史或工具执行日志。

❸ **异构多引擎存储**。该类别动态构建多种索引类型，或将数据分布到异构后端中，例如将密集向量存储与拓扑图数据库配对。SimpleMem 将记忆摄入 LanceDB，并使用 IVF-PQ 机制同时维护密集嵌入、稀疏 BM25 索引和 SQL 谓词。MemoryOS 依赖融合密集余弦相似度与离散 Jaccard 相似度的混合索引。相反，MemOS 将序列化载荷委派给高度专门化的独立后端，并通过标准化记忆适配器接口融合向量数据库和图数据库。

---

## 图 3：记忆存储方法

图 3 将物理存储分为三类：

1. 瞬时上下文寄存器：无外部存储，依赖缓冲区或上下文窗口。
2. 专用单引擎：单一专用存储，例如文件对象、关系数据库、向量数据库或图数据库。
3. 异构多引擎：多分布式存储，通过记忆路由器连接文件、关系、向量和图数据库。

---

### 3.2 记忆抽取

记忆抽取关注原始交互轨迹如何以计算方式处理。它涵盖抽取流水线，即语言模型如何将非结构化文本抽取、总结或解析为逻辑结构。如图 4 所示，它定义了智能体记忆系统如何在物理持久化之前，将异构输入流（例如多轮对话和工具执行日志）转换为逻辑记忆原语。

❶ **原始序列拼接**。为了最小化计算开销，该类别绕过显式抽取提示，将记忆直接构造为原始 token 拼接或瞬时状态摘要，例如将最近对话轮次直接追加到提示缓冲区。MEM1 和 MemAgent 等系统仅将其新形成的结构保存在活跃计算状态中，而不进行二次解析。

❷ **无模式语义抽取**。该类别系统性地将原始非结构化输入蒸馏为独立、高价值的信息单元，并将其表示为显式自由形式文本或压缩连续潜在向量。通过从更广泛的会话上下文中隔离核心知识，它确保精确且粒度化的检索。例如，Mem0 主动解析交互，抽取并存储离散、独立的事实陈述，例如“用户是素食者且不吃奶制品”。

❸ **模式约束结构化抽取**。该类别提示 LLM 解析原始输入并同步填充刚性预定义结构模式，生成严格类型化数据而不是自由文本。根据目标后端不同，受约束输出可以是用于图插入的拓扑实体-关系三元组，也可以是用于混合存储的多模态关系载荷。Zep 和 Mem0g 抽取类型化有向关系边，例如 LIVES_IN 和 WORKS_AT，并符合预定义图模式；Zep 还额外应用一种受反思启发的验证步骤，以抑制幻觉三元组。MemoChat 通过利用 LLM 将会话分割为严格 JSON 模式来填充预定义结构字段，以确保数据可预测性。

---

## 图 4：记忆抽取方法

图 4 展示三种抽取方式：

1. 原始序列拼接：将用户和机器人对话直接放入有界上下文缓冲区。
2. 无模式语义抽取：LLM 从文本中抽出事实和向量嵌入，无刚性预定义模式。
3. 模式约束结构化抽取：LLM 按预定义结构抽取事件记录和实体关系拓扑，例如 `(Kim, PREFERS, Vegetarian Food)`、`(Kim, LIVES_IN, Paris)`。

---

### 3.3 记忆检索与查询路由

记忆检索与查询路由决定智能体记忆系统如何动态识别并抽取相关历史上下文，以告知上层智能体当前推理状态。如图 5 所示，该模块涵盖完整查询执行谱系，定义遍历索引所使用的运行算法、谓词评估和智能体式工作流。

❶ **原生注意力检索**。为了绕过外部数据库 I/O，该类别使用 Transformer 的原生计算图作为唯一检索引擎，完全依赖自注意力机制隐式加权和路由信息，例如直接在 KV 缓存中扫描对话 token。MEM1 通过当前序列上的自注意力执行隐式检索，并使用二维注意力掩码保持因果一致性。MemAgent 通过将块直接拼接到提示模板中实现路由，从而不需要外部交叉编码器重排序即可执行标准基于注意力的解码。

❷ **基于语义的密集检索**。该类别在连续潜在空间上运行，将查询张量映射到统一向量索引，以抽取局部空间邻居，例如执行标准 K 近邻（KNN）搜索。Mem0 为输入查询计算向量嵌入并执行密集相似度搜索，取回受限事实子集。LightMem 在密集嵌入上使用高效余弦相似度距离计算，绕开计算开销大的迭代重排序。MemTree 实现折叠树架构，在数学上压平其层级结构，将输入向量广播到所有候选上，计算全局余弦相似度分布。

❸ **拓扑子图遍历**。不同于连续向量空间，该类别通过遍历显式关系边来检索信息，抽取在知识图谱中由结构支撑的语义簇，例如从 User 节点跳转到相关 Preference 节点。Mem0g 部署实体中心启发式方法，并与语义三元组评估同步递归遍历局部子图。A-MEM 通过密集 K 近邻选择识别候选锚点，然后执行局部图遍历，访问在同一概念簇内显式相连的拓扑邻近记忆节点。

❹ **自治智能体路由**。该类别不是执行确定性数据库扫描，而是将检索委托给 LLM 本身，使其作为主动、自治查询规划器。它生成工具调用或起草隐式搜索条件。

- **函数调用调用**。该子类通过生成显式函数调用命令，将 LLM 与外部存储桥接，以直接执行预定义数据库操作，例如输出有效 JSON 载荷来触发外部数据库 API。比如 Letta 编排自我导向记忆检索：LLM 评估其活跃上下文并显式生成局部函数调用，例如发出 `archival_storage.search()` 命令来抽取目标历史日志。

- **生成式查询扩展**。不同于刚性函数调用，该方法使用自然语言生成来合成中间线索，或在映射到索引前分解复杂意图，例如将模糊提示重写为描述性搜索字符串。SimpleMem 使用意图感知检索规划模块，LLM 在其中剖析查询、计算自适应搜索深度，并合成优化查询变体。

❺ **多阶段混合执行**。为了克服单范式搜索的召回限制，该类别执行多引擎查询流水线，编排多维候选生成，随后进行下游重排序。

- **顺序混合路由**。该子类将检索范式串联成严格有序流水线，先用确定性谓词系统性地剪枝搜索空间，再执行细粒度语义抽取，例如先应用严格 SQL 日期过滤，再执行计算成本较高的向量搜索。MemoryOS 执行联合路由策略，包含粗粒度谓词评估，随后在隔离段内执行细粒度语义排序。它以代数方式融合基于规则的结构布尔过滤和密集语义相似度路由。

- **并行集成检索**。不同于顺序过滤，该方法通过同时向多个不同索引算法分发查询来最大化初始召回，然后执行后期融合和重排序，以优化聚合候选池，例如同时通过 BM25 和密集向量搜索获取候选，然后对结果进行交叉编码。Zep 同时执行余弦语义扫描、Okapi BM25 全文搜索和拓扑 BFS，随后通过 RRF、MMR 和计算开销大的交叉编码器模型优化精度。

---

## 图 5：记忆检索方法

图 5 展示五种检索方法：原生注意力检索、基于语义的密集检索、拓扑子图遍历、自治智能体路由和多阶段混合执行。多阶段混合执行又包括顺序混合路由和并行集成检索。

---

### 3.4 记忆维护

记忆维护关注记忆如何随时间被更新、维护、压缩、遗忘并最终移除。如图 6 所示，它捕捉记忆创建之后的动态行为，包括新信息如何纳入、过时或冲突内容如何修订，以及系统如何在有限资源下控制记忆增长。

❶ **基于时间戳的多版本**。该类别不是执行物理行删除，而是通过时间戳元数据和追加式日志保存历史连续性，并逻辑废弃过期事实。Zep 和 Mem0g 通过显式元数据变异运行，使用有效性标记和时间戳将过时或冲突关系标记为逻辑失效，从而避免物理删除。LightMem 采用追加式方法，增量插入带时间戳事实流；SimpleMem 使用 ISO-8601 时间戳，通过严格时间顺序优先级解决矛盾。综合这些技术，MemOS 利用结构化 Update API 执行差分写入，无缝更新来源 ID，生成多版本链。

❷ **容量驱动的物理驱逐**。与基于时间戳的多版本不同，该类别通过物理丢弃或无条件覆盖数据来管理无界记忆增长。它通过严格确定性约束或动态计算驱逐分数来执行物理剪枝。

- **基于约束的硬驱逐**。该子类通过确定性规则强制执行刚性边界，例如严格 FIFO 队列、固定序列边界或硬 token 限制，以无条件驱逐旧状态。MemAgent 执行结构性覆盖，实现一种程序化调度算法，在每个固定段边界无条件用新合成摘要块替换旧记忆序列。MEM1 通过系统强制截断机制执行，在活跃上下文阈值被突破时自动进行 FIFO 剪枝，驱逐旧标签。Letta 通过阈值刷写运行，经由受操作系统启发的队列管理器严格处理缓冲区容量；当 token 数超过终端限制时，它强制执行刷写序列，将旧消息驱逐到二级召回存储。

- **基于评分的优先级驱逐**。该子类不是依赖静态容量限制，而是通过连续计算时间衰减或访问频率分数，动态强制数据物理过时。MemoryOS 量化访问频率，使用一个 Heat 标量来平衡检索频率与指数时间衰减，并执行优先级驱逐，物理移除最低热度段。

❸ **LLM 驱动的语义整合**。该类别作为认知治理器运行，利用 LLM 在查询或持久化阶段之前动态解决逻辑冲突，并将冗余观察抽象为密集摘要。

- **内联语义压缩**。在活跃写入阶段，该子类动态评估新摄入数据与现有记忆节点，并在数据库事务提交前系统性合并冗余断言，例如将三个相似对话轮次压缩为一个密集摘要节点。SimpleMem 即时执行在线语义合成，在数据库事务提交前系统性地将结构相似断言合并为单一密集抽象。MemTree 使用核心调度操作，在所有父节点上递归触发语义总结提示，动态融合历史状态和新载荷。

- **工具驱动的 CRUD 执行**。不同于自动融合，该子类通过离散、程序化状态变异来实现维护，并由 LLM 驱动的工具接口显式发出 Create、Read、Update 或 Delete（CRUD）命令。Mem0 严格通过结构化 LLM 工具调用接口实现动态维护，其中包含 UPDATE 和 DELETE 等离散程序化状态变异。

❹ **连续参数优化**。该类别将状态更新完全从在线推理延迟中解耦，将繁重神经优化作为异步后台进程执行，修改实际模型参数，而非外部数据库模式，例如对夜间批次持续微调。例如，MemoRAG 使活跃推理 token 严格保持静态只读，并通过基于生成反馈强化学习（RLGF）的算法框架，在离线训练阶段专门优化抽取质量。

---

## 图 6：记忆维护方法

图 6 展示四类维护方式：基于时间戳的多版本、容量驱动的物理驱逐、LLM 驱动语义整合和连续参数优化。示例包括追加式日志中的有效/失效标记、FIFO 驱逐、低分数驱逐、内联语义压缩、工具驱动 CRUD、以及基于 RLGF / LoRA 的离线训练或微调。

---

## 4 端到端评估

本节围绕五个研究问题对智能体记忆系统进行系统评估。在五种不同基准工作负载和 11 个数据集上，我们将 12 个代表性记忆系统与基线进行比较，以刻画它们的性能。五个研究问题如下。

### 4.1 整体有效性（RQ1）

**实验设置。** 对于“不同智能体记忆系统是否能在各种工作负载上成功提升端到端任务表现？”这一问题，我们在三类端到端工作负载上评估 12 个代表性记忆系统和两个参考基线（Long Context 和 Embedding RAG），以考察记忆是否能超越底层 LLM 提升任务成功率。具体使用：（1）LoCoMo [20]，一个长对话 QA 基准，测试多轮交互中的情景、时间和开放域记忆，并报告四类查询上的类别级 Exact Match（EM）和 Answer F1 的未加权均值；（2）LongMemEval [31]，一个多会话长记忆基准，评估系统是否能够跨会话重新连接事实并对时间分布证据进行推理，报告 Substring EM、ROUGE-L F1、ROUGE-L Recall，以及来自 MemoryAgentBench [22] 的基于 GPT-5.4 的 LLM Judge Accuracy；（3）DB-Bench，它评估记忆是否支持来自 LifelongAgentBench [37] 的数据库操作过程执行，并报告 Exact Match（EM）和 Task Success Rate。

**O1（跨工作负载有效性）。** 没有任何单一记忆系统主导所有工作负载，但通过结构引导过滤保留任务关键证据的方法总体上最具竞争力。如图 7 所示，领先系统随工作负载变化：（1）结构感知系统在 LongMemEval 上领先，Zep 达到 48.0 的 LLM Judge Accuracy，Cognee 达到 35.3 的 ROUGE-L F1；（2）混合过滤在 LoCoMo 精确性上最强，MemOS 达到 11.5 的 Exact Match；（3）保留轨迹的记忆在 DB-Bench 上最强，Long Context 达到 48.20 EM，MemoChat 达到 55.40 Task Success Rate。

然而，在具有完整工作负载覆盖的方法中，MemoryOS 和 MemOS 整体上最接近前沿。这表明鲁棒性并不来自某种单一通用记忆形式，而是来自在最终匹配之前，以合适的抽象级别保留正确证据。具体而言：（1）时间或图组织记忆最适合跨会话聚合和事件顺序推理，例如 LongMemEval 中分散的个人事实；（2）摘要优先或粗到细路由适合在长但语义连贯的对话中进行精确落地，例如在 LoCoMo 中恢复特定日期或个人细节；（3）当正确性取决于中间状态变化和操作顺序时，保留轨迹的记忆是必要的，例如 DB-Bench 中依赖性的 UPDATE 和 INSERT 操作。

**O2（超越 Exact Match）。** 对于具有规范且直接落地输出的任务，EM 仍然有信息价值；但当正确性依赖于释义式综合或可执行成功时，EM 就不充分。如图 7 所示，Exact Match 在 LoCoMo 上仍是有意义信号，因为许多问题针对短的落地事实，MemOS 获得最佳 Exact Match。可是，在 LongMemEval 中，当通过 ROUGE-L 和 LLM Judge Accuracy 考虑语义等价时，强系统被更清楚地区分开来，说明跨会话推理经常产生正确答案，但这些答案并不共享单一规范表面形式。在 DB-Bench 中这一限制更明显：Long Context 获得最佳 Exact Match，但 MemoChat 获得明显更高的 Task Success Rate，说明精确输出匹配不能充分反映记忆是否支持成功执行。这表明，当答案短、规范且局部可验证时（例如 LoCoMo 中的场地名或对象属性），EM 最合适；但当任务需要跨会话综合或终态验证时，应补充其他指标。

**发现 1（工作负载对齐的记忆）。** RQ1 表明，强智能体记忆不是由单一通用表示定义的，而是由它对主导工作负载瓶颈的支持程度决定：（1）对于分散的跨会话推理，关系和时间感知检索最有效，例如 Zep 和 Cognee；（2）对于长但语义连贯的对话，粗到细过滤提升精确落地，例如 MemOS 和 MemoryOS；（3）对于有状态执行，保留交互轨迹比单纯精确词面匹配更关键，例如 Long Context。

---

## 图 7：LoCoMo、MemoryAgentBench（LongMemEval）、LifeLongAgentBench（DB-Bench）上的记忆系统有效性

图 7 包括 LongMemEval 的 Substring EM、ROUGE-L F1、ROUGE-L Recall、LLM Judge Accuracy，LoCoMo 的 EM 与 Answer F1，以及 DB-Bench 的 EM 与 Task Success Rate。它比较了参考基线、顺序上下文系统、结构拓扑系统和多范式混合系统。

---

### 4.2 记忆检索保真度（RQ2）

**实验设置。** 对于“记忆系统能够多准确地浮现查询所需已存证据？”这一问题，我们评估 8 个代表性记忆系统，独立于下游答案生成，考察证据级检索保真度。具体使用 LoCoMo [20]，该数据集为具有不同证据距离的查询提供源级黄金证据。我们报告：（1）Recall@K，即 top-k 检索 source-id 组中包含标注黄金证据即为命中；（2）六个证据距离 gap 分箱上的 Recall@10，分箱范围从 1-5 到 26-31，该距离定义为查询最终会话与最早支持证据之间的会话距离，用于衡量长程检索准确性。

**O3（结构化证据扩展）。** 检索保真度较少取决于早期浮现一个相关记忆，而更多取决于保留一种显式组织的记忆结构，使其能够收集完整且时间上遥远的证据。如图 8 所示，结果显示早期命中精度和整体证据完整性之间存在清晰差异：SimpleMem 获得最高 Recall@1（39.0），但 A-MEM 和 MemTree 在更大检索预算下明显更强，Recall@5 / @10 分别达到 69.5 / 85.9 和 59.7 / 80.5，并且随着证据距离 gap 增大仍然更稳定；相比之下，扁平 Embedding RAG 基线在最短 gap 分箱之后迅速下降。

这一模式表明，强记忆检索主要不是 top-1 排序问题，而是证据补全问题，其中所需支持可能很旧、分散或跨多个轮次，例如在不同会话中提到的个人细节或很久以后引用的带日期事件。更具体地说，结果指向三种检索行为：（1）面向压缩的记忆适合早期浮现一个高度相关条目，例如单个显著个人细节或近期会话事实；（2）链接或层级记忆组织更适合在排序结果中收集互补支持证据，例如组合不同会话中提到的宠物名或带日期事件；（3）扁平密集检索主要在所需证据仍接近当前上下文时保持竞争力，例如近期会话事实。因此，对于需要分散或时间遥远支持的查询，最具竞争力的系统是那些将记忆组织为结构化证据空间，而不是扁平相似度缓存的系统。

**发现 2（以证据为中心的记忆组织）。** RQ2 表明，检索质量更多取决于系统如何组织证据以便后续重构，而不是它能多好地首先排序一个相关记忆。具体而言：（1）早期定位和证据组装应被视为两个独立设计目标；（2）当支持证据分散或时间遥远时，显式结构，例如链接或层级，最有价值，A-MEM 和 MemTree 即是如此；（3）扁平相似度搜索主要适合短程访问。

---

## 图 8：LoCoMo 上记忆系统的检索结果

图 8 显示 Recall@1、Recall@5、Recall@10，以及 Recall@10 随证据距离 gap 变化的曲线。A-MEM、MemTree、SimpleMem、MemOS、MemoryOS 等系统相比普通 Embedding RAG 展现出更高的结构化证据召回能力。

---

### 表 2：记忆更新设置下的鲁棒性

| 方法 | LoCoMo Temporal EM | LoCoMo Temporal Answer F1 | LongMemEval Knowledge Update Substring EM | LongMemEval Knowledge Update ROUGE-L F1 | LongMemEval Temporal Reasoning Substring EM | LongMemEval Temporal Reasoning ROUGE-L F1 |
|---|---:|---:|---:|---:|---:|---:|
| Long Context | 8.1 | 26.9 | 20.0 | 18.0 | 12.0 | 24.0 |
| Embedding RAG | 1.6 | 7.9 | 20.0 | 17.8 | 10.7 | 22.7 |
| Mem0 | 3.2 | 6.0 | 15.6 | 17.1 | 10.7 | 22.4 |
| MemoChat | 2.4 | 15.4 | 8.9 | 12.9 | 10.7 | 25.3 |
| Cognee | 4.0 | 28.1 | 37.8 | 34.0 | 18.7 | 35.8 |
| Zep | 4.8 | 18.1 | 44.4 | 36.8 | 13.3 | 30.5 |
| MemTree | 5.6 | 18.6 | 31.1 | 30.6 | 8.0 | 29.9 |
| Letta (MemGPT) | 0.0 | 7.1 | 17.8 | 5.7 | 12.0 | 8.8 |
| LightMem | 4.0 | 20.1 | 15.6 | 20.2 | 12.0 | 28.6 |
| SimpleMem | 4.4 | 8.1 | 6.7 | 7.4 | 8.0 | 22.6 |
| MemOS | 8.9 | 28.0 | 28.9 | 30.5 | 12.0 | 31.1 |
| MemoryOS | 3.2 | 22.7 | 35.6 | 32.2 | 16.0 | 31.6 |
| A-MEM | 4.8 | 17.7 | 26.7 | 22.8 | 8.0 | 22.5 |

### 4.3 记忆演化鲁棒性（RQ3）

**实验设置。** 对于“智能体记忆系统能否可靠纳入修订事实、在更新后保留正确时间状态，并在不同答案主干下保持鲁棒？”这一问题，我们开展两个实验：（1）更新鲁棒性比较，评估系统是否能吸收事实修订并在更新后回答时间落地查询；（2）主干鲁棒性消融，测试仅改变 LLM 主干时该行为是否保持稳定。

在（1）更新鲁棒性比较中，表 2 在 LongMemEval [31] 的 Knowledge Update 和 Temporal Reasoning、以及 LoCoMo [20] 的 Temporal 上比较 11 个代表性记忆系统。LongMemEval 两个切片报告 Substring EM 和 ROUGE-L F1，LoCoMo 切片报告 EM 和 Answer F1。在（2）主干鲁棒性消融中，图 9 在 LoCoMo 上用 4 个 LLM 主干评估 6 个代表性记忆设置。

**O4（时间状态外部化）。** 没有任何单一记忆系统主导所有面向更新的切片，但通过结构化组织保存时间有效证据的方法总体上最具竞争力。如表 2 所示，领先系统随切片变化：（1）图或关系组织记忆在直接事实修订上最强，Zep 在 Knowledge Update 上以 44.4 Substring EM 和 36.8 ROUGE-L F1 领先；（2）关系组织检索在时间分散证据上最强，Cognee 在 Temporal Reasoning 上以 18.7 Substring EM 和 35.8 ROUGE-L F1 领先；（3）混合过滤记忆在精确最新状态落地上最强，MemOS 在 LoCoMo EM 上达到最高 8.9，而 Cognee 在 Answer F1 上达到最高 28.1。

然而，在完整覆盖各切片的方法中，Cognee、MemOS 和 MemoryOS 总体上最接近前沿，说明鲁棒性不是来自某种单一通用记忆形式，而是来自在正确结构层级保留正确时间证据。具体而言：（1）时间或图组织记忆最适合修订个人事实和带日期事件，例如在 LongMemEval 中聚合分散的偏好、购买或过去活动更新；（2）混合或粗到细过滤最适合正确性取决于当前有效状态的任务，例如在 LoCoMo 长但语义连贯的对话中恢复最新日期、属性或事件顺序；（3）当需要将陈旧提及与更新后事实分离时，扁平上下文累积或密集相似度最弱，例如在重复提及后区分早期个人细节与后续更正。

**O5（主干鲁棒性）。** 主干变化改变绝对答案质量的幅度，大于它改变哪条记忆流水线有效的幅度。这表明稳定更新行为主要在最终生成之前决定。如图 9 所示，更强生成器通常提升 Answer F1，但总体排序只发生温和变化：（1）MemOS 仍然是最强记忆型配置，分别达到 32.2、41.2、38.6 和 41.2；（2）唯一显著反转是局部的，在 GPT-5.4-mini 和 GPT-5.4 下，A-MEM 超过 MemTree。这种稳定性意味着更强主干主要在相关证据已被定位之后改善答案表达，而不是弥补较弱时间落地能力。比如，在当前 LoCoMo 时间输出的日期落地最新状态查询中，MemOS 在四个主干下都保持正确，而 Embedding RAG 始终错误。更具体地说，具有更强外部组织的方法为不同 LLM 产生更稳定的证据集，而更依赖 LLM 侧综合的方法表现出更大的跨主干波动。

**发现 3（时间更新保真度）。** RQ3 表明，可靠的更新后行为是流水线级设计问题，而不是纯模型容量问题。具体而言：（1）可修订性应被构建进记忆表示，使后续事实能够绑定到同一实体或事件，而不是作为无差别文本追加，Zep 和 Cognee 即是如此；（2）查询时选择性应匹配工作负载瓶颈，当任务需要当前有效状态时使用过滤或混合路由，例如 MemOS 和 MemoryOS；（3）LLM 扩展只有在落地成功后最有价值，因此更强主干应改进答案表达，而不应作为解决陈旧或冲突记忆的主要机制。

---

## 图 9：LLM 主干消融

图 9 比较了 Embedding RAG、LightMem、MemOS、MemTree 和 A-MEM 在 Qwen3-8B、DeepSeek-Chat、GPT-5.4-mini 和 GPT-5.4 等主干下的 LoCoMo Answer F1 与 Recall。结果显示主干变化影响绝对质量，但不会完全改变有效记忆流水线的排序。

---

### 4.4 长程记忆稳定性（RQ4）

**实验设置。** 对于“随着有效记忆跨度通过更长上下文或更远支持证据而增加，智能体记忆系统有多稳定？”这一问题，我们在 3 个基准上评估 12 个代表性记忆系统，以考察其对上下文长度增加和时间距离增加的鲁棒性。具体使用：（1）LongBench [3]，在问答中评估受控长上下文难度，报告 Short、Medium 和 Long 上下文长度分箱上的 Accuracy，以衡量上下文长度鲁棒性；（2）LongMemEval [31]，在先前交互数量增长时评估多会话记忆，按历史会话数量分箱报告 ROUGE-L F1，以衡量多会话稳定性；（3）LoCoMo [20]，在支持证据位于对话较早位置时评估记忆漂移，按最终会话与最早支持证据之间的证据距离 gap 分箱报告 Answer F1。

**O6（长程证据保存）。** 当证据通过显式关系链接或层级整合组织起来，而不是作为扁平文本直接匹配时，记忆在更长跨度上更稳定。如图 10 所示，在 LongBench 中，SimpleMem 从 Short 到 Medium 分箱几乎不变（Accuracy 从 35.2 到 34.9），而 Long Context 从 42.6 降至 19.0，说明一旦长输入积累干扰项，单纯更大提示并不能维持答案质量。在 LoCoMo 中，对比更明显：当证据 gap 变宽时，Embedding RAG 从 37.1 Answer F1 降至 7.4，而 Cognee、MemOS 和 MemoryOS 等图或整合记忆系统在同样分箱中仍保持明显更高表现；LongMemEval 也显示，保留跨会话结构的方法在更长历史上具有同样优势。

这表明长程场景的主要困难不是记忆量，而是表示是否能将遥远事实连接到回答所需抽象。更具体地说，图或时间组织记忆能保留遥远事实的实体-事件-时间关系，例如恢复许多会话前重复出现的个人事件；层级或摘要优先组织保存会话级结构，例如先定位相关会话，再解析特定局部细节，使 LLM 能在最终生成前缩小注意力范围。纯长上下文提示和扁平密集记忆不提供这两种支持，因此随着有效跨度增长退化更明显。

**发现 4（跨度结构化记忆）。** RQ4 表明，随着有效记忆跨度增长，主要挑战从存储更多历史转向选择合适抽象：（1）当长输入含有许多干扰项时，多视图过滤有帮助，例如 SimpleMem；（2）当支持事实被许多轮次或会话隔开时，关系感知索引有帮助，例如 Cognee 和 Zep；（3）当系统必须先识别相关会话再解析局部细节时，粗到细总结有帮助，例如 MemOS 和 MemoryOS。

---

## 图 10：长程记忆稳定性

图 10 包含：（a）LongBench 上下文长度鲁棒性；（b）LongMemEval 会话历史增长；（c）LoCoMo 时间证据距离漂移。比较对象包括 Long Context、Embedding RAG、Mem0、MemoChat、Cognee、Zep、MemTree、Letta、LightMem、SimpleMem、MemOS、MemoryOS 和 A-MEM。

---

### 4.5 记忆运行成本（RQ5）

**实验设置。** 对于“每个记忆系统在效用-延迟权衡和跨工作负载延迟足迹方面的运行成本是多少？”这一问题，我们使用运行器记录的统一时间开销轨迹评估 8 个代表性记忆系统。我们量化两方面：（1）效用-延迟权衡，通过 Avg. Operation Latency/Query 和 Normalized Utility 衡量；（2）跨工作负载延迟足迹，通过 Outlier-Filtered Avg. Total Latency/Query 衡量。

对于（1），Avg. Operation Latency/Query 被计算为记忆构建时间加查询时间，并被解释为具有累积或突发写入系统的摊销每查询成本；Normalized Utility 是当前 LoCoMo [20] 和 LongMemEval [31] 运行中六个 min-max 归一化答案质量指标的均值。对于（2），我们报告三个基准上的 Outlier-Filtered Avg. Total Latency/Query。

**O7（局部化维护）。** 成本效率最高的记忆机制，是那些将维护局限在有界记忆状态子集上的机制；反复重组大规模全局状态的机制效率最低。如图 11 所示：（1）在记忆增强系统中，LightMem 和 MemTree 占据最强效率前沿，LightMem 在 3.67 秒 Avg. Operation Latency/Query 下达到 48.3 Normalized Utility，MemTree 在 15.9 秒下达到 63.5；二者明显比 MemoChat（28.0，15.4 秒）、Mem0（21.4，35.9 秒）和 A-MEM（57.7，17.9 秒）更高效。（2）高效用结构化系统明显移动到昂贵一侧：MemoryOS 只有在 28.6 秒下才达到 82.0 Normalized Utility，Cognee 和 Zep 则分别在 116.5 秒和 155.1 秒后才超过 84 效用。（3）工作负载特定延迟视图在 LongBench 上进一步强化了同样区分：LightMem 保持在 17.3 秒，MemTree 为 116.7 秒，而 Mem0、MemoChat、MemoryOS 和 A-MEM 分别升至 374.2、460.2、490.0 和 552.1 秒。

这表明运行效率与系统是否使用结构的关系较弱，而更多取决于每次写入在结构中传播的范围。具体而言：（1）分段压缩和有界混合检索使 LightMem 保持在低成本区间；（2）路径局部树聚合使 MemTree 能在无需全局刷新时保留明显更多效用；（3）全图整合、多存储同步或反复重写整段记忆能产生更强组织，但随着记忆增长会施加最重运行成本。

**发现 5（运行扩展规则）。** RQ5 显示，效率由维护范围而非结构本身决定。（1）局部化更新和搜索产生最佳成本-效用平衡，例如 LightMem 和 MemTree；（2）更丰富组织只有在其维护避免广泛重计算时才有帮助，否则开销会抵消收益，例如 Cognee 和 MemoryOS；（3）在长上下文工作负载下，整段记忆协调成为主要成本驱动因素。

---

## 图 11：记忆系统运行成本

图 11 展示：（a）成本-效果前沿，横轴为平均每查询运行延迟（秒，对数尺度），纵轴为归一化效用；（b）三个基准上的运行延迟，包括 LoCoMo、LongMemEval 和 LongBench。图中区分参考基线、顺序上下文、结构拓扑和多范式混合系统。

---

## 5 细粒度组件比较

为了理解端到端性能差异背后的根本原因，我们将智能体记忆系统分解为四个基本模块。通过系统性地生成一次只修改一个模块的受控变体，我们评估每个模块对整体系统性能的贡献。

### 5.1 记忆表示与存储（M1）

**实验设置。** 对于“记忆抽象层级和结构组织如何影响事实保真度和下游推理有效性？”这一问题，我们评估三个关注表示的变体：（1）LightMem [7] 比较 User-Only Raw、User-Only Summary 和 User-Only Compressed。User-Only Raw 存储逐字用户话语；User-Only Summary 将每个会话重写为 LLM 生成的抽象摘要；User-Only Compressed 去除填充和冗余 token，同时保留原始措辞和事实内容。（2）MemTree [27] 比较浅层 Flat-biased 设置和 Deeper Tree 设置，以检验层级文本组织如何影响记忆保真度。（3）Mem0 比较默认存储和 Graph Store。我们使用 LoCoMo 评估组合推理，使用 LongMemEval 评估多会话事实检索，以衡量细粒度事实保存和多步推理之间的权衡，并报告 EM、Answer F1、Substring EM 和 ROUGE-L F1 等指标。

### 表 3：表示与存储机制消融

| 方法 | 变体 | LoCoMo EM | LoCoMo Ans. F1 | LongMemEval Substr. EM | LongMemEval ROUGE-L F1 |
|---|---|---:|---:|---:|---:|
| LightMem | User-Only Raw | 24.2 | 38.9 | 26.0 | 31.4 |
| LightMem | User-Only Summary | 8.5 | 15.6 | 11.7 | 17.4 |
| LightMem | User-Only Compressed | 23.6 | 38.6 | 10.7 | 19.1 |
| MemTree | Flat-biased | 18.2 | 30.7 | 23.0 | 29.9 |
| MemTree | Deeper Tree | 18.7 | 31.2 | 23.3 | 30.9 |
| Mem0 | Default | 3.2 | 6.2 | 9.3 | 16.5 |
| Mem0 | Graph Store | 3.0 | 6.5 | 8.3 | 15.9 |

**O8（内容保真度）。** 为维持事实回忆和推理质量，保留原始会话内容比增加抽象或层级更重要。表 3 显示，LightMem User-Only Raw 在所有四个指标上取得最佳结果，而 User-Only Compressed 在 LoCoMo 上接近原始版本（Answer F1：38.6 vs. 38.9；EM：23.6 vs. 24.2），但在 LongMemEval 上显著下降（Substring EM：10.7 vs. 26.0）；User-Only Summary 在两个基准上都明显更弱；更深的 MemTree 设置相比扁平设置仅带来有限收益。这说明主要性能边界是表示所保留的可恢复证据数量，而不是是否应用更强抽象或更深结构。

具体而言：（1）当任务需要恢复精确会话级细节时，原始文本最有效，例如回忆标题 “Nu, pogodi!”；（2）轻度压缩在保留主要含义时仍可支持组合推理，但对于精确细节匹配不可靠，例如关联两个早期事件但漏掉具体日期或姓名；（3）更深层级可以改善组织，但不能恢复表示过程中被移除的信息，例如父节点有助于导航相关会话，却无法恢复被省略细节。

**发现 6（表示粒度）。** M1 表明，保留可用证据比让记忆更紧凑或更结构化更重要。（1）高保留形式最支持精确细节恢复，如 LightMem User-Only Raw；（2）轻度压缩可以保留推理，但削弱精确匹配，如 LightMem User-Only Compressed；（3）层级主要改善访问，但不能恢复被移除内容，MemTree 的 Deeper Tree 变体反映了这一点。

### 5.2 记忆抽取（M2）

**实验设置。** 对于“写入时抽取选择如何影响事实保真度和下游推理有效性？”这一问题，我们比较三组抽取相关变体：（1）MemoChat [18]，比较 Heuristic Topic 和 LLM Topic 分割；（2）MemOS [15]，比较同一 tree_text 后端上的 Fast Memorize 和 Fine Memorize；（3）LightMem [7]，比较 User-Only Raw 和 Hybrid Raw，分别只从用户轮次抽取原始记忆，或从用户和助手轮次共同抽取原始记忆。我们使用 LongMemEval 衡量多会话事实检索保真度，使用 LoCoMo 衡量下游多步推理，并报告 EM、Answer F1、Substring EM 和 ROUGE-L F1 等指标。

### 表 4：记忆抽取策略消融

| 方法 | 变体 | LoCoMo EM | LoCoMo Ans. F1 | LongMemEval Substr. EM | LongMemEval ROUGE-L F1 |
|---|---|---:|---:|---:|---:|
| MemoChat | Heuristic Topic | 23.0 | 33.5 | 10.7 | 18.6 |
| MemoChat | LLM Topic | 22.5 | 34.4 | 7.3 | 15.9 |
| MemOS | Fast Memorize | 25.5 | 40.8 | 20.7 | 26.1 |
| MemOS | Fine Memorize | 2.5 | 5.0 | 22.3 | 30.2 |
| LightMem | User-Only Raw | 24.2 | 38.9 | 26.0 | 31.4 |
| LightMem | Hybrid Raw | 25.5 | 39.7 | 25.3 | 31.4 |

**O9（保留覆盖的抽取）。** 保留覆盖的写入时抽取在事实检索和下游推理之间提供最稳定平衡。如表 4 所示，MemoChat Heuristic Topic 在 LongMemEval 上优于 LLM Topic（Substring EM：10.7 vs. 7.3；ROUGE-L F1：18.6 vs. 15.9），同时 LoCoMo 几乎不变（23.0 / 33.5 vs. 22.5 / 34.4 EM / Answer F1）。MemOS Fast Memorize 在 LoCoMo 上远超 Fine Memorize（EM：25.5 vs. 2.5；Answer F1：40.8 vs. 5.0），尽管 LongMemEval 分数更低（Substring EM：20.7 vs. 22.3；ROUGE-L F1：26.1 vs. 30.2）。LightMem Hybrid Raw 相比 User-Only Raw 轻微提升 LoCoMo（EM：25.5 vs. 24.2；Answer F1：39.7 vs. 38.9），LongMemEval 结果几乎不变（Substring EM：25.3 vs. 26.0；ROUGE-L F1 均为 31.4）。

这些结果表明，更宽泛、更少选择性的抽取更好地保留下游可回答性所需上下文，即使更选择性的抽取在词面事实检索上带来有限增益。具体而言：（1）保守主题分组不太可能拆分持续话题或孤立短暂插曲，例如一次性提到的爱好；（2）较轻的记忆化更可能保留后续需要组合推理的细节；（3）同时包含用户和助手轮次能保留仅用户抽取可能遗漏的澄清线索，例如日期或修正措辞。

**发现 7（晚过滤原则）。** M2 表明，记忆抽取应在写入时保留上下文，而不是激进过滤细节：（1）更粗分段通过将相关线索放在一起，有助于跨话题线索的问题；（2）有限重写通过保留只有在之后组合时才重要的细节，支持组合推理；（3）存储用户和助手轮次有助于澄清密集的对话，因为它保留了后续访问时需要的精炼表述。

### 5.3 记忆检索与路由（M3）

**实验设置。** 对于“检索融合和推理中介路由如何影响检索相关性和来源敏感精度？”这一问题，我们比较两组变体：（1）A-MEM 的 Hybrid-Balanced 与 Hybrid Sparse-Leaning，前者使用适度密集-稀疏融合，后者增加稀疏贡献；（2）SimpleMem 的 No Planning、Planning Only 和 Planning + Reflect，分别对应直接检索、加入显式规划步骤、以及进一步引入轻量反思阶段。我们使用 LongMemEval 衡量分散历史检索相关性，使用 LoCoMo 评估来源敏感记忆访问和支持记忆识别，报告 Answer F1、Recall、Substring EM 和 ROUGE-L F1。

### 表 5：检索与路由机制消融

| 方法 | 变体 | LoCoMo Ans. F1 | LoCoMo Recall | LongMemEval Substr. EM | LongMemEval ROUGE-L F1 |
|---|---|---:|---:|---:|---:|
| A-MEM | Hybrid-Balanced | 24.6 | 49.9 | 27.5 | 25.9 |
| A-MEM | Hybrid Sparse-Leaning | 23.0 | 44.3 | 24.3 | 22.8 |
| SimpleMem | No Planning | 18.7 | 86.4 | 17.0 | 22.9 |
| SimpleMem | Planning Only | 20.7 | 90.6 | 21.7 | 27.9 |
| SimpleMem | Planning + Reflect | 20.0 | 88.6 | 21.3 | 26.1 |

**O10（规划与融合）。** 显式规划和平衡检索融合对检索有效性的提升最强。如表 5 所示，A-MEM 在 Hybrid-Balanced 下表现最佳，达到 24.6 Answer F1 和 27.5 Substring EM，相比 Hybrid Sparse-Leaning 的 23.0 和 24.3 更高；SimpleMem 在 Planning Only 下表现最佳，达到 20.7 Answer F1、90.6 Strict Recall、21.7 Substring EM 和 27.9 ROUGE-L F1，高于 No Planning 和 Planning + Reflect。

这些结果说明，更强检索与路由性能来自添加有用结构，而不是简单增加稀疏匹配或额外推理步骤。具体而言：（1）适度融合相比偏稀疏融合更能同时保留答案质量和相关性，例如语义相关但词面多变的事实；（2）显式规划相比直接检索持续带来提升，例如多约束记忆查询；（3）在规划之上加入反思没有进一步收益，说明额外深思可能削弱而非改进路由决策。

**发现 8（检索策略指导）。** M3 表明，检索质量最受定向结构提升，而非额外复杂性提升：（1）当证据语义相关但词面多样时，适度混合融合更可取；（2）轻量规划对受约束记忆查找有效；（3）一旦路线已经指定，额外反思收益有限，主要增加开销。

### 5.4 记忆维护（M4）

**实验设置。** 对于“整合激进程度、刷写时机和摘要粒度如何影响更新正确性与长程记忆一致性？”这一问题，我们比较两组与维护相关的变体：（1）MemoChat 默认多主题整合与 Topic1，后者强制每个窗口成为单主题摘要；（2）MemoryOS 默认即时整合、Delayed-Flush 和 Conservative-Merge。Delayed-Flush 扩大短期缓冲区后再写入后端；Conservative-Merge 提高主题相似度阈值，从而更严格地同化。我们在 LoCoMo 上评估这些维护选择是否保留更新事实和连贯记忆使用。

**O11（保守整合）。** 保守整合比延迟刷写或过粗总结更能维护与答案相关的记忆。如图 12 所示，更严格的合并变体 MemoryOS（Conservative-Merge）将默认 MemoryOS 从 23.2 提升到 23.5 Answer F1，从 22.4 提升到 22.8 Substring EM；而延迟刷写将同一系统降至 20.6 / 19.5；在 MemoChat 中，强制单主题摘要也弱于默认设置，为 16.2 / 16.8，相比默认的 16.6 / 18.4；Long Context 在 Substring EM 上最高，为 23.7。

这表明维护最有效的方式，是在选择性整合证据时既不让证据未解决，也不过度压缩；同时，原始上下文仍然更好地保留精确措辞。具体而言：（1）保守合并可以保留相关细节以便后续重组，例如分散提到的爱好；（2）延迟刷写会在检索前让更多证据处于未解决状态，例如跨轮次拆分的活动。

**发现 9（维护设计原则）。** M4 表明，记忆维护在平衡更新机制下效果最好：（1）保守集成保留跨轮次链接以支持长程推理；（2）延迟刷写使近期证据在查询时保持碎片化；（3）过粗摘要会掩盖稀疏但有用的线索。

---

## 图 12：维护策略消融

图 12 在 LoCoMo 上比较 Long Context、MemoryOS 默认、MemoryOS Conservative-Merge、MemoryOS Delayed-Flush、MemoChat 默认和 MemoChat Topic1 的 Answer F1 与 Substring EM。结果表明保守合并略优于默认，延迟刷写和过粗摘要会损害性能。

---

## 6 结论

我们从数据管理视角出发，对现有智能体记忆系统进行了全面综述。我们对典型智能体记忆系统进行了完整端到端性能评估，并探索了它们适合的应用场景。此外，我们通过构建多个记忆模块变体，深入研究各个构建块的影响，从而识别表示、抽取、路由和维护中的最有效方法，以及影响运行成本和长程稳定性的最重要因素。最后，我们总结发现，为用户选择合适记忆架构提供指导，并概述有前景的研究方向。我们也将发布测试平台和评估框架。

---

## 参考文献

[1] Claude Code. (Anthropic). https://www.claude.com/product/claude-code

[2] Anthropic Engineering. 2025. Effective context engineering for AI agents. https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents.

[3] Yushi Bai, Xin Lv, Jiajie Zhang, Hongchang Lyu, Jiankai Tang, Zhidian Huang, Zhengxiao Du, Xiao Liu, Aohan Zeng, Lei Hou, Yuxiao Dong, Jie Tang, and Juanzi Li. 2024. LongBench: A Bilingual, Multitask Benchmark for Long Context Understanding. In ACL (1). Association for Computational Linguistics, 3119-3137.

[4] Liana Caminal et al. 2025. Filtered Vector Search: State-of-the-art and Research Opportunities. In Proceedings of the VLDB Endowment, Vol. 18. 5488-5491.

[5] Prateek Chhikara, Dev Khant, Saket Aryan, Taranjeet Singh, and Deshraj Yadav. 2025. Mem0: Building Production-Ready AI Agents with Scalable Long-Term Memory. arXiv preprint arXiv:2504.19413.

[6] Pengfei Du. 2026. Memory for Autonomous LLM Agents: Mechanisms, Evaluation, and Emerging Frontiers. arXiv preprint arXiv:2603.07670.

[7] Jizhan Fang, Xinle Deng, Haoming Xu, Ziyan Jiang, Yuqi Tang, Ziwen Xu, Shumin Deng, Yunzhi Yao, Mengru Wang, Shuofei Qiao, Huajun Chen, and Ningyu Zhang. 2025. LightMem: Lightweight and Efficient Memory-Augmented Generation. CoRR abs/2510.18866. arXiv:2510.18866 doi:10.48550/ARXIV.2510.18866

[8] Yunfan Gao, Yun Xiong, Xinyu Gao, Kangxiang Jia, Jinliu Pan, Yuxi Bi, Yi Dai, Jiawei Sun, Qianyu Guo, Meng Wang, and Haofen Wang. 2023. Retrieval-Augmented Generation for Large Language Models: A Survey. CoRR abs/2312.10997.

[9] Google. 2025. Memory - Agent Development Kit (ADK). https://google.github.io/adk-docs/sessions/memory/.

[10] Yifan Hu, Siyin Liu, Yifei Yue, Guoqiang Zhang, Benyou Liu, Fengbin Zhu, Jingkuan Lin, et al. 2025. Memory in the Age of AI Agents. arXiv preprint arXiv:2512.13564.

[11] Guozhang Kang, Zhenying Ge, Jie Hu, Xinyuan Zhang, Li Wang, and Jianfeng Zhan. 2025. BigVectorBench: Heterogeneous Data Embedding and Compound Queries are Essential in Evaluating Vector Databases. Proceedings of the VLDB Endowment 18, 6, 1536-1549.

[12] Jiazheng Kang, Mingming Ji, Zhe Zhao, and Ting Bai. 2025. Memory OS of AI Agent. In EMNLP. Association for Computational Linguistics, 25961-25970.

[13] Arijit Khan, Yuyu Luo, Wenjie Zhang, Mingjie Zhou, and Xiaofang Zhou. 2025. Retrieval-augmented Generation (RAG): What is There for Data Management Researchers? ACM SIGMOD Record 54, 4.

[14] Guoliang Li, Xuanhe Zhou, and Xinyang Zhao. 2024. LLM for Data Management. Proc. VLDB Endow. 17, 12, 4213-4216.

[15] Zhiyu Li et al. 2025. MemOS: A Memory OS for AI System. CoRR abs/2507.03724. arXiv:2507.03724 doi:10.48550/ARXIV.2507.03724

[16] Jiaqi Liu, Yaofeng Su, Peng Xia, Siwei Han, Zeyu Zheng, Cihang Xie, Mingyu Ding, and Huaxiu Yao. 2026. SimpleMem: Efficient Lifelong Memory for LLM Agents. CoRR abs/2601.02553.

[17] Shu Liu et al. 2026. Supporting Our AI Overlords: Redesigning Data Systems to be Agent-First. In Proceedings of the 16th Annual Conference on Innovative Data Systems Research (CIDR).

[18] Junru Lu, Siyu An, Mingbao Lin, Gabriele Pergola, Yulan He, Di Yin, Xing Sun, and Yunsheng Wu. 2023. MemoChat: Tuning LLMs to Use Memos for Consistent Long-Range Open-Domain Conversation. CoRR abs/2308.08239. arXiv:2308.08239 doi:10.48550/ARXIV.2308.08239

[19] Yuyu Luo, Guoliang Li, Ju Fan, and Nan Tang. 2026. Data Agents: Levels, State of the Art, and Open Problems. arXiv preprint arXiv:2602.04261. SIGMOD 2026 Tutorial.

[20] Adyasha Maharana, Dong-Ho Lee, Sergey Turishcheva, Kezhen Nham, Golnaz Jandaghi, Jay Pujara, and Xiang Ren. 2024. Evaluating Very Long-Term Conversational Memory of LLM Agents. In Proceedings of ACL.

[21] Vasilije Markovic, Lazar Obradovic, László Hajdu, and Jovan Pavlovic. 2025. Optimizing the Interface Between Knowledge Graphs and LLMs for Complex Reasoning. CoRR abs/2505.24478.

[22] MemoryAgentBench Team. 2026. Evaluating Memory in LLM Agents via Incremental Multi-Turn Interactions. In ICLR.

[23] Microsoft. 2025. Introducing Copilot Memory: A More Productive and Personalized AI. https://techcommunity.microsoft.com/blog/microsoft365copilotblog/introducing-copilot-memory.

[24] OpenAI. 2026. Context Engineering for Personalization - State Management with Long-Term Memory Notes using OpenAI Agents SDK. https://developers.openai.com/cookbook/examples/agents_sdk/context_personalization/.

[25] Charles Packer, Vivian Fang, Shishir G. Patil, Kevin Lin, Sarah Wooders, and Joseph E. Gonzalez. 2023. MemGPT: Towards LLMs as Operating Systems. arXiv preprint arXiv:2310.08560.

[26] Preston Rasmussen, Pavel Paliychuk, Travis Beauvais, and Jesse Ryan. 2025. Zep: A Temporal Knowledge Graph Architecture for Agent Memory. arXiv preprint arXiv:2501.13956.

[27] Alireza Rezazadeh, Zichao Li, Wei Wei, and Yujia Bao. 2025. From Isolated Conversations to Hierarchical Schemas: Dynamic Tree Memory Representation for LLMs. ICLR 2025. https://openreview.net/forum?id=moXtEmCleY

[28] Harmanpreet Singh, Nikhil Verma, Yixiao Wang, Manasa Bharadwaj, Homa Fashandi, Kevin Ferreira, and Chul Lee. 2024. Personal Large Language Model Agents: A Case Study on Tailored Travel Planning. EMNLP Industry Track, 486-514. doi:10.18653/v1/2024.emnlp-industry.37

[29] Haoran Tan, Zeyu Zhang, Chen Ma, Xu Chen, Quanyu Dai, and Zhenhua Dong. 2025. MemBench: Towards More Comprehensive Evaluation on the Memory of LLM-based Agents. ACL Findings, 19336-19352.

[30] Zhiwei Tang et al. 2026. LLM Agent Memory: A Survey from a Unified Representation. arXiv preprint arXiv:2603.0359.

[31] Di Wu, Hongwei Wang, Wenhao Yu, Yuwei Zhang, and Kai-Wei Chang. 2024. LongMemEval: Benchmarking Chat Assistants on Long-Term Interactive Memory. arXiv preprint arXiv:2410.10813.

[32] Yanchen Wu et al. 2026. Memory in the LLM Era: Modular Architectures and Strategies in a Unified Framework. Proceedings of the VLDB Endowment.

[33] Wujiang Xu et al. 2025. A-MEM: Agentic Memory for LLM Agents. arXiv preprint arXiv:2502.12110.

[34] Chao Yang, Chuan Zhou, Yanghua Xiao, Shuai Dong, Liang Zhuang, et al. 2026. Graph-based Agent Memory: Taxonomy, Techniques, and Applications. arXiv preprint arXiv:2602.05665.

[35] Hongli Yu et al. 2025. MemAgent: Reshaping Long-Context LLM with Multi-Conv RL-based Memory Agent. CoRR abs/2507.02259.

[36] Zeyu Zhang et al. 2025. A Survey on the Memory Mechanism of Large Language Model based Agents. ACM Transactions on Information Systems.

[37] Junhao Zheng et al. 2025. LifelongAgentBench: Evaluating LLM Agents as Lifelong Learners. CoRR abs/2505.11942.

[38] Junhao Zheng et al. 2025. Lifelong Learning of Large Language Model-based Agents: A Roadmap. IEEE Transactions on Pattern Analysis and Machine Intelligence.

[39] Wanjun Zhong, Lianghong Guo, Qiqi Gao, He Ye, and Yanlin Wang. 2024. MemoryBank: Enhancing Large Language Models with Long-Term Memory. In AAAI, 19724-19731.

[40] Wei Zhou, Xuanhe Zhou, Qikang He, Guoliang Li, Bingsheng He, Quanqing Xu, and Fan Wu. 2026. Automating Database-Native Function Code Synthesis with LLMs. Proc. ACM Manag. Data 3, 4, 141:1-141:26.

[41] Zijian Zhou, Ao Qu, Zhaoxuan Wu, Sunghwan Kim, Alok Prakash, Daniela Rus, Jinhua Zhao, Bryan Kian Hsiang Low, and Paul Pu Liang. 2025. MEM1: Learning to Synergize Memory and Reasoning for Efficient Long-Horizon Agents. CoRR abs/2506.15841.

