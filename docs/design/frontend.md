# AlphaPilot Web 控制台详细设计

> **状态**：草案（待评审）· **对应 spec 章节**：§2、§5.7、§5.8、§8（主文档 `docs/superpowers/specs/2026-11-08-alphapilot-design.md`）；接口细节见 `docs/design/api.md` · **最后更新**：2026-11-08（已对齐 spec v1.2）
>
> 前端只做“展示加人工决策”，**不重复实现任何业务判定**：hard_fail、能否批准、是否 self_approved 等，都以服务端返回的 `viewer` 和 `block_reasons` 为准（见 api.md §2.3）。

---

## 1. 技术栈与版本

| 领域 | 选型 | 主版本 | 说明 |
|---|---|---|---|
| 运行时与包管理 | Node 24 LTS、**pnpm 10** | — | 在 `package.json` 中声明 `packageManager: "pnpm@10.x"`；用 corepack 固定版本 |
| 框架 | **React 19** | 19.x | 不使用 RSC；纯 SPA |
| 构建 | **Vite** | 7.x | `@vitejs/plugin-react`；开发时用 `server.proxy['/api']` 代理到 `:8000` |
| 语言 | **TypeScript（strict）** | 5.9+ | 开启 `strict`、`noUncheckedIndexedAccess`、`exactOptionalPropertyTypes`、`noImplicitOverride` |
| 服务端状态 | **TanStack Query** | v5 | 加 Devtools，只在开发环境启用 |
| 路由 | **React Router** | v7（data router，`createBrowserRouter`） | 用 loader 做鉴权跳转 |
| 样式与组件 | **Tailwind CSS** v4 + **shadcn/ui**（Radix） | — | 使用 CSS-first 的 `@theme`；用 shadcn CLI 把组件生成到 `components/ui` |
| 图表 | **ECharts** | 6.x | 用 `echarts/core` 按需引入；自己写一层很薄的 `<EChart>` 封装 |
| 表格 | TanStack Table v8 + TanStack Virtual | — | trial 表和时间线都做虚拟滚动 |
| 实时 | **原生 EventSource** | — | 不使用 polyfill；需要的请求头通过同源 cookie 带上 |
| 校验 | **Zod** | 4.x | 用于表单 schema 和 SSE 事件的运行时校验 |
| API 类型 | openapi-typescript 7 + openapi-fetch | — | 见 §4.1 |
| Markdown | react-markdown 10 + remark-gfm | — | **不启用** rehype-raw，避免 XSS |
| 表单 | react-hook-form + `@hookform/resolvers/zod` | — | |
| 时间 | date-fns 4 + `@date-fns/tz` | — | 交易日按 Asia/Shanghai 展示 |
| 测试 | **Vitest** 3 + Testing Library + MSW 2；**Playwright** | — | |
| 质量 | ESLint 9（flat config，typescript-eslint strict、react-hooks、jsx-a11y）+ Prettier | — | |

---

## 2. 目录结构

```
web/
├─ package.json  pnpm-lock.yaml  vite.config.ts  tsconfig.json  eslint.config.js
├─ components.json            # shadcn 配置
├─ openapi.json               # 由后端导出，提交到仓库（drift 检查的基准）
├─ playwright.config.ts
├─ e2e/                       # Playwright 冒烟测试
│  ├─ fixtures.ts  smoke.spec.ts
├─ public/  favicon.svg
└─ src/
   ├─ main.tsx                # QueryClientProvider、ThemeProvider、RouterProvider
   ├─ router.tsx              # 路由表（§3.1）
   ├─ api/
   │  ├─ schema.gen.ts        # openapi-typescript 生成，禁止手改
   │  ├─ client.ts            # openapi-fetch 实例，加上 CSRF 和 problem 中间件
   │  ├─ problem.ts           # ProblemError 类、错误码到文案的映射
   │  ├─ keys.ts              # Query key 工厂
   │  ├─ queries/             # runs.ts approvals.ts factors.ts backtests.ts paper.ts alerts.ts reports.ts llm.ts admin.ts
   │  └─ sse/
   │     ├─ events.ts         # SSE 事件的 Zod schema（与生成类型对齐；7 种持久化事件的唯一来源是后端 core/events.py，字段以 agents.md §0.3 为准）
   │     ├─ useRunEvents.ts
   │     └─ useNotifications.ts
   ├─ auth/  AuthProvider.tsx  requireAuth.ts  useCan.ts  permissions.ts
   ├─ components/
   │  ├─ ui/                  # shadcn 生成：button dialog alert-dialog tabs table badge tooltip sheet ...
   │  ├─ layout/  AppShell.tsx  Sidebar.tsx  Topbar.tsx  KillSwitchPill.tsx  PendingBadge.tsx
   │  ├─ common/  PageHeader  DataTable  StatTile  Num  SignedNum  EvidenceBadge  StatusBadge
   │  │           IntegrityBanner  EmptyState  ErrorState  Forbidden  ConfirmDialog  CopyButton  Markdown
   │  └─ charts/  EChart.tsx  theme.ts  NavChart  DrawdownChart  QuantileBars  IcDecayBars  CostBar  SpendBars  Sparkline
   ├─ features/
   │  ├─ auth/        LoginPage.tsx
   │  ├─ dashboard/   DashboardPage.tsx  PipelineCard  PendingCard  AlertsCard  SpendCard
   │  ├─ runs/        RunsPage  NewRunDialog  RunDetailPage  Timeline/  TrialTable  BudgetBars  ReportPane
   │  ├─ approvals/   ApprovalsPage  ApprovalList  CandidateCard  DecisionPanel  ConfirmDecisionDialog
   │  ├─ factors/     FactorsBoardPage  FactorCard  FactorDetailPage  LineageFlow  PromoteDialog
   │  ├─ backtests/   BacktestPage
   │  ├─ paper/       PaperPage  KillSwitchPanel  ReconPanel  PositionsTable  OrdersTable  TradesTable
   │  ├─ alerts/      AlertsPage  DailyReportPane  AlertsTable
   │  └─ settings/    SettingsPage  UsageTab  RoutingTab  BudgetTab  ApiKeysTab  UsersTab  PreferencesTab
   ├─ lib/  format.ts  colors.ts  time.ts  url.ts  preferences.ts
   ├─ i18n/ zh-CN.ts  index.ts  enums.ts
   ├─ styles/ globals.css     # Tailwind @theme 和 CSS 变量（§8.1）
   └─ test/ setup.ts  msw/handlers.ts  msw/fixtures/*.json  render.tsx
```

规则：
- `features/*` 之间不能互相 import，共享的部分放在 `components/` 或 `api/`。
- 组件不能直接调用 `fetch`，只能使用 `api/queries` 里导出的 hook。

---

## 3. 路由与布局

### 3.1 路由表

| Path | 页面 | 最低角色 | 备注 |
|---|---|---|---|
| `/login` | LoginPage | 匿名 | 已登录时访问会跳转到 `?next=` 或 `/` |
| `/` | Dashboard | researcher | |
| `/runs` | Research Runs | researcher | `?status=&q=`；`?new=1` 打开新建对话框 |
| `/runs/:runId` | Run 详情 | researcher | `?tab=timeline|trials|report`、`&trial=42&field=icir`（用于高亮） |
| `/approvals` | 审批中心 | researcher（只读） | `?status=pending&kind=&id=apv_...`；左边列表，右边详情 |
| `/factors` | 因子库看板 | researcher | `?q=` |
| `/factors/:factorId` | 因子详情 | researcher | |
| `/backtests/:backtestId` | 回测报告 | researcher | |
| `/paper/:acct?` | 模拟盘 | researcher | 默认账户是 `paper-main`；`?date=` |
| `/alerts` | 告警与日报 | researcher | `?date=YYYY-MM-DD&tab=report|alerts` |
| `/settings` | 设置与用量 | researcher | `?tab=usage|routing|budget|keys|users|prefs`；budget 和 users 两个 tab 只对 admin 可见 |
| `*` | NotFound | — | |

所有受保护路由都挂在 `AppShell` 布局路由下面，由它的 `loader` 调用 `ensureQueryData(qk.me())`。如果返回 401，抛出 `redirect('/login?next=' + encodeURIComponent(path))`。

### 3.2 布局

```
┌──────────────┬─────────────────────────────────────────────────────────────────────┐
│ ◆ AlphaPilot │ 研究 / Runs / run_01JC9Z…           [● 交易正常] [🔔 审批 3] [perlou·admin ▾] │
│              ├─────────────────────────────────────────────────────────────────────┤
│ ▣ 总览        │ ┌ IntegrityBanner（合成数据或 kill-switch 时显示，全宽）─────────────────┐ │
│ ⟳ 研究任务     │ └──────────────────────────────────────────────────────────────────┘ │
│ ✓ 审批中心 (3) │                                                                     │
│ ▤ 因子库      │                        <Outlet />                                    │
│ ◫ 回测        │                                                                     │
│ ¥ 模拟盘      │                                                                     │
│ ! 告警与日报 (2)│                                                                     │
│ ⚙ 设置与用量   │                                                                     │
│              │                                                                     │
│ ─────────    │                                                                     │
│ ◐ 主题 · v0.5 │                                                                     │
└──────────────┴─────────────────────────────────────────────────────────────────────┘
```

