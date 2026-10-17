# Stage 08 · Skill Level 2（按名调用与正文加载）

> 课程：CodeCrafters「Build your own Claude Code」· Skill 系列
> 实现仓库：`~/github/codecrafters-claude-code-typescript`
> 日期：2026-10-18

## 本关目标

`-p "/apple"` → 解析技能名 → 读取 `.claude/skills/apple/SKILL.md` 的 **body**（closing `---` 之后的一切）→ **用 body 替换用户的原始 prompt**。只加载被调用的那一份；其他技能停留在 Level 1。

## 规格

```
$ ./your_program.sh -p "/apple"
```

messages 变成：

```jsonc
[
  { "role": "system", "content": "You have access to the following skills:\n\n- apple: ..." },
  { "role": "user",   "content": "<apple SKILL.md 的 body>" }
]
```

## 代码骨架（本关形态）

```ts
// main：
let userContent = prompt;
const skillName = parseSlashCommand(prompt);
if (skillName) {
  const body = loadSkillPrompt(skillName);
  if (body) userContent = body;
}

const messages = [
  { role: "system", content: readSkillsInfo() },
  { role: "user", content: userContent },
];
```

```ts
function parseSlashCommand(command: string): string | null {
  const m = command.match(/^\/(\S+)/);
  return m ? m[1] : null;
}
```

`parseFrontmatter` 升级为返回 `{ fields, body }`：

- `fields` 供 Level 1 广告使用（name / description）
- `body = text.slice(match[0].length).trim()`——closing `---` 之后的一切，trim 掉空行

## 核心概念

### 1. Slash command = harness 层的宏展开

模型**看不到 `/apple` 三个字**——它收到的是 body 作为 user 消息。识别、解析、加载、替换全部发生在 harness，且发生在**第一次 API 调用之前**。

这是**确定性路由**（字符串前缀匹配），不含任何模型判断。对比生产里的另一种形态：模型自主调用技能（Skill 工具）是非确定性路径，技能内容会作为 tool 结果**追加**到对话、原请求保留。**谁发起，决定替换还是追加。**

### 2. 渐进披露 Level 2

| 层级 | 内容 | 时机 |
|---|---|---|
| Level 1 | name + description | 始终 |
| **Level 2** | SKILL.md 正文 | **仅被调用的那个** |
| Level 3 | 脚本 / 参考文件 | 正文指到时（后续） |

只加载一份不仅是省 token，更是**正确性**：两份技能正文 = 两套可能冲突的指令。测试正是创建两个技能、各藏一个不同的 word，若全部加载必挂。

### 3. 为什么是"替换"而不是"拼接"

- `/apple` 是**语法**，不是内容——语法由 harness 消费，不该交给模型（类比 shell alias：`ll` 展开成 `ls -l`，不会执行 `ll ls -l`）
- body 是技能作者写的**完整指令**，拼接会让模型看到无意义的调用串、稀释真正的指令
- 可复现：同一个技能无论从 slash / CLI / API 调用，模型看到的内容都一致

### 4. 成本视角

body 一旦进入对话，后续每一轮都会随历史重发（无状态协议的账单）。这正是"按需加载而不是全量预载"的理由之一。

## 调试实录：半迁移（生产方没改）

消费方先写了 `frontmatter.body`，但 `parseFrontmatter` 仍只返回 key-value → `body` 恒为 `undefined` → `?? null` → 回退原始 prompt → 模型看到 `/apple`，测试必挂。

修复：把 `parseFrontmatter` 返回结构改成 `{ fields, body }`，并同步两个调用点：

- `readSkillsInfo` → `parsed.fields.name / parsed.fields.description`
- `loadSkillPrompt` → `parsed.body`

**元教训：调用方改了，生产方必须同步改。** 这是"半迁移"在本项目第一次出现（后面 Stage 9 又出现一次）。

## 实测结果（2026-10-18）

- Stage 通过 ✔：输出被调用技能 body 里藏的 word，exit code 0
- 前置关卡回归全绿

## 下一关

技能参数：body 里的 `$ARGUMENTS` / `$n` 占位符替换（见 `09-skill-arguments.md`）。
