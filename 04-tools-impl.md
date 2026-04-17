# 04 - 核心工具实现细节

> 源码路径: `src/tools/` 各子目录  
> 本篇深入分析 FileReadTool、FileWriteTool、GlobTool、BashTool、WebFetchTool、TodoWriteTool 的实现细节

---

## 1. 模块职责

工具层是 Claude Code 实际执行能力的载体。每个工具封装一类操作（读文件、写文件、执行命令等），通过统一的 Tool 接口暴露给 Agent 循环调用。本篇关注"它们内部是怎么做的"，而不是接口本身（接口见 03-tool-system.md）。

---

## 2. 核心文件地图

```
src/tools/
├── FileReadTool/FileReadTool.ts    # ~1184行 读文件（支持文本/图片/PDF/Notebook）
├── FileWriteTool/FileWriteTool.ts  # ~435行  写文件（原子写+并发保护）
├── GlobTool/GlobTool.ts            # ~199行  文件模式匹配
├── BashTool/BashTool.tsx           # ~1144行 执行Shell命令（最复杂）
├── WebFetchTool/WebFetchTool.ts    # ~319行  抓取网页内容
└── TodoWriteTool/TodoWriteTool.ts  # ~116行  管理任务列表
```

---

## 3. FileReadTool — 多格式读取

### 3.1 inputSchema

```typescript
// src/tools/FileReadTool/FileReadTool.ts
const inputSchema = lazySchema(() =>
  z.strictObject({
    file_path: z.string().describe('The absolute path to the file to read'),
    offset: semanticNumber(z.number().int().nonnegative().optional())
      .describe('The line number to start reading from'),
    limit: semanticNumber(z.number().int().positive().optional())
      .describe('The number of lines to read'),
    pages: z.string().optional()
      .describe('Page range for PDF files (e.g., "1-5"). Only for PDF.'),
  }),
)
```

### 3.2 输出类型的 discriminatedUnion

FileReadTool 不只返回字符串——返回类型是一个联合，根据文件类型分支：

```typescript
// 输出类型（discriminatedUnion）
type FileReadOutput =
  | { type: 'text';         filePath: string; content: string; numLines: number; startLine: number; totalLines: number }
  | { type: 'image';        base64: string; mediaType: string; originalSize: number; dimensions?: ... }
  | { type: 'notebook';     filePath: string; cells: NotebookCell[] }
  | { type: 'pdf';          filePath: string; base64: string; originalSize: number }
  | { type: 'parts';        filePath: string; outputDir: string; count: number }  // 提取的PDF页
  | { type: 'file_unchanged'; filePath: string }  // dedup：文件未变化，跳过重发
```

### 3.3 call() 主流程

```typescript
async call({ file_path, offset = 1, limit, pages }, context) {
  // 1. Dedup：若相同文件+范围已读过且未变化，返回 file_unchanged
  //    避免在同一对话中重复发送大量相同内容给 Claude
  const unchanged = checkFileUnchanged(file_path, offset, limit, context)
  if (unchanged) return { type: 'file_unchanged', filePath: file_path }

  // 2. Skill 发现：读文件时顺便激活条件技能
  await discoverSkillDirsForPaths([file_path])
  await activateConditionalSkillsForPaths([file_path])

  const ext = path.extname(file_path).toLowerCase()

  // 3. 根据扩展名分支
  if (ext === '.ipynb') {
    const cells = await readNotebook(file_path)
    await validateContentTokens(JSON.stringify(cells), ext)
    return { type: 'notebook', filePath: file_path, cells }
  }

  if (IMAGE_EXTENSIONS.has(ext)) {
    // 读图片：自动压缩到 token budget 以内
    return readImageWithTokenBudget(file_path)
  }

  if (ext === '.pdf') {
    if (pages) {
      // 提取指定页面到临时目录
      return extractPDFPages(file_path, parsePDFPageRange(pages))
    }
    return readPDF(file_path)
  }

  // 默认：文本读取，支持 offset/limit 分页
  const result = await readFileInRange(file_path, offset, limit)
  await validateContentTokens(result.content, ext)
  return { type: 'text', ...result }
}
```

