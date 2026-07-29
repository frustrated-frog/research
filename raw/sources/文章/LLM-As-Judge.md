# Ask, Don’t Judge：用于可解释 LLM 评估与自我改进的二元问题

Sangwoo Cho¹，Kushal Chawla¹，Pengshan Cai¹，Zefang Liu¹，Chenyang Zhu¹，Shi-Xiong Zhang¹，Sambit Sahu¹

¹Capital One，AI Foundations，McLean，VA 22102，USA。通讯作者：Sangwoo Cho <sangwoo.cho@capitalone.com>。

已被 ICML 2026（韩国首尔）第二届组合式学习研讨会接收。版权所有 2026，归作者所有。

arXiv:2606.27226v1 [cs.AI] 2026 年 6 月 25 日

## 摘要

评估 LLM 输出仍然是自然语言处理中的一个主要瓶颈：人工评估昂贵且缓慢，词汇指标与人类对开放式生成的判断相关性较差，而整体式 LLM 裁判往往给出不透明的分数，难以调试。

**我们提出 BINEVAL**，这是一个将评估标准分解为原子级二元问题，并把所得判定聚合为可解释、多维分数的框架。给定一个任务提示词，一个元提示词会生成细粒度的评估问题，然后一个 LLM 针对每个输出独立回答这些问题，从而同时给出透明的问题级反馈和经过校准的整体分数。这种分解使评估更容易检查、更容易诊断，也可以直接用于提示词改进。

在 SummEval、Topical-Chat 和 QAGS 上，BINEVAL 能够匹配或超过包括 UniEval 和 G-Eval 在内的强基线，尤其在 QAGS 等事实一致性基准上表现突出。除了与人类判断具有竞争性的相关性之外，BINEVAL 还更好地匹配人类分数分布，并避免了先前 LLM 裁判中常见的天花板效应，从而能更好地区分边界样本和明显有缺陷的输出。我们进一步表明，相同的问题级反馈能够支持迭代式提示词优化：在自更新和跨模型更新两种设置下，它都能改进摘要任务中的评估器提示词，以及 IFBench 上的生成提示词。总体而言，BINEVAL 提供了一个任务无关、无需训练且可解释的评估框架，它将强实证性能与实用的诊断和优化价值结合起来。

## 1. 引言

大型语言模型（LLM）的快速进展让“生成”变得容易，却让“评估”变得困难。现代系统能够在摘要、对话、推理和指令遵循等任务中生成流畅且语境适切的输出，但评估这些输出仍然是一个主要瓶颈。人工评估缓慢且昂贵；ROUGE（Lin, 2004）、BLEU（Papineni et al., 2002）和 BERTScore（Zhang et al., 2020）等词汇指标会遗漏语义正确性和事实性；整体式 LLM 裁判（Zheng et al., 2023；Liu et al., 2023）则经常返回难以诊断的不透明分数。

这一瓶颈在迭代式开发中尤其昂贵。比较提示词、模型或解码策略时，所需反馈不仅要准确，还要可操作。单个标量分数通常不够：如果一个摘要得到中等评分，我们仍然不知道问题到底是事实不一致、相关性弱、内容缺失，还是流畅性差。

我们的前提很简单：不要让模型做一个宽泛判断，而是向它提出一组小而可检查的问题。因此，我们提出 BINEVAL。它把每个评估标准分解为原子级的是/否问题，并将所得判定聚合为可解释分数。该分解把评估从黑箱裁决变为结构化诊断信号，使检查、调试以及改进评估器和生成器都更容易。

BINEVAL 包含三个组件。第一，元提示词把任务提示词分解为按评估维度组织的原子问题。第二，评估器独立回答每个问题，并把答案聚合为按维度的分数和整体分数。第三，一个两阶段优化循环使用问题级反馈改进评估器提示词和生成提示词。

我们在 SummEval（Fabbri et al., 2021）、Topical-Chat（Mehri & Eskenazi, 2020）和 QAGS（Wang et al., 2020）上评估 BINEVAL，并研究了摘要和 IFBench 上的迭代式提示词更新。

我们的贡献如下：

- 一个通用的可解释评估框架。我们将评估标准分解为原子级的是/否问题，得到一种任务无关且模块化的方法。
- 无需任务特定训练即可获得强性能。BINEVAL 在 SummEval、Topical-Chat 和 QAGS 上能够匹配或超过训练式评估器和整体式 LLM 裁判。
- 迭代式提示词改进。我们引入一个两阶段优化循环，可改进摘要任务和 IFBench 的提示词。
- 可调试分数。每个 BINEVAL 分数都建立在带解释的单独判定之上，使评估器行为更易检查和诊断。

## 2. 相关工作

**传统评估指标。** 词汇重叠指标——ROUGE（Lin, 2004）、BLEU（Papineni et al., 2002）和 METEOR（Banerjee & Lavie, 2005）——仍然是摘要和翻译评估中的标准方法，但它们往往难以捕捉开放式生成中的语义等价性。BERTScore（Zhang et al., 2020）和 MoverScore（Zhao et al., 2019）等基于嵌入的指标通过在表示空间中操作来改进语义匹配，而 BARTScore（Yuan et al., 2021）等基于生成的指标则把评估框定为文本生成。更近期的无参考方法 ParaPLUIE（Lemesle et al., 2025）使用模型困惑度衡量意义保留，不需要黄金参考；OmniScore（Alam et al., 2026）等框架使用确定性的学习评估器来支持可扩展的多语言评估。

**LLM-as-Judge。** 近期工作越来越多地把 LLM 本身用作评估器。G-Eval（Liu et al., 2023）使用思维链推理，然后给出李克特量表评分；AlpacaEval（Li et al., 2023）以及 MT-Bench / Chatbot Arena（Zheng et al., 2023）依赖成对比较或偏好式判断。这一范式也扩展到了专门的开源评估器，例如 Prometheus 2（Kim et al., 2024），它近似人类和专有模型判断的深度。然而，这些裁判仍然容易受到位置、冗长性和自我增强偏差的影响（Zheng et al., 2023）。JudgeBiasBench（Zhou et al., 2026）等近期基准进一步系统化了这些问题，提供了裁判偏差分类并提出去偏策略。

**多维评估。** 多维评估旨在把质量分解为可解释的方面，例如连贯性、忠实性、信息量和相关性。UniEval（Zhong et al., 2022）是一个重要先例：它把评估重新表述为布尔问答，并微调一个基于 T5 的评估器以支持多个维度。更近期的工作也类似地把评估分解为信息性和忠实性等方面（Alam et al., 2026）；QAEval（Yue et al., 2025）等混合框架则结合基于规则的可靠性与评估器混合模型，用于开放式生成任务。这些方法共同强化了把评估拆成更小、更结构化判断的价值。

**用于评估的原子分解。** FActScore（Min et al., 2023）通过把长文本生成分解为原子事实并逐一验证，开创了“先分解、后验证”的范式。ARES（Saad-Falcon et al., 2024）和 RAGAS（Es et al., 2024）等相关框架把类似分解思想扩展到检索增强生成；OpenFActScore（Lage & Ostermann, 2025）则支持使用原子评估进行开源事实核查。这些方法表明细粒度分解可以改善事实评估，不过它们通常分解的是生成内容，而不是评估标准本身。

**提示词优化。** 提示词优化已经逐渐从手工指令工程转向自动化和程序化细化。DSPy（Khattab et al., 2023）提供了一个声明式、自改进语言模型流水线框架；MIPRO（Opsahl-Ong et al., 2024）等算法对指令和示例进行贝叶斯搜索。OPRO（Yang et al., 2023）和 APE（Zhou et al., 2023）同样使用语言模型迭代生成和细化提示词。更新的方法如 MARS（Zhang et al., 2025）引入多智能体苏格拉底式优化；LLM-AutoDiff（Yin & Wang, 2025）则把文本输入视为图结构工作流中的可训练参数。这些方法启发我们使用由分歧驱动的提示词细化，将其作为有针对性的优化信号。

## 3. 方法

我们从三部分介绍 BINEVAL：二元问题生成（第 3.1 节）、二元评估与打分（第 3.2 节）以及迭代式提示词优化（第 3.3 和 3.4 节）。

### 3.1 二元问题生成

设 T 表示定义生成要求的任务提示词，例如摘要指令、对话系统提示词或指令遵循规范。我们定义一个分解函数，把 T 映射到一组二元问题：

```text
Q = F_LLM(T; M) = {q1, q2, ..., qN}。
```

其中 M 是一个元提示词，它指示 LLM 执行两步分解。

**步骤 1——总结。** 我们首先把任务提示词 T 总结为一组明确的要求 R = {r1, r2, ..., rK}。每个要求 rk 捕捉一个不同的评估标准，例如输出是否包含某个关键信息，或是否遵守某个格式约束。这个总结步骤旨在帮助模型在进行更细粒度分解之前，形成对完整任务的连贯表示。

**步骤 2——分解。** 对每个要求 rk，我们生成一个或多个二元问题，使得回答“是”表示输出满足该要求，回答“否”表示违反要求。隐含包含多个子任务的要求会被分解为单独的问题，并且每个问题都会配一个简短的违规示例，以澄清负例。这一设计受到先前工作的启发：复杂推理通常可以通过把任务分解为更简单的子问题并按顺序或模块化求解来改善（Zhou et al., 2022；Khot et al., 2022）。在我们的设置中，同样的直觉表明，如果模型回答关于简化子任务的目标化二元问题，而不是给出一个整体判断，评估会变得更容易。

