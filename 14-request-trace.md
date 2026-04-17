# 14 - 一次请求的完整生命周期追踪

> **场景**：用户在 REPL 中输入 `fix the bug in foo.ts`，按下回车，直到 Claude 读取文件、定位 bug、执行编辑、输出结果，全程发生了什么？
>
> 本篇不讲模块功能，只讲**这条具体请求从输入到输出的完整代码执行路径**，串联 01~13 的所有知识点。

---

## 全局时间线

```
T=0ms    用户按下回车
T=1ms    REPL 捕获输入，构建 UserMessage
T=2ms    processUserInput() 处理斜杠命令检测
T=5ms    fetchSystemPromptParts() 组装系统提示
T=10ms   query() 异步生成器启动
T=12ms   microcompact / autocompact 检查（首轮通常跳过）
T=15ms   queryModel() 发起 HTTPS 流式请求到 Anthropic API
T=?ms    [等待网络 + Claude 推理，通常 1-5 秒]
T=1200ms 第一个 content_block_delta 到达，UI 开始渲染文字
T=2000ms Claude 决定调用 FileReadTool，流式返回 tool_use 块
T=2010ms validateInput() + checkPermissions() 通过
T=2015ms FileReadTool.call() 执行，读取 foo.ts
T=2020ms 工具结果追加到消息历史，第二轮 API 请求发起
T=3500ms Claude 决定调用 FileEditTool
T=3510ms checkPermissions() → 弹出权限确认 UI
T=3515ms 用户按 y 确认
T=3520ms FileEditTool.call() 执行字符串替换
T=3530ms 生成 diff，UI 渲染编辑结果
T=3600ms Claude 输出最终回复，stop_reason = 'end_turn'
T=3610ms Stop Hooks 执行（若配置）
T=3620ms query() 返回 Terminal { reason: 'completed' }
T=3625ms REPL 等待下一次输入
```

---

## 第一阶段：输入捕获（REPL → UserMessage）

### 1.1 键盘事件捕获

```
src/components/App.tsx
  └── src/components/REPL.tsx
        └── src/hooks/useKeyboardShortcuts.ts
```

用户按下回车时，`REPL.tsx` 中的 `onSubmit` 回调触发：

```typescript
// src/components/REPL.tsx（简化）
const onSubmit = async (input: string) => {
  if (!input.trim()) return

  // 1. 写入本地历史（↑ 键可回翻）
  addToHistory(input)

  // 2. 将输入传递给 QueryEngine
  void queryEngine.submitMessage(input)
}
```

### 1.2 processUserInput() — 斜杠命令检测

```typescript
// src/QueryEngine.ts → processUserInput()
// 在进入 AI 对话前，先判断是否是斜杠命令

if (input.startsWith('/')) {
  const command = findCommand(commands, input)
  if (command) {
    // 斜杠命令直接执行，不进入 AI 对话
    // 例如 /clear → 清空历史，/model → 切换模型
    return { messages: [], shouldQuery: false }
  }
}

// 普通输入 → 构建 UserMessage
const userMessage: UserMessage = {
  type: 'user',
  uuid: generateUUID(),
  timestamp: Date.now(),
  message: {
    role: 'user',
    content: [{ type: 'text', text: input }],
  },
}
return { messages: [userMessage], shouldQuery: true }
```

**我们的输入 `fix the bug in foo.ts` 不是斜杠命令**，直接被封装为 `UserMessage`，进入下一阶段。

---

## 第二阶段：系统提示组装

```
src/QueryEngine.ts → fetchSystemPromptParts()
  ├── src/utils/systemPrompt.ts         (核心 system prompt 文本)
  ├── src/context.ts → getGitStatus()   (Git 状态)
  └── src/memdir/ → getMemoryFiles()    (CLAUDE.md + 记忆文件)
```

### 2.1 fetchSystemPromptParts()

```typescript
// src/QueryEngine.ts（简化）
const { defaultSystemPrompt, userContext, systemContext } =
  await fetchSystemPromptParts({
    tools,
    mcpClients,
    cwd,
    customSystemPrompt,
    appendSystemPrompt,
  })
```

这一步并行完成三件事：

