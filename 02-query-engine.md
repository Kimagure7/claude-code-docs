# 02 - 核心查询引擎 (Query Engine)

## 1. 模块职责

核心查询引擎是整个 Claude Code 系统的心脏，负责驱动 **Agent 对话循环**：接收用户输入、调用 AI 模型、执行工具、处理错误恢复，并产出所有中间状态事件。

涉及核心文件：
- `query.ts` — 主循环逻辑（1,729 行）
- `QueryEngine.ts` — 会话管理封装类（1,295 行）
- `services/api/claude.ts` — Claude API 调用与流式处理

---

## 2. 核心文件地图

| 文件 | 行数 | 一句话描述 |
|------|------|-----------|
| `query.ts` | 1,729 | `query()` 异步生成器主函数 + `queryLoop()` 循环体，含压缩、工具执行、错误恢复 |
| `QueryEngine.ts` | 1,295 | `QueryEngine` 类，管理会话状态（消息历史、用量统计），提供 `submitMessage()` SDK 接口 |
| `services/api/claude.ts` | 约 3,420 | `queryModel()` 封装 Anthropic API 流式调用，含重试、缓存、Thinking 配置 |
| `context.ts` | 189 | `getSystemContext()` / `getUserContext()` 构建 Git 状态和内存上下文 |
| `types/message.ts` | — | `Message` 消息类型族定义（UserMessage、AssistantMessage、SystemMessage 等） |

---

## 3. 关键数据结构

### 3.1 QueryParams — 查询入参

```typescript
// query.ts
export type QueryParams = {
  messages: Message[]                      // 当前对话消息历史
  systemPrompt: SystemPrompt               // 系统提示（数组形式）
  userContext: { [k: string]: string }     // 用户上下文（内存文件、日期）
  systemContext: { [k: string]: string }   // 系统上下文（Git 状态）
  canUseTool: CanUseToolFn                 // 权限检查函数
  toolUseContext: ToolUseContext           // 工具执行上下文（含 AbortController）
  fallbackModel?: string                   // 降级模型
  querySource: QuerySource                 // 来源标识：repl_main_thread / sdk / agent:*
  maxOutputTokensOverride?: number         // 输出 token 上限覆盖
  maxTurns?: number                        // 最大循环轮次
  skipCacheWrite?: boolean                 // 是否跳过 cache 写入
  taskBudget?: { total: number }           // token 预算限制
  deps?: QueryDeps                         // 依赖注入（microcompact/autocompact 等）
}
```

### 3.2 Message 类型族

```typescript
// types/message.ts
export type Message =
  | UserMessage
  | AssistantMessage
  | SystemMessage
  | AttachmentMessage
  | ProgressMessage
  | ToolUseSummaryMessage
  | RequestStartEvent

export type UserMessage = {
  type: 'user'
  uuid: string
  timestamp: number
  message: {
    role: 'user'
    content: ContentBlockParam[]   // TextBlock | ImageBlock | ToolResultBlock
  }
  isMeta?: boolean                 // 元信息标记（不展示给用户）
  toolUseResult?: string
}

export type AssistantMessage = {
  type: 'assistant'
  uuid: string
  timestamp: number
  message: BetaMessage & {
    content: (BetaContentBlock | ConnectorTextBlock)[]
    usage?: Usage
    stop_reason?: BetaStopReason
  }
  isApiErrorMessage?: boolean
  apiError?: string
}

export type SystemMessage = {
  type: 'system'
  uuid: string
  timestamp: number
  subtype:
    | 'compact_boundary'   // 压缩边界标记
    | 'api_error'          // API 错误
    | 'api_retry'          // 重试
    | 'informational'      // 信息通知
  compactMetadata?: CompactBoundaryMetadata
  retryAttempt?: number
  maxRetries?: number
  retryInMs?: number
  error?: APIError
}
```

### 3.3 QueryEngineConfig — 会话配置

