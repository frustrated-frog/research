# pyproject.toml 中文详解

本文解释本项目根目录下的 [pyproject.toml](/Users/fenggongye1/Downloads/pythonProject/jdt-hermes-agent/pyproject.toml:1)。

你可以先把它理解成 Python 项目的“项目说明书 + 依赖清单 + 打包配置 + 测试配置”。安装工具、打包工具、测试工具都会读取它。

---

## 1. pyproject.toml 是什么

`pyproject.toml` 是现代 Python 项目的标准配置文件。它通常负责这些事情：

- 告诉 Python 打包工具：这个项目怎么构建。
- 告诉安装工具：项目叫什么、版本是多少、需要哪些依赖。
- 告诉命令行：安装后应该生成哪些可执行命令。
- 告诉测试/格式化/类型检查工具：默认配置是什么。

这个项目的 `pyproject.toml` 主要包含 6 类内容：

| 配置块 | 作用 |
|---|---|
| `[build-system]` | 构建系统配置，说明用什么工具打包 |
| `[project]` | 项目基础信息和基础依赖 |
| `[project.optional-dependencies]` | 可选依赖组，比如 dev、messaging、voice、all |
| `[project.scripts]` | 安装后生成的命令行入口 |
| `[tool.setuptools...]` | setuptools 打包范围配置 |
| `[tool.pytest.ini_options]` | pytest 测试默认配置 |

---

## 2. TOML 基础语法

`pyproject.toml` 使用 TOML 格式。TOML 比 JSON/YAML 更适合写配置，语法比较固定。

### 2.1 注释

`#` 后面的内容是注释，不会被程序执行：

```toml
# Core — pinned to known-good ranges to limit supply chain attack surface
"openai>=2.21.0,<3",
```

这行注释是在解释：核心依赖做了版本范围限制，目的是降低供应链攻击风险。

### 2.2 字符串

字符串用双引号：

```toml
name = "hermes-agent"
version = "0.10.0"
```

含义是：

- `name` 的值是 `"hermes-agent"`
- `version` 的值是 `"0.10.0"`

### 2.3 表

方括号表示一个配置块，也叫 table：

```toml
[project]
name = "hermes-agent"
```

意思是：下面这些配置都属于 `project` 这个大类，直到遇到下一个 `[xxx]`。

### 2.4 嵌套表

点号表示嵌套：

```toml
[project.optional-dependencies]
dev = ["pytest>=9.0.2,<10"]
```

意思是：

- `project` 是大类
- `optional-dependencies` 是 `project` 下面的子类
- `dev` 是一个可选依赖组

### 2.5 数组

方括号也可以表示数组：

```toml
dependencies = [
  "openai>=2.21.0,<3",
  "anthropic>=0.39.0,<1",
]
```

意思是 `dependencies` 是一个列表，里面每一项都是一个依赖。

### 2.6 单行对象

花括号表示一个短对象：

```toml
authors = [{ name = "Nous Research" }]
license = { text = "MIT" }
```

意思是：

- `authors` 是作者列表，当前只有一个作者对象，名字是 `Nous Research`
- `license` 是许可证对象，文本是 `MIT`

### 2.7 依赖版本语法

依赖通常写成这种形式：

```toml
"openai>=2.21.0,<3"
```

意思是：

- 安装 `openai`
- 版本必须大于等于 `2.21.0`
- 版本必须小于 `3`

常见符号：

| 写法 | 含义 |
|---|---|
| `>=2.21.0` | 最低版本是 2.21.0 |
| `<3` | 不能升级到 3.x |
| `>=1.0,<2` | 允许 1.x，但不允许 2.x |
| `[socks]` | 安装这个包的额外功能，比如 `httpx[socks]` |
| `; sys_platform == 'linux'` | 环境条件，只有符合条件才安装 |
| `pkg @ git+https://...` | 从 Git 仓库安装指定依赖 |

---

## 3. `[build-system]`：构建系统

对应内容：

```toml
[build-system]
requires = ["setuptools>=61.0"]
build-backend = "setuptools.build_meta"
```

含义：

- 这个项目使用 `setuptools` 来构建和打包。
- 构建前需要安装 `setuptools>=61.0`。
- `build-backend = "setuptools.build_meta"` 表示构建后端使用 setuptools 的标准后端。