```typescript
const [gitStatus, memoryFiles, toolDescriptions] = await Promise.all([
  getGitStatus(),           // 当前 git branch、status、最近 commits
  getMemoryFiles(cwd),      // 读取 CLAUDE.md 和 ~/.claude/memory/*.md
  getToolDescriptions(tools), // 每个工具的 description() 方法
])
```

### 2.2 最终发给 Claude API 的 system prompt 结构

```
[系统基础提示]
  - Claude Code 的角色定位和行为规范
  - 工具使用规范

[环境信息]
  <system>
    Platform: darwin
    Date: 2026/04/17
    CWD: /Users/user/project
  </system>

[Git 状态]
  <git_status>
    Current branch: main
    Status: M  foo.ts
    Recent commits: ...
  </git_status>

[记忆文件]
  <memory_files>
    [CLAUDE.md 内容]
  </memory_files>

[工具描述]
  FileReadTool: A tool for reading files...
  FileEditTool: A tool for editing files...
  BashTool: ...
  ...
```

注意：`git status` 显示 `foo.ts` 有修改，这会直接帮助 Claude 定位到这个文件。

---

## 第三阶段：进入 query() 对话循环

```
src/QueryEngine.ts → submitMessage()
  └── src/query.ts → query()
        └── src/query.ts → queryLoop()
```

### 3.1 第一轮循环开始

```typescript
// src/query.ts — queryLoop() 第一次迭代

// === 压缩检查（首轮通常跳过）===
// turnCount=1，消息历史只有一条 UserMessage，无需压缩
messagesForQuery = await deps.microcompact(messages, ...)
// → 直接返回，无任何压缩

// === 调用 AI 模型 ===
for await (const event of queryModelWithStreaming({
  messages: messagesForQuery,
  systemPrompt,
  tools: assembledTools,  // 所有可用工具的 JSON Schema
  model: 'claude-sonnet-4-5',
  signal: abortController.signal,
})) {
  yield event  // 流式事件透传给 UI
}
```

---

## 第四阶段：HTTP 请求与流式响应

```
src/query.ts → queryModelWithStreaming()
  └── src/services/api/claude.ts → queryModel()
        └── src/services/api/client.ts → getAnthropicClient()
              └── @anthropic-ai/sdk → anthropic.beta.messages.stream()
```

### 4.1 构建 API 请求参数

```typescript
// src/services/api/claude.ts（简化）

const params = {
  model: 'claude-sonnet-4-5-20251101',
  messages: addCacheBreakpoints(messagesForAPI, true),
  // ↑ 在合适位置插入 cache_control: { type: 'ephemeral' }
  // 命中 Prompt Cache 可节省约 90% 的输入 token 费用

  system: systemPrompt,  // 上面组装好的系统提示

  tools: [
    // 每个工具被序列化为 Anthropic API 格式
    {
      name: 'Read',
      description: 'A tool for reading files...',
      input_schema: { type: 'object', properties: { file_path: ... } }
    },
    {
      name: 'str_replace_based_edit_tool',
      description: 'A tool for editing files...',
      input_schema: { ... }
    },
    // ... 其余工具
  ],

  max_tokens: 32000,
  stream: true,
}

stream = await anthropic.beta.messages.stream(params)
```

### 4.2 流式事件的实时处理

网络请求发出后，Anthropic 服务器开始推送 Server-Sent Events：

```
← message_start        { usage: { input_tokens: 1842 } }
← content_block_start  { index: 0, content_block: { type: 'text' } }
← content_block_delta  { delta: { text: "I'll " } }
← content_block_delta  { delta: { text: "look at " } }
← content_block_delta  { delta: { text: "foo.ts " } }
← content_block_delta  { delta: { text: "to find " } }
← content_block_delta  { delta: { text: "the bug." } }
← content_block_stop

← content_block_start  { index: 1, content_block: { type: 'tool_use', name: 'Read' } }
← content_block_delta  { delta: { type: 'input_json_delta', partial_json: '{"file' } }
← content_block_delta  { delta: { partial_json: '_path": "' } }
← content_block_delta  { delta: { partial_json: '/path/to/foo.ts"}' } }
← content_block_stop

← message_delta        { stop_reason: 'tool_use' }
← message_stop
```

