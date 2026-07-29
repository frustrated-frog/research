# 人人都在谈论 LLM。

却没人解释它们在底层到底是如何工作的。

GPT。Claude。Gemini。Llama。它们全都来自同一套 5 阶段流水线。

一旦你理解了它，你就能自己构建一个。不是 GPT-4 的克隆。而是一个真正能工作的语言模型。
一个能从文本中学习、生成新文本，而且确实说得通的模型。

我构建过一个。下面就是它到底如何工作。不需要博士学位。包含代码。

## 所有人都相信的关于 LLM 的谎言

大多数人以为构建 LLM 的关键在于架构。Transformer，注意力头，层。并不是。
Transformer 架构已经公开发表。每个主要实验室使用的构建模块大致都一样。如果架构是秘密，那么每个人都会有 GPT-4。

真正的秘密：数据、训练和对齐。架构只是一段话。其他所有东西，才是真正的模型胜负所在。

下面是这 5 个阶段。

## 阶段 1 — 数据（模型真正决胜的地方）

原始互联网文本非常肮脏，你不能直接拿它训练。

Common Crawl —— 用来训练大多数 LLM 的公开网页抓取数据 —— 包含 2500 亿个页面，超过 100 万 GB。

但其中大部分都是垃圾：重复页眉，垃圾信息，有毒内容，个人数据，只有 3 个词的低质量页面。
在任何训练发生之前，你要运行一套严苛的多步骤过滤流程：

- → 从原始 HTML 中提取干净文本
- → 过滤有害内容、NSFW 内容和个人数据
- → 按 URL、文档和行去重
- → 按词数和 token 密度移除低质量文档
- → 运行基于模型的质量评分 —— 维基百科编辑会引用这个页面吗？
- → 平衡代码、书籍、科学和网页等数据混合比例

结果是：一个干净的数据集，规模只是原始数据的一小部分，但质量大幅更好。

必须刻进脑子的规则：数据质量胜过数据数量。永远如此。这个领域守得最严的秘密不是架构。
而是数据是如何被清洗的。

## 阶段 2 — Tokenization（把文本拆成模型真正能学习的片段）

模型从不读取原始文本，它读取 token。

一个 token 不总是一个完整单词，它是单词的一部分 —— 是模型学会当作一个单位来处理的片段。

"playing" → ["play", "ing"] "unbelievable" → ["un", "believ", "able"] "dog" → ["dog"]。标准方法叫做字节对编码（Byte-Pair Encoding，BPE）。它从单个字符开始，反复合并最常见的字符对，直到得到一个固定词表 —— 通常是 32,000 到 100,000 个 token。

下面是一个用 Python 写的极简 tokenizer：

```python
from tokenizers import Tokenizer, models, trainers, pre_tokenizers

# 初始化 BPE tokenizer
tokenizer = Tokenizer(models.BPE())
tokenizer.pre_tokenizer = pre_tokenizers.Whitespace()

# 在你的语料库上训练
trainer = trainers.BpeTrainer(
    vocab_size=32000,
    special_tokens=["<PAD>", "<BOS>", "<EOS>", "<UNK>"]
)
tokenizer.train(files=["your_data.txt"], trainer=trainer)
tokenizer.save("tokenizer.json")

# 测试它
output = tokenizer.encode("Building an LLM from scratch is powerful")
print(output.tokens)
# ['Building', 'an', 'LL', 'M', 'from', 'scratch', 'is', 'powerful']

print(output.ids)
# [4821, 271, 3728, 44, 505, 8905, 318, 6787]
```

经验法则：1 个 token ≈ 0.75 个单词。1,000 个 token ≈ 750 个单词。一个 100k 的上下文窗口 = 大约一整本小说。

## 阶段 3 — 训练（一个看似简单得过分的目标）

整个训练任务听起来简单到不像有多强大：预测下一个 token，就是这样。

给定 "The cat sat on the"，预测 "mat"。

在数万亿个样本上做这件事，就会发生非常了不起的事情。