这些问题可以被组织为评估维度。对于一组维度 D，例如 coherence（连贯性）、consistency（一致性）、fluency（流畅性）和 relevance（相关性），问题划分为：

```text
Q = ⋃_{d∈D} Qd，
```

其中 Qd 包含特定于维度 d 的问题。元提示词 M 是任务无关的：同一个元提示词可以为摘要、对话、指令遵循或其他任何任务生成合适的二元问题，唯一变化的是 T。

### 3.2 二元评估与打分

给定一个评估器 LLM E，一个输入 x（例如源文档、转录文本或指令），一个输出 y（例如生成摘要、对话回复或补全），以及一个二元问题 qi，我们定义二元评估函数：

```text
f_E(x, y, qi) ∈ {0, 1}，
```

其中，如果评估器回答“是”，则 f_E(x, y, qi) = 1，否则为 0。除了每个二元判定之外，评估器还会产生自然语言解释 ei，从而实现可解释性。

维度 d 的按维度分数为：

```text
S_d(x, y) = (1 / |Qd|) * Σ_{qi∈Qd} f_E(x, y, qi)。
```

所有 N 个问题上的整体分数为：

```text
S(x, y) = (1 / N) * Σ_{i=1}^{N} f_E(x, y, qi)。
```

两个分数都位于 [0, 1]，其中 1 表示满足所有标准。为了与使用不同尺度的现有评估框架比较，可以通过仿射缩放把分数从 [0, 1] 映射到任意目标区间 [a, b]：

```text
S'(x, y) = S(x, y) · (b - a) + a。
```

### 3.3 跨模型提示词更新

BINEVAL 的二元问题框架支持评估器之间的跨模型提示词更新。关键洞见是：源评估器和目标评估器在特定二元问题上的分歧提供了细粒度的改进信号。不同于整体分数差异，二元问题分歧能够精确指出哪些标准在不同模型之间被不一致地判断。这样就可以使用更强的源模型作为参考，并迭代更新另一个通常较弱的目标模型的提示词，直到其评估行为更接近源模型。此外，当把模型迁移到不同模型家族时，更新提示词以保持相似性能也很有用。

设 Esrc 表示作为参考的源评估器，Etgt 表示我们希望改进其提示词 PE 的目标评估器。设 P_E^(t) 表示目标评估器在第 t 次迭代时的提示词。

在每次迭代 t，优化分五步进行：

1. **评估。** 对每个测试样例 (xj, yj)，从两个模型获取二元评估：

```text
A_j^src = { f_Esrc(xj, yj, qi) }_{i=1}^{N}
A_j^tgt = { f_Etgt(xj, yj, qi; P_E^(t-1)) }_{i=1}^{N}
```

2. **识别分歧。** 计算两个评估器不一致的问题集合：

```text
Δj = { qi ∈ Q : A_j^src(qi) ≠ A_j^tgt(qi) }。
```

3. **提取经验。** 一个记笔记 LLM Lnote 在上下文中分析每个分歧，提取一般化经验：

```text
Lj = Lnote(xj, yj, A_j^src, A_j^tgt, Δj)。
```

语义去重过程为：如果新经验 ℓnew 与记忆 M 中已有经验 ℓk 相似，则合并；否则添加新经验。最终唯一经验集合为：

```text
L_unique = Dedup(⋃_j Lj)。
```

4. **更新提示词。** 对每个唯一经验 ℓk ∈ Lunique，一个更新器 LLM 识别当前提示词中的相关子串 sk，并产生纳入该经验的修订子串 s'k：

```text
P_E^(t) ← P_E^(t).replace(sk, s'k)。
```

当目标评估器在所有维度上的分数都在容差 ε 内匹配源评估器时，循环终止：

```text
|S_d^{tgt,(t)} - S_d^{src}| < ε，∀ d ∈ D，
```

或者等价地，当目标评估器在所有维度上达到或超过源评估器时终止。完整算法见附录 1。

### 3.4 自提示词更新

同样的二元问题框架也可用于生成中的自提示词更新。与把一个评估器对齐到另一个模型不同，该过程通过使用评估器识别出的失败作为对自身输出的反馈，迭代改进生成器。给定生成 LLM LG，其在第 t 次迭代时的提示词为 P_G^(t)：

1. **生成。** 使用当前提示词产生输出：

```text
y_j^(t) = L_G(xj; P_G^(t))。
```

2. **评估。** 使用可能已经改进的评估器为每个输出打分，并收集失败问题：

```text
Ej = { (qi, ei) : f_E(xj, y_j^(t), qi) = 0 }，
```

其中 ei 是评估器对失败的解释。

3. **提取经验。** 记笔记 LLM 在上下文中分析评估错误：

```text
Lj = Lnote(xj, y_j^(t), Ej)。
```

4. **去重并更新。** 应用与评估器优化相同的语义去重和提示词重写流程，但现在作用于 PG。

当没有剩余评估错误，或达到最大迭代次数时，生成循环终止。

## 4. 实验设置

我们设计了两组互补实验。第一部分在带有人类标注的既有基准上评估 BINEVAL 的性能。第二部分在一个不可验证任务和一个可验证任务上展示迭代式提示词更新机制。在这些实验中，我们使用 gpt-oss-120b 和 Claude Sonnet 4。为了降低 LLM 响应中的随机性，所有实验温度均设为 0，并报告两次运行的平均值。

### 4.1 指标

对于评估质量，我们报告方法分数与人类判断在摘要层面的 Spearman 秩相关（ρ）、Kendall 秩相关（τ）和 Pearson 相关（r）。

### 4.2 第一部分：评估质量验证

我们遵循 UniEval（Zhong et al., 2022）的评估协议，并在三个既有基准上评估。

**SummEval。**（Fabbri et al., 2021）该基准包含 100 篇 CNN/DM（See et al., 2017）源文章，每篇由 16 个不同摘要模型生成摘要，得到 1,600 个摘要层面标注。人类评估者从四个维度为每个摘要评分：流畅性、连贯性、一致性和相关性。评分采用 1–5 的李克特量表。

**Topical-Chat。**（Mehri & Eskenazi, 2020）该基准包含由 6 个对话模型生成的 60 个对话回复，在六个维度上标注：自然性、连贯性、吸引力、扎根性、可理解性以及整体质量评分。遵循 Zhong 等人（2022），我们使用其中四个方面。

**QAGS。**（Wang et al., 2020）该基准专门针对摘要中的幻觉评估，包含来自 CNN/DM 的 235 个样本和来自 XSum（Narayan et al., 2018）的 239 个样本。标注者根据源文档评定每个摘要的一致性。

### 4.3 第二部分：迭代式提示词更新

我们在两个任务上评估 BINEVAL 的迭代式提示词更新机制（算法 1）：SummEval 上的评估器提示词优化，以及 IFBench（Pyatkin et al., 2025）上的生成提示词优化。SummEval 在没有程序化黄金检查器的意义上是不可验证的；IFBench 则可通过可执行约束检查器验证。对于 SummEval，我们测试两种更新模式：自更新，即单个模型（gpt-oss-120b）使用与人类判断的失败来改进自己的评估器提示词；跨模型更新，即更强的模型（Claude Sonnet 4）作为参考评估器，并利用分歧产生的经验来更新目标模型提示词。详细实验设置见附录 B。

## 5. 结果

### 5.1 评估质量：SummEval

表 1 展示了不同评估范式之间清晰的排名。BINEVAL（Claude）总体上是最强方法，取得了最佳平均 Spearman 和 Kendall 相关，并在连贯性、一致性和流畅性上领先。最大提升出现在一致性上，BINEVAL 达到 0.655 / 0.615，说明将事实质量分解为多个目标化检查对摘要评估尤其有效。相关性是主要例外：G-Eval（GPT-4）在该维度上最好，表明一些更宽泛的语义判断仍然更难用二元分解捕获。

**表 1：SummEval 上摘要层面的 Spearman ρ / Kendall τ 相关。**

| 方法 | 连贯性 | 一致性 | 流畅性 | 相关性 | 平均 |
|---|---:|---:|---:|---:|---:|
| ROUGE-1 | 0.167 / 0.126 | 0.160 / 0.130 | 0.115 / 0.094 | 0.326 / 0.252 | 0.192 / 0.150 |
| BERTScore | 0.284 / 0.211 | 0.110 / 0.090 | 0.193 / 0.158 | 0.312 / 0.243 | 0.225 / 0.175 |
| MoverScore | 0.159 / 0.118 | 0.157 / 0.127 | 0.129 / 0.105 | 0.318 / 0.244 | 0.191 / 0.148 |
| BARTScore | 0.448 / 0.342 | 0.382 / 0.315 | 0.356 / 0.292 | 0.356 / 0.273 | 0.385 / 0.305 |
| UniEval (T5) | 0.575 / 0.442 | 0.446 / 0.371 | 0.449 / 0.371 | 0.426 / 0.325 | 0.474 / 0.377 |
| G-Eval (GPT-4) | 0.582 / 0.457 | 0.507 / 0.425 | 0.506 / 0.455 | 0.547 / 0.433 | 0.514 / 0.418 |
| G-Eval (gpt-oss) | 0.451 / 0.392 | 0.559 / 0.527 | 0.217 / 0.203 | 0.515 / 0.446 | 0.436 / 0.392 |
| UniEval (gpt-oss) | 0.237 / 0.208 | 0.489 / 0.476 | 0.000 / 0.000 | 0.288 / 0.256 | 0.254 / 0.235 |
| BINEVAL (gpt-oss) | 0.523 / 0.448 | 0.585 / 0.548 | 0.252 / 0.235 | 0.428 / 0.366 | 0.447 / 0.399 |
| BINEVAL (Claude) | 0.652 / 0.541 | 0.655 / 0.615 | 0.540 / 0.470 | 0.404 / 0.339 | 0.563 / 0.491 |