每个 `content_block_delta` 都立刻被 `yield` 给 UI 层，用户实时看到文字流出。

### 4.3 UI 渲染流式文字

```typescript
// src/components/AssistantMessage.tsx（简化）
// 消费 stream_event，实时拼接文字

for await (const event of queryGenerator) {
  if (event.type === 'stream_event') {
    if (event.event.type === 'content_block_delta') {
      if (event.event.delta.type === 'text_delta') {
        // 追加到当前文字块，触发 React re-render
        setCurrentText(prev => prev + event.event.delta.text)
      }
    }
  }
}
```

终端中用户看到：
```
I'll look at foo.ts to find the bug.
```
（文字逐字出现，不是一次性渲染）

---

## 第五阶段：工具调用（FileReadTool）

流结束后，`queryLoop()` 检测到 `stop_reason === 'tool_use'`：

```typescript
// src/query.ts — queryLoop()

const toolUseBlocks = assistantMessage.message.content
  .filter(block => block.type === 'tool_use')
// → [{ type: 'tool_use', id: 'toolu_01...', name: 'Read', input: { file_path: '/path/to/foo.ts' } }]

needsFollowUp = toolUseBlocks.length > 0  // true
```

### 5.1 工具查找

```typescript
// src/query.ts → runTools()

const tool = findToolByName(tools, 'Read')
// → FileReadTool 实例
```

### 5.2 输入校验

```typescript
// src/tools/FileReadTool/FileReadTool.ts

// Zod schema 校验
const parseResult = tool.inputSchema.safeParse({
  file_path: '/path/to/foo.ts'
})
// → { success: true, data: { file_path: '/path/to/foo.ts' } }

// validateInput()（可选，FileReadTool 实现了此方法）
const validation = await tool.validateInput(input, context)
// 检查：文件是否存在？路径是否在工作目录内？
// → { ok: true }
```

### 5.3 权限检查

```typescript
// src/utils/permissions.ts

const permResult = await tool.checkPermissions(input, context)
// FileReadTool.isReadOnly() === true
// 只读工具在默认权限模式下自动允许，无需用户确认
// → { behavior: 'allow' }
```

### 5.4 工具执行

```typescript
// src/tools/FileReadTool/FileReadTool.ts → call()

const content = await fs.readFile('/path/to/foo.ts', 'utf-8')
const lines = content.split('\n')

// 文件内容按行处理，加上行号前缀（方便 Claude 定位）
const numbered = lines.map((line, i) =>
  `${String(i + 1).padStart(4, ' ')}\t${line}`
).join('\n')

// 记录到 readFileState（跨 turn 缓存，避免重复读取）
context.readFileState.set('/path/to/foo.ts', {
  content,
  mtime: stat.mtimeMs,
})

return {
  data: {
    type: 'text',
    filePath: '/path/to/foo.ts',
    content: numbered,
    numLines: lines.length,
    totalLines: lines.length,
    startLine: 1,
  }
}
```

### 5.5 工具结果追加到消息历史

```typescript
// src/query.ts — 工具结果被封装为 UserMessage（API 要求）

const toolResultMessage: UserMessage = {
  type: 'user',
  uuid: generateUUID(),
  timestamp: Date.now(),
  message: {
    role: 'user',
    content: [{
      type: 'tool_result',
      tool_use_id: 'toulu_01...',
      content: numbered,  // foo.ts 的完整内容（带行号）
    }]
  },
  toolUseResult: 'Read',
}
```

**此时消息历史变为：**
```
messages = [
  UserMessage:     "fix the bug in foo.ts"
  AssistantMessage: "I'll look at foo.ts..." + tool_use(Read, foo.ts)
  UserMessage:     tool_result(foo.ts 内容)   ← 刚追加
]
```

---

## 第六阶段：第二轮 API 请求

`queryLoop()` 的 `while(true)` 继续下一次迭代，携带更新后的消息历史再次调用 Claude：

