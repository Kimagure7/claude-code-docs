# 14 - 一次请求的完整生命周期

> 场景：用户输入 `fix the bug in foo.ts`，按下回车，直到 Claude 完成修复。串联前面所有知识点。

---

## 全局时间线

```
T=0ms    用户按回车
T=1ms    REPL 捕获输入，检查是否是 /command
T=5ms    并行获取系统提示组件（git 状态、记忆文件、工具描述）
T=10ms   query() 异步生成器启动
T=12ms   微压缩/自动压缩检查（首轮通常跳过）
T=15ms   发起 HTTPS 流式请求到 Anthropic API
T=1200ms 第一个 token 到达，UI 开始流式渲染文字
T=2000ms Claude 决定调用 FileReadTool，stop_reason = 'tool_use'
T=2010ms validateInput() + checkPermissions() 通过（只读工具自动允许）
T=2015ms FileReadTool.call() 执行，读 foo.ts
T=2020ms 工具结果追加到消息历史，第二轮 API 请求发起
T=3500ms Claude 决定调用 FileEditTool
T=3510ms checkPermissions() → 弹出权限确认 UI（写操作需确认）
T=3515ms 用户按 y 确认
T=3520ms FileEditTool 执行字符串替换，写入磁盘
T=3530ms 生成 diff，UI 渲染编辑结果
T=3600ms Claude 输出最终回复，stop_reason = 'end_turn'
T=3610ms Stop Hooks 执行（若配置）
T=3620ms query() 返回 Terminal { reason: 'completed' }
T=3625ms REPL 等待下一次输入
```

---

## 阶段 1：输入处理

用户按回车，REPL 的 `onSubmit` 触发。第一件事：检查是否是 `/` 开头的命令。是命令就走命令系统（不进入 AI 对话），不是命令就封装成 `UserMessage`。

`UserMessage` 格式是 Anthropic API 要求的：`{ role: 'user', content: [{ type: 'text', text: input }] }`。

---

## 阶段 2：系统提示组装

进入 query 前，并行获取三类上下文：

```
git 状态   → 当前分支、修改文件列表（git status 显示 foo.ts 有修改）
记忆文件   → CLAUDE.md + ~/.claude/memory/ 里的记忆
工具描述   → 每个工具的 description() 方法返回值
```

这三类信息合并成系统提示，一起发给 API。`git status` 显示 `foo.ts` 有修改这个细节，会直接帮助 Claude 定位到这个文件，减少探索步骤。

---

## 阶段 3：第一轮 API 请求

`query()` 是 async generator，内部是 `while(true)` 循环。第一轮：

1. 微压缩检查 → 消息历史只有一条，跳过
2. 调用 `queryModelWithStreaming()`，发起流式 HTTPS 请求
3. `yield` 每个流式事件给 UI → 用户实时看到文字流出

请求参数里工具列表以 JSON Schema 形式传给 API。Claude 看到工具后知道可以调用 `Read`（FileReadTool）来读文件。

流式响应中，文字 token 和 tool_use 参数 JSON 都是流式到达的，UI 实时渲染文字，等 tool_use 完整接收后再处理工具调用。

---

## 阶段 4：FileReadTool 执行

`stop_reason = 'tool_use'`，`queryLoop` 检测到需要执行工具：

1. `findToolByName(tools, 'Read')` 找到 FileReadTool 实例
2. Zod Schema 校验入参
3. `validateInput()` 检查路径合法性（存在？在工作目录内？不是设备文件？）
4. `checkPermissions()` → FileReadTool 是只读工具，默认模式自动允许，不弹 UI
5. `call()` 读文件，记录到 `readFileState`（记录 mtime，供后续写操作的时间戳验证）
6. 返回带行号的文件内容

工具结果封装为 `UserMessage`（API 要求 tool_result 在 user 角色里），追加到消息历史，循环继续。

**此时消息历史：**
```
UserMessage: "fix the bug in foo.ts"
AssistantMessage: 分析文字 + tool_use(Read, foo.ts)
UserMessage: tool_result(foo.ts 内容)
```

---

## 阶段 5：第二轮 API 请求

携带更新后的消息历史（包括 foo.ts 内容）再次调用 Claude。Claude 读到代码后发现问题，决定用 FileEditTool 修复。

这次 stop_reason = 'tool_use'，参数是具体的 `old_string` 和 `new_string`。

---

## 阶段 6：FileEditTool + 权限确认

1. `checkPermissions()` → FileEditTool 是写操作，默认模式需要用户确认
2. 弹出权限 UI，显示 diff 预览
3. 用户按 y
4. `validateInput()` 检查 `readFileState` 里有这个文件的读取记录（"先读后写"约束）
5. 验证 `old_string` 在文件中唯一匹配
6. 执行字符串替换，写入磁盘
7. 通知 LSP 服务器（触发重新诊断）、通知 VS Code（更新 diff 视图）

---

## 阶段 7：第三轮 API 请求（最终回复）

消息历史再次追加工具结果（编辑成功 + diff），第三次调用 Claude。

这次 Claude 输出最终解释文字，`stop_reason = 'end_turn'`，`needsFollowUp = false`，循环退出。

---

## 阶段 8：退出与清理

1. `handleStopHooks()` 检查是否有 Stop hooks（比如自动运行测试）
2. 如果 hooks 要求继续对话（测试失败，让 Claude 修复），循环重新启动
3. 否则 `query()` 返回 `Terminal { reason: 'completed' }`
4. 统计 token 用量（含 prompt cache 命中的 token 数）
5. REPL 恢复输入等待状态

---

## 关键设计的串联

| 环节 | 设计决策 | 为什么 |
|------|---------|--------|
| 系统提示 | 并行获取 git/记忆/工具描述 | 三者互相独立，没有依赖关系 |
| 流式事件 | AsyncGenerator yield 每个 token | UI 实时渲染，调用方不需要内部缓冲 |
| 只读工具 | 自动允许，不弹 UI | 降低摩擦，读操作风险低 |
| 写操作 | 需要确认，显示 diff | 不可逆操作需要人类监督 |
| 先读后写 | readFileState 约束 | 防止 Claude 在不了解当前内容的情况下覆盖文件 |
| Stop Hooks | 可以 preventContinuation | 外部脚本（测试）可以决定对话是否结束 |
| Prompt Cache | 在工具描述后插入 cache_control | 系统提示和工具描述几乎不变，命中率极高 |

整个过程体现了 02 讲的 Agent loop 核心：`while(true)` + 追加消息 + 检查 stop_reason。工具系统、权限系统、API 客户端、UI 渲染都是这个主循环的配件。
