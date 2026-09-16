# Stage 04 · 实现 Agent Loop（harness 的心脏）

> 课程：CodeCrafters「Build your own Claude Code」· Stage FF2
> 实现仓库：`~/github/codecrafters-claude-code-typescript`
> 日期：2026-09-16（讨论跨 09-10 ~ 09-16）

## 本关目标

把 Stage 3 的"单次 tool 执行"升级成循环：**只要模型还在发 tool_calls，harness 就执行 → 回传 → 再问，直到它不再要工具**。这一步之后，程序从"能执行一次工具的脚本"变成真正的 agent——从 Claude Code 到 Codex，所有生产 harness 的内核都是这个循环。

## 代码骨架（最终版）

```ts
const messages = [{ role: "user", content: prompt }];

while (true) {
  const response = await client.chat.completions.create({
    model: "anthropic/claude-haiku-4.5",
    messages,
    tools: [/* Read 工具声明（同 Stage 2） */],
  });

  const message = response.choices[0].message;
  messages.push(message);                  // assistant 消息先于 tool 结果进历史

  if (!message.tool_calls?.length) {       // 正常出口：模型答完了
    console.log(message.content);
    break;
  }

  for (const tool_call of message.tool_calls) {   // 遍历全部，不再是 [0]
    const { id, function: fn } = tool_call;
    let content: string;
    try {
      const { file_path } = JSON.parse(fn.arguments);  // 参数解析也属于工具执行
      content = await readFile(file_path, "utf-8");
    } catch (error) {
      const reason = error instanceof Error ? error.message : String(error);
      content = `Error reading file: ${reason}`;
    }
    messages.push({ role: "tool", tool_call_id: id, content });  // 成败唯一出口
  }
}
```

结构要点：guard 提前退出（`if/else` 包裹执行体的写法淘汰）；try/catch 在**单次调用内**（一条消息带 N 个调用，第 1 个失败不连累其他）；**push 在 try/catch 之外**——"每个 id 恰好一条回复"这个协议不变式从纪律变成结构保证。

## 核心概念

### 1. 循环的诞生：终止权在模型手里

`if` → `while` 的本质变化：**循环转几圈不再由 harness 决定，由模型决定**——它不再发 tool_calls 的那一刻，循环自然结束。harness 只负责忠实转圈。

while 条件的三写法（讨论过）：

- **`while(true)` + 多条具名 break** ← 生产形态。退出原因是一张清单：正常完成 / 最大轮数 / 用户取消 / 致命错误。而"条件依赖循环体内产生的数据"（第一次请求还没发，无从判断要不要继续），所以条件表达式根本写不下
- `do...while` 能把条件提回头部，但只能表达一个退出原因，生产中少见
- 硬塞进 `while(...)` 的多条件会拧成没人看得懂的长布尔式

### 2. 判数据不判标志位

终止检查用 `message.tool_calls?.length`，**不看** `finish_reason`——第三方 OpenAI 兼容端点对 finish_reason 的实现不统一（该给 `"tool_calls"` 却给 `"stop"`）。但 `finish_reason === "length"`（被 max_tokens 截断）值得单独留意：那时 content 是半截话，直接打印 = 静默给出残缺答案。

### 3. 错误处理：错误是一次必须回复的 observation

**协议层核心事实**：模型声明的每个 tool_call 必须收到一条配对的 tool 消息（一对一欠条），缺一条下次请求 400。所以 catch 的目的不是"别崩"，而是**把错误翻译成模型能读的文本，走正常的回传通道**——错误路径与成功路径共用同一个信封：

```jsonc
// 成功
{ "role": "tool", "tool_call_id": "call_abc", "content": "file contents..." }
// 失败：同样的信封，内容换成错误描述
{ "role": "tool", "tool_call_id": "call_abc", "content": "Error reading file: ENOENT..." }
```

Anthropic 协议把这件事显式建模为 `tool_result` 块上的 `is_error: true`（Stage 3 说的 "is_error 无对应物" 就是这个异构角落）；OpenAI 兼容协议用文本约定代替（`Error: ` 前缀）。

**错误文本是"写给模型看"的**（它是模型决策下一步的输入）：

- ✅ 三要素：哪个操作、哪个目标、为什么失败 → 模型能换路径、换假设、或如实汇报
- ❌ stack trace（噪声 token + 分散注意力）、敏感信息
- ❌ **绝不能返回空字符串**——模型会当作"文件读到了，是空的"；把失败伪装成成功比崩溃更危险

**错误分类**（决定回传还是自己处理）：

| 类型 | 例子 | 处理 |
|---|---|---|
| 工具错误（模型能应对） | 文件不存在、参数 JSON 坏了、权限不够 | catch → 回传 |
| harness 错误（模型救不了） | API 401、网络断、限流 | 重试（瞬态）或中止（持久） |

