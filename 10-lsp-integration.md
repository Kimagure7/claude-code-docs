# 10 - LSP 集成

## 1. 模块职责

为 Claude Code 集成语言服务器协议（LSP），让 AI 能够获取实时编译诊断、代码悬停信息和语义理解，从而在编辑代码时自动感知错误，无需运行构建流程。

---

## 2. 核心文件地图

| 文件 | 行数 | 职责 |
|------|------|------|
| `services/lsp/LSPClient.ts` | 447 | 底层 JSON-RPC 通信客户端，管理与 LSP 进程的 stdio 连接 |
| `services/lsp/LSPServerInstance.ts` | 511 | 单服务器实例生命周期管理，含重试与崩溃恢复 |
| `services/lsp/LSPServerManager.ts` | 420 | 多服务器协调，按文件扩展名路由请求 |
| `services/lsp/LSPDiagnosticRegistry.ts` | 386 | 诊断信息注册、去重与跨轮次追踪 |
| `services/lsp/manager.ts` | 289 | 全局单例，初始化状态管理与生命周期控制 |
| `services/lsp/passiveFeedback.ts` | 328 | 将 LSP 通知转换为 Claude 附件格式 |
| `services/lsp/config.ts` | 79 | 从插件系统加载 LSP 服务器配置 |

---

## 3. 关键数据结构

### LSPClient 接口

```typescript
// services/lsp/LSPClient.ts
type LSPClient = {
  readonly capabilities: ServerCapabilities | undefined
  readonly isInitialized: boolean
  start: (command: string, args: string[], options?: {
    env?: Record<string, string>
    cwd?: string
  }) => Promise<void>
  initialize: (params: InitializeParams) => Promise<InitializeResult>
  sendRequest: <TResult>(method: string, params: unknown) => Promise<TResult>
  sendNotification: (method: string, params: unknown) => Promise<void>
  onNotification: (method: string, handler: (params: unknown) => void) => void
  onRequest: <TParams, TResult>(
    method: string,
    handler: (params: TParams) => TResult | Promise<TResult>,
  ) => void
  stop: () => Promise<void>
}
```

### LSPServerInstance 接口

```typescript
// services/lsp/LSPServerInstance.ts
type LSPServerInstance = {
  readonly name: string
  readonly config: ScopedLspServerConfig
  readonly state: LspServerState   // 'stopped'|'starting'|'running'|'stopping'|'error'
  readonly startTime: Date | undefined
  readonly lastError: Error | undefined
  readonly restartCount: number
  start(): Promise<void>
  stop(): Promise<void>
  restart(): Promise<void>
  isHealthy(): boolean
  sendRequest<T>(method: string, params: unknown): Promise<T>
  sendNotification(method: string, params: unknown): Promise<void>
  onNotification(method: string, handler: (params: unknown) => void): void
  onRequest<TParams, TResult>(
    method: string,
    handler: (params: TParams) => TResult | Promise<TResult>,
  ): void
}
```

### LSP 服务器配置

```typescript
// 来自 schemas.ts 推断
type LspServerConfig = {
  command: string
  args?: string[]
  extensionToLanguage: Record<string, string>  // e.g. { ".ts": "typescript" }
  transport?: 'stdio' | 'socket'
  env?: Record<string, string>
  initializationOptions?: unknown
  settings?: unknown
  workspaceFolder?: string
  startupTimeout?: number
  shutdownTimeout?: number
  maxRestarts?: number
}
```

### 诊断类型

```typescript
// services/lsp/LSPDiagnosticRegistry.ts
type PendingLSPDiagnostic = {
  serverName: string
  files: DiagnosticFile[]
  timestamp: number
  attachmentSent: boolean
}

// 体积常量
const MAX_DIAGNOSTICS_PER_FILE = 10
const MAX_TOTAL_DIAGNOSTICS = 30
const MAX_DELIVERED_FILES = 500
```

---

## 4. 执行流程图

### 4.1 初始化流程

```
应用启动
    │
    ▼
initializeLspServerManager()
    │
    ├─ isBareMode()? → 跳过（无头/脚本模式不需要 LSP）
    │
    └─ 创建 LSPServerManager 实例
         │  state = 'pending'
         ▼
    manager.initialize()  [后台异步，不阻塞启动]
         │
         ├─ getAllLspServers()  从插件系统加载配置
         │
         ├─ 为每个服务器：
         │   ├─ 验证 command 和 extensionToLanguage
         │   ├─ 构建 extensionMap（".ts" → ["tsserver"]）
         │   ├─ createLSPServerInstance()
         │   └─ 注册 workspace/configuration 请求处理器
         │
         └─ 注册被动诊断处理器
              registerLSPNotificationHandlers()
```