- **侧边栏**：宽 240px，可以折叠成 64px 的图标栏，折叠状态存在 `localStorage`。徽标数字来自通知流的 `snapshot`。
- **顶栏**：
  - 面包屑；
  - `KillSwitchPill`：交易正常时显示灰绿色“交易正常”；触发后显示红底白字“交易已暂停”，并带脉冲动画（`prefers-reduced-motion` 下关闭动画），点击跳转到 `/paper`；
  - `PendingBadge`：显示待审批总数，点击打开下拉菜单，按 kind 分组列出，每项链接到 `/approvals?id=`；
  - 用户菜单：显示角色，提供退出登录和偏好设置入口。
- 全局 kill-switch 触发时，所有页面顶部都显示一条红色 `IntegrityBanner kind="kill_switch"`。

---

## 4. 数据层

### 4.1 API client

```ts
// api/client.ts
import createClient, { type Middleware } from "openapi-fetch";
import type { paths } from "./schema.gen";
import { ProblemError } from "./problem";

const csrf: Middleware = {
  onRequest({ request }) {
    if (request.method !== "GET") {
      const t = document.cookie.match(/(?:^|; )ap_csrf=([^;]+)/)?.[1];
      if (t) request.headers.set("X-CSRF-Token", decodeURIComponent(t));
    }
    request.headers.set("X-Request-ID", crypto.randomUUID());
    return request;
  },
};
const problems: Middleware = {
  async onResponse({ response }) {
    if (response.status === 401 && !response.url.includes("/auth/")) await refreshOnce(); // 单飞刷新；失败则跳转登录
    if (!response.ok && response.headers.get("content-type")?.includes("problem+json"))
      throw new ProblemError(await response.clone().json(), response.headers.get("X-Request-ID"));
    return response;
  },
};
export const api = createClient<paths>({ baseUrl: "/api/v1", credentials: "same-origin" });
api.use(csrf, problems);
```

- 只用同源相对路径 `/api/v1`，开发和生产都经过代理，避免 CORS 问题。
- `ProblemError` 带有 `code`、`status`、`detail`、`requestId`、`traceId`，以及扩展字段（例如 `hard_fail`）。`ErrorState` 组件会显示 request_id，并提供“复制排障信息”按钮。
- 401 刷新：调用 `POST /auth/refresh`，并发请求共用同一个 promise。刷新成功后重放原请求一次；仍然失败时清空缓存，跳转到 `/login?next=`。

### 4.2 Query key 规范

格式是 `[域, 子资源, 参数对象]`。参数对象必须可以序列化，并且只包含影响结果的字段。

```ts
export const qk = {
  me:          () => ["auth", "me"] as const,
  runs: {
    all:       ["runs"] as const,
    list:      (f: RunFilters) => ["runs", "list", f] as const,
    detail:    (id: string) => ["runs", "detail", id] as const,
    trials:    (id: string, f: TrialFilters) => ["runs", "trials", id, f] as const,
    report:    (id: string) => ["runs", "report", id] as const,
  },
  approvals: {
    all:       ["approvals"] as const,
    list:      (f: ApprovalFilters) => ["approvals", "list", f] as const,
    detail:    (id: string) => ["approvals", "detail", id] as const,
  },
  factors:     { all: ["factors"] as const, list: (f) => ["factors", "list", f] as const, detail: (id: number) => ["factors", "detail", id] as const },
  backtests:   { detail: (id: string) => ["backtests", "detail", id] as const },
  paper:       { nav: (a, r) => ["paper", a, "nav", r] as const, positions: (a, d) => ["paper", a, "positions", d] as const,
                 orders: (a, d) => ["paper", a, "orders", d] as const, trades: (a, d) => ["paper", a, "trades", d] as const,
                 recon: (a, d) => ["paper", a, "recon", d] as const },
  killSwitch:  () => ["ops", "kill-switch"] as const,
  pipeline:    (d: string) => ["pipeline", d] as const,
  alerts:      (f) => ["alerts", f] as const,
  reports:     { list: () => ["reports", "daily"] as const, daily: (d: string) => ["reports", "daily", d] as const },
  llmUsage:    (r) => ["llm", "usage", r] as const,
  snapshots:   () => ["snapshots"] as const,
  routing:     () => ["settings", "llm-routing"] as const,
  admin:       { users: () => ["admin", "users"] as const, jobs: (f) => ["admin", "jobs", f] as const, settings: () => ["admin", "settings"] as const },
  apiKeys:     () => ["me", "api-keys"] as const,
};
```

默认配置：
- `staleTime`：列表 15 秒，详情 30 秒，`routing` 和 `snapshots` 为 5 分钟；
- `retry`：4xx 不重试，5xx 重试 2 次；
- `refetchOnWindowFocus: true`。

失效规则汇总：

| 触发 | 失效的 key |
|---|---|
| 创建 run | `runs.all` |
| 审批决策成功 | `approvals.all`，subject 对应的 `runs.detail`，`factors.all`（kind 为 promote、retire 或 register 时），`killSwitch`（ops_action 的 `release_kill_switch` 时） |
| 通知流 `approval.*` | `approvals.all` |
| 通知流 `alert.created` | `["alerts"]` |
| 通知流 `kill_switch.changed` | `killSwitch` |
| 通知流 `pipeline.status` | `pipeline(today)` |
| 通知流 `run.status` | `runs.detail(id)`、`runs.list` |
| Run SSE `tool` 事件（带 trial_id） | `runs.trials(id)`，每秒最多失效一次 |
| Run SSE `status` | 直接 `setQueryData(runs.detail(id))` 更新状态和预算 |

### 4.3 `useRunEvents(runId)`

```ts
type RunEvent = z.infer<typeof SseEnvelope>;
interface RunEventsState { events: RunEvent[]; status?: StatusPayload; connection: "connecting"|"open"|"retrying"|"closed"; lastId: number }

export function useRunEvents(runId: string): RunEventsState {
  const [state, dispatch] = useReducer(reducer, initial);
  const lastId = useRef(0); const terminal = useRef(false); const attempt = useRef(0);
  const qc = useQueryClient();

  useEffect(() => {
    let es: EventSource | null = null; let timer: number | undefined; let disposed = false;
    const open = () => {
      const url = `/api/v1/runs/${runId}/events` + (lastId.current ? `?last_event_id=${lastId.current}` : "");
      es = new EventSource(url, { withCredentials: true });
      dispatch({ t: "conn", v: "connecting" });
      es.onopen = () => { attempt.current = 0; dispatch({ t: "conn", v: "open" }); };
      for (const type of PERSISTED_TYPES) es.addEventListener(type, (m) => {
        const id = Number((m as MessageEvent).lastEventId);
        if (!Number.isFinite(id) || id <= lastId.current) return;          // 按 id 去重：补发和实时之间的重叠
        const ev = SseEnvelope.safeParse(JSON.parse((m as MessageEvent).data));
        if (!ev.success) return reportSchemaDrift(ev.error);               // 不抛异常，只上报
        lastId.current = id; dispatch({ t: "event", ev: ev.data }); sideEffects(qc, runId, ev.data);
      });
      es.addEventListener("status", (m) => {
        const s = StatusPayload.parse(JSON.parse(m.data).payload);
        dispatch({ t: "status", s }); qc.setQueryData(qk.runs.detail(runId), patchStatus(s));
        if (TERMINAL.has(s.status)) { terminal.current = true; es?.close(); dispatch({ t: "conn", v: "closed" }); }
      });
      es.onerror = () => {
        if (terminal.current || disposed) return;
        if (es?.readyState === EventSource.CONNECTING) { dispatch({ t: "conn", v: "retrying" }); return; } // 浏览器自动重连，会自动带上 Last-Event-ID
        es?.close();                                                       // CLOSED：收到非 200 响应（401/429/5xx/204）
        const delay = Math.min(30_000, 1000 * 2 ** attempt.current++) * (0.8 + Math.random() * 0.4);
        timer = window.setTimeout(async () => { if (await ensureSession()) open(); }, delay); // 先检查会话，401 时跳转登录
      };
    };
    open();
    return () => { disposed = true; es?.close(); clearTimeout(timer); };
  }, [runId]);
  return state;
}
```

- 页面隐藏超过 5 分钟（`visibilitychange`）时主动关闭连接，回到前台后用 `lastId` 续传，以节省服务端的连接配额。
- 内存中最多保留 5000 条事件，超过时把最早的 `llm` 和 `tool` 明细折叠成计数，但 `guardrail`、`interrupt` 和 `error` 永远保留。
- `useNotifications()` 的结构类似，但不做补发。每次收到 `snapshot` 时，把计数写入一个轻量的 `NotificationsContext`（只存计数，不放进 Query 缓存，也不引入额外的状态库），侧边栏和顶栏的徽标从这里读取；其他事件只负责让相关 query 失效。

