# Claude Code 源码解析 - 00: 项目全局架构总览

> 源码路径: `/Users/zhaoziyi05/ZoeyChiu/claude-code-sourcemap/restored-src/src/`
> 版本: `@anthropic-ai/claude-code v2.1.88`

---

## 1. 项目定位

Claude Code 是 Anthropic 出品的 AI 编程助手 CLI 工具，本质是：

> **一个以 Claude API 为后端、React+Ink 为前端、运行在终端里的 AI Agent 系统**

它的核心能力：用户输入自然语言 → 调用 LLM → LLM 调用工具（读写文件、执行命令、搜索代码等）→ 完成编程任务。

---

## 2. 技术栈一览

| 层次 | 技术选型 | 说明 |
|------|---------|------|
| 运行时 | **Bun** | 编译为单文件 `cli.js`，比 Node 启动快 |
| CLI 框架 | **Commander.js** | 解析 `--flag` 和子命令 |
| 终端 UI | **React + Ink** | 在终端里渲染 React 组件树 |
| 类型系统 | **TypeScript + Zod v4** | Zod 用于 tool input schema 运行时校验 |
| AI SDK | **@anthropic-ai/sdk** | 官方 SDK，流式 API 调用 |
| 外部协议 | **MCP** (Model Context Protocol) | 扩展外部工具 |
| 语言服务 | **LSP** (Language Server Protocol) | 代码诊断集成 |

---

## 3. 整体目录结构

```
src/
├── main.tsx              # CLI 入口，启动主流程 (4683行)
├── query.ts              # 核心 AI 对话循环 (1729行) ← 最重要
├── QueryEngine.ts        # 查询执行引擎，Agent SDK 接口 (1295行)
├── Tool.ts               # 工具类型定义，所有 Tool 实现的接口契约 (792行)
├── tools.ts              # 工具注册/导出汇总 (389行)
├── commands.ts           # 命令注册/导出汇总 (754行)
├── setup.ts              # 初始化逻辑，系统提示词构建 (477行)
├── context.ts            # Git 状态/系统上下文构建 (189行)
│
├── tools/                # 43 个工具实现（每个工具一个目录）
├── commands/             # 80+ slash 命令实现（每个命令一个目录）
├── services/             # 核心服务模块
│   ├── api/              # Claude API 客户端
│   ├── oauth/            # 认证系统
│   ├── mcp/              # MCP 协议客户端
│   ├── lsp/              # LSP 语言服务
│   ├── plugins/          # 插件安装管理
│   ├── analytics/        # 事件追踪
│   └── policyLimits/     # 策略限制
│
├── components/           # 144 个 React 终端 UI 组件
├── hooks/                # 83 个自定义 React Hooks
├── ink/                  # 自定义 Ink 终端渲染引擎 (48个模块)
├── utils/                # 290+ 工具函数
├── constants/            # 21 个常量文件
├── types/                # 类型定义 (7个文件)
│
├── coordinator/          # 多 Agent 协调模式
├── bridge/               # IDE/远程桥接 (27个模块)
├── memdir/               # 记忆目录管理 (8个模块)
├── keybindings/          # 快捷键系统 (12个模块)
├── vim/                  # Vim 编辑模式 (5个模块)
├── skills/               # 技能系统
├── plugins/              # 插件系统入口
├── state/                # 应用状态管理
├── entrypoints/          # 多入口点 (cli/mcp/init)
└── migrations/           # 配置迁移 (10个文件)
```

---

## 4. 核心数据流

```
用户输入 (键盘 / --print 参数 / SDK调用)
    │
    ▼
main.tsx  ── 解析 CLI 参数，初始化服务
    │
    ▼
setup.ts  ── 构建 system prompt，加载 MCP 工具，初始化权限
    │
    ▼
query.ts  ── 核心对话循环 (processQuery / runQuery)
    │
    ├──→  调用 services/api/claude.ts  ── 发送消息给 Claude API (流式)
    │         │
    │         └──→  Claude 返回 tool_use 块
    │
    ├──→  Tool.call()  ── 执行具体工具 (BashTool/FileEditTool/GrepTool...)
    │         │
    │         ├──→  权限检查 (canUseTool / hooks)
    │         └──→  返回 ToolResult
    │
    └──→  把结果追加到 messages[] 继续对话循环
    │
    ▼
components/App.tsx  ── React 渲染到终端 (通过 Ink)
```

