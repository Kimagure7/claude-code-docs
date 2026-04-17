# 13. 多 Agent 协调系统

## 1. 模块职责

多 Agent 协调系统是 Claude Code 的"任务分发与并行执行引擎"——主 Agent 可以动态创建子 Agent，将复杂任务拆解为并行工作流，子 Agent 携带独立的工具权限、隔离的执行上下文，通过异步通知机制将结果汇聚回主 Agent。

**为什么需要这个模块**：单个 Agent 受限于对话轮次和串行执行，无法高效完成大型任务（如"研究 A 同时实现 B"）。多 Agent 模式通过分工让不同的子 Agent 同步/异步并行执行，再由 Coordinator 汇总。

---

## 2. 核心文件地图

| 文件 | 行数 | 职责 |
|------|------|------|
| `tools/AgentTool/AgentTool.tsx` | 1397 | 核心 AgentTool 实现，子 Agent 创建、执行、结果收集 |
| `tools/AgentTool/agentToolUtils.ts` | 686 | 工具过滤、异步生命周期、结果 finalize |
| `tools/SendMessageTool/SendMessageTool.ts` | 917 | 向已启动的子 Agent 发送继续消息，含 Teammate 路由 |
| `coordinator/coordinatorMode.ts` | 369 | Coordinator 模式检测、系统提示、用户上下文注入 |
| `tasks/LocalAgentTask/LocalAgentTask.tsx` | ~600 | 后台/前台任务注册、进度跟踪、通知排队 |
| `tools/shared/spawnMultiAgent.ts` | 1093 | Teammate 多窗格模式，通过 Tmux 创建队友 |
| `constants/tools.ts` | 112 | 工具权限白名单/黑名单常量 |

---

## 3. 关键数据结构

### 3.1 AgentTool 输入 Schema

```typescript
// tools/AgentTool/AgentTool.tsx:83-157
const inputSchema = z.object({
  description: z.string(),           // 任务简短描述 (3-5 词)
  prompt: z.string(),                // 传递给子 Agent 的完整任务
  subagent_type: z.string().optional(), // Agent 类型，如 'Explore'、'Plan'
  model: z.enum(['sonnet', 'opus', 'haiku']).optional(), // 模型覆盖
  run_in_background: z.boolean().optional(), // 强制异步执行
  name: z.string().optional(),       // 子 Agent 可寻址名称 (SendMessage 路由)
  team_name: z.string().optional(),  // 多 Agent 队伍名称
  mode: z.enum(['plan', 'review', 'acceptEdits']).optional(), // 权限模式
  isolation: z.enum(['worktree', 'remote']).optional(), // 文件系统隔离
  cwd: z.string().optional(),        // 工作目录覆盖
});
```

### 3.2 AgentTool 输出 Schema（多态）

```typescript
// 同步完成
{
  status: 'completed',
  content: string,           // 子 Agent 最终回复
  totalToolUseCount: number, // 工具调用总次数
  totalDurationMs: number,   // 执行耗时 (ms)
  prompt: string
}

// 异步启动 (run_in_background=true 或 Coordinator 模式)
{
  status: 'async_launched',
  agentId: string,           // 后台任务 ID，供 SendMessage 路由
  description: string,
  prompt: string,
  outputFile: string,        // 进度文件路径，可实时读取
  canReadOutputFile: boolean
}

// Teammate 模式（多窗格）
{
  status: 'teammate_spawned',
  teammate_id: string,
  agent_id: string,
  name: string,
  tmux_session_name: string,
  tmux_window_name: string,
  tmux_pane_id: string
}
```

### 3.3 异步 Agent 通知 XML

```xml
<!-- tasks/LocalAgentTask/LocalAgentTask.tsx:197-250 -->
<task-notification>
  <task-id>{agentId}</task-id>
  <status>completed|failed|killed</status>
  <summary>Agent description - status</summary>
  <result>{子 Agent 最终文本回复}</result>
  <usage>
    <total_tokens>N</total_tokens>
    <tool_uses>N</tool_uses>
    <duration_ms>N</duration_ms>
  </usage>
  <worktree>
    <path>{worktree_path}</path>     <!-- 若使用 Git Worktree 隔离 -->
    <branch>{branch_name}</branch>
  </worktree>
</task-notification>
```

---

## 4. 执行流程图

### 4.1 子 Agent 创建决策树

