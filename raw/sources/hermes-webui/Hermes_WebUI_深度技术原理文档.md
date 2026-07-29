# Hermes WebUI 深度技术原理文档

> 作者: Claude Sonnet 4.6
> 日期: 2026-04-26
> 项目: hermes-webui (基于 v0.50.164)

---

## 一、项目概览：Hermes生态系统的关键一环

### 1.1 Hermes 是什么？

Hermes 是一个**持久化自主AI代理系统**，运行在您自己的服务器上，具备以下核心特性：

1. **分层记忆系统** - 跨会话累积知识，自动学习和改进
2. **自主调度** - 后台cron任务，无需人工介入即可执行
3. **多渠道访问** - 终端、浏览器、消息应用（Telegram/Discord/Slack等）
4. **自我改进技能** - 从经验中自动生成可重用的过程知识

### 1.2 Hermes WebUI 的定位

Hermes WebUI 是 Hermes Agent 的**浏览器界面**，提供了与CLI完全等价的体验：

```
架构层次：
┌─────────────────────────────────────────┐
│   用户接入层 (User Interface)           │
│  ┌──────────┐  ┌──────────┐  ┌───────┐ │
│  │ CLI终端  │  │ WebUI    │  │ 消息App│ │
│  └──────────┘  └──────────┘  └───────┘ │
└─────────────────────────────────────────┘
              ↓ 共享层
┌─────────────────────────────────────────┐
│   Hermes Agent 核心引擎                 │
│  • 记忆系统 (MEMORY.md, USER.md)       │
│  • 技能系统 (自动生成的procedures)      │
│  • 调度系统 (cron jobs)                 │
│  • 工具集 (terminal, file, web...)     │
└─────────────────────────────────────────┘
              ↓ Provider层
┌─────────────────────────────────────────┐
│   多模型提供商支持                      │
│  OpenAI | Anthropic | Google | DeepSeek │
│  OpenRouter | Nous Portal | 本地模型... │
└─────────────────────────────────────────┘
```

**关键特性**：
- **零构建步骤** - 纯Python + 原生JS，无框架、无打包器
- **完全对等** - WebUI能做CLI能做的一切
- **自托管** - 数据完全在您控制之下
- **多Profile支持** - 独立的配置、技能、记忆空间

---

## 二、核心架构深度解析

### 2.1 技术栈选择

#### 后端：极简Python HTTP服务

```python
# server.py - 核心服务器（仅154行）
from http.server import ThreadingHTTPServer, BaseHTTPRequestHandler

class Handler(BaseHTTPRequestHandler):
    timeout = 30  # 防止线程耗尽

    def do_GET(self):
        # 路由：/, /health, /api/sessions, /api/chat/stream, /api/file...

    def do_POST(self):
        # 路由：/api/session/new, /api/chat/start, /api/upload...
```

**为什么不用Flask/FastAPI？**
- 减少依赖，降低复杂度
- stdlib足够强大，适合嵌入式场景
- 多线程模型天然支持并发请求

**线程模型**：
```
主线程 → ThreadingHTTPServer
    ├─ HTTP请求线程1 → Handler.do_GET
    ├─ HTTP请求线程2 → Handler.do_POST
    └─ Agent后台线程 → _run_agent_streaming (长时间运行)
```

#### 前端：零框架的纯JS架构

```html
static/
├── index.html    (600行 - 单页HTML模板)
├── style.css     (1050行 - 响应式CSS + 7个主题)
├── ui.js         (1740行 - DOM工具、Markdown渲染、工具卡片)
├── workspace.js  (286行 - 文件树、预览、Git检测)
├── sessions.js   (800行 - 会话管理、搜索、项目分组)
├── messages.js   (655行 - SSE流式处理、工具调用卡片)
├── panels.js     (1438行 - Cron、技能、记忆、设置面板)
├── commands.js   (267行 - 斜杠命令自动补全)
└── boot.js       (524行 - 移动端导航、语音输入、启动逻辑)
```

**设计哲学**：
- **单文件模块** - 每个JS文件独立，无构建步骤
- **即时加载** - 浏览器直接加载源码，无打包延迟
- **渐进增强** - 核心功能无JS也能工作（SEO友好）

### 2.2 核心数据流

#### 用户发送消息的完整流程