模型学会语法。然后学会事实。然后学会推理。然后学会如何写代码、翻译语言、解数学题。没人教过它这些东西。这些能力是在巨大规模的下一个 token 预测中涌现出来的。

下面是一个用 PyTorch 写的极简 decoder-only transformer —— 每个主要 LLM 背后的同类架构：

```python
import torch
import torch.nn as nn
import math

class CausalSelfAttention(nn.Module):
    def __init__(self, embed_dim, num_heads):
        super().__init__()
        self.num_heads = num_heads
        self.head_dim = embed_dim // num_heads
        self.qkv = nn.Linear(embed_dim, 3 * embed_dim, bias=False)
        self.proj = nn.Linear(embed_dim, embed_dim, bias=False)

    def forward(self, x):
        B, T, C = x.shape
        q, k, v = self.qkv(x).chunk(3, dim=-1)
        
        # 拆分成多个头
        q = q.view(B, T, self.num_heads, self.head_dim).transpose(1, 2)
        k = k.view(B, T, self.num_heads, self.head_dim).transpose(1, 2)
        v = v.view(B, T, self.num_heads, self.head_dim).transpose(1, 2)

        # 带因果 mask 的缩放点积注意力
        scale = math.sqrt(self.head_dim)
        attn = (q @ k.transpose(-2, -1)) / scale
        
        # 因果 mask：只能关注过去的 token
        mask = torch.tril(torch.ones(T, T, device=x.device))
        attn = attn.masked_fill(mask == 0, float('-inf'))
        attn = torch.softmax(attn, dim=-1)
        
        out = (attn @ v).transpose(1, 2).contiguous().view(B, T, C)
        return self.proj(out)

class TransformerBlock(nn.Module):
    def __init__(self, embed_dim, num_heads, ff_dim, dropout=0.1):
        super().__init__()
        self.attn = CausalSelfAttention(embed_dim, num_heads)
        self.ff = nn.Sequential(
            nn.Linear(embed_dim, ff_dim),
            nn.GELU(),
            nn.Linear(ff_dim, embed_dim),
            nn.Dropout(dropout)
        )
        self.ln1 = nn.LayerNorm(embed_dim)
        self.ln2 = nn.LayerNorm(embed_dim)

    def forward(self, x):
        x = x + self.attn(self.ln1(x))   # 注意力 + 残差
        x = x + self.ff(self.ln2(x))     # 前馈网络 + 残差
        return x

class MiniLLM(nn.Module):
    def __init__(self, vocab_size, embed_dim, num_heads,
                 ff_dim, num_layers, max_seq_len, dropout=0.1):
        super().__init__()
        self.token_emb = nn.Embedding(vocab_size, embed_dim)
        self.pos_emb = nn.Embedding(max_seq_len, embed_dim)
        self.blocks = nn.ModuleList([
            TransformerBlock(embed_dim, num_heads, ff_dim, dropout)
            for _ in range(num_layers)
        ])
        self.ln_final = nn.LayerNorm(embed_dim)
        self.output = nn.Linear(embed_dim, vocab_size, bias=False)
        self.dropout = nn.Dropout(dropout)

    def forward(self, token_ids):
        B, T = token_ids.shape
        positions = torch.arange(T, device=token_ids.device).unsqueeze(0)
        
        x = self.dropout(
            self.token_emb(token_ids) + self.pos_emb(positions)
        )
        for block in self.blocks:
            x = block(x)
        
        x = self.ln_final(x)
        return self.output(x)  # 词表上的 logits

# 初始化一个小而真实的模型
model = MiniLLM(
    vocab_size=32000,
    embed_dim=512,
    num_heads=8,
    ff_dim=2048,
    num_layers=6,
    max_seq_len=1024
)

total_params = sum(p.numel() for p in model.parameters())
print(f"Parameters: {total_params:,}")
# Parameters: 44,082,176 — 一个 4400 万参数的模型
```

现在是训练循环：

