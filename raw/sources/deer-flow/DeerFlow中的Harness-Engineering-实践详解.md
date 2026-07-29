# Harness Engineering 视角下的 DeerFlow 2.0 实践

> 来源：基于 deer-flow-main 源码分析 + Harness Engineering 理念
> 时间：2026-04-08
> 提示：本文档包含大量源码注释，建议配合 DeerFlow 源码一起阅读

---

## 0. 什么是 Harness Engineering

当模型已经足够聪明，工程师的核心工作不再是生成代码，而是**给 AI 搭跑道**——设计它所处的环境，让它的能力得以充分发挥。

同一模型，换个环境设计，性能可以差 64%。这不是 AI 的问题，而是**环境设计**的问题。

**核心命题：如何让 AI 更好地执行任务**——类似给一个聪明的大脑装上手和脚，让它不只能想，还能做。

DeerFlow 2.0 是这个理念的完整实现。下面从 6 个维度，结合源码逐行讲解。

---

## 1. 持久状态：解决 AI 失忆问题

### 1.1 问题本质

每次新建对话，AI 就是一张白纸。Harness 的第一个任务就是**让 AI 记住之前发生了什么**。

DeerFlow 实现了两层记忆系统：
- **会话级状态**：每个对话线程独立的状态
- **跨会话记忆**：持久化的用户偏好和事实

### 1.2 第一层：ThreadState（会话级状态）

```python
# 文件：backend/packages/harness/deerflow/agents/thread_state.py

from langchain.agents import AgentState

class ThreadState(AgentState):
    """
    ThreadState 是 Agent 的"工作内存"。

    相比 LangChain 原生的 AgentState，ThreadState 扩展了：
    - sandbox: 当前线程使用的 sandbox 实例
    - thread_data: 当前线程的文件系统路径信息
    - title: 对话标题（用于列表展示）
    - artifacts: AI 产生的文件列表（可展示给用户）
    - todos: Plan 模式下的任务清单
    - uploaded_files: 用户上传的文件记录
    - viewed_images: 已经查看过的图片（base64 缓存）

    每个字段都有明确语义，不是随意堆砌。
    """
    sandbox: NotRequired[SandboxState | None]
    thread_data: NotRequired[ThreadDataState | None]
    title: NotRequired[str | None]
    artifacts: Annotated[list[str], merge_artifacts]   # 去重合并
    todos: NotRequired[list | None]
    uploaded_files: NotRequired[list[dict] | None]
    viewed_images: Annotated[dict[str, ViewedImageData], merge_viewed_images]
```

### 1.3 自定义 Reducer：状态合并的艺术

```python
# ThreadState 中的 artifacts 字段使用了 Annotated[T, reducer_func] 模式
# 这个模式允许我们定义"当多个中间件都想更新同一个字段时，如何合并"

def merge_artifacts(existing: list[str] | None, new: list[str] | None) -> list[str]:
    """
    合并两个 artifacts 列表，自动去重。

    为什么需要去重？
    - AI 可能在不同轮次多次调用 present_files 产生相同路径
    - 如果不去重，同一个文件路径会重复出现在列表里
    - 使用 dict.fromkeys() 在去重的同时保持原始顺序（Python 3.7+ dict 有序）

    参数：
        existing: 已经存在的 artifacts 列表（可能是 None）
        new: 新产生的 artifacts 列表（可能是 None）

    返回：
        合并去重后的完整列表
    """
    if existing is None:
        return new or []    # 首次初始化
    if new is None:
        return existing     # 没有新内容，保留原样
    # 核心逻辑：dict.fromkeys() 去重 + 保持顺序
    return list(dict.fromkeys(existing + new))


def merge_viewed_images(existing: dict, new: dict) -> dict:
    """
    合并 viewed_images 字典，有特殊逻辑：空字典 = 清空所有图片。

    为什么有这个特殊设计？
    - ViewImageMiddleware 处理完图片后，需要"清除"已处理的记录
    - 但 LangChain 的 reducer 不支持"删除"操作，只能返回新值
    - 所以约定：如果 new 是空字典 {}，就表示"清空所有图片"

    这是一种通过返回值语义来表达"操作"的技巧。
    """
    if len(new) == 0:
        return {}  # 空字典 = 清空信号
    return {**existing, **new}  # 正常合并，new 覆盖 existing
```

### 1.4 第二层：跨会话持久记忆（Memory System）

