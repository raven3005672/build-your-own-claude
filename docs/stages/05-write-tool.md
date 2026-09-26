# Stage 05 · Write 工具（第一个副作用工具）

> 课程：CodeCrafters「Build your own Claude Code」· Stage oz7
> 实现仓库：`~/github/codecrafters-claude-code-typescript`
> 日期：2026-09-20

## 本关目标

工具列表从 1 个变 2 个：`Read` + `Write`，测试让模型**自己在两个工具之间选**（多工具协作），并检查参数校验。这是**第一个有副作用的工具**——"话语权 vs 执行权"从讨论题变成代码里要真处理的东西。

## 代码骨架（最终版）

```ts
import path from "path";
const ROOT = process.cwd();

// ① 分发：名字 → 处理函数（switch 的替代形态）
const toolImpls: Record<string, (id, fn) => Promise<void>> = {
  Read: handleReadToolCall,
  Write: handleWriteToolCall,
};
await (toolImpls[tool_call_function.name] ?? handleUnknownToolCall)(id, tool_call_function);

// ② 路径守卫：Read/Write 共用的唯一入口
function resolveInsideRoot(filePath: string): string {
  const resolved = path.resolve(ROOT, filePath);
  if (!resolved.startsWith(ROOT + path.sep)) {
    throw new Error(`Invalid file path: ${filePath}`);
  }
  return resolved;
}

// ③ 写：校验失败 / 写入失败都走 Stage 4 的错误信封
async function handleWriteToolCall(id: string, fn: any) {
  try {
    const { file_path, content } = JSON.parse(fn.arguments);
    const resolved = resolveInsideRoot(file_path);   // 校验的对象……
    await writeFile(resolved, content, "utf-8");      // ……就是使用的对象
    messages.push({ role: "tool", tool_call_id: id, content: `Successfully wrote to file: ${file_path}` });
  } catch (error) {
    const reason = error instanceof Error ? error.message : String(error);
    messages.push({ role: "tool", tool_call_id: id, content: `Error writing to file: ${reason}` });
  }
}
```

## 核心概念

### 1. 循环与工具解耦：分发形态的三级演进

Stage 4 的循环对工具一无所知（拿名字和参数 → 执行 → 回传）→ **加工具只改两处**：声明侧（tools 数组）+ 执行侧（分发）。本关落地了中间形态：

```
if/else 分支 → switch + handler → 映射表（本关）→ 工具注册表（生产形态）
```

映射表的隐藏收益：顺手绕开 switch 的 **case 块作用域坑**——所有 case 共享一个块作用域，两个 case 里各写一个 `const newMessage` 会直接 SyntaxError（Stage 6 加第三个工具时就会踩到）。

再加一层就是"一切皆插件"：每个工具是自包含的"声明 + 执行"对象，harness 只负责遍历注册表——`deepseek-harness` 的架构起点。

### 2. 第一个副作用工具：可逆性分级

| 工具 | 不可逆性 | 最坏情况 | 控制手段 |
|---|---|---|---|
| Read | 无（只读） | 数据泄露（看到不该看的） | 路径守卫 |
| Write | 局部（覆盖即丢） | 数据破坏（写错位置/覆盖，**不可逆**） | 守卫 + 读后写 + 确认 |
| Bash（下关） | **任意** | 一切 | 守卫失效 → 沙箱 + 审批 |

原则：**工具的信任等级 ∝ 不可逆性**。推论：同一个 Bash 在沙箱里和沙箱外信任等级不同——这就是为什么生产权限系统要感知沙箱状态（Claude Code 的 `autoAllowBashIfSandboxed`）。

### 3. 路径守卫：防目录穿越

- `path.resolve(ROOT, p)` 会把 `../../` 折叠成绝对路径，再检查是否还在根内
- **`startsWith(ROOT + path.sep)` 里的 `+ path.sep` 是必须的**——否则 `/proj-evil` 这种同前缀目录会被误判为"在根内"（`"/proj-evil/x".startsWith("/proj") === true`）
- **validate-and-use 一致性**：校验的对象必须就是使用的对象。反例（本关 review 抓到）：算了 `resolved` 做检查、却把原始 `file_path` 传给 `writeFile`——效果暂时等价，但形状是隐患
- 诚实边界：`path.resolve` **不解析符号链接**（项目内软链可指向外部绕过）；用户态守卫会被自己的 bug / TOCTOU / 没想到的路径形态打败——详见"待讨论议题 ②"