### 4.2 诊断被动推送流程

```
LSP 服务器检测到文件变化/保存
    │
    ▼
推送 textDocument/publishDiagnostics 通知
    │
    ▼
onNotification 处理器触发
    │
    ├─ 验证参数（uri + diagnostics 字段）
    ├─ formatDiagnosticsForAttachment()
    │   ├─ 解析 file:// URI → 文件系统路径
    │   └─ 映射 LSP 严重程度（1=Error, 2=Warning, 3=Info, 4=Hint）
    │
    └─ registerPendingLSPDiagnostic()
         └─ 存入 pendingDiagnostics Map（UUID 键）

          ...（等待下一轮查询）...

查询处理器检查待处理诊断
    │
    ▼
checkForLSPDiagnostics()
    ├─ 收集所有未发送的 pending 诊断
    ├─ deduplicateDiagnosticFiles()
    │   ├─ 同批次内去重（createDiagnosticKey 哈希）
    │   └─ 跨轮次去重（deliveredDiagnostics LRU 缓存）
    ├─ 体积限制（每文件 ≤10，总计 ≤30，按 Error 优先）
    ├─ 记录到 deliveredDiagnostics
    └─ 返回 { serverName, files }[] 作为附件注入对话
```

### 4.3 请求/响应流程

```
工具调用（如 LSPTool.hover）
    │
    ▼
manager.sendRequest(filePath, 'textDocument/hover', params)
    │
    ├─ getServerForFile(filePath)
    │   └─ 按 path.extname() 查找 extensionMap
    │
    └─ instance.sendRequest(method, params)
         │
         ├─ isHealthy()? → 否则抛出错误
         │
         └─ client.sendRequest()  [JSON-RPC 请求]
              │
              ├─ 成功 → 返回结果
              │
              └─ ContentModified (-32801)?
                  └─ 指数退避重试：500ms → 1000ms → 2000ms（最多 3 次）
```

---

## 5. 关键代码解析

### 5.1 LSP 服务器启动与初始化

```typescript
// services/lsp/LSPServerInstance.ts
async function start(): Promise<void> {
  if (state === 'running' || state === 'starting') return

  // 检查崩溃恢复次数上限
  const maxRestarts = config.maxRestarts ?? 3
  if (state === 'error' && crashRecoveryCount > maxRestarts) {
    throw new Error(`LSP server '${name}' exceeded max crash recovery attempts (${maxRestarts})`)
  }

  let initPromise: Promise<unknown> | undefined
  try {
    state = 'starting'
    await client.start(config.command, config.args || [], {
      env: config.env,
      cwd: config.workspaceFolder,
    })

    // 构建 InitializeParams，声明客户端能力
    const initParams: InitializeParams = {
      processId: process.pid,
      initializationOptions: config.initializationOptions ?? {},
      workspaceFolders: [{
        uri: pathToFileURL(config.workspaceFolder || getCwd()).href,
        name: path.basename(config.workspaceFolder || getCwd()),
      }],
      rootPath: config.workspaceFolder || getCwd(),    // 废弃但部分服务器仍需要
      rootUri: pathToFileURL(config.workspaceFolder || getCwd()).href,
      capabilities: {
        textDocument: {
          publishDiagnostics: {
            relatedInformation: true,
            tagSupport: { valueSet: [1, 2] },   // Unnecessary, Deprecated
          },
          hover: { contentFormat: ['markdown', 'plaintext'] },
          definition: { linkSupport: true },
        },
        general: {
          positionEncodings: ['utf-16'],
        },
      },
    }

    // 可选启动超时
    initPromise = client.initialize(initParams)
    if (config.startupTimeout !== undefined) {
      await withTimeout(initPromise, config.startupTimeout, `LSP '${name}' timed out`)
    } else {
      await initPromise
    }

    state = 'running'
    startTime = new Date()
    crashRecoveryCount = 0    // 重置崩溃计数器
  } catch (error) {
    client.stop().catch(() => {})
    initPromise?.catch(() => {})
    state = 'error'
    lastError = error as Error
    throw error
  }
}
```

### 5.2 瞬时错误重试机制