```python
# 文件：backend/packages/harness/deerflow/agents/memory/updater.py
# Memory 系统的核心职责：从对话历史中提取用户事实，持久化存储

class MemoryUpdater:
    """
    MemoryUpdater 负责从对话历史中提取结构化信息。

    工作流程：
    1. 接收一个对话历史列表（HumanMessage + AIMessage）
    2. 调用 LLM 分析对话，提取：
       - 用户工作背景（workContext）
       - 用户个人背景（personalContext）
       - 当前最关心的事（topOfMind）
       - 具体事实（Facts，带置信度分数）
    3. 与现有 memory.json 合并（去重）
    4. 原子写入（临时文件 + rename，防止写坏）

    关键设计：fact 去重
    - LLM 每次提取可能产生重复事实
    - 我们在合并前做标准化去重：strip() 后比较内容
    - 相同内容的 fact 不会重复添加
    """

    def update_memory(self, conversation: list[Message]) -> None:
        # 1. 过滤：只保留用户输入和 AI 最终回复
        #    中间的 tool_calls、tool_results 全部丢弃
        filtered = self._filter_messages(conversation)

        # 2. LLM 提取（通过 prompt 模板指导 LLM 输出结构化 JSON）
        extracted = self._llm_extract(filtered)

        # 3. 加载现有记忆
        existing = self._load_memory()

        # 4. 去重合并（关键！）
        for new_fact in extracted.facts:
            # 标准化：去除首尾空白后比较
            normalized = new_fact.content.strip()
            is_duplicate = any(
                f.content.strip() == normalized
                for f in existing.facts
            )
            if not is_duplicate:
                existing.facts.append(new_fact)

        # 5. 原子写入：先写临时文件，再 rename
        #    目的：防止写入过程中崩溃导致 memory.json 损坏
        temp_path = existing.path + ".tmp"
        with open(temp_path, "w") as f:
            json.dump(existing.to_dict(), f, ensure_ascii=False)
        os.rename(temp_path, existing.path)  # POSIX 原子操作
```

### 1.5 防抖队列：减少 LLM 调用

```python
# 文件：backend/packages/harness/deerflow/agents/memory/queue.py
# MemoryUpdateQueue 实现防抖逻辑，避免每次对话都触发 LLM 调用

class MemoryUpdateQueue:
    """
    防抖队列的核心思想：合并短时间内的多次更新请求。

    场景：
    - 用户在 30 秒内连续发了 5 条消息
    - 正常逻辑：每条消息都触发一次 LLM 记忆提取 = 5 次调用
    - 防抖逻辑：只触发 1 次 LLM 调用 = 大幅节省成本

    实现方式：
    - 每次有新消息，进入"等待"状态
    - 30 秒内再有新消息，重置等待计时器
    - 30 秒结束后，才真正触发 LLM 调用
    """

    def __init__(self, debounce_seconds: float = 30.0):
        self._debounce = debounce_seconds  # 防抖窗口（秒）
        self._pending: dict[str, list[Message]] = {}  # 待处理队列
        self._lock = threading.Lock()

    def enqueue(self, thread_id: str, messages: list[Message]) -> None:
        """
        将消息加入待处理队列。

        关键逻辑：同一线程的消息会合并，不重复触发 LLM。
        不同线程的消息分开排队，互不影响。
        """
        with self._lock:
            if thread_id in self._pending:
                # 已有待处理消息，合并（不重置计时器）
                self._pending[thread_id].extend(messages)
            else:
                # 新线程，创建队列并启动防抖计时器
                self._pending[thread_id] = messages
                self._schedule_flush(thread_id, self._debounce)
```

---

## 2. 执行环境：给 AI 手和脚

### 2.1 问题本质

AI 光能"想"不够，还得能"做"——读文件、写代码、跑命令、查网页。Harness 需要给 AI 提供**完整的执行能力**。

DeerFlow 通过 Sandbox 抽象层实现。

### 2.2 Sandbox 抽象接口