```typescript
// QueryEngine.ts:130-179（完整类型定义）
export type QueryEngineConfig = {
  cwd: string                          // 工作目录
  tools: Tools                         // 工具列表
  commands: Command[]                  // 斜杠命令
  mcpClients: MCPServerConnection[]    // MCP 连接
  agents: AgentDefinition[]            // Agent 定义（子 Agent）
  canUseTool: CanUseToolFn             // 权限检查
  getAppState: () => AppState
  setAppState: (f: (prev: AppState) => AppState) => void
  initialMessages?: Message[]          // 初始消息（用于恢复会话）
  readFileCache: FileStateCache        // 文件状态缓存（跨 turn 追踪已读文件）
  customSystemPrompt?: string
  appendSystemPrompt?: string
  userSpecifiedModel?: string
  fallbackModel?: string
  thinkingConfig?: ThinkingConfig      // Extended Thinking 配置
  maxTurns?: number
  maxBudgetUsd?: number                // USD 费用预算
  taskBudget?: { total: number }       // token 预算
  jsonSchema?: Record<string, unknown> // 结构化输出 Schema
  verbose?: boolean
  replayUserMessages?: boolean         // 是否重放用户消息（SDK 会话恢复）
  /** Handler for URL elicitations triggered by MCP tool -32042 errors. */
  handleElicitation?: ToolUseContext['handleElicitation']  // MCP URL 弹出处理
  includePartialMessages?: boolean     // 是否在流中包含部分消息
  setSDKStatus?: (status: SDKStatus) => void  // SDK 状态回调
  abortController?: AbortController   // 外部传入的取消控制器
  orphanedPermission?: OrphanedPermission  // 孤立权限恢复（重连场景）
  /**
   * Snip-boundary handler: receives each yielded system message plus the
   * current mutableMessages store. SDK-only: QueryEngine truncates here to
   * bound memory in long headless sessions (no UI to preserve).
   * Returns undefined if not a snip boundary, otherwise returns replayed result.
   */
  snipReplay?: (
    yieldedSystemMsg: Message,
    store: Message[],
  ) => { messages: Message[]; executed: boolean } | undefined
}
```

**字段说明**：
- `readFileCache`：跨 turn 追踪哪些文件被读取过，用于文件历史快照和变更检测
- `handleElicitation`：MCP 服务器返回 -32042 错误码（需要 URL 验证）时的处理回调，仅 SDK 路径使用
- `replayUserMessages`：SDK 恢复会话时，将历史消息重放给 Claude 以恢复上下文
- `snipReplay`：HISTORY_SNIP feature 的注入点，保持 QueryEngine 本身不含 feature-gated 字符串，使之在 `bun test` 下可测试

### 3.4 循环终止原因 Terminal

```typescript
type Terminal =
  | { reason: 'completed' }
  | { reason: 'aborted_streaming' }      // 用户中止（流进行中）
  | { reason: 'aborted_tools' }          // 用户中止（工具执行中）
  | { reason: 'prompt_too_long' }        // 413 上下文超长
  | { reason: 'image_error' }            // 媒体大小超限
  | { reason: 'model_error' }            // 模型调用失败
  | { reason: 'stop_hook_prevented' }    // Stop Hook 阻止继续
  | { reason: 'hook_stopped' }           // Hook 主动停止

type Transition =                        // 继续循环的原因（内部状态）
  | { reason: 'collapse_drain_retry' }
  | { reason: 'reactive_compact_retry' }
  | { reason: 'max_output_tokens_escalate' }
  | { reason: 'max_output_tokens_recovery'; attempt: number }
  | { reason: 'stop_hook_blocking' }
  | { reason: 'token_budget_continuation' }
```

---

## 4. 执行流程图

```
用户输入 (string | ContentBlockParam[])
        ↓
  [QueryEngine.submitMessage()]
        ↓
  processUserInput()  ← 处理斜杠命令、构建 UserMessage
        ↓
  fetchSystemPromptParts()  ← 获取系统提示、用户上下文
        ↓
  ┌─────────────────────────────────────┐
  │         query() 异步生成器           │
  │                                     │
  │  ┌──── queryLoop() while(true) ─────┤
  │  │                                  │
  │  │  1. 消息压缩处理                  │
  │  │     snip → microcompact          │
  │  │     → contextCollapse            │
  │  │     → autocompact                │
  │  │                                  │
  │  │  2. 调用 AI 模型                  │
  │  │     queryModel() 流式处理         │
  │  │     → yield StreamEvent[]        │
  │  │     → yield AssistantMessage     │
  │  │                                  │
  │  │  3. 检查 needsFollowUp           │
  │  │     ├─ No: 错误恢复/停止检查 →   │
  │  │     │    return Terminal         │
  │  │     └─ Yes: 执行工具 ↓          │
  │  │                                  │
  │  │  4. 工具执行                      │
  │  │     runTools() 或               │
  │  │     StreamingToolExecutor        │
  │  │     → yield ToolResultMessage    │
  │  │                                  │
  │  │  5. 收集附件 (内存、编译错误)      │
  │  │                                  │
  │  │  6. 更新 state → continue        │
  │  └──────────────────────────────────┘
  └─────────────────────────────────────┘
        ↓
  返回 Terminal { reason: 'completed' | ... }
```

