# 06 - 认证与配置系统

## 1. 模块职责

认证系统负责管理用户身份验证（OAuth 2.0 + API Key 多通道），配置系统负责持久化用户偏好和项目状态。两个模块紧密耦合：OAuth 令牌存储在配置文件中，配置读写需要锁文件防止并发损坏。

---

## 2. 核心文件地图

| 文件 | 行数 | 职责 |
|------|------|------|
| `utils/auth.ts` | 2003 | 认证令牌获取、API Key 管理、AWS/GCP 凭证刷新 |
| `utils/config.ts` | 1818 | 全局/项目配置读写、文件锁、内存缓存 |
| `services/oauth/index.ts` | 199 | OAuth 2.0 PKCE 流程入口类 |
| `services/oauth/client.ts` | 567 | 构建 Auth URL、交换 code、刷新令牌 |
| `services/oauth/crypto.ts` | 24 | PKCE code_verifier / code_challenge 生成 |
| `services/oauth/auth-code-listener.ts` | - | 本地 HTTP 监听器，捕获 redirect callback |

---

## 3. 关键数据结构

### OAuthTokens（令牌对象）

```typescript
// services/oauth/types.ts
export type OAuthTokens = {
  accessToken: string
  refreshToken: string | null
  expiresAt: number | null       // Unix ms 时间戳
  scopes: string[]               // e.g. ['user:inference', 'user:profile']
  subscriptionType: SubscriptionType | null  // 'pro' | 'max' | 'team' | 'enterprise'
  rateLimitTier: RateLimitTier | null
  profile?: OAuthProfileResponse
  tokenAccount?: {
    uuid: string
    emailAddress: string
    organizationUuid?: string
  }
}
```

### GlobalConfig（全局配置结构，节选）

```typescript
// utils/config.ts:185
export type GlobalConfig = {
  primaryApiKey?: string           // /login 管理的 API Key（macOS 存 Keychain）
  oauthAccount?: AccountInfo       // 当前登录账号信息
  theme: ThemeSetting
  autoUpdates?: boolean
  autoCompactEnabled: boolean
  numStartups: number
  verbose: boolean
  env: { [key: string]: string }   // 注入环境变量
  mcpServers?: Record<string, McpServerConfig>
  customApiKeyResponses?: {
    approved?: string[]
    rejected?: string[]
  }
  cachedStatsigGates: { [gateName: string]: boolean }
  cachedGrowthBookFeatures?: { [featureName: string]: unknown }
  projects?: Record<string, ProjectConfig>  // 按项目路径索引
  // ... 200+ 字段，记录各种 UI 状态和缓存
}
```

### ProjectConfig（项目级配置）

```typescript
// utils/config.ts:82
export type ProjectConfig = {
  allowedTools: string[]           // 已授权工具白名单
  mcpServers?: Record<string, McpServerConfig>
  hasTrustDialogAccepted?: boolean // 是否接受过工作区信任
  lastSessionId?: string
  lastCost?: number
  activeWorktreeSession?: { ... }
}
```

---

## 4. 认证令牌优先级链

Claude Code 支持多种认证方式，按严格优先级顺序查找：

```
getAuthTokenSource() 优先级链
────────────────────────────────────────────
1. --bare 模式        → apiKeyHelper only，禁止 OAuth
2. ANTHROPIC_AUTH_TOKEN 环境变量（非 managed 环境）
3. CLAUDE_CODE_OAUTH_TOKEN 环境变量（managed 注入）
4. CLAUDE_CODE_OAUTH_TOKEN_FILE_DESCRIPTOR（FD 传递）
5. apiKeyHelper 配置命令（外部程序输出 API Key）
6. Claude.ai OAuth 令牌（存储在 SecureStorage）
7. none（未认证）

getAnthropicApiKeyWithSource() 优先级链
────────────────────────────────────────────
1. --bare 模式        → ANTHROPIC_API_KEY 或 apiKeyHelper
2. ANTHROPIC_API_KEY 环境变量 + 已在 approved 列表
3. CLAUDE_CODE_API_KEY_FILE_DESCRIPTOR（FD 传递）
4. apiKeyHelper 命令执行结果
5. macOS Keychain（security find-generic-password）
6. config.primaryApiKey（~/.claude.json）
7. none
```