```python
# 文件：backend/packages/harness/deerflow/sandbox/sandbox.py

class Sandbox(ABC):
    """
    Sandbox 是 DeerFlow 执行环境的抽象核心。

    为什么需要抽象？
    - 本地开发用 LocalSandbox（直接操作文件系统）
    - 生产环境用 AioSandbox（Docker 隔离，安全）
    - 未来可能支持 K8s Sandbox（更大规模隔离）

    抽象接口定义了 AI 能做的所有操作：
    - execute_command: 执行 bash 命令
    - read_file / write_file: 文件 IO
    - list_dir: 目录浏览
    - glob / grep: 文件搜索
    - update_file: 二进制文件写入

    任何实现了这些方法的类，都可以作为 Sandbox 使用。
    这种设计叫做"依赖倒置"——上层不依赖具体实现，而依赖抽象接口。
    """

    _id: str  # 每个 sandbox 实例有唯一 ID

    @abstractmethod
    def execute_command(self, command: str) -> str:
        """
        在 sandbox 中执行 bash 命令。

        为什么返回 str 而不是结构化数据？
        - 保持简单，bash 输出本来就是文本
        - 复杂输出由调用方解析

        安全注意：
        - 实际执行前会经过 SandboxAuditMiddleware 审计
        - 高风险命令（如 rm -rf /）会被拦截
        """
        pass

    @abstractmethod
    def read_file(self, path: str) -> str:
        """
        读取文件内容。

        参数：path 是虚拟路径（如 /mnt/user-data/workspace/test.py）
        内部会翻译成真实物理路径。
        """
        pass

    @abstractmethod
    def write_file(self, path: str, content: str, append: bool = False) -> None:
        """
        写入文件。

        参数：
            path: 虚拟路径
            content: 文件内容
            append: False = 覆盖写，True = 追加写
        """
        pass

    @abstractmethod
    def list_dir(self, path: str, max_depth=2) -> list[str]:
        """
        列出目录内容。

        为什么限制 max_depth 默认 2？
        - 防止 AI 执行 ls -R 引发大量输出
        - 也防止误入极深目录树
        """
        pass
```

### 2.3 SandboxProvider：生命周期管理

```python
# 文件：backend/packages/harness/deerflow/sandbox/sandbox_provider.py

class SandboxProvider(ABC):
    """
    SandboxProvider 管理 Sandbox 实例的生命周期。

    三个核心方法对应三个生命周期阶段：
    - acquire(): 获取一个新的 sandbox 实例
    - get(): 根据 ID 获取已存在的实例
    - release(): 销毁一个 sandbox 实例

    为什么这样设计？
    - acquire/release 配对使用，确保资源不泄漏
    - get 允许在不解锁的情况下查看 sandbox 状态
    - 所有 sandbox 共用一个 provider，方便统一配置
    """

    @abstractmethod
    def acquire(self, thread_id: str | None = None) -> str:
        """
        获取一个新的 sandbox。

        参数 thread_id 的作用：
        - 将 sandbox 与特定对话线程绑定
        - 保证线程间的隔离性
        - 同一 thread_id 多次调用返回同一个 sandbox
        """
        pass

    @abstractmethod
    def release(self, sandbox_id: str) -> None:
        """
        释放 sandbox 实例。

        为什么要显式释放？
        - sandbox 可能占用文件句柄、内存等资源
        - 不及时释放会导致资源泄漏
        - release 让资源清理变得可预测
        """
        pass


# 全局单例：整个进程只存在一个 provider 实例
_default_sandbox_provider: SandboxProvider | None = None


def get_sandbox_provider(**kwargs) -> SandboxProvider:
    """
    获取全局唯一的 SandboxProvider 实例。

    这是一个经典的"单例模式"实现：
    - 首次调用时根据配置创建实例
    - 后续调用直接返回缓存的实例
    - 不需要每次都解析配置、创建对象

    为什么用函数而不是类静态方法？
    - 延迟初始化：直到真正需要时才创建
    - 更灵活：可以在创建前修改配置
    """
    global _default_sandbox_provider
    if _default_sandbox_provider is None:
        config = get_app_config()
        # resolve_class 是配置驱动的动态类加载
        # config.sandbox.use 指向具体的 Provider 类路径
        cls = resolve_class(config.sandbox.use, SandboxProvider)
        _default_sandbox_provider = cls(**kwargs)
    return _default_sandbox_provider


def shutdown_sandbox_provider() -> None:
    """
    关闭并重置 provider。

    与 reset_sandbox_provider() 的区别：
    - reset: 只清除缓存，不断开连接
    - shutdown: 先调用 provider.shutdown()，再做清除

    应用场景：
    - 应用退出时调用 shutdown，确保资源释放
    - 测试时切换配置调用 reset
    """
    global _default_sandbox_provider
    if _default_sandbox_provider is not None:
        if hasattr(_default_sandbox_provider, "shutdown"):
            _default_sandbox_provider.shutdown()
        _default_sandbox_provider = None
```

