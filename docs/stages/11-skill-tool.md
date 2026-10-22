# Stage 11 · 模型自主调用技能（Skill Tool）

> 课程：CodeCrafters「Build your own Claude Code」· Skill 系列
> 实现仓库：`~/github/codecrafters-claude-code-typescript`
> 日期：2026-10-23

## 本关目标

用户用**自然语言**描述任务（不带 `/`）→ 模型对照 system prompt 里的技能描述，判断该用哪个 → **模型主动发 `Skill` 工具调用** → harness 把该技能 body 作为 **tool 结果**回传 → 模型按 body 指示作答。

```
用户："What is the database migration status?"
  ↓ 模型看到技能列表（Level 1 广告），匹配到 apple
模型发 tool_call: Skill({ name: "apple" })
  ↓ harness 查技能、返回 apple 的 body（tool 消息）
模型按 body 指示 → 输出 blueberry
```

## 工具声明

```jsonc
{
  "type": "function",
  "function": {
    "name": "Skill",
    "description": "Load a skill's instructions into the conversation",
    "parameters": {
      "type": "object",
      "required": ["name"],
      "properties": {
        "name": { "type": "string", "description": "The name of the skill to use" },
        "args": { "type": "string", "description": "Optional arguments for the skill" }
      }
    }
  }
}
```

## system prompt 追加引导句

```
If a skill matches the user's request, call the Skill tool with its name
and follow the instructions it returns.
```

**没有这句，模型可能读完描述就直接回答、不调用工具** → 测试挂。

## 执行 handler（骨架）

```ts
async function handleSkillToolCall(id: string, tool_call_function: any) {
  const { arguments: argumentsJson } = tool_call_function;
  let content: string;
  try {
    const { name, args } = JSON.parse(argumentsJson);
    const body = loadSkillPrompt(name);
    if (body === null) {
      content = `Error: unknown skill '${name}'`;        // 错误也是 observation
    } else {
      const argsText = typeof args === "string" ? args.trim() : "";
      const argv = argsText ? argsText.split(/\s+/) : [];
      content = substitute(body, argsText, argv);        // 复用 Stage 09 的替换
    }
  } catch (error) {
    const reason = error instanceof Error ? error.message : String(error);
    content = `Error loading skill: ${reason}`;
  }
  messages.push({ role: "tool", tool_call_id: id, content });
}

// 注册表加一行：
// Skill: handleSkillToolCall
```

**循环一行都不用改**——声明 + handler + 注册表，Stage 4/5 的架构直接接住新工具。

## 核心概念

### 1. 两条调用路径的对照

| | 用户显式调用（`/apple`） | **模型自主调用（Skill tool）** |
|---|---|---|
| 触发 | harness 解析字符串前缀 | 模型根据描述判断 |
| 决策者 | 用户（确定性） | **模型**（概率性） |
| body 的位置 | **替换** user 消息 | **作为 tool 结果追加** |
| 原请求 | 被替换丢弃 | **保留在对话里** |

这就是此前讨论的"**用户发起就替换、模型发起就追加**"——本关把两条路都补齐了。

### 2. 描述是模型的"检索索引"

Level 1 广告的职责升级：它不只是"告诉模型有什么"，而是**模型做选择的唯一依据**。

> 描述要写"**什么时候用**"（触发条件），不是"是什么"（功能列表）。
> "Database utilities" 对模型来说没有可匹配的信息。

测试里的 decoy 技能正是验证这点：描述不匹配 → 模型不应调用它；调用错了或全加载，输出会混入错误的 word。

### 3. 受控的 prompt 注入

模型通过 tool_call **请求**一段指令进入上下文，harness 决定是否给、给什么。和所有工具的处理完全同构：

```
声明 → 模型调用 → harness 执行 → tool 消息回传
```

## 坑清单

- **只加载被请求的那一个**：不要预加载所有 body（decoy 的 word 会混进输出）
- **未知技能别崩**：返回错误文本，模型自己处理；exit code 0 是测试项
- **`args` 可选**：缺省按空字符串处理（占位符替换为空）
- **tool 内容就是 body 本身**（带 args 时替换后），不要加工成摘要
- **引导句必须加**，否则模型可能不调用工具

## 测试

两个技能（一个描述匹配、一个是 decoy），发一个不点名技能的请求；验证输出是对应技能 body 里的 word，exit code 0。

## 实测结果（2026-10-23）

- Stage 通过 ✔（含前置关卡回归）
- 实现与设计骨架基本一致：工具声明 + handler + 注册表，循环未改动

## 系列小结

Skill 系列至此覆盖：

1. Level 1 广告（发现 + 描述进 system prompt）
2. 用户显式调用（换行替换）
3. 参数替换（`$ARGUMENTS` / `$n`）
4. 技能叠加（多技能共享参数）
5. **模型自主调用（Skill tool，描述即索引）**

渐进披露的完整生命周期（广告 → 按需加载 → 执行）已经跑通。
