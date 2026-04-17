# 03 - 工具系统设计 (Tool System)

## 本文核心问题

工具系统要解决的问题：**让 AI 模型能调用几十种异构的系统操作，同时保证安全性、可扩展性、可渲染性**。

这三个目标天然冲突——安全需要集中控制，扩展需要分散实现，渲染需要每个工具有独立 UI。工具系统的所有设计都在化解这个矛盾。

---

## 1. 接口契约：一个工具是什么

每个工具实现 `Tool<Input, Output>` 接口，大约有这些职责分组：

| 分组 | 方法 | 职责 |
|------|------|------|
| 执行 | `call()` | 实际干活 |
| 验证 | `validateInput()` + `checkPermissions()` | 两阶段安全检查 |
| 特征 | `isReadOnly()` / `isConcurrencySafe()` / `isDestructive()` | 告诉调度器自己的"脾气" |
| 渲染 | `renderToolUseMessage()` 等4个 `render*` | 各生命周期的 UI |
| 元信息 | `inputSchema` / `prompt()` / `maxResultSizeChars` | Schema + 系统提示注入 |

接口的关键设计思路：**工具描述自己，调度器决策**。工具不直接判断"我能不能跑"，而是通过特征标记告诉外部——并发安全吗？只读吗？会破坏数据吗？外部根据这些标记统一决策调度策略。

`buildTool()` 是工厂函数，给所有工具填充安全的默认值：默认非并发安全、默认有写操作、默认允许执行。新工具只需覆盖自己关心的字段，不需要实现完整接口。这是经典的**默认 fail-safe + 按需覆盖**模式。

---

## 2. Schema 的双重身份

`inputSchema` 是每个工具最核心的定义，它同时承担两个完全不同的角色：

1. **运行时校验**：Claude 返回 tool_use 参数后，用 Zod 校验入参合法性
2. **API 工具定义**：序列化为 JSON Schema，传给 Claude API 作为工具描述

一份 Schema，两种用途。这意味着写 Schema 的时候就在写文档——字段的 `.describe()` 会直接出现在模型看到的工具说明里，影响模型的调用行为。

`lazySchema()` 包装的延迟初始化是工程优化：模块加载时不执行 Zod 构建，减少启动耗时。这在工具数量达到 40+ 时有实际意义。

---

## 3. 工具执行流程

一次完整的工具调用经历：

```
Claude API 返回 tool_use
    → validateInput()      第一关：格式/语义验证
    → checkPermissions()   第二关：权限询问（可能弹 UI）
    → canUseTool() hooks   第三关：PreToolUse 钩子
    → call()               执行
    → ToolResult           返回（含可能注入的新消息）
    → 序列化追加到消息历史
```

两个设计值得注意：

**`onProgress` 回调**：`call()` 接受一个进度回调，工具可以在执行中途推送进度，驱动 `renderToolUseProgressMessage()` 实时更新 UI。这让长耗时工具（编译、搜索）有了流式反馈，而不是执行完才刷新。

具体机制：工具在 `call()` 执行过程中随时调用 `onProgress({ toolUseID, data })`，框架收到后生成一条 `ProgressMessage` 追加到消息列表，UI 侧的 `renderToolUseProgressMessage(progressMessages, ...)` 消费这些消息实时重新渲染。`call()` 是异步的，`onProgress` 是它和 UI 之间的"流式通道"，把执行过程中的中间状态推出来，而不是等 `Promise` resolve 才更新界面。

**`ToolResult.newMessages`**：工具可以在结果里附带额外消息注入对话历史。框架执行完工具后会自动把 `tool_use` + `tool_result` 写回对话历史（`ToolResult.data` 就是 `tool_result` 的内容），这是标准流程。`newMessages` 是追加在 `tool_result` 之后额外注入的完整消息（`toolExecution.ts` 里直接 push 进 `resultingMessages`）。

这个机制的实际用途是**注入需要特定消息格式的附加内容**，而不是简单地"多塞一些数据"。两个真实使用场景：

