# Stage 06 · Bash 工具（不可逆性的顶点）

> 课程：CodeCrafters「Build your own Claude Code」· Stage oq5
> 实现仓库：`~/github/codecrafters-claude-code-typescript`
> 日期：2026-09-26
> **🎉 Phase 1 收官：6/6 关全部通过**

## 本关目标

第三个工具：执行任意 shell 命令。工具系统的能力至此闭合——**读、写、执行**三件套 + Stage 4 的循环 = 一个完整 agent 的内核。也是安全意义上最重的一关：**不可逆性的顶点**。

## 代码骨架（最终版）

```ts
// 声明（与 Read/Write 同构，一个 command 参数）
{ "type": "function", "function": {
    "name": "Bash",
    "description": "Execute a shell command",
    "parameters": { "type": "object", "required": ["command"],
      "properties": { "command": { "type": "string", "description": "The command to execute" } } }
} }

// 执行：数据通道回传三件套（不 reject）
content = await new Promise<string>((resolve) => {
  exec(command, {
    cwd: ROOT,                    // 工作目录：显式契约
    timeout: 30_000,              // 超时杀进程
    maxBuffer: 10 * 1024 * 1024,  // 输出缓冲上限（默认仅 1 MiB）
  }, (error, stdout, stderr) => {
    const exitCode = error ? (error as any).code ?? "n/a" : 0;
    resolve(`exit code: ${exitCode}\nstdout:\n${stdout}\nstderr:\n${stderr}`);
  });
});
```

## 核心概念

### 1. 命令不是数据，是程序

上一关的 `resolveInsideRoot` 在这一关**完全失效**：

```
cd ..                          不需要路径参数，直接走出去
cat ../../etc/passwd           路径写在命令里
python3 -c "..."               藏进另一个程序的参数里
$(...) / 反引号 / ; / && / |    层层嵌套
```

**shell 图灵完备，逃逸技巧无穷——无法通过解析命令文本来判断它安不安全。** 这宣告了"字符串级守卫"这条路的终点。

### 2. 数据通道 vs 异常通道（本关最重要的设计课）

`exec` callback 的 `error` 参数**不是"出异常了"**，是"命令跑完了但退出码非 0"——它是**结果的一部分**：

| 事件 | 通道 | 理由 |
|---|---|---|
| exit 0 | `resolve` | 正常数据 |
| exit 1（grep 没找到）/ 127（命令不存在） | **`resolve`** | 有信息量的正常结果 |
| 超时被杀（code = null） | **`resolve`** | 三件套照样有，模型自己判断 |
| JSON.parse 炸了 / 我们代码有 bug | `catch` | 这才是真异常 |

走异常通道的代价（本关实测踩过）：`reject(error)` → catch 里只拿 `error.message`——**丢掉 `error.stdout`（失败前已打出的输出）和 `error.code`（退出码）**，而这两样恰恰最有诊断价值。

这是"错误是 observation"的极致形态：**连"什么是错误"都交给模型定义**。

### 3. 结果三件套：stdout / stderr / exit code

- 三样全部回传、加标签格式化，让模型判断——不替它下"成功/失败"的结论
- `stdout || stderr` 是错的：exit 0 时 stderr 里的警告也会被丢掉
- 失败 ≠ 无用：编译错误、测试失败输出都是宝

### 4. 工程参数：三个默认值都是坑

| 参数 | 不给的后果 |
|---|---|
| `timeout` | 默认**无超时**：一条等输入的命令挂死整个 agent |
| `maxBuffer` | 默认仅 **1 MiB**：跑个测试套件就 "maxBuffer exceeded" 被杀——命令其实没问题 |
| `cwd` | 默认继承（当前效果等同默认）：显式写出是**契约**——防未来启动位置变化 |

生产还有一层：**输出截断用"头尾保留、中间省略"**——错误常在头部、结论常在尾部。

## 迭代实录：半迁移陷阱

按建议加"执行参数 + exitCode 行"时，把推荐片段**叠加**在旧结构上而非**替换**：`if (error) reject / else resolve` 还留着 → `exitCode` 成了没人用的死代码、stderr 照丢。

修正一句话：**`exitCode` 那行出现时，`if/else` 两块就该整块删掉**——推荐形状里没有 reject 的位置。

## 实测结果（2026-09-26）

- Stage oq5 通过 ✔（含历关回归，Phase 1 收官）
- 账单打印已随 submit 的 `[your_program]` 输出可见（`[账单] 输入=… 输出=… 总计=…`）

## 与生产 harness 对照

- Claude Code：Bash 是权限系统管得最紧的工具；沙箱 opt-in（`sandbox.enabled`）开启后 `autoAllowBashIfSandboxed` 自动放行
- Codex：本地模式默认 OS 沙箱 + 断网 + 限写工作区
- 三家共同点：**对 Bash 的管控都不在"检查命令"，而在"限制进程能力 + 审批"**（详见议题 ④）

## Phase 1 结业：六关地图

| Stage | 内容 | 能力 |
|---|---|---|
| 1 (yy2) | 与 LLM 通信 | 接入层 |
| 2 (aq1) | 声明 Read | 工具广告（schema） |
| 3 (md6) | 执行 Read | tool_use → tool_result 闭环 |
| 4 (ff2) | agent loop | **内核**（模型驱动的多轮决策） |
| 5 (oz7) | Write | 副作用工具 + 路径守卫 |
| 6 (oq5) | Bash | 任意执行 + 数据/异常通道分离 |

→ **读、写、执行 + 循环 = 完整 agent 内核**。剩下的（上下文管理、权限系统、会话持久化、UI）都是外围。

## 待讨论（并入课程结束后的综合讨论）

- 存档议题 ①②③（见 `docs/stages/05` 末节）
- **新议题 ④**：沙箱深潜——Seatbelt/bwrap 如何"限制进程能力"，为什么这条路可行而"检查命令"不可行
- **新议题 ⑤**：课程"不设限"是诚实还是偷懒？生产的中间地带（白名单 / 只读模式 / 审批制）；账单数值的第一手体感

## 待办

- [ ] 课程结束综合讨论（议程已备：①可逆性分级 ②软边界 vs 硬边界 ③写工具设计谱系 ④沙箱深潜 ⑤不设限的中间地带 + 账单体感）
- [ ] Phase 1 总览文档（形式待讨论后定：README 一节 or 独立文档）
