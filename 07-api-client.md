# 07 — API 客户端层

## 1. 模块职责

`src/services/api/` 是 Claude Code 与 Anthropic 后端通信的完整抽象层。它负责：
- 根据部署环境（直连 / AWS Bedrock / GCP Vertex / Azure Foundry）动态创建 SDK 客户端
- 封装流式/非流式请求，并注入 Prompt Caching、Beta Header、Thinking 等扩展参数
- 实现带指数退避的智能重试循环（区分前台/后台请求、Fast Mode、订阅用户）
- 将 API 错误映射为用户友好的 `AssistantMessage`，并上报分析事件

---

## 2. 核心文件地图

| 文件 | 行数 | 职责 |
|------|------|------|
| `client.ts` | ~390 | 工厂函数：`getAnthropicClient()` — 按环境变量选择 Anthropic / Bedrock / Vertex / Foundry SDK |
| `claude.ts` | ~3420 | 核心请求入口：`queryModelWithStreaming()` / `queryModelWithoutStreaming()`，组装请求参数 |
| `withRetry.ts` | ~823 | 通用重试器 `withRetry()`：指数退避 + Fast Mode 降速 + 持久重试 |
| `errors.ts` | ~1208 | 错误分类与用户消息映射 `getAssistantMessageFromError()` / `classifyAPIError()` |
| `logging.ts` | ~789 | 请求成功/失败事件上报 `logAPIQuery/Error/SuccessAndDuration()` |
| `usage.ts` | ~64 | 获取 Claude.ai 订阅用量 `fetchUtilization()` |
| `emptyUsage.ts` | — | 空 usage 对象常量 `EMPTY_USAGE` |
| `errorUtils.ts` | — | 连接错误详情解析 `extractConnectionErrorDetails()` |

---

## 3. 关键数据结构

### 3.1 请求 Options（claude.ts）

```typescript
// src/services/api/claude.ts
export type Options = {
  getToolPermissionContext: () => Promise<ToolPermissionContext>
  model: string
  toolChoice?: BetaToolChoiceTool | BetaToolChoiceAuto
  isNonInteractiveSession: boolean
  extraToolSchemas?: BetaToolUnion[]
  maxOutputTokensOverride?: number
  fallbackModel?: string          // 529 过载时的备用模型
  onStreamingFallback?: () => void
  querySource: QuerySource        // 请求来源标识（用于重试策略选择）
  agents: AgentDefinition[]
  enablePromptCaching?: boolean
  skipCacheWrite?: boolean
  temperatureOverride?: number
  effortValue?: EffortValue       // 思考努力程度
  mcpTools: Tools
  queryTracking?: QueryChainTracking
  agentId?: AgentId               // 仅 subagent 时设置
  outputFormat?: BetaJSONOutputFormat
  fastMode?: boolean              // Max/Ultra 订阅的高速模式
  taskBudget?: { total: number; remaining?: number } // API 侧 token 预算
}
```

### 3.2 重试上下文（withRetry.ts）

```typescript
// src/services/api/withRetry.ts
export interface RetryContext {
  maxTokensOverride?: number    // 上下文溢出时自动缩减 max_tokens
  model: string
  thinkingConfig: ThinkingConfig
  fastMode?: boolean
}

interface RetryOptions {
  maxRetries?: number           // 默认 10
  model: string
  fallbackModel?: string
  thinkingConfig: ThinkingConfig
  fastMode?: boolean
  signal?: AbortSignal
  querySource?: QuerySource
  initialConsecutive529Errors?: number  // 从流式切非流式时携带已有 529 计数
}
```

### 3.3 用量数据（usage.ts）

```typescript
// src/services/api/usage.ts
export type Utilization = {
  five_hour?: RateLimit | null
  seven_day?: RateLimit | null
  seven_day_opus?: RateLimit | null
  extra_usage?: ExtraUsage | null
}

export type RateLimit = {
  utilization: number | null   // 0-100 百分比
  resets_at: string | null     // ISO 8601
}
```

---

## 4. 执行流程

### 4.1 客户端创建流程（client.ts）

```
getAnthropicClient()
  ├── 构建 defaultHeaders（x-app, User-Agent, Session-Id, 自定义头）
  ├── checkAndRefreshOAuthTokenIfNeeded()  ← 确保 token 新鲜
  ├── configureApiKeyHeaders()             ← 非订阅用户注入 API Key
  ├── buildFetch()                         ← 包装 fetch：注入 x-client-request-id
  │
  ├── CLAUDE_CODE_USE_BEDROCK=true  → new AnthropicBedrock(awsRegion, awsCredentials)
  ├── CLAUDE_CODE_USE_FOUNDRY=true  → new AnthropicFoundry(azureADTokenProvider)
  ├── CLAUDE_CODE_USE_VERTEX=true   → new AnthropicVertex(region, googleAuth)
  └── 默认                          → new Anthropic(apiKey / authToken)
```

