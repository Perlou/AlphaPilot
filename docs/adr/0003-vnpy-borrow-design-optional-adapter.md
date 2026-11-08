# ADR-0003：借鉴 vnpy 的设计并提供可选适配器，不以 vnpy 为底座

- 状态：已接受
- 日期：2026-11-08

## 背景

vnpy（VeighNa）是国内最主流的开源量化交易框架，MIT 协议，当前版本 4.5.0，支持 Python 3.10–3.13。它的组成包括：

- 事件引擎和主引擎，加上 Gateway 与 App 插件体系；
- 大量交易接口，如 CTP、XT、IB 等；
- 4.0 新增的 `vnpy.alpha` 模块，包含表达式因子、Alpha158、模型模板和 lab 工作流。

但它也有以下特点：

- 以 Qt 桌面程序为主要形态；
- 实盘接口以 Windows 和 Linux 为主，在 macOS 上受限；
- 开源版**没有 LLM 或 Agent 能力**。

qtc Phase 13 的 `mini_trader` 已经实现了一套与 vnpy 思路一致的精简事件驱动交易系统。

## 决策

- **借鉴设计**：
  - `trading/` 的 Gateway 抽象在概念上与 vnpy 的 BaseGateway 对齐，包括 connect、send_order、cancel_order 以及 on_tick、on_order、on_trade 这类回调；
  - 应用层按插件化思路组织；
  - Alpha158 的一个子集用 AlphaPilot DSL 重写，作为 Agent 挖掘新因子时的**基线因子库**，用于计算新因子的增量价值和新颖度。
- **可选适配器**：`trading/gateways/vnpy_bridge.py` 通过 `pip install alphapilot[vnpy]` 安装，M7 实现。首期只对接 `paper_account`；CTP 和 XT 只保留接口入口，默认关闭。
- **不作为底座**：核心系统不依赖 vnpy，也不引入 Qt。

## 后果

- ✅ 保持 Web 服务形态，在 macOS 上可以开发，并且复用了 qtc 的课程成果。
- ✅ 留有通往实盘的路径，实盘的部署环境可以是 Windows 或 Linux。
- ⚠️ 适配器需要在 vnpy 的事件模型与 AlphaPilot 的 asyncio 事件引擎之间做桥接，这部分作为 M7 的风险项。