---

## 5. 最关键的 6 个文件

### 5.1 `query.ts` (1,729 行) — 对话循环大脑

这是整个系统最核心的文件。它实现了：
- `processQuery()` — 单轮 AI 对话处理
- `runQuery()` — 多轮对话循环（含工具调用）
- 流式 API 响应处理
- Hook 触发 (PreToolUse / PostToolUse / Stop)
- 权限提示、重试、中断处理

### 5.2 `QueryEngine.ts` (1,295 行) — Agent SDK 引擎

对外暴露的 Agent SDK 接口，提供：
- `QueryEngine` 类，封装对话状态
- 批处理 / 流式两种输出模式
- 子 Agent 管理

### 5.3 `Tool.ts` (793 行) — 工具接口契约

定义所有工具必须实现的 TypeScript 接口：

```typescript
// Tool.ts — 核心工具接口
export type Tool<
  Input extends AnyObject = AnyObject,
  Output = unknown,
  P extends ToolProgressData = ToolProgressData,
> = {
  name: string                   // 工具名，Claude API 调用时使用
  inputSchema: Input             // Zod schema，用于运行时参数校验
  description(input, options): Promise<string>  // 动态生成工具描述，注入 system prompt
  call(args, context, canUseTool, parentMessage, onProgress?): Promise<ToolResult<Output>>
  isEnabled(): boolean           // 是否启用（受 feature flags 控制）
  isReadOnly(input): boolean     // 是否只读（影响权限请求）
  isConcurrencySafe(input): boolean  // 是否可与其他工具并发执行
  validateInput?(input, context): Promise<ValidationResult>
  // ... 更多可选方法
}
```

### 5.4 `setup.ts` (477 行) — 系统初始化

负责：
- 构建完整 system prompt（项目信息 + Git 状态 + 用户记忆 + MCP 工具描述）
- 加载所有 MCP 服务器配置
- 初始化权限上下文
- 注入 CLAUDE.md 内容

### 5.5 `context.ts` (189 行) — 上下文信息构建

```typescript
// context.ts — 获取 Git 状态注入 system prompt
export const getGitStatus = memoize(async (): Promise<string | null> => {
  const [branch, mainBranch, status, log, userName] = await Promise.all([
    getBranch(),
    getDefaultBranch(),
    execFileNoThrow(gitExe(), ['--no-optional-locks', 'status', '--short'], ...),
    execFileNoThrow(gitExe(), ['--no-optional-locks', 'log', '--oneline', '-n', '5'], ...),
    execFileNoThrow(gitExe(), ['config', 'user.name'], ...),
  ])
  // 格式化为 system prompt 中 <git_status> 块
})
```

### 5.6 `main.tsx` (4,683 行) — 启动总控

启动时的关键优化：并行初始化（MDM读取、Keychain预取）提前于模块导入，减少冷启动时间。

---

## 6. 模块依赖关系图

```
main.tsx
  ├── entrypoints/init.ts        (telemetry, trust 检查)
  ├── setup.ts                   (system prompt, tool 初始化)
  │     ├── services/mcp/client  (MCP 工具加载)
  │     ├── skills/              (Skill 加载)
  │     └── utils/systemPrompt  (提示词构建)
  ├── commands.ts                (命令注册)
  ├── tools.ts                   (工具注册)
  ├── query.ts                   (对话循环) ← 核心
  │     ├── services/api/claude  (API 调用)
  │     ├── tools/*/             (工具执行)
  │     └── utils/hooks.ts       (Hook 执行)
  └── components/App.tsx         (UI 渲染)
        └── ink/                 (Ink 渲染引擎)
```

---

## 7. 关键代码解析

