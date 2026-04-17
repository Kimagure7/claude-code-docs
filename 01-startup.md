# Claude Code 源码解析 - 01: 启动流程与初始化

> 核心文件:
> - `src/main.tsx` (4683行) — CLI 入口总控
> - `src/entrypoints/init.ts` (341行) — 核心初始化逻辑
> - `src/setup.ts` (477行) — 会话级初始化
> - `src/entrypoints/cli.tsx` — CLI 命令树注册

---

## 1. 模块职责

启动系统负责从 `claude` 命令被调用到第一轮 AI 对话开始之前的全部准备工作。这分为三个阶段：

1. **运行时初始化** (`init.ts`) — 配置、认证、网络、遥测。全局一次，使用 `memoize` 防重复。
2. **会话初始化** (`setup.ts`) — 工作目录、Worktree、Hooks、内存系统。每次 `claude` 调用执行。
3. **CLI 解析** (`main.tsx`) — Commander.js 命令树注册，参数解析，REPL/print 模式分发。

---

## 2. 核心文件地图

| 文件 | 行数 | 职责 |
|------|------|------|
| `main.tsx` | 4683 | CLI 命令树注册、参数解析、调用 init/setup/query |
| `entrypoints/init.ts` | 341 | 全局初始化（配置、网络、遥测），memoized |
| `entrypoints/cli.tsx` | - | Commander.js 注册点，`program.action()` 的实现 |
| `setup.ts` | 477 | 会话级初始化（cwd、worktree、hooks、内存） |
| `bootstrap/state.ts` | - | 全局单例状态（sessionId、cwd、mainLoopModel 等） |
| `utils/startupProfiler.ts` | - | 启动性能检查点记录 |

---

## 3. 启动全流程

```
[用户执行 claude]
      │
      ▼
main.tsx 顶层代码（模块求值阶段）
  ├─ profileCheckpoint('main_tsx_entry')     ← 记录入口时间点
  ├─ startMdmRawRead()                        ← 并行: 读取 MDM 企业配置
  └─ startKeychainPrefetch()                  ← 并行: 预取 OAuth token + API key
      │
      ▼ (约 135ms 后，imports 完成)
profileCheckpoint('main_tsx_imports_loaded')
      │
      ▼
Commander.js 解析 CLI 参数
      │
      ▼
program.action() — 主命令处理器
  │
  ├─ [1] init()                               ← memoized, 全局只跑一次
  │     ├─ enableConfigs()                    ← 启用配置系统
  │     ├─ applySafeConfigEnvironmentVariables()  ← 注入安全环境变量
  │     ├─ applyExtraCACertsFromConfig()      ← 注入 CA 证书
  │     ├─ setupGracefulShutdown()            ← 注册退出信号处理
  │     ├─ configureGlobalMTLS()              ← 配置 mTLS
  │     ├─ configureGlobalAgents()            ← 配置 HTTP 代理
  │     ├─ preconnectAnthropicApi()           ← 预热 TLS 连接（fire-and-forget）
  │     ├─ initializePolicyLimitsLoadingPromise()  ← 异步加载企业策略
  │     └─ initializeRemoteManagedSettingsLoadingPromise()  ← 异步加载远程配置
  │
  ├─ [2] checkHasTrustDialogAccepted()        ← Trust 对话框检查
  │     └─ (若未接受) showTrustDialog() → 等待用户确认
  │
  ├─ [3] initializeTelemetryAfterTrust()      ← 遥测初始化（trust 后才可以）
  │
  ├─ [4] setup(cwd, permissionMode, ...)      ← 会话级初始化
  │     ├─ setCwd(cwd)
  │     ├─ captureHooksConfigSnapshot()       ← 快照 hooks 配置，防止运行时篡改
  │     ├─ (可选) createWorktreeForSession()  ← --worktree 模式创建 git worktree
  │     ├─ initSessionMemory()                ← 注册会话记忆 hook
  │     └─ lockCurrentVersion()              ← 锁定版本防止并发升级
  │
  ├─ [5] getMcpToolsCommandsAndResources()    ← 启动 MCP 服务器，加载外部工具
  │
  ├─ [6] getTools() + getCommands()           ← 注册内置工具和命令
  │
  └─ [7] launchRepl() 或 renderAndRun()      ← 进入 REPL 或 print 模式
```

