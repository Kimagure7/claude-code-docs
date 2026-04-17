# 07 - API 客户端层

## 本文核心问题

API 客户端层解决：**如何在直连、Bedrock、Vertex 等多种部署模式下统一发起请求，并智能处理各种失败情况（过载、限流、上下文溢出）？**

---

## 1. 多云统一抽象

四种部署模式（直连 Anthropic / AWS Bedrock / GCP Vertex / Azure Foundry）对上层完全透明。`getAnthropicClient()` 是工厂函数，内部根据环境变量分支，但返回的对象对调用方看起来都一样。

有趣的是，源码注释里直接写了 "we have always been lying about the return type"——四种客户端实际上是不同 SDK 类型，但都被标注为同一个类型。这是有意识的类型谎言，为了让调用层不需要感知底层差异。

共同配置（超时、headers、fetch 包装）被提取成一个 `ARGS` 基础对象，三种云提供商各自 spread 追加差异部分。这让公共逻辑只写一次。

---

## 2. 请求 ID 的注入位置

每个请求有一个客户端生成的 UUID（`x-client-request-id`）。注入位置选择在 `buildFetch()` 包装层而非 SDK 层，原因很具体：

SDK 拿到的 request ID 来自**服务端响应头**，但如果请求超时，根本没有响应，就没有 ID。客户端自生成的 UUID 在任何情况下都存在，哪怕超时、网络断开，API 团队都能在服务端日志里通过这个 ID 关联到那次失败的请求。

---

## 3. 重试策略的前台/后台分离

`withRetry()` 是一个 async generator，重试时 `yield` 一个错误消息给 UI，让用户实时看到"正在重试 (2/10)..."。

重试策略的核心设计点：**后台请求遇到 529（服务过载）直接放弃，不重试**。

理由是"容量级联"问题：服务器过载时，每次重试实际上是在放大压力。后台任务（标题生成、分析等）对用户不可见，放弃了没有大影响。但如果这些不重要的任务在服务器过载时持续重试，会进一步加剧过载，影响真正等待结果的前台请求。

反过来，前台请求（用户在等待的）则会正常重试。`FOREGROUND_529_RETRY_SOURCES` 白名单明确列出哪些请求来源算"用户在等待"。

---

## 4. 上下文溢出的自动恢复

当 API 返回"prompt is too long"错误（400），重试逻辑不是简单重试，而是：

1. 解析错误消息里的 token 数量（"137500 tokens > 135000 maximum"）
2. 计算 `adjustedMaxTokens = contextLimit - inputTokens - 1000`
3. 把这个值写入 `retryContext.maxTokensOverride`
4. 下一次请求用缩减后的 `max_tokens` 重试

这比"直接失败让用户手动处理"好得多。错误本身携带了足够的信息来自动恢复。

这里有一个联动：`errors.ts` 的 `getAssistantMessageFromError()` 在格式化 prompt 过长错误时，把原始错误字符串（含 token 数）存进 `errorDetails` 字段。下游的 compact 逻辑通过 `getPromptTooLongTokenGap()` 解析这个字段，计算需要压缩多少历史消息。错误信息成了系统内部的通信协议。

---

## 5. Opus 过载的模型降级

连续 3 次 529 错误（服务过载）后，如果配置了 `fallbackModel`，会抛出 `FallbackTriggeredError`，触发降级到 Sonnet。

这是一种优雅降级策略：Opus 最强但资源紧张时最容易过载，Sonnet 能力稍弱但容量更充裕。对用户来说，慢一点但能用比一直等好。

---

## 6. Prompt Cache 的稳定性保证

Prompt Cache 有 5 分钟和 1 小时两个 TTL 档，哪些请求用 1 小时 TTL 有资格检查。

关键设计：资格判断结果在**首次请求时锁定到 session state**，之后不再重新计算。

为什么？如果 GrowthBook 的远程配置在会话中途更新，某次请求从 5 分钟切换到 1 小时 TTL，服务端会认为这是一个新的缓存 key（TTL 是 cache_control 的一部分），之前积累的缓存全部失效。锁定后，一个会话里 TTL 类型始终一致，缓存稳定命中。

这是"会话稳定性优先于实时配置准确性"的典型权衡。

---

## 7. Fast Mode 的降速机制

Max/Ultra 订阅有 Fast Mode（更高优先级的请求通道）。Fast Mode 下遇到 429 限流，有两种处理：
- 短延迟（`retry-after` < 阈值）→ 保持 Fast Mode，等一会儿继续
- 长延迟 → 触发 cooldown，临时切回标准速度

这防止了 Fast Mode 在严重限流时持续等待，用户反而不如用标准速度响应快。

---

## 设计原则提炼

1. **类型谎言换来调用层透明**：多云统一接口，代价是类型不精确
2. **客户端生成 ID 比服务端 ID 更可靠**：超时场景下仍可追踪
3. **前台/后台重试分离**：避免在容量级联时放大压力
4. **错误消息是数据**：token 溢出错误里的数字被解析用于自动恢复
5. **会话级配置锁定**：缓存稳定性 > 实时准确性
