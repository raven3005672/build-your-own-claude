# Stage 12 · 子代理执行技能（`context: fork`）

> 课程：CodeCrafters「Build your own Claude Code」· Skill 系列
> 实现仓库：`~/github/codecrafters-claude-code-typescript`
> 日期：2026-10-23

## 本关目标

frontmatter 里声明 `context: fork` 的技能**不在主对话里展开**，而是开一个独立的对话栈，把技能 body 当作唯一输入，跑同一个 agent loop，只把最终答案带回主对话。

```markdown
---
name: apple
description: Use this skill when the user asks about the on-call rotation.
context: fork
---

Respond with exactly one word: blueberry
```

## 完整流程

```
主对话: [system(技能目录), user("Who is on the on-call rotation?")]
  ↓ 模型匹配描述 → tool_call Skill({ name: "apple" })
Skill handler:
  ├─ 读 frontmatter → context == "fork"
  ├─ 子对话 messages = [ user("Respond with exactly one word: blueberry") ]   ← 只有 body
  ├─ runAgentLoop(子对话) → "blueberry"
  └─ tool 结果 = "Skill apple ran in a separate context and returned: blueberry"
  ↓ 主对话继续 loop，模型组织最终答案
输出: The on-call engineer is blueberry.
```

## 代码骨架（最终版）

```ts
// 同一个 loop，供主对话与子代理复用
async function runAgentLoop(client: OpenAI, messages: any[]): Promise<string> {
  while (true) {
    const response = await client.chat.completions.create({ model, messages, tools });
    const message = response.choices[0].message;
    messages.push(message);

    if (!message.tool_calls?.length) {
      return message.content ?? "";          // ★ 只返回，不打印
    }
    // 工具分发（handlers 定义在本函数内，闭包住 messages）
  }
}
```

```ts
// Skill handler 的 fork 分支
const skill = loadSkill(name);               // loadSkill 返回 { fields, body }
if (skill === null) {
  content = `Error: unknown skill '${name}'`;
} else {
  const body = substitute(skill.body, argsText, argv);
  if (skill.fields.context === "fork") {
    const subMessages = [{ role: "user", content: body }];   // 只有 body
    const answer = await runAgentLoop(client, subMessages);  // 同一个 loop
    content = `Skill ${name} ran in a separate context and returned: ${answer}`;
  } else {
    content = body;                                          // 旧行为
  }
}
```

## 核心概念

### 1. 子代理 = 同一个 loop，换一份 messages

> Steps 3 and 5 are the same loop. A subagent is that loop called again with a different list of messages.

本关的主要工作量是**重构**：把 while 循环从 `main` 提取成 `runAgentLoop(client, messages)`。主对话和子代理都是它的调用者。

**函数式视角：agent loop = f(messages)。** 子代理不是新机制，是递归调用。

### 2. 上下文隔离：fork 的意义与代价

子对话从空开始，只收到 body：

- **不含触发它的原问题**（tester 会监听请求验证）
- **不含主对话的 system prompt（技能目录）**——body 必须自包含
- 子代理的中间过程（工具调用、读到的文件）不回灌主对话，**只有最终答案回来**

这是 Stage 4 上下文管理主题的又一层：用一次隔离的对话换取"只回传摘要"。代价是子代理不知道用户原始问题。

### 3. Skill handler 的分支

以前 handler 对所有技能都返回 body；本关多一个分支：

- `context === "fork"` → 子对话 → `runAgentLoop` → 包装后的答案
- 否则 → 返回 body（旧行为）

为此 `loadSkill` 必须返回 `{ fields, body }`（只返回 body 读不到 `context`）。

### 4. fork 终止叠加

spec 的 Notes：fork 技能会结束一次 stacking 展开，永远不与其他技能并排。实现要点在 `parseSlashCommand`：**先消费掉 fork 技能自己的 token，再 break**——否则 `/apple` 会漏进 `argsText`。

## 调试实录：提取循环时把"返回"写成了"消费"

重构后 loop 里保留了旧的：

```ts
console.log(message.content);
break;                        // 函数隐式返回 undefined
```

两个后果：

1. 主对话先打印答案、`main` 又打印 `undefined` → 输出多一行，测试精确比对失败
2. 子代理的 `answer` 是 `undefined` → tool 结果变成 `"…returned: undefined"`，答案丢失

修复：`return message.content ?? "";`——**打印留给调用方**。

**元教训**：把循环/流程提炼成可复用函数时，结果要 `return`，I/O 留在调用方。跨上下文调用时更致命——**返回值是唯一的通道**，`console.log` 里的东西对面拿不到。

## 测试

- tester 监听程序发出的请求：其中一个请求必须包含技能指令、**不含触发问题**
- 程序输出必须包含子代理给出的答案
- exit code 0

## 实测结果（2026-10-23）

- Stage 通过 ✔
- **课程现有内容全部完成（12 关）**

## Skill 系列小结

1. Level 1：技能发现 + 描述广告（渐进披露）
2. Level 2：用户显式调用（body 替换）
3. 参数替换（`$ARGUMENTS` / `$n`，单遍正则）
4. 技能叠加（连续消费 token，共享参数）
5. 模型自主调用（Skill 工具，描述即检索索引）
6. **子代理（context: fork，同一 loop 换 messages）**

渐进披露的完整生命周期（广告 → 按需加载 → 执行 → 隔离执行）已跑通。
