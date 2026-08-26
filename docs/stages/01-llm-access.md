# Stage 01 · 与 LLM 通信（接入层）

> 课程：CodeCrafters「Build your own Claude Code」· TypeScript 实现
> 实现仓库：`~/github/codecrafters-claude-code-typescript`
> 日期：2026-08-26

## 本关目标

让程序能调用远程 LLM 并把回复打印到 stdout。这是 harness 最底层的一层：**LLM 接入层**。没有工具、没有循环、没有记忆——只有一次"发一句话、收一段回复"。

## 代码骨架

`app/main.ts` 共 35 行，逐段拆解：

```ts
const [, , flag, prompt] = process.argv;     // CLI 输入：./your_program.sh -p "hello"
const apiKey = process.env.OPENROUTER_API_KEY;          // 凭据，绝不硬编码进 git
const baseURL = process.env.OPENROUTER_BASE_URL ?? "https://openrouter.ai/api/v1";

const client = new OpenAI({ apiKey, baseURL });         // 无状态连接配置，不是 session

const response = await client.chat.completions.create({ // 一次 HTTP POST /chat/completions
  model: "anthropic/claude-haiku-4.5",
  messages: [{ role: "user", content: prompt }],        // 对话 = 数组，每次全量发送
});

console.log(response.choices[0].message.content);       // stdout = 程序输出（测试检查它）
console.error("...");                                   // stderr = 给人看的日志（不影响测试）
```

## 核心概念

### 1. Harness 三层架构

```
UI/CLI  ←—— 用户说"帮我读这个文件"
  ↓
harness（本课程正在构建的东西）
  ↓
LLM API（远程 HTTP 服务）
```

后面 5 关全部是在"调用 LLM"这个动作上逐层加东西：工具 → 工具执行 → 循环 → 更多工具。

### 2. 消息协议与无状态性

- LLM 是**无状态**的：每次请求之间互不记得。所谓多轮对话 = 每次把完整历史 `messages` 数组重新发一遍。
- roles：`system`（系统指令）/ `user` / `assistant` / `tool`。
- **上下文管理 = 手动管理 messages 数组**。这是理解一切 agent 框架的钥匙。
- 推论："继续生成" = "带着历史再调一次 API"。状态永远在 harness 自己维护的数据结构里，不在连接里、不在模型里。

### 3. OpenAI 兼容 API

- `/chat/completions` 格式是行业事实标准（"OpenAI-compatible"）：OpenRouter、DeepSeek、vLLM 全部兼容。
- 另一大流派是 Anthropic Messages API（`tool_use`/`tool_result` 消息块）——协议格式不同，思想完全一样。本机 Claude Code 就是通过 DeepSeek 的 Anthropic 兼容端点运行的。
- baseURL 可替换 = 协议是接口约定，不是厂商锁。

### 4. openai SDK 的本质

- 它只是**包装过的 fetch**：把 TS 对象序列化成 JSON，POST 到 `{baseURL}/chat/completions`，带上 `Authorization: Bearer <key>`，再把响应解析成带类型的对象。
- 模型不在库里，在远程服务器上。SDK 附加价值：类型安全、重试、超时、错误分类、流式解析。
- 底层等价物：一个 `fetch` 调用即可复刻全部功能。

### 5. `new OpenAI()` 不是 session

- client 实例 = 无状态连接配置（baseURL + key + 默认超时），类比 axios 实例。
- session 概念由 harness 自己发明：Claude Code 的 transcript、dsh 的 `dsh-session` 包，都是把"手动管理的 messages 数组"结构化、持久化的产物。
- 特例：OpenAI Responses API 的 `previous_response_id` 可把状态存服务器端——即便如此，session 在服务器上，也不在 client 实例里。
- **判断"状态在哪一层"是设计 AI 系统的核心直觉。**

### 6. 非流式 / 流式 / 截断 —— 三个层次

| 模式 | 机制 | 是否需要手动拼接 |
|---|---|---|
| 非流式（本关，默认） | 服务器生成全部 token，拼好完整 JSON 一次性返回 | 否，`content` 直接是全文 |
| 流式（`stream: true`） | SSE 持续推碎片，每 chunk 只有 `delta.content` | 是，必须 `+=` 累积（Claude Code 打字机效果） |
| 截断（超出 `max_tokens`/上下文窗口） | `finish_reason: "length"` | 拼接无用，只能带着历史再调一次续写 |

坑位预告：tool calls 的 `arguments` 常被模型分片返回（流式下必然分片），需按 `tool_calls[].index` 归拢拼接——Stage 3 实测。

### 7. 工程约定

- API key 走环境变量，绝不硬编码（代码进 git，key 不进）。
- stdout 是程序输出（测试断言它），stderr 是日志（永远不影响测试）。
- 防御性检查：`choices` 可能为空，取 `[0]` 前先验。

## 环境事实（2026-08-26 检查）

- 本地未设置 `OPENROUTER_API_KEY` → 本地运行会在 key 检查处 throw。
- 远程（`codecrafters submit`）：CodeCrafters 沙箱注入它们自己的 OpenRouter key，Stage 1 无需本地任何配置即可通过。
- 本机 bun 未安装、依赖未安装（`node_modules` 不存在）——Stage 1 提交不依赖本地 bun，Stage 2+ 本地测试需要。

## Q&A 记录

1. **openai 库是做什么的？** 包装过的 fetch：序列化请求、发 HTTP、解析响应。模型不在库内。
2. **apiKey/baseURL 实际值是什么？** 本地：key 未设置 → throw；远程：CodeCrafters 注入 key，baseURL 用默认 OpenRouter 地址。
3. **client 实例还有什么功能？** 相关：`chat.completions`（含 `tools`、`stream`）、`responses`（新一代 API）、`embeddings`（RAG）；无关：audio/images/fine-tuning 等。
4. **一个 new 就是一个 session 吗？** 不是。client 无状态，session 在你自己维护的 messages 数组里。
5. **有替代品吗？** 分层：换协议（`@anthropic-ai/sdk`）、裸 HTTP（fetch 手拼）、上层框架（Vercel AI SDK / LangChain / Agent SDK）。生产 harness 不直接绑 SDK，而是定义自己的 LLM 接口层，SDK 作 adapter——见 dsh `packages/llm/`。
6. **21-24 行返回是拼接好的吗？** 非流式：服务器已拼好，无需再拼；流式：需手动 `+=`；截断：拼不了，需续写。

## 与生产 harness 的对照（后续深化用）

- **dsh `packages/llm/`**：Service Definition/Consumer + providers —— 生产级做法是抽象 LLM 接口，不锁死 SDK。
- **dsh `packages/core/`**：session、tools、agent-loop —— 本课程 6 关对应的概念，在那里是正式模块。
- **Claude Code**：流式输出（SSE）、transcript 持久化、Anthropic Messages 协议。

## 待办

- [ ] 安装 bun：`brew install oven-sh/bun/bun`
- [ ] `bun install`
- [ ] Stage 2+ 本地调试方案：`OPENROUTER_API_KEY=<DeepSeek key>` + `OPENROUTER_BASE_URL=https://api.deepseek.com/v1` + 模型名支持 env 覆盖（现硬编码 `anthropic/claude-haiku-4.5`，DeepSeek 端点上不存在）
