# 03 - 工具系统设计 (Tool System)

## 1. 模块职责

工具系统是 Claude Code 的"手臂"——让 AI 模型能够执行真实的系统操作。每个工具封装一种能力（执行 Shell 命令、读写文件、搜索代码等），通过标准化接口暴露给 Claude API，实现 **"模型决策 → 工具执行 → 结果返回"** 的闭环。

工具系统解决的核心问题：如何在保持安全性（权限检查）、可扩展性（统一接口）和可观测性（UI 渲染）的前提下，让 AI 调用几十种不同类型的系统操作？

核心文件：
- `src/Tool.ts`（792行）— 工具接口类型定义 + `buildTool()` 工厂函数
- `src/tools.ts`（389行）— 工具注册表，`getAllBaseTools()` / `getTools()` / `assembleToolPool()`
- `src/tools/` — 40+ 个具体工具实现，每个工具一个子目录

---

## 2. 核心数据结构

### 2.1 Tool<Input, Output, P> 接口

`Tool.ts` 中定义了工具的完整接口，是整个系统的核心契约：

```typescript
// src/Tool.ts:365-700 (简化展示核心字段)

export type Tool<
  Input extends AnyObject = AnyObject,
  Output = unknown,
  P extends ToolProgressData = ToolProgressData,
> = {
  readonly name: string          // 工具唯一标识，如 "Bash"、"FileEdit"
  aliases?: string[]             // 向后兼容的别名
  searchHint?: string            // 供 ToolSearch 工具做关键词匹配的描述

  // ── 核心执行方法 ──
  call(
    args: z.infer<Input>,
    context: ToolUseContext,
    canUseTool: CanUseToolFn,
    parentMessage: AssistantMessage,
    onProgress?: ToolCallProgress<P>,
  ): Promise<ToolResult<Output>>

  // ── Schema ──
  readonly inputSchema: Input          // Zod schema，用于校验模型传入参数
  readonly inputJSONSchema?: ToolInputJSONSchema  // MCP 工具直接使用 JSON Schema
  outputSchema?: z.ZodType<unknown>

  // ── 权限检查 ──
  validateInput?(input, context): Promise<ValidationResult>      // 第1步：格式/语义验证
  checkPermissions(input, context): Promise<PermissionResult>    // 第2步：权限询问
  preparePermissionMatcher?(input): Promise<(pattern: string) => boolean>

  // ── 工具特性标记 ──
  isConcurrencySafe(input): boolean    // 能否与其他工具并发执行
  isReadOnly(input): boolean           // 是否只读（影响权限模式）
  isDestructive?(input): boolean       // 是否不可逆（删除、覆盖等）
  isEnabled(): boolean                 // 当前环境是否可用
  interruptBehavior?(): 'cancel' | 'block'  // 用户中断时的行为

  // ── UI 渲染（React/Ink）──
  renderToolUseMessage(input, options): React.ReactNode          // 工具调用时显示
  renderToolResultMessage?(content, progressMessages, options): React.ReactNode  // 结果显示
  renderToolUseProgressMessage?(progressMessages, options): React.ReactNode      // 进行中显示
  renderToolUseRejectedMessage?(input, options): React.ReactNode // 拒绝时显示
  renderGroupedToolUse?(toolUses, options): React.ReactNode | null // 批量并发时分组显示

  // ── 分析与监控 ──
  toAutoClassifierInput(input): unknown  // 供安全分类器分析的输入摘要
  isSearchOrReadCommand?(input): { isSearch: boolean; isRead: boolean; isList?: boolean }

  // ── 辅助方法 ──
  userFacingName(input): string          // 用户看到的显示名称
  getPath?(input): string                // 工具操作的文件路径
  prompt(options): Promise<string>       // 注入到系统提示中的工具说明
  description(input, options): Promise<string>  // 动态工具描述
  maxResultSizeChars: number             // 超过此大小时结果持久化到磁盘
}
```

### 2.2 ToolResult<T>

工具执行结果的封装：

