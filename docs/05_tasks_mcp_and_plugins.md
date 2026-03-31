# 05 任务、MCP 协议、扩展与源码还原局限

如何让单兵作战的 LLM 能够协同工作？如何集成组织内部或三方的非标工具？这就依赖于任务引擎与扩展协议。

## 多任务与并发系统 (Tasks System)

在 `src/tasks/` 目录下可以看到它不仅仅是个聊天应用。
主从模式：一个主会话（Main Session）随时可以把自己的当前动作推到后台（如按两下 `Ctrl+B`）。
`LocalMainSessionTask.ts` 实现了 `registerMainSessionTask`：
1. 它分配一个新的独立 `taskId` 和隔离的持久化记录（Transcript file）。
2. 在这个挂起状态下，复用刚才在 `queryLoop` 里的参数，把它放入全局 `AppState.tasks`。
3. 终端 UI 立刻弹回空白的 Prompt。
4. 模型在后台默默读代码、执行 Bash；当它执行完毕，底层通过 `completeMainSessionTask` 回调改变状态，并在用户的 UI 底部发送一个消息通知 (Notification)。

与此同时，通过 `AgentTool` 创建出来的子代 Agent，也同样利用了这套体系：它会被看成一个 "Task"，拥有自己的状态机、内存预算和工具池。

## 外部接入的心脏：MCP 协议

MCP（Model Context Protocol）用于连接外部的数据源与工具，代码集中于 `src/services/mcp/`。

### 1. 协议通道（Transports）
代码原生接入了 `@modelcontextprotocol/sdk`。支持多种底层通道与认证方式：
- **stdio**：拉起一个子进程并在标准输入输出流上使用 JSON-RPC 交互。
- **SSE (Server-Sent Events)**：针对 Web 服务端的持久长链接。
- **WebSocket**：提供给远端需要双向全双工通讯的服务器。

### 2. 映射关系
外部的 MCP 服务可能暴露了各种各样乱七八糟的 Tools 和 Resources，系统如何处理呢？
- `src/services/mcp/client.ts` 提供了核心网关。
- `ListMcpResourcesTool.ts` / `ReadMcpResourceTool.ts`：将 MCP 的 "Resource" 体系映射为了我们主模型可以主动调用的内置工具。模型如果需要知道有哪些 MCP，可以调用这些内置工具先查询。
- 自动提取：如果有 MCP 提供了 Tools，会被合并并在 `src/tools.ts` 里通过 `assembleToolPool` 直接放入与内建 Bash 同等地位的工具池内。

### 3. Elicitation 拦截
如果 MCP 返回错误代码 `-32042`，系统将其理解为「服务遇到了麻烦或者它需要让用户登录补充参数」。它能够挂起当前大模型的查询逻辑，利用 React UI（如 `ElicitationDialog.tsx`）弹出窗口，拿到补充信息后再恢复该 MCP 工具的调用。

## 源码还原的局限性

虽然我们可以从 `cli.js.map` 得到了接近 5000 个源文件（涵盖了大量核心业务逻辑、工具抽象、UI 声明与工具），但必须清楚：这**不是**一份能用来直接跑 `npm start` 的开源代码仓库。

1. **结构不完整**：丢失了 `tsconfig.json`、原始的完整 `package.json`（带有 scripts 的版本）以及其他在编译前就被忽略的构建态脚本。
2. **闭源依赖引用缺失**：例如代码里广泛出现的：
   ```ts
   const proactive = feature('PROACTIVE') ? require('./commands/proactive.js') : null
   const assistantModule = feature('KAIROS') ? require('./assistant/index.js') : null
   ```
   在这份还原文件里，`assistant/index.js` 等很多“可能属于另一层面的功能包”或“未在最终包发布”模块是不存在的。由于引用断裂，如果要强行编译会遇到大量模块找不到（MODULE_NOT_FOUND）的报错。
3. **条件编译（Dead Code Elimination）丢失的代码**：通过宏（Macro）或者打包器在打包时删掉的代码，永远从 sourcemap 消失了，因此很难恢复全貌。

> 总结：这是一份极好的用于“学习高阶 Agent 系统设计模式、了解前端 Ink 极限开发能力以及企业级 LLM 防护思路”的技术文档库，但不是一套可即插即用的复刻版本。