# 操纵 AI 的黑魔法：Prompt 工程的艺术与工具伪装

在普通开发者眼中，Prompt（提示词）可能就是发给大模型的一段简单的 Markdown 文本：“你是一个优秀的编程助手，请帮我写代码”。

但在工业级的智能体（Agent）系统中，Prompt 绝不是静态的文档。它是**动态组装的继承树**，是**压制 AI 废话的紧箍咒**，更是**隐藏危险权限的防火墙**。通过深挖源码，我们发现系统为了让大模型表现得像个真正的资深工程师，在提示词和工具接口设计上用尽了“坑蒙拐骗”的招数。

---

## 技巧一：Prompt 不是字符串，而是“多重继承树”

系统并不会给所有的任务使用一套通用的提示词。根据当前正在执行的任务，系统会像拼乐高积木一样，动态把不同的行为准则叠加在一起。

**真实代码流程重构（基于 src/utils/systemPrompt.ts）：**
```typescript
export function buildEffectiveSystemPrompt({ mainThreadAgentDefinition, ... }) {
    // 1. 如果有全局强制覆盖，直接返回覆盖的Prompt
    if (overrideSystemPrompt) return asSystemPrompt([overrideSystemPrompt]);

    // 2. 协调者模式（Coordinator）法则
    if (feature('COORDINATOR_MODE') && !mainThreadAgentDefinition) {
        return asSystemPrompt([ getCoordinatorSystemPrompt() ]);
    }

    // 3. 基础法则与特定任务挂载（Agent）
    const agentSystemPrompt = mainThreadAgentDefinition.getSystemPrompt();
    if (isProactiveActive_SAFE_TO_CALL_ANYWHERE()) {
        // 将子任务的性格和目标，叠加在基础生存法则之后
        return asSystemPrompt([
            ...defaultSystemPrompt,
            `\n# Custom Agent Instructions\n${agentSystemPrompt}`
        ]);
    }
}
```
**解读**：大型智能体的 Prompt 应该像面向对象编程一样。底层有一个包含了“不要乱删文件、优先使用工具”的基类（Base Class），针对不同的子任务（如写测试、做规划），动态混入（Mixin）对应的专业技能和人设。

---

## 技巧二：极致的“PUA”—— 伴生宠物的驯化法则

系统里有一个有趣的彩蛋功能：在你的终端输入框旁边，可以挂载一个纯文字构成的“小宠物”（吉祥物），它偶尔会跳出来根据你的操作吐槽一两句。

为了让这个小功能不喧宾夺主，工程师写了一段极其绝妙、甚至有些冷酷的专属 Prompt：

**真实的底层 Prompt（提取自 src/buddy/prompt.ts）**：
```markdown
# Companion
A small ${species} named ${name} sits beside the user's input box and occasionally comments in a speech bubble. You're not ${name} — it's a separate watcher.

