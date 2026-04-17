# 05 - MCP 协议集成

## 1. 模块职责

MCP（Model Context Protocol）是 Anthropic 定义的开放协议，允许外部服务（MCP 服务器）向 Claude 暴露工具、资源和提示词。`services/mcp/` 目录实现了完整的 MCP 客户端侧逻辑：从配置加载、连接建立、工具发现，到工具调用代理和权限管理。

**为什么需要这个模块？**  
Claude Code 的工具能力不仅限于内置工具，用户可以通过 MCP 接入任意外部服务（数据库、搜索引擎、代码索引等）。MCP 模块是这套可扩展机制的核心基础设施。

---

## 2. 核心文件地图

```
services/mcp/
├── client.ts                  # 3348行 — MCP 客户端主逻辑（连接、工具获取、工具调用）
├── types.ts                   # 258行  — 所有 MCP 类型定义（配置 schema、连接状态、序列化）
├── config.ts                  # 1578行 — 配置加载：读取 .mcp.json / settings.json / 企业配置
├── MCPConnectionManager.tsx   # React Context 包装，提供 reconnect/toggle 给 UI
├── useManageMCPConnections.ts # 连接状态 React Hook
├── normalization.ts           # MCP 工具/服务器名称规范化（兼容 API 字符集）
├── channelPermissions.ts      # 频道权限检查（plugin 来源的服务器访问控制）
├── channelAllowlist.ts        # 频道白名单配置
├── channelNotification.ts     # 频道通知处理
├── auth.ts                    # OAuth 认证提供者（ClaudeAuthProvider）
├── elicitationHandler.ts      # URL Elicitation 处理（需用户打开浏览器完成 OAuth）
├── InProcessTransport.ts      # 同进程传输层（用于测试和 SDK 模式）
├── SdkControlTransport.ts     # SDK 控制传输层
├── envExpansion.ts            # 配置中的环境变量展开
├── headersHelper.ts           # 请求头辅助工具
├── mcpStringUtils.ts          # MCP 字符串工具
├── officialRegistry.ts        # 官方 MCP 服务器注册表
├── claudeai.ts                # Claude.ai 平台特有 MCP 配置
├── oauthPort.ts               # OAuth 回调端口管理
├── vscodeSdkMcp.ts            # VS Code SDK 集成
├── xaa.ts                     # XAA 协议支持
└── xaaIdpLogin.ts             # XAA IDP 登录流程
```

---

## 3. 核心数据结构

### 3.1 服务器配置类型（types.ts:20-120）

```typescript
// services/mcp/types.ts:20-120

// 支持 6 种传输方式
export type Transport = 'stdio' | 'sse' | 'sse-ide' | 'http' | 'ws' | 'sdk'

// 配置作用域：来自哪层配置
export type ConfigScope = 
  | 'local'       // 项目级 .mcp.json
  | 'user'        // ~/.claude/settings.json
  | 'project'     // 项目 settings
  | 'dynamic'     // 运行时动态添加
  | 'enterprise'  // 企业托管配置
  | 'claudeai'    // Claude.ai 云端配置
  | 'managed'     // 受管配置

// Stdio 本地进程
type McpStdioServerConfig = {
  type?: 'stdio'   // 可选，向后兼容
  command: string  // 可执行命令，不能为空
  args: string[]
  env?: Record<string, string>
}

// SSE 长连接
type McpSSEServerConfig = {
  type: 'sse'
  url: string
  headers?: Record<string, string>
  oauth?: { clientId?: string; callbackPort?: number; ... }
}

// HTTP 请求-响应
type McpHTTPServerConfig = {
  type: 'http'
  url: string
  headers?: Record<string, string>
  oauth?: { ... }
}

// 带 scope 的服务器配置（运行时使用）
type ScopedMcpServerConfig = McpServerConfig & {
  scope: ConfigScope
  pluginSource?: string  // plugin 来源标识，用于频道权限门控
}
```

### 3.2 连接状态类型（types.ts:175-235）

