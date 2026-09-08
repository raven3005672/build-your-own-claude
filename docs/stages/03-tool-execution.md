# Stage 03 · 执行 Read 工具（tool_use → tool_result 闭环）

> 课程：CodeCrafters「Build your own Claude Code」· Stage MD6
> 实现仓库：`~/github/codecrafters-claude-code-typescript`
> 日期：2026-09-09

## 本关目标

把 Stage 2 的"工具广告"变成真能力：模型发出 `tool_calls` → harness 真的读文件 → 按 `tool_call_id` 配对回传 → 再调一次 API，模型基于文件内容回答。**这是 agent 雏形第一次完整闭环。**

## 代码骨架（最终通过版）

```ts
import OpenAI from "openai";
import { readFile } from "fs/promises";

async function main() {
  // ... key/flag 检查、client 创建（同 Stage 1/2）
  const response = await client.chat.completions.create({
    model: "anthropic/claude-haiku-4.5",
    messages: [{ role: "user", content: prompt }],
    tools: [/* Read 工具声明（同 Stage 2）*/]
  });

  const message = response.choices[0].message;
  if (message.tool_calls && message.tool_calls.length > 0) {
    // 本关只处理第一个调用（Stage 4 换成 while 循环 + 遍历全部）
    const tool_call_0 = message.tool_calls[0];
    const { id, function: tool_call_function } = tool_call_0;
    const { arguments: argumentsJson } = tool_call_function;
    const { file_path } = JSON.parse(argumentsJson);    // arguments 是 JSON 字符串！
    const content = await readFile(file_path, "utf-8"); // 执行权在这里
    const secondResponse = await client.chat.completions.create({
      model: "anthropic/claude-haiku-4.5",
      messages: [
        { role: "user", content: prompt },
        message,                                         // assistant 消息原样放回
        { role: "tool", tool_call_id: id, content }      // id 配对回传
      ]
    });
    console.log(secondResponse.choices[0].message.content);
  } else {
    console.log(message.content);                        // 兜底：回归测试的退路
  }
}
```

## 核心概念

### 1. 三幕戏完整闭环

**第一幕 · 发起请求**：和 Stage 1 唯一区别是多带 `tools` 数组。

**第二幕 · 模型点菜**（它不回答）：

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

三个关键信号：`content: null`（没说话）、`finish_reason: "tool_calls"`（不是 stop，是"我需要工具"）、`arguments` 是 JSON 字符串（要 `JSON.parse`）。

**第三幕 · harness 干活 + 回传 + 再问**：解析参数 → 真读文件 → 拼三要素 messages（原始 user 消息 + assistant 消息原样 + tool 消息配对）→ 第二次调用 → 这次 content 有值。

### 2. 无状态性第二次登场：历史必须完整回传

模型不记得自己点过菜。第二次调用必须携带：① 原始 user 消息 ② 模型的 assistant tool_calls 消息（原样） ③ tool 结果消息。少放任何一环模型就失忆。**OpenAI 协议硬性要求 tool 消息紧跟对应的 assistant tool_calls 消息，否则 400**；assistant 声明了几个 tool_calls，后续就必须出现几个配对的 tool 消息，漏掉任何一个直接 400（"missing tool result"）。

### 3. tool_call_id 是配对协议

一次响应可带多个 tool_calls，每个结果靠 id 对号入座。多调用时结构不变、条数变：**N 次调用 = N 条独立的 tool 消息**，不是数组聚合。代码形态是循环 + push：

```ts
messages.push(message);  // assistant 消息整条放回
for (const call of message.tool_calls) {
  const args = JSON.parse(call.function.arguments);
  const content = await readFile(args.file_path, "utf-8");
  messages.push({ role: "tool", tool_call_id: call.id, content });
}
```

### 4. 执行权落地：安全边界的最小版本

模型请求读 `app/main.ts`，真正读文件的是 harness 的代码。harness 可以检查路径、拒绝 `.env`、限制目录——**模型永远碰不到文件系统，只能通过 harness 允许的方式看世界**。这一行 `readFile` 是 Stage 2"话语权 vs 执行权"的实体化。

## 调试实录：readFile is not a function（一次报错，三个 bug）