### 3.4 图片读取与 Token 预算

```typescript
export async function readImageWithTokenBudget(
  filePath: string,
  maxTokens = getDefaultFileReadingLimits().maxTokens,
): Promise<ImageResult> {
  const buf = await fs.readFile(filePath)        // 一次性读入内存
  
  // 第一次尝试：标准调整大小（保持比例缩放）
  let result = await maybeResizeAndDownsampleImageBuffer(buf)
  
  if (estimateImageTokens(result) > maxTokens) {
    // 超出 token 预算：激进压缩（JPEG + 降质量）
    result = await compressImageBufferWithTokenLimit(buf, maxTokens)
  }
  
  return {
    type: 'image',
    base64: result.toString('base64'),
    mediaType: detectMediaType(result),
    originalSize: buf.length,
  }
}
```

### 3.5 安全检查

```typescript
// validateInput() 中的安全过滤
const BLOCKED_DEVICE_PATHS = new Set([
  '/dev/zero', '/dev/random', '/dev/urandom', '/dev/full',  // 无限输出
  '/dev/stdin', '/dev/tty', '/dev/console',                  // 阻塞输入
  '/dev/stdout', '/dev/stderr',                              // 标准I/O
  '/dev/fd/0', '/dev/fd/1', '/dev/fd/2',                    // FD别名
])

// 1. UNC 路径检查（防止 NTLM 凭证泄露）
if (file_path.startsWith('\\\\')) throw new Error('UNC paths not allowed')

// 2. 设备路径检查（防止无限流 / 阻塞）
if (isBlockedDevicePath(file_path)) throw new Error('...')

// 3. 二进制扩展名检查（非图片/PDF的二进制文件直接拒绝）
if (BINARY_EXTENSIONS.has(ext) && !isViewable(ext)) throw new Error('...')
```

---

## 4. FileWriteTool — 原子写入与并发保护

### 4.1 outputSchema

```typescript
// src/tools/FileWriteTool/FileWriteTool.ts
const outputSchema = lazySchema(() =>
  z.object({
    type: z.enum(['create', 'update']),
    filePath: z.string(),
    content: z.string(),
    structuredPatch: z.array(hunkSchema()),  // 类似 git diff 的 hunk 结构
    originalFile: z.string().nullable(),
    gitDiff: gitDiffSchema().optional(),
  }),
)
```

### 4.2 并发安全：读-写时间戳验证

FileWriteTool 要求"先读后写"，且写入前检查文件没有被其他进程修改：

```typescript
// validateInput() 中的并发保护
async validateInput({ file_path, content }, { readFileState }) {
  // 1. 必须先调用 FileReadTool 读过这个文件
  const readTimestamp = readFileState.get(file_path)
  if (!readTimestamp) {
    throw new Error(`You must read the file before writing it`)
  }
  
  // 2. 检查文件修改时间：若读取后被其他进程修改，拒绝写入
  const stat = await fs.stat(file_path)
  if (stat.mtimeMs > readTimestamp.timestamp) {
    // fallback：对于全量读，做内容级别比较
    const currentContent = await fs.readFile(file_path, 'utf-8')
    if (currentContent !== readTimestamp.content) {
      throw new Error(`File has been modified since last read`)
    }
  }
}
```

### 4.3 call() 主流程

