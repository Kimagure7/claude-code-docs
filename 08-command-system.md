# 08 — Slash 命令系统

## 1. 模块职责

Slash 命令系统是用户与 Claude Code 交互的重要入口之一。用户在输入框中键入 `/compact`、`/clear`、`/config` 等以 `/` 开头的指令，系统即触发对应逻辑。与工具系统（模型主动调用）不同，Slash 命令是**用户主动触发**的，用于控制会话状态、切换配置、调用技能等。

---

## 2. 核心文件地图

| 文件 | 行数 | 说明 |
|------|------|------|
| `src/commands.ts` | 754 | 命令注册中心，汇总所有命令并导出 `getCommands()` | 
| `src/types/command.ts` | — | 命令类型定义（`Command`、`PromptCommand`、`LocalCommand`、`LocalJSXCommand`） |
| `src/commands/clear/clear.ts` | — | `/clear` 命令实现 |
| `src/commands/clear/conversation.ts` | — | 清空会话的核心逻辑（含 hooks、任务保留等） |
| `src/commands/compact/compact.ts` | — | `/compact` 命令实现，含三种压缩路径 |
| `src/commands/memory/memory.tsx` | — | `/memory` 命令，打开记忆文件编辑器 |
| `src/commands/plan/plan.tsx` | — | `/plan` 命令，Plan Mode 切换 |
| `src/commands/config/config.tsx` | — | `/config` 命令，打开设置 UI |
| `src/skills/loadSkillsDir.ts` | — | 从文件系统加载自定义技能（`.claude/skills/`） |
| `src/utils/plugins/loadPluginCommands.ts` | — | 从插件加载命令 |

---

## 3. 命令类型系统

### 三种命令类型

```typescript
// src/types/command.ts

// 1. 纯文本命令：执行后返回文本结果（无 UI）
type LocalCommand = {
  type: 'local'
  supportsNonInteractive: boolean
  load: () => Promise<{ call: LocalCommandCall }>  // 懒加载
}

// 2. JSX 命令：渲染 Ink UI 界面（如 /config、/help、/memory）
type LocalJSXCommand = {
  type: 'local-jsx'
  load: () => Promise<{ call: LocalJSXCommandCall }>
}

// 3. Prompt 命令：展开为提示词发给模型（Skills / 工作流）
type PromptCommand = {
  type: 'prompt'
  source: SettingSource | 'builtin' | 'mcp' | 'plugin' | 'bundled'
  getPromptForCommand(args: string, context: ToolUseContext): Promise<ContentBlockParam[]>
  loadedFrom?: 'commands_DEPRECATED' | 'skills' | 'plugin' | 'managed' | 'bundled' | 'mcp'
  disableModelInvocation?: boolean  // 禁止模型自动调用此技能
  context?: 'inline' | 'fork'       // 'fork' = 在子 Agent 中独立运行
  paths?: string[]                  // 文件 glob，只有接触匹配文件后才显示
}

// 完整 Command = 公共基类 + 以上三种之一
export type CommandBase = {
  name: string
  description: string
  aliases?: string[]
  isEnabled?: () => boolean   // 运行时开关（Feature Flag / env 检查）
  isHidden?: boolean          // 从补全列表隐藏
  availability?: CommandAvailability[]  // 鉴权/Provider 要求
  immediate?: boolean         // true = 跳过队列立即执行
  isSensitive?: boolean       // args 从历史记录中脱敏
}

export type Command = CommandBase & (PromptCommand | LocalCommand | LocalJSXCommand)
```

**LocalCommandCall** 的签名：
```typescript
// 纯文本命令的执行函数类型
type LocalCommandCall = (
  args: string,
  context: LocalJSXCommandContext,
) => Promise<LocalCommandResult>

// 返回值可以是：
type LocalCommandResult =
  | { type: 'text'; value: string }
  | { type: 'compact'; compactionResult: CompactionResult; displayText?: string }
  | { type: 'skip' }
```

---

## 4. 命令注册与加载流程