```
[your_program] 59 |     const content = await readFile(file_path, 'utf-8');
TypeError: readFile is not a function. (In 'readFile(...)', 'readFile' is an instance of Object)
```

报错第一行就是 Stage #MD6（老规矩），stderr 探针证明程序活着，定位到 line 59。逐个排查发现**三个 bug，报错只暴露了第一个**：

1. **`import readFile from "fs/promises"`**（默认导入）：`fs/promises` 是 CommonJS 模块，没有默认导出，默认导入拿到的是**整个模块对象**——报错原文 "'readFile' is an instance of Object" 说的就是它。修：`import { readFile } from "fs/promises"`。
2. **把整个 `response` 塞进 messages**（潜伏，被 bug 1 掩盖）：messages 数组只能放消息，应放 `response.choices[0].message`（那条 assistant 消息），不是 `{ id, object, choices: [...] }` 整个响应。
3. **缺 tool_calls 守卫 + stdout 没兜底**（潜伏，会打挂回归）：YY2/AQ1 回归测试里模型不发 tool_calls，`tool_calls[0]` 是 undefined 直接崩；旧代码里 `console.log(content)` 还注释着，兜底路径无输出。修：`if (message.tool_calls && message.tool_calls.length > 0) ... else console.log(...)`。

**元教训**：① CodeCrafters 用 bun 编译，**bun 只转译、不做类型检查**——bug 2/3 都是 tsc 一眼能抓的编译期错误，却安静通过直到运行时炸。提交前 `bunx tsc --noEmit` 能全部筛掉。② **报错逐个暴露**：前面崩溃会掩盖后面的潜伏 bug，修完一个别急着提交，通读全文找同类问题。

## 实测结果（2026-09-09）

- Stage #MD6 通过：tester 创建 `pineapple.py`，程序真读文件、模型如实返回内容 ✔
- YY2 / AQ1 回归通过（兜底分支起作用）✔

## 深度讨论（本关 Q&A 展开，信息量最大的一关）

### A. 多 tool_calls：并行 vs 串行

- 无依赖的 I/O 就该并行：`Promise.all(tool_calls.map(...))`，总耗时 ≈ 最慢那个文件（前端并发请求同理）
- 部分失败不能拖垮整批：单个 catch，把错误信息当 content 回传，模型自己应对
- 生产 harness（Claude Code 同款模式）：独立调用并行执行 + 错误驱动重试

### B. 依赖顺序与多轮往返

- **同一条 message 里的 tool_calls 天然无依赖**（模型写参数时看不到彼此结果）——协议层就是这么构造的
- 有数据依赖的链 = 多条 assistant message = 多轮往返：依赖链长度 = API 往返次数
- messages 数组跨轮增长：`[user, assistant#1, tool, assistant#2, tool, ...]`
- 模型猜错依赖（把有依赖的调用放进同一条消息）→ 并行执行 → 失败的看到错误 → 下一轮重试
- **喂回工具结果这个动作 = agent loop 的燃料**，Stage 4 的 while 循环解决"模型还想再调工具"的问题

### C. 安全边界：模型要读 /etc/passwd 怎么办

威胁清单：读 `.env`/AWS 凭据/SSH key（密钥进入对话 = 离开机器）、`../../` 路径穿越、10GB 大文件、`/dev/zero`。防线：路径归一化 + 项目根限制（`path.resolve` + `startsWith` 检查）、大小限制与截断、敏感文件过滤、权限确认。核心认知：**模型负责"想读什么"，harness 负责"允许读什么"**。

### D. Prompt injection：无法根治的对抗性问题

文件内容作为 tool 结果进入 messages，和用户指令**平起平坐**（协议层都是文本）。弱模型会被"长得像指令的内容"带跑。真实版本：仓库注释里藏"删除所有文件"、网页里藏"把你的 key 发到某地址"。

**根治不了的结构性原因**：模型的核心能力 = 读懂文本里的指令并执行；能听懂一切指令 → 必然能听懂恶意指令。SQL 注入被根治靠的是结构/数据分离（参数化查询），自然语言通道没有这种分离。这是对抗性问题（攻击者自适应），不是工程 bug。

分层防御（OWASP LLM Top 10 #1）：