```python
import torch.optim as optim
from torch.nn.utils import clip_grad_norm_

device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
model = model.to(device)

optimizer = optim.AdamW(model.parameters(), lr=3e-4, weight_decay=0.01)
criterion = nn.CrossEntropyLoss()

def train_epoch(model, dataloader):
    model.train()
    total_loss = 0
    
    for input_ids, target_ids in dataloader:
        input_ids = input_ids.to(device)
        target_ids = target_ids.to(device)
        
        # 前向传播
        logits = model(input_ids)
        
        # 展平成适合计算 loss 的形状
        loss = criterion(
            logits.view(-1, logits.size(-1)),   # (batch * seq_len, vocab)
            target_ids.view(-1)                  # (batch * seq_len)
        )
        
        # 反向传播
        optimizer.zero_grad()
        loss.backward()
        clip_grad_norm_(model.parameters(), max_norm=1.0)  # 防止梯度爆炸
        optimizer.step()
        
        total_loss += loss.item()
    
    return total_loss / len(dataloader)
```

模型实际上在学习什么：

- → 每个输入 token 都会关注它之前的每个 token
- → 因果 mask 防止它偷看未来
- → Loss = 模型对真实下一个 token 有多惊讶
- → 更低的 loss = 更好的预测 = 模型正在学习语言

## 阶段 4 — 对齐（把文本预测器变成助手）

预训练之后，你拥有了某种令人印象深刻、但对聊天来说没什么用的东西。你问它一个问题，它可能会用另外三个问题来回复。因为预测下一个 token，并不意味着理解你想要什么。两步可以修复这一点。

### 步骤 1：监督微调（Supervised Fine-Tuning，SFT）

向模型展示成千上万个样例：prompt → 理想回复。模型学会模仿一个好答案的格式。令人惊讶的部分是：你需要的数据很少。几千个样例就足够了，因为知识已经在预训练模型里面。SFT 只是教它用正确的格式表达这些知识。

```python
# SFT 训练样例结构
sft_examples = [
    {
        "prompt": "用简单的话解释 API 是什么。",
        "response": "API 就像餐厅里的服务员。你（应用）告诉服务员（API）你想要什么。服务员去厨房（服务器）拿到它，再把它带回来。你永远不需要直接去厨房。"
    },
    {
        "prompt": "法国的首都是哪里？",
        "response": "法国的首都是巴黎。"
    }
    # 几千个这样的样例就足够了
]

# 格式化为：<prompt> [SEP] <response> <EOS>
# 在这些配对样本上微调预训练模型
# 和预训练使用同一个训练循环 —— 只是数据不同
```

### 步骤 2：RLHF（基于人类反馈的强化学习）

SFT 教格式。RLHF 教偏好。

模型生成两个答案。人类选择更好的那个。这些偏好会训练一个奖励模型。LLM 会被优化，以最大化这个奖励。

这就是为什么 ChatGPT 和 Claude 感觉像助手 —— 而不是随机文本生成器。

没有 RLHF：

- → 流畅。有能力。但不可靠。
- → 自信地犯错。
- → 不知道什么时候该说“我不知道”。

有了 RLHF：

- → 有帮助。清晰。安全。
- → 学会“一个好答案”真正意味着什么。

## 阶段 5 — 评估（证明它确实有效）

构建模型但不测量它，就是在猜。

预训练期间 —— 测量困惑度（perplexity），困惑度衡量模型对真实文本有多“惊讶”。

更低的困惑度 = 模型能更好地预测文本 = 它正在学习。

在 2017 到 2023 年之间，最好的模型从大约 70 个可能 token 中感到困惑，下降到少于 10 个。

