# AlphaPilot · 阿尔法领航员

> **AI Agent 驱动、由人类把关的 A 股量化研究与交易平台**
> Agent 当副驾驶负责研究，人当机长做决策。每个数字都由代码算出、可以溯源；每个会影响资金的决定都要经过人工审批。

![status](https://img.shields.io/badge/status-design-blue) ![python](https://img.shields.io/badge/python-3.12-blue)

> 🚧 **当前阶段：设计完成，准备进入 M0（脚手架）。** 下面的“快速开始”描述的是目标形态。各项的实现进度见 [ROADMAP.md](ROADMAP.md)。

---

## 为什么做 AlphaPilot

很多“LLM 炒股”项目有一个根本问题：**LLM 的训练数据覆盖了回测区间**，模型“知道”后来发生了什么，所以回测结果天然被污染。此外，它们很少做多重检验，编造数字的情况也很常见。

AlphaPilot 换了一种思路。**LLM 不下单**，只做三件事：

1. **提出想法**：基于经济学逻辑生成因子假设；
2. **写代码**：把假设翻译成安全的因子 DSL 表达式；
3. **写文字**：撰写研究报告和监控日报，其中的数字一律用占位符，由代码填入。

所有判定由确定性代码完成，包括 IC 检验、Holm/BH 多重检验校正、Deflated Sharpe、前视检测和相关性过滤；所有关键动作由人审批。详见 [诚实性宪章](docs/honesty-charter.md)。

## 核心特性

- 🤖 **Research Agent**（LangGraph）：假设 → DSL → IS 检验 → 统计裁决 → **人工审批 holdout** → 回测 → 报告 → **人工审批入库**。中途可以暂停和恢复，执行轨迹全程可回放。
- 🛡️ **工具层护栏**：trial 预算全局累计；holdout 每个表达式只能用一次；Agent 不能自我审批；工具输出会脱敏。对外部 Agent 同样生效。
- 🔌 **MCP Server**：量化工具以标准 MCP 协议暴露，Claude Code 可以直接调用 `factor.evaluate`。
- 🧠 **多模型路由**：以 **DeepSeek V4**（`deepseek-v4-pro` / `deepseek-flash`）为核心，Claude 作为备选，本地 Ollama 用于离线开发，FakeLLM 用于测试。成本按高峰和非高峰时段计价，并有预算上限。
- 📈 **生产闭环**：批准的因子生成新的 `strategy_version`，进入每日流水线，在模拟盘撮合并对账，再由 Monitor Agent 生成日报和运维建议；kill-switch 是确定性代码。
- 🖥️ **Web 控制台**（React）：实时 Agent 轨迹（SSE）、审批中心、因子库生命周期、回测报告、模拟盘监控。
- 🧾 **可审计、可复现**：每次运行都有 manifest（配置、代码、数据快照、模型）；审计日志用哈希链防篡改；在缓存回放模式下结果逐字节一致。
- 🔗 **vnpy 适配器**（可选）：借鉴 vnpy 的 Gateway 设计，预留通往实盘的桥接（CTP / XT）。

## 架构一览

```
 Web 控制台 ──REST/SSE──▶ API (FastAPI) ──▶ Postgres（状态·队列·事件·checkpoint·审计）
 Claude Code ──MCP──▶ MCP Server          ▲
                         │                │
                         ▼                │
               Worker：Research Graph · Monitor Graph · Daily Pipeline · Scheduler
                         │  ToolGateway（护栏）
                         ▼
          quant（DSL·统计·因子·组合·回测） data（数据湖·复权·快照） trading（OMS·风控·模拟网关）
                         │
          LLM Router：DeepSeek V4（主）· Claude（备）· Ollama（本地）· Fake（测试）
```

详见 [架构文档](docs/architecture.md)。

## 快速开始（目标形态）

```bash
# 1. 安装（uv，国内建议使用清华镜像）
make setup

# 2. 启动 Postgres 并执行迁移
make up && make migrate

# 3. 导入 qtc 课程的数据缓存（可选，可以避免重新下载）
uv run ap data import-qtc --from ../qtc/data/cache

# 4. 离线跑一次研究（FakeLLM，不需要 API key）
AP_OFFLINE=1 uv run ap research "低波动+动量反转" --llm fake --interactive

# 5. 用 DeepSeek 跑真实研究
export DEEPSEEK_API_KEY=sk-...
uv run ap research "低波动+动量反转"

# 6. 启动服务和控制台
uv run ap serve & uv run ap worker & pnpm -C web dev

# 7. 在 Claude Code 中接入 MCP
claude mcp add alphapilot -- uv run ap mcp serve --profile research
```

## 文档导航

| 文档 | 内容 |
|---|---|
| [设计规格书](docs/superpowers/specs/2026-11-08-alphapilot-design.md) | **唯一权威设计来源** |
| [ROADMAP](ROADMAP.md) | 里程碑 M0–M8 与进度 |
| [架构](docs/architecture.md) | C4 视图、时序图、部署 |
| [诚实性宪章](docs/honesty-charter.md) | 8 条强制执行的诚实性原则 |
| [术语表](docs/glossary.md) | 金融、统计、工程术语 |
| [ADR](docs/adr/README.md) | 8 条架构决策记录 |
| 详细设计 | [量化内核](docs/design/quant-core.md) · [数据模型](docs/design/data-model.md) · [Agent](docs/design/agents.md) · [工具与 MCP](docs/design/tools-and-mcp.md) · [LLM 路由](docs/design/llm-router.md) · [API](docs/design/api.md) · [前端](docs/design/frontend.md) · [测试与评测](docs/design/testing-and-evals.md) |

## 技术栈

**后端**：Python 3.12 · uv · LangGraph · MCP（mcp 2.x `MCPServer`）· FastAPI · SQLAlchemy 2 · Alembic · Postgres 16 · DuckDB / Parquet · APScheduler · structlog · Typer
**LLM**：DeepSeek V4（openai SDK）· Claude（anthropic SDK）· Ollama
**量化**：pandas · numpy · scipy · statsmodels · scikit-learn · AKShare
**前端**：React 19 · Vite · TypeScript · TanStack Query · Tailwind · shadcn/ui · ECharts
**工程**：Docker Compose · GitHub Actions · ruff · mypy · pytest · Playwright

## 项目来源

AlphaPilot 是 [qtc 量化交易课程](https://github.com/Perlou/qtc)（26 周、14 阶段、138 课）的结业项目。它迁移并重构了课程中的数据层、因子 DSL、统计工具、mini_quant 研究流水线和 mini_trader 交易系统。

## 免责声明

本项目仅用于学习和研究，**不构成任何投资建议**。实盘接口默认关闭；使用真实资金的风险由使用者自行承担。

## License

待定（建议 MIT，确认后补充 `LICENSE` 文件）。