### 2.4 虚拟路径：安全隔离的核心

```python
# 文件：backend/packages/harness/deerflow/sandbox/local/local_sandbox.py

class LocalSandbox(Sandbox):
    """
    LocalSandbox 是 Sandbox 的本地文件系统实现。

    核心概念：虚拟路径（Virtual Path）

    AI 看到的路径是虚拟的：
        /mnt/user-data/workspace/test.py

    实际对应物理路径：
        backend/.deer-flow/threads/{thread_id}/user-data/workspace/test.py

    为什么这样设计？
    1. 隔离性：不同线程操作不同目录，互不干扰
    2. 安全性：AI 不知道真实路径，无法直接访问其他线程数据
    3. 清理简单：线程结束时，删除整个 thread_id 目录即可
    """

    def __init__(self, id: str, path_mappings: list[PathMapping] | None = None):
        super().__init__(id)
        self.path_mappings = path_mappings or []
        # path_mappings 定义了虚拟路径到物理路径的映射规则

    def _resolve_path(self, virtual_path: str) -> str:
        """
        将虚拟路径翻译为物理路径。

        翻译过程：
        1. 遍历 path_mappings（按容器路径长度倒序，找最匹配的）
        2. 如果虚拟路径匹配某个映射的 container_path
        3. 就替换为对应 local_path

        例子：
        path_mappings = [
            PathMapping("/mnt/skills", "/home/user/deer-flow/skills"),
            PathMapping("/mnt/user-data", "/home/user/deer-flow/threads/123/user-data"),
        ]

        "/mnt/user-data/workspace/test.py"
            → "/home/user/deer-flow/threads/123/user-data/workspace/test.py"

        "/mnt/skills/research/SKILL.md"
            → "/home/user/deer-flow/skills/research/SKILL.md"
        """
        # 实现细节：按 local_path 长度倒序排列，最长前缀匹配优先
        for mapping in sorted(self.path_mappings, key=lambda m: len(m.container_path), reverse=True):
            if virtual_path.startswith(mapping.container_path):
                return virtual_path.replace(mapping.container_path, mapping.local_path, 1)
        return virtual_path  # 没有匹配，返回原路径
```

### 2.5 完整的工具生态

```python
# 文件：backend/packages/harness/deerflow/tools/builtins/task_tool.py
# task() 工具是 DeerFlow Subagent 系统的核心入口

@tool("task", parse_docstring=True)
async def task_tool(
    runtime: ToolRuntime[ContextT, ThreadState],
    description: str,       # 任务简短描述（用于日志）
    prompt: str,             # 给 subagent 的详细指令
    subagent_type: str,      # subagent 类型：general-purpose 或 bash
    tool_call_id: Annotated[str, InjectedToolCallId],  # 工具调用 ID（框架注入）
    max_turns: int | None = None,  # 最大轮次限制
) -> str:
    """
    task() 是 DeerFlow 实现任务编排的核心工具。

    它允许主 Agent 将复杂任务分解给 subagent 并行执行。

    为什么需要 subagent？
    1. 并行处理：多个独立子任务同时执行
    2. 上下文隔离：子任务不会污染主对话的上下文
    3. 专业分工：不同 subagent 可以有不同的工具集

    使用示例：
    task(
        description="研究竞品信息",
        prompt="搜索竞品的最新动态...",
        subagent_type="general-purpose"
    )
    """
    available_subagent_names = get_available_subagent_names()
    config = get_subagent_config(subagent_type)

    if config is None:
        available = ", ".join(available_subagent_names)
        return f"Error: Unknown subagent type '{subagent_type}'. Available: {available}"

    # 构建发送给 subagent 的配置覆盖
    overrides: dict = {}

    # 注入当前启用的 skills（让 subagent 也能使用 skills）
    skills_section = get_skills_prompt_section()
    if skills_section:
        overrides["system_prompt"] = config.system_prompt + "\n\n" + skills_section

    if max_turns is not None:
        overrides["max_turns"] = max_turns

    # 提交到 executor 后台执行
    task_id = await submit_background_task(
        config=config,
        prompt=prompt,
        description=description,
        overrides=overrides,
        runtime=runtime,
        trace_id=trace_id,
    )

    # 注意：这里返回的是 task_id，不是结果
    # 结果通过轮询机制异步获取（见下方）
```

---

## 3. 上下文工程：对抗信息污染

### 3.1 问题本质

