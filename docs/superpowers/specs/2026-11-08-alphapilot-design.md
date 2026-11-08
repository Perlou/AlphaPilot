# AlphaPilot 设计规格书（Design Spec）

> 状态：**已批准方向，待评审细节** · 版本 v1.2 · 日期 2026-11-08
> v1.2：处理 spec 评审提出的 8 个阻断问题（信任边界、受控泄漏、审批契约、事件 schema、trial 身份、命名统一、本地身份），见 §13。
> v1.1：核心模型改为 DeepSeek V4；按详细设计反馈修订（新增 services 层、trial 幂等、holdout 判定、补齐字段和端点、MCPServer 改名等，见 §13 变更记录）
> 本文是 AlphaPilot 的**唯一权威设计来源**（single source of truth）。`docs/design/*.md` 是对本文各章节的展开，冲突时以本文为准，并应回头修订本文。

---

## 0. 一句话

**AlphaPilot（阿尔法领航员）是一个由 AI Agent 驱动、但由人类把关的 A 股量化研究与交易平台。** 它的定位是 **Agent 当副驾驶做研究，人当机长做决策**：Agent 负责提出因子假设、写因子表达式、跑检验、写研究报告；每一个会影响资金的决定都要经过人工审批；每一个数字都由代码计算并且可以追溯。

## 1. 背景与目标

### 1.1 背景

- 作者已学完 qtc 量化课程（26 周、14 阶段、138 课），手上有一批可运行的组件：
  - 数据层：缓存加多源回退，Sina 等比复权；
  - 安全因子 DSL；
  - 多重检验工具：Holm、BH、Deflated Sharpe；
  - mini_quant 分层研究流水线；
  - mini_trader 事件驱动模拟交易系统；
  - 一个手写的 ReAct 研究 Agent 原型（qtc `phase-10/09`）。
- 这些组件分散在各个课程目录里，靠 `sys.path` 拼接起来，没有服务化，没有持久化，也没有 UI，不能当作“作品”展示。
- 2025–2026 年 AI Agent 是行业热点，但大量“LLM 炒股”项目有一个根本缺陷：**LLM 的训练数据覆盖了回测区间，回测结果天然被污染**。另外，它们普遍不做多重检验，凭空编造数字也很常见。

### 1.2 目标

| # | 目标 | 衡量方式 |
|---|---|---|
| G1 | 把课程成果整合成一个**可独立部署的企业级系统** | `docker compose up` 一键启动；有 CI；测试覆盖核心模块 |
| G2 | 用**主流 Agent 技术栈**实现研究自动化 | LangGraph 状态图，审批时 interrupt，用 checkpoint 恢复；工具通过 MCP 暴露，Claude Code 可以直接调用 |
| G3 | **科学诚实**是第一原则 | 零信号面板上的错误发现率（FDR）不超过阈值；护栏违规为 0；报告里的数字 100% 可溯源 |
| G4 | 打通“研究 → 生产 → 模拟盘 → 监控”的闭环 | 已批准的因子自动进入每日流水线，跑出模拟盘 NAV，监控 Agent 生成日报 |
| G5 | 可观测、可复现、可审计 | 每次运行都有 manifest；Agent 全量 trace；审计日志用哈希链防篡改 |

### 1.3 非目标（Non-goals）

- **不做** LLM 直接给出买卖决策。理由见 ADR-0004。
- **不做**真实资金下单。实盘只预留 vnpy 网关接口（CTP 或 XT），默认关闭。
- **不做**多租户 SaaS。系统是单团队部署，用角色区分研究员、审批人和管理员。
- **不做**高频或日内交易。频率是日频数据加月度或周度调仓。
- 首期**不做**新闻情绪和研报 RAG。课程 Phase 10 的 06/07 课留作 M8 扩展。

## 2. 核心概念

| 概念 | 定义 |
|---|---|
| **Hypothesis（假设）** | 一条带经济学解释的选股思路，例如“低波动股票未来超额收益更高，因为存在彩票偏好”。由 planner 生成 |
| **Factor Expression（因子表达式）** | 用 AlphaPilot DSL 写成的截面因子公式，例如 `-ts_std(ret, 20)`。AST 白名单校验，禁止 eval |
| **Trial（试验）** | 一次**接触了数据**的因子评估。trials 表只能追加（append-only），项目内全局累计，永不清零。**多重检验的 N 等于全局 IS trial 总数**，`factor.evaluate` 和 `backtest.run` 都计入，各个 run、外部 MCP 会话和失败的 trial 也都计入，口径偏保守。HOLDOUT trial 单独在本 run 内做校正 |
| **Research Run（研究任务）** | 一次 Research Graph 的执行。有预算（trial 次数、LLM 美元）、有数据快照、可以暂停和恢复 |
| **Sample Split（样本切分）** | 分为 IS（样本内，Agent 可见）、embargo（隔离带）、HOLDOUT（留出集，需要审批才能使用，每个表达式只能用一次）、RESERVED（holdout 结束到因子批准之间的数据，研究阶段**任何人都不能使用**，避免“看过之后再挑”）、FORWARD（批准之后的真实时间，唯一的真样本外）。切分方案带版本号（`split_id`），每条 trial 都记录它；数据刷新后不会自动滑动窗口，需要新建切分版本，旧版本的 trial 仍计入全局 N |
| **Approval（审批）** | 人工决策记录，有五种：holdout（允许看留出集）、register（入库）、promote（上生产）、retire（下线）、ops_action（运维动作） |
| **Factor Lifecycle（因子生命周期）** | `draft → candidate → approved → production → retired`；candidate 在 register 被拒时进入终态 `rejected`。每一步迁移都有记录 |
| **Strategy Version（策略版本）** | 一组生产因子加组合配置的不可变快照，每日流水线按版本运行 |
| **Evidence Level（证据等级）** | 每个结论的可信度标签：`IS` < `PSEUDO_OOS`（模型知识截止日之前的留出集）< `TRUE_OOS`（截止日之后且在批准之后） |
| **Data Snapshot（数据快照）** | 某一时刻数据湖的内容哈希、日期范围、是否为合成数据、已知偏差。每次运行都绑定一个快照 |

## 3. 总体架构