额外的 gpt-oss 运行说明了为什么分解重要。在相同骨干模型下，BINEVAL（gpt-oss）平均优于 G-Eval（gpt-oss）和 UniEval（gpt-oss），主要受连贯性和一致性上的大幅提升驱动。使用 gpt-oss 的 G-Eval 在一致性和相关性等数字量表维度上仍然可用，但其流畅性表现崩溃。使用 gpt-oss 的 UniEval 更弱，流畅性相关接近零，说明单个是/否问题对通用模型而言往往过于粗糙。总体而言，SummEval 支持本文的核心主张：多个二元问题比单个整体分数或单个布尔判断提供更稳健、更可迁移的评估信号。

图 1 从更细的角度展示这些收益。该图给出 SummEval 四个评估维度上的分数分布小提琴图，比较人类标注与不同方法。BINEVAL 在一致性上视觉上最接近人类分布：它基本匹配人类评分集中在上端的形状，同时仍保留一些低分质量；这与表 1 中其最大相关性优势相一致。各维度上，BINEVAL（Claude）通常是在集中趋势和分布宽度上最接近人类判断的方法之一，其中一致性匹配最强。UniEval 和 G-Eval 的分布更窄、更集中，表明它们对系统之间的区分能力较弱。基于 gpt-oss 的变体相对于人类评分通常低估分数，尤其在连贯性和相关性上，BINEVAL（gpt-oss）和 G-Eval（gpt-oss）的均值明显更低。流畅性在所有方法中都紧密聚集在天花板附近，反映出现代摘要系统整体流畅性较高且该维度方差有限。值得注意的是，UniEval（gpt-oss）产生近乎退化的流畅性分布，说明它无法沿该轴区分质量。总体而言，BINEVAL 的主要优势并不是在每个维度上都完美校准，而是能够保留有意义的相对变化，尤其是事实一致性。

图 2 在系统层面提供相同比较，其中每个分数在四个 SummEval 维度上取平均，并按 16 个系统的人类均值升序排列。BINEVAL（Claude）最忠实地追踪人类排序，保留了从较弱系统到较强系统的单调趋势，同时在中低性能模型之间保持可见区分。相比之下，UniEval 和 G-Eval 的分数范围更压缩，尤其减弱了排名中部系统之间的差异。基于 gpt-oss 的方法在绝对分数水平上通常更保守，但仍恢复了大部分总体系统排序。另一个明显模式是分布宽度：BINEVAL 变体往往显示出更宽、更接近人类的系统内方差，而 UniEval 和 G-Eval 产生更紧的小提琴图，可能低估真实分数变异。方法之间对最高人类质量系统（最右侧）的一致性最强，而低质量系统差异更大，说明区分差摘要和中等摘要仍然是自动评估方法的挑战。

### 5.2 评估质量：Topical-Chat

对话结果表明，BINEVAL 能有效迁移到摘要之外的任务。BINEVAL（Claude）在 Topical-Chat 上取得最佳平均 Spearman 相关（0.632），在自然性和吸引力上尤其显著；BINEVAL（gpt-oss）仍与 G-Eval（gpt-oss）具有竞争力，并明显强于 UniEval（gpt-oss）。这些结果表明，将对话质量分解为多个具体问题，对于主观会话标准特别有帮助。详细结果见附录 D.1。

### 5.3 评估质量：QAGS

QAGS 最清楚地凸显了分解的优势。BINEVAL（Claude）取得最佳平均 Spearman 相关（0.620），即使 BINEVAL（gpt-oss）也显著优于 G-Eval（gpt-oss）。后者的二元提示产生的分数粒度过少，难以可靠排序。这说明将事实一致性分解为若干目标化问题比依赖单个整体判断或是/否判断更稳健，尤其是在 XSum 这类容易出现幻觉的数据上。详细结果和讨论见附录 D.2。

### 5.4 迭代式提示词更新

#### 5.4.1 SummEval：评估器提示词更新

表 2 报告了在四个 SummEval 维度上迭代式提示词更新后的测试集 Spearman ρ。两种更新模式都改善了四个维度中的三个。自更新在流畅性上产生最大单维收益（+0.119），该维度的基线提示词尤其弱，而对评估器评分准则和生成的二元问题进行迭代细化显著改善了与人类判断的对齐。跨模型更新在一致性上最强（+0.136），这与更强参考评估器为事实验证提供特别有用指导的想法一致。按维度平均，自更新提升 +0.075，跨模型更新提升 +0.070。

**表 2：SummEval 上评估器提示词更新。测试集 Spearman ρ 与人类判断；∆ 为相对基线的绝对提升。最佳迭代由测试性能早停选择。**

| 维度 | 自更新 Base | 自更新 Best | ∆ | 跨模型 Base | 跨模型 Best | ∆ |
|---|---:|---:|---:|---:|---:|---:|
| 连贯性 | .521 | .610 | +.089 | .524 | .594 | +.070 |
| 一致性 | .477 | .568 | +.091 | .501 | .637 | +.136 |
| 流畅性 | .255 | .375 | +.119 | .246 | .318 | +.072 |
| 相关性 | .505 | .505 | .000 | .532 | .532 | .000 |
| 平均 | .440 | .515 | +.075 | .451 | .520 | +.070 |

相关性在两种更新模式下都难以改进。检查更新后的提示词显示，经验驱动的细化倾向于把相关性过度分解为过于细粒度的要求，例如分别检查每个参与者、动机和背景事件。这些细化让评估器比人类标注者更严厉，而不是更好地与其对齐，这说明与具有更具体失败模式的维度相比，相关性仍然是一种相对整体化的判断，也较不适合细粒度二元分解。

有三个观察很突出。第一，两种更新模式互补：自更新最有助于连贯性和流畅性，而跨模型更新最有助于一致性，这表明人类分数偏差和模型间分歧会暴露不同类型的评估器错误。第二，大多数增益出现在前一两次迭代；后续迭代更可能随着经验积累成相互竞争的指令而损害提示词。第三，二元问题再生成很关键：最大收益出现在不仅改变评估器提示词、也改变诱导出的问题分解的迭代中，这进一步表明问题设计本身是评估质量的关键杠杆。

#### 5.4.2 IFBench：生成提示词更新

表 3 展示了 IFBench 上各提示词更新迭代的严格测试集准确率。自更新带来适度改进，在第 3 次迭代达到峰值 38.0%，比其第 0 次迭代基线提高 +3.4 个百分点。然而，同一运行在第 4 次迭代崩溃，显示重复提示词重写的脆弱性。跨模型更新没有改进，事实上在第一次更新后下降，说明更强裁判的更严格标准可能会过度纠正提示词，而不是细化它。

**表 3：IFBench 上生成提示词更新（测试集严格准确率，%）。**

| 方法 | 迭代 0 | 1 | 2 | 3 | 4 | 峰值 |
|---|---:|---:|---:|---:|---:|---:|
| 自更新 | 34.6 | 36.8 | 34.6 | 38.0 | 26.1 | 38.0 (+3.4) |
| 跨模型 | 35.9 | 33.8 | — | — | — | 35.9 (+0.0) |
| 无优化（基线） | 35.5 |  |  |  |  | 35.5 |

**表 4：自更新下 IFBench 各类别准确率（%）。**

| 类别 | 基线 | 峰值 | 趋势 |
|---|---:|---:|---|
| Format | 52 | 69 | +17pp，对指导有响应 |
| Sentence | 25 | 42 | +17pp，对关键词和结构敏感 |
| Count | 63 | 63 | 指令越多越退化 |
| Ratio | 22 | 22 | 无变化 |
| Words | 16 | 20 | 边际改进 |
| Repeat | 17 | 17 | 无变化 |

按类别分解的表 4 显示，promptable（可通过提示词改善）约束和 computational（计算型）约束之间存在明显分界。格式和句子约束都有显著提升，各提升 17 个百分点，说明一旦模型获得更清晰的结构指导，这些任务通常可以解决。相反，计数、比例、词语和重复约束几乎没有改善。这些约束要求生成过程中的精确计算，例如维护计数、执行比例，或按音节/词汇标准过滤词。提取的经验常常能正确诊断这些失败，但“维护一个内部计数器”这类指令并不会赋予模型新的计算能力。相反，它们会累积成提示词膨胀，最终损害先前运行良好的类别。

主要结论是：当模型已经具备相关能力、只是需要更好指导来表达时，迭代式提示词更新是有效的；当失败反映的是底层能力限制而不是提示词问题时，其效果要差得多。在这些情况下，BINEVAL 仍然能提供准确诊断，但所得修复大多不可操作，并可能通过指令过载降低性能。

### 5.5 案例研究