200K 的 context window，实际可用可能只有 160K。信息越多，推理越差。DeerFlow 投入大量工程努力在**精准控制上下文内容**上。

### 3.2 Skills 懒加载

```python
# 文件：backend/packages/harness/deerflow/agents/lead_agent/prompt.py

# DeerFlow 的 Skills 系统使用懒加载 + 版本化缓存

_enabled_skills_cache: list[Skill] | None = None
_enabled_skills_lock = threading.Lock()
_enabled_skills_refresh_version = 0  # 版本号，递增 = 失效信号

def _invalidate_enabled_skills_cache() -> threading.Event:
    """
    失效 Skills 缓存，触发异步重新加载。

    为什么用版本号而不是直接清除？
    - 如果直接清除，旧 worker 可能刚完成加载就发现缓存被清空
    - 版本号让每个 worker 知道自己是否"过时"了
    - 逻辑：
      1. 递增版本号
      2. 旧 worker 加载完后检查版本号
      3. 如果版本号变了，说明"我已经过时了"，继续循环加载
    """
    global _enabled_skills_cache, _enabled_skills_refresh_version
    _get_cached_skills_prompt_section.cache_clear()  # 清除 LRU 缓存
    with _enabled_skills_lock:
        _enabled_skills_cache = None
        _enabled_skills_refresh_version += 1  # 版本号递增 = 失效
        _enabled_skills_refresh_event.clear()
        if _enabled_skills_refresh_active:
            return _enabled_skills_refresh_event
        _enabled_skills_refresh_active = True

    _start_enabled_skills_refresh_thread()  # 启动后台线程加载
    return _enabled_skills_refresh_event
```

### 3.3 Subagent 隔离上下文

```python
# 文件：backend/packages/harness/deerflow/subagents/executor.py

# Subagent 在独立线程中执行，拥有完全隔离的上下文

_scheduler_pool = ThreadPoolExecutor(max_workers=3, thread_name_prefix="subagent-scheduler-")
_execution_pool = ThreadPoolExecutor(max_workers=3, thread_name_prefix="subagent-exec-")

class SubagentExecutor:
    """
    Subagent 执行器，管理后台任务的生命周期。

    关键设计：两层线程池
    - _scheduler_pool（3 workers）：负责任务提交、取消、超时检查
    - _execution_pool（3 workers）：实际运行 agent 代码

    为什么分开？
    - scheduler 线程需要快速响应（提交新任务、取消任务）
    - execution 线程可能长时间阻塞（LLM 调用）
    - 分离避免两种不同性质的任务互相影响
    """

    async def submit(self, config: SubagentConfig, prompt: str, ...) -> str:
        """
        提交一个 subagent 任务。

        返回值是 task_id，不是结果。
        结果通过 get_background_task_result(task_id) 异步获取。

        这样做的好处：
        - 主 agent 可以立即继续自己的对话
        - 不需要等待 subagent 完成
        - 通过轮询主动拉取结果
        """
        task_id = str(uuid.uuid4())
        # 提交到调度线程池
        future = _scheduler_pool.submit(self._schedule_and_execute, task_id, ...)
        _background_tasks[task_id] = SubagentResult(task_id=task_id, status=SubagentStatus.PENDING)
        return task_id


def get_background_task_result(task_id: str, timeout: float = 5.0) -> SubagentResult | None:
    """
    轮询获取 subagent 结果。

    主 agent 每 5 秒调用一次此函数检查 subagent 是否完成。

    为什么用轮询而不是回调？
    - LangGraph 的执行模型基于轮询
    - 回调会打破执行流的确定性
    - 轮询更简单，也更容易调试
    """
    result = _background_tasks.get(task_id)
    if result is None:
        return None

    if result.status == SubagentStatus.PENDING:
        # 仍在运行，继续等待
        return None

    return result  # COMPLETED / FAILED / TIMED_OUT / CANCELLED
```

---

## 4. 反馈验证：闭合任务回路

### 4.1 Loop Detection：防止 AI 鬼打墙

