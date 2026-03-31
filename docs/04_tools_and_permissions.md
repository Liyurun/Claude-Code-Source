# 04 工具系统、权限与沙箱隔离

这部分决定了 AI 能做什么，以及如何防止它把用户的机器搞坏。

## Tool 抽象接口：`src/Tool.ts`

整个库对外挂载能力的抽象核心。每个 Tool 必须符合 `Tool` 接口：
- `name` 与 `aliases`
- `inputSchema`：Zod 对象，用于向 LLM 声明自己需要什么格式的 JSON 参数。
- `description()`：告诉大模型这个工具是干什么的（可以根据上下文动态生成）。
- `call()`：实际执行逻辑，传入解析好的参数。
- `isDestructive` 与 `isReadOnly`：标记其是否会改变外部环境，这极大地影响免密模式下它的表现。

### 内建工具集合 (`src/tools.ts`)
- **BashTool** / **PowerShellTool**：执行终端脚本。
- **FileReadTool** / **FileEditTool** / **FileWriteTool**：最常用的文件系统读改写能力。
- **GlobTool** / **GrepTool**：高效率文件与内容搜索（系统默认用更快的 Ripgrep / fd 进行了替换封装）。
- **NotebookEditTool**：针对 Jupyter 笔记本格式的特殊编辑。
- **WebFetchTool** / **WebSearchTool**：上网拉取资料能力。
- **AgentTool**：能够“孵化子代”，启动一个具有特殊任务设定的子 Agent 进程。

## BashTool 的深入防护 (`src/tools/BashTool`)

最容易出问题的是 Shell 执行，因为 LLM 可能生成了 `rm -rf /` 或复杂的渗透提权脚本。

### 1. 语义探查
`isSearchOrReadBashCommand` 等函数会通过内置的轻量 AST/字符串分割，分析命令是不是只读的（比如由 `find`, `cat`, `jq`, `grep` 组成的管道）。
如果是读，就标为相对安全，UI 展示上也可以直接折叠输出；如果是具有副作用的指令，会在终端以更显眼的红色/黄色警告出现。

### 2. 沙箱介入
`shouldUseSandbox.ts`：如果是高危命令，且当前环境启用了隔离特性。
`SandboxManager` (`utils/sandbox/sandbox-adapter.ts`) 会将被执行的命令隔离在一个类似于 Docker / SECCOMP 的受限环境中运行。网络请求、特定目录的写权限会被剥夺。

### 3. 权限模型 (Permission Mode)
`src/utils/permissions/permissionSetup.ts` 建立了一个防御机制。
如果用户开启了 Auto Mode（自动模式），系统会主动过滤**极其危险的规则**：
例如，如果有规则声明允许 `Bash(*)`（允许跑任何命令），或者匹配到了 `python -c` 这样的模式（可以在 bash 里绕开直接执行任意代码）。它会被 `isDangerousBashPermission` 捕获并强制要求用户手动干预。

## Workspace Trust 机制

除了底层的沙箱，最外层在 `src/interactiveHelpers.tsx` 内通过 `TrustDialog` 来决定：
- 你在这个目录里，是否愿意让它读改写。
- 当 `.claude/settings.json` 被其他协作者提交下来时，如果不经确认直接生效，可能会诱导系统执行恶意指令（通过替换工具参数或覆盖工作流脚本）。因此系统在初始化完毕真正把控制权交给 Agent 前，会通过 Trust Dialog 将一切未知阻挡在外。