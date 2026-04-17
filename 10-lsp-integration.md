# 10 - LSP 集成

## 本文核心问题

LSP（Language Server Protocol）让 Claude Code 能获取实时编译错误，无需运行构建。核心问题：**如何把一个外部进程（语言服务器）的异步推送事件，干净地注入到 Agent 的对话流里？**

---

## 1. 分层架构

```
manager.ts（全局单例）— 生命周期 + 状态
    ↓
LSPServerManager — 多服务器协调 + 文件路由
    ↓
LSPServerInstance — 单服务器生命周期 + 重试
    ↓
LSPClient — JSON-RPC over stdio
    ↓
LSP 服务器进程（tsserver / pyright / gopls 等）
```

每层职责单一，LSPClient 只管 JSON-RPC 通信，不管生命周期；LSPServerInstance 管单个服务器的状态，不管路由；LSPServerManager 管路由，不管具体服务器的内部逻辑。

---

## 2. 被动推送：诊断如何进入对话

LSP 服务器发现错误时会主动推送 `textDocument/publishDiagnostics` 通知，不是 Claude Code 去查询。

这带来一个设计问题：推送是异步的，但注入对话必须在 Agent loop 的合适时机。Claude Code 的做法是**两阶段注入**：

1. 收到诊断推送 → 存入 `pendingDiagnostics` Map（不立即注入）
2. 下一轮 Agent loop 开始前 → `checkForLSPDiagnostics()` 取出待处理诊断，作为 attachment 注入消息

这样诊断不会在任意时刻打断对话，而是在对话的自然节点注入。

---

## 3. 诊断去重的两个维度

诊断的重复问题比想象中复杂：

**批内去重**：同一次推送里可能有多个文件报同一个错误（罕见但存在），用哈希 key 去重。

**跨轮次去重**：更重要的问题——如果 `foo.ts` 有一个类型错误，LSP 服务器会在每次文件保存后重新推送这个错误。如果不过滤，Claude 每轮都会看到"foo.ts 第 42 行有类型错误"，占用大量上下文。

LRU 缓存 `deliveredDiagnostics` 追踪哪些诊断已经告诉过 Claude，相同的诊断不再重复注入。文件被编辑后调用 `clearDeliveredDiagnosticsForFile()` 重置该文件的记录——编辑后可能引入新的或不同的错误，需要重新判断。

---

## 4. 体积控制

诊断数量有上限（每文件 10 条，总计 30 条），超出按错误严重程度排序截断，Error 优先于 Warning。

为什么限制数量？诊断数据会进入消息历史，每条诊断都有文件路径、行号、错误信息，累积起来 token 消耗可观。对 Claude 来说，看到"有 30 个 Error"不比看到"有 3 个 Error"更有用——能修复的先修复最严重的。

---

## 5. 懒加载策略

LSP 服务器不在应用启动时立即启动，而是等到首次访问对应扩展名的文件时才启动。

原因：项目可能涉及 TypeScript、Python、Go 等多种语言，但用户在某次会话里可能只操作其中一种。全部启动会占用大量资源（tsserver 本身就是重量级进程），而且大多数启动是浪费的。

---

## 6. ContentModified 的指数退避重试

发请求给 LSP 服务器时，可能遇到 `-32801`（ContentModified）错误——文件正在被修改，语言服务器的状态不稳定。

处理方式是指数退避重试（500ms → 1000ms → 2000ms，最多 3 次）。这个错误是暂时性的，等文件修改完成后语言服务器状态稳定，请求就能成功。比直接失败更友好。

---

## 7. 代次计数器防并发竞争

`initializationGeneration` 是一个单调递增整数，解决了一个具体的竞态问题：

场景：插件安装后调用 `reinitializeLspServerManager()`，此时上一次初始化的异步操作可能还在进行。两次初始化并发，后完成的可能会用旧状态覆盖新状态。

解决：每次 `initializeLspServerManager()` 时递增代次。异步初始化完成时检查代次是否匹配，不匹配（说明已经有新的初始化启动了）就丢弃结果，不更新状态。

---

## 8. workspace/configuration 的防御性处理

某些 LSP 服务器（比如 TypeScript 的 tsserver）即使客户端没有声明支持 `workspace/configuration` 能力，也会发送这个请求。不处理会导致服务器报错。

处理方式：直接注册处理器返回 null（空配置）。这是防御性兼容，不是应该存在的功能，但现实中的 LSP 服务器实现不总是符合规范。

---

## 设计原则提炼

1. **两阶段注入**：异步推送 → 暂存 → 在 Agent loop 节点同步注入
2. **跨轮次去重**：同一错误不重复告知，编辑后重置
3. **懒加载**：只启动被用到的语言服务器
4. **代次计数器**：无锁解决并发初始化竞争
5. **防御性兼容**：现实中的 LSP 实现不总是符合规范
