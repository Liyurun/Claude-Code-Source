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

6. **[代码库统计与体验优势深度解析](./docs/06_codebase_stats_and_insights.md)**
   - 全局文件数量与超大模块占比分析。
   - 为什么 Claude Code 用起来那么爽？揭秘其极限上下文截断、终端渲染引擎与多任务架构等工程亮点。

## 深度长文解析系列 (万字长文揭秘)

- [揭秘 Claude Code：从 51 万行源码看终端 Agent 的工程化极限（上）](./docs/08_deep_dive_architecture_part1.md)
  - 宏观架构设计与四大核心支柱。
- [揭秘 Claude Code：从 51 万行源码看终端 Agent 的工程化极限（中）](./docs/09_deep_dive_architecture_part2.md)
  - 解析三大痛点解决方案：沙箱控制、上下文压缩技术与 React 终端 UI。
- [揭秘 Claude Code：从 51 万行源码看终端 Agent 的工程化极限（下）](./docs/10_deep_dive_architecture_part3.md)
  - 真实 Case 追踪：“修改组件Bug”时的代码调用链，以及 MCP 协议大一统的前瞻。

## 核心设计理念速览

- **Agent Runtime**：它不是简单的 OpenAI wrapper，而是一个包含多任务调度、权限管控、本地沙箱验证、复杂长下文压缩的运行环境。
- **UI 与逻辑解耦**：终端绘制（Ink）与核心查询逻辑分离，允许其在 `--print` 模式、甚至远程模式（`--remote`）下无缝工作。
- **高安全性限制**：内置大量规则以识别危险的 Bash 和 PowerShell 命令，并在无沙箱或未显式授权时进行拦截。
- **模块动态加载**：大规模使用 `await import()`，通过 Feature Flags (GrowthBook) 或环境变量控制功能开关，以保证基础启动性能（极度关注 TTI）。