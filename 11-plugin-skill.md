# 11. 插件与技能系统

## 1. 模块职责

插件系统（Plugin System）与技能系统（Skill System）共同构成 Claude Code 的**可扩展性基础**。插件是可安装的功能包，携带命令、技能、MCP 服务器和钩子；技能（Skill）是 `/` 命令的具体实现——一段 Markdown 提示词（或内置代码），赋予模型执行特定任务的能力。两者形成"插件携带技能、技能被模型调用"的层次关系，并通过范围（scope）系统支持用户/项目/本地三级覆盖。

## 2. 核心文件地图

| 文件 | 行数 | 描述 |
|------|------|------|
| `src/services/plugins/pluginOperations.ts` | 1089 | 插件操作核心库（安装/卸载/启用/更新） |
| `src/services/plugins/PluginInstallationManager.ts` | 185 | 启动时后台安装管理器 |
| `src/services/plugins/pluginCliCommands.ts` | 345 | CLI 命令包装层（install/enable/disable/update） |
| `src/plugins/builtinPlugins.ts` | 160 | 内置插件注册表 |
| `src/plugins/bundled/index.ts` | 24 | 内置捆绑插件初始化入口 |
| `src/skills/bundledSkills.ts` | 221 | 内置技能定义类型 + 注册机制 |
| `src/skills/loadSkillsDir.ts` | 1087 | 技能加载核心：文件遍历、frontmatter 解析、去重、动态发现 |
| `src/skills/mcpSkillBuilders.ts` | 45 | MCP 协议技能构建器注册表 |
| `src/types/plugin.ts` | 364 | 插件类型定义（PluginError、BuiltinPluginDefinition 等） |

## 3. 关键数据结构

### 3.1 BundledSkillDefinition — 内置技能定义

```typescript
// src/skills/bundledSkills.ts:15-40
export type BundledSkillDefinition = {
  name: string                         // 技能名称（用于 /name 调用）
  description: string                  // 显示给用户和模型的描述
  aliases?: string[]                   // 别名
  whenToUse?: string                   // 何时使用（提示词上下文）
  argumentHint?: string                // 参数提示（如 "<query>"）
  allowedTools?: string[]              // 允许使用的工具白名单
  model?: string                       // 覆盖默认模型
  disableModelInvocation?: boolean     // 是否禁止调用模型（纯工具模式）
  userInvocable?: boolean              // 是否对用户可见（false = 隐藏技能）
  isEnabled?: () => boolean            // 运行时可用性检查
  hooks?: HooksSettings                // 技能级生命周期钩子
  context?: 'inline' | 'fork'         // 执行上下文：内联或 fork 子 agent
  agent?: string                       // 指定执行 agent
  files?: Record<string, string>       // 随技能捆绑的参考文件（相对路径 → 内容）
  getPromptForCommand: (               // 核心：生成提示词的函数
    args: string,
    context: ToolUseContext,
  ) => Promise<ContentBlockParam[]>
}
```

### 3.2 BuiltinPluginDefinition — 内置插件定义

```typescript
// src/types/plugin.ts:18-41
export type BuiltinPluginDefinition = {
  name: string
  description: string
  version?: string
  skills?: BundledSkillDefinition[]        // 插件提供的技能列表
  hooks?: HooksSettings                    // 插件提供的生命周期钩子
  mcpServers?: Record<string, McpServerConfig>  // 插件启动的 MCP 服务器
  isAvailable?: () => boolean              // 运行时可用性检查（如平台检测）
  defaultEnabled?: boolean                 // 默认是否启用（默认 true）
}
```

### 3.3 LoadedPlugin — 运行时插件实例

```typescript
// src/types/plugin.ts:51-77
export type LoadedPlugin = {
  name: string
  manifest: PluginManifest               // 插件清单（版本、依赖等）
  path: string                           // 文件系统路径
  source: string                         // 来源标识符（name@marketplace）
  repository: string                     // 市场仓库 ID
  enabled?: boolean
  isBuiltin?: boolean
  sha?: string                           // Git commit SHA（版本追踪）
  commandsPath?: string                  // 命令文件路径
  skillsPath?: string                    // 技能文件路径
  hooksConfig?: HooksSettings
  mcpServers?: Record<string, McpServerConfig>
  settings?: Record<string, unknown>     // 插件自定义设置
}
```

### 3.4 安装范围类型

```typescript
// src/services/plugins/pluginOperations.ts:72-82
export const VALID_INSTALLABLE_SCOPES = ['user', 'project', 'local'] as const
export type InstallableScope = (typeof VALID_INSTALLABLE_SCOPES)[number]

export const VALID_UPDATE_SCOPES: readonly PluginScope[] = [
  'user',      // ~/.claude/settings.json（全局）
  'project',   // .claude/settings.json（项目根，团队共享）
  'local',     // .claude/settings.json（worktree/cwd，个人覆盖）
  'managed',   // managed-settings.json（组织策略，只读）
] as const
```