生产上还细分三类：模型造成的（参数错）/ 永久性的（ENOENT）/ 瞬态的（EAGAIN）——前两类直接回传，瞬态的先自己重试几次。

**自愈**：回传错误后模型看到"我试了 → 失败了 → 原因是 X"，能换路径重试或如实汇报——"错误驱动重试"落地。生产对照：Claude Code 里用户按 Esc 拒绝工具调用，模型收到的是 `is_error` 的 tool_result——**一切结果皆 observation**。

### 4. 无状态性第三次登场：只增不减的消息数组

`[user, a1, t1, a2, t2, …, 终答]`——每轮把**全部历史**重新发一遍。这一次认识它的账单（下面 A 节），并学会修剪它。

## 深度讨论 A：成本与上下文（循环的物理极限）

### A1. 二次方的账单

每轮重发全部历史：累计输入 ≈ s·N²/2，**轮数翻倍、花销约翻四倍**。真正的成本大头是 **tool 结果**——读过的文件内容进入历史后，每一轮都跟着重发。

### A2. Prompt Caching——同一份前缀别重算

模型每读一个 token 都会算出注意力的 K/V 中间状态，生成下一个 token 需要前面所有 token 的 K/V。缓存 = "请求结束别扔这套状态，留 5 分钟"；下次同前缀进来，跳过 prefill（最贵的一步）从新内容接着算。

- 请求里挂 `cache_control` 标记（打在"要缓存到的块末尾"），标记之前的全部内容成为可复用前缀
- **前缀匹配**：任何一处字节变化，其后全部失效（像 git 哈希——改一个字节，后面全是新内容）
- 写缓存 ≈1.25×（存有成本），读缓存 ≈1/10 或更低（新模型更便宜）
- **不省网络流量**：整包照发，省的是服务端计算 + 首 token 延迟
- **标记不是给模型的**：它是给基础设施的指令（"存到这里"）；真正的 key 是内容字节本身。它在服务端被消费，永远不会进入模型输入——判定一个字段去不去模型的标准：**模型的决策需不需要知道它**（tools 需要 → 序列化进输入；cache_control 不需要）
- **静默失效器黑名单**：system prompt 里塞 `datetime.now()`、工具数组顺序不稳、JSON 字段顺序不稳 → 命中率归零且不报错。验证：响应 `usage.cache_read_input_tokens` 恒为 0 = 有东西在破坏
- 前缀短于 512–4096 token（视模型）静默不缓存

### A3. Compaction——给历史写会议纪要

窗口是硬墙，缓存救不了。做法：**早期历史 → 模型摘要 → 替换原文**。代码形态：

```ts
if (estimateTokens(messages) > THRESHOLD) {
  const summary = await summarize(client, messages.slice(0, -KEEP_RECENT));
  messages = [
    { role: "user", content: `[此前对话摘要]\n${summary}` },
    ...messages.slice(-KEEP_RECENT),
  ];
}
```

- 摘要本身是一次普通 API 调用；token 数从上一轮 `usage.prompt_tokens` 或 count_tokens 端点来
- 服务端版本（beta）：API 自动做，唯一代码变化是**回传 `response.content` 整块**，不能只取文本（压缩状态藏在块结构里，只 append 文本会静默丢失）
- 我们这场对话亲历过一次——`/compact` 的产物就是对话开头的 Summary

### A4. Context Editing——直接删

旧 tool 结果（体积大头）不再需要时，**清内容留骨架**：`m.content = "[cleared]"`。不能整条删消息——assistant 里的 `tool_call_id` 会失去配对 → 400。和 compaction 的区别：**清除 vs 压缩**；删除零成本、确定性强，摘要花钱但保留更多信息。

### A5. 三者的关系

| 方法 | 一句话 | 谁执行 | 网络流量 | 服务端计算 | 窗口 |
|---|---|---|---|---|---|
| Caching | 同一份前缀别重算 | 服务商（你标记） | ❌ 不变 | ✅ 复用状态 | ❌ 不变 |
| Compaction | 旧历史摘要化 | harness 或 API | ✅ 少发 | ✅ 少算 | ✅ 释放 |
| Context editing | 旧工具结果清空 | harness 或 API | ✅ 少发 | ✅ 少算 | ✅ 释放 |

**互相拉扯**：caching 想要前缀稳定，压缩想要前缀变短——改写历史 = 缓存全部失效。生产节奏："大多数时间吃缓存，偶尔付一次全价做压缩"。一句话记法：**caching 打钱和延迟，另外两个打窗口容量（顺带省钱省流量）；前者是优化，后者是续命**。