```typescript
async call({ file_path, content }, context) {
  // 1. 确保父目录存在
  await fs.mkdir(path.dirname(file_path), { recursive: true })

  // 2. LSP 诊断前快照
  diagnosticTracker.beforeFileEdited(file_path)

  // 3. 读取当前状态（原子操作的一部分）
  const currentContent = await readFileSyncWithMetadata(file_path)
  const isCreate = currentContent === null

  // 4. 原子写入（统一使用 LF 行尾）
  await writeTextContent(file_path, content)

  // 5. 通知 LSP（触发语言服务器重新分析）
  await lspManager.changeFile(file_path, content)
  await lspManager.saveFile(file_path)

  // 6. 通知 VS Code（更新 diff 视图）
  notifyVscodeFileUpdated(file_path)

  // 7. 更新 readFileState 时间戳（允许后续继续写入）
  context.updateFileHistoryState(file_path, content)

  // 8. 计算 git diff（可选，用于展示变化）
  const gitDiff = await computeGitDiff(file_path, currentContent, content)

  return {
    type: isCreate ? 'create' : 'update',
    filePath: file_path,
    content,
    structuredPatch: computeStructuredPatch(currentContent ?? '', content),
    originalFile: currentContent,
    gitDiff,
  }
}
```

### 4.4 Secrets 检查

```typescript
// 写入前检查是否包含敏感信息（团队记忆文件场景）
await checkTeamMemSecrets(file_path, content)
// 若检测到 API Key / Token 等模式，抛出错误阻止写入
```

---

## 5. GlobTool — 轻量文件搜索

### 5.1 inputSchema 与 outputSchema

```typescript
// src/tools/GlobTool/GlobTool.ts
const inputSchema = lazySchema(() =>
  z.strictObject({
    pattern: z.string().describe('The glob pattern to match files against'),
    path: z.string().optional().describe(
      'The directory to search in. Omit for CWD.',
    ),
  }),
)

const outputSchema = lazySchema(() =>
  z.object({
    durationMs: z.number(),
    numFiles: z.number(),
    filenames: z.array(z.string()),
    truncated: z.boolean(),  // 结果超过100个时截断
  }),
)
```

### 5.2 call() 实现

```typescript
async call({ pattern, path: searchPath }, { abortController, globLimits }) {
  const start = Date.now()

  const files = await glob(pattern, {
    cwd: searchPath ?? process.cwd(),
    limit: 100,              // 最多返回100个文件
    abortSignal: abortController.signal,
    checkPermissions: true,
  })

  // 将绝对路径转为相对路径，节省 token
  const filenames = files.map(f => toRelativePath(f))

  return {
    durationMs: Date.now() - start,
    numFiles: files.length,
    filenames,
    truncated: files.length >= 100,
  }
}
```

### 5.3 结果格式化

```typescript
mapToolResultToToolResultBlockParam(output) {
  if (output.filenames.length === 0) return { content: 'No files found' }

  return {
    content: [
      ...output.filenames,
      ...(output.truncated ? ['(Results are truncated...)'] : []),
    ].join('\n'),
  }
}
```

---

## 6. BashTool — 最复杂的工具

BashTool 是整个工具集中代码量最大、逻辑最复杂的工具，包含持久化 Shell、自动后台执行、输出持久化等多种机制。

### 6.1 inputSchema（完整版）

```typescript
// src/tools/BashTool/BashTool.tsx
const fullInputSchema = lazySchema(() =>
  z.strictObject({
    command: z.string().describe('The command to execute'),
    timeout: semanticNumber(z.number().optional())
      .describe(`Optional timeout in milliseconds (max ${getMaxTimeoutMs()})`),
    description: z.string().optional()
      .describe('Clear, concise description of what this command does'),
    run_in_background: semanticBoolean(z.boolean().optional())
      .describe('Run in background, returns task ID immediately'),
    dangerouslyDisableSandbox: semanticBoolean(z.boolean().optional())
      .describe('Override sandbox mode for this command'),
    _simulatedSedEdit: z.object({           // 内部字段，不暴露给 Claude
      filePath: z.string(),
      newContent: z.string(),
    }).optional(),
  }),
)
```

### 6.2 命令分类系统

BashTool 维护多个命令集合，用于行为判断：

