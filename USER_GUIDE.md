# Sub2API 用户使用指南

面向**最终调用方**——拿到 API Key 后怎么把它接进各种客户端和 SDK。

> 运维/部署相关请看同目录 `DEPLOY_NOTES.md`。

## 一、你需要的两样东西

1. **服务入口（Base URL）**
   - `https://149.28.130.84:8443`
2. **API Key**
   - 形如 `sk-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`，由管理员在 Sub2API 后台 → API Key → 「新增」生成
   - 自己拿不到？联系管理员（`admin@sub2api.local`）

## 二、关于自签证书 ⚠️ 必读

服务的 HTTPS 证书是**自签的**（CN=149.28.130.84），浏览器会显示「不安全」、代码会报「证书校验失败」。每种客户端跳过校验的方式不一样：

| 工具 | 跳过 TLS 校验的方法 |
|---|---|
| 浏览器 | 首次访问点「高级 → 继续前往」 |
| curl | 加 `-k` 或 `--insecure` |
| Python `requests` / `httpx` | `verify=False` |
| Python `anthropic` / `openai` SDK | 传自定义 httpx client：`http_client=httpx.Client(verify=False)` |
| Node.js | 设环境变量 `NODE_TLS_REJECT_UNAUTHORIZED=0` |
| Claude Code / Codex CLI | 同上 `NODE_TLS_REJECT_UNAUTHORIZED=0` |
| Go | `http.Transport{TLSClientConfig: &tls.Config{InsecureSkipVerify: true}}` |

> 后续如果换成正规域名 + Let's Encrypt 证书，这一节就可以全部忽略。

## 三、两种 API 协议都支持

sub2api 同时兼容 **Anthropic Messages API** 和 **OpenAI Chat Completions API**。挑你客户端原本支持哪个用哪个。

### 3.1 Anthropic 协议（推荐用于 Claude 系）

- 端点：`POST /v1/messages`
- 鉴权：`x-api-key: <你的 key>`
- 版本头：`anthropic-version: 2023-06-01`

```bash
curl -sk https://149.28.130.84:8443/v1/messages \
  -H "x-api-key: sk-xxxxxx" \
  -H "anthropic-version: 2023-06-01" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "claude-haiku-4-5-20251001",
    "max_tokens": 200,
    "messages": [{"role":"user","content":"用一句话介绍你自己"}]
  }'
```

### 3.2 OpenAI 协议

- 端点：`POST /v1/chat/completions`
- 鉴权：`Authorization: Bearer <你的 key>`

```bash
curl -sk https://149.28.130.84:8443/v1/chat/completions \
  -H "Authorization: Bearer sk-xxxxxx" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "claude-haiku-4-5-20251001",
    "messages": [{"role":"user","content":"用一句话介绍你自己"}]
  }'
```

### 3.3 流式

两个协议都支持流式（SSE）。请求体加 `"stream": true`，响应是 `data: {...}\n\n` 帧。

## 四、可用模型

当前已接入 **Claude（Anthropic OAuth）**，可用：

- `claude-haiku-4-5-20251001` ← 便宜快，日常推荐
- `claude-sonnet-4-6`
- `claude-opus-4-7`

模型名以 `claude-` 开头都行；非 Claude 模型暂未配置上游账号。

完整可用模型列表可调：

```bash
curl -sk https://149.28.130.84:8443/v1/models \
  -H "Authorization: Bearer sk-xxxxxx"
```

## 五、各客户端怎么接

### 5.1 Claude Code CLI

```bash
export ANTHROPIC_BASE_URL=https://149.28.130.84:8443
export ANTHROPIC_AUTH_TOKEN=sk-xxxxxx
export NODE_TLS_REJECT_UNAUTHORIZED=0   # 自签证书必须

claude   # 正常启动，已走 sub2api
```

放到 `~/.bashrc` / `~/.zshrc` 里持久化。

### 5.2 Codex CLI (`@openai/codex`)

```bash
export OPENAI_BASE_URL=https://149.28.130.84:8443/v1
export OPENAI_API_KEY=sk-xxxxxx
export NODE_TLS_REJECT_UNAUTHORIZED=0
```

### 5.3 Python anthropic SDK

```python
import httpx
from anthropic import Anthropic

client = Anthropic(
    base_url="https://149.28.130.84:8443",
    api_key="sk-xxxxxx",
    http_client=httpx.Client(verify=False),   # 自签证书
)

resp = client.messages.create(
    model="claude-haiku-4-5-20251001",
    max_tokens=200,
    messages=[{"role": "user", "content": "hi"}],
)
print(resp.content[0].text)
```