### 4.4 Mutation 与乐观更新

| 操作 | 策略 |
|---|---|
| **审批决策**、promote/retire 发起、kill-switch、取消 run | **不做乐观更新**。按钮进入 loading 状态，等服务端返回 2xx 后再让相关 query 失效。出错时把 problem 原样展示出来。所有请求都带 `Idempotency-Key`（每打开一次对话框生成一个 UUID，所以重复点击不会重复提交）。审批请求另外带 `If-Match: "<version>"` |
| 告警 ack、删除 API Key、偏好设置（主题、涨跌配色） | 乐观更新：在 `onMutate` 中修改缓存，`onError` 时回滚并提示，`onSettled` 时让 query 失效 |
| 创建 run | 不做乐观更新；成功后 `navigate(/runs/{id})` |

---

## 5. 认证与权限

- **LoginPage**：用户名和密码表单，经过 Zod 校验后调用 `POST /auth/login`，成功后 `qc.setQueryData(qk.me(), me)`，再跳转到 `next`。错误统一显示为“用户名或密码错误”，不区分具体原因。收到 429 时显示倒计时。
- 会话完全依赖 httpOnly cookie，前端**拿不到也不存储** JWT。注销时调用 `POST /auth/logout`，然后执行 `qc.clear()`。
- **权限 hook**：

  ```ts
  const RANK = { researcher: 0, approver: 1, admin: 2 } as const;
  export const useCan = () => { const me = useMe(); return (min: Role) => !!me && RANK[me.role] >= RANK[min]; };
  ```

  | 场景 | 处理方式 |
  |---|---|
  | 整页需要 admin（设置中的预算和用户 tab） | 隐藏导航入口；直接访问 URL 时显示 `<Forbidden/>` |
  | 操作按钮需要更高角色，例如 researcher 查看审批 | **显示但禁用**，tooltip 写“需要 approver 角色”，让用户知道这项能力存在 |
  | 业务原因不能执行（hard_fail、self-approval 被禁止） | 禁用按钮，并在旁边显示 `block_reasons` 对应的文案（以服务端的 `viewer` 为准） |
  | 服务端返回 403 | 弹出 toast 并刷新 `me`，因为角色可能刚被修改 |

---

## 6. 页面

所有页面都遵循同一套状态约定：
- **加载**：使用与最终布局相同的 Skeleton，不用整页转圈。
- **空**：`EmptyState`，包含插图、一句说明和主操作按钮。
- **错误**：`ErrorState`，展示 problem 的 title 和 detail，提供重试按钮，并显示 request_id。
- **权限不足**：`Forbidden`。

### 6.1 Dashboard（`/`）

**用途**：每天第一眼要看的页面，回答三个问题：流水线跑完了吗？有什么需要我处理？钱和成本怎么样？

```
┌ 今日流水线 2026-11-06 ──────────────┐┌ 待审批 ───────────────┐┌ 交易状态 ─────────┐
│ data.update ✓ 16:31  pipeline ✓ 16:39 ││ holdout 1  register 0  ││ ● 交易正常         │
│ monitor ⟳ 运行中                       ││ promote 1  ops 2       ││ 对账 ✓ 11-06       │
│ [查看日报]                             ││ [去审批中心 →]          ││ [模拟盘 →]          │
└───────────────────────────────────────┘└───────────────────────┘└──────────────────┘
┌ NAV vs 沪深300（近 90 日）────────────────────────────┐┌ 未处理告警 ─────────────────┐
│  1.08 ┤                         ╭─ 策略 +6.2%          ││ P2 因子 mom_12_1 IC 衰减  2h │
│  1.00 ┼──╮___╭──╮_____╭────────╯─ 基准 +1.9%           ││ P2 数据源回退至 sina     1d │
│       └──────────────────────────────── [30D|90D|YTD]  ││ [全部告警 →]                │
└──────────────────────────────────────────────────────┘└────────────────────────────┘
┌ LLM 花费（近 30 天，美元）───────────── 今日 $0.84 / 上限 $5.00 ┐┌ 最近任务 ─────────────┐
│ ▁▂▁▃▅▂▁▁▂▆▃▂▁▁ ……（按 provider 堆叠的柱状图，带上限参考线）     ││ run_…9Z 等待审批  2m   │
└──────────────────────────────────────────────────────────────┘│ run_…7A 已完成    1d   │
                                                                 └───────────────────────┘
```

- **API**：`GET /pipeline/runs?date=today`、`GET /approvals?status=pending`（通知流的 snapshot 提供计数）、`GET /ops/kill-switch`、`GET /paper/paper-main/nav?from=`、`GET /alerts?status=open&limit=5`、`GET /llm/usage?group_by=day,provider`、`GET /runs?limit=5`；SSE 使用 `/notifications/stream`。
- **组件**：`PipelineCard`（每一步一个 chip：✓、✗、⟳、—，失败时显示 error 摘要）、`PendingCard`、`NavChart`（mini 尺寸）、`AlertsCard`、`SpendCard`（`SpendBars` 加上 markLine 表示每日上限）。
- **状态**：
  - 非交易日时，流水线卡片显示“今日非交易日”；
  - 还没有模拟盘数据时，NAV 卡片显示空态，并引导去因子库上线因子；
  - 每张卡片单独加载、单独报错，一张失败不影响其他卡片。
- **交互**：每张卡片的标题都可以点击跳转；时间范围切换会同步到 URL（`?range=90d`）。

### 6.2 Research Runs（`/runs`）

```
┌ 研究任务 ─────────────────────────────────────────────────── [+ 新建研究] ┐
│ [状态 ▾ 全部] [类型 ▾ research/interactive] [结果 ▾ outcome] [搜索目标…]   │
├──────────┬───────────────────┬──────────┬────────┬────────┬────────┬──────┤
│ ID       │ 目标               │ 状态      │ trial  │ LLM $  │ 发起人  │ 创建  │
│ run_…9Z  │ 低波动+动量反转     │ ⏸ 等待审批 │ 9/14   │ 0.12   │ perlou │ 2m   │
│ run_…7A  │ 换手率异象          │ ✓ 已完成   │ 14/14  │ 0.31   │ perlou │ 1d   │
└──────────┴───────────────────┴──────────┴────────┴────────┴────────┴──────┘
                                                            [加载更多]
新建研究（Dialog）
┌──────────────────────────────────────────────┐
│ 研究目标*   [低波动+动量反转                    ] │
│ trial 预算  [14 ]   LLM 预算 $ [0.50]  今日剩余 $4.16 │
│ 假设数 K   [3]     holdout 上限 [2]                │
│ horizon [5] 交易日   sample_every [5] 交易日         │
│ 切分版本    [qtc09（默认）▾]                        │
│ 数据快照    [snap_…（最新，2018-01~2025-02）▾]     │
│ ▸ 高级：各角色模型                                  │
│   planner  [deepseek:deepseek-v4-pro ▾]          │
│   coder    [deepseek:deepseek-flash ▾] …         │
│ ⚠ 所选快照为合成数据：结果不能上线（选中合成快照时显示）  │
│                              [取消] [创建并查看 →] │
└──────────────────────────────────────────────┘
```

- **API**：`GET /runs`（cursor 分页，点“加载更多”时使用 `useInfiniteQuery`）、`POST /runs`、`GET /snapshots`、`GET /settings/llm-routing`（模型下拉的选项只来自这个白名单）、`GET /llm/usage`（今日剩余额度）。
- **表单**：Zod schema 与 `RunCreate` 对齐（goal 4–500 字符，trials 1–200，llm_usd 不超过 5，`horizon` 1–60、默认 5，`sample_every` 1–60、默认 5，单位都是交易日；`split_id` 为空时用服务端默认版本）。`sample_every` 旁边说明“IC 采样间隔，只影响研究评估；生产调仓频率在 strategy.yaml 中单独配置（默认月度）”。改动 horizon、sample_every 或切分版本会让已有 trial 不能复用（查重键为 `(canonical_hash, snapshot_id, split_id, horizon, sample_every)`），在字段旁提示。预算超过今日剩余额度时，在前端提示；最终以服务端返回的 409 `budget-exhausted` 为准。
- **状态**：
  - 空列表显示“还没有研究任务”，并提供“新建研究”按钮；
  - `awaiting_approval` 状态的行高亮显示，并提供“去审批”链接。

### 6.3 Run 详情（`/runs/:runId`）★