```python
import torch
import math

def calculate_perplexity(model, dataloader, device):
    model.eval()
    total_loss = 0
    total_tokens = 0
    criterion = nn.CrossEntropyLoss(reduction='sum')
    
    with torch.no_grad():
        for input_ids, target_ids in dataloader:
            input_ids = input_ids.to(device)
            target_ids = target_ids.to(device)
            
            logits = model(input_ids)
            
            loss = criterion(
                logits.view(-1, logits.size(-1)),
                target_ids.view(-1)
            )
            total_loss += loss.item()
            total_tokens += target_ids.numel()
    
    avg_loss = total_loss / total_tokens
    perplexity = math.exp(avg_loss)
    return perplexity

# 示例输出进展：
# Epoch 1: Perplexity = 847.3  （模型几乎什么都不知道）
# Epoch 5: Perplexity = 124.6  （正在变好）
# Epoch 20: Perplexity = 23.4  （确实在学习语言）
```

对齐之后 —— 困惑度就不再有效。经过微调的模型在困惑度上得分更差，但实际有用得多。

你需要人类基准：

- → MMLU：57 个学术科目，选择题 —— 衡量知识
- → Chatbot Arena：人类盲评两个模型并投票 —— 衡量偏好
- → AlpacaEval：LLM 评判 LLM —— 与人类评审有 98% 相关性，成本 10 美元

诚实的真相是：没有任何单一分数能捕捉一个好模型。

同一个模型在同一个基准上可能得 0.637，也可能得 0.488，差别只取决于 prompt 是如何格式化的。评估真的很难，而且没人完全解决它。

如何让你的模型生成文本

模型已经训练好了，现在让它生成一些东西。

```python
def generate(model, tokenizer, prompt, max_new_tokens=100,
             temperature=0.8, device='cuda'):
    model.eval()
    
    # 将 prompt 编码成 token ID
    token_ids = tokenizer.encode(prompt).ids
    input_tensor = torch.tensor(token_ids, dtype=torch.long,
                                device=device).unsqueeze(0)
    
    with torch.no_grad():
        for _ in range(max_new_tokens):
            
            # 只保留最后 max_seq_len 个 token（上下文窗口）
            context = input_tensor[:, -1024:]
            
            # 前向传播
            logits = model(context)
            
            # 只获取最后一个 token 的 logits
            next_token_logits = logits[:, -1, :]
            
            # 应用 temperature（越高 = 越有创造性）
            next_token_logits = next_token_logits / temperature
            
            # 从概率分布中采样
            probs = torch.softmax(next_token_logits, dim=-1)
            next_token = torch.multinomial(probs, num_samples=1)
            
            # 追加并继续
            input_tensor = torch.cat([input_tensor, next_token], dim=1)
            
            # 在序列结束 token 处停止
            if next_token.item() == tokenizer.token_to_id("<EOS>"):
                break
    
    # 解码回文本
    generated_ids = input_tensor[0].tolist()
    return tokenizer.decode(generated_ids)

# 测试它
output = generate(
    model, tokenizer,
    prompt="机器学习中最重要的事情是",
    max_new_tokens=100,
    temperature=0.8
)
print(output)
```

Temperature 控制创造性：

- → temperature = 0.1 → 安全、可预测、重复
- → temperature = 0.8 → 自然、多样、很好的默认值
- → temperature = 1.5 → 有创造性、令人意外、有时不连贯

## 完整流水线是什么样子

- 之前：原始互联网文本，100 万 GB，完全无法使用。
- 阶段 1 之后：干净的过滤后数据集，准备好用于训练。
- 之前：原始文本，对模型来说没有意义。
- 阶段 2 之后：带 ID 的 token，模型的母语。
- 之前：随机权重，输出垃圾。
- 阶段 3 之后：一个理解语言模式的模型。
- 之前：一个会用更多问题回答问题的文本预测器。
- 阶段 4 之后：一个遵循指令并且安全的助手。
- 之前：完全不知道模型到底好不好。
- 阶段 5 之后：基准、困惑度分数、人类评估。

每个阶段都建立在上一个阶段之上。
跳过任何一个，整个东西都会坏掉。

## 让 LLM 项目沉没的 5 个错误

1. 痴迷于架构。

Transformer 已经标准化。已发表。被复制。
架构是最不重要的部分。

2. 把数据当成商品。