```python
# 文件：backend/packages/harness/deerflow/agents/middlewares/loop_detection_middleware.py

class LoopDetectionMiddleware(AgentMiddleware[AgentState]):
    """
    检测并打破重复工具调用循环。

    问题场景：
    - AI 读文件 → 发现不对 → 再读 → 再读 → ...
    - AI 写文件 → 格式不对 → 再写 → 再写 → ...

    检测策略：
    1. 每次模型响应后，对 tool_calls 做哈希
    2. 哈希 = 工具名 + 关键参数的稳定摘要
    3. 滑动窗口跟踪最近 20 次哈希
    4. 如果同一哈希出现 >= 3 次 → 注入警告
    5. 如果同一哈希出现 >= 5 次 → 强制剥离所有 tool_calls

    为什么要用哈希而不是直接比较？
    - 参数可能有细微差异（空格、换行）
    - 哈希产生稳定的一致性判断
    - MD5 前 12 位足够区分又不会太慢
    """

    DEFAULT_WARN_THRESHOLD = 3  # 警告阈值
    DEFAULT_HARD_LIMIT = 5     # 强制停止阈值

    def _stable_tool_key(self, name: str, args: dict, fallback_key: str | None) -> str:
        """
        生成稳定的工具调用标识键。

        不同工具关注不同参数：
        - read_file: 关注 path，按行号范围分桶（200 行一个桶）
        - write_file / str_replace: 关注完整内容（可能每次都不同）
        - 其他工具: 关注 path / url / query 等关键字段
        """
        if name == "read_file":
            path = args.get("path") or ""
            start = args.get("start_line")
            end = args.get("end_line")
            # 按 200 行分桶，避免 199 行和 200 行被视为不同
            bucket = (int(start or 1) - 1) // 200
            return f"{path}:{bucket}"

        if name in {"write_file", "str_replace"}:
            # 内容敏感工具，用完整参数做 key
            return json.dumps(args, sort_keys=True, default=str)

        # 其他工具，提取关键字段
        salient = {k: args[k] for k in ("path", "url", "query", "command") if k in args}
        return json.dumps(salient, sort_keys=True, default=str)


_WARNING_MSG = "[LOOP DETECTED] You are repeating the same tool calls. Stop calling tools and produce your final answer now."

_HARD_STOP_MSG = "[FORCED STOP] Repeated tool calls exceeded the safety limit."
```

### 4.2 Sandbox 命令审计

```python
# 文件：backend/packages/harness/deerflow/agents/middlewares/sandbox_audit_middleware.py

# 高风险命令模式：这些命令会被直接阻止

_HIGH_RISK_PATTERNS: list[re.Pattern[str]] = [
    # rm -rf / 或 rm -rf ~ 或 rm -rf /home
    re.compile(r"rm\s+-[^\s]*r[^\s]*\s+(/\*?|~/?\*?|/home\b|/root\b)\s*$"),

    # dd 命令（可直写磁盘）
    re.compile(r"dd\s+if="),

    # 格式化磁盘
    re.compile(r"mkfs"),

    # 读取 shadow 密码文件
    re.compile(r"cat\s+/etc/shadow"),

    # 覆盖系统文件
    re.compile(r">+\s*/etc/"),

    # 管道到 shell（最常见的远程代码执行手段）
    re.compile(r"\|\s*(ba)?sh\b"),

    # 命令替换执行（curl | bash 的变体）
    re.compile(r"[`$]\(?\s*(curl|wget|bash|sh|python|perl|base64)"),

    # base64 解码后管道执行
    re.compile(r"base64\s+.*-d.*\|"),

    # 覆盖系统二进制
    re.compile(r">+\s*(/usr/bin/|/bin/|/sbin/)"),

    # 覆盖 shell 启动文件
    re.compile(r">+\s*~/?\.(bashrc|profile|zshrc|bash_profile)"),

    # 读取进程环境变量（可能含敏感信息）
    re.compile(r"/proc/[^/]+/environ"),

    # LD_PRELOAD 劫持（动态链接库注入）
    re.compile(r"\b(LD_PRELOAD|LD_LIBRARY_PATH)\s*="),

    # /dev/tcp 发起网络连接（绕过工具的白名单）
    re.compile(r"/dev/tcp/"),

    #  Fork 炸弹
    re.compile(r":\(\)\s*\{[^}]*\|\s*\S+\s*&"),  # :(){ :|:& };:
]

_MEDIUM_RISK_PATTERNS: list[re.Pattern[str]] = [
    # chmod 777
    re.compile(r"chmod\s+777"),

    # pip / apt 安装（可能引入恶意包）
    re.compile(r"pip3?\s+install"),
    re.compile(r"apt(-get)?\s+install"),

    # sudo / su（在 Docker 中以 root 运行，无实际威胁，但说明意图）
    re.compile(r"\b(sudo|su)\b"),

    # PATH 修改（长攻击链，先警告）
    re.compile(r"\bPATH\s*="),
]
```

### 4.3 Subagent 并发硬限制

```python
# 文件：backend/packages/harness/deerflow/agents/middlewares/subagent_limit_middleware.py