```
用户键入 /xxx
      │
      ▼
getCommands(cwd)
      │
      ├── loadAllCommands(cwd)  ← memoize by cwd（首次加载后缓存）
      │       ├── getSkills(cwd)           ← 从 .claude/skills/ 等目录读取 .md 文件
      │       ├── getPluginCommands()      ← 从已安装插件加载命令
      │       ├── getWorkflowCommands(cwd) ← feature('WORKFLOW_SCRIPTS') 才加载
      │       └── COMMANDS()              ← 内置命令列表（memoize）
      │
      ├── getDynamicSkills()  ← 文件操作中动态发现的技能（paths 匹配）
      │
      └── 过滤：meetsAvailabilityRequirement() + isCommandEnabled()
              │
              ▼
         Command[]（返回给 REPL）
```

### `COMMANDS()` 是懒加载的 memoize

```typescript
// src/commands.ts

// 声明为函数而非常量，是因为底层函数要读 config，
// 而 config 在模块初始化时还不可用
const COMMANDS = memoize((): Command[] => [
  addDir, advisor, agents, branch, btw, chrome,
  clear, color, compact, config, copy, desktop, context,
  cost, diff, doctor, effort, exit, fast, files, ...
  // 共 70+ 个内置命令
])
```

### Feature Flag 控制的条件命令

```typescript
// 只有开启 VOICE_MODE feature flag 时才加载语音命令
const voiceCommand = feature('VOICE_MODE')
  ? require('./commands/voice/index.js').default
  : null

// 只有 ant 内部用户才能看到内部命令
...(process.env.USER_TYPE === 'ant' && !process.env.IS_DEMO
    ? INTERNAL_ONLY_COMMANDS
    : [])
```

---

## 5. 可用性过滤：`meetsAvailabilityRequirement`

```typescript
// src/commands.ts

// availability 字段与 isEnabled 是两个独立层次：
// - availability：静态的"谁能用"（按认证类型/Provider）
// - isEnabled：动态的"当前是否开启"（GrowthBook、env var）

export type CommandAvailability = 'claude-ai' | 'console'

export function meetsAvailabilityRequirement(cmd: Command): boolean {
  if (!cmd.availability) return true  // 无限制 = 所有人可用
  for (const a of cmd.availability) {
    switch (a) {
      case 'claude-ai':
        if (isClaudeAISubscriber()) return true  // claude.ai OAuth 用户
        break
      case 'console':
        // 直接 API key 用户（非 Bedrock/Vertex/Foundry，非自定义 base URL）
        if (!isClaudeAISubscriber() && !isUsing3PServices() && isFirstPartyAnthropicBaseUrl())
          return true
        break
    }
  }
  return false
}
```

---

## 6. 关键命令实现解析

### `/clear` — 清空会话

```typescript
// src/commands/clear/clear.ts
export const call: LocalCommandCall = async (_, context) => {
  await clearConversation(context)
  return { type: 'text', value: '' }
}
```

`clearConversation` 做了大量工作（`conversation.ts`）：

```typescript
// src/commands/clear/conversation.ts（简化）

export async function clearConversation(context): Promise<void> {
  // 1. 执行 SessionEnd hooks（含超时限制，默认 1.5s）
  await executeSessionEndHooks('clear', { ... })

  // 2. 向后端发送 cache eviction hint（让推理层可以释放 KV cache）
  logEvent('tengu_cache_eviction_hint', { last_request_id: ... })

  // 3. 找出需要保留的后台任务（Ctrl+B 开的任务不清除）
  const preservedAgentIds = new Set<string>()
  for (const task of Object.values(getAppState().tasks)) {
    if (!shouldKillTask(task)) preservedAgentIds.add(task.agentId)
  }

  // 4. 清空消息、缓存、文件状态
  setMessages(() => [])
  clearSessionCaches(preservedAgentIds)
  readFileState.clear()

  // 5. 更新 AppState：清除任务、归因状态、文件历史、MCP 状态
  setAppState(prev => ({
    ...prev,
    tasks: nextTasks,
    attribution: createEmptyAttributionState(),
    mcp: { clients: [], tools: [], commands: [], resources: {}, pluginReconnectKey: prev.mcp.pluginReconnectKey },
  }))

  // 6. 生成新 session ID（保留旧 ID 作为 parent，用于分析追踪）
  regenerateSessionId({ setCurrentAsParent: true })

  // 7. 执行 SessionStart hooks（为新会话初始化环境）
  const hookMessages = await processSessionStartHooks('clear')
}
```

**设计亮点**：`/clear` 是一个完整的"会话重置"流程，不是简单地清空消息数组，而是处理了后台任务保留、cache 释放提示、session ID 轮换等复杂状态。

