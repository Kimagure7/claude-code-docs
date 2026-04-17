# 12. CLI UI 渲染系统（React + Ink）

## 1. 模块职责

Claude Code 的 CLI 界面并非普通的文本打印，而是一套基于 **React + Ink** 的完整终端 UI 框架，支持实时流式输出、键盘事件处理、文本选择、鼠标交互和高性能差异化渲染。Ink 是项目内嵌的定制版（非 npm 版），相比原版增加了双缓冲、DECSTBM 硬件滚动、字符池优化、Kitty 键盘协议支持等关键能力。

## 2. 核心文件地图

| 文件 | 行数 | 描述 |
|------|------|------|
| `src/ink/ink.tsx` | 1722 | Ink 主类：生命周期、渲染调度、事件分发 |
| `src/ink/screen.ts` | 1486 | 虚拟屏幕缓冲区：双缓冲、字符池、样式池 |
| `src/ink/render-node-to-output.ts` | 1462 | DOM 树 → 屏幕缓冲区转换（布局遍历） |
| `src/ink/log-update.ts` | 773 | 差异化 ANSI 渲染：帧间对比，最小化终端写入 |
| `src/ink/parse-keypress.ts` | 801 | 键盘输入解析：支持 6 种终端键盘协议 |
| `src/ink/selection.ts` | 917 | 文本选择与复制（鼠标选取） |
| `src/ink/styles.ts` | 771 | 样式系统：颜色转换、ANSI 属性映射 |
| `src/components/App.tsx` | 55 | 应用顶层组件：Provider 嵌套入口 |
| `src/components/` | 146个文件 | 全部 UI 组件 |
| `src/hooks/` | 85个文件 | 自定义 React Hooks |

## 3. 关键数据结构

### 3.1 Ink 主类字段（渲染引擎核心状态）

```typescript
// src/ink/ink.tsx:71-121
export default class Ink {
  private readonly log: LogUpdate         // 差异化渲染器
  private readonly terminal: Terminal     // 终端能力检测与写入
  private scheduleRender: () => void      // 节流 40ms 的渲染调度函数

  private isUnmounted = false             // 卸载标志（防止退出后渲染）
  private isPaused = false                // SIGTSTP 暂停标志

  private readonly container: FiberRoot   // React Reconciler 容器
  private rootNode: dom.DOMElement        // Ink 虚拟 DOM 根节点
  readonly focusManager: FocusManager     // 焦点管理（Tab 键切换）

  private frontFrame: Frame              // 当前显示帧（双缓冲前缓冲）
  private backFrame: Frame               // 渲染中的帧（双缓冲后缓冲）

  private readonly stylePool: StylePool  // 样式 ID 池（跨帧共享）
  private charPool: CharPool             // 字符串驻留池
  private hyperlinkPool: HyperlinkPool   // 超链接驻留池

  readonly selection: SelectionState = createSelectionState()  // 文本选择状态
  private searchHighlightQuery = ''      // 搜索高亮关键词
  private readonly hoveredNodes = new Set<dom.DOMElement>()  // 鼠标悬浮节点集合
  private altScreenActive = false        // 是否在备用屏幕（alt-screen）模式
}
```

### 3.2 CharPool — 字符串驻留池

```typescript
// src/ink/screen.ts:13-29
export class CharPool {
  private strings: string[] = [' ', '']  // 0=空格, 1=空（占位符）
  private stringMap = new Map<string, number>()
  private ascii: Int32Array = initCharAscii()  // ASCII 快路径：charCode → index

  intern(char: string): number {
    // ASCII 快路径：直接数组查找，避免 Map.get 开销
    if (char.length === 1) {
      const code = char.charCodeAt(0)
      if (code < 128) {
        const cached = this.ascii[code]!
        if (cached !== -1) return cached  // 命中：O(1) 查找
        const index = this.strings.length
        this.strings.push(char)
        this.ascii[code] = index
        return index
      }
    }
    // 非 ASCII：Map 查找
    const existing = this.stringMap.get(char)
    if (existing !== undefined) return existing
    const index = this.strings.length
    this.strings.push(char)
    this.stringMap.set(char, index)
    return index
  }
}
```

### 3.3 键盘协议正则表达式集

