# ADR-0001：Postgres 作为唯一状态存储，不引入 Redis

- 状态：已接受
- 日期：2026-11-08

## 背景

AlphaPilot 需要以下几样东西：

1. 一个任务队列，用于执行研究任务、数据更新和每日流水线；
2. 一个实时推送通道，用来把 Agent 执行轨迹通过 SSE 推给前端；
3. LangGraph 的 checkpoint 存储；
4. 业务数据存储，包括 runs、trials、factors、approvals 和审计日志。

常见做法是 Postgres 加 Redis（队列和 pub/sub），再引入 Celery 或 RQ。本机两者都已安装。

AlphaPilot 的负载特点：单团队使用，同时运行的研究任务通常不超过 5 个，每天几十个 job，峰值事件推送每秒几十条。

## 决策

**只使用 Postgres 16。**

- **队列**：用 `jobs` 表配合 `SELECT … FOR UPDATE SKIP LOCKED` 领取任务；用 `unique_key` 保证幂等；用 `run_after` 实现延迟和退避。
- **推送**：写入 `agent_events` 时由 trigger 执行 `pg_notify('run_events', '{"r":"<run_id>","i":<max_id>}')`，payload 是一个只带 id 的小 JSON（详见 `docs/design/data-model.md` §7）。API 保持一个 `LISTEN` 连接，按 run 分发。断线后按 `Last-Event-ID` 从表里补发。
- **Checkpoint**：用 `langgraph-checkpoint-postgres` 的 `PostgresSaver`，放在独立的 `lg` schema。
- **抽象**：定义 `EventBus` 和 `JobQueue` 两个 Protocol，Postgres 是它们的第一个实现。

## 备选方案

| 方案 | 不选的原因 |
|---|---|
| Redis + Celery/RQ | 多一个有状态组件。job 状态与业务状态分处两地，“审批写库成功、投递 resume 失败”这类不一致需要额外补偿。Celery 对 asyncio 的支持也一般 |
| Redis Streams 只做推送 | 推送量很小，NOTIFY 足够；补发本来就要从持久化的表里读 |
| 消息队列（RabbitMQ / NATS） | 规模远不需要 |

## 后果

- ✅ 审批决策、投递 resume 任务和写审计日志可以放在**同一个事务**里完成，保证一致。
- ✅ 部署和备份只需要处理一个数据库。
- ✅ trace（agent_events）既是 SSE 的数据源，也是可查询的历史记录。
- ⚠️ NOTIFY 的 payload 上限是 8000 字节，所以只发 id，消费方再回表读取。
- ⚠️ 连接数：每个 API 实例占用一个专用的 LISTEN 连接，需要做好连接池规划。
- ⚠️ 吞吐上限比 Redis 低。如果将来吞吐量增长两个数量级，可以通过 `EventBus` 和 `JobQueue` 接口换成 Redis 实现，业务代码不用改。