### 3.5 技能加载来源枚举

```typescript
// src/skills/loadSkillsDir.ts:56-62
export type LoadedFrom =
  | 'commands_DEPRECATED'  // 旧版 commands/ 目录（向后兼容）
  | 'skills'               // 标准 skills/ 目录
  | 'plugin'               // 从插件加载
  | 'managed'              // 组织管理策略
  | 'bundled'              // 编译进二进制的内置技能
  | 'mcp'                  // 通过 MCP 协议提供的技能
```

## 4. 执行流程图

### 4.1 启动时插件加载流程

```
程序启动
    │
    ├─ getDeclaredMarketplaces()      # 从配置读取已声明的市场列表
    ├─ loadKnownMarketplacesConfig()  # 加载市场元数据
    ├─ diffMarketplaces()             # 对比：哪些市场缺失 / 哪些源改变了
    │
    └─ performBackgroundPluginInstallations()   # 非阻塞后台任务
           │
           ├─ 初始化 AppState.plugins.installationStatus = 'pending'
           ├─ reconcileMarketplaces()           # 执行实际安装/更新
           │     ├─ onProgress: 'installing' → 更新 UI 状态
           │     ├─ onProgress: 'installed'  → 标记完成
           │     └─ onProgress: 'failed'     → 记录错误
           │
           ├─ 有新安装？→ refreshActivePlugins()   # 立即刷新（修复缓存为空问题）
           │               ├─ clearMarketplacesCache()
           │               └─ reloadActivePlugins()
           │
           └─ 仅更新？→ needsRefresh = true      # 通知用户运行 /reload-plugins
```

### 4.2 插件安装流程

```
claude plugin install widget[@marketplace]
    │
    installPlugin() [CLI wrapper]
    │     └─ 遥测 + 错误格式化
    │
    installPluginOp(plugin, scope)
    │     ├─ parsePluginIdentifier()       # 解析 name + marketplace
    │     ├─ 搜索市场（遍历或直接查询）
    │     ├─ 验证插件存在
    │     └─ installResolvedPlugin()
    │             ├─ isPluginBlockedByPolicy()   # 策略检查
    │             ├─ 更新 settings.enabledPlugins # 写入配置（核心动作）
    │             └─ 缓存插件到本地
    │
    logEvent('tengu_plugin_installed_cli')
    └─ process.exit(0)
```

### 4.3 技能发现与加载流程

```
getSkillDirCommands(source)  [memoized]
    │
    ├─ 确定技能目录路径（user/project/plugin）
    ├─ loadMarkdownFilesForSubdir()   # 递归读取 .md 文件
    │
    ├─ 对每个文件：
    │     ├─ parseFrontmatter()       # 解析 YAML frontmatter
    │     ├─ parseSkillFrontmatterFields()  # 提取 name/description/tools/hooks
    │     ├─ 路径过滤（paths 字段，gitignore 语法）
    │     └─ createSkillCommand()     # 构建 Command 对象
    │
    ├─ discoverSkillDirsForPaths()    # 动态发现：当前路径有没有关联的技能目录
    │     └─ getProjectDirsUpToHome() → 从 cwd 向上遍历找 .claude/skills/
    │
    └─ deduplicateSkills()            # 按 realpath 去重（防止符号链接重复）
```

## 5. 关键代码解析

### 5.1 registerBundledSkill — 内置技能注册（带延迟文件解压）

```typescript
// src/skills/bundledSkills.ts:51-94
export function registerBundledSkill(definition: BundledSkillDefinition): void {
  const { files } = definition

  let skillRoot: string | undefined
  let getPromptForCommand = definition.getPromptForCommand

  if (files && Object.keys(files).length > 0) {
    // 技能携带参考文件时，需要在首次调用时将其解压到磁盘
    skillRoot = getBundledSkillExtractDir(definition.name)
    // 进程级别的 memoization：只解压一次
    // 关键：memoize 的是 Promise 本身，防止并发调用时的竞态写入
    let extractionPromise: Promise<string | null> | undefined
    const inner = definition.getPromptForCommand
    getPromptForCommand = async (args, ctx) => {
      extractionPromise ??= extractBundledSkillFiles(definition.name, files)
      const extractedDir = await extractionPromise
      const blocks = await inner(args, ctx)
      if (extractedDir === null) return blocks   // 解压失败时优雅降级
      return prependBaseDir(blocks, extractedDir) // 在提示词前加入目录信息
    }
  }

  // 将 BundledSkillDefinition 转换为统一的 Command 接口
  const command: Command = {
    type: 'prompt',
    name: definition.name,
    source: 'bundled',    // 来源标记
    loadedFrom: 'bundled',
    userInvocable: definition.userInvocable ?? true,
    isHidden: !(definition.userInvocable ?? true),  // 隐藏非用户可调用的技能
    progressMessage: 'running',
    // ... 其他字段
    getPromptForCommand,
  }
  bundledSkills.push(command)  // 写入全局注册表
}
```