- **FileReadTool 读 PDF**：把 PDF 页面渲染成的 image blocks 通过 `newMessages` 注入一条 `isMeta: true` 的 user 消息——image blocks 需要在 `user` 角色的消息里发送；读普通图片时，`newMessages` 里放的是图片尺寸元数据文本，图片 base64 本身在 `data` 里
- **SkillTool**：把 Skill 执行结果作为 `isMeta: true` 的 user 消息注入，与 `tool_result` 的结构化数据分开

---

## 4. 权限系统的分离

权限检查是工具系统里最刻意做解耦的部分。工具只需实现 `checkPermissions()`，说明"我需要什么权限"，但真正的决策逻辑在外部 `permissions.ts` 里统一处理：

- allow/deny 规则匹配
- 模式判断（plan 模式、bypass 模式）
- hooks 拦截（PreToolUse）
- 子 Agent 权限隔离（`shouldAvoidPermissionPrompts`）

这样工具本身的权限逻辑保持简单，复杂的策略不扩散到每个工具里。新加权限规则只改一处，所有工具自动受益。

`ToolPermissionContext` 是不可变的（`DeepImmutable`），权限上下文在一次 query 里不会被工具修改——这是防止权限提升的关键约束。

---

## 5. 工具注册与过滤

工具池的组装分三层：

**`getAllBaseTools()`** — 声明所有工具。工具的条件加载有两种机制，需要区分：
- `feature('COORDINATOR_MODE')` 这类用 `bun:bundle` 的 feature flag，是真正的**构建时死代码消除**，未启用的代码不进打包产物
- `process.env.USER_TYPE === 'ant'` 这类是**运行时判断**，用 `require()` 懒加载，代码在二进制里但运行时不执行

此外有 `CLAUDE_CODE_SIMPLE=true` 的精简模式，`getTools()` 会只返回 `[BashTool, FileReadTool, FileEditTool]` 三个工具。

**`getTools()`** — 运行时过滤。调用每个工具的 `isEnabled()` 检查当前环境是否支持，应用 deny 规则，处理特殊模式（REPL 模式、简单模式）。

**`assembleToolPool()`** — 合并内置工具与 MCP 工具。两个分区**各自按名称排序后拼接**（内置在前、MCP 在后），而不是整体混排。原因是 Claude API 的 `system_cache_policy` 在最后一个内置工具后设置缓存断点——如果内置和 MCP 工具混排，每次 MCP 工具列表变化都会破坏内置工具的 prompt cache。保持有序的两段分区，才能让缓存命中稳定。同名工具内置优先（`uniqBy` 保留插入顺序中的第一个）。

MCP 工具通过 `MCPTool` 包装后实现相同的 `Tool` 接口，对调度层完全透明——这是标准的适配器模式。

---

## 6. 并发安全的设计思路

`isConcurrencySafe()` 标记决定工具能否与其他工具并发执行。大多数工具默认返回 `false`（保守策略）。

只读工具（GrepTool、FileReadTool）通常可以并发；写操作工具（FileEditTool、BashTool）通常不行。并发调度逻辑在 query.ts 里，工具只需声明自己的性质。

`ToolResult.contextModifier` 是一个特殊逃生口：非并发安全的工具可以在结果里附带一个上下文修改函数，执行完后修改全局 context。并发工具不能用这个——并发情况下多个 contextModifier 同时执行会产生竞争条件。

---

## 7. 几个反直觉的设计

**FileEditTool 用字符串替换而非行号编辑**：模型在多步骤编辑时，行号会随着前序编辑偏移，导致后续编辑错位。字符串匹配没有这个问题——只要 `old_string` 在文件里唯一存在，替换就是确定性的。

**GrepTool 底层调 ripgrep**：不是直接用 Node.js 的文件 API，而是 fork 一个 rg 进程。原因是 ripgrep 是 Rust 实现，速度和大文件处理能力远超 JS，对大型代码库的搜索体验差距明显。