附录 A 展示了评估和提示词更新示例。它包含四个 SummEval 案例研究，每个维度一个，展示 BINEVAL 能够识别单句摘要中的连贯性，发现细微事实错误，对混乱文本给予部分分，并把不完整性与不相关性区分开。附录还包括自更新和跨模型更新的 SummEval 提示词更新示例，一个过度分解损害与人类判断对齐的相关性失败案例，以及一个 IFBench 示例，突出可提示失败与底层计算限制之间的边界。这些例子共同显示，分解产生更有理由支持的分数，并帮助诊断提示词细化何时成功、何时失败。

### 5.6 为什么分解有效？

为什么通过多个原子二元问题进行评估会优于单个整体判断？我们识别出三种贡献机制，并在 SummEval 上考察证据（完整问题集见附录 E）。

**复杂度降低。** 每个二元问题都隔离一个可验证属性，用许多更简单的问题取代一个多方面判断——这与提示词中任务分解的好处相呼应（Zhou et al., 2022；Khot et al., 2022）。类似“所有命名实体是否都被准确表示？”的问题，比“把事实一致性按 1–5 打分”更容易可靠回答。在一致性上，七个目标化问题的是率分布在 0.75 到 0.95 之间（表 10），说明每个问题捕获不同难度层级。该模式也出现在其他维度：流畅性、相关性和连贯性的是率跨度分别为 0.48、0.46 和 0.86。

**通过聚合降低方差。** 聚合 N 个弱相关二元分类器会按 1/N 的比例降低方差。图 3 显示这一机制随维度变化：相关性和连贯性的平均问题间相关最低（ϕ = 0.20 和 0.28；80% 和 64% 的问题对满足 |ϕ| < 0.3），流畅性中等（ϕ = 0.39；例如拼写 Q2 与标点 Q3 的 ϕ = 0.02）。一致性是例外（ϕ = 0.58，弱相关问题对为零），其中“没有事实错误”和“不歪曲含义”等问题本质上相关（ϕ = 0.79）。

**覆盖失败模式。** 分解迫使显式枚举标准，相比整体判断提高召回。在流畅性中，拼写（Q2）和标点（Q3）几乎不相关（ϕ = 0.02），且是率不同（0.71 vs. 0.33），能捕捉不同失败。相关性 Q1（主题，0.95）和 Q3（冗余，0.64）显示 ϕ = 0.01。一致性仍然最弱：其最低相关问题对也有 ϕ = 0.32。

**维度层面总结。** 三种机制的贡献不均。相关性和连贯性表现出强方差降低与覆盖。流畅性受益于三者。一致性最有启发性：它的方差降低和覆盖最弱，却相对 UniEval 有最大提升（+0.195 Spearman ρ），说明仅仅通过复杂度降低——把事实验证分解为目标化子检查——就可以成为主导驱动因素。从实践角度看，使用者可以检查生成问题的这些属性（是率跨度、问题间相关、成对覆盖）来预判分解最有帮助的地方以及需要改进的地方。

## 6. 讨论

**失败模式。** 分解最适合事实一致性等具体标准，因为错误可以关联到特定声明或实体，因此能用相对清晰的是/否判断来检查。对于主观质量，它不那么可靠，因为人类判断更整体，也更难还原为一组二元检查。在这些情况下，评估质量很大程度取决于生成问题是否捕获了人类在形成整体判断时实际权衡的方面。附录展示了两种模式：当分解过严时，相关性可能退化；当失败反映模型基础能力而不是指令时，提示词更新帮助较小。在 IFBench 上，更清晰的提示词有助于格式和句子级约束，但不能帮助计数或比例跟踪，说明一些错误来自执行限制，而不仅是任务规范。

**计算成本。** BINEVAL 用效率换取诊断价值。与单个整体判断相比，它必须生成二元问题并逐一回答。这增加了模型调用次数和评估中处理文本总量。提示词更新还增加了记笔记、经验去重和元提示词重写，不过批处理使前两者开销适中，而且提示词重写是大多数更新方法共享的。主要的重复成本是问题级评估。

**局限。** 该方法仍然依赖问题质量：如果重要标准缺失，最终分数也会遗漏它们。它还假设满足问题的比例近似线性映射到整体质量，这并不总是成立。

### 6.1 分解式评估与整体打分

图 4 展示了整体式评估方法的一个代表性失败模式。被评估的摘要包含三个不同事实错误：把俄罗斯所述目的误归因给五角大楼、捏造了源文中不存在的外部 URL，以及混淆了双方对拦截事件的叙述。尽管有这些错误，G-Eval 和 UniEval 都给出满分一致性 5.0，因为摘要表面上看起来可信——它提到了正确的飞机型号，并大体准确描述了事件。整体式打分把局部正确性与全局一致性混为一谈，对流畅且主题连贯的文本给予奖励，即使其中具体声明是错的。

BINEVAL 通过把一致性分解为七个目标化二元问题来避免这一点，每个问题探查一种不同声明类型：事实支持、捏造、实体准确性、数字正确性、因果忠实、幻觉和范围表示。Q1、Q3 和 Q5 直接揭示误归因和混淆；Q2 标出捏造的 URL。所得 3/7 ≈ 1.57（缩放到 1–5）与人类评分 2.0（|∆| = 0.43）接近，而 G-Eval 和 UniEval 偏离 3.0 分。该示例激发了 BINEVAL 的核心设计原则：细粒度二元问题充当声明级探针，让聚合打分系统性遮蔽的错误显现出来。关键的是，这种粒度也使反馈可操作，因为每个失败问题都直接识别错误类型，从而能够对摘要器或评估器提示词进行有针对性的修正。

## 7. 结论

我们提出了 BINEVAL，一个任务无关、无需训练的框架，它通过把标准分解为原子二元问题来评估 LLM 输出。在 SummEval、Topical-Chat 和 QAGS 上，它匹配或超过强评估器，同时还支持摘要和 IFBench 上的迭代式提示词优化。由于每个分数都建立在带解释的单独判定之上，BINEVAL 提供了可解释反馈，帮助实践者诊断和改进 LLM 系统，并表明原子二元分解是更广泛评估任务中的一个有前景方向。这些结果说明，可解释性和强评估性能并不一定以可扩展性或灵活性为代价。展望未来，我们认为该方法自然可扩展到智能体和多轮设置，在这些场景中，细粒度、声明级反馈对于识别系统在哪里以及为什么出错尤其有价值。

## 影响声明

本文提出的工作旨在通过对语言模型输出进行更可解释、可扩展的评估，推动机器学习领域发展。我们的工作可能具有许多社会影响，包括提升研究和部署中自动评估流水线可靠性的可能性。同时，评估器模型可能继承用于实例化它们的底层语言模型的偏差和盲点，因此在高风险场景中部署 BINEVAL 时，应配合人工监督。

## 参考文献

Alam, F., Bhatia, G., Laskar, S. R., and Chowdhury, S. A. Beyond LLM-as-a-judge: Deterministic metrics for multilingual generative text evaluation. arXiv preprint arXiv:2604.05083, 2026.

Banerjee, S. and Lavie, A. Meteor: An automatic metric for MT evaluation with improved correlation with human judgments. In Proceedings of the ACL Workshop on Intrinsic and Extrinsic Evaluation Measures for Machine Translation and/or Summarization, 2005.

Es, S., James, J., Espinosa-Anke, L., and Schockaert, S. RAGAS: Automated evaluation of retrieval augmented generation. In Proceedings of the 18th Conference of the European Chapter of the Association for Computational Linguistics, 2024.

Fabbri, A. R., Kryściński, W., McCann, B., Xiong, C., Socher, R., and Radev, D. SummEval: Re-evaluating summarization evaluation. Transactions of the Association for Computational Linguistics, 9:391–409, 2021.

Khattab, O., Singhvi, A., Maheshwari, P., Zhang, Z., Santhanam, K., Vardhamanan, S., Haq, S., Sharma, A., Joshi, T. T., Mober, H., et al. DSPy: Compiling declarative language model calls into self-improving pipelines. arXiv preprint arXiv:2310.03714, 2023.

Khot, T., Trivedi, H., Finlayson, M., Fu, Y., Richardson, K., Clark, P., and Sabharwal, A. Decomposed prompting: A modular approach for solving complex tasks. arXiv preprint arXiv:2210.02406, 2022.

Kim, S., Suk, J., Longpre, S., Lin, B. Y., Shin, J., Welleck, S., Neubig, G., Lee, M., Lee, K., and Seo, M. Prometheus 2: An open source language model specialized in evaluating other language models. In Proceedings of the 2024 Conference on Empirical Methods in Natural Language Processing, pp. 4334–4353, 2024.

Lage, L. and Ostermann, S. OpenFActScore: Open-source atomic evaluation of factual precision in long-form text generation. arXiv preprint arXiv:2502.09676, 2025.

Lemesle, Q., Chevelu, J., Martin, P., Lolive, D., Delhay, A., and Barbot, N. Paraphrase generation evaluation powered by an LLM: A semantic metric, not a lexical one. In Proceedings of the 31st International Conference on Computational Linguistics, 2025.

Li, X. L., Zhang, T., Dubois, Y., Taori, R., Gulrajani, I., Guestrin, C., Liang, P., and Hashimoto, T. B. AlpacaEval: An automatic evaluator of instruction-following models, 2023. GitHub repository.

Lin, C.-Y. ROUGE: A package for automatic evaluation of summaries. In Text Summarization Branches Out, pp. 74–81, 2004.

Liu, Y., Iter, D., Xu, Y., Wang, S., Xu, R., and Zhu, C. G-Eval: NLG evaluation using GPT-4 with better human alignment. In Proceedings of the 2023 Conference on Empirical Methods in Natural Language Processing, 2023.