```
用户输入 → messages.js#send()
    ↓
POST /api/chat/start
    ↓
routes.py#handle_chat_start()
    ├─ 创建 Session 对象
    ├─ 保存到 ~/.hermes/webui-mvp/sessions/{id}.json
    ├─ 创建 queue.Queue() 存入 STREAMS[stream_id]
    ├─ 启动后台线程: _run_agent_streaming()
    └─ 立即返回 {stream_id}
    ↓
GET /api/chat/stream (SSE长连接)
    ↓
streaming.py#_run_agent_streaming()
    ├─ 设置环境变量: TERMINAL_CWD, HERMES_SESSION_KEY
    ├─ 初始化 AIAgent (from hermes-agent)
    │   ├─ model="anthropic/claude-sonnet-4.6"
    │   ├─ provider="anthropic"
    │   ├─ enabled_toolsets=["terminal", "file", "web"...]
    │   ├─ stream_delta_callback=on_token
    │   └─ tool_progress_callback=on_tool
    ├─ agent.run_conversation(user_message, conversation_history)
    │   ├─ [流式] on_token("每个token") → queue.put(('token', {...}))
    │   ├─ [工具调用] on_tool("terminal", "ls -la") → queue.put(('tool', {...}))
    │   └─ [完成] 返回完整消息历史
    ├─ session.save() 持久化
    └─ queue.put(('done', {session: {...}}))
    ↓
messages.js#SSE事件处理器
    ├─ 'token' → 追加到聊天气泡
    ├─ 'tool' → 渲染工具卡片
    └─ 'done' → 更新会话标题、上下文指示器
```

**关键技术点**：

1. **异步解耦** - HTTP请求立即返回，Agent在后台线程运行
2. **队列通信** - Queue实现线程间安全通信
3. **SSE流式** - 浏览器实时接收token，无轮询开销

---

## 三、与本地Hermes Agent的连接机制

### 3.1 Agent发现与路径解析

Hermes WebUI **不包含** Hermes Agent代码，而是在运行时动态发现并加载：

```python
# api/config.py - Agent目录发现逻辑
def _discover_agent_dir() -> Path:
    """多策略搜索Hermes Agent安装位置"""

    candidates = []

    # 策略1: 显式环境变量（最高优先级）
    if os.getenv("HERMES_WEBUI_AGENT_DIR"):
        candidates.append(
            Path(os.getenv("HERMES_WEBUI_AGENT_DIR")).expanduser().resolve()
        )

    # 策略2: HERMES_HOME标准位置
    hermes_home = os.getenv("HERMES_HOME", str(HOME / ".hermes"))
    candidates.append(Path(hermes_home).expanduser() / "hermes-agent")

    # 策略3: 兄弟目录（开发模式）
    candidates.append(REPO_ROOT.parent / "hermes-agent")

    # 策略4: 父目录就是Agent仓库
    if (REPO_ROOT.parent / "run_agent.py").exists():
        candidates.append(REPO_ROOT.parent)

    # 策略5: 常见安装路径
    candidates.append(HOME / ".hermes" / "hermes-agent")
    candidates.append(HOME / "hermes-agent")

    # 返回第一个包含run_agent.py的目录
    for path in candidates:
        if path.exists() and (path / "run_agent.py").exists():
            return path.resolve()

    return None
```

**找到后做什么？**

```python
# 将Agent目录注入sys.path，使模块可导入
if _AGENT_DIR is not None:
    if str(_AGENT_DIR) not in sys.path:
        sys.path.append(str(_AGENT_DIR))  # 注意：追加到末尾，避免覆盖系统包
    _HERMES_FOUND = True
```

### 3.2 Python环境发现

WebUI **复用** Hermes Agent的虚拟环境：

```python
def _discover_python(agent_dir: Path) -> str:
    """定位包含Agent依赖的Python解释器"""

    # 优先级1: 显式环境变量
    if os.getenv("HERMES_WEBUI_PYTHON"):
        return os.getenv("HERMES_WEBUI_PYTHON")

    # 优先级2: Agent的venv
    if agent_dir:
        venv_py = agent_dir / "venv" / "bin" / "python"
        if venv_py.exists():
            return str(venv_py)

    # 优先级3: 本地.venv
    local_venv = REPO_ROOT / ".venv" / "bin" / "python"
    if local_venv.exists():
        return str(local_venv)

    # 优先级4: 系统Python
    return shutil.which("python3") or "python3"
```

**为什么复用？**
- Agent依赖（openai, anthropic, httpx等）已在Agent venv中安装
- 避免重复安装，节省空间
- 确保版本一致性

### 3.3 Profile系统：隔离的Agent身份

Hermes支持**多Profile**，每个Profile拥有独立的：
- 配置文件 (`config.yaml`)
- API密钥 (`.env`)
- 记忆文件 (`MEMORY.md`, `USER.md`)
- 技能库 (`skills/`)
- Cron任务 (`cron/jobs.json`)