---

### `/compact` — 压缩上下文

```typescript
// src/commands/compact/compact.ts（简化）

export const call: LocalCommandCall = async (args, context) => {
  let { messages } = context
  // 排除 REPL scrollback 中手动截断的消息（用户用 /snip 裁剪的部分）
  messages = getMessagesAfterCompactBoundary(messages)

  const customInstructions = args.trim()

  // 路径 1: Session Memory 压缩（轻量级，无自定义指令时优先）
  if (!customInstructions) {
    const sessionMemoryResult = await trySessionMemoryCompaction(messages, context.agentId)
    if (sessionMemoryResult) {
      // ... 清理缓存、标记压缩完成
      return { type: 'compact', compactionResult: sessionMemoryResult, ... }
    }
  }

  // 路径 2: Reactive Compact（feature flag 开启时）
  if (reactiveCompact?.isReactiveOnlyMode()) {
    return await compactViaReactive(messages, context, customInstructions, reactiveCompact)
  }

  // 路径 3: 传统 LLM 压缩
  const microcompactResult = await microcompactMessages(messages, context)  // 先做微压缩降 token
  const result = await compactConversation(
    microcompactResult.messages,
    context,
    await getCacheSharingParams(context, ...),
    false,
    customInstructions,
    false,
  )

  return { type: 'compact', compactionResult: result, ... }
}
```

**三条压缩路径**：
| 路径 | 触发条件 | 特点 |
|------|----------|------|
| Session Memory 压缩 | 无自定义指令，且 session memory 可用 | 轻量、不调用 LLM |
| Reactive Compact | `feature('REACTIVE_COMPACT')` + `isReactiveOnlyMode()` | 响应式，并行执行 hooks |
| 传统 LLM 压缩 | 兜底 | 先微压缩再 LLM 总结 |

---

### `/config` — 打开设置界面（local-jsx 类型）

```typescript
// src/commands/config/config.tsx
export const call: LocalJSXCommandCall = async (onDone, context) => {
  // 返回 React 节点，由 Ink 渲染在终端中
  return <Settings onClose={onDone} context={context} defaultTab="Config" />
}
```

local-jsx 命令返回的是一个 React 元素，REPL 层负责将其挂载到终端 UI 中。命令完成后调用 `onDone()` 回调通知系统关闭 UI。

---

## 7. Skill 加载：从文件系统读取 `.md` 文件

`PromptCommand` 中来源为 `'skills'` 或 `'bundled'` 的命令是从 Markdown 文件解析出来的：

```
目录搜索顺序（优先级从高到低）：
1. ~/.claude/skills/         (全局技能目录)
2. <project>/.claude/skills/ (项目级技能目录)
3. <parent_dirs>/.claude/commands/ (DEPRECATED，向后兼容)
4. getBundledSkills()         (内置捆绑技能)
5. getBuiltinPluginSkillCommands() (内置插件技能)
6. getPluginCommands()        (外部插件命令)
```

每个 `.md` 文件的 frontmatter 决定命令的元数据：

```markdown
---
description: 分析代码质量并给出改进建议
argument-hint: <file_path>
allowed-tools: Read,Bash
model: claude-opus-4-5
effort: high
paths: ["**/*.ts", "**/*.tsx"]  # 只在接触 TS 文件后才出现
---

请分析以下文件的代码质量：$ARGUMENTS
```

`loadedFrom: 'skills'` 的命令通过 `getPromptForCommand()` 将 Markdown 内容展开为 `ContentBlockParam[]` 后发给模型。

---

## 8. 远程/Bridge 安全过滤

```typescript
// src/commands.ts

// 远程模式（--remote）下只保留这些命令
export const REMOTE_SAFE_COMMANDS: Set<Command> = new Set([
  session, exit, clear, help, theme, color, vim, cost, usage,
  copy, btw, feedback, plan, keybindings, statusline, stickers, mobile,
])

// Bridge 模式（手机/Web 客户端通过 Bridge 发来的命令）只允许：
export const BRIDGE_SAFE_COMMANDS: Set<Command> = new Set([
  compact,  // 手机上压缩上下文 — 有用
  clear,    // 清屏
  cost,     // 查费用
  summary,  // 总结对话
  releaseNotes, // 查更新日志
  files,    // 查跟踪文件
].filter(Boolean))

// local-jsx 命令（渲染 Ink UI）始终阻止来自 Bridge，
// prompt 命令展开为文本所以始终安全
export function isBridgeSafeCommand(cmd: Command): boolean {
  if (cmd.type === 'local-jsx') return false  // UI 命令不能远程执行
  if (cmd.type === 'prompt') return true       // 技能/Prompt 命令安全
  return BRIDGE_SAFE_COMMANDS.has(cmd)
}
```