---

## 4. 关键代码解析

### 4.1 启动前并行预热 (main.tsx:1-20)

```typescript
// src/main.tsx — 顶层副作用，模块被 import 时立刻执行

// [1] 记录入口时间戳，用于后续 profileReport() 计算各阶段耗时
import { profileCheckpoint, profileReport } from './utils/startupProfiler.js'
profileCheckpoint('main_tsx_entry')

// [2] 启动 MDM 子进程（macOS: plutil, Windows: reg query）
// 这是一个独立子进程，与主线程并行运行，避免阻塞约 80ms
import { startMdmRawRead } from './utils/settings/mdm/rawRead.js'
startMdmRawRead()

// [3] 并行预取两个 keychain 项：
//   - OAuth token (macOS Keychain / Windows Credential Manager)
//   - 旧版 API key
// 若不预取，applySafeConfigEnvironmentVariables() 会顺序 spawn 两次，耗时 ~65ms
import { ensureKeychainPrefetchCompleted, startKeychainPrefetch } from './utils/secureStorage/keychainPrefetch.js'
startKeychainPrefetch()

// 上面 3 步全部 fire-and-forget，约 135ms 后 imports 完成时结果已就绪
```

**设计要点**: 这三步必须在所有其他 import 之前执行（ESLint rule `no-top-level-side-effects` 的注释白名单），因为后续 import 本身就需要 ~135ms，利用这段时间做并行 I/O，将冷启动时间从 ~200ms 压缩到 ~70ms。

---

### 4.2 init() — 全局初始化（entrypoints/init.ts:57-230）

```typescript
// src/entrypoints/init.ts:57-230
// memoize 确保无论 init() 被调用多少次，只执行一次（防重入）
export const init = memoize(async (): Promise<void> => {
  const initStartTime = Date.now()
  profileCheckpoint('init_function_start')

  try {
    // [1] 启用配置系统（从 settings.json 加载用户配置）
    enableConfigs()
    profileCheckpoint('init_configs_enabled')

    // [2] 只注入"安全"的环境变量（不含 Trust 相关的隐私设置）
    // 在 Trust 对话框之前只能注入无安全风险的设置
    applySafeConfigEnvironmentVariables()

    // [3] 将 settings.json 中的 NODE_EXTRA_CA_CERTS 写入 process.env
    // 必须在第一次 TLS 握手前执行（Bun 启动时就缓存了 BoringSSL 的 CA store）
    applyExtraCACertsFromConfig()

    // [4] 注册进程退出信号处理（SIGINT, SIGTERM → graceful shutdown）
    setupGracefulShutdown()

    // [5] 初始化一方事件日志（lazy import，避免 OpenTelemetry 模块加载拖慢启动）
    void Promise.all([
      import('../services/analytics/firstPartyEventLogger.js'),
      import('../services/analytics/growthbook.js'),
    ]).then(([fp, gb]) => {
      fp.initialize1PEventLogging()
      // GrowthBook 刷新时重建日志 provider（变更检测在 handler 内部，无变化是 no-op）
      gb.onGrowthBookRefresh(() => {
        void fp.reinitialize1PEventLoggingIfConfigChanged()
      })
    })

    // [6] 填充 OAuth 账号信息（若缓存中不存在，例如通过 VSCode 扩展登录的场景）
    void populateOAuthAccountInfoIfNeeded()

    // [7] 初始化 JetBrains IDE 检测（异步，为后续同步访问填充缓存）
    void initJetBrainsDetection()

    // [8] 异步检测 GitHub 仓库（为 gitDiff PR 链接填充缓存）
    void detectCurrentRepository()

    // [9] 初始化远程设置和策略限制加载 Promise（带超时，防死锁）
    if (isEligibleForRemoteManagedSettings()) {
      initializeRemoteManagedSettingsLoadingPromise()
    }
    if (isPolicyLimitsEligible()) {
      initializePolicyLimitsLoadingPromise()
    }

    // [10] 配置全局 mTLS 和 HTTP 代理
    configureGlobalMTLS()
    configureGlobalAgents()
    profileCheckpoint('init_network_configured')

    // [11] 预热 Anthropic API 的 TCP+TLS 连接（fire-and-forget）
    // 在 CA 证书和代理配置完成后执行，节省约 100-200ms
    preconnectAnthropicApi()

    // [12] CCR 上游代理（仅 CLAUDE_CODE_REMOTE 环境）
    if (isEnvTruthy(process.env.CLAUDE_CODE_REMOTE)) {
      const { initUpstreamProxy } = await import('../upstreamproxy/upstreamproxy.js')
      await initUpstreamProxy()    // fail-open：失败不影响启动
    }

    // [13] Windows: 配置 git-bash shell 路径
    setShellIfWindows()

    // [14] 注册 LSP 管理器清理函数（初始化在 main.tsx 的 --plugin-dir 处理后）
    registerCleanup(shutdownLspServerManager)

    // [15] 初始化 scratchpad 目录（若功能启用）
    if (isScratchpadEnabled()) {
      await ensureScratchpadDir()
    }

  } catch (error) {
    if (error instanceof ConfigParseError) {
      // 非交互模式：直接写 stderr 退出
      if (getIsNonInteractiveSession()) {
        process.stderr.write(`Configuration error in ${error.filePath}: ${error.message}\n`)
        gracefulShutdownSync(1)
        return
      }
      // 交互模式：渲染 InvalidConfigDialog React 组件，等待用户操作
      return import('../components/InvalidConfigDialog.js')
        .then(m => m.showInvalidConfigDialog({ error }))
    }
    throw error
  }
})
```

