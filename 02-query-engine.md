# 02 - 核心查询引擎 (Query Engine)

> 核心文件：`query.ts` (1729行) / `QueryEngine.ts` (1295行)

---

## 1. Agent 循环的本质

整个 Agent 的核心逻辑极简：

```
while (true) {
    调用 Claude API
    if 没有 tool_use:
        结束
    执行工具，把结果追加到 messages[]
    // 继续下一轮
}
```

**messages[] 是唯一的"记忆"**。每轮工具结果追加进去，下一轮整个数组发给 Claude，Claude 通过读历史消息知道"之前做了什么"。没有任何其他状态需要维护。

这就是标准的 ReAct 模式（Reason → Act → Observe → Reason...），Claude Code 的实现非常忠实于这个范式。

---

## 2. AsyncGenerator：不只是"流式输出"

`query()` 被实现为异步生成器，yield 的内容包括：流式 token（StreamEvent）、工具执行结果（ToolResultMessage）、系统消息、错误消息等。UI 层 `for await` 消费这些事件来实时渲染界面。

真正的价值不是流式渲染，而是**代码组合性**：

```
query()
  yield* queryLoop()
    yield* handleStopHooks()
    yield* yieldMissingToolResultBlocks()
```

`yield*` 可以无缝委托给子生成器，深层函数产生的事件自动冒泡到最外层消费方，不需要把回调函数一路透传下去。UI 和 SDK 两种消费方也复用同一个生成器，不需要维护两套逻辑。

---

## 3. 消息压缩：各自独立，不是优先级链

每轮循环开头都会过一遍压缩流水线：

```
snip → microcompact → contextCollapse → autocompact
```

**每轮都检查，但大多数时候是 no-op**——每个策略内部都有条件判断，不满足就直接返回原始消息。真正触发压缩的条件各自独立：

| 策略 | 触发条件 | 做什么 |
|------|---------|--------|
| snip | 消息数超阈值 | 直接截断最早的消息 |
| microcompact | 距上次对话 > 60分钟（缓存已冷） | 清空旧工具结果的内容，只留占位符 |
| contextCollapse | 有待折叠的内容（内部feature） | 把历史大块内容替换为摘要 |
| autocompact | token 预算超出 | 让 AI 生成整段对话的摘要 |

microcompact 的触发条件尤其值得注意：它和 context 够不够长没有关系，而是基于**时间**——服务器端 prompt cache 超时过期了，既然缓存要重建，顺便清掉旧工具结果减少重建代价。

唯一真正互斥的一对是 contextCollapse 和 autocompact：collapse 启用时会主动抑制 autocompact，让 collapse 接管上下文压缩职责。

---

## 4. 错误恢复：状态转移而非异常处理

错误恢复没有用 try/catch 向上抛异常，而是修改循环的内部状态，然后 `continue` 重试。这让恢复逻辑和主流程融为一体。

### 4.1 max_output_tokens：重新生成 vs 续写

Claude 输出到一半被截断时，有两种修复思路，系统都实现了：

**第一步：重新生成（升级 token 上限）**

把 `maxOutputTokensOverride` 从默认 8k 升到 64k，原始消息不变，直接重试。截断的输出丢弃。

设计取舍：如果问题只是"上限设太保守"，重新生成一段完整输出比拼接两段更干净。代价是截断部分的 token 浪费。

**第二步：续写（注入恢复消息）**

如果 64k 也不够，把截断的输出保留在 messages[] 里，追加一条对用户不可见的 meta 消息：

> "Output token limit hit. Resume directly — no apology, no recap. Pick up mid-thought if that is where the cut happened."

让 Claude 从断点接着写，最多重试 3 次。

### 4.2 withheld 模式：主动扣住中间错误

这是一个值得单独记住的设计模式。

错误消息在流式阶段被**故意扣住**，不立刻 yield 给消费方，等确认恢复失败了才释放：

```
流式输出阶段发现 max_output_tokens 错误
  → 标记为 withheld，不 yield
  → 进入恢复流程
  → 恢复成功：错误消息永远不出现
  → 恢复失败（3次后）：才把错误消息 yield 出去
```

为什么要这样？因为 SDK 调用方（VSCode 插件等）看到任何 error 字段就会直接终止会话。如果错误可以被修复，提前暴露只会让外部调用方做出错误决策。

这个"扣住中间错误，只暴露无法恢复的错误"的模式，在 prompt_too_long（413）的处理里也有相同的逻辑。

### 4.3 用户中止：保证消息格式合法

Ctrl+C 时不能直接退出，因为 Anthropic API 要求 `tool_use` 和 `tool_result` 必须成对出现。如果 Claude 说"我要调用工具X"但没有对应结果，下次恢复会话时 API 会拒绝整个消息历史。

所以中止时必须补齐：

```
用户中止
  → 流式阶段中止：给每个未完成的 tool_use 补一个假的 tool_result
  → 工具执行阶段中止：StreamingToolExecutor 为未完成的工具生成合成结果
  → 确保 messages[] 格式始终合法
```

---

## 5. 设计思路总结

Claude Code 的 Agent 循环体现了几个值得借鉴的思路：

1. **极简状态**：messages[] 就是全部状态，Agent 循环不需要额外的状态机
2. **错误是状态转移**：能自动修复的错误在循环内部消化，不暴露给外部
3. **消息可以不对称**：系统可以往 messages[] 插入用户不可见的 meta 消息来引导模型行为，这是一个在对话历史里做"旁白"的技巧
4. **格式合法性是硬约束**：任何情况下都要保证 messages[] 对 API 合法，这是持久化和恢复会话的基础
