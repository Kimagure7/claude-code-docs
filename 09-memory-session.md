# 09 - 记忆与会话管理

## 1. 模块职责

负责在 Claude Code 中实现对话上下文的持久化、压缩与长期记忆管理，解决 LLM 上下文窗口有限而用户会话无限增长的核心矛盾。

---

## 2. 核心文件地图

| 文件 | 行数 | 职责 |
|------|------|------|
| `services/compact/compact.ts` | 1705 | 核心压缩引擎，API 调用生成摘要 |
| `services/compact/autoCompact.ts` | 351 | 自动压缩触发与阈值管理 |
| `services/compact/sessionMemoryCompact.ts` | 630 | 基于会话记忆的轻量压缩路径 |
| `services/compact/microCompact.ts` | 530 | 微压缩，清理工具结果无需 API 调用 |
| `services/compact/grouping.ts` | 63 | 按 API 轮次分组消息 |
| `services/compact/prompt.ts` | 374 | 压缩提示模板 |
| `services/compact/apiMicrocompact.ts` | 153 | API 调用式微压缩（缓存编辑优化） |
| `services/compact/postCompactCleanup.ts` | 77 | 压缩后清理：readFileState 重置等 |
| `services/compact/timeBasedMCConfig.ts` | 43 | 时间触发的微压缩配置 |
| `services/compact/compactWarningHook.ts` | 16 | 压缩警告 Hook |
| `services/compact/compactWarningState.ts` | 18 | 压缩警告状态管理 |
| `services/SessionMemory/sessionMemory.ts` | 495 | 记忆提取控制器，后台 agent |
| `services/SessionMemory/sessionMemoryUtils.ts` | 207 | 记忆状态工具函数 |
| `services/SessionMemory/prompts.ts` | 324 | 记忆提取提示模板 |
| `memdir/memdir.ts` | 507 | 记忆文件目录管理 |
| `memdir/memoryTypes.ts` | 271 | 记忆分类系统定义 |
| `memdir/memoryScan.ts` | 94 | 记忆文件扫描与列表 |
| `memdir/findRelevantMemories.ts` | 141 | 相关记忆检索 |
| `memdir/memoryAge.ts` | 53 | 记忆新鲜度计算 |
| `memdir/paths.ts` | 278 | 记忆路径解析与安全验证 |
| `utils/sessionStorage.ts` | 5105 | 消息持久化与转录读写 |
| `utils/sessionState.ts` | 151 | 会话状态管理 |
| `utils/teamMemoryOps.ts` | 89 | 团队记忆操作 |

---

## 3. 关键数据结构

### CompactionResult

```typescript
// services/compact/compact.ts
interface CompactionResult {
  boundaryMarker: SystemMessage           // 压缩边界标记消息
  summaryMessages: UserMessage[]          // 摘要内容消息数组
  attachments: AttachmentMessage[]        // 重建的附件（文件/工具/技能）
  hookResults: HookResultMessage[]        // Hook 执行结果
  messagesToKeep?: Message[]              // 保留的原始消息（SM 压缩路径专用）
  userDisplayMessage?: string             // 显示给用户的提示信息
  preCompactTokenCount?: number           // 压缩前 token 数
  postCompactTokenCount?: number          // 压缩 API 调用消耗的 token 数
  truePostCompactTokenCount?: number      // 真实后处理 context token 数
  compactionUsage?: TokenUsageInfo        // 详细 token 用量
}
```

### SessionMemoryCompactConfig

```typescript
// services/compact/sessionMemoryCompact.ts
type SessionMemoryCompactConfig = {
  minTokens: number              // 触发压缩的最小保留 token 数（默认 10,000）
  minTextBlockMessages: number   // 最少保留的文本块消息数（默认 5）
  maxTokens: number              // 保留 token 上限（默认 40,000）
}
```

### SessionMemoryConfig

```typescript
// services/SessionMemory/sessionMemory.ts
type SessionMemoryConfig = {
  minimumMessageTokensToInit: number      // 初始化阈值（默认 10,000）
  minimumTokensBetweenUpdate: number      // 更新最小 token 增长（默认 5,000）
  toolCallsBetweenUpdates: number         // 触发更新的工具调用次数（默认 3）
}
```

### MemoryHeader

```typescript
// memdir/memoryScan.ts
type MemoryHeader = {
  filename: string              // 相对路径
  filePath: string              // 绝对路径
  mtimeMs: number               // 修改时间戳
  description: string | null    // 首行描述
  type: MemoryType | undefined  // 'user'|'feedback'|'project'|'reference'
}
```