```python
# api/profiles.py - Profile状态管理
_ACTIVE_PROFILE = 'default'  # 进程级别活动Profile
_tls = threading.local()     # 线程本地存储（支持多客户端并发）

def get_active_hermes_home() -> Path:
    """返回当前活动Profile的HERMES_HOME路径"""

    name = get_active_profile_name()  # 从TLS或全局变量获取
    if name == 'default':
        return _DEFAULT_HERMES_HOME  # ~/.hermes
    profile_dir = _DEFAULT_HERMES_HOME / 'profiles' / name
    if profile_dir.is_dir():
        return profile_dir
    return _DEFAULT_HERMES_HOME
```

**Profile切换流程**：

```python
def switch_profile(name: str, *, process_wide: bool = True) -> dict:
    """切换活动Profile"""

    # 1. 检查Agent是否运行（防止并发冲突）
    with STREAMS_LOCK:
        if len(STREAMS) > 0:
            raise RuntimeError('Cannot switch profiles while agent is running')

    # 2. 解析Profile目录
    home = _resolve_named_profile_home(name)
    if not home.is_dir():
        raise ValueError(f"Profile '{name}' does not exist")

    # 3. 更新环境变量
    with _profile_lock:
        if process_wide:
            _active_profile = name
            os.environ['HERMES_HOME'] = str(home)

            # 4. 猴子补丁修复模块级缓存
            import tools.skills_tool as _sk
            _sk.HERMES_HOME = home
            _sk.SKILLS_DIR = home / 'skills'

            import cron.jobs as _cj
            _cj.HERMES_DIR = home
            _cj.CRON_DIR = home / 'cron'

    # 5. 重新加载.env文件
    _reload_dotenv(home)

    # 6. 重新加载config.yaml
    reload_config()

    return {'profiles': list_profiles_api(), 'active': name}
```

**为什么需要猴子补丁？**
- Hermes Agent的某些模块在import时捕获了`HERMES_HOME`
- 切换Profile后，这些缓存的路径不会自动更新
- 必须手动修补模块级变量

---

## 四、AI网关的核心功能

### 4.1 模型路由与提供商解析

Hermes WebUI支持**10+个模型提供商**，通过统一接口访问：

```python
# api/config.py - 模型ID解析
def resolve_model_provider(model_id: str) -> tuple:
    """解析模型ID，返回(模型名, 提供商, base_url)

    支持格式：
    - "claude-sonnet-4.6"           (裸名称，使用默认提供商)
    - "anthropic/claude-sonnet-4.6" (OpenRouter风格)
    - "@minimax:MiniMax-M2.7"       (显式提供商前缀)
    """

    # 1. 检查自定义提供商配置
    custom_providers = cfg.get("custom_providers", [])
    for entry in custom_providers:
        if model_id == entry.get("model"):
            return model_id, "custom:" + entry["name"], entry.get("base_url")

    # 2. 解析@provider:model格式
    if model_id.startswith("@") and ":" in model_id:
        provider_hint, bare_model = model_id[1:].split(":", 1)
        return bare_model, provider_hint, None

    # 3. 解析provider/model格式
    if "/" in model_id:
        prefix, bare = model_id.split("/", 1)

        # OpenRouter需要完整路径
        if config_provider == "openrouter":
            return model_id, "openrouter", config_base_url

        # Portal提供商（Nous, OpenCode）保持完整路径
        if config_provider in {"nous", "opencode-zen", "opencode-go"}:
            return model_id, config_provider, config_base_url

        # 跨提供商选择 → 路由到OpenRouter
        if prefix in _PROVIDER_MODELS and prefix != config_provider:
            return model_id, "openrouter", None

    return model_id, config_provider, config_base_url
```

**实际案例**：

| 用户选择 | 配置提供商 | 解析结果 | 路由路径 |
|---------|----------|---------|---------|
| `claude-sonnet-4.6` | anthropic | `(claude-sonnet-4.6, anthropic, None)` | 直连Anthropic API |
| `openai/gpt-5.4-mini` | anthropic | `(openai/gpt-5.4-mini, openrouter, None)` | 通过OpenRouter访问OpenAI |
| `@minimax:MiniMax-M2.7` | openai | `(MiniMax-M2.7, minimax, None)` | 直连MiniMax API |
| `google/gemini-3-flash` | nous | `(google/gemini-3-flash, nous, ...)` | 通过Nous Portal访问 |

### 4.2 凭证池与密钥管理

Hermes支持**多源凭证**，优先级如下：

```python
# 凭证解析优先级
def resolve_runtime_provider(requested=None):
    """
    1. 显式提供商API密钥（providers.<name>.api_key in config.yaml）
    2. 通用API密钥（OPENAI_API_KEY, ANTHROPIC_API_KEY in .env）
    3. 凭证池（credential_pool in auth.json）
    4. 环境变量（HERMES_API_KEY等）
    """

    # 示例：GitHub Copilot凭证解析
    if requested == "copilot":
        # 检查auth.json中的credential_pool
        pool = load_pool("copilot")
        if pool.entries():
            # 使用第一个有效token
            return {
                "provider": "copilot",
                "api_key": pool.entries()[0].key,
                "base_url": "https://api.githubcopilot.com"
            }

        # 检查环境变量
        if os.getenv("GITHUB_TOKEN"):
            return {
                "provider": "copilot",
                "api_key": os.getenv("GITHUB_TOKEN"),
                "base_url": "https://api.githubcopilot.com"
            }
```