```typescript
// 搜索类（isReadOnly = true 时允许）
const BASH_SEARCH_COMMANDS = new Set([
  'find', 'grep', 'rg', 'ag', 'ack', 'locate', 'which', 'whereis'
])

// 读取类（只读模式允许）
const BASH_READ_COMMANDS = new Set([
  'cat', 'head', 'tail', 'less', 'more', 'wc', 'stat',
  'file', 'strings', 'jq', 'awk', 'cut', 'sort', 'uniq', 'tr'
])

// 列表类
const BASH_LIST_COMMANDS = new Set(['ls', 'tree', 'du'])

// 语义中性（无副作用）
const BASH_SEMANTIC_NEUTRAL_COMMANDS = new Set(['echo', 'printf', 'true', 'false', ':'])

// 静默类（成功执行但无输出，显示"Done"而不是"(No output)"）
const BASH_SILENT_COMMANDS = new Set([
  'mv', 'cp', 'rm', 'mkdir', 'rmdir', 'chmod',
  'chown', 'touch', 'ln', 'cd', 'export', 'unset'
])
```

### 6.3 Sleep 检测（防阻塞）

```typescript
export function detectBlockedSleepPattern(command: string): string | null {
  // 解析命令中的 sleep 调用
  // 仅阻止 sleep N（N >= 2秒）
  // 允许：管道中的 sleep、子shell 中的 sleep
  // 被阻止时返回错误说明，否则返回 null
  const sleepMatch = command.match(/\bsleep\s+(\d+(?:\.\d+)?)\b/)
  if (sleepMatch) {
    const seconds = parseFloat(sleepMatch[1])
    if (seconds >= 2) {
      return `Use run_in_background=true or reduce sleep duration`
    }
  }
  return null
}
```

### 6.4 call() 核心流程

```typescript
async call(input, context, _canUseTool, parentMessage, onProgress) {
  // 1. sed 模拟编辑（内部优化路径，绕过 Shell 直接写文件）
  if (input._simulatedSedEdit) {
    return applySedEdit(input._simulatedSedEdit, context, parentMessage)
  }

  // 2. 启动 Shell 命令（async generator）
  const gen = runShellCommand({
    input,
    abortController: context.abortController,
    setAppState: context.setAppState,
    toolUseId: context.toolUseId,
  })

  let accumulated = ''

  // 3. 消费进度事件
  for await (const event of gen) {
    if (event.type === 'progress') {
      accumulated = event.fullOutput
      onProgress?.({ type: 'output', output: event.output })
    }
  }

  // 4. 获取最终结果
  const result = await gen.return(undefined)

  // 5. 大输出持久化（>30KB 写到临时文件）
  if (accumulated.length > 30_000) {
    const outputPath = await persistLargeOutput(accumulated, context.toolUseId)
    return { ...result.value, rawOutputPath: outputPath }
  }

  // 6. 图像输出检测
  if (isImageOutput(result.value.stdout)) {
    const resized = await resizeShellImageOutput(result.value.stdout)
    return { ...result.value, isImage: true, stdout: resized }
  }

  return result.value
}
```

### 6.5 runShellCommand() — async generator

这是 BashTool 的核心：一个 async generator，持续 yield 进度事件：

```typescript
async function* runShellCommand({ input, abortController, ... }) {
  // 1. 决定是否自动后台执行
  const shouldAutoBackground =
    !isBackgroundTasksDisabled &&
    isAutobackgroundingAllowed(input.command) &&
    looksLikeServerCommand(input.command)

  if (input.run_in_background || shouldAutoBackground) {
    // 后台模式：立即返回 taskId，输出异步累积
    const taskId = await launchBackgroundTask(input.command)
    yield { type: 'progress', output: `Started background task: ${taskId}` }
    return { backgroundTaskId: taskId, stdout: '', stderr: '', interrupted: false }
  }

  // 2. 前台模式：逐行 yield 输出
  const proc = await shell.exec(input.command, {
    timeout: input.timeout ?? DEFAULT_TIMEOUT,
    signal: abortController.signal,
  })

  for await (const chunk of proc.stdout) {
    yield {
      type: 'progress',
      output: chunk,
      fullOutput: accumulated,
      elapsedTimeSeconds: (Date.now() - startTime) / 1000,
      totalLines: lineCount,
    }
  }

  return {
    stdout: proc.stdout,
    stderr: proc.stderr,
    interrupted: proc.interrupted,
  }
}
```

