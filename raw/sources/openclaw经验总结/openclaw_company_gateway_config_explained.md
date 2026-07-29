# OpenClaw 公司网关接入配置说明

## 一、目标

本文说明：在 OpenClaw 中，如何通过 **公司内部模型网关** 接入 MiniMax 类模型，并解释为什么“`auth-profiles.json` + 主配置文件”这一组合可以正常工作，以及它背后的解析原理。

---

## 二、先区分三个层次

OpenClaw 的配置里，最容易混淆的是下面三层：

### 1）OpenClaw 网关认证
这是 OpenClaw 自己的本地网关/代理服务认证，用来控制谁可以访问本地 gateway。

它的职责是：
- 保护 OpenClaw 网关本身
- 控制本地服务是否允许被连接
- 不直接等同于模型厂商的 API key

### 2）模型 Provider 认证
这是 **真正调用公司内部模型网关** 时使用的凭证。

它的职责是：
- 告诉 OpenClaw 连接哪个模型服务
- 携带调用模型服务所需的 API key 或 token
- 决定请求能不能成功到达上游模型网关

### 3）模型路由与模型名
这是告诉 OpenClaw：
- 使用哪个 provider
- 取哪个具体模型
- 该模型在 OpenClaw 中显示成什么名字

也就是说，`minimax/MiniMax-M2.7` 这种写法，本质上是“provider/model”的路由标识。

---

## 三、两个文件分别负责什么

## 文件 A：`auth-profiles.json`

这个文件存的是 **模型认证信息**，也就是“怎么拿到调用上游网关所需的 key”。

典型结构是：

```json
{
  "version": 1,
  "profiles": {
    "minimax:cn": {
      "type": "api_key",
      "provider": "minimax",
      "key": "公司网关发放的模型Key"
    }
  },
  "lastGood": {
    "minimax": "minimax:cn"
  }
}
```

它的作用可以理解为：
- `minimax:cn` 是一个认证档案名
- `provider: minimax` 表示它服务于这个 provider
- `type: api_key` 表示这是 API key 型认证
- `key` 是实际秘密值

### 为什么它能工作
因为 OpenClaw 在激活配置时，会读取这个认证档案，并把它绑定到对应的 provider 上。这样一来，当 provider 发起请求时，就能拿到正确的身份凭证。

### `lastGood` 的作用
`lastGood` 用来告诉 OpenClaw：
- 在多个可用认证档案里，优先用哪个
- 当重新加载配置时，保留最后一次成功工作的凭证选择

这可以降低因为切换配置导致的短暂失败。

---

## 文件 B：主配置 `openclaw.json`

主配置负责两件事：
1. 声明模型 provider 怎么连
2. 声明默认要用哪个模型

典型写法如下：

```json
{
  "agents": {
    "defaults": {
      "workspace": "/Users/xxx/.openclaw/workspace",
      "models": {
        "minimax/MiniMax-M2.7": {
          "alias": "Minimax"
        }
      },
      "model": {
        "primary": "minimax/MiniMax-M2.7"
      }
    }
  },
  "models": {
    "mode": "merge",
    "providers": {
      "minimax": {
        "baseUrl": "https://modelservice.jdcloud.com/v1",
        "api": "openai-completions",
        "apiKeyFromProfile": "minimax:cn",
        "authHeader": true,
        "models": [
          {
            "id": "MiniMax-M2.7",
            "name": "MiniMax M2.7",
            "reasoning": true,
            "input": ["text"],
            "contextWindow": 204800,
            "maxTokens": 131072
          }
        ]
      }
    }
  }
}
```

### 这里每一项的意义

#### `models.providers.minimax`
表示定义一个叫 `minimax` 的 provider。

#### `baseUrl`
表示上游模型网关地址，也就是实际请求要发送到哪里。

#### `api`
表示该网关遵循哪种请求/响应协议。
常见是：
- `openai-completions`
- `anthropic-messages`

如果你的公司网关是 OpenAI 兼容，就用前者；如果是 Anthropic 兼容，就用后者。

#### `apiKeyFromProfile`
表示这个 provider 不直接写死 key，而是从 `auth-profiles.json` 中指定档案读取。

#### `authHeader: true`
表示把认证信息放进标准请求头里，通常是 `Authorization`。

#### `models`
表示该 provider 下有哪些模型可用。
这里的 `id` 要和 `agents.defaults.model.primary` 的路径保持一致。

---

## 四、为什么这种配置可以工作

这个方案能工作，原因是 OpenClaw 的配置和认证机制是分层设计的。