```typescript
// services/mcp/types.ts:175-235

// 已连接：持有 SDK Client 实例
type ConnectedMCPServer = {
  client: Client              // @modelcontextprotocol/sdk Client
  name: string
  type: 'connected'
  capabilities: ServerCapabilities
  serverInfo?: { name: string; version: string }
  instructions?: string
  config: ScopedMcpServerConfig
  cleanup: () => Promise<void>
}

// 其他状态均只有 name + type + config
type FailedMCPServer    = { name: string; type: 'failed';     config; error?: string }
type NeedsAuthMCPServer = { name: string; type: 'needs-auth'; config }
type PendingMCPServer   = { name: string; type: 'pending';    config; reconnectAttempt?: number }
type DisabledMCPServer  = { name: string; type: 'disabled';   config }

// 判别联合类型
type MCPServerConnection =
  | ConnectedMCPServer | FailedMCPServer | NeedsAuthMCPServer
  | PendingMCPServer   | DisabledMCPServer
```

### 3.3 序列化状态（types.ts:235-259，CLI 状态快照）

```typescript
// services/mcp/types.ts:235-259

interface SerializedTool {
  name: string
  description: string
  inputJSONSchema?: { type: 'object'; properties?: {} }
  isMcp?: boolean
  originalToolName?: string  // 规范化前的原始名称
}

interface MCPCliState {
  clients: SerializedClient[]
  configs: Record<string, ScopedMcpServerConfig>
  tools: SerializedTool[]
  resources: Record<string, ServerResource[]>
  normalizedNames?: Record<string, string>  // normalized → original 映射
}
```

---

## 4. 执行流程图

### 完整数据流：启动 → 工具调用

```
应用启动
    │
    ▼
getAllMcpConfigs()               ← config.ts: 合并 local/user/enterprise/claudeai 配置
    │
    ▼
getMcpToolsCommandsAndResources()  ← client.ts:2226
    │
    ├── 分类: localServers (stdio/sdk) → 并发数 3
    │         remoteServers (sse/http/ws) → 并发数 20
    │
    ├── 每个服务器: connectToServer(name, config)
    │       │
    │       ├── type='stdio'  → StdioClientTransport(command, args, env)
    │       ├── type='sse'    → SSEClientTransport(url, {authProvider, headers})
    │       ├── type='http'   → StreamableHTTPClientTransport(url, {headers})
    │       ├── type='ws'     → WebSocketTransport(wsClient)
    │       └── type='sdk'    → SdkControlClientTransport(name)
    │
    ├── client.connect(transport)   ← SDK 握手
    │
    ├── fetchToolsForClient(conn)   ← RPC: tools/list
    │       │
    │       └── tools.map → MCPTool(name=mcp__serverName__toolName)
    │
    └── onConnectionAttempt({ client, tools, commands, resources })
             │
             └── 写入 AppState.mcpClients + AppState.tools
                      │
                      ▼
Claude 收到工具列表，可以调用 mcp__serverName__toolName(args)
    │
    ▼
MCPTool.call(args)
    │
    └── callMCPToolWithUrlElicitationRetry()
            │
            └── callMCPTool() → client.request('tools/call', {name, arguments})
```

---

## 5. 关键代码解析

### 5.1 名称规范化（normalization.ts:1-24）

```typescript
// services/mcp/normalization.ts:1-24

// Claude API 要求工具名满足正则: ^[a-zA-Z0-9_-]{1,64}$
// MCP 服务器名可能含点、空格等非法字符，必须规范化

const CLAUDEAI_SERVER_PREFIX = 'claude.ai '

export function normalizeNameForMCP(name: string): string {
  // 所有非法字符替换为下划线
  let normalized = name.replace(/[^a-zA-Z0-9_-]/g, '_')
  
  if (name.startsWith(CLAUDEAI_SERVER_PREFIX)) {
    // claude.ai 服务器：合并连续下划线，去除首尾下划线
    // 原因：MCP 工具名用 __ 作分隔符（mcp__server__tool），
    // 必须防止服务器名自身包含 __ 导致解析歧义
    normalized = normalized.replace(/_+/g, '_').replace(/^_|_$/g, '')
  }
  return normalized
}

// 示例：
// "my.server" → "my_server"
// "claude.ai my-server" → "claude_ai_my_server" → "claude_ai_my_server"（已规范）
// "a..b" → "a__b" → "a_b"（claude.ai 前缀时）
```