**设计要点**：
- `memoize` 来自 `lodash-es`，对同一 Promise 去重，并发调用只执行一次初始化
- 大量使用 `void`（fire-and-forget）：非关键初始化异步执行，不阻塞启动关键路径
- OpenTelemetry（约 400KB）、gRPC（约 700KB）均通过动态 `import()` 延迟加载，大幅减少初始模块求值时间

---

### 4.3 setup() — 会话级初始化（setup.ts:64-477）

```typescript
// src/setup.ts:64-73
// setup() 的完整函数签名（9 个参数）
export async function setup(
  cwd: string,                                   // 工作目录（来自 CLI --cwd 或 process.cwd()）
  permissionMode: PermissionMode,                // 权限模式：default / acceptEdits / bypassPermissions / plan
  allowDangerouslySkipPermissions: boolean,      // --dangerously-skip-permissions flag
  worktreeEnabled: boolean,                      // --worktree flag：是否创建 git worktree
  worktreeName: string | undefined,              // worktree 名称（可选，默认从 getPlanSlug() 生成）
  tmuxEnabled: boolean,                          // --tmux flag：是否为 worktree 创建 tmux 会话
  customSessionId?: string | null,               // 自定义 session ID（用于恢复会话）
  worktreePRNumber?: number,                     // PR 编号（用于 pr-{number} 格式的 worktree 名）
  messagingSocketPath?: string,                  // UDS 消息服务器套接字路径（bare 模式逃生舱）
): Promise<void> {
  logForDiagnosticsNoPII('info', 'setup_started')

  // [1] 设置工作目录 — 必须最先执行，后续所有 I/O 依赖此路径
  setCwd(cwd)

  // [2] 快照 hooks 配置，防止恶意代码在对话过程中修改 hooks
  captureHooksConfigSnapshot()

  // [3] Worktree 模式：创建 git worktree 并切换到新目录
  if (worktreeEnabled) {
    const slug = worktreePRNumber
      ? `pr-${worktreePRNumber}`           // PR 编号模式：pr-123
      : (worktreeName ?? getPlanSlug())    // 计划名或自动生成 slug
    // ...创建 worktree，chdir 到新路径，更新 projectRoot
  }

  // [4] 初始化会话记忆系统（注册 hooks，懒加载）
  if (!isBareMode()) {
    initSessionMemory()
  }

  // [5] 锁定当前版本（防止并发 auto-update 删除正在运行的二进制文件）
  void lockCurrentVersion()
}
```

