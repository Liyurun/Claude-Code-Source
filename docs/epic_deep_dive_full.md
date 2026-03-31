
---

## 第六部分：硬核拓展 —— 隐藏在源码中的 Prompt 工程与工具 Schema 哲学

在前面五部分的架构解析之外，如果我们把目光聚焦到 Claude Code 是如何具体“跟大模型沟通”的，会发现许多绝妙的 Prompt 技巧和 Schema（参数结构）设计。这些是很多开发者在调用大模型 API 时最容易写得很糙的地方。

### 6.1 动态级联的 System Prompt (`src/utils/systemPrompt.ts`)

在大部分开源项目中，系统提示（System Prompt）就是一个写死在字符串里的几十行文字。但在 Claude Code 中，由于存在主副 Agent、多任务以及 MCP 插件扩展，它的 System Prompt 是一个**复杂的优先级拼装函数**（`buildEffectiveSystemPrompt`）：

1. **绝对覆写 (Override)**：如果系统处于特殊循环模式，强行替换所有规则。
2. **协调者模式 (Coordinator)**：如果是主管 Agent（不干活，只分配任务），它会被加载一套纯粹用于拆解任务的 Prompt。
3. **主线程 Agent 叠加**：这非常巧妙。如果你使用 `AgentTool` 生成了一个专门写测试的子模型，系统的做法是：**保留基础生存规则**（比如怎么用工具、怎么认文件系统），但在最后追加一块 `\n# Custom Agent Instructions\n`，将子任务的设定（“你是一个专注的测试编写者”）贴上去。
4. **伴生模型 (Buddy Prompt)**：甚至在 UI 旁边偶尔跳出来吐槽的吉祥物（Buddy），也有专门的一套极简 Prompt：*“你是一个小宠物，当用户没叫你时只准回一句话，不要解释你不是谁谁谁，不准喧宾夺主。”* (`src/buddy/prompt.ts`)

**工程启示**：大型智能体的 Prompt 不应该是一个静态的大锅饭，而应该是像面向对象编程一样，有着基类（底层安全规则）和继承类（具体角色设定）的动态组装系统。

### 6.2 骗过大模型的 "Sed 模拟器" (FileEditTool Schema 设计)

让大模型改代码，最怕的就是它把原文件清空，只写了一句 `// 其余代码省略...`。

如果你去查阅 `src/tools/FileEditTool/types.ts`，会发现 Anthropic 的工程师为了让模型能精准编辑，并没有让它输出完整的文件内容，也没有使用极其硬核、容易出错的 Git Patch 格式，而是发明了一套类似“查找并替换”的抽象：

```typescript
// FileEditTool 期望模型输出的参数结构 (简化)
inputSchema: {
  file_path: "要改的文件绝对路径",
  old_string: "要被替换掉的原代码段",
  new_string: "要替换成的新代码段"
}
```

但这带来了一个问题：大模型很难算准缩进。比如它提供的 `old_string` 开头少了两个空格，这个替换在严格匹配下就会失败。

为了弥补大模型的“粗心”，在 `src/tools/FileEditTool/utils.ts` 中，系统实现了一个**智能对齐挂点（Smart Indent/Quote Matcher）**。当 `old_string` 无法在原文件中找到 100% 匹配时，系统会把它剥离空格、剥离换行甚至将单双引号模糊化，去寻找“实际上模型想要匹配的那段代码”，然后在替换时，强行套用原文件所在行的缩进量！

**工程启示**：永远不要相信大模型输出格式的严格程度。优秀的 Agent 工具应该把 Schema 定义得对模型足够简单直观（用 `old` 换 `new`），然后把处理边界条件（缩进、引号混用）的“脏活”通过纯代码在本地兜底。

### 6.3 BashTool 的极度克制：隐藏的内部 Schema

上面我们在沙箱防御里提到，如果模型要执行高危操作，系统会要求人工审批。但在自动化测试场景下，如果不给它一些自主权，系统就没法往下跑。