**安全措施**：
1. **凭证隔离** - 每个Profile的`.env`独立加载，切换Profile时清除旧密钥
2. **秘钥掩盖** - API响应中的密钥自动redact
3. **SSRF防护** - 自定义端点禁止访问私有IP

### 4.3 流式响应处理（SSE）

**前端发起SSE连接**：

```javascript
// messages.js
function startStream(sessionId, streamId) {
    const eventSource = new EventSource(
        `/api/chat/stream?stream_id=${streamId}`,
        { withCredentials: true }
    );

    eventSource.addEventListener('token', (e) => {
        const data = JSON.parse(e.data);
        appendToChatBubble(data.text);  // 实时追加token
    });

    eventSource.addEventListener('tool', (e) => {
        const data = JSON.parse(e.data);
        renderToolCard(data.name, data.preview);  // 渲染工具卡片
    });

    eventSource.addEventListener('done', (e) => {
        const data = JSON.parse(e.data);
        updateSession(data.session);  // 更新会话状态
        eventSource.close();
    });
}
```

**后端SSE事件生成**：

```python
# api/streaming.py
def _run_agent_streaming(session_id, msg_text, model, workspace, stream_id):
    q = STREAMS[stream_id]  # 获取队列

    def on_token(text):
        if text is None: return  # 流结束信号
        q.put_nowait(('token', {'text': text}))

    def on_tool(event_type, name, preview, args):
        q.put_nowait(('tool', {
            'event_type': event_type,
            'name': name,
            'preview': preview,
            'args': args
        }))

    # 创建Agent并运行
    agent = AIAgent(
        model=model,
        stream_delta_callback=on_token,
        tool_progress_callback=on_tool
    )

    result = agent.run_conversation(user_message=msg_text)

    # 发送完成事件
    q.put_nowait(('done', {'session': session.compact()}))
```

**SSE路由处理器**：

```python
# api/routes.py
def handle_chat_stream(handler, parsed):
    stream_id = parse_qs(parsed.query).get('stream_id', [None])[0]

    handler.send_response(200)
    handler.send_header('Content-Type', 'text/event-stream')
    handler.send_header('Cache-Control', 'no-cache')
    handler.send_header('Connection', 'keep-alive')
    handler.end_headers()

    q = STREAMS[stream_id]
    while True:
        try:
            event, data = q.get(timeout=30)  # 30秒超时

            # 写入SSE事件
            payload = f"event: {event}\ndata: {json.dumps(data)}\n\n"
            handler.wfile.write(payload.encode('utf-8'))
            handler.wfile.flush()

            if event in ('done', 'error', 'cancel'):
                break

        except queue.Empty:
            # 心跳包防止超时
            handler.wfile.write(b": heartbeat\n\n")
            handler.wfile.flush()
```

### 4.4 工具调用与审批系统

**工具调用流程**：

```
Agent调用工具 → tools.terminal.execute("rm -rf /")
    ↓
tools/approval.py检测危险命令
    ↓
_pending[session_id] = {
    "command": "rm -rf /",
    "description": "递归删除根目录",
    "pattern_keys": ["rm -rf"]
}
    ↓
streaming.py#on_tool()回调
    ↓
put('approval', pending_data)  # SSE推送审批请求
    ↓
前端渲染审批卡片
    ↓
用户选择: [允许一次] [允许本次会话] [总是允许] [拒绝]
    ↓
POST /api/approval/respond
    ↓
approve_session() + approve_permanent() (如果选择"总是允许")
    ↓
Agent继续执行
```

**审批持久化**：

```python
# tools/approval.py
_permanent_approved = set()  # 永久允许的pattern

def save_permanent_allowlist():
    """保存到 ~/.hermes/approval_allowlist.json"""
    path = HERMES_HOME / "approval_allowlist.json"
    path.write_text(json.dumps(list(_permanent_approved)))
```

### 4.5 上下文压缩与会话持久化

**会话数据结构**：