```typescript
// src/Tool.ts:305-320

export type ToolResult<T> = {
  data: T                          // 工具返回的实际数据
  newMessages?: (UserMessage | AssistantMessage | ...)[]  // 工具可注入新消息到对话中
  contextModifier?: (context: ToolUseContext) => ToolUseContext  // 修改上下文（非并发安全工具专用）
  mcpMeta?: {
    _meta?: Record<string, unknown>
    structuredContent?: Record<string, unknown>
  }
}
```

### 2.3 ToolUseContext

工具执行时的运行时上下文（从 `query.ts` 传入），包含所有执行所需的环境信息：

```typescript
// src/Tool.ts:165-290 (关键字段)

export type ToolUseContext = {
  options: {
    commands: Command[]            // 可用的 slash 命令列表
    tools: Tools                   // 可用工具列表
    mainLoopModel: string          // 当前使用的 Claude 模型
    mcpClients: MCPServerConnection[]
    isNonInteractiveSession: boolean
    // ...
  }
  abortController: AbortController  // 用于取消正在执行的工具
  messages: Message[]               // 完整对话历史
  readFileState: FileStateCache     // 文件读取缓存
  getAppState(): AppState
  setAppState(f): void
  setToolJSX?: SetToolJSXFn         // 工具更新 React UI 的回调
  addNotification?: (notif) => void
  requestPrompt?: (sourceName, summary?) => (req) => Promise<PromptResponse>  // 弹出权限确认
  localDenialTracking?: DenialTrackingState  // 子 Agent 的权限拒绝计数
  // ...
}
```

### 2.4 ToolPermissionContext

权限系统专用上下文，控制工具调用是否需要用户确认：

```typescript
// src/Tool.ts:115-140

export type ToolPermissionContext = DeepImmutable<{
  mode: PermissionMode   // 'default' | 'plan' | 'bypassPermissions' | 'auto'
  additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>
  alwaysAllowRules: ToolPermissionRulesBySource   // 无需询问就允许的规则
  alwaysDenyRules: ToolPermissionRulesBySource    // 直接拒绝的规则
  alwaysAskRules: ToolPermissionRulesBySource     // 强制询问的规则
  isBypassPermissionsModeAvailable: boolean
  shouldAvoidPermissionPrompts?: boolean          // 后台 Agent 不能显示 UI 时为 true
}>
```

---

## 3. buildTool() 工厂函数

所有工具都通过 `buildTool()` 创建，它为常用方法填充安全默认值：

```typescript
// src/Tool.ts:750-793

// 默认值：fail-closed（安全优先）
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: () => false,   // 默认非并发安全
  isReadOnly: () => false,          // 默认有写操作
  isDestructive: () => false,       // 默认非破坏性
  checkPermissions: (input, _ctx) =>
    Promise.resolve({ behavior: 'allow', updatedInput: input }),  // 默认允许（通用权限系统会拦截）
  toAutoClassifierInput: () => '',  // 默认跳过安全分类器
  userFacingName: () => '',
}

export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return {
    ...TOOL_DEFAULTS,
    userFacingName: () => def.name,  // 默认显示名 = 工具名
    ...def,                           // 具体工具定义覆盖默认值
  } as BuiltTool<D>
}
```

典型工具定义模式（以 FileEditTool 为例）：

```typescript
// src/tools/FileEditTool/FileEditTool.ts:90-120

export const FileEditTool = buildTool({
  name: FILE_EDIT_TOOL_NAME,         // = "str_replace_based_edit_tool"
  searchHint: 'modify file contents in place',
  maxResultSizeChars: 100_000,
  strict: true,                       // 启用 API strict mode，更严格遵守 schema

  async description() {
    return 'A tool for editing files'
  },
  async prompt() {
    return getEditToolDescription()   // 从 prompt.ts 加载详细的系统提示文本
  },

  get inputSchema() {
    return inputSchema()              // 延迟求值的 Zod schema
  },

  toAutoClassifierInput(input) {
    return `${input.file_path}: ${input.new_string}`  // 安全分类器看到的摘要
  },

  getPath(input): string {
    return input.file_path            // 工具操作的文件路径（供权限检查使用）
  },

  backfillObservableInput(input) {
    // hook 观察者看到的输入预处理
    if (typeof input.file_path === 'string') {
      input.file_path = expandPath(input.file_path)  // 展开 ~ 路径
    }
  },
  // ... call(), checkPermissions(), renderToolResultMessage() 等
})
```