```
← content_block_start  { type: 'text' }
← content_block_delta  { text: "I found the bug on line 42: ..." }
← content_block_stop

← content_block_start  { type: 'tool_use', name: 'str_replace_based_edit_tool' }
← content_block_delta  { partial_json: '{"file_path":"/path/to/foo.ts",' }
← content_block_delta  { partial_json: '"old_string":"return x * 2",' }
← content_block_delta  { partial_json: '"new_string":"return x + 2"}' }
← content_block_stop

← message_delta  { stop_reason: 'tool_use' }
```

---

## 第七阶段：工具调用（FileEditTool）+ 权限确认

### 7.1 权限检查（需要用户确认）

```typescript
// src/utils/permissions.ts

// FileEditTool.isReadOnly() === false
// 写操作在默认权限模式下需要用户确认

const permResult = await tool.checkPermissions(
  { file_path: '/path/to/foo.ts', old_string: 'return x * 2', new_string: 'return x + 2' },
  context
)
// → { behavior: 'ask', message: '是否允许编辑 foo.ts？' }
```

### 7.2 弹出权限确认 UI

```typescript
// src/query.ts → canUseTool()

// setToolJSX 回调触发 React 渲染权限确认组件
context.setToolJSX?.(
  <PermissionRequest
    tool={tool}
    input={input}
    onAllow={() => resolve({ allow: true })}
    onDeny={() => resolve({ allow: false })}
  />
)

// 阻塞等待用户操作
const response = await permissionPromise
```

终端中渲染：
```
┌─ Permission Request ────────────────────────────────┐
│ str_replace_based_edit_tool wants to edit foo.ts    │
│                                                      │
│ - return x * 2                                       │
│ + return x + 2                                       │
│                                                      │
│ [y] Allow  [n] Deny  [a] Always Allow               │
└─────────────────────────────────────────────────────┘
```

### 7.3 用户按 `y`，执行编辑

```typescript
// src/tools/FileEditTool/FileEditTool.ts → call()

// 1. 读取文件当前内容
const currentContent = await fs.readFile(file_path, 'utf-8')

// 2. 验证 old_string 唯一性
const occurrences = countOccurrences(currentContent, old_string)
if (occurrences === 0) throw new Error('old_string not found')
if (occurrences > 1) throw new Error('old_string matches multiple locations')

// 3. 执行替换
const newContent = currentContent.replace(old_string, new_string)

// 4. 原子写入（先写 .tmp 再 rename，防止写入中断导致文件损坏）
await fs.writeFile(file_path + '.tmp', newContent, 'utf-8')
await fs.rename(file_path + '.tmp', file_path)

// 5. 生成 diff 供 UI 展示
const diff = await fetchSingleFileGitDiff(file_path)

return {
  data: {
    type: 'edit',
    filePath: file_path,
    diff,
    numLines: newContent.split('\n').length,
  }
}
```

### 7.4 UI 渲染编辑结果

```typescript
// src/tools/FileEditTool/UI.tsx → renderToolResultMessage()

// 渲染 diff 格式的编辑结果
return (
  <Box flexDirection="column">
    <Text>Edited foo.ts</Text>
    <DiffView diff={diff} />
  </Box>
)
```

终端中显示：
```
✓ Edited foo.ts
  - return x * 2
  + return x + 2
```

---

## 第八阶段：第三轮 API 请求（最终回复）

消息历史继续增长：
```
messages = [
  UserMessage:      "fix the bug in foo.ts"
  AssistantMessage: text + tool_use(Read)
  UserMessage:      tool_result(foo.ts 内容)
  AssistantMessage: "I found the bug..." + tool_use(FileEdit)
  UserMessage:      tool_result(edit success, diff)   ← 新追加
]
```

Claude 第三次被调用，这次没有工具调用，直接输出文字：

```
← content_block_delta  { text: "I've fixed the bug in foo.ts." }
← content_block_delta  { text: " The issue was a multiplication operator..." }
← content_block_delta  { text: " I changed `return x * 2` to `return x + 2`." }
← message_delta        { stop_reason: 'end_turn' }
```

`stop_reason === 'end_turn'` → `needsFollowUp = false`

---

## 第九阶段：循环终止

### 9.1 Stop Hooks 执行