这部分通常不用你二开时频繁修改。只有当你要换打包工具，比如从 `setuptools` 换成 `hatchling`、`poetry-core`，才需要动它。

---

## 4. `[project]`：项目基础信息

对应内容从：

```toml
[project]
name = "hermes-agent"
version = "0.10.0"
description = "..."
readme = "README.md"
requires-python = ">=3.11"
authors = [{ name = "Nous Research" }]
license = { text = "MIT" }
dependencies = [
  ...
]
```

### 4.1 项目元信息

| 字段 | 含义 |
|---|---|
| `name = "hermes-agent"` | 包名，安装时识别为 `hermes-agent` |
| `version = "0.10.0"` | 当前项目版本 |
| `description = "..."` | 项目简介 |
| `readme = "README.md"` | PyPI 或打包展示时读取的说明文件 |
| `requires-python = ">=3.11"` | Python 版本必须 >= 3.11 |
| `authors = [{ name = "Nous Research" }]` | 作者信息 |
| `license = { text = "MIT" }` | 许可证是 MIT |

对二开来说，最常关注的是：

- 如果公司内部要改包名，可能会改 `name`。
- 如果发布内部版本，可能会改 `version`。
- 如果 Python 版本策略变化，可能会改 `requires-python`。

### 4.2 `dependencies`：基础依赖

`dependencies` 是“安装这个项目就必须安装”的依赖。也就是说，不管你是不是用 Telegram、语音、RL，只要安装 Hermes 的基础包，这些依赖都会被安装。

当前基础依赖大致分为几类。

### 4.3 大模型 SDK 和 HTTP 能力

```toml
"openai>=2.21.0,<3",
"anthropic>=0.39.0,<1",
"httpx[socks]>=0.28.1,<1",
"requests>=2.33.0,<3",
```

作用：

- `openai`：OpenAI 兼容接口调用。很多第三方 provider 也复用 OpenAI Chat Completions 格式。
- `anthropic`：Anthropic Claude 原生接口调用。
- `httpx[socks]`：HTTP 客户端，并带 socks 代理支持。
- `requests`：常见 HTTP 请求库。

你要把大模型调用改成公司内部大模型时，最相关的是这里的 `openai` 依赖，以及代码里对 OpenAI-compatible API 的封装。

### 4.4 配置、CLI 和重试能力

```toml
"python-dotenv>=1.2.1,<2",
"fire>=0.7.1,<1",
"rich>=14.3.3,<15",
"tenacity>=9.1.4,<10",
"pyyaml>=6.0.2,<7",
"prompt_toolkit>=3.0.52,<4",
```

作用：

- `python-dotenv`：读取 `.env` 文件里的环境变量，比如 API key。
- `fire`：把 Python 函数快速暴露成命令行命令。
- `rich`：终端彩色输出、表格、面板。
- `tenacity`：重试库。
- `pyyaml`：读取 YAML 配置，比如 `~/.hermes/config.yaml`。
- `prompt_toolkit`：交互式 CLI 输入、补全、多行编辑。

### 4.5 数据结构和模板

```toml
"jinja2>=3.1.5,<4",
"pydantic>=2.12.5,<3",
```

作用：

- `jinja2`：模板渲染，常用于 prompt、配置文本或代码生成。
- `pydantic`：数据校验和结构化模型。

### 4.6 工具类依赖

```toml
"exa-py>=2.9.0,<3",
"firecrawl-py>=4.16.0,<5",
"parallel-web>=0.4.2,<1",
"fal-client>=0.13.1,<1",
```

作用：

- `exa-py`：Exa 搜索能力。
- `firecrawl-py`：网页抓取/抽取。
- `parallel-web`：网页搜索/联网工具。
- `fal-client`：调用 fal.ai 一类的生成式服务。

这些通常对应 `tools/` 目录里的联网、搜索、生成工具。

### 4.7 语音与 Skills Hub

```toml
"edge-tts>=7.2.7,<8",
"PyJWT[crypto]>=2.12.0,<3",
```

作用：

- `edge-tts`：微软 Edge TTS，免费文本转语音，不需要 API key。
- `PyJWT[crypto]`：JWT 签名能力，注释里说明主要用于 GitHub App 身份认证。

---

## 5. `[project.optional-dependencies]`：可选依赖组

这一块是整个文件里最长的部分。它定义了很多“可选能力包”。