```typescript
// src/ink/parse-keypress.ts:18-54
// CSI u 协议（Kitty）: ESC [ codepoint [; modifier] u
const CSI_U_RE = /^\x1b\[(\d+)(?:;(\d+))?u/

// xterm modifyOtherKeys: ESC [ 27 ; modifier ; keycode ~
const MODIFY_OTHER_KEYS_RE = /^\x1b\[27;(\d+);(\d+)~/

// 传统功能键: ESC [ ... 以字母结尾
const FN_KEY_RE = /^(?:\x1b+)(O|N|\[|\[\[)(?:(\d+)(?:;(\d+))?([~^$])|(?:1;)?(\d+)?([a-zA-Z]))/

// SGR 鼠标事件: ESC [ < button ; col ; row M/m
const SGR_MOUSE_RE = /^\x1b\[<(\d+);(\d+);(\d+)([Mm])$/

// 终端响应模式查询
const DECRPM_RE = /^\x1b\[\?(\d+);(\d+)\$y$/   // DEC 私有模式状态响应
const XTVERSION_RE = /^\x1bP>\|(.*?)(?:\x07|\x1b\\)$/s  // 终端名称/版本
```

## 4. 执行流程图

### 4.1 整体渲染管道（每帧 6 阶段）

```
用户操作 / 状态变更
    │
    ├─ stdin 原始字节流
    │       │
    │       └─ parse-keypress.ts    # 解析转义序列 → KeyboardEvent / MouseEvent
    │               │
    │               └─ dispatchEvent → React 事件系统 → setState
    │
    scheduleRender()  [throttle 40ms ≈ 25fps]
    │
    onRender()  [6 子阶段]
    │    │
    │    ├─ 1. React Reconciler reconcile    # React diff + 提交到虚拟 DOM
    │    ├─ 2. Yoga 布局计算                  # Flexbox 布局，计算每个节点的 x/y/w/h
    │    ├─ 3. renderNodeToOutput()          # DOM 树 → 后缓冲区（backFrame.screen）
    │    ├─ 4. 双缓冲交换                     # frontFrame ↔ backFrame 原子交换
    │    ├─ 5. writeDiffToTerminal()         # 前后帧 ANSI 差异 → stdout 写入
    │    └─ 6. 选择/搜索高亮覆盖层            # applySelectionOverlay / applySearchHighlight
    │
    终端显示更新
```

### 4.2 差异化渲染流程（减少写入量 70-90%）

```
backFrame.screen（新帧）
    │
    └─ diffEach(frontFrame, backFrame)   # 逐单元格 ID 比较（整数比较，无字符串分配）
            │
            ├─ 相同单元格 → 跳过（零写入）
            └─ 变更单元格 → 生成 ANSI 差异序列
                    │
                    ├─ 移动光标（cursorMove CSI 序列）
                    ├─ 输出颜色/样式差异（SGR 属性）
                    ├─ 输出字符内容
                    └─ 硬件滚动优化（DECSTBM，仅 scrollTop 变化时）
```

### 4.3 App 组件层次结构

```
App.tsx (56行，编译器优化版)
    │
    ├─ FpsMetricsProvider      # 帧率指标 Context
    │   └─ StatsProvider       # 统计信息 Context
    │       └─ AppStateProvider  # 全局应用状态 Context（最内层）
    │               │
    │               └─ children（实际 UI 组件树）
    │
ink/components/App.tsx (Ink 内部 App)
    ├─ CursorDeclarationContext
    ├─ ScrollDeclarationContext
    ├─ FocusContext（Tab 焦点管理）
    ├─ TerminalWriteProvider
    ├─ InternalApp
    │    ├─ stdin 事件监听（键盘/鼠标/resize）
    │    └─ 用户组件树
    └─ AlternateScreen（可选）
```

## 5. 关键代码解析

### 5.1 Ink 构造函数与渲染调度

```typescript
// src/ink/ink.tsx（构造器核心片段）
export default class Ink {
  constructor(options: Options) {
    // 初始化 React Fiber 并发渲染器
    this.container = reconciler.createContainer(
      this.rootNode,       // Ink 虚拟 DOM 作为根
      ConcurrentRoot,      // 并发模式（支持 Suspense / Transition）
      null, false, null, '', false, null
    )

    // 节流渲染调度：最高 40ms 一帧（25fps），避免高频状态更新导致性能问题
    this.scheduleRender = throttle(
      this.onRender.bind(this),
      FRAME_INTERVAL_MS,   // 40ms
      { leading: false, trailing: true }
    )

    // 注册 signal-exit：程序退出时确保恢复终端状态
    onExit(() => this.cleanup())
  }
```

### 5.2 CharPool + HyperlinkPool — 内存优化设计