```
┌ run_01JC9Z… · 低波动+动量反转                       ⏸ 等待审批（holdout） [去审批] [取消任务] ┐
│ 快照 snap_…  2018-01~2025-02 · 切分 qtc09 · IS 2018-01~2022-12 · HOLDOUT 2023-02~2025-02-14 · horizon 5 · sample_every 5 · 模型截止 2026-10-17 │
├──────────────────────────────────────────────────────────────────────────────────────────┤
│ ⚠ PSEUDO_OOS · 污染 full：holdout 结束日（2025-02-14）早于模型知识截止日（2026-10-17），holdout 结果可能已被模型“见过”│
├──────────────────────────────────────────────────────────────────────────────────────────┤
│ trial 预算  ███████████████░░░░░░  9 / 14        LLM 预算  ████░░░░░░░░░░░░  $0.12 / $0.50   │
│ 假设 3 · 候选 2 · 护栏事件 1 · 全局 N = 1,287                                    ● 实时已连接   │
├ [时间线] [Trials 9] [报告] ────────────────────────────────────────────────────────────────┤
│ 节点泳道（概览，可以拖动缩放）                                                                 │
│ plan     ██                                                                               │
│ code       ▌ ▌  ▌   ▌                                                                     │
│ validate    ▏ ▏  ▏   ▏                                                                    │
│ evaluate     ██ ██  ██  ██                                                                │
│ critic          ▌    ▌   ▌                                                                │
│ gate               ………………………………⏸                                                         │
│ ──────────────────────────────────────────────────────────────────────────────────────── │
│ 筛选：[✓节点][✓LLM][✓工具][✓护栏][✓错误]  [仅看 evaluate ▾]  [自动滚动 ✓]                     │
│ 08:00:03  ▶ plan 开始                                                                     │
│ 08:00:05    🤖 LLM call · planner · deepseek-v4-pro · 3.2k→1.1k tok (思考 2.4k) · $0.0061 · 4.1s ▸ │
│ 08:00:09    🔧 data.describe · 112ms · ok                                              ▸ │
│ 08:00:15  ■ plan 结束 · 12.0s · 更新 hypotheses                                           │
│ 08:00:16  ▶ code (H1)                                                                     │
│ 08:00:18    🤖 LLM call · coder · deepseek-flash · 0.9k→0.1k · $0.0003 · 1.2s · 缓存 hit     ▸ │
│ 08:00:19    🔧 factor.validate · ok · 112ms                                              ▸ │
│ 08:00:21    🔧 factor.evaluate · ok · trial #42 · 计入 trial（9/14）               [→ #42] ▸ │
│ 08:00:22    🛡 护栏 E_DATA_CUTOFF · warn · 请求区间越过 is_end，已截断                       ▸ │
│ 08:03:40  ⏸ interrupt enter · gate holdout · apv_01JCA…  [去审批]                          │
│   ▾ 展开后显示 payload JSON（等宽字体，可折叠，带复制按钮；工具参数只有 params_digest，不含原文）  │
└──────────────────────────────────────────────────────────────────────────────────────────┘
```

**时间线**（`features/runs/Timeline/`）
- 事件字段以后端 `core/events.py`（即 agents.md §0.3）为准；节点名取外层信封的 `node`，细分类取 `payload.kind`。
- 上方是**节点泳道**（`LaneOverview`）：用 ECharts custom series 画甘特图，每个节点一行，每次 node_start 到 node_end 画一段（`node_start.resumed=true` 时在段首画一个恢复标记），并在上面标出 interrupt、guardrail 和 error 的位置。点击某一段，下方列表会滚动到对应位置。
- 下方是**垂直时间线**（`EventList`），用 TanStack Virtual 做虚拟滚动：
  - `node_start` 和 `node_end` 作为分组标题，组内缩进显示 llm、tool 和 guardrail 事件；`node_end` 显示 `duration_ms` 和 `updates`（更新的 state 字段名）；`kind="route"` 时额外显示 `route` 和 `rule`（例如“R3 → gate_holdout”）；
  - 每行右侧的 ▸ 可以展开，看到原始 `payload` JSON；
  - `llm` 行按 `payload.kind` 区分（字段见 llm-router.md §10.2）：`call` 显示 role、provider 和 model、`tokens.hit`/`tokens.miss`（输入中缓存命中和未命中的部分）、`tokens.out`、`tokens.reasoning`、`cost_usd`、`latency_ms`、`cache`（hit/miss/off）；`retry` 显示 reason 和 attempt；`fallback` 用橙色 chip 显示“回退”；`budget_warn` 显示 run_spent / limit；
  - `tool` 行显示 `tool`、`ok`、`duration_ms` 和 `warnings`；`counted_trial=true` 时显示 `计入 trial` 徽标和当前预算（预算取自最新的 `status` 事件）；`trial_id` 可以点击跳到 Trials tab 并高亮对应行；
  - `guardrail` 行显示 `code`、`tool` 和 `message`，`severity=warn` 使用 warning 底色和盾牌图标，`severity=block` 用 critical 颜色；
  - `error` 行使用 critical 颜色，显示 `exc_type` 和 `message`；`handled=true` 时附加说明“已在节点内处理”；
  - `interrupt` 行按 `kind` 区分：`enter` 带有“去审批”主按钮；`resume` 显示 `decision`（approve、reject、expire），expire 显示为“审批过期，按拒绝处理”。
- **自动滚动**：用户位于底部时自动跟随新事件；用户向上滚动后暂停跟随，并显示浮动按钮“↓ 有 N 条新事件”。
- **连接状态**指示：● 已连接、◌ 重连中、○ 已结束。重连过程中不清空列表，续传的事件按 id 去重后追加。
- 一个 node_end 之后很久没有新的 node_start，并且 run 状态仍是 running 时（超过 120 秒），显示一行“等待中…”占位，避免用户误以为页面卡住了。

**Trials tab**（`TrialTable`）
- 列：#、窗口（IS 或 HOLDOUT 徽标）、表达式（等宽字体，过长时省略，悬停显示全文）、所属假设、IC、ICIR、t、p、前后半段 IC（两个小条形，符号相反时显示 ⚠）、覆盖率、最大相关、前视检测、p_holm、q_bh、DSR、hard_fail 和 warnings（chip）、证据等级。
- 支持排序和列显隐。`?trial=42&field=icir` 会把该行滚动到可见区域，加 2 秒的高亮闪烁，并给对应的单元格加描边。
- hard_fail 的行左侧显示红色竖条；candidate 行显示 ★。
- 还没有任何 trial 时（例如仍在 plan 阶段），显示空态“尚未评估任何表达式”。
- 页脚说明：“p_holm、q_bh 以全局 N = 1,287 计算（包括外部 MCP 产生的 trial）”。

**预算进度条**（`BudgetBars`）
- trial 进度和美元进度各一条。超过 80% 显示 warning 颜色，达到 100% 显示 critical 颜色，并标注“预算耗尽 → 已进入报告阶段”。
- 数值来自 `status` 事件的 `budget`，实时更新（`llm` 事件只带单次 `cost_usd`，不做前端累加）。

**报告 tab**（`ReportPane`）

```
┌ 研究报告（final）  faithfulness ✓ 通过     [复制 Markdown] [下载 .md] ┐
│ ## 结论                                                               │
│ 候选 F1 在 IS 区间 ICIR 为 [0.52]，Holm 校正后 p = [0.018]……          │
│                         ↑ 带虚线下划线的数字，悬停显示“trial #42 · icir · IS”，点击跳到 Trials 并高亮 │
│ ## 未通过的假设及原因 …                                                 │
│ ## 附录：已知偏差 · 数据快照                                             │
└──────────────────────────────────────────────────────────────────────┘
```

- 数据来自 `GET /runs/{id}/report`。服务端把数字渲染成 `[0.52](ap://trial/42?field=icir)` 链接。
- `Markdown` 组件做两件事：
  1. 自定义 `urlTransform`：只放行 `http`、`https`、`ap` 三种协议。react-markdown 默认会去掉未知协议，所以必须自己处理。
  2. 重写 `components.a`：遇到 `ap://trial/{id}?field=` 链接时，渲染成 `<TrialRef>` 按钮（虚线下划线、tabular-nums，悬停显示 tooltip），点击时执行 `setSearchParams({tab: "trials", trial, field})`。`ap://backtest/{id}` 跳到回测页。
- 如果 faithfulness 没有通过，顶部显示 critical 横幅并列出 `issues`，同时提示“该报告不能提交审批”。
- 报告还没生成时（404），显示空态“报告将在 holdout 与回测完成后生成”。
- 附录中的 `known_biases` 会合并显示 `IntegrityBanner kind="biases"`。

**API 与 SSE**：`GET /runs/{id}`、`GET /runs/{id}/trials`、`GET /runs/{id}/report`、`POST /runs/{id}/cancel`；SSE 使用 `useRunEvents`。

**状态**：
- run 不存在（404）时显示 NotFound；
- 状态为 completed 时，在标题旁显示 `outcome`（`registered`、`holdout_declined`、`holdout_failed`、`register_declined`、`negative_budget`、`negative_exhausted`、`llm_unavailable`，中文见 `i18n/enums.ts`）；阴性结论和审批被拒或过期都属于正常完成，不用错误色；
- 状态为 failed 时，顶部显示错误摘要，内容取最后一条 error 事件（`exc_type` 和 `message`）；
- cancelled 时置灰，并显示取消人和时间；
- “取消任务”只有发起人或 admin 可以使用，并且需要二次确认。