### 5.2 连接建立：SSE 传输（client.ts:618-695）

```typescript
// services/mcp/client.ts:618-695
// SSE（Server-Sent Events）是最常用的远程 MCP 连接方式

if (serverRef.type === 'sse') {
  const authProvider = new ClaudeAuthProvider(name, serverRef)
  const combinedHeaders = await getMcpServerHeaders(name, serverRef)

  const transportOptions: SSEClientTransportOptions = {
    authProvider,                              // OAuth 认证提供者
    fetch: wrapFetchWithTimeout(              // 超时包装（避免 AbortSignal 过期 bug）
      wrapFetchWithStepUpDetection(           // 检测 403 Step-Up 认证要求
        createFetchWithInit(), 
        authProvider
      )
    ),
    requestInit: {
      headers: {
        'User-Agent': getMCPUserAgent(),
        ...combinedHeaders,
      },
    },
  }

  // 关键：SSE 流本身（EventSource）不能用超时 fetch！
  // EventSource 是长连接，需要一直保持打开以接收事件
  // 超时 fetch 会在 60s 后关闭连接，必须单独配置
  transportOptions.eventSourceInit = {
    fetch: async (url, init) => {
      const authHeaders: Record<string, string> = {}
      const tokens = await authProvider.tokens()
      if (tokens) {
        authHeaders.Authorization = `Bearer ${tokens.access_token}` // Bearer 认证
      }
      return fetch(url, {
        ...init,
        ...getProxyFetchOptions(),            // 代理支持
        headers: {
          'User-Agent': getMCPUserAgent(),
          ...authHeaders,
          ...init?.headers,
          ...combinedHeaders,
          Accept: 'text/event-stream',        // SSE 必须指定
        },
      })
    },
  }

  transport = new SSEClientTransport(new URL(serverRef.url), transportOptions)
}
```

### 5.3 并发批处理连接（client.ts:2226-2310）

```typescript
// services/mcp/client.ts:2226-2310
// 核心启动函数：并行连接所有 MCP 服务器

export async function getMcpToolsCommandsAndResources(
  onConnectionAttempt: (params: { client; tools; commands; resources? }) => void,
  mcpConfigs?: Record<string, ScopedMcpServerConfig>,
): Promise<void> {
  const allConfigEntries = Object.entries(
    mcpConfigs ?? (await getAllMcpConfigs()).servers,
  )

  // 第一步：disabled 服务器直接跳过，不建立连接
  const configEntries: typeof allConfigEntries = []
  for (const entry of allConfigEntries) {
    if (isMcpServerDisabled(entry[0])) {
      onConnectionAttempt({ client: { name: entry[0], type: 'disabled', config: entry[1] }, tools: [], commands: [] })
    } else {
      configEntries.push(entry)
    }
  }

  // 第二步：按传输类型分组，设置不同并发上限
  // 本地（stdio/sdk）：并发 3 — 避免进程爆炸
  // 远程（sse/http/ws）：并发 20 — 网络请求可以更多
  const LOCAL_CONCURRENCY = 3
  const REMOTE_CONCURRENCY = 20
  const localServers = configEntries.filter(([_, c]) => isLocalMcpServer(c))
  const remoteServers = configEntries.filter(([_, c]) => !isLocalMcpServer(c))

  // pMap 替代了旧的固定分批方案（2026-03）
  // 旧方案：每批固定 N 个，批内最慢的服务器阻塞后续批次
  // 新方案：pMap 槽位空闲就立即补充下一个，无阻塞
  await Promise.all([
    processBatched(localServers, LOCAL_CONCURRENCY, processServer),
    processBatched(remoteServers, REMOTE_CONCURRENCY, processServer),
  ])
}
```