---

## 5. OAuth 2.0 PKCE 完整流程

### 5.1 PKCE 密码学实现

```typescript
// services/oauth/crypto.ts:1-24（完整文件）
import { createHash, randomBytes } from 'crypto'

function base64URLEncode(buffer: Buffer): string {
  return buffer.toString('base64')
    .replace(/\+/g, '-')   // URL 安全替换
    .replace(/\//g, '_')
    .replace(/=/g, '')     // 去掉 padding
}

export function generateCodeVerifier(): string {
  return base64URLEncode(randomBytes(32))  // 256-bit 随机数
}

export function generateCodeChallenge(verifier: string): string {
  const hash = createHash('sha256')
  hash.update(verifier)
  return base64URLEncode(hash.digest())   // SHA256(verifier) base64url
}

export function generateState(): string {
  return base64URLEncode(randomBytes(32)) // CSRF 防护 state 参数
}
```

### 5.2 OAuthService 主流程

```typescript
// services/oauth/index.ts:35-105
async startOAuthFlow(authURLHandler, options?): Promise<OAuthTokens> {
  // 1. 启动本地 HTTP 监听器，等待 redirect callback
  this.authCodeListener = new AuthCodeListener()
  this.port = await this.authCodeListener.start()   // 监听随机端口

  // 2. 生成 PKCE 参数
  const codeChallenge = crypto.generateCodeChallenge(this.codeVerifier)
  const state = crypto.generateState()

  // 3. 构建两个 URL：自动流（localhost redirect）/ 手动流（复制粘贴）
  const manualFlowUrl = client.buildAuthUrl({ ...opts, isManual: true })
  const automaticFlowUrl = client.buildAuthUrl({ ...opts, isManual: false })

  // 4. 等待授权码（自动或手动二选一，先到先得）
  const authorizationCode = await this.waitForAuthorizationCode(
    state,
    async () => {
      await authURLHandler(manualFlowUrl)  // 显示给用户的手动链接
      await openBrowser(automaticFlowUrl) // 尝试自动打开浏览器
    },
  )

  // 5. 用 code 换 tokens（PKCE code_verifier 防止拦截攻击）
  const tokenResponse = await client.exchangeCodeForTokens(
    authorizationCode, state, this.codeVerifier, this.port!,
    !isAutomaticFlow, options?.expiresIn,
  )

  // 6. 获取 Profile（订阅类型、速率等级）
  const profileInfo = await client.fetchProfileInfo(tokenResponse.access_token)

  // 7. 格式化返回（含订阅信息）
  return this.formatTokens(tokenResponse, profileInfo.subscriptionType, ...)
}
```

### 5.3 Auth URL 构建

```typescript
// services/oauth/client.ts:53-110
export function buildAuthUrl({ codeChallenge, state, port, isManual, ... }): string {
  // 支持两种 OAuth 提供商：Console 或 Claude.ai
  const authUrlBase = loginWithClaudeAi
    ? getOauthConfig().CLAUDE_AI_AUTHORIZE_URL
    : getOauthConfig().CONSOLE_AUTHORIZE_URL

  const authUrl = new URL(authUrlBase)
  authUrl.searchParams.append('client_id', getOauthConfig().CLIENT_ID)
  authUrl.searchParams.append('response_type', 'code')
  authUrl.searchParams.append(
    'redirect_uri',
    isManual
      ? getOauthConfig().MANUAL_REDIRECT_URL       // 固定 URL，用户手动复制
      : `http://localhost:${port}/callback`,        // 本地监听自动捕获
  )
  authUrl.searchParams.append('scope', scopesToUse.join(' '))
  authUrl.searchParams.append('code_challenge', codeChallenge)
  authUrl.searchParams.append('code_challenge_method', 'S256')
  authUrl.searchParams.append('state', state)      // CSRF token
  return authUrl.toString()
}
```

---

## 6. apiKeyHelper：外部命令式 API Key

这是一个安全边界敏感的功能：允许用户配置一个 shell 命令，每次需要 API Key 时执行该命令获取。

```typescript
// utils/auth.ts:465-545（核心实现）

