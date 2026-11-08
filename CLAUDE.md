# CLAUDE.md — AlphaPilot 开发约定

本文件供 Claude Code 等 AI 编码助手和新加入的开发者阅读。**动手之前先读完。**

## 项目是什么

AlphaPilot：AI Agent 驱动、人工把关的 A 股量化研究与交易平台，是 qtc 课程的结业项目。
**权威设计来源**：`docs/superpowers/specs/2026-11-08-alphapilot-design.md`。实现与 spec 冲突时，先停下来讨论，不要擅自偏离。

## 不可违反的原则

1. **Agent 不下单**（ADR-0004）：LLM 只负责提想法、写 DSL、写文字；判定由确定性代码完成；关键动作必须经过人工审批。
2. **护栏放在工具层**（ADR-0007）：新增工具时必须明确三件事：是否计入 trial、适用哪些护栏、输出是否已脱敏。
3. **诚实性宪章**（`docs/honesty-charter.md`）：代码评审时逐条核对文末的“设计约束清单”。
4. **数字不能由 LLM 生成**：报告中的数字只能通过占位符由代码渲染。

## 分层与依赖方向

```
cli / api / worker / mcp      （薄适配层，不放业务逻辑）
        ↓
agents / pipeline             （编排）
        ↓
tools                          （唯一的研究数据入口，护栏在这一层）
        ↓
quant / data / trading         （纯内核：不能 import db、llm、api、agents）
llm / db                       （基础设施，由上层注入）
```

依赖方向由 import-linter 在 CI 中强制检查。`quant/`、`data/`、`trading/` 必须能在不连数据库、不调 LLM 的情况下单独测试。

## 领域规则（迁移自 qtc，务必保持）

- 收益**只用 Sina 等比复权**（`get_stock_daily_adj`）。东财的 qfq/hfq 是加减法复权，除息日会失真。ETF 用 `get_etf_daily_adj`。
- 撮合用不复权价，收益用 `adj_close`。
- 创业板用 `startswith("30")` 判断（包含 302 号段）。主板 ST/*ST 自 2026-07-06 起涨跌幅为 ±10%。
- 用 `is_synthetic(df)` 判断合成数据，不要用 `is_offline()`。
- 缺少 libomp 时，lightgbm 和 xgboost 降级到 sklearn HistGradientBoosting。qlib 使用 `kernels=1`。

## LLM 约定

- **核心模型是 DeepSeek V4**（ADR-0008）：主力用 `deepseek-v4-pro`，轻量任务用 `deepseek-flash`。**禁止使用** `deepseek-chat` 和 `deepseek-reasoner`，它们已经下线。
- 模型调用直接用官方 SDK（ADR-0005）：DeepSeek 和 Ollama 用 `openai`，Claude 用 `anthropic`。不使用 LangChain ChatModel。
- DeepSeek 在 thinking 模式下，只要请求带了 tools，就**必须回传历史的 `reasoning_content`**。这由 router 统一处理，节点代码不要自己拼消息。
- DeepSeek 只支持 `json_object`：schema 写进 prompt，返回后用 Pydantic 校验，空白响应要重试。
- 不传 `temperature`（thinking 模式和 Claude Opus 5/Sonnet 5 都不支持）。可复现性依靠 LLM 缓存回放。
- 测试和 CI 一律使用 `AP_LLM_DEFAULT=fake:scripted`，不发起真实调用。真实调用必须由人手动触发，并受预算限制。

## 开发命令（M0 之后可用）

```bash
make setup      # uv sync（清华镜像）
make check      # ruff、mypy、import-linter
make test       # unit、golden、agent（AP_OFFLINE=1）
make itest      # integration（需要 make up 启动 Postgres）
make up / down  # docker compose
make migrate    # alembic upgrade head
make web        # pnpm -C web dev
```

## 环境注意事项

- **uv 安装依赖必须加清华镜像** `-i https://pypi.tuna.tsinghua.edu.cn/simple`，默认的 PyPI 源极慢。
- macOS 没有 `timeout` 命令。
- 本机设置了 `https_proxy=127.0.0.1:7897`。东财、新浪和 `api.deepseek.com` 需要加入 `NO_PROXY`。
- 密钥（`DEEPSEEK_API_KEY`、可选的 `ANTHROPIC_API_KEY`）只从环境变量或 `.env` 读取，**不入库、不进日志**。
- qtc 课程仓库在 `../qtc`，其中 `data/cache` 有大量 parquet 缓存，可以用 `ap data import-qtc` 导入。

## 工作方式

- 按里程碑推进（见 `ROADMAP.md`）。每个里程碑先写测试（TDD），完成后进行评审、提交，然后暂停，等待确认。
- 提交信息使用 Conventional Commits（`feat:`、`fix:`、`docs:`、`refactor:`、`test:`、`chore:`）。
- 修改设计时，先改 spec，有取舍的决策要补写 ADR，然后再写代码。
- 文档和注释用中文，标识符用英文。
