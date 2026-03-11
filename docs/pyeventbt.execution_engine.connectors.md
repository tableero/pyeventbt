# pyeventbt.execution_engine.connectors

## Package
`pyeventbt.execution_engine.connectors`

## Purpose
Contains the two concrete `IExecutionEngine` implementations: one for backtesting (simulated MT5) and one for live trading (real MT5 broker). Both connectors share the same interface but differ in how they execute trades, track state, and emit fill events.

## Tags
`connectors`, `execution`, `mt5`, `backtesting`, `live-trading`

## Modules

| Module | Description |
|---|---|
| `mt5_simulator_execution_engine_connector` | Full MT5 simulator for backtesting (~1000+ lines). Manages in-memory positions, orders, deals, account state, margin, commission, swap, and currency conversion. |
| `mt5_live_execution_engine_connector` | Real MT5 broker connector (~723 lines). Sends orders via the `MetaTrader5` Python package, retries deal confirmation up to 5 seconds, supports partial close and futures continuous trading. |

## Internal Architecture

Both connectors implement `IExecutionEngine` and follow the same flow:

1. `_process_order_event` routes by `order_type`:
   - `"MARKET"` --> `_send_market_order()` --> immediate fill --> `FillEvent`
   - `"LIMIT"` / `"STOP"` --> `_send_pending_order()` --> added to pending list
   - `"CONT"` --> `execute_desired_continuous_trade()` (live only)
2. `_update_values_and_check_executions_and_fills` is called on each bar:
   - **Simulator**: updates floating P&L, checks pending order triggers, checks SL/TP hits, calculates swap/commission
   - **Live**: currently a no-op (`pass`)
3. Account query methods (`_get_account_balance`, etc.) either read from in-memory state (simulator) or call `mt5.account_info()` (live).

**Key architectural differences:**

| Aspect | Simulator | Live |
|---|---|---|
| State storage | In-memory dicts (`open_positions`, `pending_orders`, `executed_deals`) | MT5 server-side |
| Fill confirmation | Immediate (synchronous) | Retry loop up to 5s polling `history_deals_get` |
| P&L calculation | Manual currency conversion, swap, commission logic | MT5 server calculates |
| Pending order fills | Checked each bar in `_update_values_and_check_executions_and_fills` | Checked via `_check_if_pending_orders_filled` (bar-driven) |
| MT5 dependency | Uses `Mt5SimulatorWrapper` (mock) | Uses real `MetaTrader5` package (conditionally imported) |

## Cross-Package Dependencies

| Dependency | Usage |
|---|---|
| `pyeventbt.broker.mt5_broker` | `OrderSendResult`, `TradePosition`, `TradeRequest`, `TradeOrder`, `TradeDeal`, `AccountInfo`, `Mt5SimulatorWrapper`, `SharedData` |
| `pyeventbt.portfolio.core.entities` | `OpenPosition`, `PendingOrder` |
| `pyeventbt.events.events` | `BarEvent`, `FillEvent`, `OrderEvent`, `SignalType` |
| `pyeventbt.data_provider.core.interfaces` | `IDataProvider` |
| `pyeventbt.utils.utils` | `Utils`, `check_platform_compatibility` |
| `MetaTrader5` | External package, conditionally imported in live connector |

## Gaps & Issues

1. **`__init__.py` is empty.** No re-exports from the connectors package.
2. **Simulator margin check is incomplete.** Explicit TODO: "Add Check if margin is enough to keep positions opened."
3. **Live `_update_values_and_check_executions_and_fills` is `pass`.** Pending order fills and SL/TP monitoring between bar events rely on MT5 server-side processing and are not actively checked by the connector.
4. **No shared base class.** Both connectors duplicate validation logic (`_check_common_trade_values`) rather than sharing it via a common base.
5. **No automated tests.** Complex financial logic (P&L, swap, commission, currency conversion) in the simulator has no test coverage.
