# Claude Code 源码架构解析（基于 cli.js.map 还原）

这份文档集合是对从 `cli.js.map` 中提取出来的源码进行的全面架构分析。
请注意，由于这并非原始 Git 仓库，部分未被打包引用的代码可能缺失（如配置、纯测试等），但这足以让我们看清其核心骨架。

## 文档导航

1. **[总体架构与启动流程](./docs/01_architecture_and_startup.md)**
   - 宏观视角：这不是一个简单的 CLI 脚本，而是一个“带终端 UI 的智能代理操作系统”。
   - 解析从进程入口到 REPL 界面加载的全生命周期。

2. **[命令系统与 UI 交互层](./docs/02_commands_and_ui.md)**
   - 讲解斜杠命令（`/help`, `/cost` 等）的注册、加载与分发机制。
   - 解析基于 React/Ink 构建的终端 UI，特别是大总管 `REPL.tsx` 是如何运作的。

3. **[Agent 核心：查询循环与状态管理](./docs/03_agent_loop_and_state.md)**
   - 揭秘核心对话与工具调用链：`queryLoop`。
   - 讲解大上下文如何通过 Snip、Microcompact、Autocompact 机制进行压缩与存档。
   - 剖析全局神仙状态 `AppState` 的设计与流转。

4. **[工具、权限与沙箱隔离](./docs/04_tools_and_permissions.md)**
   - Tool 抽象：从内建到外接工具的统一接口。
   - 深入 Bash/Edit 等系统级工具的实现。
   - 详细解读 Trust Dialog、ToolPermissionContext 以及自动模式下对高危操作的阻断。

5. **[任务、MCP 协议与插件扩展](./docs/05_tasks_mcp_and_plugins.md)**
   - 后台任务与 Agent 任务的分发机制。
   - MCP（Model Context Protocol）客户端是如何被深度集成到系统并转化为可用 Tool 的。
   - 插件与技能系统的动态加载方案。

## 核心设计理念速览

- **Agent Runtime**：它不是简单的 OpenAI wrapper，而是一个包含多任务调度、权限管控、本地沙箱验证、复杂长下文压缩的运行环境。
- **UI 与逻辑解耦**：终端绘制（Ink）与核心查询逻辑分离，允许其在 `--print` 模式、甚至远程模式（`--remote`）下无缝工作。
- **高安全性限制**：内置大量规则以识别危险的 Bash 和 PowerShell 命令，并在无沙箱或未显式授权时进行拦截。
- **模块动态加载**：大规模使用 `await import()`，通过 Feature Flags (GrowthBook) 或环境变量控制功能开关，以保证基础启动性能（极度关注 TTI）。