```
                ┌──────────────── Web 控制台 (React+Vite+TS) ────────────────┐
                │ Dashboard · Runs · 审批中心 · 因子库 · 回测 · 模拟盘 · 告警日报 │
                └───────────────▲──────────── REST + SSE ────────────────────┘
 Claude Code / 其他 MCP 客户端    │
        │ MCP (stdio / streamable-http)
        ▼                       │
 ┌─────────────┐        ┌───────┴────────┐       ┌──────────────────────────┐
 │ MCP Server  │        │  API (FastAPI) │──────▶│ Postgres（唯一状态存储）    │
 │ research/ops│        │  auth · SSE     │◀──────│ runs/trials/factors/      │
 └──────┬──────┘        └───────┬────────┘ LISTEN│ approvals/jobs/events/    │
        │                       │ enqueue  NOTIFY│ paper/audit + LangGraph ckpt│
        ▼                       ▼                └──────────▲───────────────┘
 ┌──────────────────────────────────────────────────┐         │
 │ Worker（jobs 队列消费者 + APScheduler 调度）         │─────────┘
 │  ├─ Research Graph (LangGraph)  ──┐               │
 │  ├─ Monitor Graph  (LangGraph)  ──┤ ToolGateway   │
 │  ├─ Daily Pipeline              ──┤               │
 │  └─ Data Update                   ▼               │
 │        tools/（带护栏的工具服务层）                   │
 │          │            │             │             │
 │       quant/       data/        trading/          │
 │   DSL·统计·因子·    数据湖·        事件引擎·OMS·     │
 │   模型·组合·回测    复权·质量      风控·模拟网关 ──┐  │
 └──────────────────────────────────────────────┼───┘
        LLM Router ── openai SDK: DeepSeek V4(主) · Ollama │ anthropic SDK: Claude(备) │ FakeLLM
                                                 ▼
                                  [可选] vnpy 适配器（paper_account；CTP/XT 预留）
```

**分层原则**

1. **内核层**：`quant/`、`data/`、`trading/` 都是纯 Python，不依赖 LLM、Web 和数据库，可以单独测试。这部分迁移自 qtc。
2. **工具层**：`tools/` 是**唯一**能触碰研究数据的入口，所有护栏都在这一层。Agent 和 MCP 客户端走同一套工具，享有同样的权限，也受同样的约束。
3. **编排层**：`agents/` 用 LangGraph 编排图；`pipeline/` 是确定性的每日流水线。
4. **应用服务层**：`services/` 负责跨表事务逻辑，包括审批决策、创建 run、promote/retire、kill-switch，并负责幂等、审计和 self_approved 判定。API、worker 和 CLI 离线模式共用这一层。**`services.approvals.decide()` 是唯一的审批入口**，只对已认证的人类主体开放；tools、agents、mcp 都不能调用它。
5. **接入层**：`api/`、`worker/`、`mcp/`、`cli/` 都是薄适配层，不放业务逻辑。
6. **持久层**：`db/` 用 Postgres 存全部状态；市场数据放 Parquet 数据湖，用 DuckDB 查询。

## 4. 关键设计决策（摘要，详见 ADR）

| ADR | 决策 | 一句话理由 |
|---|---|---|
| 0001 | 用 Postgres 承担队列、事件推送和 checkpoint，**不引入 Redis** | 单团队低吞吐；一个数据库内事务一致；少一个组件。用 `EventBus` 接口预留扩展 |
| 0002 | 用 **LangGraph** 编排，工具通过 **MCP** 暴露 | 主流技术栈；interrupt 和 checkpoint 天然适合 HITL；MCP 让工具可被外部 Agent 复用 |
| 0003 | **借鉴 vnpy 设计并提供可选适配器**，不以 vnpy 为底座 | vnpy 是 Qt 桌面程序、平台受限，且没有 LLM 能力；它的 Gateway/App 抽象和 alpha lab 值得学习 |
| 0004 | **Agent 不下单**，只产出可验证的研究成果 | LLM 训练数据覆盖回测区间，回测不可信；把 LLM 限定在“提想法、写代码、写文字”，其余交给确定性统计 |
| 0005 | LangGraph 只做编排，**模型调用走官方 SDK**（openai / anthropic），不用 LangChain ChatModel 封装 | DeepSeek 的 `thinking`、`reasoning_content` 回传、strict beta 端点，以及 Claude 的 adaptive thinking 和 `output_config` 都是各家特有的参数，封装层容易跟不上，还会掩盖细节 |
| 0006 | qtc 代码**迁移重构**进本仓库，不跨仓库 import | 项目要能独立部署；借迁移去掉 sys.path hack，统一接口 |
| 0007 | 护栏放在**工具层**，不靠 prompt | prompt 可以被绕过，工具层的约束是硬性的；同时对 MCP 外部客户端生效 |
| 0008 | **DeepSeek V4 作为核心模型**（Pro 做主力，Flash 做轻量），Claude 作为备选和回退 | 成本低一个数量级；境内访问稳定；1M 上下文；支持 tool call 和 thinking；代价是结构化输出较弱，用校验和重试弥补 |

## 5. 模块设计

### 5.1 目录结构

```
AlphaPilot/
├─ pyproject.toml  uv.lock  Makefile  .env.example  README.md  ROADMAP.md  CLAUDE.md
├─ src/alphapilot/
│  ├─ core/      settings · logging · errors · clock · ids · manifest
│  ├─ data/      sources/ · adjust · lake · panel · quality · snapshot · universe
│  ├─ quant/     rules · dsl/ · stats/ · factors/ · models/ · portfolio/ · backtest/ · research/
│  ├─ trading/   models · contracts · engine · events · gateway · oms · positions · risk · persistence · recovery · sessions · monitor · system · mock_exchange · gateways/
│  ├─ pipeline/  daily · executor · health · strategy_version
│  ├─ services/  approvals · runs · factors · killswitch（应用服务：跨表事务、幂等、审计、self_approved 判定）
│  ├─ tools/     context · research · registry · trade · monitor · guards
│  ├─ mcp/       server
│  ├─ llm/       router · providers/ · fake · cache · accounting · schemas · prompts/
│  ├─ agents/    research/ · monitor/ · common/
│  ├─ db/        models · repositories · migrations/
│  ├─ api/       app · routers/ · sse · auth · schemas
│  ├─ worker/    jobs · scheduler · handlers
│  ├─ evals/     datasets · metrics · runner
│  └─ cli/       main（Typer：ap）
├─ tests/        unit · golden · agent · integration · evals
├─ configs/      strategy.yaml · llm.yaml · research.yaml · universe.yaml
├─ web/          React + Vite + TS
├─ deploy/       compose.yaml · Dockerfile.api · Dockerfile.web · nginx.conf
└─ docs/         architecture · honesty-charter · glossary · design/ · adr/ · superpowers/specs/
```

### 5.2 内核层（迁移自 qtc，详见 `docs/design/quant-core.md`）

- **data**：
  - 数据源依次是本地缓存、东财、新浪，最后回退到合成数据；合成数据通过 `attrs["synthetic"]` 标记。
  - 股票收益**只用 Sina 等比复权因子**。东财的 qfq/hfq 是加减法复权，除息日收益会失真。
  - 数据湖布局：`data/lake/{daily,adj_factor,index,universe,meta}/`，Parquet **按 symbol 分目录、目录内按年分片**，文件按内容寻址，保证快照不可变；DuckDB 建视图（详见 quant-core.md）。
  - `snapshot.create()` 计算内容哈希并写入 `data_snapshots` 表。
  - 股票池：M1 沿用课程面板（132 只，数据覆盖 2018-01 至 2025-02-28；qtc 09 课的切分是 IS 2019-07 至 2022-12，HOLDOUT 2023-01 至 2025-02-14）；M6 扩到沪深 300 当前成分股，数据刷新到最新交易日，并在 `known_biases` 中注明幸存者偏差。