**BashTool 的后台任务**：有三种触发方式，本质都是让长时命令不阻塞 Agent loop：

- **显式后台**：模型设置 `run_in_background: true`，`call()` 立即调用 `spawnShellTask()` 把命令注册为 `LocalShellTask`，返回一个 `backgroundTaskId`，`tool_result` 只有这个 ID
- **超时自动后台**：命令跑太久触发 timeout，自动转为后台任务继续执行，`call()` 不再阻塞（`sleep` 等命令被排除在外）
- **Assistant 模式自动后台**（KAIROS feature）：主线程命令超过 15 秒（`ASSISTANT_BLOCKING_BUDGET_MS`）未完成，自动后台，防止 Agent 卡死等单个命令

`TaskOutputTool` 用 `task_id` 取结果，`block: false` 立即返回当前状态，`block: true` 等待完成再返回。模型可以先并发启动多个后台命令，再逐一收集结果。

---

## 8. AgentTool：递归能力

AgentTool 是工具系统里最特殊的工具——它让工具调用变成递归结构：Agent 调用工具，工具内部启动子 Agent，子 Agent 再调用工具。

子 Agent 有独立的工具集和上下文，通过 worktree 隔离可以在 Git 工作树副本里操作，不影响主工作区。完成后可以 merge 回主分支，也可以直接丢弃。

这个设计的核心价值：**隔离风险**。探索性任务、实验性修改可以放到子 Agent 的沙箱里跑，失败了不污染主状态。成功了再合并。

---

## 9. 案例：启动一个持续输出 log 的后端服务

用户让 Claude 启动 `npm run dev`，以下是完整流程：

**第一步：模型发出 tool_use**

模型知道 dev server 不会退出，主动设置 `run_in_background: true`。BashTool 的 `call()` 调用 `spawnShellTask()`，立即返回：

```
tool_result: "Command running in background with ID: task_abc123. Output is being written to: /tmp/claude/tasks/task_abc123.log"
```

进程已经在跑，stdout/stderr 直接通过文件描述符写入磁盘上的 output 文件（不经过 JS），`TaskOutput` 用共享 poller 每秒读一次文件 tail 更新 UI 进度。

**第二步：模型检查服务是否启动成功**

模型调 `TaskOutputTool`，`block: false`，拿当前已写入的输出，确认有没有 "listening on port 3000" 之类的字样。如果还没出现，可以等一会再轮询。

**第三步：服务持续运行，log 不断产生**

`LocalShellTask` 的 stall watchdog 每 5 秒检查一次输出文件大小：
- 文件还在增长（log 持续写入）→ 正常，不发通知
- 文件 45 秒没有增长，且尾部内容匹配 prompt 模式（如 `(y/n)?`、`Press Enter`）→ 向模型发一条 `task-notification`，提示命令可能在等待交互输入

**第四步：生命周期管理**

dev server 通常不会自己退出。当 Agent session 结束时，`killShellTasksForAgent()` 会扫描所有属于该 Agent 的 running 任务并逐一 kill，防止进程变成僵尸进程。如果模型想主动停止，可以用 `TaskOutputTool` 的 `kill` 或直接 BashTool 执行 kill 命令，`killTask()` 会调 `shellCommand.kill()` 并把 task 状态标记为 `killed`，同时清理磁盘上的 output 文件（`evictTaskOutput`）。

---

## 设计原则提炼

1. **接口描述能力，调度器决策**：工具通过特征标记（只读/并发安全/破坏性）暴露自己的语义，调度逻辑集中在外部
2. **Schema 即文档**：Zod Schema 同时是校验器和 API 工具定义，写 `.describe()` 就是在给模型写说明书
3. **权限集中，实现分散**：工具各自实现，权限策略统一管理，解耦扩展性与安全性
4. **构建时而非运行时的特性开关**：feature flag 做死代码消除，未启用功能不进二进制
5. **保守默认值**：`isConcurrencySafe: false`，`isReadOnly: false`——默认最严格，按需放宽