### 5.4 工具发现（client.ts:1743-1860）

```typescript
// services/mcp/client.ts:1743-1860
// LRU 缓存，避免同一服务器重复拉取工具列表

export const fetchToolsForClient = memoizeWithLRU(
  async (client: MCPServerConnection): Promise<Tool[]> => {
    if (client.type !== 'connected') return []
    if (!client.capabilities?.tools) return []  // 服务器不支持工具则跳过

    // RPC 调用：tools/list
    const result = await client.client.request(
      { method: 'tools/list' },
      ListToolsResultSchema,
    ) as ListToolsResult

    // Unicode 安全清洗（防止恶意服务器注入控制字符）
    const toolsToProcess = recursivelySanitizeUnicode(result.tools)

    // SDK 模式可通过环境变量跳过 mcp__ 前缀
    const skipPrefix = client.config.type === 'sdk' && isEnvTruthy(process.env.CLAUDE_AGENT_SDK_MCP_NO_PREFIX)

    return toolsToProcess.map((tool): Tool => {
      // 完全限定名：mcp__serverName__toolName
      const fullyQualifiedName = buildMcpToolName(client.name, tool.name)
      return {
        ...MCPTool,                          // 继承 MCPTool 基础实现
        name: skipPrefix ? tool.name : fullyQualifiedName,
        mcpInfo: { serverName: client.name, toolName: tool.name },
        isMcp: true,

        // 工具描述超过 2048 字节时截断（防 OpenAPI 生成的超长描述）
        async prompt() {
          const desc = tool.description ?? ''
          return desc.length > MAX_MCP_DESCRIPTION_LENGTH
            ? desc.slice(0, MAX_MCP_DESCRIPTION_LENGTH) + '… [truncated]'
            : desc
        },

        // 只读注解 → 无副作用，可并发
        isConcurrencySafe() { return tool.annotations?.readOnlyHint ?? false },
        isReadOnly()        { return tool.annotations?.readOnlyHint ?? false },
        isDestructive()     { return tool.annotations?.destructiveHint ?? false },

        // 实际调用：代理到 callMCPToolWithUrlElicitationRetry
        async call(args, context, _canUseTool, parentMessage, onProgress) {
          // ... 见 5.5
        },
      }
    })
  },
  MCP_FETCH_CACHE_SIZE,  // LRU 最多缓存 20 个服务器的工具列表
)
```

### 5.5 工具调用与 URL Elicitation（client.ts:2800-2940）

```typescript
// services/mcp/client.ts:2800-2940
// URL Elicitation：MCP 服务器要求用户在浏览器完成认证时触发

export async function callMCPToolWithUrlElicitationRetry({
  client: connectedClient, clientConnection, tool, args, meta,
  signal, setAppState, onProgress, handleElicitation,
}): Promise<MCPToolCallResult> {
  const MAX_URL_ELICITATION_RETRIES = 3

  for (let attempt = 0; ; attempt++) {
    try {
      // 正常路径：直接调用底层 RPC
      return await callMCPTool({ client: connectedClient, tool, args, meta, signal, onProgress })
    } catch (error) {
      // 只处理 -32042 错误（UrlElicitationRequired）
      if (!(error instanceof McpError) || error.code !== ErrorCode.UrlElicitationRequired) {
        throw error
      }
      if (attempt >= MAX_URL_ELICITATION_RETRIES) throw error  // 超过重试次数放弃

      // 解析 elicitation 数据
      const elicitations = (error.data?.elicitations ?? []).filter(
        (e): e is ElicitRequestURLParams =>
          e?.mode === 'url' && typeof e.url === 'string'
      )

      for (const elicitation of elicitations) {
        // 优先让 hooks 处理（程序化解决，无需用户交互）
        const hookResponse = await runElicitationHooks(serverName, elicitation, signal)
        if (hookResponse?.action === 'accept') continue  // hook 接受，继续重试

        // 否则显示 URL 给用户，等待用户在浏览器完成操作
        if (handleElicitation) {
          // 打印/SDK 模式：通过 structuredIO 发送控制请求
          userResult = await handleElicitation(serverName, elicitation, signal)
        } else {
          // REPL 模式：加入队列，等待 ElicitationDialog 组件处理
          // ... 用户操作完成后继续 for 循环重试工具调用
        }
      }
      // 所有 elicitation 处理完毕 → 自动重试 callMCPTool
    }
  }
}
```