---

## 4. 工具注册与加载

### 4.1 getAllBaseTools() — 所有工具的来源

```typescript
// src/tools.ts:192-280（getAllBaseTools() 完整工具列表，精简注释）
export function getAllBaseTools(): Tools {
  return [
    AgentTool,              // 子 Agent 执行
    TaskOutputTool,         // 获取后台任务输出
    BashTool,               // Shell 命令执行
    // 内嵌搜索工具时跳过：ant 构建内嵌了 bfs/ugrep，无需独立 Glob/Grep 工具
    ...(hasEmbeddedSearchTools() ? [] : [GlobTool, GrepTool]),
    ExitPlanModeV2Tool,     // 退出 Plan 模式
    FileReadTool,           // 文件读取
    FileEditTool,           // 文件字符串替换编辑
    FileWriteTool,          // 文件创建/覆写
    NotebookEditTool,       // Jupyter Notebook 编辑
    WebFetchTool,           // 网页抓取
    TodoWriteTool,          // Todo 列表写入
    WebSearchTool,          // 网络搜索
    TaskStopTool,           // 停止后台任务
    AskUserQuestionTool,    // 向用户提问
    SkillTool,              // 执行技能文件
    EnterPlanModeTool,      // 进入 Plan 模式
    // ant 内部用户专有工具
    ...(process.env.USER_TYPE === 'ant' ? [ConfigTool, TungstenTool] : []),
    ...(SuggestBackgroundPRTool ? [SuggestBackgroundPRTool] : []),
    // feature flag 控制的工具（Bun bundle-time 死代码消除）
    ...(WebBrowserTool ? [WebBrowserTool] : []),
    ...(isTodoV2Enabled() ? [TaskCreateTool, TaskGetTool, TaskUpdateTool, TaskListTool] : []),
    ...(OverflowTestTool ? [OverflowTestTool] : []),    // OVERFLOW_TEST_TOOL feature
    ...(CtxInspectTool ? [CtxInspectTool] : []),        // CONTEXT_COLLAPSE feature
    ...(TerminalCaptureTool ? [TerminalCaptureTool] : []), // TERMINAL_PANEL feature
    // LSP 工具：通过环境变量 ENABLE_LSP_TOOL 条件加载（非永久内置）
    ...(isEnvTruthy(process.env.ENABLE_LSP_TOOL) ? [LSPTool] : []),
    ...(isWorktreeModeEnabled() ? [EnterWorktreeTool, ExitWorktreeTool] : []),
    // 延迟加载（打破循环依赖：TeamCreateTool → ... → tools.ts）
    getSendMessageTool(),                               // 多 Agent 消息通信
    ...(ListPeersTool ? [ListPeersTool] : []),           // UDS_INBOX feature
    ...(isAgentSwarmsEnabled() ? [getTeamCreateTool(), getTeamDeleteTool()] : []),
    ...(VerifyPlanExecutionTool ? [VerifyPlanExecutionTool] : []),
    ...(process.env.USER_TYPE === 'ant' && REPLTool ? [REPLTool] : []),
    ...(WorkflowTool ? [WorkflowTool] : []),             // WORKFLOW_SCRIPTS feature
    ...(SleepTool ? [SleepTool] : []),                   // PROACTIVE/KAIROS feature
    ...cronTools,                                        // AGENT_TRIGGERS（3 个 cron 工具）
    ...(RemoteTriggerTool ? [RemoteTriggerTool] : []),   // AGENT_TRIGGERS_REMOTE feature
    ...(MonitorTool ? [MonitorTool] : []),               // MONITOR_TOOL feature
    BriefTool,                                           // 文件摘要/上传
    ...(SendUserFileTool ? [SendUserFileTool] : []),     // KAIROS feature
    ...(PushNotificationTool ? [PushNotificationTool] : []),
    ...(SubscribePRTool ? [SubscribePRTool] : []),       // KAIROS_GITHUB_WEBHOOKS
    ...(getPowerShellTool() ? [getPowerShellTool()] : []), // Windows PowerShell
    ...(SnipTool ? [SnipTool] : []),                     // HISTORY_SNIP feature
    ...(process.env.NODE_ENV === 'test' ? [TestingPermissionTool] : []),
    ListMcpResourcesTool,   // 列出 MCP 资源
    ReadMcpResourceTool,    // 读取 MCP 资源
    ...(isToolSearchEnabledOptimistic() ? [ToolSearchTool] : []),
  ]
}
```