```json
{
  "session_id": "a1b2c3d4e5f6",
  "title": "Hermes架构分析",
  "workspace": "/Users/user/workspace",
  "model": "anthropic/claude-sonnet-4.6",
  "messages": [
    {"role": "user", "content": "解释Hermes的工作原理"},
    {"role": "assistant", "content": "Hermes是..."},
    {"role": "tool", "tool_call_id": "call_123", "content": "{...}"}
  ],
  "tool_calls": [
    {
      "name": "terminal",
      "snippet": "执行成功",
      "args": {"command": "ls -la"}
    }
  ],
  "input_tokens": 1250,
  "output_tokens": 3200,
  "estimated_cost": 0.0234,
  "created_at": 1714147200.0,
  "updated_at": 1714150800.0,
  "profile": "work"
}
```

**上下文压缩触发**：

```python
# Agent内部逻辑（在run_agent.py中）
class ContextCompressor:
    def should_compress(self, prompt_tokens: int) -> bool:
        return prompt_tokens > self.threshold_tokens

    def compress(self, messages: list) -> list:
        """
        压缩策略：
        1. 保留system消息
        2. 保留最近N轮对话
        3. 对历史对话生成摘要
        4. 更新session_id（压缩计数+1）
        """

        # 生成摘要
        summary = self._summarize_old_messages(messages[:-10])

        # 构造新历史
        compressed = [
            {"role": "system", "content": f"历史摘要: {summary}"},
            *messages[-10:]  # 保留最近10轮
        ]

        # 通知WebUI压缩发生
        self.compression_count += 1

        return compressed
```

**WebUI处理压缩事件**：

```python
# api/streaming.py
_agent_sid = getattr(agent, 'session_id', None)
if _agent_sid and _agent_sid != session_id:
    # 检测到session_id变更（压缩导致）
    old_sid = session_id
    new_sid = _agent_sid

    # 重命名会话文件
    old_path = SESSION_DIR / f'{old_sid}.json'
    new_path = SESSION_DIR / f'{new_sid}.json'
    old_path.rename(new_path)

    # 更新内存缓存
    with LOCK:
        SESSIONS[new_sid] = SESSIONS.pop(old_sid)

    # 通知前端
    put('compressed', {'message': '上下文已自动压缩'})
```

---

## 五、安全与隔离机制

### 5.1 认证与授权

**可选密码认证**：

```python
# api/auth.py
def _hash_password(password: str) -> str:
    """PBKDF2-HMAC-SHA256, 600k次迭代"""
    import hashlib
    salt = os.urandom(16)
    key = hashlib.pbkdf2_hmac(
        'sha256',
        password.encode('utf-8'),
        salt,
        600000  # OWASP推荐
    )
    return f"pbkdf2_sha256${salt.hex()}${key.hex()}"

def check_auth(handler, parsed) -> bool:
    """检查hermes_session cookie"""

    if not is_auth_enabled():
        return True  # 未启用认证

    cookie = handler.headers.get('Cookie', '')
    match = re.search(r'hermes_session=([^;]+)', cookie)
    if not match:
        handler.send_response(302)
        handler.send_header('Location', '/login')
        return False

    token = match.group(1)
    if not verify_session_token(token):
        handler.send_response(302)
        handler.send_header('Location', '/login')
        return False

    return True
```

**登录流程**：

```
GET /login → 返回登录页面
POST /api/login
    ├─ 验证密码（从settings.json读取password_hash）
    ├─ 生成HMAC签名token（24小时有效期）
    ├─ 设置HTTP-only cookie: hermes_session
    └─ 重定向到 /
```

### 5.2 CSRF防护

```python
# api/routes.py
def _check_csrf(handler) -> bool:
    """验证POST请求来源"""

    origin = handler.headers.get("Origin", "")
    referer = handler.headers.get("Referer", "")
    host = handler.headers.get("Host", "")

    if not origin and not referer:
        return True  # 非浏览器客户端

    # 提取origin的host:port
    m = re.match(r"^https?://([^/]+)", origin or referer)
    if not m:
        return False

    origin_host = m.group(1)

    # 检查显式允许的origin
    if origin.rstrip('/').lower() in _allowed_public_origins():
        return True

    # 检查同源（Host, X-Forwarded-Host, X-Real-Host）
    allowed_hosts = [
        host,
        handler.headers.get("X-Forwarded-Host", ""),
        handler.headers.get("X-Real-Host", "")
    ]

    for allowed in allowed_hosts:
        if origin_host == allowed.strip():
            return True

    return False
```

### 5.3 路径遍历防护

```python
# api/helpers.py
def safe_resolve(root: Path, requested: str) -> Path:
    """防止路径遍历攻击"""

    resolved = (root / requested).resolve()

    # 检查解析后的路径是否仍在root内
    try:
        resolved.relative_to(root)
    except ValueError:
        raise ValueError(f"Path traversal detected: {requested}")

    return resolved
```

**示例攻击与防御**：

```
攻击: GET /api/file?path=../../etc/passwd
防御: safe_resolve(workspace, "../../etc/passwd")
      → ValueError: Path traversal detected
```

