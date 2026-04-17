# Claude Code 源码解析文档

对 `@anthropic-ai/claude-code v2.1.88` 的深度源码分析，覆盖架构设计、核心模块、关键流程与代码细节。

> 源码路径: `/Users/zhaoziyi05/ZoeyChiu/claude-code-sourcemap/restored-src/src/`

---

## 文档目录

| 文件 | 主题 |
|------|------|
| [00-overview.md](./00-overview.md) | 全局架构总览、技术栈、数据流、工具与命令全览 |
| [01-startup.md](./01-startup.md) | 启动流程与初始化（冷启动优化、Trust 机制、Worktree） |
| [02-query-engine.md](./02-query-engine.md) | 核心 AI 对话循环（processQuery / runQuery / 流式处理） |
| [03-tool-system.md](./03-tool-system.md) | 工具系统设计（Tool 接口、权限模型、并发控制） |
| [04-tools-impl.md](./04-tools-impl.md) | 核心工具实现（BashTool、FileEditTool、GrepTool 等） |
| [05-mcp-protocol.md](./05-mcp-protocol.md) | MCP 协议集成（外部工具扩展机制） |
| [06-auth-config.md](./06-auth-config.md) | 认证与配置系统（OAuth、API Key、MDM 企业配置） |
| [07-api-client.md](./07-api-client.md) | API 客户端（流式调用、重试、错误处理） |
| [08-command-system.md](./08-command-system.md) | Slash 命令系统（80+ 命令的注册与分发） |
| [09-memory-session.md](./09-memory-session.md) | 记忆与会话管理（历史压缩、CLAUDE.md、持久化） |
| [10-lsp-integration.md](./10-lsp-integration.md) | LSP 语言服务集成（代码诊断、补全） |
| [11-plugin-skill.md](./11-plugin-skill.md) | 插件与技能系统 |
| [12-ui-components.md](./12-ui-components.md) | React + Ink 终端 UI 组件体系 |
| [13-agent-coordinator.md](./13-agent-coordinator.md) | 多 Agent 协调模式 |

---

## 推荐阅读顺序

```
00-overview       → 建立全局视图
01-startup        → 理解启动流程
03-tool-system    → 理解工具接口设计
04-tools-impl     → 理解具体工具实现
02-query-engine   → 理解核心 AI 对话循环（最复杂）
07-api-client     → 理解 API 通信层
06-auth-config    → 理解认证与配置
05-mcp-protocol   → 理解 MCP 扩展机制
08~13             → 按需深入各扩展功能
```

---

## 系统简介

Claude Code 是 Anthropic 出品的 AI 编程助手 CLI 工具，本质是：

> **一个以 Claude API 为后端、React + Ink 为前端、运行在终端里的 AI Agent 系统**

**技术栈**: Bun · TypeScript · React + Ink · Zod v4 · Commander.js · MCP · LSP