---

## 5. 关键代码解析

### 5.1 query() 函数签名 — 异步生成器入口

```typescript
// query.ts 第 156-225 行

// query() 是一个异步生成器函数
// 它 yield 所有中间事件供 UI 层消费，最终 return Terminal
export async function* query(
  params: QueryParams,
): AsyncGenerator<
  | StreamEvent          // 流式 API 事件
  | RequestStartEvent    // 请求开始
  | Message              // 各类消息
  | TombstoneMessage
  | ToolUseSummaryMessage,
  Terminal               // 最终返回值
> {
  const consumedCommandUuids: string[] = []

  // 委托给真正的循环函数，使用 yield* 透传所有事件
  const terminal = yield* queryLoop(params, consumedCommandUuids)

  // 仅在正常完成时执行（中止时不会到达此处）
  for (const uuid of consumedCommandUuids) {
    notifyCommandLifecycle(uuid, 'completed')
  }
  return terminal
}
```

**设计要点**：`yield*` 将生成器委托给 `queryLoop()`，让整个调用链成为可组合的异步流。消费方（UI 层或 SDK）只需 `for await (const event of query(...))` 即可处理所有事件。

### 5.2 queryLoop() — 主循环骨架

```typescript
// query.ts 第 376-550 行（简化）

async function* queryLoop(
  params: QueryParams,
  consumedCommandUuids: string[],
): AsyncGenerator<...> {

  // 可变循环状态
  let state: State = {
    messages: params.messages,
    toolUseContext: params.toolUseContext,
    maxOutputTokensOverride: params.maxOutputTokensOverride,
    turnCount: 1,
    transition: undefined,    // undefined = 首次迭代
  }

  while (true) {
    let { toolUseContext } = state
    const { messages, turnCount } = state

    // === 1. 消息压缩（分层策略）===

    // 历史截断（HISTORY_SNIP feature flag 控制）
    if (feature('HISTORY_SNIP')) {
      const snipResult = snipModule!.snipCompactIfNeeded(messagesForQuery)
      messagesForQuery = snipResult.messages
    }

    // 微压缩（缓存编辑优化）
    queryCheckpoint('query_microcompact_start')
    messagesForQuery = (await deps.microcompact(...)).messages
    queryCheckpoint('query_microcompact_end')

    // 上下文折叠（CONTEXT_COLLAPSE feature flag）
    if (feature('CONTEXT_COLLAPSE') && contextCollapse) {
      messagesForQuery = (await contextCollapse.applyCollapsesIfNeeded(...)).messages
    }

    // 自动压缩（token 预算超出时触发摘要压缩）
    queryCheckpoint('query_autocompact_start')
    const { compactionResult } = await deps.autocompact(messagesForQuery, ...)
    queryCheckpoint('query_autocompact_end')

    // === 2. 调用 AI 模型 ===

    let assistantMessages: AssistantMessage[] = []
    let needsFollowUp = false
    let toolUseBlocks: BetaToolUseBlock[] = []
    let toolResults: Message[] = []

    for await (const event of queryModelWithStreaming(...)) {
      if (event.type === 'assistant') {
        assistantMessages.push(event)

        // 检测是否包含 tool_use 块
        const blocks = event.message.content.filter(b => b.type === 'tool_use')
        toolUseBlocks.push(...blocks)
        if (toolUseBlocks.length > 0) needsFollowUp = true

        yield event
      }
      // ... 其他 StreamEvent 直接 yield 给消费方
    }

    // === 3. 无工具调用 → 退出或错误恢复 ===

    if (!needsFollowUp) {
      // 检查 max_output_tokens 超限
      if (isWithheldMaxOutputTokens(lastMessage)) {
        // 详见 5.4 错误恢复逻辑
      }

      // 执行 Stop Hooks
      const stopHookResult = yield* handleStopHooks(...)
      if (stopHookResult.preventContinuation) {
        return { reason: 'stop_hook_prevented' }
      }

      return { reason: 'completed' }  // 正常完成
    }

    // === 4. 执行工具 ===

    queryCheckpoint('query_tool_execution_start')

    const toolUpdates = streamingToolExecutor
      ? streamingToolExecutor.getRemainingResults()
      : runTools(toolUseBlocks, assistantMessages, canUseTool, toolUseContext)

    for await (const update of toolUpdates) {
      if (update.message) {
        yield update.message
        toolResults.push(...)
      }
      if (update.newContext) {
        updatedToolUseContext = update.newContext
      }
    }

    queryCheckpoint('query_tool_execution_end')

    // === 5. 收集附件 + 准备下一轮 ===

    for await (const attachment of getAttachmentMessages(...)) {
      yield attachment
      toolResults.push(attachment)
    }

    // 更新状态，触发下一次迭代
    state = {
      messages: [...messagesForQuery, ...assistantMessages, ...toolResults],
      toolUseContext: updatedToolUseContext,
      turnCount: turnCount + 1,
      transition: undefined,
    }
    // while(true) 继续下一轮
  }
}
```