- **quant**：
  - `rules`：A 股规则，包括板块涨跌停（创业板 30 开头）、主板 ST 自 2026-07-06 起 ±10%、一手 100 股、印花税和佣金。
  - `dsl`：校验、求值、`lookahead_test`、`canonical_hash`（AST 规范化去重）、`compile_to_factor`。
  - `stats`：IC 统计、绩效、Holm/BH、PSR/DSR、`expected_max_sr`。
  - `factors`：注册表、内置因子、预处理（MAD、z-score、行业和市值中性化）、诊断。
  - `models`、`portfolio`（TopN、换手约束、手数规划）、`backtest`（月度回测、容量）。
  - `research/splits`：定义 IS、embargo、HOLDOUT、FORWARD 的切分。
- **trading**：
  - asyncio 事件引擎、OMS 状态机、PositionBook（T+1 冻结）、RiskEngine、事件日志与恢复对账。
  - `gateways/paper.py` 是默认网关，包装 MockExchange；`gateways/vnpy_bridge.py` 是可选的 vnpy 适配器。
- **pipeline**：
  - `DailyPipeline` 依次执行：数据 → 因子 → 模型 → 组合 → 订单 → 执行，任一步失败即停止。
  - 按 `strategy_version` 运行，`replay` 和 `forward` 两种模式。

### 5.3 工具层（详见 `docs/design/tools-and-mcp.md`）

所有工具都是 `def tool(ctx: ToolContext, **params) -> ToolResult`。

`ToolContext` 包含：`run_id`、`actor`（agent、user 或 mcp-client）、`profile`、`snapshot_id`、`split`、`budget`（trial 预算和 LLM 预算）、`ledger`（TrialLedger 接口）、`clock`。

**Research profile 工具**

| 工具 | 是否计 trial | 说明 |
|---|---|---|
| `data.describe` | 否 | 字段表、股票池规模、IS 区间长度。**不返回具体日期和股票代码** |
| `data.query` | 否 | 只返回 IS 区间内的**聚合统计**，比如截面分布、行业均值；不返回逐日、逐股明细 |
| `factor.validate` | 否 | DSL 校验，返回 canonical_hash 和是否重复 |
| `factor.evaluate` | **是** | IS 区间的 IC、ICIR、t、p、前后半段 IC、覆盖率、前视检测、与生产因子的最大相关；写入 trial |
| `factor.correlate` | 否 | 与已有因子（生产因子加本 run 的候选）的相关矩阵，只用 IS 数据 |
| `backtest.run` | **是** | 只能对已有 trial 的表达式运行；IS 区间的 TopN 和分层回测，扣除成本 |
| `stats.correct` | 否 | 从 ledger **读取**全局 trial，计算 Holm、BH、DSR。不接受调用方传入的 p 值 |
| `research.trials` | 否 | 查询历史 IS trial（表达式和 IS 指标），用于去重。**holdout 结果对外永远不可见** |
| `registry.list` | 否 | 对 Agent 和 MCP 只返回 **production 因子和基线因子（Alpha158 子集）的名称与表达式**，用于去重；**不返回**状态区分、来源 trial 或任何 holdout 指标（受控泄漏，见 §8.3） |
| `registry.propose` | 否 | 把候选因子提交为 `candidate`，并创建一条 register 审批 |

**MCP 工具命名**：MCP 规范允许工具名中带点号，但 Claude 和 OpenAI 的工具名不允许。因此**内部统一用点号**（`data.describe`），**通过 MCP 和 LLM tool 对外暴露时一律换成下划线**（`data_describe`），由 `mcp/server.py` 和 router 负责映射。

**Ops profile 工具**：`trade.state`、`trade.orders`、`monitor.alerts`、`monitor.factor_health`、`report.verify_numbers`。

**仅限图内部使用的工具**：`holdout.evaluate`（必须带已批准的 approval_id）、`registry.register`（审批通过后由 handler 调用）。

**护栏（`tools/guards.py`）**，违反时抛 `GuardrailError`，同时写 `agent_events` 和审计日志：

1. 数据截止：IS 工具读取的数据不超过 `split.is_end`，越界时直接截断，并记录一条警告事件。
2. 预算：run 级 trial 预算用完抛 `BudgetExceeded`；项目级每日 trial 上限可配置。
3. Holdout：必须带已批准的 approval_id；同一个 canonical_hash，或与已测表达式相关性超过 0.95 的，一律拒绝。
4. 审批不可由工具完成：任何工具都**不能**把 approval 设为 approved。
5. 输出脱敏：工具返回内容中不出现 holdout 区间的日期、股票代码和收益。

### 5.4 Agent 层（详见 `docs/design/agents.md`）

#### Research Graph

```
START → plan → code → validate ──(invalid & retry<2)──▶ code
                         │(valid & unique)
                         ▼
                     evaluate → critic ──(有预算且没有可审候选)──▶ code | plan
                                  │(有可审候选：候选就绪，或预算耗尽但已有候选)
                                  │                    └─(预算耗尽或假设用完，且没有可审候选)──▶ report（阴性结论）
                                  ▼
                          gate_holdout ⟂ interrupt(人选 ≤ holdout_limit 个候选；可以全部拒绝)
                                  ▼
                     holdout → backtest → report → faithfulness ──(fail & 未重写)──▶ report
                                                         │(pass)
                                                         ▼
                                          gate_register ⟂ interrupt → register → END
```

| 节点 | 执行者 | 输入 → 输出 |
|---|---|---|
| plan | LLM（planner），**有限 ReAct**：可以调用只读工具 `data.describe`、`research.trials`、`registry.list`，最多 6 次 | goal → K 个 Hypothesis（statement、rationale、family、expected_sign） |
| code | LLM（coder），结构化输出 | Hypothesis（加上一次的校验错误）→ DSL 表达式 |
| validate | 确定性代码 | 表达式 → valid、canonical_hash、是否重复 |
| evaluate | 工具 `factor.evaluate` | 表达式 → trial_id 和 IS 指标 |
| critic | **确定性代码**；LLM（critic）只写评语 | 全部 trial → candidates（p_holm、q_bh、DSR、hard_fail、warnings）以及下一步的路由 |
| gate_holdout | `interrupt()` | 候选卡片 → 人的选择（approval_id） |
| holdout | 内部工具 | 已批准的候选 → HOLDOUT 指标；evidence_level 记为 `PSEUDO_OOS` |
| backtest | 工具 | 候选 → IS 和 HOLDOUT 回测（成本后） |
| report | LLM（reporter），只写文字和占位符 | 结构化事实 → Markdown 报告（数字写成 `{{trial:42.icir}}` 这样的占位符） |
| faithfulness | 确定性代码 | 报告 → 渲染结果加问题列表（占位符无法解析，或出现无来源的数字） |
| gate_register | `interrupt()` | 报告加候选 → 人的决定 |
| register | 工具 | 已批准的候选 → factors(approved) |

