# KiroGate 代理工作流程（基于 `main.ts`）

本文只分析 `main.ts`，梳理这个 AI API Proxy 的整体工作流程。

## 1. 启动阶段

1. **加载配置**：`loadSettings()` 从环境变量读取运行配置（端口、密钥、重试阈值等）。
2. **校验凭据**：`main()` 检查 `REFRESH_TOKEN` 或 `KIRO_CREDS_FILE` 是否存在，否则直接退出。
3. **初始化认证管理器**：`KiroAuthManager` 读取 refresh token/配置文件，准备刷新 access token。
4. **加载监控数据**：`loadMetricsFromKV()` 从 Deno KV 恢复统计数据。
5. **定时持久化**：每 30 秒触发 `persistMetricsIfNeeded()`，进程退出时调用 `forcePersistMetrics()`。
6. **启动服务**：`Deno.serve({ port }, handleRequest)` 开始监听请求。

## 2. 请求入口与路由

所有请求进入 `handleRequest()`，按 **HTTP 方法 + 路径** 路由：

- **GET**
  - `/`、`/docs`、`/swagger`、`/playground`、`/dashboard` 等：渲染前端页面。
  - `/v1/models`：返回模型列表（需要 API Key 认证）。
  - `/health`、`/api/metrics`：健康检查与监控数据。
- **POST**
  - `/v1/chat/completions`：OpenAI 格式请求 → `handleChatCompletions()`。
  - `/v1/messages` 或 `/messages`：Anthropic 格式请求 → `handleAnthropicMessages()`。

路由分支结束后，如果是模型相关接口（`/v1/chat/completions`、`/v1/messages`、`/messages`），会记录请求日志进入 `recordRequest()`，再由 KV 批量持久化。

## 3. 认证与多租户处理

`verifyApiKey()` 支持两种 API Key 形式：

1. **简单模式**：`PROXY_API_KEY`
2. **组合模式**：`PROXY_API_KEY:REFRESH_TOKEN`

- 组合模式允许每个请求自带 `refresh token`，实现多租户使用。
- `getUserAuthManager()` 会为不同的 `refresh token` 创建/缓存独立的 `KiroAuthManager`，缓存最多 100 个用户。

### 3.1 Access Token 刷新流程

`KiroAuthManager.getAccessToken()` 内部调用 `refreshTokenRequest()`：  

- `refreshTokenRequest()` 向 `https://prod.${region}.auth.desktop.kiro.dev/refreshToken` 发起刷新请求。
- 失败重试策略：`429` 或 `5xx` 指数退避重试；`4xx`（除 429）视为不可重试错误。
- 返回 `accessToken`、`refreshToken`、`profileArn` 等字段时，会更新本地缓存，并计算 `expiresAt`。

## 4. OpenAI → Kiro 的主流程

在 `handleChatCompletions()` 中执行：

1. **鉴权**：`verifyApiKey()`。
2. **读取请求体**：解析 OpenAI `ChatCompletionRequest`。
3. **构建 Kiro 请求**：
   - `buildKiroPayload()` 将消息格式转换为 Kiro 的 `conversationState` 结构。
   - 内部逻辑包括：
     - 合并相邻消息（`mergeAdjacentMessages()`）
     - 提取/合并 tool_calls 与 tool_result
     - 处理超长 tool description（超限内容移入 system prompt）
4. **调用上游 Kiro API**：
   - 通过 `requestWithRetry()` 发送到 `${apiHost}/generateAssistantResponse`。
   - 403/429/5xx 自动重试或刷新 token。
5. **返回响应**：
   - `stream=true` → `streamKiroToOpenAI()` 解析 AWS SSE 流并转为 OpenAI SSE。
   - `stream=false` → `collectStreamResponse()` 收集完整内容再返回 JSON。

## 5. Anthropic → Kiro 的主流程

在 `handleAnthropicMessages()` 中执行：

1. **鉴权**：`verifyApiKey()`。
2. **读取请求体**：解析 Anthropic `Messages` 请求。
3. **转为 OpenAI 格式**：`convertAnthropicToOpenAI()`。
4. **复用 OpenAI → Kiro 流程**：调用 `buildKiroPayload()` + `requestWithRetry()`。
5. **返回 Anthropic 响应**：
   - `stream=true` → `streamKiroToAnthropic()` 将 Kiro 流转换为 Anthropic SSE。
   - `stream=false` → `collectAnthropicResponse()` 组装成 Anthropic message JSON。

## 6. 流式解析与 token 估算

- `AwsEventStreamParser` 解析 Kiro 返回的 AWS SSE 事件流。
- 支持识别：`content`、`tool_start`、`tool_input`、`tool_stop`、`usage` 等。
- `calculateUsageTokens()` 负责估算 tokens：
  - 优先使用 Kiro 返回的 `contextUsagePercentage`。
  - 若无，则按 **4 字符 ≈ 1 token** 的粗略算法回退估算。

## 7. 错误处理与重试策略

- **认证失败**：返回 401。
- **Kiro 400**：记录请求 payload 和错误响应（不重试）。
- **Kiro 403**：强制刷新 token 后重试。
- **Kiro 429 / 5xx**：指数退避重试。
- **流式首 token 超时**：`streamWithFirstTokenRetry()` 可进行多次重试（目前 OpenAI/Anthropic 流程使用无超时重试版本）。

## 8. 监控与日志

- `recordRequest()` 记录模型相关接口的请求信息（状态码、耗时、模型、错误信息）。  
- 统计指标存于内存 `metrics`，由 `persistMetricsIfNeeded()` 批量写入 Deno KV。  
- `/api/metrics` 返回当前监控信息，便于外部面板或调试使用。  

## 9. 上游主机与区域配置

- Kiro Auth：`https://prod.${region}.auth.desktop.kiro.dev/refreshToken`  
- Kiro API：`https://codewhisperer.${region}.amazonaws.com`  
- Kiro Q：`https://q.${region}.amazonaws.com`（目前在 `main.ts` 中仅初始化未使用）  

---

以上流程均来自 `main.ts` 内部逻辑，没有涉及其他文件。