```typescript
// src/ink/screen.ts:31-41（HyperlinkPool）
export class HyperlinkPool {
  private strings: string[] = ['']    // 0 = 无超链接的哨兵值
  private stringMap = new Map<string, number>()

  intern(hyperlink: string | undefined): number {
    if (!hyperlink) return 0          // 无超链接：直接返回 0，无 Map 查找
    let id = this.stringMap.get(hyperlink)
    if (id === undefined) {
      id = this.strings.length
      this.strings.push(hyperlink)
      this.stringMap.set(hyperlink, id)
    }
    return id
  }

  get(id: number): string | undefined {
    return id === 0 ? undefined : this.strings[id]  // 0 映射回 undefined
  }
}
// 设计价值：同一 URL 字符串在所有 Cell 中只存储一次
// 单元格只存 4 字节整数 ID，大幅减少内存占用和 diffEach 的比较开销
```

### 5.3 差异化 ANSI 渲染 — renderFullFrame

```typescript
// src/ink/log-update.ts:45-70
private renderFullFrame(frame: Frame): Diff {
  const { screen } = frame
  const lines: string[] = []
  let currentStyles: AnsiCode[] = []
  let currentHyperlink: Hyperlink = undefined

  for (let y = 0; y < screen.height; y++) {
    let line = ''
    for (let x = 0; x < screen.width; x++) {
      const cell = cellAt(screen, x, y)
      if (cell && cell.width !== CellWidth.SpacerTail) {  // 跳过宽字符占位格
        // 超链接切换：输出 OSC 8 序列
        if (cell.hyperlink !== currentHyperlink) {
          if (currentHyperlink !== undefined) line += LINK_END
          if (cell.hyperlink !== undefined) line += oscLink(cell.hyperlink)
          currentHyperlink = cell.hyperlink
        }
        // 样式差异：只输出变化的 ANSI 属性（避免重复全量重置）
        const cellStyles = this.options.stylePool.get(cell.styleId)
        const styleDiff = diffAnsiCodes(currentStyles, cellStyles)
        if (styleDiff.length > 0) {
          line += ansiCodesToString(styleDiff)   // 仅输出增量属性
          currentStyles = cellStyles
        }
        line += cell.char
      }
    }
    // 行尾：关闭悬挂超链接 + 重置样式（trimEnd 前重置，防止尾部空格带颜色）
    if (currentHyperlink !== undefined) { line += LINK_END; currentHyperlink = undefined }
    const resetCodes = diffAnsiCodes(currentStyles, [])
    if (resetCodes.length > 0) { line += ansiCodesToString(resetCodes); currentStyles = [] }
    lines.push(line.trimEnd())
  }
  return lines.length === 0 ? [] : [{ type: 'stdout', content: lines.join('\n') }]
}
```

### 5.4 DOM → 输出转换的布局优化变量

```typescript
// src/ink/render-node-to-output.ts:21-67
// 布局变更检测标志：每帧重置，任何节点位置/尺寸变化时置 true
// ink.tsx 读取此值来决定是否需要全帧重绘（"sledgehammer"）
// 稳态帧（仅 spinner/时钟/文本追加）不触发布局变更 → O(changed cells)差异
let layoutShifted = false

// DECSTBM 硬件滚动优化提示
// 当 ScrollBox 的 scrollTop 在帧间变化（且无其他布局变更）时
// log-update.ts 可以发送 DECSTBM+SU/SD 硬件滚动而非重写整个视口
// 性能提升高达 100x（硬件滚动 vs 软件重绘）
export type ScrollHint = { top: number; bottom: number; delta: number }
let scrollHint: ScrollHint | null = null

// 跟随滚动事件（流式输出时自动滚到底部）
// 记录 delta + 视口范围，供 ink.tsx 在渲染后调整文本选择位置
// 防止选中文本在内容滚动时"跑偏"
export type FollowScroll = { delta: number; viewportTop: number; viewportBottom: number }
let followScroll: FollowScroll | null = null
```

### 5.5 App 组件 — React Compiler 优化后的 Provider 嵌套