基础安装只装 `[project].dependencies`。如果你想额外安装某些功能，可以装 extra：

```bash
uv pip install -e ".[dev]"
uv pip install -e ".[messaging]"
uv pip install -e ".[all]"
```

语法里的 `.[dev]` 表示：

- `.`：安装当前目录这个项目
- `[dev]`：同时安装 `dev` 这组可选依赖
- `-e`：editable install，开发模式安装，改源码后不用重新安装

### 5.1 环境/后端相关

```toml
modal = ["modal>=1.0.0,<2"]
daytona = ["daytona>=0.148.0,<1"]
pty = [
  "ptyprocess>=0.7.0,<1; sys_platform != 'win32'",
  "pywinpty>=2.0.0,<3; sys_platform == 'win32'",
]
```

含义：

- `modal`：Modal serverless 后端。
- `daytona`：Daytona 开发环境/远程环境。
- `pty`：伪终端支持。非 Windows 用 `ptyprocess`，Windows 用 `pywinpty`。

### 5.2 开发测试相关

```toml
dev = ["debugpy>=1.8.0,<2", "pytest>=9.0.2,<10", "pytest-asyncio>=1.3.0,<2", "pytest-xdist>=3.0,<4", "mcp>=1.2.0,<2"]
```

含义：

- `debugpy`：Python 调试支持。
- `pytest`：测试框架。
- `pytest-asyncio`：测试 async/await 代码。
- `pytest-xdist`：并行跑测试。
- `mcp`：MCP 相关开发/测试依赖。

二开时如果要跑测试，通常需要安装 `dev`。

### 5.3 消息平台相关

```toml
messaging = ["python-telegram-bot[webhooks]>=22.6,<23", "discord.py[voice]>=2.7.1,<3", "aiohttp>=3.13.3,<4", "slack-bolt>=1.18.0,<2", "slack-sdk>=3.27.0,<4", "qrcode>=7.0,<8"]
slack = ["slack-bolt>=1.18.0,<2", "slack-sdk>=3.27.0,<4"]
matrix = ["mautrix[encryption]>=0.20,<1", "Markdown>=3.6,<4", "aiosqlite>=0.20", "asyncpg>=0.29"]
dingtalk = ["dingtalk-stream>=0.20,<1", "alibabacloud-dingtalk>=2.0.0", "qrcode>=7.0,<8"]
feishu = ["lark-oapi>=1.5.3,<2", "qrcode>=7.0,<8"]
```

含义：

- `messaging`：常用消息平台总包，包含 Telegram、Discord、Slack 等。
- `slack`：只装 Slack 相关依赖。
- `matrix`：Matrix 平台相关依赖。
- `dingtalk`：钉钉相关依赖。
- `feishu`：飞书相关依赖。

这类依赖主要服务于 `gateway/` 目录。

### 5.4 定时任务、MCP、外部系统

```toml
cron = ["croniter>=6.0.0,<7"]
mcp = ["mcp>=1.2.0,<2"]
homeassistant = ["aiohttp>=3.9.0,<4"]
sms = ["aiohttp>=3.9.0,<4"]
acp = ["agent-client-protocol>=0.9.0,<1.0"]
```

含义：

- `cron`：定时任务解析。
- `mcp`：Model Context Protocol 相关能力。
- `homeassistant`：Home Assistant 平台适配。
- `sms`：短信相关平台适配。
- `acp`：Agent Client Protocol，供 VS Code / Zed / JetBrains 等客户端集成。

### 5.5 模型 provider 相关

```toml
mistral = ["mistralai>=2.3.0,<3"]
bedrock = ["boto3>=1.35.0,<2"]
```

含义：

- `mistral`：Mistral 官方 SDK。
- `bedrock`：AWS Bedrock 通过 `boto3` 调用。

### 5.6 语音相关

```toml
tts-premium = ["elevenlabs>=1.0,<2"]
voice = [
  "faster-whisper>=1.0.0,<2",
  "sounddevice>=0.4.6,<1",
  "numpy>=1.24.0,<3",
]
```

含义：

- `tts-premium`：ElevenLabs 高级文本转语音。
- `voice`：本地语音转文字能力，包含 `faster-whisper`、录音设备支持和 `numpy`。

注释里特别说明：`voice` 没有放进基础安装，是因为它会拉取一些只提供 wheel 的传递依赖，对 Homebrew 这类源码构建环境不友好。