### A6. Token 意识（认知必备，实现不需要）

- 数 token 用**服务商自己的计数器**（Anthropic `count_tokens` 端点、OpenAI 系 `tiktoken`），别混用——不同 tokenizer 切法不同，同一段文本新老模型能差 30%+，估算成本会离谱
- 心算量级：英文 ≈4 字符/token，中文 ≈1 字/token 上下；精确值永远看计数器
- 工程影响：缩进空白也烧钱；UUID/hash/base64 这类无规律串切分极差——这就是"工具输出要在源头截断"的量化依据
- **tokenizer 本身不实现**——它和权重一样住在分界线下方（见 C 节）

## 深度讨论 B：终止权归谁

两个不同的问题：**"任务完成了吗"（语义问题）归模型；"允许继续吗"（政策问题）归 harness。**

- 为什么不能硬编码"最多 3 次"：任务深度先验不可知（读 README 是 2 次调用，重构模块是 50 次）——想定这个数就得理解任务，等于替模型干活
- 模型侧的"停"是训练出来的行为，两种失败模式都存在：停太早（没验证就说完成）/ 停不下来（死循环）
- harness 侧的围栏：最大轮数（对模型不可见的硬闸，即代码里那个 `MAX_TURNS`）、用户取消（abort 进循环条件）、超时
- **预算的两形态**（2026 API 现状）：**可见预算**（task budget beta——服务端注入模型能看到的倒计时标记，它自己省着花、优雅收尾）vs **不可见硬顶**（session budget、max_turns）——想让它配合就给它看，要保证拦得住就不能放在它手里
- **统一哲学**：模型决定意图，harness 决定边界——与"想读什么/允许读什么"（权限）、"想跑什么/允许跑什么"（沙箱）完全同构。围栏必须住在被约束对象之外：不能让模型自己执行预算（注入一句"忽略预算"就绕过），就像沙箱必须在核心里而不是模型里

## 深度讨论 C：分界线与职业定位（09-16）

**模型的输入三层**：

```
messages JSON（有结构：role / tool_calls / 参数）
  ↓ 服务栈套 chat template，拍平成一个序列
[特殊token]system…[特殊token]user…[特殊token]assistant…
  ↓ tokenizer
[ 17, 382, 9021, … ]        ← 物理意义上的模型输入就是这行数字
  ↓ 模型内部（embedding → 注意力 → …）
下一个 token 的概率分布
```

线上方的所有"结构"（role 之分、工具定义、id 配对）在模型眼里只是**文本约定 + 特殊 token**——模型是训练出来认得这些标记的。这是 prompt injection 结构上无法根治的最底层原因。工具定义是**真进去的**（序列化后占 token，所以它写多长直接影响成本与行为）。

**分界线 = token ID 那一行。职责地图**：

| | 线之上（组装输入） | 线之下（消费 token） |
|---|---|---|
| 干什么 | 消息/工具/上下文/重试/权限/UX | KV 缓存调度、批处理、量化、GPU、训练 |
| 职能 | **AI 应用 / Agent 工程师** | 推理平台工程师 / 模型研究工程师 |
| 技能栈 | TS/Python、API 集成、状态管理、评测 | CUDA、分布式系统、编译器、数学 |
| 离前端的距离 | **前端经验的延长线** | 几乎零重合 |

**API 网关 = AI 系统里的"前后端接口"**：前端工程师的迁移路径就在线之上——用 HTTP API 消费后端的经验，平移到用 LLM API 消费"推理"。链路全景：

```
本地 harness → API 网关（鉴权/限流/路由/计费）
             → 推理引擎（批处理 / KV 缓存 / 采样）
             → GPU 上的模型权重（只看到 token）
```

课程链路多一跳翻译：`main.ts → OpenRouter 网关 → Anthropic 服务栈`——模型名里的 `anthropic/` 前缀就是给 OpenRouter 的路由信息，**协议翻译发生在网关层**，这正是 Stage 3"同构"的工程化。模型对整条链路无感知。

## 调试实录：FF2 首次失败（输出是输入的指纹）

**现象**：期望 `"12"`，实际得到 "I'd be happy to summarize the README for you, but I need to know the file path…"

**根因**：`messages` 的 user 消息被**硬编码**成 `"Summarize the README for me."`，没有用命令行传入的 `prompt`——模型回答的根本是另一句话。

**证据法（新技巧）**：**LLM 程序的输出是输入的指纹**——回复文本对应的是 "summarize the README"，而测试问的是 "chemical expiry period"，两者对不上 → 实际发送的输入 ≠ 你以为的输入。回复与预期不符时，先核对该"实际发出去的"，别先怀疑模型。

