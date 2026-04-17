# 13 - 多 Agent 协调系统

## 本文核心问题

单个 Agent 是串行的，受限于对话轮次。多 Agent 系统让主 Agent 能把复杂任务拆分给多个子 Agent 并行处理。核心问题：**如何在不破坏安全约束的前提下，实现 Agent 的递归嵌套和异步协调？**

---

## 1. 四种执行模式

| 模式 | 触发条件 | 特性 |
|------|---------|------|
| 同步前台 | 默认 | 阻塞主线程，可随时背景化 |
| 异步后台 | `run_in_background=true` 或 Coordinator 模式 | 立即返回任务 ID，通知机制汇结果 |
| Fork 模式 | 无 `subagent_type` 且功能启用 | 继承父上下文，prompt cache 对齐 |
| Teammate 多窗格 | `team_name && name` | 独立 Tmux 窗格，独立进程 |

四种模式不是功能重复，而是针对不同场景的优化：短任务用同步，并行研究用异步，需要父上下文用 Fork，可视化协作用 Teammate。

---

## 2. 同步前台的动态背景化

前台执行的子 Agent 可以随时被用户转为后台，靠 `Promise.race()` 实现：

```
Promise.race([下一条消息, 背景化信号])
```

用户按 ESC 时，背景化信号 resolve，race 结束。此时：
1. 已收集的消息完整保留
2. 新启动后台迭代器从 `runAgent()` 继续
3. 主线程立即返回 `async_launched`

子 Agent 没有被重启，只是从"主线程等待它"切换到"它在后台自己跑"。无损中断，已执行的工作不丢失。

2 秒后如果子 Agent 还在运行，UI 会显示背景化提示——这是在引导用户"知道可以这么做"，而不是强制的。

---

## 3. 三级权限隔离

```
ALL_AGENT_DISALLOWED_TOOLS   — 所有子 Agent 均不可用
CUSTOM_AGENT_DISALLOWED_TOOLS — 自定义 Agent 额外限制
ASYNC_AGENT_ALLOWED_TOOLS    — 异步 Agent 只能访问白名单
```

最关键的约束：`AgentTool` 本身在 `ALL_AGENT_DISALLOWED_TOOLS` 里。子 Agent 无法再生成子 Agent，防止递归嵌套失控。

异步 Agent 的工具限制最严格（只能访问文件操作和搜索），因为异步执行时没有用户交互，不能弹出权限确认对话框。保守白名单是这个场景下的唯一安全选择。

---

## 4. Fork 模式的 Prompt Cache 对齐

Fork 子 Agent 强制使用父 Agent 的 `renderedSystemPrompt`（字节精确复用），而不是重新生成系统提示。

原因：Anthropic API 的 Prompt Cache 基于内容的字节精确匹配。如果子 Agent 重新生成系统提示，即使语义相同，顺序不同或有任何字节差异，缓存就会 MISS。系统提示在所有请求里是最大的固定成本，MISS 意味着每次请求都要重新计算这几千 token。

字节精确复用确保缓存命中，是一个有实际成本影响的设计决策。

---

## 5. 异步结果汇聚：通知 XML

异步子 Agent 完成后，通过 XML 格式的通知消息把结果注入主 Agent 的下一轮对话：

```xml
<task-notification>
  <task-id>...</task-id>
  <status>completed</status>
  <result>子 Agent 的最终回复</result>
  <usage><total_tokens>N</total_tokens>...</usage>
</task-notification>
```

这个 XML 作为 UserMessage 注入，Claude 看到它就知道哪个子任务完成了、结果是什么。不需要特殊的 API，就是普通的消息历史内容。

---

## 6. SendMessage 的邮件模型

`SendMessage → queuePendingMessage → drainPendingMessages` 构成异步邮件系统：

- 发送者调用 `SendMessage` 后立即继续，不等待接收
- 消息追加到接收者的 `pendingMessages` 队列
- 接收者（子 Agent）在下一轮执行开始时自动 drain 队列

这是完全解耦的：发送者和接收者在各自的时间轴上运行，通过消息队列交换信息。支持点对点（指定名称）和广播（`*`）两种路由。

---

## 7. Teammate 多窗格：独立进程架构

Teammate 是最重型的协调模式——每个 Teammate 是独立的 Claude Code 进程，运行在独立的 Tmux 窗格里。

进程间通信通过**邮箱文件**：
```
~/.claude/team/{teamName}/mailbox/{name}/inbox/*.json
```

Team Lead 写入邮箱文件，Teammate 进程启动后轮询邮箱，取出消息作为第一轮 UserMessage。这是文件系统作为 IPC 机制的典型应用：简单、持久化、不需要共享内存或 socket。

Teammate 的模型选择有继承链：显式指定 > `'inherit'`（复用 leader 模型）> 全局配置 > 硬编码兜底。

---

## 8. Coordinator 模式

Coordinator 是一种特殊的运行模式，Agent 承担"调度者"角色，把任务分解给多个 Worker 并行处理：

```
Research Phase  — 并行 spawn 多 Worker 探索代码库
Synthesis Phase — Coordinator 汇总所有通知，整合发现
Implementation Phase — spawn Worker 并行实现
Verification Phase — spawn 验证 Worker 检查结果
```

Coordinator 模式下，所有子 Agent 调用强制异步——Coordinator 不等待任何单个 Worker，而是等待所有通知汇集后再综合。

Worker 可访问共享的 scratchpad 目录，跨 Worker 持久化知识，不需要把所有发现都塞进消息历史。

---

## 设计原则提炼

1. **Promise.race 实现随时可中断**：同步执行任何时刻可转后台，无损
2. **三级权限隔离**：最保守的约束在最外层（禁止嵌套）
3. **字节精确 prompt 复用**：缓存命中率是一个有实际成本的设计目标
4. **文件系统作为 IPC**：邮箱文件让进程间通信持久化、简单
5. **XML 通知作为普通消息**：不需要特殊协议，结果就是对话里的一条消息