**Critic 的硬性否决（hard_fail，审批中心会禁用“批准”按钮）**

- `lookahead_ok == False`
- 覆盖率低于 0.6
- 与 production 或 approved 因子、或其他候选之间的最大 |相关| 超过 0.7
- IS 区间 `p_holm ≥ 0.05` 且 `q_bh ≥ 0.10`

**软性警告**：前后半段 IC 符号相反；DSR < 0.95；换手过高；只在单一行业有效。

**State**（`TypedDict`；大对象只存 id，明细放数据库）：

```python
class ResearchState(TypedDict):
    run_id: str
    goal: str
    cfg: ResearchCfg                     # splits, horizon, k_hypotheses, budget, holdout_limit, snapshot_id
    hypotheses: list[Hypothesis]
    cursor: int                          # 当前处理的 hypothesis
    draft: DraftExpr | None              # 当前表达式和修复次数
    trial_ids: Annotated[list[int], operator.add]
    candidates: list[Candidate]
    budget_used: int
    llm_cost_usd: float
    approvals: dict[str, str]            # gate 名 → approval_id
    holdout_trial_ids: list[int]
    backtest_ids: list[str]
    report_md: str | None
    faithfulness_issues: list[str]
    route: str | None                    # critic 的路由结论（R0–R5），用于审计
    outcome: str | None                  # registered | holdout_declined | holdout_failed（全部 reversed）| register_declined | negative_budget | negative_exhausted | llm_unavailable
    status: RunStatus
    errors: Annotated[list[str], operator.add]
```

**路由规则**（critic 按 R0–R5 判定，详见 `docs/design/agents.md`）：
- 预算耗尽或假设用完时：**有可审候选就进 gate_holdout，没有就进 report**，写阴性结论报告（“本次研究未发现通过多重检验的因子”）。**得出阴性结论也算研究成功。**
- validate 修复用尽：放弃当前表达式，转到下一个假设。
- 审批人在 gate_holdout 拒绝全部候选：跳过 holdout 和 backtest，直接写报告（`outcome=holdout_declined`）。
- holdout 之后**不再回到 plan 或 code**，以免根据 holdout 结果修改表达式。
- faithfulness 连续两次不通过：改用代码生成的兜底报告。

**Holdout 判定（确定性）**：在本 run 送检的 holdout 集合内做 Holm 校正。
- `consistent`：IS 与 HOLDOUT 的 IC 同号，且 holdout 的 p < 0.05；
- `weak`：同号但不显著；
- `reversed`：异号。

只有 holdout 判定不是 `reversed`、并且 IS 没有 hard_fail 的候选，才能进入 gate_register。`weak` 的候选在审批卡片上会有显著警告。

**审批决策契约（唯一口径）**
- API 输入：`decision: "approve" | "reject"`、`reason`（必填）、`selected_candidate_keys: list[str]`（候选 key 是 canonical_hash 的前 12 位；holdout 审批批准时必填）、`names: dict[key, str] | None`（只用于 register；默认取 `registry.propose` 时给出的建议名，审批人可以改）。
- 存储状态：`approvals.status ∈ {pending, approved, rejected, expired}`。
- Resume payload：`decision ∈ {"approve", "reject", "expire"}`。
- **过期**：scheduler 每 10 分钟扫描一次，默认 7 天过期。过期的审批状态置为 `expired`，同样投递 resume（`expire`），图按拒绝分支处理（`holdout_declined` / `register_declined`），run 最终为 `completed`，不会是 failed。
- **register 被拒**：因子进入终态 `rejected`（写 `factor_events(register_rejected)`），不再占用 live hash 唯一索引，以后仍可以重新 propose。

#### Monitor Graph

```
START → collect → diagnose → propose → report → faithfulness ─(fail & 未重写)→ report ─→ END
```

- **collect**（确定性）：NAV、持仓、对账结果、`build_alerts`、因子 IC 衰减、PSI 漂移、行业暴露。
- **diagnose**（LLM，monitor 角色）：对每条告警给出可能原因，必须引用证据 id。
- **propose**（LLM 加 schema）：从 Agent 白名单 `pause_strategy | retire_factor | rerun_data_update | rerun_pipeline | reduce_gross | no_action` 中选择建议动作，写成 `approvals(kind=ops_action)`。**图本身不执行任何动作。**`release_kill_switch` 只能由 admin 发起，不在 Agent 白名单中。
- P1 告警，例如对账不一致或单日亏损超限，由 `trading/risk` 中的确定性 kill-switch **直接**暂停交易，不经过 LLM；Agent 也无法解除，解除需要 admin 审批并且对账通过。

### 5.5 LLM Router（详见 `docs/design/llm-router.md`）

**核心模型是 DeepSeek V4 系列**（ADR-0008）。Claude 作为可选的备选和回退，本地 Ollama 用于离线开发，FakeLLM 用于测试。以下事实已于 2026-11-08 按 DeepSeek 官方文档核对：

| 模型 ID | 实际版本 | 上下文 / 最大输出 | 定位 |
|---|---|---|---|
| `deepseek-v4-pro` | DeepSeek-V4-Pro-0813 | 1M / 384K | **主力**：假设生成、报告撰写等高质量推理 |
| `deepseek-flash` | DeepSeek-V4.1-Flash-1017 | 1M / 384K | **轻量**：DSL 翻译、评语、监控诊断；价格约为 Pro 的 1/4 到 1/3 |

> 旧名 `deepseek-chat` 和 `deepseek-reasoner` 已经下线，配置中禁止使用。`deepseek-v4-flash` 是遗留名，会被路由到 Flash，不建议使用。

**角色与模型**（`configs/llm.yaml`，可以按 run 覆盖）：

| 角色 | 默认链（按顺序回退） | 推理设置 | 理由 |
|---|---|---|---|
| planner | `deepseek:deepseek-v4-pro` → `anthropic:claude-opus-5` → `ollama:qwen3.5` | thinking 开启，effort `high` | 需要创造性和金融常识 |
| coder | `deepseek:deepseek-flash` → `deepseek:deepseek-v4-pro` → `ollama:qwen3.5` | thinking 关闭 | 结构化翻译任务，量大，需要低延迟 |
| critic | `deepseek:deepseek-flash` | thinking 关闭 | 只写评语，判定是确定性的 |
| reporter | `deepseek:deepseek-v4-pro` → `anthropic:claude-opus-5` | thinking 开启，effort `high` | 长文写作，要求严谨 |
| monitor | `deepseek:deepseek-flash` → `ollama:qwen3.5` | thinking 开启，effort `high` | 诊断推理 |
| fake | `fake:scripted` | — | 测试和 CI；按 `(role, step)` 读取 YAML 脚本 |

**Provider 实现**（ADR-0005：模型调用直接用官方 SDK，不用 LangChain 封装）