### 5.4 SSRF防护（自定义端点）

```python
# api/config.py
def get_available_models():
    # ...

    if cfg_base_url:
        # 解析URL
        parsed_url = urlparse(endpoint_url)

        # 验证scheme
        if parsed_url.scheme not in ("", "http", "https"):
            raise ValueError(f"Invalid URL scheme: {parsed_url.scheme}")

        # DNS解析后检查IP
        if parsed_url.hostname:
            resolved_ips = socket.getaddrinfo(parsed_url.hostname, None)
            for _, _, _, _, addr in resolved_ips:
                addr_obj = ipaddress.ip_address(addr[0])

                if addr_obj.is_private or addr_obj.is_loopback:
                    # 允许已知本地提供商
                    is_known_local = any(
                        k in parsed_url.hostname.lower()
                        for k in ["ollama", "localhost", "127.0.0.1", "lmstudio"]
                    )

                    if not is_known_local:
                        raise ValueError(
                            f"SSRF: resolved hostname to private IP {addr[0]}"
                        )
```

---

## 六、性能优化与扩展性

### 6.1 会话索引优化

**问题**：`all_sessions()`每次扫描整个`sessions/`目录，O(n)文件IO

**解决方案**：维护索引文件

```python
# api/models.py
SESSION_INDEX_FILE = SESSION_DIR / "_index.json"

def _write_session_index(updates=None):
    """增量更新索引，避免全量扫描"""

    if updates is None:
        # 全量重建（启动时）
        entries = []
        for p in SESSION_DIR.glob('*.json'):
            s = Session.load(p.stem)
            if s:
                entries.append(s.compact())
        entries.sort(key=lambda s: s['updated_at'], reverse=True)
        SESSION_INDEX_FILE.write_text(json.dumps(entries))
        return

    # 增量更新（会话保存时）
    with LOCK:
        existing = json.loads(SESSION_INDEX_FILE.read_text())
        updated_map = {s.session_id: s.compact() for s in updates}

        # 就地替换
        for i, e in enumerate(existing):
            if e.get('session_id') in updated_map:
                existing[i] = updated_map[e['session_id']]

        # 追加新会话
        for sid, entry in updated_map.items():
            if sid not in {e['session_id'] for e in existing}:
                existing.append(entry)

        existing.sort(key=lambda s: s['updated_at'], reverse=True)
        SESSION_INDEX_FILE.write_text(json.dumps(existing))
```

**效果**：从O(n)降至O(1)（单会话更新）

### 6.2 LRU缓存

```python
# api/config.py
SESSIONS = collections.OrderedDict()  # LRU缓存
SESSIONS_MAX = 100  # 最多缓存100个会话

def get_session(sid):
    with LOCK:
        if sid in SESSIONS:
            SESSIONS.move_to_end(sid)  # 标记为最近使用
            return SESSIONS[sid]

    s = Session.load(sid)  # 磁盘加载
    if s:
        with LOCK:
            SESSIONS[sid] = s
            SESSIONS.move_to_end(sid)

            # 驱逐最久未使用的会话
            while len(SESSIONS) > SESSIONS_MAX:
                SESSIONS.popitem(last=False)

    return s
```

### 6.3 模型列表TTL缓存

```python
# api/config.py
_available_models_cache = None
_available_models_cache_ts = 0.0
_AVAILABLE_MODELS_CACHE_TTL = 60.0  # 60秒

def get_available_models():
    global _available_models_cache, _available_models_cache_ts

    with _available_models_cache_lock:
        now = time.monotonic()

        # 检查TTL
        if (_available_models_cache is not None and
            (now - _available_models_cache_ts) < _AVAILABLE_MODELS_CACHE_TTL):
            return copy.deepcopy(_available_models_cache)

        # 重新计算
        result = _compute_available_models()

        # 更新缓存
        _available_models_cache = result
        _available_models_cache_ts = now

        return result
```

**优化效果**：
- 避免重复检测API密钥
- 减少对`config.yaml`的读取
- 自动检测文件修改（mtime检查）

---

## 七、前端关键技术实现

### 7.1 响应式三栏布局

```css
/* style.css */
.sidebar {
    width: 280px;
    transition: transform 0.2s;
}

.main {
    flex: 1;
    min-width: 0;  /* 防止内容溢出 */
}

.rightpanel {
    width: 0;
    transition: width 0.2s;
}

.rightpanel.open {
    width: 400px;
}

/* 移动端（<640px） */
@media (max-width: 640px) {
    .sidebar {
        position: fixed;
        transform: translateX(-100%);
        z-index: 100;
    }

    .sidebar.open {
        transform: translateX(0);
    }

    .rightpanel {
        position: fixed;
        right: 0;
        top: 0;
        height: 100vh;
        z-index: 100;
    }
}
```

