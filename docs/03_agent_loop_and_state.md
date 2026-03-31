# 03 Agent 核心：查询循环与状态管理

这是整个项目的核心大脑。Claude Code 并非简单把历史对话一股脑推给 LLM，它通过一套严密的 `queryLoop` 控制会话生命周期、预算和记忆长度。

## 查询引擎的入口：`src/query.ts`

`query()` 函数接收消息数组、用户上下文、可用的工具、并返回一个**异步生成器 (AsyncGenerator)**，供 REPL 消费并渲染 UI 进度。

### `queryLoop()` 的流水线
每一次回答，或大模型需要链式调用工具时，都会在 `while (true)` 里发生以下阶段：

1. **上下文修剪 (Microcompact & Autocompact)**
   模型窗口有限，系统会利用几种机制主动缩小发往 LLM 的负载：
   - **`snipCompact`**：基于特定的“修剪标记”，丢弃中间无用的日志。
   - **`microcompact`**：将工具的结果（尤其是那些非常长但未被系统利用的读取结果）进行激进裁剪，只留下必要的元信息。
   - **`autocompact`**：如果 Tokens 已经极高，系统会自动触发类似于总结的动作，将历史多次对话浓缩为一条摘要（由 `TaskSummary` 之类的模块支持），保证新的对话不会因为超出 max_tokens 被拒。

2. **工具开销与限制 (Tool Result Budget)**
   `applyToolResultBudget`：如果工具返回的数据极大（如 `cat` 一个 50MB 的日志），为了防止冲爆 Prompt，它会被截断，并持久化到本地一个缓存文件中，模型实际上看到的是一个“预览”加上“该文件太长已保存至路径 X”的说明。

3. **并发调用与分发 (runTools)**
   向模型发出流式请求。模型返回的 `tool_use` 可能是多个。
   - `StreamingToolExecutor`：能够一边接收模型生成参数，一边解析，在收到完整 Schema 后立刻交由具体的工具模块（如 Bash, Edit）去执行。
   - 工具执行完后，收集结果，封装为 `user` 角色携带 `tool_result` 类型的消息。

4. **过渡判定 (Transitions)**
   - 如果 LLM 发起了工具调用并拿到了结果，循环**继续**（`Continue`），开启下一个迭代。
   - 如果遇到权限拒绝（Denied），进入提示模式。
   - 如果对话自然完结（LLM 发出了最终总结文字），循环**退出**（`Terminal`）。

## 工具交互上下文：`ToolUseContext`

`src/Tool.ts` 中定义的 `ToolUseContext` 承载了单个回合极其庞大的状态面：
它让每一个单独的 Tool 知道：
- 它是由主进程还是后台子任务触发的 (`agentId`)。
- 当下的截断预算还有多少 (`contentReplacementState`)。
- UI 是否期待弹出一个输入框来让用户回答 (`requestPrompt`)。

## 全局状态管理：`AppState`

系统摒弃了零散的状态，将跨组件的需求归一在 `src/state/AppStateStore.ts`。

### 状态树概览
通过一个朴素的、支持订阅的自定义小 Store（`createStore`），它维护着：
- `mcp`：已连接的各种协议的服务器列表。
- `plugins`：安装、加载中或挂掉的插件池。
- `tasks`：非常关键，它包含了当前运行的所有“并行会话”（见下文任务系统）。
- `replBridge`：控制当前是否开放给远程 IDE、外部平台使用的网络桥接状态。
- `toolPermissionContext`：管理着目前用户对哪些命令设置了 `alwaysAllow` 或 `alwaysDeny` 的规则。

这种设计使得无论是后台定时任务、还是底层 Bash 执行器，都能在需要时 `setAppState` 来通知顶部 UI（例如在终端底部弹出新完成的 TODO 指示）。