```
AgentTool.call(params)
       │
       ├─ team_name && name ──► spawnTeammate() ──► status: teammate_spawned
       │
       ├─ !subagent_type && isForkSubagentEnabled()
       │         └──► Fork Agent (继承父上下文，强制异步)
       │
       ├─ isolation === 'remote' ──► teleportToRemote() ──► status: remote_launched
       │
       └─ 标准路径
             │
             ├─ 从 activeAgents 查找 subagent_type
             ├─ filterDeniedAgents() 权限检查
             ├─ 等待 MCP 服务器就绪（最多 30 秒）
             ├─ isolation === 'worktree' ──► createAgentWorktree()
             ├─ 构建独立工具池 workerPermissionContext
             │
             └─ shouldRunAsync ?
                    ├─ YES ──► registerAsyncAgent() → fire-and-forget → status: async_launched
                    └─ NO  ──► registerAgentForeground() → 消息迭代循环
                                       │
                                       └─ 用户触发背景化 ──► 动态转换为 async_launched
```

### 4.2 异步结果汇聚流程

```
后台子 Agent 执行中
       │
       ▼
runAsyncAgentLifecycle()
       │
       ├─ 每条消息 ──► updateAsyncAgentProgress() ──► 实时进度更新
       │
       └─ 执行完毕
             ├─ finalizeAgentTool() ──► 提取最终回复、统计
             ├─ enqueueAgentNotification() ──► 构建 <task-notification> XML
             └─ enqueuePendingNotification()
                    │
                    ▼
             主 Agent 下一轮对话
             ◄──── 接收 <task-notification> 用户消息
```

---

## 5. 关键代码解析

### 5.1 同步/异步执行决定逻辑

```typescript
// tools/AgentTool/AgentTool.tsx:706-715
const shouldRunAsync = (
  run_in_background === true ||           // 用户显式要求后台化
  selectedAgent.background === true ||    // Agent 定义强制后台
  isCoordinator ||                        // Coordinator 模式强制异步
  forceAsync ||                           // Fork 实验强制异步
  assistantForceAsync ||                  // KAIROS 模式强制异步
  (proactiveModule?.isProactiveActive() ?? false)  // 主动模块活跃
) && !isBackgroundTasksDisabled;          // 全局后台任务未禁用
```

**设计亮点**：不是一个开关，而是多个条件的"或"，任一满足即异步——既支持用户主动要求，也支持 Agent 定义内置，还支持运行模式自动推断。

### 5.2 同步前台执行：消息迭代与动态背景化

```typescript
// tools/AgentTool/AgentTool.tsx:1013-1060
const agentIterator = runAgent({ ...runAgentParams })[Symbol.asyncIterator]();

while (true) {
  const elapsed = Date.now() - agentStartTime;

  // 2 秒后展示背景化提示 UI
  if (!backgroundHintShown && elapsed >= 2000 && toolUseContext.setToolJSX) {
    toolUseContext.setToolJSX({
      jsx: <BackgroundHint />,
      shouldHidePromptInput: false,
      shouldContinueAnimation: true,
      showSpinner: true
    });
    backgroundHintShown = true;
  }

  // 关键：竞速 —— 下一条消息 vs 背景化信号
  const nextMessagePromise = agentIterator.next();
  const raceResult = backgroundPromise
    ? await Promise.race([
        nextMessagePromise.then(r => ({ type: 'message', result: r })),
        backgroundPromise  // 用户按下 ESC 或超时自动背景化
      ])
    : { type: 'message', result: await nextMessagePromise };

  if (raceResult.type === 'background' && foregroundTaskId) {
    // 触发动态背景化转换：清理前台迭代器，重新以异步模式继续
    wasBackgrounded = true;
    // ... 启动后台闭包，立即返回 async_launched
    return { data: { isAsync: true, status: 'async_launched', ... } };
  }

  if (raceResult.result.done) break;
  agentMessages.push(raceResult.result.value);
}
```

**设计亮点**：`Promise.race()` 实现"随时可中断"——子 Agent 在前台运行时，任何时刻用户可以按下 ESC 将其转为后台，无需重启，已执行的消息完整保留并继续。

### 5.3 子 Agent 工具权限隔离

```typescript
// tools/AgentTool/AgentTool.tsx:568-577
const workerPermissionContext = {
  ...appState.toolPermissionContext,
  mode: selectedAgent.permissionMode ?? 'acceptEdits'  // 每个 Agent 有独立权限模式
};
const workerTools = assembleToolPool(workerPermissionContext, appState.mcp.tools);

// ...
availableTools: isForkPath
  ? toolUseContext.options.tools   // Fork：使用父工具池（保证提示缓存一致）
  : workerTools,                   // 标准：完全独立的工具池
```

