# 【核心大脑】代理循环与四层记忆压缩机制深度解析

在长达数小时的排错会话中，大模型可能会执行几十次系统命令、阅读上百个文件。如果将所有的历史信息原封不动地发给大模型，其上下文窗口（Context Window）会在极短时间内被撑爆。哪怕没有撑爆，也会因为填充了海量垃圾信息导致模型“失忆”或“幻觉”。

在源码的 src/query.ts 中，我们看到了堪称艺术级的上下文控制技术。系统设计了**四重防线**，通过剥离、剪枝、折叠和总结，在有限的脑容量里挤出了无限的续航能力。

---

## 代理主循环：queryLoop 引擎

一切的起跑线在于一个被称为 queryLoop 的异步生成器。这不是一个简单的“发请求 -> 收回复”函数，它是一个拥有极强控制欲的状态机。

**真实代码流程重构（基于 query.ts）：**
```typescript
while (true) { // 智能体死循环
    // 1. 获取基础消息队列
    let messagesForQuery = [...getMessagesAfterCompactBoundary(messages)];

    // 2. 第一道防线：大段工具返回结果强制截断
    messagesForQuery = await applyToolResultBudget(messagesForQuery);

    // 3. 第二道防线：思考气泡剪枝
    if (feature('HISTORY_SNIP')) {
        const snipResult = snipModule.snipCompactIfNeeded(messagesForQuery);
        messagesForQuery = snipResult.messages;
    }

    // 4. 第三道防线：微观塌缩（打扫无用工具记录）
    const microcompactResult = await deps.microcompact(messagesForQuery);
    messagesForQuery = microcompactResult.messages;

    // 5. 第四道防线：终极杀招，宏观总结
    const { compactionResult } = await deps.autocompact(messagesForQuery, ...);
    if (compactionResult) {
        messagesForQuery = buildPostCompactMessages(compactionResult);
    }

    // 发送给大模型，等待它返回工具调用或最终文本
    const 决策 = await 发送请求(messagesForQuery);
    if (决策是工具调用) {
        执行工具并记录();
        continue; // 带着新结果进入下一轮
    } else {
        break; // 任务完成
    }
}
```

---

## 防线 1：结果截断与隐藏缓存（Tool Result Budget）

这是防范大模型“急性撑死”的最前线。假设大模型执行了 cat package-lock.json，产生了 5 万行输出。

**底层逻辑 (applyToolResultBudget)：**
- **掐头去尾**：系统预先给每个工具设定了返回上限（例如 4000 字符）。一旦超出，系统会强行截断，保留开头的数百字（通常包含命令本身）和结尾的数百字（通常包含报错堆栈），中间的几万字直接被丢弃。
- **本地缓存兜底**：丢弃的内容并没有真正消失，系统会调用 recordContentReplacement 将完整日志写入本地的 ~/.claude/ 隐藏文件中。
- **引导式补丁**：系统会在发给大模型的信息中强行插入一句话，例如：
  > “中间的大量输出已被截断。如果以上内容不足以排查问题，请你调用 FileReadTool 去阅读本地缓存文件 /tmp/xxx.log 获取完整内容。”

**效果**：模型不会被垃圾信息淹没，但它依然拥有“去指定地点查阅完整监控”的自主权。

---

## 防线 2：思考气泡剪枝（Snip Compact）

为了让模型在解决复杂问题时更有条理，系统鼓励大模型在输出工具调用前，先输出一段 <thinking>我应该先搜索一下代码</thinking> 的心理活动。
但这在几十轮对话后，过去的心理活动只会白白浪费 Token。

**底层逻辑 (snipCompactIfNeeded)：**
剪枝算法会巡视历史消息队列。当它发现某些包含 thinking 标签的回复距离当前回合已经足够遥远（比如早于 10 个回合），它会像园丁剪除枯枝一样，将大模型曾经的心理活动暴力抹除，只保留它当年真正执行的那个工具记录。
*对于模型而言，它只需要知道自己“曾经调用过某个工具”，至于当时怎么想的，根本不重要。*

---

## 防线 3：微观塌缩与打扫战场（Microcompact）

大模型为了找一个 Bug，可能会连续用 10 次 grep 命令，翻阅 20 个文件的目录。当它终于找到目标并修改了代码后，前面那 10 次的搜索结果就全成了垃圾。

**底层逻辑 (microcompact)：**
该算法会分析历史记忆中的工具依赖树。如果它发现有一大段长文本的工具返回结果（如巨大的 ls 或 grep 输出），在随后的几十轮对话中**再也没有被提及或被其他动作强依赖**，它会判定这是一块“被废弃的战场”。

此时，系统会将这段几千字的冗长结果从数组中抽离，直接替换成一行极简的“墓碑标记”（Tombstone）：
> Context microcompacted (saved ~14,000 tokens, cleared 8 tool results)

**效果**：瞬间释放海量脑容量，让大模型的注意力死死盯在当前正在解决的问题上，杜绝了“聊久了就变蠢”的通病。

---

## 防线 4：启动子模型写回忆录（Autocompact）

如果前面三招都用尽了，面对那种聊了一整天、真的快要顶破 200k Token 上限的马拉松级对话，系统会触发最后的核武器。

**底层逻辑 (autocompact)：**
当检测到即将触及危险红线时，系统会在主循环外偷偷挂起，并在后台唤醒另一个价格便宜、速度极快的小模型（如 Claude 3.5 Haiku）。

系统会将最古老的那几十轮对话喂给这个小模型，并强制下达归纳任务：
> “请阅读以下冗长的人机对话，用精简的语言总结他们在这个项目里排查了什么 Bug，目前遇到了什么困难，并务必保留所有出现过的关键变量名、文件路径和系统状态。”

拿到这几百字的浓缩摘要后，主进程痛下杀手，将那几十轮原始对话硬生生从数组中切断（boundaryMessage 诞生），并在顶端塞入这段摘要。

**效果**：系统通过“宏观记忆总结”，抛弃了具体的对话细节，只保留核心的知识结晶。这赋予了该终端工具理论上“无限”的连续工作能力。