### 7.1 启动并行预热 — main.tsx 顶层副作用

```typescript
// src/main.tsx:1-19
// 这三行副作用必须在所有其他 import 之前执行（ESLint 白名单注释保护）
// 原因：后续 import 本身需要约 135ms，利用这段时间做并行 I/O

import { profileCheckpoint } from './utils/startupProfiler.js'
// eslint-disable-next-line custom-rules/no-top-level-side-effects
profileCheckpoint('main_tsx_entry')          // 记录入口时间戳

import { startMdmRawRead } from './utils/settings/mdm/rawRead.js'
// eslint-disable-next-line custom-rules/no-top-level-side-effects
startMdmRawRead()                            // 并行启动 MDM 子进程（plutil/reg query）

import { startKeychainPrefetch } from './utils/secureStorage/keychainPrefetch.js'
// eslint-disable-next-line custom-rules/no-top-level-side-effects
startKeychainPrefetch()                      // 并行预取 OAuth token + 旧版 API key
// 若不预取，applySafeConfigEnvironmentVariables() 会顺序 spawn 两次，耗时约 65ms
```

**设计要点**：将必要 I/O 提前到模块求值阶段，与约 135ms 的 import 时间完全重叠，将冷启动从约 200ms 压缩到约 70ms。

### 7.2 getGitStatus — memoize + 并行 git 命令

```typescript
// src/context.ts:36-107
export const getGitStatus = memoize(async (): Promise<string | null> => {
  if (process.env.NODE_ENV === 'test') {
    return null                              // 测试环境跳过，避免循环依赖
  }

  const isGit = await getIsGit()
  if (!isGit) return null                    // 非 git 仓库直接返回

  const [branch, mainBranch, status, log, userName] = await Promise.all([
    getBranch(),                             // 当前分支
    getDefaultBranch(),                      // 主分支（用于 PR）
    execFileNoThrow(gitExe(),
      ['--no-optional-locks', 'status', '--short'], ...).then(r => r.stdout.trim()),
    execFileNoThrow(gitExe(),
      ['--no-optional-locks', 'log', '--oneline', '-n', '5'], ...).then(r => r.stdout.trim()),
    execFileNoThrow(gitExe(),
      ['config', 'user.name'], ...).then(r => r.stdout.trim()),
  ])

  // status 超出 2000 字符时截断，防止 system prompt 过长
  const truncatedStatus = status.length > MAX_STATUS_CHARS
    ? status.substring(0, MAX_STATUS_CHARS) + '\n... (truncated ...)'
    : status

  return [
    `This is the git status at the start of the conversation. ...`,
    `Current branch: ${branch}`,
    `Main branch (you will usually use this for PRs): ${mainBranch}`,
    ...(userName ? [`Git user: ${userName}`] : []),
    `Status:\n${truncatedStatus || '(clean)'}`,
    `Recent commits:\n${log}`,
  ].join('\n\n')
})
```

**设计要点**：`memoize` 确保整次对话只读一次 git 状态（快照语义）；5 个 git 命令用 `Promise.all` 并发，总耗时等于最慢的那个而非全部之和。

### 7.3 Tool 接口契约 — Tool.ts 核心类型

```typescript
// src/Tool.ts:80-200（节选核心字段）
export type Tool<
  Input extends AnyObject = AnyObject,
  Output = unknown,
  P extends ToolProgressData = ToolProgressData,
> = {
  name: string                        // 工具名，Claude API tool_use 的 name 字段
  inputSchema: Input                  // Zod schema，运行时校验 Claude 传入的参数
  description(                        // 动态生成工具描述（注入 system prompt）
    input: Partial<Input>,
    options: DescriptionOptions,
  ): Promise<string>
  call(                               // 实际执行工具逻辑
    args: z.infer<Input>,
    context: ToolUseContext,
    canUseTool: CanUseToolFn,         // 权限检查回调
    parentMessage: AssistantMessage,
    onProgress?: (data: P) => void,   // 流式进度回调
  ): Promise<ToolResult<Output>>
  isEnabled(): boolean                // 是否启用（受 feature flag 和环境变量控制）
  isReadOnly(input: z.infer<Input>): boolean       // 只读工具不需要额外权限
  isConcurrencySafe(input: z.infer<Input>): boolean // 可与其他工具并发执行
  getSystemPromptInjections?: (      // 向 system prompt 注入额外说明
    tools: Tools,
  ) => Promise<SystemPromptSection[]>
  validateInput?: (                   // 可选：执行前参数预校验
    input: z.infer<Input>,
    context: ToolUseContext,
  ) => Promise<ValidationResult>
  renderResultForAssistant?(         // 可选：格式化返回给 Claude 的结果
    result: ToolResult<Output>,
    input: z.infer<Input>,
  ): string
}
```