```typescript
// tools/AgentTool/agentToolUtils.ts:60-100
export function filterToolsForAgent({ tools, isBuiltIn, isAsync, permissionMode }) {
  return tools.filter(tool => {
    if (tool.name.startsWith('mcp__')) return true;      // MCP 工具始终允许
    if (ALL_AGENT_DISALLOWED_TOOLS.has(tool.name)) return false; // 全局禁用
    if (!isBuiltIn && CUSTOM_AGENT_DISALLOWED_TOOLS.has(tool.name)) return false;
    if (isAsync && !ASYNC_AGENT_ALLOWED_TOOLS.has(tool.name)) return false; // 异步严格限制
    return true;
  });
}
```

**三级过滤逻辑**：
- `ALL_AGENT_DISALLOWED_TOOLS`：所有子 Agent 均不可用（如 `Agent` 工具本身——防止无限嵌套）
- `CUSTOM_AGENT_DISALLOWED_TOOLS`：自定义 Agent 额外限制（无法生成子 Agent）
- `ASYNC_AGENT_ALLOWED_TOOLS`：异步 Agent 只能访问白名单工具（Bash、Read/Write、Grep、Glob 等）

### 5.4 Coordinator 模式：多 Worker 调度

```typescript
// coordinator/coordinatorMode.ts:85-138
export function getCoordinatorUserContext(mcpClients, scratchpadDir) {
  if (!isCoordinatorMode()) return {};

  // 告知 Coordinator 它的 Worker 有哪些工具可用
  const workerTools = Array.from(ASYNC_AGENT_ALLOWED_TOOLS)
    .filter(name => !INTERNAL_WORKER_TOOLS.has(name))
    .sort()
    .join(', ');

  let content = `Workers spawned via the ${AGENT_TOOL_NAME} tool have access to these tools: ${workerTools}`;

  // MCP 服务器也透传给 Worker
  if (mcpClients.length > 0) {
    content += `\n\nWorkers also have access to MCP tools: ${mcpClients.map(c => c.name).join(', ')}`;
  }

  // 共享便签本：跨 Worker 持久化知识
  if (scratchpadDir && isScratchpadGateEnabled()) {
    content += `\n\nScratchpad directory: ${scratchpadDir}\nWorkers can read and write here without permission prompts...`;
  }

  return { workerToolsContext: content };
}
```

**Coordinator 工作流设计**（系统提示 coordinatorMode.ts:141-231）：
```
Research Phase
  └─ spawn 多个 Worker 并行探索代码库
Synthesis Phase  
  └─ Coordinator 汇总所有 <task-notification>，整合发现
Implementation Phase
  └─ spawn Worker(s) 并行实现，写入序列化
Verification Phase
  └─ spawn 验证 Worker，检查结果
```

### 5.5 spawnTeammate：Teammate 多窗格实现

Teammate 模式是多 Agent 协调的最高级形态——每个 Teammate 是一个独立的 Claude Code 进程，运行在独立的 Tmux 窗格（或 iTerm2 Split Pane）中，通过邮箱文件与 Team Lead 异步通信。