**参数说明补充**：
- `worktreePRNumber`：当通过 `--pr-number 123` 启动时，worktree 分支名为 `pr-123`，方便与 PR review 工作流集成
- `messagingSocketPath`：bare 模式（`--bare`/`--print`）默认跳过 UDS 消息服务器，但明确传入此参数可强制启用，作为脚本化调用的逃生舱（escape hatch）

---

### 4.4 Feature Flag 驱动的条件导入 (main.tsx)

```typescript
// src/main.tsx — 编译时死代码消除（Bun bundle feature flags）

// feature() 在 Bun 编译时求值，false 分支会被完全删除出产物
// 这意味着外部版本的 cli.js 里不含任何 COORDINATOR_MODE 代码
const coordinatorModeModule = feature('COORDINATOR_MODE')
  ? require('./coordinator/coordinatorMode.js')
  : null

const assistantModule = feature('KAIROS')
  ? require('./assistant/index.js')
  : null

// TRANSCRIPT_CLASSIFIER 控制自动模式的分类器
const autoModeStateModule = feature('TRANSCRIPT_CLASSIFIER')
  ? require('./utils/permissions/autoModeState.js')
  : null
```

**设计要点**: Bun 的 `feature()` 等价于编译时常量，结合 tree-shaking 可以做到外部版本和 Anthropic 内部版本使用同一份源码，但产物完全不同。

---

### 4.5 Trust 对话框机制 (main.tsx)

```typescript
// src/main.tsx — Trust 对话框检查（简化版）

// 检查用户是否已接受信任对话框（存储在 global config）
const hasTrust = checkHasTrustDialogAccepted()

if (!hasTrust) {
  // 渲染 React 组件（Ink），阻塞等待用户操作
  await showSetupScreens(/* Trust 对话框 */)
}

// Trust 建立后才可以初始化遥测
// 遥测初始化本身需要从 remote settings 加载，而 remote settings 的加载
// 需要 OAuth token（Trust 之后才确认了登录状态）
initializeTelemetryAfterTrust()

// Trust 建立后才注入完整环境变量（含隐私相关配置）
applyConfigEnvironmentVariables()
```

---

## 5. 启动性能监控

`utils/startupProfiler.ts` 会记录各阶段耗时，可通过 `CLAUDE_CODE_PROFILE=1` 启用：

```
main_tsx_entry            →  0ms    (入口)
main_tsx_imports_loaded   →  135ms  (模块求值完成)
init_function_start       →  137ms
init_configs_enabled      →  139ms
init_safe_env_vars_applied →  145ms
init_network_configured   →  190ms  (mTLS + 代理配置)
setup_hooks_captured      →  220ms
setup_before_prefetch     →  230ms
[REPL 启动]               →  ~350ms (冷启动)
```

---

## 8. 关键边界条件与错误处理

### 8.1 bare 模式下的初始化跳过策略

```typescript
// src/setup.ts:82-91
// --bare / --print 模式：跳过 UDS 消息服务器和 Teammate 快照
// 脚本化调用不接收注入消息，也不使用 swarm teammates
if (!isBareMode() || messagingSocketPath !== undefined) {
  if (feature('UDS_INBOX')) {
    await startUdsMessaging(
      messagingSocketPath ?? getDefaultUdsSocketPath(),
      { isExplicit: messagingSocketPath !== undefined },
    )
  }
}
// --bare 下跳过 teammate 快照（无逃生舱，swarm 不在 bare 模式使用）
if (!isBareMode() && isAgentSwarmsEnabled()) {
  captureTeammateModeSnapshot()
}
```

**要点**：bare 模式跳过的是"不需要的"初始化，但明确传入 `messagingSocketPath` 仍可强制启用 UDS 服务器，这是刻意设计的逃生舱模式（#23222 gate pattern）。

### 8.2 --dangerously-skip-permissions 安全检查