**设计要点**：`description()` 是异步的，允许工具根据运行时状态（当前目录、已有文件等）动态生成描述，让 Claude 更准确地决策何时调用该工具。

### 7.4 Feature Flag 编译时死代码消除

```typescript
// src/main.tsx（节选）
import { feature } from 'bun:bundle'

// feature() 在 Bun 编译时求值，false 分支在打包产物中被完全删除
// 这是 Bun 专有能力，等价于 C/C++ 的 #ifdef 预处理器宏

// COORDINATOR_MODE：多 Agent 协调（内部版本专有）
const coordinatorModeModule = feature('COORDINATOR_MODE')
  ? require('./coordinator/coordinatorMode.js')
  : null

// KAIROS：主动/助手模式（内部版本专有）
const assistantModule = feature('KAIROS')
  ? require('./assistant/index.js')
  : null

// HISTORY_SNIP：历史截断压缩
const snipModule = feature('HISTORY_SNIP')
  ? require('./services/compact/snipCompact.js')
  : null

// 结果：外部发布的 cli.js 不含任何内部功能代码
// Anthropic 内部构建与外部构建使用同一份源码，产物完全不同
```

**设计要点**：Bun 的 `feature()` + tree-shaking 实现了"一套源码，多种产物"——外部版、内部版、KAIROS 版从同一代码库构建，无需维护多个分支。

---

## 8. 工具系统全览

| 类别 | 工具名 | 功能简述 |
|------|--------|---------|
| **文件操作** | FileReadTool | 读取文件（支持行范围、图片） |
| | FileWriteTool | 写入/覆盖文件 |
| | FileEditTool | 精确字符串替换编辑 |
| | GlobTool | 文件路径模式匹配 |
| | GrepTool | 文件内容正则搜索 |
| **命令执行** | BashTool | 执行 Shell 命令 |
| | PowerShellTool | Windows PowerShell |
| | REPLTool | 交互式 REPL (ant-only) |
| **AI/Agent** | AgentTool | 启动子 Agent |
| | SendMessageTool | 向并发 Agent 发消息 |
| | TaskCreateTool | 创建后台任务 |
| | TaskGetTool | 获取任务状态 |
| | TaskListTool | 列出所有任务 |
| | TaskStopTool | 停止任务 |
| | TaskUpdateTool | 更新任务 |
| | TaskOutputTool | 获取任务输出 |
| | TeamCreateTool | 创建 Agent Team |
| | TeamDeleteTool | 删除 Agent Team |
| **MCP** | MCPTool | MCP 协议工具（动态注入） |
| | McpAuthTool | MCP 认证 |
| | ListMcpResourcesTool | 列出 MCP 资源 |
| | ReadMcpResourceTool | 读取 MCP 资源 |
| **语言/代码** | LSPTool | LSP 诊断/补全 |
| | NotebookEditTool | Jupyter Notebook 编辑 |
| **配置** | ConfigTool | 读写 Claude Code 配置 |
| | TodoWriteTool | 写入 todo 列表 |
| **模式控制** | EnterPlanModeTool | 进入计划模式 |
| | ExitPlanModeTool | 退出计划模式 |
| | EnterWorktreeTool | 进入 Worktree 模式 |
| | ExitWorktreeTool | 退出 Worktree 模式 |
| **网络** | WebFetchTool | 获取网页内容 |
| | WebSearchTool | 网络搜索 |
| **交互** | AskUserQuestionTool | 向用户提问 |
| | BriefTool | 文件上传/摘要 |
| | ToolSearchTool | 搜索可用工具 |
| **远程/定时** | RemoteTriggerTool | 远程触发 |
| | ScheduleCronTool | 定时任务 |
| **KAIROS特有** | SleepTool | 延迟执行 |
| | SendUserFileTool | 发送文件给用户 |
| | PushNotificationTool | 推送通知 |
| | MonitorTool | 监控（feature gate） |
| | SyntheticOutputTool | 合成输出 |
| | SkillTool | 执行技能 |
| **内部** | SuggestBackgroundPRTool | PR 建议 (ant-only) |

