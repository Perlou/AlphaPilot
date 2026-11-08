# AlphaPilot 路线图

> 对应 spec §10。每个里程碑都**能单独运行和验收**。完成一个里程碑的流程是：测试通过 → 代码评审 → 提交 → 暂停，等项目所有者确认后再进入下一个。
>
> 状态图例：✅ 完成 · 🔄 进行中 · ⏳ 未开始 · 🅾️ 可选

**当前阶段：设计（M0 的文档部分）🔄**

---

## M0 · 脚手架与设计文档 🔄

- [x] 设计规格书 `docs/superpowers/specs/2026-11-08-alphapilot-design.md`
- [x] ADR 0001–0008、诚实性宪章、术语表、README
- [x] 详细设计 `docs/design/*.md`（8 篇）与架构文档
- [ ] spec 评审通过，并经项目所有者确认
- [ ] M0–M1 实施计划（`docs/superpowers/plans/`）
- [ ] uv 项目（`pyproject.toml`，src 布局）与 Makefile
- [ ] ruff、mypy、pytest、import-linter 配置
- [ ] `core/`：settings（pydantic-settings）、structlog 日志、错误类型、ID 生成
- [ ] `ap` CLI 骨架（Typer）
- [ ] `deploy/compose.yaml`（先只有 Postgres）
- [ ] GitHub Actions CI（lint、typecheck、test）

**验收**：`make check && uv run ap --help`

## M1 · 量化内核迁移 ⏳

- [ ] `data/`：数据源（东财、新浪、合成）、Sina 等比复权、Parquet 数据湖、DuckDB、质量检查、快照
- [ ] `quant/`：rules、dsl（加 canonical_hash）、stats、factors、models、portfolio、backtest、research/splits
- [ ] `trading/`：事件引擎、OMS、PositionBook、RiskEngine、模拟网关、事件日志与恢复
- [ ] `pipeline/`：DailyPipeline
- [ ] 去掉全部 `sys.path` hack
- [ ] 迁移 qtc 测试：mini_quant、mini_trader、phase-01
- [ ] golden 测试：与 qtc 输出一致（误差 1e-8 以内）
- [ ] CLI：`ap data import-qtc | status | snapshot`、`ap backtest run`

**验收**：`AP_OFFLINE=1 pytest tests/unit tests/golden` 全部通过

## M2 · 工具层与 MCP ⏳

- [ ] `tools/`：ToolContext、ToolResult、内存版 TrialLedger
- [ ] 护栏：数据截止、预算、holdout 只能用一次、输出脱敏、工具不能审批
- [ ] research profile 的 10 个工具、ops profile 的 5 个工具
- [ ] MCP server（mcp 2.x 的 `MCPServer`；stdio 和 streamable-http；对外的工具名把点号换成下划线，如 `factor_evaluate`）
- [ ] 外部 MCP 会话映射为 interactive run

**验收**：护栏测试通过；在 MCP Inspector 和 Claude Code 中能调用 `factor.evaluate`

## M3 · Research Graph（最小闭环）⏳

- [ ] LLM Router：DeepSeek（主）、Anthropic（备）、Ollama、Fake；结构化输出、tool loop（含 `reasoning_content` 回传）、回退、缓存、按时段计价
- [ ] 实测 DeepSeek 结构化输出两种方案（json_object 加校验，或 strict tool call），选择合法率更高的一种
- [ ] Research Graph：plan（有限 ReAct）→ code → validate → evaluate → critic → gate_holdout → holdout → backtest → report → faithfulness → gate_register → register
- [ ] 报告占位符渲染与 faithfulness 检查
- [ ] `ap research "<goal>" --interactive`（在终端审批），报告输出到 `outputs/`
- [ ] agent 测试（FakeLLM 加 MemorySaver），覆盖 9 个场景

**验收**：`pytest tests/agent` 通过；用 DeepSeek V4 跑通一次真实 run

## M4 · 持久化与服务 ⏳

- [ ] SQLAlchemy 模型与 Alembic 迁移（全部表，含 trials 只追加的 trigger 和审计哈希链）
- [ ] PostgresSaver（放在 `lg` schema）
- [ ] jobs 队列（SKIP LOCKED）与 worker
- [ ] FastAPI：runs、approvals、factors、backtests、llm usage、health
- [ ] SSE（NOTIFY/LISTEN 加 Last-Event-ID 补发）
- [ ] 认证：Web 端 JWT、CLI 和 MCP 用 API Key；RBAC
- [ ] CLI：`ap serve | worker | review`

**验收**：`make itest` 通过；用 curl 走通“创建 run → SSE → 审批 → 恢复 → 因子 approved”

## M5 · Web 控制台 ⏳

- [ ] 工程搭建（Vite、TS、Tailwind、shadcn/ui、TanStack Query），并由 OpenAPI 生成类型
- [ ] 页面：Dashboard、Runs、Run 详情（实时时间线）、审批中心、因子库、回测报告、模拟盘、告警与日报、设置与用量
- [ ] Playwright 冒烟测试（compose 加 fake LLM）

**验收**：`pnpm -C web build` 成功，e2e 冒烟测试通过

## M6 · 生产闭环 ⏳

- [ ] `compile_to_factor`：已批准的 DSL 因子进入注册表，promote 后生成 strategy_version
- [ ] 数据刷新到最新交易日；股票池扩展到沪深 300 成分股（附带 known_biases）
- [ ] `ap trade paper --replay A:B | --date D`，每日对账
- [ ] Scheduler：交易日 16:30 依次运行数据更新、每日流水线、监控
- [ ] Monitor Graph：日报、ops 建议审批
- [ ] kill-switch（确定性代码，处理 P1 告警）
- [ ] runbook 初版

**验收**：replay 能产出 NAV 并且对账通过；kill 掉 worker 后重跑结果幂等；日报 faithfulness 检查通过

## M7 · 扩展与收尾 ⏳

- [ ] vnpy 适配器（`[vnpy]` extra）：对接 paper_account，CTP 和 XT 只保留接口
- [ ] Alpha158 子集作为基线因子库
- [ ] eval 套件：植入因子召回率、零信号面板 FDR、多模型对比
- [ ] 文档站（mkdocs），演示脚本和演示数据

**验收**：`ap eval run --suite core` 达到阈值；`mkdocs build` 成功

## M8 · 另类数据 🅾️

- [ ] 新闻和公告情绪（发布时间晚于 15:00 的算作下一个交易日）
- [ ] 研报的 point-in-time RAG
- [ ] 另写一份 spec

---

## 变更记录

| 日期 | 变更 |
|---|---|
| 2026-11-08 | 初版：确定方向（Agent 研究加交易平台）、名称 AlphaPilot、技术栈 LangGraph 加 MCP、vnpy 定位、React 控制台 |
| 2026-11-08 | 核心模型改为 DeepSeek V4（ADR-0008）；模型调用直接使用官方 SDK（ADR-0005） |
