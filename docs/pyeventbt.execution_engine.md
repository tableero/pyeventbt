# pyeventbt.execution_engine

## Package
`pyeventbt.execution_engine`

## Purpose
Top-level package for the execution engine module. Responsible for processing `OrderEvent`s into actual trades (simulated or live), emitting `FillEvent`s back onto the shared event queue, and providing account/position query methods to other components. The package supports two execution backends: a full MT5 simulator for backtesting and a real MT5 connector for live trading.

## Tags
`execution`, `order-management`, `position-management`, `mt5`, `backtesting`, `live-trading`

## Modules

| Submodule | Description |
|---|---|
| `core.configurations.execution_engine_configurations` | Pydantic config models (`MT5SimulatedExecutionConfig`, `MT5LiveExecutionConfig`) |
| `core.interfaces.execution_engine_interface` | `IExecutionEngine` interface with ~20+ abstract methods |
| `connectors.mt5_simulator_execution_engine_connector` | Full MT5 backtest simulator (~1000+ lines) |
| `connectors.mt5_live_execution_engine_connector` | Real MT5 broker connector (~723 lines) |
| `services.execution_engine_service` | `ExecutionEngine` facade that delegates to the selected connector |

## Internal Architecture

The execution engine follows a facade + connector pattern:

1. **`ExecutionEngine`** (service layer) is the single entry point used by `TradingDirector` and `PortfolioHandler`. It holds a reference to one concrete connector.
2. At construction, the config type determines which connector is instantiated:
   - `MT5SimulatedExecutionConfig` --> `Mt5SimulatorExecutionEngineConnector`
   - `MT5LiveExecutionConfig` --> `Mt5LiveExecutionEngineConnector`
3. All ~20 `IExecutionEngine` methods are delegated 1:1 from the facade to the active connector.
4. The facade adds a `enable_trading` / `disable_trading` toggle that gates `_process_order_event`.

```
OrderEvent --> ExecutionEngine._process_order_event()
                  |  (if trading enabled)
                  +--> Connector._process_order_event()
                       |
                       +--> MARKET: _send_market_order() --> FillEvent --> queue
                       +--> LIMIT/STOP: _send_pending_order() --> pending list
                       +--> CONT: execute_desired_continuous_trade() (live only)
```

## Cross-Package Dependencies

| Dependency | Usage |
|---|---|
| `pyeventbt.events.events` | `BarEvent`, `OrderEvent`, `FillEvent`, `SignalType` |
| `pyeventbt.broker.mt5_broker` | `OrderSendResult`, `TradePosition`, `TradeRequest`, `TradeOrder`, `TradeDeal`, `AccountInfo`, `Mt5SimulatorWrapper`, `SharedData` |
| `pyeventbt.portfolio.core.entities` | `OpenPosition`, `PendingOrder` |
| `pyeventbt.data_provider.core.interfaces` | `IDataProvider` for bar data access |
| `pyeventbt.utils.utils` | `Utils`, `TerminalColors`, `colorize`, `check_platform_compatibility` |
| `MetaTrader5` | External MT5 Python package (live connector only, imported conditionally) |
| `queue.Queue` | Shared event queue |
| `decimal.Decimal` | Precise numeric handling for volumes, prices, balances |

## Gaps & Issues

1. **`__init__.py` is empty.** No public re-exports; users must import from submodules directly.
2. **Margin check TODO.** The simulator connector has an explicit TODO comment: "Add Check if margin is enough to keep positions opened."
3. **Live `_update_values_and_check_executions_and_fills` is a no-op.** The method body is `pass`, meaning pending order fills and SL/TP checks are not proactively detected in live mode between bar events.
4. **Typo in log message.** `ExecutionEngine._process_order_event` logs "Trading is disbled" (should be "disabled").
5. **No unit tests.** The simulator connector is ~1000+ lines of complex logic (P&L, swap, commission, currency conversion) with no automated test coverage.
