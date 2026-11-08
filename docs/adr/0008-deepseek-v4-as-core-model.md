# ADR-0008：DeepSeek V4 作为核心模型，Claude 作为备选

- 状态：已接受
- 日期：2026-11-08

## 背景

AlphaPilot 需要一个能做金融假设推理、写 DSL 和撰写长篇报告的 LLM。候选包括 DeepSeek V4、Claude（Opus 5 / Sonnet 5 / Haiku 4.5）和本地 Ollama（qwen3.5）。项目所有者决定以 **DeepSeek V4** 作为核心模型。

已按 DeepSeek 官方文档核实（2026-11-08）：

- `deepseek-v4-pro`（DeepSeek-V4-Pro-0813）和 `deepseek-flash`（DeepSeek-V4.1-Flash-1017），都是 1M 上下文，最多输出 384K。
- 支持 thinking 模式、Tool Calls、JSON Output、strict 模式（Beta），以及 OpenAI 和 Anthropic 两种格式的 API。
- 价格分高峰和非高峰时段：Pro 的输入（缓存未命中）为 $1.32/M，输出为 $3.96/M（高峰），非高峰减半；Flash 大约便宜 4 倍。
- `deepseek-chat` 和 `deepseek-reasoner` 已下线。

## 决策

- **主力模型**：`deepseek-v4-pro`，用于 planner 和 reporter，thinking 强度设为 high。
- **轻量模型**：`deepseek-flash`，用于 coder、critic 和 monitor，按角色决定是否开启 thinking。
- **备选和回退**：Claude（`claude-opus-5` 等），在 DeepSeek 不可用或结构化输出反复失败时启用。
- **离线开发**：Ollama qwen3.5。**测试**：FakeLLM。
- 具体的角色链见 `configs/llm.yaml` 和 `docs/design/llm-router.md`。

## 后果

- ✅ 成本比 Claude Opus 低一个数量级，单个 run 的默认预算可以设为 0.5 美元。
- ✅ 境内访问稳定（`api.deepseek.com` 加入 NO_PROXY）。
- ⚠️ **结构化输出较弱**：只支持 json_object，不支持 json_schema，偶尔返回空白。对策是在 prompt 中嵌入 schema 和示例，用 Pydantic 校验，失败时做空白重试和修复重试，必要时改用 strict tool call，最后回退到 Claude。以 DSL 合法率作为 eval 指标持续监控。
- ⚠️ **thinking 加 tools 时必须回传 `reasoning_content`**，由 router 统一处理（参见 ADR-0005）。
- ⚠️ **官方未公布知识截止日**：在 evidence_level 计算中保守地以发布日期作为截止日的上界（Pro 为 2026-08-13，Flash 为 2026-10-17），官方公布后再更新。
- ⚠️ 价格和模型名称变化较快：价格表集中放在配置里，CI 检查配置中不得出现已下线的模型名。