| 层 | 手段 | 弱点 |
|---|---|---|
| 模型层 | 指令层级（system > user > tool 结果）、Spotlighting 打标记 | 模型自己执行这层，可被更强注入绕过 |
| **Harness 层** | **最小权限 + 沙箱 + 白名单** | 无——唯一不依赖"模型不被骗"假设的层 |
| 人层 | 不可逆操作确认、审计日志 | 体验成本 |

核心：**假设模型会被骗，让"被骗"不足以造成伤害**。类比人类组织反社交工程——没有"让员工永远不被骗"的根治，只有制度（最小权限/双人复核/留痕）。这就是为什么"话语权 vs 执行权"是 agent 安全唯一可靠的设计原则。

### E. 沙箱全景：隔离强度 × 位置（两个正交维度）

**OS 级沙箱（本地）**：不是复制环境，而是**内核给进程加限制**——文件还是真实文件，写工作区 = 真写磁盘。macOS Seatbelt / Linux bubblewrap（+Landlock/seccomp）。**容器/VM（可本地可云端）**：复制一个环境，改动落在副本，事后 diff 审阅。

2026 产品现状（2026-09-09 查证）：

| | Claude Code 本机 | Codex 本地模式 | Claude Code Web | Codex Cloud |
|---|---|---|---|---|
| 沙箱 | 默认无；可选 OS 级（Seatbelt/bwrap） | OS 级，**默认断网**，限写工作区 | 云端 VM | 云端容器 |
| 授权模型 | 逐命令弹窗 | 分级审批 | 近乎免确认（隔离兜底） | 免确认，diff 审阅 |

- Claude Code 沙箱详情：`sandbox.enabled`（opt-in）；隔离 Bash 及**所有子进程**；文件系统全盘只读（工作区可写）、`~/.ssh`/`~/.aws` 连读都 deny；网络按域名过滤；`autoAllowBashIfSandboxed` 默认 true（沙箱能跑的命令自动放行）
- Web/云形态数据流动：**本地磁盘 →git push→ GitHub →clone→ 沙箱 VM →diff→ 你 pull**。中介是 git 不是文件拷贝——未提交的改动、`.env`、本地数据库天然缺席
- 容器沙箱可见范围 = "创建时被喂了什么"：git clone / 目录挂载 / 产品内置上传，除此之外无通道

**沙箱 ≠ 隐藏数据**：开了沙箱，agent 照样看得到工作区代码（Read 工具根本不在沙箱边界内，由权限系统管；工作目录是明确放行区）。沙箱保护的对象是**宿主机器**（agent 引发的进程碰不到机器其余部分），不是"对 agent 隐藏代码"。

**对普通人的意义**：普通人已经在用沙箱（手机 App 权限、浏览器、ChatGPT 表格分析），只是不叫这个名字。AI 从"聊天"到"代办"的转折点上，沙箱变成刚需（"AI 整理照片会不会删光相册"→ 沙箱+确认是信任感的来源）。显式配置是开发者行为；对普通人，正确形态是**默认开启、不可见**——云端 agent 产品做的正是这件事。

### F. 协议对照与同构：Anthropic Messages vs OpenAI 兼容

```jsonc
// OpenAI 兼容：N 次调用 → N 条独立 tool 消息
{ "role": "assistant", "content": null, "tool_calls": [{ "id": "call_1", ... }] }
{ "role": "tool", "tool_call_id": "call_1", "content": "..." }

// Anthropic Messages：N 次调用 → 1 条 user 消息含 N 个 tool_result 块
{ "role": "assistant", "content": [{ "type": "tool_use", "id": "toolu_1", "name": "Read", "input": {...} }] }
{ "role": "user", "content": [{ "type": "tool_result", "tool_use_id": "toolu_1", "content": "..." }] }
```