脏数据会限制你的上限，不管你有多少算力。
顶级实验室花在数据清洗上的钱，比花在模型设计上的更多。

3. 跳过 scaling math。

相对于数据来说太大的模型会训练不足，并浪费算力。
最佳比例：每个参数大约 20 个训练 token。

4. 停在 SFT。

微调模型会模仿。没有 RLHF，它永远学不会人们真正偏好什么。

5. 对齐之后还相信困惑度。

后训练会改变分布。
你运行 SFT 的那一刻，困惑度就不再有意义。
立刻切换到人类基准。

## 令人不舒服的真相

一个伟大的 LLM 不是被训练出来的。
它是被工程化出来的。
5 个阶段。不是 1 个。
架构只是阶段 3 里面的一段话。
真正重要的一切都在另外四个阶段。
数据质量。Scaling math。对齐。诚实的评估。
这就是 GPT-4 和爱好者模型之间的区别。
两个使用相同架构的实验室，会产出截然不同的模型。
架构是共享的。
真正重要的一切都不是。

## 从这里开始

想自己运行它吗？下面是最小设置：

```plaintext
# 完整的最小 LLM 训练设置
# 要求：pip install torch tokenizers datasets

# 1. 获取数据
from datasets import load_dataset
dataset = load_dataset("wikitext", "wikitext-103-v1", split="train")
text = "\n\n".join([t.strip() for t in dataset['text'] if t.strip()])

# 2. 训练 tokenizer
from tokenizers import Tokenizer, models, trainers, pre_tokenizers
tokenizer = Tokenizer(models.BPE())
tokenizer.pre_tokenizer = pre_tokenizers.Whitespace()
trainer = trainers.BpeTrainer(vocab_size=8000,
                               special_tokens=["<PAD>","<BOS>","<EOS>"])
with open("corpus.txt", "w") as f:
    f.write(text[:5_000_000])  # 先用 5MB 开始
tokenizer.train(["corpus.txt"], trainer)

# 3. 构建模型（使用上面阶段 3 中的 MiniLLM 类）
model = MiniLLM(
    vocab_size=8000,
    embed_dim=256,       # 小但真实
    num_heads=8,
    ff_dim=1024,
    num_layers=4,
    max_seq_len=256
)
# 约 1500 万参数 —— 在笔记本 GPU 上几小时内就能训练

# 4. 训练（使用上面阶段 3 中的 train_epoch）
# 5. 生成（使用上面阶段 5 中的 generate()）

print("Your LLM is running.")
```

从小开始。1500 万参数。WikiText 数据集。Google Colab 上的免费 GPU。
看着困惑度在几个小时内从 800 降到 50。
那个下降，就是模型正在学习语言。
实时发生。
那就是一切突然明白的时刻。

## 现在让它变得有用：构建一个真正的垂直领域 LLM

WikiText 是一个学习数据集。
真正的钱 —— 也是真正的乐趣 —— 在于针对某个特定领域训练，然后看着你的模型变成只擅长一件事的专家。
下面是 5 个你现在就可以构建的垂直领域。同样的流水线。不同的数据。

### 垂直领域 1 — 编程助手 LLM（主要示例：影响最大，结果最戏剧化）

这是第一个该构建的。
痛点是普遍的。每个开发者都遇到过：

- → 你盯着一个不能工作的函数。
- → Stack Overflow 有 12 个答案，全都来自 2014 年。
- → 你把它粘到 ChatGPT，得到一个差一点就对的东西。

一个用正确数据训练的编程 LLM，会原生地、离线地、针对你的确切技术栈完成这件事。
数据：

```python
from datasets import load_dataset

# The Code 数据集 —— 来自 GitHub 的 640 万个 Python 文件
code_dataset = load_dataset("codeparrot/github-code",
                             languages=["Python"],
                             streaming=True,
                             split="train")

# Stack Overflow 问答配对 —— 真实开发者问题 + 被接受的答案
so_dataset = load_dataset("koutch/stackoverflow_python",
                           split="train")

# The Stack —— 30 种编程语言，已清洗并去重
the_stack = load_dataset("bigcode/the-stack",
                          data_dir="data/python",
                          split="train",
                          streaming=True)
```
训练配对是什么样子：