在 `src/tools/BashTool/BashTool.tsx` 中，我们看到了一个特别有意思的设计——**Schema 的两面派**：

在这个工具真实的运行时结构里，其实隐藏了一些极其特殊的内部参数，比如 `_simulatedSedEdit` 或者是 `dangerouslyDisableSandbox`：

```typescript
const fullInputSchema = z.strictObject({
  command: z.string().describe('The command to execute'),
  timeout: z.number().optional(),
  // 注意这段巨长的 Description，它在教模型怎么写描述词，甚至给出了具体例子
  description: z.string().optional().describe(`Clear, concise description of what this command does in active voice. Never use words like "complex" or "risk"...
For simple commands: ls → "List files"
For commands harder to parse: curl -s url | jq → "Fetch JSON..."`),
  
  // 危险字段，内部机制使用，不对外暴露
  dangerouslyDisableSandbox: z.boolean().optional(),
  _simulatedSedEdit: z.object({...}).optional()
});

// 在传递给大模型时，它把这些后门参数全部 omit (剔除) 掉了！
const inputSchema = fullInputSchema.omit({
  _simulatedSedEdit: true,
  dangerouslyDisableSandbox: true // 根据配置剔除
});
```

**工程启示**：向大模型暴露的 Tool Schema，其实就是攻击面（Attack Surface）。对于内部状态流转需要的参数，绝对不能写进发给 API 的 Schema 里，否则聪明的大模型（甚至是被 Prompt 注入黑掉的模型）极有可能会发现这些后门参数，从而自己给自己开启“免沙箱权限”强行提权。

### 6.4 终极外挂 MCP 协议：让模型自己发现“自己能干什么”

在 `src/services/mcp/client.ts` 文件中，揭示了 Claude Code 为什么敢宣称能接入一切企业内部系统。

Model Context Protocol (MCP) 是一个类似于 LSP（语言服务器协议）的东西，只不过它服务的是大模型。在这套代码里，它处理 MCP 接入的逻辑堪称典范：

当 Claude Code 启动并连上你配置的本地/远端 MCP Server 后：
1. 它并不会一口气把远端所有的 Tool 都注册给大模型（因为外部服务可能有上百个 API，全都塞进 Prompt 会导致 Token 爆炸）。
2. 相反，系统注册了一个内置工具：`ListMcpResourcesTool`。
3. 并在 System Prompt 里悄悄加上一句：“如果你找不到解决当前问题的内部工具，请尝试调用 `ListMcpResources` 看看有没有外部服务能帮你。”
4. 大模型调用了这个 List 工具后，才发现：“哦！原来这里还有一个 Jira 接口！”
5. 此时它才会进一步调用动态注册进来的特定 MCP Tool 去抓取工单信息。

**工程启示**：动态能力发现（Capability Discovery）。这是实现通用高阶智能体的终极方向。不预先加载所有能力，而是给模型一个“发现能力的工具”，让它自己去“翻抽屉找工具”，既解决了上下文溢出问题，又实现了系统能力的无限横向扩展。

---

## 最后的终章

通过这 51 万行代码的逆向旅程，这六个部分的剖析已经将 Claude Code 从皮到骨扒得干干净净。
它用极其雄厚的代码量，向整个业界证明了一个残酷的事实：**在 2025 年往后的 AI 应用开发中，调用大模型 API 的那几行代码早就不值钱了。**

真正的壁垒，在于：
- 你如何用极其复杂的本地逻辑（如 Bash AST）去填补大模型在执行力上的缺陷与危险；
- 你如何用极致的终端 UI 渲染引擎去抚平大模型等待时间所带来的用户焦虑；
- 你如何用层层叠叠的缓存与摘要算法（Microcompact/Autocompact）去对抗大模型的“失忆症”。

与其说 Claude Code 是一个 AI 产品，不如说它是一套“**如何驯服、压榨、保护并伪装大语言模型**”的工业级工程模板。对于每一个想在 AI Agent 赛道上创业的开发者来说，这套源码，就是一座金矿。