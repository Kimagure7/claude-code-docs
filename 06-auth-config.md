# 06 - 认证与配置系统

## 本文核心问题

认证系统要解决：**多种身份验证方式（OAuth、API Key、企业凭证、外部命令）如何统一管理，同时保证安全性和可用性？** 配置系统要解决：**多进程并发读写同一个 JSON 文件不会损坏数据。**

---

## 1. 认证优先级链：为什么有这么多方式

Claude Code 需要支持多种使用场景：
- 个人用户用 API Key（最简单）
- 订阅用户用 OAuth（享受 claude.ai 账号体验）
- 企业用户走 Bedrock/Vertex（集成内部权限系统）
- CI 环境用环境变量注入（无交互）
- 自定义场景用外部命令（apiKeyHelper）

这些场景的优先级是硬编码的严格顺序，不是"哪个有用哪个"。环境变量优先级高于存储的 OAuth token，是因为 CI 场景需要能覆盖用户的个人账号配置。

---

## 2. OAuth PKCE 流程

PKCE（Proof Key for Code Exchange）是 OAuth 2.0 的安全扩展，专为无法安全存储客户端密钥的场景设计（CLI 工具就是这种场景）。

核心思路：客户端生成一个随机的 `code_verifier`，用 SHA256 哈希得到 `code_challenge`，在发起授权时把 challenge 发给服务器。授权完成后用 verifier 换 token——服务器用 challenge 验证 verifier，即使有人截获了授权码，没有 verifier 也换不了 token。

实现上支持两种流：
- **自动流**：本地启动 HTTP 监听器，浏览器 redirect 自动回调
- **手动流**：显示固定 URL，用户手动复制粘贴授权码

两种流并行等待，先到先得。这样即使自动流失败（某些终端环境无法打开浏览器），手动流还是可用的。

---

## 3. apiKeyHelper：SWR 缓存策略

apiKeyHelper 允许用户配置一个 shell 命令来获取 API Key（比如从公司 Secret Manager 拉取）。这个命令可能很慢（网络调用），所以用 **Stale-While-Revalidate** 策略：

```
首次调用 → 同步等待（必须有结果才能继续）
TTL 内   → 直接返回缓存（零延迟）
TTL 过期  → 立即返回旧值 + 后台异步刷新（不阻塞）
```

这样用户几乎感知不到延迟，同时 token 保持相对新鲜。

安全约束：来自项目设置的 apiKeyHelper 命令，在工作区信任对话框被接受之前不会执行。这防止了"恶意 git 仓库通过 `.claude/settings.json` 中的 apiKeyHelper 在 clone 时执行任意命令"的攻击向量。

---

## 4. API Key 安全存储

macOS 上，API Key 存 Keychain 而非配置文件，且写入时用 stdin 而非命令行参数：

```
security -i  ← 从 stdin 读取命令
```

理由：`ps aux` 可以看到进程的命令行参数，如果 API Key 以参数形式传入 `security` 命令，任何能运行 `ps` 的人都能看到。stdin 是进程间私有通道，不会泄漏到进程列表。

---

## 5. 配置文件的并发保护

多个 Claude Code 实例可能同时运行（不同项目），都在读写 `~/.claude.json`。没有并发保护的话，A 读取 → B 读取 → A 写入 → B 写入（基于旧版本），B 的写入会丢失 A 的更新。

两层保护：

**锁文件**：`saveGlobalConfig()` 使用锁文件机制，保证同一时刻只有一个进程在写。写失败时降级为无锁写入，但加了认证丢失守卫。

**认证丢失守卫**：`wouldLoseAuthState()` 检测一种特殊情况——文件在另一个进程 mid-write 时被读取，可能读到截断或空文件，内容解析后是默认值，没有 `oauthAccount` 字段。如果内存里有有效的 OAuth token，而要写入的内容会清除这个 token，守卫会阻止这次写入。这修复了一个真实发生过的 bug：用户被意外登出。

**文件变更监听**：`fs.watchFile` 以 1 秒间隔轮询，检测其他进程的写入并刷新内存缓存。写入后会把内存缓存的 mtime 设为 `Date.now()`（超前于磁盘 mtime），下次 watchFile 回调时检测到 `curr.mtimeMs <= cache.mtime` 就跳过——这防止了自己的写入触发自己的重读。

---

## 6. 工作区信任的传递语义

信任一个目录，意味着其所有子目录也被信任。实现是从当前工作目录向上遍历父目录，检查每个目录是否被信任过。

这和 Git 的工作区信任模型一致：信任了 `/home/user/project`，就意味着信任了 `/home/user/project/src/` 里的所有内容。用户不需要对每个子目录单独授权。

---

## 7. 订阅类型与功能访问

OAuth token 里携带了 `subscriptionType`（pro/max/team/enterprise），不同订阅类型能访问不同功能。比如 1 小时 Prompt Cache TTL 只有特定订阅类型才能用。

订阅类型在首次请求时被锁定到 session state，之后不再重新检查。这防止了一个微妙的缓存失效问题：如果会话中途订阅状态变化（比如 GrowthBook 配置更新），导致 `cache_control.ttl` 从 5 分钟变成 1 小时，服务端的缓存边界会失效，之前积累的缓存 token 全部作废。会话稳定性比实时准确性更重要。

---

## 设计原则提炼

1. **stdin 而非参数传递密钥**：防止进程列表泄漏
2. **SWR 缓存外部命令**：慢操作对用户透明，后台保持新鲜
3. **守卫而非乐观写入**：认证丢失比配置不一致代价更高
4. **会话级锁定不稳定配置**：缓存命中率比实时准确性更重要