### 5.3 Claude API 调用层 — 流式处理

```typescript
// services/api/claude.ts 第 400-600 行（简化）

async function* queryModel(
  messages: Message[],
  systemPrompt: SystemPrompt,
  thinkingConfig: ThinkingConfig,
  tools: Tools,
  signal: AbortSignal,
  options: Options,
): AsyncGenerator<StreamEvent | AssistantMessage | SystemAPIErrorMessage> {

  // 构建 API 请求参数
  const params = {
    model: normalizeModelStringForAPI(options.model),
    messages: addCacheBreakpoints(messagesForAPI, enablePromptCaching, ...),
    system: systemPrompt,
    tools: allTools,
    max_tokens: maxOutputTokens,
    thinking: thinkingConfig,      // Extended Thinking 配置
    betas: betasParams,            // Beta feature 头部
  }

  // 发起流式请求
  stream = await anthropic.beta.messages.stream(params)

  // 处理流事件
  for await (const event of stream) {
    if (event.type === 'message_start') {
      usage = updateUsage(EMPTY_USAGE, event.message.usage)
      yield { type: 'stream_event', event }
    }

    if (event.type === 'content_block_start') {
      contentBlocks.push(event.content_block)
      yield { type: 'stream_event', event }
    }

    if (event.type === 'content_block_delta') {
      // 累积文本 delta
      if (event.delta.type === 'text_delta') {
        contentBlocks[event.index].text += event.delta.text
      }
      yield { type: 'stream_event', event }
    }

    if (event.type === 'message_stop') {
      yield { type: 'stream_event', event }
    }
  }

  // 流结束后，构建完整的 AssistantMessage 产出
  yield {
    type: 'assistant',
    uuid: generateUUID(),
    timestamp: Date.now(),
    message: {
      ...partialMessage,
      content: contentBlocks,
      usage,
      stop_reason: stopReason,
    },
  }
}
```

**设计要点**：每个 `content_block_delta` 事件都立刻 yield，UI 层可以实时渲染增量文本，不需要等待整个响应。

### 5.4 错误恢复机制

```typescript
// query.ts — 三层恢复策略

// === 场景 1：max_output_tokens 超限 ===
if (isWithheldMaxOutputTokens(lastMessage)) {

  // 恢复 1：升级 token 上限（escalate to 64k）
  if (capEnabled && maxOutputTokensOverride === undefined) {
    state = { ...state, maxOutputTokensOverride: ESCALATED_MAX_TOKENS,
              transition: { reason: 'max_output_tokens_escalate' } }
    continue  // 重新循环
  }

  // 恢复 2：发送恢复消息（最多重试 3 次）
  if (maxOutputTokensRecoveryCount < MAX_OUTPUT_TOKENS_RECOVERY_LIMIT) {
    const recoveryMessage = createUserMessage({
      content: 'Output token limit hit. Resume directly ...',
      isMeta: true,  // 不展示给用户
    })
    state = {
      messages: [...messagesForQuery, ...assistantMessages, recoveryMessage],
      maxOutputTokensRecoveryCount: maxOutputTokensRecoveryCount + 1,
      transition: { reason: 'max_output_tokens_recovery', attempt: ... },
    }
    continue
  }
}

// === 场景 2：Prompt 过长 (413 错误) ===
if (isWithheld413) {

  // 尝试 1：上下文折叠（把历史对话压缩掉）
  if (contextCollapse && state.transition?.reason !== 'collapse_drain_retry') {
    const drained = contextCollapse.recoverFromOverflow(...)
    if (drained.committed > 0) {
      state = { ...state, transition: { reason: 'collapse_drain_retry' } }
      continue
    }
  }

  // 尝试 2：反应式压缩（全面摘要压缩）
  if (reactiveCompact) {
    const compacted = await reactiveCompact.tryReactiveCompact(...)
    if (compacted) {
      state = { ...state, transition: { reason: 'reactive_compact_retry' } }
      continue
    }
  }

  // 全部失败 → 返回错误
  return { reason: 'prompt_too_long' }
}

// === 场景 3：用户中止（AbortController 触发）===
if (toolUseContext.abortController.signal.aborted) {

  // 确保工具 use/result 配对完整（API 格式要求）
  yield* yieldMissingToolResultBlocks(assistantMessages, 'Interrupted by user')

  return { reason: 'aborted_streaming' }
}
```