### 关键常量

```typescript
// services/compact/autoCompact.ts
const AUTOCOMPACT_BUFFER_TOKENS = 13_000         // 触发压缩的 token 缓冲区
const WARNING_THRESHOLD_BUFFER_TOKENS = 20_000   // 警告阈值缓冲区
const ERROR_THRESHOLD_BUFFER_TOKENS = 20_000     // 错误阈值缓冲区
const MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3   // 电路断路器最大连续失败次数

// memdir/memdir.ts
const ENTRYPOINT_NAME = "MEMORY.md"
const MAX_ENTRYPOINT_LINES = 200
const MAX_ENTRYPOINT_BYTES = 25_000

// 微压缩可清理的工具（清理其结果以节省 token）
const COMPACTABLE_TOOLS = [
  FILE_READ_TOOL_NAME,
  SHELL_TOOL_NAMES,
  GREP_TOOL_NAME,
  GLOB_TOOL_NAME,
  WEB_SEARCH_TOOL_NAME,
  WEB_FETCH_TOOL_NAME,
  FILE_EDIT_TOOL_NAME,
  FILE_WRITE_TOOL_NAME,
]
```

---

## 4. 执行流程图

### 4.1 整体压缩决策树

```
收到查询 / 每轮结束
         │
         ▼
 autoCompactIfNeeded()
         │
    shouldAutoCompact()?
    ├─ No → 继续
    └─ Yes
         │
         ▼
 ┌──────────────────────────────────┐
 │   SessionMemory 路径 (优先尝试)   │
 │  shouldUseSessionMemoryCompaction?│
 │  ├─ No  →  Legacy 路径           │
 │  └─ Yes                          │
 │       ↓                          │
 │  trySessionMemoryCompaction()    │
 │  ├─ 成功 → 返回 CompactionResult  │
 │  └─ 失败 → 级联 Legacy 路径       │
 └──────────────────────────────────┘
         │
         ▼
 ┌──────────────────────────────────┐
 │   Legacy 路径 (API 调用)          │
 │   compactConversation()          │
 │   ├─ API 流式摘要                 │
 │   ├─ PTL 重试（最多 3 次）         │
 │   └─ 重建附件 + 边界标记          │
 └──────────────────────────────────┘
         │
         ▼
 将 CompactionResult 应用到 context
```

### 4.2 微压缩双路径

```
microcompactMessages()
         │
    ┌────┴────┐
    │         │
路径 1       路径 2
时间基础     缓存驱动
    │         │
距上次助手   模型支持
消息 > N 分钟  缓存编辑?
    │         │
清空旧工具   创建 cache_edits
结果内容     不修改消息
    │         │
    └────┬────┘
         │
  返回处理后消息
```

### 4.3 会话记忆提取触发

```
每轮结束
    │
    ▼
shouldExtractMemory(messages)
    │
    ├─ tokenCount < 10K → false（未初始化）
    │
    └─ tokenCount >= 10K（已初始化）
          │
          ├─ tokenGrowth < 5K → false（增长不足）
          │
          └─ tokenGrowth >= 5K
                │
                ├─ 工具调用 >= 3 → 触发提取
                │
                └─ 最后消息无工具调用 → 触发提取
                         │
                         ▼
                后台 forked agent 提取记忆
                写入 memory file
```

---

## 5. 关键代码解析

### 5.1 自动压缩阈值计算

```typescript
// services/compact/autoCompact.ts
function getAutoCompactThreshold(contextWindowSize: number): number {
  // 有效窗口 = 上下文窗口 - API 输出预留
  const effectiveWindow = getEffectiveContextWindowSize(contextWindowSize)
  // 触发阈值 = 有效窗口 - 缓冲区
  return effectiveWindow - AUTOCOMPACT_BUFFER_TOKENS
}

async function autoCompactIfNeeded(
  messages: Message[],
  context: AppContext,
  currentTokenCount: number,
): Promise<CompactionResult | null> {
  // 1. 电路断路器：连续失败超过上限则跳过
  if (consecutiveAutoCompactFailures >= MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES) {
    return null
  }
  // 2. 获取阈值并比较
  const threshold = getAutoCompactThreshold(context.contextWindowSize)
  if (currentTokenCount < threshold) {
    return null  // 未达阈值，无需压缩
  }
  // 3. 防递归：压缩过程中不触发二次压缩
  if (!shouldAutoCompact(context)) {
    return null
  }
  // 4. 优先尝试 SM 压缩，失败则级联 Legacy
  const smResult = await trySessionMemoryCompaction(messages, context, threshold)
  if (smResult) return smResult
  return await compactConversation(messages, context, { isAutoCompact: true })
}
```