// SWR（Stale-While-Revalidate）模式缓存
let _apiKeyHelperCache: { value: string; timestamp: number } | null = null
let _apiKeyHelperInflight: { promise: Promise<string | null>; startedAt: number | null } | null = null
let _apiKeyHelperEpoch = 0  // 版本号，防止并发写覆盖

export async function getApiKeyFromApiKeyHelper(isNonInteractiveSession: boolean) {
  const ttl = calculateApiKeyHelperTTL()  // 默认 5 分钟，可用环境变量覆盖

  if (_apiKeyHelperCache) {
    if (Date.now() - _apiKeyHelperCache.timestamp < ttl) {
      return _apiKeyHelperCache.value  // 缓存命中
    }
    // 缓存过期 → 返回旧值，后台刷新（SWR）
    if (!_apiKeyHelperInflight) {
      _apiKeyHelperInflight = {
        promise: _runAndCache(isNonInteractiveSession, false, _apiKeyHelperEpoch),
        startedAt: null,  // null = 后台刷新，不阻塞
      }
    }
    return _apiKeyHelperCache.value
  }
  // 冷缓存 → 同步等待（阻塞，显示进度）
  _apiKeyHelperInflight = {
    promise: _runAndCache(isNonInteractiveSession, true, _apiKeyHelperEpoch),
    startedAt: Date.now(),
  }
  return _apiKeyHelperInflight.promise
}

async function _executeApiKeyHelper(isNonInteractiveSession: boolean) {
  // 安全检查：来自项目设置的命令必须先通过信任对话框
  if (isApiKeyHelperFromProjectOrLocalSettings()) {
    const hasTrust = checkHasTrustDialogAccepted()
    if (!hasTrust && !isNonInteractiveSession) {
      logEvent('tengu_apiKeyHelper_missing_trust11', {})
      return null  // 拒绝执行，防止工程师陷阱
    }
  }

  const result = await execa(apiKeyHelper, {
    shell: true,
    timeout: 10 * 60 * 1000,  // 10 分钟超时
    reject: false,
  })
  return result.stdout?.trim()
}
```

**SWR 缓存策略图：**

```
首次调用（冷缓存）
  └─ 同步等待执行 → 缓存结果（TTL=5min）→ 返回

TTL 内再次调用
  └─ 直接返回缓存 → 无延迟

TTL 过期后调用
  └─ 立即返回旧值（防抖）
  └─ 后台启动新一轮执行（不阻塞）
  └─ 执行完成 → 更新缓存

设置变更 / 401 响应
  └─ clearApiKeyHelperCache() → epoch++ → 废弃所有 inflight → 下次冷启动