关键设计：
- 使用 `feature('FLAG_NAME')` (Bun bundle-time feature flag) 做**死代码消除**，未启用的工具不打包进二进制
- 循环依赖的工具（TeamCreateTool/TeamDeleteTool/SendMessageTool）用 getter 函数延迟加载
- `USER_TYPE === 'ant'` 区分 Anthropic 内部用户和外部用户的工具集
- `LSPTool` 通过 `ENABLE_LSP_TOOL` 环境变量条件加载，而非永久内置（与其他工具不同）
- 外部版本约 30 个常驻工具，内部完整构建含 feature-gated 工具可达 40+ 个

### 4.2 getTools() — 运行时工具过滤

```typescript
// src/tools.ts:285-340

export const getTools = (permissionContext: ToolPermissionContext): Tools => {
  // 简单模式（CLAUDE_CODE_SIMPLE）：只提供 Bash + FileRead + FileEdit
  if (isEnvTruthy(process.env.CLAUDE_CODE_SIMPLE)) {
    return filterToolsByDenyRules([BashTool, FileReadTool, FileEditTool], permissionContext)
  }

  const tools = getAllBaseTools().filter(tool => !specialTools.has(tool.name))
  let allowedTools = filterToolsByDenyRules(tools, permissionContext)  // 应用 deny 规则

  // REPL 模式：隐藏底层工具，改由 REPLTool 包装调用
  if (isReplModeEnabled()) {
    allowedTools = allowedTools.filter(tool => !REPL_ONLY_TOOLS.has(tool.name))
  }

  // 调用每个工具的 isEnabled() 检查，过滤不可用工具
  const isEnabled = allowedTools.map(_ => _.isEnabled())
  return allowedTools.filter((_, i) => isEnabled[i])
}
```

### 4.3 assembleToolPool() — 合并内置工具与 MCP 工具

```typescript
// src/tools.ts:360-390

export function assembleToolPool(
  permissionContext: ToolPermissionContext,
  mcpTools: Tools,
): Tools {
  const builtInTools = getTools(permissionContext)
  const allowedMcpTools = filterToolsByDenyRules(mcpTools, permissionContext)

  // 排序：保持内置工具在前，MCP 工具在后
  // 这样可以利用 API 的系统提示缓存（缓存边界在内置工具之后）
  const byName = (a: Tool, b: Tool) => a.name.localeCompare(b.name)
  return uniqBy(
    [...builtInTools].sort(byName).concat(allowedMcpTools.sort(byName)),
    'name',  // 同名工具以内置为准（built-in 优先）
  )
}
```

---

## 5. 典型工具实现解析

### 5.1 BashTool — Shell 命令执行

```typescript
// src/tools/BashTool/BashTool.tsx:230-265 (InputSchema)

const fullInputSchema = lazySchema(() => z.strictObject({
  command: z.string().describe('The command to execute'),
  timeout: semanticNumber(z.number().optional())
    .describe(`Optional timeout in milliseconds (max ${getMaxTimeoutMs()})`),
  description: z.string().optional()
    .describe('Clear, concise description of what this command does'),
  run_in_background: semanticBoolean(z.boolean().optional())
    .describe('Set to true to run this command in the background'),
  dangerouslyDisableSandbox: semanticBoolean(z.boolean().optional())
    .describe('Set this to true to dangerously override sandbox mode'),
  _simulatedSedEdit: z.object({...}).optional()
    // 内部字段：sed 编辑的预计算结果，不暴露给模型
}))

// 对外暴露的 schema 会剔除内部字段 _simulatedSedEdit（安全隔离）
const inputSchema = lazySchema(() =>
  isBackgroundTasksDisabled
    ? fullInputSchema().omit({ run_in_background: true, _simulatedSedEdit: true })
    : fullInputSchema().omit({ _simulatedSedEdit: true })
)
```