### 5.2 保留消息范围计算

```typescript
// services/compact/sessionMemoryCompact.ts
function calculateMessagesToKeepIndex(
  messages: Message[],
  lastSummarizedIndex: number,
  config: SessionMemoryCompactConfig,
): number {
  // 从已摘要边界的下一条开始
  let startIndex = lastSummarizedIndex + 1

  // 计算当前保留范围的 token 和文本块数
  let tokenCount = estimateMessageTokens(messages.slice(startIndex))
  let textBlockCount = countTextBlockMessages(messages.slice(startIndex))

  // 超过最大 token，直接返回（不再扩展）
  if (tokenCount >= config.maxTokens) {
    return adjustIndexToPreserveAPIInvariants(messages, startIndex)
  }

  // 已满足最小要求，直接返回
  if (tokenCount >= config.minTokens && textBlockCount >= config.minTextBlockMessages) {
    return adjustIndexToPreserveAPIInvariants(messages, startIndex)
  }

  // 向前扩展，直到满足最小值或超过最大 token
  for (let i = startIndex - 1; i >= 0; i--) {
    const addedTokens = estimateMessageTokens([messages[i]])
    if (tokenCount + addedTokens > config.maxTokens) break

    startIndex = i
    tokenCount += addedTokens
    textBlockCount += hasTextBlocks(messages[i]) ? 1 : 0

    if (tokenCount >= config.minTokens && textBlockCount >= config.minTextBlockMessages) {
      break  // 满足最小值，停止扩展
    }
  }

  // 最终调整：确保不分割 tool_use/tool_result 对和 thinking 块
  return adjustIndexToPreserveAPIInvariants(messages, startIndex)
}
```

### 5.3 API 不变量保护

```typescript
// services/compact/sessionMemoryCompact.ts
function adjustIndexToPreserveAPIInvariants(
  messages: Message[],
  startIndex: number,
): number {
  let adjustedIndex = startIndex

  // 步骤 1：处理 tool_use / tool_result 配对
  // 收集保留范围内的所有 tool_result IDs
  const keptToolResultIds = new Set<string>()
  for (const msg of messages.slice(adjustedIndex)) {
    if (isToolResultMessage(msg)) keptToolResultIds.add(msg.toolUseId)
  }

  // 收集保留范围内已有的 tool_use IDs
  const keptToolUseIds = new Set<string>()
  for (const msg of messages.slice(adjustedIndex)) {
    if (isToolUseMessage(msg)) keptToolUseIds.add(msg.id)
  }

  // 找出缺失的 tool_use（tool_result 有对应 tool_use，但 tool_use 在切割线之前）
  const missingToolUseIds = new Set(
    [...keptToolResultIds].filter(id => !keptToolUseIds.has(id))
  )

  // 向前查找这些 tool_use 所在的 assistant 消息，调整边界
  if (missingToolUseIds.size > 0) {
    for (let i = adjustedIndex - 1; i >= 0; i--) {
      const msg = messages[i]
      if (isAssistantMessage(msg) && msgContainsToolUseId(msg, missingToolUseIds)) {
        adjustedIndex = i  // 边界前移到包含该 tool_use 的消息
        break
      }
    }
  }

  // 步骤 2：处理 thinking 块（同 message.id 不可分割）
  const keptMessageIds = new Set(
    messages.slice(adjustedIndex).map(m => m.messageId).filter(Boolean)
  )
  for (let i = adjustedIndex - 1; i >= 0; i--) {
    const msg = messages[i]
    if (msg.messageId && keptMessageIds.has(msg.messageId)) {
      adjustedIndex = i  // 边界前移到同组的最早消息
    } else {
      break
    }
  }

  return adjustedIndex
}
```

### 5.4 Legacy 压缩核心流程