class SubagentLimitMiddleware(AgentMiddleware[AgentState]):
    """
    强制执行 subagent 并发上限。

    问题场景：
    - AI 在一次响应中发起 10 个 task() 调用
    - 系统只能处理 3 个并行
    - 多余的 7 个怎么处理？

    设计决策：
    - 超出上限的 task() 调用被静默丢弃
    - AI 不会收到任何错误提示
    - 下一轮 AI 可以继续发起剩余的 task()

    为什么静默丢弃而不是报错？
    - 工具调用的错误处理很复杂
    - AI 收到错误后可能做出不可预测的行为
    - 静默丢弃更简单，也更安全
    """

    def after_model(self, model_output: AIMessage, state: AgentState) -> Command | None:
        tool_calls = model_output.tool_calls or []
        task_calls = [tc for tc in tool_calls if tc["name"] == "task"]

        if len(task_calls) <= self.max_concurrent:
            return None  # 没超标，不干预

        # 截断多余的 task 调用
        kept = [tc for tc in tool_calls if tc["name"] != "task"]
        kept.extend(task_calls[:self.max_concurrent])  # 只保留前 N 个

        # 构建更新命令：保留其他工具调用，只截断 task
        return Command(
            goto=...,
            update={"messages": [AIMessage(tool_calls=kept)]}
        )
```

---

## 5. 架构约束：防止能力漂移

### 5.1 Thread 隔离

```python
# 文件：backend/packages/harness/deerflow/agents/middlewares/thread_data_middleware.py

class ThreadDataMiddleware(AgentMiddleware[AgentState]):
    """
    为每个对话线程创建独立的文件系统目录。

    每个 thread_id 对应一组独立目录：
    /mnt/user-data/
    ├── workspace/   # AI 的工作目录，可以随意读写
    ├── uploads/     # 用户上传的文件
    └── outputs/     # AI 产生的最终产物（可通过 present_files 展示）

    为什么分离？
    - workspace: AI 可以自由操作，不担心影响 uploads/outputs
    - uploads: 用户文件，AI 不应随意修改
    - outputs: 最终产物，需要与 workspace 分离方便清理

    线程结束时的清理：
    - 删除整个 thread_id 目录即可
    - 不会出现漏删、残留
    """

    async def aafter_model(self, output: AIMessage, state: AgentState) -> dict | None:
        thread_id = state.get("configurable", {}).get("thread_id")
        if not thread_id:
            return None

        # 确保目录存在（os.makedirs 幂等）
        os.makedirs(self.base_dir / thread_id / "user-data" / "workspace", exist_ok=True)
        os.makedirs(self.base_dir / thread_id / "user-data" / "uploads", exist_ok=True)
        os.makedirs(self.base_dir / thread_id / "user-data" / "outputs", exist_ok=True)

        # 将路径信息注入 ThreadState
        return {
            "thread_data": {
                "workspace_path": f"/mnt/user-data/workspace",
                "uploads_path": f"/mnt/user-data/uploads",
                "outputs_path": f"/mnt/user-data/outputs",
            }
        }
```

### 5.2 配置驱动的模块加载

```python
# 文件：backend/packages/harness/deerflow/reflection/resolve.py

def resolve_class(path: str, base_class: type[_T]) -> type[_T]:
    """
    动态导入模块并验证继承关系。

    这是 DeerFlow 实现"可插拔架构"的核心机制。

    用法示例：
    config.yaml 中写：
        sandbox:
          use: deerflow.community.aio_sandbox:AioSandboxProvider

    代码中调用：
        cls = resolve_class(config.sandbox.use, SandboxProvider)
        provider = cls()

    这样做的好处：
    1. 切换实现不需要改代码，只改配置
    2. 第三方可以提供自己的实现
    3. 测试时可以注入 mock 对象
    """
    module_path, class_name = path.rsplit(":", 1)
    module = importlib.import_module(module_path)
    cls = getattr(module, class_name)

    # 运行时验证：确保加载的类确实是 base_class 的子类
    if not issubclass(cls, base_class):
        raise TypeError(
            f"{cls.__name__} in {module_path} must be subclass of {base_class.__name__}"
        )
    return cls