### 6.6 mapToolResultToToolResultBlockParam() 组合逻辑

```typescript
mapToolResultToToolResultBlockParam(result) {
  // 优先：结构化内容（Claude Code Hints 标签解析结果）
  if (result.structuredContent) return { content: result.structuredContent }

  // 图像输出
  if (result.isImage) return buildImageToolResult(result.stdout)

  // 大输出：引导 Claude 去读临时文件
  if (result.persistedOutputPath) {
    return {
      content: `Output too large (${result.persistedOutputSize} bytes). ` +
               `Full output saved to: ${result.persistedOutputPath}. ` +
               `Use FileReadTool to read it.`
    }
  }

  // 后台任务
  if (result.backgroundTaskId) {
    return { content: `Background task started with ID: ${result.backgroundTaskId}` }
  }

  // 普通输出：stdout + stderr 拼接
  return {
    content: [
      result.stdout,
      result.stderr ? `<stderr>\n${result.stderr}\n</stderr>` : '',
    ].filter(Boolean).join('\n'),
  }
}
```

---

## 7. WebFetchTool — 带权限的网页抓取

### 7.1 权限检查链（最细致的权限逻辑）

```typescript
// src/tools/WebFetchTool/WebFetchTool.ts
async checkPermissions({ url }, context): Promise<PermissionDecision> {
  const hostname = new URL(url).hostname

  // 1. 预批准主机：直接允许，不走权限系统
  if (isPreapprovedHost(hostname)) return { behavior: 'allow', updatedInput: input }

  // 2. deny 规则：直接拒绝
  const denyRule = getRuleByContentsForTool(context, 'deny', hostname)
  if (denyRule) return { behavior: 'deny', reason: denyRule.reason }

  // 3. ask 规则：弹出用户确认
  const askRule = getRuleByContentsForTool(context, 'ask', hostname)
  if (askRule) return { behavior: 'ask', updatedInput: input }

  // 4. allow 规则：允许
  const allowRule = getRuleByContentsForTool(context, 'allow', hostname)
  if (allowRule) return { behavior: 'allow', updatedInput: input }

  // 5. 默认：询问用户
  return { behavior: 'ask', updatedInput: input }
}
```

### 7.2 call() 流程

```typescript
async call({ url, prompt }, context) {
  const start = Date.now()

  // 1. 获取页面 Markdown 内容（内置转换：HTML → Markdown）
  const { content, statusCode, statusText, bytes } = await getURLMarkdownContent(url)

  // 2. 重定向拦截（301/307/308 需用户确认）
  if ([301, 307, 308].includes(statusCode)) {
    return {
      result: `Redirect to ${content}. Please fetch the new URL directly.`,
      code: statusCode, ...
    }
  }

  // 3. 根据内容大小决定处理方式
  const isPreapproved = isPreapprovedHost(new URL(url).hostname)
  if (isPreapproved && content.length < DIRECT_RETURN_THRESHOLD) {
    // 小内容 + 预批准：直接返回，不经过 prompt 处理
    return { result: content, bytes, code: statusCode, ... }
  }

  // 4. 用 Claude 处理内容（applyPromptToMarkdown 内部调用 API）
  const result = await applyPromptToMarkdown(content, prompt, context)

  return { result, bytes, code: statusCode, url, durationMs: Date.now() - start }
}
```

---

## 8. TodoWriteTool — 任务状态管理

### 8.1 核心逻辑：验证提示（Verification Nudge）

TodoWriteTool 最有趣的部分是当 Agent 关闭 3+ 个任务时，检查是否有验证步骤：