Mehri, S. and Eskenazi, M. USR: An unsupervised and reference free evaluation metric for dialog generation. In Proceedings of the 58th Annual Meeting of the Association for Computational Linguistics, 2020.

Min, S., Krishna, K., Lyu, X., Lewis, M., tau Yih, W., Koh, P. W., Iyyer, M., Zettlemoyer, L., and Hajishirzi, H. FActScore: Fine-grained atomic evaluation of factual precision in long form text generation. In Proceedings of the 2023 Conference on Empirical Methods in Natural Language Processing, 2023.

Narayan, S., Cohen, S. B., and Lapata, M. Don’t give me the details, just the summary! Topic-aware convolutional neural networks for extreme summarization. In Proceedings of EMNLP 2018, pp. 1797–1807, Brussels, Belgium, October-November 2018. doi: 10.18653/v1/D18-1206. URL https://aclanthology.org/D18-1206/.

Opsahl-Ong, K., Ryan, M. J., Purtell, J., Broman, D., Potts, C., Zaharia, M., and Khattab, O. Optimizing instructions and demonstrations for multi-stage language model programs. In Proceedings of EMNLP 2024, pp. 9340–9366, 2024.

Papineni, K., Roukos, S., Ward, T., and Zhu, W.-J. BLEU: A method for automatic evaluation of machine translation. In Proceedings of ACL 2002.

Pyatkin, V., Malik, S., Graf, V., Ivison, H., Huang, S., Dasigi, P., Lambert, N., and Hajishirzi, H. Generalizing verifiable instruction following, 2025. URL https://arxiv.org/abs/2507.02833.

Saad-Falcon, J., Khattab, O., Potts, C., and Zaharia, M. ARES: An automated evaluation framework for retrieval-augmented generation systems. arXiv preprint arXiv:2311.09476, 2024.

See, A., Liu, P. J., and Manning, C. D. Get to the point: Summarization with pointer-generator networks, 2017. URL https://arxiv.org/abs/1704.04368.

Wang, A., Cho, K., and Lewis, M. Asking and answering questions to evaluate the factual consistency of summaries. In Proceedings of ACL 2020.

Yang, C., Wang, X., Lu, Y., Liu, H., Le, Q. V., Zhou, D., and Chen, X. Large language models as optimizers. arXiv preprint arXiv:2309.03409, 2023.

Yin, L. and Wang, Z. LLM-AutoDiff: Auto-differentiate any LLM workflow. arXiv preprint arXiv:2501.16673, 2025.

Yuan, W., Neubig, G., and Liu, P. BARTScore: Evaluating generated text as text generation. In Advances in Neural Information Processing Systems, 2021.

Yue, T., Mao, R., Shi, X., Zhan, S., Yang, Z., and Zhao, D. QAEval: Mixture of evaluators for question-answering task evaluation. In Proceedings of ACL 2025, pp. 14717–14730, 2025.

Zhang, J., Wang, Z., Zhu, H., Liu, J., Lin, Q., and Cambria, E. MARS: A multi-agent framework incorporating socratic guidance for automated prompt optimization. arXiv preprint arXiv:2503.16874, 2025.

Zhang, T., Kishore, V., Wu, F., Weinberger, K. Q., and Artzi, Y. BERTScore: Evaluating text generation with BERT. In Proceedings of ICLR 2020.

Zhao, W., Peyrard, M., Liu, F., Gao, Y., Meyer, C. M., and Eger, S. MoverScore: Text generation evaluating with contextualized embeddings and earth mover distance. In Proceedings of EMNLP 2019.

Zheng, L., Chiang, W.-L., Sheng, Y., Zhuang, S., Wu, Z., Zhuang, Y., Lin, Z., Li, Z., Li, D., Xing, E. P., et al. Judging LLM-as-a-judge with MT-bench and chatbot arena. In Advances in Neural Information Processing Systems, 2023.

Zhong, M., Liu, Y., Yin, D., Zhu, Y., Zhu, C., and Zeng, M. Towards a unified multi-dimensional evaluator for text generation. In Proceedings of EMNLP 2022.

Zhou, D., Schärli, N., Hou, L., Wei, J., Scales, N., Wang, X., Schuurmans, D., Cui, C., Bousquet, O., Le, Q., and Chi, E. H. Least-to-most prompting enables complex reasoning in large language models. arXiv preprint arXiv:2205.10625, 2022.

Zhou, H., Huang, H., Zhang, R., Chen, K., Xu, B., Zhu, C., Zhao, T., and Yang, M. Toward robust LLM-based judges: Taxonomic bias evaluation and debiasing optimization. arXiv preprint arXiv:2603.08091, 2026.

Zhou, Y., Muresanu, A. I., Han, Z., Paster, K., Pitis, S., Chan, H., and Ba, J. Large language models are human-level prompt engineers. In Proceedings of ICLR 2023.

## 附录 A. 案例研究

### A.1 有效评估：示例

#### (a) 连贯性——单句摘要

摘要：“Speed camera has been turned round and is pointing at this house in Birmingham, West Midlands.”

| 方法 | 分数 | |∆| | 解释 |
|---|---:|---:|---|
| Human | 4.67 | — | 连贯：清楚且切题 |
| BINEVAL (Claude) | 4.56 | 0.11 | 7/8 个问题为是——作为单句天然连贯 |
| G-Eval (gpt-oss) | 1.00 | 3.67 | 把简短惩罚为“不连贯” |
| UniEval (gpt-oss) | 1.00 | 3.67 | “这是否连贯？”→ 否 |

分解推理：是 Q1（有结构）；是 Q2（逻辑顺序）；是 Q3（有过渡）；是 Q4（无重复）；是 Q5（统一焦点）；是 Q6（主题）；否 Q7（遗漏一些细节）；是 Q8（无矛盾）。

洞见：单句天然满足排序、无矛盾和焦点标准。唯一的“否”（覆盖不完整）给出成比例惩罚：7/8 → 4.56，与人类分数非常接近。整体式方法把完整性和连贯性混淆，给出最低分。

#### (b) 一致性——看似可信摘要中的细微事实错误

源文（摘录）：“The Pentagon called the intercept unsafe and unprofessional... The Russian Defense Ministry said the jet was scrambled to identify the aircraft...”

摘要：“The U.S. RC-135U was flying over the Baltic Sea when it was intercepted by a Russian SU-27 Flanker. The Pentagon said the Russian jet flew around the plane to identify it. Read more: http://dailycaller.com/...”。

| 方法 | 分数 | |∆| | 解释 |
|---|---:|---:|---|
| Human | 2.00 | — | 多个事实错误 |
| BINEVAL (Claude) | 1.57 | 0.43 | 3/7 个问题为是——捕捉误归因、URL、叙述混淆 |
| G-Eval (gpt-oss) | 5.00 | 3.00 | 表面“看起来一致” |
| UniEval (gpt-oss) | 5.00 | 3.00 | “这是否一致？”→ 是 |

分解推理：否 Q1（把俄罗斯的目的误归给五角大楼）；否 Q2（捏造 URL）；否 Q3（实体角色错误）；是 Q4（飞机型号正确）；否 Q5（混淆五角大楼和俄罗斯的叙述）；是 Q6–7（核心事件已描述）。

洞见：摘要提到正确实体并描述真实事件，所以整体方法认为它一致。BINEVAL 的分解问题独立探查每个声明，捕捉误归因、捏造 URL 和叙述混淆。分数 3/7 → 1.57，接近人类 2.0。

#### (c) 流畅性——混乱但部分可读的摘要

摘要：“‘Space invaders’ was developed in japan back in 1970. Japanese can sleep soundly in their beds tonight as government’s top military official. He also fought muhammad ali in 1976. Inoki has appeared in the u.s.-based wwe.”

| 方法 | 分数 | |∆| | 解释 |
|---|---:|---:|---|
| Human | 2.00 | — | 有一些错误但部分可读 |
| BINEVAL (Claude) | 1.50 | 0.50 | 2/8 个问题为是——识别部分可读性 |
| G-Eval (gpt-oss) | 1.00 | 1.00 | 最低分：“质量差” |
| UniEval (gpt-oss) | 1.00 | 1.00 | “这是否流畅？”→ 否 |

分解推理：否 Q1（句子片段）；是 Q2（无拼写错误）；否 Q3（标点和大小写问题）；否 Q4（措辞不准确）；否 Q5（句子 2 有残缺）；否 Q6（话题跳跃不自然）；否 Q7（需要重读）；是 Q8（要点仍可理解）。

洞见：人类给 2/3，因为文本虽有错误但部分可读。BINEVAL 捕捉到这种细微差别：Q2 和 Q8 的“是”避免了地板分。G-Eval 和 UniEval 因任何流畅性问题触发整体否定而给最低分。

#### (d) 相关性——简短但切题的一句话

源文（摘录）：“ISIS released more than 200 Yazidis... mostly women, children, and elderly... A senior Peshmerga commander said they were released in groups... The freed captives appeared very tired...”

摘要：“ISIS released over 200 Yazidis on Wednesday.”

| 方法 | 分数 | |∆| | 解释 |
|---|---:|---:|---|
| Human | 3.33 | — | 相关但不完整 |
| BINEVAL (Claude) | 3.67 | 0.33 | 4/6 个问题为是——切题、无填充，但稀疏 |
| G-Eval (gpt-oss) | 1.00 | 2.33 | 把简短惩罚为“不相关” |
| UniEval (gpt-oss) | 1.00 | 2.33 | “这是否相关？”→ 否 |