### 4. 参数校验：校验失败也是 observation

- `JSON.parse` 成功 ≠ 参数合法：模型可能缺参数、类型错、给怪路径
- schema 里的 `required` 是**给模型看的广告**，不是强制执行——执行侧仍要校验
- 校验失败 `throw` → 同一个 catch → 错误信封回传，模型自己修参数重试（Stage 4 的模式原样复用）
- 顺带：`handleUnknownToolCall` 也走信封——"未知工具"是 observation，不该 throw 炸掉会话

## 迭代实录：从"简单版本"到"有边界"

1. 简单版：声明 + switch 分发 + try/catch → **直接通过测试**（课程测试只走 happy path）
2. Review 发现：`writeFile(file_path)` 照单全收——模型若被注入，`../../.ssh/authorized_keys` 会被照写
3. 加守卫 → 暴露"算了 `resolved` 却写 `file_path`"（validate A, use B）
4. 对称化：Read 也没有守卫（`~/.ssh/id_rsa` 照读）→ 抽出 `resolveInsideRoot` 共用；写入改用 `resolved`
5. 加上"账单"打印：`console.error(usage.prompt_tokens / completion_tokens)`

**元教训：测试通过 ≠ 安全。** 课程测试只测"设计者想到的正常路径"，路径穿越、覆盖语义、符号链接一概不测——**安全靠结构保证，不靠测试保证**（详见议题 ②）。

## 实测结果（2026-09-20）

- Stage oz7 通过：模型自行在 Read / Write 间选择，多工具协作成立
- 账单打印已上线（stderr 通道）；下次 submit 时可在测试输出的 `[your_program]` 行看到每轮 token 数——**输入每轮递增、输出每轮几十个 token**，成本画像肉眼可见

## 待讨论议程（已存档，课程结束后综合讨论）

本关的三个讨论方向按用户决定推迟到课程结束，要点存档：

### ① 可逆性分级：工具的信任等级 ∝ 不可逆性

Read（无副作用）→ Write（局部不可逆）→ Bash（任意不可逆）；控制手段随级别加码：路径守卫 → 读后写/确认 → 沙箱 + 审批。同一条命令（Bash）在沙箱内外信任等级不同 → 权限系统必须感知沙箱状态。

### ② 软边界 vs 硬边界：为什么"测试通过 ≠ 安全"

- 用户态**软边界**（代码守卫）的四类失效：自己的 bug、符号链接、TOCTOU 竞态、任何没想到的路径形态
- 攻防不对称：**防守方要覆盖所有路径，攻击方只需要找到一条**
- 生产架构两层：软边界（体验好、报错清楚、能解释给模型）+ 硬边界（内核沙箱，不依赖任何代码正确性）
- 核心句：**安全靠结构保证，不靠测试保证**；LLM-as-sensor 测试更弱一层——它连恶意输入都不测

### ③ 写工具的设计谱系：覆盖语义与"读后写"

- **读后写规则**：没 Read 过的文件不许 Write——防盲写覆盖（Claude Code 的规矩）
- **Edit vs Write 分工**：整文件重写会摧毁模型没复现的细节（格式、注释、文件其它部分）→ 真实 harness 配一个"外科手术式"Edit（字符串替换，找不到匹配即报错，天然防呆）
- **原子写**：先写临时文件再 rename——防进程死在写一半留下半截文件
- 上层观察：coding agent 敢大改文件，是因为**工作区有 git 兜底**——"不可逆"被版本控制变回"可逆"。**给不可逆操作配一个撤销层，比试图禁止所有危险操作更实用**

## 与生产 harness 对照

- Claude Code：Write / Edit 分家；读后写规则；写入类操作默认需要确认
- `autoAllowBashIfSandboxed`：沙箱状态影响权限决策（信任等级不是工具的固有属性）
- `deepseek-harness`：工具 = 插件注册表（`packages/core/tools`）——"一切皆插件"

## Stage 6 预习（oq5 · bash 工具）

- 任意执行 = 不可逆性的顶点：`cd ..` 一条命令就让字符串级路径守卫彻底失效
- 将第一次面对"软边界失效"——为什么必须要有硬边界
- 命令的 stdout / stderr / exit code 三件套都是要回传的 observation（Stage 4 错误模式的延伸）

## 待办

- [x] 删 `main.ts` 的 switch 注释死代码 ✔（映射表已替代）
- [x] submit 验回归 ✔（2026-09-26 全绿）