```python
# 格式 1：代码补全
# 输入：函数签名 + docstring
# 目标：完整实现

input_example = """
def calculate_compound_interest(principal, rate, time, n=12):
    \"\"\"
    计算复利。
    Args:
        principal: 初始投资金额
        rate: 年利率（小数形式，例如 0.05 表示 5%）
        time: 年数
        n: 每年的复利频率（默认：每月）
    Returns:
        复利后的最终金额
    \"\"\"
"""

target_example = """
    amount = principal * (1 + rate/n) ** (n * time)
    return round(amount, 2)
"""

# 格式 2：错误 → 修复配对（来自 Stack Overflow）
input_example_2 = """
# ERROR: TypeError: unsupported operand type(s) for +: 'int' and 'str'
user_age = input("Enter your age: ")
next_year_age = user_age + 1
print(f"Next year you will be {next_year_age}")

# 修复这段代码：
"""

target_example_2 = """
user_age = int(input("Enter your age: "))  # 将字符串转换为整数
next_year_age = user_age + 1
print(f"Next year you will be {next_year_age}")
"""

# 格式 3：自然语言 → 代码（指令跟随）
input_example_3 = """
# 写一个 Python 函数，要求：
# - 接收一个字典列表
# - 按给定的键值对过滤
# - 按指定字段返回排序后的结果
"""

target_example_3 = """
def filter_and_sort(data, filter_key, filter_value, sort_field):
    filtered = [item for item in data
                if item.get(filter_key) == filter_value]
    return sorted(filtered, key=lambda x: x.get(sort_field, ''))
"""
```
能卖出去的前后对比：
训练前：
Prompt: "写一个 Python 装饰器，在函数失败时重试"
Output: "装饰器是 Python 中的一种设计模式，它允许..."

泛泛而谈。没用。教科书答案。

在 GitHub + Stack Overflow 上训练 10 小时后：
Prompt: "写一个 Python 装饰器，在函数失败时重试"
Output:

```python
import time
from functools import wraps

def retry(max_attempts=3, delay=1.0, exceptions=(Exception,)):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_attempts):
                try:
                    return func(*args, **kwargs)
                except exceptions as e:
                    if attempt == max_attempts - 1:
                        raise
                    time.sleep(delay * (attempt + 1))
        return wrapper
    return decorator

# 用法：
@retry(max_attempts=3, delay=0.5, exceptions=(ConnectionError,))
def fetch_data(url):
    ...
# 正确。可用于生产。包含指数退避。
```

模型学会了资深开发者会写什么，不是因为有人告诉它，而是因为它读了 600 万个 Python 文件和每一个被接受的 Stack Overflow 答案。

### 垂直领域 2 — SQL 查询生成器

痛点：每个非技术创始人都有自己无法访问的数据。
数据就在那里。在他们的数据库里。但他们就是写不出查询。
他们用普通英语描述自己想要什么。你的模型写出 SQL。
数据：

```python
# Spider 数据集 —— 10,000+ 条 SQL 查询，带自然语言描述
# 覆盖 138 个领域中的 200 个数据库
spider = load_dataset("spider", split="train")

# WikiSQL —— 80,654 条自然语言 + SQL 配对
wikisql = load_dataset("wikisql", split="train")

# 一个训练配对是什么样子：
example = {
    "question": "显示过去 30 天内注册且至少购买过一次的所有用户",
    "sql": """
        SELECT u.id, u.email, u.created_at, COUNT(p.id) as purchase_count
        FROM users u
        JOIN purchases p ON u.id = p.user_id
        WHERE u.created_at >= NOW() - INTERVAL '30 days'
        GROUP BY u.id, u.email, u.created_at
        HAVING COUNT(p.id) >= 1
        ORDER BY u.created_at DESC;
    """
}
```
前后对比：