- **deepseek**（主 provider）：官方 `openai` SDK，`base_url=https://api.deepseek.com`，密钥从 `DEEPSEEK_API_KEY` 读取。
  - **thinking**：通过 `extra_body={"thinking": {"type": "enabled"|"disabled"}}` 开关，`reasoning_effort` 取 `low`、`high` 或 `max`。thinking 模式下 `temperature` 等采样参数**不生效**，因此不传。
  - **结构化输出**：DeepSeek 只支持 `response_format={"type": "json_object"}`，**不支持 json_schema**。具体做法：
    1. prompt 里必须出现 "json" 字样，并附上 schema 和示例；
    2. 用 Pydantic 校验返回结果；
    3. 返回空白内容时重试（官方说明过这种情况）；
    4. 校验失败时修复重试 1 次。

    作为加强方案，可以改用 `strict` 模式的 tool call，让模型以函数参数的形式提交结果（Beta，需要 `base_url=https://api.deepseek.com/beta`，所有 function 都设 `strict: true`）。M3 时实测两种方案，选择合法率更高的一种。
  - **Tool loop**：thinking 模式下只要请求带了 `tools`，**后续每一轮都必须把历史的 `reasoning_content` 完整传回**，否则返回 400。这个处理由 router 统一负责，节点代码不需要关心。
  - **计费**：价格**分高峰和非高峰时段**。高峰是 UTC 周一至周五 01:00–04:00 和 06:00–10:00（中国法定节假日除外），其余时段半价。缓存命中和未命中分开计价。`llm/prices.py` 按调用时间戳计算费用。
  - **网络**：DeepSeek 是境内服务，`api.deepseek.com` 加入 `NO_PROXY`。
- **anthropic**（备选和回退）：官方 `anthropic` SDK。
  - 结构化输出用 `client.messages.parse()` 或 `output_config.format`；工具设 `strict: true`，`tool_choice` 用 `auto`。
  - thinking 用 adaptive，通过 `output_config.effort` 控制深度。
  - 不传 `temperature`（Opus 5 和 Sonnet 5 已经移除）。
  - 处理 `stop_reason == "refusal"`。
- **ollama**：`openai` SDK 加 `base_url=http://localhost:11434/v1`。本机已有 qwen3.5，用于离线开发和联调。结构化输出采用 json_object 加校验。
- **fake**：确定性的脚本回放，可以注入错误，比如非法 DSL、空白 JSON、编造数字、超时、429。

**通用能力**

- `complete_structured(role, messages, schema) -> (obj, CallMeta)`：统一处理 schema 注入、JSON 解析、空白重试和修复重试。
- `run_tool_loop(role, messages, tools, max_steps)`：给 plan 节点用的有限 ReAct 循环，统一不同 provider 的 tool 调用格式，并负责 `reasoning_content` 的回传。
- **回退触发条件**：429、5xx、超时、refusal、修复后仍无法解析。每次回退都记一条事件。
- **缓存**：键为 `sha256(provider, model, messages, schema, params)`。`AP_LLM_CACHE=readwrite|readonly|off`；eval 和回放必须用 `readonly`。（这是 AlphaPilot 自己的结果缓存，和 DeepSeek 服务端的 KV 前缀缓存是两回事。后者只影响计价；为了提高命中率，system prompt 和工具定义要保持字节级稳定。）
- **可复现性**：thinking 模式不支持 temperature，**可复现靠缓存回放实现，不靠采样参数**。
- **计费与预算**：每次调用写入 `llm_calls`（run_id、node、role、provider、model、输入 token（区分缓存命中与未命中）、输出 token、推理 token、cost_usd、latency_ms、cache_hit、fallback_from、peak）。run 级设美元上限，超过后抛 `BudgetExceeded`。
- **knowledge_cutoff**：每个模型在配置中登记知识截止日，用于计算 evidence_level（见 §8）。**官方未公布截止日时，保守地取模型发布日期**：`deepseek-v4-pro` 取 2026-08-13，`deepseek-flash` 取 2026-10-17。

### 5.6 持久化（详见 `docs/design/data-model.md`）

只有 Postgres 16 一个状态存储；用 Alembic 管理迁移；LangGraph checkpoint 由 `PostgresSaver.setup()` 建表，放在 `lg` schema。

| 表 | 要点 |
|---|---|
| `research_runs` | 状态有 queued、running、awaiting_approval、completed、failed、cancelled；kind 为 `research` 或 `interactive`（外部 MCP 会话）；记录 created_by、thread_id、snapshot_id、预算、花费、cancel_requested、outcome |
| `hypotheses` | run_id、statement、rationale、family、model |
| `trials` | **只能追加**（trigger 禁止 UPDATE 和 DELETE）；记录 window（IS 或 HOLDOUT）、expr、canonical_hash、各项指标、lookahead_ok、snapshot_id、**split_id、horizon、sample_every**、evidence_level。IS 指标复用（查重）的键是 `(canonical_hash, snapshot_id, split_id, horizon, sample_every)`。研究默认 `horizon=5`、`sample_every=5` 个交易日；生产调仓频率在 `strategy.yaml` 中单独配置（默认月度）。**`idempotency_key` 唯一**（例如 `run_id:node:canonical_hash:window`）：worker 崩溃后重跑节点不会重复写入 trial，避免全局 N 虚增 |
| `factors` / `factor_events` | 生命周期状态机加迁移历史（含 `rejected` 终态；live hash 唯一索引不包括 rejected 和 retired）；origin_trial_id 指向来源 trial |
| `approvals` | kind、subject、payload、status（pending、approved、rejected、expired）、requested_by、decided_by、reason（必填）、self_approved、expires_at、**version**（乐观锁，配合 If-Match） |
| `strategy_versions` | 因子集合、配置 YAML、effective_from、approval_id；不可变 |
| `backtests` | 配置、指标、NAV 的 Parquet 路径、manifest |
| `data_snapshots` | 内容哈希、日期范围、is_synthetic、known_biases |
| `agent_events` | 自增 id、run_id、node、type（node_start、node_end、llm、tool、interrupt、guardrail、error）、payload。**每种事件 payload 的 schema 只有一个来源：`alphapilot/core/events.py`**（Pydantic），字段以 agents.md §0.3 为准；API、SSE 和前端类型都从它生成；既是 trace，也是 SSE 的数据源。SSE 中的 `status`、`heartbeat` 是**不入库的瞬时事件**，不带 id，断线重连时用状态快照补齐 |
| `llm_calls` / `llm_cache` | 计费记录和缓存 |
| `jobs` | kind、payload、status（`queued | running | succeeded | dead | cancelled`；重试期间保持 queued，达到上限后转为 dead，人工可以重试 dead 的 job）、run_after、attempts、unique_key、locked_by、heartbeat_at；用 `FOR UPDATE SKIP LOCKED` 领取任务；心跳超时（默认 60 秒）即视为租约过期，可以被其他 worker 回收 |
| `paper_accounts` / `orders` / `trades` / `positions_daily` / `nav_daily` / `pipeline_runs` | 模拟盘状态 |
| `alerts` / `daily_reports` | 告警（按指纹去重）；日报加 faithfulness 结果 |
| `audit_log` | actor、action、subject、before、after、prev_hash、hash；哈希链，可以校验是否被篡改 |
| `users` / `api_keys` | 角色：researcher、approver、admin |