```typescript
// src/query.ts → handleStopHooks()

// 检查用户是否配置了 stop hooks（.claude/settings.json 中的 hooks.Stop）
const stopHooks = getStopHooks()
if (stopHooks.length === 0) {
  // 没有 stop hooks，直接返回
  return { preventContinuation: false }
}

// 有 stop hooks → 执行，例如自动运行测试、格式化代码等
for (const hook of stopHooks) {
  const result = await executeHook(hook, { lastMessage, messages })
  if (result.continue) {
    // hook 要求继续对话（例如：测试失败，让 Claude 继续修复）
    return { preventContinuation: true, injectedMessages: result.messages }
  }
}
```

### 9.2 query() 返回

```typescript
// src/query.ts
return { reason: 'completed' }
// ↑ Terminal 类型，结束整个对话循环
```

### 9.3 统计 token 用量

```typescript
// src/QueryEngine.ts
this.totalUsage = accumulateUsage(this.totalUsage, currentRequestUsage)
// totalUsage = {
//   input_tokens: 1842 + 3200 + 1100,
//   output_tokens: 45 + 89 + 52,
//   cache_read_input_tokens: 1650,   // 命中 Prompt Cache 的 token 数
// }
```

---

## 第十阶段：REPL 恢复等待

```typescript
// src/components/REPL.tsx

// query() 生成器耗尽，REPL 恢复 input 状态
setIsLoading(false)
setCurrentInput('')
// 重新渲染输入框，等待下一次用户输入
```

终端最终状态：
```
> fix the bug in foo.ts

I'll look at foo.ts to find the bug.

✓ Read foo.ts (87 lines)

I found the bug on line 42: the function was using multiplication
instead of addition.

✓ Edited foo.ts
  - return x * 2
  + return x + 2

I've fixed the bug in foo.ts. The issue was a multiplication operator
being used where addition was expected. Changed `return x * 2` to
`return x + 2`.

>                                ← 等待下一次输入
```

---

## 完整调用栈汇总

```
REPL.tsx::onSubmit()
  └── QueryEngine::submitMessage()
        ├── processUserInput()              → UserMessage
        ├── fetchSystemPromptParts()        → SystemPrompt + contexts
        └── query()  ─── 异步生成器 ────────────────────────┐
              └── queryLoop()                               │
                    ├── [微压缩/自动压缩]                    │
                    ├── queryModelWithStreaming()            │  yield StreamEvents
                    │     └── queryModel()                  │  → UI 实时渲染
                    │           └── anthropic.messages.stream() │
                    ├── [检测 tool_use 块]                   │
                    ├── runTools()                           │
                    │     ├── findToolByName()               │
                    │     ├── tool.validateInput()           │
                    │     ├── tool.checkPermissions()        │
                    │     │     └── [权限 UI 弹出/等待]       │
                    │     └── tool.call()                    │
                    │           ├── FileReadTool: fs.readFile │
                    │           └── FileEditTool: fs.writeFile│
                    ├── [更新 messages，继续下一轮]           │
                    └── return Terminal { reason: 'completed' }
```

---

## 关键设计回顾

| 环节 | 设计决策 | 理由 |
|------|---------|------|
| `query()` 用异步生成器 | `yield` 每个流式事件 | UI 可以实时渲染，无需内部缓冲队列 |
| 工具结果封装为 `UserMessage` | Anthropic API 要求 `tool_result` 在 user 角色中 | 协议约束 |
| `FileEditTool` 用字符串替换而非行号 | 多次编辑后行号偏移 | 稳定性，Claude 不会因前一次编辑而行号错位 |
| Prompt Cache + 缓存断点 | 在工具描述后插入 `cache_control` | 系统提示和工具描述几乎不变，命中率极高 |
| `while(true)` + `state` 对象 | 隐式状态机 | 可通过 `state.transition` 追踪每轮原因，便于调试 |
| `readFileState` 跨 turn 缓存 | 记录已读文件的 mtime | `file_unchanged` 响应避免重发相同内容，节省 token |
| Stop Hooks 在循环终止处执行 | `handleStopHooks()` 可 `preventContinuation` | 允许外部脚本决定是否继续对话（如测试失败自动重试） |