```typescript
// src/tools/TodoWriteTool/TodoWriteTool.ts
async call({ todos }, context) {
  const key = context.agentId ?? context.sessionId
  const oldTodos = context.appState.todos.get(key) ?? []

  const allDone = todos.every(t => t.status === 'completed' || t.status === 'cancelled')

  // 验证提示逻辑
  let verificationNudgeNeeded = false
  if (
    isFeatureEnabled('VERIFICATION_AGENT') &&
    !context.agentId &&         // 只在主线程触发（非子 Agent）
    allDone &&
    todos.length >= 3 &&
    !todos.some(t => /verif/i.test(t.content))  // 没有验证步骤
  ) {
    verificationNudgeNeeded = true
  }

  context.appState.todos.set(key, todos)

  return { oldTodos, newTodos: todos, verificationNudgeNeeded }
}
```

### 8.2 给 Claude 的反馈消息

```typescript
mapToolResultToToolResultBlockParam({ verificationNudgeNeeded }) {
  const base = `Todos have been modified successfully. Ensure that you continue ` +
               `to use the todo list to track your progress.`

  const nudge = verificationNudgeNeeded
    ? `\n\nNOTE: You just closed out 3+ tasks and none included a verification ` +
      `step. Please consider adding a verification task to confirm the work is correct.`
    : ''

  return { content: base + nudge }
}
```

---

## 9. 执行流程图

```
Agent 调用工具
    │
    ▼
validateInput()  ──失败──→  返回错误给 Claude
    │成功
    ▼
checkPermissions()
    ├── allow  ──→  直接执行
    ├── deny   ──→  返回拒绝原因
    └── ask    ──→  展示用户确认 UI
                        │用户批准
                        ▼
call() 执行
    ├── FileReadTool: 按类型分支 → 文本/图片/PDF/Notebook
    ├── FileWriteTool: 时间戳验证 → 原子写入 → LSP通知
    ├── GlobTool: glob() → 路径相对化 → 截断到100
    ├── BashTool: sleep检测 → runShellCommand generator → 输出持久化
    ├── WebFetchTool: 权限链检查 → HTML→Markdown → prompt处理
    └── TodoWriteTool: 更新状态 → 验证提示检查
    │
    ▼
mapToolResultToToolResultBlockParam()
    │
    ▼
返回 ToolResultBlockParam 给 Claude
```

---

## 10. 设计思路总结

### 10.1 Dedup 机制（FileReadTool）
`file_unchanged` 类型避免了在长对话中反复发送相同的大文件内容给 Claude，节省 token 和费用。

### 10.2 "先读后写"约束（FileWriteTool）
强制 readFileState 机制防止 Claude 在不了解当前文件内容的情况下直接覆盖文件，同时时间戳检查防止并发冲突。

### 10.3 async generator 进度流（BashTool）
`runShellCommand` 使用 async generator 而非回调，使得"进度事件流"和"最终结果"可以用一个统一的接口处理，调用方用 `for await` 消费进度，然后 `.return()` 拿结果。

### 10.4 Token 预算（FileReadTool 图片/文本）
所有内容在发给 Claude 之前都会估算 token 数，超限时：文本直接报错；图片先压缩再报错，尽量让读取成功。

### 10.5 权限链优先级（WebFetchTool）
deny > ask > allow > 默认ask，确保安全规则优先，用户意图次之。

---

## 11. 与其他模块的联系

| 依赖模块 | 用途 |
|---------|------|
| `src/Tool.ts` | 所有工具实现的基础接口 |
| `src/utils/permissions/` | checkReadPermissionForTool / checkWritePermissionForTool |
| `src/services/lsp/manager.ts` | FileWriteTool 写入后通知 LSP |
| `src/services/mcp/` | MCP 工具通过 assembleToolPool() 注入 |
| `src/tools/BashTool/Shell.ts` | BashTool 底层的持久化 Shell 进程管理 |
| `src/utils/config.ts` | getDefaultFileReadingLimits() / getMaxTimeoutMs() |
| `src/context.ts` | readFileState / appState / agentId 等上下文字段 |