```typescript
// src/setup.ts:380-430（简化）
if (permissionMode === 'bypassPermissions' || allowDangerouslySkipPermissions) {
  // Unix 系统：禁止 root 用户使用（沙箱除外）
  if (
    process.platform !== 'win32' &&
    typeof process.getuid === 'function' &&
    process.getuid() === 0 &&
    process.env.IS_SANDBOX !== '1' &&
    !isEnvTruthy(process.env.CLAUDE_CODE_BUBBLEWRAP)
  ) {
    console.error('--dangerously-skip-permissions cannot be used with root/sudo privileges')
    process.exit(1)
  }

  // Anthropic 内部用户：只允许在 Docker/Bubblewrap 沙箱且无网络时使用
  if (process.env.USER_TYPE === 'ant' && /* 非本地 agent 模式 */ ...) {
    const [isDocker, hasInternet] = await Promise.all([
      envDynamic.getIsDocker(),
      env.hasInternetAccess(),
    ])
    const isSandboxed = isDocker || isBubblewrap || isSandbox
    if (!isSandboxed || hasInternet) {
      console.error('--dangerously-skip-permissions can only be used in Docker/sandbox...')
      process.exit(1)
    }
  }
}
```

### 8.3 initializeTelemetryAfterTrust — Trust 后的遥测初始化时序

```typescript
// src/entrypoints/init.ts:245-285
export function initializeTelemetryAfterTrust(): void {
  if (isEligibleForRemoteManagedSettings()) {
    // SDK/headless + beta tracing：先立刻初始化，保证 tracer 在首次 query 前就绪
    if (getIsNonInteractiveSession() && isBetaTracingEnabled()) {
      void doInitializeTelemetry()   // 立即执行
    }
    // 同时等待远程设置加载完成后再次初始化（含远程配置）
    void waitForRemoteManagedSettingsToLoad().then(async () => {
      applyConfigEnvironmentVariables()  // 重新注入包含远程设置的完整环境变量
      await doInitializeTelemetry()      // 可能是第二次调用，内部有 guard
    })
  } else {
    void doInitializeTelemetry()
  }
}

// doInitializeTelemetry() 内部有幂等保护
let telemetryInitialized = false
async function doInitializeTelemetry(): Promise<void> {
  if (telemetryInitialized) return     // 防止双重初始化
  telemetryInitialized = true
  // 延迟加载约 400KB 的 OpenTelemetry 模块
  const { initializeTelemetry } = await import('../utils/telemetry/instrumentation.js')
  const meter = await initializeTelemetry()
  // ...设置 meter，增加会话计数器
}
```

---

## 9. 与其他模块的联系

| 模块 | 依赖方向 | 说明 |
|------|---------|------|
| `bootstrap/state.ts` | `main.tsx` → | 全局单例状态（sessionId、cwd、model），setup 写入，query 读取 |
| `utils/config.ts` | `init.ts` → | 配置系统，init 中 `enableConfigs()` 后才可读 |
| `services/oauth/` | `init.ts` → | OAuth token 预取，`populateOAuthAccountInfoIfNeeded()` |
| `services/mcp/client.ts` | `main.tsx` → | MCP 服务器启动，init/setup 完成后才调用 |
| `query.ts` | `main.tsx` → | 对话循环，setup 完成后传入 ToolUseContext |
| `tools.ts` / `commands.ts` | `main.tsx` → | 工具和命令注册，在 setup 完成后组装 |

---

## 10. 设计思路总结

1. **并行优化**: 所有能并行的 I/O 操作（Keychain、MDM、API 预连接、远程策略加载）都做成 fire-and-forget + 后续 await，压缩冷启动时间。

2. **安全边界**: Trust 对话框是一个硬性边界，遥测和完整环境变量初始化必须在 Trust 之后。

3. **memoize 防重入**: `init()` 用 `memoize` 包装，无论被调用多少次（REPL 重新进入等场景），只执行一次初始化逻辑。

4. **hooks 快照**: `captureHooksConfigSnapshot()` 在 `setCwd()` 后立刻执行，锁定 hooks 配置，防止对话中代码修改 `.claude/settings.json` 来影响 hooks 行为（安全考虑）。

5. **Worktree 支持**: `--worktree` 模式在 `setup()` 中创建 git worktree 并切换 cwd，之后的所有操作（CLAUDE.md 加载、hooks、技能）都从新目录解析。
