# A2UI 前端本地联调启动指南

本文记录本项目在本机启动 API Server，并让 A2UI 前端页面请求转发到本地服务的完整流程。

目标效果：

- 浏览器打开：`http://open-joycoder-jd-a2ui-editor-t.clear.jd.com/#/editor?debug=1`
- 前端请求：`http://pre-a2ui-manage.jd.com/api/models`、`/api/chat`、`/api/threads/{thread_id}/a2ui`
- 实际命中本机：`127.0.0.1:80`
- 端口转发到项目 API Server：`127.0.0.1:8642`

整体链路：

```text
浏览器页面
  -> http://pre-a2ui-manage.jd.com/api/...
  -> hosts 解析到 127.0.0.1
  -> 本机 80 端口
  -> scripts/forward_api_port.py
  -> 本机 8642 端口
  -> Hermes API Server
```

## 1. 前置条件

进入项目根目录：

```bash
cd /Users/fenggongye1/Downloads/pythonProject/jdt-hermes-agent
```

本项目使用 `venv`，不是 `.venv`：

```bash
source venv/bin/activate
```

确认公司大模型环境变量已配置。至少需要：

```bash
export JD_API_KEY="你的公司大模型 key"
export JD_BASE_URL="http://ai-api.jdcloud.com/v1"
```

如果在 PyCharm 里启动，也要确保这些环境变量能被 PyCharm 运行配置读到。

## 2. 配置本地域名解析

A2UI 前端会请求 `pre-a2ui-manage.jd.com`，本地联调时要把它指向本机。

推荐用 SwitchHosts 或直接编辑 `/etc/hosts`，增加：

```text
127.0.0.1 pre-a2ui-manage.jd.com
```

验证：

```bash
ping pre-a2ui-manage.jd.com
```

预期能看到解析到 `127.0.0.1`。

## 3. PyCharm 启动方式

适合日常调试代码、打断点、看日志。

在 PyCharm / JoyCode 里创建或选择运行配置，例如截图中的 `Hermes API Server`：

- 配置类型：Python
- Python 解释器：`/Users/fenggongye1/Downloads/pythonProject/jdt-hermes-agent/venv/bin/python`
- Working directory：`/Users/fenggongye1/Downloads/pythonProject/jdt-hermes-agent`
- Module name：`hermes_cli.main`
- Parameters：`gateway run --replace -v`

需要的环境变量建议配置：

```text
API_SERVER_ENABLED=true
API_SERVER_HOST=127.0.0.1
API_SERVER_PORT=8642
API_SERVER_CORS_ORIGINS=http://open-joycoder-jd-a2ui-editor-t.clear.jd.com,http://open-joycoder-a2ui-editor-t.clear.jd.com,http://pre-a2ui-manage.jd.com,http://127.0.0.1,http://localhost
JD_BASE_URL=http://ai-api.jdcloud.com/v1
JD_API_KEY=你的公司大模型 key
```

启动后看到类似日志，说明 API Server 已经起来：

```text
Hermes Gateway Starting...
INFO gateway.run: Connecting to api_server...
INFO gateway.platforms.api_server: [Api_Server] API server listening on http://127.0.0.1:8642
INFO gateway.run: ✓ api_server connected
```

## 4. 命令行启动方式

适合快速验证，不依赖 IDE。

```bash
cd /Users/fenggongye1/Downloads/pythonProject/jdt-hermes-agent
source venv/bin/activate

export API_SERVER_ENABLED=true
export API_SERVER_HOST=127.0.0.1
export API_SERVER_PORT=8642
export API_SERVER_CORS_ORIGINS="http://open-joycoder-jd-a2ui-editor-t.clear.jd.com,http://open-joycoder-a2ui-editor-t.clear.jd.com,http://pre-a2ui-manage.jd.com,http://127.0.0.1,http://localhost"
export JD_BASE_URL="http://ai-api.jdcloud.com/v1"
export JD_API_KEY="你的公司大模型 key"

python -m hermes_cli.main gateway run --replace -v
```

如果你习惯使用项目里的 `uv` 启动，也可以使用类似 PyCharm 日志中的命令：

```bash
/Users/fenggongye1/.local/bin/uv run /Users/fenggongye1/Downloads/pythonProject/jdt-hermes-agent/venv/bin/python -m hermes_cli.main gateway run --replace -v
```

启动成功后，本机直接验证：

```bash
curl http://127.0.0.1:8642/health
curl http://127.0.0.1:8642/api/models
```

预期：

- `/health` 返回 `{"status":"ok", ...}`
- `/api/models` 返回 JD 模型列表

## 5. 启动 80 到 8642 的端口转发

浏览器访问的是 `http://pre-a2ui-manage.jd.com/api/...`，默认 HTTP 端口是 80。

项目 API Server 实际监听 `8642`，所以需要另开一个终端执行端口转发：

```bash
cd /Users/fenggongye1/Downloads/pythonProject/jdt-hermes-agent
source venv/bin/activate
sudo scripts/forward_api_port.py
```

预期输出：

