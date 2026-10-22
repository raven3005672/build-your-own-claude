# 课程总结 · 从 0 到 Agent Harness

> CodeCrafters「Build your own Claude Code」
> 完成日期：2026-10-23
> 覆盖：12 关（6 个主关卡 + 6 个 Skill 扩展关）

## 一、课程地图

| # | Stage | 内容 | 关键概念 |
|---|-------|------|----------|
| 1 | yy2 | 与 LLM 通信 | 接入层 / 消息协议 / 无状态性 |
| 2 | aq1 | 声明 Read 工具 | 工具 schema（JSON Schema 广告） |
| 3 | md6 | 执行 Read 工具 | tool_use → tool_result 闭环 / id 配对 |
| 4 | ff2 | Agent Loop | while 循环 / 终止条件 / 错误即 observation |
| 5 | oz7 | Write 工具 | 副作用工具 / 路径守卫 / 可逆性分级 |
| 6 | oq5 | Bash 工具 | 任意执行 / 数据通道 vs 异常通道 |
| 7 | vh1 | Skill Level 1 | 渐进披露 / frontmatter 广告 |
| 8 | — | Skill Level 2 | slash 宏展开 / body 替换 |
| 9 | — | Skill 参数 | `$ARGUMENTS` / `$n` / 单遍替换 |
| 10 | — | 技能叠加 | 连续消费 token / 共享参数 |
| 11 | — | 模型自主调用 | Skill 工具 / 描述即检索索引 |
| 12 | — | 子代理 | `context: fork` / 同一 loop 换 messages |

（Stage 8–12 的课程 ID 未记录，按顺序编号）

## 二、最终架构

```
main()
 ├─ 启动：扫描 .claude/skills → system prompt（Level 1 广告）
 ├─ 输入：slash 解析（叠加 / 参数 / 占位符替换） 或 原始 prompt
 └─ runAgentLoop(client, messages)                 ← 同一个循环
      ├─ LLM 请求（tools: Read / Write / Bash / Skill）
      ├─ 工具执行 → 成功/失败都作为 observation 回传
      └─ Skill handler
           ├─ 普通技能 → 返回 body（替换 user 消息 / 作为 tool 结果）
           └─ context: fork → runAgentLoop(子对话) → 只回传答案
```

## 三、贯穿全程的七条原则

1. **无状态性**：服务器不记任何东西，历史由 harness 每轮全量回传。
   它同时解释了 caching（复用计算）、compaction（缩写历史）、context editing（删旧结果）的全部设计。
2. **话语权 vs 执行权**：模型决定想做什么，harness 决定允许做什么
   （工具白名单、权限确认、沙箱）。
3. **错误是 observation**：工具失败、命令非 0、未知技能、参数错误——
   都翻译成模型可读的文本回传，不炸循环。
4. **广告与执行分离**：工具 schema、技能 frontmatter 都是索引；
   内容按需加载（渐进披露：name/description → body → 附属文件）。
5. **语法在 harness 消费**：slash 命令、`$ARGUMENTS`、fork——
   模型只看到展开后的结果，永远看不到调用语法。
6. **可逆性分级**：Read（无副作用）< Write（局部不可逆）< Bash（任意）。
   检查提高门槛，结构提供保证（软边界 vs 硬边界）。
7. **循环是函数**：agent loop = f(messages)。
   子代理不是新机制，是同一个函数换一份 messages 递归调用。

## 四、你现在拥有 / 还缺什么

| 已具备 | 生产 harness 还需要的 |
|--------|----------------------|
| 接入层 + 消息历史管理 | 上下文管理：缓存断点、压缩、编辑 |
| 工具注册表 + 错误回传 | 权限系统：逐工具确认、白名单、审计 |
| 读 / 写 / 执行三工具 | 沙箱：OS 级（Seatbelt/bwrap）或容器隔离 |
| Skill 广告 / 调用 / 参数 / 叠加 / 模型自主调用 | 会话持久化 / 多会话管理 |
| 子代理递归 | 流式输出 / UI / 多代理编排 / MCP |

## 五、沉淀物

- `docs/stages/01`–`12`：每关的代码骨架、核心概念、调试实录、Q&A
- `docs/discussions/01`：综合讨论 ①–③（可逆性分级 / 软边界 vs 硬边界 / 写工具设计谱系）
- 本文件：课程全景

## 六、待续

- 综合讨论 ④ 沙箱深潜（Seatbelt/bwrap 如何限制进程能力）
- 综合讨论 ⑤ 不设限的诚实 vs 生产中间地带 + 账单数值体感
- Phase 2（规划）：Web UI 壳、插件系统、对照 `deepseek-harness` 源码、
  Edit 工具 + 读后写规则练习
- 课程若更新，按新关卡继续