BashTool 的特殊能力：
- **沙箱模式**：通过 `SandboxManager` 可在隔离环境运行命令
- **后台任务**：`run_in_background: true` 时注册为 `LocalShellTask`，通过 `TaskOutputTool` 获取输出
- **UI 折叠**：识别 `find/grep/ls/cat` 等只读命令，UI 中自动折叠显示
- **自动后台**：assistant 模式下，阻塞超过 15 秒的命令自动转后台

### 5.2 FileEditTool — 文件字符串替换

FileEditTool 采用**字符串替换**而非行号编辑，避免了多次编辑后行号偏移的问题：

```typescript
// src/tools/FileEditTool/types.ts (inputSchema 简化)

z.strictObject({
  file_path: z.string(),          // 目标文件的绝对路径
  old_string: z.string(),         // 要替换的原始字符串（必须唯一）
  new_string: z.string(),         // 替换后的新字符串
  replace_all: z.boolean().optional().default(false),  // 替换所有匹配还是只替换第一个
})
```

关键设计：
- `old_string` 必须在文件中唯一匹配，防止误替换
- 通过 `fetchSingleFileGitDiff()` 生成 diff 显示给用户
- 编辑前记录文件 mtime，防止并发修改冲突（`FILE_UNEXPECTEDLY_MODIFIED_ERROR`）
- 操作记录到 `fileHistory`，支持撤销

### 5.3 GrepTool — 文件内容搜索

```typescript
// src/tools/GrepTool/GrepTool.ts:35-100

const inputSchema = lazySchema(() =>
  z.strictObject({
    pattern: z.string().describe('正则表达式搜索模式'),
    path: z.string().optional().describe('搜索路径，默认为当前工作目录'),
    glob: z.string().optional().describe('文件过滤 glob，如 "*.{ts,tsx}"'),
    output_mode: z.enum(['content', 'files_with_matches', 'count']).optional(),
    '-B': semanticNumber(z.number().optional()).describe('匹配前 N 行'),
    '-A': semanticNumber(z.number().optional()).describe('匹配后 N 行'),
    '-C': semanticNumber(z.number().optional()).describe('上下文行数（等同 -B + -A）'),
    '-n': semanticBoolean(z.boolean().optional()).describe('显示行号'),
    '-i': semanticBoolean(z.boolean().optional()).describe('大小写不敏感'),
    type: z.string().optional().describe('文件类型过滤，如 js/py/rust'),
    head_limit: semanticNumber(z.number().optional()).describe('限制返回结果数量'),
  })
)
```

底层调用 `ripgrep`（高性能 Rust 实现），`ripGrep()` 工具函数封装了 rg 命令行调用。

### 5.4 AgentTool — 子 Agent 执行

AgentTool 是整个系统中最复杂的工具，它允许 Claude 启动独立的子 Agent 来完成子任务：

```typescript
// src/tools/AgentTool/AgentTool.tsx — 核心能力

// 子 Agent 可以：
// 1. 拥有独立的工具集（通过 assembleToolPool() 装配）
// 2. 通过 worktree 在 Git 工作树副本中隔离执行
// 3. 以后台任务形式异步运行（LocalAgentTask / RemoteAgentTask）
// 4. 组成 Agent 团队（Agent Swarms），通过 SendMessageTool 通信

// 内置 Agent 类型（built-in/目录）：
// - generalPurposeAgent - 通用代理
// - exploreAgent        - 代码库探索
// - planAgent           - 任务规划
// - verificationAgent   - 验证执行
```