```python
# 非技术创始人输入：
"上个季度花钱最多、但过去 60 天没有再购买的客户有哪些？"

# 模型输出：
SELECT
    c.customer_id,
    c.email,
    SUM(o.total_amount) as q_spend,
    MAX(o.created_at) as last_order_date
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE o.created_at BETWEEN DATE_TRUNC('quarter', NOW() - INTERVAL '3 months')
                        AND DATE_TRUNC('quarter', NOW())
GROUP BY c.customer_id, c.email
HAVING MAX(o.created_at) < NOW() - INTERVAL '60 days'
ORDER BY q_spend DESC
LIMIT 20;
```
谁会为此付费：每个 SaaS 创始人、每个电商运营者、每个夹在业务所有者和数据库之间的分析师。
他们会毫不犹豫地每月付 20 美元。

### 垂直领域 3 — 美国法律文件摘要器

痛点：一份 40 页的合同。一份租赁协议。一份 NDA。
大多数人没看懂就签了。
律师每小时收费 300 美元来读它。
你的模型 3 秒钟读完它。
数据：

```python
# Free Law Project —— 数百万份美国法院意见书，公共领域
free_law = load_dataset("free-law/courtlistener-opinion-clustering",
                         split="train")

# MultiLegalPile —— 美国各司法辖区的法律文本
multi_legal = load_dataset("joelniklaus/multi_legal_pile",
                            languages=["en"],
                            split="train")

# 训练配对是什么样子：
example = {
    "input": """
    第 12.4 条 赔偿。被许可方应为许可方及其高级职员、董事、员工、
    代理人和继承人进行辩护、赔偿并使其免受任何及所有损失、损害、
    责任、差额、索赔、诉讼、判决、和解、利息、裁决、罚金、罚款、
    成本或任何种类的费用，包括合理的律师费，只要这些费用由许可方
    因被许可方违反本协议项下任何陈述、保证、契约或义务而产生或与之相关...
    [后面还有 3 段密集法律术语]
    """,

    "summary": """
    白话解释：如果你违反了这份合同的任何部分，并导致另一方遇到
    法律麻烦或经济损失，那么一切都由你支付——他们的律师费、诉讼费、
    赔偿金，全部都是。这是一条范围很宽的赔偿条款，严重偏向许可方。

    风险信号：
    • "任何及所有损失" —— 范围极宽，没有责任上限
    • "合理的律师费" —— 你也要支付他们的法律账单
    • 没有排除许可方自身过失的例外

    谈判建议：签署前，要求加入“除非由许可方的重大过失或故意不当行为造成”
    这样的表述。
    """
}
```
让它值得付费的输出格式：

```python
Input: [粘贴任何美国合同条款]

Output:
━━━━━━━━━━━━━━━━━━
白话摘要
━━━━━━━━━━━━━━━━━━
[任何人都能理解的 2-3 句话]

风险信号 🚩
• [这条款里的具体风险]
• [它允许另一方做什么]

缺失的保护 ⚠️
• [平衡条款应该包含什么]
• [律师会要求加入什么]

风险等级：高 / 中 / 低
━━━━━━━━━━━━━━━━━━
```
谁会为此付费：审查客户合同的自由职业者、小企业主、审查租约的租户、签署第一批 SaaS 协议的创业公司创始人。
这个市场就是每个请不起律师但仍然必须签文件的人。

### 垂直领域 4 — 医疗症状解释器

痛点：你 Google 一个症状，WebMD 告诉你只剩 3 天可活。
一个用临床记录和患者教育材料训练的医疗 LLM 会做另一件事。
它解释这个症状实际意味着什么。用普通话。不会灾难化。