---

## 9. Slash 命令系统全览 (80+ 命令)

命令通过 `/命令名` 在 REPL 中触发，或通过 `--cmd` 传入。实现位于 `commands/` 目录。

**主要命令类别：**
- **会话管理**: `/resume`, `/rewind`, `/rename`, `/export`, `/share`
- **代码工作流**: `/commit`, `/review`, `/diff`, `/pr_comments`
- **配置**: `/config`, `/model`, `/theme`, `/keybindings`
- **认证**: `/login`, `/logout`, `/mcp`
- **模式切换**: `/plan`, `/fast`, `/vim`, `/voice`
- **插件/技能**: `/plugin`, `/skills`, `/reload-plugins`
- **记忆**: `/memory`, `/compact`, `/clear`
- **诊断**: `/doctor`, `/stats`, `/cost`, `/effort`
- **IDE集成**: `/ide`, `/bridge`, `/desktop`, `/chrome`, `/mobile`

---

## 10. 服务模块总览

| 模块 | 路径 | 核心职责 |
|------|------|---------|
| API 客户端 | `services/api/` | Claude API 通信、重试、流式处理 |
| OAuth | `services/oauth/` | 认证流程、Token 管理 |
| MCP | `services/mcp/` | Model Context Protocol 客户端 |
| LSP | `services/lsp/` | 语言服务器集成 |
| 插件 | `services/plugins/` | 插件安装/卸载/配置 |
| 分析 | `services/analytics/` | GrowthBook、Datadog 事件追踪 |
| 策略限制 | `services/policyLimits/` | 企业策略约束 |
| 远程设置 | `services/remoteManagedSettings/` | MDM 远程配置 |
| 记忆压缩 | `services/compact/` | 对话历史压缩 |
| 提示建议 | `services/promptSuggestion/` | 自动补全提示 |

---

## 11. 特殊功能开关 (Feature Flags)

代码中大量使用 `feature()` 函数做编译时条件代码消除（Bun 的 dead code elimination）：

```typescript
// 编译时开关，影响打包结果
feature('KAIROS')              // 助手/主动模式
feature('COORDINATOR_MODE')   // 多 Agent 协调
feature('PROACTIVE')           // 主动执行模式
feature('AGENT_TRIGGERS')     // Agent 触发器
feature('MONITOR_TOOL')        // 监控工具
```

运行时开关：
```typescript
process.env.USER_TYPE === 'ant'  // Anthropic 内部版本功能
```

---

## 12. 文档阅读顺序建议

建议按以下顺序阅读后续文档：

```
00-overview.md (本文)
    ↓
01-startup.md          # 理解启动流程
    ↓
03-tool-system.md      # 理解工具接口设计
    ↓
04-tools-impl.md       # 理解具体工具实现
    ↓
02-query-engine.md     # 理解核心 AI 对话循环（最复杂）
    ↓
07-api-client.md       # 理解 API 通信层
    ↓
06-auth-config.md      # 理解认证与配置
    ↓
05-mcp-protocol.md     # 理解 MCP 扩展机制
    ↓
08-command-system.md   # 理解命令系统
    ↓
09-memory-session.md   # 理解记忆与会话
    ↓
10~13.md               # 其他扩展功能
```