```typescript
// services/lsp/LSPServerInstance.ts
const LSP_ERROR_CONTENT_MODIFIED = -32801   // rust-analyzer 特有错误码
const MAX_RETRIES_FOR_TRANSIENT_ERRORS = 3
const RETRY_BASE_DELAY_MS = 500

async function sendRequest<T>(method: string, params: unknown): Promise<T> {
  if (!isHealthy()) {
    throw new Error(`Cannot send request to LSP server '${name}': server is ${state}`)
  }

  let lastAttemptError: Error | undefined

  for (let attempt = 0; attempt <= MAX_RETRIES_FOR_TRANSIENT_ERRORS; attempt++) {
    try {
      return await client.sendRequest(method, params)
    } catch (error) {
      lastAttemptError = error as Error
      const errorCode = (error as { code?: number }).code

      // ContentModified：文件正在被修改，等待后重试
      if (errorCode === LSP_ERROR_CONTENT_MODIFIED && attempt < MAX_RETRIES_FOR_TRANSIENT_ERRORS) {
        const delay = RETRY_BASE_DELAY_MS * Math.pow(2, attempt)  // 指数退避
        await sleep(delay)
        continue
      }
      break
    }
  }

  throw new Error(`LSP request '${method}' failed for '${name}': ${lastAttemptError?.message}`)
}
```

### 5.3 多服务器路由初始化

```typescript
// services/lsp/LSPServerManager.ts
async function initialize(): Promise<void> {
  const { servers: serverConfigs } = await getAllLspServers()

  for (const [serverName, config] of Object.entries(serverConfigs)) {
    // 验证必填字段
    if (!config.command) throw new Error(`Server ${serverName} missing 'command'`)
    if (!config.extensionToLanguage || Object.keys(config.extensionToLanguage).length === 0) {
      throw new Error(`Server ${serverName} missing 'extensionToLanguage'`)
    }

    // 构建扩展名 → 服务器名称映射
    for (const ext of Object.keys(config.extensionToLanguage)) {
      const normalized = ext.toLowerCase()
      if (!extensionMap.has(normalized)) extensionMap.set(normalized, [])
      extensionMap.get(normalized)!.push(serverName)
    }

    const instance = createLSPServerInstance(serverName, config)
    servers.set(serverName, instance)

    // 注册 workspace/configuration 处理器
    // 某些服务器（TypeScript）即使我们声明不支持也会发送此请求
    instance.onRequest('workspace/configuration', (params: { items: Array<{ section?: string }> }) => {
      return params.items.map(() => null)   // 返回空配置
    })
  }
}
```

### 5.4 诊断去重与跨轮次追踪

```typescript
// services/lsp/LSPDiagnosticRegistry.ts
function deduplicateDiagnosticFiles(allFiles: DiagnosticFile[]): DiagnosticFile[] {
  const fileMap = new Map<string, Set<string>>()
  const dedupedFiles: DiagnosticFile[] = []

  for (const file of allFiles) {
    if (!fileMap.has(file.uri)) {
      fileMap.set(file.uri, new Set())
      dedupedFiles.push({ uri: file.uri, diagnostics: [] })
    }

    const seenDiagnostics = fileMap.get(file.uri)!
    const dedupedFile = dedupedFiles.find(f => f.uri === file.uri)!

    // 获取此文件上一轮已交付的诊断（LRU 缓存）
    const previouslyDelivered = deliveredDiagnostics.get(file.uri) || new Set()

    for (const diag of file.diagnostics) {
      const key = createDiagnosticKey(diag)   // JSON 哈希 {message, severity, range, source, code}

      // 跳过：本批次重复 或 已在前几轮交付过
      if (seenDiagnostics.has(key) || previouslyDelivered.has(key)) continue

      seenDiagnostics.add(key)
      dedupedFile.diagnostics.push(diag)
    }
  }

  return dedupedFiles.filter(f => f.diagnostics.length > 0)
}

export function checkForLSPDiagnostics(): Array<{ serverName: string; files: DiagnosticFile[] }> {
  const allFiles: DiagnosticFile[] = []
  const serverNames = new Set<string>()

  for (const diagnostic of pendingDiagnostics.values()) {
    if (!diagnostic.attachmentSent) {
      allFiles.push(...diagnostic.files)
      serverNames.add(diagnostic.serverName)
      diagnostic.attachmentSent = true
    }
  }

  if (allFiles.length === 0) return []

  let dedupedFiles = deduplicateDiagnosticFiles(allFiles)

  // 体积限制：按 Error 优先排序，截断到上限
  let totalDiagnostics = 0
  for (const file of dedupedFiles) {
    file.diagnostics.sort((a, b) => severityToNumber(a.severity) - severityToNumber(b.severity))
    file.diagnostics = file.diagnostics.slice(0, MAX_DIAGNOSTICS_PER_FILE)
    const remaining = MAX_TOTAL_DIAGNOSTICS - totalDiagnostics
    file.diagnostics = file.diagnostics.slice(0, remaining)
    totalDiagnostics += file.diagnostics.length
  }

  dedupedFiles = dedupedFiles.filter(f => f.diagnostics.length > 0)

  // 记录到跨轮次 LRU 缓存
  for (const file of dedupedFiles) {
    if (!deliveredDiagnostics.has(file.uri)) deliveredDiagnostics.set(file.uri, new Set())
    const delivered = deliveredDiagnostics.get(file.uri)!
    for (const diag of file.diagnostics) delivered.add(createDiagnosticKey(diag))
  }

  if (dedupedFiles.length === 0) return []
  return [{ serverName: Array.from(serverNames).join(', '), files: dedupedFiles }]
}
```