### 6.4 审批中心（`/approvals`）★

```
┌ 审批中心  [待处理 3][已批准][已拒绝][已过期]   类型 [全部 ▾] ─────────────────────────────────────┐
├ 列表 ───────────────────┬ 详情：holdout 审批 · run_01JC9Z… 低波动+动量反转 ────────────────────────┤
│ ● holdout  run_…9Z  2m  │ 发起人 perlou · 2026-11-08 16:03 · 7 天后过期 · version 1                  │
│   promote  F:lowvol 1h  │ ┌ ⚠ 污染警告 ─────────────────────────────────────────────────────────┐ │
│   ops  pause_strategy 1d│ │ holdout 结束于 2025-02-14，早于模型知识截止日 2026-10-17 → 证据等级       │ │
│                         │ │ PSEUDO_OOS · 污染 full。已知偏差：幸存者偏差（当前成分股）、当前行业分类     │ │
│                         │ └────────────────────────────────────────────────────────────────────┘ │
│                         │ ┌ 候选 #42 ☐ 选择 ────────────────────── 证据 IS ─┐┌ 候选 #57 ─── ⛔ 硬性否决 ┐│
│                         │ │ -ts_std(ret, 20)                                ││ rank(volume)/…          ││
│                         │ │ 假设：低波动溢价（彩票偏好）  family low_vol  符号 + ││ ⛔ 前视检测失败           ││
│                         │ │ IC 0.031 · ICIR 0.52 · t 3.10 · p 0.002         ││ ⛔ 与 mom_12_1 相关 0.82  ││
│                         │ │ 前半 0.034 / 后半 0.027 · 覆盖 93% · 换手 38%     ││   > 0.7                 ││
│                         │ │ Holm p 0.018 ✓  BH q 0.011 ✓  （全局 N=1,287）   ││ 该候选不能被选中          ││
│                         │ │ DSR 0.97 ✓ · 最大相关 0.41（mom_12_1）           ││                         ││
│                         │ │ 前视检测 ✓ · ⚠ 只在单一行业有效                    ││                         ││
│                         │ │ critic：“…”          key a1b2c3d4e5f6           ││                         ││
│                         │ └─────────────────────────────────────────────────┘└─────────────────────────┘│
│                         │ 已选 1 / 2（holdout 上限）                                                  │
│                         │ 理由*  [两个候选经济逻辑清晰……                              ] 18/2000（至少 10）│
│                         │ ⓘ 你是该 run 的发起人，此次审批会被标记为 self_approved                      │
│                         │                                          [拒绝] [批准查看 holdout（1）]      │
└─────────────────────────┴───────────────────────────────────────────────────────────────────────┘
```

**候选卡片字段**（`CandidateCard`，数据来自 `payload.candidates[]`）

| 区块 | 字段 | 展示规则 |
|---|---|---|
| 头部 | candidate_key、expr、evidence_level | candidate_key 是 canonical_hash 的前 12 位，也是提交时 `selected_candidate_keys` 和 `names` 的 key；表达式用等宽字体，可以复制；`EvidenceBadge` 中 IS 灰色、PSEUDO_OOS 琥珀色、TRUE_OOS 蓝色；HOLDOUT 一律为 PSEUDO_OOS，旁边显示 contamination 标签（full、partial、none、unknown，unknown 按 full 展示） |
| 假设 | statement、rationale、family、expected_sign | rationale 默认折叠，显示两行 |
| IS 指标 | ic_mean、icir、t、p、ic_h1、ic_h2、coverage、turnover | 前后半段 IC 符号相反时显示 ⚠ |
| 多重检验 | p_holm、q_bh、n_trials_global、dsr | 用 ✓ 或 ✗ 对照阈值（0.05、0.10、0.95），并始终显示全局 N |
| 相关性 | max_corr.value、with | 比较范围是 production 因子、approved 因子和本 run 的其他候选；超过 0.7 时显示 ⛔（对应 hard_fail） |
| 前视检测 | lookahead_ok | ✓ 或 ⛔ |
| 判定 | hard_fail[]、warnings[] | 用 `i18n/enums.ts` 把代码映射成中文，例如 `lookahead_failed` 映射为“前视检测失败”，`max_corr_exceeded` 映射为“与 production/approved 因子或其他候选的相关性超过 0.7”，`low_coverage` 映射为“覆盖率低于 60%”，`not_significant` 映射为“Holm p ≥ 0.05 且 BH q ≥ 0.10” |
| 评语 | critic_comment | 标注“LLM 评语，仅供参考，判定由确定性代码完成” |
| 全局 | payload.contamination、snapshot.known_biases、is_synthetic | 用卡片组上方的 `IntegrityBanner` 显示（§8.4） |

**按钮与校验**
1. 所有 disabled 状态都以服务端的 `viewer.can_decide`、`viewer.can_approve` 和 `block_reasons` 为准。前端只负责把它们映射成文案并显示在按钮下方，例如“所有候选都存在硬性否决，只能拒绝”“你的角色是 researcher，需要 approver 才能审批”“当前配置 `approval.allow_self=false`，不能审批自己发起的任务”（默认 `allow_self=true`，此时只提示会标记 self_approved）。
2. hard_fail 的候选：复选框禁用，卡片置灰并显示红色描边，tooltip 列出全部原因。
3. holdout 的选择：已选数量达到 `holdout_limit` 后，其他候选的复选框变为禁用，提示“最多选 2 个”。没有选任何候选时，“批准”按钮禁用。提交体为 `DecisionIn {decision: "approve"|"reject", reason, selected_candidate_keys: string[], names?}`（见 api.md §2.3），`selected_candidate_keys` 取所选卡片的 candidate_key。
4. 理由：去掉首尾空白后至少 10 个字符，最多 2000 个，显示实时计数。没有填写时两个按钮都禁用。
5. register 审批：显示报告摘要和 faithfulness 状态，并提供“查看完整报告”链接（跳到 Run 详情的报告 tab），还要显示 HOLDOUT 指标（只有人类会话能看到）。每个候选有一个因子名输入框，默认填入 `suggested_name`（`registry.propose` 给出的建议名），审批人可以修改；前端按 `^[a-z][a-z0-9_]{2,40}$` 实时校验，提交时放进 `names`（key 为 candidate_key）。拒绝时提示“被拒的因子将进入终态 rejected，以后可以重新提交”。promote 审批：显示 strategy_diff（新增或移除的因子）。ops_action：显示 action、params 和证据链接（告警或日报）；Agent 建议的 action 只会是白名单 `pause_strategy | retire_factor | rerun_data_update | rerun_pipeline | reduce_gross | no_action`；`release_kill_switch` 是 admin 单独发起的审批，只有 admin 能决策。
6. 过期：pending 审批默认 7 天过期（scheduler 每 10 分钟扫描一次）。详情页显示剩余时间；“已过期”tab 中的 gate 类审批注明“过期按拒绝处理，run 以 completed 结束（outcome 为 holdout_declined 或 register_declined）”，不显示为失败。

**二次确认**（`ConfirmDecisionDialog`，使用 shadcn AlertDialog）

```
┌ 确认批准 holdout ─────────────────────────────────┐
│ 将对以下候选各运行一次 holdout，且每个表达式只能运行一次： │
│   #42  -ts_std(ret, 20)                           │
│ 证据等级上限：PSEUDO_OOS（存在污染风险）             │
│ ☐ 我已阅读污染警告与已知偏差                         │
│ ⓘ 本次审批将标记为 self_approved                    │
│ 理由：两个候选经济逻辑清晰……                         │
│                           [返回修改] [确认批准]      │
└──────────────────────────────────────────────────┘
```

- 只要有污染风险、已知偏差或合成数据，就必须勾选“我已阅读”之后才能点“确认批准”。拒绝也需要确认，但不要求勾选。
- 提交时带上 `If-Match: "<version>"` 和 `Idempotency-Key`（每次打开对话框生成一个）。对话框处于提交中状态时不能关闭。

**错误处理**

| 返回 | UI 反应 |
|---|---|
| 200 | toast 提示“已批准，研究将继续”；列表中该项移到“已批准”；如果是 holdout 或 register，提供“查看 run 进度”链接 |
| 409 `approval-hard-fail` | 在对应卡片上用红框标出 `hard_fail` 明细，不关闭对话框（说明数据已经变化） |
| 409 `approval-already-decided` / 412 `version-mismatch` | 显示提示“该审批已被他人处理或更新”，自动重新获取详情并展示最新状态；理由输入框的内容保留 |
| 403 `self-approval-forbidden` | 在按钮区显示原因 |
| 422 | 在字段旁显示错误（包括 `holdout-limit-exceeded`、`candidate-not-in-approval`、因子名不合法） |

**API 与 SSE**：`GET /approvals`、`GET /approvals/{id}`（从响应头读 ETag）、`POST /approvals/{id}/decision`；SSE 通过 `/notifications/stream` 实时把新审批加入列表。如果有人在你查看时处理了这条审批，详情页上方会显示条幅“此审批已被 X 处理”。