**设计亮点**：Promise memoization 防止竞态，文件解压失败时优雅降级（技能仍可运行，只是没有参考文件路径前缀）。

### 5.2 safeWriteFile — 安全文件写入（防符号链接攻击）

```typescript
// src/skills/bundledSkills.ts:153-169
// O_NOFOLLOW|O_EXCL 双重防护（belt-and-suspenders）：
// - 进程级随机 nonce 目录是主要防线（inotify 即使获知路径也无法写入）
// - 显式 0o700/0o600 权限确保即使 umask=0 也只有 owner 可访问
// - O_EXCL 防止 TOCTOU，O_NOFOLLOW 防止最终组件的符号链接跳转
// 注意：故意不在 EEXIST 时 unlink+retry，因为 unlink() 也会跟随符号链接
const O_NOFOLLOW = fsConstants.O_NOFOLLOW ?? 0
const SAFE_WRITE_FLAGS =
  process.platform === 'win32'
    ? 'wx'                                    // Windows 用字符串标志
    : fsConstants.O_WRONLY |
      fsConstants.O_CREAT |
      fsConstants.O_EXCL |                    // 文件已存在则失败（防 TOCTOU）
      O_NOFOLLOW                              // 最终组件不跟随符号链接

async function safeWriteFile(p: string, content: string) {
  const fh = await open(p, SAFE_WRITE_FLAGS, 0o600)  // 只有 owner 可读写
  try {
    await fh.writeFile(content, 'utf8')
  } finally {
    await fh.close()  // 确保 fd 释放
  }
}
```

**安全设计**：多层防御抵御符号链接注入攻击，Windows/Unix 平台差异处理，文件权限严格限制。

### 5.3 setPluginEnabledOp — 启用/禁用状态管理

