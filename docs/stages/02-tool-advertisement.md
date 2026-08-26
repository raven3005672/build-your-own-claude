# Stage 02 · 声明 Read 工具（工具广告）

> 课程：CodeCrafters「Build your own Claude Code」· Stage AQ1
> 实现仓库：`~/github/codecrafters-claude-code-typescript`
> 日期：2026-08-27

## 本关目标

在 API 请求中声明 `Read` 工具（只声明，不执行）。改一个地方：`chat.completions.create({...})` 里加 `tools` 数组。

## 核心概念

### 1. 缸中之脑：模型默认没有任何环境访问权

LLM 默认碰不到文件系统、终端、网络——只能输出文本。给它能力分两步：

- **声明（本关）**：在请求里带上工具 spec，告诉模型"我有这个函数可用"
- **执行（Stage 3）**：模型发出调用请求后，harness 实际执行并回传结果

### 2. 话语权 vs 执行权 —— agent 安全模型的地基

模型永远只能**请求**调用工具（返回 `tool_calls`），真正执行的是 harness。harness 可以拒绝、检查参数、限制范围。**模型在请求侧，权限在执行侧。**

### 3. 工具 spec 四层结构

```js
{
  "type": "function",     // 判别字段（discriminator）：决定对象剩余部分的形状
  "function": {
    "name": "Read",       // 契约标识符：模型点名用它，Stage 3 代码按它分发
    "description": "...", // 写给 LLM 的"使用说明书"——提示词工程的子集
    "parameters": { ... } // JSON Schema：参数契约（生成指南 + 解析依据）
  }
}
```

- `type` 是可辨识联合的判别字段，当前 OpenAI 兼容协议只有 `"function"` 一种取值；`function` 包装是 function-calling beta 的历史包袱
- Anthropic Messages API 同物平铺：`{ name, description, input_schema }` —— 协议形状不同，思想相同
- `name` 三条约束：① 协议约束（字母/数字/下划线/短横线，≤64 字符）② **Stage 3 代码匹配（最重要，改两边）** ③ 语义影响模型使用准确率（叫 `Tool1` 模型就想不起何时用它）

### 4. 完整调用闭环（Stage 3 预习）

请求（带 tools + 用户消息）→ 模型响应：

```json
{
  "choices": [{
    "message": {
      "role": "assistant",
      "content": null,
      "tool_calls": [{
        "id": "call_abc123",
        "type": "function",
        "function": { "name": "Read", "arguments": "{\"file_path\": \"app/main.ts\"}" }
      }]
    },
    "finish_reason": "tool_calls"
  }]
}
```

坑位预告：
- **`arguments` 是 JSON 字符串**，要 `JSON.parse()` 解开
- `tool_call_id` 是调用身份证，回传结果时配对用：`{ role: "tool", tool_call_id, content }`

## 调试实录：神秘的"0"

Stage 1→2 之间遇到的失败，最后定位为**非 bug**：

```
[tester::#AQ1] Running tests for Stage #AQ1 (Advertise the read tool)
$ ./your_program.sh -p 'Count the number of tools available... Respond with only a number.'
[your_program] 0
[tester::#AQ1] Expected value to be at least 1, got 0
```

复盘要点：

1. **仔细读测试输出第一行的 stage 编号**——这已经是 AQ1（Stage 2）的测试，不是 Stage 1
2. `[your_program] Logs from your program will appear here!` 出现 = 程序完整跑完、API 调用成功 → 排除"程序挂了 stdout 为空"假设
3. 模型答 0 是**诚实**的：程序一个工具都没声明，测试 prompt 问"你有几个工具可用"，正确答案就是 0
4. 课程测试是**端到端 + LLM-as-sensor**：不检查代码结构，只问模型"数数"。声明 1 个工具 → 模型答 1 → 过；声明 3 个 → 答 3 → 也过
5. 该测试风格启发：LLM 本身成了被测系统的"观测器"，测试断言的是模型对环境的如实汇报

## 实测结果（2026-08-27）

- Stage #AQ1：模型答 `1` ✔（Value is at least 1）
- Stage #YY2 回归：`3+7` 答 `10` ✔
- 顺带揭晓 Stage 1 测试真面目：一道 `3+7` 算术题，验证程序能与 LLM 正常对话

## Q&A 记录

1. **type 影响下面结构吗？** 是。type 是判别字段，决定对象剩余部分形状（可辨识联合）。当前协议只有 function 一种取值，所以像固定搭配。
2. **name 可以随便写吗？** 对测试随便；工程上有协议约束、Stage 3 匹配契约、模型行为语义三条约束。
3. **parameters 在实际调用时长什么样？** 见上方"完整调用闭环"三幕戏：模型按 schema 产出 arguments（JSON 字符串），harness 按 schema 解析执行，结果用 tool_call_id 配对回传。

## 与生产 harness 的对照

- 每个工具的 spec 都是这样一份 JSON Schema。Claude Code 的 Read/Write/Bash/Grep 工具集 = 一批这样的声明
- dsh `packages/core/` 的 tools 模块：生产 harness 把工具声明、校验、执行组织成正式模块
- 工具描述（description）的写作质量 = 工具是否被正确使用的第一决定因素，属于提示词工程的范畴