```typescript
// tools/shared/spawnMultiAgent.ts:310-400（handleSpawnSplitPane 核心流程）
async function handleSpawnSplitPane(input, context) {
  const { name, prompt, agent_type, cwd, plan_mode_required } = input;

  // 1. 解析模型：'inherit' → 继承 leader 模型；undefined → 默认 Opus
  const model = resolveTeammateModel(input.model, getAppState().mainLoopModel);

  // 2. 检测后端（Tmux / iTerm2 / 外部 Swarm Session）
  let detectionResult = await detectAndGetBackend();

  // 3. 若检测到 iTerm2 但 it2 未配置，展示 UI 引导用户安装或切换到 Tmux
  if (detectionResult.needsIt2Setup && context.setToolJSX) {
    const setupResult = await new Promise(resolve => {
      context.setToolJSX({ jsx: React.createElement(It2SetupPrompt, { onDone: resolve }) });
    });
    if (setupResult === 'cancelled') throw new Error('Teammate spawn cancelled');
    resetBackendDetection();
    detectionResult = await detectAndGetBackend();  // 重新检测
  }

  // 4. 在 Swarm View 中创建窗格（自动处理 tmux 内/外两种情况）
  const { paneId, isFirstTeammate } = await createTeammatePaneInSwarmView(sanitizedName, teammateColor);

  // 5. 构造启动命令：传入身份 CLI 参数，继承 permission-mode/model/settings
  const spawnCommand = `cd ${quote([workingDir])} && env ${envStr} ${quote([binaryPath])} \
    --agent-id ${quote([teammateId])} \
    --agent-name ${quote([sanitizedName])} \
    --team-name ${quote([teamName])} \
    --agent-color ${quote([teammateColor])} \
    --parent-session-id ${quote([getSessionId()])} \
    ${inheritedFlags}`;

  await sendCommandToPane(paneId, spawnCommand, !insideTmux);

  // 6. 注册到 AppState.teamContext，让 Tasks Pill 显示队友状态
  setAppState(prev => ({
    ...prev,
    teamContext: {
      ...prev.teamContext,
      teammates: {
        ...(prev.teamContext?.teammates || {}),
        [teammateId]: { name: sanitizedName, color: teammateColor, tmuxPaneId: paneId, ... }
      }
    }
  }));

  // 7. 写入团队文件，记录成员信息（持久化到磁盘）
  teamFile.members.push({ agentId: teammateId, name: sanitizedName, ... });
  await writeTeamFileAsync(teamName, teamFile);

  // 8. 通过邮箱文件发送初始 prompt（Teammate 启动后自动从邮箱取第一条消息）
  await writeToMailbox(sanitizedName, { from: TEAM_LEAD_NAME, text: prompt }, teamName);
}
```

**三种后端的启动差异**：

```
detectAndGetBackend()
       │
       ├─ 已在 Tmux 内 ──► split-window ──► 左侧 Leader，右侧 Teammate 竖排
       │
       ├─ iTerm2 可用 ──► it2 split-h ──► 原生 iTerm2 分屏，视觉更美观
       │
       └─ 两者都不可用 ──► 创建独立 claude-swarm Session ──► tiled 平铺布局
```

**模型继承链 `resolveTeammateModel`**：

```typescript
// tools/shared/spawnMultiAgent.ts:90-100
export function resolveTeammateModel(inputModel, leaderModel) {
  if (inputModel === 'inherit') {
    return leaderModel ?? getDefaultTeammateModel(leaderModel);  // 'inherit' → 复用 leader 模型
  }
  return inputModel ?? getDefaultTeammateModel(leaderModel);  // undefined → 使用全局 teammateDefaultModel
}
// 优先级：显式指定 > 'inherit'（leader 模型） > 全局配置 > 硬编码兜底
```

**邮箱通信机制**（Teammate 独立进程的唯一初始化入口）：

```
Team Lead 调用 writeToMailbox(name, { from, text }, teamName)
       │
       ▼
  写入 ~/.claude/team/{teamName}/mailbox/{name}/inbox/*.json
       │
       ▼
  Teammate 进程启动后的邮箱轮询器自动检测新文件
       │
       ▼
  取出消息作为第一轮 UserMessage 提交给自身的 query 引擎
```

**重名去重**（`generateUniqueTeammateName`）：

```typescript
// tools/shared/spawnMultiAgent.ts:265-295
// 若 "tester" 已存在，自动生成 "tester-2"、"tester-3"...
export async function generateUniqueTeammateName(baseName, teamName) {
  const teamFile = await readTeamFileAsync(teamName);
  const existingNames = new Set(teamFile.members.map(m => m.name.toLowerCase()));
  if (!existingNames.has(baseName.toLowerCase())) return baseName;
  let suffix = 2;
  while (existingNames.has(`${baseName}-${suffix}`.toLowerCase())) suffix++;
  return `${baseName}-${suffix}`;
}
```

### 5.6 SendMessageTool：向存活子 Agent 继续发送消息

```typescript
// tools/SendMessageTool/SendMessageTool.ts:102-139
async function handleMessage(recipientName, messageContent, summaryText, toolUseContext) {
  const appState = toolUseContext.getAppState();

  // 路由到本地后台 Agent 任务
  const task = appState.tasks[recipientName];
  if (isLocalAgentTask(task)) {
    queuePendingMessage(recipientName, messageContent, toolUseContext.setAppState);
    return { success: true, message: `Queued message for agent ${recipientName}...` };
  }

  // 路由到 Teammate（多窗格模式）
  if (isTeammate() && !isTeamLead()) { /* ... */ }

  // 广播到所有队友
  if (recipientName === '*') { /* ... */ }
}
```