**随后两轮 review 的修复**：`tool_calls[0]` → 遍历全部；`JSON.parse` 从 try 外挪进 try 内；两个分支各自 push → 单一出口；`error.message` → `instanceof Error` 窄化（catch 的 error 在严格 TS 下是 `unknown`）；清理全部注释死代码。

## 实测结果（2026-09-16）

- Stage #FF2 通过：**两跳依赖链**——模型先读 README.md（得知 "app/chemical.py contains chemical properties"），再读 app/chemical.py（`chemical_expiry_period = 12`），第三轮答出 "12"。**依赖链长度 = API 往返次数**（Stage 3 讨论的落地）
- 回归 YY2 / AQ1 / MD6 通过
- 之后追加了错误处理路径（try/catch 改写主循环），**待 submit 复验回归**

## Q&A 记录（本关讨论）

1. **错误回传的结构怎么写？** 信封与成功完全一致，content 换成错误文本（Anthropic 另有 `is_error` 标志）；catch 放单次调用内；push 提到 try/catch 外（单一出口）
2. **模型靠 id 匹配、content 识别结果？** 精分：**id = 配对号**（结构层，API 强制校验完整性，缺一条 400）；**content = 载荷**（语义层，模型读它判断结果）。多文件并读时，模型靠"id + 前一条 assistant 消息"区分哪段内容属于哪个文件。协议只保证配对**完整**，不保证配对**正确**——错挂 API 不拦，正确性责任在 harness（id 和 content 必须在同一次执行里成对产生）
3. **正常的 while 条件怎么写？** `while(true)` + 多条具名 break 是生产形态；`do-while` 是"条件进头部"的教科书版；判 `tool_calls` 数组不判 `finish_reason`
4. **caching 不省流量，只是加标识符？** 前半对（字节一个不少）；后半修正：不是给模型的标识符——是给基础设施的指令，真正的 key 是内容字节
5. **为什么 `cache_control` 会被网关消费？** 它不是内容而是"怎么算"的指令；模型是纯文本进出的机器，只接受 token。判定标准：模型的决策需不需要知道它
6. **链路是 harness → 网关 → model？** 对，但末端更精确：网关 → 推理引擎（KV/批处理/采样）→ GPU 上的权重；"模型"是权重不是进程，链路对模型不可见
7. **模型输入到底是什么？谁负责？** token ID 序列；线之上（组装）是应用工程师，线之下（消费）是推理平台/训练工程师——前端经验的迁移路径在线之上
8. **tokenizer 要掌握吗？** 不实现（住在分界线下方），但"token 意识"是应用侧素养——数预算、估成本、裁输出全靠它
9. **成本三方法在代码上各是什么？** caching = 请求加标记（内容不变）；compaction = 循环里判断阈值 → 摘要调用 → 重写 messages 数组；context editing = 遍历替换旧 tool content 为占位符 / 或加一个请求参数

## 与生产 harness 对照（2026 现状）

- Claude Code：自动压缩 + `/compact`；压缩后接受一次缓存清零，换长期窗口健康
- Anthropic API：`cache_control` 显式缓存（5 分钟 TTL、命中看 `usage.cache_read_input_tokens`）；服务端 compaction（beta，默认约 150K 触发）；context editing（beta，清旧 tool 结果）；task budgets（beta，模型可见的 token 预算）——**这三个方法已从"harness 自己实现"下沉为"API 提供能力"**
- 工具结果在生产 harness 里被标记为 untrusted（Stage 3 的注入防御）
- 参考：<https://docs.claude.com/en/docs/build-with-claude/prompt-caching>、`/context-editing`、`/compaction`、<https://code.claude.com/docs>

## Stage 5 预习（oz7 · write 工具）

- 第二个工具：写文件——**第一个有副作用的工具**（read 的最坏情况是"看到了不该看的"，write 是"改坏了东西"，安全边界完全不同）
- 多工具协作：工具列表变长，模型自己选；参数校验（写哪个路径？写什么内容？）
- 权限、确认、路径限制——Stage 3 讨论的安全边界在这里第一次真正兑现

## 待办

- [ ] submit 复验回归（错误处理路径改动了主循环）
- [ ] `newMessage` 变量可内联为 `messages.push({ role: "tool", tool_call_id: id, content })`——将来给 messages 标 `ChatCompletionMessageParam[]` 时，内联对象字面量能推导出 `role: "tool"` 字面量类型，const 中转会宽化成 `string` 而报错
- [ ] 安装 bun + `bunx tsc --noEmit`（messages 的类型标注欠着）
- [ ] 成本感知第一步：打印 `response.usage.prompt_tokens`，亲眼看"每轮全量重发"的账单