### 5.7 服务层（详见 `docs/design/api.md`）

**API**（FastAPI，前缀 `/api/v1`）

- `POST /runs`（返回 202）；`GET /runs`、`GET /runs/{id}`、`GET /runs/{id}/trials`、`GET /runs/{id}/events`（SSE）；`POST /runs/{id}/cancel`
- `GET /approvals?status=pending`；`POST /approvals/{id}/decision`（approver 角色，`reason` 必填）
- `GET /factors`、`GET /factors/{id}`、`POST /factors/{id}/promote`、`POST /factors/{id}/retire`（这两个接口只创建审批）
- `GET /backtests/{id}`、`GET /paper/{acct}/nav|positions|orders`、`GET /alerts`、`GET /reports/daily/{date}`
- `GET /paper/{acct}/trades|reconciliation`、`GET /notifications/stream`（全局 SSE：待审批和告警）
- kill-switch：`GET /ops/kill-switch`、`POST /ops/kill-switch/engage`（admin）、`POST /ops/kill-switch/release-request`（创建 release 审批，要求对账通过）
- `POST /auth/login|refresh|logout`、`GET /auth/me`、`/me/api-keys`、`/admin/users`、`POST /admin/jobs`（手动投递）、`POST /admin/schedules/daily/trigger`
- `GET /llm/usage`、`GET /snapshots`
- 运维端点挂在**根路径**：`GET /healthz`、`GET /readyz`、`GET /metrics`（nginx 对外屏蔽 `/metrics`）

**SSE**：`agent_events` 写入时触发 `NOTIFY run_events`。API 用一个 LISTEN 连接按 run_id 分发；断线重连时依据 `Last-Event-ID` 从表里补发。

**Worker**：

- 任务种类：`research.start`、`research.resume`、`data.update`、`pipeline.daily`、`monitor.daily`、`eval.run`。
- 图遇到 interrupt 时，run 状态设为 awaiting_approval，当前任务结束。审批决定写入后，API 再投递 `research.resume`，用 `Command(resume=decision)` 恢复同一个 thread。

**Scheduler**：APScheduler 运行在 worker 内，用 `pg_advisory_lock` 保证只有一个实例生效。每个交易日 16:30（Asia/Shanghai）依次投递 `data.update`、`pipeline.daily`、`monitor.daily`，unique_key 用 `kind:date`，保证幂等。

**Auth**：Web 端使用 JWT（登录后签发，httpOnly cookie，非 GET 请求做 CSRF double-submit 校验）；CLI 和 MCP 使用 `X-API-Key`。权限按 RBAC 划分（层级关系 researcher < approver < admin）：researcher 可以创建 run；approver 可以审批；admin 可以管理用户、kill-switch 和配置。HOLDOUT 区间的 trial 只返回给人类会话，或带有 `holdout:read` scope 的 API Key；这个 scope 默认关闭，**MCP 会话使用的 key 不能授予它**。

**CLI（`ap`）**

```
ap data import-qtc|update|status|snapshot
ap backtest run -c configs/strategy.yaml [--factors builtin|production] [--allow-sealed-window]
ap research "<goal>" [--llm role=provider:model] [--budget N] [--interactive]
ap review list | show <id> | approve <id> -r "<reason>" | reject <id> -r "<reason>"
ap factor list | show <name> | promote <name> | retire <name>
ap trade paper --date D | --replay A:B
ap monitor report --date D
ap mcp serve --profile research|ops [--transport stdio|http]
ap serve | ap worker | ap eval run --suite core
ap user create <name> --role R | ap user apikey create --name N
```

**CLI 本地模式的身份**：本地模式直接调用 services，必须设置 `AP_LOCAL_USER`，它对应 users 表中的一个用户（初始化时用 `ap user create --role admin` 创建）。`decide()` 记录的 `decided_by` 就是这个用户，审计中 `channel=cli_local`。M3 阶段使用内存存储，会自动创建 `usr_local`。

**CLI 的信任边界**：
- `ap backtest run` 默认只评估 builtin 或 production 因子，数据截断到当前 split 的 `is_end`；
- 访问 HOLDOUT 或 RESERVED 区间需要 admin 身份加 `--allow-sealed-window`，同时写入 ledger（trial，`tool=cli_backtest`）和 audit_log；
- `ap trade paper --replay` 只能回放已上线的 strategy_version；
- 研究网格（`run_study` / `run_grid`）只能使用 IS 区间。

### 5.8 Web 控制台（详见 `docs/design/frontend.md`）

技术栈：React 19、Vite、TypeScript、TanStack Query、React Router、Tailwind 加 shadcn/ui、ECharts、原生 EventSource。API 类型由 `openapi-typescript` 生成。

| 页面 | 核心内容 |
|---|---|
| Dashboard | 今日流水线状态、待审批数、NAV 与基准对比、未处理告警、近 30 天 LLM 花费 |
| Research Runs | 任务列表；新建表单（目标、预算、各角色模型） |
| Run 详情 | **实时时间线**（节点、工具调用、LLM 的 token 和成本）、trial 表、预算进度、报告预览 |
| 审批中心 | 候选卡片（IS 统计、Holm/BH、DSR、相关性、前视检测、证据等级、污染警告）；有 hard_fail 时按钮禁用；理由必填 |
| 因子库 | 生命周期看板；详情页显示 IC 衰减和来源链路（因子 ← trial ← run ← hypothesis） |
| 回测报告 | NAV、回撤、分层收益、成本拆解、manifest |
| 模拟盘 | NAV 与基准、持仓、订单和成交、对账状态、kill-switch 状态 |
| 告警与日报 | 日报 Markdown；ops 建议的审批入口 |
| 设置与用量 | 模型路由、预算、API Key、用户管理（admin） |

## 6. 关键流程

### 6.1 研究任务全生命周期

```
用户: ap research "低波动+动量反转" 或 Web 新建
 → API 写 research_runs(queued) + jobs(research.start)
 → Worker 取任务 → 建 thread → 运行 Research Graph
     plan(ReAct 只读工具) → code → validate → evaluate(写 trial) → critic … 循环至候选就绪/预算耗尽
 → gate_holdout: interrupt → run=awaiting_approval → 写 approvals(holdout, pending) → SSE 推送
 → 审批人在审批中心选择候选 + 填理由 → approvals=approved → audit_log → jobs(research.resume)
 → Worker 恢复 thread → holdout → backtest → report → faithfulness
 → gate_register: interrupt → 审批 → register → factors(approved) → run=completed
```

### 6.2 因子上线与每日流水线