### 5.5 全局单例与生命周期管理

```typescript
// services/lsp/manager.ts
let lspManagerInstance: LSPServerManager | undefined
let initializationState: InitializationState = 'not-started'
let initializationGeneration = 0    // 用于使过期初始化失效

export function initializeLspServerManager(): void {
  if (isBareMode()) return   // 无头/脚本模式不需要 LSP

  // 幂等性：已初始化或正在初始化则跳过
  if (lspManagerInstance !== undefined && initializationState !== 'failed') return

  if (initializationState === 'failed') {
    lspManagerInstance = undefined
    initializationError = undefined
  }

  lspManagerInstance = createLSPServerManager()
  initializationState = 'pending'

  // 代次计数器：如果再次调用 initializeLspServerManager，
  // 老的异步初始化会检测到代次变化并放弃更新状态
  const currentGeneration = ++initializationGeneration

  // 后台异步初始化，不阻塞应用启动
  initializationPromise = lspManagerInstance.initialize()
    .then(() => {
      if (currentGeneration === initializationGeneration) {
        initializationState = 'success'
        if (lspManagerInstance) registerLSPNotificationHandlers(lspManagerInstance)
      }
    })
    .catch((error: unknown) => {
      if (currentGeneration === initializationGeneration) {
        initializationState = 'failed'
        initializationError = error as Error
        lspManagerInstance = undefined
        logError(error as Error)
      }
    })
}

// 插件重新加载后调用，重新初始化以获取新的 LSP 配置
export function reinitializeLspServerManager(): void {
  if (initializationState === 'not-started') return

  // 发后即弃地关闭旧实例（防止子进程泄漏）
  if (lspManagerInstance) {
    void lspManagerInstance.shutdown().catch(() => {})
  }

  lspManagerInstance = undefined
  initializationState = 'not-started'
  initializeLspServerManager()
}
```

---

## 6. 设计思路总结

### 分层架构

LSP 集成采用三层设计，每层职责单一：

```
manager.ts（全局单例）
    │ 生命周期 + 状态管理
    ▼
LSPServerManager（多服务器协调）
    │ 文件路由 + 文件同步
    ▼
LSPServerInstance（单服务器管理）
    │ 生命周期 + 重试
    ▼
LSPClient（底层通信）
    │ JSON-RPC over stdio
    ▼
LSP 服务器进程（tsserver / pyright / gopls 等）
```

### 懒加载策略

LSP 服务器不在配置加载时立即启动，而是在首次访问对应扩展名的文件时才启动（`ensureServerStarted`）。这避免了启动大量未使用的语言服务器。

### 诊断去重的两个维度

1. **批内去重**：同一 `publishDiagnostics` 通知批次内，相同诊断只取一次。
2. **跨轮次去重**：使用 LRU 缓存追踪已经注入过对话的诊断，避免每轮都重复告知 AI 同一个错误。文件编辑时调用 `clearDeliveredDiagnosticsForFile()` 重置该文件的记录。

### 代次计数器防止并发初始化竞争

`initializationGeneration` 是一个单调递增计数器。每次调用 `initializeLspServerManager()` 时递增，异步初始化完成时检查代次是否匹配，不匹配则丢弃结果。这优雅地解决了快速调用 `reinitializeLspServerManager()` 时可能产生的竞态条件。

---

## 7. 与其他模块的联系

```
setup.ts
    │  应用启动时调用
    └──→ initializeLspServerManager()

services/plugins/PluginInstallationManager.ts
    │  插件刷新时调用
    └──→ reinitializeLspServerManager()

services/lsp/config.ts
    │  从插件系统读取 LSP 配置
    └──→ getAllLspServers()

tools/LSPTool/ (通过 ENABLE_LSP_TOOL 环境变量条件加载)
    │  显式请求 Hover/Completion/Definition
    └──→ manager.sendRequest()
         └──→ LSPServerInstance.sendRequest()

query.ts / QueryEngine.ts
    │  每轮响应后检查待处理诊断
    └──→ checkForLSPDiagnostics()
         └──→ 注入诊断附件到下一轮消息

tools/FileEditTool / FileWriteTool
    │  文件写入后同步到 LSP
    └──→ manager.changeFile() / saveFile()
         └──→ 触发 LSP 服务器重新诊断
```