### 5.6 认证缓存机制（client.ts:310-380）

```typescript
// services/mcp/client.ts:310-380
// 15 分钟 TTL 的认证缓存，避免每次连接都触发 OAuth 流

const MCP_AUTH_CACHE_TTL_MS = 15 * 60 * 1000  // 15 分钟

// 内存中 memoize 文件读取（N 个并发连接共享一次 IO）
let authCachePromise: Promise<McpAuthCacheData> | null = null

async function isMcpAuthCached(serverId: string): Promise<boolean> {
  const cache = await getMcpAuthCache()
  const entry = cache[serverId]
  if (!entry) return false
  return Date.now() - entry.timestamp < MCP_AUTH_CACHE_TTL_MS  // 15 分钟内有效
}

// 写入串行化：避免多服务器同时 401 时的读-改-写竞态
let writeChain = Promise.resolve()
function setMcpAuthCacheEntry(serverId: string): void {
  writeChain = writeChain.then(async () => {
    const cache = await getMcpAuthCache()
    cache[serverId] = { timestamp: Date.now() }
    await writeFile(cachePath, jsonStringify(cache))
    authCachePromise = null  // 使内存缓存失效，下次读取最新文件
  }).catch(() => {})  // best-effort，失败不影响主流程
}
```

---

## 6. 设计思路总结

### 1. 传输层抽象（6 种传输统一接口）
所有传输方式（stdio/sse/http/ws/sdk）都实现同一个 `Transport` 接口，连接建立后上层代码完全透明。这使得新增传输方式不需要修改工具调用逻辑。

### 2. 双并发策略（本地 3 / 远程 20）
本地进程（stdio）并发受操作系统进程数限制，设为 3；远程网络连接 IO 密集，可并发 20。pMap 的"槽位空闲即补充"策略比固定批次更高效，消除了批次边界的等待。

### 3. 状态机设计（5 态）
`pending → connected | failed | needs-auth | disabled` 的判别联合类型是 TypeScript 类型安全的状态机：通过 `client.type` 收窄类型后，只有 `connected` 状态才持有 `client` 实例，防止对未连接客户端进行 RPC 调用。

### 4. URL Elicitation（用户交互驱动重试）
当 MCP 服务器返回 `-32042`（需要浏览器认证）时，系统暂停工具调用，等待用户完成浏览器操作，然后自动重试，最多 3 次。这种"暂停-恢复"模式比抛出错误让用户重新发起更友好。

### 5. 工具名命名空间（`mcp__server__tool`）
使用双下划线分隔服务器名和工具名，同时规范化非法字符，解决了多 MCP 服务器可能存在同名工具的冲突问题，且满足 Anthropic API 的工具名约束。

---

## 7. 与其他模块的联系

| 依赖方向 | 模块 | 说明 |
|---------|------|------|
| 被调用 | `query.ts` / `QueryEngine.ts` | 启动时调用 `getMcpToolsCommandsAndResources()` 获取工具列表 |
| 被调用 | `AppState.ts` | 将 MCP 连接状态写入全局状态 |
| 调用 | `utils/auth.ts` | OAuth token 刷新 |
| 调用 | `utils/config.ts` | 读取全局/项目配置 |
| 调用 | `tools/MCPTool/MCPTool.ts` | 工具基础实现（MCPTool 模板） |
| 调用 | `services/analytics/` | 连接/调用事件上报 |
| 提供 | `MCPConnectionManager.tsx` | React Context，暴露 reconnect/toggle 给 UI 组件 |
| 提供 | `useManageMCPConnections.ts` | React Hook，管理连接状态响应式更新 |