关键机制：**forked agent**（工作树隔离）—— 当 `FORK_AGENT` 模式启用时，子 Agent 在 Git worktree 的独立副本中运行，与主工作区完全隔离，完成后可选择 merge 回主分支。

---

## 6. 工具目录结构规范

每个工具遵循固定目录结构：

```
tools/FooTool/
├── FooTool.ts (or .tsx)   # 工具主体实现（buildTool() 调用）
├── UI.tsx                  # React/Ink 渲染函数（renderToolUseMessage 等）
├── prompt.ts               # 工具的系统提示文本（注入到 Claude 系统提示中）
├── constants.ts            # 工具名称等常量（TOOL_NAME）
└── types.ts                # 输入/输出 Zod Schema（部分工具）
```

`.tsx` 后缀的工具（BashTool、AgentTool 等）表示主体文件中包含 React 渲染逻辑；`.ts` 工具（FileEditTool、GrepTool 等）将渲染逻辑分离到 `UI.tsx`。

---

## 7. 工具执行流程

工具从被 Claude 选择到执行完毕的完整流程（配合 `query.ts` 的 agent loop）：

```
Claude API 流式返回 tool_use 块
        ↓
1. query.ts: 从 tool_use 中提取 toolName + input
        ↓
2. findToolByName(tools, toolName) — 查找工具实例
        ↓
3. tool.validateInput(input, context)  — 输入合法性验证（可选）
   ↓ 失败 → 返回错误给 Claude
4. tool.checkPermissions(input, context) — 权限检查
   ↓ 需要询问 → 弹出 React 权限确认 UI
   ↓ 拒绝 → 返回拒绝消息给 Claude
5. canUseTool(tool, input) — hooks 检查（PreToolUse）
        ↓
6. tool.call(args, context, canUseTool, parentMsg, onProgress)
   ↓ onProgress 回调驱动 renderToolUseProgressMessage() 实时更新
        ↓
7. 返回 ToolResult<Output>
        ↓
8. tool.mapToolResultToToolResultBlockParam(content, toolUseID)
   → 序列化为 Anthropic SDK 的 ToolResultBlockParam
        ↓
9. 追加到消息历史，继续下一轮 Claude 调用
```

---

## 8. 设计亮点

1. **接口驱动 + 工厂模式**：`Tool<I,O,P>` 接口定义契约，`buildTool()` 填充默认值，60+ 个工具实现都遵循同一模式，新增工具成本极低。

2. **Zod Schema 双用途**：`inputSchema` 既用于运行时参数校验，也被序列化为 JSON Schema 传给 Claude API 作为工具定义——一份 Schema，两种用途。

3. **UI 与逻辑分离**：每个工具的渲染函数（`render*`）独立在 `UI.tsx`，主体文件只关心逻辑，符合 React 组件化思想。

4. **lazySchema 延迟初始化**：工具 Schema 使用 `lazySchema()` 包装，避免模块加载时就执行 Zod 构建，减少启动时间。

5. **权限系统解耦**：工具本身只需实现 `checkPermissions()`，通用权限逻辑（hooks、allow/deny 规则、模式检查）由 `permissions.ts` 统一处理，工具无需重复实现。

6. **功能开关与死代码消除**：通过 `feature('FLAG_NAME')` (Bun bundle-time) 控制工具开关，未启用的功能完全不进入打包产物，二进制体积更小。

---

## 9. 与其他模块的联系

- **query.ts** — 调用 `findToolByName()` 查找工具，调用 `tool.call()` 执行，传入 `ToolUseContext`
- **tools.ts** — 工具注册表，`assembleToolPool()` 合并内置工具与 MCP 工具
- **services/mcp/** — MCP 工具通过 `MCPTool` 包装后也实现 `Tool` 接口，融入同一工具池
- **utils/permissions/** — `checkPermissions()` 最终委托给 `permissions.ts` 的通用规则引擎
- **components/** — `renderToolUseMessage()` 等方法返回 React 节点，由 REPL.tsx 渲染到终端
- **tasks/** — BashTool/AgentTool 的后台任务注册到 `LocalShellTask` / `LocalAgentTask` 管理器