---

## 9. 命令查找与执行

```typescript
// src/commands.ts

export function findCommand(commandName: string, commands: Command[]): Command | undefined {
  return commands.find(
    _ =>
      _.name === commandName ||
      getCommandName(_) === commandName ||  // 支持 userFacingName() 重写
      _.aliases?.includes(commandName),     // 支持别名
  )
}

// 格式化命令描述（用于 typeahead / help 显示）
export function formatDescriptionWithSource(cmd: Command): string {
  if (cmd.type !== 'prompt') return cmd.description
  if (cmd.source === 'plugin') {
    const pluginName = cmd.pluginInfo?.pluginManifest.name
    return pluginName ? `(${pluginName}) ${cmd.description}` : `${cmd.description} (plugin)`
  }
  if (cmd.source === 'bundled') return `${cmd.description} (bundled)`
  return `${cmd.description} (${getSettingSourceName(cmd.source)})`
}
```

---

## 10. 缓存与缓存失效

```typescript
// 所有命令加载开销大（磁盘 I/O + 动态 import），所以按 cwd memoize
const loadAllCommands = memoize(async (cwd: string): Promise<Command[]> => { ... })

// 动态技能（paths 匹配触发）需要独立失效：
export function clearCommandMemoizationCaches(): void {
  loadAllCommands.cache?.clear?.()
  getSkillToolCommands.cache?.clear?.()
  getSlashCommandToolSkills.cache?.clear?.()
  clearSkillIndexCache?.()  // 技能搜索索引也需要清
}

// 完整清理（包括插件技能缓存）：
export function clearCommandsCache(): void {
  clearCommandMemoizationCaches()
  clearPluginCommandCache()
  clearPluginSkillsCache()
  clearSkillCaches()
}
```

触发 `clearCommandsCache()` 的时机：
- `/reload-plugins` 命令执行后
- 插件安装/卸载后
- `clearConversation()` 中（`/clear` 时）

---

## 11. 设计思路总结

1. **三类命令类型**设计精妙：`local`（返回文本）/ `local-jsx`（渲染 UI）/ `prompt`（展开为技能 Prompt）各司其职，类型系统在编译期区分用途。

2. **懒加载 + memoize 双重优化**：命令实现通过 `load()` 懒加载避免启动时加载所有依赖；`loadAllCommands` 按 cwd memoize 避免重复磁盘 I/O。

3. **可用性 vs. 开关分离**：`availability`（静态鉴权要求）和 `isEnabled()`（动态 Feature Flag）是两个独立维度，互不干扰，让命令的"谁能用"和"现在开没开"可以独立变化。

4. **`/clear` 是完整会话重置**：不只是清空消息数组，还触发 hooks、cache eviction hint、session ID 轮换、后台任务保留逻辑，确保状态一致性。

5. **Skill 即 Markdown**：用户可以在 `~/.claude/skills/` 下放 `.md` 文件，系统自动解析 frontmatter 转为可调用的技能命令，零代码扩展 Claude Code 能力。

---

## 12. 与其他模块的关联

| 依赖模块 | 关联方式 |
|----------|----------|
| `Tool.ts` / `ToolUseContext` | `LocalJSXCommandContext extends ToolUseContext`，命令执行需要工具上下文 |
| `services/compact/` | `/compact` 命令调用三条压缩路径 |
| `services/analytics/` | `/clear` 触发 `tengu_cache_eviction_hint` 事件 |
| `skills/loadSkillsDir.ts` | 从文件系统加载 `PromptCommand`（技能） |
| `utils/plugins/loadPluginCommands.ts` | 从插件系统加载额外命令 |
| `bootstrap/state.ts` | `/clear` 调用 `regenerateSessionId()` 轮换会话 |
| `components/Settings/` | `/config` 渲染设置 UI 组件 |
| `utils/hooks.ts` | `/clear`、`/compact` 都触发相应生命周期 hooks |
