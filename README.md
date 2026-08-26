# build-your-own-claude

从零理解并构建一个 AI coding agent（harness）。

## 学习路径

- **Phase 1（进行中）**：跟随 CodeCrafters「Build your own Claude Code」课程，在 `~/github/codecrafters-claude-code-typescript` 实现 6 个 stage。每关学完，把概念与讨论沉淀到本仓库 `docs/stages/`。
- **Phase 2（规划中）**：回到本仓库，基于 Phase 1 的 agent 内核做深化。方向待定（候选：Web UI 壳、插件系统、对照 deepseek-harness 源码）。

配合方式：结对模式——每个 stage 先讲概念 → 亲手写代码 → 本地验证 → 提交远程测试 → 复盘沉淀。

## 课程知识地图

| Stage | 内容 | harness 概念 |
|-------|------|-------------|
| 1 | 与 LLM 通信 | 接入层 / 消息协议 / 无状态性 |
| 2 | 声明 read 工具 | 工具 schema（JSON Schema）|
| 3 | 执行 read 工具 | tool_use → tool_result 闭环 |
| 4 | agent loop | 循环 + 终止条件（harness 的心脏）|
| 5 | write 工具 | 多工具协作、参数校验 |
| 6 | bash 工具 | 副作用工具、错误处理 |

## 文档索引

- [Stage 01 · 与 LLM 通信（接入层）](docs/stages/01-llm-access.md)
- [Stage 02 · 声明 Read 工具（工具广告）](docs/stages/02-tool-advertisement.md)

## 参考

- 生产级 harness 源码：`~/github/deepseek-harness`（DeepSeek 开源，一切皆插件架构）
  - `packages/core/`：session / tools / agent-loop
  - `packages/llm/`：LLM 接口抽象 + providers
- 课程平台：<https://app.codecrafters.io/courses/claude-code/overview>