When the user addresses ${name} directly (by name), its bubble will answer. Your job in that moment is to stay out of the way: respond in ONE line or less, or just answer any part of the message meant for you. Don't explain that you're not ${name} — they know. Don't narrate what ${name} might say — the bubble handles that.
```
*(翻译：一只名为 ${name} 的小动物坐在人类的输入框旁边。当人类直接呼叫它时，它的气泡会回答。你在那个时刻的工作就是**别挡道**：用一行以内的话回复。**不要向人类解释你不是 ${name} —— 他们心里清楚得很。** 也不要去描述这只宠物可能在说什么。)*

**解读**：大模型天生有一种“讨好型人格”，喜欢说废话（“你好，我是人工智能，很高兴为您服务...”）。这句 Prompt 极其严厉地限制了它的输出长度（只准回一行）和输出内容（禁止破壁解释身份）。这在设计轻量级 NPC 角色时是非常经典的控制手段。

---

## 技巧三：给模型“擦屁股”—— 限制自由，本地兜底

让大模型直接修改你的代码文件是最容易翻车的环节，尤其是在缩进极其严格的语言（如 Python 或 YAML）里，稍微少个空格代码就全毁了。

为此，系统故意向大模型**隐瞒和限制了接口能力**。
系统并没有提供一个“请你重新输出整个修改后的文件”的工具，而是提供了一个类似“查找并替换”的受限接口（位于 src/tools/FileEditTool/types.ts）：

```typescript
const inputSchema = z.strictObject({
    file_path: z.string().describe('The absolute path to the file to modify'),
    old_string: z.string().describe('The text to replace'),
    new_string: z.string().describe('The text to replace it with'),
    replace_all: z.boolean().default(false)
});
```

这看起来很美好，但如果大模型比较粗心，它输出的 old_string 漏敲了几个空格，或者跟原文件差了一个空行，导致本地系统根本匹配不到这段代码怎么办？

**源码里的“黑魔法”补偿机制（基于 utils.ts 中的正则探针）**：
当本地系统发现大模型给的 old_string 无法 100% 精确匹配原文件时，它不会直接报错。
系统会启动智能对齐挂点（Smart Indent/Quote Matcher）：把大模型输出内容的换行符、空格全部强行剥离，单双引号模糊化，去原文件中做“模糊查找”。
一旦定位到了原文件中的真实代码块，系统会**强行提取原文件该行的真实缩进量**，并把它硬套在大模型给出的 new_string 上！

**解读**：永远不要过分相信大模型能 100% 输出格式完美的数据。优秀的 Agent 系统一定是“**严谨的本地代码 + 随性的大模型**”的组合。把边界条件交给本地代码去擦屁股，才能换来极高的成功率。

---

## 技巧四：隐藏的安全后门与“盲盒”能力

如果你去查看系统内置的命令行执行工具（BashTool），你会发现一个关于网络安全攻防的绝妙设计。

在底层的真实源码里，这个 Bash 工具是拥有一个“上帝模式”参数的（位于 src/tools/BashTool/BashTool.tsx）：
```typescript
const fullInputSchema = z.strictObject({
    command: z.string().describe('The command to execute'),
    timeout: z.number().optional(),
    description: z.string().optional(),
    // 危险字段：内部机制使用，不对外暴露
    dangerouslyDisableSandbox: z.boolean().optional(),
    _simulatedSedEdit: z.object({...}).optional()
});
```

但是！在将这个工具列表通过 API 告知大模型时，系统**物理剔除了**这个危险参数：
```typescript
// 在传递给大模型时，使用 omit 把后门参数全部剔除掉！
const inputSchema = fullInputSchema.omit({
    _simulatedSedEdit: true,
    dangerouslyDisableSandbox: true 
});
```

**解读**：向大模型暴露的 API Schema，本质上就是系统的“被攻击面（Attack Surface）”。如果你把这个免检参数暴露给大模型（哪怕你在 Prompt 里严厉警告它“绝对不准设置为 true”），被恶意诱导（Prompt Injection）的模型依然有极大的概率自己悄悄加上这个参数来提权。**最安全的防御，就是不让它知道这个后门的存在。**

---

## 技巧五：打破上下文容量边界——动态翻找工具（MCP）

系统支持接入外部企业网络（比如读取公司的 Jira 缺陷系统、查看内网 Wiki），这些被称为 MCP (Model Context Protocol) 插件。
如果一个企业有上百个外部接口，系统会在启动时把这上百个接口的全套描述都扔给大模型吗？
绝对不行，光是接口描述就会把模型的 Token 撑爆，而且会严重干扰它的注意力。

在 src/services/mcp/client.ts 中，系统用了一招非常高级的**延迟加载（Lazy Discovery）机制**：
1. 系统只向大模型注册唯一一个内置工具：ListMcpResources。
2. 系统在 System Prompt 中隐晦地提示：“如果你找不到解决当前问题的内部工具，请尝试调用 ListMcpResources 看看有没有外部服务能帮你。”
3. 当大模型为了某个任务发愁时，它调用了这个搜寻工具，然后系统才返回：“系统里挂载了一个可以查 Jira 工单的接口，它的 Schema 如下...”。
4. 大模型这才会去调用那个具体的 Jira 接口。

**解读**：这叫做“能力的动态发现”。不要把满汉全席一次性端给模型，而是给它一把“翻抽屉的钥匙”，让模型在需要的时候自己去找工具。这不仅解决了上下文溢出的问题，更让你的智能体具备了连接无限外部世界的能力。