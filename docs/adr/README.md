# 架构决策记录（ADR）

我们用 ADR 记录**有取舍的、以后难以推翻的**技术决策。格式参考 MADR 精简版：背景 → 决策 → 备选方案 → 后果。

| 编号 | 标题 | 状态 | 日期 |
|---|---|---|---|
| [0001](0001-postgres-only-no-redis.md) | Postgres 作为唯一状态存储，不引入 Redis | 已接受 | 2026-11-08 |
| [0002](0002-langgraph-and-mcp.md) | 用 LangGraph 编排 Agent，工具通过 MCP 暴露 | 已接受 | 2026-11-08 |
| [0003](0003-vnpy-borrow-design-optional-adapter.md) | 借鉴 vnpy 的设计并提供可选适配器，不以 vnpy 为底座 | 已接受 | 2026-11-08 |
| [0004](0004-agents-do-not-trade.md) | Agent 不下单，只产出可验证的研究成果 | 已接受 | 2026-11-08 |
| [0005](0005-native-sdk-no-langchain-chatmodel.md) | 模型调用走官方 SDK，不用 LangChain ChatModel | 已接受 | 2026-11-08 |
| [0006](0006-migrate-qtc-code-not-import.md) | qtc 代码迁移重构进本仓库，不跨仓库 import | 已接受 | 2026-11-08 |
| [0007](0007-guardrails-in-tool-layer.md) | 护栏放在工具层，不靠 prompt | 已接受 | 2026-11-08 |
| [0008](0008-deepseek-v4-as-core-model.md) | DeepSeek V4 作为核心模型，Claude 作为备选 | 已接受 | 2026-11-08 |

新增 ADR：复制最近的一份，编号递增，状态从“提议”开始。已接受的 ADR 不要修改；要推翻某条决策，就新写一条 ADR，并把旧的那条标为“被 00xx 取代”。