```typescript
// services/compact/compact.ts
async function compactConversation(
  messages: Message[],
  context: AppContext,
  options: CompactOptions,
): Promise<CompactionResult> {
  const { isAutoCompact = false } = options
  const preCompactTokenCount = tokenCountWithEstimation(messages)

  // 1. 执行 PreCompact Hook
  await executePreCompactHooks(context)

  // 2. 构建压缩提示
  const compactPrompt = getCompactPrompt(context.customInstructions)
  const summaryRequest = createUserMessage(compactPrompt)

  // 3. 流式 API 调用（最多 3 次 PTL 重试）
  let messagesToSummarize = messages
  let summaryResponse: AssistantMessage | null = null

  for (let attempt = 0; attempt < 3; attempt++) {
    summaryResponse = await streamCompactSummary({
      messages: messagesToSummarize,
      summaryRequest,
      appState,
      context,
      preCompactTokenCount,
    })

    // PTL 错误：裁剪最老的 API 轮次后重试
    if (isPromptTooLongError(summaryResponse)) {
      messagesToSummarize = truncateHeadForPTLRetry(messagesToSummarize)
    } else {
      break
    }
  }

  const summary = getAssistantMessageText(summaryResponse!)

  // 4. 清理 context 中的缓存状态
  const preCompactReadFileState = cacheToObject(context.readFileState)
  context.readFileState.clear()
  context.loadedNestedMemoryPaths?.clear()

  // 5. 并行重建附件（文件、plan、技能等）
  const [fileAttachments, skillAttachment, planAttachment] = await Promise.all([
    createPostCompactFileAttachments(preCompactReadFileState, context),
    createSkillAttachmentIfNeeded(agentId),
    createPlanAttachmentIfNeeded(agentId),
  ])

  // 6. 运行 PostCompact Hook
  const hookMessages = await processSessionStartHooks('compact', { model })

  // 7. 构建边界标记
  const boundaryMarker = createCompactBoundaryMessage(
    isAutoCompact ? 'auto' : 'manual',
    preCompactTokenCount,
    messages.at(-1)?.uuid,
  )

  // 8. 构建摘要消息
  const summaryMessages = [
    createUserMessage({
      content: getCompactUserSummaryMessage(
        summary,
        /* suppressFollowUpQuestions */ false,
        transcriptPath,
      ),
      isCompactSummary: true,
      isVisibleInTranscriptOnly: true,
    }),
  ]

  // 9. 记录遥测
  logEvent('tengu_compact', {
    preCompactTokenCount,
    postCompactTokenCount: compactionCallTotalTokens,
    truePostCompactTokenCount,
    isAutoCompact,
    isRecompactionInChain,
    turnsSincePreviousCompact,
  })

  return {
    boundaryMarker,
    summaryMessages,
    attachments: fileAttachments,
    hookResults: hookMessages,
    preCompactTokenCount,
    postCompactTokenCount: compactionCallTotalTokens,
    truePostCompactTokenCount,
    compactionUsage,
  }
}
```

### 5.5 会话记忆提取触发条件

```typescript
// services/SessionMemory/sessionMemory.ts
function shouldExtractMemory(
  messages: Message[],
  config: SessionMemoryConfig,
): boolean {
  const currentTokenCount = tokenCountWithEstimation(messages)

  // 未初始化：检查是否达到初始化阈值
  if (!isSessionMemoryInitialized()) {
    if (currentTokenCount < config.minimumMessageTokensToInit) {
      return false  // token 数不够，还不需要记忆
    }
    markSessionMemoryInitialized()  // 标记为已初始化
  }

  // 计算自上次提取以来的 token 增长
  const tokenGrowth = currentTokenCount - getTokensAtLastExtraction()
  const hasMetTokenThreshold = tokenGrowth >= config.minimumTokensBetweenUpdate

  if (!hasMetTokenThreshold) {
    return false  // 增长不足，不触发
  }

  // 满足 token 阈值后，还需满足以下条件之一：
  // 1. 工具调用次数达到阈值
  const toolCallsSince = countToolCallsSince(messages, getLastMemoryMessageUuid())
  const hasMetToolCallThreshold = toolCallsSince >= config.toolCallsBetweenUpdates

  // 2. 最后一轮没有工具调用（纯对话轮次，适合提取）
  const hasToolCallsInLastTurn = hasToolCallsInLastAssistantTurn(messages)

  return hasMetToolCallThreshold || !hasToolCallsInLastTurn
}
```

### 5.6 记忆文件扫描