### 原理 1：配置解析是“先定义 provider，再绑定认证档案”
OpenClaw 不是把所有东西写在一个地方，而是把：
- provider 定义
- 凭证存储
- 默认模型选择
分开管理。

因此只要三者能对应上，就可以完成一次完整调用：
1. 根据 `primary` 找到要用的模型
2. 根据 `provider` 找到对应 provider 定义
3. 根据 `apiKeyFromProfile` 找到认证档案
4. 从认证档案读取 key
5. 组装请求并发送到 `baseUrl`

### 原理 2：模型名采用 `provider/model` 形式
`minimax/MiniMax-M2.7` 不是单纯的字符串，而是一个路由标识。

它告诉 OpenClaw：
- 前半部分 `minimax` 是 provider id
- 后半部分 `MiniMax-M2.7` 是具体模型 id

这样系统就能把“模型选择”和“上游服务”绑定起来。

### 原理 3：认证档案和主配置解耦
把 key 放在 `auth-profiles.json`，有两个好处：
- 便于不同环境切换
- 便于重载时只替换凭证，不改业务配置

### 原理 4：激活时会提前校验秘密是否可解析
OpenClaw 在激活配置时会检查 secret 是否能解析成功。
也就是说：
- 如果 key 缺失，启动阶段就会失败
- 如果 key 为空，也会立即报错
- 不会等到真正发请求时才崩

这就是你之前看到 `SecretRefResolutionError` 的原因。

---

## 五、为什么你的报错会出现

你遇到的错误，本质上通常来自以下几类原因：

### 1）环境变量不存在
如果配置写的是：

```json
"apiKey": "${COMPANY_MODEL_GATEWAY_KEY}"
```

但系统里没有设置这个环境变量，就会报“missing or empty”。

### 2）provider 认证方式和实际网关不一致
例如：
- 配成 `openai-completions`
- 但公司网关实际上要求 Anthropic 格式

这时请求虽然发出去了，但上游会解析失败。

### 3）模型 ID 没对齐
如果主配置里写的是：

```json
"primary": "minimax/MiniMax-M2.7"
```

那 `models.providers.minimax.models` 里就必须真的存在 `MiniMax-M2.7`。
大小写、连字符都要一致。

### 4）把网关 token 当成模型 key
OpenClaw 网关 token 和模型 provider key 是两类不同的凭证。
前者保护 OpenClaw 自身，后者用于访问上游模型。
二者不能互相替代。

---

## 六、推荐的工作流程

### 第一步：确定公司网关协议
先确认公司网关属于哪一类：
- OpenAI 兼容
- Anthropic 兼容

### 第二步：在 `auth-profiles.json` 中放模型 key
把公司网关发放的模型凭证写到认证档案里，或者通过 `keyRef` 指向环境变量。

### 第三步：在主配置中定义 provider
填写：
- `baseUrl`
- `api`
- `apiKeyFromProfile`
- `models`

### 第四步：把默认模型指向对应路由
保证：
- `agents.defaults.model.primary`
- `models.providers.<id>.models[].id`

能够一一对应。

### 第五步：执行校验
运行 `openclaw doctor` 和 `openclaw models status --probe`，看 provider 和认证是否都正常。

---

## 七、这套设计的优点

### 1）安全性更高
key 不必散落在多个配置里。

### 2）可维护性更好
换 key、换环境、换 provider 时，只需要改对应层。

### 3）更适合公司内部网关
公司网关经常会有：
- 租户头
- 统一鉴权
- 自定义路径
- 协议兼容层

分层配置更适合这种复杂环境。

### 4）启动时就能发现问题
因为 OpenClaw 会在激活阶段解析 secret，很多错误不会拖到运行时才暴露。

---

## 八、最终可记住的一句话

这套配置之所以能工作，是因为 OpenClaw 把 **“认证信息”**、**“模型路由”**、**“网关协议”** 三件事分开管理：
- `auth-profiles.json` 负责提供 key
- `models.providers` 负责定义怎么连上游
- `agents.defaults.model.primary` 负责指定默认模型

只要这三层对齐，OpenClaw 就能把请求正确地转发到公司内部模型网关。

---

## 九、检查清单

在交付或排障时，可以按下面顺序检查：

- `auth-profiles.json` 里是否有正确的 provider 和 key
- 主配置里 `provider` 名称是否一致
- `baseUrl` 是否是公司网关真实地址
- `api` 是否与网关协议匹配
- `primary` 是否与 `models[].id` 一致
- 环境变量是否存在且非空
- 启动后 `openclaw doctor` 是否通过
- `openclaw models status --probe` 是否能探测成功