### 4.2 重试循环流程（withRetry.ts）

```
withRetry(getClient, operation, options)
  for attempt = 1..maxRetries+1:
    ├── client 懒加载（首次 or 401/403/ECONNRESET 后重建）
    ├── operation(client, attempt, retryContext)
    │    └── 成功 → return 结果
    │
    └── 捕获错误:
         ├── Fast Mode 429/529 → 短延迟继续 or 触发 cooldown 降速
         ├── 后台请求 529 → 直接 throw CannotRetryError（不放大容量压力）
         ├── 连续 3 次 529 (Opus) → throw FallbackTriggeredError(fallbackModel)
         ├── 上下文溢出 400 → 计算 adjustedMaxTokens，retryContext.maxTokensOverride = N
         ├── 持久重试模式 → yield keep-alive SystemAPIErrorMessage（防 host idle）
         └── 指数退避: delay = min(500 * 2^attempt, 32000ms) + jitter(25%)
```

---

## 5. 关键代码解析

### 5.1 多云客户端工厂（client.ts）

```typescript
// src/services/api/client.ts
export async function getAnthropicClient({ apiKey, maxRetries, model, fetchOverride, source }) {
  const defaultHeaders = {
    'x-app': 'cli',
    'User-Agent': getUserAgent(),
    'X-Claude-Code-Session-Id': getSessionId(),
    ...getCustomHeaders(),                    // ANTHROPIC_CUSTOM_HEADERS 环境变量解析
  }

  await checkAndRefreshOAuthTokenIfNeeded()   // OAuth token 刷新（无锁，幂等）

  if (!isClaudeAISubscriber()) {
    await configureApiKeyHeaders(defaultHeaders, ...)  // 注入 Authorization: Bearer
  }

  const ARGS = {
    defaultHeaders,
    maxRetries,
    timeout: parseInt(process.env.API_TIMEOUT_MS || '600000'),  // 默认 10 分钟
    dangerouslyAllowBrowser: true,
    fetchOptions: getProxyFetchOptions({ forAnthropicAPI: true }),
    fetch: buildFetch(fetchOverride, source),  // 包装 fetch 注入请求 ID
  }

  if (isEnvTruthy(process.env.CLAUDE_CODE_USE_BEDROCK)) {
    const awsRegion = model === getSmallFastModel()
      ? process.env.ANTHROPIC_SMALL_FAST_MODEL_AWS_REGION ?? getAWSRegion()
      : getAWSRegion()
    const cachedCredentials = await refreshAndGetAwsCredentials()
    return new AnthropicBedrock({ ...ARGS, awsRegion, ...cachedCredentials })
  }
  if (isEnvTruthy(process.env.CLAUDE_CODE_USE_VERTEX)) {
    const region = getVertexRegionForModel(model)  // 每个模型可配置不同区域
    const googleAuth = new GoogleAuth({ scopes: ['...cloud-platform'] })
    return new AnthropicVertex({ ...ARGS, region, googleAuth })
  }
  // 直连（默认）
  return new Anthropic({
    apiKey: isClaudeAISubscriber() ? null : apiKey || getAnthropicApiKey(),
    authToken: isClaudeAISubscriber() ? getClaudeAIOAuthTokens()?.accessToken : undefined,
    ...ARGS,
  })
}
```

**设计亮点**：同一个 `ARGS` 基础配置被三种云提供商共享，差异部分通过 spread 追加，避免了重复代码。

### 5.2 请求 ID 注入（client.ts）

```typescript
// src/services/api/client.ts
function buildFetch(fetchOverride, source) {
  const inner = fetchOverride ?? globalThis.fetch
  const injectClientRequestId = getAPIProvider() === 'firstParty' && isFirstPartyAnthropicBaseUrl()

  return (input, init) => {
    const headers = new Headers(init?.headers)
    if (injectClientRequestId && !headers.has(CLIENT_REQUEST_ID_HEADER)) {
      // 客户端生成 UUID，超时后无 server request-id 时仍可关联日志
      headers.set(CLIENT_REQUEST_ID_HEADER, randomUUID())
    }
    logForDebugging(`[API REQUEST] ${pathname} ${id} source=${source}`)
    return inner(input, { ...init, headers })
  }
}
```

