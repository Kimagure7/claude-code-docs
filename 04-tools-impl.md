# 04 - 核心工具实现细节

## 本文核心问题

03 讲了工具系统的接口设计，这篇讲**具体工具内部的有趣实现决策**——那些"为什么这样做而不是那样做"的地方。

覆盖：FileReadTool、FileWriteTool、GlobTool、BashTool、WebFetchTool、TodoWriteTool。

---

## 1. FileReadTool：读取不只是读文件

FileReadTool 最有意思的设计是**输出类型是一个 discriminated union**，而不是简单返回字符串：

```
text | image | notebook | pdf | parts | file_unchanged
```

每种文件类型走不同路径，返回不同结构。这意味着调用方（Claude）看到的不是"文件内容字符串"，而是有类型的数据——图片是 base64，notebook 是解析后的 cells，PDF 可以指定页码范围。

**为什么有 `file_unchanged` 类型？**  
Dedup 机制：如果同一文件在同一次对话里已经读过且文件没有变化（通过 mtime 判断），返回 `file_unchanged` 而不是再次传输内容。长对话里同一文件被多次提到时，这能节省大量 token。

**图片的 token 预算控制**：图片不是直接 base64 发出去，而是先估算会消耗多少 token，超过预算就先压缩（保持比例缩放），还超就激进压缩（JPEG 降质量）。这是少数在工具层就做 token 预算管理的地方。

**安全检查里有一条反直觉的**：阻止读取 `/dev/random`、`/dev/urandom` 等无限输出的设备文件。这些文件在 POSIX 系统上是合法路径，但读取会无限返回数据，直接让工具挂死。

**读文件时顺便激活技能**：`call()` 里有一行 `discoverSkillDirsForPaths([file_path])`，读文件的同时动态发现并激活与该路径相关的技能。这是"就近激活"的实现点——你打开某个目录下的文件，与该目录关联的技能自动可用。

---

## 2. FileWriteTool："先读后写"的强制约束

FileWriteTool 有一个不寻常的约束：**必须先读过这个文件才能写它**。具体实现是 validateInput() 里检查 `readFileState`——如果这个路径没有读取记录，直接拒绝写入。

为什么？两个原因：
1. **防止盲写**：强制 Claude 在修改文件前先了解文件当前内容，避免直接覆盖
2. **并发保护**：读取后记录 mtime，写入前再检查 mtime 是否变化——如果文件被其他进程修改过，拒绝写入，防止踩踏

写入后做了一整套通知链：更新 `readFileState`（允许后续继续写）、通知 LSP 服务器（触发重新诊断）、通知 VS Code（更新 diff 视图）。写一个文件不只是写磁盘，还是触发一系列下游更新的事件。

写入前有一个 secrets 检查——如果内容里包含 API Key 等敏感信息的模式，会被阻止写入。这在团队记忆文件（CLAUDE.md 类型文件）场景下尤为重要。

---

## 3. GlobTool：简单工具的一个细节

GlobTool 很简单，但有一个值得注意的点：结果上限是 100 个文件，超过后 `truncated: true`，并在返回给 Claude 的文本里明确标注 "(Results are truncated...)"。

为什么要让 Claude 知道结果被截断了？因为如果 Claude 不知道，它可能会基于"完整的"文件列表做出错误判断。明确告知截断让 Claude 知道需要更精确的 glob pattern。

路径处理：结果从绝对路径转成相对路径，节省 token。

---

## 4. BashTool：最复杂的工具

BashTool 的复杂性主要来自三件事：**命令分类、后台任务、大输出处理**。

**命令分类系统**：BashTool 维护多个命令集合（搜索类、读取类、列表类、静默类等），用于行为判断。比如 `mv`/`rm` 这类命令成功后通常没有 stdout 输出，UI 显示 "Done" 而不是空白的 "(No output)"，这是通过静默命令集合来判断的。

**Sleep 检测**：BashTool 专门检测命令里的 `sleep N`（N >= 2秒）并报错，提示用户改用 `run_in_background`。这是防止 Agent loop 被 sleep 命令阻塞的保护机制。类似地，看起来像服务器启动命令（如 `npm start`、`python server.py`）的命令会被自动转为后台执行。

**后台任务**：`run_in_background: true` 时，命令注册为 `LocalShellTask`，当前 tool_use 立即返回任务 ID。这让 Agent 可以启动多个长耗时命令（编译、测试、服务器）然后并发等待结果，而不是串行阻塞。

**大输出处理**：超过约 30KB 的输出被持久化到临时文件，tool_use 结果里只返回文件路径，告诉 Claude 用 FileReadTool 去读。这避免了把大量输出塞进消息历史里。

**`_simulatedSedEdit` 内部字段**：Schema 里有一个对外隐藏的字段，用于 sed 编辑的"快速路径"——直接写文件而不经过 Shell。这是个内部优化，避免启动 Shell 进程。

---

## 5. WebFetchTool：内容处理的两条路径

WebFetchTool 抓取网页后，内容不是原样返回，而是先转成 Markdown（去掉 HTML 标签）。这减少了大量噪音 token。

关键的分叉点：**是否经过 Claude 处理**。
- 预批准域名 + 内容较小 → 直接返回 Markdown，无额外 API 调用
- 其他情况 → 把内容传给 Claude 处理（`applyPromptToMarkdown`），提取用户真正需要的部分

这里有一个隐藏的 API 嵌套：WebFetchTool 内部可能会调用 Claude API。这意味着一次用户对话中，Agent 调用 WebFetchTool，WebFetchTool 又去调用 Claude——递归的 LLM 调用。

重定向处理是另一个细节：301/307/308 不自动跟随，而是返回给 Claude，让 Claude 决定是否抓取新 URL。理由：某些重定向可能带有安全或权限含义，不应该自动跟随。

---

## 6. TodoWriteTool：给自己设置的提醒机制

TodoWriteTool 最有趣的部分是**验证提示（Verification Nudge）**：当主 Agent 一次性关闭 3 个以上任务、且没有任何任务涉及"验证"时，在返回给 Claude 的消息里额外插入一段提醒："你刚关闭了多个任务，考虑添加验证步骤"。

这是系统通过工具结果反向影响 Claude 行为的一个例子——不是通过系统提示，而是通过工具调用的结果消息。本质上是把行为约束编码进工具里，而不是依赖提示工程。

---

## 设计规律

这几个工具里有几个共同的设计规律：

**结果里有元信息**：FileReadTool 返回 `numLines/totalLines`，GlobTool 返回 `durationMs/truncated`，BashTool 返回 `interrupted`。这些元信息让 Claude 能做更好的决策（比如知道文件被截断，或者知道命令被用户中断了）。

**安全约束在工具层而非权限层**：设备文件阻止、secrets 检测、sleep 检测——这些不是通用权限规则，而是嵌入在各工具的 `validateInput()` 里。通用权限规则处理"是否允许操作"，工具级别的校验处理"这个操作本身是否有意义"。

**大数据不进消息历史**：图片压缩、大输出持久化——都是在避免让大量数据进入消息历史。消息历史的体积直接影响每次 API 调用的成本和延迟。