```typescript
// src/components/App.tsx:18-52（完整代码，已由 React Compiler 自动优化）
export function App(t0) {
  const $ = _c(9)  // React Compiler 分配 9 个 memo 槽
  const { getFpsMetrics, stats, initialState, children } = t0

  // memo 化每一层 Provider，只有 props 变化时才重建节点
  // 层次从内到外：AppState → Stats → FpsMetrics
  let t1
  if ($[0] !== children || $[1] !== initialState) {
    // 最内层：AppState Context，持有全局应用状态
    t1 = <AppStateProvider initialState={initialState} onChangeAppState={onChangeAppState}>
      {children}
    </AppStateProvider>
    $[0] = children; $[1] = initialState; $[2] = t1
  } else {
    t1 = $[2]  // 命中缓存：复用上一次渲染结果
  }

  let t2
  if ($[3] !== stats || $[4] !== t1) {
    t2 = <StatsProvider store={stats}>{t1}</StatsProvider>
    $[3] = stats; $[4] = t1; $[5] = t2
  } else {
    t2 = $[5]
  }

  let t3
  if ($[6] !== getFpsMetrics || $[7] !== t2) {
    // 最外层：FPS 指标 Context（变化最频繁，放最外层减少内层重渲染）
    t3 = <FpsMetricsProvider getFpsMetrics={getFpsMetrics}>{t2}</FpsMetricsProvider>
    $[6] = getFpsMetrics; $[7] = t2; $[8] = t3
  } else {
    t3 = $[8]
  }
  return t3
}
// React Compiler 将 useMemo 模式自动内联为槽位缓存，消除运行时 memo 开销
```

## 6. 设计思路总结

### 双缓冲渲染防撕裂
Ink 维护 `frontFrame`（当前显示）和 `backFrame`（渲染中）两块屏幕缓冲区。每帧渲染完成后原子交换，确保 `writeDiffToTerminal` 总是读到完整的新帧，不会出现部分更新导致的视觉撕裂。

### 字符池驻留 + 整数 ID 比较
每个屏幕单元格（Cell）不直接存储字符串，而是存储 `CharPool` 中的整数 ID。`diffEach` 比较两帧时只需整数比较，避免字符串哈希和内存分配。ASCII 字符走 `Int32Array` 快路径，命中率极高（~80% 的终端字符为 ASCII）。

### DECSTBM 硬件滚动优化
当 ScrollBox 仅 `scrollTop` 变化（无布局变更）时，利用 `DECSTBM`（终端硬件滚动区域）+ `SU/SD`（向上/下滚动行）命令，让终端硬件执行行移动而非软件重绘所有行。理论性能提升 100x，对流式输出场景（内容持续追加）效果显著。

### 6 种键盘协议兼容栈
终端键盘协议碎片化严重，从 SSH 连接的老式 VT100 到本地 Ghostty/Kitty 的现代协议各不相同。`parse-keypress.ts` 构建了协议优先级栈（Kitty > modifyOtherKeys > 传统转义 > ASCII），通过终端版本探测（XTVERSION）自动选择最优协议，向下兼容同时利用现代特性。

### React Compiler 自动 Memo
`App.tsx` 中的手动 memo 槽位代码是 **React Compiler 的编译输出**，而非手写代码。编译器将 `useMemo` 模式内联为固定大小的槽位数组 `_c(9)`，消除了 `useMemo` 本身的 Hook 调用开销，同时保持精确的依赖追踪。

## 7. 与其他模块的联系

```
UI 渲染系统
    │
    ├─→ 查询引擎 (02-query-engine.md)
    │       流式 token 通过 React 状态更新触发 scheduleRender
    │       每个 token 不直接触发渲染，而是批量在 40ms 内合并
    │
    ├─→ Agent 循环 (02-query-engine.md)
    │       思考过程、工具调用结果、错误信息均通过
    │       React 组件展示，状态通过 AppStateContext 传递
    │
    ├─→ 工具系统 (03-tool-system.md)
    │       BashTool 等工具的实时输出通过流式 React 状态更新
    │       在 ToolOutputComponent 中渲染
    │
    ├─→ 命令系统 (08-command-system.md)
    │       / 命令的补全列表、当前命令状态均为 React 组件
    │       useInput hook 处理 Tab 键触发补全
    │
    └─→ 认证配置 (06-auth-config.md)
            登录流程 (OAuth) 展示在专用 React 组件中
            配置错误通过 AppState 传递给 UI 层渲染
```

**关键依赖**：
- `react-reconciler` — Ink 基于自定义 reconciler 将 React 组件映射到虚拟 DOM
- `yoga-layout` — Flexbox 布局引擎（WebAssembly 编译，处理终端字符宽度）
- `@alcalzone/ansi-tokenize` — ANSI 转义序列解析与差异计算
- `signal-exit` — 进程退出时恢复终端状态（重要：防止终端损坏）