### 7.2 实时Markdown渲染

```javascript
// ui.js
function renderMd(text) {
    // 1. 代码块语法高亮
    text = text.replace(/```(\w+)?\n([\s\S]*?)```/g, (match, lang, code) => {
        const highlighted = Prism.highlight(
            code.trim(),
            Prism.languages[lang] || Prism.languages.plaintext,
            lang
        );
        return `<pre class="language-${lang}"><code>${highlighted}</code></pre>`;
    });

    // 2. 行内代码
    text = text.replace(/`([^`]+)`/g, '<code>$1</code>');

    // 3. Mermaid图表
    text = text.replace(/```mermaid\n([\s\S]*?)```/g, (match, code) => {
        const id = 'mermaid-' + Math.random().toString(36).substr(2, 9);
        setTimeout(() => mermaid.render(id, code), 0);
        return `<div class="mermaid" id="${id}"></div>`;
    });

    // 4. 标准Markdown（使用marked库）
    return marked.parse(text);
}
```

### 7.3 斜杠命令自动补全

```javascript
// commands.js
const COMMANDS = {
    'help': '显示帮助信息',
    'clear': '清空当前会话',
    'compress': '压缩上下文',
    'model': '切换模型',
    'workspace': '切换工作空间',
    'new': '创建新会话',
    'usage': '显示token使用量',
    'theme': '切换主题'
};

function showCommandDropdown(input) {
    const value = input.value;
    if (!value.startsWith('/')) return;

    const query = value.slice(1).toLowerCase();
    const matches = Object.entries(COMMANDS)
        .filter(([cmd]) => cmd.startsWith(query));

    if (matches.length === 0) return;

    // 渲染下拉菜单
    const dropdown = document.createElement('div');
    dropdown.className = 'command-dropdown';

    matches.forEach(([cmd, desc]) => {
        const item = document.createElement('div');
        item.className = 'command-item';
        item.innerHTML = `<strong>/${cmd}</strong> - ${desc}`;
        item.onclick = () => insertCommand(cmd);
        dropdown.appendChild(item);
    });

    // 定位到输入框下方
    const rect = input.getBoundingClientRect();
    dropdown.style.top = rect.bottom + 'px';
    dropdown.style.left = rect.left + 'px';

    document.body.appendChild(dropdown);
}
```

---

## 八、部署与运维

### 8.1 Docker多容器架构

```yaml
# docker-compose.three-container.yml
version: '3.8'

services:
  hermes-agent:
    image: nousresearch/hermes-agent:latest
    volumes:
      - hermes-home:/root/.hermes
    environment:
      - HERMES_UID=${HERMES_UID}
      - HERMES_GID=${HERMES_GID}
    ports:
      - "8642:8642"  # Gateway API

  hermes-dashboard:
    image: nousresearch/hermes-agent:latest
    command: ["hermes-dashboard"]
    volumes:
      - hermes-home:/root/.hermes
    environment:
      - HERMES_UID=${HERMES_UID}
      - HERMES_GID=${HERMES_GID}
    ports:
      - "9119:9119"  # Monitoring UI

  hermes-webui:
    image: ghcr.io/nesquena/hermes-webui:latest
    volumes:
      - hermes-home:/home/hermeswebui/.hermes
      - ./workspace:/workspace
    environment:
      - WANTED_UID=${UID}
      - WANTED_GID=${GID}
      - HERMES_WEBUI_PASSWORD=${HERMES_WEBUI_PASSWORD}
    ports:
      - "8787:8787"  # Web UI

volumes:
  hermes-home:  # 共享状态卷
```

**UID/GID匹配的重要性**：

- 三容器共享`hermes-home`卷
- 文件所有权必须一致，否则`PermissionError`
- 解决方案：所有容器以相同UID/GID运行

### 8.2 环境变量完整列表

| 变量 | 默认值 | 说明 |
|-----|-------|-----|
| `HERMES_WEBUI_HOST` | `127.0.0.1` | 绑定地址 |
| `HERMES_WEBUI_PORT` | `8787` | 端口 |
| `HERMES_WEBUI_STATE_DIR` | `~/.hermes/webui` | 状态目录 |
| `HERMES_WEBUI_DEFAULT_WORKSPACE` | `~/workspace` | 默认工作空间 |
| `HERMES_WEBUI_DEFAULT_MODEL` | (空) | 默认模型 |
| `HERMES_WEBUI_PASSWORD` | (空) | 启用密码认证 |
| `HERMES_WEBUI_AGENT_DIR` | (自动发现) | Agent目录 |
| `HERMES_WEBUI_PYTHON` | (自动发现) | Python解释器 |
| `HERMES_WEBUI_TLS_CERT` | (空) | TLS证书路径 |
| `HERMES_WEBUI_TLS_KEY` | (空) | TLS私钥路径 |
| `HERMES_HOME` | `~/.hermes` | Hermes基础目录 |
| `HERMES_CONFIG_PATH` | `~/.hermes/config.yaml` | 配置文件路径 |

### 8.3 健康检查与监控

```python
# server.py
def main():
    # ...

    # 健康检查端点
    def health_check():
        return {"status": "ok", "uptime": time.time() - SERVER_START_TIME}

    # 路由注册
    # GET /health → {"status": "ok"}

    # Docker健康检查
    # HEALTHCHECK --interval=30s --timeout=3s \
    #   CMD curl -f http://localhost:8787/health || exit 1