**设计亮点**：在 fetch 层注入 `x-client-request-id`，而不是在 SDK 层，因为 SDK 只在有服务端响应时才能获取 request-id，超时场景下客户端 ID 是唯一追踪手段。

### 5.3 智能重试机制（withRetry.ts）

```typescript
// src/services/api/withRetry.ts
const FOREGROUND_529_RETRY_SOURCES = new Set<QuerySource>([
  'repl_main_thread',    // 用户主线程——用户在等待，必须重试
  'sdk',
  'agent:default',
  'compact',
  'auto_mode',
  // ... 其他用户可见的来源
])

// 后台请求（标题生成、分析等）遇到 529 直接放弃，不重试
// 理由：容量级联时每次重试是 3-10× 网关放大
function shouldRetry529(querySource) {
  return querySource === undefined || FOREGROUND_529_RETRY_SOURCES.has(querySource)
}

export async function* withRetry(getClient, operation, options) {
  let consecutive529Errors = options.initialConsecutive529Errors ?? 0

  for (let attempt = 1; attempt <= maxRetries + 1; attempt++) {
    try {
      // 懒加载客户端：401/403/ECONNRESET 触发重建
      if (client === null || lastError instanceof APIError && lastError.status === 401) {
        if (lastError?.status === 401) await handleOAuth401Error(failedAccessToken)
        client = await getClient()
      }
      return await operation(client, attempt, retryContext)
    } catch (error) {
      // Fast Mode 降速：短 retry-after 等待继续，长延迟则触发 cooldown 切标准速
      if (wasFastModeActive && error.status === 429) {
        const retryAfterMs = getRetryAfterMs(error)
        if (retryAfterMs < SHORT_RETRY_THRESHOLD_MS) {
          await sleep(retryAfterMs); continue  // 保持 fast mode 继续
        }
        triggerFastModeCooldown(Date.now() + cooldownMs, 'rate_limit')
        retryContext.fastMode = false; continue
      }

      // Opus 连续 529 → 触发 fallback 到 Sonnet
      if (is529Error(error) && ++consecutive529Errors >= MAX_529_RETRIES) {
        if (options.fallbackModel) throw new FallbackTriggeredError(options.model, options.fallbackModel)
      }

      // 上下文溢出：自动缩减 max_tokens
      const overflowData = parseMaxTokensContextOverflowError(error)
      if (overflowData) {
        retryContext.maxTokensOverride = Math.max(FLOOR_OUTPUT_TOKENS, contextLimit - inputTokens - 1000)
        continue
      }

      // 指数退避（500ms 起，最大 32s，+25% jitter）
      const delayMs = getRetryDelay(attempt, getRetryAfter(error))
      yield createSystemAPIErrorMessage(error, delayMs, attempt, maxRetries)  // UI 实时显示重试状态
      await sleep(delayMs, options.signal)
    }
  }
}
```

### 5.4 错误分类体系（errors.ts）

```typescript
// src/services/api/errors.ts
export function getAssistantMessageFromError(error, model, options?): AssistantMessage {
  // 超时
  if (error instanceof APIConnectionTimeoutError) {
    return createAssistantAPIErrorMessage({ content: API_TIMEOUT_ERROR_MESSAGE })
  }
  // 429 限流：解析 anthropic-ratelimit-unified-* 响应头
  if (error instanceof APIError && error.status === 429 && isClaudeAISubscriber()) {
    const rateLimitType = error.headers?.get('anthropic-ratelimit-unified-representative-claim')
    const limits = { status: 'rejected', rateLimitType, resetsAt: ... }
    return createAssistantAPIErrorMessage({ content: getRateLimitErrorMessage(limits, model) })
  }
  // Prompt 过长：提取 token 数到 errorDetails（reactive compact 会读取计算缩减量）
  if (error.message.includes('prompt is too long')) {
    return createAssistantAPIErrorMessage({
      content: PROMPT_TOO_LONG_ERROR_MESSAGE,
      error: 'invalid_request',
      errorDetails: error.message,  // 包含 "137500 tokens > 135000 maximum"
    })
  }
  // OAuth token 被撤销
  if (error instanceof APIError && error.status === 403 && error.message.includes('OAuth token has been revoked')) {
    return createAssistantAPIErrorMessage({ content: TOKEN_REVOKED_ERROR_MESSAGE, error: 'authentication_failed' })
  }
  // ... 20+ 种错误类型的精确匹配
}

// 用于 Analytics 标签的错误分类
export function classifyAPIError(error): string {
  if (is529Error(error)) return 'server_overload'
  if (error.status === 429) return 'rate_limit'
  if (error.status === 401 || error.status === 403) return 'auth_error'
  if (error.message.includes('prompt is too long')) return 'prompt_too_long'
  // ...
  return 'unknown'
}
```