**状态**：没有待处理审批时显示“没有待处理的审批 🎉”；researcher 只能查看，按钮禁用并说明原因。

### 6.5 因子库（`/factors`、`/factors/:id`）★

```
┌ 因子库  [搜索名称/表达式…]                                                                     ┐
├ draft 4 ──────┬ candidate 2 ──┬ approved 3 ───┬ production 2 ──────┬ retired 5 ──────────────┤
│ ┌───────────┐ │ ┌───────────┐ │ ┌───────────┐ │ ┌────────────────┐ │ ┌───────────┐            │
│ │ f_draft_1 │ │ │ lowvol_20 │ │ │ rev_5d    │ │ │ mom_12_1       │ │ │ size_old  │            │
│ │ -ts_mean… │ │ │ ⏳ 注册审批中│ │ │ PSEUDO_OOS│ │ │ TRUE_OOS · 62日 │ │ │ 2026-09   │            │
│ │ IS        │ │ │ IS        │ │ │ ICIR .48  │ │ │ IC20 ▁▂▃▂▁ .021 │ │ │ 原因：衰减  │            │
│ └───────────┘ │ └───────────┘ │ │ [发起上线]  │ │ │ [发起下线]       │ │ └───────────┘            │
│               │               │ └───────────┘ │ └────────────────┘ │                          │
└───────────────┴───────────────┴───────────────┴────────────────────┴──────────────────────────┘
```

- **看板**：数据来自 `GET /factors`（一次拉取全部，前端按 status 分组；超过 200 条时按列分别分页）。除图中五列外，终态 `rejected`（register 被拒，见 `factor_events(register_rejected)`）默认折叠在 retired 右侧的“已拒绝”列，只读。
  - 卡片显示名称、表达式、证据等级、核心指标、IC sparkline（只有 production 有）、待审批徽标、合成数据 chip。
  - **不支持拖拽**：因子的状态迁移必须经过审批（spec §2），拖拽会让人误以为可以直接修改状态。卡片上只提供“发起上线”和“发起下线”按钮，点击后弹出 `PromoteDialog`（理由必填，带 Idempotency-Key），成功后在卡片上显示“⏳ 上线审批中”，并提供链接跳到审批中心。
  - 基于合成数据的因子：“发起上线”按钮禁用，tooltip 写“合成数据不能上线（§8.6）”。
- **详情页**：

```
┌ lowvol_20 · approved · PSEUDO_OOS                                  [发起上线] ┐
│ 表达式 -ts_std(ret, 20)  canonical_hash c1a9…                               │
├ 来源链路 ─────────────────────────────────────────────────────────────────────┤
│ [假设 H1 低波动溢价] ─▶ [Run run_…9Z] ─▶ [Trial #42 IS] ─▶ [Trial #88 HOLDOUT] ─▶ [因子 lowvol_20] ─▶ [策略 v7] │
│  statement/rationale     goal·发起人·日期   ICIR .52           ICIR .31 PSEUDO_OOS     approved 11-08     production? │
├ IC 衰减（lag 1..20）────────────────┬ 生命周期事件 ─────────────────────────────┤
│ ▇▆▅▄▃▂▂▁▁ …                         │ 11-08 16:40 candidate → approved  apv_… 理由… │
│                                     │ 11-08 16:20 draft → candidate     registry   │
├ 健康度（production 才显示）：IC20 · PSI · 最近评估时间 ───────────────────────────┤
└──────────────────────────────────────────────────────────────────────────────┘
```

- **来源链路**（`LineageFlow`）：结构固定为线性链，所以用 flex 加 SVG 连接线自己实现，不引入图可视化库。每个节点是一张可以点击的卡片：
  - 假设节点：展开显示 rationale；
  - Run 节点：跳到 `/runs/:id`；
  - Trial 节点：跳到 `/runs/:id?tab=trials&trial=`；
  - HOLDOUT trial 节点：只对人类会话显示；
  - 策略版本节点：显示 effective_from。

  来源缺失时（例如因子是手工导入的），对应节点显示虚线框“无记录”。
- **API**：`GET /factors`、`GET /factors/{id}`、`POST /factors/{id}/promote|retire`；SSE 通过通知流的 `approval.decided` 让相关 query 失效。
- **状态**：某一列为空时显示浅色占位“暂无”；整个因子库为空时引导“先发起一次研究”。

### 6.6 回测报告（`/backtests/:id`）

```
┌ 回测 bt_01… · 来自 run_…9Z · IS+HOLDOUT · 证据 PSEUDO_OOS                         ┐
│ 年化 12.4% │ 波动 18.1% │ Sharpe 0.69 │ 最大回撤 −21.3% │ 年换手 4.6x │ 容量 ¥3.2 亿 │
├ NAV vs 基准（IS 与 HOLDOUT 之间画分界竖线，embargo 区间用灰色带标出）─────────────────┤
├ 回撤（面积图，跌色）──────────────────────────────────────────────────────────────┤
├ 分层收益 Q1..Q5（按正负着色）──────────┬ 成本拆解（水平堆叠条：佣金 · 印花税 · 滑点）────┤
├ manifest：git e1e1819 · 代码指纹 · 快照 b3:9f2c… · 模型 · 缓存键（每项都带复制按钮）────┤
```

- **API**：`GET /backtests/{id}`。如果响应中 `downsampled: true`，在图表角落标注“周频降采样”。
- 合成数据时显示红色横幅；NAV 和回撤两张图联动 dataZoom，十字线同步。

### 6.7 模拟盘（`/paper/:acct?`）★

```
┌ 模拟盘 paper-main ▾  策略 v7（3 个因子）  日期 [2026-11-06 ▾]                                   ┐
├ ┌ Kill-switch ──────────────────────────┐ ┌ 对账 ───────────────────┐ ┌ 指标 ─────────────────┐ │
│ │ ● 交易正常                              │ │ ✓ 11-06 一致  16:39      │ │ 累计 +6.2% / 基准 +1.9% │ │
│ │ （触发后：⛔ 交易已暂停 · 原因：对账不一致  │ │ （不一致时列出差异表）     │ │ 超额 +4.3% · MDD −5.1% │ │
│ │  · risk_engine · 11-05 16:38）          │ │                         │ │ 今日 +0.42%            │ │
│ │ [手动暂停]（admin） [申请解除]（admin）    │ └─────────────────────────┘ └───────────────────────┘ │
│ └────────────────────────────────────────┘                                                    │
├ NAV vs 基准 ─────────────────────────────────────────────────── [1M|3M|6M|全部] [净值|超额] ────┤
│ 策略 NAV（蓝线）· 基准（灰线）· 超额（下方单独一张小图，不使用双轴）                               │
├ [持仓 12] [订单 8] [成交 7] ─────────────────────────────────────────────────────────────────┤
│ 代码     名称   数量    冻结(T+1)  成本    现价    市值         权重   浮盈                      │
│ 600519  贵州茅台  100   100      1,650   1,702  ¥170,200.00  8.1%  +3.15%                   │
└───────────────────────────────────────────────────────────────────────────────────────────────┘
```

- **API**：`GET /paper/{acct}/nav|positions|orders|trades|reconciliation`、`GET /ops/kill-switch`、`POST /ops/kill-switch/engage`、`POST /ops/kill-switch/release-request`；SSE 通过通知流的 `kill_switch.changed` 和 `pipeline.status` 让相关 query 失效。
- **Kill-switch 面板**：
  - 触发时整块显示红底，标明触发者（`risk_engine` 或某位用户）、原因和时间。
  - “申请解除”调用 `POST /ops/kill-switch/release-request`，创建一条 `ops_action(release_kill_switch)` 审批，按钮旁说明“需要 admin 审批并且对账通过”。这条审批只能由 admin 发起和决策，不属于 Agent 建议。如果已经有 pending 的解除审批，显示链接跳到审批中心。
  - “手动暂停”调用 `POST /ops/kill-switch/engage`，需要二次确认，并填写理由。
  - 状态来自 `GET /ops/kill-switch`（`engaged`、`reason`、`engaged_at`、`engaged_by`、`release_approval_id`）。
  - 非 admin 用户看不到这两个按钮，只能看到状态。
- **对账面板**：`ok` 显示绿色 ✓（同时带图标和文字）；`mismatch` 显示 critical 颜色，并展开差异表（symbol、账簿数量、券商数量、差额）；`pending` 显示 ⟳。
- **订单表**：状态 chip（已提交、已成交、部分成交、已拒绝、已撤销）；被拒绝的订单显示 `reject_reason`，例如“涨停无法买入”或“T+1 冻结”。
- **成交表**：显示 fee 明细的 tooltip（佣金和印花税）。
- **状态**：还没有 strategy_version 时，空态说明“尚无生产策略。到因子库发起上线”；非交易日选择器会禁用对应日期（使用交易日历接口，或者只允许选择 nav 序列中出现过的日期）。