```

---

## 6. 编排协调：跨会话接力

### 6.1 Subagent 并行分解

```python
# 文件：backend/packages/harness/deerflow/agents/lead_agent/prompt.py

def _build_subagent_section(max_concurrent: int) -> str:
    """
    生成 subagent 系统的 prompt 章节。

    这个章节告诉 AI：
    1. 什么时候应该用 subagent（复杂多步骤任务）
    2. 什么时候不该用（简单直接操作）
    3. 怎么分解任务（分批次并行）
    4. 硬性限制（每轮最多 N 个 task 调用）

    为什么要在 prompt 里详细说明？
    - AI 需要理解任务分解策略
    - 直接告诉它"每轮最多 3 个"比让它自己摸索更高效
    - 示例比规则更容易理解
    """
    return f"""<subagent_system>
**🚀 SUBAGENT MODE ACTIVE - DECOMPOSE, DELEGATE, SYNTHESIZE**

**⛔ HARD CONCURRENCY LIMIT: MAXIMUM {n} `task` CALLS PER RESPONSE.**

**✅ DECOMPOSE + PARALLEL EXECUTION (Preferred Approach):**
- Complex research: 多信息源并行获取
- Multi-aspect analysis: 不同维度同时探索
- Large codebases: 不同部分同时分析

**❌ DO NOT use subagents (execute directly):**
- Task cannot be decomposed into 2+ parallel sub-tasks
- Ultra-simple actions: 读一个文件、单个命令
- Need immediate clarification

**Example - Multi-batch (> {n} sub-tasks):**
# User asks: "Compare 5 cloud providers"
# Thinking: 5 sub-tasks → need 2 batches

# Turn 1: Launch first {n}
task(...)
task(...)
task(...)

# Turn 2: Launch remaining (after first batch completes)
task(...)
task(...)

# Final: Synthesize all results
"""
```

### 6.2 Checkpointer：会话恢复

```python
# 文件：backend/packages/harness/deerflow/agents/checkpointer/

# LangGraph 的 checkpointer 机制：
# - 每个 turn 后自动保存 ThreadState 到磁盘
# - 同一 thread_id 的下次请求从断点恢复
# - 用户不会因为网络断开而丢失进度

# DeerFlow 支持多种 checkpointer 实现：
# - MemoryCheckpointer（内存，重启丢失）
# - SqliteCheckpointer（SQLite，持久化）
```

---

## 7. 核心洞察：判断力才是工程师的价值

文章最关键的一句话：

> 工具层的配置很快就能学会，真正拉开差距的是**判断力**。

DeerFlow 的源码印证了这一点。**实现这些机制并不难**，难的是：

### 7.1 定义"做好"的标准

`task_tool` 的 docstring 花了 50+ 行告诉 AI 什么叫"做完"：
- 不是返回了结果就叫做完
- 有未解决的错误不算完
- 遇到 blocker 要描述清楚而不是假装没事

这些规则需要工程师对任务目标的**深刻理解**。

### 7.2 定义"架构约束"的红线

`rm -rf /` 必须拦截，但 `rm -rf /tmp/*` 可以放过。这个判断来自：
- 对系统行为的理解
- 对业务场景的了解
- 对风险的评估

**不是算法能替代的**。

### 7.3 定义"上下文边界"

200K tokens 的窗口：
- 多少留给历史对话？
- 多少留给 skills？
- 多少留给 memory？

DeerFlow 的答案（可配置）：
- Skills：按需加载，不预设上限
- Memory：最多注入 2000 tokens
- 历史：通过 summarization 动态调整

这个比例是**根据实际场景调出来的**，需要持续观察和迭代。

---

## 8. 总结

| Harness 维度 | DeerFlow 核心机制 | 关键代码 |
|-------------|-----------------|---------|
| **持久状态** | ThreadState + memory.json | `Annotated[T, merge_func]` + LLM 提取 + 防抖队列 |
| **执行环境** | Sandbox Provider + 虚拟路径 | `execute_command` / `read_file` / `write_file` + 路径映射 |
| **上下文工程** | Skills 懒加载 + Subagent 隔离 | 版本化缓存 + 版本号递增失效 |
| **反馈验证** | Loop Detection + Sandbox Audit | 滑动窗口哈希 + 高风险命令正则 |
| **架构约束** | Thread 隔离 + 并发限制 | 独立 workspace + `SubagentLimitMiddleware` |
| **编排协调** | 后台线程池 + 轮询 | `_scheduler_pool` + `get_background_task_result()` |