### 5.5 QueryEngine.submitMessage() — SDK 接口

```typescript
// QueryEngine.ts 第 162-350 行（简化）

async *submitMessage(
  prompt: string | ContentBlockParam[],
  options?: { uuid?: string; isMeta?: boolean },
): AsyncGenerator<SDKMessage, void, unknown> {

  // 1. 获取系统提示各部分
  const { defaultSystemPrompt, userContext, systemContext } =
    await fetchSystemPromptParts({ tools, mcpClients, ... })

  // 2. 处理用户输入（可能是斜杠命令）
  const { messages: messagesFromUserInput, shouldQuery } =
    await processUserInput({
      input: prompt,
      mode: 'prompt',
      messages: this.mutableMessages,
      querySource: 'sdk',
    })

  // 3. 追加到可变消息历史
  this.mutableMessages.push(...messagesFromUserInput)

  // 4. 进入 query() 循环
  const queryGenerator = query({
    messages: [...this.mutableMessages],
    systemPrompt,
    userContext,
    systemContext,
    canUseTool: wrappedCanUseTool,  // 包装了权限拒绝统计
    toolUseContext,
    querySource: 'sdk',
    maxTurns: this.config.maxTurns,
  })

  // 5. 消费生成器，转换为 SDK 格式并 yield
  for await (const message of queryGenerator) {
    // 累积 token 使用量统计
    if (message.type === 'stream_event' && message.event.type === 'message_stop') {
      this.totalUsage = accumulateUsage(this.totalUsage, currentMessageUsage)
    }
    // 转换为 SDK 消息格式并产出
    yield convertToSDKMessage(message)
  }
}
```

---

## 6. 设计思路总结

### 异步生成器作为事件总线

`query()` 不是普通的 async 函数，而是 `AsyncGenerator`。这带来几个优势：
- **惰性求值**：消费方（UI）控制消费速度，无需内部维护缓冲队列
- **背压支持**：如果 UI 渲染慢，生成器自然暂停
- **组合性**：`yield*` 可以无缝委托给子生成器（`queryLoop`、`handleStopHooks` 等）

### 状态机式循环

`while(true) + state 对象 + continue` 形成一个隐式状态机。每次迭代可以通过修改 `state.transition` 来记录"为什么进入下一轮"，方便日志分析和 telemetry 上报。

### 分层压缩策略

上下文压缩分为四层，按代价从低到高依次尝试：
1. **历史截断 (snip)**：直接删除早期消息
2. **微压缩 (microcompact)**：优化缓存编辑格式
3. **上下文折叠 (collapse)**：把大块内容替换为摘要
4. **自动压缩 (autocompact)**：让 AI 生成完整对话摘要

### 工具执行两路径

- `StreamingToolExecutor`：并发执行多个工具，流式返回结果
- `runTools()`：串行同步执行（作为降级路径）

---

## 7. 与其他模块的联系

```
query.ts
  ├── 依赖 → services/api/claude.ts      (queryModel 调用)
  ├── 依赖 → tools/ (通过 runTools)       (工具执行)
  ├── 依赖 → services/mcp/              (MCP 工具执行)
  ├── 依赖 → utils/permissions/         (canUseTool 权限检查)
  ├── 依赖 → context.ts                  (getSystemContext / getUserContext)
  ├── 被调用 → QueryEngine.ts            (submitMessage 封装)
  ├── 被调用 → entrypoints/init.ts       (REPL 模式入口)
  └── 被调用 → tools/AgentTool           (子 Agent 递归调用)
```
