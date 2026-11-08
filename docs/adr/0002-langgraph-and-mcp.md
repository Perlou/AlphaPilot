# ADR-0002：用 LangGraph 编排 Agent，工具通过 MCP 暴露

- 状态：已接受
- 日期：2026-11-08

## 背景

研究流程是一个**有环、有条件分支、需要中途等待人工审批（可能等待数小时甚至数天）、要求可恢复**的多步过程。qtc 的 09 课用手写的 ReAct 循环实现了原型，但没有持久化，没有中断恢复，也无法观测。

同时，我们希望量化工具不只服务于 AlphaPilot 自己的 Agent，Claude Code 等外部 Agent 也能在同一套护栏下调用它们。

## 决策

1. **编排用 LangGraph**（≥ 1.2）。
   - 用 StateGraph 定义 Research Graph 和 Monitor Graph。
   - 人工审批用 `interrupt()`，审批后用 `Command(resume=...)` 恢复。
   - 状态持久化用 `PostgresSaver`。
   - 测试时用 `MemorySaver`。
2. **工具用 MCP 暴露**：用官方 `mcp` SDK 的高层 API 实现（mcp 2.x 把 FastMCP 改名为 `MCPServer`，导入路径为 `from mcp.server.mcpserver import MCPServer`） research 和 ops 两个 profile，支持 stdio 和 streamable-http 两种传输方式。
3. **两条调用路径**：图内部通过 `ToolGateway` 调用工具，默认走进程内实现（`InProcessGateway`）。MCP 只是同一套 `tools/` 服务函数之上的一层薄适配，**两条路径共享同一套护栏**。

## 备选方案

| 方案 | 不选的原因 |
|---|---|
| 继续手写循环（qtc 09 风格） | checkpoint、interrupt、trace、并发分支都要自己实现，等于重复造轮子 |
| CrewAI / AutoGen 等多 Agent 框架 | 偏向“角色对话”式的协作，难以把统计判定这类确定性步骤和 LLM 步骤严格分开；HITL 和持久化也较弱 |
| Claude Agent SDK | 只能用 Claude 模型，与 ADR-0008（以 DeepSeek 为核心）冲突 |
| 工具只做进程内函数，不提供 MCP | 外部 Agent 无法复用，失去一个重要的展示点 |

## 后果

- ✅ 审批等待期间 worker 不占用资源；进程崩溃后可以从 checkpoint 恢复。
- ✅ Claude Code 可以直接调用 `factor.evaluate`，同样受预算和护栏约束。
- ⚠️ LangGraph 的版本演进较快，需要锁定版本，并用 agent 测试覆盖 interrupt 和 resume 的语义。
- ⚠️ checkpoint 里只存 id 和轻量状态，明细存业务表，以免 checkpoint 过大或重复存储。