### 6.8 告警与日报（`/alerts`）

```
┌ 告警与日报 ─────────────────────────────────────────────────────────────────────────┐
├ 日期 ───────┬ [日报] [告警] ──────────────────────────────────────────────────────────┤
│ 11-06 ✓     │ 日报 2026-11-06   faithfulness ✓                                         │
│ 11-05 ✓     │ ## 今日概况 …（Markdown 中的数字可以点击，悬停显示来源，例如 nav_daily 或 alert）│
│ 11-04 ✗     │ ## 告警诊断 …（引用的证据 id 链接到告警）                                     │
│ …           │ ┌ Agent 建议（需要审批）──────────────────────────────────────────────┐   │
│             │ │ pause_strategy v7 · 依据 alert#31   pending  [去审批中心 →]          │   │
│             │ └────────────────────────────────────────────────────────────────────┘   │
└─────────────┴───────────────────────────────────────────────────────────────────────┘
告警 tab：[状态 open ▾][级别 P1/P2 ▾]  表格：级别 · 标题 · 首次 · 最近 · 次数 · 状态 · [确认]（approver）
```

- **API**：`GET /reports/daily`、`GET /reports/daily/{date}`、`GET /alerts`、`POST /alerts/{id}/ack`（乐观更新）。
- faithfulness 没有通过的日报：显示 critical 横幅并列出问题。日期列表中对应日期显示 ✗。
- Agent 建议**只能跳到审批中心**，本页不提供“执行”按钮（spec §5.4：图本身不执行任何动作）。建议动作只来自白名单 `pause_strategy | retire_factor | rerun_data_update | rerun_pipeline | reduce_gross | no_action`，不会出现 `release_kill_switch`。

### 6.9 设置与用量（`/settings`）

| tab | 内容 | 权限 |
|---|---|---|
| 用量 | `GET /llm/usage`：按天堆叠的柱状图（按 provider 或角色分组，可切换）；表格按角色和模型分组，显示调用次数、各类 token、费用、回退次数；显示今日花费与上限的对比 | 全部 |
| 模型路由 | `GET /settings/llm-routing`：只读表格，列出角色、回退链、thinking/effort、knowledge_cutoff；注明“修改需要编辑 configs/llm.yaml 并重新部署” | 全部 |
| 预算 | `GET/PATCH /admin/settings`：默认 run 预算、每日上限、`approval.allow_self`、审批 TTL；保存前二次确认 | admin |
| API Keys | `GET/POST/DELETE /me/api-keys`：创建后用对话框**只显示一次**明文，提供复制按钮并提示“关闭后不能再查看”；可以设置 scope（`holdout:read` 默认不勾选，旁边说明风险） | 全部 |
| 用户 | `GET/POST/PATCH /admin/users`：创建用户、修改角色、禁用用户 | admin |
| 偏好 | 主题（跟随系统、浅色、深色）；**涨跌配色（红涨绿跌是默认值，可切换为绿涨红跌）**；数字密度 | 全部（存在 localStorage） |

---

## 7. 组件规约（选摘）

| 组件 | Props | 说明 |
|---|---|---|
| `<Num value kind="pct|bp|int|float|cny|usd|pvalue" digits? />` | | 统一调用 `lib/format.ts`；`null` 显示为“—”；使用 tabular-nums |
| `<SignedNum value kind />` | | 根据正负选择 `--up` 或 `--down` 颜色，**同时**显示 ▲/▼ 和正负号，保证不只靠颜色传达含义 |
| `<EvidenceBadge level />` | IS、PSEUDO_OOS、TRUE_OOS | tooltip 解释三种等级的含义（spec §2） |
| `<IntegrityBanner kind="synthetic|contamination|biases|kill_switch|faithfulness" details />` | | 见 §8.4 |
| `<EChart option theme loading onEvents />` | | 用 ResizeObserver 自适应大小；主题切换时重新 `setOption`；卸载时调用 `dispose` |

---

## 8. 可视化规范

### 8.1 颜色 token

图表配色沿用 dataviz 参考调色板（已经过 CVD 校验），用 CSS 变量定义在 `styles/globals.css` 的 `@theme` 中，分为浅色和深色两套取值。用户切换主题时设置 `:root[data-theme=dark]`，同时支持 `prefers-color-scheme`。

| 角色 | 浅色 | 深色 | 用途 |
|---|---|---|---|
| `--surface-1` | `#fcfcfb` | `#1a1a19` | 图表背景 |
| `--ink-1` / `--ink-2` / `--ink-muted` | `#0b0b0b` / `#52514e` / `#898781` | `#ffffff` / `#c3c2b7` / `#898781` | 文字和坐标轴 |
| `--grid` / `--axis` | `#e1e0d9` / `#c3c2b7` | `#2c2c2a` / `#383835` | 网格线和基线 |
| `--series-1..8` | `#2a78d6 #eb6834 #1baf7a #eda100 #e87ba4 #008300 #4a3aa7 #e34948` | `#3987e5 #d95926 #199e70 #c98500 #d55181 #008300 #9085e9 #e66767` | 分类色，**按固定顺序分配，不循环使用** |
| `--bench` | `#898781` | `#898781` | 基准线，灰色，2px 虚线 |
| `--status-good/warning/serious/critical` | `#0ca30c #fab219 #ec835a #d03b3b` | 同左 | 只用于状态，永远搭配图标和文字 |
| `--up` / `--down` | 见 §8.2 | | 涨跌颜色 |

规则：
- 策略 NAV 固定用 `series-1`；
- 同一张图不使用双 y 轴（超额收益单独画一张小图）；
- 2 条或更多序列时必须有图例，不超过 4 条时直接在线条末端标注；
- 文字使用 ink 颜色，不使用序列颜色；
- ECharts 主题在 `charts/theme.ts` 中运行时读取 CSS 变量生成 `ap-light` 和 `ap-dark`，主题变化时重新注册；
- 折线宽 2px；hover 时显示十字线和 tooltip（tooltip 里的数字也走 `format.ts`）；
- 每个图表都提供“表格视图”切换，满足可访问性要求。

### 8.2 涨跌配色

```ts
// lib/preferences.ts
export type ColorConvention = "cn" | "intl";       // 默认 "cn"：红涨绿跌
const UPDOWN = {
  cn:   { up: "var(--series-8)", down: "var(--series-6)" },   // 红 #e34948 / 绿 #008300
  intl: { up: "var(--series-6)", down: "var(--series-8)" },
};
```

- 偏好存在 `localStorage['ap.colorConvention']` 中，通过 `<html data-updown="cn|intl">` 切换 `--up` 和 `--down` 两个变量。所有组件只引用 `--up` 和 `--down`，不直接引用红色或绿色。
- 红和绿对红绿色盲用户不友好，因此**所有涨跌显示都必须同时带正负号和 ▲/▼**。分层收益柱和回撤面积图通过 tooltip 文字补充说明。

### 8.3 数字格式（`lib/format.ts`，基于 `Intl.NumberFormat('zh-CN')`）

| 函数 | 输入 → 输出 | 规则 |
|---|---|---|
| `pct(x, d=2)` | `0.01234` → `1.23%` | 小数比例乘以 100 |
| `signedPct(x)` | `0.0042` → `+0.42%` | 显式显示正负号；0 显示为 `0.00%` |
| `bp(x, d=1)` | `0.00045` → `4.5 bp` | 用于成本和滑点 |
| `int(x)` | `1287` → `1,287` | 千分位 |
| `float(x, d=3)` | `0.5213` → `0.521` | IC、ICIR 等 |
| `pvalue(p)` | `0.0004` → `<0.001`；`0.018` → `0.018` | |
| `cny(x)` | `170200` → `¥170,200.00`；加 `compact` 时 → `¥17.02 万`、`¥3.2 亿` | |
| `usd(x)` | `0.0061` → `$0.0061`（小于 1 时保留 4 位小数），`4.16` → `$4.16` | LLM 成本 |
| `tokens(n)` | `3200` → `3.2k` | |
| `dur(ms)` | `4100` → `4.1s`，`720000` → `12m` | |
| `tradeDate(d)` | `2026-11-06`（Asia/Shanghai） | 时间戳统一显示为本地时区，tooltip 中显示 UTC |

### 8.4 统一横幅 `IntegrityBanner`

| kind | 颜色和图标 | 文案模板（来自 i18n） | 出现位置 |
|---|---|---|---|
| `synthetic` | critical 红底白字 ⛔ | “当前数据快照为**合成数据**，结果仅供演示，不能上线（spec §8.6）” | Run、审批、因子、回测、模拟盘，凡是用到合成快照的地方 |
| `contamination` | warning 琥珀色 ⚠ | “证据等级 PSEUDO_OOS（污染 {label}）：holdout 结束于 {holdout_end}，模型知识截止日 {cutoff}”；label 为 full、partial、none、unknown，unknown 按 full 展示 | Run 详情、审批卡片、回测、因子详情 |
| `biases` | info 灰蓝色 ⓘ | “已知偏差：{known_biases 列表}” | 同上，可以折叠 |
| `kill_switch` | critical 红色 ⛔ | “交易已暂停：{reason}（{engaged_by}，{time}）” | 全局顶部 |
| `faithfulness` | critical 红色 ⛔ | “报告未通过数字核验：{n} 个问题” | 报告和日报 |