```typescript
// memdir/memoryScan.ts
async function scanMemoryFiles(memoryDir: string): Promise<MemoryHeader[]> {
  const files = await fs.readdir(memoryDir, { recursive: true })
  const mdFiles = files
    .filter(f => f.endsWith('.md'))
    .slice(0, 200)  // 最多 200 个文件

  const headers = await Promise.all(
    mdFiles.map(async (filename): Promise<MemoryHeader | null> => {
      const filePath = path.join(memoryDir, filename)
      const stat = await fs.stat(filePath)

      // 读取第一行作为描述
      const firstLine = await readFirstLine(filePath)
      // 解析 frontmatter 获取 type
      const type = await parseMemoryType(filePath)

      return {
        filename,
        filePath,
        mtimeMs: stat.mtimeMs,
        description: firstLine || null,
        type,
      }
    })
  )

  // 按修改时间降序排列（最新的在前）
  return headers
    .filter(Boolean)
    .sort((a, b) => b!.mtimeMs - a!.mtimeMs) as MemoryHeader[]
}

function formatMemoryManifest(headers: MemoryHeader[]): string {
  return headers
    .map(h => `[${h.type ?? 'unknown'}] ${h.filename} (${formatAge(h.mtimeMs)}): ${h.description ?? ''}`)
    .join('\n')
}
```

---

## 6. 设计思路总结

### 三层压缩架构

Claude Code 设计了三层压缩机制，按资源消耗从低到高递进：

```
微压缩 (microCompact)
  - 成本：零 API 调用
  - 方式：直接清除旧工具结果内容
  - 触发：时间间隔 or 缓存编辑
  - 效果：小幅减少 token

SM 压缩 (sessionMemoryCompact)
  - 成本：仅读取本地文件
  - 方式：以 memory file 替代历史消息摘要
  - 触发：SM 功能启用 + 达到阈值
  - 效果：大幅减少 token，无需新 API 调用

Legacy 压缩 (compactConversation)
  - 成本：一次完整 API 调用
  - 方式：让模型生成历史摘要
  - 触发：SM 失败或未启用
  - 效果：最彻底，但有延迟和成本
```

### SM 压缩的精妙之处

Session Memory Compact 将"记忆提取"与"压缩"解耦：
- **异步提取**：在每轮结束后，后台 forked agent 持续更新 memory file，用户无感知
- **同步压缩**：触发压缩时，直接读取已有 memory file，无需等待新 API 调用
- **降级策略**：如果 memory file 不存在或内容为空，自动级联到 Legacy 路径

### API 不变量保护

LLM API 有严格的消息格式要求：
- `tool_result` 必须紧跟对应的 `tool_use`
- 同一 `message.id` 的 thinking 块必须完整

`adjustIndexToPreserveAPIInvariants()` 确保切割后的消息序列始终满足这些约束，否则 API 会返回错误。

### 电路断路器模式

`autoCompact.ts` 实现了经典的电路断路器：连续失败超过 3 次后，自动跳过压缩。防止在 API 故障时反复浪费资源，同时允许自然恢复（下次会话重置计数器）。

### 记忆路径安全验证

`memdir/paths.ts` 的 `validateMemoryPath()` 对用户输入的路径做了严格验证：
- 防止路径遍历（`../../../etc/passwd`）
- 防止 UNC 路径（`\\server\share`）
- 防止 null 字节注入

---

## 7. 与其他模块的联系

```
query.ts / QueryEngine.ts
    │  每轮查询结束后触发自动压缩
    ↓
autoCompact.ts → compact.ts / sessionMemoryCompact.ts
                      │
                      ├── services/api/claude.ts  (Legacy: 流式 API 调用)
                      │
                      ├── SessionMemory/sessionMemory.ts  (SM: 读取 memory file)
                      │
                      └── utils/sessionStorage.ts  (写入压缩后转录)

SessionMemory/sessionMemory.ts
    │  触发后台 agent
    ↓
AgentTool/AgentTool.ts  (forked agent 提取记忆)
    │
    ↓
memdir/memdir.ts  (写入 MEMORY.md)

memdir/findRelevantMemories.ts
    │  每次对话开始时注入相关记忆
    ↓
query.ts  (作为 system prompt 的一部分)

utils/sessionState.ts
    │  通知外部（IDE 插件、状态栏）
    ↓
components/  (UI 显示 token 使用量和压缩状态)
```

---

## 8. 特征门与环境变量参考

| 标识符 | 类型 | 说明 |
|--------|------|------|
| `tengu_session_memory` | Feature flag | 启用记忆提取功能 |
| `tengu_sm_compact` | Feature flag | 启用 SM 压缩路径 |
| `DISABLE_AUTO_COMPACT=1` | 环境变量 | 禁用自动压缩 |
| `ENABLE_CLAUDE_CODE_SM_COMPACT=1` | 环境变量 | 强制启用 SM 压缩 |
| `CLAUDE_CODE_AUTO_COMPACT_WINDOW` | 环境变量 | 自定义上下文窗口大小 |
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` | 环境变量 | 百分比阈值覆盖（0-100）|
| `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1` | 环境变量 | 禁用所有记忆功能 |
