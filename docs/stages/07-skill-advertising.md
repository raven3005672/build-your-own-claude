# Stage 07 · Skill Level 1（技能发现与广告）

> 课程：CodeCrafters「Build your own Claude Code」· Stage vh1（6 关之后新增的 Skill 系列第一关）
> 实现仓库：`~/github/codecrafters-claude-code-typescript`
> 日期：2026-10-18

## 本关目标

扫描 `.claude/skills/`，解析每个 `SKILL.md` 的 **frontmatter**，把 `name + description` 广告进 system prompt。**不加载正文**——这是渐进披露（progressive disclosure）的 Level 1。

## 规格

```
.claude/
└── skills/
    ├── apple/
    │   └── SKILL.md
    └── grape/
        └── SKILL.md
```

SKILL.md = frontmatter（`---` 之间）+ body：

```markdown
---
name: apple
description: Deploys the apple service to production.
---

Run the deploy script and report the version that was deployed.
```

system prompt 格式：

```
You have access to the following skills:

- apple: Deploys the apple service to production.
- grape: Runs the grape test suite and reports failures.
```

messages 变成 `[system, user]`（第一次真正使用 system role）。

## 代码骨架（最终版）

```ts
function readSkillsInfo(): string {
  const prefix = `You have access to the following skills:\n\n`;
  const skillsInfo: Array<{ name: string; description: string }> = [];
  const skillsDirPath = path.join(ROOT, SKILLS_DIR);

  let skillFiles;
  try {
    skillFiles = readdirSync(skillsDirPath, { withFileTypes: true });   // 目录级失败
  } catch (error) {
    return prefix + "No skills available.";
  }

  for (const skillFile of skillFiles) {
    if (!skillFile.isDirectory()) continue;
    try {                                                               // 技能级失败
      const skillPath = path.join(skillsDirPath, skillFile.name, 'SKILL.md');
      const frontmatter = parseFrontmatter(readFileSync(skillPath, 'utf-8'));
      if (frontmatter.name && frontmatter.description) {
        skillsInfo.push({ name: frontmatter.name, description: frontmatter.description });
      } else {
        console.warn(`Missing name or description in ${skillPath}`);
      }
    } catch (error) {
      console.warn(`Skipping skill ${skillFile.name}: ${error}`);
    }
  }

  return prefix + skillsInfo.map(s => `- ${s.name}: ${s.description}`).join('\n');
}

function parseFrontmatter(text: string): Record<string, string> {
  const match = text.match(/^---\s*([\s\S]*?)\s*---/);
  if (!match) return {};
  const result: Record<string, string> = {};
  for (const line of match[1].split('\n')) {
    const [key, ...rest] = line.split(':');
    if (key && rest.length > 0) {
      result[key.trim()] = stripQuotes(rest.join(':').trim());
    }
  }
  return result;
}
```

## 核心概念

### 1. 渐进披露（Progressive Disclosure）三层

| 层级 | 加载什么 | 体积 | 时机 |
|---|---|---|---|
| **Level 1（本关）** | name + description 等 frontmatter | ~100 token/技能 | **始终** |
| Level 2 | SKILL.md 正文 | < 5,000 token | 技能被调用时 |
| Level 3 | 附带的脚本/参考文件 | 无上限 | 正文指到它们时 |

把 10 个各 200 行的 SKILL.md 全塞进 prompt，用户还没开口上下文就满了。**Level 1 只花"一行广告"的信息量**——这是上下文管理的又一招：先给索引，按需取内容。

### 2. 广告模式第二次出现（与工具同构）

```
工具：schema 广告（名字/描述/参数结构） → 模型调用 → harness 执行
技能：frontmatter 广告（名字/描述）      → 模型调用 → harness 加载正文（后续关卡）
```

本质相同：**给模型一个索引，而不是把它可能需要的一切都塞给它**。本关实现的是"技能版工具声明"，而 body 是"技能版工具执行"。

### 3. frontmatter 是 YAML：语法 ≠ 内容

- `key: value` 按**第一个冒号**切分；值里的冒号要保留（`Deploys: production`）
- **引号是定界符，要剥掉**：`name: "falcon"` 的值是 `falcon`
  - 只剥**首尾成对的同一个引号**，且长度 ≥ 2
  - 值内部引号不动（`He said "hi"` 首字符不是引号）
  - 不剥的风险实例：name 带引号时，广告列表变成 `- "falcon": ...`，模型可能照抄引号回答 `"falcon"`，精确匹配失败
- 后续关卡会用到更多字段（`allowed-tools` 等数组/布尔类型），所以课程建议实现 proper YAML parser

### 4. system role 与稳定前缀

messages 从 `[user]` 变成 `[system, user]`。system 是每次请求都稳定在最前面的内容——**技能列表的排序稳定对 prompt caching 友好**（Stage 4 的账）。

## 调试实录

**第一次提交失败：所有技能被判"invalid metadata"。**

```ts
// 脑补的格式（注释里也这么写的）：
// name: xxxx,
// description: xxxx
const match = skillContent.match(/---\s*name:\s*(.+?),\s*description:\s*(.+?)\s*---/s);
//                                          ↑ 正则要求 name 后面有个逗号
```

真实 YAML 是换行分隔、**没有逗号** → 正则永远不匹配 → 系统提示词里没有技能 → 模型回答"I don't have access to a skill…"（**输出是输入的指纹**，CLAUDE.md #30）。

**教训**：写解析器前，先拿真实样本（哪怕一行）核对格式，别对着脑补的格式写正则。正则里的每个标点都必须是样本里真实存在的。

**第二处修复：try/catch 分级。** 目录读不到 → 整个技能系统降级（外层 catch）；单个技能坏 → 只跳过它（循环体内 catch）。catch 的粒度 = 失效的隔离级别。

## 实测结果（2026-10-18）

- Stage vh1 通过 ✔（含历关回归）
- 测试流程：随机创建 N 个技能文件夹 → 用某个技能的 description 提问"哪个技能合适" → 模型必须从广告列表里选出并只回答名字

## 与生产对照

- Claude Code Skills：项目 `.claude/skills/`（另有个人 `~/.claude/skills/`）；SKILL.md frontmatter + body；按需加载；遵循 Agent Skills 标准
- 本关实现 = 标准的第 1 层
- 标准要求 frontmatter 的 name 与文件夹名一致；**后续关卡按文件夹名调用技能**

## 下一关预期

**Skill Level 2**：技能被调用时加载 SKILL.md 正文（progressive disclosure 第二层）。可能会引入"Skill 工具"或调用协议——以课程页面为准。

## 待办 / 小尾巴

- [ ] 目录不存在时的措辞（`"No skills available."` 与前缀连读略绕）
- [ ] 技能列表排序（确定性 + 缓存友好）
- [ ] 下一关：Level 2