分解推理：否 Q1（遗漏人口结构和状况等关键细节）；是 Q2（无捏造）；是 Q3（无冗余）；是 Q4（无琐碎填充）；否 Q5（过于稀疏，遗漏重要方面）；是 Q6（包含内容是相关的）。

洞见：摘要准确捕捉中心事件，但太短。BINEVAL 奖励其做对之处，同时惩罚遗漏。分数 4/6 → 3.67，与人类 3.33 匹配。G-Eval 和 UniEval 又一次把不完整性和不相关性混淆，即使摘要事实上切题，也给最低分。

图 5：四个 SummEval 示例，每个评估维度一个。每种情况下，BINEVAL 的问题分解都通过独立评估多个质量方面，生成与人类判断接近的分数。整体式方法在边界样本上会塌缩到极端分数，例如简短但正确的摘要、部分可读文本或简洁的一句话，因为单一判断会混淆正交质量维度。

图 6：四个 SummEval 示例的分数比较，每个维度一个。虚线标记人类参考。BINEVAL（Claude）在所有维度上都持续追踪人类分数。G-Eval（GPT-4）和 UniEval（T5）这些已发表基线表现尚可，但当它们的评估格式不配合蒙特卡洛采样或微调而直接应用到 gpt-oss 时，边界样本上的分数会崩溃。

### A.2 提示词演化：示例

本节展示 BINEVAL 的迭代式提示词更新如何跨迭代修改评估和生成提示词，包括成功更新和失败模式。

#### A.2.1 示例 1：SummEval 连贯性上的自更新

结果：Spearman ρ 从 .521（基线）提升到 .610（第 1 次迭代）。

自更新流水线识别出，基线连贯性提示词对单句摘要过于严格，并惩罚背景细节缺失，而人类标注者主要关注逻辑流。提取出三个代表性经验：

**提取经验（自更新，连贯性）**

1. 隐式过渡是可以接受的。要求逻辑连接，但不要强制显式提示词（“because”“therefore”）。如果叙事流畅，隐式连续性就足够。
2. 增加中心主张相关性标准。每个句子都应推进文章的主张；不贡献的句子无论语法是否正确都属于无贡献。
3. 不要惩罚背景细节的省略。只要核心事实和冲突清楚，缺失上下文不应降低连贯性。

**表 5：连贯性提示词从迭代 0 到迭代 1 的关键变化。**

| 迭代 0（基线） | 迭代 1（更新后） |
|---|---|
| “……句子之间的逻辑连接（显式线索如 because、therefore，或隐式连续性）……” | “……句子之间的逻辑连接（隐式连接可以接受；after、because 等显式标记有帮助但不是必须）……” |
| “……全局焦点——每个句子都与主题直接相关。” | “……与中心主张相关：识别文章主张或核心事实，确保每个句子推进或支持该主张。如果句子不推进主论点，将其视为无贡献。” |
| “只因逻辑流差、冗余、顺序错误或跑题内容而惩罚，不因缺少事实而惩罚。” | “只要核心冲突或核心事实仍然清楚，不要因摘要省略背景细节而惩罚。” |

为什么有效：这些经验正确识别出一个系统性偏差——模型过度惩罚简短性——并且更新后的评分准则明确要求评估器容忍省略，同时增加了更符合人类连贯性判断的具体“中心主张”标准。

#### A.2.2 示例 2：SummEval 一致性上的跨模型更新

结果：Spearman ρ 从 .501（基线）提升到 .637（第 1 次迭代）。

Claude（源评估器）正确地区分了省略（未提及源事实）和矛盾（陈述了不受支持的内容）。gpt-oss（目标）混淆了二者，惩罚只是省略细节的摘要。由分歧驱动的关键经验如下：

**提取经验（跨模型，一致性）**

1. 省略 ≠ 不一致。摘要省略源文细节不是事实不一致；只有摘要中出现但不受支持的陈述才应受到惩罚。
2. 通过算术实现语义等价。把“第 83 分钟”转换成“剩余七分钟”（在 90 分钟比赛中）是有效转换，不是幻觉。
3. 主体-角色误归因。当摘要重组从句时，要验证实体是否附着到正确动词上。

更新后的提示词显著变长（从 4 个评估步骤增加到 6 个，并包含关于字面解释、主体验证和语义等价的详细指导）。关键结构性补充如下：

```text
对摘要中的每个陈述，检查它是否受文章支持。
摘要不需要覆盖文章中的所有细节。
省略信息不是事实错误。
只标记摘要中存在但不受支持或被源文矛盾的陈述。
```

为什么有效：跨模型信号精确指出了一个根本概念错误（混淆省略与矛盾），而单靠人类分数偏差无法如此清晰地暴露它。+.136 的提升是我们实验中最大的，说明模型间分歧可以识别自反思容易漏掉的系统性评估偏差。

#### A.2.3 示例 3：失败案例——SummEval 相关性

结果：应用经验后，Spearman ρ 从 .505 降到 .357。

自更新流水线正确诊断出模型对相关性过于宽松——给那些捕捉标题事实但遗漏关键参与者和动机的摘要满分。然而，修复让提示词过于严格：

**提取经验（自更新，相关性——导致退化）**

1. 使评分准则对基本上下文覆盖更严格，而不只是标题事实。
2. 要求评估器检查每个关键参与者、每个动机和每个背景事件。
3. 应用定量惩罚：每缺少一个关键参与者扣 1 分；每缺少一个动机或背景事件扣 0.5 分。

结果提示词把相关性分解为穷尽式子标准（参与者、动机、背景事件、事实命题、冗余），并采用刚性惩罚系统。重新生成的二元问题反映了这种过度具体性：

1. 摘要是否包含源文中提到的每个关键参与者？
2. 摘要是否包含源文中陈述的每个行动动机？
3. 摘要是否包含所有与标题直接相关的背景事件？
4. 摘要是否包含所有其他事实命题（日期、地点、金额）？
5. 摘要是否避免不相关或冗余信息？

为什么失败：人类标注者对相关性使用整体判断——“摘要是否捕捉要点？”——并且对次要细节缺失有软容忍。更新后问题要求穷尽覆盖，导致模型把几乎所有摘要都评为有缺陷。所得分数系统性低于人类分数，破坏了秩相关。这说明一个根本局限：当人类评估标准本质上是整体且宽容时，把它分解为严格原子检查会产生更苛刻、偏离人类行为的评估器。

#### A.2.4 示例 4：IFBench——可提示约束与计算型约束

结果：格式准确率从 52% 提升到 69%；计数准确率从 63% 降到 31%。

IFBench 元提示词从极简开始（22 个字符：“Respond to the query.”）。如表 6 所示，经过 4 次经验提取和提示词重写，它增长到 6,248 个字符。经验分为两类：

**可提示经验（有效）。** 对格式和句子约束，经验识别出模型可以遵循的缺失指导：

- “除非明确要求，输出必须是纯文本，不带标记。”
- “对于 repeat 类型任务，只输出带有指定最小修改的确切原始请求。不要添加解释。”
- “严格遵守请求格式。每一行都必须按照描述的结构（缩进、列表标记、换行）。”

**计算型经验（无效）。** 对计数和比例约束，经验正确诊断了问题，但开出不可操作的指令：

- “为每个所需元素维护运行计数器，并在达到目标计数时停止。” ← 模型无法执行。
- “程序化验证每个 token 的位置；如果计数错误，重写直到满足。” ← 需要自验证循环。
- “构建结构大纲，并在放置所需词时参考它。” ← 隐式推理，未被强制执行。

这些不可操作指令的累积造成提示词膨胀。

**表 6：IFBench 元提示词增长及其对准确率的影响。**

| 迭代 | 提示词大小 | 经验数 | Format | Count |
|---|---:|---:|---:|---:|
| 0（基线） | 22 chars | — | 52 | 63 |
| 1 | 1,841 | 10 | 57 | 63 |
| 2 | 3,425 | 10 | 52 | 52 |
| 3 | 4,890 | 10 | 69 | 42 |
| 4 | 6,248 | 10 | 48 | 31 |

洞见：到第 3 次迭代时，提示词足够大，包含有用格式指导，但还没有膨胀到注意力竞争损害所有类别。到第 4 次迭代时，累积的计算型指令（模型无法遵循）制造噪声，干扰原本有效的格式指导，导致所有类别崩溃。这揭示了基于提示词优化的承载容量：超过临界提示词长度后，额外指令无论是否正确都会适得其反。

## 附录 B. 实验设置

**SummEval——评估器提示词优化。** 我们在四个 SummEval 维度上优化 gpt-oss-120b 的评估器提示词：连贯性、一致性、流畅性和相关性。SummEval 包含 1,600 个样本（100 篇文档 × 16 个摘要系统），并具有 1–5 李克特量表的人类评分。