```
因子库: approved 因子 → 发起 promote → 审批 → 生成新 strategy_version(含该因子) → factors=production
每个交易日 16:30:
 data.update(拉数据→质量检查→新快照) → pipeline.daily(按当前 strategy_version 生成订单→paper 网关撮合→对账→nav_daily)
 → monitor.daily(Monitor Graph → 日报 + ops 建议审批)
P1 告警 → kill-switch 直接暂停（不经 LLM）
```

### 6.3 外部 Agent 接入（MCP）

```
claude mcp add alphapilot -- uv run ap mcp serve --profile research
```

Claude Code 会话会自动对应一个 `interactive` 类型的 run，拥有独立预算，所有 trial 计入全局 ledger。外部 Agent **看不到 holdout，也无法审批**。

## 7. 非功能需求

| 维度 | 要求 |
|---|---|
| 可复现 | 每次运行生成 `run_manifest`，包含配置、git commit、代码指纹、数据快照哈希、模型 ID 和 LLM 缓存键。在 readonly 缓存下，回放结果逐字节一致 |
| 可观测 | structlog JSON 日志，带 run_id、trace_id、node 字段；`/metrics` 输出 Prometheus 指标（run 数、trial 数、LLM 成本、job 延迟）；Agent trace 全部存入数据库，不依赖 LangSmith |
| 可靠 | job 失败按指数退避重试（首次执行加最多 3 次重试）；worker 被 kill 后，由租约回收，再从 checkpoint 和幂等键继续（包括 trial 的 idempotency_key）；pipeline 每一步可以安全重入；审批被拒绝时同样投递 resume，由图走拒绝分支 |
| 安全 | 密钥只从环境变量读取，绝不入库；JWT 密钥轮换；DSL 用 AST 白名单，禁止 eval；所有审批进入审计哈希链；MCP http 模式要求 API key |
| 性能 | 单个 IS 因子评估（约 300 只股票、5 年日频）不超过 2 秒；一次 run（14 个 trial）在真实 LLM 下少于 10 分钟 |
| 成本 | 以 DeepSeek V4 计价：每个 run 默认上限 0.5 美元，每天总上限 5 美元，可以配置；若回退到 Claude，同样受这两个上限约束 |
| 离线 | `AP_OFFLINE=1` 时不访问网络（数据和 LLM 都走缓存或 fake），所有 unit、agent 测试和 CI 都在这个模式下运行 |

## 8. 诚实性设计（详见 `docs/honesty-charter.md`）

1. **知识截止日污染**：`evidence_level` 的计算规则如下：
   - IS 指标一律记为 `IS`。
   - HOLDOUT 窗口**一律**记为 `PSEUDO_OOS`，另附 contamination 标签：`full`（窗口完全在最晚的 `knowledge_cutoff` 之前）、`partial`（窗口跨过截止日）、`none`（窗口完全在截止日之后，但数据在批准前已经存在，所以仍然不是 TRUE_OOS）、`unknown`（截止日未知，按 full 展示）。
   - 只有 FORWARD 区间，即批准之后的真实时间，才记为 `TRUE_OOS`。
   - 报告和审批卡片强制显示这个等级；planner 的上下文里不出现日期和股票代码。
2. **多重检验**：N 取全局 trial 总数（包括失败的和外部 MCP 产生的）；DSR 用全部 trial 的 SR 方差计算。
3. **Holdout 只能用一次**：holdout 的**指标**永远不回流到任何 planner 或 coder 的上下文。**受控泄漏（明确接受）**：因子入库要经过 holdout 判定和人工决策，所以 production 因子的存在本身就带有每个因子 1 bit 的 holdout 信息。人工决策之后的因子库视为公开知识；Agent 只能看到 production 因子和基线因子的名称与表达式。

**信任边界**：以上保证覆盖**所有系统入口**，即 Agent、MCP、API 和 CLI（见 §5.7 中 CLI 的信任边界）。直接在 Python 中 import 内核库进行计算，属于信任边界之外，系统无法阻止；但这样得到的结果在 ledger 中没有对应的 trial，**不能注册，也不能 promote**。
4. **数字幻觉**：所有数字由代码渲染占位符生成，再用 `verify_numbers` 兜底；faithfulness 不通过的报告不能提交审批。
5. **防止审批走过场**：hard_fail 时禁止批准（API 返回 409）；理由必填；审批人与发起人为同一人时**默认允许但标记 `self_approved`**（单人部署只能如此），可以通过 `approval.allow_self=false` 禁止（默认 true）；全部进入审计链。
6. **合成数据**：快照标记 `is_synthetic` 时，结果不能 promote，UI 显示红色横幅。
7. **偏差披露**：快照的 `known_biases` 自动写入报告附录，包括当前成分股导致的幸存者偏差、当前行业分类和停牌处理方式。
8. **阴性结论同样是成果**：报告模板必须包含“未通过的假设及原因”一节。

## 9. 测试与评测（详见 `docs/design/testing-and-evals.md`）

| 层 | 内容 | 运行方式 |
|---|---|---|
| unit | 迁移 qtc 测试：qtc 共 103 个测试函数、133 个用例，迁移其中 99 个函数、125 个用例，不迁移 4 个面试题函数。来源是 mini_quant、mini_trader 和 phase-01 项目，逐文件清单见 testing-and-evals.md。另外补充 DSL、统计、护栏、router 的测试 | `AP_OFFLINE=1 pytest tests/unit` |
| golden | 在 qtc 缓存数据上，AlphaPilot 的回测和 IC 结果与 qtc 原输出一致（误差 1e-8 以内） | `pytest tests/golden` |
| agent | FakeLLM 加 MemorySaver 覆盖 9 个场景：正常流程（含两次 interrupt）、DSL 修复、预算耗尽产出阴性报告、越权调用 holdout、伪造数字被拦截、DeepSeek 空白 JSON 重试、429 回退、审批人拒绝全部候选、worker 崩溃后从 checkpoint 恢复 | `pytest tests/agent` |
| integration | docker 中的 Postgres：迁移、队列并发、SSE 补发、审计哈希链、完整 API 流程 | `pytest -m integration` |
| e2e | Playwright：创建 run、审批、查看报告（compose 加 fake LLM） | `pnpm -C web e2e` |
| evals | DSL 合法率、重复率、护栏违规（必须为 0）、faithfulness 通过率、单次成本；**植入因子召回率**；**零信号面板 FDR** | `ap eval run --suite core` |

CI（GitHub Actions）依次运行：ruff、mypy（core、quant、tools 用 strict）、unit 加 golden 加 agent、带 Postgres 的 integration、web 的 lint、typecheck、vitest 和 build，最后是 eval-smoke（fake LLM 加阈值检查）。

## 10. 里程碑