```text
[转发] 正在监听 http://127.0.0.1:80 -> http://127.0.0.1:8642
[转发] 按 Ctrl+C 停止
```

说明：

- `80` 是 macOS/Linux 特权端口，所以必须用 `sudo`。
- 这个脚本只负责端口转发，不负责启动 API Server。
- API Server 必须先在 `127.0.0.1:8642` 正常运行。

转发验证：

```bash
curl http://pre-a2ui-manage.jd.com/api/models
```

如果返回模型列表，说明：

```text
pre-a2ui-manage.jd.com -> 127.0.0.1:80 -> 127.0.0.1:8642
```

整条链路已打通。

## 6. 打开 A2UI 前端页面

浏览器打开：

```text
http://open-joycoder-jd-a2ui-editor-t.clear.jd.com/#/editor?debug=1
```

打开 Chrome DevTools：

1. 进入 `网络 / Network`
2. 选择 `Fetch/XHR`
3. 刷新页面
4. 点击 `models` 请求

重点检查：

- 请求网址：`http://pre-a2ui-manage.jd.com/api/models`
- 请求方法：`GET`
- 状态码：`200 OK`
- 远程地址：`127.0.0.1:80`

如果远程地址是 `127.0.0.1:80`，说明浏览器确实打到了本机 80 端口，再由脚本转发到本地 API Server。

## 7. 发送消息验证 `/api/chat`

在页面输入框里发送一条普通消息，例如：

```text
你好，用一句话回复我
```

或者发送 A2UI 生成请求：

```text
生成京东金融风格首页
```

在 Network 里观察 `chat` 请求：

- 请求网址应该是：`http://pre-a2ui-manage.jd.com/api/chat`
- 状态码应该是：`200 OK`
- 远程地址应该是：`127.0.0.1:80`
- 响应类型是 SSE 流式返回，应该能看到 `event: meta`、`event: text`、`event: step`、`event: a2ui`、`event: done` 等事件

也可以用命令行模拟：

```bash
curl -N -X POST http://pre-a2ui-manage.jd.com/api/chat \
  -H 'Content-Type: application/json' \
  -d '{"message":"你好，用一句话回复我","model_name":"jd-api:GLM-5.1"}'
```

预期至少看到：

```text
event: meta
event: text
event: done
```

A2UI 生成场景预期能看到：

```text
event: step
event: a2ui
event: done
```

## 8. 常见问题排查

### 8.1 浏览器请求不是 127.0.0.1:80

检查 hosts 是否生效：

```bash
ping pre-a2ui-manage.jd.com
```

如果不是 `127.0.0.1`，说明 hosts 没配好，或 SwitchHosts 对应配置没有开启。

### 8.2 `sudo scripts/forward_api_port.py` 启动失败

如果提示 80 端口被占用：

```bash
sudo lsof -iTCP:80 -sTCP:LISTEN
```

停掉占用 80 端口的进程后再启动转发脚本。

### 8.3 `/api/models` 返回 403 或 CORS 报错

检查 API Server 启动环境变量：

```text
API_SERVER_CORS_ORIGINS=http://open-joycoder-jd-a2ui-editor-t.clear.jd.com,http://open-joycoder-a2ui-editor-t.clear.jd.com,http://pre-a2ui-manage.jd.com,http://127.0.0.1,http://localhost
```

修改后需要重启 API Server。

### 8.4 `/api/chat` 报 JD_API_KEY 找不到

说明启动 API Server 的进程没有读到公司大模型 key。

检查启动终端或 PyCharm Run Configuration 是否配置了：

```text
JD_API_KEY=你的公司大模型 key
JD_BASE_URL=http://ai-api.jdcloud.com/v1
```

### 8.5 请求连接失败

按顺序检查：

```bash
curl http://127.0.0.1:8642/health
curl http://127.0.0.1:80/health
curl http://pre-a2ui-manage.jd.com/health
curl http://pre-a2ui-manage.jd.com/api/models
```

判断方式：

- `8642` 不通：API Server 没启动或启动失败。
- `8642` 通、`80` 不通：端口转发脚本没启动或 80 被占用。
- `80` 通、域名不通：hosts 没生效。
- `/health` 通、`/api/models` 不通：检查 API Server 路由、CORS 或运行日志。

## 9. 正常联调 checklist

每次本地联调按这个顺序执行：

1. 启动 API Server：PyCharm 运行 `Hermes API Server`，或命令行执行 `python -m hermes_cli.main gateway run --replace -v`。
2. 确认 `curl http://127.0.0.1:8642/health` 正常。
3. 确认 hosts 有 `127.0.0.1 pre-a2ui-manage.jd.com`。
4. 启动端口转发：`sudo scripts/forward_api_port.py`。
5. 确认 `curl http://pre-a2ui-manage.jd.com/api/models` 正常。
6. 打开 `http://open-joycoder-jd-a2ui-editor-t.clear.jd.com/#/editor?debug=1`。
7. DevTools 查看 `models` 请求，确认远程地址是 `127.0.0.1:80`。
8. 发送聊天或 A2UI 生成请求，确认 `/api/chat` 有 SSE 事件返回。