```

---

## 7. API Key 安全存储

```typescript
// utils/auth.ts:1060-1140
export async function saveApiKey(apiKey: string): Promise<void> {
  // macOS：优先存 Keychain，以十六进制避免命令行参数泄露
  if (process.platform === 'darwin') {
    const hexValue = Buffer.from(apiKey, 'utf-8').toString('hex')
    // security -i 使用 stdin 而非命令行参数（防止 ps aux 暴露密码）
    const command = `add-generic-password -U -a "${username}" -s "${storageServiceName}" -X "${hexValue}"\n`
    await execa('security', ['-i'], { input: command, reject: false })
  }

  // 同时更新 config 白名单（保存 truncated key 用于显示）
  saveGlobalConfig(current => ({
    ...current,
    primaryApiKey: savedToKeychain ? current.primaryApiKey : apiKey,
    customApiKeyResponses: {
      approved: [...(current.customApiKeyResponses?.approved ?? []), normalizedKey],
      rejected: current.customApiKeyResponses?.rejected ?? [],
    },
  }))
}
```

---

## 8. 配置系统：读写与缓存

### 8.1 配置文件位置

```
~/.claude.json                    ← GlobalConfig（全局偏好）
~/.claude/projects/{hash}.json    ← ProjectConfig（项目级）
```

### 8.2 getGlobalConfig()：两级读取

```typescript
// utils/config.ts:1044-1090
export function getGlobalConfig(): GlobalConfig {
  // 快路径：内存缓存命中（运行期主路径，零 I/O）
  if (globalConfigCache.config) {
    configCacheHits++
    return globalConfigCache.config
  }

  // 慢路径：启动时首次同步读盘（只发生一次）
  configCacheMisses++
  const config = migrateConfigFields(
    getConfig(getGlobalClaudeFile(), createDefaultGlobalConfig),
  )
  globalConfigCache = { config, mtime: stats?.mtimeMs ?? Date.now() }
  startGlobalConfigFreshnessWatcher()  // 启动文件监听
  return config
}
```

### 8.3 文件新鲜度监听（多进程协调）

```typescript
// utils/config.ts:990-1030
function startGlobalConfigFreshnessWatcher(): void {
  // fs.watchFile 轮询（1秒），检测其他进程的写入
  watchFile(file, { interval: 1000, persistent: false }, curr => {
    // 自己的写入：cache.mtime > file.mtime（write-through 超前计时），跳过
    if (curr.mtimeMs <= globalConfigCache.mtime) return

    // 其他进程写入：异步重读
    void readFile(file).then(content => {
      if (curr.mtimeMs <= globalConfigCache.mtime) return  // 双检查
      globalConfigCache = {
        config: migrateConfigFields({ ...defaults, ...parsed }),
        mtime: curr.mtimeMs,
      }
    })
  })
}

// 写直通：更新内存缓存的 mtime 超前于磁盘 mtime
// 下一次 watchFile 回调时，curr.mtimeMs <= cache.mtime，跳过自己的写
function writeThroughGlobalConfigCache(config: GlobalConfig): void {
  globalConfigCache = { config, mtime: Date.now() }  // Date.now() > 磁盘 mtime
}
```

### 8.4 saveGlobalConfig()：防止认证丢失

```typescript
// utils/config.ts:795-860
export function saveGlobalConfig(updater: (current: GlobalConfig) => GlobalConfig): void {
  try {
    // 主路径：带锁文件写入（防止多实例竞争）
    const didWrite = saveConfigWithLock(
      getGlobalClaudeFile(),
      createDefaultGlobalConfig,
      current => {
        const config = updater(current)
        if (config === current) return current  // 无变化，跳过写入
        written = { ...config, projects: removeProjectHistory(current.projects) }
        return written
      },
    )
    if (didWrite && written) writeThroughGlobalConfigCache(written)
  } catch (error) {
    // 降级路径：无锁写入，但防止「认证丢失」
    const currentConfig = getConfig(...)
    if (wouldLoseAuthState(currentConfig)) {
      // 如果磁盘内容丢失了 oauthAccount 但内存还有 → 拒绝写入
      logEvent('tengu_config_auth_loss_prevented', {})
      return
    }
    saveConfig(getGlobalClaudeFile(), updater(currentConfig), DEFAULT_GLOBAL_CONFIG)
  }
}

// 认证丢失检测
function wouldLoseAuthState(fresh: { oauthAccount?: unknown; hasCompletedOnboarding?: boolean }): boolean {
  const cached = globalConfigCache.config
  if (!cached) return false
  const lostOauth = cached.oauthAccount !== undefined && fresh.oauthAccount === undefined
  const lostOnboarding = cached.hasCompletedOnboarding === true && fresh.hasCompletedOnboarding !== true
  return lostOauth || lostOnboarding
}
```

---

## 9. 工作区信任（Trust Dialog）

在工作区信任被接受之前，来自项目设置的危险命令（apiKeyHelper、awsCredentialExport、gcpAuthRefresh）不会执行。

```typescript
// utils/config.ts:715-760
export function checkHasTrustDialogAccepted(): boolean {
  // 一旦变为 true，缓存（信任只会单向增加）
  return (_trustAccepted ||= computeTrustDialogAccepted())
}