### 5.7 记忆/用户建模相关

```toml
honcho = ["honcho-ai>=2.0.1,<3"]
```

含义：

- `honcho`：Honcho 用户建模/记忆相关能力。

### 5.8 Termux 专用依赖

```toml
termux = [
  "python-telegram-bot[webhooks]>=22.6,<23",
  "hermes-agent[cron]",
  "hermes-agent[cli]",
  "hermes-agent[pty]",
  "hermes-agent[mcp]",
  "hermes-agent[honcho]",
  "hermes-agent[acp]",
]
```

含义：

- 这是 Android / Termux 的精选安装组合。
- 它刻意避开了 `voice`，因为 `faster-whisper -> ctranslate2` 这条依赖链在 Android 上兼容性不好。
- `hermes-agent[cron]` 这种写法表示安装本项目自己的某个 extra。

### 5.9 Web / RL / Benchmark

```toml
web = ["fastapi>=0.104.0,<1", "uvicorn[standard]>=0.24.0,<1"]
rl = [
  "atroposlib @ git+https://github.com/NousResearch/atropos.git@...",
  "tinker @ git+https://github.com/thinking-machines-lab/tinker.git@...",
  "fastapi>=0.104.0,<1",
  "uvicorn[standard]>=0.24.0,<1",
  "wandb>=0.15.0,<1",
]
yc-bench = ["yc-bench @ git+https://github.com/collinear-ai/yc-bench.git@... ; python_version >= '3.12'"]
```

含义：

- `web`：FastAPI + Uvicorn，提供 Web 服务能力。
- `rl`：强化学习/训练相关依赖，包含从 GitHub 指定 commit 安装的包。
- `yc-bench`：从 GitHub 安装 benchmark 依赖，并且只在 Python >= 3.12 时安装。

`@ git+https://...@commit` 这种写法表示依赖不是从 PyPI 下载，而是从 GitHub 的指定 commit 下载。这样能固定版本，减少“今天能跑，明天因为上游改了就不能跑”的风险。

### 5.10 `all`：几乎全部功能

```toml
all = [
  "hermes-agent[modal]",
  "hermes-agent[daytona]",
  "hermes-agent[messaging]",
  ...
]
```

`all` 是一个聚合 extra，意思是“尽量把大部分功能都装上”。

注意：

- `all` 不是基础安装。
- `all` 会安装很多依赖，安装更慢，也更容易遇到平台兼容问题。
- `matrix` 只在 Linux 上通过条件 marker 安装：`"hermes-agent[matrix]; sys_platform == 'linux'"`。

---

## 6. `[project.scripts]`：命令行入口

对应内容：

```toml
[project.scripts]
hermes = "hermes_cli.main:main"
hermes-agent = "run_agent:main"
hermes-acp = "acp_adapter.entry:main"
```

这表示安装项目后，会生成 3 个命令。

| 命令 | 实际调用 |
|---|---|
| `hermes` | 调用 `hermes_cli/main.py` 里的 `main()` |
| `hermes-agent` | 调用 `run_agent.py` 里的 `main()` |
| `hermes-acp` | 调用 `acp_adapter/entry.py` 里的 `main()` |

例如你执行：

```bash
hermes
```

本质上就是 Python 包安装系统帮你转到：

```python
hermes_cli.main:main
```

二开时如果你新增一个命令，比如 `hermes-company`，通常就在这里加一行。

---

## 7. `[tool.setuptools]`：打包哪些模块

对应内容：

```toml
[tool.setuptools]
py-modules = ["run_agent", "model_tools", "toolsets", ...]
```

`py-modules` 表示“项目根目录下的单文件 Python 模块”。

例如：

- `run_agent.py`
- `model_tools.py`
- `toolsets.py`
- `cli.py`
- `hermes_state.py`

这些文件不是放在某个 package 目录里，而是在项目根目录，所以要通过 `py-modules` 告诉 setuptools：打包时别漏掉它们。

---

## 8. `[tool.setuptools.package-data]`：包内静态资源

对应内容：

```toml
[tool.setuptools.package-data]
hermes_cli = ["web_dist/**/*"]
```

意思是：打包 `hermes_cli` 这个 Python 包时，把 `web_dist/` 下面的所有文件也带上。

