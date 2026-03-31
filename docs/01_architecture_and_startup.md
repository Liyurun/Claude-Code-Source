# 01 总体架构与启动流程

Claude Code 的启动并非只是简单地解析参数发请求，它建立了一个完整的环境，包括了配置预取、跨进程数据收集、环境隔离及安全沙箱初始化。

## 真实入口：`src/entrypoints/cli.tsx`

`cli.tsx` 是真正的第一落脚点。为了保证最快的响应速度（例如 `--version`），这个文件极其轻量，其主要功能是：

1. **环境变量清理与准备**：处理 Node 的最大内存等问题（例如为远程容器预留 8GB 堆）。
2. **Fast-path 分流**：
   - 如果是 `claude --version`，直接打印内置宏变量并退出，不加载任何后续库。
   - 识别各种隐式模式入口，例如：`--claude-in-chrome-mcp`, `--chrome-native-host`, `daemon` 工作进程。
   - 识别 `bridge` 或 `remote-control` 模式。
3. **委托主线**：如果不是上述快速通道，它将导入 `../main.js`（对应源码的 `main.tsx`）并调用其上的 `run()` 函数。

## 完整装配与模式分发：`src/main.tsx`

进入 `main.tsx` 后，系统才开始挂载重量级的逻辑。

### 副作用与前置依赖
在 `main.tsx` 的最开头（Imports之前），它就会立即调用：
- `profileCheckpoint` 记录启动耗时。
- `startMdmRawRead()` 并行读取企业 MDM 策略（注册表/macOS描述文件）。
- `startKeychainPrefetch()` 并行读取系统的钥匙串（获取已有的 OAuth Token）。

### `run()` 与 `Commander` 拦截
它使用 `@commander-js/extra-typings` 声明所有的参数。
最巧妙的是，它利用 Commander 的 `.hook('preAction', ...)` 进行**拦截初始化**：

```typescript
program.hook('preAction', async thisCommand => {
    // 1. 等待 MDM 与 Keychain 异步预取完成
    await Promise.all([ensureMdmSettingsLoaded(), ensureKeychainPrefetchCompleted()]);
    // 2. 核心底座初始化
    await init();
    // 3. 挂载日志 Sink
    const { initSinks } = await import('./utils/sinks.js');
    initSinks();
    // 4. 其他...
});
```

### 动作分发 (Action Handler)
在 `.action(async (prompt, options) => { ... })` 中，执行了以下关键操作：
1. **构建工具与权限模型**：`getTools(toolPermissionContext)` 获取当前所有允许挂载的内置 Tools。
2. **Setup 流程**：调用 `src/setup.ts`，准备 Socket 通信、Git 工作区上下文。
3. **命令聚合**：并行调用 `getCommands()` 收集内置命令、插件命令和本地技能。
4. **渲染引导屏幕**：调用 `showSetupScreens()`。

## 安全与初始化基座：`src/entrypoints/init.ts`

`preAction` 调用的 `init()` 函数才是系统真正的“底座起搏器”。它处理：
1. **环境变量沙箱**：先加载安全的环境变量，等待信任后才加载完整环境变量。
2. **TLS / 代理设置**：加载 CA 证书，配置全局 mTLS 以及代理，保证后续所有网络请求的合法性。
3. **遥测延迟加载**：通过异步导入的方式，延迟加载 OpenTelemetry 库，不拖慢界面出现的时间。
4. **Scratchpad 目录**：如果是需要沙箱化运行的模式，在这里分配隔离的临时运行目录。

## 交互式的终点：`showSetupScreens()` 与 `launchRepl()`

在 `src/interactiveHelpers.tsx` 里的 `showSetupScreens()` 负责给用户看那些**必须确认的东西**：
1. **Onboarding**：新手引导界面。
2. **Trust Dialog (信任边界)**：检查当前目录是不是一个受信任的 Git 仓库，拦截在未知目录下被恶意代码反噬的风险。
3. **MCP 授权**：发现新的 MCP Server 配置时弹出。
4. **外部文件包含警告**：如果 `CLAUDE.md` 企图去读取当前项目外的绝对路径，在此阻拦。

完成这一切后，系统调用 `src/replLauncher.tsx` 的 `launchRepl()`。
此时，React 树正式挂载：
`<App><REPL /></App>` 
这标志着生命周期正式交给了 React 的事件循环。