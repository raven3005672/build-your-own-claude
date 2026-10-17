# Stage 09 · Skill 参数（Placeholder 替换）

> 课程：CodeCrafters「Build your own Claude Code」· Skill 系列
> 实现仓库：`~/github/codecrafters-claude-code-typescript`
> 日期：2026-10-18

## 本关目标

`/apple staging eu-west` → 把 body 里的占位符替换后再发给模型。

| 占位符 | 替换为 |
|---|---|
| `$ARGUMENTS` | 技能名之后的**整段字符串** |
| `$ARGUMENTS[0]`、`$ARGUMENTS[1]`… | 按零基位置取单个参数 |
| `$0`、`$1`… | 同上，简写 |

- 参数按空白切分（无需处理引号）
- 两族写法可混用、同一占位符可多次出现
- **无参数调用也要能用**：所有占位符替换为空字符串

## 代码骨架（最终版）

```ts
// main：
let userContent = prompt;
const cmd = parseSlashCommand(prompt);
if (cmd) {
  const skillPrompt = loadSkillPrompt(cmd.name);
  if (skillPrompt) {
    userContent = substitute(skillPrompt, cmd.argsText, cmd.args);
  }
}
```

```ts
function parseSlashCommand(command: string): { name: string; argsText: string; args: string[] } | null {
  const m = command.match(/^\/(\S+)([\s\S]*)$/);
  if (!m) return null;
  const argsText = m[2].trim();
  return { name: m[1], argsText, args: argsText ? argsText.split(/\s+/) : [] };
}

const ARG_PATTERN = /\$ARGUMENTS(?:\[(\d+)\])?|\$(\d+)/g;

function substitute(body: string, argsText: string, args: string[]): string {
  return body.replace(ARG_PATTERN, (_match, bracedIdx, plainIdx) => {
    if (plainIdx !== undefined) return args[Number(plainIdx)] ?? '';   // $0 / $1 / $10
    if (bracedIdx !== undefined) return args[Number(bracedIdx)] ?? ''; // $ARGUMENTS[0]
    return argsText;                                                    // 裸 $ARGUMENTS
  });
}
```

## 核心概念

### 1. 带参数的宏展开

上一关 `/apple` 是零参宏；本关 body 变成**模板**，参数是运行时输入：

```
技能作者写：Deploy to $0 in region $1. Full request was: $ARGUMENTS
用户输入：  /apple staging eu-west
模型看到：  Deploy to staging in region eu-west. Full request was: staging eu-west
```

模型依旧看不到调用语法。

### 2. 单遍替换：本关的技术核心

顺序替换会**串味**，因为存在前缀重叠：

- `$ARGUMENTS` 是 `$ARGUMENTS[0]` 的前缀
- `$1` 是 `$10` 的前缀（多位数参数）

```ts
body.replaceAll('$ARGUMENTS', argsText);   // ❌ 先把 $ARGUMENTS[0] 的前缀吃掉
body.replaceAll('$1', args[1]);            // ❌ 把 $10 拆成 "参数 + 0"
```

正确做法：**一个正则、一次扫描，交替分支"更具体的在前"**：

```
/\$ARGUMENTS(?:\[(\d+)\])?|\$(\d+)/g
```

- `$ARGUMENTS[n]` 的可选捕获组要放在裸 `$ARGUMENTS` 之前，否则后者先命中、留下 `[n]` 残骸
- `\$(\d+)` 用数字串匹配，天然支持多位数

### 3. JS `replace` 的 `$` 陷阱

替换**字符串**里的 `$&`、`$1`、`$'` 会被特殊解释；用户参数可能含 `$`。所以替换值必须用**函数回调**返回（字面值），不能传字符串。

### 4. 参数解析

- `/^\/(\S+)([\s\S]*)$/`：`[\s\S]` 兼容多行（`.*` 会漏掉换行）
- `argsText = rest.trim()`：`$ARGUMENTS` 用
- `args = argsText ? argsText.split(/\s+/) : []`：空串不能直接 split（会得到 `['']`）
- 两者语义不同：`$ARGUMENTS` 保留内部原始空格，`$n` 用切分后的数组

### 5. 占位符不是 YAML，也不是"临时应付测试"

分层：**YAML 只管 frontmatter**（name/description）；`$...` 出现在 **body** 里，是技能调用的**参数协议**。

这是 Claude Code 体系的真实约定（自定义命令/技能），课程取了子集——真实还支持命名参数、session 变量等。两族写法同时存在是规范设计：

- `$0` 简洁
- `$ARGUMENTS[0]` 显式，可消歧：`$10` 是"第 10 个参数"还是"第 1 个参数 + 字面 0"？写成 `$ARGUMENTS[1]0` 就明确

## 调试实录：半迁移（第二次）

把"解析 + 加载 + 替换"都塞进了 `parseSlashCommand`，返回值变成"展开后的完整 prompt"，但 `main` 仍把它当**技能名**传给 `loadSkillPrompt`：

```
loadSkillPrompt("<整段展开后的 body>")
→ 读 .claude/skills/<body>/SKILL.md → 失败 → 回退原始 prompt
```

修复：**拆开职责**——`parseSlashCommand` 只解析；调用方负责 load + substitute：

- 解析、加载、替换各自单一职责，可独立测试
- 技能不存在时保留原始 prompt，且**不会**对它做占位符替换

## 测试

- `/apple 4127`，body 回显 `$ARGUMENTS` → 输出 `4127`
- `/grape mango pear`，body 回显 `$1 $0` → 输出 `pear mango`（验证零基位置与顺序）
- 无参数调用正常（占位符 → 空串）

## 实测结果（2026-10-18）

- Stage 通过 ✔（含前置关卡回归）

## 待办

- 下一关以课程页面为准（Skill 系列可能还有更多扩展：插件解析、Level 3 引用文件等）