### 5.4 Python openai SDK

```python
import httpx
from openai import OpenAI

client = OpenAI(
    base_url="https://149.28.130.84:8443/v1",
    api_key="sk-xxxxxx",
    http_client=httpx.Client(verify=False),
)

resp = client.chat.completions.create(
    model="claude-haiku-4-5-20251001",
    messages=[{"role": "user", "content": "hi"}],
)
print(resp.choices[0].message.content)
```

### 5.5 Node.js (`@anthropic-ai/sdk` / `openai`)

```js
// 在进程入口最早处设置（自签证书必须）
process.env.NODE_TLS_REJECT_UNAUTHORIZED = '0';

import Anthropic from "@anthropic-ai/sdk";
const client = new Anthropic({
  baseURL: "https://149.28.130.84:8443",
  apiKey: "sk-xxxxxx",
});

const msg = await client.messages.create({
  model: "claude-haiku-4-5-20251001",
  max_tokens: 200,
  messages: [{ role: "user", content: "hi" }],
});
console.log(msg.content[0].text);
```

### 5.6 LobeChat / ChatBox / NextChat 等聊天客户端

按 OpenAI 兼容服务来配：
- **API 类型**：OpenAI
- **API 基地址 / Base URL**：`https://149.28.130.84:8443/v1`
- **API Key**：你的 sk-xxxxxx
- **模型**：`claude-haiku-4-5-20251001` 等

> 这类客户端运行在浏览器/Electron 里，证书校验通常用浏览器内核——浏览器需要先访问一次 `https://149.28.130.84:8443/` 点过"继续访问"，否则 fetch 会被拦。

### 5.7 Cursor / Continue / Cline 等 IDE 插件

- **Continue**（VSCode）：在 `~/.continue/config.json` 加：
  ```json
  {
    "models": [{
      "title": "Claude via sub2api",
      "provider": "openai",
      "apiBase": "https://149.28.130.84:8443/v1",
      "apiKey": "sk-xxxxxx",
      "model": "claude-sonnet-4-6"
    }]
  }
  ```
- **Cursor**：Settings → Models → 「OpenAI API Key」展开 → 「Override OpenAI Base URL」填上面 base URL，API Key 填 sub2api 的 key。

> Cursor 的自签证书坑比较深，没法在 UI 关 TLS 校验。要么先用浏览器接受一次证书，要么部署正式证书。

### 5.8 Go

```go
import (
    "crypto/tls"
    "net/http"
    "time"
)

httpClient := &http.Client{
    Timeout: 5 * time.Minute,
    Transport: &http.Transport{
        TLSClientConfig: &tls.Config{InsecureSkipVerify: true},
    },
}
// 用这个 httpClient 配你的 SDK
```

## 六、查用量 / 配额

- **管理员后台**：https://149.28.130.84:8443 → 「用量」/「Dashboard」
- **自助查询接口**（带 key）：
  ```bash
  curl -sk https://149.28.130.84:8443/api/v1/usage/summary \
    -H "Authorization: Bearer sk-xxxxxx"
  ```
- 后台可以给每个 Key 配 RPM、5h/1d/7d 用量上限，超了会返回 429。

## 七、常见报错对照表

| 报错 | 含义 | 你能做的 |
|---|---|---|
| `INSUFFICIENT_BALANCE` | 账号余额不足 | 找管理员充值 |
| `No available accounts` | 上游 Claude 账号都不可用 | 找管理员检查账号状态/调度配置 |
| 401 / `Unauthorized` | Key 错或被禁用 | 找管理员确认 |
| 429 / `rate_limit` | 触发了 Key 上的 RPM / 用量限制 | 等限速窗口过去，或让管理员放宽 |
| `certificate signed by unknown authority` / `unable to verify the first certificate` | 自签证书没跳过校验 | 见本指南第二节 |
| `502 upstream_error` | 上游瞬时故障/被限速 | 重试，或反馈管理员 |

## 八、限制

- 只接了 Claude 系列模型，没有 OpenAI/Gemini 上游。
- 自签证书，公网客户端体验不佳；建议自己用或部署正式证书后再分发给团队。
- 服务跑在 Vultr 单机（149.28.130.84），无高可用、无 SLA。
