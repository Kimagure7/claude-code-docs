# 08 - Slash 命令系统

## 本文核心问题

Slash 命令是用户主动触发的控制入口，与工具系统（模型主动调用）方向相反。核心问题：**怎么把几十种不同性质的命令（执行逻辑/渲染UI/展开为提示词）统一管理，同时支持用户扩展？**

---

## 1. 三种命令类型的设计逻辑

```
local      → 执行 → 返回文本结果
local-jsx  → 执行 → 返回 React 组件（终端 UI）
prompt     → 展开为提示词 → 发给模型
```

这三种类型不是随意分类，而是对应三种完全不同的执行路径：

`local` 命令在本进程同步执行，典型如 `/clear`（清空历史）、`/cost`（显示费用）。

`local-jsx` 命令渲染 Ink UI 组件，典型如 `/config`（设置界面）、`/memory`（记忆编辑器）。命令返回一个 React 元素，REPL 层挂载它，用户操作完成后通过 `onDone()` 回调关闭。这意味着命令可以有自己的完整交互界面，不只是输出文字。

`prompt` 命令展开为 `ContentBlockParam[]` 注入模型对话，典型如技能（Skills）。这类命令本质上是"把用户的 `/skill args` 转化为一段提示词发给 Claude"。

类型系统在编译期区分三者，不同类型的命令在不同路径处理，不会混用。

---

## 2. 两个独立的过滤维度

命令有两个独立的"是否可用"维度：

**`availability`（静态权限）**：这个命令对哪类用户可见？`claude-ai`（OAuth 订阅用户）还是 `console`（API Key 用户）。这是身份属性，不会在运行中变化。

**`isEnabled()`（动态开关）**：这个命令在当前环境是否开启？读 feature flag、环境变量、运行时状态。同一个用户，开了某个 feature flag 后才能看到某些命令。

两个维度独立变化，互不干扰。`meetsAvailabilityRequirement()` 检查前者，`isEnabled()` 检查后者，都通过才显示命令。

---

## 3. 命令加载的懒加载+缓存

命令列表加载涉及磁盘 I/O（读技能 Markdown 文件）和动态 import，开销较大。两层优化：

`COMMANDS()` 是 `memoize` 包装的函数（不是常量），首次调用时执行，之后返回缓存。设计成函数而非模块级常量，是因为内部需要读 config，而 config 在模块初始化时还不可用。

`loadAllCommands` 按工作目录 memoize——不同项目目录可能有不同的技能文件，所以按 cwd 分别缓存。切换项目会自动重建缓存。

---

## 4. `/clear` 是完整的会话重置，不是清空数组

`/clear` 看起来只是"清空消息历史"，实际上是一个完整的会话状态机转换：

1. 执行 `SessionEnd` hooks（给外部脚本通知当前会话结束）
2. 向后端发送 cache eviction hint（允许推理层释放 KV cache）
3. 保留需要继续运行的后台任务（Ctrl+B 开的任务不受影响）
4. 清空消息历史、文件状态缓存、MCP 状态
5. 生成新 session ID，保留旧 ID 作为 parent（用于追踪"这是同一用户的连续会话"）
6. 执行 `SessionStart` hooks（为新会话初始化环境）

如果 `/clear` 只是清空数组，后台任务会丢失，hooks 不会触发，session ID 不会轮换，统计追踪会断链。"清空"的语义比外表深得多。

---

## 5. `/compact` 的三条路径

```
无自定义指令 + session memory 可用 → SM 压缩（轻量，不调 LLM）
reactive compact 模式              → 响应式压缩
兜底                               → 传统 LLM 压缩
```

用户执行 `/compact "聚焦在数据库相关的内容"` 时，自定义指令使得 SM 压缩路径不可用（SM 压缩不支持自定义指令），直接走传统 LLM 压缩。

这三条路径的优先级和触发条件是独立的，不是统一的"优先级链"——每条路径有自己的前提条件，满足哪条走哪条。

---

## 6. Skill 即 Markdown 文件

用户在 `~/.claude/skills/` 下放一个 `.md` 文件，就能创建一个新的 slash 命令。frontmatter 里可以声明：

- `allowed-tools`：这个技能可以用哪些工具
- `paths`：只有当前目录匹配这些 glob 时才显示这个技能（比如只在 React 项目里显示 React 相关技能）
- `model`：覆盖默认模型
- `context: fork`：在独立的子 Agent 里执行

这是零代码扩展 Claude Code 能力的机制。Markdown 内容就是发给 Claude 的提示词，`$ARGUMENTS` 占位符会被用户输入替换。

---

## 7. 远程/Bridge 安全过滤

在远程模式或手机客户端（Bridge 模式）下，不是所有命令都能执行：

- `local-jsx` 命令（需要终端 UI）始终被 Bridge 阻止——手机客户端无法渲染 Ink UI
- `prompt` 命令（展开为文本）始终安全——纯文本传输没问题
- `local` 命令按白名单过滤——只有明确声明安全的命令才能远程执行

这个分层过滤让命令系统可以在不同运行环境里安全使用，而不需要每个命令各自处理环境判断。

---

## 设计原则提炼

1. **类型驱动分支**：三种命令类型决定三条执行路径，不是运行时 if/else
2. **两维度独立过滤**：身份（availability）和状态（isEnabled）分离
3. **语义上的完整性**：`/clear` 不只是清空，而是完整的状态重置
4. **配置即扩展**：Markdown + frontmatter 让用户无代码创建命令