`**/*` 表示递归匹配所有文件。

这通常用于前端构建产物、静态资源、模板文件等。

---

## 9. `[tool.setuptools.packages.find]`：自动发现 package

对应内容：

```toml
[tool.setuptools.packages.find]
include = ["agent", "agent.*", "tools", "tools.*", "hermes_cli", "gateway", "gateway.*", "tui_gateway", "tui_gateway.*", "cron", "acp_adapter", "plugins", "plugins.*"]
```

这表示 setuptools 自动寻找并打包这些 Python package。

规则解释：

- `"agent"`：包含 `agent` 包本身。
- `"agent.*"`：包含 `agent` 下面的子包。
- `"tools"` 和 `"tools.*"`：包含工具系统。
- `"gateway"` 和 `"gateway.*"`：包含消息平台 gateway。
- `"plugins"` 和 `"plugins.*"`：包含插件系统。

简单理解：

- 根目录单文件模块靠 `[tool.setuptools].py-modules`。
- 文件夹形式的 Python package 靠 `[tool.setuptools.packages.find].include`。

---

## 10. `[tool.pytest.ini_options]`：测试配置

对应内容：

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
markers = [
    "integration: marks tests requiring external services (API keys, Modal, etc.)",
]
addopts = "-m 'not integration' -n auto"
```

含义：

- `testpaths = ["tests"]`：默认只从 `tests/` 目录找测试。
- `markers`：定义一个测试标记 `integration`。
- `addopts`：默认传给 pytest 的参数。

`addopts = "-m 'not integration' -n auto"` 的意思是：

- `-m 'not integration'`：默认跳过 integration 测试。
- `-n auto`：使用 pytest-xdist 自动并行跑测试。

不过本项目的开发指南里强调：日常不要直接跑 `pytest`，要用：

```bash
scripts/run_tests.sh
```

原因是这个脚本会模拟 CI 环境，清理 API key、固定时区/语言环境、控制并行 worker 数，减少“本地能过、CI 不过”的问题。

---

## 11. 二开时最常改哪里

如果你只是接公司内部大模型，通常最可能关注这些地方。

### 11.1 可能需要加依赖

如果公司内部大模型提供 Python SDK，比如叫 `company-llm-sdk`，可能要加到：

```toml
dependencies = [
  ...
  "company-llm-sdk>=1.0,<2",
]
```

如果只是 OpenAI-compatible HTTP 接口，通常不一定要加新依赖，因为项目已经有：

```toml
"openai>=2.21.0,<3",
"httpx[socks]>=0.28.1,<1",
"requests>=2.33.0,<3",
```

### 11.2 可能需要加 optional extra

如果内部大模型 SDK 只在公司环境需要，可以新增一个 extra：

```toml
[project.optional-dependencies]
company-llm = ["company-llm-sdk>=1.0,<2"]
```

以后安装时用：

```bash
uv pip install -e ".[company-llm]"
```

### 11.3 可能需要加命令入口

如果二开后需要新增一个专门的命令，可以在 `[project.scripts]` 加：

```toml
hermes-company = "company_entry.main:main"
```

但一般接内部大模型不需要新增命令，更多是在 provider/model 配置和调用代码里改。

### 11.4 通常不建议随便改的地方

这些地方不熟时先别动：

- `[build-system]`：改错会导致项目无法安装/打包。
- `[tool.setuptools.packages.find]`：改错会导致某些 package 安装后丢失。
- `all` extra：改动会影响“全功能安装”的用户。
- `requires-python`：改低可能引入兼容问题，改高会影响用户环境。

---

## 12. 这份 pyproject.toml 的一句话总结

这个文件说明了：

- Hermes Agent 是一个 Python 3.11+ 项目。
- 使用 setuptools 打包。
- 默认安装 OpenAI/Anthropic、CLI、工具、TTS、配置读取等核心能力。
- 通过 optional dependencies 按需安装消息平台、语音、MCP、Bedrock、RL、Web 等扩展能力。
- 安装后会生成 `hermes`、`hermes-agent`、`hermes-acp` 三个命令。
- setuptools 会同时打包根目录单文件模块、多个 package 目录和 `hermes_cli/web_dist` 静态资源。
- pytest 默认从 `tests/` 跑测试，并跳过 integration 测试，但本项目推荐使用 `scripts/run_tests.sh`。