- **数据划分。** 我们随机抽取每个系统 10 个样本（seed = 42），得到 160 个用于经验提取的开发样本和 1,440 个用于评估的测试样本。开发集覆盖 100 篇文档中的 82 篇，既提供广覆盖，又保持更新循环可管理。
- **模型。** 目标评估器为温度 0 的 gpt-oss-120b。自更新中，同一模型也作为经验提取、语义去重和提示词重写的记笔记器。跨模型更新中，Claude Sonnet 4 同时作为源评估器和记笔记器，温度同样为 0。
- **流程。** 每次迭代遵循算法 1：（1）用当前提示词和二元问题评估开发集和测试集；（2）自更新时，识别模型分数与人类分数差异最大的样本（归一化后 |s_model - s_human| > 0.3），跨模型更新时，识别源评估器和目标评估器的问题级分歧；（3）从这些失败或分歧中批量提取经验，用 LLM 语义去重，并保留最多 10 条唯一经验；（4）用一次 LLM 调用重写评估器提示词，纳入所有保留经验；（5）自更新时从更新后的提示词重新生成二元问题。
- **早停。** 最多运行 5 次迭代；当测试集 Spearman ρ 相对上一次迭代下降时停止。
- **指标。** 我们报告所有测试样本上的 pooled Spearman 秩相关，而不是按文档平均，因为按文档平均会丢弃稀疏开发划分中系统数少于两个的文档。

**IFBench——生成提示词优化。** 我们在 IFBench 上优化 gpt-oss-120b 的生成元提示词。IFBench 是一个指令遵循基准，包含 290 个测试样本，覆盖 56 种约束类型和 7 个类别：count、words、format、ratio、sentence、repeat 和 custom。每个样本都包含程序化验证函数。

- **数据划分。** 开发集包含 56 个样本，每种约束类型一个，优先选择先前失败样本；测试集包含剩余 238 个样本。
- **模型。** 生成器为温度 0 的 gpt-oss-120b。自更新中，裁判同样为 gpt-oss-120b；跨模型更新中，裁判为 Claude Sonnet 4。
- **二元问题分解。** 对每个开发样本，我们把 IFBench 约束规范转换为自然语言描述，并使用 BINEVAL 元提示词把它分解为二元是/否问题。例如，一个约束要求 door 出现一次、bread 出现两次，会变成诸如“回复是否恰好包含一次 door 和恰好两次 bread”的问题。
- **流程。** 每次迭代：（1）用当前元提示词在全部 290 个样本上生成回复；（2）使用带二元问题的 LLM 裁判评估开发集回复；（3）从开发失败中提取并去重经验；（4）重写生成元提示词；（5）使用官方 IFBench 验证函数以严格模式在测试集上评估。
- **迭代。** 自更新运行 5 次迭代。跨模型更新在 2 次迭代后停止，因为测试准确率下降。

## 附录 C. 自动提示词更新算法

**算法 1：通过二元问题分歧进行迭代提示词更新**

输入：源评估器 Esrc，带初始提示词 P_E^(0) 的目标评估器 Etgt，二元问题 Q = {q1, ..., qN}，测试数据 {(xj, yj)}_{j=1}^{J}，记笔记 LLM Lnote，更新器 LLM Lupdate，容差 ε，最大迭代次数 T。

输出：更新后的提示词 P_E^(T)。

```text
for t ← 1 to T do
  // 步骤 1：用两个模型评估
  for each (xj, yj) in test data do
    A_j^src ← { f_Esrc(xj, yj, qi) }_{i=1}^{N}
    A_j^tgt ← { f_Etgt(xj, yj, qi; P_E^(t-1)) }_{i=1}^{N}

  // 检查收敛
  for each dimension d in D do
    S_d^tgt ← (1/|Qd|) * Σ_{qi∈Qd} mean_j[A_j^tgt(qi)]
    S_d^src ← (1/|Qd|) * Σ_{qi∈Qd} mean_j[A_j^src(qi)]
  if |S_d^tgt - S_d^src| < ε for all d then
    return P_E^(t-1)  // 已收敛

  // 步骤 2：识别分歧
  for each (xj, yj) in test data do
    Δj ← { qi : A_j^src(qi) ≠ A_j^tgt(qi) }

  // 步骤 3：从分歧中提取经验
  L_all ← empty list
  for each j where |Δj| > 0 do
    Lj ← Lnote(xj, yj, A_j^src, A_j^tgt, Δj)
    L_all ← L_all + Lj

  // 步骤 4：语义去重
  M ← empty list  // 经验记忆
  for each l_new in L_all do
    (is_dup, merge_idx, merged) ← Dedup LLM(l_new, M)
    if is_dup then
      M[merge_idx] ← merged
    else
      M.append(l_new)
  L_unique ← M

  // 步骤 5：更新目标评估器提示词
  P_E^(t) ← P_E^(t-1)
  for each lk in L_unique do
    (sk, s'_k) ← Lupdate(P_E^(t), lk)
    P_E^(t) ← P_E^(t).replace(sk, s'_k)
return P_E^(T)
```

## 附录 D. 结果

### D.1 BINEVAL 在 Topical-Chat 上的评估结果

表 7 确立了四个主要发现。第一，BINEVAL（Claude）是 Topical-Chat 上总体最强方法，具有最佳平均 Spearman 和 Kendall 相关（0.632 / 0.525），优于 G-Eval（GPT-4）、UniEval（T5）和所有词汇基线。这表明多问题二元分解是对话评估的强范式，因为对话质量依赖多个部分独立的标准，而不是单个聚合印象。

**表 7：Topical-Chat 上 turn-level Spearman ρ / Kendall τ 相关。**

| 方法 | 自然性 | 连贯性 | 吸引力 | 扎根性 | 平均 |
|---|---:|---:|---:|---:|---:|
| ROUGE-L | 0.176 / 0.146 | 0.193 / 0.203 | 0.295 / 0.300 | 0.310 / 0.327 | 0.243 / 0.244 |
| BLEU-4 | 0.180 / 0.175 | 0.131 / 0.235 | 0.232 / 0.316 | 0.213 / 0.310 | 0.189 / 0.259 |
| METEOR | 0.212 / 0.191 | 0.250 / 0.302 | 0.367 / 0.439 | 0.333 / 0.391 | 0.290 / 0.331 |
| BERTScore | 0.226 / 0.209 | 0.214 / 0.233 | 0.317 / 0.335 | 0.291 / 0.317 | 0.262 / 0.273 |
| UniEval (T5) | 0.455 / 0.330 | 0.602 / 0.455 | 0.573 / 0.430 | 0.577 / 0.453 | 0.552 / 0.417 |
| G-Eval (GPT-4) | 0.549 / 0.565 | 0.594 / 0.605 | 0.627 / 0.631 | 0.531 / 0.551 | 0.575 / 0.588 |
| G-Eval (gpt-oss) | 0.422 / 0.368 | 0.526 / 0.444 | 0.642 / 0.560 | 0.576 / 0.539 | 0.541 / 0.478 |
| UniEval (gpt-oss) | 0.000 / 0.000 | 0.132 / 0.116 | 0.073 / 0.064 | 0.370 / 0.346 | 0.144 / 0.132 |
| BINEVAL (gpt-oss) | 0.483 / 0.420 | 0.447 / 0.359 | 0.648 / 0.532 | 0.578 / 0.488 | 0.539 / 0.450 |
| BINEVAL (Claude) | 0.686 / 0.565 | 0.564 / 0.447 | 0.740 / 0.606 | 0.538 / 0.485 | 0.632 / 0.525 |

第二，不同方法在不同维度上最强。BINEVAL（Claude）在最主观的维度——自然性和吸引力——上表现最好，Spearman 相关分别比 G-Eval（GPT-4）高 0.137 和 0.113。相比之下，UniEval（T5）和 G-Eval 在连贯性上更强，更整体的表示可能更好捕捉全局逻辑流。扎根性相对与方法无关：所有 LLM 评估器都在窄范围内，说明无论评估格式如何，该维度更易捕捉。

第三，评估器质量是一阶因素。BINEVAL（gpt-oss）平均达到 0.539 / 0.450，接近 G-Eval（gpt-oss）的 0.541 / 0.478，并远高于 UniEval（gpt-oss）的 0.144 / 0.132，但仍明显低于 Claude 版本。换言之，良好的问题分解有帮助，但评估器仍必须有足够能力细致回答会话问题。UniEval（gpt-oss）尤其明显：单个二元问题经常塌缩为几乎常量输出，例如自然性为 0。

第四，问题设计有帮助但有边界。BINEVAL（gpt-oss）仍具竞争力，因为多个二元问题即使在单分数校准较弱时也能创造有用的分数粒度，但它仍无法匹配 Claude 评估器。总体上，Topical-Chat 结果表明分解对主观对话质量尤其有价值，同时最佳性能仍依赖评估器强度。

小提琴图强化了这些趋势。在图 7 中，BINEVAL（Claude）在所有四个维度上最接近人类分布的宽度和偏斜，既保留了吸引力上的更大离散度，也保留了自然性、连贯性和扎根性上更集中但非退化的分布。UniEval（T5）保持高分但明显压缩，在自然性、连贯性和扎根性上有明显天花板效应，并且在吸引力上对齐较弱。在基于 gpt-oss 的评估器中，BINEVAL（gpt-oss）比 Claude 版本更保守，但仍保留样本间有意义的变化；G-Eval（gpt-oss）更压缩，UniEval（gpt-oss）在所有维度上几乎退化，几乎没有区分能力。

图 8 在系统层面显示相同模式。BINEVAL（Claude）最好地保留了人类的系统排序，能把强系统与弱系统区分开，同时保留真实系统内变化，而不是把所有输出压缩到窄的高分带中。BINEVAL（gpt-oss）也追踪总体排序，但绝对分数更低，说明即使底层评估器较弱，分解仍有帮助。相比之下，G-Eval（gpt-oss）压缩了低到中等区间的大部分，而 UniEval（gpt-oss）在系统间几乎平坦。这些图共同说明分解的核心优势：多个目标化问题会产生比单个整体判断或近似布尔判断更真实、更有区分度的分数变化。

