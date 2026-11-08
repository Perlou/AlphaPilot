# ADR-0007：护栏放在工具层，不靠 prompt

- 状态：已接受
- 日期：2026-11-08

## 背景

研究流程的诚实性依赖几条硬约束：不能偷看 holdout，trial 次数有预算，统计校正不能被挑着做，Agent 不能自我审批。如果这些约束只写在 prompt 里（“请不要……”），模型可能会违反，也可能被 prompt injection 绕过；外部 MCP 客户端更是完全不受我们的 prompt 控制。

## 决策

- 所有护栏都在 `tools/guards.py` 中实现，由工具服务函数在执行前后强制检查。违反时抛出 `GuardrailError`（附错误码），并写入 `agent_events` 和 `audit_log`。
- **最小权限**：
  - `holdout.evaluate` 和 `registry.register` 不在任何 MCP profile 中暴露；
  - 任何工具都不能把 approval 设为 approved；
  - `stats.correct` 只从 ledger 读取 trial，不接受调用方传入的统计量。
- **输出脱敏**：工具的返回内容中不出现 holdout 区间的日期、股票代码和收益。
- prompt 里可以**说明**这些规则，帮助模型少走弯路，但正确性不依赖 prompt。

## 后果

- ✅ AlphaPilot 自己的 Agent、Claude Code 和人工调用都受同一套约束。
- ✅ 护栏可以被单元测试覆盖：eval 指标“成功越权次数”必须为 0。
- ⚠️ 工具接口的设计需要更谨慎，例如 `data.query` 只能返回聚合统计，可能会限制 Agent 的探索能力。这是有意的取舍。