```typescript
// tasks/LocalAgentTask/LocalAgentTask.tsx:162-191
// 消息入队（SendMessage 写入）
export function queuePendingMessage(taskId, msg, setAppState) {
  updateTaskState(taskId, setAppState, task => ({
    ...task,
    pendingMessages: [...task.pendingMessages, msg]  // 追加到队列
  }));
}

// 消息出队（runAgent 在每轮开始时吸收）
export function drainPendingMessages(taskId, setAppState): string[] {
  let drained: string[] = [];
  updateTaskState(taskId, setAppState, task => {
    drained = [...task.pendingMessages];
    return { ...task, pendingMessages: [] };  // 清空队列
  });
  return drained;
}
```

**设计亮点**：主 Agent 发出 `SendMessage` 后立即继续，消息以"邮件"方式排队，子 Agent 下一轮执行时自动取出——无需同步等待，完全解耦。

---

## 6. 设计思路总结

### 6.1 四种执行模式分层

| 模式 | 触发条件 | 特性 | 适用场景 |
|------|--------|------|--------|
| **同步前台** | 默认 | 阻塞主线程，可随时背景化 | 短任务、需要立即结果 |
| **异步后台** | `run_in_background=true` 或 Coordinator | 立即返回，通知机制汇结果 | 长任务、并行研究 |
| **Fork 模式** | 无 `subagent_type` 且功能开启 | 继承父上下文，缓存对齐 | 需要父对话上下文的分叉任务 |
| **Teammate 多窗格** | `team_name && name` | 独立 Tmux 窗格，可视化 | 协作型多步骤项目 |

### 6.2 缓存优化：Fork 的字节级系统提示匹配

Fork 子 Agent 强制使用父 Agent 的 `renderedSystemPrompt`（字节精确），这是精心设计的——若子 Agent 重新生成系统提示，哪怕内容相同但顺序变化，会导致 Anthropic API 的提示缓存 MISS，浪费大量 Token 费用。

```typescript
// 优先使用已渲染的系统提示，确保缓存 HIT
if (toolUseContext.renderedSystemPrompt) {
  forkParentSystemPrompt = toolUseContext.renderedSystemPrompt; // 字节精确
} else {
  forkParentSystemPrompt = buildEffectiveSystemPrompt({ ... }); // 重计算，可能 MISS
}
```

### 6.3 动态背景化：无损中断

子 Agent 在前台执行时，`Promise.race()` 让其随时可被"转为后台"而不中断执行：
1. 前台迭代器被 `.return()` 优雅关闭
2. 已收集的消息完整转移
3. 新的后台迭代器从 `runAgent()` 重新启动
4. 主线程立即返回 `async_launched`，子 Agent 在后台继续

### 6.4 三级权限隔离防止滥用

- 子 Agent 无法再生成子 Agent（`Agent` 工具在 `ALL_AGENT_DISALLOWED_TOOLS` 中）
- 自定义 Agent 比内置 Agent 受到更严格限制
- 异步 Agent 只能访问文件操作和搜索类工具，无法执行交互型操作

### 6.5 消息队列解耦设计

`SendMessage → queuePendingMessage → drainPendingMessages` 三步构成异步邮件系统：
- 发送者不需要等待接收者
- 接收者（子 Agent）在自己的执行周期中自然取消
- 支持广播（`*`）和点对点两种路由

---

## 7. 与其他模块的联系

| 依赖模块 | 联系方式 | 说明 |
|--------|--------|------|
| **query.ts / QueryEngine.ts** | `querySource` 路由 | Agent 消息持久化到 Sidechain，主线程与子线程通过 agentId 隔离消息队列 |
| **tools/AgentTool/agentToolUtils.ts** | 工具过滤、生命周期 | `filterToolsForAgent()`、`runAsyncAgentLifecycle()`、`finalizeAgentTool()` |
| **tasks/LocalAgentTask** | 任务注册与通知 | `registerAsyncAgent()`、`enqueueAgentNotification()`、进度跟踪 |
| **services/mcp/** | 工具继承 | 子 Agent 的工具池包含父 Agent 的 MCP 工具 |
| **06-auth-config（PermissionMode）** | 权限上下文 | 每个子 Agent 有独立的 `permissionMode`（acceptEdits / plan / review） |
| **02-query-engine（runAgent）** | 执行引擎 | `AgentTool` 调用 `runAgent()` 创建子 Agent 的消息迭代器 |
| **components/App.tsx** | UI 反馈 | `setToolJSX(<BackgroundHint />)` 在前台执行超过 2 秒时展示提示 |