### D.2 BINEVAL 在 QAGS 上的评估结果

表 8 突出了问题分解最有帮助的设置。BINEVAL（Claude）总体最强，平均 Pearson、Spearman 和 Kendall 相关最佳（0.604 / 0.620 / 0.534）。在两个数据集的基于排名指标上都最强，在 CNN/DM 和 XSum 上分别达到最高 Spearman 值 0.702 和 0.539，同时在 Pearson 相关上也能与 CNN/DM 上的 BARTScore、XSum 上的 G-Eval（GPT-4）等更强回归式基线竞争。BINEVAL（gpt-oss）同样稳健，平均达到 0.543 / 0.563 / 0.492，显著优于其他基于 gpt-oss 的评估器。

数据集层面的分解也很有信息量。CNN/DM 是更容易的划分：多数强评估器达到较高相关性，两个 BINEVAL 变体在那里表现良好，BINEVAL（Claude）为 0.665 / 0.702 / 0.597，BINEVAL（gpt-oss）为 0.651 / 0.642 / 0.551。XSum 对所有方法都更难，但相对模式保持一致：BINEVAL（Claude）仍给出最佳 Spearman 相关（0.539），略高于 G-Eval（GPT-4）的 0.537，而 BINEVAL（gpt-oss）仍具竞争力（0.483）。

**表 8：QAGS 上相关结果。QAGS-CNN、QAGS-XSUM 及平均的 Pearson r / Spearman ρ / Kendall τ。**

| 指标 | CNN r | CNN ρ | CNN τ | XSUM r | XSUM ρ | XSUM τ | 平均 r | 平均 ρ | 平均 τ |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| ROUGE-L | 0.357 | 0.324 | 0.254 | 0.024 | -0.011 | -0.009 | 0.190 | 0.156 | 0.122 |
| BERTScore | 0.576 | 0.505 | 0.399 | 0.024 | 0.008 | 0.006 | 0.300 | 0.256 | 0.202 |
| MoverScore | 0.414 | 0.347 | 0.271 | 0.054 | 0.044 | 0.036 | 0.234 | 0.195 | 0.153 |
| FactCC | 0.416 | 0.484 | 0.376 | 0.297 | 0.259 | 0.212 | 0.356 | 0.371 | 0.294 |
| QAGS | 0.545 | — | — | 0.175 | — | — | 0.375 | — | — |
| BARTScore | 0.735 | 0.680 | 0.557 | 0.184 | 0.159 | 0.130 | 0.459 | 0.420 | 0.343 |
| CTC | 0.619 | 0.564 | 0.450 | 0.309 | 0.295 | 0.242 | 0.464 | 0.430 | 0.346 |
| UniEval (T5) | 0.682 | 0.662 | 0.532 | 0.461 | 0.488 | 0.399 | 0.571 | 0.575 | 0.465 |
| G-Eval (GPT-4) | 0.631 | 0.685 | 0.591 | 0.558 | 0.537 | 0.472 | 0.599 | 0.611 | 0.525 |
| G-Eval (gpt-oss) | 0.045 | 0.028 | 0.027 | 0.236 | 0.236 | 0.236 | 0.140 | 0.132 | 0.131 |
| UniEval (gpt-oss) | 0.415 | 0.382 | 0.357 | 0.490 | 0.490 | 0.490 | 0.452 | 0.436 | 0.424 |
| BINEVAL (gpt-oss) | 0.651 | 0.642 | 0.551 | 0.435 | 0.483 | 0.433 | 0.543 | 0.563 | 0.492 |
| BINEVAL (Claude) | 0.665 | 0.702 | 0.597 | 0.543 | 0.539 | 0.470 | 0.604 | 0.620 | 0.534 |

额外的 gpt-oss 基线让分解优势特别清楚。G-Eval（gpt-oss）在 QAGS 上几乎崩溃，平均只有 0.140 / 0.132 / 0.131；UniEval（gpt-oss）恢复了一些信号，因为事实一致性更接近二元属性，但仍落后于两个 BINEVAL 变体。简言之，QAGS 显示，当单个整体提示无法保留足够排序粒度时，分解最有价值。

图 9 在分布形式上支持相同结论。对于 CNN/DM 和 XSum，人类评分都明显双峰，在 0 和 1 附近都有大量质量。BINEVAL（Claude）最清晰地保留这种结构：它在整个范围内保持广泛支持，而不是向尺度顶部塌缩。BINEVAL（gpt-oss）稍微更保守，但仍保留可见的分布宽度和分离。相比之下，UniEval（T5）在两个数据集上都过度自信，大部分质量集中在高分附近；G-Eval（gpt-oss）和 UniEval（gpt-oss）则以错误方式近乎二元化——它们把很多分布放在极端值上，中间变化很有限。这一点很重要，因为有用的事实评估器必须区分轻微有缺陷的摘要和明显不一致的摘要，而不只是把明显正确和明显错误的样本分开。

图 10 进一步清楚展示排序行为。BINEVAL（Claude）在两个数据集上都显示出最清晰的正趋势，拟合线比其他方法更好地跟随对角线。BINEVAL（gpt-oss）遵循相同模式，但离散更大，与表 8 中强但略低的相关性一致。UniEval（T5）产生正趋势，但将许多预测压缩到较窄的上部区间，从而限制了区分能力。G-Eval（gpt-oss）在 CNN/DM 上几乎平坦，在 XSum 上也只弱增长；UniEval（gpt-oss）则只呈现粗糙、量化的输出。表格和图共同显示，QAGS 上分解的关键收益不仅是更高相关性，还有更好使用分数范围：BINEVAL 会为不同事实错误赋予有意义的不同分数，而不是把它们塌缩成一小组几乎相同的预测。

## 附录 E. SummEval 的二元问题

表 9–12 列出了 BINEVAL 为每个 SummEval 评估维度自动生成的二元问题。这些问题即第 5.6 节和图 3 中引用的问题。每个问题都被设计成“是”表示输出满足标准，“否”表示违反标准。

**表 9：SummEval 连贯性的二元问题（8 个问题）。**

| ID | 问题 | 是率 |
|---|---|---:|
| Q1 | 摘要是否具有清晰定义的结构（例如清楚的开头、中间和/或结尾），而不是看起来随机拼接？ | 0.66 |
| Q2 | 摘要中的句子是否以合理且符合逻辑的顺序排列？ | 0.68 |
| Q3 | 摘要是否避免仅仅成为一堆松散相关事实或信息？ | 0.43 |
| Q4 | 摘要中的句子是否从一个到下一个逻辑流动，并有清楚的过渡或连接？ | 0.44 |
| Q5 | 摘要是否保持对单一主题的统一关注，而不是漂移到多个无关主题？ | 0.96 |
| Q6 | 摘要是否覆盖新闻文章的主题？ | 0.85 |
| Q7 | 摘要是否覆盖新闻文章的关键点？ | 0.10 |
| Q8 | 摘要是否以清楚、易于跟随和理解的方式呈现信息？ | 0.61 |

**表 10：SummEval 一致性的二元问题（7 个问题）。**

| ID | 问题 | 是率 |
|---|---|---:|
| Q1 | 摘要中的所有陈述是否都由源文章蕴含或支持？ | 0.75 |
| Q2 | 与源文章相比，摘要是否没有事实错误？ | 0.82 |
| Q3 | 摘要是否没有幻觉事实，即捏造且不在源文章中的信息？ | 0.81 |
| Q4 | 摘要中的所有命名实体（人物、组织、地点）是否都按照源文章准确表示？ | 0.91 |
| Q5 | 摘要中的所有数字声明（日期、统计、数量、金额）是否与源文章一致？ | 0.95 |
| Q6 | 摘要中描述的因果关系和事件顺序是否与源文章一致？ | 0.87 |
| Q7 | 摘要是否避免歪曲或扭曲源文章信息的含义？ | 0.76 |

**表 11：SummEval 流畅性的二元问题（7 个问题）。**

| ID | 问题 | 是率 |
|---|---|---:|
| Q1 | 摘要是否没有语法错误？ | 0.54 |
| Q2 | 摘要是否没有拼写错误？ | 0.71 |
| Q3 | 摘要是否没有标点错误？ | 0.33 |
| Q4 | 摘要是否使用恰当且自然的词语选择？ | 0.81 |
| Q5 | 摘要是否具有结构良好且易于跟随的句子？ | 0.70 |
| Q6 | 摘要整体上是否读起来流畅且自然？ | 0.52 |
| Q7 | 摘要是否容易理解，不会因为语言问题而需要重读？ | 0.76 |

**表 12：SummEval 相关性的二元问题（5 个问题）。**

| ID | 问题 | 是率 |
|---|---|---:|
| Q1 | 摘要是否涉及源文章的主题或中心事件？ | 0.95 |
| Q2 | 摘要是否至少覆盖源文章的一些关键点或重要细节？ | 0.50 |
| Q3 | 摘要是否没有显著冗余，例如用不同措辞多次重复同一点？ | 0.64 |
| Q4 | 摘要是否没有过多琐碎或不重要细节，以至于冲淡主要内容覆盖？ | 0.72 |
| Q5 | 摘要是否优先呈现源文章中最有新闻价值或最重要的信息，而不是关注次要或离题方面？ | 0.49 |