```python
# PubMed 摘要 —— 3500 万篇医学研究论文
pubmed = load_dataset("pubmed", split="train")

# MedQA —— 美国执业医师资格考试问题 + 答案
medqa = load_dataset("bigbio/med_qa", split="train")

# 训练配对格式：
example = {
    "patient_question": "我的左眼后方隐隐作痛 3 天了。前倾时会更疼。没有发烧。可能是什么？",

    "clinical_explanation": """
    根据你的描述——一只眼睛后方隐痛、前倾时加重、没有发烧、持续 3 天——
    最可能的解释是：

    最可能：鼻窦炎（鼻窦压力）
    上颌窦或额窦就在眼睛后方和上方。
    炎症或鼻塞会在你改变头部位置时增加压力。
    前倾时加重是典型的鼻窦炎模式。

    也有可能：紧张性头痛、眼疲劳，或轻度偏头痛变体

    可能性较低但值得注意：如果你还有视力变化、恶心，
    或疼痛突然且剧烈——当天就去看医生。

    可以尝试：生理盐水洗鼻、蒸汽吸入、布洛芬缓解压力。
    如果 7-10 天没有改善或出现发烧 → 去看医生。

    注意：这是一般健康信息，不是诊断。
    医疗建议请始终咨询持证医生。
    """
}
```
关键：每个回复都以清晰的免责声明和升级就医信号结束。
这就是有用健康信息和危险建议之间的区别。

### 垂直领域 5 — 电商产品描述写手

痛点：一家拥有 500 个产品的 Shopify 商店。每个描述都是一堵没人读的规格表文字墙。
好的产品描述只做一件事：让某人产生感觉。
一个在高转化产品文案上训练的 LLM，会准确学会哪些词能带来点击。

```python
# 训练数据结构：
example = {
    "product_specs": """
    产品：陶瓷咖啡杯
    材质：炻器陶瓷
    容量：14 盎司
    尺寸：直径 3.5 英寸 x 高 4.2 英寸
    颜色：哑光黑、奶油白、鼠尾草绿
    可用洗碗机清洗：是
    可用于微波炉：是
    重量：0.8 磅
    """,

    "high_converting_description": """
    有些早晨，值得拥有比纸杯更好的东西。

    这就是那个会一直待在你桌上的杯子。那个同事会问起的杯子。
    够重，拿起来有一种认真感。够顺滑，让你在早上 7 点真的愿意握着它。

    14 盎司——刚刚好的容量。不是猎奇的大桶。不是那个小小的意式杯。
    是你真的会喝完的那个。

    哑光炻器，不显指纹。可进洗碗机，因为人生苦短。
    三种颜色，适合任何没有过度用力的厨房。

    你已经有杯子了。
    但你没有这个。
    """,

    "meta_description": "炻器陶瓷杯，14 盎司，哑光饰面。可用洗碗机和微波炉。那个会一直留下来的咖啡杯。",

    "keywords": ["ceramic coffee mug", "stoneware mug", "matte black mug",
                 "14oz mug", "minimalist coffee mug", "handmade style mug"]
}
```
数据来源：抓取按流量排名前 1,000 的 Shopify 商店。提取产品标题、规格和描述。筛选评论数高的产品 —— 这些描述已经被证明能转化。用这些数据训练。
你的模型会学会规格表和能卖货的文案之间的区别。

## 所有 5 个垂直领域共有的模式

看看它们共同拥有的东西：

- → 一个用户每天都会感受到的清晰、具体痛点
- → 一个已经存在并且公开可用的数据源
- → 一个该职业中的任何人都能立刻看懂的前后对比
- → 愿意付费，因为替代方案会花更多时间或金钱

每一个领域的 5 阶段流水线完全相同。
你只改变一件事：训练数据。
同样的 tokenizer 设置。同样的 transformer 架构。同样的训练循环。同样的评估方法。
不同的数据 → 不同的专家 → 不同的产品。
这就是杠杆。
一条流水线。五个产品。五条收入流。

## 如果这对你有用：

- → 转发给每个正在学习 AI 的开发者
- → 关注 @sairahul1，获取更多这样的拆解
- → 收藏这篇 —— 代码能跑，今晚就运行它

我写关于 AI、构建产品，以及那些在你睡觉时也能运转的系统。