function computeTrustDialogAccepted(): boolean {
  if (getSessionTrustAccepted()) return true  // 本次会话内存信任

  const config = getGlobalConfig()
  const projectPath = getProjectPathForConfig()
  if (config.projects?.[projectPath]?.hasTrustDialogAccepted) return true

  // 向上遍历父目录——父目录信任传递给子目录
  let currentPath = normalizePathForConfigKey(getCwd())
  while (true) {
    if (config.projects?.[currentPath]?.hasTrustDialogAccepted) return true
    const parentPath = normalizePathForConfigKey(resolve(currentPath, '..'))
    if (parentPath === currentPath) break  // 到根目录了
    currentPath = parentPath
  }
  return false
}
```

---

## 10. 订阅类型系统

```typescript
// utils/auth.ts:1660-1730
export function getSubscriptionType(): SubscriptionType | null {
  if (shouldUseMockSubscription()) return getMockSubscriptionType() // 测试用
  if (!isAnthropicAuthEnabled()) return null
  const oauthTokens = getClaudeAIOAuthTokens()
  return oauthTokens?.subscriptionType ?? null
}

// 订阅类型对应权限
export function hasOpusAccess(): boolean {
  const sub = getSubscriptionType()
  return sub === 'max' || sub === 'enterprise' || sub === 'team' || sub === 'pro' || sub === null
}

export function isConsumerSubscriber(): boolean {
  return isClaudeAISubscriber() && (sub === 'max' || sub === 'pro')
}
```

---

## 11. 完整认证决策流程图

```
Claude Code 启动
     │
     ▼
isBareMode()? ──YES──→ API Key Only（apiKeyHelper / ANTHROPIC_API_KEY）
     │NO
     ▼
ANTHROPIC_UNIX_SOCKET? ──YES──→ CLAUDE_CODE_OAUTH_TOKEN? → 1P Auth / 3P Proxy
     │NO
     ▼
is3P (Bedrock/Vertex/Foundry)? ──YES──→ 3rd-party provider credentials
     │NO
     ▼
hasExternalApiKey or hasExternalAuthToken? ──YES──→ Disable Anthropic Auth
     │NO
     ▼
isAnthropicAuthEnabled() = true
     │
     ▼
getAuthTokenSource() 优先级链
  → ANTHROPIC_AUTH_TOKEN（env）
  → CLAUDE_CODE_OAUTH_TOKEN（env）
  → FD token（file descriptor）
  → apiKeyHelper（外部命令）
  → Claude.ai OAuth（SecureStorage）
  → none
```

---

## 12. 设计思路总结

**多通道认证的必要性**：
- 纯终端用户用 API Key 最简单
- 企业用户通过 Bedrock/Vertex 集成内部权限系统
- 订阅用户用 OAuth 享受 claude.ai 账号体验
- CI 环境通过环境变量注入，无交互

**SWR 缓存用于 apiKeyHelper**：外部命令可能很慢（网络调用），SWR 模式让正常调用零延迟，同时后台保持令牌新鲜。TTL 可通过 `CLAUDE_CODE_API_KEY_HELPER_TTL_MS` 调整。

**认证丢失防护（GH #3117）**：文件在其他进程 mid-write 时可能截断为空，读取后会拿到 defaults。`wouldLoseAuthState()` 检查阻止用这种截断内容覆盖掉有效的认证信息。

**配置文件锁**：`saveConfigWithLock()` 使用本地锁文件，防止多个 Claude Code 实例同时写导致 JSON 损坏。降级路径（`saveConfig`）在锁失败时仍执行，但加了认证丢失守卫。

**Trust 传递语义**：对父目录授权信任意味着子目录也被信任，与 git 工作区的直觉一致。

---

## 13. 与其他模块的关系

```
auth.ts ──→ config.ts（读写 globalConfig、oauthAccount）
auth.ts ──→ services/oauth/client.ts（令牌刷新、Profile 获取）
auth.ts ──→ secureStorage/（OAuth tokens 持久化）
config.ts ──→ lockfile.ts（写锁防竞争）
config.ts ──→ fsOperations.ts（跨平台文件读写）
services/oauth/index.ts ──→ auth-code-listener.ts（本地 HTTP 服务）
services/api/client.ts ──→ auth.ts（每次 API 调用前获取 token）
setup.ts ──→ auth.ts（启动时 prefetch API Key）
```