### 5.5 Prompt Cache 控制（claude.ts）

```typescript
// src/services/api/claude.ts
export function getCacheControl({ scope, querySource } = {}) {
  return {
    type: 'ephemeral',
    ...(should1hCacheTTL(querySource) && { ttl: '1h' }),  // 订阅用户可升级到 1 小时缓存
    ...(scope === 'global' && { scope }),                   // global scope 跨请求共享
  }
}

// 1h TTL 资格判断（会话级锁定，防止中途变化破坏缓存）
function should1hCacheTTL(querySource) {
  // Bedrock 用户通过环境变量单独开启
  if (getAPIProvider() === 'bedrock' && isEnvTruthy(ENABLE_PROMPT_CACHING_1H_BEDROCK)) return true

  // 首次判断后锁定到 session state，防止 GrowthBook 远程配置变化导致
  // 同一会话中混用 ephemeral 和 1h TTL（会使服务端缓存失效）
  let userEligible = getPromptCache1hEligible()
  if (userEligible === null) {
    userEligible = process.env.USER_TYPE === 'ant' || (isClaudeAISubscriber() && !currentLimits.isUsingOverage)
    setPromptCache1hEligible(userEligible)
  }
  if (!userEligible) return false

  // 从 GrowthBook 读取允许使用 1h TTL 的 querySource 白名单（支持通配符）
  const allowlist = getPromptCache1hAllowlist() ?? fetchAndCacheAllowlist()
  return allowlist.some(pattern =>
    pattern.endsWith('*') ? querySource.startsWith(pattern.slice(0, -1)) : querySource === pattern
  )
}
```

---

## 6. 设计思路总结

### 6.1 多云统一抽象
`getAnthropicClient()` 通过同一个 `ARGS` 基础配置 + 分支 spread 的方式，将四种部署模式（直连/Bedrock/Vertex/Foundry）的差异封装在工厂函数内，调用方完全不感知底层 SDK 差异。所有客户端都伪装成 `Anthropic` 类型（注释里直接写 "we have always been lying about the return type"）。

### 6.2 前台/后台重试分离
`FOREGROUND_529_RETRY_SOURCES` 白名单明确区分"用户在等待"和"后台静默任务"。这是为了避免在容量级联（服务过载）时，后台重试进一步放大压力。这种设计在分布式系统中很常见，但在客户端工具里能看到如此清晰的实现比较罕见。

### 6.3 错误上下文保存
`getAssistantMessageFromError()` 不只返回用户友好的消息文本，还通过 `errorDetails` 字段保存原始 API 错误字符串（含 token 数量）。下游的 reactive compact（自动压缩对话历史）通过 `getPromptTooLongTokenGap()` 解析这个字段，一次性跳过足够多的历史消息，而不是一条一条地试探。

### 6.4 Session 稳定性设计
Prompt Cache 的 1h TTL 资格在首次请求时锁定到 session state，之后不再重新计算。这避免了 GrowthBook 远程配置在会话中途更新时，导致 `cache_control.ttl` 发生变化，使服务器端缓存失效（浪费约 20K 缓存 token）。

### 6.5 x-client-request-id 设计
在 `buildFetch()` 包装层而非 SDK 层注入客户端请求 ID，是因为 SDK 的请求 ID 来自服务端响应头，超时场景下没有响应就没有 ID。客户端自生成的 UUID 在任何情况下都存在，方便 API 团队在服务端日志中关联超时请求。

---

## 7. 与其他模块的联系

```
claude.ts (查询入口)
  ├── 依赖 client.ts → getAnthropicClient()
  ├── 依赖 withRetry.ts → withRetry() (重试循环)
  ├── 依赖 errors.ts → getAssistantMessageFromError()
  ├── 依赖 logging.ts → logAPIQuery/Error/SuccessAndDuration()
  │
  ├── 被 query.ts 调用 → queryModelWithStreaming/WithoutStreaming()
  ├── 被 QueryEngine.ts 调用 → 同上
  └── 被 tools/AgentTool 调用 → subagent 发起嵌套 API 请求

errors.ts
  ├── 被 claude.ts 调用 → 流式/非流式错误处理
  └── 为 reactive compact 提供 getPromptTooLongTokenGap() → services/compact/

usage.ts
  └── 被 UI 组件调用 → 显示用量进度条（claude.ai 订阅用户）

withRetry.ts (AsyncGenerator)
  └── yield SystemAPIErrorMessage → QueryEngine → UI 实时显示 "正在重试 (2/10)..."
```