横幅不能被永久关闭（`synthetic`、`contamination`、`kill_switch` 没有关闭按钮），并且使用 `role="alert"`（kill_switch）或 `role="status"`（其他）。

---

## 9. 测试

### 9.1 Vitest 组件测试（jsdom 环境，MSW 模拟 API）

| 对象 | 用例 |
|---|---|
| `format.ts` | 各个函数的边界值：null、负数、极小值、`compact` 模式 |
| `CandidateCard` / `DecisionPanel` | hard_fail 时复选框禁用并显示原因；达到 holdout_limit 后其他选项禁用；理由少于 10 个字符时按钮禁用；`viewer.can_approve=false` 时显示 block_reasons 文案；显示 self_approved 提示 |
| `ConfirmDecisionDialog` | 有污染风险时必须勾选；请求带上 `If-Match` 和 `Idempotency-Key`，请求体使用 `selected_candidate_keys`（register 时带 `names`，默认等于建议名）；412 后自动重新获取，且理由内容保留；409 hard_fail 时在对应卡片标红 |
| `useRunEvents` | 用 fake EventSource（`test/fakeEventSource.ts`）测试：去重（重复的 id 被忽略）、CLOSED 后带 `?last_event_id=` 重连、收到终态 status 后不再重连、schema 不匹配时不崩溃；fixture 的 payload 从 `core/events.py` 导出的 schema 生成 |
| `Markdown` / `TrialRef` | 渲染 `ap://trial/42?field=icir` 链接，点击后 URL 参数变为 `tab=trials&trial=42`；`javascript:` 链接被过滤 |
| `SignedNum` | 在 cn 和 intl 两种配色下颜色变量正确，并且始终显示 ▲/▼ |
| `useCan` / 路由守卫 | researcher 访问 admin tab 时显示 Forbidden；收到 401 时跳转到 `/login?next=` |

覆盖率目标：`lib/` 和 `api/sse/` 不低于 90%，`features/approvals` 不低于 80%。

### 9.2 Playwright 冒烟测试（`pnpm -C web e2e`）

环境：`docker compose -f deploy/compose.yaml -f deploy/compose.e2e.yaml up -d`，其中设置 `AP_LLM_DEFAULT=fake:scripted`、`AP_OFFLINE=1`，使用合成或 qtc 缓存快照。seed 脚本会创建 `e2e-admin` 用户。fake 脚本保证图会依次走到 gate_holdout 和 gate_register。

```ts
test("研究主路径：创建 → 实时时间线 → holdout 审批 → 注册审批 → 报告溯源", async ({ page }) => {
  await login(page, "e2e-admin");
  await page.goto("/runs?new=1");
  await page.getByLabel("研究目标").fill("低波动+动量反转");
  await page.getByRole("button", { name: "创建并查看" }).click();
  await expect(page).toHaveURL(/\/runs\/run_/);
  await expect(page.getByText("plan 开始")).toBeVisible();                         // SSE 实时推送
  await expect(page.getByText(/interrupt gate_holdout/)).toBeVisible({ timeout: 60_000 });
  await page.getByRole("link", { name: "去审批" }).first().click();
  await page.getByRole("checkbox", { name: /候选 #\d+/ }).first().check();
  await page.getByLabel("理由").fill("e2e：经济逻辑清晰，同意查看 holdout");
  await page.getByRole("button", { name: /批准查看 holdout/ }).click();
  await page.getByLabel("我已阅读污染警告与已知偏差").check();
  await page.getByRole("button", { name: "确认批准" }).click();
  await expect(page.getByText("已批准")).toBeVisible();
  // register gate
  await page.goto(page.url().replace(/approvals.*/, "approvals?kind=register"));
  /* …同样的流程… */
  await page.getByRole("link", { name: /run_/ }).click();
  await page.getByRole("tab", { name: "报告" }).click();
  await page.locator("[data-testid=trial-ref]").first().click();
  await expect(page.locator("[data-highlighted=true]")).toBeVisible();          // 数字溯源到 trial
});
```

另外两个用例：hard_fail 候选的复选框是禁用的（fake 脚本中植入了一个前视因子）；断网后恢复，时间线没有重复也没有遗漏（用 `page.route` 中断 SSE 再恢复）。CI 中失败时上传 trace 和截图。

---

## 10. 构建与部署

- 脚本：`pnpm dev`、`pnpm build`（先 `tsc -b`，再 `vite build`）、`pnpm lint`、`pnpm typecheck`、`pnpm test`、`pnpm e2e`、`pnpm gen:api`。CI 的 web job 依次执行 lint、typecheck、vitest、build，以及 openapi drift 检查（见 api.md §7）。
- `deploy/Dockerfile.web`：分两个阶段。第一阶段用 `node:24-alpine` 执行 `pnpm install --frozen-lockfile && pnpm build`；第二阶段用 `nginx:1.27-alpine`，把 `dist/` 复制到 `/usr/share/nginx/html`。

`deploy/nginx.conf`（关键部分）：

```nginx
upstream ap_api { server api:8000; keepalive 16; }

server {
  listen 80;
  root /usr/share/nginx/html;

  # SPA 路由回退
  location / { try_files $uri /index.html; add_header Cache-Control "no-cache"; }
  location /assets/ { add_header Cache-Control "public, max-age=31536000, immutable"; }

  # SSE：两个事件流单独配置
  location ~ ^/api/v1/(runs/[^/]+/events|notifications/stream)$ {
    proxy_pass http://ap_api;
    proxy_http_version 1.1;
    proxy_set_header Connection "";          # 允许 upstream keepalive，不发送 close
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_buffering off;                     # 关键：不缓冲，事件实时下发
    proxy_cache off;
    gzip off;                                # 压缩会导致缓冲
    proxy_read_timeout 1h;                   # 远大于 15 秒的心跳间隔
    proxy_send_timeout 1h;
    chunked_transfer_encoding on;
    add_header X-Accel-Buffering no;
  }

  # 普通 API
  location /api/ {
    proxy_pass http://ap_api;
    proxy_http_version 1.1;
    proxy_set_header Connection "";
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_read_timeout 60s;
    client_max_body_size 1m;
  }

  # 观测端点不对外暴露（Prometheus 在内网直接抓取 api:8000/metrics）
  location ~ ^/(api/v1/)?metrics$ { deny all; }

  add_header X-Content-Type-Options nosniff always;
  add_header Referrer-Policy same-origin always;
  add_header Content-Security-Policy "default-src 'self'; img-src 'self' data:; style-src 'self' 'unsafe-inline'; connect-src 'self'; frame-ancestors 'none'" always;
}
```

注意：nginx 中，一旦某个 location 里写了 `add_header`，就不会再继承 server 级别的 `add_header`。如果需要安全头在所有 location 都生效，要在各 location 中重复声明，或者改用 `include snippets/security-headers.conf`。

---

## 11. 可访问性与国际化

- **中文优先**：所有文案集中放在 `src/i18n/zh-CN.ts`，以带类型的嵌套对象组织，通过 `t("approvals.reason.min", {n: 10})` 访问，key 有类型检查，写错 key 会在编译期报错。枚举值（status、outcome、hard_fail 代码、guardrail code、kind、action）的中文统一放在 `i18n/enums.ts`；遇到未知代码时原样显示英文代码，并上报到控制台。这种结构为以后增加 `en-US` 预留了位置，但首期不做语言切换。
- 不在组件里硬编码中文；ESLint 规则 `no-restricted-syntax` 禁止在 `features/**` 的 JSX 文本中直接写 CJK 字符。
- **键盘**：
  - 所有操作都可以用 Tab 完成，焦点环清晰可见；
  - 审批中心支持 `j`/`k` 在列表中移动、`Enter` 打开详情；不设置一键批准的快捷键，避免误操作；
  - 对话框的焦点陷阱和 Esc 关闭由 Radix 处理，但提交过程中 Esc 被禁用。
- **ARIA**：
  - 时间线使用 `role="log"` 和 `aria-live="polite"`，只播报节点开始和结束以及 interrupt，不播报每一条 llm 和 tool 事件；
  - 状态徽标同时包含文字和图标；
  - 图表容器提供 `aria-label` 摘要（例如“策略 NAV 1.062，基准 1.019，近 90 日”），并提供表格视图切换。
- **对比度**：正文对比度不低于 4.5:1；状态色不单独承载信息；尊重 `prefers-reduced-motion`（关闭脉冲动画和自动滚动的平滑效果）。
- 支持 `forced-colors` 模式：涨跌依靠 ▲/▼ 和正负号区分，图表在这种模式下提供纹理填充的选项。
