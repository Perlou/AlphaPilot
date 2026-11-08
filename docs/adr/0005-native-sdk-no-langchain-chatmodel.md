# ADR-0005：模型调用走官方 SDK，不用 LangChain ChatModel

- 状态：已接受
- 日期：2026-11-08

## 背景

LangGraph 通常配合 LangChain 的 ChatModel（如 `ChatOpenAI`、`ChatAnthropic`）使用。但我们要用到的模型都有不少**厂商特有的参数和行为**：

- **DeepSeek V4**：
  - 用 `extra_body.thinking` 开关思考，用 `reasoning_effort` 控制强度；
  - 带 tools 时必须完整回传历史的 `reasoning_content`，否则返回 400；
  - strict 模式要换到 `/beta` 端点；
  - 只支持 `json_object`，偶尔会返回空白内容。
- **Claude**（Opus 5 / Sonnet 5）：
  - 删除了 `temperature`；
  - thinking 改为 adaptive，深度用 `output_config.effort` 控制；
  - 结构化输出用 `output_config.format`；
  - 部分模型禁止强制 `tool_choice`；
  - 有 `refusal` 停止原因。

通用封装往往跟不上这些变化，还会掩盖问题：例如 `reasoning_content` 被丢掉，导致 400。

## 决策

- **LangGraph 只负责编排**。节点函数调用我们自己的 `LLMRouter`。
- `LLMRouter` 的 provider 直接使用**官方 SDK**：DeepSeek 和 Ollama 用 `openai` SDK，Claude 用 `anthropic` SDK，测试用 FakeLLM。
- 由 router 统一提供 `complete_structured` 和 `run_tool_loop`，内部屏蔽各厂商的差异，包括 schema 注入、空白重试、修复重试、回退、缓存、计费和 `reasoning_content` 回传。
- 不依赖 `langchain-*` 的模型封装和 `langchain-mcp-adapters`。MCP 客户端直接使用官方 `mcp` SDK。

## 备选方案

| 方案 | 不选的原因 |
|---|---|
| langchain-openai / langchain-anthropic | 新参数支持滞后；`reasoning_content` 和 strict beta 端点处理不透明；依赖树庞大 |
| LiteLLM 统一网关 | 又多一层抽象，同样存在新参数滞后的问题；我们只需要 3 个 provider |

## 后果

- ✅ 能第一时间使用厂商的新特性，出问题时也能直接看到原始请求和响应。
- ✅ 依赖更少，FakeLLM 与真实 provider 实现同一个接口，测试简单。
- ⚠️ 需要自己维护约 3 个 provider 适配器，以及 tool loop 的格式转换。用 provider 契约测试来保证行为一致。