```

---

## 九、故障排查指南

### 9.1 常见问题

**Q1: 启动时报错"Could not find Hermes agent directory"**

解决：
```bash
# 方法1: 设置环境变量
export HERMES_WEBUI_AGENT_DIR=/path/to/hermes-agent
./start.sh

# 方法2: 克隆为兄弟目录
git clone https://github.com/NousResearch/hermes-agent.git ../hermes-agent
```

**Q2: 模型下拉列表为空或显示不正确的模型**

原因：API密钥未配置或provider配置错误

解决：
```bash
# 检查config.yaml
cat ~/.hermes/config.yaml

# 示例配置
model:
  provider: anthropic
  default: claude-sonnet-4.6

providers:
  anthropic:
    api_key: sk-ant-...
```

**Q3: Profile切换后技能/记忆不生效**

原因：模块级缓存未更新

解决：
```bash
# 重启WebUI
./start.sh
```

### 9.2 日志与调试

```bash
# 查看实时日志
tail -f /tmp/webui-mvp.log

# 启用详细日志
export HERMES_WEBUI_DEBUG=1
./start.sh

# 测试环境（隔离端口）
HERMES_WEBUI_PORT=8788 \
HERMES_WEBUI_STATE_DIR=~/.hermes/webui-test \
./start.sh
```

### 9.3 性能调优

**会话索引重建**：
```bash
# 删除损坏的索引
rm ~/.hermes/webui-mvp/sessions/_index.json

# 重启WebUI（自动重建）
./start.sh
```

**缓存清理**：
```bash
# 清空LRU缓存（重启即可）
./start.sh

# 调整缓存大小
export HERMES_WEBUI_SESSIONS_MAX=200
./start.sh
```

---

## 十、总结与展望

### 核心设计理念

1. **极简主义** - 零构建、零框架、纯标准库
2. **完全对等** - WebUI = CLI的功能体验
3. **自托管优先** - 数据在您控制之下
4. **可扩展性** - Profile系统支持多环境

### 架构优势

✅ **轻量级** - 后端154行路由，前端7个JS文件
✅ **可维护** - 清晰的模块分离，代码易读
✅ **高性能** - 异步SSE流式、LRU缓存、增量索引
✅ **安全** - CSRF防护、路径遍历防护、SSRF防护

### 未来改进方向

1. **WebSocket替代SSE** - 双向通信，支持服务器推送
2. **数据库存储** - SQLite替代JSON文件，支持复杂查询
3. **插件系统** - 支持第三方扩展
4. **多租户** - 企业级隔离和权限管理

---

## 附录：关键技术术语表

| 术语 | 说明 |
|-----|------|
| **SSE** | Server-Sent Events，服务器向客户端单向推送事件流 |
| **Profile** | 独立的配置/记忆/技能空间，支持多环境隔离 |
| **AIAgent** | Hermes核心Agent类，负责执行对话和工具调用 |
| **Toolset** | 工具集合，如terminal、file、web等 |
| **Context Compression** | 上下文压缩，将长对话历史压缩为摘要 |
| **Credential Pool** | 凭证池，支持多API密钥轮换和负载均衡 |
| **LRU Cache** | 最近最少使用缓存，自动驱逐旧数据 |
| **CSRF** | 跨站请求伪造，Web安全攻击方式 |
| **SSRF** | 服务器端请求伪造，利用服务器访问内网 |
| **PBKDF2** | 基于密码的密钥派生函数，用于密码哈希 |

---

**文档结束**

> 本文档深入剖析了Hermes WebUI的技术原理，从架构设计到实现细节，从安全机制到性能优化。希望这份文档能帮助您完全理解这个优雅而强大的AI网关系统。

**推荐阅读**：
- `ARCHITECTURE.md` - 官方架构文档
- `HERMES.md` - Hermes生态系统对比分析
- `api/streaming.py` - SSE引擎核心实现
- `api/config.py` - 配置和模型路由逻辑
