# 02 命令系统与 UI 交互层

在进入 REPL 之前，用户输入的东西需要被正确地分类：是一条和系统对话的命令（如 `/clear`）还是打算发给 LLM 的指令？

## 斜杠命令系统：`src/commands.ts`

`src/commands.ts` 是命令的大脑，采用了一种非常模块化的架构：

### 1. 命令来源的多样化
这里维护了一个 `getCommands()` 异步函数，它每次调用时都会去取以下不同来源的命令进行合并：
- **内建命令**：写死在代码里的 `/help`, `/cost`, `/session`, `/clear` 等。
- **技能（Skills）**：来自工作区 `.claude/skills` 或系统级别存放的 Prompt 命令。
- **插件命令**：通过插件系统扩展加载。
- **MCP 注入命令**：一些配置好的 MCP 同样能暴露出终端命令。

### 2. 命令类型 (CommandType)
一个 Command 不仅仅是一个“可执行函数”，它被分为几种具体执行类型：
- **`local-jsx`**：代表纯前端交互命令（如 `/config` 弹出一个表单），这种命令会直接注入 React 树 `setToolJSX`。
- **`prompt`**：并不是立刻执行某个本地行为，而是把一段特殊构造的 Prompt 隐式喂给模型。例如 `/review` 就是拼好审查代码的要求发给模型。

### 3. 可用性过滤
`meetsAvailabilityRequirement(cmd)` 检查特定功能是否需要特定权限：
例如，只有真正的 API key 用户或者特定的计费配置，才能看到某些内建排障命令。这会在每次请求提示时动态重新评估，保证“退出登录”时相关命令立刻消失。

## REPL 组件：总装车间 `src/screens/REPL.tsx`

`REPL.tsx` 是整个项目**最为复杂**的前端组件。它不仅渲染文本，更是控制整个事件循环的主循环。

### 1. Hook 驱动的设计
它不再是类组件中大块头的 `render` 方法，而是把不同子域的能力抽成了几十个 Hooks，例如：
```tsx
const { queue, isProcessing } = useCommandQueue()
const { activeSession } = useRemoteSession()
const { pollInbox } = useMailboxBridge()
// ... 还有处理输入缓冲、处理快捷键历史、处理 IDE 状态指示等的 hooks
```

### 2. 状态下放
UI 并没有把一切放入 `useState`，而是极大地依赖全局的 `AppStateStore`。
这让背景任务、网络事件（如 MCP 断线重连）可以直接更新 `AppState`，然后 `REPL.tsx` 响应重绘，不用层层通过 props 回传。

### 3. 拦截器与生命周期
当用户在控制台按下 Enter 时：
- `handlePromptSubmit`（在 `src/utils/handlePromptSubmit.ts`）被触发。
- 它检测是否为 `exit/quit/:q`，转为触发保存逻辑。
- 解析文本中的拖拽内容、本地文件引用块。
- 如果是斜杠命令 `/xxx`，寻找 `commands` 列表匹配：
  - 如果匹配到 `local-jsx`（比如设置、任务管理），立刻弹出对话框，并清空输入。
  - 如果不匹配，或者是一个正常的对话文字，它将会将其打包为 `Message` 推给 `QueryGuard`。
- 最后调用并挂起 `query()` （进入 Agent 主循环）。

### 4. 纯展示与交互组件
在 `src/components/` 目录下，包含大量子界面：
- `Message.tsx` / `Messages.tsx`：负责将模型返回的内容，以及模型使用工具的中间过程（如 “Bash 运行中…”）以不同样式打印到终端。
- `Spinner` 系列：不仅能转圈，还能根据状态流（如 MCP 响应迟缓）变换提示文字。
- `FileEditToolDiff.tsx`：直观展示文本修改时的前后 Diff，给用户强烈的掌控感。