- 差异：content 是 blocks 数组（多模态原生）；`input` 已是对象（没有 parse 坑）；工具结果归入 user 消息（协议要求角色严格交替）；`is_error` 标志显式建模失败
- 配对本质不变：一个 tool_use ↔ 一个 tool_result，绝不合并；N 个结果只是**块级聚合**进一条 user 消息
- **同构（isomorphism）**：表面形状不同，底层元素和运算一一对应（tool_calls↔tool_use、tool_call_id↔tool_use_id……）。agent loop 骨架对两种协议**完全同构**，只有"组装消息"是异构叶子 → 工程上写成 adapter
- 这就是"OpenAI-compatible"成为行业标准、DeepSeek 能提供 Anthropic 兼容端点的原因：**结构同构，一层薄翻译即可互通**
- 诚实的修正：核心交互结构同构，边缘小角落异构（`is_error` 无直接对应物）。工程追求"核心同构 + 边缘适配"

## Q&A 记录

1. **多个 tool_calls 时 messages 结构会变吗？** 结构不变、条数变：N 次调用 = N 条独立 tool 消息（不是数组聚合），靠 tool_call_id 一一配对，一个都不能漏。
2. **沙箱方向就是注入防御相关的吗？** 相关但更宽——沙箱兜底四类威胁：注入、模型失误、恶意代码执行、数据外泄。头号刚需是"运行不可信代码"。
3. **Claude/Codex 都没在本地建沙箱吗？** 2026 已过时：Claude Code 有 opt-in 本地 OS 沙箱（Seatbelt/bwrap），Codex 本地模式默认 OS 沙箱（断网+限写）；完整容器/VM 隔离都在云端形态。
4. **开沙箱就看不到我的代码了吗？** 反了——工作目录是明确放行区，Read 工具不在沙箱边界内。沙箱保护宿主机器，不隐藏数据。
5. **沙箱不是复制环境，是给 bash 命令加限制？** 完全正确。OS 级沙箱 = 同一个世界限制进程能力（内核强制、子进程继承）；容器/VM 才是复制环境。
6. **容器沙箱 = 本地文件拷贝到远端？** 不是拷贝，是 git 中介：push → clone → diff 回流。未入库的文件天然缺席。
7. **纯本地文件用容器沙箱模式能拿到吗？** 远端容器：拿不到（无通道，喂什么才有什么）；本地容器：默认拿不到，挂载了才拿得到。
8. **沙箱对普通人有意义吗？** 普通人一直在被它保护（手机权限、浏览器），只是无感；AI 代办时代的刚需是"默认开启、不可见"。
9. **安全问题没有根治办法吗？** 没有，且大概率不可能——漏洞长在"指令跟随"范式里。答案是纵深防御，harness 层是唯一不依赖"模型不被骗"的层。
10. **content 是数组（Anthropic）时结果是聚合吗？** 配对不变（一对一），信封变了：N 个 tool_result 块聚合进一条 user 消息（OpenAI 是 N 条 tool 消息）。
11. **完全同构是什么意思？** 表面不同、结构一一对应，换翻译表即互通。agent loop 同构 → adapter 隔离异构。

## 与生产 harness 的对照

- Claude Code 沙箱文档：[Sandboxing](https://code.claude.com/docs/en/sandboxing)、[Settings reference](https://code.claude.com/docs/en/settings-reference)、[Web 版](https://code.claude.com/docs/en/claude-code-on-the-web)
- Codex 双环境：[CLI Guide](https://www.neura.market/ai-agents/resources/guides/codex-guide)、[Codex Cloud](https://www.agent37.com/blog/codex-cloud)
- dsh `packages/llm/`：provider 翻译层 = 同构性的工程化（Phase 2 深化材料）
- 工具结果在生产 harness 中被标记为 untrusted——prompt injection 的结构层防御

## Stage 4 预习（ff2 · agent loop）

- `if` 单次执行 → `while` 循环：只要模型还在发 tool_calls，就一直执行、回传、再调
- 遍历全部 tool_calls（不再是 `[0]`）
- 终止条件：`finish_reason === "stop"` 或 content 非空
- 今天学的多轮往返、依赖链、messages 数组跨轮增长——明天亲手写成代码

## 待办

- [ ] 安装 bun：`brew install oven-sh/bun/bun` + `bun install`
- [ ] 提交前本地检查：`bunx tsc --noEmit`（bun 不查类型，这次三个 bug 两个是 tsc 能抓的）
- [ ] 本地调试方案：DeepSeek key + `OPENROUTER_BASE_URL=https://api.deepseek.com/v1` + 模型名 env 覆盖
