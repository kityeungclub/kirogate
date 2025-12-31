# KiroGate 配置格式与原理（基于 `main.ts`）

本文聚焦 `main.ts` 内的配置与请求格式处理机制，说明不同配置形式与其工作原理。

## 1. 环境变量配置（运行级别）

`loadSettings()` 读取的环境变量：

- `PROXY_API_KEY`：代理层 API Key。
- `REFRESH_TOKEN`：Kiro 刷新 token（全局默认值）。
- `PROFILE_ARN`：Kiro Profile ARN（可选）。
- `KIRO_REGION`：Kiro 区域，默认 `us-east-1`。
- `KIRO_CREDS_FILE`：凭据文件路径或 URL（JSON）。
- `TOKEN_REFRESH_THRESHOLD`：access token 刷新阈值（秒）。
- `MAX_RETRIES`：普通请求最大重试次数。
- `BASE_RETRY_DELAY`：重试基准延迟（秒）。
- `FIRST_TOKEN_TIMEOUT`：首 token 超时阈值（秒）。
- `FIRST_TOKEN_MAX_RETRIES`：首 token 超时重试次数。
- `LOG_LEVEL`：日志级别（TRACE/DEBUG/INFO/WARNING/ERROR）。
- `RATE_LIMIT_PER_MINUTE`：预留的限速配置（目前 `main.ts` 未使用）。
- `PORT`：HTTP 服务端口，默认 `8000`。

> `KIRO_REGION` 会影响上游域名：`codewhisperer.${region}.amazonaws.com` 与 `prod.${region}.auth.desktop.kiro.dev`。

## 2. 凭据来源与加载方式

`KiroAuthManager` 会从以下来源获取认证信息：

1. **环境变量**（`REFRESH_TOKEN` / `PROFILE_ARN` / `KIRO_REGION`）。
2. **凭据文件**（`KIRO_CREDS_FILE`）：
   - 支持本地文件路径或 URL。
   - JSON 字段可包含：`refreshToken`、`accessToken`、`profileArn`、`region`、`expiresAt`。

加载优先级以构造器参数为基础，若 `KIRO_CREDS_FILE` 存在会覆盖相应字段；  
如果文件中包含 `accessToken` 或 `expiresAt`，会直接用于初始化缓存。

## 3. API Key 认证格式（代理层）

`verifyApiKey()` 支持两种模式：

### 3.1 简单模式（单租户）

```
Authorization: Bearer PROXY_API_KEY
```

- 仅校验 `PROXY_API_KEY`。
- refresh token 使用服务端全局 `REFRESH_TOKEN`。
- 也支持使用 `x-api-key: PROXY_API_KEY` 头部。

### 3.2 组合模式（多租户）

```
Authorization: Bearer PROXY_API_KEY:REFRESH_TOKEN
```

- 以第一个 `:` 作为分隔符。
- `PROXY_API_KEY` 必须匹配服务端配置。
- `REFRESH_TOKEN` 使用请求方自带值。
- `getUserAuthManager()` 为每个 refresh token 缓存独立认证管理器（最多 100 个）。
- 同样支持 `x-api-key: PROXY_API_KEY:REFRESH_TOKEN`。

> 这使得同一服务可为不同用户代理各自的 Kiro 凭据，无需改动服务端配置。

## 4. 请求格式与转换原理

### 4.1 OpenAI Chat Completions

入口：`POST /v1/chat/completions`

- 请求体满足 `ChatCompletionRequest`。
- 通过 `buildKiroPayload()` 转为 Kiro 所需的 `conversationState`。
- 消息处理逻辑：
  - 合并相邻消息与 tool_calls（`mergeAdjacentMessages()`）。
  - 将 `tool_result` 变成 Kiro `toolResults`。
  - 超长工具描述移入 system prompt（`processToolsWithLongDescriptions()`）。

### 4.2 Anthropic Messages

入口：`POST /v1/messages` 或 `POST /messages`

- 请求体满足 `AnthropicMessagesRequest`。
- 先通过 `convertAnthropicToOpenAI()` 转为 OpenAI 结构，再走 OpenAI → Kiro 的主流程。
- 转换要点：
  - Anthropic `system` 合并成 OpenAI `system` 消息。
  - `tool_use` 转成 OpenAI `tool_calls`。
  - `tool_result` 被转成 `user` 消息内容块。
  - `thinking`、`image` 等内容被转为文本或占位符。

## 5. 模型映射与内部模型 ID

- `MODEL_MAPPING` 定义了外部模型名 → Kiro 内部模型名的映射。
- `getInternalModelId()` 会优先使用映射，否则原样透传。
- `/v1/models` 返回的模型列表来源于 `AVAILABLE_MODELS`。

## 6. Token 刷新阈值与重试参数

- `TOKEN_REFRESH_THRESHOLD`：在 access token 即将过期时提前刷新（秒级阈值）。  
- `MAX_RETRIES` / `BASE_RETRY_DELAY`：普通请求的重试与指数退避配置。  
- `FIRST_TOKEN_TIMEOUT` / `FIRST_TOKEN_MAX_RETRIES`：首 token 超时相关配置（代码中有实现，但当前主流程未启用）。  

## 7. 工具描述限制与处理策略

- `TOOL_DESCRIPTION_MAX_LENGTH` 限制 tool 描述长度（默认 4000 字符）。
- 超长描述会被移动到 `system prompt`，tool 的 `description` 只保留引用说明。
- 这样可以避免 Kiro 对 tool 描述长度的限制导致请求失败。

---

以上内容仅依据 `main.ts` 中的实现逻辑整理。
