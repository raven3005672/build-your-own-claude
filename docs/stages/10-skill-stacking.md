# Stage 10 · 技能叠加（Stacking）

> 课程：CodeCrafters「Build your own Claude Code」· Skill 系列
> 实现仓库：`~/github/codecrafters-claude-code-typescript`
> 日期：2026-10-23

## 本关目标

一条 prompt 开头可以连续点名**多个技能**：每个都展开 body、用**同一份参数**替换，然后各自变成一条独立的 user 消息。

```
$ ./your_program.sh -p "/apple /grape 4127"
```

messages：

```jsonc
[
  { "role": "system", "content": "You have access to the following skills:\n\n- apple: ..." },
  { "role": "user",   "content": "Add this exact line to your response: blueberry-4127" },
  { "role": "user",   "content": "Add this exact line to your response: cherry-4127" }
]
```

## 展开规则（核心）

从 prompt 开头逐个 token 看：

1. token 形如 `/name` **且技能存在** → 展开，继续
2. 第一个不是有效技能的 token → **展开结束**
3. 从该 token 到结尾的**原始文本** → 所有已展开技能的共享参数

| Prompt | 展开的技能 | `$ARGUMENTS` |
|---|---|---|
| `/apple /grape 4127` | apple, grape | `4127` |
| `/apple 4127 /grape` | apple | `4127 /grape` |
| `/apple /pear 4127`（pear 不存在） | apple | `/pear 4127` |

要点：**"是不是技能"取决于存在性**，不是只看 `/` 前缀。`/pear` 长得像调用但技能不存在 → 终止，并把 `/pear 4127` 整段当参数。原始 prompt 不出现在 messages 里（延续"替换而非拼接"）。

## 代码骨架（最终版）

```ts
function parseSlashCommand(command: string): { names: string[]; argsText: string; args: string[] } | null {
  let rest = command;
  const names: string[] = [];

  while (true) {
    const m = rest.match(/^\/(\S+)(?:\s+|$)/);   // 下一个 token 是 /name 吗
    if (!m) break;
    const name = m[1];
    if (!existsSync(path.join(ROOT, SKILLS_DIR, name, 'SKILL.md'))) break;  // 不存在 → 终止
    names.push(name);
    rest = rest.slice(m[0].length);              // 吃掉 token 与它后面的空白
  }

  if (names.length === 0) return null;
  const argsText = rest.trim();
  return { names, argsText, args: argsText ? argsText.split(/\s+/) : [] };
}
```

```ts
// main：
const messages: any[] = [{ role: "system", content: readSkillsInfo() }];
const cmd = parseSlashCommand(prompt);
if (cmd) {
  for (const name of cmd.names) {
    const body = loadSkillPrompt(name);
    if (body !== null) {
      messages.push({
        role: "user",
        content: substitute(body, cmd.argsText, cmd.args),   // 同一份参数，各自替换
      });
    }
  }
}
if (messages.length === 1) {
  messages.push({ role: "user", content: prompt });          // 没展开任何技能 → 原始 prompt
}
```

`substitute` 完全复用（Stage 09 的实现），无需改动。

## 实现要点

1. **存在性检查**：`existsSync(.../SKILL.md)`，不能只看 `/` 前缀
2. **保留参数原文**：用 `rest.slice(m[0].length)`，不要 split/join 重建（会改写原始空白/换行）
3. **每个 body 独立替换**：同一份 `argsText/args`，多次调用 `substitute`
4. **顺序**：按展开顺序生成 user 消息（确定性更好）
5. **回退**：没展开技能（或加载失败）→ 原始 prompt，保持历关行为

## 坑清单

- 只看 `/` 不看存在性 → `/pear` 被误展开，参数全错
- split/join 重建参数 → 多个空格、换行被改写
- 展开后仍把原始 prompt 塞进 messages → 变成"追加"，模型可能照原命令回答
- 空参数（`/apple /grape`）→ `argsText = ""`，占位符替换为空串

## 测试

tester 创建 3 个技能，调用其中 2 个并带共享随机数参数；验证：

- 两条 line 都出现、都带参数
- 第 3 个未调用技能的 word **不出现**

## 实测结果（2026-10-23）

- Stage 通过 ✔（含前置关卡回归）
- 调试过程：先是单技能解析 → 改为"连续消费 token"；卡点在于**存在性判断**与**保留原始参数文本**