| M | 主题 | 交付物 | 验收 |
|---|---|---|---|
| M0 | 脚手架与文档 | 本 spec 和全部设计文档、ADR；uv 项目、ruff/mypy/pytest、settings 和 logging、`ap` 骨架、compose（只有 pg）、CI | `make check && uv run ap --help` |
| M1 | 量化内核迁移 | data、quant、trading、pipeline 迁移完成，去掉 sys.path hack；`ap data import-qtc`、`ap data status`、`ap backtest run` | `AP_REQUIRE_GOLDEN=1 AP_OFFLINE=1 pytest tests/unit tests/golden` 全部通过（缺少 qtc 缓存时直接失败，不允许跳过）；冒烟测试：`ap data import-qtc`、`ap data status`、`ap data snapshot`、`ap backtest run` |
| M2 | 工具层与 MCP | `tools/`（内存版 ledger、RunStore 和用户映射，用来替代 M4 才有的数据库表；以及护栏）、MCP server（mcp 2.x 的 `MCPServer`，即原 FastMCP）的 research 和 ops 两个 profile | 护栏测试通过；在 MCP Inspector 和 Claude Code 中能调用 `factor.evaluate` |
| M3 | Research Graph | 图、LLM Router、FakeLLM、`ap research --interactive`（在终端审批），报告写到 `outputs/` | `pytest tests/agent` 通过；用 DeepSeek V4 跑通一次真实 run |
| M4 | 持久化与服务 | ORM、Alembic、PostgresSaver、jobs、SSE、auth、审计；`ap serve`、`ap worker`、`ap review` | `make itest` 通过；curl 走通完整研究生命周期 |
| M5 | Web 控制台 | §5.8 的全部页面 | 构建通过，Playwright 冒烟测试通过 |
| M6 | 生产闭环 | DSL 因子编译进注册表并生成 strategy_version；数据刷新和沪深 300 股票池；paper replay 和 forward；scheduler；Monitor Graph 和日报；kill-switch | replay 产出 NAV 且对账通过；kill worker 后重跑结果幂等；日报 faithfulness 通过 |
| M7 | 扩展与收尾 | vnpy 适配器（`[vnpy]` extra）；Alpha158 子集作为基线；eval 套件；runbook；mkdocs | `ap eval run --suite core` 达标；文档站可以构建 |
| M8（可选） | 另类数据 | 新闻和公告情绪（按发布时间对齐）、研报 point-in-time RAG | 另立 spec |

每个里程碑结束时：测试通过 → 代码评审 → 提交 → 暂停，由用户确认后再进入下一个。

## 11. 风险

| 风险 | 影响 | 缓解 |
|---|---|---|
| LLM 提出的因子与已知因子高度重复 | 研究价值低 | canonical_hash 去重；与生产因子、Alpha158 基线算相关；报告标注新颖度 |
| DeepSeek 只支持 json_object（无 json_schema），偶尔返回空白内容；Ollama 也不稳定 | 流程中断 | prompt 内嵌 schema 和示例；Pydantic 校验；空白重试和修复重试；可切到 strict tool call；按角色设回退链到 Claude；每个模型单独跑 eval，指标是 DSL 合法率 |
| DeepSeek 未公布知识截止日 | evidence_level 可能定得过于乐观 | 保守地取发布日期作为截止日上界；官方公布后更新配置 |
| 数据源接口变动（AKShare） | 数据更新失败 | 多源回退、质量检查、快照不可变；更新失败时流水线跳过当天并告警 |
| 全局 N 只增不减，Holm 门槛越来越严 | 长期来看越来越难发现新因子 | 这是诚实性的代价，**明确接受**。hard_fail 要求 Holm 和 BH（q<0.10）两项都不通过才触发，所以以 BH 为主的发现仍然可行。新数据积累后可以建新的 split 版本，但旧 trial 仍计入 N。报告会展示当前 N 和对应的门槛 |
| 小样本股票池导致统计功效不足 | 很难发现显著因子 | M6 把股票池扩到沪深 300；报告给出功效说明；**接受阴性结论** |
| Postgres 单点 | 服务不可用 | 本项目定位单机部署；runbook 中写明备份和恢复 |
| 范围蔓延 | 迟迟无法交付 | 里程碑门禁；非目标清单；M8 单独立项 |

## 12. 待定问题

- [x] ~~审批人能否与发起人为同一人？~~ 已决定：默认允许并标记 `self_approved`，可以通过配置禁止（§8.5）。
- [x] ~~外部 MCP 客户端的 trial 默认预算~~ 已决定：每个会话 20 个，每个 API key 每天 60 个。
- [ ] 是否需要支持多个 strategy_version 并行做模拟盘（A/B 对比）？默认不支持，留到 M6 再评估。

## 13. 变更记录

| 版本 | 日期 | 变更 |
|---|---|---|
| v1.0 | 2026-11-08 | 初版 |
| v1.2 | 2026-11-08 | 处理 spec 评审的 8 个阻断问题：(1) 明确信任边界，CLI 增加封存窗口开关并写 ledger 和审计，直接 import 内核库得到的结果不能注册；(2) 承认因子库是受控泄漏，registry.list 对 Agent 脱敏；(3) 审批过期走 resume(expire) 的拒绝分支，默认 7 天；(4) 统一审批决策契约，因子增加 rejected 终态；(5) agent_events 的 payload schema 只有一个来源（core/events.py）；(6) trial 增加 split_id 和 sample_every，并纳入查重键，默认 horizon=5、sample_every=5；(7) 名称以支撑文档为准统一（kill-switch 和 admin 端点、approval.allow_self、ops 白名单、job 状态、run kind、outcome 增加 holdout_failed）；(8) CLI 本地身份使用 AP_LOCAL_USER。此外：HOLDOUT 一律记为 PSEUDO_OOS 并附 contamination 标签；相关性检查范围扩到 approved 因子；Monitor Graph 增加重写边；M1 验收要求 AP_REQUIRE_GOLDEN；M2 使用内存 RunStore；统计功效风险；关闭两个待定问题 |
| v1.1 | 2026-11-08 | 核心模型改为 DeepSeek V4（ADR-0008）；模型调用直接使用官方 SDK（ADR-0005）。根据 8 篇详细设计的反馈修订：新增 `services/` 应用服务层和唯一的审批入口；trial 增加 `idempotency_key`；critic 路由规则（预算耗尽时，有候选进 gate，没有就写阴性报告）；holdout 判定分为 consistent、weak、reversed；State 增加 route 和 outcome，errors 使用 reducer；approvals、research_runs、jobs 补齐字段（version 乐观锁、self_approved、租约等）；补齐成交、对账、kill-switch、认证、通知流、管理类端点；运维端点挂在根路径；self_approved 口径统一；N 的口径写明；MCP 工具对外命名使用下划线；mcp 2.x 的 FastMCP 已改名为 MCPServer；测试数量按实际核对；M3 验收改为 DeepSeek；trading 目录补齐；新增 RESERVED 段和切分版本 `split_id`；数据湖按 symbol 分目录、按年分片；股票池和样本区间的事实按 qtc 实际数据更正（132 只，截至 2025-02-14） |