```typescript
// src/services/plugins/pluginOperations.ts:573-754（核心片段）
export async function setPluginEnabledOp(
  plugin: string,
  enabled: boolean,
  scope?: InstallableScope,
): Promise<PluginOperationResult> {

  // 内置插件走特殊路径：只修改 user 级设置
  if (isBuiltinPluginId(plugin)) {
    updateSettingsForSource('userSettings', {
      enabledPlugins: { ...existing, [plugin]: enabled }
    })
    return { success: true, message: ... }
  }

  // 范围解析：明确 > 自动检测 > 默认
  const foundScope = scope ?? findPluginInSettings(plugin)?.scope ?? 'user'
  
  // 范围优先级定义（local 可覆盖 project 可覆盖 user）
  const SCOPE_PRECEDENCE: Record<InstallableScope, number> = {
    user: 0, project: 1, local: 2,
  }

  // 策略检查：组织政策阻止时拒绝启用
  if (enabled && isPluginBlockedByPolicy(pluginId)) {
    return { success: false, message: `Plugin is blocked by organization's policy` }
  }

  // 范围冲突检测：是否需要 override 提示
  const found = findPluginInSettings(plugin)
  if (found && scope && SCOPE_PRECEDENCE[scope] <= SCOPE_PRECEDENCE[found.scope]) {
    // 低优先级范围不能覆盖高优先级范围
    return {
      success: false,
      message: `Plugin is installed at ${found.scope} scope. Use --scope local to override.`
    }
  }

  // 幂等性检查：已经是目标状态时返回错误
  if (enabled === isCurrentlyEnabled) {
    return { success: false, message: `Already ${enabled ? 'enabled' : 'disabled'}` }
  }

  // 实际更新设置
  updateSettingsForSource(scopeToSettingSource(scope), {
    enabledPlugins: { ...existing, [pluginId]: enabled }
  })
}
```

### 5.4 performBackgroundPluginInstallations — 后台安装管理

```typescript
// src/services/plugins/PluginInstallationManager.ts:41-189（核心片段）
export async function performBackgroundPluginInstallations(
  setAppState: SetState<AppState>,
): Promise<void> {

  // 对比已声明的市场 vs 已物化的市场，找出需要安装/更新的内容
  const { missing, sourceChanged } = await diffMarketplaces()
  if (!missing.length && !sourceChanged.length) return  // 无变更，快速返回

  // 初始化 UI 状态为 pending，显示进度指示
  initializeInstallationStatus(setAppState, [...missing, ...sourceChanged])

  const result = await reconcileMarketplaces({
    onProgress: event => {
      switch (event.type) {
        case 'installing':
          updateMarketplaceStatus(setAppState, event.name, 'installing')
          break
        case 'installed':
          updateMarketplaceStatus(setAppState, event.name, 'installed')
          break
        case 'failed':
          updateMarketplaceStatus(setAppState, event.name, 'failed', event.error)
          break
      }
    },
  })

  // 关键决策：有新安装 vs 仅更新，处理方式不同
  if (result.newInstallations > 0) {
    // 立即刷新活跃插件（避免缓存为空导致的错误）
    await refreshActivePlugins()
  } else if (result.updates > 0) {
    // 仅更新时：设置 needsRefresh 标志，让用户手动 /reload-plugins
    // 原因：原地更新可能破坏正在运行的插件实例
    setAppState(prev => ({
      ...prev,
      plugins: { ...prev.plugins, needsRefresh: true }
    }))
  }
}
```

### 5.5 路径安全验证 — 防目录穿越

```typescript
// src/skills/bundledSkills.ts:171-177
function resolveSkillFilePath(baseDir: string, relPath: string): string {
  const normalized = normalize(relPath)
  if (
    isAbsolute(normalized) ||                      // 拒绝绝对路径
    normalized.split(pathSep).includes('..') ||    // 拒绝 Unix/Win 路径穿越
    normalized.split('/').includes('..')           // 拒绝 URL 风格的路径穿越
  ) {
    throw new Error(`bundled skill file path escapes skill dir: ${relPath}`)
  }
  return join(baseDir, normalized)
}
```

## 6. 设计思路总结

### Settings-First（设置先行）原则
插件系统的所有操作——安装、启用、禁用——的核心动作是**更新配置文件**，而非直接修改运行时状态。实际加载由下一次启动或 `/reload-plugins` 命令触发。这种设计让操作天然幂等，配置文件成为"意图声明"（intent），而非"当前状态"（current state）。

### 三级范围覆盖系统
`user < project < local` 的优先级设计解决了团队协作中的典型冲突场景：
- 项目强制启用某插件（project scope）
- 个人开发时想临时禁用（local scope 覆盖）
- 无需修改团队配置

### Promise Memoization 防竞态
`bundledSkills.ts` 中 `extractionPromise ??= ...` 的设计是一个精妙细节：memoize 的是 Promise 对象本身，而非结果值。若多个调用并发到达（如快速连续输入两个命令），它们都会 await 同一个 Promise，而不是各自触发一次文件写入竞争。

### 内置技能的懒解压机制
携带参考文件的内置技能（`files` 字段）不在启动时解压，而是在**首次调用时**才写入磁盘。写入目录包含进程级随机 nonce（`getBundledSkillsRoot()`），配合 `O_EXCL|O_NOFOLLOW` 和严格文件权限，构建多层安全防护。

### 技能发现的路径关联
`discoverSkillDirsForPaths()` 实现了"就近原则"：从当前工作目录向上遍历，自动发现项目级 `.claude/skills/` 目录，无需用户手动配置路径。这与 `gitignore` 语义的 `paths` 字段结合，实现了"只在特定目录结构下激活"的条件技能。

## 7. 与其他模块的联系

```
插件系统
    │
    ├─→ MCP 系统 (05-mcp-protocol.md)
    │       每个插件可携带 mcpServers 配置，插件激活时
    │       MCP 服务器随之启动
    │
    ├─→ 工具系统 (03-tool-system.md)
    │       技能通过 allowedTools 声明可用工具白名单
    │       Skill 工具（AgentTool 变体）执行技能
    │
    ├─→ 命令系统 (08-command-system.md)
    │       技能最终被转换为 Command 对象，加入 / 命令列表
    │       /skill-name 触发对应的 getPromptForCommand()
    │
    ├─→ 认证配置 (06-auth-config.md)
    │       插件安装范围存储在 settings.json 中
    │       managed scope 来自组织策略配置
    │
    └─→ 查询引擎 (02-query-engine.md)
            技能执行产生的提示词块通过 ContentBlockParam[]
            注入到查询引擎的消息流中
```

**关键依赖**：
- `utils/settings/` — 读写各范围配置文件
- `services/analytics/` — 所有操作均有遥测埋点
- `utils/git/gitignore.ts` — 技能路径过滤使用 gitignore 语法
- `services/mcp/` — 插件携带的 MCP 服务器配置在此处被实例化
